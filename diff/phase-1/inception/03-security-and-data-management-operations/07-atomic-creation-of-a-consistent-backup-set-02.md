## `feat(backup): 백업 CLI와 Make 타깃 연결`

diff --git a/Makefile b/Makefile
index 05cc1a5..a616140 100644
--- a/Makefile
+++ b/Makefile
@@ -3,10 +3,11 @@ COMPOSE_FILE := srcs/docker-compose.yml
 ENV_FILE ?= .env
 PROJECT_NAME ?= container-stack
 WAIT_TIMEOUT ?= 300
+BACKUP_DIR ?=
 
 COMPOSE_RUN := $(COMPOSE) --project-name $(PROJECT_NAME) --env-file $(ENV_FILE) -f $(COMPOSE_FILE)
 
-.PHONY: up start-database start-application down build logs ps clean fclean test config smoke bootstrap-test e2e persistence
+.PHONY: up start-database start-application down build logs ps clean fclean test config smoke bootstrap-test e2e persistence backup
 
 up:
 	python3 tools/start_stack.py start --project "$(PROJECT_NAME)" --env-file "$(ENV_FILE)" --wait-timeout "$(WAIT_TIMEOUT)"
@@ -57,3 +58,7 @@ e2e:
 
 persistence:
 	python3 tests/runtime_stack.py persistence
+
+backup:
+	@test -n "$(BACKUP_DIR)" || { echo "BACKUP_DIR is required" >&2; exit 2; }
+	python3 tools/stack_backup.py backup --project "$(PROJECT_NAME)" --env-file "$(ENV_FILE)" --output "$(BACKUP_DIR)"
diff --git a/tools/stack_backup.py b/tools/stack_backup.py
index 19427f3..743bc0e 100644
--- a/tools/stack_backup.py
+++ b/tools/stack_backup.py
@@ -3,6 +3,7 @@
 
 from __future__ import annotations
 
+import argparse
 from contextlib import contextmanager
 from datetime import datetime, timezone
 import fcntl
@@ -15,12 +16,14 @@ import shutil
 import signal
 import stat
 import subprocess
+import sys
 import tarfile
 import tempfile
 import time
 from typing import BinaryIO, Iterator
 
 from stack_runtime import (
+    StackRuntimeError,
     load_secret_values,
     secret_payload,
 )
@@ -532,3 +535,38 @@ def create_backup(
                 pause_stage,
                 pause_ready_file,
             )
+
+
+def parse_arguments() -> argparse.Namespace:
+    parser = argparse.ArgumentParser(description="컨테이너 스택 백업")
+    parser.add_argument("operation", choices=("backup",))
+    parser.add_argument("--project", required=True)
+    parser.add_argument("--env-file", type=Path, required=True)
+    parser.add_argument("--compose-file", type=Path, default=DEFAULT_COMPOSE_FILE)
+    parser.add_argument("--output", type=Path, required=True)
+    parser.add_argument("--fail-after", choices=("database-dump",), help=argparse.SUPPRESS)
+    parser.add_argument("--pause-after", choices=("backup-stop",), help=argparse.SUPPRESS)
+    parser.add_argument("--pause-ready-file", type=Path, help=argparse.SUPPRESS)
+    return parser.parse_args()
+
+
+def main() -> int:
+    args = parse_arguments()
+    if shutil.which("docker") is None:
+        print("docker 명령을 찾을 수 없습니다", file=sys.stderr)
+        return 2
+    try:
+        if (args.pause_after is None) != (args.pause_ready_file is None):
+            raise BackupError("일시정지 단계와 준비 파일을 함께 지정해야 합니다")
+        project = ComposeProject(args.project, args.env_file, args.compose_file)
+        create_backup(
+            project, args.output, args.fail_after, args.pause_after, args.pause_ready_file
+        )
+        return 0
+    except (BackupError, StackRuntimeError, OSError, subprocess.SubprocessError) as error:
+        print(f"backup 실패: {error}", file=sys.stderr)
+        return 1
+
+
+if __name__ == "__main__":
+    raise SystemExit(main())


## `test(backup): 게시 실패와 중단 정리 검증`

diff --git a/Makefile b/Makefile
index a616140..97ea445 100644
--- a/Makefile
+++ b/Makefile
@@ -7,7 +7,7 @@ BACKUP_DIR ?=
 
 COMPOSE_RUN := $(COMPOSE) --project-name $(PROJECT_NAME) --env-file $(ENV_FILE) -f $(COMPOSE_FILE)
 
-.PHONY: up start-database start-application down build logs ps clean fclean test config smoke bootstrap-test e2e persistence backup
+.PHONY: up start-database start-application down build logs ps clean fclean test config smoke bootstrap-test e2e persistence backup backup-restore-test
 
 up:
 	python3 tools/start_stack.py start --project "$(PROJECT_NAME)" --env-file "$(ENV_FILE)" --wait-timeout "$(WAIT_TIMEOUT)"
@@ -62,3 +62,6 @@ persistence:
 backup:
 	@test -n "$(BACKUP_DIR)" || { echo "BACKUP_DIR is required" >&2; exit 2; }
 	python3 tools/stack_backup.py backup --project "$(PROJECT_NAME)" --env-file "$(ENV_FILE)" --output "$(BACKUP_DIR)"
+
+backup-restore-test:
+	python3 tests/runtime_stack.py backup-restore
diff --git a/tests/runtime_stack.py b/tests/runtime_stack.py
index d9eef12..da52251 100644
--- a/tests/runtime_stack.py
+++ b/tests/runtime_stack.py
@@ -9,23 +9,27 @@ import os
 from pathlib import Path
 import secrets
 import shutil
+import signal
 import socket
 import stat
 import subprocess
 import sys
 import tempfile
+import time
 
 
 ROOT = Path(__file__).resolve().parents[1]
 COMPOSE_FILE = ROOT / "srcs" / "docker-compose.yml"
-CONTROL_TIMEOUT_SECONDS = 330
+PROCESS_TIMEOUT_SECONDS = 120
+CONTROL_TIMEOUT_SECONDS = 600
 BUILD_TIMEOUT_SECONDS = 1200
-PROCESS_TIMEOUT_SECONDS = 60
+BACKUP_TOOL_TIMEOUT_SECONDS = 1200
 PORT_RETRY_LIMIT = 3
 PORT_CONFLICT_MARKERS = (
     "address already in use",
-    "bind for 127.0.0.1",
+    "bind: address already in use",
     "port is already allocated",
+    "failed to bind host port",
 )
 
 
@@ -60,7 +64,13 @@ def replace_private(path: Path, value: str) -> None:
 
 
 class RuntimeStack:
-    def __init__(self, *, keep: bool, diagnostics_dir: Path | None) -> None:
+    def __init__(
+        self,
+        *,
+        keep: bool,
+        diagnostics_dir: Path | None,
+        credential_values: dict[str, str] | None = None,
+    ) -> None:
         self.keep = keep
         self.diagnostics_dir = diagnostics_dir
         self.temp = Path(tempfile.mkdtemp(prefix="container-stack-e2e-"))
@@ -70,7 +80,7 @@ class RuntimeStack:
         self.port = reserve_port()
         self.env_file = self.temp / ".env"
         self.started = False
-        self.credential_values = {
+        self.credential_values = credential_values or {
             "db_root_password.txt": f"root#-{secrets.token_urlsafe(24)}",
             "db_password.txt": f"db#-{secrets.token_urlsafe(24)}",
             "wp_admin_password.txt": f"admin-{secrets.token_urlsafe(24)}",
@@ -233,23 +243,45 @@ class RuntimeStack:
         )
         return result.stdout.strip() if capture else ""
 
+    def project_resources(self) -> dict[str, set[str]]:
+        resources: dict[str, set[str]] = {}
+        for kind in ("container", "volume", "network"):
+            command = ["docker", kind, "ls"]
+            if kind == "container":
+                command.append("--all")
+            command.extend(
+                (
+                    "--filter",
+                    f"label=com.docker.compose.project={self.project}",
+                    "--format",
+                    "{{.Names}}" if kind == "container" else "{{.Name}}",
+                )
+            )
+            result = subprocess.run(
+                command,
+                check=True,
+                text=True,
+                capture_output=True,
+                timeout=PROCESS_TIMEOUT_SECONDS,
+            )
+            identifiers = {line for line in result.stdout.splitlines() if line}
+            if identifiers:
+                resources[kind] = identifiers
+        return resources
+
     def project_volumes(self) -> set[str]:
-        result = subprocess.run(
-            [
-                "docker",
-                "volume",
-                "ls",
-                "--filter",
-                f"label=com.docker.compose.project={self.project}",
-                "--format",
-                "{{.Name}}",
-            ],
-            check=True,
-            text=True,
-            capture_output=True,
-            timeout=PROCESS_TIMEOUT_SECONDS,
+        return self.project_resources().get("volume", set())
+
+    def verify_services_running(self) -> None:
+        result = self.run_compose(
+            "ps", "--status", "running", "--services", capture=True
         )
-        return {line for line in result.stdout.splitlines() if line}
+        running = {line for line in result.stdout.splitlines() if line}
+        expected = {"mariadb", "wordpress", "nginx"}
+        if running != expected or self.fetch("/healthz").strip() != "ok":
+            raise StackError(
+                f"관리 작업 뒤 서비스가 모두 복구되지 않았습니다: {sorted(running)}"
+            )
 
     def inspect_service(self, service: str) -> dict[str, object]:
         container_id = self.run_compose(
@@ -537,6 +569,314 @@ class RuntimeStack:
         )
         print(f"restart and recreation persistence passed: project={self.project}")
 
+    def _backup_tool(
+        self,
+        operation: str,
+        project: "RuntimeStack",
+        path: Path,
+        *,
+        fail_after: str | None = None,
+        environment: dict[str, str] | None = None,
+        check: bool = True,
+    ) -> subprocess.CompletedProcess[str]:
+        path_option = "--output" if operation == "backup" else "--input"
+        command = [
+            sys.executable,
+            str(ROOT / "tools" / "stack_backup.py"),
+            operation,
+            "--project",
+            project.project,
+            "--env-file",
+            str(project.env_file),
+            path_option,
+            str(path),
+        ]
+        if fail_after is not None:
+            command.extend(("--fail-after", fail_after))
+        process = subprocess.Popen(
+            command,
+            cwd=ROOT,
+            text=True,
+            stdout=subprocess.PIPE,
+            stderr=subprocess.PIPE,
+            env=environment,
+        )
+        try:
+            stdout, stderr = process.communicate(timeout=BACKUP_TOOL_TIMEOUT_SECONDS)
+        except subprocess.TimeoutExpired as error:
+            stdout, stderr = self._terminate_process(process)
+            raise StackError(
+                f"{operation} 도구가 {BACKUP_TOOL_TIMEOUT_SECONDS}초 안에 끝나지 않았습니다: "
+                f"{stderr.strip() or stdout.strip()}"
+            ) from error
+        result = subprocess.CompletedProcess(command, process.returncode, stdout, stderr)
+        if check and result.returncode != 0:
+            raise StackError(
+                f"{operation} 도구가 실패했습니다: "
+                f"{result.stderr.strip() or result.stdout.strip()}"
+            )
+        return result
+
+    def _terminate_process(
+        self, process: subprocess.Popen[str]
+    ) -> tuple[str, str]:
+        if process.poll() is None:
+            process.send_signal(signal.SIGTERM)
+        try:
+            return process.communicate(timeout=PROCESS_TIMEOUT_SECONDS)
+        except subprocess.TimeoutExpired:
+            process.kill()
+            try:
+                return process.communicate(timeout=PROCESS_TIMEOUT_SECONDS)
+            except subprocess.TimeoutExpired:
+                if process.stdout is not None:
+                    process.stdout.close()
+                if process.stderr is not None:
+                    process.stderr.close()
+                process.wait(timeout=PROCESS_TIMEOUT_SECONDS)
+                return "", "종료된 자식 프로세스가 출력 파이프를 닫지 않았습니다"
+
+    def _wait_for_ready_file(
+        self,
+        process: subprocess.Popen[str],
+        ready_file: Path,
+        description: str,
+    ) -> None:
+        deadline = time.monotonic() + PROCESS_TIMEOUT_SECONDS
+        while not ready_file.exists():
+            if process.poll() is not None:
+                stdout, stderr = self._terminate_process(process)
+                raise StackError(
+                    f"{description} 준비 전에 프로세스가 끝났습니다: "
+                    f"{stderr.strip() or stdout.strip()}"
+                )
+            if time.monotonic() >= deadline:
+                stdout, stderr = self._terminate_process(process)
+                ready_file.unlink(missing_ok=True)
+                raise StackError(
+                    f"{description} 준비를 {PROCESS_TIMEOUT_SECONDS}초 안에 확인하지 못했습니다: "
+                    f"{stderr.strip() or stdout.strip()}"
+                )
+            time.sleep(0.1)
+
+    def _interrupt_backup_tool(
+        self,
+        operation: str,
+        project: "RuntimeStack",
+        path: Path,
+        *,
+        pause_after: str,
+        signum: signal.Signals,
+    ) -> subprocess.CompletedProcess[str]:
+        ready_file = project.temp / f"{operation}-{pause_after}.ready"
+        path_option = "--output" if operation == "backup" else "--input"
+        command = [
+            sys.executable,
+            str(ROOT / "tools" / "stack_backup.py"),
+            operation,
+            "--project",
+            project.project,
+            "--env-file",
+            str(project.env_file),
+            path_option,
+            str(path),
+            "--pause-after",
+            pause_after,
+            "--pause-ready-file",
+            str(ready_file),
+        ]
+        process = subprocess.Popen(
+            command,
+            cwd=ROOT,
+            text=True,
+            stdout=subprocess.PIPE,
+            stderr=subprocess.PIPE,
+        )
+        try:
+            self._wait_for_ready_file(
+                process, ready_file, f"{operation} {pause_after} 일시정지"
+            )
+            process.send_signal(signum)
+            try:
+                stdout, stderr = process.communicate(timeout=PROCESS_TIMEOUT_SECONDS)
+            except subprocess.TimeoutExpired as error:
+                stdout, stderr = self._terminate_process(process)
+                raise StackError(
+                    f"{operation} 신호 정리가 {PROCESS_TIMEOUT_SECONDS}초 안에 끝나지 않았습니다"
+                ) from error
+        finally:
+            if process.poll() is None:
+                self._terminate_process(process)
+        result = subprocess.CompletedProcess(command, process.returncode, stdout, stderr)
+        if (
+            result.returncode != 1
+            or signum.name not in result.stderr
+            or ready_file.exists()
+        ):
+            raise StackError(
+                f"{operation} 도구가 {signum.name} 중단을 안전하게 처리하지 못했습니다: "
+                f"{result.stderr.strip() or result.stdout.strip()}"
+            )
+        return result
+
+    def _verify_shared_operation_lock(self) -> None:
+        first_tmp = self.temp / "lock-tmp-first"
+        second_tmp = self.temp / "lock-tmp-second"
+        first_tmp.mkdir(mode=0o700)
+        second_tmp.mkdir(mode=0o700)
+        ready_file = self.temp / "operation-lock.ready"
+        holder_script = "\n".join(
+            (
+                "import os, signal, sys, time",
+                "from pathlib import Path",
+                "sys.path.insert(0, sys.argv[1])",
+                "from stack_backup import project_operation_lock",
+                "ready = Path(sys.argv[3])",
+                "signal.signal(signal.SIGTERM, lambda *_: sys.exit(0))",
+                "try:",
+                "    with project_operation_lock(sys.argv[2]):",
+                "        descriptor = os.open(ready, os.O_WRONLY | os.O_CREAT | os.O_EXCL, 0o600)",
+                "        os.write(descriptor, b'locked\\n')",
+                "        os.fsync(descriptor)",
+                "        os.close(descriptor)",
+                "        time.sleep(3600)",
+                "finally:",
+                "    ready.unlink(missing_ok=True)",
+            )
+        )
+        holder_environment = os.environ.copy()
+        holder_environment["TMPDIR"] = str(first_tmp)
+        holder = subprocess.Popen(
+            [
+                sys.executable,
+                "-c",
+                holder_script,
+                str(ROOT / "tools"),
+                self.project,
+                str(ready_file),
+            ],
+            cwd=ROOT,
+            text=True,
+            stdout=subprocess.PIPE,
+            stderr=subprocess.PIPE,
+            env=holder_environment,
+        )
+        try:
+            self._wait_for_ready_file(holder, ready_file, "공유 관리 잠금")
+            contender_environment = os.environ.copy()
+            contender_environment["TMPDIR"] = str(second_tmp)
+            contested_output = self.temp / "contested-backup"
+            result = self._backup_tool(
+                "backup",
+                self,
+                contested_output,
+                environment=contender_environment,
+                check=False,
+            )
+            if (
+                result.returncode == 0
+                or "다른 관리 작업이 실행 중입니다" not in result.stderr
+                or contested_output.exists()
+                or list(self.temp.glob(".contested-backup.tmp-*"))
+            ):
+                raise StackError("서로 다른 TMPDIR의 관리 작업이 같은 잠금을 공유하지 않았습니다")
+            self.verify_services_running()
+        finally:
+            self._terminate_process(holder)
+            ready_file.unlink(missing_ok=True)
+
+    def verify_backup_restore(self) -> None:
+        self.start()
+        nonce = secrets.token_hex(8)
+        title = f"복원 검증 {nonce}"
+        content = f"backup-database-{nonce}"
+        filename = f"backup-{nonce}.txt"
+        file_value = f"backup-volume-{nonce}\n"
+        post_id = self.wordpress(
+            "post",
+            "create",
+            f"--post_title={title}",
+            f"--post_content={content}",
+            "--post_status=publish",
+            "--porcelain",
+            capture=True,
+        )
+        self.wordpress("option", "update", "container_stack_persistence", content)
+        self.wordpress(
+            "eval",
+            "wp_mkdir_p(WP_CONTENT_DIR . '/uploads'); "
+            f'file_put_contents(WP_CONTENT_DIR . "/uploads/{filename}", "{file_value}");',
+        )
+        self._verify_shared_operation_lock()
+        existing_backup = self.temp / "existing-backup"
+        existing_backup.mkdir(mode=0o700)
+        write_private(existing_backup / "sentinel.txt", "preserve")
+        existing_snapshot = (existing_backup / "sentinel.txt").read_bytes()
+        existing_result = self._backup_tool(
+            "backup", self, existing_backup, check=False
+        )
+        if (
+            existing_result.returncode == 0
+            or "이미 존재합니다" not in existing_result.stderr
+            or (existing_backup / "sentinel.txt").read_bytes() != existing_snapshot
+            or set(path.name for path in existing_backup.iterdir()) != {"sentinel.txt"}
+        ):
+            raise StackError("백업 도구가 기존 출력 디렉터리를 안전하게 보존하지 않았습니다")
+
+        dangling_backup = self.temp / "backup-link"
+        missing_target = self.temp / "missing-backup-target"
+        dangling_backup.symlink_to(missing_target, target_is_directory=True)
+        dangling_result = self._backup_tool(
+            "backup", self, dangling_backup, check=False
+        )
+        if (
+            dangling_result.returncode == 0
+            or "이미 존재합니다" not in dangling_result.stderr
+            or not dangling_backup.is_symlink()
+            or missing_target.exists()
+        ):
+            raise StackError("백업 도구가 dangling symlink 출력 경로를 거부하지 않았습니다")
+
+        failed_backup = self.temp / "failed-backup"
+        failed_result = self._backup_tool(
+            "backup",
+            self,
+            failed_backup,
+            fail_after="database-dump",
+            check=False,
+        )
+        if (
+            failed_result.returncode == 0
+            or "실패 주입: database-dump" not in failed_result.stderr
+            or failed_backup.exists()
+            or list(self.temp.glob(".failed-backup.tmp-*"))
+        ):
+            raise StackError("실패한 백업이 임시 파일을 남겼거나 서비스를 복구하지 못했습니다")
+        self.verify_services_running()
+
+        backup = self.temp / "backup"
+        self._interrupt_backup_tool(
+            "backup",
+            self,
+            backup,
+            pause_after="backup-stop",
+            signum=signal.SIGTERM,
+        )
+        if (
+            backup.exists()
+            or list(self.temp.glob(".backup.tmp-*"))
+        ):
+            raise StackError("SIGTERM으로 중단한 백업이 출력·임시 파일을 정리하거나 서비스를 복구하지 못했습니다")
+        self.verify_services_running()
+        self._backup_tool("backup", self, backup)
+        if (backup.stat().st_mode & 0o077) != 0:
+            raise StackError("백업 디렉터리가 소유자 외 사용자에게 열려 있습니다")
+        for filename_in_backup in ("database.sql", "wordpress.tar.gz", "manifest.json"):
+            if ((backup / filename_in_backup).stat().st_mode & 0o077) != 0:
+                raise StackError(f"백업 파일 권한이 안전하지 않습니다: {filename_in_backup}")
+
+        print("backup publication, interruption, and cleanup passed")
+
     def collect_diagnostics(self) -> Path:
         destination = self.diagnostics_dir
         if destination is None:
@@ -578,7 +918,10 @@ class RuntimeStack:
 
 def parse_arguments() -> argparse.Namespace:
     parser = argparse.ArgumentParser(description="격리된 컨테이너 스택 검증")
-    parser.add_argument("scenario", choices=("bootstrap", "e2e", "persistence"))
+    parser.add_argument(
+        "scenario",
+        choices=("bootstrap", "e2e", "persistence", "backup-restore"),
+    )
     parser.add_argument("--keep", action="store_true", help="검사 뒤 프로젝트를 유지합니다")
     parser.add_argument("--diagnostics-dir", type=Path)
     return parser.parse_args()
@@ -606,8 +949,10 @@ def main() -> int:
             stack.verify_bootstrap()
         elif args.scenario == "persistence":
             stack.verify_persistence()
-        else:
+        elif args.scenario == "e2e":
             stack.verify_e2e()
+        else:
+            stack.verify_backup_restore()
         failed = False
         return 0
     except (OSError, StackError, subprocess.SubprocessError) as error:


