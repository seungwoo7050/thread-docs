## `test(restore): 거부·롤백·복원 상태 검증`

diff --git a/Makefile b/Makefile
index 88e4abd..1e3019a 100644
--- a/Makefile
+++ b/Makefile
@@ -7,7 +7,7 @@ BACKUP_DIR ?=
 
 COMPOSE_RUN := $(COMPOSE) --project-name $(PROJECT_NAME) --env-file $(ENV_FILE) -f $(COMPOSE_FILE)
 
-.PHONY: up start-database start-application down build logs ps clean fclean test config smoke bootstrap-test e2e persistence backup backup-restore-test restore
+.PHONY: up start-database start-application down build logs ps clean fclean test config smoke bootstrap-test e2e persistence backup restore backup-restore-test
 
 up:
 	python3 tools/start_stack.py start --project "$(PROJECT_NAME)" --env-file "$(ENV_FILE)" --wait-timeout "$(WAIT_TIMEOUT)"
diff --git a/tests/runtime_stack.py b/tests/runtime_stack.py
index da52251..91d2286 100644
--- a/tests/runtime_stack.py
+++ b/tests/runtime_stack.py
@@ -875,7 +875,72 @@ class RuntimeStack:
             if ((backup / filename_in_backup).stat().st_mode & 0o077) != 0:
                 raise StackError(f"백업 파일 권한이 안전하지 않습니다: {filename_in_backup}")
 
-        print("backup publication, interruption, and cleanup passed")
+        restored = RuntimeStack(
+            keep=False,
+            diagnostics_dir=self.diagnostics_dir,
+            credential_values=dict(self.credential_values),
+        )
+        restored.started = True
+        restored_failed = True
+        try:
+            unsafe_backup = self.temp / "unsafe-backup"
+            unsafe_backup.mkdir(mode=0o700)
+            for safe_name in ("manifest.json", "wordpress.tar.gz"):
+                shutil.copyfile(backup / safe_name, unsafe_backup / safe_name)
+                (unsafe_backup / safe_name).chmod(0o600)
+            (unsafe_backup / "database.sql").symlink_to(backup / "database.sql")
+            unsafe_result = self._backup_tool(
+                "restore", restored, unsafe_backup, check=False
+            )
+            if (
+                unsafe_result.returncode == 0
+                or "안전하게 열 수 없습니다" not in unsafe_result.stderr
+                or restored.project_resources()
+            ):
+                raise StackError("복원 도구가 백업 내부의 심볼릭 링크를 거부하지 않았습니다")
+
+            failed_restore = self._backup_tool(
+                "restore",
+                restored,
+                backup,
+                fail_after="database-restore",
+                check=False,
+            )
+            if (
+                failed_restore.returncode == 0
+                or "실패 주입: database-restore" not in failed_restore.stderr
+                or restored.project_resources()
+            ):
+                raise StackError("실패한 복원이 프로젝트 자원을 정리하지 못했습니다")
+
+            self._interrupt_backup_tool(
+                "restore",
+                restored,
+                backup,
+                pause_after="database-restore",
+                signum=signal.SIGINT,
+            )
+            remaining_resources = restored.project_resources()
+            if remaining_resources:
+                raise StackError(
+                    "SIGINT로 중단한 복원이 프로젝트 자원을 남겼습니다: "
+                    f"{remaining_resources}"
+                )
+            self._backup_tool("restore", restored, backup)
+            restored._verify_persistent_values(
+                post_id=post_id,
+                title=title,
+                content=content,
+                filename=filename,
+                file_value=file_value,
+            )
+            repeated = self._backup_tool("restore", restored, backup, check=False)
+            if repeated.returncode == 0 or "비어 있지 않습니다" not in repeated.stderr:
+                raise StackError("복원 도구가 사용 중인 프로젝트를 거부하지 않았습니다")
+            restored_failed = False
+        finally:
+            restored.close(failed=restored_failed)
+        print("backup path safety, failure cleanup, fresh restore, and refusal passed")
 
     def collect_diagnostics(self) -> Path:
         destination = self.diagnostics_dir
diff --git a/tests/validate_stack.py b/tests/validate_stack.py
index 3151c40..7a92661 100755
--- a/tests/validate_stack.py
+++ b/tests/validate_stack.py
@@ -184,8 +184,44 @@ def validate_tools() -> None:
             r"runtime_stack\.py e2e",
             r"^persistence:",
             r"runtime_stack\.py persistence",
+            r"^backup:",
+            r"stack_backup\.py backup",
+            r"^restore:",
+            r"stack_backup\.py restore",
+            r"^backup-restore-test:",
+            r"runtime_stack\.py backup-restore",
         ],
     )
+    require_text(
+        "tools/stack_backup.py",
+        [
+            r"--single-transaction",
+            r"sha256",
+            r"ensure_fresh_project",
+            r"validate_archive",
+            r"os\.replace",
+            r"O_NOFOLLOW",
+            r"fcntl\.flock",
+            r"defaults-extra-file",
+            r"mktemp /run/container-stack",
+            r"output\.mkdir\(mode=0o700\)",
+            r"fsync_directory",
+            r"cleanup_failed_restore",
+            r"database-restore",
+            r"input_stream",
+            r"operation_signal_handlers",
+            r"signal\.SIGINT",
+            r"signal\.SIGTERM",
+            r"project_operation_lock",
+            r"Path\([\"']\/tmp[\"']\)",
+            r"pause_for_test",
+            r"--pause-after",
+            r"--pause-ready-file",
+        ],
+    )
+    backup_tool = require_file("tools/stack_backup.py").read_text()
+    if re.search(r"-p(?:\\?['\"])?\$\(cat", backup_tool):
+        fail("database client passwords must not be exposed in command arguments")
     require_text(
         "tests/runtime_stack.py",
         [
@@ -196,6 +232,28 @@ def validate_tools() -> None:
             r'"bootstrap",\s*"e2e"',
             r"def verify_persistence",
             r"len\(initial_volumes\) != 3",
+            r'command = \["docker", kind, "ls"\]',
+            r'"restart"',
+            r'"down", "--remove-orphans"',
+            r"def verify_backup_restore",
+            r"missing-backup-target",
+            r"database-dump",
+            r"database-restore",
+            r"BACKUP_TOOL_TIMEOUT_SECONDS\s*=\s*1200",
+            r"time\.monotonic\(\)",
+            r"process\.kill\(\)",
+            r"--pause-after",
+            r"--pause-ready-file",
+            r"backup-stop",
+            r"signal\.SIGTERM",
+            r"signal\.SIGINT",
+            r"def project_resources",
+            r"def verify_services_running",
+            r'"ps", "--status", "running", "--services"',
+            r"\(\"container\", \"volume\", \"network\"\)",
+            r"command\.append\(\"--all\"\)",
+            r"TMPDIR",
+            r"다른 관리 작업이 실행 중입니다",
         ],
     )
 


## `test(backup): 자원 충돌과 시그널 경계 검증`

diff --git a/tests/runtime_stack.py b/tests/runtime_stack.py
index e5080d3..e56d919 100644
--- a/tests/runtime_stack.py
+++ b/tests/runtime_stack.py
@@ -707,6 +707,58 @@ class RuntimeStack:
                 )
             time.sleep(0.1)
 
+    def _verify_pause_signal_race(self) -> None:
+        script = "\n".join(
+            (
+                "import sys",
+                "from pathlib import Path",
+                "sys.path.insert(0, sys.argv[1])",
+                "from stack_backup import BackupError, operation_signal_handlers, pause_for_test",
+                "try:",
+                "    with operation_signal_handlers():",
+                "        pause_for_test('race', 'race', Path(sys.argv[2]))",
+                "except BackupError as error:",
+                "    print(error, file=sys.stderr)",
+                "    raise SystemExit(1)",
+            )
+        )
+        for index in range(12):
+            ready_file = self.temp / f"pause-race-{index}.ready"
+            process = subprocess.Popen(
+                [
+                    sys.executable,
+                    "-c",
+                    script,
+                    str(ROOT / "tools"),
+                    str(ready_file),
+                ],
+                cwd=ROOT,
+                text=True,
+                stdout=subprocess.PIPE,
+                stderr=subprocess.PIPE,
+            )
+            current_signal = signal.SIGINT if index % 2 == 0 else signal.SIGTERM
+            try:
+                self._wait_for_ready_file(
+                    process, ready_file, "신호 경합 준비 파일"
+                )
+                process.send_signal(current_signal)
+                stdout, stderr = process.communicate(
+                    timeout=PROCESS_TIMEOUT_SECONDS
+                )
+            finally:
+                if process.poll() is None:
+                    self._terminate_process(process)
+            if (
+                process.returncode != 1
+                or current_signal.name not in stderr
+                or ready_file.exists()
+            ):
+                raise StackError(
+                    "관리 작업 준비 파일의 신호 경합 검사가 실패했습니다: "
+                    f"{stderr.strip() or stdout.strip()}"
+                )
+
     def _interrupt_bootstrap(
         self, *, action: str, service: str, stage: str
     ) -> None:
@@ -1004,8 +1056,105 @@ class RuntimeStack:
             self._terminate_process(holder)
             ready_file.unlink(missing_ok=True)
 
+    def _verify_restore_resource_refusal(
+        self, backup: Path, restored: "RuntimeStack"
+    ) -> None:
+        labelled_name = f"{restored.project}-stopped"
+        create = subprocess.run(
+            [
+                "docker",
+                "create",
+                "--name",
+                labelled_name,
+                "--label",
+                f"com.docker.compose.project={restored.project}",
+                f"{self.project}-image-mariadb:local",
+            ],
+            check=True,
+            text=True,
+            capture_output=True,
+            timeout=PROCESS_TIMEOUT_SECONDS,
+        )
+        container_id = create.stdout.strip()
+        try:
+            refused = self._backup_tool(
+                "restore", restored, backup, check=False
+            )
+            inspect = subprocess.run(
+                ["docker", "container", "inspect", container_id],
+                text=True,
+                capture_output=True,
+                timeout=PROCESS_TIMEOUT_SECONDS,
+            )
+            if (
+                refused.returncode == 0
+                or "비어 있지 않습니다" not in refused.stderr
+                or inspect.returncode != 0
+            ):
+                raise StackError("복원이 정지 컨테이너를 보존하며 거부되지 않았습니다")
+        finally:
+            subprocess.run(
+                ["docker", "rm", "--force", container_id],
+                check=False,
+                text=True,
+                capture_output=True,
+                timeout=PROCESS_TIMEOUT_SECONDS,
+            )
+
+        collisions = (
+            ("container", f"{restored.project}-wordpress-1"),
+            ("volume", f"{restored.project}_mariadb_data"),
+            ("network", f"{restored.project}_backend"),
+        )
+        for kind, name in collisions:
+            if kind == "container":
+                command = [
+                    "docker",
+                    "container",
+                    "create",
+                    "--name",
+                    name,
+                    f"{self.project}-image-wordpress:local",
+                ]
+            else:
+                command = ["docker", kind, "create", name]
+            subprocess.run(
+                command,
+                check=True,
+                text=True,
+                capture_output=True,
+                timeout=PROCESS_TIMEOUT_SECONDS,
+            )
+            try:
+                refused = self._backup_tool(
+                    "restore", restored, backup, check=False
+                )
+                inspect = subprocess.run(
+                    ["docker", kind, "inspect", name],
+                    text=True,
+                    capture_output=True,
+                    timeout=PROCESS_TIMEOUT_SECONDS,
+                )
+                if (
+                    refused.returncode == 0
+                    or "비어 있지 않습니다" not in refused.stderr
+                    or inspect.returncode != 0
+                ):
+                    raise StackError(
+                        f"복원이 기존 {kind} 자원을 보존하며 거부되지 않았습니다"
+                    )
+            finally:
+                subprocess.run(
+                    ["docker", kind, "rm", name],
+                    check=False,
+                    text=True,
+                    capture_output=True,
+                    timeout=PROCESS_TIMEOUT_SECONDS,
+                )
+
     def verify_backup_restore(self) -> None:
         self.start()
+        self._verify_pause_signal_race()
         nonce = secrets.token_hex(8)
         title = f"복원 검증 {nonce}"
         content = f"backup-database-{nonce}"
@@ -1026,6 +1175,33 @@ class RuntimeStack:
             "wp_mkdir_p(WP_CONTENT_DIR . '/uploads'); "
             f'file_put_contents(WP_CONTENT_DIR . "/uploads/{filename}", "{file_value}");',
         )
+        large_filename = f"backup-large-{nonce}.bin"
+        self.run_compose(
+            "exec",
+            "--no-TTY",
+            "wordpress",
+            "sh",
+            "-ceu",
+            "umask 077; head -c 33554432 /dev/urandom >\"$1\"",
+            "large-backup-fixture",
+            f"/var/www/html/wp-content/uploads/{large_filename}",
+            capture=True,
+        )
+        large_hash = self.run_compose(
+            "exec",
+            "--no-TTY",
+            "wordpress",
+            "sha256sum",
+            f"/var/www/html/wp-content/uploads/{large_filename}",
+            capture=True,
+        ).stdout.split()[0]
+        table_name = f"container_stack_large_{nonce}"
+        self.wordpress(
+            "db",
+            "query",
+            f"CREATE TABLE {table_name} (payload LONGTEXT NOT NULL); "
+            f"INSERT INTO {table_name} VALUES (REPEAT('x', 4194304));",
+        )
         self._verify_shared_operation_lock()
         existing_backup = self.temp / "existing-backup"
         existing_backup.mkdir(mode=0o700)
@@ -1102,6 +1278,7 @@ class RuntimeStack:
         restored.started = True
         restored_failed = True
         try:
+            self._verify_restore_resource_refusal(backup, restored)
             unsafe_backup = self.temp / "unsafe-backup"
             unsafe_backup.mkdir(mode=0o700)
             for safe_name in ("manifest.json", "wordpress.tar.gz"):
@@ -1153,12 +1330,40 @@ class RuntimeStack:
                 filename=filename,
                 file_value=file_value,
             )
+            restored_large_hash = restored.run_compose(
+                "exec",
+                "--no-TTY",
+                "wordpress",
+                "sha256sum",
+                f"/var/www/html/wp-content/uploads/{large_filename}",
+                capture=True,
+            ).stdout.split()[0]
+            if restored_large_hash != large_hash:
+                raise StackError("큰 WordPress 파일의 복원 체크섬이 다릅니다")
+            restored_length = restored.wordpress(
+                "db",
+                "query",
+                f"SELECT LENGTH(payload) FROM {table_name};",
+                "--skip-column-names",
+                capture=True,
+            )
+            if restored_length != "4194304":
+                raise StackError("큰 MariaDB 값이 온전히 복원되지 않았습니다")
             repeated = self._backup_tool("restore", restored, backup, check=False)
             if repeated.returncode == 0 or "비어 있지 않습니다" not in repeated.stderr:
                 raise StackError("복원 도구가 사용 중인 프로젝트를 거부하지 않았습니다")
             restored_failed = False
         finally:
-            restored.close(failed=restored_failed)
+            cleanup_failures = restored.close(failed=restored_failed)
+            if cleanup_failures:
+                detail = "; ".join(cleanup_failures)
+                if restored_failed:
+                    print(
+                        f"복원 검증 정리 중 추가 오류가 발생했습니다: {detail}",
+                        file=sys.stderr,
+                    )
+                else:
+                    raise StackError(f"복원 검증 자원을 정리하지 못했습니다: {detail}")
         print("backup path safety, failure cleanup, fresh restore, and refusal passed")
 
     def _new_secret_set(self, name: str, prefix: str) -> tuple[Path, dict[str, str]]:
