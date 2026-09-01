## `fix(ops): align live replay source boundary`

diff --git a/operations/management/commands/check_live_parser_replay.py b/operations/management/commands/check_live_parser_replay.py
index f54206a..fae3bdf 100644
--- a/operations/management/commands/check_live_parser_replay.py
+++ b/operations/management/commands/check_live_parser_replay.py
@@ -10,6 +10,10 @@ from operations.source_replay import (
     LiveParserReplayReceipt,
     verify_live_parser_replay,
 )
+from travel_windows.contracts import (
+    DATA_GO_SECRET_REFERENCE,
+    LEGACY_DATA_GO_SECRET_REFERENCE,
+)
 
 
 _SUCCESS_RECEIPT = LiveParserReplayReceipt(
@@ -23,14 +27,13 @@ _SUCCESS_RECEIPT = LiveParserReplayReceipt(
     typed_revisions=2,
 )
 
-_SECRET_REFERENCE = "MOFA_TRAVEL_ALARM_SERVICE_KEY"
-
-
 def _invoke_without_exception_context():
-    secret_value = os.environ.pop(_SECRET_REFERENCE, None)
+    canonical_value = os.environ.pop(DATA_GO_SECRET_REFERENCE, None)
+    legacy_value = os.environ.pop(LEGACY_DATA_GO_SECRET_REFERENCE, None)
+    secret_value = canonical_value or legacy_value
 
     def one_secret(reference_name: str):
-        if reference_name != _SECRET_REFERENCE:
+        if reference_name != DATA_GO_SECRET_REFERENCE:
             return None
         return secret_value
 
@@ -40,6 +43,8 @@ def _invoke_without_exception_context():
         return None
     finally:
         secret_value = None
+        canonical_value = None
+        legacy_value = None
 
 
 def _closed_failed_receipt(receipt) -> bool:
diff --git a/operations/source_replay.py b/operations/source_replay.py
index c384709..6a53ddb 100644
--- a/operations/source_replay.py
+++ b/operations/source_replay.py
@@ -17,6 +17,7 @@ from typing import Callable
 from django.db import connection
 
 from entry_requirements.ingestion import (
+    ENTRY_SOURCE_CODE,
     EntryIngestionCode,
     EntryIngestionOutcome,
     ingest_entry_snapshot,
@@ -39,7 +40,11 @@ from sources.models import (
     SourceConfiguration,
     SourceRightsDecision,
 )
+from sources.management.commands.register_approved_sources import (
+    APPROVED_SOURCE_SPECS,
+)
 from travel_warnings.ingestion import (
+    WARNING_SOURCE_CODE,
     TravelWarningIngestionCode,
     TravelWarningIngestionOutcome,
     ingest_travel_warning,
@@ -86,6 +91,9 @@ _DATABASE_PATTERN = re.compile(
 )
 _SAFETY_TOKEN_ENV = "TRAVEL_READINESS_REPLAY_SAFETY_TOKEN"
 _SAFETY_TOKEN_PREFIX = "LIVE_PARSER_REPLAY_DISPOSABLE:"
+_APPROVED_SOURCE_CODES = frozenset(
+    spec.code for spec in APPROVED_SOURCE_SPECS
+)
 
 EntryIngestor = Callable[..., EntryIngestionOutcome]
 WarningIngestor = Callable[..., TravelWarningIngestionOutcome]
@@ -154,6 +162,14 @@ def _storage_boundary_is_clean() -> bool:
                        'raw_retention_seconds'
                    )
                )
+               AND NOT (
+                   column_name = 'destination_airport_id'
+                   AND table_name IN (
+                       'travel_windows_flightschedule',
+                       'travel_windows_routedurationrevision',
+                       'travel_windows_flightpublicationduration'
+                   )
+               )
             """
         )
         forbidden_columns = cursor.fetchone()
@@ -225,7 +241,7 @@ def _no_evidence_rows() -> bool:
 
 
 def _source_baseline_is_exact() -> bool:
-    expected_codes = {"MOFA_ENTRY_CSV", "MOFA_TRAVEL_ALARM_API_JP"}
+    expected_codes = _APPROVED_SOURCE_CODES
     active_codes = set(
         SourceConfiguration.objects.filter(
             state=SourceConfiguration.State.ACTIVE,
@@ -244,9 +260,9 @@ def _source_baseline_is_exact() -> bool:
         evidence_retention=SourceRightsDecision.Retention.PRODUCT_HISTORY,
     )
     return (
-        SourceConfiguration.objects.count() == 2
+        SourceConfiguration.objects.count() == len(expected_codes)
         and active_codes == expected_codes
-        and SourceRightsDecision.objects.count() == 2
+        and SourceRightsDecision.objects.count() == len(expected_codes)
         and set(approved_rights.values_list("source__code", flat=True))
         == expected_codes
         and _no_evidence_rows()
@@ -254,7 +270,7 @@ def _source_baseline_is_exact() -> bool:
 
 
 def _evidence_is_exact() -> bool:
-    expected_sources = {"MOFA_ENTRY_CSV", "MOFA_TRAVEL_ALARM_API_JP"}
+    expected_sources = {ENTRY_SOURCE_CODE, WARNING_SOURCE_CODE}
     attempts = FetchAttempt.objects.all()
     artifacts = SourceArtifact.objects.all()
     parse_runs = ParseRun.objects.all()
diff --git a/operations/tests/test_live_parser_replay.py b/operations/tests/test_live_parser_replay.py
index ed4d5cf..fc4d152 100644
--- a/operations/tests/test_live_parser_replay.py
+++ b/operations/tests/test_live_parser_replay.py
@@ -27,11 +27,20 @@ from operations.source_replay import (
     WARNING_POINTER_ID,
     VERIFIED,
     LiveParserReplayReceipt,
+    _APPROVED_SOURCE_CODES,
     _disposable_database_is_approved,
     _empty_current_pointers_are_exact,
+    _storage_boundary_is_clean,
     verify_live_parser_replay,
 )
+from sources.management.commands.register_approved_sources import (
+    APPROVED_SOURCE_SPECS,
+)
 from sources.models import ParseRun
+from travel_windows.contracts import (
+    DATA_GO_SECRET_REFERENCE,
+    LEGACY_DATA_GO_SECRET_REFERENCE,
+)
 from travel_warnings.ingestion import (
     TravelWarningIngestionCode,
     TravelWarningIngestionOutcome,
@@ -148,7 +157,7 @@ class LiveParserReplayTests(SimpleTestCase):
         def warning_ingestor(*, parser, secret_loader):
             calls["warning_ingest"] += 1
             self.assertEqual(
-                secret_loader("MOFA_TRAVEL_ALARM_SERVICE_KEY"),
+                secret_loader(DATA_GO_SECRET_REFERENCE),
                 SECRET_MARKER,
             )
             self.assertEqual(parser(RAW_MARKER), _warning_success())
@@ -264,6 +273,29 @@ class LiveParserReplayTests(SimpleTestCase):
         self.assertIn("ingest_entry_snapshot", source)
         self.assertIn("ingest_travel_warning", source)
 
+    def test_registered_source_allowlist_is_the_exact_replay_baseline(self):
+        self.assertEqual(
+            _APPROVED_SOURCE_CODES,
+            frozenset(spec.code for spec in APPROVED_SOURCE_SPECS),
+        )
+        self.assertEqual(len(_APPROVED_SOURCE_CODES), 8)
+
+    def test_storage_boundary_allows_only_the_three_typed_airport_links(self):
+        fake_connection = MagicMock()
+        cursor = fake_connection.cursor.return_value.__enter__.return_value
+        cursor.fetchone.side_effect = ((0,), (0,))
+        with patch("operations.source_replay.connection", fake_connection):
+            self.assertTrue(_storage_boundary_is_clean())
+
+        statement = cursor.execute.call_args_list[0].args[0]
+        self.assertIn("column_name = 'destination_airport_id'", statement)
+        for table in (
+            "travel_windows_flightschedule",
+            "travel_windows_routedurationrevision",
+            "travel_windows_flightpublicationduration",
+        ):
+            self.assertIn(table, statement)
+
     def test_disposable_database_requires_exact_name_token_and_postgresql_18_6(self):
         database_name = "travel_readiness_replaycheck_abc12345"
         fake_connection = MagicMock()
@@ -328,17 +360,29 @@ class LiveParserReplayTests(SimpleTestCase):
         )
 
     def test_management_command_pops_secret_before_invoking_verifier(self):
-        secret_name = "MOFA_TRAVEL_ALARM_SERVICE_KEY"
         receipt, _ = self._verify()
+        legacy_marker = "private-legacy-live-replay-marker"
 
         def verifier(*, secret_loader):
-            self.assertNotIn(secret_name, os.environ)
+            self.assertNotIn(DATA_GO_SECRET_REFERENCE, os.environ)
+            self.assertNotIn(LEGACY_DATA_GO_SECRET_REFERENCE, os.environ)
             self.assertIsNone(secret_loader("UNAPPROVED_REFERENCE"))
-            self.assertEqual(secret_loader(secret_name), SECRET_MARKER)
+            self.assertIsNone(secret_loader(LEGACY_DATA_GO_SECRET_REFERENCE))
+            self.assertEqual(
+                secret_loader(DATA_GO_SECRET_REFERENCE),
+                SECRET_MARKER,
+            )
             return receipt
 
         with (
-            patch.dict(os.environ, {secret_name: SECRET_MARKER}, clear=False),
+            patch.dict(
+                os.environ,
+                {
+                    DATA_GO_SECRET_REFERENCE: SECRET_MARKER,
+                    LEGACY_DATA_GO_SECRET_REFERENCE: legacy_marker,
+                },
+                clear=False,
+            ),
             patch(
                 "operations.management.commands.check_live_parser_replay."
                 "verify_live_parser_replay",
@@ -346,20 +390,28 @@ class LiveParserReplayTests(SimpleTestCase):
             ),
         ):
             observed = check_live_parser_replay._invoke_without_exception_context()
-            self.assertNotIn(secret_name, os.environ)
+            self.assertNotIn(DATA_GO_SECRET_REFERENCE, os.environ)
+            self.assertNotIn(LEGACY_DATA_GO_SECRET_REFERENCE, os.environ)
             self.assertNotIn(SECRET_MARKER, repr(observed))
+            self.assertNotIn(legacy_marker, repr(observed))
         self.assertEqual(observed, receipt)
 
-    def test_management_command_sanitizes_exception_after_secret_pop(self):
-        secret_name = "MOFA_TRAVEL_ALARM_SERVICE_KEY"
+    def test_management_command_uses_legacy_fallback_and_sanitizes_exception(self):
+        environment = os.environ.copy()
+        environment.pop(DATA_GO_SECRET_REFERENCE, None)
+        environment[LEGACY_DATA_GO_SECRET_REFERENCE] = SECRET_MARKER
 
         def failed_verifier(*, secret_loader):
-            self.assertNotIn(secret_name, os.environ)
-            self.assertEqual(secret_loader(secret_name), SECRET_MARKER)
+            self.assertNotIn(DATA_GO_SECRET_REFERENCE, os.environ)
+            self.assertNotIn(LEGACY_DATA_GO_SECRET_REFERENCE, os.environ)
+            self.assertEqual(
+                secret_loader(DATA_GO_SECRET_REFERENCE),
+                SECRET_MARKER,
+            )
             raise RuntimeError(SECRET_MARKER)
 
         with (
-            patch.dict(os.environ, {secret_name: SECRET_MARKER}, clear=False),
+            patch.dict(os.environ, environment, clear=True),
             patch(
                 "operations.management.commands.check_live_parser_replay."
                 "verify_live_parser_replay",
@@ -367,7 +419,8 @@ class LiveParserReplayTests(SimpleTestCase):
             ),
         ):
             observed = check_live_parser_replay._invoke_without_exception_context()
-            self.assertNotIn(secret_name, os.environ)
+            self.assertNotIn(DATA_GO_SECRET_REFERENCE, os.environ)
+            self.assertNotIn(LEGACY_DATA_GO_SECRET_REFERENCE, os.environ)
         self.assertIsNone(observed)
 
     def test_management_command_rejects_an_unclosed_receipt(self):
diff --git a/operations/tests/test_live_parser_replay_runner.py b/operations/tests/test_live_parser_replay_runner.py
index 3af9314..68801d6 100644
--- a/operations/tests/test_live_parser_replay_runner.py
+++ b/operations/tests/test_live_parser_replay_runner.py
@@ -159,6 +159,7 @@ class LiveParserReplayRunnerTests(unittest.TestCase):
     def test_both_secret_references_are_required_without_disclosure(self):
         environment = os.environ.copy()
         environment.pop("TRAVEL_READINESS_REPLAY_ADMIN_PASSWORD", None)
+        environment.pop("DATA_GO_KR_SERVICE_KEY", None)
         environment.pop("MOFA_TRAVEL_ALARM_SERVICE_KEY", None)
         missing_admin = self.run_script(
             None,
@@ -212,13 +213,15 @@ class LiveParserReplayRunnerTests(unittest.TestCase):
                 fi
                 case " $* " in
                     *" check_live_parser_replay "*)
-                        [ "${MOFA_TRAVEL_ALARM_SERVICE_KEY:-}" = "${FAKE_WARNING_SECRET}" ] || exit 91
+                        [ "${DATA_GO_KR_SERVICE_KEY:-}" = "${FAKE_WARNING_SECRET}" ] || exit 91
+                        [ -z "${MOFA_TRAVEL_ALARM_SERVICE_KEY:-}" ] || exit 98
                         [ "${TRAVEL_READINESS_REPLAY_SAFETY_TOKEN:-}" = "${FAKE_SAFETY_TOKEN}" ] || exit 92
                         printf '%s\\n' live >> "$FAKE_LOG"
                         [ "${FAKE_LIVE_FAIL:-0}" = 0 ] || exit 93
                         printf '%s\\n' 'live_parser_replay_result=VERIFIED sources=2 requests=2 parser_invocations=4 durable_attempts=2 artifacts=2 parse_runs=2 typed=2'
                         ;;
                     *)
+                        [ -z "${DATA_GO_KR_SERVICE_KEY:-}" ] || exit 99
                         [ -z "${MOFA_TRAVEL_ALARM_SERVICE_KEY:-}" ] || exit 94
                         printf '%s\\n' manage >> "$FAKE_LOG"
                         [ "${FAKE_MANAGE_FAIL:-0}" = 0 ] || exit 97
@@ -240,7 +243,8 @@ class LiveParserReplayRunnerTests(unittest.TestCase):
                     printf '%s\\n' 'psql (PostgreSQL) 18.6'
                     exit 0
                 fi
-                [ -z "${MOFA_TRAVEL_ALARM_SERVICE_KEY:-}" ] || exit 95
+                [ -z "${DATA_GO_KR_SERVICE_KEY:-}" ] || exit 95
+                [ -z "${MOFA_TRAVEL_ALARM_SERVICE_KEY:-}" ] || exit 96
                 input=$(cat)
                 combined="$* $input"
                 case "$combined" in
@@ -298,7 +302,7 @@ class LiveParserReplayRunnerTests(unittest.TestCase):
             {
                 "PATH": f"{fake_bin}:/usr/bin:/bin",
                 "TRAVEL_READINESS_REPLAY_ADMIN_PASSWORD": "admin-marker",
-                "MOFA_TRAVEL_ALARM_SERVICE_KEY": "warning-marker",
+                "DATA_GO_KR_SERVICE_KEY": "warning-marker",
                 "FAKE_WARNING_SECRET": "warning-marker",
                 "FAKE_SAFETY_TOKEN": (
                     f"LIVE_PARSER_REPLAY_DISPOSABLE:{self.database_name}"
diff --git a/scripts/check-live-parser-replay b/scripts/check-live-parser-replay
index 554097e..c5237f2 100755
--- a/scripts/check-live-parser-replay
+++ b/scripts/check-live-parser-replay
@@ -157,9 +157,13 @@ command -v printenv >/dev/null 2>&1 \
     || fail 'live_parser_replay_check=REQUIRED_TOOL_MISSING' 69
 admin_password=$(printenv "$admin_password_env" 2>/dev/null) \
     || fail 'live_parser_replay_check=ADMIN_PASSWORD_MISSING' 66
-warning_secret=$(printenv MOFA_TRAVEL_ALARM_SERVICE_KEY 2>/dev/null) \
-    || fail 'live_parser_replay_check=WARNING_SECRET_MISSING' 66
-unset TRAVEL_READINESS_REPLAY_ADMIN_PASSWORD MOFA_TRAVEL_ALARM_SERVICE_KEY
+data_go_secret=$(printenv DATA_GO_KR_SERVICE_KEY 2>/dev/null) || data_go_secret=''
+if [ -z "$data_go_secret" ]; then
+    data_go_secret=$(printenv MOFA_TRAVEL_ALARM_SERVICE_KEY 2>/dev/null) \
+        || fail 'live_parser_replay_check=WARNING_SECRET_MISSING' 66
+fi
+unset TRAVEL_READINESS_REPLAY_ADMIN_PASSWORD DATA_GO_KR_SERVICE_KEY
+unset MOFA_TRAVEL_ALARM_SERVICE_KEY
 unset PGDATABASE PGUSER PGHOST PGPORT PGSERVICE PGSERVICEFILE PGOPTIONS PGPASSWORD
 unset TRAVEL_READINESS_SECRET_KEY TRAVEL_READINESS_DB_NAME
 unset TRAVEL_READINESS_DB_USER TRAVEL_READINESS_DB_PASSWORD
@@ -168,9 +172,9 @@ unset TRAVEL_READINESS_REPLAY_SAFETY_TOKEN
 
 [ -n "$admin_password" ] && [ "${#admin_password}" -le 1024 ] \
     || fail 'live_parser_replay_check=ADMIN_PASSWORD_INVALID' 66
-[ -n "$warning_secret" ] && [ "${#warning_secret}" -le 512 ] \
+[ -n "$data_go_secret" ] && [ "${#data_go_secret}" -le 512 ] \
     || fail 'live_parser_replay_check=WARNING_SECRET_INVALID' 66
-case "$admin_password$warning_secret" in
+case "$admin_password$data_go_secret" in
     *'
 '*) fail 'live_parser_replay_check=SECRET_INVALID' 66 ;;
 esac
@@ -249,7 +253,7 @@ run_manage() {
 }
 
 run_live_replay() {
-    MOFA_TRAVEL_ALARM_SERVICE_KEY="$warning_secret" \
+    DATA_GO_KR_SERVICE_KEY="$data_go_secret" \
     TRAVEL_READINESS_REPLAY_SAFETY_TOKEN="$safety_token" \
         run_manage check_live_parser_replay --verbosity 0
 }
@@ -340,7 +344,7 @@ live_receipt=$(run_live_replay 2>/dev/null) \
 
 cleanup || fail 'live_parser_replay_check_cleanup=FAILED' 77
 trap - EXIT HUP INT TERM
-warning_secret=''
+data_go_secret=''
 admin_password=''
 django_secret=''
 printf '%s\n' "$INNER_RECEIPT cleanup=match"


