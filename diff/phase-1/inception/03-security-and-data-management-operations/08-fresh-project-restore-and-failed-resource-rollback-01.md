# 신규 프로젝트 복원과 실패 자원 롤백

## `feat(restore): Compose 리소스 이름과 기존 객체 조회`

diff --git a/tools/stack_backup.py b/tools/stack_backup.py
index 743bc0e..6d81c6a 100644
--- a/tools/stack_backup.py
+++ b/tools/stack_backup.py
@@ -209,6 +209,28 @@ class ComposeProject:
                 f"Compose {operation} 명령이 {timeout}초 안에 끝나지 않았습니다"
             ) from error
 
+    def labelled_resources(self, kind: str) -> set[str]:
+        format_field = "{{.Names}}" if kind == "container" else "{{.Name}}"
+        command = ["docker", kind, "ls"]
+        if kind == "container":
+            command.append("--all")
+        command.extend(
+            (
+                "--filter",
+                f"label=com.docker.compose.project={self.project}",
+                "--format",
+                format_field,
+            )
+        )
+        result = subprocess.run(
+            command,
+            check=True,
+            text=True,
+            capture_output=True,
+            timeout=QUERY_TIMEOUT_SECONDS,
+        )
+        return {line for line in result.stdout.splitlines() if line}
+
     def config(self) -> dict[str, object]:
         result = self.run(
             "config",
@@ -241,6 +263,78 @@ class ComposeProject:
         }
 
 
+def rendered_resource_names(project: ComposeProject) -> dict[str, set[str]]:
+    result = project.run(
+        "config",
+        "--format",
+        "json",
+        capture=True,
+        timeout=QUERY_TIMEOUT_SECONDS,
+    )
+    try:
+        config = json.loads(result.stdout)
+    except (UnicodeDecodeError, json.JSONDecodeError) as error:
+        raise BackupError(f"Compose 설정 JSON을 읽을 수 없습니다: {error}") from error
+    if not isinstance(config, dict):
+        raise BackupError("Compose 설정이 객체 형식이 아닙니다")
+
+    names: dict[str, set[str]] = {"volume": set(), "network": set()}
+    for config_key, resource_kind in (("volumes", "volume"), ("networks", "network")):
+        resources = config.get(config_key, {})
+        if not isinstance(resources, dict):
+            raise BackupError(f"Compose {config_key} 설정이 객체 형식이 아닙니다")
+        for resource in resources.values():
+            if not isinstance(resource, dict) or not isinstance(resource.get("name"), str):
+                raise BackupError(f"Compose {resource_kind} 이름을 확인할 수 없습니다")
+            names[resource_kind].add(resource["name"])
+    return names
+
+
+def existing_named_resources(kind: str, expected: set[str]) -> set[str]:
+    if not expected:
+        return set()
+    result = subprocess.run(
+        ["docker", kind, "ls", "--format", "{{.Name}}"],
+        check=True,
+        text=True,
+        capture_output=True,
+        timeout=QUERY_TIMEOUT_SECONDS,
+    )
+    return expected.intersection(result.stdout.splitlines())
+
+
+def expected_container_names(project: ComposeProject) -> set[str]:
+    config = project.config()
+    services = config.get("services")
+    if not isinstance(services, dict):
+        raise BackupError("Compose 서비스 설정을 찾을 수 없습니다")
+    names: set[str] = set()
+    for service, service_config in services.items():
+        if not isinstance(service, str) or not isinstance(service_config, dict):
+            raise BackupError("Compose 서비스 설정 형식이 올바르지 않습니다")
+        configured_name = service_config.get("container_name")
+        if configured_name is not None:
+            if not isinstance(configured_name, str):
+                raise BackupError("Compose 컨테이너 이름이 문자열이 아닙니다")
+            names.add(configured_name)
+        else:
+            names.add(f"{project.project}-{service}-1")
+            names.add(f"{project.project}_{service}_1")
+        names.add(f"{project.project}-{service}-bootstrap")
+    return names
+
+
+def existing_named_containers(expected: set[str]) -> set[str]:
+    result = subprocess.run(
+        ["docker", "container", "ls", "--all", "--format", "{{.Names}}"],
+        check=True,
+        text=True,
+        capture_output=True,
+        timeout=QUERY_TIMEOUT_SECONDS,
+    )
+    return expected.intersection(result.stdout.splitlines())
+
+
 def validate_archive_stream(stream: BinaryIO) -> None:
     try:
         stream.seek(0)


## `feat(restore): 대상 프로젝트 자원 충돌 사전 차단`

diff --git a/tools/stack_backup.py b/tools/stack_backup.py
index 6d81c6a..9693fa8 100644
--- a/tools/stack_backup.py
+++ b/tools/stack_backup.py
@@ -335,6 +335,25 @@ def existing_named_containers(expected: set[str]) -> set[str]:
     return expected.intersection(result.stdout.splitlines())
 
 
+def ensure_fresh_project(project: ComposeProject) -> None:
+    found: dict[str, set[str]] = {}
+    for kind in ("container", "volume", "network"):
+        identifiers = project.labelled_resources(kind)
+        if identifiers:
+            found[kind] = identifiers
+    named_containers = existing_named_containers(expected_container_names(project))
+    if named_containers:
+        found.setdefault("container", set()).update(named_containers)
+    rendered = rendered_resource_names(project)
+    for kind in ("volume", "network"):
+        identifiers = existing_named_resources(kind, rendered[kind])
+        if identifiers:
+            found.setdefault(kind, set()).update(identifiers)
+    if found:
+        summary = ", ".join(f"{kind}={len(items)}" for kind, items in found.items())
+        raise BackupError(f"복원 대상 프로젝트가 비어 있지 않습니다: {summary}")
+
+
 def validate_archive_stream(stream: BinaryIO) -> None:
     try:
         stream.seek(0)


## `feat(restore): 백업 입력의 형식과 체크섬 검증`

diff --git a/tools/stack_backup.py b/tools/stack_backup.py
index 9693fa8..ca9116e 100644
--- a/tools/stack_backup.py
+++ b/tools/stack_backup.py
@@ -383,6 +383,113 @@ def validate_archive(path: Path) -> None:
         validate_archive_stream(stream)
 
 
+class VerifiedBackup:
+    def __init__(
+        self,
+        directory_descriptor: int,
+        database: BinaryIO,
+        wordpress: BinaryIO,
+        manifest: dict[str, object],
+    ) -> None:
+        self.directory_descriptor = directory_descriptor
+        self.database = database
+        self.wordpress = wordpress
+        self.manifest = manifest
+
+    def __enter__(self) -> "VerifiedBackup":
+        return self
+
+    def __exit__(self, *_: object) -> None:
+        self.database.close()
+        self.wordpress.close()
+        os.close(self.directory_descriptor)
+
+
+def open_regular_file(directory_descriptor: int, filename: str) -> BinaryIO:
+    try:
+        descriptor = os.open(
+            filename,
+            os.O_RDONLY | NOFOLLOW | NONBLOCK,
+            dir_fd=directory_descriptor,
+        )
+    except OSError as error:
+        raise BackupError(f"백업 파일을 안전하게 열 수 없습니다: {filename}") from error
+    info = os.fstat(descriptor)
+    if (
+        not stat.S_ISREG(info.st_mode)
+        or info.st_nlink != 1
+        or info.st_uid != os.getuid()
+        or stat.S_IMODE(info.st_mode) & 0o077
+    ):
+        os.close(descriptor)
+        raise BackupError(f"백업 항목의 형식이나 권한이 안전하지 않습니다: {filename}")
+    try:
+        fcntl.flock(descriptor, fcntl.LOCK_SH | fcntl.LOCK_NB)
+    except OSError as error:
+        os.close(descriptor)
+        raise BackupError(f"다른 작업이 백업 파일을 변경하고 있습니다: {filename}") from error
+    return os.fdopen(descriptor, "rb")
+
+
+def load_and_verify_backup(source: Path) -> VerifiedBackup:
+    source = source.expanduser()
+    if not source.is_absolute():
+        source = Path.cwd() / source
+    try:
+        directory_descriptor = os.open(
+            source,
+            os.O_RDONLY | DIRECTORY | NOFOLLOW,
+        )
+    except OSError as error:
+        raise BackupError("백업 입력은 심볼릭 링크가 아닌 디렉터리여야 합니다") from error
+    opened: list[BinaryIO] = []
+    try:
+        directory_info = os.fstat(directory_descriptor)
+        if (
+            not stat.S_ISDIR(directory_info.st_mode)
+            or directory_info.st_uid != os.getuid()
+            or stat.S_IMODE(directory_info.st_mode) & 0o077
+        ):
+            raise BackupError("백업 입력 디렉터리의 형식이나 권한이 안전하지 않습니다")
+        present = set(os.listdir(directory_descriptor))
+        if present != EXPECTED_FILES:
+            raise BackupError(f"백업 파일 구성이 올바르지 않습니다: {sorted(present)}")
+        manifest_stream = open_regular_file(directory_descriptor, MANIFEST)
+        opened.append(manifest_stream)
+        if os.fstat(manifest_stream.fileno()).st_size > 64 * 1024:
+            raise BackupError("백업 manifest가 허용 크기를 넘었습니다")
+        try:
+            manifest = json.loads(manifest_stream.read().decode("utf-8"))
+        except (UnicodeDecodeError, json.JSONDecodeError) as error:
+            raise BackupError(f"백업 manifest를 읽을 수 없습니다: {error}") from error
+        if not isinstance(manifest, dict) or manifest.get("format") != 1:
+            raise BackupError("지원하지 않는 백업 형식입니다")
+        checksums = manifest.get("sha256")
+        if not isinstance(checksums, dict):
+            raise BackupError("백업 manifest에 체크섬이 없습니다")
+        database = open_regular_file(directory_descriptor, DATABASE_DUMP)
+        opened.append(database)
+        wordpress = open_regular_file(directory_descriptor, WORDPRESS_ARCHIVE)
+        opened.append(wordpress)
+        for filename, stream in (
+            (DATABASE_DUMP, database),
+            (WORDPRESS_ARCHIVE, wordpress),
+        ):
+            expected = checksums.get(filename)
+            actual = sha256_stream(stream)
+            if not isinstance(expected, str) or expected != actual:
+                raise BackupError(f"백업 체크섬이 일치하지 않습니다: {filename}")
+        validate_archive_stream(wordpress)
+        manifest_stream.close()
+        opened.remove(manifest_stream)
+        return VerifiedBackup(directory_descriptor, database, wordpress, manifest)
+    except Exception:
+        for stream in opened:
+            stream.close()
+        os.close(directory_descriptor)
+        raise
+
+
 @contextmanager
 def project_operation_lock(project_name: str) -> Iterator[None]:
     lock_directory = Path("/tmp") / f"container-stack-operation-locks-{os.getuid()}"


## `feat(restore): DB와 WordPress 데이터를 새 볼륨에 주입`

diff --git a/tools/stack_backup.py b/tools/stack_backup.py
index ca9116e..b31dbba 100644
--- a/tools/stack_backup.py
+++ b/tools/stack_backup.py
@@ -27,6 +27,7 @@ from stack_runtime import (
     load_secret_values,
     secret_payload,
 )
+from start_stack import start_application, start_database
 
 
 ROOT = Path(__file__).resolve().parents[1]
@@ -757,6 +758,48 @@ def create_backup(
             )
 
 
+def restore_database(
+    project: ComposeProject, source: BinaryIO, root_password: str
+) -> None:
+    with tempfile.TemporaryFile(mode="w+b") as payload:
+        payload.write(secret_payload(root_password))
+        shutil.copyfileobj(source, payload, length=1024 * 1024)
+        payload.seek(0)
+        project.run(
+            "exec",
+            "--no-TTY",
+            "mariadb",
+            "sh",
+            "-ceu",
+            "umask 077; auth=\"$(mktemp /run/container-stack-restore.XXXXXX)\"; "
+            "trap 'rm -f -- \"$auth\"' EXIT HUP INT TERM; "
+            "IFS= read -r password; "
+            "printf '[client]\\npassword=\"%s\"\\n' \"$password\" >\"$auth\"; "
+            "mariadb --defaults-extra-file=\"$auth\" "
+            "--socket=/run/mysqld/mysqld.sock -uroot",
+            input_stream=payload,
+            timeout=TRANSFER_TIMEOUT_SECONDS,
+        )
+
+
+def restore_wordpress(project: ComposeProject, source: BinaryIO) -> None:
+    project.run(
+        "run",
+        "--rm",
+        "--no-TTY",
+        "--no-deps",
+        "--entrypoint",
+        "sh",
+        "wordpress",
+        "-ceu",
+        "test -z \"$(find /var/www/html -mindepth 1 -print -quit)\"; "
+        "test -z \"$(find /var/www/config -mindepth 1 -print -quit)\"; "
+        "exec tar -xzf - -C /var/www",
+        input_stream=source,
+        timeout=TRANSFER_TIMEOUT_SECONDS,
+    )
+
+
 def parse_arguments() -> argparse.Namespace:
     parser = argparse.ArgumentParser(description="컨테이너 스택 백업")
     parser.add_argument("operation", choices=("backup",))


## `feat(restore): 실패한 복원 자원을 정리하고 롤백`

diff --git a/tools/stack_backup.py b/tools/stack_backup.py
index b31dbba..6c7e861 100644
--- a/tools/stack_backup.py
+++ b/tools/stack_backup.py
@@ -800,6 +800,89 @@ def restore_wordpress(project: ComposeProject, source: BinaryIO) -> None:
     )
 
 
+def cleanup_failed_restore(project: ComposeProject) -> None:
+    result = project.run(
+        "down",
+        "--volumes",
+        "--remove-orphans",
+        "--timeout",
+        "20",
+        capture=True,
+        check=False,
+        timeout=CONTROL_TIMEOUT_SECONDS,
+    )
+    remaining = {
+        kind: project.labelled_resources(kind)
+        for kind in ("container", "volume", "network")
+    }
+    remaining["container"].update(
+        existing_named_containers(expected_container_names(project))
+    )
+    rendered = rendered_resource_names(project)
+    for kind in ("volume", "network"):
+        remaining[kind].update(existing_named_resources(kind, rendered[kind]))
+    remaining = {kind: values for kind, values in remaining.items() if values}
+    if result.returncode != 0 or remaining:
+        detail = result.stderr.decode(errors="replace").strip()
+        raise BackupError(f"실패한 복원 자원을 정리하지 못했습니다: {remaining}; {detail}")
+
+
+def restore_backup(
+    project: ComposeProject,
+    source: Path,
+    failure_stage: str | None = None,
+    pause_stage: str | None = None,
+    pause_ready_file: Path | None = None,
+) -> None:
+    with operation_signal_handlers():
+        with project_operation_lock(project.project):
+            with load_and_verify_backup(source) as backup:
+                ensure_fresh_project(project)
+                secrets = load_secret_values(project)
+                restoration_started = False
+                try:
+                    restoration_started = True
+                    start_database(
+                        project,
+                        secrets,
+                        build=True,
+                        pause_after_stage=None,
+                        pause_ready_file=None,
+                    )
+                    restore_database(
+                        project,
+                        backup.database,
+                        secrets["db_root_password"],
+                    )
+                    pause_for_test(
+                        pause_stage, "database-restore", pause_ready_file
+                    )
+                    maybe_fail(failure_stage, "database-restore")
+                    project.run(
+                        "build",
+                        "wordpress",
+                        timeout=TRANSFER_TIMEOUT_SECONDS,
+                    )
+                    restore_wordpress(project, backup.wordpress)
+                    start_application(
+                        project,
+                        secrets,
+                        build=True,
+                        pause_after_stage=None,
+                        pause_ready_file=None,
+                    )
+                except BaseException as original_error:
+                    if restoration_started:
+                        try:
+                            cleanup_failed_restore(project)
+                        except Exception as cleanup_error:
+                            raise BackupError(
+                                f"복원과 실패 자원 정리가 모두 실패했습니다: {cleanup_error}"
+                            ) from original_error
+                    raise
+    print(f"새 프로젝트에 백업을 복원했습니다: {project.project}")
+
+
 def parse_arguments() -> argparse.Namespace:
     parser = argparse.ArgumentParser(description="컨테이너 스택 백업")
     parser.add_argument("operation", choices=("backup",))


## `feat(restore): 복원 CLI와 Make 타깃 연결`

diff --git a/Makefile b/Makefile
index 97ea445..88e4abd 100644
--- a/Makefile
+++ b/Makefile
@@ -7,7 +7,7 @@ BACKUP_DIR ?=
 
 COMPOSE_RUN := $(COMPOSE) --project-name $(PROJECT_NAME) --env-file $(ENV_FILE) -f $(COMPOSE_FILE)
 
-.PHONY: up start-database start-application down build logs ps clean fclean test config smoke bootstrap-test e2e persistence backup backup-restore-test
+.PHONY: up start-database start-application down build logs ps clean fclean test config smoke bootstrap-test e2e persistence backup backup-restore-test restore
 
 up:
 	python3 tools/start_stack.py start --project "$(PROJECT_NAME)" --env-file "$(ENV_FILE)" --wait-timeout "$(WAIT_TIMEOUT)"
@@ -63,5 +63,9 @@ backup:
 	@test -n "$(BACKUP_DIR)" || { echo "BACKUP_DIR is required" >&2; exit 2; }
 	python3 tools/stack_backup.py backup --project "$(PROJECT_NAME)" --env-file "$(ENV_FILE)" --output "$(BACKUP_DIR)"
 
+restore:
+	@test -n "$(BACKUP_DIR)" || { echo "BACKUP_DIR is required" >&2; exit 2; }
+	python3 tools/stack_backup.py restore --project "$(PROJECT_NAME)" --env-file "$(ENV_FILE)" --input "$(BACKUP_DIR)"
+
 backup-restore-test:
 	python3 tests/runtime_stack.py backup-restore
diff --git a/tools/stack_backup.py b/tools/stack_backup.py
index 6c7e861..140c64c 100644
--- a/tools/stack_backup.py
+++ b/tools/stack_backup.py
@@ -884,14 +884,16 @@ def restore_backup(
 
 
 def parse_arguments() -> argparse.Namespace:
-    parser = argparse.ArgumentParser(description="컨테이너 스택 백업")
-    parser.add_argument("operation", choices=("backup",))
+    parser = argparse.ArgumentParser(description="컨테이너 스택 백업·복원")
+    parser.add_argument("operation", choices=("backup", "restore"))
     parser.add_argument("--project", required=True)
     parser.add_argument("--env-file", type=Path, required=True)
     parser.add_argument("--compose-file", type=Path, default=DEFAULT_COMPOSE_FILE)
-    parser.add_argument("--output", type=Path, required=True)
-    parser.add_argument("--fail-after", choices=("database-dump",), help=argparse.SUPPRESS)
-    parser.add_argument("--pause-after", choices=("backup-stop",), help=argparse.SUPPRESS)
+    paths = parser.add_mutually_exclusive_group(required=True)
+    paths.add_argument("--output", type=Path)
+    paths.add_argument("--input", type=Path)
+    parser.add_argument("--fail-after", choices=FAILURE_STAGES, help=argparse.SUPPRESS)
+    parser.add_argument("--pause-after", choices=PAUSE_STAGES, help=argparse.SUPPRESS)
     parser.add_argument("--pause-ready-file", type=Path, help=argparse.SUPPRESS)
     return parser.parse_args()
 
@@ -905,12 +907,42 @@ def main() -> int:
         if (args.pause_after is None) != (args.pause_ready_file is None):
             raise BackupError("일시정지 단계와 준비 파일을 함께 지정해야 합니다")
         project = ComposeProject(args.project, args.env_file, args.compose_file)
-        create_backup(
-            project, args.output, args.fail_after, args.pause_after, args.pause_ready_file
-        )
+        if args.operation == "backup":
+            if args.output is None:
+                raise BackupError("backup에는 --output이 필요합니다")
+            if args.fail_after not in (None, "database-dump"):
+                raise BackupError("backup에서 사용할 수 없는 실패 주입 단계입니다")
+            if args.pause_after not in (None, "backup-stop"):
+                raise BackupError("backup에서 사용할 수 없는 일시정지 단계입니다")
+            create_backup(
+                project,
+                args.output,
+                args.fail_after,
+                args.pause_after,
+                args.pause_ready_file,
+            )
+        else:
+            if args.input is None:
+                raise BackupError("restore에는 --input이 필요합니다")
+            if args.fail_after not in (None, "database-restore"):
+                raise BackupError("restore에서 사용할 수 없는 실패 주입 단계입니다")
+            if args.pause_after not in (None, "database-restore"):
+                raise BackupError("restore에서 사용할 수 없는 일시정지 단계입니다")
+            restore_backup(
+                project,
+                args.input,
+                args.fail_after,
+                args.pause_after,
+                args.pause_ready_file,
+            )
         return 0
-    except (BackupError, StackRuntimeError, OSError, subprocess.SubprocessError) as error:
-        print(f"backup 실패: {error}", file=sys.stderr)
+    except (
+        BackupError,
+        StackRuntimeError,
+        OSError,
+        subprocess.SubprocessError,
+    ) as error:
+        print(f"{args.operation} 실패: {error}", file=sys.stderr)
         return 1
 
 


