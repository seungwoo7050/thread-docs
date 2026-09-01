## `test(flights): distinguish duration rights revisions`

diff --git a/travel_windows/tests/test_source_publication.py b/travel_windows/tests/test_source_publication.py
index c3b5121..89f2283 100644
--- a/travel_windows/tests/test_source_publication.py
+++ b/travel_windows/tests/test_source_publication.py
@@ -53,6 +53,7 @@ from travel_windows.contracts import (
     DURATION_SCHEMA_FINGERPRINT_SHA256,
     DURATION_SOURCE_CODE,
     DURATION_SOURCE_LOCATOR,
+    DURATION_V1_SOURCE_REVISION,
     SCHEDULE_SOURCE_CODE,
 )
 from travel_windows.models import (
@@ -1220,12 +1221,13 @@ class FlightEvidencePublicationTests(TransactionTestCase):
         source = SourceConfiguration.objects.get(code=DURATION_SOURCE_CODE)
         prior = SourceRightsDecision.objects.get(
             source=source,
+            source_revision=DURATION_V1_SOURCE_REVISION,
             decision_sequence=1,
         )
         reviewed_at = max(timezone.now(), prior.decided_at)
         values = {
             "source": source,
-            "source_revision": source.revision,
+            "source_revision": prior.source_revision,
             "body_sha256": "a" * 64,
             "byte_count": 1,
             "parser_contract_fingerprint_sha256": (
@@ -1256,37 +1258,18 @@ class FlightEvidencePublicationTests(TransactionTestCase):
                 byte_count=2
             )
 
-        SourceRightsDecision.objects.create(
+        current = SourceRightsDecision.objects.get(
             source=source,
-            source_revision=prior.source_revision,
-            decision_sequence=2,
-            decision=SourceRightsDecision.Decision.REJECTED,
-            access_mode=SourceRightsDecision.AccessMode.NO_ACCESS,
-            access_allowed=False,
-            automated_collection_allowed=False,
-            typed_field_storage_allowed=False,
-            raw_body_storage_allowed=False,
-            typed_republication_allowed=False,
-            raw_retention_seconds=0,
-            typed_retention=SourceRightsDecision.Retention.NONE,
-            evidence_retention=(
-                SourceRightsDecision.Retention.PRODUCT_HISTORY
-            ),
-            field_scope_code="",
-            attribution_text="",
-            contract_fingerprint_sha256=(
-                prior.contract_fingerprint_sha256
-            ),
-            decided_by="TEST_DURATION_RIGHTS_REVIEWER",
-            decision_basis_code="TEST_DURATION_RIGHTS_REVOKED_V1",
-            decided_at=timezone.now(),
+            source_revision=source.revision,
+            decision_sequence=1,
         )
+        self.assertNotEqual(current.pk, prior.pk)
         with self.assertRaisesRegex(
             DatabaseError,
             "outside the approved source contract",
         ), transaction.atomic():
             ReviewedDurationReceipt.objects.create(
-                rights_decision=prior,
+                rights_decision=current,
                 **values,
             )
 
