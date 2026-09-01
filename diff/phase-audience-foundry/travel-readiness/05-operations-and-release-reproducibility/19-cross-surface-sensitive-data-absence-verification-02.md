## `fix(ops): isolate canonical data key`

diff --git a/operations/tests/test_browser_acceptance_harness.py b/operations/tests/test_browser_acceptance_harness.py
index 3f38690..a036dc5 100644
--- a/operations/tests/test_browser_acceptance_harness.py
+++ b/operations/tests/test_browser_acceptance_harness.py
@@ -407,6 +407,7 @@ class BrowserAcceptanceHarnessTests(unittest.TestCase):
         self.assertEqual(environment["NO_UPDATE_NOTIFIER"], "1")
         self.assertEqual(environment["PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD"], "1")
         for name in (
+            "DATA_GO_KR_SERVICE_KEY",
             "MOFA_TRAVEL_ALARM_SERVICE_KEY",
             "DATABASE_URL",
             "HTTPS_PROXY",
@@ -1955,6 +1956,10 @@ class BrowserAcceptanceHarnessTests(unittest.TestCase):
         self.assertIn("script_directory=${0%/*}", source)
         self.assertIn("exec /usr/bin/env -i PATH=/usr/bin:/bin", source)
         self.assertIn('"${repository_directory}/.venv/bin/python" -I -S -B', source)
+        self.assertIn(
+            "unset DATA_GO_KR_SERVICE_KEY MOFA_TRAVEL_ALARM_SERVICE_KEY",
+            source,
+        )
         self.assertNotIn("dirname", source)
         self.assertNotIn("set -x", source)
         self.assertEqual(stat.S_IMODE(entrypoint.stat().st_mode), 0o755)
diff --git a/operations/tests/test_browser_scenario_servers.py b/operations/tests/test_browser_scenario_servers.py
index 95ab1e3..a32666a 100644
--- a/operations/tests/test_browser_scenario_servers.py
+++ b/operations/tests/test_browser_scenario_servers.py
@@ -185,6 +185,7 @@ class BrowserScenarioCardTests(SimpleTestCase):
         for changed in (
             {"TRAVEL_READINESS_E2E_SCENARIO_MARKER": ""},
             {"TRAVEL_READINESS_DB_HOST": "192.0.2.1"},
+            {"DATA_GO_KR_SERVICE_KEY": "synthetic-test-only"},
             {"MOFA_TRAVEL_ALARM_SERVICE_KEY": "synthetic-test-only"},
         ):
             with self.subTest(changed=tuple(changed)):
@@ -277,6 +278,10 @@ class BrowserScenarioCardTests(SimpleTestCase):
                     scenario_server.build_scenario_cards("ready", lambda: cards)
 
     def test_e2e_marker_and_loopback_database_are_mandatory(self):
+        self.assertIn(
+            "unset DATA_GO_KR_SERVICE_KEY MOFA_TRAVEL_ALARM_SERVICE_KEY",
+            LAUNCHER.read_text(encoding="utf-8"),
+        )
         valid = e2e_environment()
         with patch.dict(os.environ, valid, clear=True):
             self.assertEqual(scenario_server._validate_environment("ready"), RELEASE_SHA)
@@ -287,6 +292,7 @@ class BrowserScenarioCardTests(SimpleTestCase):
         for changed in (
             {"TRAVEL_READINESS_E2E_SCENARIO_MARKER": ""},
             {"TRAVEL_READINESS_DB_HOST": "192.0.2.1"},
+            {"DATA_GO_KR_SERVICE_KEY": "synthetic-test-only"},
             {"MOFA_TRAVEL_ALARM_SERVICE_KEY": "synthetic-test-only"},
         ):
             environment = {**valid, **changed}
diff --git a/operations/tests/test_postgresql_backup_restore.py b/operations/tests/test_postgresql_backup_restore.py
index 3eaf66f..1925b00 100644
--- a/operations/tests/test_postgresql_backup_restore.py
+++ b/operations/tests/test_postgresql_backup_restore.py
@@ -153,6 +153,7 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
         fake_psql = fake_bin / "psql"
         fake_psql.write_text(
             "#!/bin/sh\n"
+            "[ -z \"${DATA_GO_KR_SERVICE_KEY:-}\" ] || exit 97\n"
             "[ -z \"${MOFA_TRAVEL_ALARM_SERVICE_KEY:-}\" ] || exit 97\n"
             "[ -z \"${TRAVEL_READINESS_SECRET_KEY:-}\" ] || exit 97\n"
             "[ -z \"${TRAVEL_READINESS_DB_PASSWORD:-}\" ] || exit 97\n"
@@ -189,6 +190,7 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
         fake_dump = fake_bin / "pg_dump"
         fake_dump.write_text(
             "#!/bin/sh\n"
+            "[ -z \"${DATA_GO_KR_SERVICE_KEY:-}\" ] || exit 97\n"
             "[ -z \"${MOFA_TRAVEL_ALARM_SERVICE_KEY:-}\" ] || exit 97\n"
             "[ -z \"${TRAVEL_READINESS_SECRET_KEY:-}\" ] || exit 97\n"
             "[ -z \"${TRAVEL_READINESS_DB_PASSWORD:-}\" ] || exit 97\n"
@@ -208,6 +210,7 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
         fake_restore = fake_bin / "pg_restore"
         fake_restore.write_text(
             "#!/bin/sh\n"
+            "[ -z \"${DATA_GO_KR_SERVICE_KEY:-}\" ] || exit 97\n"
             "[ -z \"${MOFA_TRAVEL_ALARM_SERVICE_KEY:-}\" ] || exit 97\n"
             "[ -z \"${TRAVEL_READINESS_SECRET_KEY:-}\" ] || exit 97\n"
             "[ -z \"${TRAVEL_READINESS_DB_PASSWORD:-}\" ] || exit 97\n"
@@ -223,6 +226,7 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
         env = os.environ.copy()
         env["PATH"] = f"{fake_bin}:{env.get('PATH', '')}"
         env["FAKE_PG_FIXTURES"] = str(fixtures)
+        env["DATA_GO_KR_SERVICE_KEY"] = "must-not-reach-child"
         env["MOFA_TRAVEL_ALARM_SERVICE_KEY"] = "must-not-reach-child"
         env["TRAVEL_READINESS_SECRET_KEY"] = "must-not-reach-child"
         env["TRAVEL_READINESS_DB_PASSWORD"] = "must-not-reach-child"
@@ -833,6 +837,7 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
                 encoding="utf-8"
             )
             for forbidden in (
+                "DATA_GO_KR_SERVICE_KEY",
                 "MOFA_TRAVEL_ALARM_SERVICE_KEY",
                 "serviceKey",
                 "destination",
diff --git a/operations/tests/test_runtime_config.py b/operations/tests/test_runtime_config.py
index 2aed2b6..98a70e0 100644
--- a/operations/tests/test_runtime_config.py
+++ b/operations/tests/test_runtime_config.py
@@ -1650,7 +1650,10 @@ class RuntimeConfigTests(SimpleTestCase):
             self.assertIn(dependency, verifier)
         self.assertIn('call_command("check", deploy=True', verifier)
         self.assertIn('call_command("migrate", check=True', verifier)
-        self.assertIn("unset MOFA_TRAVEL_ALARM_SERVICE_KEY", script)
+        self.assertIn(
+            "unset DATA_GO_KR_SERVICE_KEY MOFA_TRAVEL_ALARM_SERVICE_KEY",
+            script,
+        )
         self.assertIn("unset TRAVEL_READINESS_RELEASE_SHA", script)
         self.assertIn("PYTHONDONTWRITEBYTECODE=1", script)
         self.assertIn("RELEASE_RUNTIME_INVALID", script)
diff --git a/scripts/backup-postgresql b/scripts/backup-postgresql
index b84eccb..d2c4568 100755
--- a/scripts/backup-postgresql
+++ b/scripts/backup-postgresql
@@ -6,7 +6,8 @@ umask 077
 LC_ALL=C
 export LC_ALL
 unset PGDATABASE PGUSER PGHOST PGPORT PGSERVICE PGSERVICEFILE PGOPTIONS
-unset MOFA_TRAVEL_ALARM_SERVICE_KEY TRAVEL_READINESS_SECRET_KEY TRAVEL_READINESS_DB_PASSWORD
+unset DATA_GO_KR_SERVICE_KEY MOFA_TRAVEL_ALARM_SERVICE_KEY
+unset TRAVEL_READINESS_SECRET_KEY TRAVEL_READINESS_DB_PASSWORD
 
 SCRIPT_DIR=$(CDPATH='' cd "$(dirname "$0")" && pwd -P)
 # shellcheck source-path=SCRIPTDIR
diff --git a/scripts/check-backup-restore b/scripts/check-backup-restore
index 0c2185c..cccb2a5 100755
--- a/scripts/check-backup-restore
+++ b/scripts/check-backup-restore
@@ -6,7 +6,8 @@ umask 077
 LC_ALL=C
 export LC_ALL
 unset PGDATABASE PGUSER PGHOST PGPORT PGSERVICE PGSERVICEFILE PGOPTIONS PGPASSWORD PGPASSFILE
-unset MOFA_TRAVEL_ALARM_SERVICE_KEY TRAVEL_READINESS_SECRET_KEY TRAVEL_READINESS_DB_PASSWORD
+unset DATA_GO_KR_SERVICE_KEY MOFA_TRAVEL_ALARM_SERVICE_KEY
+unset TRAVEL_READINESS_SECRET_KEY TRAVEL_READINESS_DB_PASSWORD
 
 usage() {
     printf '%s\n' 'usage: check-backup-restore --release-sha SHA --host LOOPBACK --port PORT --admin-role ROLE --admin-password-env TRAVEL_READINESS_BACKUP_RESTORE_ADMIN_PASSWORD --backup-role ROLE --backup-password-env TRAVEL_READINESS_BACKUP_ROLE_PASSWORD --source-database DATABASE --database-prefix travel_readiness_restorecheck_NAME --safety-token BACKUP_RESTORE_REHEARSAL_DISPOSABLE:travel_readiness_restorecheck_NAME --writers-quiesced-confirmation WRITERS_QUIESCED'
@@ -356,6 +357,7 @@ _RUNTIME_ENV_NAMES = frozenset(
 )
 _FORBIDDEN_RUNTIME_ENV_NAMES = frozenset(
     {
+        "DATA_GO_KR_SERVICE_KEY",
         "MOFA_TRAVEL_ALARM_SERVICE_KEY",
         "PGPASSFILE",
         "PGPASSWORD",
diff --git a/scripts/check-browser-acceptance b/scripts/check-browser-acceptance
index 4ea4226..28fb58e 100755
--- a/scripts/check-browser-acceptance
+++ b/scripts/check-browser-acceptance
@@ -8,7 +8,7 @@ umask 077
 
 unset BASH_ENV CDPATH ENV IFS
 unset LD_LIBRARY_PATH DYLD_LIBRARY_PATH DYLD_INSERT_LIBRARIES
-unset MOFA_TRAVEL_ALARM_SERVICE_KEY PGPASSWORD PGOPTIONS
+unset DATA_GO_KR_SERVICE_KEY MOFA_TRAVEL_ALARM_SERVICE_KEY PGPASSWORD PGOPTIONS
 unset PYTHONHOME PYTHONPATH PYTHONSTARTUP SSLKEYLOGFILE
 IFS=' '
 
diff --git a/scripts/restore-postgresql b/scripts/restore-postgresql
index f9db89b..fe3e3a4 100755
--- a/scripts/restore-postgresql
+++ b/scripts/restore-postgresql
@@ -6,7 +6,8 @@ umask 077
 LC_ALL=C
 export LC_ALL
 unset PGDATABASE PGUSER PGHOST PGPORT PGSERVICE PGSERVICEFILE PGOPTIONS
-unset MOFA_TRAVEL_ALARM_SERVICE_KEY TRAVEL_READINESS_SECRET_KEY TRAVEL_READINESS_DB_PASSWORD
+unset DATA_GO_KR_SERVICE_KEY MOFA_TRAVEL_ALARM_SERVICE_KEY
+unset TRAVEL_READINESS_SECRET_KEY TRAVEL_READINESS_DB_PASSWORD
 
 SCRIPT_DIR=$(CDPATH='' cd "$(dirname "$0")" && pwd -P)
 # shellcheck source-path=SCRIPTDIR
diff --git a/scripts/run-browser-scenarios b/scripts/run-browser-scenarios
index 7f77218..f5d0f5e 100755
--- a/scripts/run-browser-scenarios
+++ b/scripts/run-browser-scenarios
@@ -8,7 +8,7 @@ umask 077
 
 unset BASH_ENV CDPATH ENV IFS
 unset LD_LIBRARY_PATH DYLD_LIBRARY_PATH DYLD_INSERT_LIBRARIES
-unset MOFA_TRAVEL_ALARM_SERVICE_KEY PGPASSWORD PGOPTIONS
+unset DATA_GO_KR_SERVICE_KEY MOFA_TRAVEL_ALARM_SERVICE_KEY PGPASSWORD PGOPTIONS
 unset PYTHONHOME PYTHONPATH PYTHONSTARTUP SSLKEYLOGFILE
 IFS=' '
 
diff --git a/scripts/run-production b/scripts/run-production
index e32ea2c..16588b6 100755
--- a/scripts/run-production
+++ b/scripts/run-production
@@ -40,7 +40,7 @@ debug_enabled=${TRAVEL_READINESS_DEBUG-0}
 
 unset BASH_ENV CDPATH ENV GUNICORN_CMD_ARGS IFS
 unset LD_LIBRARY_PATH DYLD_LIBRARY_PATH DYLD_INSERT_LIBRARIES
-unset MOFA_TRAVEL_ALARM_SERVICE_KEY PGPASSWORD PGOPTIONS
+unset DATA_GO_KR_SERVICE_KEY MOFA_TRAVEL_ALARM_SERVICE_KEY PGPASSWORD PGOPTIONS
 unset PYTHONHOME PYTHONPATH PYTHONSTARTUP SSLKEYLOGFILE
 unset TRAVEL_READINESS_BACKUP_RESTORE_ADMIN_PASSWORD
 unset TRAVEL_READINESS_BACKUP_ROLE_PASSWORD


## `fix(ops): close canonical secret absence boundary`

diff --git a/operations/sensitive_absence_cli.py b/operations/sensitive_absence_cli.py
index 91bcd12..defa0d5 100644
--- a/operations/sensitive_absence_cli.py
+++ b/operations/sensitive_absence_cli.py
@@ -6,7 +6,16 @@ import os
 
 # These removals precede argument parsing, path access, Django/psycopg imports,
 # subprocess creation and every possible diagnostic path.
-_TARGET_SECRET = os.environ.pop("MOFA_TRAVEL_ALARM_SERVICE_KEY", None)
+_TARGET_SECRET = os.environ.pop("DATA_GO_KR_SERVICE_KEY", None)
+_LEGACY_TARGET_SECRET = os.environ.pop("MOFA_TRAVEL_ALARM_SERVICE_KEY", None)
+if _TARGET_SECRET is None:
+    _TARGET_SECRET = _LEGACY_TARGET_SECRET
+elif (
+    _LEGACY_TARGET_SECRET is not None
+    and _LEGACY_TARGET_SECRET != _TARGET_SECRET
+):
+    _TARGET_SECRET = None
+_LEGACY_TARGET_SECRET = None
 _TRIP_MARKERS = os.environ.pop(
     "TRAVEL_READINESS_SENSITIVE_SCAN_TRIP_MARKERS", None
 )
diff --git a/operations/tests/test_release_artifact.py b/operations/tests/test_release_artifact.py
index 87bcd6e..9e0dab2 100644
--- a/operations/tests/test_release_artifact.py
+++ b/operations/tests/test_release_artifact.py
@@ -102,6 +102,8 @@ class ReleaseArtifactTests(unittest.TestCase):
             from pathlib import Path
             import django
 
+            if os.environ.get("DATA_GO_KR_SERVICE_KEY"):
+                raise RuntimeError("credential crossed build boundary")
             if os.environ.get("MOFA_TRAVEL_ALARM_SERVICE_KEY"):
                 raise RuntimeError("credential crossed build boundary")
             if os.environ.get("UNRELATED_RELEASE_PARENT_MARKER"):
@@ -147,6 +149,7 @@ class ReleaseArtifactTests(unittest.TestCase):
             "tools/uv",
             f"""
             #!/bin/sh
+            [ -z "${{DATA_GO_KR_SERVICE_KEY:-}}" ] || exit 97
             [ -z "${{MOFA_TRAVEL_ALARM_SERVICE_KEY:-}}" ] || exit 97
             [ -z "${{UNRELATED_RELEASE_PARENT_MARKER:-}}" ] || exit 97
             [ "${{PATH:-}}" = "/usr/bin:/bin" ] || exit 97
@@ -174,6 +177,7 @@ class ReleaseArtifactTests(unittest.TestCase):
         environment = {
             "LANG": "C",
             "LC_ALL": "C",
+            "DATA_GO_KR_SERVICE_KEY": self.secret_marker,
             "MOFA_TRAVEL_ALARM_SERVICE_KEY": self.secret_marker,
             "PATH": "/private/untrusted-parent-path",
             "TZ": "UTC",
diff --git a/operations/tests/test_sensitive_absence.py b/operations/tests/test_sensitive_absence.py
index 52f66d2..8ef1529 100644
--- a/operations/tests/test_sensitive_absence.py
+++ b/operations/tests/test_sensitive_absence.py
@@ -1241,6 +1241,7 @@ class ConfigurationAndCliTests(unittest.TestCase):
     def test_cli_pops_only_env_inputs_and_emits_one_fixed_failure_receipt(self):
         environment = {
             "PATH": "/usr/bin:/bin",
+            "DATA_GO_KR_SERVICE_KEY": SYNTHETIC_SECRET,
             "MOFA_TRAVEL_ALARM_SERVICE_KEY": SYNTHETIC_SECRET,
             "TRAVEL_READINESS_SENSITIVE_SCAN_TRIP_MARKERS": json.dumps(
                 [SYNTHETIC_TRIP], ensure_ascii=False
@@ -1275,12 +1276,16 @@ class ConfigurationAndCliTests(unittest.TestCase):
 
     def test_secret_pop_precedes_path_and_application_imports(self):
         source = CLI.read_text(encoding="utf-8")
-        secret_pop = source.index(
+        canonical_pop = source.index(
+            'os.environ.pop("DATA_GO_KR_SERVICE_KEY", None)'
+        )
+        legacy_pop = source.index(
             'os.environ.pop("MOFA_TRAVEL_ALARM_SERVICE_KEY", None)'
         )
-        self.assertLess(secret_pop, source.index("import sys"))
-        self.assertLess(secret_pop, source.index("from pathlib import Path"))
-        self.assertLess(secret_pop, source.index("def _bootstrap"))
+        for secret_pop in (canonical_pop, legacy_pop):
+            self.assertLess(secret_pop, source.index("import sys"))
+            self.assertLess(secret_pop, source.index("from pathlib import Path"))
+            self.assertLess(secret_pop, source.index("def _bootstrap"))
         self.assertNotIn("dotenv", source.lower())
         self.assertNotIn(".env.local", source)
 


## `test(ops): detect canonical key exposure`

diff --git a/e2e/browser_acceptance.py b/e2e/browser_acceptance.py
index a9622fe..bbe7afb 100644
--- a/e2e/browser_acceptance.py
+++ b/e2e/browser_acceptance.py
@@ -1783,7 +1783,7 @@ def _dom_privacy_source(*, origin: str, path: str) -> str:
     }} else if (hidden.length) return 'hidden-input';
     const allowedNames = new Set(['action','aria-atomic','aria-busy','aria-current','aria-describedby','aria-hidden','aria-invalid','aria-label','aria-labelledby','aria-live','aria-disabled','autocomplete','charset','class','content','data-error-summary','data-state','data-submit-button','data-submit-label','data-submit-status','data-submitting','data-trip-form','defer','for','href','id','lang','method','name','novalidate','rel','required','role','selected','src','style','tabindex','type','value']);
     const approvedHrefs = new Set(contract.href); const approvedSources = new Set(['/static/public_web/site.js']);
-    const forbidden = /(?:MOFA_TRAVEL_ALARM_SERVICE_KEY|serviceKey|api[_-]?key|raw[_-]?body|secret|authorization:)/i;
+    const forbidden = /(?:DATA_GO_KR_SERVICE_KEY|MOFA_TRAVEL_ALARM_SERVICE_KEY|service[_-]?key|api[_-]?key|raw[_-]?body|secret|authorization:)/i;
     const opaque = /[a-z0-9+/_=-]{{32,}}/i; const encoded = /(?:%[0-9a-f]{{2}}){{4,}}/i;
     const attributeLeak = [...document.querySelectorAll('*')].some((node) => [...node.attributes].some((attribute) => {{
       if (!allowedNames.has(attribute.name)) return true;
@@ -2572,7 +2572,7 @@ def results_javascript(*, origin: str, state: str, check: str) -> str:
   const hiddenLeak = await page.evaluate((approved) => {{
     const allowedNames = new Set(['aria-current','aria-describedby','aria-hidden','aria-label','aria-labelledby','charset','class','content','data-state','defer','href','id','lang','name','rel','role','src','style','tabindex']);
     const approvedHrefs = new Set(approved.href); const approvedSources = new Set(approved.src);
-    const forbidden = /(?:MOFA_TRAVEL_ALARM_SERVICE_KEY|serviceKey|api[_-]?key|raw[_-]?body|secret|authorization:)/i;
+    const forbidden = /(?:DATA_GO_KR_SERVICE_KEY|MOFA_TRAVEL_ALARM_SERVICE_KEY|service[_-]?key|api[_-]?key|raw[_-]?body|secret|authorization:)/i;
     const opaque = /[a-z0-9+/_=-]{{24,}}/i; const encoded = /(?:%[0-9a-f]{{2}}){{4,}}/i;
     const attributeLeak = [...document.querySelectorAll('*')].some((node) => [...node.attributes].some((attribute) => {{
       if (!allowedNames.has(attribute.name)) return true;
diff --git a/operations/tests/test_browser_acceptance_harness.py b/operations/tests/test_browser_acceptance_harness.py
index a036dc5..b0d3aaf 100644
--- a/operations/tests/test_browser_acceptance_harness.py
+++ b/operations/tests/test_browser_acceptance_harness.py
@@ -812,6 +812,8 @@ class BrowserAcceptanceHarnessTests(unittest.TestCase):
         self.assertIn("hidden.length !== 1", form_source)
         self.assertIn("allowedNames = new Set", form_source)
         self.assertIn("dom-privacy-", form_source)
+        self.assertIn("DATA_GO_KR_SERVICE_KEY", form_source)
+        self.assertIn("service[_-]?key", form_source)
         self.assertIn("crypto.subtle.digest('SHA-256'", form_source)
 
         cookie_source = acceptance.csrf_cookie_contract_javascript(
diff --git a/operations/tests/test_dependency_gate.py b/operations/tests/test_dependency_gate.py
index de8c188..f97db6a 100644
--- a/operations/tests/test_dependency_gate.py
+++ b/operations/tests/test_dependency_gate.py
@@ -1200,6 +1200,7 @@ class DependencyContractTests(unittest.TestCase):
         fixed_temporary = Path("/private/tmp" if gate.sys.platform == "darwin" else "/tmp")
         self.assertTrue(all(workspace.parent == fixed_temporary for workspace in workspaces))
         for _, _, environment, _ in calls:
+            self.assertNotIn("DATA_GO_KR_SERVICE_KEY", environment)
             self.assertNotIn("MOFA_TRAVEL_ALARM_SERVICE_KEY", environment)
             self.assertNotIn("DATABASE_URL", environment)
             self.assertNotIn("HTTP_PROXY", environment)


