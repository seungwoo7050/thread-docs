## `test(operations): exercise restored production runtime`

diff --git a/operations/tests/test_postgresql_backup_restore.py b/operations/tests/test_postgresql_backup_restore.py
index 9635c93..3eaf66f 100644
--- a/operations/tests/test_postgresql_backup_restore.py
+++ b/operations/tests/test_postgresql_backup_restore.py
@@ -1,12 +1,15 @@
 from __future__ import annotations
 
+import ast
 import hashlib
 import importlib
 import os
 from pathlib import Path
 import re
+import signal
 import stat
 import subprocess
+import sys
 import tempfile
 import unittest
 
@@ -22,6 +25,13 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
         cls.common = cls.root / "scripts" / "postgresql-common"
         cls.integrity_sql = cls.root / "scripts" / "postgresql-integrity.sql"
         cls.runbook = cls.root / "docs" / "OPERATIONS-RUNBOOK.md"
+        cls.head_sha = subprocess.run(
+            ["git", "rev-parse", "--verify", "HEAD"],
+            cwd=cls.root,
+            capture_output=True,
+            text=True,
+            check=True,
+        ).stdout.strip()
 
     def run_script(self, script: Path, *arguments: str, env=None):
         return subprocess.run(
@@ -39,6 +49,7 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
             "database_prefix", "travel_readiness_restorecheck_unit"
         )
         values = {
+            "release_sha": self.head_sha,
             "host": "127.0.0.1",
             "port": "5432",
             "admin_role": "postgres",
@@ -56,6 +67,8 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
         }
         values.update(overrides)
         return [
+            "--release-sha",
+            values["release_sha"],
             "--host",
             values["host"],
             "--port",
@@ -78,6 +91,19 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
             values["writers_quiesced_confirmation"],
         ]
 
+    def restored_wsgi_probe_namespace(self):
+        script = self.rehearsal.read_text(encoding="utf-8")
+        source = script.split("# RESTORED_WSGI_PROBE_START\n", 1)[1].split(
+            "# RESTORED_WSGI_PROBE_END\n", 1
+        )[0]
+        tree = ast.parse(source, filename=str(self.rehearsal))
+        self.assertIsInstance(tree.body[-1], ast.Raise)
+        tree.body = tree.body[:-1]
+        ast.fix_missing_locations(tree)
+        namespace = {"__name__": "restore_probe_contract_test"}
+        exec(compile(tree, str(self.rehearsal), "exec"), namespace)
+        return namespace
+
     def make_fake_postgresql_18_6(self, parent: Path):
         common = self.common.read_text(encoding="utf-8")
         schema_digests = (
@@ -218,6 +244,7 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
         help_result = self.run_script(self.rehearsal, "--help")
         self.assertEqual(help_result.returncode, 0)
         self.assertEqual(help_result.stderr, "")
+        self.assertIn("--release-sha SHA", help_result.stdout)
         self.assertIn("--writers-quiesced-confirmation WRITERS_QUIESCED", help_result.stdout)
 
         remote = self.run_script(
@@ -283,6 +310,69 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
             "backup_restore_check=ADMIN_PASSWORD_MISSING\n",
         )
 
+    def test_rehearsal_requires_exact_release_identity_before_database_access(self):
+        invalid = self.run_script(
+            self.rehearsal,
+            *self.rehearsal_arguments(release_sha="A" * 40),
+        )
+        self.assertEqual(invalid.returncode, 65)
+        self.assertEqual(invalid.stdout, "")
+        self.assertEqual(
+            invalid.stderr,
+            "backup_restore_check=RELEASE_IDENTITY_INVALID\n",
+        )
+
+        different_first = "0" if self.head_sha[0] != "0" else "1"
+        environment = os.environ.copy()
+        environment[
+            "TRAVEL_READINESS_BACKUP_RESTORE_ADMIN_PASSWORD"
+        ] = "fixed-test-admin-input"
+        environment[
+            "TRAVEL_READINESS_BACKUP_ROLE_PASSWORD"
+        ] = "fixed-test-backup-input"
+        mismatch = self.run_script(
+            self.rehearsal,
+            *self.rehearsal_arguments(
+                release_sha=different_first + self.head_sha[1:]
+            ),
+            env=environment,
+        )
+        self.assertEqual(mismatch.returncode, 65)
+        self.assertEqual(mismatch.stdout, "")
+        self.assertEqual(
+            mismatch.stderr,
+            "backup_restore_check=RELEASE_IDENTITY_MISMATCH\n",
+        )
+
+        with tempfile.TemporaryDirectory() as temporary:
+            fake_git = Path(temporary) / "git"
+            fake_git.write_text(
+                "#!/bin/sh\n"
+                "case \"$*\" in\n"
+                "  *'rev-parse --verify HEAD'*) printf '%s\\n' \"$FAKE_RELEASE_SHA\"; exit 0 ;;\n"
+                "  *'status --porcelain=v1 --untracked-files=normal'*) printf '%s\\n' ' M tracked-file'; exit 0 ;;\n"
+                "esac\n"
+                "exit 98\n",
+                encoding="utf-8",
+            )
+            fake_git.chmod(fake_git.stat().st_mode | stat.S_IXUSR)
+            dirty_environment = environment.copy()
+            dirty_environment["PATH"] = (
+                f"{temporary}:{dirty_environment.get('PATH', '')}"
+            )
+            dirty_environment["FAKE_RELEASE_SHA"] = self.head_sha
+            dirty = self.run_script(
+                self.rehearsal,
+                *self.rehearsal_arguments(),
+                env=dirty_environment,
+            )
+        self.assertEqual(dirty.returncode, 65)
+        self.assertEqual(dirty.stdout, "")
+        self.assertEqual(
+            dirty.stderr,
+            "backup_restore_check=WORKTREE_NOT_CLEAN\n",
+        )
+
     def test_rehearsal_closes_backup_restore_ssr_and_cleanup_loop(self):
         script = self.rehearsal.read_text(encoding="utf-8")
         lower = script.lower()
@@ -291,9 +381,20 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
             '"$script_dir/restore-postgresql"',
             "PublishedEntryFacts",
             "PublishedTravelWarning",
-            'Client().get("/results/", secure=True)',
-            'body.count("id=\\"entry-card\\"") == 1',
-            'body.count("id=\\"warning-card\\"") == 1',
+            "subprocess.Popen(",
+            'str(project_dir / "runtime" / "gunicorn.conf.py")',
+            '"travel_readiness.wsgi:application"',
+            "pass_fds=(listener.fileno(),)",
+            "start_new_session=True",
+            "ssl.create_default_context(cafile=",
+            '"/healthz"',
+            '"/readyz"',
+            '"/releasez"',
+            '"/results/"',
+            "document.count('id=\"entry-card\"') == 1",
+            "document.count('id=\"warning-card\"') == 1",
+            "os.killpg(process_group, signal.SIGTERM)",
+            "os.killpg(process_group, signal.SIGKILL)",
             "DROP DATABASE",
             "writers_quiesced=confirmed",
             "integrity_manifest=match",
@@ -301,6 +402,9 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
             "entry_marker=match",
             "travel_warning_marker=match",
             "source_attribution=match",
+            "restored_wsgi_tls=match",
+            "release_identity=match",
+            "runtime_cleanup=match",
             "cleanup=match",
             "rehearsal_seconds=",
         ):
@@ -335,12 +439,19 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
         self.assertNotIn("DROP OWNED BY", script)
         self.assertNotIn("GRANT SELECT", script)
         self.assertNotIn("\nassert ", script)
+        self.assertNotIn("django.test", script)
+        self.assertNotIn("Client()", script)
+        self.assertNotIn("run-production", script)
         self.assertNotIn(".env.local", lower)
-        self.assertEqual(lower.count("mofa_travel_alarm_service_key"), 1)
+        self.assertEqual(lower.count("mofa_travel_alarm_service_key"), 2)
         self.assertIn("unset mofa_travel_alarm_service_key", lower)
         self.assertNotIn("set -x", lower)
         self.assertNotIn("eval ", lower)
         self.assertIn('escape_pgpass_field "$backup_password"', script)
+        self.assertIn("--porcelain=v1 --untracked-files=normal", script)
+        self.assertIn("_RUNTIME_ENV_NAMES", script)
+        self.assertIn("_FORBIDDEN_RUNTIME_ENV_NAMES", script)
+        self.assertIn('rm -f -- "$pgpass_file" "$tls_cert_file"', script)
 
         cleanup_call = script.rindex("if ! cleanup; then")
         success_receipt = script.index("printf 'backup_restore_check=ok")
@@ -359,10 +470,86 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
             "unset TRAVEL_READINESS_BACKUP_ROLE_PASSWORD"
         )
         first_database_child = script.index("require_pg_tool psql")
+        release_check = script.index('[ "$release_head" = "$release_sha" ]')
+        first_database_mutation = script.index('"CREATE ROLE')
         self.assertLess(admin_read, admin_unset)
         self.assertLess(admin_unset, first_database_child)
         self.assertLess(backup_read, backup_unset)
         self.assertLess(backup_unset, first_database_child)
+        self.assertLess(release_check, first_database_child)
+        self.assertLess(release_check, first_database_mutation)
+
+    def test_restored_wsgi_runtime_environment_is_an_exact_allowlist(self):
+        namespace = self.restored_wsgi_probe_namespace()
+        required = {
+            "TRAVEL_READINESS_DB_HOST": "127.0.0.1",
+            "TRAVEL_READINESS_DB_NAME": "restore_contract",
+            "TRAVEL_READINESS_DB_PASSWORD": "restore-role-input",
+            "TRAVEL_READINESS_DB_PORT": "5432",
+            "TRAVEL_READINESS_DB_USER": "restore_contract_role",
+            "TRAVEL_READINESS_RELEASE_SHA": "a" * 40,
+            "TRAVEL_READINESS_RESTORE_WORK_DIR": "/tmp/restore-contract",
+            "TRAVEL_READINESS_SECRET_KEY": "runtime-secret-input",
+            "TRAVEL_READINESS_TLS_CERT_FILE": "/tmp/restore-contract/ca.pem",
+            "TRAVEL_READINESS_TLS_KEY_FILE": "/tmp/restore-contract/key.pem",
+        }
+        runtime = namespace["_runtime_environment"](required, 17)
+        self.assertEqual(set(runtime), namespace["_RUNTIME_ENV_NAMES"])
+        self.assertFalse(
+            set(runtime) & namespace["_FORBIDDEN_RUNTIME_ENV_NAMES"]
+        )
+        self.assertEqual(runtime["TRAVEL_READINESS_BIND"], "fd://17")
+        self.assertEqual(runtime["TRAVEL_READINESS_HTTPS"], "1")
+        self.assertNotIn("HOME", runtime)
+        self.assertNotIn("PYTHONPATH", runtime)
+
+    def test_restored_wsgi_runtime_cleanup_kills_stubborn_process_group(self):
+        namespace = self.restored_wsgi_probe_namespace()
+        grandchild = (
+            "import signal,time; "
+            "signal.signal(signal.SIGTERM, signal.SIG_IGN); "
+            "time.sleep(60)"
+        )
+        leader = (
+            "import signal,subprocess,sys,time; "
+            "signal.signal(signal.SIGTERM, signal.SIG_IGN); "
+            f"subprocess.Popen([sys.executable, '-c', {grandchild!r}], "
+            "stdin=subprocess.DEVNULL, stdout=subprocess.DEVNULL, "
+            "stderr=subprocess.DEVNULL); "
+            "print('ready', flush=True); "
+            "time.sleep(60)"
+        )
+        process = subprocess.Popen(
+            [sys.executable, "-c", leader],
+            stdin=subprocess.DEVNULL,
+            stdout=subprocess.PIPE,
+            stderr=subprocess.PIPE,
+            text=True,
+            start_new_session=True,
+        )
+        try:
+            self.assertEqual(process.stdout.readline(), "ready\n")
+            self.assertTrue(
+                namespace["_terminate_process_group"](
+                    process,
+                    term_timeout=0.2,
+                    kill_timeout=3.0,
+                )
+            )
+            self.assertIsNotNone(process.returncode)
+            with self.assertRaises(ProcessLookupError):
+                os.killpg(process.pid, 0)
+        finally:
+            if process.returncode is None:
+                try:
+                    os.killpg(process.pid, signal.SIGKILL)
+                except ProcessLookupError:
+                    pass
+                process.wait(timeout=3)
+            if process.stdout is not None:
+                process.stdout.close()
+            if process.stderr is not None:
+                process.stderr.close()
 
     def test_rehearsal_secret_input_names_do_not_reach_database_children(self):
         with tempfile.TemporaryDirectory() as temporary:
@@ -383,8 +570,22 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
                 encoding="utf-8",
             )
             fake_psql.chmod(fake_psql.stat().st_mode | stat.S_IXUSR)
+            fake_git = fake_bin / "git"
+            fake_git.write_text(
+                "#!/bin/sh\n"
+                "[ -z \"${TRAVEL_READINESS_BACKUP_RESTORE_ADMIN_PASSWORD:-}\" ] || exit 97\n"
+                "[ -z \"${TRAVEL_READINESS_BACKUP_ROLE_PASSWORD:-}\" ] || exit 97\n"
+                "case \"$*\" in\n"
+                "  *'rev-parse --verify HEAD'*) printf '%s\\n' \"$FAKE_RELEASE_SHA\"; exit 0 ;;\n"
+                "  *'status --porcelain=v1 --untracked-files=normal'*) exit 0 ;;\n"
+                "esac\n"
+                "exit 98\n",
+                encoding="utf-8",
+            )
+            fake_git.chmod(fake_git.stat().st_mode | stat.S_IXUSR)
             environment = os.environ.copy()
             environment["PATH"] = f"{fake_bin}:{environment.get('PATH', '')}"
+            environment["FAKE_RELEASE_SHA"] = self.head_sha
             environment[
                 "TRAVEL_READINESS_BACKUP_RESTORE_ADMIN_PASSWORD"
             ] = "admin-input-must-be-unset"
diff --git a/scripts/check-backup-restore b/scripts/check-backup-restore
index 38439d9..0c2185c 100755
--- a/scripts/check-backup-restore
+++ b/scripts/check-backup-restore
@@ -5,11 +5,11 @@ set -eu
 umask 077
 LC_ALL=C
 export LC_ALL
-unset PGDATABASE PGUSER PGHOST PGPORT PGSERVICE PGSERVICEFILE PGOPTIONS PGPASSWORD
+unset PGDATABASE PGUSER PGHOST PGPORT PGSERVICE PGSERVICEFILE PGOPTIONS PGPASSWORD PGPASSFILE
 unset MOFA_TRAVEL_ALARM_SERVICE_KEY TRAVEL_READINESS_SECRET_KEY TRAVEL_READINESS_DB_PASSWORD
 
 usage() {
-    printf '%s\n' 'usage: check-backup-restore --host LOOPBACK --port PORT --admin-role ROLE --admin-password-env TRAVEL_READINESS_BACKUP_RESTORE_ADMIN_PASSWORD --backup-role ROLE --backup-password-env TRAVEL_READINESS_BACKUP_ROLE_PASSWORD --source-database DATABASE --database-prefix travel_readiness_restorecheck_NAME --safety-token BACKUP_RESTORE_REHEARSAL_DISPOSABLE:travel_readiness_restorecheck_NAME --writers-quiesced-confirmation WRITERS_QUIESCED'
+    printf '%s\n' 'usage: check-backup-restore --release-sha SHA --host LOOPBACK --port PORT --admin-role ROLE --admin-password-env TRAVEL_READINESS_BACKUP_RESTORE_ADMIN_PASSWORD --backup-role ROLE --backup-password-env TRAVEL_READINESS_BACKUP_ROLE_PASSWORD --source-database DATABASE --database-prefix travel_readiness_restorecheck_NAME --safety-token BACKUP_RESTORE_REHEARSAL_DISPOSABLE:travel_readiness_restorecheck_NAME --writers-quiesced-confirmation WRITERS_QUIESCED'
 }
 
 fail() {
@@ -33,6 +33,13 @@ is_port() {
     [ "$value" -le 65535 ] 2>/dev/null || return 1
 }
 
+is_release_sha() {
+    value=$1
+    [ "${#value}" -eq 40 ] || return 1
+    case "$value" in *[!0-9a-f]*) return 1 ;; esac
+}
+
+release_sha=''
 host=''
 port=''
 admin_role=''
@@ -51,12 +58,13 @@ while [ "$#" -gt 0 ]; do
             usage
             exit 0
             ;;
-        --host|--port|--admin-role|--admin-password-env|--backup-role|--backup-password-env|--source-database|--database-prefix|--safety-token|--writers-quiesced-confirmation)
+        --release-sha|--host|--port|--admin-role|--admin-password-env|--backup-role|--backup-password-env|--source-database|--database-prefix|--safety-token|--writers-quiesced-confirmation)
             [ "$#" -ge 2 ] || fail 'backup_restore_check=INVALID_ARGUMENTS' 64
             option=$1
             option_value=$2
             shift 2
             case "$option" in
+                --release-sha) [ -z "$release_sha" ] || fail 'backup_restore_check=INVALID_ARGUMENTS' 64; release_sha=$option_value ;;
                 --host) [ -z "$host" ] || fail 'backup_restore_check=INVALID_ARGUMENTS' 64; host=$option_value ;;
                 --port) [ -z "$port" ] || fail 'backup_restore_check=INVALID_ARGUMENTS' 64; port=$option_value ;;
                 --admin-role) [ -z "$admin_role" ] || fail 'backup_restore_check=INVALID_ARGUMENTS' 64; admin_role=$option_value ;;
@@ -73,12 +81,13 @@ while [ "$#" -gt 0 ]; do
     esac
 done
 
-[ -n "$host" ] && [ -n "$port" ] && [ -n "$admin_role" ] \
+[ -n "$release_sha" ] && [ -n "$host" ] && [ -n "$port" ] && [ -n "$admin_role" ] \
     && [ -n "$admin_password_env" ] && [ -n "$backup_role" ] \
     && [ -n "$backup_password_env" ] && [ -n "$source_database" ] \
     && [ -n "$database_prefix" ] && [ -n "$safety_token" ] \
     && [ -n "$writers_quiesced_confirmation" ] \
     || fail 'backup_restore_check=INVALID_ARGUMENTS' 64
+is_release_sha "$release_sha" || fail 'backup_restore_check=RELEASE_IDENTITY_INVALID' 65
 case "$host" in 127.0.0.1|localhost) ;; *) fail 'backup_restore_check=NON_LOOPBACK_REFUSED' 65 ;; esac
 is_port "$port" || fail 'backup_restore_check=INVALID_ARGUMENTS' 64
 is_identifier "$admin_role" || fail 'backup_restore_check=INVALID_ARGUMENTS' 64
@@ -127,6 +136,15 @@ python_bin="$project_dir/.venv/bin/python"
 # shellcheck source=postgresql-common
 . "$script_dir/postgresql-common"
 
+command -v git >/dev/null 2>&1 || fail 'backup_restore_check=REQUIRED_TOOL_MISSING' 69
+release_head=$(git -C "$project_dir" rev-parse --verify HEAD 2>/dev/null) \
+    || fail 'backup_restore_check=RELEASE_IDENTITY_FAILED' 69
+[ "$release_head" = "$release_sha" ] \
+    || fail 'backup_restore_check=RELEASE_IDENTITY_MISMATCH' 65
+worktree_state=$(git -C "$project_dir" status --porcelain=v1 --untracked-files=normal 2>/dev/null) \
+    || fail 'backup_restore_check=RELEASE_IDENTITY_FAILED' 69
+[ -z "$worktree_state" ] || fail 'backup_restore_check=WORKTREE_NOT_CLEAN' 65
+
 require_pg_tool psql || fail 'backup_restore_check=POSTGRESQL_18_6_REQUIRED' 69
 command -v openssl >/dev/null 2>&1 || fail 'backup_restore_check=REQUIRED_TOOL_MISSING' 69
 command -v mktemp >/dev/null 2>&1 || fail 'backup_restore_check=REQUIRED_TOOL_MISSING' 69
@@ -196,6 +214,8 @@ work_dir=$(mktemp -d /tmp/travel-readiness-backup-restore.XXXXXX 2>/dev/null) \
 case "$work_dir" in /tmp/travel-readiness-backup-restore.*) ;; *) fail 'backup_restore_check=TEMP_CREATE_FAILED' 73 ;; esac
 backup_dir="$work_dir/backup"
 pgpass_file="$work_dir/pgpass"
+tls_cert_file="$work_dir/runtime-ca.pem"
+tls_key_file="$work_dir/runtime-key.pem"
 restore_role_created=0
 restore_database_created=0
 
@@ -209,7 +229,8 @@ cleanup() {
     if [ "$restore_role_created" = 1 ]; then
         admin_psql postgres --quiet --command="DROP ROLE \"$restore_role\"" >/dev/null 2>&1 || cleanup_result=1
     fi
-    rm -f -- "$pgpass_file" "$backup_dir/database.dump" \
+    rm -f -- "$pgpass_file" "$tls_cert_file" "$tls_key_file" \
+        "$backup_dir/database.dump" \
         "$backup_dir/integrity.manifest" 2>/dev/null || cleanup_result=1
     rmdir "$backup_dir" 2>/dev/null || [ ! -e "$backup_dir" ] || cleanup_result=1
     rmdir "$work_dir" 2>/dev/null || cleanup_result=1
@@ -264,60 +285,493 @@ PGPASSFILE="$pgpass_file" "$script_dir/restore-postgresql" \
     --safety-token "RESTORE_DISPOSABLE:$restore_database"
 restore_finished=$(date +%s)
 
+openssl req -x509 -newkey rsa:2048 -sha256 -nodes -days 1 \
+    -subj '/CN=127.0.0.1' \
+    -addext 'subjectAltName=IP:127.0.0.1' \
+    -addext 'basicConstraints=critical,CA:TRUE' \
+    -addext 'keyUsage=critical,keyCertSign,digitalSignature,keyEncipherment' \
+    -addext 'extendedKeyUsage=serverAuth' \
+    -keyout "$tls_key_file" -out "$tls_cert_file" \
+    >/dev/null 2>&1 || fail 'backup_restore_check=TLS_MATERIAL_FAILED' 76
+chmod 600 "$tls_cert_file" "$tls_key_file" \
+    || fail 'backup_restore_check=TLS_MATERIAL_FAILED' 76
+
+if ! \
 TRAVEL_READINESS_SECRET_KEY="$django_secret" \
 TRAVEL_READINESS_DB_NAME="$restore_database" \
 TRAVEL_READINESS_DB_USER="$restore_role" \
 TRAVEL_READINESS_DB_PASSWORD="$restore_password" \
 TRAVEL_READINESS_DB_HOST="$host" \
 TRAVEL_READINESS_DB_PORT="$port" \
-TRAVEL_READINESS_ALLOWED_HOSTS='testserver,localhost' \
-TRAVEL_READINESS_BUILD=0 TRAVEL_READINESS_DEBUG=0 TRAVEL_READINESS_HTTPS=0 \
-DJANGO_SETTINGS_MODULE=travel_readiness.settings PYTHONPATH="$project_dir" \
-    "$python_bin" -s -c '
-import django
-django.setup()
-from django.test import Client
-from django.utils.html import escape
-from reviews.models import PublishedEntryFacts, PublishedTravelWarning
-def require(condition):
+TRAVEL_READINESS_TLS_CERT_FILE="$tls_cert_file" \
+TRAVEL_READINESS_TLS_KEY_FILE="$tls_key_file" \
+TRAVEL_READINESS_RESTORE_WORK_DIR="$work_dir" \
+TRAVEL_READINESS_RELEASE_SHA="$release_sha" \
+    "$python_bin" -I -B - "$project_dir" >/dev/null 2>&1 <<'PY'
+# RESTORED_WSGI_PROBE_START
+import errno
+import html
+import http.client
+import os
+from pathlib import Path
+import re
+import signal
+import socket
+import ssl
+import subprocess
+import sys
+import threading
+import time
+
+
+_LOG_LIMIT = 64 * 1024
+_BODY_LIMIT = 1024 * 1024
+_INTERRUPTED_SIGNAL = 0
+_RUNTIME_ENV_NAMES = frozenset(
+    {
+        "DJANGO_SETTINGS_MODULE",
+        "LANG",
+        "LC_ALL",
+        "PATH",
+        "PYTHONDONTWRITEBYTECODE",
+        "PYTHONHASHSEED",
+        "PYTHONUTF8",
+        "TMPDIR",
+        "TRAVEL_READINESS_ALLOWED_HOSTS",
+        "TRAVEL_READINESS_BIND",
+        "TRAVEL_READINESS_BUILD",
+        "TRAVEL_READINESS_DB_HOST",
+        "TRAVEL_READINESS_DB_NAME",
+        "TRAVEL_READINESS_DB_PASSWORD",
+        "TRAVEL_READINESS_DB_PORT",
+        "TRAVEL_READINESS_DB_USER",
+        "TRAVEL_READINESS_DEBUG",
+        "TRAVEL_READINESS_HTTPS",
+        "TRAVEL_READINESS_RELEASE_SHA",
+        "TRAVEL_READINESS_SECRET_KEY",
+        "TRAVEL_READINESS_TLS_CERT_FILE",
+        "TRAVEL_READINESS_TLS_KEY_FILE",
+        "TZ",
+    }
+)
+_FORBIDDEN_RUNTIME_ENV_NAMES = frozenset(
+    {
+        "MOFA_TRAVEL_ALARM_SERVICE_KEY",
+        "PGPASSFILE",
+        "PGPASSWORD",
+        "TRAVEL_READINESS_BACKUP_RESTORE_ADMIN_PASSWORD",
+        "TRAVEL_READINESS_BACKUP_ROLE_PASSWORD",
+    }
+)
+
+
+class _ProbeInterrupted(Exception):
+    def __init__(self, signum):
+        super().__init__(signum)
+        self.signum = signum
+
+
+def _require(condition):
     if not condition:
-        raise SystemExit(1)
-
-entry_pointer = PublishedEntryFacts.objects.select_related("current_publication__entry_fact_revision__country").get()
-warning_pointer = PublishedTravelWarning.objects.select_related("current_publication__travel_warning_revision__country").get()
-entry = entry_pointer.current_publication
-warning = warning_pointer.current_publication
-require(entry is not None and warning is not None)
-require(entry_pointer.version == entry.generation)
-require(warning_pointer.version == warning.generation)
-require(entry.entry_fact_revision.ordinary_passport_period_text == "90일")
-require(entry.entry_fact_revision.country.iso_alpha2 == "JP")
-require(warning.travel_warning_revision.source_alarm_level_code == "3")
-require(warning.travel_warning_revision.country.iso_alpha2 == "JP")
-response = Client().get("/results/", secure=True)
-require(response.status_code == 200)
-require(response.headers.get("Cache-Control") == "no-store")
-require(not response.cookies)
-body = response.content.decode("utf-8")
-require(body.count("id=\"entry-card\"") == 1)
-require(body.count("id=\"warning-card\"") == 1)
-entry_start = body.index("id=\"entry-card\"")
-warning_start = body.index("id=\"warning-card\"")
-entry_card = body[entry_start:warning_start]
-warning_card = body[warning_start:]
-require(any(f"data-state=\"{state}\"" in entry_card for state in ("ready", "stale")))
-require(any(f"data-state=\"{state}\"" in warning_card for state in ("ready", "stale")))
-require(f"generation {entry.generation}" in entry_card)
-require(f"generation {warning.generation}" in warning_card)
-require(entry.source_owner_snapshot in entry_card)
-require(entry.attribution_text_snapshot in entry_card)
-require(f"href=\"{escape(entry.source_locator_snapshot)}\"" in entry_card)
-require(warning.source_owner_snapshot in warning_card)
-require(warning.attribution_text_snapshot in warning_card)
-require(f"href=\"{escape(warning.source_locator_snapshot)}\"" in warning_card)
-for forbidden in ("ALLOWED", "DENIED", "입국 가능", "법적 판단"):
-    require(forbidden not in body)
-' >/dev/null 2>&1 || fail 'backup_restore_check=SSR_VERIFY_FAILED' 76
+        raise RuntimeError("fixed probe failure")
+
+
+def _drain_stream(stream, sink, state):
+    try:
+        while True:
+            chunk = stream.read(4096)
+            if not chunk:
+                break
+            remaining = _LOG_LIMIT - len(sink)
+            if remaining > 0:
+                sink.extend(chunk[:remaining])
+            if len(chunk) > remaining:
+                state["overflow"] = True
+    except Exception:
+        state["read_failed"] = True
+    finally:
+        try:
+            stream.close()
+        except Exception:
+            state["read_failed"] = True
+
+
+def _process_group_exists(process_group):
+    try:
+        os.killpg(process_group, 0)
+    except ProcessLookupError:
+        return False
+    except PermissionError:
+        return True
+    return True
+
+
+def _terminate_process_group(process, term_timeout=5.0, kill_timeout=5.0):
+    if process is None:
+        return True
+    process_group = process.pid
+    clean = True
+    try:
+        os.killpg(process_group, signal.SIGTERM)
+    except ProcessLookupError:
+        pass
+    except OSError as error:
+        if error.errno != errno.ESRCH:
+            clean = False
+    try:
+        process.wait(timeout=term_timeout)
+    except subprocess.TimeoutExpired:
+        try:
+            os.killpg(process_group, signal.SIGKILL)
+        except ProcessLookupError:
+            pass
+        except OSError as error:
+            if error.errno != errno.ESRCH:
+                clean = False
+        try:
+            process.wait(timeout=kill_timeout)
+        except subprocess.TimeoutExpired:
+            clean = False
+    except Exception:
+        clean = False
+    if process.returncode is None:
+        try:
+            process.kill()
+            process.wait(timeout=kill_timeout)
+        except Exception:
+            clean = False
+    if _process_group_exists(process_group):
+        try:
+            os.killpg(process_group, signal.SIGKILL)
+        except ProcessLookupError:
+            pass
+        except OSError as error:
+            if error.errno != errno.ESRCH:
+                clean = False
+        deadline = time.monotonic() + kill_timeout
+        while _process_group_exists(process_group) and time.monotonic() < deadline:
+            time.sleep(0.05)
+    return clean and process.returncode is not None and not _process_group_exists(process_group)
+
+
+def _https_get(context, port, path):
+    connection = http.client.HTTPSConnection(
+        "127.0.0.1",
+        port,
+        timeout=2.0,
+        context=context,
+    )
+    try:
+        connection.request(
+            "GET",
+            path,
+            headers={
+                "Accept": "text/html,application/json,text/plain",
+                "Connection": "close",
+                "User-Agent": "travel-readiness-restore-probe",
+            },
+        )
+        response = connection.getresponse()
+        body = response.read(_BODY_LIMIT + 1)
+        _require(len(body) <= _BODY_LIMIT)
+        headers = {name.lower(): value for name, value in response.getheaders()}
+        _require(not response.headers.get_all("Set-Cookie", []))
+        return response.status, headers, body
+    finally:
+        connection.close()
+
+
+def _exact_response(context, port, path, expected_status, expected_type, expected_body):
+    status, headers, body = _https_get(context, port, path)
+    _require(status == expected_status)
+    _require(headers.get("cache-control") == "no-store")
+    _require(headers.get("content-type") == expected_type)
+    _require(body == expected_body)
+
+
+def _interrupt(signum, _frame):
+    global _INTERRUPTED_SIGNAL
+    if _INTERRUPTED_SIGNAL == 0:
+        _INTERRUPTED_SIGNAL = signum
+
+
+def _raise_if_interrupted():
+    if _INTERRUPTED_SIGNAL:
+        raise _ProbeInterrupted(_INTERRUPTED_SIGNAL)
+
+
+def _runtime_environment(required, listener_fd):
+    runtime = {
+        "DJANGO_SETTINGS_MODULE": "travel_readiness.settings",
+        "LANG": "C",
+        "LC_ALL": "C",
+        "PATH": "/usr/bin:/bin",
+        "PYTHONDONTWRITEBYTECODE": "1",
+        "PYTHONHASHSEED": "0",
+        "PYTHONUTF8": "1",
+        "TMPDIR": required["TRAVEL_READINESS_RESTORE_WORK_DIR"],
+        "TRAVEL_READINESS_ALLOWED_HOSTS": "127.0.0.1",
+        "TRAVEL_READINESS_BIND": f"fd://{listener_fd}",
+        "TRAVEL_READINESS_BUILD": "0",
+        "TRAVEL_READINESS_DB_HOST": required["TRAVEL_READINESS_DB_HOST"],
+        "TRAVEL_READINESS_DB_NAME": required["TRAVEL_READINESS_DB_NAME"],
+        "TRAVEL_READINESS_DB_PASSWORD": required["TRAVEL_READINESS_DB_PASSWORD"],
+        "TRAVEL_READINESS_DB_PORT": required["TRAVEL_READINESS_DB_PORT"],
+        "TRAVEL_READINESS_DB_USER": required["TRAVEL_READINESS_DB_USER"],
+        "TRAVEL_READINESS_DEBUG": "0",
+        "TRAVEL_READINESS_HTTPS": "1",
+        "TRAVEL_READINESS_RELEASE_SHA": required["TRAVEL_READINESS_RELEASE_SHA"],
+        "TRAVEL_READINESS_SECRET_KEY": required["TRAVEL_READINESS_SECRET_KEY"],
+        "TRAVEL_READINESS_TLS_CERT_FILE": required["TRAVEL_READINESS_TLS_CERT_FILE"],
+        "TRAVEL_READINESS_TLS_KEY_FILE": required["TRAVEL_READINESS_TLS_KEY_FILE"],
+        "TZ": "UTC",
+    }
+    _require(set(runtime) == _RUNTIME_ENV_NAMES)
+    _require(not (_FORBIDDEN_RUNTIME_ENV_NAMES & set(runtime)))
+    return runtime
+
+
+def _load_publication_expectations(project_dir, runtime):
+    os.environ.clear()
+    os.environ.update(runtime)
+    sys.path.insert(0, str(project_dir))
+    import django
+
+    django.setup()
+    from django.db import connections
+    from reviews.models import PublishedEntryFacts, PublishedTravelWarning
+
+    entry_pointer = PublishedEntryFacts.objects.select_related(
+        "current_publication__entry_fact_revision__country"
+    ).get()
+    warning_pointer = PublishedTravelWarning.objects.select_related(
+        "current_publication__travel_warning_revision__country"
+    ).get()
+    entry = entry_pointer.current_publication
+    warning = warning_pointer.current_publication
+    _require(entry is not None and warning is not None)
+    _require(entry_pointer.version == entry.generation)
+    _require(warning_pointer.version == warning.generation)
+    _require(entry.entry_fact_revision.ordinary_passport_period_text == "90일")
+    _require(entry.entry_fact_revision.country.iso_alpha2 == "JP")
+    _require(warning.travel_warning_revision.source_alarm_level_code == "3")
+    _require(warning.travel_warning_revision.country.iso_alpha2 == "JP")
+    expectations = {
+        "entry": {
+            "generation": entry.generation,
+            "owner": entry.source_owner_snapshot,
+            "attribution": entry.attribution_text_snapshot,
+            "locator": entry.source_locator_snapshot,
+        },
+        "warning": {
+            "generation": warning.generation,
+            "owner": warning.source_owner_snapshot,
+            "attribution": warning.attribution_text_snapshot,
+            "locator": warning.source_locator_snapshot,
+        },
+    }
+    for expectation in expectations.values():
+        _require(type(expectation["generation"]) is int and expectation["generation"] > 0)
+        _require(all(type(expectation[name]) is str and expectation[name] for name in ("owner", "attribution", "locator")))
+        _require(expectation["locator"].startswith("https://"))
+    connections.close_all()
+    return expectations
+
+
+def _validate_results(body, expectations, sensitive_values):
+    for sensitive in sensitive_values:
+        _require(sensitive.encode("utf-8") not in body)
+    document = body.decode("utf-8", "strict")
+    _require(document.count('id="entry-card"') == 1)
+    _require(document.count('id="warning-card"') == 1)
+    entry_start = document.index('id="entry-card"')
+    warning_start = document.index('id="warning-card"')
+    _require(entry_start < warning_start)
+    cards = {
+        "entry": document[entry_start:warning_start],
+        "warning": document[warning_start:],
+    }
+    for name, card in cards.items():
+        opening_tag = card.split(">", 1)[0]
+        _require(re.search(r'data-state="(?:ready|stale)"', opening_tag) is not None)
+        expectation = expectations[name]
+        _require(f'generation {expectation["generation"]}' in card)
+        _require(html.escape(expectation["owner"], quote=True) in card)
+        _require(html.escape(expectation["attribution"], quote=True) in card)
+        escaped_locator = html.escape(expectation["locator"], quote=True)
+        _require(f'href="{escaped_locator}"' in card)
+    for forbidden in ("ALLOWED", "DENIED", "입국 가능", "법적 판단"):
+        _require(forbidden not in document)
+
+
+def main():
+    global _INTERRUPTED_SIGNAL
+    _INTERRUPTED_SIGNAL = 0
+    _require(len(sys.argv) == 2)
+    project_dir = Path(sys.argv[1]).resolve(strict=True)
+    _require((project_dir / "runtime" / "gunicorn.conf.py").is_file())
+    _require((project_dir / "travel_readiness" / "wsgi.py").is_file())
+    required_names = {
+        "TRAVEL_READINESS_DB_HOST",
+        "TRAVEL_READINESS_DB_NAME",
+        "TRAVEL_READINESS_DB_PASSWORD",
+        "TRAVEL_READINESS_DB_PORT",
+        "TRAVEL_READINESS_DB_USER",
+        "TRAVEL_READINESS_RELEASE_SHA",
+        "TRAVEL_READINESS_RESTORE_WORK_DIR",
+        "TRAVEL_READINESS_SECRET_KEY",
+        "TRAVEL_READINESS_TLS_CERT_FILE",
+        "TRAVEL_READINESS_TLS_KEY_FILE",
+    }
+    required = {name: os.environ.get(name, "") for name in required_names}
+    _require(all(required.values()))
+    _require(re.fullmatch(r"[0-9a-f]{40}", required["TRAVEL_READINESS_RELEASE_SHA"]) is not None)
+    _require(not (_FORBIDDEN_RUNTIME_ENV_NAMES & set(os.environ)))
+    sensitive_values = (
+        required["TRAVEL_READINESS_DB_PASSWORD"],
+        required["TRAVEL_READINESS_SECRET_KEY"],
+    )
+    listener = None
+    process = None
+    threads = []
+    stdout_log = bytearray()
+    stderr_log = bytearray()
+    stream_state = {"overflow": False, "read_failed": False}
+    probe_succeeded = False
+    interrupted_status = 1
+    old_handlers = {}
+    try:
+        for signum in (signal.SIGHUP, signal.SIGINT, signal.SIGTERM):
+            old_handlers[signum] = signal.signal(signum, _interrupt)
+        listener = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
+        listener.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
+        listener.bind(("127.0.0.1", 0))
+        listener.listen(128)
+        port = listener.getsockname()[1]
+        runtime = _runtime_environment(required, listener.fileno())
+        expectations = _load_publication_expectations(project_dir, runtime)
+        launcher = (
+            "import sys\n"
+            "sys.path.insert(0, sys.argv[1])\n"
+            "from gunicorn.app.wsgiapp import run\n"
+            "sys.argv = [\"gunicorn\", *sys.argv[2:]]\n"
+            "run()\n"
+        )
+        command = [
+            sys.executable,
+            "-I",
+            "-B",
+            "-c",
+            launcher,
+            str(project_dir),
+            "--chdir",
+            str(project_dir),
+            "--config",
+            str(project_dir / "runtime" / "gunicorn.conf.py"),
+            "--bind",
+            f"fd://{listener.fileno()}",
+            "travel_readiness.wsgi:application",
+        ]
+        process = subprocess.Popen(
+            command,
+            cwd=project_dir,
+            env=runtime,
+            stdin=subprocess.DEVNULL,
+            stdout=subprocess.PIPE,
+            stderr=subprocess.PIPE,
+            close_fds=True,
+            pass_fds=(listener.fileno(),),
+            start_new_session=True,
+            bufsize=0,
+        )
+        for stream, sink in ((process.stdout, stdout_log), (process.stderr, stderr_log)):
+            thread = threading.Thread(
+                target=_drain_stream,
+                args=(stream, sink, stream_state),
+                daemon=True,
+            )
+            thread.start()
+            threads.append(thread)
+        listener.close()
+        listener = None
+        context = ssl.create_default_context(cafile=required["TRAVEL_READINESS_TLS_CERT_FILE"])
+        context.check_hostname = True
+        deadline = time.monotonic() + 15.0
+        while True:
+            _raise_if_interrupted()
+            try:
+                _exact_response(
+                    context,
+                    port,
+                    "/healthz",
+                    200,
+                    "text/plain; charset=utf-8",
+                    b"ok\n",
+                )
+                break
+            except (ConnectionError, OSError, ssl.SSLError):
+                if time.monotonic() >= deadline:
+                    raise RuntimeError("fixed startup failure")
+                time.sleep(0.1)
+        _raise_if_interrupted()
+        _exact_response(
+            context,
+            port,
+            "/readyz",
+            200,
+            "application/json; charset=utf-8",
+            b'{"status":"ready"}',
+        )
+        _raise_if_interrupted()
+        _exact_response(
+            context,
+            port,
+            "/releasez",
+            200,
+            "application/json; charset=utf-8",
+            ('{"release_sha":"' + required["TRAVEL_READINESS_RELEASE_SHA"] + '"}').encode("ascii"),
+        )
+        _raise_if_interrupted()
+        status, headers, body = _https_get(context, port, "/results/")
+        _require(status == 200)
+        _require(headers.get("cache-control") == "no-store")
+        _require(headers.get("content-type") == "text/html; charset=utf-8")
+        _validate_results(body, expectations, sensitive_values)
+        del body
+        _raise_if_interrupted()
+        probe_succeeded = True
+    except _ProbeInterrupted as error:
+        interrupted_status = 128 + error.signum
+    except Exception:
+        interrupted_status = 1
+    finally:
+        if listener is not None:
+            try:
+                listener.close()
+            except Exception:
+                probe_succeeded = False
+        runtime_clean = _terminate_process_group(process)
+        for thread in threads:
+            thread.join(timeout=2.0)
+        threads_clean = all(not thread.is_alive() for thread in threads)
+        for signum, handler in old_handlers.items():
+            signal.signal(signum, handler)
+        combined_log = bytes(stdout_log) + bytes(stderr_log)
+        log_clean = not stream_state["overflow"] and not stream_state["read_failed"]
+        for sensitive in sensitive_values:
+            if sensitive.encode("utf-8") in combined_log:
+                log_clean = False
+        if not runtime_clean or not threads_clean or not log_clean:
+            probe_succeeded = False
+    return 0 if probe_succeeded else interrupted_status
+
+
+raise SystemExit(main())
+# RESTORED_WSGI_PROBE_END
+PY
+then
+    fail 'backup_restore_check=RESTORED_WSGI_TLS_VERIFY_FAILED' 76
+fi
 
 session_admin_counts=$(PGPASSFILE="$pgpass_file" run_psql \
     "$host" "$port" "$restore_database" "$restore_role" \
@@ -338,6 +792,6 @@ rehearsal_finished=$(date +%s)
 unset admin_password backup_password backup_pgpass_password
 unset restore_password restore_pgpass_password django_secret
 
-printf 'backup_restore_check=ok writers_quiesced=confirmed backup_seconds=%s restore_seconds=%s rehearsal_seconds=%s backup_bytes=%s session_rows=0 admin_log_rows=0 integrity_manifest=match publication_pointers=match ssr_results_status=200 entry_marker=match travel_warning_marker=match source_attribution=match cleanup=match\n' \
+printf 'backup_restore_check=ok writers_quiesced=confirmed backup_seconds=%s restore_seconds=%s rehearsal_seconds=%s backup_bytes=%s session_rows=0 admin_log_rows=0 integrity_manifest=match publication_pointers=match ssr_results_status=200 entry_marker=match travel_warning_marker=match source_attribution=match restored_wsgi_tls=match release_identity=match runtime_cleanup=match cleanup=match\n' \
     "$((backup_finished - backup_started))" "$((restore_finished - restore_started))" \
     "$((rehearsal_finished - rehearsal_started))" "$backup_bytes"


