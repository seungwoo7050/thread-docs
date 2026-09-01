## `fix(history): preserve last-known-good rollback`

diff --git a/grocery/historical_activations.py b/grocery/historical_activations.py
index 3749c42..201ac3e 100644
--- a/grocery/historical_activations.py
+++ b/grocery/historical_activations.py
@@ -119,8 +119,13 @@ def transition_historical_publication(
             .filter(pk=target_revision_id)
             .first()
         )
-        if target is None or not _target_is_currently_approved(target):
-            raise ValidationError("Historical publication target is not a current sealed bundle.")
+        if target is None or target.sealed_at is None:
+            raise ValidationError("Historical publication target is not a sealed bundle.")
+        if (
+            operation == HistoricalRetailPublicationActivation.Operation.ACTIVATE
+            and not _target_is_currently_approved(target)
+        ):
+            raise ValidationError("Historical activation requires current reviewed approvals.")
         if (
             operation == HistoricalRetailPublicationActivation.Operation.ROLLBACK
             and not HistoricalRetailPublicationActivation.objects.filter(
diff --git a/grocery/migrations/0027_allow_historical_last_known_good_rollback.py b/grocery/migrations/0027_allow_historical_last_known_good_rollback.py
new file mode 100644
index 0000000..5e12ebb
--- /dev/null
+++ b/grocery/migrations/0027_allow_historical_last_known_good_rollback.py
@@ -0,0 +1,92 @@
+# ruff: noqa: S608 -- only fixed migration fragments are composed below.
+
+from django.db import migrations
+
+
+def _activation_guard(approval_condition: str) -> str:
+    return rf"""
+CREATE OR REPLACE FUNCTION grocery_guard_historical_activation()
+RETURNS trigger AS $$
+DECLARE
+    capability text := current_setting('grocery.historical_transition_id', true);
+    channel_record grocery_historicalretailpublicationchannel%ROWTYPE;
+BEGIN
+    IF TG_OP <> 'INSERT' THEN
+        RAISE EXCEPTION 'historical publication events are append-only'
+            USING ERRCODE = '55000';
+    END IF;
+    IF capability IS NULL OR capability IS DISTINCT FROM NEW.id::text THEN
+        RAISE EXCEPTION 'historical activation requires transition capability'
+            USING ERRCODE = '42501';
+    END IF;
+    SELECT * INTO channel_record
+      FROM grocery_historicalretailpublicationchannel
+     WHERE channel = NEW.channel_id
+     FOR SHARE;
+    IF NOT FOUND
+       OR NEW.sequence IS DISTINCT FROM channel_record.version + 1
+       OR NEW.previous_revision_id IS DISTINCT FROM channel_record.current_revision_id THEN
+        RAISE EXCEPTION 'historical activation does not match current state'
+            USING ERRCODE = '23514';
+    END IF;
+    IF NEW.operation = 'WITHDRAW' THEN
+        IF NEW.target_revision_id IS NOT NULL OR NEW.previous_revision_id IS NULL THEN
+            RAISE EXCEPTION 'historical withdrawal shape is invalid'
+                USING ERRCODE = '23514';
+        END IF;
+        RETURN NEW;
+    END IF;
+    IF NEW.operation NOT IN ('ACTIVATE', 'ROLLBACK')
+       OR NEW.target_revision_id IS NULL
+       OR NEW.target_revision_id IS NOT DISTINCT FROM NEW.previous_revision_id
+       OR NOT EXISTS (
+           SELECT 1
+             FROM grocery_historicalretailpublicationrevision revision
+            WHERE revision.id = NEW.target_revision_id
+              AND revision.sealed_at IS NOT NULL
+              AND ({approval_condition})
+       ) THEN
+        RAISE EXCEPTION 'historical activation target is not eligible and sealed'
+            USING ERRCODE = '23514';
+    END IF;
+    IF NEW.operation = 'ROLLBACK' AND NOT EXISTS (
+        SELECT 1
+          FROM grocery_historicalretailpublicationactivation prior
+         WHERE prior.channel_id = NEW.channel_id
+           AND prior.sequence < NEW.sequence
+           AND prior.operation IN ('ACTIVATE', 'ROLLBACK')
+           AND prior.target_revision_id = NEW.target_revision_id
+    ) THEN
+        RAISE EXCEPTION 'historical rollback target was not previously current'
+            USING ERRCODE = '23514';
+    END IF;
+    RETURN NEW;
+END;
+$$ LANGUAGE plpgsql;
+"""
+
+
+_CURRENT_REVIEWS = r"""
+grocery_historical_review_matches(
+    revision.monthly_review_id, 'MONTHLY', revision.code_manifest_sha256
+)
+AND grocery_historical_review_matches(
+    revision.regional_review_id, 'REGIONAL_DAILY', revision.code_manifest_sha256
+)
+AND grocery_historical_review_matches(
+    revision.market_review_id, 'MARKET_DAILY', revision.code_manifest_sha256
+)
+""".strip()
+
+ALLOW_LAST_KNOWN_GOOD = _activation_guard(
+    f"NEW.operation = 'ROLLBACK' OR ({_CURRENT_REVIEWS})"
+)
+RESTORE_CURRENT_REVIEW_REQUIREMENT = _activation_guard(_CURRENT_REVIEWS)
+
+
+class Migration(migrations.Migration):
+    dependencies = [("grocery", "0026_guard_historical_activation_cas")]
+
+    operations = [
+        migrations.RunSQL(ALLOW_LAST_KNOWN_GOOD, RESTORE_CURRENT_REVIEW_REQUIREMENT)
+    ]
diff --git a/grocery/tests/test_historical_activation_models.py b/grocery/tests/test_historical_activation_models.py
index 77f64c9..edb65b2 100644
--- a/grocery/tests/test_historical_activation_models.py
+++ b/grocery/tests/test_historical_activation_models.py
@@ -15,6 +15,8 @@ from grocery.historical_activations import (
     transition_historical_publication,
 )
 from grocery.historical_publications import seal_historical_publication
+from grocery.historical_review_models import HistoricalCollectionReviewDecision
+from grocery.historical_reviews import record_historical_review_decision
 from grocery.tests.historical_bundle_factory import create_reviewed_historical_bundle
 
 
@@ -80,6 +82,54 @@ def test_historical_activation_is_authorized_idempotent_cas(transactional_db: No
         orphan.save()
     assert HistoricalRetailPublicationActivation.objects.count() == 1
 
+    transition_historical_publication(
+        operation_id=uuid.uuid4(),
+        actor=publisher,
+        operation=HistoricalRetailPublicationActivation.Operation.WITHDRAW,
+        target_revision_id=None,
+        expected_current_revision_id=revision.id,
+        expected_version=1,
+        reason_code="WITHDRAW_FOR_REVIEW_UPDATE",
+        acceptance_evidence_sha256="5" * 64,
+    )
+    monthly_review = bundle.monthly_review
+    record_historical_review_decision(
+        decision_id=uuid.uuid4(),
+        actor=monthly_review.reviewer,
+        collection_id=monthly_review.collection_id,
+        decision=HistoricalCollectionReviewDecision.Decision.APPROVE,
+        reconciliation_report_sha256="6" * 64,
+        acceptance_evidence_sha256="7" * 64,
+        reason_code="REVIEW_REFRESHED",
+        approved_result_sha256=monthly_review.approved_result_sha256,
+        approved_partition_manifest_sha256=(
+            monthly_review.approved_partition_manifest_sha256
+        ),
+        supersedes_id=monthly_review.id,
+    )
+    with pytest.raises(ValidationError, match="current reviewed"):
+        transition_historical_publication(
+            operation_id=uuid.uuid4(),
+            actor=publisher,
+            operation=HistoricalRetailPublicationActivation.Operation.ACTIVATE,
+            target_revision_id=revision.id,
+            expected_current_revision_id=None,
+            expected_version=2,
+            reason_code="REACTIVATE_SUPERSEDED",
+            acceptance_evidence_sha256="8" * 64,
+        )
+    rolled_back, rolled_back_created = transition_historical_publication(
+        operation_id=uuid.uuid4(),
+        actor=publisher,
+        operation=HistoricalRetailPublicationActivation.Operation.ROLLBACK,
+        target_revision_id=revision.id,
+        expected_current_revision_id=None,
+        expected_version=2,
+        reason_code="LAST_KNOWN_GOOD_ROLLBACK",
+        acceptance_evidence_sha256="9" * 64,
+    )
+    assert rolled_back_created is True and rolled_back.sequence == 3
+
     outsider = get_user_model().objects.create_user(username="historical-outsider")
     with pytest.raises(PermissionDenied):
         transition_historical_publication(
@@ -88,12 +138,12 @@ def test_historical_activation_is_authorized_idempotent_cas(transactional_db: No
             operation=HistoricalRetailPublicationActivation.Operation.WITHDRAW,
             target_revision_id=None,
             expected_current_revision_id=revision.id,
-            expected_version=1,
+            expected_version=3,
             reason_code="UNAUTHORIZED",
             acceptance_evidence_sha256="5" * 64,
         )
 
     with pytest.raises(DatabaseError), transaction.atomic():
         HistoricalRetailPublicationChannel.objects.filter(pk=channel.pk).update(
-            version=2,
+            version=4,
         )
