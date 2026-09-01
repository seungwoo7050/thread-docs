## `fix(history): authorize review decisions`

diff --git a/grocery/historical_review_models.py b/grocery/historical_review_models.py
index 0ec820d..d324007 100644
--- a/grocery/historical_review_models.py
+++ b/grocery/historical_review_models.py
@@ -60,6 +60,8 @@ class HistoricalCollectionReviewDecision(models.Model):
         blank=True,
     )
 
+    _review_write = False
+
     class Meta:
         permissions = [("review_historical_collection", "Can review historical collections")]
         constraints = [
@@ -106,8 +108,8 @@ class HistoricalCollectionReviewDecision(models.Model):
         return f"{self.collection_id}:{self.decision}:{self.id}"
 
     def save(self, *args: Any, **kwargs: Any) -> None:
-        if not self._state.adding:
-            raise ValidationError("Historical review decisions are append-only.")
+        if not self._state.adding or not self._review_write:
+            raise ValidationError("Historical review decisions use the review service.")
         self.full_clean()
         super().save(*args, **kwargs)
 
diff --git a/grocery/historical_reviews.py b/grocery/historical_reviews.py
index e8bb0fb..ea0902a 100644
--- a/grocery/historical_reviews.py
+++ b/grocery/historical_reviews.py
@@ -6,12 +6,21 @@ import uuid
 from typing import Any
 
 from django.core.exceptions import PermissionDenied, ValidationError
-from django.db import transaction
+from django.db import connection, transaction
 
 from grocery.historical_collection_models import HistoricalSourceCollection
 from grocery.historical_review_models import HistoricalCollectionReviewDecision
 
 
+def _set_historical_review_token(decision_id: uuid.UUID | None) -> None:
+    token = "" if decision_id is None else str(decision_id)
+    with connection.cursor() as cursor:
+        cursor.execute(
+            "SELECT set_config('grocery.historical_review_id', %s, true)",
+            [token],
+        )
+
+
 @transaction.atomic
 def record_historical_review_decision(
     *,
@@ -64,5 +73,10 @@ def record_historical_review_decision(
         )
     )
     candidate = HistoricalCollectionReviewDecision(id=decision_id, **fields)
-    candidate.save()
+    candidate._review_write = True
+    _set_historical_review_token(decision_id)
+    try:
+        candidate.save()
+    finally:
+        _set_historical_review_token(None)
     return candidate, True
diff --git a/grocery/migrations/0024_guard_historical_reviews.py b/grocery/migrations/0024_guard_historical_reviews.py
new file mode 100644
index 0000000..ea04695
--- /dev/null
+++ b/grocery/migrations/0024_guard_historical_reviews.py
@@ -0,0 +1,99 @@
+from django.db import migrations
+
+CREATE_REVIEW_GUARDS = r"""
+CREATE OR REPLACE FUNCTION grocery_guard_historical_review_insert()
+RETURNS trigger AS $$
+DECLARE
+    capability text := current_setting('grocery.historical_review_id', true);
+    collection_record grocery_historicalsourcecollection%ROWTYPE;
+    source_record grocery_sourceconfiguration%ROWTYPE;
+    expected_mode varchar;
+BEGIN
+    IF capability IS NULL OR capability IS DISTINCT FROM NEW.id::text THEN
+        RAISE EXCEPTION 'historical review insert requires review capability'
+            USING ERRCODE = '42501';
+    END IF;
+    SELECT * INTO collection_record
+      FROM grocery_historicalsourcecollection
+     WHERE id = NEW.collection_id
+     FOR SHARE;
+    IF NOT FOUND THEN
+        RAISE EXCEPTION 'historical review collection is missing'
+            USING ERRCODE = '23514';
+    END IF;
+    SELECT * INTO source_record
+      FROM grocery_sourceconfiguration
+     WHERE id = collection_record.source_configuration_id
+     FOR SHARE;
+    expected_mode := CASE collection_record.kind
+        WHEN 'MONTHLY' THEN 'HISTORICAL_MONTHLY'
+        WHEN 'REGIONAL_DAILY' THEN 'HISTORICAL_REGIONAL'
+        WHEN 'MARKET_DAILY' THEN 'HISTORICAL_MARKET'
+        ELSE ''
+    END;
+    IF source_record.state IS DISTINCT FROM 'ACTIVE'
+       OR source_record.publication_mode IS DISTINCT FROM expected_mode
+       OR collection_record.completed_at IS NULL
+       OR collection_record.completed_at > NEW.decided_at THEN
+        RAISE EXCEPTION 'historical review source and collection are not reviewable'
+            USING ERRCODE = '23514';
+    END IF;
+    IF NEW.supersedes_id IS NOT NULL AND NOT EXISTS (
+        SELECT 1
+          FROM grocery_historicalcollectionreviewdecision previous
+         WHERE previous.id = NEW.supersedes_id
+           AND previous.collection_id = NEW.collection_id
+           AND NOT EXISTS (
+               SELECT 1
+                 FROM grocery_historicalcollectionreviewdecision replacement
+                WHERE replacement.supersedes_id = previous.id
+           )
+    ) THEN
+        RAISE EXCEPTION 'historical review supersedes must be the current tail'
+            USING ERRCODE = '23514';
+    END IF;
+    IF NEW.decision = 'APPROVE' AND (
+        collection_record.state IS DISTINCT FROM 'VALIDATED'
+        OR NEW.approved_result_sha256 IS DISTINCT FROM collection_record.result_sha256
+        OR NEW.approved_partition_manifest_sha256
+           IS DISTINCT FROM collection_record.partition_manifest_sha256
+    ) THEN
+        RAISE EXCEPTION 'historical approval hashes do not match the collection'
+            USING ERRCODE = '23514';
+    END IF;
+    RETURN NEW;
+END;
+$$ LANGUAGE plpgsql;
+
+CREATE OR REPLACE FUNCTION grocery_reject_historical_review_mutation()
+RETURNS trigger AS $$
+BEGIN
+    RAISE EXCEPTION 'historical review decisions are append-only'
+        USING ERRCODE = '55000';
+END;
+$$ LANGUAGE plpgsql;
+
+CREATE TRIGGER grocery_history_review_validate
+BEFORE INSERT ON grocery_historicalcollectionreviewdecision
+FOR EACH ROW EXECUTE FUNCTION grocery_guard_historical_review_insert();
+
+CREATE TRIGGER grocery_history_review_immutable
+BEFORE UPDATE OR DELETE ON grocery_historicalcollectionreviewdecision
+FOR EACH ROW EXECUTE FUNCTION grocery_reject_historical_review_mutation();
+"""
+
+
+DROP_REVIEW_GUARDS = r"""
+DROP TRIGGER IF EXISTS grocery_history_review_immutable
+    ON grocery_historicalcollectionreviewdecision;
+DROP TRIGGER IF EXISTS grocery_history_review_validate
+    ON grocery_historicalcollectionreviewdecision;
+DROP FUNCTION IF EXISTS grocery_reject_historical_review_mutation();
+DROP FUNCTION IF EXISTS grocery_guard_historical_review_insert();
+"""
+
+
+class Migration(migrations.Migration):
+    dependencies = [("grocery", "0023_serialize_historical_collection_writes")]
+
+    operations = [migrations.RunSQL(CREATE_REVIEW_GUARDS, DROP_REVIEW_GUARDS)]
diff --git a/grocery/tests/historical_bundle_factory.py b/grocery/tests/historical_bundle_factory.py
index 54d7fdb..628479e 100644
--- a/grocery/tests/historical_bundle_factory.py
+++ b/grocery/tests/historical_bundle_factory.py
@@ -1,9 +1,11 @@
 import hashlib
+import uuid
 from dataclasses import dataclass
 from datetime import date
 from decimal import Decimal
 
 from django.contrib.auth import get_user_model
+from django.contrib.auth.models import Permission
 from django.utils import timezone
 
 from grocery.historical_collection_models import (
@@ -19,6 +21,7 @@ from grocery.historical_identity_models import (
 )
 from grocery.historical_monthly_models import MonthlyRegionalRetailPrice
 from grocery.historical_review_models import HistoricalCollectionReviewDecision
+from grocery.historical_reviews import record_historical_review_decision
 from grocery.models import ParseRun, SourceConfiguration
 from grocery.tests.historical_test_support import create_scoped_artifact
 from grocery.tests.test_acquisition_models import create_source_configuration
@@ -105,16 +108,18 @@ def _approve(
     collection: HistoricalSourceCollection,
     reviewer: object,
 ) -> HistoricalCollectionReviewDecision:
-    return HistoricalCollectionReviewDecision.objects.create(
-        collection=collection,
+    decision, _created = record_historical_review_decision(
+        decision_id=uuid.uuid4(),
+        actor=reviewer,
+        collection_id=collection.id,
         decision=HistoricalCollectionReviewDecision.Decision.APPROVE,
-        reviewer=reviewer,
         reconciliation_report_sha256="d" * 64,
         acceptance_evidence_sha256="e" * 64,
         reason_code="RECONCILED",
         approved_result_sha256=collection.result_sha256,
         approved_partition_manifest_sha256=collection.partition_manifest_sha256,
     )
+    return decision
 
 
 def create_reviewed_historical_bundle() -> ReviewedHistoricalBundle:
@@ -208,6 +213,9 @@ def create_reviewed_historical_bundle() -> ReviewedHistoricalBundle:
     for part, count in ((monthly_part, 36), (regional_part, 1), (market_part, 1)):
         _complete(part, count)
     reviewer = get_user_model().objects.create_user(username="bundle-reviewer")
+    reviewer.user_permissions.add(
+        Permission.objects.get(codename="review_historical_collection")
+    )
     return ReviewedHistoricalBundle(
         monthly_review=_approve(monthly_part.collection, reviewer),
         regional_review=_approve(regional_part.collection, reviewer),
diff --git a/grocery/tests/test_historical_review_models.py b/grocery/tests/test_historical_review_models.py
index b177872..54ba794 100644
--- a/grocery/tests/test_historical_review_models.py
+++ b/grocery/tests/test_historical_review_models.py
@@ -1,10 +1,14 @@
+import uuid
+
 import pytest
 from django.contrib.auth import get_user_model
+from django.contrib.auth.models import Permission
 from django.core.exceptions import ValidationError
 from django.utils import timezone
 
 from grocery.historical_collection_models import HistoricalSourceCollection
 from grocery.historical_review_models import HistoricalCollectionReviewDecision
+from grocery.historical_reviews import record_historical_review_decision
 from grocery.models import SourceConfiguration
 from grocery.tests.test_acquisition_models import create_source_configuration
 
@@ -28,10 +32,12 @@ def test_approval_is_bound_to_the_validated_collection_hashes(db: None) -> None:
         completed_at=timezone.now(),
     )
     actor = get_user_model().objects.create_user(username="historical-reviewer")
+    actor.user_permissions.add(Permission.objects.get(codename="review_historical_collection"))
     values = {
-        "collection": collection,
+        "decision_id": uuid.uuid4(),
+        "actor": actor,
+        "collection_id": collection.id,
         "decision": HistoricalCollectionReviewDecision.Decision.APPROVE,
-        "reviewer": actor,
         "reconciliation_report_sha256": "d" * 64,
         "acceptance_evidence_sha256": "e" * 64,
         "reason_code": "RECONCILED",
@@ -39,9 +45,11 @@ def test_approval_is_bound_to_the_validated_collection_hashes(db: None) -> None:
         "approved_partition_manifest_sha256": collection.partition_manifest_sha256,
     }
 
-    decision = HistoricalCollectionReviewDecision.objects.create(**values)
+    decision, created = record_historical_review_decision(**values)
+    assert created is True
     assert decision.collection_id == collection.id
 
+    values["decision_id"] = uuid.uuid4()
     values["approved_result_sha256"] = "f" * 64
     with pytest.raises(ValidationError, match="hashes"):
-        HistoricalCollectionReviewDecision.objects.create(**values)
+        record_historical_review_decision(**values)
diff --git a/grocery/tests/test_historical_reviews.py b/grocery/tests/test_historical_reviews.py
index 960b6c2..10f51e6 100644
--- a/grocery/tests/test_historical_reviews.py
+++ b/grocery/tests/test_historical_reviews.py
@@ -1,7 +1,9 @@
 import uuid
 
+import pytest
 from django.contrib.auth import get_user_model
 from django.contrib.auth.models import Permission
+from django.db import DatabaseError, transaction
 from django.utils import timezone
 
 from grocery.historical_collection_models import HistoricalSourceCollection
@@ -47,3 +49,26 @@ def test_authorized_review_uuid_replay_is_idempotent(db: None) -> None:
     replay, replay_created = record_historical_review_decision(**values)
 
     assert created is True and replay_created is False and replay.id == decision.id
+
+    with pytest.raises(DatabaseError), transaction.atomic():
+        HistoricalCollectionReviewDecision.objects.filter(pk=decision.pk).update(
+            reason_code="FORGED"
+        )
+    with pytest.raises(DatabaseError), transaction.atomic():
+        HistoricalCollectionReviewDecision.objects.bulk_create(
+            [
+                HistoricalCollectionReviewDecision(
+                    collection=collection,
+                    decision=HistoricalCollectionReviewDecision.Decision.APPROVE,
+                    reviewer=actor,
+                    reconciliation_report_sha256="d" * 64,
+                    acceptance_evidence_sha256="e" * 64,
+                    reason_code="FORGED",
+                    approved_result_sha256=collection.result_sha256,
+                    approved_partition_manifest_sha256=(
+                        collection.partition_manifest_sha256
+                    ),
+                    supersedes=decision,
+                )
+            ]
+        )


