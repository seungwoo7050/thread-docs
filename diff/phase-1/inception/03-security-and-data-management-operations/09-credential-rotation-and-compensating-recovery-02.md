## `feat(secrets): 회전 실패 시 기존 자격증명 복구`

diff --git a/tools/rotate_secrets.py b/tools/rotate_secrets.py
index eb51f15..7694ec1 100644
--- a/tools/rotate_secrets.py
+++ b/tools/rotate_secrets.py
@@ -528,3 +528,98 @@ def find_root_password(
         if root_sql(project, candidate, "SELECT 1;", check=False).returncode == 0:
             return candidate
     return None
+
+
+def rollback_rotation(
+    project: ComposeProject,
+    paths: dict[str, Path],
+    current: dict[str, str],
+    replacement: dict[str, str],
+    database_user: str,
+) -> tuple[list[str], bool]:
+    errors: list[str] = []
+    result = project.run(
+        "up",
+        "--detach",
+        "--no-recreate",
+        "--wait",
+        "--wait-timeout",
+        "180",
+        "mariadb",
+        capture=True,
+        check=False,
+    )
+    if result.returncode != 0:
+        detail = result.stderr.decode(errors="replace").strip()
+        errors.append(f"서비스 준비: {detail or 'docker compose up 실패'}")
+
+    root_auth = find_root_password(
+        project,
+        (replacement["db_root_password"], current["db_root_password"]),
+    )
+    if root_auth is None:
+        errors.append("DB 계정: 사용할 수 있는 root 자격증명을 찾지 못했습니다")
+    else:
+        try:
+            alter_database_passwords(
+                project,
+                root_auth,
+                database_user,
+                app_password=current["db_password"],
+            )
+        except Exception as error:
+            errors.append(f"DB 애플리케이션 계정: {error}")
+
+    try:
+        set_wordpress_db_config(
+            project,
+            current["db_password"],
+            one_off=True,
+        )
+    except Exception as error:
+        errors.append(f"WordPress DB 설정: {error}")
+
+    for kind, secret_name in (
+        ("admin", "wp_admin_password"),
+        ("user", "wp_user_password"),
+    ):
+        try:
+            set_wordpress_user(
+                project,
+                kind,
+                current[secret_name],
+                one_off=True,
+            )
+        except Exception as error:
+            errors.append(f"WordPress {kind} 계정: {error}")
+
+    if root_auth is not None:
+        try:
+            alter_database_passwords(
+                project,
+                root_auth,
+                database_user,
+                new_root_password=current["db_root_password"],
+            )
+        except Exception as error:
+            errors.append(f"DB root 계정: {error}")
+
+    for name, path in paths.items():
+        try:
+            atomic_secret_write(path, current[name])
+        except Exception as error:
+            errors.append(f"호스트 비밀 파일 {path.name}: {error}")
+
+    recovered = False
+    try:
+        project.run(
+            "up", "--detach", "--force-recreate", "--wait", "--wait-timeout", "300"
+        )
+        verify_rotation(project, database_user, current, replacement)
+        for name, path in paths.items():
+            if read_secret(path, require_owner=True) != current[name]:
+                raise RotationError(f"호스트 비밀 파일 복구 검증 실패: {path.name}")
+        recovered = True
+    except Exception as error:
+        errors.append(f"최종 재기동·검증: {error}")
+    return errors, recovered


## `feat(secrets): 스택 자격증명 회전 절차 연결`

diff --git a/Makefile b/Makefile
index 1e3019a..08913cf 100644
--- a/Makefile
+++ b/Makefile
@@ -4,10 +4,11 @@ ENV_FILE ?= .env
 PROJECT_NAME ?= container-stack
 WAIT_TIMEOUT ?= 300
 BACKUP_DIR ?=
+NEW_SECRETS_DIR ?=
 
 COMPOSE_RUN := $(COMPOSE) --project-name $(PROJECT_NAME) --env-file $(ENV_FILE) -f $(COMPOSE_FILE)
 
-.PHONY: up start-database start-application down build logs ps clean fclean test config smoke bootstrap-test e2e persistence backup restore backup-restore-test
+.PHONY: up start-database start-application down build logs ps clean fclean test config smoke bootstrap-test e2e persistence backup restore backup-restore-test rotate-secrets
 
 up:
 	python3 tools/start_stack.py start --project "$(PROJECT_NAME)" --env-file "$(ENV_FILE)" --wait-timeout "$(WAIT_TIMEOUT)"
@@ -69,3 +70,7 @@ restore:
 
 backup-restore-test:
 	python3 tests/runtime_stack.py backup-restore
+
+rotate-secrets:
+	@test -n "$(NEW_SECRETS_DIR)" || { echo "NEW_SECRETS_DIR is required" >&2; exit 2; }
+	python3 tools/rotate_secrets.py --project "$(PROJECT_NAME)" --env-file "$(ENV_FILE)" --new-secrets-dir "$(NEW_SECRETS_DIR)"
diff --git a/tools/rotate_secrets.py b/tools/rotate_secrets.py
index 7694ec1..fe18cd7 100644
--- a/tools/rotate_secrets.py
+++ b/tools/rotate_secrets.py
@@ -623,3 +623,102 @@ def rollback_rotation(
     except Exception as error:
         errors.append(f"최종 재기동·검증: {error}")
     return errors, recovered
+
+
+def _rotate(project: ComposeProject, new_secret_dir: Path) -> None:
+    config = compose_config(project)
+    paths = current_secret_paths(config, project.compose_file.parent)
+    canonical_paths = [path.resolve(strict=True) for path in paths.values()]
+    if len(set(canonical_paths)) != len(canonical_paths):
+        raise RotationError("Compose 비밀값 파일 경로는 서로 달라야 합니다")
+    current = {name: read_secret(path, require_owner=True) for name, path in paths.items()}
+    directory = new_secret_dir.expanduser()
+    directory_info = os.lstat(directory)
+    if not stat.S_ISDIR(directory_info.st_mode):
+        raise RotationError("새 비밀값 경로는 일반 디렉터리여야 합니다")
+    if directory_info.st_uid != os.getuid() or stat.S_IMODE(directory_info.st_mode) & 0o077:
+        raise RotationError("새 비밀값 디렉터리는 현재 사용자만 접근할 수 있어야 합니다")
+    replacement = {
+        name: read_secret(directory / filename, require_owner=True)
+        for name, filename in SECRET_FILES.items()
+    }
+    if any(current[name] == replacement[name] for name in SECRET_FILES):
+        raise RotationError("모든 비밀값은 기존 값과 달라야 합니다")
+    mariadb_environment = service_environment(config, "mariadb")
+    database_user = mariadb_environment.get("MYSQL_USER", "")
+    if not NAME_PATTERN.fullmatch(database_user):
+        raise RotationError("MYSQL_USER 형식이 안전하지 않습니다")
+    wordpress_environment = service_environment(config, "wordpress")
+    admin_user = wordpress_environment.get("WORDPRESS_ADMIN_USER", "")
+    regular_user = wordpress_environment.get("WORDPRESS_USER", "")
+    if not admin_user or not regular_user or admin_user == regular_user:
+        raise RotationError("WordPress 관리자와 일반 사용자는 서로 다른 계정이어야 합니다")
+    verify_rotation(project, database_user, current, replacement)
+    blocked = True
+    try:
+        project.run("stop", "nginx")
+        set_wordpress_user(project, "admin", replacement["wp_admin_password"])
+        set_wordpress_user(project, "user", replacement["wp_user_password"])
+        set_wordpress_db_config(project, replacement["db_password"])
+        alter_database_passwords(
+            project, current["db_root_password"], database_user,
+            app_password=replacement["db_password"],
+        )
+        alter_database_passwords(
+            project, current["db_root_password"], database_user,
+            new_root_password=replacement["db_root_password"],
+        )
+        for name, path in paths.items():
+            atomic_secret_write(path, replacement[name])
+        project.run("up", "--detach", "--force-recreate", "--wait", "--wait-timeout", "300")
+        verify_rotation(project, database_user, replacement, current)
+        for name, path in paths.items():
+            if read_secret(path, require_owner=True) != replacement[name]:
+                raise RotationError(f"호스트 비밀 파일 회전 검증 실패: {path.name}")
+        blocked = False
+    except BaseException as original_error:
+        rollback_errors, recovered = rollback_rotation(
+            project, paths, current, replacement, database_user
+        )
+        if recovered:
+            blocked = False
+        detail = "롤백 완료" if recovered else "롤백 불완전"
+        if rollback_errors:
+            detail += ": " + "; ".join(rollback_errors)
+        raise RotationError(f"회전 실패 ({original_error}); {detail}") from original_error
+    finally:
+        if blocked:
+            project.run("up", "--detach", check=False)
+    print("비밀값 회전과 재검증을 완료했습니다")
+
+
+def rotate(project: ComposeProject, new_secret_dir: Path) -> None:
+    with project_operation_lock(project.project):
+        _rotate(project, new_secret_dir)
+
+
+def parse_arguments() -> argparse.Namespace:
+    parser = argparse.ArgumentParser(description="컨테이너 스택 비밀값 회전")
+    parser.add_argument("--project", required=True)
+    parser.add_argument("--env-file", type=Path, required=True)
+    parser.add_argument("--compose-file", type=Path, default=DEFAULT_COMPOSE_FILE)
+    parser.add_argument("--new-secrets-dir", type=Path, required=True)
+    return parser.parse_args()
+
+
+def main() -> int:
+    args = parse_arguments()
+    if shutil.which("docker") is None:
+        print("docker 명령을 찾을 수 없습니다", file=sys.stderr)
+        return 2
+    try:
+        project = ComposeProject(args.project, args.env_file, args.compose_file)
+        rotate(project, args.new_secrets_dir)
+        return 0
+    except (BackupError, RotationError, OSError, subprocess.SubprocessError) as error:
+        print(f"비밀값 회전 실패: {error}", file=sys.stderr)
+        return 1
+
+
+if __name__ == "__main__":
+    raise SystemExit(main())


## `fix(secrets): 회전 중단과 불명확한 상태를 보상`

diff --git a/tools/rotate_secrets.py b/tools/rotate_secrets.py
index fe18cd7..5579d2e 100644
--- a/tools/rotate_secrets.py
+++ b/tools/rotate_secrets.py
@@ -9,6 +9,7 @@ import os
 from pathlib import Path
 import re
 import shutil
+import signal
 import stat
 import subprocess
 import sys
@@ -19,6 +20,7 @@ from stack_backup import (
     ComposeProject,
     DEFAULT_COMPOSE_FILE,
     QUERY_TIMEOUT_SECONDS,
+    pause_for_test,
     project_operation_lock,
 )
 from stack_runtime import StackRuntimeError, secret_source_paths
@@ -32,6 +34,21 @@ SECRET_FILES = {
 }
 PASSWORD_PATTERN = re.compile(r"^[A-Za-z0-9_.~!@#%^+=,-]{24,128}$")
 NAME_PATTERN = re.compile(r"^[A-Za-z0-9_]{1,64}$")
+FAILURE_STAGES = (
+    "admin-user-command",
+    "users",
+    "config-command",
+    "config",
+    "app-password-command",
+    "app-password",
+    "root-password-command",
+    "root-password",
+    "host-file",
+    "host-files",
+    "recreate-wordpress-removed",
+    "recreate",
+)
+PAUSE_STAGES = ("host-files",)
 NOFOLLOW = getattr(os, "O_NOFOLLOW", 0)
 NONBLOCK = getattr(os, "O_NONBLOCK", 0)
 DIRECTORY = getattr(os, "O_DIRECTORY", 0)
@@ -321,6 +338,28 @@ def alter_database_passwords(
     root_sql(project, root_password, ";\n".join(statements) + ";")
 
 
+def maybe_fail(stage: str | None, current: str) -> None:
+    if stage == current:
+        raise RotationError(f"실패 주입: {current}")
+
+
+def publish_test_marker(path: Path, value: str) -> Path:
+    marker = path.expanduser()
+    if not marker.is_absolute():
+        marker = Path.cwd() / marker
+    descriptor = os.open(
+        marker,
+        os.O_WRONLY | os.O_CREAT | os.O_EXCL | NOFOLLOW,
+        0o600,
+    )
+    try:
+        os.write(descriptor, (value + "\n").encode())
+        os.fsync(descriptor)
+    finally:
+        os.close(descriptor)
+    return marker
+
+
 def app_sql(
     project: ComposeProject,
     database_user: str,
@@ -625,7 +664,15 @@ def rollback_rotation(
     return errors, recovered
 
 
-def _rotate(project: ComposeProject, new_secret_dir: Path) -> None:
+def _rotate(
+    project: ComposeProject,
+    new_secret_dir: Path,
+    failure_stage: str | None,
+    pause_stage: str | None,
+    pause_ready_file: Path | None,
+    rollback_ready_file: Path | None,
+    signal_state: dict[str, bool],
+) -> None:
     config = compose_config(project)
     paths = current_secret_paths(config, project.compose_file.parent)
     canonical_paths = [path.resolve(strict=True) for path in paths.values()]
@@ -633,7 +680,10 @@ def _rotate(project: ComposeProject, new_secret_dir: Path) -> None:
         raise RotationError("Compose 비밀값 파일 경로는 서로 달라야 합니다")
     current = {name: read_secret(path, require_owner=True) for name, path in paths.items()}
     directory = new_secret_dir.expanduser()
-    directory_info = os.lstat(directory)
+    try:
+        directory_info = os.lstat(directory)
+    except OSError as error:
+        raise RotationError("새 비밀값 경로를 확인할 수 없습니다") from error
     if not stat.S_ISDIR(directory_info.st_mode):
         raise RotationError("새 비밀값 경로는 일반 디렉터리여야 합니다")
     if directory_info.st_uid != os.getuid() or stat.S_IMODE(directory_info.st_mode) & 0o077:
@@ -644,6 +694,7 @@ def _rotate(project: ComposeProject, new_secret_dir: Path) -> None:
     }
     if any(current[name] == replacement[name] for name in SECRET_FILES):
         raise RotationError("모든 비밀값은 기존 값과 달라야 합니다")
+
     mariadb_environment = service_environment(config, "mariadb")
     database_user = mariadb_environment.get("MYSQL_USER", "")
     if not NAME_PATTERN.fullmatch(database_user):
@@ -653,38 +704,82 @@ def _rotate(project: ComposeProject, new_secret_dir: Path) -> None:
     regular_user = wordpress_environment.get("WORDPRESS_USER", "")
     if not admin_user or not regular_user or admin_user == regular_user:
         raise RotationError("WordPress 관리자와 일반 사용자는 서로 다른 계정이어야 합니다")
+
     verify_rotation(project, database_user, current, replacement)
-    blocked = True
+    blocked = False
     try:
+        blocked = True
         project.run("stop", "nginx")
-        set_wordpress_user(project, "admin", replacement["wp_admin_password"])
+        set_wordpress_user(
+            project,
+            "admin",
+            replacement["wp_admin_password"],
+            fail_after_write=failure_stage == "admin-user-command",
+        )
         set_wordpress_user(project, "user", replacement["wp_user_password"])
-        set_wordpress_db_config(project, replacement["db_password"])
+        maybe_fail(failure_stage, "users")
+        set_wordpress_db_config(
+            project,
+            replacement["db_password"],
+            fail_after_write=failure_stage == "config-command",
+        )
+        maybe_fail(failure_stage, "config")
         alter_database_passwords(
-            project, current["db_root_password"], database_user,
+            project,
+            current["db_root_password"],
+            database_user,
             app_password=replacement["db_password"],
+            fail_after_write=failure_stage == "app-password-command",
         )
+        maybe_fail(failure_stage, "app-password")
         alter_database_passwords(
-            project, current["db_root_password"], database_user,
+            project,
+            current["db_root_password"],
+            database_user,
             new_root_password=replacement["db_root_password"],
+            fail_after_write=failure_stage == "root-password-command",
         )
-        for name, path in paths.items():
+        maybe_fail(failure_stage, "root-password")
+        for index, (name, path) in enumerate(paths.items()):
             atomic_secret_write(path, replacement[name])
-        project.run("up", "--detach", "--force-recreate", "--wait", "--wait-timeout", "300")
+            if index == 0:
+                maybe_fail(failure_stage, "host-file")
+        maybe_fail(failure_stage, "host-files")
+        pause_for_test(pause_stage, "host-files", pause_ready_file)
+        if failure_stage == "recreate-wordpress-removed":
+            project.run("rm", "--stop", "--force", "wordpress")
+            raise RotationError("실패 주입: recreate-wordpress-removed")
+        project.run(
+            "up", "--detach", "--force-recreate", "--wait", "--wait-timeout", "300"
+        )
+        maybe_fail(failure_stage, "recreate")
         verify_rotation(project, database_user, replacement, current)
         for name, path in paths.items():
             if read_secret(path, require_owner=True) != replacement[name]:
                 raise RotationError(f"호스트 비밀 파일 회전 검증 실패: {path.name}")
         blocked = False
     except BaseException as original_error:
-        rollback_errors, recovered = rollback_rotation(
-            project, paths, current, replacement, database_user
-        )
+        signal_state["rollback_active"] = True
+        marker: Path | None = None
+        try:
+            if rollback_ready_file is not None:
+                marker = publish_test_marker(rollback_ready_file, "rollback")
+            rollback_errors, recovered = rollback_rotation(
+                project, paths, current, replacement, database_user
+            )
+        finally:
+            if marker is not None:
+                marker.unlink(missing_ok=True)
         if recovered:
             blocked = False
-        detail = "롤백 완료" if recovered else "롤백 불완전"
-        if rollback_errors:
-            detail += ": " + "; ".join(rollback_errors)
+        if recovered:
+            detail = "롤백 완료"
+            if rollback_errors:
+                detail += "; 중간 보상 오류: " + "; ".join(rollback_errors)
+        else:
+            detail = "롤백 불완전: " + "; ".join(rollback_errors)
+        if signal_state["deferred"]:
+            detail += "; 롤백 중 추가 종료 신호 지연 처리"
         raise RotationError(f"회전 실패 ({original_error}); {detail}") from original_error
     finally:
         if blocked:
@@ -692,9 +787,40 @@ def _rotate(project: ComposeProject, new_secret_dir: Path) -> None:
     print("비밀값 회전과 재검증을 완료했습니다")
 
 
-def rotate(project: ComposeProject, new_secret_dir: Path) -> None:
-    with project_operation_lock(project.project):
-        _rotate(project, new_secret_dir)
+def rotate(
+    project: ComposeProject,
+    new_secret_dir: Path,
+    failure_stage: str | None,
+    pause_stage: str | None = None,
+    pause_ready_file: Path | None = None,
+    rollback_ready_file: Path | None = None,
+) -> None:
+    previous_handlers: dict[signal.Signals, object] = {}
+    signal_state = {"rollback_active": False, "deferred": False}
+
+    def interrupt(signum: int, _frame: object) -> None:
+        if signal_state["rollback_active"]:
+            signal_state["deferred"] = True
+            return
+        signal_name = signal.Signals(signum).name
+        raise RotationError(f"{signal_name} 신호로 회전이 중단되었습니다")
+
+    for current_signal in (signal.SIGINT, signal.SIGTERM):
+        previous_handlers[current_signal] = signal.signal(current_signal, interrupt)
+    try:
+        with project_operation_lock(project.project):
+            _rotate(
+                project,
+                new_secret_dir,
+                failure_stage,
+                pause_stage,
+                pause_ready_file,
+                rollback_ready_file,
+                signal_state,
+            )
+    finally:
+        for current_signal, previous in previous_handlers.items():
+            signal.signal(current_signal, previous)
 
 
 def parse_arguments() -> argparse.Namespace:
@@ -703,6 +829,10 @@ def parse_arguments() -> argparse.Namespace:
     parser.add_argument("--env-file", type=Path, required=True)
     parser.add_argument("--compose-file", type=Path, default=DEFAULT_COMPOSE_FILE)
     parser.add_argument("--new-secrets-dir", type=Path, required=True)
+    parser.add_argument("--fail-after", choices=FAILURE_STAGES, help=argparse.SUPPRESS)
+    parser.add_argument("--pause-after", choices=PAUSE_STAGES, help=argparse.SUPPRESS)
+    parser.add_argument("--pause-ready-file", type=Path, help=argparse.SUPPRESS)
+    parser.add_argument("--rollback-ready-file", type=Path, help=argparse.SUPPRESS)
     return parser.parse_args()
 
 
@@ -712,8 +842,19 @@ def main() -> int:
         print("docker 명령을 찾을 수 없습니다", file=sys.stderr)
         return 2
     try:
+        if (args.pause_after is None) != (args.pause_ready_file is None):
+            raise RotationError("일시정지 단계와 준비 파일을 함께 지정해야 합니다")
+        if args.rollback_ready_file is not None and args.pause_after is None:
+            raise RotationError("롤백 준비 파일은 일시정지 검사와 함께 지정해야 합니다")
         project = ComposeProject(args.project, args.env_file, args.compose_file)
-        rotate(project, args.new_secrets_dir)
+        rotate(
+            project,
+            args.new_secrets_dir,
+            args.fail_after,
+            args.pause_after,
+            args.pause_ready_file,
+            args.rollback_ready_file,
+        )
         return 0
     except (BackupError, RotationError, OSError, subprocess.SubprocessError) as error:
         print(f"비밀값 회전 실패: {error}", file=sys.stderr)


