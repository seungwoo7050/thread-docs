## `test(operations): 자원·격리·삭제 보호·진단 검증`

diff --git a/Makefile b/Makefile
index 13a8a93..52da767 100644
--- a/Makefile
+++ b/Makefile
@@ -5,12 +5,12 @@ PROJECT_NAME ?= container-stack
 WAIT_TIMEOUT ?= 300
 BACKUP_DIR ?=
 NEW_SECRETS_DIR ?=
-DESTROY_CONFIRM ?=
 DIAGNOSTICS_DIR ?= diagnostics/$(PROJECT_NAME)
+DESTROY_CONFIRM ?=
 
 COMPOSE_RUN := $(COMPOSE) --project-name $(PROJECT_NAME) --env-file $(ENV_FILE) -f $(COMPOSE_FILE)
 
-.PHONY: up start-database start-application down build logs ps clean fclean test config smoke bootstrap-test e2e persistence backup restore backup-restore-test rotate-secrets rotation-test config-strict diagnostics
+.PHONY: up start-database start-application down build logs ps clean fclean test config config-strict smoke bootstrap-test e2e persistence backup restore backup-restore-test rotate-secrets rotation-test diagnostics operations-test
 
 up:
 	python3 tools/start_stack.py start --project "$(PROJECT_NAME)" --env-file "$(ENV_FILE)" --wait-timeout "$(WAIT_TIMEOUT)"
@@ -91,3 +91,6 @@ rotation-test:
 
 diagnostics:
 	python3 tools/diagnose_stack.py --project "$(PROJECT_NAME)" --env-file "$(ENV_FILE)" --output "$(DIAGNOSTICS_DIR)"
+
+operations-test:
+	python3 tests/runtime_stack.py operations
diff --git a/tests/runtime_stack.py b/tests/runtime_stack.py
index 7a35b3d..ab38e5c 100644
--- a/tests/runtime_stack.py
+++ b/tests/runtime_stack.py
@@ -1356,23 +1356,246 @@ if ($text === false || !preg_match($pattern, $text, $matches) || !hash_equals($p
                     raise StackError("Compose 로그에 자격증명 값이 포함되었습니다")
         print("secret rotation, ambiguous failures, rollback, and retry passed")
 
-    def collect_diagnostics(self) -> Path:
-        destination = self.diagnostics_dir
-        if destination is None:
-            destination = Path(tempfile.mkdtemp(prefix="container-stack-diagnostics-"))
-            destination.chmod(0o700)
-        else:
-            destination.mkdir(parents=True, exist_ok=True)
-        commands = {
-            "compose-ps.txt": ("ps", "--all"),
-            "compose-logs.txt": ("logs", "--no-color", "--timestamps"),
-            "compose-config.txt": ("config", "--no-interpolate"),
+    def verify_operations(self) -> None:
+        self.start()
+        expected = {
+            "nginx": {
+                "memory": 128 * 1024 * 1024,
+                "nano_cpus": 500_000_000,
+                "pids": 64,
+                "signal": "SIGQUIT",
+                "timeout": 15,
+                "networks": {f"{self.project}_frontend"},
+            },
+            "wordpress": {
+                "memory": 512 * 1024 * 1024,
+                "nano_cpus": 1_000_000_000,
+                "pids": 256,
+                "signal": "SIGQUIT",
+                "timeout": 30,
+                "networks": {
+                    f"{self.project}_frontend",
+                    f"{self.project}_backend",
+                },
+            },
+            "mariadb": {
+                "memory": 512 * 1024 * 1024,
+                "nano_cpus": 1_000_000_000,
+                "pids": 256,
+                "signal": "SIGTERM",
+                "timeout": 60,
+                "networks": {f"{self.project}_backend"},
+            },
+        }
+        container_ids: dict[str, str] = {}
+        for service, policy in expected.items():
+            inspected = self.inspect_service(service)
+            container_ids[service] = str(inspected["Id"])
+            host = inspected["HostConfig"]
+            config = inspected["Config"]
+            actual = {
+                "memory": host["Memory"],
+                "nano_cpus": host["NanoCpus"],
+                "pids": host["PidsLimit"],
+                "signal": config["StopSignal"],
+                "timeout": config["StopTimeout"],
+                "networks": set(inspected["NetworkSettings"]["Networks"]),
+            }
+            if actual != policy:
+                raise StackError(
+                    f"{service} 실행 정책이 Compose 설정과 다릅니다: {actual!r}"
+                )
+            log_config = host["LogConfig"]
+            if log_config["Type"] != "json-file" or log_config["Config"] != {
+                "max-file": "3",
+                "max-size": "10m",
+            }:
+                raise StackError(f"{service} 로그 회전 정책이 적용되지 않았습니다")
+            if "no-new-privileges:true" not in (host["SecurityOpt"] or []):
+                raise StackError(f"{service} 권한 상승 차단 정책이 적용되지 않았습니다")
+            expected_nofile = {
+                "nginx": (1024, 4096),
+                "wordpress": (1024, 4096),
+                "mariadb": (4096, 65536),
+            }[service]
+            nofile = next(
+                (item for item in host["Ulimits"] if item["Name"] == "nofile"), None
+            )
+            if nofile is None or (
+                nofile["Soft"], nofile["Hard"]
+            ) != expected_nofile:
+                raise StackError(f"{service} 파일 디스크립터 제한이 적용되지 않았습니다")
+
+        network_policies = {
+            "frontend": (False, {container_ids["nginx"], container_ids["wordpress"]}),
+            "backend": (True, {container_ids["wordpress"], container_ids["mariadb"]}),
+        }
+        for name, (expected_internal, expected_members) in network_policies.items():
+            network = subprocess.run(
+                ["docker", "network", "inspect", f"{self.project}_{name}"],
+                check=True,
+                text=True,
+                capture_output=True,
+                timeout=PROCESS_TIMEOUT_SECONDS,
+            )
+            inspected_network = json.loads(network.stdout)[0]
+            actual_members = set((inspected_network.get("Containers") or {}).keys())
+            if inspected_network.get("Internal") is not expected_internal:
+                raise StackError(f"{name} 네트워크의 내부망 정책이 예상과 다릅니다")
+            if actual_members != expected_members:
+                raise StackError(f"{name} 네트워크의 연결 서비스가 예상과 다릅니다")
+
+        refused = subprocess.run(
+            [
+                "make",
+                "--silent",
+                "fclean",
+                f"PROJECT_NAME={self.project}",
+                f"ENV_FILE={self.env_file}",
+            ],
+            cwd=ROOT,
+            text=True,
+            capture_output=True,
+            timeout=PROCESS_TIMEOUT_SECONDS,
+        )
+        if refused.returncode != 2 or "DESTROY_CONFIRM" not in refused.stderr:
+            raise StackError("fclean이 명시적인 프로젝트 이름 확인 없이 실행될 수 있습니다")
+        if self.fetch("/healthz").strip() != "ok":
+            raise StackError("삭제 거부 뒤 실행 중인 스택이 손상되었습니다")
+
+        log_secret = self.credential_values["wp_user_password.txt"]
+        self.fetch(f"/?diagnostic_token={log_secret}")
+        unreadable_secret = self.temp / "wp_user_password.txt"
+        unreadable_output = self.temp / "unreadable-secret-diagnostics"
+        unreadable_command = [
+            sys.executable,
+            str(ROOT / "tools" / "diagnose_stack.py"),
+            "--project",
+            self.project,
+            "--env-file",
+            str(self.env_file),
+            "--output",
+            str(unreadable_output),
+        ]
+        unreadable_secret.chmod(0)
+        try:
+            refused_unredacted = subprocess.run(
+                unreadable_command,
+                cwd=ROOT,
+                text=True,
+                capture_output=True,
+                timeout=PROCESS_TIMEOUT_SECONDS,
+            )
+        finally:
+            unreadable_secret.chmod(0o600)
+        if (
+            refused_unredacted.returncode != 2
+            or "가릴 비밀값을 읽을 수 없습니다" not in refused_unredacted.stderr
+            or unreadable_output.exists()
+        ):
+            raise StackError("진단 도구가 읽지 못한 비밀값을 제외한 채 계속 실행했습니다")
+
+        diagnostics = self.temp / "operations-diagnostics"
+        diagnostic_command = [
+            sys.executable,
+            str(ROOT / "tools" / "diagnose_stack.py"),
+            "--project",
+            self.project,
+            "--env-file",
+            str(self.env_file),
+            "--output",
+            str(diagnostics),
+        ]
+        subprocess.run(
+            diagnostic_command,
+            cwd=ROOT,
+            check=True,
+            timeout=PROCESS_TIMEOUT_SECONDS,
+        )
+        expected_files = {
+            "versions.txt",
+            "compose-ps.txt",
+            "compose-logs.txt",
+            "compose-model.txt",
+            "container-state.txt",
+        }
+        if {path.name for path in diagnostics.iterdir()} != expected_files:
+            raise StackError("진단 자료 파일 구성이 예상과 다릅니다")
+        if stat.S_IMODE(diagnostics.stat().st_mode) != 0o700:
+            raise StackError("진단 디렉터리 권한이 0700이 아닙니다")
+        combined = ""
+        for path in diagnostics.iterdir():
+            if not path.is_file() or path.is_symlink():
+                raise StackError(f"진단 결과에 일반 파일이 아닌 항목이 있습니다: {path}")
+            if stat.S_IMODE(path.stat().st_mode) != 0o600:
+                raise StackError(f"진단 파일 권한이 0600이 아닙니다: {path}")
+            combined += path.read_text(encoding="utf-8")
+        leaked = [
+            value for value in self.credential_values.values() if value in combined
+        ]
+        if leaked:
+            raise StackError("진단 자료에 비밀값이 남아 있습니다")
+        if "<redacted>" not in combined:
+            raise StackError("진단 자료의 실제 비밀값 제거를 확인하지 못했습니다")
+        for filename in self.credential_values:
+            if str(self.temp / filename) in combined:
+                raise StackError("진단 자료에 비밀 파일 경로가 남아 있습니다")
+        original = {
+            path.name: path.read_bytes() for path in diagnostics.iterdir()
         }
-        for filename, arguments in commands.items():
-            result = self.run_compose(*arguments, capture=True, check=False)
-            (destination / filename).write_text(
-                result.stdout + result.stderr, encoding="utf-8"
+        repeated = subprocess.run(
+            diagnostic_command,
+            cwd=ROOT,
+            text=True,
+            capture_output=True,
+            timeout=PROCESS_TIMEOUT_SECONDS,
+        )
+        if repeated.returncode != 2 or "이미 존재합니다" not in repeated.stderr:
+            raise StackError("진단 도구가 기존 출력 경로 덮어쓰기를 거부하지 않았습니다")
+        if original != {
+            path.name: path.read_bytes() for path in diagnostics.iterdir()
+        }:
+            raise StackError("진단 도구의 덮어쓰기 거부 뒤 기존 결과가 변경되었습니다")
+
+        dangling_target = self.temp / "missing-diagnostics-target"
+        symlink_output = self.temp / "operations-diagnostics-link"
+        symlink_output.symlink_to(dangling_target)
+        symlink_command = [*diagnostic_command[:-1], str(symlink_output)]
+        refused_symlink = subprocess.run(
+            symlink_command,
+            cwd=ROOT,
+            text=True,
+            capture_output=True,
+            timeout=PROCESS_TIMEOUT_SECONDS,
+        )
+        if refused_symlink.returncode != 2 or "이미 존재합니다" not in refused_symlink.stderr:
+            raise StackError("진단 도구가 dangling symlink 출력 경로를 거부하지 않았습니다")
+        if not symlink_output.is_symlink() or dangling_target.exists():
+            raise StackError("진단 도구의 symlink 거부 과정에서 출력 경로가 변경되었습니다")
+        print("runtime limits, network isolation, and private diagnostics passed")
+
+    def collect_diagnostics(self) -> Path:
+        if self.diagnostics_dir is None:
+            destination = Path(tempfile.gettempdir()) / (
+                f"container-stack-diagnostics-{self.project}-{secrets.token_hex(3)}"
             )
+        else:
+            destination = self.diagnostics_dir / self.project
+        subprocess.run(
+            [
+                sys.executable,
+                str(ROOT / "tools" / "diagnose_stack.py"),
+                "--project",
+                self.project,
+                "--env-file",
+                str(self.env_file),
+                "--output",
+                str(destination),
+            ],
+            cwd=ROOT,
+            check=True,
+            timeout=PROCESS_TIMEOUT_SECONDS,
+        )
         print(f"진단 자료: {destination}", file=sys.stderr)
         return destination
 
@@ -1380,7 +1603,7 @@ if ($text === false || !preg_match($pattern, $text, $matches) || !hash_equals($p
         if failed:
             try:
                 self.collect_diagnostics()
-            except OSError as error:
+            except (OSError, subprocess.CalledProcessError) as error:
                 print(f"진단 자료를 저장하지 못했습니다: {error}", file=sys.stderr)
         if self.started and not self.keep:
             self.run_compose(
@@ -1399,7 +1622,14 @@ def parse_arguments() -> argparse.Namespace:
     parser = argparse.ArgumentParser(description="격리된 컨테이너 스택 검증")
     parser.add_argument(
         "scenario",
-        choices=("bootstrap", "e2e", "persistence", "backup-restore", "rotation"),
+        choices=(
+            "bootstrap",
+            "e2e",
+            "persistence",
+            "backup-restore",
+            "rotation",
+            "operations",
+        ),
     )
     parser.add_argument("--keep", action="store_true", help="검사 뒤 프로젝트를 유지합니다")
     parser.add_argument("--diagnostics-dir", type=Path)
@@ -1411,6 +1641,7 @@ def main() -> int:
     try:
         require_command("docker")
         require_command("curl")
+        require_command("make")
         subprocess.run(
             ["docker", "compose", "version"],
             check=True,
@@ -1432,8 +1663,10 @@ def main() -> int:
             stack.verify_e2e()
         elif args.scenario == "backup-restore":
             stack.verify_backup_restore()
-        else:
+        elif args.scenario == "rotation":
             stack.verify_secret_rotation()
+        else:
+            stack.verify_operations()
         failed = False
         return 0
     except (OSError, StackError, subprocess.SubprocessError) as error:
diff --git a/tests/validate_stack.py b/tests/validate_stack.py
index 69fc346..3fca706 100755
--- a/tests/validate_stack.py
+++ b/tests/validate_stack.py
@@ -97,6 +97,25 @@ def validate_compose() -> None:
     for image in ("wordpress:", "mariadb:", "nginx:"):
         if re.search(rf"image:\s*{image}", text):
             fail(f"compose must not use the official {image.rstrip(':')} image directly")
+    if "container_name:" in text:
+        fail("compose services must use project-scoped generated container names")
+    for pattern in (r"WORDPRESS_URL:", r"STACK_IMAGE_PREFIX", r"STACK_IMAGE_TAG"):
+        if not re.search(pattern, text):
+            fail(f"compose does not match {pattern!r}")
+    for required in (
+        "cpus:",
+        "mem_limit:",
+        "pids_limit:",
+        "no-new-privileges:true",
+        "driver: json-file",
+        'max-size: "10m"',
+        'max-file: "3"',
+        "stop_grace_period:",
+    ):
+        if text.count(required) != 3:
+            fail(f"all three services must set the runtime policy: {required}")
+    if not re.search(r"backend:\s+driver: bridge\s+internal: true", text):
+        fail("database network must be an internal bridge")
 
 
 def validate_dockerfiles() -> None:
@@ -160,6 +179,8 @@ def validate_configs() -> None:
             r"fastcgi_pass wordpress:9000",
             r"ssl_certificate",
             r"location = /healthz",
+            r"access_log /dev/stdout",
+            r"error_log /dev/stderr warn",
         ],
     )
     if "http2 on;" in require_file("srcs/requirements/nginx/conf/nginx.conf").read_text():
@@ -219,6 +240,13 @@ def validate_tools() -> None:
             r"rotate_secrets\.py",
             r"^rotation-test:",
             r"runtime_stack\.py rotation",
+            r"^config-strict:",
+            r"config --quiet",
+            r"^diagnostics:",
+            r"diagnose_stack\.py",
+            r"^operations-test:",
+            r"runtime_stack\.py operations",
+            r"DESTROY_CONFIRM",
         ],
     )
     require_text(
@@ -284,6 +312,21 @@ def validate_tools() -> None:
     rotation_tool = require_file("tools/rotate_secrets.py").read_text()
     if re.search(r"auth=/tmp/container-stack-(?:root|app)\.\$\$", rotation_tool):
         fail("rotation database clients must use unpredictable private option files")
+    require_text(
+        "tools/diagnose_stack.py",
+        [
+            r"secret_values",
+            r"def redact",
+            r"0o600",
+            r"0o700",
+            r"--no-interpolate",
+            r"--tail",
+            r"container_state",
+            r"read_private_secret",
+            r"가릴 비밀값을 읽을 수 없습니다",
+        ],
+    )
+    require_executable("tools/diagnose_stack.py")
     require_text(
         "tests/runtime_stack.py",
         [
@@ -328,6 +371,11 @@ def validate_tools() -> None:
             r"rotation-host-files\.ready",
             r"rotation-rollback\.ready",
             r"추가 종료 신호 지연 처리",
+            r"def verify_operations",
+            r"no-new-privileges:true",
+            r"operations-diagnostics",
+            r"unreadable-secret-diagnostics",
+            r"missing-diagnostics-target",
         ],
     )
 


## `test(runtime): 프로세스·비밀값·정리 제어 흐름 강화`

diff --git a/tests/runtime_stack.py b/tests/runtime_stack.py
index ab38e5c..bac892b 100644
--- a/tests/runtime_stack.py
+++ b/tests/runtime_stack.py
@@ -15,6 +15,7 @@ import stat
 import subprocess
 import sys
 import tempfile
+from typing import Any
 import time
 
 
@@ -54,13 +55,23 @@ def write_private(path: Path, value: str) -> None:
         stream.write(value)
         stream.write("\n")
     if stat.S_IMODE(path.stat().st_mode) != 0o600:
-        raise StackError(f"비밀 파일 권한이 0600이 아닙니다: {path}")
+        raise StackError(f"비공개 파일 권한이 0600이 아닙니다: {path}")
 
 
 def replace_private(path: Path, value: str) -> None:
-    temporary = path.with_name(f".{path.name}.{secrets.token_hex(4)}")
-    write_private(temporary, value.rstrip("\n"))
-    os.replace(temporary, path)
+    temporary = path.with_name(f".{path.name}.{secrets.token_hex(6)}")
+    descriptor = os.open(temporary, os.O_WRONLY | os.O_CREAT | os.O_EXCL, 0o600)
+    try:
+        with os.fdopen(descriptor, "w", encoding="utf-8") as stream:
+            descriptor = -1
+            stream.write(value)
+            stream.flush()
+            os.fsync(stream.fileno())
+        os.replace(temporary, path)
+    finally:
+        if descriptor >= 0:
+            os.close(descriptor)
+        temporary.unlink(missing_ok=True)
 
 
 class RuntimeStack:
@@ -166,17 +177,19 @@ class RuntimeStack:
                 timeout=timeout,
             )
         except subprocess.TimeoutExpired as error:
+            operation = arguments[0] if arguments else "command"
             raise StackError(
-                f"Compose 명령이 {timeout}초 안에 끝나지 않았습니다"
+                f"Compose {operation} 명령이 {timeout}초 안에 끝나지 않았습니다"
             ) from error
 
-    def _run_start(
+    def _start_command(
         self,
         action: str,
         *,
         build: bool = False,
-        check: bool = True,
-    ) -> subprocess.CompletedProcess[str]:
+        pause_after: str | None = None,
+        pause_ready_file: Path | None = None,
+    ) -> list[str]:
         command = [
             sys.executable,
             str(ROOT / "tools" / "start_stack.py"),
@@ -190,9 +203,21 @@ class RuntimeStack:
         ]
         if build:
             command.append("--build")
+        if pause_after is not None and pause_ready_file is not None:
+            command.extend(("--pause-after", pause_after))
+            command.extend(("--pause-ready-file", str(pause_ready_file)))
+        return command
+
+    def _run_start(
+        self,
+        action: str,
+        *,
+        build: bool = False,
+        check: bool = True,
+    ) -> subprocess.CompletedProcess[str]:
         try:
             result = subprocess.run(
-                command,
+                self._start_command(action, build=build),
                 cwd=ROOT,
                 text=True,
                 capture_output=True,
@@ -600,7 +625,8 @@ class RuntimeStack:
 
         self.run_compose("down", "--remove-orphans", "--timeout", "20")
         self.run_compose("up", "--detach", "--wait", "--wait-timeout", "240")
-        if self.project_volumes() != initial_volumes:
+        recreated_volumes = self.project_volumes()
+        if recreated_volumes != initial_volumes:
             raise StackError("서비스 재생성 과정에서 영구 볼륨이 교체되었습니다")
         self._verify_persistent_values(
             post_id=post_id,
@@ -1599,23 +1625,28 @@ if ($text === false || !preg_match($pattern, $text, $matches) || !hash_equals($p
         print(f"진단 자료: {destination}", file=sys.stderr)
         return destination
 
-    def close(self, *, failed: bool) -> None:
+    def close(self, *, failed: bool) -> list[str]:
+        failures: list[str] = []
         if failed:
             try:
                 self.collect_diagnostics()
-            except (OSError, subprocess.CalledProcessError) as error:
-                print(f"진단 자료를 저장하지 못했습니다: {error}", file=sys.stderr)
+            except Exception as error:
+                failures.append(f"진단 자료: {error}")
         if self.started and not self.keep:
-            self.run_compose(
-                "down", "--volumes", "--remove-orphans", "--timeout", "20", check=False
-            )
+            try:
+                result = self.run_compose(
+                    "down", "--volumes", "--remove-orphans", "--timeout", "20",
+                    capture=True, check=False,
+                )
+                if result.returncode != 0:
+                    failures.append(result.stderr.strip() or "Compose 자원 정리 실패")
+            except Exception as error:
+                failures.append(f"Compose 자원 정리: {error}")
         if self.keep:
-            print(
-                f"검사용 프로젝트를 유지합니다: {self.project} ({self.temp})",
-                file=sys.stderr,
-            )
+            print(f"검사용 프로젝트를 유지합니다: {self.project} ({self.temp})", file=sys.stderr)
         else:
             shutil.rmtree(self.temp, ignore_errors=True)
+        return failures
 
 
 def parse_arguments() -> argparse.Namespace:
@@ -1648,19 +1679,23 @@ def main() -> int:
             stdout=subprocess.DEVNULL,
             timeout=PROCESS_TIMEOUT_SECONDS,
         )
-        stack = RuntimeStack(keep=args.keep, diagnostics_dir=args.diagnostics_dir)
+        stack = RuntimeStack(
+            keep=args.keep,
+            diagnostics_dir=args.diagnostics_dir,
+        )
     except (OSError, StackError, subprocess.SubprocessError) as error:
         print(f"검증 환경을 준비하지 못했습니다: {error}", file=sys.stderr)
         return 2
 
     failed = True
+    result = 0
     try:
         if args.scenario == "bootstrap":
             stack.verify_bootstrap()
-        elif args.scenario == "persistence":
-            stack.verify_persistence()
         elif args.scenario == "e2e":
             stack.verify_e2e()
+        elif args.scenario == "persistence":
+            stack.verify_persistence()
         elif args.scenario == "backup-restore":
             stack.verify_backup_restore()
         elif args.scenario == "rotation":
@@ -1668,12 +1703,19 @@ def main() -> int:
         else:
             stack.verify_operations()
         failed = False
-        return 0
     except (OSError, StackError, subprocess.SubprocessError) as error:
         print(f"{args.scenario} 검증 실패: {error}", file=sys.stderr)
-        return 1
+        result = 1
     finally:
-        stack.close(failed=failed)
+        cleanup_failures = stack.close(failed=failed)
+        if cleanup_failures:
+            print(
+                "검증 정리 중 오류가 발생했습니다: " + "; ".join(cleanup_failures),
+                file=sys.stderr,
+            )
+            if result == 0:
+                result = 1
+    return result
 
 
 if __name__ == "__main__":
