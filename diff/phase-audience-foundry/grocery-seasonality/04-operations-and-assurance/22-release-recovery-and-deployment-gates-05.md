## `fix(ops): harden postgres recovery boundaries`

diff --git a/scripts/postgres_backup_restore.py b/scripts/postgres_backup_restore.py
index daeb848..2247853 100644
--- a/scripts/postgres_backup_restore.py
+++ b/scripts/postgres_backup_restore.py
@@ -1,14 +1,16 @@
 """Local PostgreSQL custom-format backup and isolated restore assurance.
 
-The command is deliberately bound to the repository's fixed Docker Compose ``db``
-service. Database credentials and ``DATABASE_URL`` are neither read nor forwarded.
-All subprocess output is captured or discarded and converted to fixed error codes.
+The command talks directly to one fully inspected local Docker container over the fixed
+Unix socket. It does not use a mutable Docker context or the Compose plugin. Ambient
+database, Docker, and KAMIS credentials are neither read nor forwarded. All subprocess
+output is captured or discarded and converted to fixed error codes.
 """
 
 from __future__ import annotations
 
 import argparse
 import hashlib
+import hmac
 import json
 import os
 import re
@@ -26,14 +28,19 @@ from typing import Final, Never
 
 _SOURCE_DATABASE: Final = "grocery"
 _DATABASE_USER: Final = "grocery"
+_COMPOSE_PROJECT: Final = "audience-foundry-grocery-seasonality"
 _COMPOSE_SERVICE: Final = "db"
-_FORMAT_VERSION: Final = "grocery-postgres-custom-v1"
+_LOCAL_DOCKER_HOST: Final = "unix:///var/run/docker.sock"
+_LOCAL_DATABASE_PORT: Final = 55_434
+_LOCAL_DATABASE_PASSWORD: Final = "local-grocery-only"  # noqa: S105 - tracked local fixture
+_FORMAT_VERSION: Final = "grocery-postgres-custom-v2"
 _POSTGRES_MAJOR: Final = 18
 _DUMP_FILENAME: Final = "database.dump"
 _MANIFEST_FILENAME: Final = "manifest.json"
 _DUMP_MAGIC: Final = b"PGDMP"
 _MAX_MANIFEST_BYTES: Final = 4 * 1024 * 1024
 _MAX_INVENTORY_BYTES: Final = 4 * 1024 * 1024
+_MAX_DOCKER_INSPECT_BYTES: Final = 256 * 1024
 _MAX_DUMP_BYTES: Final = 8 * 1024 * 1024 * 1024
 _MAX_TABLES: Final = 1_024
 _MAX_MIGRATIONS: Final = 16_384
@@ -41,15 +48,20 @@ _MAX_ROW_COUNT: Final = (2**63) - 1
 _MAX_ACTIVATIONS: Final = 10_000
 _MAX_PUBLICATION_ENTRIES: Final = 100_000
 _TOOL_TIMEOUT_SECONDS: Final = 30
+_CREATE_INNER_TIMEOUT_SECONDS: Final = 20
 _INVENTORY_TIMEOUT_SECONDS: Final = 120
 _DUMP_TIMEOUT_SECONDS: Final = 600
 _RESTORE_TIMEOUT_SECONDS: Final = 600
+_RESTORE_INNER_TIMEOUT_SECONDS: Final = 570
+_INSPECTION_TIMEOUT_SECONDS: Final = 120
 _SHA256 = re.compile(r"[0-9a-f]{64}\Z")
 _TABLE_NAME = re.compile(r"public\.[a-z0-9_]{1,63}\Z")
 _MIGRATION_TOKEN = re.compile(r"[A-Za-z0-9_.-]{1,128}\Z")
 _REASON_CODE = re.compile(r"[A-Z][A-Z0-9_]{0,63}\Z")
 _TARGET_DATABASE = re.compile(r"grocery_restore_[a-z0-9][a-z0-9_]{0,45}\Z")
 _VERSION_TOKEN = re.compile(r"([0-9]+)(?:\.[0-9]+)*\Z")
+_CONTAINER_ID = re.compile(r"[0-9a-f]{64}\Z")
+_APPLICATION_NAME = re.compile(r"grocery_(?:backup|restore)_[0-9a-f]{32}\Z")
 
 _INVENTORY_SQL: Final = r"""
 CREATE TEMP TABLE assurance_counts (
@@ -176,6 +188,7 @@ _CODES: Final = frozenset(
         "create_target_failed",
         "destination_inside_repository",
         "docker_command_failed",
+        "database_container_invalid",
         "docker_unavailable",
         "dump_failed",
         "dump_file_invalid",
@@ -183,9 +196,11 @@ _CODES: Final = frozenset(
         "inventory_invalid",
         "inventory_mismatch",
         "migration_mismatch",
+        "manifest_receipt_mismatch",
         "postgres_tool_missing",
         "postgres_version_mismatch",
         "publication_contract_mismatch",
+        "publication_inspection_failed",
         "repository_invalid",
         "restore_failed",
         "row_count_mismatch",
@@ -193,6 +208,7 @@ _CODES: Final = frozenset(
         "target_database_exists",
         "target_database_invalid",
         "target_database_is_source",
+        "target_cleanup_failed",
         "target_preflight_failed",
         "usage_error",
     }
@@ -238,6 +254,55 @@ class Inventory:
         return hashlib.sha256(_canonical_json(self.publication)).hexdigest()
 
 
+@dataclass(frozen=True, slots=True)
+class CanonicalPublication:
+    version: int
+    current_revision_id: str
+    typed_fact_set_sha256: str
+    entry_count: int
+    last_activation_id: str
+    last_activation_operation: str
+    last_activation_sequence: int
+
+    def canonical_data(self) -> dict[str, object]:
+        return {
+            "channel": "RECENT_RETAIL",
+            "current_revision_id": self.current_revision_id,
+            "entry_count": self.entry_count,
+            "last_activation_id": self.last_activation_id,
+            "last_activation_operation": self.last_activation_operation,
+            "last_activation_sequence": self.last_activation_sequence,
+            "publication_state": "AVAILABLE",
+            "typed_fact_set_sha256": self.typed_fact_set_sha256,
+            "version": self.version,
+        }
+
+    @property
+    def sha256(self) -> str:
+        return hashlib.sha256(_canonical_json(self.canonical_data())).hexdigest()
+
+
+@dataclass(frozen=True, slots=True)
+class _FileIdentity:
+    device: int
+    inode: int
+    size: int
+    modified_ns: int
+    changed_ns: int
+
+
+@dataclass(frozen=True, slots=True)
+class _DatabaseContainer:
+    docker_binary: str
+    container_id: str
+
+
+@dataclass(frozen=True, slots=True)
+class _OpenedDirectory:
+    path: Path
+    descriptor: int
+
+
 @dataclass(frozen=True, slots=True)
 class BackupReceipt:
     backup_id: uuid.UUID
@@ -274,7 +339,8 @@ class RestoreReceipt:
                 "target_database_verified=yes",
                 "row_counts_consistent=yes",
                 "migrations_consistent=yes",
-                "publication_contract_consistent=yes",
+                "publication_metadata_consistent=yes",
+                "publication_canonical_consistent=yes",
                 f"tables={self.table_count}",
                 f"migrations={self.migration_count}",
                 "cleanup=drop_explicit_restore_target_after_review",
@@ -285,60 +351,91 @@ class RestoreReceipt:
 @dataclass(frozen=True, slots=True)
 class LoadedBackup:
     backup_id: uuid.UUID
-    dump_path: Path
+    dump_descriptor: int
+    dump_identity: _FileIdentity
     inventory: Inventory
+    canonical_publication: CanonicalPublication
 
 
 def create_backup(*, repository: Path, destination_root: Path) -> BackupReceipt:
     root = _repository_root(repository)
-    destination = _operator_directory(destination_root, repository=root)
-    _preflight(root)
-    backup_id = uuid.uuid4()
-    backup_directory = destination / f"postgres-backup-{backup_id}"
-
-    with _restrictive_umask():
-        try:
-            backup_directory.mkdir(mode=0o700)
-        except OSError:
-            raise BackupRestoreError("backup_directory_unavailable") from None
-        _require_private_directory(backup_directory)
-        dump_path = backup_directory / _DUMP_FILENAME
-        before = _read_inventory(root, _SOURCE_DATABASE)
-        dump_fd = _create_private_file(dump_path)
-        try:
-            _dump_database(root, dump_fd)
-            os.fsync(dump_fd)
-        finally:
-            os.close(dump_fd)
-        _require_private_regular_file(dump_path, maximum_bytes=_MAX_DUMP_BYTES)
-        if _read_prefix(dump_path, len(_DUMP_MAGIC)) != _DUMP_MAGIC:
-            raise BackupRestoreError("backup_not_custom_format")
-        after = _read_inventory(root, _SOURCE_DATABASE)
-        if before != after:
-            raise BackupRestoreError("backup_changed_during_dump")
-
-        dump_sha256, dump_bytes = _file_sha256(dump_path)
-        manifest = {
-            "backup_id": str(backup_id),
-            "created_at": datetime.now(tz=UTC).isoformat(timespec="seconds").replace("+00:00", "Z"),
-            "dump": {
-                "bytes": dump_bytes,
-                "filename": _DUMP_FILENAME,
-                "sha256": dump_sha256,
-            },
-            "format_version": _FORMAT_VERSION,
-            "inventory": {
-                **before.canonical_data(),
-                "publication_sha256": before.publication_sha256,
-                "sha256": before.sha256,
-            },
-            "postgres_major": _POSTGRES_MAJOR,
-            "source_database": _SOURCE_DATABASE,
-        }
-        manifest_bytes = _canonical_json(manifest) + b"\n"
-        manifest_path = backup_directory / _MANIFEST_FILENAME
-        _write_private_file(manifest_path, manifest_bytes)
-        _fsync_directory(backup_directory)
+    with _open_operator_directory(destination_root, repository=root) as destination:
+        application_name = _new_application_name("backup")
+        container = _preflight(root, application_name)
+        before_canonical = _inspect_publication(
+            root,
+            _SOURCE_DATABASE,
+            container,
+            application_name,
+        )
+        before = _read_inventory(root, _SOURCE_DATABASE, container, application_name)
+        _require_canonical_inventory_match(before_canonical, before)
+        backup_id = uuid.uuid4()
+        backup_name = f"postgres-backup-{backup_id}"
+
+        with _restrictive_umask():
+            backup_descriptor = _create_private_directory_at(
+                destination.descriptor,
+                backup_name,
+            )
+            try:
+                dump_fd = _create_private_file_at(backup_descriptor, _DUMP_FILENAME)
+                try:
+                    _dump_database(root, dump_fd, container, application_name)
+                    os.fsync(dump_fd)
+                    dump_identity = _require_private_regular_descriptor(
+                        dump_fd,
+                        maximum_bytes=_MAX_DUMP_BYTES,
+                    )
+                    if _read_prefix_descriptor(dump_fd, len(_DUMP_MAGIC)) != _DUMP_MAGIC:
+                        raise BackupRestoreError("backup_not_custom_format")
+                    dump_sha256, dump_bytes = _file_sha256_descriptor(dump_fd)
+                    _require_descriptor_identity(dump_fd, dump_identity)
+                finally:
+                    os.close(dump_fd)
+
+                after = _read_inventory(root, _SOURCE_DATABASE, container, application_name)
+                after_canonical = _inspect_publication(
+                    root,
+                    _SOURCE_DATABASE,
+                    container,
+                    application_name,
+                )
+                _require_canonical_inventory_match(after_canonical, after)
+                if before != after or before_canonical != after_canonical:
+                    raise BackupRestoreError("backup_changed_during_dump")
+
+                manifest = {
+                    "backup_id": str(backup_id),
+                    "created_at": (
+                        datetime.now(tz=UTC).isoformat(timespec="seconds").replace("+00:00", "Z")
+                    ),
+                    "dump": {
+                        "bytes": dump_bytes,
+                        "filename": _DUMP_FILENAME,
+                        "sha256": dump_sha256,
+                    },
+                    "format_version": _FORMAT_VERSION,
+                    "inventory": {
+                        **before.canonical_data(),
+                        "canonical_publication": before_canonical.canonical_data(),
+                        "canonical_publication_sha256": before_canonical.sha256,
+                        "publication_sha256": before.publication_sha256,
+                        "sha256": before.sha256,
+                    },
+                    "postgres_major": _POSTGRES_MAJOR,
+                    "source_database": _SOURCE_DATABASE,
+                }
+                manifest_bytes = _canonical_json(manifest) + b"\n"
+                _write_private_file_at(
+                    backup_descriptor,
+                    _MANIFEST_FILENAME,
+                    manifest_bytes,
+                )
+                _fsync_directory_descriptor(backup_descriptor)
+            finally:
+                os.close(backup_descriptor)
+            _fsync_directory_descriptor(destination.descriptor)
 
     return BackupReceipt(
         backup_id=backup_id,
@@ -354,33 +451,83 @@ def restore_backup(
     repository: Path,
     backup_directory: Path,
     target_database: str,
+    expected_manifest_sha256: str,
 ) -> RestoreReceipt:
     root = _repository_root(repository)
     target = _validated_target_database(target_database)
-    selected_backup = _load_backup(backup_directory, repository=root)
-    _preflight(root)
-    _validate_custom_dump(root, selected_backup.dump_path)
-    if _database_exists(root, target):
-        raise BackupRestoreError("target_database_exists")
-    _create_target_database(root, target)
-    _restore_database(root, selected_backup.dump_path, target)
-    restored = _read_inventory(root, target)
-    if restored.rows != selected_backup.inventory.rows:
-        raise BackupRestoreError("row_count_mismatch")
-    if restored.migrations != selected_backup.inventory.migrations:
-        raise BackupRestoreError("migration_mismatch")
-    if (
-        restored.publication != selected_backup.inventory.publication
-        or restored.publication_sha256 != selected_backup.inventory.publication_sha256
-    ):
-        raise BackupRestoreError("publication_contract_mismatch")
-    if restored.sha256 != selected_backup.inventory.sha256:
-        raise BackupRestoreError("inventory_mismatch")
-    return RestoreReceipt(
-        backup_id=selected_backup.backup_id,
-        table_count=len(restored.rows),
-        migration_count=len(restored.migrations),
-    )
+    expected_manifest = _validated_manifest_sha256(expected_manifest_sha256)
+    with _load_backup(
+        backup_directory,
+        repository=root,
+        expected_manifest_sha256=expected_manifest,
+    ) as selected_backup:
+        application_name = _new_application_name("restore")
+        container = _preflight(root, application_name)
+        _validate_custom_dump(
+            root,
+            selected_backup.dump_descriptor,
+            container,
+            application_name,
+        )
+        _require_descriptor_identity(
+            selected_backup.dump_descriptor,
+            selected_backup.dump_identity,
+        )
+        if _database_exists(root, target, container, application_name):
+            raise BackupRestoreError("target_database_exists")
+        cleanup_required = False
+        try:
+            _create_target_database(root, target, container, application_name)
+            # A failed or timed-out ``createdb`` is ambiguous: another local process
+            # may have won the same validated name after the absence check. Never
+            # delete by name until this invocation has observed a fully successful
+            # command and the post-command container identity check.
+            cleanup_required = True
+            _restore_database(
+                root,
+                selected_backup.dump_descriptor,
+                target,
+                container,
+                application_name,
+            )
+            _require_descriptor_identity(
+                selected_backup.dump_descriptor,
+                selected_backup.dump_identity,
+            )
+            restored = _read_inventory(root, target, container, application_name)
+            restored_canonical = _inspect_publication(
+                root,
+                target,
+                container,
+                application_name,
+            )
+            _require_canonical_inventory_match(restored_canonical, restored)
+            if restored.rows != selected_backup.inventory.rows:
+                raise BackupRestoreError("row_count_mismatch")
+            if restored.migrations != selected_backup.inventory.migrations:
+                raise BackupRestoreError("migration_mismatch")
+            if (
+                restored.publication != selected_backup.inventory.publication
+                or restored.publication_sha256 != selected_backup.inventory.publication_sha256
+            ):
+                raise BackupRestoreError("publication_contract_mismatch")
+            if restored_canonical != selected_backup.canonical_publication:
+                raise BackupRestoreError("publication_contract_mismatch")
+            if restored.sha256 != selected_backup.inventory.sha256:
+                raise BackupRestoreError("inventory_mismatch")
+            cleanup_required = False
+            return RestoreReceipt(
+                backup_id=selected_backup.backup_id,
+                table_count=len(restored.rows),
+                migration_count=len(restored.migrations),
+            )
+        except BackupRestoreError:
+            if cleanup_required:
+                try:
+                    _cleanup_target_database(root, target, container, application_name)
+                except BackupRestoreError:
+                    raise BackupRestoreError("target_cleanup_failed") from None
+            raise
 
 
 def main(arguments: list[str] | None = None) -> int:
@@ -405,6 +552,7 @@ def main(arguments: list[str] | None = None) -> int:
                 repository=Path.cwd(),
                 backup_directory=Path(parsed.backup_dir),
                 target_database=parsed.target_database,
+                expected_manifest_sha256=parsed.expected_manifest_sha256,
             )
     except BackupRestoreError as error:
         _print_failure(error.code, restore_requested=restore_requested)
@@ -424,6 +572,7 @@ def _parser() -> argparse.ArgumentParser:
     restore = subparsers.add_parser("restore")
     restore.add_argument("--backup-dir", required=True)
     restore.add_argument("--target-database", required=True)
+    restore.add_argument("--expected-manifest-sha256", required=True)
     return parser
 
 
@@ -432,7 +581,16 @@ def _print_failure(code: str, *, restore_requested: bool) -> None:
     print("status=failed")
     print(f"code={selected}")
     if restore_requested:
-        print("cleanup=inspect_and_drop_explicit_restore_target_if_created")
+        if selected in {
+            "create_target_failed",
+            "database_container_invalid",
+            "docker_command_failed",
+            "internal_error",
+            "target_cleanup_failed",
+        }:
+            print("cleanup=manual_target_cleanup_required")
+        else:
+            print("cleanup=automatic_created_target_cleanup_verified_or_not_created")
     else:
         print("cleanup=remove_incomplete_backup_directory_if_created")
 
@@ -451,23 +609,6 @@ def _repository_root(repository: Path) -> Path:
     return root
 
 
-def _operator_directory(path: Path, *, repository: Path) -> Path:
-    if not path.is_absolute():
-        raise BackupRestoreError("backup_directory_invalid")
-    try:
-        metadata = path.lstat()
-        resolved = path.resolve(strict=True)
-    except OSError:
-        raise BackupRestoreError("backup_directory_unavailable") from None
-    if stat.S_ISLNK(metadata.st_mode) or not stat.S_ISDIR(metadata.st_mode):
-        raise BackupRestoreError("backup_directory_invalid")
-    if metadata.st_uid != os.geteuid():
-        raise BackupRestoreError("backup_directory_permissions")
-    if _is_within(resolved, repository):
-        raise BackupRestoreError("destination_inside_repository")
-    return resolved
-
-
 def _validated_target_database(value: object) -> str:
     if value == _SOURCE_DATABASE:
         raise BackupRestoreError("target_database_is_source")
@@ -476,12 +617,30 @@ def _validated_target_database(value: object) -> str:
     return value
 
 
-def _preflight(repository: Path) -> None:
-    if shutil.which("docker") is None:
-        raise BackupRestoreError("docker_unavailable")
-    for tool in ("pg_dump", "pg_restore", "createdb", "psql"):
-        output = _capture_compose(
+def _validated_manifest_sha256(value: object) -> str:
+    if not isinstance(value, str) or _SHA256.fullmatch(value) is None:
+        raise BackupRestoreError("manifest_receipt_mismatch")
+    return value
+
+
+def _new_application_name(operation: str) -> str:
+    if operation not in {"backup", "restore"}:
+        raise BackupRestoreError("internal_error")
+    return f"grocery_{operation}_{uuid.uuid4().hex}"
+
+
+def _preflight(repository: Path, application_name: str) -> _DatabaseContainer:
+    _require_application_name(application_name)
+    docker_binary = _resolve_docker_binary()
+    container = _DatabaseContainer(
+        docker_binary=docker_binary,
+        container_id=_discover_database_container(repository, docker_binary),
+    )
+    for tool in ("pg_dump", "pg_restore", "createdb", "dropdb", "psql"):
+        output = _capture_database_command(
             repository,
+            container,
+            application_name,
             (tool, "--version"),
             timeout=_TOOL_TIMEOUT_SECONDS,
             failure_code="postgres_tool_missing",
@@ -497,8 +656,20 @@ def _preflight(repository: Path) -> None:
         match = _VERSION_TOKEN.fullmatch(version_token)
         if match is None or int(match.group(1)) != _POSTGRES_MAJOR:
             raise BackupRestoreError("postgres_version_mismatch")
-    source_probe = _capture_compose(
+    timeout_version = _capture_database_command(
         repository,
+        container,
+        application_name,
+        ("timeout", "--version"),
+        timeout=_TOOL_TIMEOUT_SECONDS,
+        failure_code="postgres_tool_missing",
+    )
+    if not timeout_version.startswith(b"timeout (GNU coreutils) ") or len(timeout_version) > 512:
+        raise BackupRestoreError("postgres_tool_missing")
+    source_probe = _capture_database_command(
+        repository,
+        container,
+        application_name,
         (
             "psql",
             "--no-password",
@@ -513,11 +684,19 @@ def _preflight(repository: Path) -> None:
     )
     if source_probe.strip() != b"1":
         raise BackupRestoreError("source_database_unavailable")
+    return container
 
 
-def _dump_database(repository: Path, output_fd: int) -> None:
-    _run_compose(
+def _dump_database(
+    repository: Path,
+    output_fd: int,
+    container: _DatabaseContainer,
+    application_name: str,
+) -> None:
+    _run_database_command(
         repository,
+        container,
+        application_name,
         (
             "pg_dump",
             "--no-password",
@@ -534,27 +713,35 @@ def _dump_database(repository: Path, output_fd: int) -> None:
     )
 
 
-def _validate_custom_dump(repository: Path, dump_path: Path) -> None:
-    descriptor = os.open(
-        dump_path,
-        os.O_RDONLY | getattr(os, "O_CLOEXEC", 0) | getattr(os, "O_NOFOLLOW", 0),
+def _validate_custom_dump(
+    repository: Path,
+    dump_descriptor: int,
+    container: _DatabaseContainer,
+    application_name: str,
+) -> None:
+    _rewind_descriptor(dump_descriptor)
+    _run_database_command(
+        repository,
+        container,
+        application_name,
+        ("pg_restore", "--list"),
+        stdin=dump_descriptor,
+        stdout=subprocess.DEVNULL,
+        timeout=_INVENTORY_TIMEOUT_SECONDS,
+        failure_code="backup_not_custom_format",
     )
-    try:
-        _run_compose(
-            repository,
-            ("pg_restore", "--list"),
-            stdin=descriptor,
-            stdout=subprocess.DEVNULL,
-            timeout=_INVENTORY_TIMEOUT_SECONDS,
-            failure_code="backup_not_custom_format",
-        )
-    finally:
-        os.close(descriptor)
 
 
-def _database_exists(repository: Path, target: str) -> bool:
-    output = _capture_compose(
+def _database_exists(
+    repository: Path,
+    target: str,
+    container: _DatabaseContainer,
+    application_name: str,
+) -> bool:
+    output = _capture_database_command(
         repository,
+        container,
+        application_name,
         (
             "psql",
             "--no-password",
@@ -577,10 +764,21 @@ def _database_exists(repository: Path, target: str) -> bool:
     return result == b"1"
 
 
-def _create_target_database(repository: Path, target: str) -> None:
-    _run_compose(
+def _create_target_database(
+    repository: Path,
+    target: str,
+    container: _DatabaseContainer,
+    application_name: str,
+) -> None:
+    _run_database_command(
         repository,
+        container,
+        application_name,
         (
+            "timeout",
+            "--signal=TERM",
+            "--kill-after=5s",
+            f"{_CREATE_INNER_TIMEOUT_SECONDS}s",
             "createdb",
             "--no-password",
             f"--username={_DATABASE_USER}",
@@ -596,36 +794,122 @@ def _create_target_database(repository: Path, target: str) -> None:
     )
 
 
-def _restore_database(repository: Path, dump_path: Path, target: str) -> None:
-    descriptor = os.open(
-        dump_path,
-        os.O_RDONLY | getattr(os, "O_CLOEXEC", 0) | getattr(os, "O_NOFOLLOW", 0),
+def _restore_database(
+    repository: Path,
+    dump_descriptor: int,
+    target: str,
+    container: _DatabaseContainer,
+    application_name: str,
+) -> None:
+    _rewind_descriptor(dump_descriptor)
+    _run_database_command(
+        repository,
+        container,
+        application_name,
+        (
+            "timeout",
+            "--signal=TERM",
+            "--kill-after=5s",
+            f"{_RESTORE_INNER_TIMEOUT_SECONDS}s",
+            "pg_restore",
+            "--no-password",
+            f"--username={_DATABASE_USER}",
+            f"--dbname={target}",
+            "--exit-on-error",
+            "--single-transaction",
+            "--no-owner",
+            "--no-privileges",
+        ),
+        stdin=dump_descriptor,
+        stdout=subprocess.DEVNULL,
+        timeout=_RESTORE_TIMEOUT_SECONDS,
+        failure_code="restore_failed",
     )
-    try:
-        _run_compose(
-            repository,
+
+
+def _cleanup_target_database(
+    repository: Path,
+    target: str,
+    container: _DatabaseContainer,
+    application_name: str,
+) -> None:
+    terminate = _capture_database_command(
+        repository,
+        container,
+        application_name,
+        (
+            "psql",
+            "--no-password",
+            f"--username={_DATABASE_USER}",
+            "--dbname=postgres",
+            "--tuples-only",
+            "--no-align",
+            "--set=ON_ERROR_STOP=1",
             (
-                "pg_restore",
-                "--no-password",
-                f"--username={_DATABASE_USER}",
-                f"--dbname={target}",
-                "--exit-on-error",
-                "--single-transaction",
-                "--no-owner",
-                "--no-privileges",
+                f"--command=WITH target_sessions AS MATERIALIZED ("  # noqa: S608
+                "SELECT pid FROM pg_stat_activity "
+                f"WHERE datname = '{target}' AND pid <> pg_backend_pid()"
+                ") SELECT count(*) FROM target_sessions "
+                "WHERE pg_terminate_backend(pid);"
             ),
-            stdin=descriptor,
-            stdout=subprocess.DEVNULL,
-            timeout=_RESTORE_TIMEOUT_SECONDS,
-            failure_code="restore_failed",
-        )
-    finally:
-        os.close(descriptor)
+        ),
+        timeout=_TOOL_TIMEOUT_SECONDS,
+        failure_code="target_cleanup_failed",
+    )
+    if not terminate.strip().isdigit():
+        raise BackupRestoreError("target_cleanup_failed")
+    sessions = _capture_database_command(
+        repository,
+        container,
+        application_name,
+        (
+            "psql",
+            "--no-password",
+            f"--username={_DATABASE_USER}",
+            "--dbname=postgres",
+            "--tuples-only",
+            "--no-align",
+            "--set=ON_ERROR_STOP=1",
+            (
+                f"--command=SELECT count(*) FROM pg_stat_activity WHERE datname = '{target}';"  # noqa: S608
+            ),
+        ),
+        timeout=_TOOL_TIMEOUT_SECONDS,
+        failure_code="target_cleanup_failed",
+    )
+    if sessions.strip() != b"0":
+        raise BackupRestoreError("target_cleanup_failed")
+    _run_database_command(
+        repository,
+        container,
+        application_name,
+        (
+            "dropdb",
+            "--no-password",
+            f"--username={_DATABASE_USER}",
+            "--if-exists",
+            "--force",
+            target,
+        ),
+        stdin=subprocess.DEVNULL,
+        stdout=subprocess.DEVNULL,
+        timeout=_TOOL_TIMEOUT_SECONDS,
+        failure_code="target_cleanup_failed",
+    )
+    if _database_exists(repository, target, container, application_name):
+        raise BackupRestoreError("target_cleanup_failed")
 
 
-def _read_inventory(repository: Path, database: str) -> Inventory:
-    output = _capture_compose(
+def _read_inventory(
+    repository: Path,
+    database: str,
+    container: _DatabaseContainer,
+    application_name: str,
+) -> Inventory:
+    output = _capture_database_command(
         repository,
+        container,
+        application_name,
         (
             "psql",
             "--no-password",
@@ -868,14 +1152,224 @@ def _canonical_sha256_text(value: object) -> str:
     return value
 
 
-def _load_backup(path: Path, *, repository: Path) -> LoadedBackup:
-    backup_directory = _operator_directory(path, repository=repository)
-    _require_private_directory(backup_directory)
-    manifest_path = backup_directory / _MANIFEST_FILENAME
-    dump_path = backup_directory / _DUMP_FILENAME
-    _require_private_regular_file(manifest_path, maximum_bytes=_MAX_MANIFEST_BYTES)
-    _require_private_regular_file(dump_path, maximum_bytes=_MAX_DUMP_BYTES)
-    manifest_bytes = _read_bounded(manifest_path, _MAX_MANIFEST_BYTES)
+def _parse_canonical_publication(value: object) -> CanonicalPublication:
+    expected_keys = {
+        "channel",
+        "current_revision_id",
+        "entry_count",
+        "last_activation_id",
+        "last_activation_operation",
+        "last_activation_sequence",
+        "publication_state",
+        "typed_fact_set_sha256",
+        "version",
+    }
+    if not isinstance(value, dict) or set(value) != expected_keys:
+        raise BackupRestoreError("publication_inspection_failed")
+    version = value["version"]
+    entry_count = value["entry_count"]
+    activation_sequence = value["last_activation_sequence"]
+    activation_operation = value["last_activation_operation"]
+    if (
+        value["channel"] != "RECENT_RETAIL"
+        or value["publication_state"] != "AVAILABLE"
+        or not isinstance(version, int)
+        or isinstance(version, bool)
+        or version < 1
+        or version > _MAX_ACTIVATIONS
+        or not isinstance(entry_count, int)
+        or isinstance(entry_count, bool)
+        or entry_count < 1
+        or entry_count > _MAX_PUBLICATION_ENTRIES
+        or not isinstance(activation_sequence, int)
+        or isinstance(activation_sequence, bool)
+        or activation_sequence != version
+        or not isinstance(activation_operation, str)
+        or activation_operation not in {"ACTIVATE", "ROLLBACK"}
+    ):
+        raise BackupRestoreError("publication_inspection_failed")
+    try:
+        current_revision_id = _canonical_uuid_text(value["current_revision_id"])
+        activation_id = _canonical_uuid_text(value["last_activation_id"])
+        fact_set_sha256 = _canonical_sha256_text(value["typed_fact_set_sha256"])
+    except BackupRestoreError:
+        raise BackupRestoreError("publication_inspection_failed") from None
+    return CanonicalPublication(
+        version=version,
+        current_revision_id=current_revision_id,
+        typed_fact_set_sha256=fact_set_sha256,
+        entry_count=entry_count,
+        last_activation_id=activation_id,
+        last_activation_operation=activation_operation,
+        last_activation_sequence=activation_sequence,
+    )
+
+
+def _require_canonical_inventory_match(
+    canonical: CanonicalPublication,
+    inventory: Inventory,
+) -> None:
+    channel = inventory.publication["channel"]
+    revision = inventory.publication["active_revision"]
+    activations = inventory.publication["activations"]
+    if (
+        not isinstance(channel, dict)
+        or not isinstance(revision, dict)
+        or not isinstance(activations, list)
+        or not activations
+    ):
+        raise BackupRestoreError("publication_contract_mismatch")
+    latest = activations[-1]
+    if not isinstance(latest, dict):
+        raise BackupRestoreError("publication_contract_mismatch")
+    if (
+        canonical.version != channel.get("version")
+        or canonical.current_revision_id != channel.get("current_revision_id")
+        or canonical.current_revision_id != revision.get("id")
+        or canonical.typed_fact_set_sha256 != revision.get("typed_fact_set_sha256")
+        or canonical.entry_count != revision.get("entry_count")
+        or canonical.last_activation_id != latest.get("id")
+        or canonical.last_activation_operation != latest.get("operation")
+        or canonical.last_activation_sequence != latest.get("sequence")
+    ):
+        raise BackupRestoreError("publication_contract_mismatch")
+
+
+def _inspect_publication(
+    repository: Path,
+    database: str,
+    container: _DatabaseContainer,
+    application_name: str,
+) -> CanonicalPublication:
+    if database != _SOURCE_DATABASE and _TARGET_DATABASE.fullmatch(database) is None:
+        raise BackupRestoreError("publication_inspection_failed")
+    _require_application_name(application_name)
+    python = repository / ".venv" / "bin" / "python"
+    manage = repository / "manage.py"
+    try:
+        python_metadata = python.stat()
+        manage_metadata = manage.lstat()
+    except OSError:
+        raise BackupRestoreError("publication_inspection_failed") from None
+    if (
+        not stat.S_ISREG(python_metadata.st_mode)
+        or stat.S_ISLNK(manage_metadata.st_mode)
+        or not stat.S_ISREG(manage_metadata.st_mode)
+    ):
+        raise BackupRestoreError("publication_inspection_failed")
+    _require_same_database_container(repository, container)
+    completed: subprocess.CompletedProcess[bytes] | None = None
+    command_failed = False
+    try:
+        completed = subprocess.run(  # noqa: S603 - exact local interpreter and command.
+            (str(python), str(manage), "inspect_recent_publication"),
+            cwd=repository,
+            env=_inspection_environment(database, application_name),
+            stdin=subprocess.DEVNULL,
+            stdout=subprocess.PIPE,
+            stderr=subprocess.DEVNULL,
+            check=False,
+            timeout=_INSPECTION_TIMEOUT_SECONDS,
+        )
+    except OSError, subprocess.SubprocessError:
+        command_failed = True
+    _require_same_database_container(repository, container)
+    if command_failed or completed is None:
+        raise BackupRestoreError("publication_inspection_failed")
+    output = completed.stdout
+    if (
+        completed.returncode != 0
+        or not isinstance(output, bytes)
+        or len(output) < 1
+        or len(output) > _MAX_MANIFEST_BYTES
+    ):
+        raise BackupRestoreError("publication_inspection_failed")
+    try:
+        decoded = json.loads(output.decode("ascii", errors="strict"))
+    except UnicodeError, json.JSONDecodeError:
+        raise BackupRestoreError("publication_inspection_failed") from None
+    return _parse_canonical_publication(decoded)
+
+
+def _inspection_environment(database: str, application_name: str) -> dict[str, str]:
+    _require_application_name(application_name)
+    return {
+        "ADMIN_ENABLED": "0",
+        "CONTROL_PLANE_OPERATIONS_ENABLED": "0",
+        "DATABASE_CONN_MAX_AGE": "0",
+        "DATABASE_URL": (
+            f"postgresql://{_DATABASE_USER}:{_LOCAL_DATABASE_PASSWORD}"
+            f"@127.0.0.1:{_LOCAL_DATABASE_PORT}/{database}"
+        ),
+        "DJANGO_DEBUG": "1",
+        "LANG": "C",
+        "LC_ALL": "C",
+        "PATH": os.defpath,
+        "PGAPPNAME": application_name,
+        "PYTHONDONTWRITEBYTECODE": "1",
+        "QA_STATE_PREVIEWS_ENABLED": "0",
+    }
+
+
+@contextmanager
+def _load_backup(
+    path: Path,
+    *,
+    repository: Path,
+    expected_manifest_sha256: str,
+) -> Iterator[LoadedBackup]:
+    expected_manifest = _validated_manifest_sha256(expected_manifest_sha256)
+    with _open_operator_directory(
+        path,
+        repository=repository,
+        require_private=True,
+    ) as backup_directory:
+        manifest_descriptor = _open_private_file_at(
+            backup_directory.descriptor,
+            _MANIFEST_FILENAME,
+            maximum_bytes=_MAX_MANIFEST_BYTES,
+        )
+        try:
+            manifest_identity = _require_private_regular_descriptor(
+                manifest_descriptor,
+                maximum_bytes=_MAX_MANIFEST_BYTES,
+            )
+            manifest_bytes = _read_bounded_descriptor(
+                manifest_descriptor,
+                _MAX_MANIFEST_BYTES,
+            )
+            _require_descriptor_identity(manifest_descriptor, manifest_identity)
+            actual_manifest = hashlib.sha256(manifest_bytes).hexdigest()
+            if not hmac.compare_digest(actual_manifest, expected_manifest):
+                raise BackupRestoreError("manifest_receipt_mismatch")
+        finally:
+            os.close(manifest_descriptor)
+        dump_descriptor = _open_private_file_at(
+            backup_directory.descriptor,
+            _DUMP_FILENAME,
+            maximum_bytes=_MAX_DUMP_BYTES,
+        )
+        try:
+            dump_identity = _require_private_regular_descriptor(
+                dump_descriptor,
+                maximum_bytes=_MAX_DUMP_BYTES,
+            )
+            selected = _parse_loaded_backup(
+                manifest_bytes=manifest_bytes,
+                dump_descriptor=dump_descriptor,
+                dump_identity=dump_identity,
+            )
+            yield selected
+        finally:
+            os.close(dump_descriptor)
+
+
+def _parse_loaded_backup(
+    *,
+    manifest_bytes: bytes,
+    dump_descriptor: int,
+    dump_identity: _FileIdentity,
+) -> LoadedBackup:
     try:
         manifest = json.loads(manifest_bytes.decode("utf-8", errors="strict"))
     except UnicodeError, json.JSONDecodeError:
@@ -918,13 +1412,16 @@ def _load_backup(path: Path, *, repository: Path) -> LoadedBackup:
         or _SHA256.fullmatch(dump["sha256"]) is None
     ):
         raise BackupRestoreError("backup_manifest_invalid")
-    dump_sha256, dump_bytes = _file_sha256(dump_path)
+    dump_sha256, dump_bytes = _file_sha256_descriptor(dump_descriptor)
+    _require_descriptor_identity(dump_descriptor, dump_identity)
     if dump_sha256 != dump["sha256"] or dump_bytes != dump["bytes"]:
         raise BackupRestoreError("checksum_mismatch")
-    if _read_prefix(dump_path, len(_DUMP_MAGIC)) != _DUMP_MAGIC:
+    if _read_prefix_descriptor(dump_descriptor, len(_DUMP_MAGIC)) != _DUMP_MAGIC:
         raise BackupRestoreError("backup_not_custom_format")
 
     if not isinstance(inventory_value, dict) or set(inventory_value) != {
+        "canonical_publication",
+        "canonical_publication_sha256",
         "migrations",
         "publication",
         "publication_sha256",
@@ -953,18 +1450,37 @@ def _load_backup(path: Path, *, repository: Path) -> LoadedBackup:
         or inventory.sha256 != inventory_sha
     ):
         raise BackupRestoreError("checksum_mismatch")
-    return LoadedBackup(backup_id=backup_id, dump_path=dump_path, inventory=inventory)
+    canonical = _parse_canonical_publication(inventory_value["canonical_publication"])
+    canonical_sha = inventory_value["canonical_publication_sha256"]
+    if (
+        not isinstance(canonical_sha, str)
+        or _SHA256.fullmatch(canonical_sha) is None
+        or canonical.sha256 != canonical_sha
+    ):
+        raise BackupRestoreError("publication_contract_mismatch")
+    _require_canonical_inventory_match(canonical, inventory)
+    return LoadedBackup(
+        backup_id=backup_id,
+        dump_descriptor=dump_descriptor,
+        dump_identity=dump_identity,
+        inventory=inventory,
+        canonical_publication=canonical,
+    )
 
 
-def _capture_compose(
+def _capture_database_command(
     repository: Path,
+    container: _DatabaseContainer,
+    application_name: str,
     tool_arguments: tuple[str, ...],
     *,
     timeout: int,
     failure_code: str,
 ) -> bytes:
-    completed = _run_compose(
+    completed = _run_database_command(
         repository,
+        container,
+        application_name,
         tool_arguments,
         stdin=subprocess.DEVNULL,
         stdout=subprocess.PIPE,
@@ -977,8 +1493,148 @@ def _capture_compose(
     return output
 
 
-def _run_compose(
+def _resolve_docker_binary() -> str:
+    selected = shutil.which("docker")
+    if selected is None:
+        raise BackupRestoreError("docker_unavailable")
+    try:
+        resolved = Path(selected).resolve(strict=True)
+        metadata = resolved.stat()
+    except OSError:
+        raise BackupRestoreError("docker_unavailable") from None
+    if (
+        not resolved.is_absolute()
+        or not stat.S_ISREG(metadata.st_mode)
+        or metadata.st_uid not in {0, os.geteuid()}
+        or stat.S_IMODE(metadata.st_mode) & 0o022
+        or not os.access(resolved, os.X_OK)
+    ):
+        raise BackupRestoreError("docker_unavailable")
+    return str(resolved)
+
+
+def _require_local_docker_socket() -> None:
+    socket_path = Path(_LOCAL_DOCKER_HOST.removeprefix("unix://"))
+    try:
+        metadata = socket_path.stat()
+    except OSError:
+        raise BackupRestoreError("database_container_invalid") from None
+    if not stat.S_ISSOCK(metadata.st_mode) or metadata.st_uid not in {0, os.geteuid()}:
+        raise BackupRestoreError("database_container_invalid")
+
+
+def _database_container_ids(
     repository: Path,
+    docker_binary: str,
+) -> tuple[str, ...]:
+    completed = _run_docker_cli(
+        repository,
+        docker_binary,
+        (
+            "container",
+            "ls",
+            "--no-trunc",
+            f"--filter=label=com.docker.compose.project={_COMPOSE_PROJECT}",
+            f"--filter=label=com.docker.compose.service={_COMPOSE_SERVICE}",
+            "--format={{.ID}}",
+        ),
+        stdin=subprocess.DEVNULL,
+        stdout=subprocess.PIPE,
+        timeout=_TOOL_TIMEOUT_SECONDS,
+        failure_code="database_container_invalid",
+    )
+    output = completed.stdout
+    if not isinstance(output, bytes) or len(output) > 512:
+        raise BackupRestoreError("database_container_invalid")
+    try:
+        identifiers = tuple(
+            line for line in output.decode("ascii", errors="strict").splitlines() if line
+        )
+    except UnicodeError:
+        raise BackupRestoreError("database_container_invalid") from None
+    if len(identifiers) != 1 or _CONTAINER_ID.fullmatch(identifiers[0]) is None:
+        raise BackupRestoreError("database_container_invalid")
+    return identifiers
+
+
+def _discover_database_container(repository: Path, docker_binary: str) -> str:
+    _require_local_docker_socket()
+    before = _database_container_ids(repository, docker_binary)
+    container_id = before[0]
+    completed = _run_docker_cli(
+        repository,
+        docker_binary,
+        ("container", "inspect", container_id),
+        stdin=subprocess.DEVNULL,
+        stdout=subprocess.PIPE,
+        timeout=_TOOL_TIMEOUT_SECONDS,
+        failure_code="database_container_invalid",
+    )
+    output = completed.stdout
+    if not isinstance(output, bytes) or len(output) < 1 or len(output) > _MAX_DOCKER_INSPECT_BYTES:
+        raise BackupRestoreError("database_container_invalid")
+    try:
+        decoded = json.loads(output.decode("utf-8", errors="strict"))
+    except UnicodeError, json.JSONDecodeError:
+        raise BackupRestoreError("database_container_invalid") from None
+    _validate_database_container_inspection(repository, container_id, decoded)
+    after = _database_container_ids(repository, docker_binary)
+    if after != before:
+        raise BackupRestoreError("database_container_invalid")
+    return container_id
+
+
+def _validate_database_container_inspection(
+    repository: Path,
+    container_id: str,
+    decoded: object,
+) -> None:
+    if not isinstance(decoded, list) or len(decoded) != 1 or not isinstance(decoded[0], dict):
+        raise BackupRestoreError("database_container_invalid")
+    inspection = decoded[0]
+    config = inspection.get("Config")
+    state = inspection.get("State")
+    network = inspection.get("NetworkSettings")
+    if not isinstance(config, dict) or not isinstance(state, dict) or not isinstance(network, dict):
+        raise BackupRestoreError("database_container_invalid")
+    labels = config.get("Labels")
+    health = state.get("Health")
+    ports = network.get("Ports")
+    expected_labels = {
+        "com.docker.compose.project": _COMPOSE_PROJECT,
+        "com.docker.compose.project.config_files": str(repository / "compose.yaml"),
+        "com.docker.compose.project.working_dir": str(repository),
+        "com.docker.compose.service": _COMPOSE_SERVICE,
+    }
+    expected_port = [{"HostIp": "127.0.0.1", "HostPort": str(_LOCAL_DATABASE_PORT)}]
+    if (
+        inspection.get("Id") != container_id
+        or not isinstance(labels, dict)
+        or any(labels.get(key) != value for key, value in expected_labels.items())
+        or state.get("Status") != "running"
+        or state.get("Running") is not True
+        or not isinstance(health, dict)
+        or health.get("Status") != "healthy"
+        or not isinstance(ports, dict)
+        or {key: value for key, value in ports.items() if value is not None}
+        != {"5432/tcp": expected_port}
+    ):
+        raise BackupRestoreError("database_container_invalid")
+
+
+def _require_same_database_container(
+    repository: Path,
+    container: _DatabaseContainer,
+) -> None:
+    discovered = _discover_database_container(repository, container.docker_binary)
+    if not hmac.compare_digest(discovered, container.container_id):
+        raise BackupRestoreError("database_container_invalid")
+
+
+def _run_database_command(
+    repository: Path,
+    container: _DatabaseContainer,
+    application_name: str,
     tool_arguments: tuple[str, ...],
     *,
     stdin: int,
@@ -986,19 +1642,51 @@ def _run_compose(
     timeout: int,
     failure_code: str,
 ) -> subprocess.CompletedProcess[bytes]:
-    docker = shutil.which("docker")
-    if docker is None:
+    _require_application_name(application_name)
+    _require_same_database_container(repository, container)
+    completed: subprocess.CompletedProcess[bytes] | None = None
+    command_error: BackupRestoreError | None = None
+    try:
+        completed = _run_docker_cli(
+            repository,
+            container.docker_binary,
+            (
+                "exec",
+                "-i",
+                f"--env=PGAPPNAME={application_name}",
+                container.container_id,
+                *tool_arguments,
+            ),
+            stdin=stdin,
+            stdout=stdout,
+            timeout=timeout,
+            failure_code=failure_code,
+        )
+    except BackupRestoreError as error:
+        command_error = error
+    _require_same_database_container(repository, container)
+    if command_error is not None:
+        raise command_error
+    if completed is None:
+        raise BackupRestoreError(failure_code)
+    return completed
+
+
+def _run_docker_cli(
+    repository: Path,
+    docker_binary: str,
+    arguments: tuple[str, ...],
+    *,
+    stdin: int,
+    stdout: int,
+    timeout: int,
+    failure_code: str,
+) -> subprocess.CompletedProcess[bytes]:
+    if not Path(docker_binary).is_absolute():
         raise BackupRestoreError("docker_unavailable")
-    command = (
-        docker,
-        "compose",
-        "exec",
-        "-T",
-        _COMPOSE_SERVICE,
-        *tool_arguments,
-    )
+    command = (docker_binary, f"--host={_LOCAL_DOCKER_HOST}", *arguments)
     try:
-        completed = subprocess.run(  # noqa: S603 - fixed docker + validated tool arguments.
+        completed = subprocess.run(  # noqa: S603 - resolved docker and fixed local socket.
             command,
             cwd=repository,
             env=_docker_environment(),
@@ -1015,11 +1703,18 @@ def _run_compose(
     return completed
 
 
+def _require_application_name(value: str) -> None:
+    if _APPLICATION_NAME.fullmatch(value) is None:
+        raise BackupRestoreError("internal_error")
+
+
 def _docker_environment() -> dict[str, str]:
-    allowed = ("DOCKER_CONTEXT", "DOCKER_HOST", "HOME", "PATH")
-    environment = {key: os.environ[key] for key in allowed if key in os.environ}
-    environment.update({"LANG": "C", "LC_ALL": "C"})
-    return environment
+    return {
+        "DOCKER_CONFIG": "/var/empty",
+        "HOME": "/var/empty",
+        "LANG": "C",
+        "LC_ALL": "C",
+    }
 
 
 @contextmanager
@@ -1031,16 +1726,105 @@ def _restrictive_umask() -> Iterator[None]:
         os.umask(previous)
 
 
-def _create_private_file(path: Path) -> int:
-    flags = os.O_WRONLY | os.O_CREAT | os.O_EXCL | getattr(os, "O_CLOEXEC", 0)
+@contextmanager
+def _open_operator_directory(
+    path: Path,
+    *,
+    repository: Path,
+    require_private: bool = False,
+) -> Iterator[_OpenedDirectory]:
+    if not path.is_absolute():
+        raise BackupRestoreError("backup_directory_invalid")
+    try:
+        before = path.lstat()
+        resolved = path.resolve(strict=True)
+        resolved_metadata = resolved.lstat()
+        parent_metadata = resolved.parent.lstat()
+    except OSError:
+        raise BackupRestoreError("backup_directory_unavailable") from None
+    if (
+        stat.S_ISLNK(before.st_mode)
+        or not stat.S_ISDIR(before.st_mode)
+        or not stat.S_ISDIR(resolved_metadata.st_mode)
+        or (before.st_dev, before.st_ino) != (resolved_metadata.st_dev, resolved_metadata.st_ino)
+    ):
+        raise BackupRestoreError("backup_directory_invalid")
+    _require_trusted_directory_metadata(resolved_metadata, require_private=require_private)
+    _require_trusted_parent_metadata(parent_metadata)
+    if _is_within(resolved, repository):
+        raise BackupRestoreError("destination_inside_repository")
+
+    flags = os.O_RDONLY | os.O_DIRECTORY | os.O_CLOEXEC | os.O_NOFOLLOW
+    try:
+        descriptor = os.open(resolved, flags)
+    except OSError:
+        raise BackupRestoreError("backup_directory_unavailable") from None
+    try:
+        opened = os.fstat(descriptor)
+        if (opened.st_dev, opened.st_ino) != (resolved_metadata.st_dev, resolved_metadata.st_ino):
+            raise BackupRestoreError("backup_directory_invalid")
+        _require_trusted_directory_metadata(opened, require_private=require_private)
+        yield _OpenedDirectory(path=resolved, descriptor=descriptor)
+    finally:
+        os.close(descriptor)
+
+
+def _require_trusted_directory_metadata(
+    metadata: os.stat_result,
+    *,
+    require_private: bool,
+) -> None:
+    mode = stat.S_IMODE(metadata.st_mode)
+    if (
+        not stat.S_ISDIR(metadata.st_mode)
+        or metadata.st_uid != os.geteuid()
+        or mode & 0o022
+        or (require_private and mode & 0o077)
+    ):
+        raise BackupRestoreError("backup_directory_permissions")
+
+
+def _require_trusted_parent_metadata(metadata: os.stat_result) -> None:
+    mode = stat.S_IMODE(metadata.st_mode)
+    owner_is_trusted = metadata.st_uid in {0, os.geteuid()}
+    not_broadly_writable = mode & 0o022 == 0
+    sticky_directory = stat.S_ISDIR(metadata.st_mode) and bool(mode & stat.S_ISVTX)
+    if (
+        not stat.S_ISDIR(metadata.st_mode)
+        or not owner_is_trusted
+        or not (not_broadly_writable or sticky_directory)
+    ):
+        raise BackupRestoreError("backup_directory_permissions")
+
+
+def _create_private_directory_at(parent_descriptor: int, name: str) -> int:
     try:
-        return os.open(path, flags, 0o600)
+        os.mkdir(name, mode=0o700, dir_fd=parent_descriptor)
+        descriptor = os.open(
+            name,
+            os.O_RDONLY | os.O_DIRECTORY | os.O_CLOEXEC | os.O_NOFOLLOW,
+            dir_fd=parent_descriptor,
+        )
+    except OSError:
+        raise BackupRestoreError("backup_directory_unavailable") from None
+    try:
+        _require_trusted_directory_metadata(os.fstat(descriptor), require_private=True)
+    except Exception:
+        os.close(descriptor)
+        raise
+    return descriptor
+
+
+def _create_private_file_at(directory_descriptor: int, name: str) -> int:
+    flags = os.O_RDWR | os.O_CREAT | os.O_EXCL | os.O_CLOEXEC | os.O_NOFOLLOW
+    try:
+        return os.open(name, flags, 0o600, dir_fd=directory_descriptor)
     except OSError:
         raise BackupRestoreError("dump_file_invalid") from None
 
 
-def _write_private_file(path: Path, content: bytes) -> None:
-    descriptor = _create_private_file(path)
+def _write_private_file_at(directory_descriptor: int, name: str, content: bytes) -> None:
+    descriptor = _create_private_file_at(directory_descriptor, name)
     try:
         written = 0
         while written < len(content):
@@ -1055,70 +1839,106 @@ def _write_private_file(path: Path, content: bytes) -> None:
         os.close(descriptor)
 
 
-def _require_private_directory(path: Path) -> None:
+def _open_private_file_at(
+    directory_descriptor: int,
+    name: str,
+    *,
+    maximum_bytes: int,
+) -> int:
     try:
-        metadata = path.lstat()
+        descriptor = os.open(
+            name,
+            os.O_RDONLY | os.O_CLOEXEC | os.O_NOFOLLOW,
+            dir_fd=directory_descriptor,
+        )
     except OSError:
-        raise BackupRestoreError("backup_directory_invalid") from None
-    if (
-        stat.S_ISLNK(metadata.st_mode)
-        or not stat.S_ISDIR(metadata.st_mode)
-        or metadata.st_uid != os.geteuid()
-        or stat.S_IMODE(metadata.st_mode) & 0o077
-    ):
-        raise BackupRestoreError("backup_directory_permissions")
+        raise BackupRestoreError("dump_file_invalid") from None
+    try:
+        _require_private_regular_descriptor(descriptor, maximum_bytes=maximum_bytes)
+    except Exception:
+        os.close(descriptor)
+        raise
+    return descriptor
 
 
-def _require_private_regular_file(path: Path, *, maximum_bytes: int) -> None:
+def _require_private_regular_descriptor(
+    descriptor: int,
+    *,
+    maximum_bytes: int,
+) -> _FileIdentity:
     try:
-        metadata = path.lstat()
+        metadata = os.fstat(descriptor)
     except OSError:
         raise BackupRestoreError("dump_file_invalid") from None
     if (
-        stat.S_ISLNK(metadata.st_mode)
-        or not stat.S_ISREG(metadata.st_mode)
+        not stat.S_ISREG(metadata.st_mode)
         or metadata.st_uid != os.geteuid()
         or stat.S_IMODE(metadata.st_mode) & 0o077
     ):
         raise BackupRestoreError("backup_directory_permissions")
     if metadata.st_size < 1 or metadata.st_size > maximum_bytes:
         raise BackupRestoreError("backup_too_large")
+    return _file_identity(metadata)
 
 
-def _read_bounded(path: Path, maximum_bytes: int) -> bytes:
-    descriptor = os.open(
-        path,
-        os.O_RDONLY | getattr(os, "O_CLOEXEC", 0) | getattr(os, "O_NOFOLLOW", 0),
+def _file_identity(metadata: os.stat_result) -> _FileIdentity:
+    return _FileIdentity(
+        device=metadata.st_dev,
+        inode=metadata.st_ino,
+        size=metadata.st_size,
+        modified_ns=metadata.st_mtime_ns,
+        changed_ns=metadata.st_ctime_ns,
     )
+
+
+def _require_descriptor_identity(descriptor: int, expected: _FileIdentity) -> None:
     try:
-        data = os.read(descriptor, maximum_bytes + 1)
+        actual = _file_identity(os.fstat(descriptor))
     except OSError:
         raise BackupRestoreError("dump_file_invalid") from None
-    finally:
-        os.close(descriptor)
+    if actual != expected:
+        raise BackupRestoreError("checksum_mismatch")
+
+
+def _read_bounded_descriptor(descriptor: int, maximum_bytes: int) -> bytes:
+    _rewind_descriptor(descriptor)
+    chunks: list[bytes] = []
+    total = 0
+    try:
+        while total <= maximum_bytes:
+            chunk = os.read(descriptor, min(1024 * 1024, maximum_bytes + 1 - total))
+            if not chunk:
+                break
+            chunks.append(chunk)
+            total += len(chunk)
+    except OSError:
+        raise BackupRestoreError("dump_file_invalid") from None
+    data = b"".join(chunks)
     if len(data) > maximum_bytes:
         raise BackupRestoreError("backup_too_large")
     return data
 
 
-def _read_prefix(path: Path, length: int) -> bytes:
-    descriptor = os.open(
-        path,
-        os.O_RDONLY | getattr(os, "O_CLOEXEC", 0) | getattr(os, "O_NOFOLLOW", 0),
-    )
+def _read_prefix_descriptor(descriptor: int, length: int) -> bytes:
+    _rewind_descriptor(descriptor)
+    chunks: list[bytes] = []
+    remaining = length
     try:
-        return os.read(descriptor, length)
-    finally:
-        os.close(descriptor)
+        while remaining > 0:
+            chunk = os.read(descriptor, remaining)
+            if not chunk:
+                break
+            chunks.append(chunk)
+            remaining -= len(chunk)
+    except OSError:
+        raise BackupRestoreError("dump_file_invalid") from None
+    return b"".join(chunks)
 
 
-def _file_sha256(path: Path) -> tuple[str, int]:
+def _file_sha256_descriptor(descriptor: int) -> tuple[str, int]:
+    _rewind_descriptor(descriptor)
     digest = hashlib.sha256()
     total = 0
-    descriptor = os.open(
-        path,
-        os.O_RDONLY | getattr(os, "O_CLOEXEC", 0) | getattr(os, "O_NOFOLLOW", 0),
-    )
     try:
         while True:
             chunk = os.read(descriptor, 1024 * 1024)
@@ -1130,19 +1950,21 @@ def _file_sha256(path: Path) -> tuple[str, int]:
             digest.update(chunk)
     except OSError:
         raise BackupRestoreError("dump_file_invalid") from None
-    finally:
-        os.close(descriptor)
     return digest.hexdigest(), total
 
 
-def _fsync_directory(path: Path) -> None:
-    descriptor = os.open(path, os.O_RDONLY | getattr(os, "O_CLOEXEC", 0))
+def _rewind_descriptor(descriptor: int) -> None:
+    try:
+        os.lseek(descriptor, 0, os.SEEK_SET)
+    except OSError:
+        raise BackupRestoreError("dump_file_invalid") from None
+
+
+def _fsync_directory_descriptor(descriptor: int) -> None:
     try:
         os.fsync(descriptor)
     except OSError:
         raise BackupRestoreError("backup_directory_unavailable") from None
-    finally:
-        os.close(descriptor)
 
 
 def _canonical_json(value: object) -> bytes:
diff --git a/scripts/tests/test_postgres_backup_restore.py b/scripts/tests/test_postgres_backup_restore.py
index 3afece4..b40f215 100644
--- a/scripts/tests/test_postgres_backup_restore.py
+++ b/scripts/tests/test_postgres_backup_restore.py
@@ -2,9 +2,11 @@
 
 from __future__ import annotations
 
+import hashlib
 import json
 import os
 import stat
+import subprocess
 from copy import deepcopy
 from pathlib import Path
 from unittest.mock import call, patch
@@ -15,11 +17,21 @@ from scripts.postgres_backup_restore import (
     _INVENTORY_SQL,
     BackupReceipt,
     BackupRestoreError,
+    CanonicalPublication,
     Inventory,
+    _cleanup_target_database,
     _database_exists,
+    _DatabaseContainer,
+    _discover_database_container,
     _docker_environment,
+    _inspect_publication,
+    _new_application_name,
     _parse_inventory,
     _preflight,
+    _restore_database,
+    _run_database_command,
+    _run_docker_cli,
+    _validate_database_container_inspection,
     _validated_target_database,
     create_backup,
     main,
@@ -34,6 +46,31 @@ _ACTIVATION_V1 = "55555555-5555-4555-8555-555555555555"
 _ACTIVATION_V2 = "66666666-6666-4666-8666-666666666666"
 _FACT_SET_HASH = "a" * 64
 _EVIDENCE_HASH = "8" * 64
+_CONTAINER_ID = "c" * 64
+_DOCKER = "/safe/docker"
+_APPLICATION_NAME = "grocery_restore_" + ("d" * 32)
+_BACKUP_APPLICATION_NAME = "grocery_backup_" + ("e" * 32)
+_CONTAINER = _DatabaseContainer(docker_binary=_DOCKER, container_id=_CONTAINER_ID)
+
+
+def _container_inspection(repository: Path) -> list[dict[str, object]]:
+    return [
+        {
+            "Config": {
+                "Labels": {
+                    "com.docker.compose.project": "audience-foundry-grocery-seasonality",
+                    "com.docker.compose.project.config_files": str(repository / "compose.yaml"),
+                    "com.docker.compose.project.working_dir": str(repository),
+                    "com.docker.compose.service": "db",
+                }
+            },
+            "Id": _CONTAINER_ID,
+            "NetworkSettings": {
+                "Ports": {"5432/tcp": [{"HostIp": "127.0.0.1", "HostPort": "55434"}]}
+            },
+            "State": {"Health": {"Status": "healthy"}, "Running": True, "Status": "running"},
+        }
+    ]
 
 
 def _publication_contract() -> dict[str, object]:
@@ -75,6 +112,18 @@ def _publication_contract() -> dict[str, object]:
     }
 
 
+def _canonical_publication() -> CanonicalPublication:
+    return CanonicalPublication(
+        version=2,
+        current_revision_id=_REVISION_V2,
+        typed_fact_set_sha256=_FACT_SET_HASH,
+        entry_count=10,
+        last_activation_id=_ACTIVATION_V2,
+        last_activation_operation="ACTIVATE",
+        last_activation_sequence=2,
+    )
+
+
 @pytest.fixture
 def repository(tmp_path: Path) -> Path:
     root = tmp_path / "repository"
@@ -106,7 +155,12 @@ def inventory() -> Inventory:
     )
 
 
-def _synthetic_dump(_repository: Path, descriptor: int) -> None:
+def _synthetic_dump(
+    _repository: Path,
+    descriptor: int,
+    _container: _DatabaseContainer,
+    _application_name: str,
+) -> None:
     os.write(descriptor, b"PGDMPsynthetic-custom-format")
 
 
@@ -116,7 +170,11 @@ def _make_backup(
     inventory: Inventory,
 ) -> tuple[Path, BackupReceipt]:
     with (
-        patch("scripts.postgres_backup_restore._preflight"),
+        patch("scripts.postgres_backup_restore._preflight", return_value=_CONTAINER),
+        patch(
+            "scripts.postgres_backup_restore._inspect_publication",
+            return_value=_canonical_publication(),
+        ),
         patch(
             "scripts.postgres_backup_restore._read_inventory",
             side_effect=(inventory, inventory),
@@ -131,6 +189,10 @@ def _make_backup(
     return backup_directory, receipt
 
 
+def _manifest_receipt(backup_directory: Path) -> str:
+    return hashlib.sha256((backup_directory / "manifest.json").read_bytes()).hexdigest()
+
+
 def test_backup_creates_private_custom_dump_and_secret_free_checksum_manifest(
     repository: Path,
     destination: Path,
@@ -145,12 +207,18 @@ def test_backup_creates_private_custom_dump_and_secret_free_checksum_manifest(
     assert stat.S_IMODE(dump.stat().st_mode) == 0o600
     assert stat.S_IMODE(manifest_path.stat().st_mode) == 0o600
     assert dump.read_bytes().startswith(b"PGDMP")
-    assert manifest["format_version"] == "grocery-postgres-custom-v1"
+    assert manifest["format_version"] == "grocery-postgres-custom-v2"
     assert manifest["postgres_major"] == 18
     assert manifest["source_database"] == "grocery"
     assert manifest["inventory"]["sha256"] == inventory.sha256
     assert manifest["inventory"]["publication"] == inventory.publication
     assert manifest["inventory"]["publication_sha256"] == inventory.publication_sha256
+    assert manifest["inventory"]["canonical_publication"] == (
+        _canonical_publication().canonical_data()
+    )
+    assert manifest["inventory"]["canonical_publication_sha256"] == (
+        _canonical_publication().sha256
+    )
     assert manifest["dump"]["sha256"] == receipt.dump_sha256
     serialized = manifest_path.read_text(encoding="utf-8")
     assert "DATABASE_URL" not in serialized
@@ -177,7 +245,15 @@ def test_backup_reads_inventory_before_and_after_dump(
     inventory: Inventory,
 ) -> None:
     with (
-        patch("scripts.postgres_backup_restore._preflight"),
+        patch(
+            "scripts.postgres_backup_restore._new_application_name",
+            return_value=_BACKUP_APPLICATION_NAME,
+        ),
+        patch("scripts.postgres_backup_restore._preflight", return_value=_CONTAINER),
+        patch(
+            "scripts.postgres_backup_restore._inspect_publication",
+            return_value=_canonical_publication(),
+        ) as inspect_publication,
         patch(
             "scripts.postgres_backup_restore._read_inventory",
             side_effect=(inventory, inventory),
@@ -190,8 +266,12 @@ def test_backup_reads_inventory_before_and_after_dump(
         create_backup(repository=repository, destination_root=destination)
 
     assert read_inventory.call_args_list == [
-        call(repository.resolve(), "grocery"),
-        call(repository.resolve(), "grocery"),
+        call(repository.resolve(), "grocery", _CONTAINER, _BACKUP_APPLICATION_NAME),
+        call(repository.resolve(), "grocery", _CONTAINER, _BACKUP_APPLICATION_NAME),
+    ]
+    assert inspect_publication.call_args_list == [
+        call(repository.resolve(), "grocery", _CONTAINER, _BACKUP_APPLICATION_NAME),
+        call(repository.resolve(), "grocery", _CONTAINER, _BACKUP_APPLICATION_NAME),
     ]
 
 
@@ -207,6 +287,10 @@ def test_backup_fails_if_inventory_changes_during_dump(
     )
     with (
         patch("scripts.postgres_backup_restore._preflight"),
+        patch(
+            "scripts.postgres_backup_restore._inspect_publication",
+            return_value=_canonical_publication(),
+        ),
         patch(
             "scripts.postgres_backup_restore._read_inventory",
             side_effect=(inventory, changed),
@@ -236,6 +320,56 @@ def test_backup_rejects_relative_or_repository_destination(
     assert inside.value.code == "destination_inside_repository"
 
 
+@pytest.mark.parametrize("unsafe_mode", (0o770, 0o707))
+def test_backup_rejects_group_or_world_writable_output_root(
+    repository: Path,
+    destination: Path,
+    unsafe_mode: int,
+) -> None:
+    os.chmod(destination, unsafe_mode)
+
+    with (
+        patch("scripts.postgres_backup_restore._preflight") as preflight,
+        pytest.raises(BackupRestoreError) as caught,
+    ):
+        create_backup(repository=repository, destination_root=destination)
+
+    assert caught.value.code == "backup_directory_permissions"
+    preflight.assert_not_called()
+
+
+def test_backup_rejects_untrusted_immediate_parent(
+    repository: Path,
+    tmp_path: Path,
+) -> None:
+    unsafe_parent = tmp_path / "unsafe-parent"
+    unsafe_parent.mkdir(mode=0o700)
+    destination = unsafe_parent / "backups"
+    destination.mkdir(mode=0o700)
+    os.chmod(unsafe_parent, 0o777)  # noqa: S103 - intentional unsafe-boundary fixture.
+
+    with pytest.raises(BackupRestoreError) as caught:
+        create_backup(repository=repository, destination_root=destination)
+
+    assert caught.value.code == "backup_directory_permissions"
+
+
+def test_backup_accepts_root_or_operator_owned_sticky_immediate_parent(
+    repository: Path,
+    tmp_path: Path,
+    inventory: Inventory,
+) -> None:
+    sticky_parent = tmp_path / "sticky-parent"
+    sticky_parent.mkdir(mode=0o700)
+    destination = sticky_parent / "backups"
+    destination.mkdir(mode=0o700)
+    os.chmod(sticky_parent, 0o1777)  # noqa: S103 - intentional sticky-directory fixture.
+
+    _backup_directory, receipt = _make_backup(repository, destination, inventory)
+
+    assert receipt.table_count == len(inventory.rows)
+
+
 def test_restore_validates_then_creates_separate_target_and_compares_inventory(
     repository: Path,
     destination: Path,
@@ -243,32 +377,61 @@ def test_restore_validates_then_creates_separate_target_and_compares_inventory(
 ) -> None:
     backup_directory, backup_receipt = _make_backup(repository, destination, inventory)
     with (
-        patch("scripts.postgres_backup_restore._preflight") as preflight,
+        patch(
+            "scripts.postgres_backup_restore._new_application_name",
+            return_value=_APPLICATION_NAME,
+        ),
+        patch(
+            "scripts.postgres_backup_restore._preflight",
+            return_value=_CONTAINER,
+        ) as preflight,
         patch("scripts.postgres_backup_restore._validate_custom_dump") as validate_dump,
         patch("scripts.postgres_backup_restore._database_exists", return_value=False) as exists,
         patch("scripts.postgres_backup_restore._create_target_database") as create_target,
         patch("scripts.postgres_backup_restore._restore_database") as restore,
+        patch("scripts.postgres_backup_restore._cleanup_target_database") as cleanup,
         patch("scripts.postgres_backup_restore._read_inventory", return_value=inventory),
+        patch(
+            "scripts.postgres_backup_restore._inspect_publication",
+            return_value=_canonical_publication(),
+        ),
     ):
         receipt = restore_backup(
             repository=repository,
             backup_directory=backup_directory,
             target_database="grocery_restore_rehearsal1",
+            expected_manifest_sha256=backup_receipt.manifest_sha256,
         )
 
-    preflight.assert_called_once_with(repository.resolve())
-    validate_dump.assert_called_once_with(repository.resolve(), backup_directory / "database.dump")
-    exists.assert_called_once_with(repository.resolve(), "grocery_restore_rehearsal1")
-    create_target.assert_called_once_with(repository.resolve(), "grocery_restore_rehearsal1")
+    preflight.assert_called_once_with(repository.resolve(), _APPLICATION_NAME)
+    assert validate_dump.call_count == 1
+    validated_descriptor = validate_dump.call_args.args[1]
+    assert isinstance(validated_descriptor, int)
+    exists.assert_called_once_with(
+        repository.resolve(),
+        "grocery_restore_rehearsal1",
+        _CONTAINER,
+        _APPLICATION_NAME,
+    )
+    create_target.assert_called_once_with(
+        repository.resolve(),
+        "grocery_restore_rehearsal1",
+        _CONTAINER,
+        _APPLICATION_NAME,
+    )
     restore.assert_called_once_with(
         repository.resolve(),
-        backup_directory / "database.dump",
+        validated_descriptor,
         "grocery_restore_rehearsal1",
+        _CONTAINER,
+        _APPLICATION_NAME,
     )
     assert receipt.backup_id == backup_receipt.backup_id
+    cleanup.assert_not_called()
     assert "row_counts_consistent=yes" in receipt.render()
     assert "migrations_consistent=yes" in receipt.render()
-    assert "publication_contract_consistent=yes" in receipt.render()
+    assert "publication_metadata_consistent=yes" in receipt.render()
+    assert "publication_canonical_consistent=yes" in receipt.render()
     assert "grocery_restore_rehearsal1" not in receipt.render()
     for internal_value in (
         _REVISION_V2,
@@ -287,12 +450,13 @@ def test_restore_refuses_source_invalid_or_existing_target_before_create(
     destination: Path,
     inventory: Inventory,
 ) -> None:
-    backup_directory, _receipt = _make_backup(repository, destination, inventory)
+    backup_directory, backup_receipt = _make_backup(repository, destination, inventory)
     with pytest.raises(BackupRestoreError) as source:
         restore_backup(
             repository=repository,
             backup_directory=backup_directory,
             target_database="grocery",
+            expected_manifest_sha256=backup_receipt.manifest_sha256,
         )
     assert source.value.code == "target_database_is_source"
 
@@ -301,6 +465,7 @@ def test_restore_refuses_source_invalid_or_existing_target_before_create(
             repository=repository,
             backup_directory=backup_directory,
             target_database="production",
+            expected_manifest_sha256=backup_receipt.manifest_sha256,
         )
     assert invalid.value.code == "target_database_invalid"
 
@@ -315,6 +480,7 @@ def test_restore_refuses_source_invalid_or_existing_target_before_create(
             repository=repository,
             backup_directory=backup_directory,
             target_database="grocery_restore_existing",
+            expected_manifest_sha256=backup_receipt.manifest_sha256,
         )
     assert existing.value.code == "target_database_exists"
     create_target.assert_not_called()
@@ -352,23 +518,30 @@ def test_restore_fails_closed_on_row_or_migration_drift(
     restored: Inventory,
     expected_code: str,
 ) -> None:
-    backup_directory, _receipt = _make_backup(repository, destination, inventory)
+    backup_directory, backup_receipt = _make_backup(repository, destination, inventory)
     with (
         patch("scripts.postgres_backup_restore._preflight"),
         patch("scripts.postgres_backup_restore._validate_custom_dump"),
         patch("scripts.postgres_backup_restore._database_exists", return_value=False),
         patch("scripts.postgres_backup_restore._create_target_database"),
         patch("scripts.postgres_backup_restore._restore_database"),
+        patch("scripts.postgres_backup_restore._cleanup_target_database") as cleanup,
         patch("scripts.postgres_backup_restore._read_inventory", return_value=restored),
+        patch(
+            "scripts.postgres_backup_restore._inspect_publication",
+            return_value=_canonical_publication(),
+        ),
         pytest.raises(BackupRestoreError) as caught,
     ):
         restore_backup(
             repository=repository,
             backup_directory=backup_directory,
             target_database="grocery_restore_drift",
+            expected_manifest_sha256=backup_receipt.manifest_sha256,
         )
 
     assert caught.value.code == expected_code
+    cleanup.assert_called_once()
 
 
 def test_restore_fails_closed_on_publication_contract_drift(
@@ -385,23 +558,114 @@ def test_restore_fails_closed_on_publication_contract_drift(
         migrations=inventory.migrations,
         publication=changed_contract,
     )
-    backup_directory, _receipt = _make_backup(repository, destination, inventory)
+    backup_directory, backup_receipt = _make_backup(repository, destination, inventory)
     with (
         patch("scripts.postgres_backup_restore._preflight"),
         patch("scripts.postgres_backup_restore._validate_custom_dump"),
         patch("scripts.postgres_backup_restore._database_exists", return_value=False),
         patch("scripts.postgres_backup_restore._create_target_database"),
         patch("scripts.postgres_backup_restore._restore_database"),
+        patch("scripts.postgres_backup_restore._cleanup_target_database") as cleanup,
         patch("scripts.postgres_backup_restore._read_inventory", return_value=restored),
+        patch(
+            "scripts.postgres_backup_restore._inspect_publication",
+            return_value=_canonical_publication(),
+        ),
         pytest.raises(BackupRestoreError) as caught,
     ):
         restore_backup(
             repository=repository,
             backup_directory=backup_directory,
             target_database="grocery_restore_publication_drift",
+            expected_manifest_sha256=backup_receipt.manifest_sha256,
         )
 
     assert caught.value.code == "publication_contract_mismatch"
+    cleanup.assert_called_once()
+
+
+@pytest.mark.parametrize(
+    ("stage", "expected_code", "cleanup_expected"),
+    (
+        ("create", "create_target_failed", False),
+        ("restore", "restore_failed", True),
+        ("inventory", "inventory_invalid", True),
+        ("inspection", "publication_inspection_failed", True),
+    ),
+)
+def test_restore_only_cleans_target_after_unambiguous_create_success(
+    repository: Path,
+    destination: Path,
+    inventory: Inventory,
+    stage: str,
+    expected_code: str,
+    cleanup_expected: bool,
+) -> None:
+    backup_directory, backup_receipt = _make_backup(repository, destination, inventory)
+    with (
+        patch("scripts.postgres_backup_restore._preflight", return_value=_CONTAINER),
+        patch("scripts.postgres_backup_restore._validate_custom_dump"),
+        patch("scripts.postgres_backup_restore._database_exists", return_value=False),
+        patch("scripts.postgres_backup_restore._create_target_database") as create_target,
+        patch("scripts.postgres_backup_restore._restore_database") as restore,
+        patch("scripts.postgres_backup_restore._read_inventory", return_value=inventory) as read,
+        patch(
+            "scripts.postgres_backup_restore._inspect_publication",
+            return_value=_canonical_publication(),
+        ) as inspect,
+        patch("scripts.postgres_backup_restore._cleanup_target_database") as cleanup,
+        pytest.raises(BackupRestoreError) as caught,
+    ):
+        failing = {
+            "create": create_target,
+            "restore": restore,
+            "inventory": read,
+            "inspection": inspect,
+        }[stage]
+        failing.side_effect = BackupRestoreError(expected_code)
+        restore_backup(
+            repository=repository,
+            backup_directory=backup_directory,
+            target_database="grocery_restore_failure_cleanup",
+            expected_manifest_sha256=backup_receipt.manifest_sha256,
+        )
+
+    assert caught.value.code == expected_code
+    if cleanup_expected:
+        cleanup.assert_called_once()
+    else:
+        cleanup.assert_not_called()
+
+
+def test_restore_reports_fixed_cleanup_failure_code(
+    repository: Path,
+    destination: Path,
+    inventory: Inventory,
+) -> None:
+    backup_directory, backup_receipt = _make_backup(repository, destination, inventory)
+    with (
+        patch("scripts.postgres_backup_restore._preflight", return_value=_CONTAINER),
+        patch("scripts.postgres_backup_restore._validate_custom_dump"),
+        patch("scripts.postgres_backup_restore._database_exists", return_value=False),
+        patch("scripts.postgres_backup_restore._create_target_database"),
+        patch(
+            "scripts.postgres_backup_restore._restore_database",
+            side_effect=BackupRestoreError("restore_failed"),
+        ),
+        patch(
+            "scripts.postgres_backup_restore._cleanup_target_database",
+            side_effect=BackupRestoreError("target_cleanup_failed"),
+        ),
+        pytest.raises(BackupRestoreError) as caught,
+    ):
+        restore_backup(
+            repository=repository,
+            backup_directory=backup_directory,
+            target_database="grocery_restore_cleanup_failure",
+            expected_manifest_sha256=backup_receipt.manifest_sha256,
+        )
+
+    assert caught.value.code == "target_cleanup_failed"
 
 
 def test_restore_rejects_checksum_tamper_before_preflight(
@@ -409,7 +673,7 @@ def test_restore_rejects_checksum_tamper_before_preflight(
     destination: Path,
     inventory: Inventory,
 ) -> None:
-    backup_directory, _receipt = _make_backup(repository, destination, inventory)
+    backup_directory, backup_receipt = _make_backup(repository, destination, inventory)
     dump_path = backup_directory / "database.dump"
     dump_path.write_bytes(dump_path.read_bytes() + b"tampered")
     os.chmod(dump_path, 0o600)
@@ -420,18 +684,71 @@ def test_restore_rejects_checksum_tamper_before_preflight(
                 repository=repository,
                 backup_directory=backup_directory,
                 target_database="grocery_restore_tamper",
+                expected_manifest_sha256=backup_receipt.manifest_sha256,
             )
 
     assert caught.value.code == "checksum_mismatch"
     preflight.assert_not_called()
 
 
+def test_restore_requires_canonical_matching_manifest_receipt_before_preflight(
+    repository: Path,
+    destination: Path,
+    inventory: Inventory,
+) -> None:
+    backup_directory, backup_receipt = _make_backup(repository, destination, inventory)
+    for expected in ("A" * 64, "0" * 64):
+        with (
+            patch("scripts.postgres_backup_restore._preflight") as preflight,
+            pytest.raises(BackupRestoreError) as caught,
+        ):
+            restore_backup(
+                repository=repository,
+                backup_directory=backup_directory,
+                target_database="grocery_restore_manifest_receipt",
+                expected_manifest_sha256=expected,
+            )
+
+        assert caught.value.code == "manifest_receipt_mismatch"
+        preflight.assert_not_called()
+    assert backup_receipt.manifest_sha256 == _manifest_receipt(backup_directory)
+
+
+def test_alternate_valid_rehashed_archive_fails_pinned_manifest_receipt(
+    repository: Path,
+    tmp_path: Path,
+    inventory: Inventory,
+) -> None:
+    first_root = tmp_path / "first-backups"
+    second_root = tmp_path / "second-backups"
+    first_root.mkdir()
+    second_root.mkdir()
+    _first_directory, first_receipt = _make_backup(repository, first_root, inventory)
+    second_directory, second_receipt = _make_backup(repository, second_root, inventory)
+    assert first_receipt.manifest_sha256 != second_receipt.manifest_sha256
+    assert second_receipt.manifest_sha256 == _manifest_receipt(second_directory)
+
+    with (
+        patch("scripts.postgres_backup_restore._preflight") as preflight,
+        pytest.raises(BackupRestoreError) as caught,
+    ):
+        restore_backup(
+            repository=repository,
+            backup_directory=second_directory,
+            target_database="grocery_restore_alternate_archive",
+            expected_manifest_sha256=first_receipt.manifest_sha256,
+        )
+
+    assert caught.value.code == "manifest_receipt_mismatch"
+    preflight.assert_not_called()
+
+
 def test_restore_rejects_publication_contract_hash_tamper_before_preflight(
     repository: Path,
     destination: Path,
     inventory: Inventory,
 ) -> None:
-    backup_directory, _receipt = _make_backup(repository, destination, inventory)
+    backup_directory, _backup_receipt = _make_backup(repository, destination, inventory)
     manifest_path = backup_directory / "manifest.json"
     manifest = json.loads(manifest_path.read_text(encoding="utf-8"))
     manifest["inventory"]["publication_sha256"] = "f" * 64
@@ -446,6 +763,34 @@ def test_restore_rejects_publication_contract_hash_tamper_before_preflight(
             repository=repository,
             backup_directory=backup_directory,
             target_database="grocery_restore_contract_hash_tamper",
+            expected_manifest_sha256=_manifest_receipt(backup_directory),
+        )
+
+    assert caught.value.code == "publication_contract_mismatch"
+    preflight.assert_not_called()
+
+
+def test_restore_rejects_canonical_publication_tamper_before_preflight(
+    repository: Path,
+    destination: Path,
+    inventory: Inventory,
+) -> None:
+    backup_directory, _backup_receipt = _make_backup(repository, destination, inventory)
+    manifest_path = backup_directory / "manifest.json"
+    manifest = json.loads(manifest_path.read_text(encoding="utf-8"))
+    manifest["inventory"]["canonical_publication"]["typed_fact_set_sha256"] = "b" * 64
+    manifest_path.write_text(json.dumps(manifest), encoding="utf-8")
+    os.chmod(manifest_path, 0o600)
+
+    with (
+        patch("scripts.postgres_backup_restore._preflight") as preflight,
+        pytest.raises(BackupRestoreError) as caught,
+    ):
+        restore_backup(
+            repository=repository,
+            backup_directory=backup_directory,
+            target_database="grocery_restore_canonical_tamper",
+            expected_manifest_sha256=_manifest_receipt(backup_directory),
         )
 
     assert caught.value.code == "publication_contract_mismatch"
@@ -457,7 +802,7 @@ def test_restore_rejects_broad_backup_permissions(
     destination: Path,
     inventory: Inventory,
 ) -> None:
-    backup_directory, _receipt = _make_backup(repository, destination, inventory)
+    backup_directory, backup_receipt = _make_backup(repository, destination, inventory)
     os.chmod(backup_directory / "manifest.json", 0o644)
 
     with pytest.raises(BackupRestoreError) as caught:
@@ -465,11 +810,104 @@ def test_restore_rejects_broad_backup_permissions(
             repository=repository,
             backup_directory=backup_directory,
             target_database="grocery_restore_permissions",
+            expected_manifest_sha256=backup_receipt.manifest_sha256,
         )
 
     assert caught.value.code == "backup_directory_permissions"
 
 
+@pytest.mark.parametrize("filename", ("manifest.json", "database.dump"))
+def test_restore_rejects_symlinked_backup_member_before_preflight(
+    repository: Path,
+    destination: Path,
+    inventory: Inventory,
+    filename: str,
+) -> None:
+    backup_directory, backup_receipt = _make_backup(repository, destination, inventory)
+    member = backup_directory / filename
+    original = backup_directory / f"{filename}.original"
+    member.rename(original)
+    member.symlink_to(original.name)
+
+    with (
+        patch("scripts.postgres_backup_restore._preflight") as preflight,
+        pytest.raises(BackupRestoreError) as caught,
+    ):
+        restore_backup(
+            repository=repository,
+            backup_directory=backup_directory,
+            target_database="grocery_restore_symlink",
+            expected_manifest_sha256=backup_receipt.manifest_sha256,
+        )
+
+    assert caught.value.code == "dump_file_invalid"
+    preflight.assert_not_called()
+
+
+def test_restore_keeps_one_dump_descriptor_when_path_is_renamed_after_load(
+    repository: Path,
+    destination: Path,
+    inventory: Inventory,
+) -> None:
+    backup_directory, backup_receipt = _make_backup(repository, destination, inventory)
+    moved_directory = destination / "moved-after-load"
+    validated_descriptors: list[int] = []
+    restored_descriptors: list[int] = []
+
+    def validate_dump(
+        _repository: Path,
+        descriptor: int,
+        _container: _DatabaseContainer,
+        _application_name: str,
+    ) -> None:
+        validated_descriptors.append(descriptor)
+        backup_directory.rename(moved_directory)
+        backup_directory.mkdir(mode=0o700)
+
+    def restore_dump(
+        _repository: Path,
+        descriptor: int,
+        _target: str,
+        _container: _DatabaseContainer,
+        _application_name: str,
+    ) -> None:
+        restored_descriptors.append(descriptor)
+        duplicate = os.dup(descriptor)
+        try:
+            os.lseek(duplicate, 0, os.SEEK_SET)
+            assert os.read(duplicate, 5) == b"PGDMP"
+        finally:
+            os.close(duplicate)
+
+    with (
+        patch("scripts.postgres_backup_restore._preflight"),
+        patch(
+            "scripts.postgres_backup_restore._validate_custom_dump",
+            side_effect=validate_dump,
+        ),
+        patch("scripts.postgres_backup_restore._database_exists", return_value=False),
+        patch("scripts.postgres_backup_restore._create_target_database"),
+        patch(
+            "scripts.postgres_backup_restore._restore_database",
+            side_effect=restore_dump,
+        ),
+        patch("scripts.postgres_backup_restore._read_inventory", return_value=inventory),
+        patch(
+            "scripts.postgres_backup_restore._inspect_publication",
+            return_value=_canonical_publication(),
+        ),
+    ):
+        restore_backup(
+            repository=repository,
+            backup_directory=backup_directory,
+            target_database="grocery_restore_path_swap",
+            expected_manifest_sha256=backup_receipt.manifest_sha256,
+        )
+
+    assert validated_descriptors == restored_descriptors
+    assert len(validated_descriptors) == 1
+
+
 def test_inventory_parser_is_bounded_and_canonical(inventory: Inventory) -> None:
     parsed = _parse_inventory(inventory.canonical_data())
     assert parsed == inventory
@@ -573,23 +1011,394 @@ def test_publication_inventory_sql_is_ordered_and_omits_publisher_identity() ->
     assert "publisher" not in _INVENTORY_SQL
 
 
+def test_canonical_inspector_uses_bounded_sanitized_subprocess_environment(
+    repository: Path,
+    monkeypatch: pytest.MonkeyPatch,
+) -> None:
+    binary_directory = repository / ".venv" / "bin"
+    binary_directory.mkdir(parents=True)
+    python = binary_directory / "python"
+    python.write_bytes(b"synthetic")
+    (repository / "manage.py").write_text("# synthetic\n", encoding="utf-8")
+    monkeypatch.setenv("KAMIS_API_KEY", "must-not-cross-boundary")
+    monkeypatch.setenv("DATABASE_URL", "must-not-cross-boundary")
+    completed = subprocess.CompletedProcess(
+        args=(),
+        returncode=0,
+        stdout=json.dumps(_canonical_publication().canonical_data()).encode("ascii"),
+    )
+
+    with (
+        patch("scripts.postgres_backup_restore._require_same_database_container") as identity,
+        patch("scripts.postgres_backup_restore.subprocess.run", return_value=completed) as run,
+    ):
+        inspected = _inspect_publication(
+            repository,
+            "grocery_restore_inspection",
+            _CONTAINER,
+            _APPLICATION_NAME,
+        )
+
+    assert inspected == _canonical_publication()
+    assert identity.call_count == 2
+    command = run.call_args.args[0]
+    environment = run.call_args.kwargs["env"]
+    assert command == (str(python), str(repository / "manage.py"), "inspect_recent_publication")
+    assert run.call_args.kwargs["timeout"] == 120
+    assert "KAMIS_API_KEY" not in environment
+    assert "must-not-cross-boundary" not in str(environment)
+    assert environment["DATABASE_URL"].endswith("/grocery_restore_inspection")
+    assert environment["PGAPPNAME"] == _APPLICATION_NAME
+    assert "grocery_restore_inspection" not in str(command)
+
+
+@pytest.mark.parametrize(
+    "unsafe_receipt",
+    (
+        {**_canonical_publication().canonical_data(), "unexpected": "value"},
+        {**_canonical_publication().canonical_data(), "publication_state": "ERROR"},
+        {**_canonical_publication().canonical_data(), "last_activation_operation": []},
+    ),
+)
+def test_canonical_inspector_rejects_noncanonical_receipt(
+    repository: Path,
+    unsafe_receipt: dict[str, object],
+) -> None:
+    binary_directory = repository / ".venv" / "bin"
+    binary_directory.mkdir(parents=True)
+    (binary_directory / "python").write_bytes(b"synthetic")
+    (repository / "manage.py").write_text("# synthetic\n", encoding="utf-8")
+    completed = subprocess.CompletedProcess(
+        args=(),
+        returncode=0,
+        stdout=json.dumps(unsafe_receipt).encode("ascii"),
+    )
+
+    with (
+        patch("scripts.postgres_backup_restore._require_same_database_container"),
+        patch("scripts.postgres_backup_restore.subprocess.run", return_value=completed),
+        pytest.raises(BackupRestoreError) as caught,
+    ):
+        _inspect_publication(repository, "grocery", _CONTAINER, _APPLICATION_NAME)
+
+    assert caught.value.code == "publication_inspection_failed"
+
+
 def test_docker_environment_drops_database_and_password_values(
     monkeypatch: pytest.MonkeyPatch,
 ) -> None:
     monkeypatch.setenv("DATABASE_URL", "private-database-url")
     monkeypatch.setenv("PGPASSWORD", "private-password")
     monkeypatch.setenv("POSTGRES_PASSWORD", "private-postgres-password")
-    monkeypatch.setenv("DOCKER_HOST", "unix:///safe/docker.sock")
+    monkeypatch.setenv("DOCKER_HOST", "tcp://remote.invalid:2376")
+    monkeypatch.setenv("DOCKER_CONTEXT", "remote-context")
+    monkeypatch.setenv("DOCKER_TLS_VERIFY", "1")
+    monkeypatch.setenv("DOCKER_CERT_PATH", "/private/cert-path")
+    monkeypatch.setenv("DOCKER_CONFIG", "/private/docker-config")
 
     environment = _docker_environment()
 
-    assert environment["DOCKER_HOST"] == "unix:///safe/docker.sock"
-    assert "DATABASE_URL" not in environment
-    assert "PGPASSWORD" not in environment
-    assert "POSTGRES_PASSWORD" not in environment
+    for removed in (
+        "DATABASE_URL",
+        "PGPASSWORD",
+        "POSTGRES_PASSWORD",
+        "DOCKER_HOST",
+        "DOCKER_CONTEXT",
+        "DOCKER_TLS_VERIFY",
+        "DOCKER_CERT_PATH",
+    ):
+        assert removed not in environment
+    assert environment == {
+        "DOCKER_CONFIG": "/var/empty",
+        "HOME": "/var/empty",
+        "LANG": "C",
+        "LC_ALL": "C",
+    }
     assert "private" not in str(environment)
 
 
+def test_container_discovery_accepts_exact_local_compose_identity(
+    repository: Path,
+) -> None:
+    inspection = subprocess.CompletedProcess(
+        args=(),
+        returncode=0,
+        stdout=json.dumps(_container_inspection(repository)).encode(),
+    )
+    with (
+        patch("scripts.postgres_backup_restore._require_local_docker_socket"),
+        patch(
+            "scripts.postgres_backup_restore._database_container_ids",
+            side_effect=((_CONTAINER_ID,), (_CONTAINER_ID,)),
+        ),
+        patch(
+            "scripts.postgres_backup_restore._run_docker_cli",
+            return_value=inspection,
+        ) as run,
+    ):
+        discovered = _discover_database_container(repository, _DOCKER)
+
+    assert discovered == _CONTAINER_ID
+    assert run.call_args.args[2] == (
+        "container",
+        "inspect",
+        _CONTAINER_ID,
+    )
+
+
+@pytest.mark.parametrize(
+    ("field", "value"),
+    (
+        ("project", "other-project"),
+        ("service", "web"),
+        ("working_dir", "/wrong/repository"),
+        ("config_files", "/wrong/compose.yaml"),
+        ("status", "exited"),
+        ("health", "starting"),
+        ("host_ip", "0.0.0.0"),  # noqa: S104 - unsafe published-port fixture.
+        ("host_port", "6543"),
+        ("extra_port", True),
+    ),
+)
+def test_container_inspection_rejects_wrong_identity_or_broad_port(
+    repository: Path,
+    field: str,
+    value: object,
+) -> None:
+    decoded = _container_inspection(repository)
+    inspection = decoded[0]
+    config = inspection["Config"]
+    state = inspection["State"]
+    network = inspection["NetworkSettings"]
+    assert isinstance(config, dict)
+    assert isinstance(state, dict)
+    assert isinstance(network, dict)
+    labels = config["Labels"]
+    ports = network["Ports"]
+    assert isinstance(labels, dict)
+    assert isinstance(ports, dict)
+    if field == "project":
+        labels["com.docker.compose.project"] = value
+    elif field == "service":
+        labels["com.docker.compose.service"] = value
+    elif field == "working_dir":
+        labels["com.docker.compose.project.working_dir"] = value
+    elif field == "config_files":
+        labels["com.docker.compose.project.config_files"] = value
+    elif field == "status":
+        state["Status"] = value
+    elif field == "health":
+        health = state["Health"]
+        assert isinstance(health, dict)
+        health["Status"] = value
+    elif field == "host_ip":
+        mapping = ports["5432/tcp"]
+        assert isinstance(mapping, list) and isinstance(mapping[0], dict)
+        mapping[0]["HostIp"] = value
+    elif field == "host_port":
+        mapping = ports["5432/tcp"]
+        assert isinstance(mapping, list) and isinstance(mapping[0], dict)
+        mapping[0]["HostPort"] = value
+    else:
+        ports["9999/tcp"] = [{"HostIp": "127.0.0.1", "HostPort": "9999"}]
+
+    with pytest.raises(BackupRestoreError) as caught:
+        _validate_database_container_inspection(repository, _CONTAINER_ID, decoded)
+
+    assert caught.value.code == "database_container_invalid"
+
+
+def test_direct_docker_cli_uses_only_fixed_host_and_sanitized_environment(
+    repository: Path,
+) -> None:
+    completed = subprocess.CompletedProcess(args=(), returncode=0, stdout=b"")
+    with patch("scripts.postgres_backup_restore.subprocess.run", return_value=completed) as run:
+        _run_docker_cli(
+            repository,
+            _DOCKER,
+            ("version",),
+            stdin=subprocess.DEVNULL,
+            stdout=subprocess.PIPE,
+            timeout=30,
+            failure_code="docker_command_failed",
+        )
+
+    assert run.call_args.args[0] == (
+        _DOCKER,
+        "--host=unix:///var/run/docker.sock",
+        "version",
+    )
+    assert run.call_args.kwargs["env"] == _docker_environment()
+
+
+def test_database_command_uses_direct_exec_and_fails_on_container_swap(
+    repository: Path,
+) -> None:
+    completed = subprocess.CompletedProcess(args=(), returncode=0, stdout=b"")
+    with (
+        patch("scripts.postgres_backup_restore._require_same_database_container") as identity,
+        patch(
+            "scripts.postgres_backup_restore._run_docker_cli",
+            return_value=completed,
+        ) as run,
+    ):
+        _run_database_command(
+            repository,
+            _CONTAINER,
+            _APPLICATION_NAME,
+            ("psql", "--version"),
+            stdin=subprocess.DEVNULL,
+            stdout=subprocess.PIPE,
+            timeout=30,
+            failure_code="docker_command_failed",
+        )
+
+    assert identity.call_count == 2
+    assert run.call_args.args[2] == (
+        "exec",
+        "-i",
+        f"--env=PGAPPNAME={_APPLICATION_NAME}",
+        _CONTAINER_ID,
+        "psql",
+        "--version",
+    )
+
+    with (
+        patch(
+            "scripts.postgres_backup_restore._require_same_database_container",
+            side_effect=(None, BackupRestoreError("database_container_invalid")),
+        ),
+        patch("scripts.postgres_backup_restore._run_docker_cli", return_value=completed),
+        pytest.raises(BackupRestoreError) as swapped,
+    ):
+        _run_database_command(
+            repository,
+            _CONTAINER,
+            _APPLICATION_NAME,
+            ("psql", "--version"),
+            stdin=subprocess.DEVNULL,
+            stdout=subprocess.PIPE,
+            timeout=30,
+            failure_code="docker_command_failed",
+        )
+
+    assert swapped.value.code == "database_container_invalid"
+
+
+def test_container_discovery_rejects_identity_swap(
+    repository: Path,
+) -> None:
+    replacement_id = "f" * 64
+    inspection = subprocess.CompletedProcess(
+        args=(),
+        returncode=0,
+        stdout=json.dumps(_container_inspection(repository)).encode(),
+    )
+    with (
+        patch("scripts.postgres_backup_restore._require_local_docker_socket"),
+        patch(
+            "scripts.postgres_backup_restore._database_container_ids",
+            side_effect=((_CONTAINER_ID,), (replacement_id,)),
+        ),
+        patch("scripts.postgres_backup_restore._run_docker_cli", return_value=inspection),
+        pytest.raises(BackupRestoreError) as caught,
+    ):
+        _discover_database_container(repository, _DOCKER)
+
+    assert caught.value.code == "database_container_invalid"
+
+
+def test_restore_command_has_shorter_gnu_timeout_and_unique_application_name(
+    repository: Path,
+    tmp_path: Path,
+) -> None:
+    dump_path = tmp_path / "database.dump"
+    dump_path.write_bytes(b"PGDMPsynthetic")
+    descriptor = os.open(dump_path, os.O_RDONLY)
+    try:
+        with patch("scripts.postgres_backup_restore._run_database_command") as run:
+            _restore_database(
+                repository,
+                descriptor,
+                "grocery_restore_timeout",
+                _CONTAINER,
+                _APPLICATION_NAME,
+            )
+    finally:
+        os.close(descriptor)
+
+    arguments = run.call_args.args[3]
+    assert arguments[:5] == (
+        "timeout",
+        "--signal=TERM",
+        "--kill-after=5s",
+        "570s",
+        "pg_restore",
+    )
+    assert run.call_args.kwargs["timeout"] == 600
+    first = _new_application_name("restore")
+    second = _new_application_name("restore")
+    assert first != second
+    assert first.startswith("grocery_restore_")
+    assert len(first) == len("grocery_restore_") + 32
+
+
+def test_target_cleanup_terminates_sessions_verifies_zero_drops_and_verifies_absence(
+    repository: Path,
+) -> None:
+    with (
+        patch(
+            "scripts.postgres_backup_restore._capture_database_command",
+            side_effect=(b"2\n", b"0\n", b"0\n"),
+        ) as capture,
+        patch("scripts.postgres_backup_restore._run_database_command") as run,
+    ):
+        _cleanup_target_database(
+            repository,
+            "grocery_restore_cleanup",
+            _CONTAINER,
+            _APPLICATION_NAME,
+        )
+
+    assert capture.call_count == 3
+    terminate_arguments = capture.call_args_list[0].args[3]
+    session_arguments = capture.call_args_list[1].args[3]
+    absence_arguments = capture.call_args_list[2].args[3]
+    assert "target_sessions AS MATERIALIZED" in terminate_arguments[-1]
+    assert "pg_terminate_backend(pid)" in terminate_arguments[-1]
+    assert "pg_stat_activity" in session_arguments[-1]
+    assert "pg_database" in absence_arguments[-1]
+    drop_arguments = run.call_args.args[3]
+    assert drop_arguments == (
+        "dropdb",
+        "--no-password",
+        "--username=grocery",
+        "--if-exists",
+        "--force",
+        "grocery_restore_cleanup",
+    )
+
+
+def test_target_cleanup_fails_closed_when_sessions_remain(repository: Path) -> None:
+    with (
+        patch(
+            "scripts.postgres_backup_restore._capture_database_command",
+            side_effect=(b"1\n", b"1\n"),
+        ),
+        patch("scripts.postgres_backup_restore._run_database_command") as drop,
+        pytest.raises(BackupRestoreError) as caught,
+    ):
+        _cleanup_target_database(
+            repository,
+            "grocery_restore_cleanup",
+            _CONTAINER,
+            _APPLICATION_NAME,
+        )
+
+    assert caught.value.code == "target_cleanup_failed"
+    drop.assert_not_called()
+
+
 def test_preflight_requires_all_postgres_18_tools_and_source_connectivity(
     repository: Path,
 ) -> None:
@@ -597,44 +1406,60 @@ def test_preflight_requires_all_postgres_18_tools_and_source_connectivity(
         b"pg_dump (PostgreSQL) 18.6 (Debian 18.6-1)\n",
         b"pg_restore (PostgreSQL) 18.6 (Debian 18.6-1)\n",
         b"createdb (PostgreSQL) 18.6 (Debian 18.6-1)\n",
+        b"dropdb (PostgreSQL) 18.6 (Debian 18.6-1)\n",
         b"psql (PostgreSQL) 18.6 (Debian 18.6-1)\n",
+        b"timeout (GNU coreutils) 9.7\n",
         b"1\n",
     )
     with (
-        patch("scripts.postgres_backup_restore.shutil.which", return_value="/safe/docker"),
+        patch("scripts.postgres_backup_restore._resolve_docker_binary", return_value=_DOCKER),
         patch(
-            "scripts.postgres_backup_restore._capture_compose",
+            "scripts.postgres_backup_restore._discover_database_container",
+            return_value=_CONTAINER_ID,
+        ),
+        patch(
+            "scripts.postgres_backup_restore._capture_database_command",
             side_effect=versions,
         ) as capture,
     ):
-        _preflight(repository)
+        result = _preflight(repository, _APPLICATION_NAME)
 
-    assert [entry.args[1][0] for entry in capture.call_args_list] == [
+    assert result == _CONTAINER
+    assert [entry.args[3][0] for entry in capture.call_args_list] == [
         "pg_dump",
         "pg_restore",
         "createdb",
+        "dropdb",
         "psql",
+        "timeout",
         "psql",
     ]
 
 
 def test_preflight_rejects_missing_or_wrong_major_tool(repository: Path) -> None:
     with (
-        patch("scripts.postgres_backup_restore.shutil.which", return_value=None),
+        patch(
+            "scripts.postgres_backup_restore._resolve_docker_binary",
+            side_effect=BackupRestoreError("docker_unavailable"),
+        ),
         pytest.raises(BackupRestoreError) as missing,
     ):
-        _preflight(repository)
+        _preflight(repository, _APPLICATION_NAME)
     assert missing.value.code == "docker_unavailable"
 
     with (
-        patch("scripts.postgres_backup_restore.shutil.which", return_value="/safe/docker"),
+        patch("scripts.postgres_backup_restore._resolve_docker_binary", return_value=_DOCKER),
+        patch(
+            "scripts.postgres_backup_restore._discover_database_container",
+            return_value=_CONTAINER_ID,
+        ),
         patch(
-            "scripts.postgres_backup_restore._capture_compose",
+            "scripts.postgres_backup_restore._capture_database_command",
             return_value=b"pg_dump (PostgreSQL) 17.9\n",
         ),
         pytest.raises(BackupRestoreError) as wrong_version,
     ):
-        _preflight(repository)
+        _preflight(repository, _APPLICATION_NAME)
     assert wrong_version.value.code == "postgres_version_mismatch"
 
 
@@ -672,12 +1497,36 @@ def test_cli_failure_never_reflects_path_target_or_arbitrary_exception(
                 path_marker,
                 "--target-database",
                 target_marker,
+                "--expected-manifest-sha256",
+                "0" * 64,
             ]
         )
     output = capsys.readouterr()
     assert result == 1
     assert "code=internal_error" in output.out
-    assert "cleanup=inspect_and_drop_explicit_restore_target_if_created" in output.out
+    assert "cleanup=manual_target_cleanup_required" in output.out
+    assert "private" not in output.out
+    assert output.err == ""
+
+    with patch(
+        "scripts.postgres_backup_restore.restore_backup",
+        side_effect=BackupRestoreError("restore_failed"),
+    ):
+        result = main(
+            [
+                "restore",
+                "--backup-dir",
+                path_marker,
+                "--target-database",
+                target_marker,
+                "--expected-manifest-sha256",
+                "0" * 64,
+            ]
+        )
+    output = capsys.readouterr()
+    assert result == 1
+    assert "code=restore_failed" in output.out
+    assert "cleanup=automatic_created_target_cleanup_verified_or_not_created" in output.out
     assert "private" not in output.out
     assert output.err == ""
 
@@ -694,6 +1543,27 @@ def test_cli_usage_error_does_not_reflect_unknown_argument(
     assert output.err == ""
 
 
+def test_restore_cli_requires_expected_manifest_receipt(
+    capsys: pytest.CaptureFixture[str],
+) -> None:
+    with patch("scripts.postgres_backup_restore.restore_backup") as restore:
+        result = main(
+            [
+                "restore",
+                "--backup-dir",
+                "/private/backup",
+                "--target-database",
+                "grocery_restore_missing_receipt",
+            ]
+        )
+
+    output = capsys.readouterr()
+    assert result == 2
+    assert "code=usage_error" in output.out
+    assert "private" not in output.out
+    restore.assert_not_called()
+
+
 def test_target_database_contract_is_explicit_and_separate() -> None:
     assert _validated_target_database("grocery_restore_rehearsal_20260830") == (
         "grocery_restore_rehearsal_20260830"
@@ -708,12 +1578,12 @@ def test_target_preflight_uses_validated_literal_in_command_mode(
 ) -> None:
     target = "grocery_restore_rehearsal_20260830"
     with patch(
-        "scripts.postgres_backup_restore._capture_compose",
+        "scripts.postgres_backup_restore._capture_database_command",
         return_value=b"0\n",
     ) as capture:
-        assert _database_exists(repository, target) is False
+        assert _database_exists(repository, target, _CONTAINER, _APPLICATION_NAME) is False
 
-    arguments = capture.call_args.args[1]
+    arguments = capture.call_args.args[3]
     assert not any(argument.startswith("--set=") for argument in arguments)
     expected_command = (
         "--command=SELECT count(*) FROM pg_database WHERE datname = "


