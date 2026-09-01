## `test(cleanup): 테스트 프로젝트 소유 자원만 정리`

diff --git a/.gitignore b/.gitignore
index b8efaf7..6a5abed 100644
--- a/.gitignore
+++ b/.gitignore
@@ -6,3 +6,4 @@ secrets/*.txt
 __pycache__/
 *.py[cod]
 diagnostics/
+artifacts/
diff --git a/tests/runtime_stack.py b/tests/runtime_stack.py
index e56d919..471942c 100644
--- a/tests/runtime_stack.py
+++ b/tests/runtime_stack.py
@@ -15,7 +15,6 @@ import stat
 import subprocess
 import sys
 import tempfile
-from typing import Any
 import time
 
 
@@ -80,10 +79,14 @@ class RuntimeStack:
         *,
         keep: bool,
         diagnostics_dir: Path | None,
+        project_record_dir: Path | None = None,
         credential_values: dict[str, str] | None = None,
+        image_prefix: str | None = None,
+        owns_images: bool = True,
     ) -> None:
         self.keep = keep
         self.diagnostics_dir = diagnostics_dir
+        self.project_record_dir = project_record_dir
         self.temp = Path(tempfile.mkdtemp(prefix="container-stack-e2e-"))
         self.temp.chmod(0o700)
         self.project = f"container-stack-{os.getpid()}-{secrets.token_hex(3)}"
@@ -91,6 +94,8 @@ class RuntimeStack:
         self.port = reserve_port()
         self.env_file = self.temp / ".env"
         self.started = False
+        self.image_prefix = image_prefix or f"{self.project}-image"
+        self.owns_images = owns_images
         self.credential_values = credential_values or {
             "db_root_password.txt": f"root#-{secrets.token_urlsafe(24)}",
             "db_password.txt": f"db#-{secrets.token_urlsafe(24)}",
@@ -98,11 +103,23 @@ class RuntimeStack:
             "wp_user_password.txt": f"user-{secrets.token_urlsafe(24)}",
         }
         try:
+            self._record_project()
             self._prepare_environment()
         except Exception:
             shutil.rmtree(self.temp, ignore_errors=True)
             raise
 
+    def _record_project(self) -> None:
+        if self.project_record_dir is None:
+            return
+        directory = self.project_record_dir
+        if directory.is_symlink():
+            raise StackError(f"프로젝트 기록 경로가 심볼릭 링크입니다: {directory}")
+        directory.mkdir(parents=True, mode=0o700, exist_ok=True)
+        if not directory.is_dir() or stat.S_IMODE(directory.stat().st_mode) & 0o077:
+            raise StackError(f"프로젝트 기록 경로 권한이 안전하지 않습니다: {directory}")
+        write_private(directory / self.project, self.project)
+
     def _prepare_environment(self) -> None:
         for filename, value in self.credential_values.items():
             write_private(self.temp / filename, value)
@@ -112,7 +129,7 @@ class RuntimeStack:
             "WORDPRESS_URL": f"https://{self.domain}:{self.port}",
             "HTTPS_BIND_ADDRESS": "127.0.0.1",
             "HTTPS_PORT": str(self.port),
-            "STACK_IMAGE_PREFIX": f"{self.project}-image",
+            "STACK_IMAGE_PREFIX": self.image_prefix,
             "STACK_IMAGE_TAG": "local",
             "MYSQL_DATABASE": "wordpress",
             "MYSQL_USER": "wpuser",
@@ -310,24 +327,6 @@ class RuntimeStack:
                 f"관리 작업 뒤 서비스가 모두 복구되지 않았습니다: {sorted(running)}"
             )
 
-    def inspect_service(self, service: str) -> dict[str, object]:
-        container_id = self.run_compose(
-            "ps", "--quiet", service, capture=True
-        ).stdout.strip()
-        if not container_id or "\n" in container_id:
-            raise StackError(f"{service} 컨테이너를 하나로 식별하지 못했습니다")
-        result = subprocess.run(
-            ["docker", "inspect", container_id],
-            check=True,
-            text=True,
-            capture_output=True,
-            timeout=PROCESS_TIMEOUT_SECONDS,
-        )
-        inspected = json.loads(result.stdout)
-        if not isinstance(inspected, list) or len(inspected) != 1:
-            raise StackError(f"{service} 컨테이너 검사 결과가 예상과 다릅니다")
-        return inspected[0]
-
     def assert_runtime_secret_boundary(
         self, expected_values: dict[str, str] | None = None
     ) -> None:
@@ -441,6 +440,24 @@ class RuntimeStack:
             if value and value in log_output:
                 raise StackError("Compose 로그에 비밀값이 남았습니다")
 
+    def inspect_service(self, service: str) -> dict[str, object]:
+        container_id = self.run_compose(
+            "ps", "--quiet", service, capture=True
+        ).stdout.strip()
+        if not container_id or "\n" in container_id:
+            raise StackError(f"{service} 컨테이너를 하나로 식별하지 못했습니다")
+        result = subprocess.run(
+            ["docker", "inspect", container_id],
+            check=True,
+            text=True,
+            capture_output=True,
+            timeout=PROCESS_TIMEOUT_SECONDS,
+        )
+        inspected = json.loads(result.stdout)
+        if not isinstance(inspected, list) or len(inspected) != 1:
+            raise StackError(f"{service} 컨테이너 검사 결과가 예상과 다릅니다")
+        return inspected[0]
+
     def fetch(self, path: str) -> str:
         url = f"https://{self.domain}:{self.port}{path}"
         result = subprocess.run(
@@ -463,7 +480,6 @@ class RuntimeStack:
         )
         return result.stdout
 
-
     def verify_e2e(self) -> None:
         blocked_port = self.port
         with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as listener:
@@ -1068,7 +1084,7 @@ class RuntimeStack:
                 labelled_name,
                 "--label",
                 f"com.docker.compose.project={restored.project}",
-                f"{self.project}-image-mariadb:local",
+                f"{self.image_prefix}-mariadb:local",
             ],
             check=True,
             text=True,
@@ -1114,7 +1130,7 @@ class RuntimeStack:
                     "create",
                     "--name",
                     name,
-                    f"{self.project}-image-wordpress:local",
+                    f"{self.image_prefix}-wordpress:local",
                 ]
             else:
                 command = ["docker", kind, "create", name]
@@ -1273,6 +1289,7 @@ class RuntimeStack:
         restored = RuntimeStack(
             keep=False,
             diagnostics_dir=self.diagnostics_dir,
+            project_record_dir=self.project_record_dir,
             credential_values=dict(self.credential_values),
         )
         restored.started = True
@@ -1551,7 +1568,6 @@ if ($text === false || !preg_match($pattern, $text, $matches) || !hash_equals($p
         )
         return result.returncode == 0
 
-
     def _assert_no_rotation_temporary_files(self) -> None:
         if list(self.temp.glob(".*.txt.*")):
             raise StackError("호스트에 자격증명 임시 파일이 남았습니다")
@@ -1981,27 +1997,68 @@ if ($text === false || !preg_match($pattern, $text, $matches) || !hash_equals($p
         print(f"진단 자료: {destination}", file=sys.stderr)
         return destination
 
+    def _remove_test_images(self) -> None:
+        failures: list[str] = []
+        for service in ("nginx", "wordpress", "mariadb"):
+            image = f"{self.image_prefix}-{service}:local"
+            result = subprocess.run(
+                ["docker", "image", "rm", image],
+                text=True,
+                capture_output=True,
+                timeout=PROCESS_TIMEOUT_SECONDS,
+            )
+            if result.returncode != 0 and "No such image" not in result.stderr:
+                failures.append(
+                    f"{image}: {result.stderr.strip() or result.stdout.strip()}"
+                )
+        if failures:
+            raise StackError(
+                "검사용 이미지 태그를 정리하지 못했습니다: " + "; ".join(failures)
+            )
+
     def close(self, *, failed: bool) -> list[str]:
         failures: list[str] = []
         if failed:
             try:
                 self.collect_diagnostics()
-            except Exception as error:
-                failures.append(f"진단 자료: {error}")
-        if self.started and not self.keep:
-            try:
+            except (OSError, StackError, subprocess.SubprocessError) as error:
+                failures.append(f"진단 자료 저장: {error}")
+        if self.keep:
+            print(
+                f"검사용 프로젝트를 유지합니다: {self.project} ({self.temp})",
+                file=sys.stderr,
+            )
+            return failures
+
+        try:
+            if self.started:
                 result = self.run_compose(
-                    "down", "--volumes", "--remove-orphans", "--timeout", "20",
-                    capture=True, check=False,
+                    "down",
+                    "--volumes",
+                    "--remove-orphans",
+                    "--timeout",
+                    "20",
+                    capture=True,
+                    check=False,
                 )
                 if result.returncode != 0:
-                    failures.append(result.stderr.strip() or "Compose 자원 정리 실패")
-            except Exception as error:
-                failures.append(f"Compose 자원 정리: {error}")
-        if self.keep:
-            print(f"검사용 프로젝트를 유지합니다: {self.project} ({self.temp})", file=sys.stderr)
-        else:
-            shutil.rmtree(self.temp, ignore_errors=True)
+                    failures.append(
+                        "Compose 자원 정리: "
+                        + (result.stderr.strip() or result.stdout.strip())
+                    )
+        except (OSError, StackError, subprocess.SubprocessError) as error:
+            failures.append(f"Compose 자원 정리: {error}")
+
+        try:
+            if self.owns_images:
+                self._remove_test_images()
+        except (OSError, StackError, subprocess.SubprocessError) as error:
+            failures.append(f"이미지 태그 정리: {error}")
+
+        try:
+            shutil.rmtree(self.temp)
+        except OSError as error:
+            failures.append(f"임시 비밀 파일 정리: {error}")
         return failures
 
 
@@ -2020,6 +2077,7 @@ def parse_arguments() -> argparse.Namespace:
     )
     parser.add_argument("--keep", action="store_true", help="검사 뒤 프로젝트를 유지합니다")
     parser.add_argument("--diagnostics-dir", type=Path)
+    parser.add_argument("--project-record-dir", type=Path)
     return parser.parse_args()
 
 
@@ -2038,6 +2096,7 @@ def main() -> int:
         stack = RuntimeStack(
             keep=args.keep,
             diagnostics_dir=args.diagnostics_dir,
+            project_record_dir=args.project_record_dir,
         )
     except (OSError, StackError, subprocess.SubprocessError) as error:
         print(f"검증 환경을 준비하지 못했습니다: {error}", file=sys.stderr)
diff --git a/tools/cleanup_test_resources.py b/tools/cleanup_test_resources.py
new file mode 100755
index 0000000..dd0710d
--- /dev/null
+++ b/tools/cleanup_test_resources.py
@@ -0,0 +1,182 @@
+#!/usr/bin/env python3
+"""현재 검증이 기록한 Compose 프로젝트의 잔여 자원만 회수합니다."""
+
+from __future__ import annotations
+
+import argparse
+import os
+from pathlib import Path
+import re
+import stat
+import subprocess
+import sys
+
+
+PROJECT_PATTERN = re.compile(r"^container-stack-[0-9]+-[0-9a-f]{6}$")
+PROJECT_LABEL = "com.docker.compose.project"
+IMAGE_SERVICES = ("nginx", "wordpress", "mariadb")
+
+
+class CleanupError(RuntimeError):
+    pass
+
+
+def load_projects(directory: Path) -> list[str]:
+    if directory.is_symlink() or not directory.is_dir():
+        raise CleanupError(f"프로젝트 기록 디렉터리가 올바르지 않습니다: {directory}")
+    if stat.S_IMODE(directory.stat().st_mode) & 0o077:
+        raise CleanupError(f"프로젝트 기록 디렉터리 권한이 안전하지 않습니다: {directory}")
+    projects: list[str] = []
+    for path in sorted(directory.iterdir()):
+        if path.is_symlink() or not path.is_file():
+            raise CleanupError(f"프로젝트 기록에 일반 파일이 아닌 항목이 있습니다: {path}")
+        if stat.S_IMODE(path.stat().st_mode) != 0o600:
+            raise CleanupError(f"프로젝트 기록 파일 권한이 0600이 아닙니다: {path}")
+        raw_project = path.read_text(encoding="utf-8")
+        project = raw_project.removesuffix("\n")
+        if (
+            not PROJECT_PATTERN.fullmatch(project)
+            or path.name != project
+            or raw_project != f"{project}\n"
+        ):
+            raise CleanupError(f"프로젝트 기록 내용이 올바르지 않습니다: {path}")
+        projects.append(project)
+    return projects
+
+
+def list_resources(kind: str, project: str) -> list[str]:
+    if kind == "image":
+        images: list[str] = []
+        for service in IMAGE_SERVICES:
+            tag = f"{project}-image-{service}:local"
+            result = subprocess.run(
+                [
+                    "docker",
+                    "image",
+                    "ls",
+                    "--filter",
+                    f"reference={tag}",
+                    "--format",
+                    "{{.Repository}}:{{.Tag}}",
+                ],
+                check=True,
+                text=True,
+                capture_output=True,
+                timeout=30,
+            )
+            if tag in result.stdout.splitlines():
+                images.append(tag)
+        return images
+
+    commands = {
+        "container": [
+            "docker",
+            "ps",
+            "--all",
+            "--filter",
+            f"label={PROJECT_LABEL}={project}",
+            "--format",
+            "{{.ID}}",
+        ],
+        "volume": [
+            "docker",
+            "volume",
+            "ls",
+            "--filter",
+            f"label={PROJECT_LABEL}={project}",
+            "--format",
+            "{{.Name}}",
+        ],
+        "network": [
+            "docker",
+            "network",
+            "ls",
+            "--filter",
+            f"label={PROJECT_LABEL}={project}",
+            "--format",
+            "{{.ID}}",
+        ],
+    }
+    result = subprocess.run(
+        commands[kind], check=True, text=True, capture_output=True, timeout=30
+    )
+    return [line for line in result.stdout.splitlines() if line]
+
+
+def remove(kind: str, identifier: str) -> subprocess.CompletedProcess[str]:
+    commands = {
+        "container": ["docker", "rm", "--force", identifier],
+        "volume": ["docker", "volume", "rm", identifier],
+        "network": ["docker", "network", "rm", identifier],
+        "image": ["docker", "image", "rm", identifier],
+    }
+    return subprocess.run(
+        commands[kind], text=True, capture_output=True, timeout=30
+    )
+
+
+def write_private(path: Path, text: str) -> None:
+    if path.exists() or path.is_symlink():
+        raise CleanupError(f"정리 보고서 경로가 이미 존재합니다: {path}")
+    path.parent.mkdir(parents=True, exist_ok=True)
+    descriptor = os.open(path, os.O_WRONLY | os.O_CREAT | os.O_EXCL, 0o600)
+    with os.fdopen(descriptor, "w", encoding="utf-8") as stream:
+        stream.write(text)
+
+
+def cleanup(project_record_dir: Path, report: Path | None) -> int:
+    projects = load_projects(project_record_dir)
+    discovered: dict[str, list[tuple[str, str]]] = {
+        kind: [] for kind in ("container", "volume", "network", "image")
+    }
+    for project in projects:
+        for kind in discovered:
+            discovered[kind].extend(
+                (identifier, project)
+                for identifier in list_resources(kind, project)
+            )
+    if not any(discovered.values()):
+        print("현재 검증이 남긴 Compose 자원이 없습니다")
+        return 0
+
+    lines = ["현재 검증에서 회수한 자원입니다."]
+    failures: list[str] = []
+    for kind in ("container", "volume", "network", "image"):
+        for identifier, project in discovered[kind]:
+            result = remove(kind, identifier)
+            outcome = "removed" if result.returncode == 0 else "failed"
+            lines.append(f"{kind}\t{identifier}\t{project}\t{outcome}")
+            if result.returncode != 0:
+                failures.append(
+                    f"{kind} {identifier}: {result.stderr.strip() or result.stdout.strip()}"
+                )
+    if failures:
+        lines.extend(("", "회수 실패:", *failures))
+    if report is not None:
+        write_private(report, "\n".join(lines) + "\n")
+        print(f"정리 보고서: {report}", file=sys.stderr)
+    if failures:
+        print("일부 검증 자원을 회수하지 못했습니다", file=sys.stderr)
+        return 2
+    print("검증 자원 누수를 발견해 기록된 프로젝트만 회수했습니다", file=sys.stderr)
+    return 1
+
+
+def parse_arguments() -> argparse.Namespace:
+    parser = argparse.ArgumentParser(description="현재 검증의 Compose 자원 누수 검사")
+    parser.add_argument("--project-record-dir", type=Path, required=True)
+    parser.add_argument("--report", type=Path)
+    return parser.parse_args()
+
+
+def main() -> int:
+    args = parse_arguments()
+    try:
+        return cleanup(args.project_record_dir, args.report)
+    except (CleanupError, OSError, subprocess.SubprocessError) as error:
+        print(f"검증 자원 정리 실패: {error}", file=sys.stderr)
+        return 2
+
+
+if __name__ == "__main__":
+    raise SystemExit(main())


## `test(verify): 전체 스택 검증을 직렬 실행`

diff --git a/Makefile b/Makefile
index 52da767..7230ff8 100644
--- a/Makefile
+++ b/Makefile
@@ -10,7 +10,7 @@ DESTROY_CONFIRM ?=
 
 COMPOSE_RUN := $(COMPOSE) --project-name $(PROJECT_NAME) --env-file $(ENV_FILE) -f $(COMPOSE_FILE)
 
-.PHONY: up start-database start-application down build logs ps clean fclean test config config-strict smoke bootstrap-test e2e persistence backup restore backup-restore-test rotate-secrets rotation-test diagnostics operations-test
+.PHONY: up start-database start-application down build logs ps clean fclean test config config-strict smoke bootstrap-test e2e persistence backup restore backup-restore-test rotate-secrets rotation-test diagnostics operations-test verify
 
 up:
 	python3 tools/start_stack.py start --project "$(PROJECT_NAME)" --env-file "$(ENV_FILE)" --wait-timeout "$(WAIT_TIMEOUT)"
@@ -94,3 +94,6 @@ diagnostics:
 
 operations-test:
 	python3 tests/runtime_stack.py operations
+
+verify:
+	python3 tools/verify_stack.py
diff --git a/tools/verify_stack.py b/tools/verify_stack.py
new file mode 100755
index 0000000..53145c1
--- /dev/null
+++ b/tools/verify_stack.py
@@ -0,0 +1,95 @@
+#!/usr/bin/env python3
+"""정적 검사와 런타임 시나리오를 직렬 실행하고 잔여 자원을 확인합니다."""
+
+from __future__ import annotations
+
+from pathlib import Path
+import shutil
+import subprocess
+import sys
+import tempfile
+
+
+ROOT = Path(__file__).resolve().parents[1]
+SCENARIOS = (
+    "bootstrap",
+    "e2e",
+    "persistence",
+    "backup-restore",
+    "rotation",
+    "operations",
+)
+SCENARIO_TIMEOUTS = {
+    "bootstrap": 2400,
+    "e2e": 1500,
+    "persistence": 1500,
+    "backup-restore": 1800,
+    "rotation": 1800,
+    "operations": 1500,
+}
+
+
+def run(command: list[str], *, timeout: int) -> int:
+    try:
+        return subprocess.run(command, cwd=ROOT, timeout=timeout).returncode
+    except subprocess.TimeoutExpired:
+        print(
+            f"검증 명령이 {timeout}초 안에 끝나지 않았습니다: {' '.join(command)}",
+            file=sys.stderr,
+        )
+        return 124
+
+
+def main() -> int:
+    temporary = Path(tempfile.mkdtemp(prefix="container-stack-verify-"))
+    temporary.chmod(0o700)
+    records = temporary / "projects"
+    records.mkdir(mode=0o700)
+    cleanup_report = temporary / "cleanup.txt"
+    result = 0
+    try:
+        commands = (
+            ["make", "test"],
+            ["make", "config-strict", "ENV_FILE=.env.example"],
+        )
+        for command in commands:
+            result = run(command, timeout=300)
+            if result != 0:
+                break
+        if result == 0:
+            for scenario in SCENARIOS:
+                result = run(
+                    [
+                        sys.executable,
+                        str(ROOT / "tests" / "runtime_stack.py"),
+                        scenario,
+                        "--project-record-dir",
+                        str(records),
+                    ],
+                    timeout=SCENARIO_TIMEOUTS[scenario],
+                )
+                if result != 0:
+                    break
+    finally:
+        cleanup_result = run(
+            [
+                sys.executable,
+                str(ROOT / "tools" / "cleanup_test_resources.py"),
+                "--project-record-dir",
+                str(records),
+                "--report",
+                str(cleanup_report),
+            ],
+            timeout=300,
+        )
+        if cleanup_result == 2 or result == 0:
+            result = cleanup_result
+        if cleanup_result != 0:
+            print(f"누수 재확인 자료를 보존했습니다: {temporary}", file=sys.stderr)
+        else:
+            shutil.rmtree(temporary, ignore_errors=True)
+    return result
+
+
+if __name__ == "__main__":
+    raise SystemExit(main())


