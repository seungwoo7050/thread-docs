## `feat(ops): rehearse postgres recovery`

diff --git a/scripts/postgres_backup_restore.py b/scripts/postgres_backup_restore.py
new file mode 100644
index 0000000..daeb848
--- /dev/null
+++ b/scripts/postgres_backup_restore.py
@@ -0,0 +1,1166 @@
+"""Local PostgreSQL custom-format backup and isolated restore assurance.
+
+The command is deliberately bound to the repository's fixed Docker Compose ``db``
+service. Database credentials and ``DATABASE_URL`` are neither read nor forwarded.
+All subprocess output is captured or discarded and converted to fixed error codes.
+"""
+
+from __future__ import annotations
+
+import argparse
+import hashlib
+import json
+import os
+import re
+import shutil
+import stat
+import subprocess
+import sys
+import uuid
+from collections.abc import Iterator
+from contextlib import contextmanager
+from dataclasses import dataclass
+from datetime import UTC, datetime
+from pathlib import Path
+from typing import Final, Never
+
+_SOURCE_DATABASE: Final = "grocery"
+_DATABASE_USER: Final = "grocery"
+_COMPOSE_SERVICE: Final = "db"
+_FORMAT_VERSION: Final = "grocery-postgres-custom-v1"
+_POSTGRES_MAJOR: Final = 18
+_DUMP_FILENAME: Final = "database.dump"
+_MANIFEST_FILENAME: Final = "manifest.json"
+_DUMP_MAGIC: Final = b"PGDMP"
+_MAX_MANIFEST_BYTES: Final = 4 * 1024 * 1024
+_MAX_INVENTORY_BYTES: Final = 4 * 1024 * 1024
+_MAX_DUMP_BYTES: Final = 8 * 1024 * 1024 * 1024
+_MAX_TABLES: Final = 1_024
+_MAX_MIGRATIONS: Final = 16_384
+_MAX_ROW_COUNT: Final = (2**63) - 1
+_MAX_ACTIVATIONS: Final = 10_000
+_MAX_PUBLICATION_ENTRIES: Final = 100_000
+_TOOL_TIMEOUT_SECONDS: Final = 30
+_INVENTORY_TIMEOUT_SECONDS: Final = 120
+_DUMP_TIMEOUT_SECONDS: Final = 600
+_RESTORE_TIMEOUT_SECONDS: Final = 600
+_SHA256 = re.compile(r"[0-9a-f]{64}\Z")
+_TABLE_NAME = re.compile(r"public\.[a-z0-9_]{1,63}\Z")
+_MIGRATION_TOKEN = re.compile(r"[A-Za-z0-9_.-]{1,128}\Z")
+_REASON_CODE = re.compile(r"[A-Z][A-Z0-9_]{0,63}\Z")
+_TARGET_DATABASE = re.compile(r"grocery_restore_[a-z0-9][a-z0-9_]{0,45}\Z")
+_VERSION_TOKEN = re.compile(r"([0-9]+)(?:\.[0-9]+)*\Z")
+
+_INVENTORY_SQL: Final = r"""
+CREATE TEMP TABLE assurance_counts (
+    table_name text PRIMARY KEY,
+    row_count bigint NOT NULL
+) ON COMMIT DROP;
+DO $assurance$
+DECLARE
+    table_record record;
+BEGIN
+    FOR table_record IN
+        SELECT schemaname, tablename
+        FROM pg_tables
+        WHERE schemaname = 'public'
+        ORDER BY schemaname, tablename
+    LOOP
+        EXECUTE format(
+            'INSERT INTO assurance_counts(table_name, row_count) '
+            'SELECT %L, count(*) FROM %I.%I',
+            table_record.schemaname || '.' || table_record.tablename,
+            table_record.schemaname,
+            table_record.tablename
+        );
+    END LOOP;
+END
+$assurance$;
+SELECT json_build_object(
+    'rows', COALESCE(
+        (
+            SELECT json_object_agg(table_name, row_count ORDER BY table_name)
+            FROM assurance_counts
+        ),
+        '{}'::json
+    ),
+    'migrations', COALESCE(
+        (
+            SELECT json_agg(json_build_array(app, name) ORDER BY app, name)
+            FROM public.django_migrations
+        ),
+        '[]'::json
+    ),
+    'publication', json_build_object(
+        'channel', (
+            SELECT json_build_object(
+                'channel', channel,
+                'version', version,
+                'current_revision_id', current_revision_id
+            )
+            FROM public.grocery_publicationchannel
+            WHERE channel = 'RECENT_RETAIL'
+        ),
+        'active_revision', (
+            SELECT json_build_object(
+                'id', revision.id,
+                'typed_fact_set_sha256', revision.typed_fact_set_sha256,
+                'generation_id', revision.generation_id,
+                'review_decision_id', revision.review_decision_id,
+                'review_parse_run_id', decision.parse_run_id,
+                'review_decision', decision.decision,
+                'entry_count', revision.entry_count
+            )
+            FROM public.grocery_publicationchannel AS channel
+            JOIN public.grocery_publicationrevision AS revision
+              ON revision.id = channel.current_revision_id
+             AND revision.channel = 'RECENT_RETAIL'
+             AND revision.sealed_at IS NOT NULL
+            JOIN public.grocery_reviewdecision AS decision
+              ON decision.id = revision.review_decision_id
+            JOIN public.grocery_parserun AS generation
+              ON generation.id = revision.generation_id
+            WHERE channel.channel = 'RECENT_RETAIL'
+              AND decision.decision = 'APPROVE'
+              AND decision.parse_run_id = generation.id
+              AND decision.approved_mode = 'RECENT_COMPARISON'
+              AND generation.status = 'VALIDATED'
+              AND generation.accepted_row_count = revision.entry_count
+              AND revision.entry_count = (
+                  SELECT count(*)
+                  FROM public.grocery_publicationentry AS entry
+                  WHERE entry.revision_id = revision.id
+              )
+              AND NOT EXISTS (
+                  SELECT 1
+                  FROM public.grocery_reviewdecision AS replacement
+                  WHERE replacement.supersedes_id = decision.id
+              )
+        ),
+        'activations', COALESCE(
+            (
+                SELECT json_agg(
+                    json_build_object(
+                        'id', activation.id,
+                        'operation', activation.operation,
+                        'sequence', activation.sequence,
+                        'previous_revision_id', activation.previous_revision_id,
+                        'target_revision_id', activation.target_revision_id,
+                        'reason_code', activation.reason_code,
+                        'acceptance_evidence_sha256',
+                            activation.acceptance_evidence_sha256
+                    )
+                    ORDER BY activation.sequence
+                )
+                FROM public.grocery_publicationactivation AS activation
+                WHERE activation.channel_id = 'RECENT_RETAIL'
+            ),
+            '[]'::json
+        )
+    )
+)::text;
+"""
+
+_CODES: Final = frozenset(
+    {
+        "backup_changed_during_dump",
+        "backup_directory_invalid",
+        "backup_directory_permissions",
+        "backup_directory_unavailable",
+        "backup_id_invalid",
+        "backup_manifest_invalid",
+        "backup_not_custom_format",
+        "backup_too_large",
+        "checksum_mismatch",
+        "compose_file_missing",
+        "create_target_failed",
+        "destination_inside_repository",
+        "docker_command_failed",
+        "docker_unavailable",
+        "dump_failed",
+        "dump_file_invalid",
+        "internal_error",
+        "inventory_invalid",
+        "inventory_mismatch",
+        "migration_mismatch",
+        "postgres_tool_missing",
+        "postgres_version_mismatch",
+        "publication_contract_mismatch",
+        "repository_invalid",
+        "restore_failed",
+        "row_count_mismatch",
+        "source_database_unavailable",
+        "target_database_exists",
+        "target_database_invalid",
+        "target_database_is_source",
+        "target_preflight_failed",
+        "usage_error",
+    }
+)
+
+
+class BackupRestoreError(RuntimeError):
+    """A failure represented only by a fixed, non-sensitive code."""
+
+    def __init__(self, code: str) -> None:
+        selected = code if code in _CODES else "internal_error"
+        self.code = selected
+        super().__init__(selected)
+
+
+class _SafeArgumentParser(argparse.ArgumentParser):
+    """Reject invalid CLI shapes without reflecting supplied argument text."""
+
+    def error(self, message: str) -> Never:
+        del message
+        raise BackupRestoreError("usage_error")
+
+
+@dataclass(frozen=True, slots=True)
+class Inventory:
+    rows: dict[str, int]
+    migrations: tuple[tuple[str, str], ...]
+    publication: dict[str, object]
+
+    def canonical_data(self) -> dict[str, object]:
+        return {
+            "migrations": [list(migration) for migration in self.migrations],
+            "publication": self.publication,
+            "rows": dict(sorted(self.rows.items())),
+        }
+
+    @property
+    def sha256(self) -> str:
+        return hashlib.sha256(_canonical_json(self.canonical_data())).hexdigest()
+
+    @property
+    def publication_sha256(self) -> str:
+        return hashlib.sha256(_canonical_json(self.publication)).hexdigest()
+
+
+@dataclass(frozen=True, slots=True)
+class BackupReceipt:
+    backup_id: uuid.UUID
+    dump_sha256: str
+    manifest_sha256: str
+    table_count: int
+    migration_count: int
+
+    def render(self) -> str:
+        return "\n".join(
+            (
+                "status=backup_complete",
+                f"backup_id={self.backup_id}",
+                f"dump_sha256={self.dump_sha256}",
+                f"manifest_sha256={self.manifest_sha256}",
+                f"tables={self.table_count}",
+                f"migrations={self.migration_count}",
+                "cleanup=retain_or_remove_explicit_backup_directory",
+            )
+        )
+
+
+@dataclass(frozen=True, slots=True)
+class RestoreReceipt:
+    backup_id: uuid.UUID
+    table_count: int
+    migration_count: int
+
+    def render(self) -> str:
+        return "\n".join(
+            (
+                "status=restore_verified",
+                f"backup_id={self.backup_id}",
+                "target_database_verified=yes",
+                "row_counts_consistent=yes",
+                "migrations_consistent=yes",
+                "publication_contract_consistent=yes",
+                f"tables={self.table_count}",
+                f"migrations={self.migration_count}",
+                "cleanup=drop_explicit_restore_target_after_review",
+            )
+        )
+
+
+@dataclass(frozen=True, slots=True)
+class LoadedBackup:
+    backup_id: uuid.UUID
+    dump_path: Path
+    inventory: Inventory
+
+
+def create_backup(*, repository: Path, destination_root: Path) -> BackupReceipt:
+    root = _repository_root(repository)
+    destination = _operator_directory(destination_root, repository=root)
+    _preflight(root)
+    backup_id = uuid.uuid4()
+    backup_directory = destination / f"postgres-backup-{backup_id}"
+
+    with _restrictive_umask():
+        try:
+            backup_directory.mkdir(mode=0o700)
+        except OSError:
+            raise BackupRestoreError("backup_directory_unavailable") from None
+        _require_private_directory(backup_directory)
+        dump_path = backup_directory / _DUMP_FILENAME
+        before = _read_inventory(root, _SOURCE_DATABASE)
+        dump_fd = _create_private_file(dump_path)
+        try:
+            _dump_database(root, dump_fd)
+            os.fsync(dump_fd)
+        finally:
+            os.close(dump_fd)
+        _require_private_regular_file(dump_path, maximum_bytes=_MAX_DUMP_BYTES)
+        if _read_prefix(dump_path, len(_DUMP_MAGIC)) != _DUMP_MAGIC:
+            raise BackupRestoreError("backup_not_custom_format")
+        after = _read_inventory(root, _SOURCE_DATABASE)
+        if before != after:
+            raise BackupRestoreError("backup_changed_during_dump")
+
+        dump_sha256, dump_bytes = _file_sha256(dump_path)
+        manifest = {
+            "backup_id": str(backup_id),
+            "created_at": datetime.now(tz=UTC).isoformat(timespec="seconds").replace("+00:00", "Z"),
+            "dump": {
+                "bytes": dump_bytes,
+                "filename": _DUMP_FILENAME,
+                "sha256": dump_sha256,
+            },
+            "format_version": _FORMAT_VERSION,
+            "inventory": {
+                **before.canonical_data(),
+                "publication_sha256": before.publication_sha256,
+                "sha256": before.sha256,
+            },
+            "postgres_major": _POSTGRES_MAJOR,
+            "source_database": _SOURCE_DATABASE,
+        }
+        manifest_bytes = _canonical_json(manifest) + b"\n"
+        manifest_path = backup_directory / _MANIFEST_FILENAME
+        _write_private_file(manifest_path, manifest_bytes)
+        _fsync_directory(backup_directory)
+
+    return BackupReceipt(
+        backup_id=backup_id,
+        dump_sha256=dump_sha256,
+        manifest_sha256=hashlib.sha256(manifest_bytes).hexdigest(),
+        table_count=len(before.rows),
+        migration_count=len(before.migrations),
+    )
+
+
+def restore_backup(
+    *,
+    repository: Path,
+    backup_directory: Path,
+    target_database: str,
+) -> RestoreReceipt:
+    root = _repository_root(repository)
+    target = _validated_target_database(target_database)
+    selected_backup = _load_backup(backup_directory, repository=root)
+    _preflight(root)
+    _validate_custom_dump(root, selected_backup.dump_path)
+    if _database_exists(root, target):
+        raise BackupRestoreError("target_database_exists")
+    _create_target_database(root, target)
+    _restore_database(root, selected_backup.dump_path, target)
+    restored = _read_inventory(root, target)
+    if restored.rows != selected_backup.inventory.rows:
+        raise BackupRestoreError("row_count_mismatch")
+    if restored.migrations != selected_backup.inventory.migrations:
+        raise BackupRestoreError("migration_mismatch")
+    if (
+        restored.publication != selected_backup.inventory.publication
+        or restored.publication_sha256 != selected_backup.inventory.publication_sha256
+    ):
+        raise BackupRestoreError("publication_contract_mismatch")
+    if restored.sha256 != selected_backup.inventory.sha256:
+        raise BackupRestoreError("inventory_mismatch")
+    return RestoreReceipt(
+        backup_id=selected_backup.backup_id,
+        table_count=len(restored.rows),
+        migration_count=len(restored.migrations),
+    )
+
+
+def main(arguments: list[str] | None = None) -> int:
+    parser = _parser()
+    selected = sys.argv[1:] if arguments is None else arguments
+    try:
+        parsed = parser.parse_args(selected)
+    except BackupRestoreError:
+        _print_failure("usage_error", restore_requested="restore" in selected)
+        return 2
+
+    restore_requested = parsed.operation == "restore"
+    try:
+        receipt: BackupReceipt | RestoreReceipt
+        if parsed.operation == "backup":
+            receipt = create_backup(
+                repository=Path.cwd(),
+                destination_root=Path(parsed.output_dir),
+            )
+        else:
+            receipt = restore_backup(
+                repository=Path.cwd(),
+                backup_directory=Path(parsed.backup_dir),
+                target_database=parsed.target_database,
+            )
+    except BackupRestoreError as error:
+        _print_failure(error.code, restore_requested=restore_requested)
+        return 1
+    except Exception:  # noqa: BLE001 - CLI must not reflect dependency or secret text.
+        _print_failure("internal_error", restore_requested=restore_requested)
+        return 1
+    print(receipt.render())
+    return 0
+
+
+def _parser() -> argparse.ArgumentParser:
+    parser = _SafeArgumentParser(add_help=True)
+    subparsers = parser.add_subparsers(dest="operation", required=True)
+    backup = subparsers.add_parser("backup")
+    backup.add_argument("--output-dir", required=True)
+    restore = subparsers.add_parser("restore")
+    restore.add_argument("--backup-dir", required=True)
+    restore.add_argument("--target-database", required=True)
+    return parser
+
+
+def _print_failure(code: str, *, restore_requested: bool) -> None:
+    selected = code if code in _CODES else "internal_error"
+    print("status=failed")
+    print(f"code={selected}")
+    if restore_requested:
+        print("cleanup=inspect_and_drop_explicit_restore_target_if_created")
+    else:
+        print("cleanup=remove_incomplete_backup_directory_if_created")
+
+
+def _repository_root(repository: Path) -> Path:
+    try:
+        root = repository.resolve(strict=True)
+        compose = root / "compose.yaml"
+        if not root.is_dir():
+            raise OSError
+        metadata = compose.lstat()
+    except OSError:
+        raise BackupRestoreError("repository_invalid") from None
+    if stat.S_ISLNK(metadata.st_mode) or not stat.S_ISREG(metadata.st_mode):
+        raise BackupRestoreError("compose_file_missing")
+    return root
+
+
+def _operator_directory(path: Path, *, repository: Path) -> Path:
+    if not path.is_absolute():
+        raise BackupRestoreError("backup_directory_invalid")
+    try:
+        metadata = path.lstat()
+        resolved = path.resolve(strict=True)
+    except OSError:
+        raise BackupRestoreError("backup_directory_unavailable") from None
+    if stat.S_ISLNK(metadata.st_mode) or not stat.S_ISDIR(metadata.st_mode):
+        raise BackupRestoreError("backup_directory_invalid")
+    if metadata.st_uid != os.geteuid():
+        raise BackupRestoreError("backup_directory_permissions")
+    if _is_within(resolved, repository):
+        raise BackupRestoreError("destination_inside_repository")
+    return resolved
+
+
+def _validated_target_database(value: object) -> str:
+    if value == _SOURCE_DATABASE:
+        raise BackupRestoreError("target_database_is_source")
+    if not isinstance(value, str) or _TARGET_DATABASE.fullmatch(value) is None:
+        raise BackupRestoreError("target_database_invalid")
+    return value
+
+
+def _preflight(repository: Path) -> None:
+    if shutil.which("docker") is None:
+        raise BackupRestoreError("docker_unavailable")
+    for tool in ("pg_dump", "pg_restore", "createdb", "psql"):
+        output = _capture_compose(
+            repository,
+            (tool, "--version"),
+            timeout=_TOOL_TIMEOUT_SECONDS,
+            failure_code="postgres_tool_missing",
+        )
+        try:
+            version = output.decode("ascii", errors="strict").strip()
+        except UnicodeError:
+            raise BackupRestoreError("postgres_version_mismatch") from None
+        prefix = f"{tool} (PostgreSQL) "
+        if not version.startswith(prefix) or len(version) > 256:
+            raise BackupRestoreError("postgres_version_mismatch")
+        version_token = version.removeprefix(prefix).split(maxsplit=1)[0]
+        match = _VERSION_TOKEN.fullmatch(version_token)
+        if match is None or int(match.group(1)) != _POSTGRES_MAJOR:
+            raise BackupRestoreError("postgres_version_mismatch")
+    source_probe = _capture_compose(
+        repository,
+        (
+            "psql",
+            "--no-password",
+            f"--username={_DATABASE_USER}",
+            f"--dbname={_SOURCE_DATABASE}",
+            "--tuples-only",
+            "--no-align",
+            "--command=SELECT 1;",
+        ),
+        timeout=_TOOL_TIMEOUT_SECONDS,
+        failure_code="source_database_unavailable",
+    )
+    if source_probe.strip() != b"1":
+        raise BackupRestoreError("source_database_unavailable")
+
+
+def _dump_database(repository: Path, output_fd: int) -> None:
+    _run_compose(
+        repository,
+        (
+            "pg_dump",
+            "--no-password",
+            f"--username={_DATABASE_USER}",
+            f"--dbname={_SOURCE_DATABASE}",
+            "--format=custom",
+            "--no-owner",
+            "--no-privileges",
+        ),
+        stdin=subprocess.DEVNULL,
+        stdout=output_fd,
+        timeout=_DUMP_TIMEOUT_SECONDS,
+        failure_code="dump_failed",
+    )
+
+
+def _validate_custom_dump(repository: Path, dump_path: Path) -> None:
+    descriptor = os.open(
+        dump_path,
+        os.O_RDONLY | getattr(os, "O_CLOEXEC", 0) | getattr(os, "O_NOFOLLOW", 0),
+    )
+    try:
+        _run_compose(
+            repository,
+            ("pg_restore", "--list"),
+            stdin=descriptor,
+            stdout=subprocess.DEVNULL,
+            timeout=_INVENTORY_TIMEOUT_SECONDS,
+            failure_code="backup_not_custom_format",
+        )
+    finally:
+        os.close(descriptor)
+
+
+def _database_exists(repository: Path, target: str) -> bool:
+    output = _capture_compose(
+        repository,
+        (
+            "psql",
+            "--no-password",
+            f"--username={_DATABASE_USER}",
+            "--dbname=postgres",
+            "--tuples-only",
+            "--no-align",
+            # ``psql --command`` does not perform psql-variable interpolation.
+            # The target has already passed the strict lowercase/underscore
+            # allowlist in ``_validated_target_database`` and is therefore safe
+            # to place in this fixed string literal.
+            f"--command=SELECT count(*) FROM pg_database WHERE datname = '{target}';",  # noqa: S608
+        ),
+        timeout=_TOOL_TIMEOUT_SECONDS,
+        failure_code="target_preflight_failed",
+    )
+    result = output.strip()
+    if result not in {b"0", b"1"}:
+        raise BackupRestoreError("target_preflight_failed")
+    return result == b"1"
+
+
+def _create_target_database(repository: Path, target: str) -> None:
+    _run_compose(
+        repository,
+        (
+            "createdb",
+            "--no-password",
+            f"--username={_DATABASE_USER}",
+            f"--owner={_DATABASE_USER}",
+            "--template=template0",
+            "--encoding=UTF8",
+            target,
+        ),
+        stdin=subprocess.DEVNULL,
+        stdout=subprocess.DEVNULL,
+        timeout=_TOOL_TIMEOUT_SECONDS,
+        failure_code="create_target_failed",
+    )
+
+
+def _restore_database(repository: Path, dump_path: Path, target: str) -> None:
+    descriptor = os.open(
+        dump_path,
+        os.O_RDONLY | getattr(os, "O_CLOEXEC", 0) | getattr(os, "O_NOFOLLOW", 0),
+    )
+    try:
+        _run_compose(
+            repository,
+            (
+                "pg_restore",
+                "--no-password",
+                f"--username={_DATABASE_USER}",
+                f"--dbname={target}",
+                "--exit-on-error",
+                "--single-transaction",
+                "--no-owner",
+                "--no-privileges",
+            ),
+            stdin=descriptor,
+            stdout=subprocess.DEVNULL,
+            timeout=_RESTORE_TIMEOUT_SECONDS,
+            failure_code="restore_failed",
+        )
+    finally:
+        os.close(descriptor)
+
+
+def _read_inventory(repository: Path, database: str) -> Inventory:
+    output = _capture_compose(
+        repository,
+        (
+            "psql",
+            "--no-password",
+            f"--username={_DATABASE_USER}",
+            f"--dbname={database}",
+            "--quiet",
+            "--tuples-only",
+            "--no-align",
+            "--set=ON_ERROR_STOP=1",
+            f"--command={_INVENTORY_SQL}",
+        ),
+        timeout=_INVENTORY_TIMEOUT_SECONDS,
+        failure_code="inventory_invalid",
+    )
+    if len(output) > _MAX_INVENTORY_BYTES:
+        raise BackupRestoreError("inventory_invalid")
+    try:
+        decoded = json.loads(output.decode("utf-8", errors="strict"))
+    except UnicodeError, json.JSONDecodeError:
+        raise BackupRestoreError("inventory_invalid") from None
+    return _parse_inventory(decoded)
+
+
+def _parse_inventory(value: object) -> Inventory:
+    if not isinstance(value, dict) or set(value) != {"rows", "migrations", "publication"}:
+        raise BackupRestoreError("inventory_invalid")
+    raw_rows = value["rows"]
+    raw_migrations = value["migrations"]
+    raw_publication = value["publication"]
+    if (
+        not isinstance(raw_rows, dict)
+        or len(raw_rows) < 1
+        or len(raw_rows) > _MAX_TABLES
+        or not isinstance(raw_migrations, list)
+        or len(raw_migrations) < 1
+        or len(raw_migrations) > _MAX_MIGRATIONS
+    ):
+        raise BackupRestoreError("inventory_invalid")
+    rows: dict[str, int] = {}
+    for key, count in raw_rows.items():
+        if (
+            not isinstance(key, str)
+            or _TABLE_NAME.fullmatch(key) is None
+            or not isinstance(count, int)
+            or isinstance(count, bool)
+            or count < 0
+            or count > _MAX_ROW_COUNT
+        ):
+            raise BackupRestoreError("inventory_invalid")
+        rows[key] = count
+    migrations: list[tuple[str, str]] = []
+    for raw in raw_migrations:
+        if (
+            not isinstance(raw, list)
+            or len(raw) != 2
+            or not all(isinstance(token, str) for token in raw)
+        ):
+            raise BackupRestoreError("inventory_invalid")
+        app, name = raw
+        if _MIGRATION_TOKEN.fullmatch(app) is None or _MIGRATION_TOKEN.fullmatch(name) is None:
+            raise BackupRestoreError("inventory_invalid")
+        migrations.append((app, name))
+    ordered_migrations = tuple(sorted(migrations))
+    if len(set(ordered_migrations)) != len(ordered_migrations):
+        raise BackupRestoreError("inventory_invalid")
+    publication = _parse_publication_contract(raw_publication)
+    return Inventory(
+        rows=dict(sorted(rows.items())),
+        migrations=ordered_migrations,
+        publication=publication,
+    )
+
+
+def _parse_publication_contract(value: object) -> dict[str, object]:
+    if not isinstance(value, dict) or set(value) != {
+        "channel",
+        "active_revision",
+        "activations",
+    }:
+        raise BackupRestoreError("inventory_invalid")
+    raw_channel = value["channel"]
+    raw_revision = value["active_revision"]
+    raw_activations = value["activations"]
+    if not isinstance(raw_channel, dict) or set(raw_channel) != {
+        "channel",
+        "version",
+        "current_revision_id",
+    }:
+        raise BackupRestoreError("inventory_invalid")
+    if raw_channel["channel"] != "RECENT_RETAIL":
+        raise BackupRestoreError("inventory_invalid")
+    version = raw_channel["version"]
+    if (
+        not isinstance(version, int)
+        or isinstance(version, bool)
+        or version < 1
+        or version > _MAX_ACTIVATIONS
+    ):
+        raise BackupRestoreError("inventory_invalid")
+    current_revision_id = _canonical_uuid_text(raw_channel["current_revision_id"])
+
+    if not isinstance(raw_revision, dict) or set(raw_revision) != {
+        "id",
+        "typed_fact_set_sha256",
+        "generation_id",
+        "review_decision_id",
+        "review_parse_run_id",
+        "review_decision",
+        "entry_count",
+    }:
+        raise BackupRestoreError("inventory_invalid")
+    revision_id = _canonical_uuid_text(raw_revision["id"])
+    generation_id = _canonical_uuid_text(raw_revision["generation_id"])
+    review_decision_id = _canonical_uuid_text(raw_revision["review_decision_id"])
+    review_parse_run_id = _canonical_uuid_text(raw_revision["review_parse_run_id"])
+    typed_fact_set_sha256 = _canonical_sha256_text(raw_revision["typed_fact_set_sha256"])
+    entry_count = raw_revision["entry_count"]
+    if (
+        revision_id != current_revision_id
+        or generation_id != review_parse_run_id
+        or raw_revision["review_decision"] != "APPROVE"
+        or not isinstance(entry_count, int)
+        or isinstance(entry_count, bool)
+        or entry_count < 1
+        or entry_count > _MAX_PUBLICATION_ENTRIES
+    ):
+        raise BackupRestoreError("inventory_invalid")
+
+    if (
+        not isinstance(raw_activations, list)
+        or len(raw_activations) != version
+        or len(raw_activations) > _MAX_ACTIVATIONS
+    ):
+        raise BackupRestoreError("inventory_invalid")
+    activations: list[dict[str, object]] = []
+    derived_current: str | None = None
+    prior_targets: set[str] = set()
+    seen_ids: set[str] = set()
+    for expected_sequence, raw_activation in enumerate(raw_activations, start=1):
+        parsed = _parse_activation(raw_activation, expected_sequence=expected_sequence)
+        activation_id = parsed["id"]
+        operation = parsed["operation"]
+        target = parsed["target_revision_id"]
+        if (
+            not isinstance(activation_id, str)
+            or not isinstance(operation, str)
+            or (target is not None and not isinstance(target, str))
+        ):
+            raise BackupRestoreError("inventory_invalid")
+        if activation_id in seen_ids or parsed["previous_revision_id"] != derived_current:
+            raise BackupRestoreError("inventory_invalid")
+        seen_ids.add(activation_id)
+        if operation == "WITHDRAW":
+            if derived_current is None or target is not None:
+                raise BackupRestoreError("inventory_invalid")
+        else:
+            if target is None or target == derived_current:
+                raise BackupRestoreError("inventory_invalid")
+            if operation == "ROLLBACK" and target not in prior_targets:
+                raise BackupRestoreError("inventory_invalid")
+            prior_targets.add(target)
+        derived_current = target if isinstance(target, str) else None
+        activations.append(parsed)
+    if derived_current != current_revision_id:
+        raise BackupRestoreError("inventory_invalid")
+
+    return {
+        "active_revision": {
+            "entry_count": entry_count,
+            "generation_id": generation_id,
+            "id": revision_id,
+            "review_decision": "APPROVE",
+            "review_decision_id": review_decision_id,
+            "review_parse_run_id": review_parse_run_id,
+            "typed_fact_set_sha256": typed_fact_set_sha256,
+        },
+        "activations": activations,
+        "channel": {
+            "channel": "RECENT_RETAIL",
+            "current_revision_id": current_revision_id,
+            "version": version,
+        },
+    }
+
+
+def _parse_activation(value: object, *, expected_sequence: int) -> dict[str, object]:
+    if not isinstance(value, dict) or set(value) != {
+        "id",
+        "operation",
+        "sequence",
+        "previous_revision_id",
+        "target_revision_id",
+        "reason_code",
+        "acceptance_evidence_sha256",
+    }:
+        raise BackupRestoreError("inventory_invalid")
+    operation = value["operation"]
+    if not isinstance(operation, str) or operation not in {
+        "ACTIVATE",
+        "ROLLBACK",
+        "WITHDRAW",
+    }:
+        raise BackupRestoreError("inventory_invalid")
+    sequence = value["sequence"]
+    if not isinstance(sequence, int) or isinstance(sequence, bool) or sequence != expected_sequence:
+        raise BackupRestoreError("inventory_invalid")
+    reason_code = value["reason_code"]
+    if not isinstance(reason_code, str) or _REASON_CODE.fullmatch(reason_code) is None:
+        raise BackupRestoreError("inventory_invalid")
+    return {
+        "acceptance_evidence_sha256": _canonical_sha256_text(value["acceptance_evidence_sha256"]),
+        "id": _canonical_uuid_text(value["id"]),
+        "operation": operation,
+        "previous_revision_id": _nullable_uuid_text(value["previous_revision_id"]),
+        "reason_code": reason_code,
+        "sequence": expected_sequence,
+        "target_revision_id": _nullable_uuid_text(value["target_revision_id"]),
+    }
+
+
+def _canonical_uuid_text(value: object) -> str:
+    if not isinstance(value, str):
+        raise BackupRestoreError("inventory_invalid")
+    try:
+        parsed = uuid.UUID(value)
+    except ValueError:
+        raise BackupRestoreError("inventory_invalid") from None
+    if str(parsed) != value:
+        raise BackupRestoreError("inventory_invalid")
+    return value
+
+
+def _nullable_uuid_text(value: object) -> str | None:
+    return None if value is None else _canonical_uuid_text(value)
+
+
+def _canonical_sha256_text(value: object) -> str:
+    if not isinstance(value, str) or _SHA256.fullmatch(value) is None:
+        raise BackupRestoreError("inventory_invalid")
+    return value
+
+
+def _load_backup(path: Path, *, repository: Path) -> LoadedBackup:
+    backup_directory = _operator_directory(path, repository=repository)
+    _require_private_directory(backup_directory)
+    manifest_path = backup_directory / _MANIFEST_FILENAME
+    dump_path = backup_directory / _DUMP_FILENAME
+    _require_private_regular_file(manifest_path, maximum_bytes=_MAX_MANIFEST_BYTES)
+    _require_private_regular_file(dump_path, maximum_bytes=_MAX_DUMP_BYTES)
+    manifest_bytes = _read_bounded(manifest_path, _MAX_MANIFEST_BYTES)
+    try:
+        manifest = json.loads(manifest_bytes.decode("utf-8", errors="strict"))
+    except UnicodeError, json.JSONDecodeError:
+        raise BackupRestoreError("backup_manifest_invalid") from None
+    if not isinstance(manifest, dict) or set(manifest) != {
+        "backup_id",
+        "created_at",
+        "dump",
+        "format_version",
+        "inventory",
+        "postgres_major",
+        "source_database",
+    }:
+        raise BackupRestoreError("backup_manifest_invalid")
+    if (
+        manifest["format_version"] != _FORMAT_VERSION
+        or manifest["postgres_major"] != _POSTGRES_MAJOR
+        or manifest["source_database"] != _SOURCE_DATABASE
+        or not isinstance(manifest["created_at"], str)
+    ):
+        raise BackupRestoreError("backup_manifest_invalid")
+    try:
+        backup_id = uuid.UUID(manifest["backup_id"])
+    except ValueError, AttributeError, TypeError:
+        raise BackupRestoreError("backup_id_invalid") from None
+    if str(backup_id) != manifest["backup_id"]:
+        raise BackupRestoreError("backup_id_invalid")
+
+    dump = manifest["dump"]
+    inventory_value = manifest["inventory"]
+    if not isinstance(dump, dict) or set(dump) != {"bytes", "filename", "sha256"}:
+        raise BackupRestoreError("backup_manifest_invalid")
+    if (
+        dump["filename"] != _DUMP_FILENAME
+        or not isinstance(dump["bytes"], int)
+        or isinstance(dump["bytes"], bool)
+        or dump["bytes"] < len(_DUMP_MAGIC)
+        or dump["bytes"] > _MAX_DUMP_BYTES
+        or not isinstance(dump["sha256"], str)
+        or _SHA256.fullmatch(dump["sha256"]) is None
+    ):
+        raise BackupRestoreError("backup_manifest_invalid")
+    dump_sha256, dump_bytes = _file_sha256(dump_path)
+    if dump_sha256 != dump["sha256"] or dump_bytes != dump["bytes"]:
+        raise BackupRestoreError("checksum_mismatch")
+    if _read_prefix(dump_path, len(_DUMP_MAGIC)) != _DUMP_MAGIC:
+        raise BackupRestoreError("backup_not_custom_format")
+
+    if not isinstance(inventory_value, dict) or set(inventory_value) != {
+        "migrations",
+        "publication",
+        "publication_sha256",
+        "rows",
+        "sha256",
+    }:
+        raise BackupRestoreError("backup_manifest_invalid")
+    inventory = _parse_inventory(
+        {
+            "migrations": inventory_value["migrations"],
+            "publication": inventory_value["publication"],
+            "rows": inventory_value["rows"],
+        }
+    )
+    publication_sha = inventory_value["publication_sha256"]
+    if (
+        not isinstance(publication_sha, str)
+        or _SHA256.fullmatch(publication_sha) is None
+        or inventory.publication_sha256 != publication_sha
+    ):
+        raise BackupRestoreError("publication_contract_mismatch")
+    inventory_sha = inventory_value["sha256"]
+    if (
+        not isinstance(inventory_sha, str)
+        or _SHA256.fullmatch(inventory_sha) is None
+        or inventory.sha256 != inventory_sha
+    ):
+        raise BackupRestoreError("checksum_mismatch")
+    return LoadedBackup(backup_id=backup_id, dump_path=dump_path, inventory=inventory)
+
+
+def _capture_compose(
+    repository: Path,
+    tool_arguments: tuple[str, ...],
+    *,
+    timeout: int,
+    failure_code: str,
+) -> bytes:
+    completed = _run_compose(
+        repository,
+        tool_arguments,
+        stdin=subprocess.DEVNULL,
+        stdout=subprocess.PIPE,
+        timeout=timeout,
+        failure_code=failure_code,
+    )
+    output = completed.stdout
+    if not isinstance(output, bytes) or len(output) > _MAX_INVENTORY_BYTES:
+        raise BackupRestoreError(failure_code)
+    return output
+
+
+def _run_compose(
+    repository: Path,
+    tool_arguments: tuple[str, ...],
+    *,
+    stdin: int,
+    stdout: int,
+    timeout: int,
+    failure_code: str,
+) -> subprocess.CompletedProcess[bytes]:
+    docker = shutil.which("docker")
+    if docker is None:
+        raise BackupRestoreError("docker_unavailable")
+    command = (
+        docker,
+        "compose",
+        "exec",
+        "-T",
+        _COMPOSE_SERVICE,
+        *tool_arguments,
+    )
+    try:
+        completed = subprocess.run(  # noqa: S603 - fixed docker + validated tool arguments.
+            command,
+            cwd=repository,
+            env=_docker_environment(),
+            stdin=stdin,
+            stdout=stdout,
+            stderr=subprocess.DEVNULL,
+            check=False,
+            timeout=timeout,
+        )
+    except OSError, subprocess.SubprocessError:
+        raise BackupRestoreError(failure_code) from None
+    if completed.returncode != 0:
+        raise BackupRestoreError(failure_code)
+    return completed
+
+
+def _docker_environment() -> dict[str, str]:
+    allowed = ("DOCKER_CONTEXT", "DOCKER_HOST", "HOME", "PATH")
+    environment = {key: os.environ[key] for key in allowed if key in os.environ}
+    environment.update({"LANG": "C", "LC_ALL": "C"})
+    return environment
+
+
+@contextmanager
+def _restrictive_umask() -> Iterator[None]:
+    previous = os.umask(0o077)
+    try:
+        yield
+    finally:
+        os.umask(previous)
+
+
+def _create_private_file(path: Path) -> int:
+    flags = os.O_WRONLY | os.O_CREAT | os.O_EXCL | getattr(os, "O_CLOEXEC", 0)
+    try:
+        return os.open(path, flags, 0o600)
+    except OSError:
+        raise BackupRestoreError("dump_file_invalid") from None
+
+
+def _write_private_file(path: Path, content: bytes) -> None:
+    descriptor = _create_private_file(path)
+    try:
+        written = 0
+        while written < len(content):
+            chunk_size = os.write(descriptor, content[written:])
+            if chunk_size < 1:
+                raise OSError
+            written += chunk_size
+        os.fsync(descriptor)
+    except OSError:
+        raise BackupRestoreError("backup_directory_unavailable") from None
+    finally:
+        os.close(descriptor)
+
+
+def _require_private_directory(path: Path) -> None:
+    try:
+        metadata = path.lstat()
+    except OSError:
+        raise BackupRestoreError("backup_directory_invalid") from None
+    if (
+        stat.S_ISLNK(metadata.st_mode)
+        or not stat.S_ISDIR(metadata.st_mode)
+        or metadata.st_uid != os.geteuid()
+        or stat.S_IMODE(metadata.st_mode) & 0o077
+    ):
+        raise BackupRestoreError("backup_directory_permissions")
+
+
+def _require_private_regular_file(path: Path, *, maximum_bytes: int) -> None:
+    try:
+        metadata = path.lstat()
+    except OSError:
+        raise BackupRestoreError("dump_file_invalid") from None
+    if (
+        stat.S_ISLNK(metadata.st_mode)
+        or not stat.S_ISREG(metadata.st_mode)
+        or metadata.st_uid != os.geteuid()
+        or stat.S_IMODE(metadata.st_mode) & 0o077
+    ):
+        raise BackupRestoreError("backup_directory_permissions")
+    if metadata.st_size < 1 or metadata.st_size > maximum_bytes:
+        raise BackupRestoreError("backup_too_large")
+
+
+def _read_bounded(path: Path, maximum_bytes: int) -> bytes:
+    descriptor = os.open(
+        path,
+        os.O_RDONLY | getattr(os, "O_CLOEXEC", 0) | getattr(os, "O_NOFOLLOW", 0),
+    )
+    try:
+        data = os.read(descriptor, maximum_bytes + 1)
+    except OSError:
+        raise BackupRestoreError("dump_file_invalid") from None
+    finally:
+        os.close(descriptor)
+    if len(data) > maximum_bytes:
+        raise BackupRestoreError("backup_too_large")
+    return data
+
+
+def _read_prefix(path: Path, length: int) -> bytes:
+    descriptor = os.open(
+        path,
+        os.O_RDONLY | getattr(os, "O_CLOEXEC", 0) | getattr(os, "O_NOFOLLOW", 0),
+    )
+    try:
+        return os.read(descriptor, length)
+    finally:
+        os.close(descriptor)
+
+
+def _file_sha256(path: Path) -> tuple[str, int]:
+    digest = hashlib.sha256()
+    total = 0
+    descriptor = os.open(
+        path,
+        os.O_RDONLY | getattr(os, "O_CLOEXEC", 0) | getattr(os, "O_NOFOLLOW", 0),
+    )
+    try:
+        while True:
+            chunk = os.read(descriptor, 1024 * 1024)
+            if not chunk:
+                break
+            total += len(chunk)
+            if total > _MAX_DUMP_BYTES:
+                raise BackupRestoreError("backup_too_large")
+            digest.update(chunk)
+    except OSError:
+        raise BackupRestoreError("dump_file_invalid") from None
+    finally:
+        os.close(descriptor)
+    return digest.hexdigest(), total
+
+
+def _fsync_directory(path: Path) -> None:
+    descriptor = os.open(path, os.O_RDONLY | getattr(os, "O_CLOEXEC", 0))
+    try:
+        os.fsync(descriptor)
+    except OSError:
+        raise BackupRestoreError("backup_directory_unavailable") from None
+    finally:
+        os.close(descriptor)
+
+
+def _canonical_json(value: object) -> bytes:
+    return json.dumps(
+        value,
+        ensure_ascii=True,
+        sort_keys=True,
+        separators=(",", ":"),
+    ).encode("ascii")
+
+
+def _is_within(candidate: Path, parent: Path) -> bool:
+    try:
+        candidate.relative_to(parent)
+    except ValueError:
+        return False
+    return True
+
+
+if __name__ == "__main__":
+    raise SystemExit(main())
diff --git a/scripts/tests/test_postgres_backup_restore.py b/scripts/tests/test_postgres_backup_restore.py
new file mode 100644
index 0000000..3afece4
--- /dev/null
+++ b/scripts/tests/test_postgres_backup_restore.py
@@ -0,0 +1,722 @@
+"""Unit tests for bounded local PostgreSQL backup and restore assurance."""
+
+from __future__ import annotations
+
+import json
+import os
+import stat
+from copy import deepcopy
+from pathlib import Path
+from unittest.mock import call, patch
+
+import pytest
+
+from scripts.postgres_backup_restore import (
+    _INVENTORY_SQL,
+    BackupReceipt,
+    BackupRestoreError,
+    Inventory,
+    _database_exists,
+    _docker_environment,
+    _parse_inventory,
+    _preflight,
+    _validated_target_database,
+    create_backup,
+    main,
+    restore_backup,
+)
+
+_REVISION_V1 = "11111111-1111-4111-8111-111111111111"
+_REVISION_V2 = "22222222-2222-4222-8222-222222222222"
+_GENERATION_ID = "33333333-3333-4333-8333-333333333333"
+_REVIEW_ID = "44444444-4444-4444-8444-444444444444"
+_ACTIVATION_V1 = "55555555-5555-4555-8555-555555555555"
+_ACTIVATION_V2 = "66666666-6666-4666-8666-666666666666"
+_FACT_SET_HASH = "a" * 64
+_EVIDENCE_HASH = "8" * 64
+
+
+def _publication_contract() -> dict[str, object]:
+    return {
+        "active_revision": {
+            "entry_count": 10,
+            "generation_id": _GENERATION_ID,
+            "id": _REVISION_V2,
+            "review_decision": "APPROVE",
+            "review_decision_id": _REVIEW_ID,
+            "review_parse_run_id": _GENERATION_ID,
+            "typed_fact_set_sha256": _FACT_SET_HASH,
+        },
+        "activations": [
+            {
+                "acceptance_evidence_sha256": _EVIDENCE_HASH,
+                "id": _ACTIVATION_V1,
+                "operation": "ACTIVATE",
+                "previous_revision_id": None,
+                "reason_code": "LOCAL_PHASE0_PUBLICATION_ACTIVATED",
+                "sequence": 1,
+                "target_revision_id": _REVISION_V1,
+            },
+            {
+                "acceptance_evidence_sha256": _EVIDENCE_HASH,
+                "id": _ACTIVATION_V2,
+                "operation": "ACTIVATE",
+                "previous_revision_id": _REVISION_V1,
+                "reason_code": "LOCAL_PHASE0_PUBLICATION_ACTIVATED",
+                "sequence": 2,
+                "target_revision_id": _REVISION_V2,
+            },
+        ],
+        "channel": {
+            "channel": "RECENT_RETAIL",
+            "current_revision_id": _REVISION_V2,
+            "version": 2,
+        },
+    }
+
+
+@pytest.fixture
+def repository(tmp_path: Path) -> Path:
+    root = tmp_path / "repository"
+    root.mkdir()
+    (root / "compose.yaml").write_text("services: {}\n", encoding="utf-8")
+    return root
+
+
+@pytest.fixture
+def destination(tmp_path: Path) -> Path:
+    root = tmp_path / "backups"
+    root.mkdir()
+    return root
+
+
+@pytest.fixture
+def inventory() -> Inventory:
+    return Inventory(
+        rows={
+            "public.auth_user": 1,
+            "public.django_migrations": 17,
+            "public.grocery_publicationrevision": 2,
+        },
+        migrations=(
+            ("contenttypes", "0001_initial"),
+            ("grocery", "0008_publication_activation"),
+        ),
+        publication=_publication_contract(),
+    )
+
+
+def _synthetic_dump(_repository: Path, descriptor: int) -> None:
+    os.write(descriptor, b"PGDMPsynthetic-custom-format")
+
+
+def _make_backup(
+    repository: Path,
+    destination: Path,
+    inventory: Inventory,
+) -> tuple[Path, BackupReceipt]:
+    with (
+        patch("scripts.postgres_backup_restore._preflight"),
+        patch(
+            "scripts.postgres_backup_restore._read_inventory",
+            side_effect=(inventory, inventory),
+        ),
+        patch(
+            "scripts.postgres_backup_restore._dump_database",
+            side_effect=_synthetic_dump,
+        ),
+    ):
+        receipt = create_backup(repository=repository, destination_root=destination)
+    backup_directory = next(destination.iterdir())
+    return backup_directory, receipt
+
+
+def test_backup_creates_private_custom_dump_and_secret_free_checksum_manifest(
+    repository: Path,
+    destination: Path,
+    inventory: Inventory,
+) -> None:
+    backup_directory, receipt = _make_backup(repository, destination, inventory)
+
+    dump = backup_directory / "database.dump"
+    manifest_path = backup_directory / "manifest.json"
+    manifest = json.loads(manifest_path.read_text(encoding="utf-8"))
+    assert stat.S_IMODE(backup_directory.stat().st_mode) == 0o700
+    assert stat.S_IMODE(dump.stat().st_mode) == 0o600
+    assert stat.S_IMODE(manifest_path.stat().st_mode) == 0o600
+    assert dump.read_bytes().startswith(b"PGDMP")
+    assert manifest["format_version"] == "grocery-postgres-custom-v1"
+    assert manifest["postgres_major"] == 18
+    assert manifest["source_database"] == "grocery"
+    assert manifest["inventory"]["sha256"] == inventory.sha256
+    assert manifest["inventory"]["publication"] == inventory.publication
+    assert manifest["inventory"]["publication_sha256"] == inventory.publication_sha256
+    assert manifest["dump"]["sha256"] == receipt.dump_sha256
+    serialized = manifest_path.read_text(encoding="utf-8")
+    assert "DATABASE_URL" not in serialized
+    assert "password" not in serialized.lower()
+    assert "local-grocery-only" not in serialized
+    rendered = receipt.render()
+    assert str(destination) not in rendered
+    assert "cleanup=retain_or_remove_explicit_backup_directory" in rendered
+    for internal_value in (
+        _REVISION_V2,
+        _GENERATION_ID,
+        _REVIEW_ID,
+        _ACTIVATION_V1,
+        _FACT_SET_HASH,
+        _EVIDENCE_HASH,
+        "LOCAL_PHASE0_PUBLICATION_ACTIVATED",
+    ):
+        assert internal_value not in rendered
+
+
+def test_backup_reads_inventory_before_and_after_dump(
+    repository: Path,
+    destination: Path,
+    inventory: Inventory,
+) -> None:
+    with (
+        patch("scripts.postgres_backup_restore._preflight"),
+        patch(
+            "scripts.postgres_backup_restore._read_inventory",
+            side_effect=(inventory, inventory),
+        ) as read_inventory,
+        patch(
+            "scripts.postgres_backup_restore._dump_database",
+            side_effect=_synthetic_dump,
+        ),
+    ):
+        create_backup(repository=repository, destination_root=destination)
+
+    assert read_inventory.call_args_list == [
+        call(repository.resolve(), "grocery"),
+        call(repository.resolve(), "grocery"),
+    ]
+
+
+def test_backup_fails_if_inventory_changes_during_dump(
+    repository: Path,
+    destination: Path,
+    inventory: Inventory,
+) -> None:
+    changed = Inventory(
+        rows={**inventory.rows, "public.auth_user": 2},
+        migrations=inventory.migrations,
+        publication=inventory.publication,
+    )
+    with (
+        patch("scripts.postgres_backup_restore._preflight"),
+        patch(
+            "scripts.postgres_backup_restore._read_inventory",
+            side_effect=(inventory, changed),
+        ),
+        patch(
+            "scripts.postgres_backup_restore._dump_database",
+            side_effect=_synthetic_dump,
+        ),
+        pytest.raises(BackupRestoreError) as caught,
+    ):
+        create_backup(repository=repository, destination_root=destination)
+
+    assert caught.value.code == "backup_changed_during_dump"
+
+
+def test_backup_rejects_relative_or_repository_destination(
+    repository: Path,
+) -> None:
+    with pytest.raises(BackupRestoreError) as relative:
+        create_backup(repository=repository, destination_root=Path("relative"))
+    assert relative.value.code == "backup_directory_invalid"
+
+    child = repository / "backups"
+    child.mkdir()
+    with pytest.raises(BackupRestoreError) as inside:
+        create_backup(repository=repository, destination_root=child)
+    assert inside.value.code == "destination_inside_repository"
+
+
+def test_restore_validates_then_creates_separate_target_and_compares_inventory(
+    repository: Path,
+    destination: Path,
+    inventory: Inventory,
+) -> None:
+    backup_directory, backup_receipt = _make_backup(repository, destination, inventory)
+    with (
+        patch("scripts.postgres_backup_restore._preflight") as preflight,
+        patch("scripts.postgres_backup_restore._validate_custom_dump") as validate_dump,
+        patch("scripts.postgres_backup_restore._database_exists", return_value=False) as exists,
+        patch("scripts.postgres_backup_restore._create_target_database") as create_target,
+        patch("scripts.postgres_backup_restore._restore_database") as restore,
+        patch("scripts.postgres_backup_restore._read_inventory", return_value=inventory),
+    ):
+        receipt = restore_backup(
+            repository=repository,
+            backup_directory=backup_directory,
+            target_database="grocery_restore_rehearsal1",
+        )
+
+    preflight.assert_called_once_with(repository.resolve())
+    validate_dump.assert_called_once_with(repository.resolve(), backup_directory / "database.dump")
+    exists.assert_called_once_with(repository.resolve(), "grocery_restore_rehearsal1")
+    create_target.assert_called_once_with(repository.resolve(), "grocery_restore_rehearsal1")
+    restore.assert_called_once_with(
+        repository.resolve(),
+        backup_directory / "database.dump",
+        "grocery_restore_rehearsal1",
+    )
+    assert receipt.backup_id == backup_receipt.backup_id
+    assert "row_counts_consistent=yes" in receipt.render()
+    assert "migrations_consistent=yes" in receipt.render()
+    assert "publication_contract_consistent=yes" in receipt.render()
+    assert "grocery_restore_rehearsal1" not in receipt.render()
+    for internal_value in (
+        _REVISION_V2,
+        _GENERATION_ID,
+        _REVIEW_ID,
+        _ACTIVATION_V1,
+        _FACT_SET_HASH,
+        _EVIDENCE_HASH,
+        "LOCAL_PHASE0_PUBLICATION_ACTIVATED",
+    ):
+        assert internal_value not in receipt.render()
+
+
+def test_restore_refuses_source_invalid_or_existing_target_before_create(
+    repository: Path,
+    destination: Path,
+    inventory: Inventory,
+) -> None:
+    backup_directory, _receipt = _make_backup(repository, destination, inventory)
+    with pytest.raises(BackupRestoreError) as source:
+        restore_backup(
+            repository=repository,
+            backup_directory=backup_directory,
+            target_database="grocery",
+        )
+    assert source.value.code == "target_database_is_source"
+
+    with pytest.raises(BackupRestoreError) as invalid:
+        restore_backup(
+            repository=repository,
+            backup_directory=backup_directory,
+            target_database="production",
+        )
+    assert invalid.value.code == "target_database_invalid"
+
+    with (
+        patch("scripts.postgres_backup_restore._preflight"),
+        patch("scripts.postgres_backup_restore._validate_custom_dump"),
+        patch("scripts.postgres_backup_restore._database_exists", return_value=True),
+        patch("scripts.postgres_backup_restore._create_target_database") as create_target,
+        pytest.raises(BackupRestoreError) as existing,
+    ):
+        restore_backup(
+            repository=repository,
+            backup_directory=backup_directory,
+            target_database="grocery_restore_existing",
+        )
+    assert existing.value.code == "target_database_exists"
+    create_target.assert_not_called()
+
+
+@pytest.mark.parametrize(
+    ("restored", "expected_code"),
+    (
+        (
+            Inventory(
+                rows={"public.django_migrations": 99},
+                migrations=(("grocery", "0008_publication_activation"),),
+                publication=_publication_contract(),
+            ),
+            "row_count_mismatch",
+        ),
+        (
+            Inventory(
+                rows={
+                    "public.auth_user": 1,
+                    "public.django_migrations": 17,
+                    "public.grocery_publicationrevision": 2,
+                },
+                migrations=(("grocery", "0007_publication_revision"),),
+                publication=_publication_contract(),
+            ),
+            "migration_mismatch",
+        ),
+    ),
+)
+def test_restore_fails_closed_on_row_or_migration_drift(
+    repository: Path,
+    destination: Path,
+    inventory: Inventory,
+    restored: Inventory,
+    expected_code: str,
+) -> None:
+    backup_directory, _receipt = _make_backup(repository, destination, inventory)
+    with (
+        patch("scripts.postgres_backup_restore._preflight"),
+        patch("scripts.postgres_backup_restore._validate_custom_dump"),
+        patch("scripts.postgres_backup_restore._database_exists", return_value=False),
+        patch("scripts.postgres_backup_restore._create_target_database"),
+        patch("scripts.postgres_backup_restore._restore_database"),
+        patch("scripts.postgres_backup_restore._read_inventory", return_value=restored),
+        pytest.raises(BackupRestoreError) as caught,
+    ):
+        restore_backup(
+            repository=repository,
+            backup_directory=backup_directory,
+            target_database="grocery_restore_drift",
+        )
+
+    assert caught.value.code == expected_code
+
+
+def test_restore_fails_closed_on_publication_contract_drift(
+    repository: Path,
+    destination: Path,
+    inventory: Inventory,
+) -> None:
+    changed_contract = deepcopy(inventory.publication)
+    active_revision = changed_contract["active_revision"]
+    assert isinstance(active_revision, dict)
+    active_revision["typed_fact_set_sha256"] = "b" * 64
+    restored = Inventory(
+        rows=inventory.rows,
+        migrations=inventory.migrations,
+        publication=changed_contract,
+    )
+    backup_directory, _receipt = _make_backup(repository, destination, inventory)
+    with (
+        patch("scripts.postgres_backup_restore._preflight"),
+        patch("scripts.postgres_backup_restore._validate_custom_dump"),
+        patch("scripts.postgres_backup_restore._database_exists", return_value=False),
+        patch("scripts.postgres_backup_restore._create_target_database"),
+        patch("scripts.postgres_backup_restore._restore_database"),
+        patch("scripts.postgres_backup_restore._read_inventory", return_value=restored),
+        pytest.raises(BackupRestoreError) as caught,
+    ):
+        restore_backup(
+            repository=repository,
+            backup_directory=backup_directory,
+            target_database="grocery_restore_publication_drift",
+        )
+
+    assert caught.value.code == "publication_contract_mismatch"
+
+
+def test_restore_rejects_checksum_tamper_before_preflight(
+    repository: Path,
+    destination: Path,
+    inventory: Inventory,
+) -> None:
+    backup_directory, _receipt = _make_backup(repository, destination, inventory)
+    dump_path = backup_directory / "database.dump"
+    dump_path.write_bytes(dump_path.read_bytes() + b"tampered")
+    os.chmod(dump_path, 0o600)
+
+    with patch("scripts.postgres_backup_restore._preflight") as preflight:
+        with pytest.raises(BackupRestoreError) as caught:
+            restore_backup(
+                repository=repository,
+                backup_directory=backup_directory,
+                target_database="grocery_restore_tamper",
+            )
+
+    assert caught.value.code == "checksum_mismatch"
+    preflight.assert_not_called()
+
+
+def test_restore_rejects_publication_contract_hash_tamper_before_preflight(
+    repository: Path,
+    destination: Path,
+    inventory: Inventory,
+) -> None:
+    backup_directory, _receipt = _make_backup(repository, destination, inventory)
+    manifest_path = backup_directory / "manifest.json"
+    manifest = json.loads(manifest_path.read_text(encoding="utf-8"))
+    manifest["inventory"]["publication_sha256"] = "f" * 64
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
+            target_database="grocery_restore_contract_hash_tamper",
+        )
+
+    assert caught.value.code == "publication_contract_mismatch"
+    preflight.assert_not_called()
+
+
+def test_restore_rejects_broad_backup_permissions(
+    repository: Path,
+    destination: Path,
+    inventory: Inventory,
+) -> None:
+    backup_directory, _receipt = _make_backup(repository, destination, inventory)
+    os.chmod(backup_directory / "manifest.json", 0o644)
+
+    with pytest.raises(BackupRestoreError) as caught:
+        restore_backup(
+            repository=repository,
+            backup_directory=backup_directory,
+            target_database="grocery_restore_permissions",
+        )
+
+    assert caught.value.code == "backup_directory_permissions"
+
+
+def test_inventory_parser_is_bounded_and_canonical(inventory: Inventory) -> None:
+    parsed = _parse_inventory(inventory.canonical_data())
+    assert parsed == inventory
+    assert len(parsed.sha256) == 64
+    assert len(parsed.publication_sha256) == 64
+
+    with pytest.raises(BackupRestoreError) as caught:
+        _parse_inventory({"rows": {"private.table": 1}, "migrations": [["app", "0001"]]})
+    assert caught.value.code == "inventory_invalid"
+
+
+def test_publication_contract_parser_rejects_invalid_shapes_and_chain(
+    inventory: Inventory,
+) -> None:
+    invalid_contracts: list[dict[str, object]] = []
+
+    extra_publisher = deepcopy(inventory.publication)
+    activations = extra_publisher["activations"]
+    assert isinstance(activations, list)
+    assert isinstance(activations[0], dict)
+    activations[0]["publisher_id"] = 7
+    invalid_contracts.append(extra_publisher)
+
+    boolean_sequence = deepcopy(inventory.publication)
+    activations = boolean_sequence["activations"]
+    assert isinstance(activations, list)
+    assert isinstance(activations[0], dict)
+    activations[0]["sequence"] = True
+    invalid_contracts.append(boolean_sequence)
+
+    broken_pointer = deepcopy(inventory.publication)
+    activations = broken_pointer["activations"]
+    assert isinstance(activations, list)
+    assert isinstance(activations[1], dict)
+    activations[1]["previous_revision_id"] = None
+    invalid_contracts.append(broken_pointer)
+
+    noncanonical_uuid = deepcopy(inventory.publication)
+    channel = noncanonical_uuid["channel"]
+    assert isinstance(channel, dict)
+    channel["current_revision_id"] = "AAAAAAAA-AAAA-4AAA-8AAA-AAAAAAAAAAAA"
+    invalid_contracts.append(noncanonical_uuid)
+
+    invalid_hash = deepcopy(inventory.publication)
+    active_revision = invalid_hash["active_revision"]
+    assert isinstance(active_revision, dict)
+    active_revision["typed_fact_set_sha256"] = "a" * 63
+    invalid_contracts.append(invalid_hash)
+
+    version_mismatch = deepcopy(inventory.publication)
+    channel = version_mismatch["channel"]
+    assert isinstance(channel, dict)
+    channel["version"] = 3
+    invalid_contracts.append(version_mismatch)
+
+    entry_count_out_of_bounds = deepcopy(inventory.publication)
+    active_revision = entry_count_out_of_bounds["active_revision"]
+    assert isinstance(active_revision, dict)
+    active_revision["entry_count"] = 100_001
+    invalid_contracts.append(entry_count_out_of_bounds)
+
+    for invalid_contract in invalid_contracts:
+        invalid_inventory = inventory.canonical_data()
+        invalid_inventory["publication"] = invalid_contract
+        with pytest.raises(BackupRestoreError) as caught:
+            _parse_inventory(invalid_inventory)
+        assert caught.value.code == "inventory_invalid"
+
+
+def test_publication_contract_rejects_rollback_to_never_active_revision(
+    inventory: Inventory,
+) -> None:
+    invalid_inventory = inventory.canonical_data()
+    contract = deepcopy(inventory.publication)
+    activations = contract["activations"]
+    channel = contract["channel"]
+    active_revision = contract["active_revision"]
+    assert isinstance(activations, list)
+    assert isinstance(activations[1], dict)
+    assert isinstance(channel, dict)
+    assert isinstance(active_revision, dict)
+    never_active = "77777777-7777-4777-8777-777777777777"
+    activations[1]["operation"] = "ROLLBACK"
+    activations[1]["target_revision_id"] = never_active
+    channel["current_revision_id"] = never_active
+    active_revision["id"] = never_active
+    invalid_inventory["publication"] = contract
+
+    with pytest.raises(BackupRestoreError) as caught:
+        _parse_inventory(invalid_inventory)
+
+    assert caught.value.code == "inventory_invalid"
+
+
+def test_publication_inventory_sql_is_ordered_and_omits_publisher_identity() -> None:
+    assert "ORDER BY activation.sequence" in _INVENTORY_SQL
+    assert "replacement.supersedes_id = decision.id" in _INVENTORY_SQL
+    assert "generation.status = 'VALIDATED'" in _INVENTORY_SQL
+    assert "generation.accepted_row_count = revision.entry_count" in _INVENTORY_SQL
+    assert "entry.revision_id = revision.id" in _INVENTORY_SQL
+    assert "publisher" not in _INVENTORY_SQL
+
+
+def test_docker_environment_drops_database_and_password_values(
+    monkeypatch: pytest.MonkeyPatch,
+) -> None:
+    monkeypatch.setenv("DATABASE_URL", "private-database-url")
+    monkeypatch.setenv("PGPASSWORD", "private-password")
+    monkeypatch.setenv("POSTGRES_PASSWORD", "private-postgres-password")
+    monkeypatch.setenv("DOCKER_HOST", "unix:///safe/docker.sock")
+
+    environment = _docker_environment()
+
+    assert environment["DOCKER_HOST"] == "unix:///safe/docker.sock"
+    assert "DATABASE_URL" not in environment
+    assert "PGPASSWORD" not in environment
+    assert "POSTGRES_PASSWORD" not in environment
+    assert "private" not in str(environment)
+
+
+def test_preflight_requires_all_postgres_18_tools_and_source_connectivity(
+    repository: Path,
+) -> None:
+    versions = (
+        b"pg_dump (PostgreSQL) 18.6 (Debian 18.6-1)\n",
+        b"pg_restore (PostgreSQL) 18.6 (Debian 18.6-1)\n",
+        b"createdb (PostgreSQL) 18.6 (Debian 18.6-1)\n",
+        b"psql (PostgreSQL) 18.6 (Debian 18.6-1)\n",
+        b"1\n",
+    )
+    with (
+        patch("scripts.postgres_backup_restore.shutil.which", return_value="/safe/docker"),
+        patch(
+            "scripts.postgres_backup_restore._capture_compose",
+            side_effect=versions,
+        ) as capture,
+    ):
+        _preflight(repository)
+
+    assert [entry.args[1][0] for entry in capture.call_args_list] == [
+        "pg_dump",
+        "pg_restore",
+        "createdb",
+        "psql",
+        "psql",
+    ]
+
+
+def test_preflight_rejects_missing_or_wrong_major_tool(repository: Path) -> None:
+    with (
+        patch("scripts.postgres_backup_restore.shutil.which", return_value=None),
+        pytest.raises(BackupRestoreError) as missing,
+    ):
+        _preflight(repository)
+    assert missing.value.code == "docker_unavailable"
+
+    with (
+        patch("scripts.postgres_backup_restore.shutil.which", return_value="/safe/docker"),
+        patch(
+            "scripts.postgres_backup_restore._capture_compose",
+            return_value=b"pg_dump (PostgreSQL) 17.9\n",
+        ),
+        pytest.raises(BackupRestoreError) as wrong_version,
+    ):
+        _preflight(repository)
+    assert wrong_version.value.code == "postgres_version_mismatch"
+
+
+def test_cli_failure_never_reflects_path_target_or_arbitrary_exception(
+    capsys: pytest.CaptureFixture[str],
+) -> None:
+    path_marker = "/private/operator/path-marker"
+    with patch(
+        "scripts.postgres_backup_restore.create_backup",
+        side_effect=RuntimeError("private-database-password-marker"),
+    ):
+        result = main(["backup", "--output-dir", path_marker])
+    output = capsys.readouterr()
+    assert result == 1
+    assert output.out == "\n".join(
+        (
+            "status=failed",
+            "code=internal_error",
+            "cleanup=remove_incomplete_backup_directory_if_created",
+            "",
+        )
+    )
+    assert "private" not in output.out
+    assert output.err == ""
+
+    target_marker = "grocery_restore_private_marker"
+    with patch(
+        "scripts.postgres_backup_restore.restore_backup",
+        side_effect=RuntimeError("private-database-password-marker"),
+    ):
+        result = main(
+            [
+                "restore",
+                "--backup-dir",
+                path_marker,
+                "--target-database",
+                target_marker,
+            ]
+        )
+    output = capsys.readouterr()
+    assert result == 1
+    assert "code=internal_error" in output.out
+    assert "cleanup=inspect_and_drop_explicit_restore_target_if_created" in output.out
+    assert "private" not in output.out
+    assert output.err == ""
+
+
+def test_cli_usage_error_does_not_reflect_unknown_argument(
+    capsys: pytest.CaptureFixture[str],
+) -> None:
+    result = main(["backup", "--private-password-marker"])
+    output = capsys.readouterr()
+
+    assert result == 2
+    assert "code=usage_error" in output.out
+    assert "private-password-marker" not in output.out
+    assert output.err == ""
+
+
+def test_target_database_contract_is_explicit_and_separate() -> None:
+    assert _validated_target_database("grocery_restore_rehearsal_20260830") == (
+        "grocery_restore_rehearsal_20260830"
+    )
+    for invalid in ("grocery", "other", "grocery_restore_", "grocery_restore_UPPER"):
+        with pytest.raises(BackupRestoreError):
+            _validated_target_database(invalid)
+
+
+def test_target_preflight_uses_validated_literal_in_command_mode(
+    repository: Path,
+) -> None:
+    target = "grocery_restore_rehearsal_20260830"
+    with patch(
+        "scripts.postgres_backup_restore._capture_compose",
+        return_value=b"0\n",
+    ) as capture:
+        assert _database_exists(repository, target) is False
+
+    arguments = capture.call_args.args[1]
+    assert not any(argument.startswith("--set=") for argument in arguments)
+    expected_command = (
+        "--command=SELECT count(*) FROM pg_database WHERE datname = "
+        "'grocery_restore_rehearsal_20260830';"
+    )
+    assert arguments[-1] == expected_command


