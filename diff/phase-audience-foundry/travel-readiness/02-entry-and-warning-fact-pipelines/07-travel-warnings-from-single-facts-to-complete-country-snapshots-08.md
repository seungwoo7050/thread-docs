## `fix(warnings): reconcile prior ingestion attempts`

diff --git a/travel_warnings/ingestion.py b/travel_warnings/ingestion.py
index bb0f62d..b57c101 100644
--- a/travel_warnings/ingestion.py
+++ b/travel_warnings/ingestion.py
@@ -336,14 +336,15 @@ def _discard_warning_ingestion_connection() -> None:
         pass
 
 
-def _close_abandoned_started_attempts(
-    contract: _WarningIngestionContract = _V1_WARNING_CONTRACT,
-) -> None:
+def _close_abandoned_started_attempts() -> None:
     with transaction.atomic(durable=True):
         abandoned = list(
             FetchAttempt.objects.select_for_update()
             .filter(
-                source__code=contract.source_code,
+                source__code__in=(
+                    _V1_WARNING_CONTRACT.source_code,
+                    _V2_WARNING_CONTRACT.source_code,
+                ),
                 result=FetchAttempt.Result.STARTED,
             )
             .only("id", "started_at")
@@ -1515,7 +1516,7 @@ def _run_travel_warning_ingestion(
         )
     try:
         try:
-            _close_abandoned_started_attempts(contract)
+            _close_abandoned_started_attempts()
         except Exception:
             return TravelWarningIngestionOutcome(
                 TravelWarningIngestionCode.PERSISTENCE_FAILED,
@@ -1602,7 +1603,7 @@ def _run_travel_warning_ingestion(
                 closed_attempt = _close_attempt(attempt.id, verified)
             except Exception:
                 try:
-                    _close_abandoned_started_attempts(contract)
+                    _close_abandoned_started_attempts()
                 except BaseException:
                     pass
                 loaded_secret = None
@@ -1617,7 +1618,7 @@ def _run_travel_warning_ingestion(
                 exception_kind = _base_exception_kind(exception)
                 del exception
                 try:
-                    _close_abandoned_started_attempts(contract)
+                    _close_abandoned_started_attempts()
                 except BaseException:
                     pass
                 loaded_secret = None
@@ -1665,7 +1666,7 @@ def _run_travel_warning_ingestion(
         exception_kind = _base_exception_kind(exception)
         del exception
         try:
-            _close_abandoned_started_attempts(contract)
+            _close_abandoned_started_attempts()
         except BaseException:
             pass
         loaded_secret = None
diff --git a/travel_warnings/tests/test_snapshot_ingestion.py b/travel_warnings/tests/test_snapshot_ingestion.py
index 5582e8e..e4be837 100644
--- a/travel_warnings/tests/test_snapshot_ingestion.py
+++ b/travel_warnings/tests/test_snapshot_ingestion.py
@@ -1,5 +1,6 @@
 import hashlib
 import json
+import uuid
 from io import StringIO
 from unittest.mock import patch
 
@@ -7,15 +8,22 @@ from django.core.management import call_command
 from django.test import SimpleTestCase, TransactionTestCase
 
 from countries.models import SUPPORTED_COUNTRY_ROWS, Country
-from sources.models import FetchAttempt, ParseRun, SourceArtifact
+from sources.models import (
+    FetchAttempt,
+    ParseRun,
+    SourceArtifact,
+    SourceConfiguration,
+)
 from sources.transport import (
     ATTEMPT_SUCCEEDED,
     PROVIDER_SUCCESS_0,
     SingleAttemptResult,
+    warning_request_fingerprint,
     warning_snapshot_request_fingerprint,
 )
 from travel_warnings.contracts import WARNING_SNAPSHOT_SOURCE_CODE
 from travel_warnings.ingestion import (
+    WARNING_SOURCE_CODE,
     TravelWarningIngestionCode,
     TravelWarningIngestionOutcome,
     ingest_travel_warning_snapshot,
@@ -218,6 +226,30 @@ class SnapshotIngestionTests(TransactionTestCase):
         self.assertFalse(SourceArtifact.objects.exists())
         self.assertFalse(TravelWarningRevision.objects.exists())
 
+    def test_v2_run_closes_abandoned_v1_attempt(self):
+        v1_source = SourceConfiguration.objects.get(code=WARNING_SOURCE_CODE)
+        request = warning_request_fingerprint("JP")
+        abandoned = FetchAttempt.objects.create(
+            source=v1_source,
+            source_revision=v1_source.revision,
+            rights_decision=v1_source.rights_decisions.get(),
+            operation_id=uuid.uuid4(),
+            attempt_number=1,
+            request_fingerprint_revision=request.revision,
+            normalized_request_sha256=request.normalized_request_sha256,
+        )
+
+        outcome = self.ingest("TW", snapshot_payload("TW", 0))
+
+        abandoned.refresh_from_db()
+        self.assertEqual(outcome.code, TravelWarningIngestionCode.REVIEW_REQUIRED)
+        self.assertEqual(abandoned.result, FetchAttempt.Result.TERMINAL_FAILED)
+        self.assertEqual(
+            abandoned.failure_code,
+            FetchAttempt.FailureCode.WORKER_INTERRUPTED,
+        )
+        self.assertIsNotNone(abandoned.completed_at)
+
     def test_hongkong_official_identity_persists_one_fact(self):
         outcome = self.ingest("HK", snapshot_payload("HK", 1))
 


