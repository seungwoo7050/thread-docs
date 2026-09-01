## `test(operations): rehearse exact backup restore`

diff --git a/docs/IMPLEMENTATION-PLAN.md b/docs/IMPLEMENTATION-PLAN.md
index 4c91359..34b97ca 100644
--- a/docs/IMPLEMENTATION-PLAN.md
+++ b/docs/IMPLEMENTATION-PLAN.md
@@ -649,7 +649,8 @@ allowlist합니다.
 - 목적: exact-version `pg_dump` custom backup, empty database restore/check와 application-level warning 및
   entry rollback 절차를 실행 가능한 local runbook으로 고정합니다.
 - 의존성: atom 10~15
-- 예상 파일: `scripts/backup.sh`, `scripts/restore-check.sh`, `docs/OPERATIONS-RUNBOOK.md`, tests
+- 예상 파일: `scripts/backup-postgresql`, `scripts/restore-postgresql`,
+  `scripts/check-backup-restore`, shared integrity contract, `docs/OPERATIONS-RUNBOOK.md`, tests
 - 검증: disposable DB에 actual backup restore, row/hash/pointer 비교, RPO 24h/RTO 4h 측정,
   backup manifest secret/raw/user-input scan, restored Admin session invalidation,
   warning rollback이 entry HTML을 바꾸지 않음
diff --git a/docs/OPERATIONS-RUNBOOK.md b/docs/OPERATIONS-RUNBOOK.md
new file mode 100644
index 0000000..3b9023d
--- /dev/null
+++ b/docs/OPERATIONS-RUNBOOK.md
@@ -0,0 +1,265 @@
+# Phase 0 operations runbook
+
+This runbook covers the local PostgreSQL backup/restore acceptance boundary and
+the release-candidate checks that depend on it. It does not authorize creation
+of production infrastructure, production credentials, a domain, DNS changes,
+or deployment.
+
+## Backup contract
+
+The backup command is deliberately narrower than a whole-database dump. It
+requires PostgreSQL client and server version 18.6, verifies the exact Phase 0
+table, sequence, trigger, and trigger-function allowlists, rejects unreviewed
+storage-shaped columns and large objects, and asks `pg_dump` for only the
+`public` schema. The relation check also requires permanent tables, indexes and
+sequences, default replica identity, disabled row-level security, and each
+sequence's exact table/column ownership mapping. The function allowlist pins the
+SHA-256 of each final migration function body, and the trigger allowlist pins
+table, function, event/timing bits, row/statement scope, enabled mode, and
+deferred-constraint state. A future migration therefore fails closed until this
+contract and its tests are reviewed together.
+
+The archive contains the country and passport-policy canonical rows, source
+configuration and rights evidence, fetch/artifact/parse receipts, typed entry
+and travel-warning revisions, review decisions, immutable publications,
+current pointers, audit events, and the Django schema/authentication metadata
+needed to restore their foreign keys. It cannot include an environment file or
+an API key value because those are not database columns. Source artifacts hold
+only hashes and byte counts; raw response bodies are not a database field.
+
+`django_session` and `django_admin_log` schemas are included so the migrated
+schema remains complete, but their rows are always excluded. Their sequence
+definitions and next-value state remain part of the schema archive so inserts
+after restore cannot collide; do not describe the archive as erasing historical
+sequence state. The approved schema has no trip destination, departure date,
+return date, purpose, raw response body, credential, or key-value column. The
+only names containing `raw` are the exact rights-policy metadata fields
+`raw_body_storage_allowed` and `raw_retention_seconds`; they do not contain a
+body. The preflight rejects any other matching storage shape. PostgreSQL large
+objects are also rejected so they cannot become an unreviewed side channel.
+
+Use libpq's non-interactive credential mechanism. For example, point
+`PGPASSFILE` at a mode-0600 file supplied outside the repository. Never put a
+password in a command argument, connection URL, shell trace, receipt, or log.
+The scripts suppress provider diagnostics and return only fixed result codes.
+
+Create a new absolute output directory by running:
+
+```sh
+scripts/backup-postgresql \
+  --host DB_HOST \
+  --port 5432 \
+  --database travel_readiness \
+  --username travel_readiness_backup \
+  --backup-dir /ABSOLUTE/PRIVATE/PARENT/phase0-backup-YYYYMMDDTHHMMSSZ
+```
+
+The named backup directory must not already exist, must not be directly under
+the filesystem root, and must have a real (non-symlink) parent. Success prints
+only `backup_result=ok`. The directory is mode 0700 and contains only:
+
+- `database.dump`: PostgreSQL custom-format archive, mode 0600.
+- `integrity.manifest`: archive SHA-256 plus deterministic counts, row-set
+  SHA-256 values, and entry/travel-warning pointer identity, mode 0600.
+
+The command calculates the database integrity snapshot both before and after
+`pg_dump`; the snapshot includes every protected row set, atomic pointers,
+sequence `last_value`/`is_called` state, and a hash of the database encoding,
+locale provider, collation, ctype, locale and collation version. A restore on a
+different cluster must match that database locale profile. Any net difference fails with
+`SOURCE_CHANGED_DURING_BACKUP`. `pg_dump` itself uses a transactionally
+consistent snapshot, but the surrounding integrity reads are not the same
+exported snapshot and cannot detect a mutate-then-revert ABA in a mutable Django
+auth table. Pause source jobs, reviewer/admin writes and all other writers for
+the entire accepted backup window. The core source, review and publication
+records are append-only or immutable, but that invariant alone is not a
+substitute for the write pause.
+
+Store the completed directory in access-controlled encrypted storage. Copy the
+two files as one unit. A checksum is corruption evidence, not an authenticity
+signature; accept backup directories only from the controlled backup channel.
+
+The Phase 0 local policy target is **RPO 24 hours** and **RTO 4 hours**
+(`RPO 24시간`, `RTO 4시간`). Take at
+least one accepted backup in every rolling 24-hour interval and retain enough
+accepted generations to meet the selected retention policy. The four-hour RTO
+timer covers obtaining the archive, creating the disposable target, restore and
+integrity verification, server-rendered marker verification, and the separately
+authorized service cutover or rollback decision.
+
+Every backup failure exits nonzero and emits exactly one fixed
+`backup_result=...` receipt. The scheduler must treat any nonzero exit, a missing
+`backup_result=ok`, or the absence of an accepted backup within 24 hours as an
+alert. Emit only the fixed result code, release identity, and timestamp to
+monitoring; do not attach command lines, stderr from PostgreSQL, database names,
+paths, URLs, hashes, manifest contents, or credentials. Alert delivery and
+acknowledgement must be exercised before release.
+
+## Disposable restore rehearsal
+
+The restore script never creates, drops, cleans, or overwrites a database. An
+operator must first create a separate empty database from `template0`. Its exact
+name must start with `travel_readiness_restore_`; do not reuse an application or
+production database name.
+
+Example creation step for a local rehearsal:
+
+```sh
+createdb \
+  --host DB_HOST \
+  --port 5432 \
+  --username travel_readiness_restore_operator \
+  --template=template0 \
+  travel_readiness_restore_YYYYMMDD
+```
+
+PostgreSQL 18 creates an empty `public` schema even from `template0`, while the
+custom archive contains the reviewed `public` schema definition. On this
+explicitly disposable database only, remove that still-empty schema before the
+restore:
+
+```sh
+psql \
+  --host DB_HOST \
+  --port 5432 \
+  --username travel_readiness_restore_operator \
+  --dbname travel_readiness_restore_YYYYMMDD \
+  --command='DROP SCHEMA public'
+```
+
+The restore preflight requires the `public` schema to be absent and rejects a
+target that contains it as `TARGET_NOT_EMPTY`; the restore script never removes
+the schema itself.
+
+Then restore with a target-bound safety token:
+
+```sh
+scripts/restore-postgresql \
+  --host DB_HOST \
+  --port 5432 \
+  --database travel_readiness_restore_YYYYMMDD \
+  --username travel_readiness_restore_operator \
+  --backup-dir /ABSOLUTE/PRIVATE/PARENT/phase0-backup-YYYYMMDDTHHMMSSZ \
+  --safety-token RESTORE_DISPOSABLE:travel_readiness_restore_YYYYMMDD
+```
+
+Before restoring, the script checks the archive hash, PostgreSQL 18.6 server,
+the disposable naming rule, the exact token, and that the target contains no
+user table, view, materialized view, sequence, or foreign table. `pg_restore`
+runs with `--single-transaction`, `--exit-on-error`, `--no-owner`, and
+`--no-acl`. It never uses `--clean` or `--create`.
+
+After restore it verifies the exact allowlisted schema, function bodies,
+triggers, constraints, indexes, sequence ownership and sequence state;
+forbidden-storage and large-object absence; zero session and Django admin-log
+rows; every data-bearing backed-up table count and row-set SHA-256 in
+the manifest; and both atomic current-pointer identities. Success prints only
+`restore_result=ok`. Any verification failure
+leaves the explicitly disposable target for diagnosis; discard that target by
+a separately authorized operator action and repeat with a newly named empty
+database. The script itself performs no destructive cleanup.
+
+The archive is the complete allowlisted application database backup, not a
+cluster backup. Both source snapshots and restored targets must contain exactly
+the `public` application schema plus PostgreSQL system namespaces. Any other
+user namespace or namespace-owned user object fails the gate instead of being
+silently omitted by the `--schema=public` archive scope.
+Unapproved rewrite rules, object comments and security labels on the application
+schema also fail closed; the dump independently disables comment and security
+label output so those metadata channels cannot carry raw source or trip input.
+The only accepted comment is PostgreSQL 18.6's standard `public` schema comment,
+matched by an approved SHA-256 and intentionally omitted from the archive.
+
+## Server-rendered integrity marker rehearsal
+
+After a successful database verification, start the local production command
+against only the restored disposable database. Do not enable request or SQL
+logging and do not retain the response body. Make a credential-free `GET` of
+the queryless `/results/` route and verify all of the following:
+
+1. HTTP status is 200 and the response has `Cache-Control: no-store`.
+2. The document contains `id="entry-card"` and `id="warning-card"` exactly once.
+3. Each card has a non-colour `data-state` marker. A restored publication is
+   `ready` or `stale`; it must not silently become `empty` or `unavailable`.
+4. Each published card renders its `publication revision` generation and its
+   source attribution, while no verdict such as `ALLOWED` or `DENIED` appears.
+5. The displayed generations correspond to the restored entry and warning
+   pointer rows recorded by the integrity manifest.
+
+This check proves that the restored pointers cross the server-rendered boundary;
+the database hash comparison alone does not prove that the web process can read
+them. Record only fixed pass/fail markers, never the HTML, a source URL, a
+database credential, or trip form values.
+
+## Rehearsal receipt
+
+For the loopback-only local candidate, the guarded wrapper performs the role
+creation, writer-confirmed backup, empty-schema preparation, restore, integrity
+comparison, SSR marker probe and exact cleanup in one command. Supply the local
+superuser password through the named environment reference without printing it:
+
+```sh
+scripts/check-backup-restore \
+  --host 127.0.0.1 \
+  --port 5432 \
+  --admin-role LOCAL_REHEARSAL_ADMIN \
+  --admin-password-env TRAVEL_READINESS_BACKUP_RESTORE_ADMIN_PASSWORD \
+  --backup-role PREPROVISIONED_READ_ONLY_BACKUP_ROLE \
+  --backup-password-env TRAVEL_READINESS_BACKUP_ROLE_PASSWORD \
+  --source-database travel_readiness \
+  --database-prefix travel_readiness_restorecheck_CANDIDATE \
+  --safety-token BACKUP_RESTORE_REHEARSAL_DISPOSABLE:travel_readiness_restorecheck_CANDIDATE \
+  --writers-quiesced-confirmation WRITERS_QUIESCED
+```
+
+The wrapper accepts only `127.0.0.1` or `localhost` and refuses other hosts,
+including IPv6 literals, so its private password-file fields remain
+unambiguous. It escapes password delimiters required by libpq and refuses
+production-like disposable names.
+`WRITERS_QUIESCED` is an operator assertion, not an automatic lock; give it only
+after all source, reviewer and Admin writers are actually paused. It generates
+a single-use restore credential in memory, keeps libpq material in a mode-0600
+temporary file, and removes the exact restore role, database, archive and
+temporary directory on success and trap-handled failure. `SIGKILL`, host loss,
+or an uncertain database-command outcome cannot be trap-cleaned; after every
+run, an administrator must independently confirm that the exact disposable
+role, database and temporary path are absent before accepting the receipt. The
+backup role must already exist;
+the wrapper verifies that it is a NOINHERIT login with no role memberships or
+owned database objects,
+has only CONNECT, schema USAGE, table SELECT and sequence SELECT, and has no
+database CREATE/TEMPORARY, schema CREATE, table- or column-level write,
+table trigger/MAINTAIN, or sequence mutation privilege. It never grants,
+revokes, creates or drops anything in the source database.
+
+For each release SHA, record these non-sensitive fields in the release evidence:
+
+- exact release SHA and `working_tree=clean`;
+- `postgresql_client_version=18.6` and `postgresql_server_version=18.6`;
+- `backup_result=ok`, `restore_result=ok`, and the UTC rehearsal timestamp;
+- `writers_quiesced=confirmed` for the full accepted backup window;
+- `session_rows=0`, `admin_log_rows=0`, `integrity_manifest=match`, and
+  `publication_pointers=match`;
+- `ssr_results_status=200`, `entry_marker=match`,
+  `travel_warning_marker=match`, `source_attribution=match`, and
+  `cleanup=match`;
+- an external exact-name check showing zero remaining disposable database,
+  role and temporary-directory objects, even when no success receipt exists;
+- backup duration/size, restore duration, and total rehearsal duration (target
+  creation through backup, restore, SSR verification and exact cleanup) against
+  the RPO-24h/RTO-4h policy, without
+  database names, filesystem paths, URLs, hashes, content, or credentials.
+
+The implementation and credential-free tests do not count as a restore
+rehearsal. The release candidate is not backup/restore accepted until the two
+scripts and the server-rendered marker plan have run successfully against a real
+PostgreSQL 18.6 disposable database.
+
+## Production handoff checkpoints
+
+Before production deployment, a human must select and provision the platform,
+PostgreSQL 18.6 service, separate least-privilege runtime/backup/restore roles,
+credential injection, encrypted backup destination and retention, alert route,
+domain, TLS termination, and DNS. Validate the provider's point-in-time recovery
+and custom-archive restore behavior separately. Production deployment and any
+drop of a rehearsal database remain human-only actions.
diff --git a/operations/tests/test_postgresql_backup_restore.py b/operations/tests/test_postgresql_backup_restore.py
new file mode 100644
index 0000000..9635c93
--- /dev/null
+++ b/operations/tests/test_postgresql_backup_restore.py
@@ -0,0 +1,982 @@
+from __future__ import annotations
+
+import hashlib
+import importlib
+import os
+from pathlib import Path
+import re
+import stat
+import subprocess
+import tempfile
+import unittest
+
+
+class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
+    @classmethod
+    def setUpClass(cls):
+        super().setUpClass()
+        cls.root = Path(__file__).resolve().parents[2]
+        cls.backup = cls.root / "scripts" / "backup-postgresql"
+        cls.restore = cls.root / "scripts" / "restore-postgresql"
+        cls.rehearsal = cls.root / "scripts" / "check-backup-restore"
+        cls.common = cls.root / "scripts" / "postgresql-common"
+        cls.integrity_sql = cls.root / "scripts" / "postgresql-integrity.sql"
+        cls.runbook = cls.root / "docs" / "OPERATIONS-RUNBOOK.md"
+
+    def run_script(self, script: Path, *arguments: str, env=None):
+        return subprocess.run(
+            ["/bin/sh", str(script), *arguments],
+            cwd=self.root,
+            env=env,
+            capture_output=True,
+            text=True,
+            check=False,
+            timeout=10,
+        )
+
+    def rehearsal_arguments(self, **overrides):
+        prefix = overrides.get(
+            "database_prefix", "travel_readiness_restorecheck_unit"
+        )
+        values = {
+            "host": "127.0.0.1",
+            "port": "5432",
+            "admin_role": "postgres",
+            "admin_password_env": (
+                "TRAVEL_READINESS_BACKUP_RESTORE_ADMIN_PASSWORD"
+            ),
+            "backup_role": "travel_readiness_backup_unit",
+            "backup_password_env": "TRAVEL_READINESS_BACKUP_ROLE_PASSWORD",
+            "source_database": "travel_readiness",
+            "database_prefix": prefix,
+            "safety_token": (
+                "BACKUP_RESTORE_REHEARSAL_DISPOSABLE:" + prefix
+            ),
+            "writers_quiesced_confirmation": "WRITERS_QUIESCED",
+        }
+        values.update(overrides)
+        return [
+            "--host",
+            values["host"],
+            "--port",
+            values["port"],
+            "--admin-role",
+            values["admin_role"],
+            "--admin-password-env",
+            values["admin_password_env"],
+            "--backup-role",
+            values["backup_role"],
+            "--backup-password-env",
+            values["backup_password_env"],
+            "--source-database",
+            values["source_database"],
+            "--database-prefix",
+            values["database_prefix"],
+            "--safety-token",
+            values["safety_token"],
+            "--writers-quiesced-confirmation",
+            values["writers_quiesced_confirmation"],
+        ]
+
+    def make_fake_postgresql_18_6(self, parent: Path):
+        common = self.common.read_text(encoding="utf-8")
+        schema_digests = (
+            common.split("EXPECTED_SCHEMA_DIGESTS='", 1)[1]
+            .split("'\n", 1)[0]
+        )
+        fixtures = parent / "fixtures"
+        fixtures.mkdir()
+        fixture_values = {
+            "tables": common.split("EXPECTED_PUBLIC_TABLES='", 1)[1]
+            .split("'\n", 1)[0],
+            "sequences": common.split("EXPECTED_PUBLIC_SEQUENCES='", 1)[1]
+            .split("'\n", 1)[0],
+            "sequence_ownership": common.split(
+                "EXPECTED_SEQUENCE_OWNERSHIP='", 1
+            )[1].split("'\n", 1)[0],
+            "sequence_definition": re.search(
+                r'EXPECTED_SEQUENCE_DEFINITION_SHA256="([0-9a-f]{64})"',
+                common,
+            ).group(1),
+            "functions": common.split("EXPECTED_TRIGGER_FUNCTIONS='", 1)[1]
+            .split("'\n", 1)[0],
+            "triggers": common.split("EXPECTED_PUBLIC_TRIGGERS='", 1)[1]
+            .split("'\n", 1)[0],
+            "snapshot": (
+                "postgresql.version_num=180006\n"
+                + schema_digests
+                + "\n"
+                + "database.locale_profile.sha256="
+                + ("0" * 64)
+                + "\n"
+                + "schema.sequences.sha256="
+                + ("0" * 64)
+                + "\n"
+                + "schema.sequence_state.sha256="
+                + ("0" * 64)
+                + "\n"
+                "pointer.entry=NONE:0\n"
+                "pointer.travel_warning=NONE:0\n"
+            ),
+        }
+        for name, value in fixture_values.items():
+            (fixtures / name).write_text(value + ("" if value.endswith("\n") else "\n"))
+
+        fake_bin = parent / "bin"
+        fake_bin.mkdir()
+        fake_psql = fake_bin / "psql"
+        fake_psql.write_text(
+            "#!/bin/sh\n"
+            "[ -z \"${MOFA_TRAVEL_ALARM_SERVICE_KEY:-}\" ] || exit 97\n"
+            "[ -z \"${TRAVEL_READINESS_SECRET_KEY:-}\" ] || exit 97\n"
+            "[ -z \"${TRAVEL_READINESS_DB_PASSWORD:-}\" ] || exit 97\n"
+            "case \"$*\" in\n"
+            "  *--version*) printf '%s\\n' 'psql (PostgreSQL) 18.6'; exit 0 ;;\n"
+            "  *'SHOW server_version_num'*) printf '%s\\n' '180006'; exit 0 ;;\n"
+            "  *'SELECT table_name FROM information_schema.tables'*) /bin/cat \"$FAKE_PG_FIXTURES/tables\"; exit 0 ;;\n"
+            "  *\"c.relkind = 'S'\"*) /bin/cat \"$FAKE_PG_FIXTURES/sequences\"; exit 0 ;;\n"
+            "  *'FROM pg_catalog.pg_sequences'*) /bin/cat \"$FAKE_PG_FIXTURES/sequence_definition\"; exit 0 ;;\n"
+            "  *\"c.relkind NOT IN\"*) printf '%s\\n' '0'; exit 0 ;;\n"
+            "  *'c.relpersistence'*) printf '%s\\n' '0'; exit 0 ;;\n"
+            "  *\"SELECT s.relname || '|' || t.relname\"*) /bin/cat \"$FAKE_PG_FIXTURES/sequence_ownership\"; exit 0 ;;\n"
+            "  *\"SELECT p.proname || '|'\"*) /bin/cat \"$FAKE_PG_FIXTURES/functions\"; exit 0 ;;\n"
+            "  *\"SELECT t.tgname || '|'\"*) /bin/cat \"$FAKE_PG_FIXTURES/triggers\"; exit 0 ;;\n"
+            "  *\"t.tgenabled <> 'O'\"*) printf '%s\\n' '0'; exit 0 ;;\n"
+            "  *'pg_get_function_identity_arguments'*) printf '%s\\n' '0'; exit 0 ;;\n"
+            "  *'SELECT (SELECT count(*) FROM pg_catalog.pg_policy'*) printf '%s\\n' '0'; exit 0 ;;\n"
+            "  *'SELECT count(*) FROM pg_catalog.pg_rewrite AS r'*) printf '%s\\n' \"${FAKE_REWRITE_COUNT:-0}\"; exit 0 ;;\n"
+            "  *'pg_catalog.pg_description AS d'*) printf '%s\\n' \"${FAKE_METADATA_COUNT:-0}\"; exit 0 ;;\n"
+            "  *'information_schema.columns'*) printf '%s\\n' '0'; exit 0 ;;\n"
+            "  *'pg_largeobject_metadata'*) printf '%s\\n' '0'; exit 0 ;;\n"
+            "  *\"n.nspname NOT IN ('pg_catalog'\"*) printf '%s\\n' \"${FAKE_NON_PUBLIC_OBJECT_COUNT:-0}\"; exit 0 ;;\n"
+            "  *\"nspname = 'public'\"*) printf '%s\\n' \"${FAKE_PUBLIC_NAMESPACE_COUNT:-0}\"; exit 0 ;;\n"
+            "  *'FROM pg_catalog.pg_namespace WHERE nspname NOT IN'*) printf '%s\\n' \"${FAKE_USER_NAMESPACE_COUNT:-0}\"; exit 0 ;;\n"
+            "  *'FROM pg_catalog.pg_db_role_setting'*) printf '%s\\n' '0'; exit 0 ;;\n"
+            "  *'SELECT (SELECT count(*) FROM pg_catalog.pg_proc'*) printf '%s\\n' '0'; exit 0 ;;\n"
+            "  *'public.django_session'*) printf '%s\\n' '0'; exit 0 ;;\n"
+            "  *'public.django_admin_log'*) printf '%s\\n' '0'; exit 0 ;;\n"
+            "  *--file=*) /bin/cat \"$FAKE_PG_FIXTURES/snapshot\"; exit 0 ;;\n"
+            "esac\n"
+            "exit 99\n",
+            encoding="utf-8",
+        )
+        fake_dump = fake_bin / "pg_dump"
+        fake_dump.write_text(
+            "#!/bin/sh\n"
+            "[ -z \"${MOFA_TRAVEL_ALARM_SERVICE_KEY:-}\" ] || exit 97\n"
+            "[ -z \"${TRAVEL_READINESS_SECRET_KEY:-}\" ] || exit 97\n"
+            "[ -z \"${TRAVEL_READINESS_DB_PASSWORD:-}\" ] || exit 97\n"
+            "if [ \"${1:-}\" = '--version' ]; then\n"
+            "  printf '%s\\n' 'pg_dump (PostgreSQL) 18.6'\n"
+            "  exit 0\n"
+            "fi\n"
+            "for argument in \"$@\"; do\n"
+            "  case \"$argument\" in\n"
+            "    --file=*) output=${argument#--file=} ;;\n"
+            "  esac\n"
+            "done\n"
+            "[ -n \"${output:-}\" ] || exit 98\n"
+            "printf '%s\\n' 'synthetic custom archive' >\"$output\"\n",
+            encoding="utf-8",
+        )
+        fake_restore = fake_bin / "pg_restore"
+        fake_restore.write_text(
+            "#!/bin/sh\n"
+            "[ -z \"${MOFA_TRAVEL_ALARM_SERVICE_KEY:-}\" ] || exit 97\n"
+            "[ -z \"${TRAVEL_READINESS_SECRET_KEY:-}\" ] || exit 97\n"
+            "[ -z \"${TRAVEL_READINESS_DB_PASSWORD:-}\" ] || exit 97\n"
+            "if [ \"${1:-}\" = '--version' ]; then\n"
+            "  printf '%s\\n' 'pg_restore (PostgreSQL) 18.6'\n"
+            "  exit 0\n"
+            "fi\n"
+            "exit 0\n",
+            encoding="utf-8",
+        )
+        for executable in (fake_psql, fake_dump, fake_restore):
+            executable.chmod(executable.stat().st_mode | stat.S_IXUSR)
+        env = os.environ.copy()
+        env["PATH"] = f"{fake_bin}:{env.get('PATH', '')}"
+        env["FAKE_PG_FIXTURES"] = str(fixtures)
+        env["MOFA_TRAVEL_ALARM_SERVICE_KEY"] = "must-not-reach-child"
+        env["TRAVEL_READINESS_SECRET_KEY"] = "must-not-reach-child"
+        env["TRAVEL_READINESS_DB_PASSWORD"] = "must-not-reach-child"
+        return env
+
+    def test_missing_and_unknown_arguments_fail_with_fixed_receipts(self):
+        backup = self.run_script(self.backup)
+        self.assertEqual(backup.returncode, 64)
+        self.assertEqual(backup.stdout, "")
+        self.assertEqual(backup.stderr, "backup_result=INVALID_ARGUMENTS\n")
+
+        restore = self.run_script(self.restore, "--unknown")
+        self.assertEqual(restore.returncode, 64)
+        self.assertEqual(restore.stdout, "")
+        self.assertEqual(restore.stderr, "restore_result=INVALID_ARGUMENTS\n")
+
+    def test_rehearsal_help_guards_and_secret_reference_are_fixed(self):
+        self.assertEqual(stat.S_IMODE(self.rehearsal.stat().st_mode), 0o755)
+        help_result = self.run_script(self.rehearsal, "--help")
+        self.assertEqual(help_result.returncode, 0)
+        self.assertEqual(help_result.stderr, "")
+        self.assertIn("--writers-quiesced-confirmation WRITERS_QUIESCED", help_result.stdout)
+
+        remote = self.run_script(
+            self.rehearsal,
+            *self.rehearsal_arguments(host="database.invalid"),
+        )
+        self.assertEqual(remote.returncode, 65)
+        self.assertEqual(
+            remote.stderr,
+            "backup_restore_check=NON_LOOPBACK_REFUSED\n",
+        )
+
+        ipv6_loopback = self.run_script(
+            self.rehearsal,
+            *self.rehearsal_arguments(host="::1"),
+        )
+        self.assertEqual(ipv6_loopback.returncode, 65)
+        self.assertEqual(
+            ipv6_loopback.stderr,
+            "backup_restore_check=NON_LOOPBACK_REFUSED\n",
+        )
+
+        production_like = self.run_script(
+            self.rehearsal,
+            *self.rehearsal_arguments(
+                database_prefix="travel_readiness_restorecheck_prod",
+                safety_token=(
+                    "BACKUP_RESTORE_REHEARSAL_DISPOSABLE:"
+                    "travel_readiness_restorecheck_prod"
+                ),
+            ),
+        )
+        self.assertEqual(production_like.returncode, 65)
+        self.assertEqual(
+            production_like.stderr,
+            "backup_restore_check=PRODUCTION_LIKE_PREFIX_REFUSED\n",
+        )
+
+        no_quiescence = self.run_script(
+            self.rehearsal,
+            *self.rehearsal_arguments(
+                writers_quiesced_confirmation="NOT_CONFIRMED"
+            ),
+        )
+        self.assertEqual(no_quiescence.returncode, 65)
+        self.assertEqual(
+            no_quiescence.stderr,
+            "backup_restore_check=WRITER_QUIESCENCE_REQUIRED\n",
+        )
+
+        environment = os.environ.copy()
+        environment.pop(
+            "TRAVEL_READINESS_BACKUP_RESTORE_ADMIN_PASSWORD", None
+        )
+        missing_secret = self.run_script(
+            self.rehearsal,
+            *self.rehearsal_arguments(),
+            env=environment,
+        )
+        self.assertEqual(missing_secret.returncode, 66)
+        self.assertEqual(
+            missing_secret.stderr,
+            "backup_restore_check=ADMIN_PASSWORD_MISSING\n",
+        )
+
+    def test_rehearsal_closes_backup_restore_ssr_and_cleanup_loop(self):
+        script = self.rehearsal.read_text(encoding="utf-8")
+        lower = script.lower()
+        for required in (
+            '"$script_dir/backup-postgresql"',
+            '"$script_dir/restore-postgresql"',
+            "PublishedEntryFacts",
+            "PublishedTravelWarning",
+            'Client().get("/results/", secure=True)',
+            'body.count("id=\\"entry-card\\"") == 1',
+            'body.count("id=\\"warning-card\\"") == 1',
+            "DROP DATABASE",
+            "writers_quiesced=confirmed",
+            "integrity_manifest=match",
+            "publication_pointers=match",
+            "entry_marker=match",
+            "travel_warning_marker=match",
+            "source_attribution=match",
+            "cleanup=match",
+            "rehearsal_seconds=",
+        ):
+            self.assertIn(required, script)
+        self.assertIn("set +x", script)
+        self.assertIn("umask 077", script)
+        self.assertIn("--no-password", script)
+        self.assertIn("BACKUP_ROLE_NOT_READ_ONLY", script)
+        self.assertIn("NOT rolinherit", script)
+        self.assertIn("pg_catalog.pg_auth_members", script)
+        self.assertIn("pg_catalog.pg_shdepend", script)
+        self.assertIn(
+            "d.refclassid = 'pg_catalog.pg_authid'::pg_catalog.regclass",
+            script,
+        )
+        self.assertIn("d.deptype = 'o'", script)
+        self.assertIn(
+            "NOT has_database_privilege('$backup_role', "
+            "'$source_database', 'CREATE')",
+            script,
+        )
+        self.assertIn(
+            "NOT has_table_privilege('$backup_role', c.oid, 'MAINTAIN')",
+            script,
+        )
+        for column_privilege in ("INSERT", "UPDATE", "REFERENCES"):
+            self.assertIn(
+                "NOT has_any_column_privilege('$backup_role', c.oid, "
+                f"'{column_privilege}')",
+                script,
+            )
+        self.assertNotIn("DROP OWNED BY", script)
+        self.assertNotIn("GRANT SELECT", script)
+        self.assertNotIn("\nassert ", script)
+        self.assertNotIn(".env.local", lower)
+        self.assertEqual(lower.count("mofa_travel_alarm_service_key"), 1)
+        self.assertIn("unset mofa_travel_alarm_service_key", lower)
+        self.assertNotIn("set -x", lower)
+        self.assertNotIn("eval ", lower)
+        self.assertIn('escape_pgpass_field "$backup_password"', script)
+
+        cleanup_call = script.rindex("if ! cleanup; then")
+        success_receipt = script.index("printf 'backup_restore_check=ok")
+        self.assertLess(cleanup_call, success_receipt)
+
+        admin_read = script.index(
+            "admin_password=${TRAVEL_READINESS_BACKUP_RESTORE_ADMIN_PASSWORD-}"
+        )
+        admin_unset = script.index(
+            "unset TRAVEL_READINESS_BACKUP_RESTORE_ADMIN_PASSWORD"
+        )
+        backup_read = script.index(
+            "backup_password=${TRAVEL_READINESS_BACKUP_ROLE_PASSWORD-}"
+        )
+        backup_unset = script.index(
+            "unset TRAVEL_READINESS_BACKUP_ROLE_PASSWORD"
+        )
+        first_database_child = script.index("require_pg_tool psql")
+        self.assertLess(admin_read, admin_unset)
+        self.assertLess(admin_unset, first_database_child)
+        self.assertLess(backup_read, backup_unset)
+        self.assertLess(backup_unset, first_database_child)
+
+    def test_rehearsal_secret_input_names_do_not_reach_database_children(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            temporary_path = Path(temporary)
+            fake_bin = temporary_path / "bin"
+            fake_bin.mkdir()
+            fake_psql = fake_bin / "psql"
+            fake_psql.write_text(
+                "#!/bin/sh\n"
+                "[ -z \"${TRAVEL_READINESS_BACKUP_RESTORE_ADMIN_PASSWORD:-}\" ] || exit 97\n"
+                "[ -z \"${TRAVEL_READINESS_BACKUP_ROLE_PASSWORD:-}\" ] || exit 97\n"
+                "case \"$*\" in\n"
+                "  *--version*) printf '%s\\n' 'psql (PostgreSQL) 18.6'; exit 0 ;;\n"
+                "  *server_version_num*) printf '%s\\n' '180006'; exit 0 ;;\n"
+                "  *rolsuper*) printf '%s\\n' 'f'; exit 0 ;;\n"
+                "esac\n"
+                "exit 98\n",
+                encoding="utf-8",
+            )
+            fake_psql.chmod(fake_psql.stat().st_mode | stat.S_IXUSR)
+            environment = os.environ.copy()
+            environment["PATH"] = f"{fake_bin}:{environment.get('PATH', '')}"
+            environment[
+                "TRAVEL_READINESS_BACKUP_RESTORE_ADMIN_PASSWORD"
+            ] = "admin-input-must-be-unset"
+            environment[
+                "TRAVEL_READINESS_BACKUP_ROLE_PASSWORD"
+            ] = "backup-input-must-be-unset"
+            result = self.run_script(
+                self.rehearsal,
+                *self.rehearsal_arguments(),
+                env=environment,
+            )
+
+        self.assertEqual(result.returncode, 70)
+        self.assertEqual(result.stdout, "")
+        self.assertEqual(
+            result.stderr,
+            "backup_restore_check=ADMIN_CAPABILITY_REQUIRED\n",
+        )
+
+    def test_pgpass_field_escaping_covers_delimiters(self):
+        result = subprocess.run(
+            [
+                "/bin/sh",
+                "-c",
+                '. "$1"; escape_pgpass_field "$2"',
+                "sh",
+                str(self.common),
+                "colon:slash\\value",
+            ],
+            cwd=self.root,
+            capture_output=True,
+            text=True,
+            check=False,
+            timeout=10,
+        )
+        self.assertEqual(result.returncode, 0)
+        self.assertEqual(result.stderr, "")
+        self.assertEqual(result.stdout, "colon\\:slash\\\\value")
+
+    def test_backup_rejects_relative_or_existing_output_before_tool_access(self):
+        relative = self.run_script(
+            self.backup,
+            "--host",
+            "localhost",
+            "--port",
+            "5432",
+            "--database",
+            "travel_readiness",
+            "--username",
+            "backup_operator",
+            "--backup-dir",
+            "relative-backup",
+        )
+        self.assertEqual(relative.returncode, 64)
+        self.assertEqual(relative.stderr, "backup_result=INVALID_ARGUMENTS\n")
+
+        with tempfile.TemporaryDirectory() as existing:
+            duplicate = self.run_script(
+                self.backup,
+                "--host",
+                "localhost",
+                "--port",
+                "5432",
+                "--database",
+                "travel_readiness",
+                "--username",
+                "backup_operator",
+                "--backup-dir",
+                existing,
+            )
+        self.assertEqual(duplicate.returncode, 64)
+        self.assertEqual(duplicate.stderr, "backup_result=INVALID_ARGUMENTS\n")
+
+    def test_restore_rejects_non_disposable_target_and_wrong_token_first(self):
+        unsafe = self.run_script(
+            self.restore,
+            "--host",
+            "localhost",
+            "--port",
+            "5432",
+            "--database",
+            "travel_readiness",
+            "--username",
+            "restore_operator",
+            "--backup-dir",
+            "/private/tmp/not-used",
+            "--safety-token",
+            "RESTORE_DISPOSABLE:travel_readiness",
+        )
+        self.assertEqual(unsafe.returncode, 65)
+        self.assertEqual(unsafe.stderr, "restore_result=UNSAFE_TARGET\n")
+
+        wrong_token = self.run_script(
+            self.restore,
+            "--host",
+            "localhost",
+            "--port",
+            "5432",
+            "--database",
+            "travel_readiness_restore_test",
+            "--username",
+            "restore_operator",
+            "--backup-dir",
+            "/private/tmp/not-used",
+            "--safety-token",
+            "RESTORE_DISPOSABLE:travel_readiness_restore_other",
+        )
+        self.assertEqual(wrong_token.returncode, 65)
+        self.assertEqual(wrong_token.stderr, "restore_result=SAFETY_TOKEN_MISMATCH\n")
+
+    def test_restore_rejects_incomplete_extra_or_public_backup_inputs(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            base = Path(temporary)
+            for case in ("incomplete", "extra", "public"):
+                backup_dir = base / case
+                backup_dir.mkdir(mode=0o700)
+                (backup_dir / "database.dump").write_bytes(b"archive")
+                (backup_dir / "integrity.manifest").write_text(
+                    "manifest\n",
+                    encoding="utf-8",
+                )
+                (backup_dir / "database.dump").chmod(0o600)
+                (backup_dir / "integrity.manifest").chmod(0o600)
+                if case == "incomplete":
+                    (backup_dir / ".incomplete").touch()
+                elif case == "extra":
+                    (backup_dir / "unexpected").touch()
+                else:
+                    (backup_dir / "database.dump").chmod(0o644)
+
+                with self.subTest(case=case):
+                    result = self.run_script(
+                        self.restore,
+                        "--host",
+                        "localhost",
+                        "--port",
+                        "5432",
+                        "--database",
+                        "travel_readiness_restore_unit",
+                        "--username",
+                        "restore_operator",
+                        "--backup-dir",
+                        str(backup_dir),
+                        "--safety-token",
+                        "RESTORE_DISPOSABLE:travel_readiness_restore_unit",
+                    )
+                    self.assertEqual(result.returncode, 66)
+                    self.assertEqual(
+                        result.stderr,
+                        "restore_result=INVALID_BACKUP\n",
+                    )
+
+    def test_postgresql_tool_version_mismatch_is_fixed_and_never_connects(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            temporary_path = Path(temporary)
+            fake_bin = temporary_path / "bin"
+            fake_bin.mkdir()
+            fake_psql = fake_bin / "psql"
+            fake_psql.write_text(
+                "#!/bin/sh\n"
+                "if [ \"${1:-}\" = \"--version\" ]; then\n"
+                "  printf '%s\\n' 'psql (PostgreSQL) 17.0'\n"
+                "  exit 0\n"
+                "fi\n"
+                "printf '%s\\n' 'unexpected connection' >&2\n"
+                "exit 99\n",
+                encoding="utf-8",
+            )
+            fake_psql.chmod(fake_psql.stat().st_mode | stat.S_IXUSR)
+            output = temporary_path / "new-backup"
+            env = os.environ.copy()
+            env["PATH"] = f"{fake_bin}:{env.get('PATH', '')}"
+            result = self.run_script(
+                self.backup,
+                "--host",
+                "localhost",
+                "--port",
+                "5432",
+                "--database",
+                "travel_readiness",
+                "--username",
+                "backup_operator",
+                "--backup-dir",
+                str(output),
+                env=env,
+            )
+        self.assertEqual(result.returncode, 69)
+        self.assertEqual(result.stdout, "")
+        self.assertEqual(result.stderr, "backup_result=POSTGRESQL_18_6_REQUIRED\n")
+        self.assertNotIn("unexpected connection", result.stderr)
+
+    def test_synthetic_success_path_builds_and_verifies_private_archive(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            temporary_path = Path(temporary)
+            env = self.make_fake_postgresql_18_6(temporary_path)
+            backup_dir = temporary_path / "accepted-backup"
+            backup = self.run_script(
+                self.backup,
+                "--host",
+                "localhost",
+                "--port",
+                "5432",
+                "--database",
+                "travel_readiness",
+                "--username",
+                "backup_operator",
+                "--backup-dir",
+                str(backup_dir),
+                env=env,
+            )
+            self.assertEqual(backup.returncode, 0, backup.stderr)
+            self.assertEqual(backup.stdout, "backup_result=ok\n")
+            self.assertEqual(backup.stderr, "")
+            self.assertEqual(stat.S_IMODE(backup_dir.stat().st_mode), 0o700)
+            self.assertEqual(
+                {path.name for path in backup_dir.iterdir()},
+                {"database.dump", "integrity.manifest"},
+            )
+            for filename in ("database.dump", "integrity.manifest"):
+                self.assertEqual(
+                    stat.S_IMODE((backup_dir / filename).stat().st_mode), 0o600
+                )
+
+            restore = self.run_script(
+                self.restore,
+                "--host",
+                "localhost",
+                "--port",
+                "5432",
+                "--database",
+                "travel_readiness_restore_synthetic",
+                "--username",
+                "restore_operator",
+                "--backup-dir",
+                str(backup_dir),
+                "--safety-token",
+                "RESTORE_DISPOSABLE:travel_readiness_restore_synthetic",
+                env=env,
+            )
+            self.assertEqual(restore.returncode, 0, restore.stderr)
+            self.assertEqual(restore.stdout, "restore_result=ok\n")
+            self.assertEqual(restore.stderr, "")
+
+            manifest = (backup_dir / "integrity.manifest").read_text(
+                encoding="utf-8"
+            )
+            for forbidden in (
+                "MOFA_TRAVEL_ALARM_SERVICE_KEY",
+                "serviceKey",
+                "destination",
+                "departure_date",
+                "return_date",
+                "session_data",
+                "https://",
+            ):
+                self.assertNotIn(forbidden, manifest)
+
+    def test_backup_rejects_unapproved_schema_side_channels(self):
+        cases = (
+            "FAKE_USER_NAMESPACE_COUNT",
+            "FAKE_NON_PUBLIC_OBJECT_COUNT",
+            "FAKE_REWRITE_COUNT",
+            "FAKE_METADATA_COUNT",
+        )
+        for environment_name in cases:
+            with self.subTest(environment_name=environment_name):
+                with tempfile.TemporaryDirectory() as temporary:
+                    temporary_path = Path(temporary)
+                    env = self.make_fake_postgresql_18_6(temporary_path)
+                    env[environment_name] = "1"
+                    backup_dir = temporary_path / "rejected-backup"
+                    result = self.run_script(
+                        self.backup,
+                        "--host",
+                        "localhost",
+                        "--port",
+                        "5432",
+                        "--database",
+                        "travel_readiness",
+                        "--username",
+                        "backup_operator",
+                        "--backup-dir",
+                        str(backup_dir),
+                        env=env,
+                    )
+                    self.assertEqual(result.returncode, 65)
+                    self.assertEqual(result.stdout, "")
+                    self.assertEqual(
+                        result.stderr,
+                        "backup_result=SCHEMA_OBJECTS_NOT_APPROVED\n",
+                    )
+                    self.assertFalse(backup_dir.exists())
+
+    def test_dump_and_restore_commands_are_allowlisted_and_non_destructive(self):
+        backup = self.backup.read_text(encoding="utf-8")
+        restore = self.restore.read_text(encoding="utf-8")
+        common = self.common.read_text(encoding="utf-8")
+        combined = "\n".join((backup, restore, common)).lower()
+
+        approved_tables = {
+            "auth_group",
+            "auth_group_permissions",
+            "auth_permission",
+            "auth_user",
+            "auth_user_groups",
+            "auth_user_user_permissions",
+            "countries_country",
+            "django_admin_log",
+            "django_content_type",
+            "django_migrations",
+            "django_session",
+            "entry_requirements_entryfactrevision",
+            "entry_requirements_passportpolicy",
+            "reviews_auditevent",
+            "reviews_publicationrevision",
+            "reviews_publishedentryfacts",
+            "reviews_publishedtravelwarning",
+            "reviews_reviewdecision",
+            "sources_fetchattempt",
+            "sources_parserun",
+            "sources_sourceartifact",
+            "sources_sourceconfiguration",
+            "sources_sourcerightsdecision",
+            "travel_warnings_travelwarningrevision",
+        }
+        contract_tables = set(
+            common.split("EXPECTED_PUBLIC_TABLES='", 1)[1]
+            .split("'\n", 1)[0]
+            .splitlines()
+        )
+        self.assertEqual(contract_tables, approved_tables)
+        self.assertIn("--schema=public", backup)
+        self.assertIn("--no-comments", backup)
+        self.assertIn("--no-security-labels", backup)
+        self.assertNotIn("--table=public.", backup)
+        self.assertIn("--exclude-table-data=public.django_admin_log", backup)
+        self.assertIn("--exclude-table-data=public.django_session", backup)
+        self.assertIn("--single-transaction", restore)
+        self.assertIn("--exit-on-error", restore)
+        self.assertIn("travel_readiness_restore_", restore)
+        self.assertIn("RESTORE_DISPOSABLE:", restore)
+        self.assertIn('POSTGRESQL_REQUIRED_VERSION="18.6"', common)
+        self.assertIn("EXPECTED_PUBLIC_SCHEMA_COMMENT_SHA256=", common)
+        self.assertIn("EXPECTED_SCHEMA_DIGESTS=", common)
+        self.assertIn("EXPECTED_SEQUENCE_DEFINITION_SHA256=", common)
+        self.assertEqual(
+            backup.count("database_has_exact_schema_digests"),
+            2,
+        )
+        self.assertIn("database_has_exact_schema_digests", restore)
+        self.assertIn("database_has_exact_schema_objects", backup)
+        self.assertIn("database_has_exact_schema_objects", restore)
+        self.assertIn("t.tgenabled <> 'O'", common)
+        self.assertIn("unexpected_relation_policy_count", common)
+        self.assertIn("unexpected_rewrite_count", common)
+        self.assertIn("unexpected_object_metadata_count", common)
+        self.assertIn("pg_catalog.pg_description", common)
+        self.assertIn("pg_catalog.pg_seclabel", common)
+        self.assertIn("d.objsubid = 0", common)
+        self.assertIn("unexpected_user_namespace_count", common)
+        self.assertIn("unexpected_non_public_user_object_count", common)
+        self.assertIn("EXPECTED_SEQUENCE_OWNERSHIP", common)
+        self.assertIn("actual_sequence_ownership", common)
+        self.assertIn("user_namespace_count", common)
+        self.assertIn("public_namespace_count", common)
+        self.assertIn("applicable_setting_count", common)
+        self.assertIn(
+            "column_name IN ('raw_body_storage_allowed', "
+            "'raw_retention_seconds')",
+            common,
+        )
+        self.assertIn("backup_result=INTERRUPTED", backup)
+        self.assertIn("restore_result=INTERRUPTED", restore)
+        self.assertNotIn("trap cleanup exit hup int term", combined)
+
+        for forbidden in (
+            "drop database",
+            "dropdb",
+            "createdb",
+            "--clean",
+            "--create",
+            "database_url",
+            "set -x",
+            "cat .env",
+        ):
+            self.assertNotIn(forbidden, combined)
+
+    def test_restore_requires_default_public_schema_to_be_removed_first(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            temporary_path = Path(temporary)
+            env = self.make_fake_postgresql_18_6(temporary_path)
+            backup_dir = temporary_path / "accepted-backup"
+            backup = self.run_script(
+                self.backup,
+                "--host",
+                "localhost",
+                "--port",
+                "5432",
+                "--database",
+                "travel_readiness",
+                "--username",
+                "backup_operator",
+                "--backup-dir",
+                str(backup_dir),
+                env=env,
+            )
+            self.assertEqual(backup.returncode, 0, backup.stderr)
+
+            env["FAKE_PUBLIC_NAMESPACE_COUNT"] = "1"
+            restore = self.run_script(
+                self.restore,
+                "--host",
+                "localhost",
+                "--port",
+                "5432",
+                "--database",
+                "travel_readiness_restore_public_schema",
+                "--username",
+                "restore_operator",
+                "--backup-dir",
+                str(backup_dir),
+                "--safety-token",
+                "RESTORE_DISPOSABLE:travel_readiness_restore_public_schema",
+                env=env,
+            )
+            self.assertEqual(restore.returncode, 65)
+            self.assertEqual(restore.stdout, "")
+            self.assertEqual(restore.stderr, "restore_result=TARGET_NOT_EMPTY\n")
+
+    def test_integrity_manifest_covers_evidence_publications_and_pointers(self):
+        sql = self.integrity_sql.read_text(encoding="utf-8")
+        protected_tables = (
+            "countries_country",
+            "sources_sourceconfiguration",
+            "sources_sourcerightsdecision",
+            "sources_fetchattempt",
+            "sources_sourceartifact",
+            "sources_parserun",
+            "entry_requirements_passportpolicy",
+            "entry_requirements_entryfactrevision",
+            "travel_warnings_travelwarningrevision",
+            "reviews_reviewdecision",
+            "reviews_publicationrevision",
+            "reviews_publishedentryfacts",
+            "reviews_publishedtravelwarning",
+            "reviews_auditevent",
+        )
+        for table in protected_tables:
+            self.assertIn(f"table.{table}.count=", sql)
+            self.assertIn(f"table.{table}.sha256=", sql)
+            self.assertIn(f"public.{table}", sql)
+        self.assertIn("pointer.entry=", sql)
+        self.assertIn("pointer.travel_warning=", sql)
+        self.assertIn("schema.sequence_state.sha256=", sql)
+        self.assertIn("database.locale_profile.sha256=", sql)
+        self.assertIn("db.datlocprovider", sql)
+        self.assertIn("db.datlocale", sql)
+        self.assertIn("is_called", sql)
+        self.assertIn("row_number() OVER", sql)
+        self.assertIn("'::character varying::text'", sql)
+        self.assertIn("']::text[]'", sql)
+        self.assertIn("sha256(convert_to", sql)
+        self.assertNotIn("FROM public.django_admin_log AS t", sql)
+        self.assertNotIn("FROM public.django_session AS t", sql)
+        for schema_digest in (
+            "schema.columns.sha256=",
+            "schema.constraints.sha256=",
+            "schema.indexes.sha256=",
+            "schema.trigger_functions.sha256=",
+            "schema.triggers.sha256=",
+            "schema.sequences.sha256=",
+        ):
+            self.assertIn(schema_digest, sql)
+
+    def test_trigger_function_bodies_and_trigger_shapes_are_exact_contracts(self):
+        common = self.common.read_text(encoding="utf-8")
+        functions = (
+            common.split("EXPECTED_TRIGGER_FUNCTIONS='", 1)[1]
+            .split("'\n", 1)[0]
+            .splitlines()
+        )
+        triggers = (
+            common.split("EXPECTED_PUBLIC_TRIGGERS='", 1)[1]
+            .split("'\n", 1)[0]
+            .splitlines()
+        )
+        self.assertEqual(len(functions), 25)
+        self.assertEqual(len({line.split("|", 1)[0] for line in functions}), 25)
+        expected_function_lines = set(functions)
+        for line in functions:
+            name, digest = line.split("|")
+            self.assertRegex(name, r"^[a-z][a-z0-9_]+$")
+            self.assertRegex(digest, r"^[0-9a-f]{64}$")
+        self.assertEqual(len(triggers), 30)
+        self.assertEqual(len({line.split("|", 1)[0] for line in triggers}), 30)
+        for line in triggers:
+            self.assertEqual(len(line.split("|")), 7)
+
+        from django.conf import settings
+        from django.db.migrations.operations.special import RunSQL
+
+        if not settings.configured:
+            settings.configure(AUTH_USER_MODEL="auth.User")
+        forward_sql = []
+        for app in (
+            "countries",
+            "sources",
+            "entry_requirements",
+            "travel_warnings",
+            "reviews",
+        ):
+            migration_dir = self.root / app / "migrations"
+            for migration in sorted(migration_dir.glob("[0-9]*.py")):
+                module = importlib.import_module(
+                    f"{app}.migrations.{migration.stem}"
+                )
+                forward_sql.extend(
+                    operation.sql
+                    for operation in module.Migration.operations
+                    if isinstance(operation, RunSQL)
+                    and isinstance(operation.sql, str)
+                )
+
+        function_pattern = re.compile(
+            r"CREATE(?: OR REPLACE)? FUNCTION ([a-z0-9_]+)\(\) "
+            r"RETURNS trigger\s+LANGUAGE plpgsql AS \$\$(.*?)\$\$;",
+            re.DOTALL,
+        )
+        final_function_bodies = {}
+        for statement in forward_sql:
+            final_function_bodies.update(function_pattern.findall(statement))
+        migration_function_lines = {
+            f"{name}|{hashlib.sha256(body.encode('utf-8')).hexdigest()}"
+            for name, body in final_function_bodies.items()
+        }
+        self.assertEqual(migration_function_lines, expected_function_lines)
+
+        trigger_pattern = re.compile(
+            r"CREATE (?P<constraint>CONSTRAINT )?TRIGGER (?P<name>[a-z0-9_]+)\s+"
+            r"(?P<timing>BEFORE|AFTER) (?P<events>(?:INSERT|UPDATE|DELETE)"
+            r"(?: OR (?:INSERT|UPDATE|DELETE))*) ON (?P<table>[a-z0-9_]+)\s+"
+            r"(?:(?P<deferred>DEFERRABLE INITIALLY DEFERRED)\s+)?"
+            r"FOR EACH (?P<scope>ROW|STATEMENT) EXECUTE FUNCTION "
+            r"(?P<function>[a-z0-9_]+)\(\);",
+            re.DOTALL,
+        )
+        event_bits = {"INSERT": 4, "DELETE": 8, "UPDATE": 16}
+        migration_trigger_lines = set()
+        for statement in forward_sql:
+            for match in trigger_pattern.finditer(statement):
+                trigger_type = 2 if match["timing"] == "BEFORE" else 0
+                if match["scope"] == "ROW":
+                    trigger_type += 1
+                trigger_type += sum(
+                    event_bits[event]
+                    for event in match["events"].split(" OR ")
+                )
+                is_constraint = match["constraint"] is not None
+                is_deferred = match["deferred"] is not None
+                bool_text = lambda value: "true" if value else "false"
+                migration_trigger_lines.add(
+                    "|".join(
+                        (
+                            match["name"],
+                            match["table"],
+                            match["function"],
+                            str(trigger_type),
+                            bool_text(is_deferred),
+                            bool_text(is_deferred),
+                            bool_text(is_constraint),
+                        )
+                    )
+                )
+        self.assertEqual(migration_trigger_lines, set(triggers))
+
+    def test_runbook_requires_real_restore_and_ssr_rehearsal(self):
+        runbook = self.runbook.read_text(encoding="utf-8")
+        self.assertIn("do not count as a restore\nrehearsal", runbook)
+        self.assertIn('id="entry-card"', runbook)
+        self.assertIn('id="warning-card"', runbook)
+        self.assertIn("`/results/`", runbook)
+        self.assertIn("session_rows=0", runbook)
+        self.assertIn("RPO 24 hours", runbook)
+        self.assertIn("RTO 4 hours", runbook)
+        self.assertIn("RPO 24시간", runbook)
+        self.assertIn("RTO 4시간", runbook)
+        self.assertIn("Every backup failure exits nonzero", runbook)
+        self.assertIn("backup_result=...", runbook)
+        self.assertIn("Production deployment", runbook)
+        self.assertIn("`SIGKILL`, host loss", runbook)
+        self.assertIn("external exact-name check", runbook)
+
+
+if __name__ == "__main__":
+    unittest.main()
diff --git a/scripts/backup-postgresql b/scripts/backup-postgresql
new file mode 100755
index 0000000..b84eccb
--- /dev/null
+++ b/scripts/backup-postgresql
@@ -0,0 +1,191 @@
+#!/bin/sh
+
+set +x
+set -eu
+umask 077
+LC_ALL=C
+export LC_ALL
+unset PGDATABASE PGUSER PGHOST PGPORT PGSERVICE PGSERVICEFILE PGOPTIONS
+unset MOFA_TRAVEL_ALARM_SERVICE_KEY TRAVEL_READINESS_SECRET_KEY TRAVEL_READINESS_DB_PASSWORD
+
+SCRIPT_DIR=$(CDPATH='' cd "$(dirname "$0")" && pwd -P)
+# shellcheck source-path=SCRIPTDIR
+# shellcheck source=postgresql-common
+. "$SCRIPT_DIR/postgresql-common"
+
+usage() {
+    printf '%s\n' 'usage: backup-postgresql --host HOST --port PORT --database DATABASE --username USERNAME --backup-dir NEW_ABSOLUTE_DIRECTORY'
+}
+
+fail() {
+    printf '%s\n' "$1" >&2
+    exit "$2"
+}
+
+host=''
+port=''
+database=''
+username=''
+backup_dir=''
+
+while [ "$#" -gt 0 ]; do
+    case "$1" in
+        --help)
+            [ "$#" -eq 1 ] || fail 'backup_result=INVALID_ARGUMENTS' 64
+            usage
+            exit 0
+            ;;
+        --host|--port|--database|--username|--backup-dir)
+            [ "$#" -ge 2 ] || fail 'backup_result=INVALID_ARGUMENTS' 64
+            option=$1
+            value=$2
+            shift 2
+            case "$option" in
+                --host) [ -z "$host" ] || fail 'backup_result=INVALID_ARGUMENTS' 64; host=$value ;;
+                --port) [ -z "$port" ] || fail 'backup_result=INVALID_ARGUMENTS' 64; port=$value ;;
+                --database) [ -z "$database" ] || fail 'backup_result=INVALID_ARGUMENTS' 64; database=$value ;;
+                --username) [ -z "$username" ] || fail 'backup_result=INVALID_ARGUMENTS' 64; username=$value ;;
+                --backup-dir) [ -z "$backup_dir" ] || fail 'backup_result=INVALID_ARGUMENTS' 64; backup_dir=$value ;;
+            esac
+            ;;
+        *) fail 'backup_result=INVALID_ARGUMENTS' 64 ;;
+    esac
+done
+
+is_host "$host" || fail 'backup_result=INVALID_ARGUMENTS' 64
+is_port "$port" || fail 'backup_result=INVALID_ARGUMENTS' 64
+is_identifier "$database" || fail 'backup_result=INVALID_ARGUMENTS' 64
+is_identifier "$username" || fail 'backup_result=INVALID_ARGUMENTS' 64
+is_new_absolute_directory "$backup_dir" || fail 'backup_result=INVALID_ARGUMENTS' 64
+
+require_pg_tool psql || fail 'backup_result=POSTGRESQL_18_6_REQUIRED' 69
+require_pg_tool pg_dump || fail 'backup_result=POSTGRESQL_18_6_REQUIRED' 69
+command -v shasum >/dev/null 2>&1 || fail 'backup_result=REQUIRED_TOOL_MISSING' 69
+command -v awk >/dev/null 2>&1 || fail 'backup_result=REQUIRED_TOOL_MISSING' 69
+command -v cmp >/dev/null 2>&1 || fail 'backup_result=REQUIRED_TOOL_MISSING' 69
+
+database_is_postgresql_18_6 "$host" "$port" "$database" "$username" \
+    || fail 'backup_result=DATABASE_VERSION_MISMATCH' 65
+database_has_exact_tables "$host" "$port" "$database" "$username" \
+    || fail 'backup_result=SCHEMA_NOT_APPROVED' 65
+database_has_exact_schema_objects "$host" "$port" "$database" "$username" \
+    || fail 'backup_result=SCHEMA_OBJECTS_NOT_APPROVED' 65
+database_has_exact_schema_digests \
+    "$host" "$port" "$database" "$username" \
+    "$SCRIPT_DIR/postgresql-integrity.sql" \
+    || fail 'backup_result=SCHEMA_DIGEST_NOT_APPROVED' 65
+database_has_no_forbidden_storage "$host" "$port" "$database" "$username" \
+    || fail 'backup_result=FORBIDDEN_STORAGE_PRESENT' 65
+
+mkdir "$backup_dir" 2>/dev/null || fail 'backup_result=OUTPUT_CREATE_FAILED' 73
+backup_created=1
+
+cleanup() {
+    if [ "${backup_created:-0}" = "1" ]; then
+        rm -f \
+            "$backup_dir/.incomplete" \
+            "$backup_dir/snapshot.before" \
+            "$backup_dir/snapshot.after" \
+            "$backup_dir/database.dump" \
+            "$backup_dir/integrity.manifest" 2>/dev/null || :
+        rmdir "$backup_dir" 2>/dev/null || :
+    fi
+}
+
+cleanup_on_exit() {
+    original_status=$?
+    trap - EXIT HUP INT TERM
+    cleanup
+    exit "$original_status"
+}
+
+interrupt() {
+    signal_status=$1
+    trap - EXIT HUP INT TERM
+    cleanup
+    printf '%s\n' 'backup_result=INTERRUPTED' >&2
+    exit "$signal_status"
+}
+
+trap cleanup_on_exit EXIT
+trap 'interrupt 129' HUP
+trap 'interrupt 130' INT
+trap 'interrupt 143' TERM
+
+chmod 700 "$backup_dir" 2>/dev/null || fail 'backup_result=OUTPUT_CREATE_FAILED' 73
+
+: 2>/dev/null >"$backup_dir/.incomplete" \
+    || fail 'backup_result=OUTPUT_CREATE_FAILED' 73
+chmod 600 "$backup_dir/.incomplete" 2>/dev/null \
+    || fail 'backup_result=OUTPUT_CREATE_FAILED' 73
+
+write_integrity_snapshot \
+    "$host" "$port" "$database" "$username" \
+    "$SCRIPT_DIR/postgresql-integrity.sql" "$backup_dir/snapshot.before" \
+    || fail 'backup_result=SNAPSHOT_FAILED' 74
+chmod 600 "$backup_dir/snapshot.before" 2>/dev/null \
+    || fail 'backup_result=SNAPSHOT_FAILED' 74
+
+PGAPPNAME=travel-readiness-backup PGCONNECT_TIMEOUT=5 pg_dump \
+    --no-password \
+    --host="$host" \
+    --port="$port" \
+    --dbname="$database" \
+    --username="$username" \
+    --format=custom \
+    --compress=gzip:9 \
+    --no-owner \
+    --no-acl \
+    --no-comments \
+    --no-security-labels \
+    --lock-wait-timeout=5000ms \
+    --schema=public \
+    --exclude-table-data=public.django_admin_log \
+    --exclude-table-data=public.django_session \
+    --file="$backup_dir/database.dump" \
+    >/dev/null 2>&1 \
+    || fail 'backup_result=DUMP_FAILED' 74
+chmod 600 "$backup_dir/database.dump" 2>/dev/null \
+    || fail 'backup_result=DUMP_FAILED' 74
+
+write_integrity_snapshot \
+    "$host" "$port" "$database" "$username" \
+    "$SCRIPT_DIR/postgresql-integrity.sql" "$backup_dir/snapshot.after" \
+    || fail 'backup_result=SNAPSHOT_FAILED' 74
+chmod 600 "$backup_dir/snapshot.after" 2>/dev/null \
+    || fail 'backup_result=SNAPSHOT_FAILED' 74
+
+database_has_exact_tables "$host" "$port" "$database" "$username" \
+    || fail 'backup_result=SCHEMA_CHANGED_DURING_BACKUP' 75
+database_has_exact_schema_objects "$host" "$port" "$database" "$username" \
+    || fail 'backup_result=SCHEMA_CHANGED_DURING_BACKUP' 75
+database_has_exact_schema_digests \
+    "$host" "$port" "$database" "$username" \
+    "$SCRIPT_DIR/postgresql-integrity.sql" \
+    || fail 'backup_result=SCHEMA_CHANGED_DURING_BACKUP' 75
+database_has_no_forbidden_storage "$host" "$port" "$database" "$username" \
+    || fail 'backup_result=SCHEMA_CHANGED_DURING_BACKUP' 75
+
+cmp -s "$backup_dir/snapshot.before" "$backup_dir/snapshot.after" \
+    || fail 'backup_result=SOURCE_CHANGED_DURING_BACKUP' 75
+
+archive_sha256=$(sha256_file "$backup_dir/database.dump") \
+    || fail 'backup_result=HASH_FAILED' 74
+is_sha256 "$archive_sha256" || fail 'backup_result=HASH_FAILED' 74
+
+{
+    printf '%s\n' 'manifest.format=travel-readiness-postgresql-backup-v1'
+    printf 'archive.sha256=%s\n' "$archive_sha256"
+    printf '%s\n' 'archive.scope=approved-public-schema-without-ephemeral-data'
+    cat "$backup_dir/snapshot.after" 2>/dev/null
+} 2>/dev/null >"$backup_dir/integrity.manifest" \
+    || fail 'backup_result=MANIFEST_FAILED' 74
+chmod 600 "$backup_dir/integrity.manifest" 2>/dev/null \
+    || fail 'backup_result=MANIFEST_FAILED' 74
+
+rm -f "$backup_dir/snapshot.before" "$backup_dir/snapshot.after" "$backup_dir/.incomplete" \
+    2>/dev/null || fail 'backup_result=FINALIZE_FAILED' 74
+backup_created=0
+trap - EXIT HUP INT TERM
+
+printf '%s\n' 'backup_result=ok'
diff --git a/scripts/check-backup-restore b/scripts/check-backup-restore
new file mode 100755
index 0000000..38439d9
--- /dev/null
+++ b/scripts/check-backup-restore
@@ -0,0 +1,343 @@
+#!/bin/sh
+
+set +x
+set -eu
+umask 077
+LC_ALL=C
+export LC_ALL
+unset PGDATABASE PGUSER PGHOST PGPORT PGSERVICE PGSERVICEFILE PGOPTIONS PGPASSWORD
+unset MOFA_TRAVEL_ALARM_SERVICE_KEY TRAVEL_READINESS_SECRET_KEY TRAVEL_READINESS_DB_PASSWORD
+
+usage() {
+    printf '%s\n' 'usage: check-backup-restore --host LOOPBACK --port PORT --admin-role ROLE --admin-password-env TRAVEL_READINESS_BACKUP_RESTORE_ADMIN_PASSWORD --backup-role ROLE --backup-password-env TRAVEL_READINESS_BACKUP_ROLE_PASSWORD --source-database DATABASE --database-prefix travel_readiness_restorecheck_NAME --safety-token BACKUP_RESTORE_REHEARSAL_DISPOSABLE:travel_readiness_restorecheck_NAME --writers-quiesced-confirmation WRITERS_QUIESCED'
+}
+
+fail() {
+    printf '%s\n' "$1" >&2
+    exit "$2"
+}
+
+is_identifier() {
+    value=$1
+    [ -n "$value" ] || return 1
+    [ "${#value}" -le 63 ] || return 1
+    case "$value" in [a-z_]*) ;; *) return 1 ;; esac
+    case "$value" in *[!a-z0-9_]*) return 1 ;; esac
+}
+
+is_port() {
+    value=$1
+    case "$value" in ''|*[!0-9]*) return 1 ;; esac
+    [ "${#value}" -le 5 ] || return 1
+    [ "$value" -ge 1 ] 2>/dev/null || return 1
+    [ "$value" -le 65535 ] 2>/dev/null || return 1
+}
+
+host=''
+port=''
+admin_role=''
+admin_password_env=''
+backup_role=''
+backup_password_env=''
+source_database=''
+database_prefix=''
+safety_token=''
+writers_quiesced_confirmation=''
+
+while [ "$#" -gt 0 ]; do
+    case "$1" in
+        --help)
+            [ "$#" -eq 1 ] || fail 'backup_restore_check=INVALID_ARGUMENTS' 64
+            usage
+            exit 0
+            ;;
+        --host|--port|--admin-role|--admin-password-env|--backup-role|--backup-password-env|--source-database|--database-prefix|--safety-token|--writers-quiesced-confirmation)
+            [ "$#" -ge 2 ] || fail 'backup_restore_check=INVALID_ARGUMENTS' 64
+            option=$1
+            option_value=$2
+            shift 2
+            case "$option" in
+                --host) [ -z "$host" ] || fail 'backup_restore_check=INVALID_ARGUMENTS' 64; host=$option_value ;;
+                --port) [ -z "$port" ] || fail 'backup_restore_check=INVALID_ARGUMENTS' 64; port=$option_value ;;
+                --admin-role) [ -z "$admin_role" ] || fail 'backup_restore_check=INVALID_ARGUMENTS' 64; admin_role=$option_value ;;
+                --admin-password-env) [ -z "$admin_password_env" ] || fail 'backup_restore_check=INVALID_ARGUMENTS' 64; admin_password_env=$option_value ;;
+                --backup-role) [ -z "$backup_role" ] || fail 'backup_restore_check=INVALID_ARGUMENTS' 64; backup_role=$option_value ;;
+                --backup-password-env) [ -z "$backup_password_env" ] || fail 'backup_restore_check=INVALID_ARGUMENTS' 64; backup_password_env=$option_value ;;
+                --source-database) [ -z "$source_database" ] || fail 'backup_restore_check=INVALID_ARGUMENTS' 64; source_database=$option_value ;;
+                --database-prefix) [ -z "$database_prefix" ] || fail 'backup_restore_check=INVALID_ARGUMENTS' 64; database_prefix=$option_value ;;
+                --safety-token) [ -z "$safety_token" ] || fail 'backup_restore_check=INVALID_ARGUMENTS' 64; safety_token=$option_value ;;
+                --writers-quiesced-confirmation) [ -z "$writers_quiesced_confirmation" ] || fail 'backup_restore_check=INVALID_ARGUMENTS' 64; writers_quiesced_confirmation=$option_value ;;
+            esac
+            ;;
+        *) fail 'backup_restore_check=INVALID_ARGUMENTS' 64 ;;
+    esac
+done
+
+[ -n "$host" ] && [ -n "$port" ] && [ -n "$admin_role" ] \
+    && [ -n "$admin_password_env" ] && [ -n "$backup_role" ] \
+    && [ -n "$backup_password_env" ] && [ -n "$source_database" ] \
+    && [ -n "$database_prefix" ] && [ -n "$safety_token" ] \
+    && [ -n "$writers_quiesced_confirmation" ] \
+    || fail 'backup_restore_check=INVALID_ARGUMENTS' 64
+case "$host" in 127.0.0.1|localhost) ;; *) fail 'backup_restore_check=NON_LOOPBACK_REFUSED' 65 ;; esac
+is_port "$port" || fail 'backup_restore_check=INVALID_ARGUMENTS' 64
+is_identifier "$admin_role" || fail 'backup_restore_check=INVALID_ARGUMENTS' 64
+is_identifier "$backup_role" || fail 'backup_restore_check=INVALID_ARGUMENTS' 64
+is_identifier "$source_database" || fail 'backup_restore_check=INVALID_ARGUMENTS' 64
+is_identifier "$database_prefix" || fail 'backup_restore_check=UNSAFE_PREFIX' 65
+[ "${#database_prefix}" -le 42 ] || fail 'backup_restore_check=UNSAFE_PREFIX' 65
+case "$database_prefix" in
+    travel_readiness_restorecheck_[a-z0-9]*) ;;
+    *) fail 'backup_restore_check=UNSAFE_PREFIX' 65 ;;
+esac
+case "$database_prefix" in
+    *prod*|*live*|*stag*|*main*|*master*|*release*)
+        fail 'backup_restore_check=PRODUCTION_LIKE_PREFIX_REFUSED' 65
+        ;;
+esac
+[ "$safety_token" = "BACKUP_RESTORE_REHEARSAL_DISPOSABLE:$database_prefix" ] \
+    || fail 'backup_restore_check=SAFETY_TOKEN_MISMATCH' 65
+[ "$writers_quiesced_confirmation" = 'WRITERS_QUIESCED' ] \
+    || fail 'backup_restore_check=WRITER_QUIESCENCE_REQUIRED' 65
+[ "$admin_password_env" = 'TRAVEL_READINESS_BACKUP_RESTORE_ADMIN_PASSWORD' ] \
+    || fail 'backup_restore_check=UNSAFE_PASSWORD_REFERENCE' 65
+[ "$backup_password_env" = 'TRAVEL_READINESS_BACKUP_ROLE_PASSWORD' ] \
+    || fail 'backup_restore_check=UNSAFE_PASSWORD_REFERENCE' 65
+admin_password=${TRAVEL_READINESS_BACKUP_RESTORE_ADMIN_PASSWORD-}
+unset TRAVEL_READINESS_BACKUP_RESTORE_ADMIN_PASSWORD
+[ -n "$admin_password" ] || fail 'backup_restore_check=ADMIN_PASSWORD_MISSING' 66
+[ "${#admin_password}" -le 1024 ] || fail 'backup_restore_check=ADMIN_PASSWORD_INVALID' 66
+case "$admin_password" in
+    *'
+'*) fail 'backup_restore_check=ADMIN_PASSWORD_INVALID' 66 ;;
+esac
+backup_password=${TRAVEL_READINESS_BACKUP_ROLE_PASSWORD-}
+unset TRAVEL_READINESS_BACKUP_ROLE_PASSWORD
+[ -n "$backup_password" ] || fail 'backup_restore_check=BACKUP_PASSWORD_MISSING' 66
+[ "${#backup_password}" -le 1024 ] || fail 'backup_restore_check=BACKUP_PASSWORD_INVALID' 66
+case "$backup_password" in
+    *'
+'*) fail 'backup_restore_check=BACKUP_PASSWORD_INVALID' 66 ;;
+esac
+
+script_dir=$(CDPATH='' cd "$(dirname "$0")" && pwd -P)
+project_dir=$(CDPATH='' cd "$script_dir/.." && pwd -P)
+python_bin="$project_dir/.venv/bin/python"
+# shellcheck source-path=SCRIPTDIR
+# shellcheck source=postgresql-common
+. "$script_dir/postgresql-common"
+
+require_pg_tool psql || fail 'backup_restore_check=POSTGRESQL_18_6_REQUIRED' 69
+command -v openssl >/dev/null 2>&1 || fail 'backup_restore_check=REQUIRED_TOOL_MISSING' 69
+command -v mktemp >/dev/null 2>&1 || fail 'backup_restore_check=REQUIRED_TOOL_MISSING' 69
+command -v wc >/dev/null 2>&1 || fail 'backup_restore_check=REQUIRED_TOOL_MISSING' 69
+command -v tr >/dev/null 2>&1 || fail 'backup_restore_check=REQUIRED_TOOL_MISSING' 69
+command -v sed >/dev/null 2>&1 || fail 'backup_restore_check=REQUIRED_TOOL_MISSING' 69
+[ -x "$python_bin" ] || fail 'backup_restore_check=PINNED_PYTHON_REQUIRED' 69
+
+database_suffix=${database_prefix#travel_readiness_restorecheck_}
+restore_database="travel_readiness_restore_${database_suffix}"
+restore_role="${database_prefix}_restore"
+is_identifier "$restore_database" || fail 'backup_restore_check=UNSAFE_PREFIX' 65
+is_identifier "$backup_role" || fail 'backup_restore_check=UNSAFE_PREFIX' 65
+is_identifier "$restore_role" || fail 'backup_restore_check=UNSAFE_PREFIX' 65
+[ "$source_database" != "$restore_database" ] || fail 'backup_restore_check=SOURCE_TARGET_COLLISION' 65
+[ "$admin_role" != "$backup_role" ] || fail 'backup_restore_check=BACKUP_ROLE_NOT_SEPARATE' 65
+[ "$admin_role" != "$restore_role" ] || fail 'backup_restore_check=ADMIN_ROLE_COLLISION' 65
+[ "$backup_role" != "$restore_role" ] || fail 'backup_restore_check=BACKUP_ROLE_NOT_SEPARATE' 65
+
+admin_psql() {
+    connection_database=$1
+    shift
+    PGPASSWORD="$admin_password" PGAPPNAME=travel-readiness-backup-restore-admin \
+    PGCONNECT_TIMEOUT=5 psql --no-password --host="$host" --port="$port" \
+        --dbname="$connection_database" --username="$admin_role" --no-psqlrc \
+        --set=ON_ERROR_STOP=1 "$@"
+}
+
+admin_scalar() {
+    query=$1
+    admin_psql postgres --quiet --tuples-only --no-align --command="$query" \
+        2>/dev/null
+}
+
+server_version=$(admin_scalar "SELECT current_setting('server_version_num')") \
+    || fail 'backup_restore_check=ADMIN_CONNECTION_FAILED' 70
+[ "$server_version" = 180006 ] || fail 'backup_restore_check=DATABASE_VERSION_MISMATCH' 70
+admin_superuser=$(admin_scalar "SELECT rolsuper FROM pg_catalog.pg_roles WHERE rolname = current_user") \
+    || fail 'backup_restore_check=ADMIN_CONNECTION_FAILED' 70
+[ "$admin_superuser" = t ] || fail 'backup_restore_check=ADMIN_CAPABILITY_REQUIRED' 70
+source_exists=$(admin_scalar "SELECT count(*) FROM pg_catalog.pg_database WHERE datname = '$source_database'") \
+    || fail 'backup_restore_check=PREFLIGHT_FAILED' 70
+[ "$source_exists" = 1 ] || fail 'backup_restore_check=SOURCE_DATABASE_MISSING' 70
+existing_targets=$(admin_scalar "SELECT (SELECT count(*) FROM pg_catalog.pg_database WHERE datname = '$restore_database') + (SELECT count(*) FROM pg_catalog.pg_roles WHERE rolname = '$restore_role')") \
+    || fail 'backup_restore_check=PREFLIGHT_FAILED' 70
+[ "$existing_targets" = 0 ] || fail 'backup_restore_check=TARGET_ALREADY_EXISTS' 71
+
+backup_privileges=$(admin_psql "$source_database" --quiet --tuples-only --no-align --command="SELECT (SELECT rolcanlogin AND NOT rolsuper AND NOT rolcreatedb AND NOT rolcreaterole AND NOT rolinherit AND NOT rolreplication AND NOT rolbypassrls FROM pg_catalog.pg_roles WHERE rolname = '$backup_role') AND NOT EXISTS (SELECT 1 FROM pg_catalog.pg_auth_members AS m JOIN pg_catalog.pg_roles AS r ON r.oid = m.member WHERE r.rolname = '$backup_role') AND NOT EXISTS (SELECT 1 FROM pg_catalog.pg_shdepend AS d JOIN pg_catalog.pg_roles AS r ON r.oid = d.refobjid WHERE r.rolname = '$backup_role' AND d.refclassid = 'pg_catalog.pg_authid'::pg_catalog.regclass AND d.deptype = 'o') AND has_database_privilege('$backup_role', '$source_database', 'CONNECT') AND NOT has_database_privilege('$backup_role', '$source_database', 'CREATE') AND NOT has_database_privilege('$backup_role', '$source_database', 'TEMPORARY') AND has_schema_privilege('$backup_role', 'public', 'USAGE') AND NOT has_schema_privilege('$backup_role', 'public', 'CREATE') AND (SELECT bool_and(has_table_privilege('$backup_role', c.oid, 'SELECT') AND NOT has_table_privilege('$backup_role', c.oid, 'INSERT') AND NOT has_table_privilege('$backup_role', c.oid, 'UPDATE') AND NOT has_table_privilege('$backup_role', c.oid, 'DELETE') AND NOT has_table_privilege('$backup_role', c.oid, 'TRUNCATE') AND NOT has_table_privilege('$backup_role', c.oid, 'REFERENCES') AND NOT has_table_privilege('$backup_role', c.oid, 'TRIGGER') AND NOT has_table_privilege('$backup_role', c.oid, 'MAINTAIN') AND NOT has_any_column_privilege('$backup_role', c.oid, 'INSERT') AND NOT has_any_column_privilege('$backup_role', c.oid, 'UPDATE') AND NOT has_any_column_privilege('$backup_role', c.oid, 'REFERENCES')) FROM pg_catalog.pg_class AS c JOIN pg_catalog.pg_namespace AS n ON n.oid = c.relnamespace WHERE n.nspname = 'public' AND c.relkind = 'r') AND (SELECT bool_and(has_sequence_privilege('$backup_role', c.oid, 'SELECT') AND NOT has_sequence_privilege('$backup_role', c.oid, 'USAGE') AND NOT has_sequence_privilege('$backup_role', c.oid, 'UPDATE')) FROM pg_catalog.pg_class AS c JOIN pg_catalog.pg_namespace AS n ON n.oid = c.relnamespace WHERE n.nspname = 'public' AND c.relkind = 'S')" 2>/dev/null) \
+    || fail 'backup_restore_check=BACKUP_PRIVILEGE_CHECK_FAILED' 70
+[ "$backup_privileges" = t ] \
+    || fail 'backup_restore_check=BACKUP_ROLE_NOT_READ_ONLY' 70
+
+restore_password=$(openssl rand -hex 24 2>/dev/null) \
+    || fail 'backup_restore_check=PASSWORD_GENERATION_FAILED' 69
+django_secret=$(openssl rand -hex 32 2>/dev/null) \
+    || fail 'backup_restore_check=PASSWORD_GENERATION_FAILED' 69
+case "$restore_password$django_secret" in
+    *[!a-f0-9]*) fail 'backup_restore_check=PASSWORD_GENERATION_FAILED' 69 ;;
+esac
+backup_pgpass_password=$(escape_pgpass_field "$backup_password") \
+    || fail 'backup_restore_check=PASSWORD_ESCAPE_FAILED' 69
+restore_pgpass_password=$(escape_pgpass_field "$restore_password") \
+    || fail 'backup_restore_check=PASSWORD_ESCAPE_FAILED' 69
+
+work_dir=$(mktemp -d /tmp/travel-readiness-backup-restore.XXXXXX 2>/dev/null) \
+    || fail 'backup_restore_check=TEMP_CREATE_FAILED' 73
+case "$work_dir" in /tmp/travel-readiness-backup-restore.*) ;; *) fail 'backup_restore_check=TEMP_CREATE_FAILED' 73 ;; esac
+backup_dir="$work_dir/backup"
+pgpass_file="$work_dir/pgpass"
+restore_role_created=0
+restore_database_created=0
+
+cleanup() {
+    cleanup_result=0
+    if [ "$restore_database_created" = 1 ]; then
+        admin_psql postgres --quiet --command="REVOKE CONNECT ON DATABASE \"$restore_database\" FROM PUBLIC, \"$restore_role\"" >/dev/null 2>&1 || cleanup_result=1
+        admin_psql postgres --quiet --command="SELECT pg_catalog.pg_terminate_backend(pid) FROM pg_catalog.pg_stat_activity WHERE datname = '$restore_database' AND pid <> pg_catalog.pg_backend_pid()" >/dev/null 2>&1 || cleanup_result=1
+        admin_psql postgres --quiet --command="DROP DATABASE \"$restore_database\"" >/dev/null 2>&1 || cleanup_result=1
+    fi
+    if [ "$restore_role_created" = 1 ]; then
+        admin_psql postgres --quiet --command="DROP ROLE \"$restore_role\"" >/dev/null 2>&1 || cleanup_result=1
+    fi
+    rm -f -- "$pgpass_file" "$backup_dir/database.dump" \
+        "$backup_dir/integrity.manifest" 2>/dev/null || cleanup_result=1
+    rmdir "$backup_dir" 2>/dev/null || [ ! -e "$backup_dir" ] || cleanup_result=1
+    rmdir "$work_dir" 2>/dev/null || cleanup_result=1
+    return "$cleanup_result"
+}
+
+cleanup_on_exit() {
+    original_status=$?
+    trap - EXIT HUP INT TERM
+    if ! cleanup; then
+        printf '%s\n' 'backup_restore_check_cleanup=FAILED' >&2
+        exit 77
+    fi
+    exit "$original_status"
+}
+trap cleanup_on_exit EXIT
+trap 'exit 129' HUP
+trap 'exit 130' INT
+trap 'exit 143' TERM
+
+rehearsal_started=$(date +%s)
+printf '%s\n' "CREATE ROLE \"$restore_role\" LOGIN NOSUPERUSER NOCREATEDB NOCREATEROLE NOINHERIT NOREPLICATION NOBYPASSRLS PASSWORD '$restore_password';" \
+    | admin_psql postgres --quiet \
+    >/dev/null 2>&1 || fail 'backup_restore_check=ROLE_CREATE_FAILED' 72
+restore_role_created=1
+printf '%s\n' "CREATE DATABASE \"$restore_database\" OWNER \"$restore_role\" TEMPLATE template0;" \
+    | admin_psql postgres --quiet >/dev/null 2>&1 \
+    || fail 'backup_restore_check=DATABASE_CREATE_FAILED' 72
+restore_database_created=1
+admin_psql "$restore_database" --quiet --command='DROP SCHEMA public' \
+    >/dev/null 2>&1 || fail 'backup_restore_check=TARGET_PREPARE_FAILED' 72
+
+{
+    printf '%s:%s:%s:%s:%s\n' "$host" "$port" "$source_database" "$backup_role" "$backup_pgpass_password"
+    printf '%s:%s:%s:%s:%s\n' "$host" "$port" "$restore_database" "$restore_role" "$restore_pgpass_password"
+} >"$pgpass_file"
+chmod 600 "$pgpass_file"
+
+backup_started=$(date +%s)
+PGPASSFILE="$pgpass_file" "$script_dir/backup-postgresql" \
+    --host "$host" --port "$port" --database "$source_database" \
+    --username "$backup_role" --backup-dir "$backup_dir"
+backup_finished=$(date +%s)
+backup_bytes=$(wc -c <"$backup_dir/database.dump" | tr -d ' ')
+case "$backup_bytes" in ''|*[!0-9]*) fail 'backup_restore_check=SIZE_FAILED' 74 ;; esac
+[ "$backup_bytes" -gt 0 ] || fail 'backup_restore_check=SIZE_FAILED' 74
+
+restore_started=$(date +%s)
+PGPASSFILE="$pgpass_file" "$script_dir/restore-postgresql" \
+    --host "$host" --port "$port" --database "$restore_database" \
+    --username "$restore_role" --backup-dir "$backup_dir" \
+    --safety-token "RESTORE_DISPOSABLE:$restore_database"
+restore_finished=$(date +%s)
+
+TRAVEL_READINESS_SECRET_KEY="$django_secret" \
+TRAVEL_READINESS_DB_NAME="$restore_database" \
+TRAVEL_READINESS_DB_USER="$restore_role" \
+TRAVEL_READINESS_DB_PASSWORD="$restore_password" \
+TRAVEL_READINESS_DB_HOST="$host" \
+TRAVEL_READINESS_DB_PORT="$port" \
+TRAVEL_READINESS_ALLOWED_HOSTS='testserver,localhost' \
+TRAVEL_READINESS_BUILD=0 TRAVEL_READINESS_DEBUG=0 TRAVEL_READINESS_HTTPS=0 \
+DJANGO_SETTINGS_MODULE=travel_readiness.settings PYTHONPATH="$project_dir" \
+    "$python_bin" -s -c '
+import django
+django.setup()
+from django.test import Client
+from django.utils.html import escape
+from reviews.models import PublishedEntryFacts, PublishedTravelWarning
+def require(condition):
+    if not condition:
+        raise SystemExit(1)
+
+entry_pointer = PublishedEntryFacts.objects.select_related("current_publication__entry_fact_revision__country").get()
+warning_pointer = PublishedTravelWarning.objects.select_related("current_publication__travel_warning_revision__country").get()
+entry = entry_pointer.current_publication
+warning = warning_pointer.current_publication
+require(entry is not None and warning is not None)
+require(entry_pointer.version == entry.generation)
+require(warning_pointer.version == warning.generation)
+require(entry.entry_fact_revision.ordinary_passport_period_text == "90일")
+require(entry.entry_fact_revision.country.iso_alpha2 == "JP")
+require(warning.travel_warning_revision.source_alarm_level_code == "3")
+require(warning.travel_warning_revision.country.iso_alpha2 == "JP")
+response = Client().get("/results/", secure=True)
+require(response.status_code == 200)
+require(response.headers.get("Cache-Control") == "no-store")
+require(not response.cookies)
+body = response.content.decode("utf-8")
+require(body.count("id=\"entry-card\"") == 1)
+require(body.count("id=\"warning-card\"") == 1)
+entry_start = body.index("id=\"entry-card\"")
+warning_start = body.index("id=\"warning-card\"")
+entry_card = body[entry_start:warning_start]
+warning_card = body[warning_start:]
+require(any(f"data-state=\"{state}\"" in entry_card for state in ("ready", "stale")))
+require(any(f"data-state=\"{state}\"" in warning_card for state in ("ready", "stale")))
+require(f"generation {entry.generation}" in entry_card)
+require(f"generation {warning.generation}" in warning_card)
+require(entry.source_owner_snapshot in entry_card)
+require(entry.attribution_text_snapshot in entry_card)
+require(f"href=\"{escape(entry.source_locator_snapshot)}\"" in entry_card)
+require(warning.source_owner_snapshot in warning_card)
+require(warning.attribution_text_snapshot in warning_card)
+require(f"href=\"{escape(warning.source_locator_snapshot)}\"" in warning_card)
+for forbidden in ("ALLOWED", "DENIED", "입국 가능", "법적 판단"):
+    require(forbidden not in body)
+' >/dev/null 2>&1 || fail 'backup_restore_check=SSR_VERIFY_FAILED' 76
+
+session_admin_counts=$(PGPASSFILE="$pgpass_file" run_psql \
+    "$host" "$port" "$restore_database" "$restore_role" \
+    --quiet --tuples-only --no-align \
+    --command="SELECT (SELECT count(*) FROM public.django_session)::text || ':' || (SELECT count(*) FROM public.django_admin_log)::text" \
+    2>/dev/null) || fail 'backup_restore_check=EPHEMERAL_CHECK_FAILED' 76
+[ "$session_admin_counts" = '0:0' ] \
+    || fail 'backup_restore_check=EPHEMERAL_DATA_PRESENT' 76
+
+if ! cleanup; then
+    trap - EXIT HUP INT TERM
+    fail 'backup_restore_check=CLEANUP_FAILED' 77
+fi
+restore_database_created=0
+restore_role_created=0
+trap - EXIT HUP INT TERM
+rehearsal_finished=$(date +%s)
+unset admin_password backup_password backup_pgpass_password
+unset restore_password restore_pgpass_password django_secret
+
+printf 'backup_restore_check=ok writers_quiesced=confirmed backup_seconds=%s restore_seconds=%s rehearsal_seconds=%s backup_bytes=%s session_rows=0 admin_log_rows=0 integrity_manifest=match publication_pointers=match ssr_results_status=200 entry_marker=match travel_warning_marker=match source_attribution=match cleanup=match\n' \
+    "$((backup_finished - backup_started))" "$((restore_finished - restore_started))" \
+    "$((rehearsal_finished - rehearsal_started))" "$backup_bytes"
diff --git a/scripts/postgresql-common b/scripts/postgresql-common
new file mode 100644
index 0000000..31d8901
--- /dev/null
+++ b/scripts/postgresql-common
@@ -0,0 +1,456 @@
+#!/bin/sh
+
+# Shared, non-interactive PostgreSQL backup/restore safety helpers.
+# This file is sourced by backup-postgresql and restore-postgresql.
+
+POSTGRESQL_REQUIRED_VERSION="18.6"
+EXPECTED_PUBLIC_SCHEMA_COMMENT_SHA256="998e3ff6338d1b13197feaef3c7657578604caf2185d3fe9e5ac04ba171682c8"
+
+EXPECTED_SCHEMA_DIGESTS='schema.columns.sha256=450ce02bd3aa172d92c00f2cbc736ac851f9e39089396fd85dbdf240fe8dd7e6
+schema.constraints.sha256=36b698ba6fe1f5a3493b90faf2a15bfdcf98c895159a530aa7906d2447fa8cc7
+schema.indexes.sha256=d85db30e3d8cd7fbfed2bf984e398b88275c138ff7f6d897625dde855b5b0fdd
+schema.trigger_functions.sha256=9a3322cb65b144f59a63eef53d1ca0ec0864f3f5fb8c06a6af24fbd92b178568
+schema.triggers.sha256=feba15c435e509614846c4a1a773e038ecff1c75dbc796f6454295ffc8b244be'
+
+EXPECTED_SEQUENCE_DEFINITION_SHA256="72e22883eef7beb6b8ab681c7833c0e37ee0c32dcf8ab481e56bf6e1228667e5"
+
+EXPECTED_PUBLIC_TABLES='auth_group
+auth_group_permissions
+auth_permission
+auth_user
+auth_user_groups
+auth_user_user_permissions
+countries_country
+django_admin_log
+django_content_type
+django_migrations
+django_session
+entry_requirements_entryfactrevision
+entry_requirements_passportpolicy
+reviews_auditevent
+reviews_publicationrevision
+reviews_publishedentryfacts
+reviews_publishedtravelwarning
+reviews_reviewdecision
+sources_fetchattempt
+sources_parserun
+sources_sourceartifact
+sources_sourceconfiguration
+sources_sourcerightsdecision
+travel_warnings_travelwarningrevision'
+
+EXPECTED_PUBLIC_SEQUENCES='auth_group_id_seq
+auth_group_permissions_id_seq
+auth_permission_id_seq
+auth_user_groups_id_seq
+auth_user_id_seq
+auth_user_user_permissions_id_seq
+django_admin_log_id_seq
+django_content_type_id_seq
+django_migrations_id_seq'
+
+EXPECTED_SEQUENCE_OWNERSHIP='auth_group_id_seq|auth_group|id
+auth_group_permissions_id_seq|auth_group_permissions|id
+auth_permission_id_seq|auth_permission|id
+auth_user_groups_id_seq|auth_user_groups|id
+auth_user_id_seq|auth_user|id
+auth_user_user_permissions_id_seq|auth_user_user_permissions|id
+django_admin_log_id_seq|django_admin_log|id
+django_content_type_id_seq|django_content_type|id
+django_migrations_id_seq|django_migrations|id'
+
+# Function bodies are the SHA-256 of pg_proc.prosrc. These values are derived
+# from the final forward migration state, including every CREATE OR REPLACE.
+EXPECTED_TRIGGER_FUNCTIONS='countries_reject_country_mutation|2c24befcb088485a5457f26bfbeb88317a6b0404ea2f79500ae51ceec7c8af7f
+entry_requirements_guard_fact_revision|7b29941d34ea432b169697c44368b857ab5ba6482ce67db19560f090ab690989
+entry_requirements_reject_policy_mutation|bb0e61d8492ed4e5daafb44553f1cf738bc0a46954f611e5298f93ad021f3ef9
+reviews_enforce_deferred_closure|dea6ac39c8852e21c032db3f58201f6d0b9138aef935738cee2fc815df4e6f05
+reviews_guard_audit_event|f9e4f24b27d120b7fadaf269a65d089105c11bd3d3d62cc5a9657f7e80d2d78c
+reviews_guard_entry_pointer|ceb7622f76da0bd04f83161641be7f3eb164b9aec5f96982d1d52dc987ee2efb
+reviews_guard_publication_revision|fb05f01e0d8ba85e61b47742e0299ff2cbf6e4accdc84c425c173dfbc0ca56aa
+reviews_guard_review_decision|0e3a7bff4a65f80f0064e3eaea56d2a816b5e716f2c807ff61402b1e5df16f9e
+reviews_guard_warning_pointer|489e84fd8da3d102cbc09926ab383766d1775928e09d64f16248d07f677a1afc
+reviews_prelock_audit_insert|b25ce76e52f602c3cda0bd154954144fa4e14e8caa1c44c496dda44bfbd11aa4
+reviews_prelock_pointer_statement|a8a60c63736dd9d81a1e64742a7119bc7f1bc78210ad5813a590e59a0c52c259
+reviews_prelock_publication_insert|52b934439fb39669e2e1aa56a28c88d5f5ff3c42bbf158a05ec0ab14ed3ae7ce
+reviews_prelock_review_insert|b7609586516ed9a27239b8b095c489aa6ab0f4dd3472559c62e3a8abce48d041
+sources_apply_rights_rejection|e02368696d3a102bd0c7cbfd2f1078b58c4d63f96046c625ea773632595a12d0
+sources_guard_artifact_mutation|0a1690c980d4e413defd6a592adbe2ae4b87c7896a613cf8e26e83cb2a157946
+sources_guard_configuration_change|1eb4fbbf3cfe95d940364bc8f36d278b044739409100df1b5a9ad42049334fc7
+sources_guard_fetch_attempt_mutation|78195da0ef6c4dfc3012edfbc98dcb8323ee2d09fd1e52c0d1f999b5391c98c4
+sources_guard_parse_run_change|0236312fea9a24334ee624e4721ccd6e19a2625722dd6958d92a3d40b6cc14cc
+sources_reject_rights_mutation|76be8ad85eab7939fe21596429e52e83ad1c7e1a1cac6954244a52606a730d46
+sources_reject_rights_revision_reuse|f96554369f58cd3017114acfd0e8772c9c7fd0252d8ebcbf493359af36310fad
+sources_validate_artifact_insert|e4e7711cc826af34a8a1638e21be3d3c01a6db25824230b4cf31d0238e0ea820
+sources_validate_fetch_attempt_insert|633cb98ad1ed6d016032902c3f0946a3ade515eb4b838b03e92ebc6b3fc58f08
+sources_validate_rights_insert|aef6e2ae5686f25b00295aa24c2cb8466cd882e7b41f4526b295e50a2a9ee182
+travel_warnings_guard_revision_change|da8c78f96078e7d0ddd2e6632a11843c643864e5d4bac3a9b6a08a7322f77f30
+travel_warnings_validate_revision_source_rights|10aed086d8e3bc8750df862907e87d2bba16e05e533270c3994ecd3757e4e01b'
+
+EXPECTED_PUBLIC_TRIGGERS='countries_country_immutable_guard|countries_country|countries_reject_country_mutation|27|false|false|false
+entry_requirements_fact_revision_guard|entry_requirements_entryfactrevision|entry_requirements_guard_fact_revision|31|false|false|false
+entry_requirements_policy_immutable_guard|entry_requirements_passportpolicy|entry_requirements_reject_policy_mutation|27|false|false|false
+reviews_00_audit_insert_prelock|reviews_auditevent|reviews_prelock_audit_insert|7|false|false|false
+reviews_00_entry_pointer_statement_lock|reviews_publishedentryfacts|reviews_prelock_pointer_statement|30|false|false|false
+reviews_00_publication_insert_prelock|reviews_publicationrevision|reviews_prelock_publication_insert|7|false|false|false
+reviews_00_review_insert_prelock|reviews_reviewdecision|reviews_prelock_review_insert|7|false|false|false
+reviews_00_warning_pointer_statement_lock|reviews_publishedtravelwarning|reviews_prelock_pointer_statement|30|false|false|false
+reviews_audit_deferred_closure|reviews_auditevent|reviews_enforce_deferred_closure|5|true|true|true
+reviews_audit_event_guard|reviews_auditevent|reviews_guard_audit_event|31|false|false|false
+reviews_entry_pointer_deferred_closure|reviews_publishedentryfacts|reviews_enforce_deferred_closure|17|true|true|true
+reviews_entry_pointer_guard|reviews_publishedentryfacts|reviews_guard_entry_pointer|31|false|false|false
+reviews_publication_deferred_closure|reviews_publicationrevision|reviews_enforce_deferred_closure|5|true|true|true
+reviews_publication_revision_guard|reviews_publicationrevision|reviews_guard_publication_revision|31|false|false|false
+reviews_review_decision_guard|reviews_reviewdecision|reviews_guard_review_decision|31|false|false|false
+reviews_review_deferred_closure|reviews_reviewdecision|reviews_enforce_deferred_closure|5|true|true|true
+reviews_warning_pointer_deferred_closure|reviews_publishedtravelwarning|reviews_enforce_deferred_closure|17|true|true|true
+reviews_warning_pointer_guard|reviews_publishedtravelwarning|reviews_guard_warning_pointer|31|false|false|false
+sources_artifact_insert_guard|sources_sourceartifact|sources_validate_artifact_insert|7|false|false|false
+sources_artifact_mutation_guard|sources_sourceartifact|sources_guard_artifact_mutation|27|false|false|false
+sources_configuration_change_guard|sources_sourceconfiguration|sources_guard_configuration_change|23|false|false|false
+sources_configuration_revision_reuse_guard|sources_sourceconfiguration|sources_reject_rights_revision_reuse|19|false|false|false
+sources_fetch_attempt_insert_guard|sources_fetchattempt|sources_validate_fetch_attempt_insert|7|false|false|false
+sources_fetch_attempt_mutation_guard|sources_fetchattempt|sources_guard_fetch_attempt_mutation|27|false|false|false
+sources_parse_run_change_guard|sources_parserun|sources_guard_parse_run_change|31|false|false|false
+sources_rights_append_only_guard|sources_sourcerightsdecision|sources_reject_rights_mutation|27|false|false|false
+sources_rights_insert_guard|sources_sourcerightsdecision|sources_validate_rights_insert|7|false|false|false
+sources_rights_rejection_gate|sources_sourcerightsdecision|sources_apply_rights_rejection|5|false|false|false
+travel_warnings_revision_change_guard|travel_warnings_travelwarningrevision|travel_warnings_guard_revision_change|31|false|false|false
+travel_warnings_revision_source_rights_guard|travel_warnings_travelwarningrevision|travel_warnings_validate_revision_source_rights|7|false|false|false'
+
+is_identifier() {
+    value=$1
+    [ -n "$value" ] || return 1
+    [ "${#value}" -le 63 ] || return 1
+    case "$value" in
+        [a-z_]* ) ;;
+        * ) return 1 ;;
+    esac
+    case "$value" in
+        *[!a-z0-9_]* ) return 1 ;;
+    esac
+}
+
+is_host() {
+    value=$1
+    [ -n "$value" ] || return 1
+    [ "${#value}" -le 253 ] || return 1
+    case "$value" in
+        .*|-*|*..*|*.-*|*-.*|*.|*[!A-Za-z0-9.-]* ) return 1 ;;
+    esac
+}
+
+is_port() {
+    value=$1
+    case "$value" in
+        ''|*[!0-9]* ) return 1 ;;
+    esac
+    [ "${#value}" -le 5 ] || return 1
+    [ "$value" -ge 1 ] 2>/dev/null || return 1
+    [ "$value" -le 65535 ] 2>/dev/null || return 1
+}
+
+escape_pgpass_field() {
+    printf '%s' "$1" | sed -e 's/\\/\\\\/g' -e 's/:/\\:/g'
+}
+
+is_new_absolute_directory() {
+    value=$1
+    case "$value" in
+        /* ) ;;
+        * ) return 1 ;;
+    esac
+    case "$value" in
+        *[!A-Za-z0-9_./-]*|*//*|*/../*|*/..|*/./*|*/.) return 1 ;;
+    esac
+    [ "$value" != "/" ] || return 1
+    [ ! -e "$value" ] && [ ! -L "$value" ] || return 1
+    parent=${value%/*}
+    [ -n "$parent" ] || return 1
+    [ "$parent" != "/" ] || return 1
+    [ -d "$parent" ] && [ ! -L "$parent" ]
+}
+
+is_backup_directory() {
+    value=$1
+    case "$value" in
+        /* ) ;;
+        * ) return 1 ;;
+    esac
+    case "$value" in
+        *[!A-Za-z0-9_./-]*|*//*|*/../*|*/..|*/./*|*/.) return 1 ;;
+    esac
+    [ "$value" != "/" ] || return 1
+    [ -d "$value" ] && [ ! -L "$value" ] || return 1
+    [ -f "$value/database.dump" ] && [ ! -L "$value/database.dump" ] || return 1
+    [ -f "$value/integrity.manifest" ] && [ ! -L "$value/integrity.manifest" ] || return 1
+    [ ! -e "$value/.incomplete" ] && [ ! -L "$value/.incomplete" ] || return 1
+    entry_count=$(find "$value" -mindepth 1 -maxdepth 1 -print 2>/dev/null \
+        | wc -l | tr -d ' ') || return 1
+    [ "$entry_count" = "2" ] || return 1
+    [ "$(path_mode "$value")" = "700" ] || return 1
+    [ "$(path_mode "$value/database.dump")" = "600" ] || return 1
+    [ "$(path_mode "$value/integrity.manifest")" = "600" ]
+}
+
+path_mode() {
+    path=$1
+    stat -c '%a' "$path" 2>/dev/null \
+        || stat -f '%Lp' "$path" 2>/dev/null
+}
+
+require_pg_tool() {
+    tool=$1
+    command -v "$tool" >/dev/null 2>&1 || return 1
+    reported_version=$("$tool" --version 2>/dev/null) || return 1
+    case "$reported_version" in
+        "$tool (PostgreSQL) $POSTGRESQL_REQUIRED_VERSION"|"$tool (PostgreSQL) $POSTGRESQL_REQUIRED_VERSION "*) return 0 ;;
+        *) return 1 ;;
+    esac
+}
+
+run_psql() {
+    connection_host=$1
+    connection_port=$2
+    connection_database=$3
+    connection_username=$4
+    shift 4
+    PGAPPNAME=travel-readiness-backup-restore PGCONNECT_TIMEOUT=5 psql \
+        --no-password \
+        --host="$connection_host" \
+        --port="$connection_port" \
+        --dbname="$connection_database" \
+        --username="$connection_username" \
+        --no-psqlrc \
+        --set=ON_ERROR_STOP=1 \
+        "$@"
+}
+
+database_is_postgresql_18_6() {
+    connection_host=$1
+    connection_port=$2
+    connection_database=$3
+    connection_username=$4
+    database_version=$(run_psql \
+        "$connection_host" "$connection_port" "$connection_database" "$connection_username" \
+        --quiet --tuples-only --no-align \
+        --command='SHOW server_version_num' 2>/dev/null) || return 1
+    [ "$database_version" = "180006" ]
+}
+
+database_has_exact_tables() {
+    connection_host=$1
+    connection_port=$2
+    connection_database=$3
+    connection_username=$4
+    actual_tables=$(run_psql \
+        "$connection_host" "$connection_port" "$connection_database" "$connection_username" \
+        --quiet --tuples-only --no-align \
+        --command="SELECT table_name FROM information_schema.tables WHERE table_schema = 'public' AND table_type = 'BASE TABLE' ORDER BY table_name COLLATE \"C\"" \
+        2>/dev/null) || return 1
+    [ "$actual_tables" = "$EXPECTED_PUBLIC_TABLES" ]
+}
+
+database_has_exact_schema_digests() {
+    connection_host=$1
+    connection_port=$2
+    connection_database=$3
+    connection_username=$4
+    integrity_sql=$5
+    integrity_output=$(run_psql \
+        "$connection_host" "$connection_port" "$connection_database" "$connection_username" \
+        --quiet --tuples-only --no-align \
+        --file="$integrity_sql" 2>/dev/null) || return 1
+    actual_schema_digests=$(printf '%s\n' "$integrity_output" \
+        | sed -n '2,6p') || return 1
+    [ "$actual_schema_digests" = "$EXPECTED_SCHEMA_DIGESTS" ]
+}
+
+database_has_exact_schema_objects() {
+    connection_host=$1
+    connection_port=$2
+    connection_database=$3
+    connection_username=$4
+
+    unexpected_user_namespace_count=$(run_psql \
+        "$connection_host" "$connection_port" "$connection_database" "$connection_username" \
+        --quiet --tuples-only --no-align \
+        --command="SELECT count(*) FROM pg_catalog.pg_namespace WHERE nspname NOT IN ('pg_catalog', 'information_schema', 'public') AND nspname !~ '^pg_'" \
+        2>/dev/null) || return 1
+    [ "$unexpected_user_namespace_count" = "0" ] || return 1
+
+    unexpected_non_public_user_object_count=$(run_psql \
+        "$connection_host" "$connection_port" "$connection_database" "$connection_username" \
+        --quiet --tuples-only --no-align \
+        --command="SELECT (SELECT count(*) FROM pg_catalog.pg_class AS c JOIN pg_catalog.pg_namespace AS n ON n.oid = c.relnamespace WHERE n.nspname NOT IN ('pg_catalog', 'information_schema', 'public') AND n.nspname !~ '^pg_') + (SELECT count(*) FROM pg_catalog.pg_proc AS p JOIN pg_catalog.pg_namespace AS n ON n.oid = p.pronamespace WHERE n.nspname NOT IN ('pg_catalog', 'information_schema', 'public') AND n.nspname !~ '^pg_') + (SELECT count(*) FROM pg_catalog.pg_type AS t JOIN pg_catalog.pg_namespace AS n ON n.oid = t.typnamespace WHERE n.nspname NOT IN ('pg_catalog', 'information_schema', 'public') AND n.nspname !~ '^pg_') + (SELECT count(*) FROM pg_catalog.pg_extension AS e JOIN pg_catalog.pg_namespace AS n ON n.oid = e.extnamespace WHERE n.nspname NOT IN ('pg_catalog', 'information_schema', 'public') AND n.nspname !~ '^pg_')" \
+        2>/dev/null) || return 1
+    [ "$unexpected_non_public_user_object_count" = "0" ] || return 1
+
+    actual_sequences=$(run_psql \
+        "$connection_host" "$connection_port" "$connection_database" "$connection_username" \
+        --quiet --tuples-only --no-align \
+        --command="SELECT c.relname FROM pg_catalog.pg_class AS c JOIN pg_catalog.pg_namespace AS n ON n.oid = c.relnamespace WHERE n.nspname = 'public' AND c.relkind = 'S' ORDER BY c.relname COLLATE \"C\"" \
+        2>/dev/null) || return 1
+    [ "$actual_sequences" = "$EXPECTED_PUBLIC_SEQUENCES" ] || return 1
+
+    actual_sequence_definition_sha256=$(run_psql \
+        "$connection_host" "$connection_port" "$connection_database" "$connection_username" \
+        --quiet --tuples-only --no-align \
+        --command="SELECT pg_catalog.encode(pg_catalog.sha256(pg_catalog.convert_to(coalesce((SELECT string_agg(row_to_json(s)::text, E'\\n' ORDER BY s.sequencename COLLATE \"C\") FROM (SELECT schemaname, sequencename, data_type, start_value, min_value, max_value, increment_by, cycle, cache_size FROM pg_catalog.pg_sequences WHERE schemaname = 'public') AS s), ''), 'UTF8')), 'hex')" \
+        2>/dev/null) || return 1
+    [ "$actual_sequence_definition_sha256" = \
+        "$EXPECTED_SEQUENCE_DEFINITION_SHA256" ] || return 1
+
+    unexpected_relation_count=$(run_psql \
+        "$connection_host" "$connection_port" "$connection_database" "$connection_username" \
+        --quiet --tuples-only --no-align \
+        --command="SELECT count(*) FROM pg_catalog.pg_class AS c JOIN pg_catalog.pg_namespace AS n ON n.oid = c.relnamespace WHERE n.nspname = 'public' AND c.relkind NOT IN ('r', 'i', 'S')" \
+        2>/dev/null) || return 1
+    [ "$unexpected_relation_count" = "0" ] || return 1
+
+    unexpected_relation_policy_count=$(run_psql \
+        "$connection_host" "$connection_port" "$connection_database" "$connection_username" \
+        --quiet --tuples-only --no-align \
+        --command="SELECT count(*) FROM pg_catalog.pg_class AS c JOIN pg_catalog.pg_namespace AS n ON n.oid = c.relnamespace WHERE n.nspname = 'public' AND ((c.relkind IN ('r', 'i', 'S') AND c.relpersistence <> 'p') OR (c.relkind = 'r' AND (c.relrowsecurity OR c.relforcerowsecurity OR c.relreplident <> 'd')))" \
+        2>/dev/null) || return 1
+    [ "$unexpected_relation_policy_count" = "0" ] || return 1
+
+    actual_sequence_ownership=$(run_psql \
+        "$connection_host" "$connection_port" "$connection_database" "$connection_username" \
+        --quiet --tuples-only --no-align \
+        --command="SELECT s.relname || '|' || t.relname || '|' || a.attname FROM pg_catalog.pg_class AS s JOIN pg_catalog.pg_namespace AS sn ON sn.oid = s.relnamespace JOIN pg_catalog.pg_depend AS d ON d.classid = 'pg_catalog.pg_class'::pg_catalog.regclass AND d.objid = s.oid AND d.refclassid = 'pg_catalog.pg_class'::pg_catalog.regclass AND d.deptype IN ('a', 'i') JOIN pg_catalog.pg_class AS t ON t.oid = d.refobjid JOIN pg_catalog.pg_namespace AS tn ON tn.oid = t.relnamespace JOIN pg_catalog.pg_attribute AS a ON a.attrelid = t.oid AND a.attnum = d.refobjsubid WHERE sn.nspname = 'public' AND s.relkind = 'S' AND tn.nspname = 'public' AND t.relkind = 'r' AND d.refobjsubid > 0 AND NOT a.attisdropped ORDER BY s.relname COLLATE \"C\"" \
+        2>/dev/null) || return 1
+    [ "$actual_sequence_ownership" = "$EXPECTED_SEQUENCE_OWNERSHIP" ] \
+        || return 1
+
+    actual_functions=$(run_psql \
+        "$connection_host" "$connection_port" "$connection_database" "$connection_username" \
+        --quiet --tuples-only --no-align \
+        --command="SELECT p.proname || '|' || pg_catalog.encode(pg_catalog.sha256(pg_catalog.convert_to(p.prosrc, 'UTF8')), 'hex') FROM pg_catalog.pg_proc AS p JOIN pg_catalog.pg_namespace AS n ON n.oid = p.pronamespace WHERE n.nspname = 'public' ORDER BY p.proname COLLATE \"C\", pg_catalog.pg_get_function_identity_arguments(p.oid) COLLATE \"C\"" \
+        2>/dev/null) || return 1
+    [ "$actual_functions" = "$EXPECTED_TRIGGER_FUNCTIONS" ] || return 1
+
+    invalid_function_count=$(run_psql \
+        "$connection_host" "$connection_port" "$connection_database" "$connection_username" \
+        --quiet --tuples-only --no-align \
+        --command="SELECT count(*) FROM pg_catalog.pg_proc AS p JOIN pg_catalog.pg_namespace AS n ON n.oid = p.pronamespace JOIN pg_catalog.pg_language AS l ON l.oid = p.prolang WHERE n.nspname = 'public' AND (pg_catalog.pg_get_function_identity_arguments(p.oid) <> '' OR p.prorettype <> 'pg_catalog.trigger'::pg_catalog.regtype OR p.prokind <> 'f' OR l.lanname <> 'plpgsql' OR p.prosecdef OR p.provolatile <> 'v' OR p.proconfig IS NOT NULL)" \
+        2>/dev/null) || return 1
+    [ "$invalid_function_count" = "0" ] || return 1
+
+    actual_triggers=$(run_psql \
+        "$connection_host" "$connection_port" "$connection_database" "$connection_username" \
+        --quiet --tuples-only --no-align \
+        --command="SELECT t.tgname || '|' || c.relname || '|' || p.proname || '|' || t.tgtype::text || '|' || t.tgdeferrable::text || '|' || t.tginitdeferred::text || '|' || (t.tgconstraint <> 0)::text FROM pg_catalog.pg_trigger AS t JOIN pg_catalog.pg_class AS c ON c.oid = t.tgrelid JOIN pg_catalog.pg_namespace AS n ON n.oid = c.relnamespace JOIN pg_catalog.pg_proc AS p ON p.oid = t.tgfoid WHERE n.nspname = 'public' AND NOT t.tgisinternal ORDER BY t.tgname COLLATE \"C\"" \
+        2>/dev/null) || return 1
+    [ "$actual_triggers" = "$EXPECTED_PUBLIC_TRIGGERS" ] || return 1
+
+    disabled_trigger_count=$(run_psql \
+        "$connection_host" "$connection_port" "$connection_database" "$connection_username" \
+        --quiet --tuples-only --no-align \
+        --command="SELECT count(*) FROM pg_catalog.pg_trigger AS t JOIN pg_catalog.pg_class AS c ON c.oid = t.tgrelid JOIN pg_catalog.pg_namespace AS n ON n.oid = c.relnamespace WHERE n.nspname = 'public' AND NOT t.tgisinternal AND t.tgenabled <> 'O'" \
+        2>/dev/null) || return 1
+    [ "$disabled_trigger_count" = "0" ] || return 1
+
+    unexpected_policy_count=$(run_psql \
+        "$connection_host" "$connection_port" "$connection_database" "$connection_username" \
+        --quiet --tuples-only --no-align \
+        --command="SELECT (SELECT count(*) FROM pg_catalog.pg_policy AS p JOIN pg_catalog.pg_class AS c ON c.oid = p.polrelid JOIN pg_catalog.pg_namespace AS n ON n.oid = c.relnamespace WHERE n.nspname = 'public') + (SELECT count(*) FROM pg_catalog.pg_event_trigger) + (SELECT count(*) FROM pg_catalog.pg_extension AS e JOIN pg_catalog.pg_namespace AS n ON n.oid = e.extnamespace WHERE n.nspname = 'public') + (SELECT count(*) FROM pg_catalog.pg_type AS t JOIN pg_catalog.pg_namespace AS n ON n.oid = t.typnamespace LEFT JOIN pg_catalog.pg_class AS c ON c.oid = t.typrelid WHERE n.nspname = 'public' AND (t.typtype IN ('d', 'e', 'r', 'm') OR (t.typtype = 'c' AND c.relkind IS DISTINCT FROM 'r')))" \
+        2>/dev/null) || return 1
+    [ "$unexpected_policy_count" = "0" ] || return 1
+
+    unexpected_rewrite_count=$(run_psql \
+        "$connection_host" "$connection_port" "$connection_database" "$connection_username" \
+        --quiet --tuples-only --no-align \
+        --command="SELECT count(*) FROM pg_catalog.pg_rewrite AS r JOIN pg_catalog.pg_class AS c ON c.oid = r.ev_class JOIN pg_catalog.pg_namespace AS n ON n.oid = c.relnamespace WHERE n.nspname = 'public'" \
+        2>/dev/null) || return 1
+    [ "$unexpected_rewrite_count" = "0" ] || return 1
+
+    unexpected_object_metadata_count=$(run_psql \
+        "$connection_host" "$connection_port" "$connection_database" "$connection_username" \
+        --quiet --tuples-only --no-align \
+        --command="WITH public_namespace AS (SELECT oid FROM pg_catalog.pg_namespace WHERE nspname = 'public'), public_objects AS (SELECT 'pg_catalog.pg_namespace'::pg_catalog.regclass::oid AS classoid, oid AS objoid FROM public_namespace UNION SELECT d.classid, d.objid FROM pg_catalog.pg_depend AS d JOIN public_namespace AS n ON d.refclassid = 'pg_catalog.pg_namespace'::pg_catalog.regclass AND d.refobjid = n.oid UNION SELECT 'pg_catalog.pg_trigger'::pg_catalog.regclass::oid, t.oid FROM pg_catalog.pg_trigger AS t JOIN pg_catalog.pg_class AS c ON c.oid = t.tgrelid JOIN public_namespace AS n ON n.oid = c.relnamespace UNION SELECT 'pg_catalog.pg_rewrite'::pg_catalog.regclass::oid, r.oid FROM pg_catalog.pg_rewrite AS r JOIN pg_catalog.pg_class AS c ON c.oid = r.ev_class JOIN public_namespace AS n ON n.oid = c.relnamespace UNION SELECT 'pg_catalog.pg_constraint'::pg_catalog.regclass::oid, con.oid FROM pg_catalog.pg_constraint AS con JOIN public_namespace AS n ON n.oid = con.connamespace) SELECT (SELECT count(*) FROM pg_catalog.pg_description AS d JOIN public_objects AS o ON o.classoid = d.classoid AND o.objoid = d.objoid WHERE NOT (d.classoid = 'pg_catalog.pg_namespace'::pg_catalog.regclass AND d.objoid = (SELECT oid FROM public_namespace) AND d.objsubid = 0 AND pg_catalog.encode(pg_catalog.sha256(pg_catalog.convert_to(d.description, 'UTF8')), 'hex') = '$EXPECTED_PUBLIC_SCHEMA_COMMENT_SHA256')) + (SELECT count(*) FROM pg_catalog.pg_seclabel AS s JOIN public_objects AS o ON o.classoid = s.classoid AND o.objoid = s.objoid)" \
+        2>/dev/null) || return 1
+    [ "$unexpected_object_metadata_count" = "0" ]
+}
+
+database_has_no_forbidden_storage() {
+    connection_host=$1
+    connection_port=$2
+    connection_database=$3
+    connection_username=$4
+    forbidden_count=$(run_psql \
+        "$connection_host" "$connection_port" "$connection_database" "$connection_username" \
+        --quiet --tuples-only --no-align \
+        --command="SELECT count(*) FROM information_schema.columns WHERE table_schema = 'public' AND lower(column_name) ~ '(raw|secret_value|api_?key|service_?key|credential|destination|departure|return_date|travel_date|trip_date|travel_purpose)' AND NOT (table_name = 'sources_sourceconfiguration' AND column_name = 'secret_reference_name') AND NOT (table_name = 'sources_sourcerightsdecision' AND column_name IN ('raw_body_storage_allowed', 'raw_retention_seconds'))" \
+        2>/dev/null) || return 1
+    [ "$forbidden_count" = "0" ] || return 1
+    large_object_count=$(run_psql \
+        "$connection_host" "$connection_port" "$connection_database" "$connection_username" \
+        --quiet --tuples-only --no-align \
+        --command='SELECT count(*) FROM pg_catalog.pg_largeobject_metadata' \
+        2>/dev/null) || return 1
+    [ "$large_object_count" = "0" ]
+}
+
+database_is_empty_restore_target() {
+    connection_host=$1
+    connection_port=$2
+    connection_database=$3
+    connection_username=$4
+    object_count=$(run_psql \
+        "$connection_host" "$connection_port" "$connection_database" "$connection_username" \
+        --quiet --tuples-only --no-align \
+        --command="SELECT count(*) FROM pg_catalog.pg_class AS c JOIN pg_catalog.pg_namespace AS n ON n.oid = c.relnamespace WHERE n.nspname NOT IN ('pg_catalog', 'information_schema') AND n.nspname !~ '^pg_toast' AND c.relkind IN ('r', 'p', 'v', 'm', 'S', 'f')" \
+        2>/dev/null) || return 1
+    [ "$object_count" = "0" ] || return 1
+    public_namespace_count=$(run_psql \
+        "$connection_host" "$connection_port" "$connection_database" "$connection_username" \
+        --quiet --tuples-only --no-align \
+        --command="SELECT count(*) FROM pg_catalog.pg_namespace WHERE nspname = 'public'" \
+        2>/dev/null) || return 1
+    [ "$public_namespace_count" = "0" ] || return 1
+    user_namespace_count=$(run_psql \
+        "$connection_host" "$connection_port" "$connection_database" "$connection_username" \
+        --quiet --tuples-only --no-align \
+        --command="SELECT count(*) FROM pg_catalog.pg_namespace WHERE nspname NOT IN ('pg_catalog', 'information_schema', 'public') AND nspname !~ '^pg_'" \
+        2>/dev/null) || return 1
+    [ "$user_namespace_count" = "0" ] || return 1
+    applicable_setting_count=$(run_psql \
+        "$connection_host" "$connection_port" "$connection_database" "$connection_username" \
+        --quiet --tuples-only --no-align \
+        --command="SELECT count(*) FROM pg_catalog.pg_db_role_setting WHERE setdatabase IN (0, (SELECT oid FROM pg_catalog.pg_database WHERE datname = current_database())) AND setrole IN (0, (SELECT oid FROM pg_catalog.pg_roles WHERE rolname = current_user))" \
+        2>/dev/null) || return 1
+    [ "$applicable_setting_count" = "0" ] || return 1
+    namespace_object_count=$(run_psql \
+        "$connection_host" "$connection_port" "$connection_database" "$connection_username" \
+        --quiet --tuples-only --no-align \
+        --command="SELECT (SELECT count(*) FROM pg_catalog.pg_proc AS p JOIN pg_catalog.pg_namespace AS n ON n.oid = p.pronamespace WHERE n.nspname = 'public') + (SELECT count(*) FROM pg_catalog.pg_type AS t JOIN pg_catalog.pg_namespace AS n ON n.oid = t.typnamespace WHERE n.nspname = 'public') + (SELECT count(*) FROM pg_catalog.pg_extension AS e JOIN pg_catalog.pg_namespace AS n ON n.oid = e.extnamespace WHERE n.nspname = 'public') + (SELECT count(*) FROM pg_catalog.pg_event_trigger)" \
+        2>/dev/null) || return 1
+    [ "$namespace_object_count" = "0" ]
+}
+
+write_integrity_snapshot() {
+    connection_host=$1
+    connection_port=$2
+    connection_database=$3
+    connection_username=$4
+    integrity_sql=$5
+    output_file=$6
+    run_psql \
+        "$connection_host" "$connection_port" "$connection_database" "$connection_username" \
+        --quiet --tuples-only --no-align \
+        --file="$integrity_sql" 2>/dev/null >"$output_file"
+}
+
+sha256_file() {
+    file=$1
+    shasum -a 256 "$file" 2>/dev/null | awk '{print $1}'
+}
+
+is_sha256() {
+    value=$1
+    [ "${#value}" -eq 64 ] || return 1
+    case "$value" in
+        *[!0-9a-f]* ) return 1 ;;
+    esac
+}
diff --git a/scripts/postgresql-integrity.sql b/scripts/postgresql-integrity.sql
new file mode 100644
index 0000000..5f4ee52
--- /dev/null
+++ b/scripts/postgresql-integrity.sql
@@ -0,0 +1,243 @@
+SET TIME ZONE 'UTC';
+SET DateStyle = 'ISO, YMD';
+SET search_path = pg_catalog, public;
+
+WITH integrity_lines(sort_key, line) AS (
+    SELECT '001', 'postgresql.version_num=' || current_setting('server_version_num')
+
+    UNION ALL SELECT '002', 'schema.columns.sha256=' || encode(sha256(convert_to(
+      coalesce((
+        SELECT string_agg(row_to_json(s)::text, E'\n' ORDER BY s.table_name COLLATE "C", s.ordinal_position)
+        FROM (
+          SELECT c.relname AS table_name,
+                 row_number() OVER (
+                   PARTITION BY c.oid ORDER BY a.attnum
+                 ) AS ordinal_position,
+                 a.attname AS column_name, pg_catalog.format_type(a.atttypid, a.atttypmod) AS data_type,
+                 a.attnotnull AS not_null, a.attidentity AS identity_kind,
+                 a.attgenerated AS generated_kind, coalesce(coll.collname, '') AS collation_name,
+                 coalesce(pg_catalog.pg_get_expr(d.adbin, d.adrelid), '') AS default_expression
+          FROM pg_catalog.pg_attribute AS a
+          JOIN pg_catalog.pg_class AS c ON c.oid = a.attrelid
+          JOIN pg_catalog.pg_namespace AS n ON n.oid = c.relnamespace
+          LEFT JOIN pg_catalog.pg_attrdef AS d ON d.adrelid = a.attrelid AND d.adnum = a.attnum
+          LEFT JOIN pg_catalog.pg_collation AS coll ON coll.oid = a.attcollation
+          WHERE n.nspname = 'public' AND c.relkind = 'r'
+            AND a.attnum > 0 AND NOT a.attisdropped
+        ) AS s
+      ), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '003', 'schema.constraints.sha256=' || encode(sha256(convert_to(
+      coalesce((
+        SELECT string_agg(row_to_json(s)::text, E'\n' ORDER BY s.table_name COLLATE "C", s.constraint_name COLLATE "C")
+        FROM (
+          SELECT c.relname AS table_name, con.conname AS constraint_name,
+                 con.contype AS constraint_type, con.condeferrable AS deferrable,
+                 con.condeferred AS initially_deferred, con.convalidated AS validated,
+                 replace(
+                   replace(
+                     pg_catalog.pg_get_constraintdef(con.oid, true),
+                     '::character varying::text', '::character varying'
+                   ),
+                   ']::text[]', ']'
+                 ) AS definition
+          FROM pg_catalog.pg_constraint AS con
+          JOIN pg_catalog.pg_class AS c ON c.oid = con.conrelid
+          JOIN pg_catalog.pg_namespace AS n ON n.oid = c.relnamespace
+          WHERE n.nspname = 'public'
+        ) AS s
+      ), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '004', 'schema.indexes.sha256=' || encode(sha256(convert_to(
+      coalesce((
+        SELECT string_agg(row_to_json(s)::text, E'\n' ORDER BY s.table_name COLLATE "C", s.index_name COLLATE "C")
+        FROM (
+          SELECT c.relname AS table_name, i.relname AS index_name,
+                 x.indisunique AS is_unique, x.indisprimary AS is_primary,
+                 x.indisvalid AS is_valid, pg_catalog.pg_get_indexdef(i.oid) AS definition
+          FROM pg_catalog.pg_index AS x
+          JOIN pg_catalog.pg_class AS c ON c.oid = x.indrelid
+          JOIN pg_catalog.pg_class AS i ON i.oid = x.indexrelid
+          JOIN pg_catalog.pg_namespace AS n ON n.oid = c.relnamespace
+          WHERE n.nspname = 'public'
+        ) AS s
+      ), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '005', 'schema.trigger_functions.sha256=' || encode(sha256(convert_to(
+      coalesce((
+        SELECT string_agg(row_to_json(s)::text, E'\n' ORDER BY s.function_name COLLATE "C")
+        FROM (
+          SELECT p.proname AS function_name,
+                 pg_catalog.pg_get_function_identity_arguments(p.oid) AS identity_arguments,
+                 pg_catalog.pg_get_function_result(p.oid) AS result_type,
+                 l.lanname AS language_name, p.provolatile AS volatility,
+                 p.prosecdef AS security_definer, p.proconfig AS configuration,
+                 pg_catalog.pg_get_functiondef(p.oid) AS definition
+          FROM pg_catalog.pg_proc AS p
+          JOIN pg_catalog.pg_namespace AS n ON n.oid = p.pronamespace
+          JOIN pg_catalog.pg_language AS l ON l.oid = p.prolang
+          WHERE n.nspname = 'public'
+        ) AS s
+      ), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '006', 'schema.triggers.sha256=' || encode(sha256(convert_to(
+      coalesce((
+        SELECT string_agg(row_to_json(s)::text, E'\n' ORDER BY s.trigger_name COLLATE "C")
+        FROM (
+          SELECT t.tgname AS trigger_name, c.relname AS table_name,
+                 t.tgenabled AS enabled_mode, t.tgtype AS trigger_type,
+                 t.tgdeferrable AS deferrable, t.tginitdeferred AS initially_deferred,
+                 pg_catalog.pg_get_triggerdef(t.oid, true) AS definition
+          FROM pg_catalog.pg_trigger AS t
+          JOIN pg_catalog.pg_class AS c ON c.oid = t.tgrelid
+          JOIN pg_catalog.pg_namespace AS n ON n.oid = c.relnamespace
+          WHERE n.nspname = 'public' AND NOT t.tgisinternal
+        ) AS s
+      ), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '006a', 'database.locale_profile.sha256=' || encode(sha256(convert_to(
+      coalesce((
+        SELECT row_to_json(s)::text
+        FROM (
+          SELECT pg_catalog.pg_encoding_to_char(db.encoding) AS encoding_name,
+                 db.datlocprovider AS locale_provider,
+                 db.datcollate AS collate_name,
+                 db.datctype AS ctype_name,
+                 db.datlocale AS locale_name,
+                 coalesce(db.datcollversion, '') AS collation_version
+          FROM pg_catalog.pg_database AS db
+          WHERE db.datname = current_database()
+        ) AS s
+      ), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '007', 'schema.sequences.sha256=' || encode(sha256(convert_to(
+      coalesce((
+        SELECT string_agg(row_to_json(s)::text, E'\n' ORDER BY s.sequencename COLLATE "C")
+        FROM (
+          SELECT schemaname, sequencename, data_type, start_value, min_value,
+                 max_value, increment_by, cycle, cache_size, last_value
+          FROM pg_catalog.pg_sequences WHERE schemaname = 'public'
+        ) AS s
+      ), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '007a', 'schema.sequence_state.sha256=' || encode(sha256(convert_to(
+      coalesce((
+        SELECT string_agg(
+          s.sequence_name || '|' || s.last_value || '|' || s.is_called::text,
+          E'\n' ORDER BY s.sequence_name COLLATE "C"
+        )
+        FROM (
+          SELECT 'auth_group_id_seq' AS sequence_name,
+                 last_value::text, is_called FROM public.auth_group_id_seq
+          UNION ALL SELECT 'auth_group_permissions_id_seq',
+                 last_value::text, is_called FROM public.auth_group_permissions_id_seq
+          UNION ALL SELECT 'auth_permission_id_seq',
+                 last_value::text, is_called FROM public.auth_permission_id_seq
+          UNION ALL SELECT 'auth_user_groups_id_seq',
+                 last_value::text, is_called FROM public.auth_user_groups_id_seq
+          UNION ALL SELECT 'auth_user_id_seq',
+                 last_value::text, is_called FROM public.auth_user_id_seq
+          UNION ALL SELECT 'auth_user_user_permissions_id_seq',
+                 last_value::text, is_called FROM public.auth_user_user_permissions_id_seq
+          UNION ALL SELECT 'django_admin_log_id_seq',
+                 last_value::text, is_called FROM public.django_admin_log_id_seq
+          UNION ALL SELECT 'django_content_type_id_seq',
+                 last_value::text, is_called FROM public.django_content_type_id_seq
+          UNION ALL SELECT 'django_migrations_id_seq',
+                 last_value::text, is_called FROM public.django_migrations_id_seq
+        ) AS s
+      ), ''), 'UTF8')), 'hex')
+
+    UNION ALL SELECT '008', 'table.auth_group.count=' || count(*)::text
+      FROM public.auth_group
+    UNION ALL SELECT '009', 'table.auth_group.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.auth_group AS t), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '00a', 'table.auth_group_permissions.count=' || count(*)::text
+      FROM public.auth_group_permissions
+    UNION ALL SELECT '00b', 'table.auth_group_permissions.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.auth_group_permissions AS t), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '00c', 'table.auth_permission.count=' || count(*)::text
+      FROM public.auth_permission
+    UNION ALL SELECT '00d', 'table.auth_permission.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.auth_permission AS t), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '00e', 'table.auth_user.count=' || count(*)::text
+      FROM public.auth_user
+    UNION ALL SELECT '00f', 'table.auth_user.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.auth_user AS t), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '00g', 'table.auth_user_groups.count=' || count(*)::text
+      FROM public.auth_user_groups
+    UNION ALL SELECT '00h', 'table.auth_user_groups.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.auth_user_groups AS t), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '00i', 'table.auth_user_user_permissions.count=' || count(*)::text
+      FROM public.auth_user_user_permissions
+    UNION ALL SELECT '00j', 'table.auth_user_user_permissions.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.auth_user_user_permissions AS t), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '00m', 'table.django_content_type.count=' || count(*)::text
+      FROM public.django_content_type
+    UNION ALL SELECT '00n', 'table.django_content_type.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.django_content_type AS t), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '00o', 'table.django_migrations.count=' || count(*)::text
+      FROM public.django_migrations
+    UNION ALL SELECT '00p', 'table.django_migrations.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.django_migrations AS t), ''), 'UTF8')), 'hex')
+
+    UNION ALL SELECT '010', 'table.countries_country.count=' || count(*)::text
+      FROM public.countries_country
+    UNION ALL SELECT '011', 'table.countries_country.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.countries_country AS t), ''), 'UTF8')), 'hex')
+
+    UNION ALL SELECT '020', 'table.sources_sourceconfiguration.count=' || count(*)::text
+      FROM public.sources_sourceconfiguration
+    UNION ALL SELECT '021', 'table.sources_sourceconfiguration.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.sources_sourceconfiguration AS t), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '022', 'table.sources_sourcerightsdecision.count=' || count(*)::text
+      FROM public.sources_sourcerightsdecision
+    UNION ALL SELECT '023', 'table.sources_sourcerightsdecision.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.sources_sourcerightsdecision AS t), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '024', 'table.sources_fetchattempt.count=' || count(*)::text
+      FROM public.sources_fetchattempt
+    UNION ALL SELECT '025', 'table.sources_fetchattempt.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.sources_fetchattempt AS t), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '026', 'table.sources_sourceartifact.count=' || count(*)::text
+      FROM public.sources_sourceartifact
+    UNION ALL SELECT '027', 'table.sources_sourceartifact.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.sources_sourceartifact AS t), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '028', 'table.sources_parserun.count=' || count(*)::text
+      FROM public.sources_parserun
+    UNION ALL SELECT '029', 'table.sources_parserun.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.sources_parserun AS t), ''), 'UTF8')), 'hex')
+
+    UNION ALL SELECT '030', 'table.entry_requirements_passportpolicy.count=' || count(*)::text
+      FROM public.entry_requirements_passportpolicy
+    UNION ALL SELECT '031', 'table.entry_requirements_passportpolicy.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.entry_requirements_passportpolicy AS t), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '032', 'table.entry_requirements_entryfactrevision.count=' || count(*)::text
+      FROM public.entry_requirements_entryfactrevision
+    UNION ALL SELECT '033', 'table.entry_requirements_entryfactrevision.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.entry_requirements_entryfactrevision AS t), ''), 'UTF8')), 'hex')
+
+    UNION ALL SELECT '040', 'table.travel_warnings_travelwarningrevision.count=' || count(*)::text
+      FROM public.travel_warnings_travelwarningrevision
+    UNION ALL SELECT '041', 'table.travel_warnings_travelwarningrevision.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.travel_warnings_travelwarningrevision AS t), ''), 'UTF8')), 'hex')
+
+    UNION ALL SELECT '050', 'table.reviews_reviewdecision.count=' || count(*)::text
+      FROM public.reviews_reviewdecision
+    UNION ALL SELECT '051', 'table.reviews_reviewdecision.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.reviews_reviewdecision AS t), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '052', 'table.reviews_publicationrevision.count=' || count(*)::text
+      FROM public.reviews_publicationrevision
+    UNION ALL SELECT '053', 'table.reviews_publicationrevision.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.reviews_publicationrevision AS t), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '054', 'table.reviews_publishedentryfacts.count=' || count(*)::text
+      FROM public.reviews_publishedentryfacts
+    UNION ALL SELECT '055', 'table.reviews_publishedentryfacts.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.reviews_publishedentryfacts AS t), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '056', 'table.reviews_publishedtravelwarning.count=' || count(*)::text
+      FROM public.reviews_publishedtravelwarning
+    UNION ALL SELECT '057', 'table.reviews_publishedtravelwarning.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.reviews_publishedtravelwarning AS t), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '058', 'table.reviews_auditevent.count=' || count(*)::text
+      FROM public.reviews_auditevent
+    UNION ALL SELECT '059', 'table.reviews_auditevent.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.reviews_auditevent AS t), ''), 'UTF8')), 'hex')
+
+    UNION ALL SELECT '060', 'pointer.entry=' || coalesce(current_publication_id::text, 'NONE') || ':' || version::text
+      FROM public.reviews_publishedentryfacts
+    UNION ALL SELECT '061', 'pointer.travel_warning=' || coalesce(current_publication_id::text, 'NONE') || ':' || version::text
+      FROM public.reviews_publishedtravelwarning
+)
+SELECT line FROM integrity_lines ORDER BY sort_key;
diff --git a/scripts/restore-postgresql b/scripts/restore-postgresql
new file mode 100755
index 0000000..f9db89b
--- /dev/null
+++ b/scripts/restore-postgresql
@@ -0,0 +1,187 @@
+#!/bin/sh
+
+set +x
+set -eu
+umask 077
+LC_ALL=C
+export LC_ALL
+unset PGDATABASE PGUSER PGHOST PGPORT PGSERVICE PGSERVICEFILE PGOPTIONS
+unset MOFA_TRAVEL_ALARM_SERVICE_KEY TRAVEL_READINESS_SECRET_KEY TRAVEL_READINESS_DB_PASSWORD
+
+SCRIPT_DIR=$(CDPATH='' cd "$(dirname "$0")" && pwd -P)
+# shellcheck source-path=SCRIPTDIR
+# shellcheck source=postgresql-common
+. "$SCRIPT_DIR/postgresql-common"
+
+usage() {
+    printf '%s\n' 'usage: restore-postgresql --host HOST --port PORT --database travel_readiness_restore_NAME --username USERNAME --backup-dir ABSOLUTE_DIRECTORY --safety-token RESTORE_DISPOSABLE:travel_readiness_restore_NAME'
+}
+
+fail() {
+    printf '%s\n' "$1" >&2
+    exit "$2"
+}
+
+host=''
+port=''
+database=''
+username=''
+backup_dir=''
+safety_token=''
+
+while [ "$#" -gt 0 ]; do
+    case "$1" in
+        --help)
+            [ "$#" -eq 1 ] || fail 'restore_result=INVALID_ARGUMENTS' 64
+            usage
+            exit 0
+            ;;
+        --host|--port|--database|--username|--backup-dir|--safety-token)
+            [ "$#" -ge 2 ] || fail 'restore_result=INVALID_ARGUMENTS' 64
+            option=$1
+            value=$2
+            shift 2
+            case "$option" in
+                --host) [ -z "$host" ] || fail 'restore_result=INVALID_ARGUMENTS' 64; host=$value ;;
+                --port) [ -z "$port" ] || fail 'restore_result=INVALID_ARGUMENTS' 64; port=$value ;;
+                --database) [ -z "$database" ] || fail 'restore_result=INVALID_ARGUMENTS' 64; database=$value ;;
+                --username) [ -z "$username" ] || fail 'restore_result=INVALID_ARGUMENTS' 64; username=$value ;;
+                --backup-dir) [ -z "$backup_dir" ] || fail 'restore_result=INVALID_ARGUMENTS' 64; backup_dir=$value ;;
+                --safety-token) [ -z "$safety_token" ] || fail 'restore_result=INVALID_ARGUMENTS' 64; safety_token=$value ;;
+            esac
+            ;;
+        *) fail 'restore_result=INVALID_ARGUMENTS' 64 ;;
+    esac
+done
+
+is_host "$host" || fail 'restore_result=INVALID_ARGUMENTS' 64
+is_port "$port" || fail 'restore_result=INVALID_ARGUMENTS' 64
+is_identifier "$database" || fail 'restore_result=INVALID_ARGUMENTS' 64
+is_identifier "$username" || fail 'restore_result=INVALID_ARGUMENTS' 64
+case "$database" in
+    travel_readiness_restore_[a-z0-9_]*) ;;
+    *) fail 'restore_result=UNSAFE_TARGET' 65 ;;
+esac
+[ "$safety_token" = "RESTORE_DISPOSABLE:$database" ] \
+    || fail 'restore_result=SAFETY_TOKEN_MISMATCH' 65
+is_backup_directory "$backup_dir" || fail 'restore_result=INVALID_BACKUP' 66
+
+require_pg_tool psql || fail 'restore_result=POSTGRESQL_18_6_REQUIRED' 69
+require_pg_tool pg_restore || fail 'restore_result=POSTGRESQL_18_6_REQUIRED' 69
+command -v shasum >/dev/null 2>&1 || fail 'restore_result=REQUIRED_TOOL_MISSING' 69
+command -v awk >/dev/null 2>&1 || fail 'restore_result=REQUIRED_TOOL_MISSING' 69
+command -v cmp >/dev/null 2>&1 || fail 'restore_result=REQUIRED_TOOL_MISSING' 69
+command -v find >/dev/null 2>&1 || fail 'restore_result=REQUIRED_TOOL_MISSING' 69
+command -v stat >/dev/null 2>&1 || fail 'restore_result=REQUIRED_TOOL_MISSING' 69
+command -v tr >/dev/null 2>&1 || fail 'restore_result=REQUIRED_TOOL_MISSING' 69
+command -v wc >/dev/null 2>&1 || fail 'restore_result=REQUIRED_TOOL_MISSING' 69
+
+manifest_format=$(sed -n '1p' "$backup_dir/integrity.manifest" 2>/dev/null) \
+    || fail 'restore_result=INVALID_MANIFEST' 66
+[ "$manifest_format" = 'manifest.format=travel-readiness-postgresql-backup-v1' ] \
+    || fail 'restore_result=INVALID_MANIFEST' 66
+manifest_archive_line=$(sed -n '2p' "$backup_dir/integrity.manifest" 2>/dev/null) \
+    || fail 'restore_result=INVALID_MANIFEST' 66
+case "$manifest_archive_line" in
+    archive.sha256=*) expected_archive_sha256=${manifest_archive_line#archive.sha256=} ;;
+    *) fail 'restore_result=INVALID_MANIFEST' 66 ;;
+esac
+is_sha256 "$expected_archive_sha256" || fail 'restore_result=INVALID_MANIFEST' 66
+[ "$(sed -n '3p' "$backup_dir/integrity.manifest" 2>/dev/null)" = \
+    'archive.scope=approved-public-schema-without-ephemeral-data' ] \
+    || fail 'restore_result=INVALID_MANIFEST' 66
+[ "$(grep -c '^archive\.sha256=' "$backup_dir/integrity.manifest" 2>/dev/null)" = "1" ] \
+    || fail 'restore_result=INVALID_MANIFEST' 66
+
+actual_archive_sha256=$(sha256_file "$backup_dir/database.dump") \
+    || fail 'restore_result=HASH_FAILED' 66
+[ "$actual_archive_sha256" = "$expected_archive_sha256" ] \
+    || fail 'restore_result=ARCHIVE_HASH_MISMATCH' 66
+
+database_is_postgresql_18_6 "$host" "$port" "$database" "$username" \
+    || fail 'restore_result=DATABASE_VERSION_MISMATCH' 65
+database_is_empty_restore_target "$host" "$port" "$database" "$username" \
+    || fail 'restore_result=TARGET_NOT_EMPTY' 65
+
+restore_tmp=$(mktemp -d /tmp/travel-readiness-restore.XXXXXX 2>/dev/null) \
+    || fail 'restore_result=TEMP_CREATE_FAILED' 73
+cleanup() {
+    rm -f "$restore_tmp/expected.snapshot" "$restore_tmp/actual.snapshot" 2>/dev/null || :
+    rmdir "$restore_tmp" 2>/dev/null || :
+}
+
+cleanup_on_exit() {
+    original_status=$?
+    trap - EXIT HUP INT TERM
+    cleanup
+    exit "$original_status"
+}
+
+interrupt() {
+    signal_status=$1
+    trap - EXIT HUP INT TERM
+    cleanup
+    printf '%s\n' 'restore_result=INTERRUPTED' >&2
+    exit "$signal_status"
+}
+
+trap cleanup_on_exit EXIT
+trap 'interrupt 129' HUP
+trap 'interrupt 130' INT
+trap 'interrupt 143' TERM
+
+tail -n +4 "$backup_dir/integrity.manifest" 2>/dev/null >"$restore_tmp/expected.snapshot" \
+    || fail 'restore_result=INVALID_MANIFEST' 66
+chmod 600 "$restore_tmp/expected.snapshot" 2>/dev/null \
+    || fail 'restore_result=INVALID_MANIFEST' 66
+[ -s "$restore_tmp/expected.snapshot" ] || fail 'restore_result=INVALID_MANIFEST' 66
+
+PGAPPNAME=travel-readiness-restore PGCONNECT_TIMEOUT=5 pg_restore \
+    --no-password \
+    --host="$host" \
+    --port="$port" \
+    --dbname="$database" \
+    --username="$username" \
+    --exit-on-error \
+    --single-transaction \
+    --no-owner \
+    --no-acl \
+    "$backup_dir/database.dump" \
+    >/dev/null 2>&1 \
+    || fail 'restore_result=RESTORE_FAILED' 74
+
+database_has_exact_tables "$host" "$port" "$database" "$username" \
+    || fail 'restore_result=SCHEMA_MISMATCH' 76
+database_has_exact_schema_objects "$host" "$port" "$database" "$username" \
+    || fail 'restore_result=SCHEMA_OBJECTS_MISMATCH' 76
+database_has_exact_schema_digests \
+    "$host" "$port" "$database" "$username" \
+    "$SCRIPT_DIR/postgresql-integrity.sql" \
+    || fail 'restore_result=SCHEMA_DIGEST_MISMATCH' 76
+database_has_no_forbidden_storage "$host" "$port" "$database" "$username" \
+    || fail 'restore_result=FORBIDDEN_STORAGE_PRESENT' 76
+session_count=$(run_psql \
+    "$host" "$port" "$database" "$username" \
+    --quiet --tuples-only --no-align \
+    --command='SELECT count(*) FROM public.django_session' 2>/dev/null) \
+    || fail 'restore_result=SESSION_CHECK_FAILED' 76
+[ "$session_count" = "0" ] || fail 'restore_result=SESSION_DATA_PRESENT' 76
+admin_log_count=$(run_psql \
+    "$host" "$port" "$database" "$username" \
+    --quiet --tuples-only --no-align \
+    --command='SELECT count(*) FROM public.django_admin_log' 2>/dev/null) \
+    || fail 'restore_result=EPHEMERAL_DATA_CHECK_FAILED' 76
+[ "$admin_log_count" = "0" ] || fail 'restore_result=EPHEMERAL_DATA_PRESENT' 76
+
+write_integrity_snapshot \
+    "$host" "$port" "$database" "$username" \
+    "$SCRIPT_DIR/postgresql-integrity.sql" "$restore_tmp/actual.snapshot" \
+    || fail 'restore_result=VERIFY_FAILED' 76
+chmod 600 "$restore_tmp/actual.snapshot" 2>/dev/null \
+    || fail 'restore_result=VERIFY_FAILED' 76
+cmp -s "$restore_tmp/expected.snapshot" "$restore_tmp/actual.snapshot" \
+    || fail 'restore_result=INTEGRITY_MISMATCH' 76
+
+cleanup
+trap - EXIT HUP INT TERM
+printf '%s\n' 'restore_result=ok'


