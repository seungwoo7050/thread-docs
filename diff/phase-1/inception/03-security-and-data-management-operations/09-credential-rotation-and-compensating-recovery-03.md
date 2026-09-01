## `test(secrets): 회전 롤백과 재시도 검증`

diff --git a/Makefile b/Makefile
index 08913cf..1fd6193 100644
--- a/Makefile
+++ b/Makefile
@@ -8,7 +8,7 @@ NEW_SECRETS_DIR ?=
 
 COMPOSE_RUN := $(COMPOSE) --project-name $(PROJECT_NAME) --env-file $(ENV_FILE) -f $(COMPOSE_FILE)
 
-.PHONY: up start-database start-application down build logs ps clean fclean test config smoke bootstrap-test e2e persistence backup restore backup-restore-test rotate-secrets
+.PHONY: up start-database start-application down build logs ps clean fclean test config smoke bootstrap-test e2e persistence backup restore backup-restore-test rotate-secrets rotation-test
 
 up:
 	python3 tools/start_stack.py start --project "$(PROJECT_NAME)" --env-file "$(ENV_FILE)" --wait-timeout "$(WAIT_TIMEOUT)"
@@ -74,3 +74,6 @@ backup-restore-test:
 rotate-secrets:
 	@test -n "$(NEW_SECRETS_DIR)" || { echo "NEW_SECRETS_DIR is required" >&2; exit 2; }
 	python3 tools/rotate_secrets.py --project "$(PROJECT_NAME)" --env-file "$(ENV_FILE)" --new-secrets-dir "$(NEW_SECRETS_DIR)"
+
+rotation-test:
+	python3 tests/runtime_stack.py rotation
diff --git a/tests/runtime_stack.py b/tests/runtime_stack.py
index 91d2286..4602e99 100644
--- a/tests/runtime_stack.py
+++ b/tests/runtime_stack.py
@@ -150,6 +150,7 @@ class RuntimeStack:
     def run_compose(
         self,
         *arguments: str,
+        input_data: str | None = None,
         capture: bool = False,
         check: bool = True,
         timeout: int = CONTROL_TIMEOUT_SECONDS,
@@ -158,6 +159,7 @@ class RuntimeStack:
             return subprocess.run(
                 self.compose_command(*arguments),
                 cwd=ROOT,
+                input=input_data,
                 check=check,
                 text=True,
                 capture_output=capture,
@@ -301,7 +303,10 @@ class RuntimeStack:
             raise StackError(f"{service} 컨테이너 검사 결과가 예상과 다릅니다")
         return inspected[0]
 
-    def assert_runtime_secret_boundary(self) -> None:
+    def assert_runtime_secret_boundary(
+        self, expected_values: dict[str, str] | None = None
+    ) -> None:
+        values = expected_values or self.credential_values
         forbidden_names = (
             "MYSQL_ROOT_PASSWORD",
             "MYSQL_PASSWORD",
@@ -320,14 +325,18 @@ class RuntimeStack:
                 for mount in mounts
                 if isinstance(mount, dict)
             }
-            if any(path.startswith("/run/secrets") for path in destinations):
+            if any(
+                destination == "/run/secrets"
+                or destination.startswith("/run/secrets/")
+                for destination in destinations
+            ):
                 raise StackError(f"{service} 런타임에 비밀 파일이 마운트되었습니다")
             if service == "wordpress" and "/var/www/config" not in destinations:
                 raise StackError("WordPress 설정 전용 볼륨이 마운트되지 않았습니다")
             if service == "nginx":
                 if "/var/www/config" in destinations:
                     raise StackError("nginx가 WordPress 설정 전용 볼륨을 볼 수 있습니다")
-                hidden = self.run_compose(
+                hidden_config = self.run_compose(
                     "exec",
                     "--no-TTY",
                     "nginx",
@@ -339,28 +348,51 @@ class RuntimeStack:
                     capture=True,
                     check=False,
                 )
-                if hidden.returncode != 0:
-                    raise StackError("nginx에서 WordPress 설정 파일이 격리되지 않았습니다")
+                if hidden_config.returncode != 0:
+                    raise StackError("nginx에서 WordPress DB 설정 파일이 격리되지 않았습니다")
+
             config = inspected.get("Config")
             if not isinstance(config, dict):
                 raise StackError(f"{service} 실행 환경을 읽지 못했습니다")
             environment = config.get("Env") or []
+            if not isinstance(environment, list):
+                raise StackError(f"{service} 실행 환경 형식이 올바르지 않습니다")
             config_text = "\n".join(str(item) for item in environment)
             if any(name in config_text for name in forbidden_names):
                 raise StackError(f"{service} 런타임 환경에 비밀번호 변수가 남았습니다")
             observed += config_text
+
+            process_environment = self.run_compose(
+                "exec",
+                "--no-TTY",
+                service,
+                "sh",
+                "-ceu",
+                "for path in /proc/[0-9]*/environ; do "
+                "test -r \"$path\" || continue; "
+                "tr '\\000' '\\n' <\"$path\" || true; "
+                "done",
+                capture=True,
+            ).stdout
+            if any(name in process_environment for name in forbidden_names):
+                raise StackError(f"{service} 프로세스 환경에 비밀번호 변수가 남았습니다")
+            observed += process_environment
+
             container_id = str(inspected.get("Id", ""))
-            observed += subprocess.run(
+            process_arguments = subprocess.run(
                 ["docker", "top", container_id, "-eo", "pid,args"],
                 check=True,
                 text=True,
                 capture_output=True,
                 timeout=PROCESS_TIMEOUT_SECONDS,
             ).stdout
-        for value in self.credential_values.values():
-            if value in observed:
+            observed += process_arguments
+
+        for value in values.values():
+            if value and value in observed:
                 raise StackError("런타임 환경이나 프로세스 인자에 비밀값이 남았습니다")
-        config = self.run_compose(
+
+        wordpress_config = self.run_compose(
             "exec",
             "--no-TTY",
             "wordpress",
@@ -368,16 +400,22 @@ class RuntimeStack:
             "/var/www/config/wp-config.php",
             capture=True,
         ).stdout
-        if self.credential_values["db_password.txt"] not in config:
-            raise StackError("wp-config.php에 DB 자격증명이 없습니다")
+        if values["db_password.txt"] not in wordpress_config:
+            raise StackError("wp-config.php에 애플리케이션 DB 자격증명이 없습니다")
         for filename in (
             "db_root_password.txt",
             "wp_admin_password.txt",
             "wp_user_password.txt",
         ):
-            if self.credential_values[filename] in config:
+            if values[filename] in wordpress_config:
                 raise StackError(f"wp-config.php에 불필요한 비밀값이 남았습니다: {filename}")
 
+        logs = self.run_compose("logs", "--no-color", capture=True, check=False)
+        log_output = logs.stdout + logs.stderr
+        for value in values.values():
+            if value and value in log_output:
+                raise StackError("Compose 로그에 비밀값이 남았습니다")
+
     def fetch(self, path: str) -> str:
         url = f"https://{self.domain}:{self.port}{path}"
         result = subprocess.run(
@@ -942,6 +980,378 @@ class RuntimeStack:
             restored.close(failed=restored_failed)
         print("backup path safety, failure cleanup, fresh restore, and refusal passed")
 
+    def _new_secret_set(self, name: str, prefix: str) -> tuple[Path, dict[str, str]]:
+        directory = self.temp / name
+        directory.mkdir(mode=0o700)
+        values = {
+            "db_root_password.txt": f"{prefix}-root#-{secrets.token_urlsafe(24)}",
+            "db_password.txt": f"{prefix}-db#-{secrets.token_urlsafe(24)}",
+            "wp_admin_password.txt": f"{prefix}-admin-{secrets.token_urlsafe(24)}",
+            "wp_user_password.txt": f"{prefix}-user-{secrets.token_urlsafe(24)}",
+        }
+        for filename, value in values.items():
+            write_private(directory / filename, value)
+        return directory, values
+
+    def _rotation_command(
+        self,
+        directory: Path,
+        *,
+        fail_after: str | None = None,
+        pause_after: str | None = None,
+        pause_ready_file: Path | None = None,
+        rollback_ready_file: Path | None = None,
+    ) -> list[str]:
+        command = [
+            sys.executable,
+            str(ROOT / "tools" / "rotate_secrets.py"),
+            "--project",
+            self.project,
+            "--env-file",
+            str(self.env_file),
+            "--new-secrets-dir",
+            str(directory),
+        ]
+        if fail_after is not None:
+            command.extend(("--fail-after", fail_after))
+        if pause_after is not None and pause_ready_file is not None:
+            command.extend(("--pause-after", pause_after))
+            command.extend(("--pause-ready-file", str(pause_ready_file)))
+        if rollback_ready_file is not None:
+            command.extend(("--rollback-ready-file", str(rollback_ready_file)))
+        return command
+
+    def _rotation_tool(
+        self, directory: Path, *, fail_after: str | None = None
+    ) -> subprocess.CompletedProcess[str]:
+        command = self._rotation_command(directory, fail_after=fail_after)
+        try:
+            return subprocess.run(
+                command,
+                cwd=ROOT,
+                text=True,
+                capture_output=True,
+                timeout=600,
+            )
+        except subprocess.TimeoutExpired as error:
+            raise StackError("자격증명 회전 도구가 제한 시간 안에 끝나지 않았습니다") from error
+
+    def _interrupt_rotation_tool(
+        self,
+        directory: Path,
+    ) -> subprocess.CompletedProcess[str]:
+        pause_ready_file = self.temp / "rotation-host-files.ready"
+        rollback_ready_file = self.temp / "rotation-rollback.ready"
+        command = self._rotation_command(
+            directory,
+            pause_after="host-files",
+            pause_ready_file=pause_ready_file,
+            rollback_ready_file=rollback_ready_file,
+        )
+        process = subprocess.Popen(
+            command,
+            cwd=ROOT,
+            text=True,
+            stdout=subprocess.PIPE,
+            stderr=subprocess.PIPE,
+        )
+        try:
+            self._wait_for_ready_file(
+                process,
+                pause_ready_file,
+                "자격증명 회전 변경 완료",
+            )
+            process.send_signal(signal.SIGTERM)
+            self._wait_for_ready_file(
+                process,
+                rollback_ready_file,
+                "자격증명 회전 롤백 시작",
+            )
+            process.send_signal(signal.SIGINT)
+            try:
+                stdout, stderr = process.communicate(timeout=600)
+            except subprocess.TimeoutExpired as error:
+                stdout, stderr = self._terminate_process(process)
+                raise StackError("신호로 중단한 자격증명 회전이 제한 시간 안에 끝나지 않았습니다") from error
+        finally:
+            if process.poll() is None:
+                self._terminate_process(process)
+            pause_ready_file.unlink(missing_ok=True)
+            rollback_ready_file.unlink(missing_ok=True)
+        result = subprocess.CompletedProcess(command, process.returncode, stdout, stderr)
+        if (
+            result.returncode != 1
+            or "SIGTERM" not in result.stderr
+            or "롤백 완료" not in result.stderr
+            or "추가 종료 신호 지연 처리" not in result.stderr
+        ):
+            raise StackError(
+                "종료 신호가 자격증명 회전 롤백을 중단했습니다: "
+                f"{result.stderr.strip() or result.stdout.strip()}"
+            )
+        return result
+
+    def _sql_password_works(self, kind: str, password: str) -> bool:
+        if kind == "root":
+            service = "mariadb"
+            command = (
+                "mariadb --defaults-extra-file=\"$auth\" "
+                "--socket=/run/mysqld/mysqld.sock -uroot --execute='SELECT 1'"
+            )
+        else:
+            service = "wordpress"
+            command = (
+                "mariadb --defaults-extra-file=\"$auth\" -hmariadb "
+                "-u\"$MYSQL_USER\" \"$MYSQL_DATABASE\" --execute='SELECT 1'"
+            )
+        result = self.run_compose(
+            "exec",
+            "--no-TTY",
+            service,
+            "sh",
+            "-ceu",
+            "umask 077; auth=\"$(mktemp /run/container-stack-test.XXXXXX)\"; "
+            "trap 'rm -f -- \"$auth\"' EXIT HUP INT TERM; "
+            "IFS= read -r password; "
+            "printf '[client]\\npassword=\"%s\"\\n' \"$password\" >\"$auth\"; "
+            + command,
+            input_data=password + "\n",
+            capture=True,
+            check=False,
+        )
+        return result.returncode == 0
+
+    def _wordpress_password_works(self, kind: str, password: str) -> bool:
+        code = r"""
+$payload = json_decode(stream_get_contents(STDIN), true, 8, JSON_THROW_ON_ERROR);
+require '/var/www/html/wp-load.php';
+$login = getenv($payload['kind'] === 'admin' ? 'WORDPRESS_ADMIN_USER' : 'WORDPRESS_USER');
+$account = get_user_by('login', $login);
+if (!$account) { exit(1); }
+clean_user_cache($account->ID);
+$account = get_user_by('login', $login);
+if (!$account || !wp_check_password($payload['password'], $account->user_pass, $account->ID)) { exit(1); }
+"""
+        result = self.run_compose(
+            "exec",
+            "--no-TTY",
+            "wordpress",
+            "php",
+            "-r",
+            code,
+            input_data=json.dumps({"kind": kind, "password": password}),
+            capture=True,
+            check=False,
+        )
+        return result.returncode == 0
+
+    def _wordpress_config_matches(self, password: str) -> bool:
+        code = r"""
+$payload = json_decode(stream_get_contents(STDIN), true, 8, JSON_THROW_ON_ERROR);
+$text = file_get_contents('/var/www/html/wp-config.php');
+$pattern = "/define\\(\\s*['\"]DB_PASSWORD['\"]\\s*,\\s*['\"]([^'\"]*)['\"]\\s*\\);/";
+if ($text === false || !preg_match($pattern, $text, $matches) || !hash_equals($payload['password'], $matches[1])) { exit(1); }
+"""
+        result = self.run_compose(
+            "exec",
+            "--no-TTY",
+            "wordpress",
+            "php",
+            "-r",
+            code,
+            input_data=json.dumps({"password": password}),
+            capture=True,
+            check=False,
+        )
+        return result.returncode == 0
+
+
+    def _assert_no_rotation_temporary_files(self) -> None:
+        if list(self.temp.glob(".*.txt.*")):
+            raise StackError("호스트에 자격증명 임시 파일이 남았습니다")
+        checks = (
+            (
+                "mariadb",
+                "test -z \"$(find /run -maxdepth 1 -type f "
+                "\\( -name 'container-stack-root.*' -o -name 'container-stack-test.*' \\) "
+                "-print -quit)\"",
+            ),
+            (
+                "wordpress",
+                "test -z \"$(find /run -maxdepth 1 -type f "
+                "\\( -name 'container-stack-app.*' -o -name 'container-stack-test.*' \\) "
+                "-print -quit)\"; "
+                "test -z \"$(find /var/www/config -maxdepth 1 -type f "
+                "-name '.wp-config.rotate.*' -print -quit)\"",
+            ),
+        )
+        for service, script in checks:
+            result = self.run_compose(
+                "exec",
+                "--no-TTY",
+                service,
+                "sh",
+                "-ceu",
+                script,
+                capture=True,
+                check=False,
+            )
+            if result.returncode != 0:
+                raise StackError(f"{service} 컨테이너에 자격증명 임시 파일이 남았습니다")
+
+    def _assert_rotation_state(
+        self,
+        expected: dict[str, str],
+        rejected: dict[str, str],
+    ) -> None:
+        for filename, value in expected.items():
+            path = self.temp / filename
+            if path.read_text(encoding="utf-8").rstrip("\n") != value:
+                raise StackError(f"호스트 비밀 파일 값이 예상과 다릅니다: {filename}")
+            info = path.stat()
+            if stat.S_IMODE(info.st_mode) != 0o600 or info.st_uid != os.getuid():
+                raise StackError(f"호스트 비밀 파일 권한이나 소유자가 다릅니다: {filename}")
+
+        root = expected["db_root_password.txt"]
+        app = expected["db_password.txt"]
+        if not self._sql_password_works("root", root):
+            raise StackError("예상한 MariaDB root 비밀번호가 동작하지 않습니다")
+        if self._sql_password_works("root", rejected["db_root_password.txt"]):
+            raise StackError("폐기한 MariaDB root 비밀번호가 여전히 동작합니다")
+        if not self._sql_password_works("app", app):
+            raise StackError("예상한 MariaDB 애플리케이션 비밀번호가 동작하지 않습니다")
+        if self._sql_password_works("app", rejected["db_password.txt"]):
+            raise StackError("폐기한 MariaDB 애플리케이션 비밀번호가 여전히 동작합니다")
+        if not self._wordpress_config_matches(app):
+            raise StackError("wp-config.php가 예상한 DB 비밀번호를 사용하지 않습니다")
+
+        for kind, filename in (
+            ("admin", "wp_admin_password.txt"),
+            ("user", "wp_user_password.txt"),
+        ):
+            if not self._wordpress_password_works(kind, expected[filename]):
+                raise StackError(f"예상한 WordPress {kind} 비밀번호가 동작하지 않습니다")
+            if self._wordpress_password_works(kind, rejected[filename]):
+                raise StackError(f"폐기한 WordPress {kind} 비밀번호가 여전히 동작합니다")
+
+        if self.fetch("/healthz").strip() != "ok":
+            raise StackError("자격증명 상태 검증 뒤 HTTPS 상태 확인이 실패했습니다")
+        nonce = secrets.token_hex(8)
+        content = f"rotation-state-{nonce}"
+        post_id = self.wordpress(
+            "post",
+            "create",
+            f"--post_title=회전 상태 {nonce}",
+            f"--post_content={content}",
+            "--post_status=publish",
+            "--porcelain",
+            capture=True,
+        )
+        if content not in self.fetch(f"/?p={post_id}"):
+            raise StackError("자격증명 상태 검증 뒤 WordPress 쓰기·읽기가 실패했습니다")
+        self._assert_no_rotation_temporary_files()
+        self.assert_runtime_secret_boundary(expected)
+
+    def verify_secret_rotation(self) -> None:
+        self.start()
+        initial_values = dict(self.credential_values)
+
+        def snapshot(directory: Path) -> dict[str, tuple[bytes, int]]:
+            return {
+                path.name: (path.read_bytes(), stat.S_IMODE(path.stat().st_mode))
+                for path in directory.iterdir()
+            }
+
+        def assert_input_unchanged(
+            directory: Path, expected: dict[str, tuple[bytes, int]]
+        ) -> None:
+            if snapshot(directory) != expected:
+                raise StackError(f"회전 입력 파일이 변경되었습니다: {directory.name}")
+
+        def assert_no_secret_output(
+            result: subprocess.CompletedProcess[str],
+            *secret_sets: dict[str, str],
+        ) -> None:
+            output = result.stdout + result.stderr
+            for secret_set in secret_sets:
+                for value in secret_set.values():
+                    if value in output:
+                        raise StackError("회전 도구 출력에 비밀값이 포함되었습니다")
+
+        first_dir, first_values = self._new_secret_set("rotation-first", "first")
+        first_snapshot = snapshot(first_dir)
+        first = self._rotation_tool(first_dir)
+        if first.returncode != 0:
+            raise StackError(f"정상 회전이 실패했습니다: {first.stderr}")
+        assert_no_secret_output(first, initial_values, first_values)
+        assert_input_unchanged(first_dir, first_snapshot)
+        self._assert_rotation_state(first_values, initial_values)
+
+        tested_values: list[dict[str, str]] = [initial_values, first_values]
+        retry_dir: Path | None = None
+        retry_values: dict[str, str] | None = None
+        retry_snapshot: dict[str, tuple[bytes, int]] | None = None
+        for index, failure_stage in enumerate(
+            (
+                "admin-user-command",
+                "config-command",
+                "app-password-command",
+                "root-password-command",
+                "host-file",
+                "recreate-wordpress-removed",
+            ),
+            start=1,
+        ):
+            candidate_dir, candidate_values = self._new_secret_set(
+                f"rotation-failure-{index}", f"failure-{index}"
+            )
+            candidate_snapshot = snapshot(candidate_dir)
+            injected = self._rotation_tool(
+                candidate_dir, fail_after=failure_stage
+            )
+            if injected.returncode == 0 or "롤백 완료" not in injected.stderr:
+                raise StackError(
+                    f"{failure_stage} 실패 주입 뒤 롤백 결과를 확인하지 못했습니다: "
+                    f"{injected.stderr}"
+                )
+            assert_no_secret_output(injected, first_values, candidate_values)
+            assert_input_unchanged(candidate_dir, candidate_snapshot)
+            self._assert_rotation_state(first_values, candidate_values)
+            tested_values.append(candidate_values)
+            retry_dir = candidate_dir
+            retry_values = candidate_values
+            retry_snapshot = candidate_snapshot
+
+        signal_dir, signal_values = self._new_secret_set(
+            "rotation-signal", "signal"
+        )
+        signal_snapshot = snapshot(signal_dir)
+        interrupted = self._interrupt_rotation_tool(signal_dir)
+        assert_no_secret_output(interrupted, first_values, signal_values)
+        assert_input_unchanged(signal_dir, signal_snapshot)
+        self._assert_rotation_state(first_values, signal_values)
+        tested_values.append(signal_values)
+        retry_dir = signal_dir
+        retry_values = signal_values
+        retry_snapshot = signal_snapshot
+
+        if retry_dir is None or retry_values is None or retry_snapshot is None:
+            raise StackError("재시도할 회전 입력을 만들지 못했습니다")
+        retried = self._rotation_tool(retry_dir)
+        if retried.returncode != 0:
+            raise StackError(f"롤백 직후 같은 입력으로 재시도하지 못했습니다: {retried.stderr}")
+        assert_no_secret_output(retried, first_values, retry_values)
+        assert_input_unchanged(retry_dir, retry_snapshot)
+        self._assert_rotation_state(retry_values, first_values)
+
+        logs = self.run_compose("logs", "--no-color", capture=True, check=False)
+        log_output = logs.stdout + logs.stderr
+        for secret_set in tested_values:
+            for value in secret_set.values():
+                if value in log_output:
+                    raise StackError("Compose 로그에 자격증명 값이 포함되었습니다")
+        print("secret rotation, ambiguous failures, rollback, and retry passed")
+
     def collect_diagnostics(self) -> Path:
         destination = self.diagnostics_dir
         if destination is None:
@@ -985,7 +1395,7 @@ def parse_arguments() -> argparse.Namespace:
     parser = argparse.ArgumentParser(description="격리된 컨테이너 스택 검증")
     parser.add_argument(
         "scenario",
-        choices=("bootstrap", "e2e", "persistence", "backup-restore"),
+        choices=("bootstrap", "e2e", "persistence", "backup-restore", "rotation"),
     )
     parser.add_argument("--keep", action="store_true", help="검사 뒤 프로젝트를 유지합니다")
     parser.add_argument("--diagnostics-dir", type=Path)
@@ -1016,8 +1426,10 @@ def main() -> int:
             stack.verify_persistence()
         elif args.scenario == "e2e":
             stack.verify_e2e()
-        else:
+        elif args.scenario == "backup-restore":
             stack.verify_backup_restore()
+        else:
+            stack.verify_secret_rotation()
         failed = False
         return 0
     except (OSError, StackError, subprocess.SubprocessError) as error:
diff --git a/tests/validate_stack.py b/tests/validate_stack.py
index 7a92661..08c1bea 100755
--- a/tests/validate_stack.py
+++ b/tests/validate_stack.py
@@ -190,6 +190,10 @@ def validate_tools() -> None:
             r"stack_backup\.py restore",
             r"^backup-restore-test:",
             r"runtime_stack\.py backup-restore",
+            r"^rotate-secrets:",
+            r"rotate_secrets\.py",
+            r"^rotation-test:",
+            r"runtime_stack\.py rotation",
         ],
     )
     require_text(
@@ -222,6 +226,39 @@ def validate_tools() -> None:
     backup_tool = require_file("tools/stack_backup.py").read_text()
     if re.search(r"-p(?:\\?['\"])?\$\(cat", backup_tool):
         fail("database client passwords must not be exposed in command arguments")
+    require_text(
+        "tools/rotate_secrets.py",
+        [
+            r"def atomic_secret_write",
+            r"def verify_rotation",
+            r"def maybe_fail",
+            r"rollback_errors",
+            r"--new-secrets-dir",
+            r"project_operation_lock",
+            r"tempnam",
+            r"SIGNAL SQLSTATE",
+            r"admin-user-command",
+            r"root-password-command",
+            r"host-file",
+            r"find_root_password",
+            r"verify_runtime_secret_boundary",
+            r"O_NOFOLLOW",
+            r"signal\.SIGINT",
+            r"signal\.SIGTERM",
+            r"one_off",
+            r'"run",\s*\n\s*"--rm"',
+            r'"--entrypoint",\s*\n\s*"php"',
+            r"recreate-wordpress-removed",
+            r'project\.run\("rm", "--stop", "--force", "wordpress"\)',
+            r'"rollback_active"',
+            r'"deferred"',
+            r"--pause-after",
+            r"--rollback-ready-file",
+        ],
+    )
+    rotation_tool = require_file("tools/rotate_secrets.py").read_text()
+    if re.search(r"auth=/tmp/container-stack-(?:root|app)\.\$\$", rotation_tool):
+        fail("rotation database clients must use unpredictable private option files")
     require_text(
         "tests/runtime_stack.py",
         [
@@ -254,6 +291,18 @@ def validate_tools() -> None:
             r"command\.append\(\"--all\"\)",
             r"TMPDIR",
             r"다른 관리 작업이 실행 중입니다",
+            r"def verify_secret_rotation",
+            r"timeout=600",
+            r"def _assert_rotation_state",
+            r"admin-user-command",
+            r"root-password-command",
+            r"host-file",
+            r"config-command",
+            r"recreate-wordpress-removed",
+            r"def _interrupt_rotation_tool",
+            r"rotation-host-files\.ready",
+            r"rotation-rollback\.ready",
+            r"추가 종료 신호 지연 처리",
         ],
     )
 


