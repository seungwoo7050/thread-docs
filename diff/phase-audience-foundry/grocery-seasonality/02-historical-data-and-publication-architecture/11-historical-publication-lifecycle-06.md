## `fix(history): seal only exact reviewed bundles`

diff --git a/grocery/historical_publication_models.py b/grocery/historical_publication_models.py
index 6fe716a..cf23a32 100644
--- a/grocery/historical_publication_models.py
+++ b/grocery/historical_publication_models.py
@@ -51,12 +51,20 @@ class HistoricalRetailPublicationRevision(models.Model):
     created_at = models.DateTimeField(auto_now_add=True)
     sealed_at = models.DateTimeField(null=True, blank=True)
 
+    _seal_write = False
+
     class Meta:
         constraints = [
             models.UniqueConstraint(
                 fields=("typed_fact_set_sha256", "public_copy_revision"),
                 name="grocery_history_publication_set_copy_uniq",
             ),
+            models.CheckConstraint(
+                condition=~Q(monthly_review=F("regional_review"))
+                & ~Q(monthly_review=F("market_review"))
+                & ~Q(regional_review=F("market_review")),
+                name="grocery_history_publication_reviews_distinct",
+            ),
             models.CheckConstraint(
                 condition=Q(code_manifest_sha256__regex=SHA256_PATTERN)
                 & Q(compatibility_report_sha256__regex=SHA256_PATTERN)
@@ -98,10 +106,41 @@ class HistoricalRetailPublicationRevision(models.Model):
         return f"HISTORICAL_RETAIL:{self.typed_fact_set_sha256}:{self.public_copy_revision}"
 
     def save(self, *args: Any, **kwargs: Any) -> None:
-        if not self._state.adding:
-            raise ValidationError("Historical publication revisions are immutable.")
+        if not self._state.adding or not self._seal_write or self.sealed_at is not None:
+            raise ValidationError("Historical publication revisions use the seal service.")
         self.full_clean()
         super().save(*args, **kwargs)
 
     def delete(self, *args: Any, **kwargs: Any) -> tuple[int, dict[str, int]]:
         raise ValidationError("Historical publication revisions are immutable.")
+
+    def clean(self) -> None:
+        super().clean()
+        reviews = (
+            (self.monthly_review_id, "monthly_review", "MONTHLY"),
+            (self.regional_review_id, "regional_review", "REGIONAL_DAILY"),
+            (self.market_review_id, "market_review", "MARKET_DAILY"),
+        )
+        review_ids = [review_id for review_id, _name, _kind in reviews if review_id]
+        if len(review_ids) != len(set(review_ids)):
+            raise ValidationError("Historical publication reviews must be distinct.")
+        for review_id, attribute, expected_kind in reviews:
+            if not review_id:
+                continue
+            review = getattr(self, attribute)
+            collection = review.collection
+            if (
+                review.decision != HistoricalCollectionReviewDecision.Decision.APPROVE
+                or collection.kind != expected_kind
+                or collection.state != "VALIDATED"
+                or review.approved_result_sha256 != collection.result_sha256
+                or review.approved_partition_manifest_sha256
+                != collection.partition_manifest_sha256
+                or collection.code_manifest_sha256 != self.code_manifest_sha256
+                or HistoricalCollectionReviewDecision.objects.filter(
+                    supersedes_id=review.id
+                ).exists()
+            ):
+                raise ValidationError(
+                    "Historical publication requires current exact approved reviews."
+                )
diff --git a/grocery/historical_publications.py b/grocery/historical_publications.py
index 702cfe0..ba36608 100644
--- a/grocery/historical_publications.py
+++ b/grocery/historical_publications.py
@@ -11,7 +11,7 @@ from datetime import date
 from decimal import Decimal
 
 from django.core.exceptions import ValidationError
-from django.db import transaction
+from django.db import connection, transaction
 from django.utils import timezone
 
 from grocery.historical_collection_models import HistoricalSourceCollection
@@ -21,6 +21,15 @@ from grocery.historical_publication_models import HistoricalRetailPublicationRev
 from grocery.historical_review_models import HistoricalCollectionReviewDecision
 
 
+def _set_historical_seal_token(revision_id: uuid.UUID | None) -> None:
+    token = "" if revision_id is None else str(revision_id)
+    with connection.cursor() as cursor:
+        cursor.execute(
+            "SELECT set_config('grocery.historical_seal_id', %s, true)",
+            [token],
+        )
+
+
 def _month_number(value: str) -> int:
     return int(value[:4]) * 12 + int(value[4:]) - 1
 
@@ -242,10 +251,16 @@ def seal_historical_publication(
         ):
             raise ValidationError("Historical publication replay conflicts with stored evidence.")
         return existing
+    candidate._seal_write = True
+    _set_historical_seal_token(candidate.id)
     candidate.save()
-    if HistoricalRetailPublicationRevision.objects.filter(
-        pk=candidate.id, sealed_at__isnull=True
-    ).update(sealed_at=timezone.now()) != 1:
+    if (
+        HistoricalRetailPublicationRevision.objects.filter(
+            pk=candidate.id, sealed_at__isnull=True
+        ).update(sealed_at=timezone.now())
+        != 1
+    ):
         raise ValidationError("Historical publication seal did not affect one revision.")
+    _set_historical_seal_token(None)
     candidate.refresh_from_db()
     return candidate
diff --git a/grocery/historical_reviews.py b/grocery/historical_reviews.py
index ea0902a..77e2009 100644
--- a/grocery/historical_reviews.py
+++ b/grocery/historical_reviews.py
@@ -75,8 +75,6 @@ def record_historical_review_decision(
     candidate = HistoricalCollectionReviewDecision(id=decision_id, **fields)
     candidate._review_write = True
     _set_historical_review_token(decision_id)
-    try:
-        candidate.save()
-    finally:
-        _set_historical_review_token(None)
+    candidate.save()
+    _set_historical_review_token(None)
     return candidate, True
diff --git a/grocery/migrations/0025_guard_historical_publication_seals.py b/grocery/migrations/0025_guard_historical_publication_seals.py
new file mode 100644
index 0000000..9177241
--- /dev/null
+++ b/grocery/migrations/0025_guard_historical_publication_seals.py
@@ -0,0 +1,122 @@
+from django.db import migrations, models
+
+CREATE_SEAL_GUARDS = r"""
+CREATE OR REPLACE FUNCTION grocery_historical_review_matches(
+    review_key uuid,
+    expected_kind varchar,
+    expected_manifest varchar
+)
+RETURNS boolean AS $$
+    SELECT EXISTS (
+        SELECT 1
+          FROM grocery_historicalcollectionreviewdecision review
+          JOIN grocery_historicalsourcecollection collection
+            ON collection.id = review.collection_id
+         WHERE review.id = review_key
+           AND review.decision = 'APPROVE'
+           AND collection.kind = expected_kind
+           AND collection.state = 'VALIDATED'
+           AND collection.code_manifest_sha256 = expected_manifest
+           AND review.approved_result_sha256 = collection.result_sha256
+           AND review.approved_partition_manifest_sha256
+               = collection.partition_manifest_sha256
+           AND NOT EXISTS (
+               SELECT 1
+                 FROM grocery_historicalcollectionreviewdecision replacement
+                WHERE replacement.supersedes_id = review.id
+           )
+    );
+$$ LANGUAGE sql STABLE;
+
+CREATE OR REPLACE FUNCTION grocery_guard_historical_publication_revision()
+RETURNS trigger AS $$
+DECLARE
+    capability text := current_setting('grocery.historical_seal_id', true);
+BEGIN
+    IF TG_OP = 'DELETE' THEN
+        RAISE EXCEPTION 'historical publication revisions are immutable'
+            USING ERRCODE = '55000';
+    END IF;
+    IF capability IS NULL OR capability IS DISTINCT FROM NEW.id::text THEN
+        RAISE EXCEPTION 'historical publication revision requires seal capability'
+            USING ERRCODE = '42501';
+    END IF;
+    IF NEW.monthly_review_id = NEW.regional_review_id
+       OR NEW.monthly_review_id = NEW.market_review_id
+       OR NEW.regional_review_id = NEW.market_review_id
+       OR NOT grocery_historical_review_matches(
+           NEW.monthly_review_id, 'MONTHLY', NEW.code_manifest_sha256
+       )
+       OR NOT grocery_historical_review_matches(
+           NEW.regional_review_id, 'REGIONAL_DAILY', NEW.code_manifest_sha256
+       )
+       OR NOT grocery_historical_review_matches(
+           NEW.market_review_id, 'MARKET_DAILY', NEW.code_manifest_sha256
+       ) THEN
+        RAISE EXCEPTION 'historical publication reviews are not current exact approvals'
+            USING ERRCODE = '23514';
+    END IF;
+    IF TG_OP = 'INSERT' THEN
+        IF NEW.sealed_at IS NOT NULL THEN
+            RAISE EXCEPTION 'historical publication revisions must start unsealed'
+                USING ERRCODE = '23514';
+        END IF;
+        RETURN NEW;
+    END IF;
+    IF OLD.sealed_at IS NOT NULL OR NEW.sealed_at IS NULL
+       OR NEW.id IS DISTINCT FROM OLD.id
+       OR NEW.monthly_review_id IS DISTINCT FROM OLD.monthly_review_id
+       OR NEW.regional_review_id IS DISTINCT FROM OLD.regional_review_id
+       OR NEW.market_review_id IS DISTINCT FROM OLD.market_review_id
+       OR NEW.code_manifest_sha256 IS DISTINCT FROM OLD.code_manifest_sha256
+       OR NEW.compatibility_report_sha256 IS DISTINCT FROM OLD.compatibility_report_sha256
+       OR NEW.fact_hash_version IS DISTINCT FROM OLD.fact_hash_version
+       OR NEW.typed_fact_set_sha256 IS DISTINCT FROM OLD.typed_fact_set_sha256
+       OR NEW.series_count IS DISTINCT FROM OLD.series_count
+       OR NEW.monthly_fact_count IS DISTINCT FROM OLD.monthly_fact_count
+       OR NEW.regional_fact_count IS DISTINCT FROM OLD.regional_fact_count
+       OR NEW.market_fact_count IS DISTINCT FROM OLD.market_fact_count
+       OR NEW.month_min IS DISTINCT FROM OLD.month_min
+       OR NEW.month_max IS DISTINCT FROM OLD.month_max
+       OR NEW.date_min IS DISTINCT FROM OLD.date_min
+       OR NEW.date_max IS DISTINCT FROM OLD.date_max
+       OR NEW.public_copy_revision IS DISTINCT FROM OLD.public_copy_revision
+       OR NEW.created_at IS DISTINCT FROM OLD.created_at THEN
+        RAISE EXCEPTION 'historical publication only permits a one-time seal'
+            USING ERRCODE = '55000';
+    END IF;
+    RETURN NEW;
+END;
+$$ LANGUAGE plpgsql;
+
+CREATE TRIGGER grocery_history_publication_revision_guard
+BEFORE INSERT OR UPDATE OR DELETE ON grocery_historicalretailpublicationrevision
+FOR EACH ROW EXECUTE FUNCTION grocery_guard_historical_publication_revision();
+"""
+
+
+DROP_SEAL_GUARDS = r"""
+DROP TRIGGER IF EXISTS grocery_history_publication_revision_guard
+    ON grocery_historicalretailpublicationrevision;
+DROP FUNCTION IF EXISTS grocery_guard_historical_publication_revision();
+DROP FUNCTION IF EXISTS grocery_historical_review_matches(uuid, varchar, varchar);
+"""
+
+
+class Migration(migrations.Migration):
+    dependencies = [("grocery", "0024_guard_historical_reviews")]
+
+    operations = [
+        migrations.AddConstraint(
+            model_name="historicalretailpublicationrevision",
+            constraint=models.CheckConstraint(
+                condition=(
+                    ~models.Q(monthly_review=models.F("regional_review"))
+                    & ~models.Q(monthly_review=models.F("market_review"))
+                    & ~models.Q(regional_review=models.F("market_review"))
+                ),
+                name="grocery_history_publication_reviews_distinct",
+            ),
+        ),
+        migrations.RunSQL(CREATE_SEAL_GUARDS, DROP_SEAL_GUARDS),
+    ]
diff --git a/grocery/tests/test_historical_publications.py b/grocery/tests/test_historical_publications.py
index 2d6df88..866269f 100644
--- a/grocery/tests/test_historical_publications.py
+++ b/grocery/tests/test_historical_publications.py
@@ -1,3 +1,9 @@
+import uuid
+
+import pytest
+from django.db import DatabaseError, transaction
+
+from grocery.historical_publication_models import HistoricalRetailPublicationRevision
 from grocery.historical_publications import seal_historical_publication
 from grocery.tests.historical_bundle_factory import create_reviewed_historical_bundle
 
@@ -18,3 +24,30 @@ def test_seal_binds_complete_three_source_fact_set_and_replays(db: None) -> None
     assert revision.typed_fact_set_sha256 == replay.typed_fact_set_sha256
     assert (revision.series_count, revision.monthly_fact_count) == (1, 36)
     assert (revision.regional_fact_count, revision.market_fact_count) == (1, 1)
+
+    with pytest.raises(DatabaseError), transaction.atomic():
+        HistoricalRetailPublicationRevision.objects.filter(pk=revision.pk).update(
+            series_count=2
+        )
+    with pytest.raises(DatabaseError), transaction.atomic():
+        HistoricalRetailPublicationRevision.objects.bulk_create(
+            [
+                HistoricalRetailPublicationRevision(
+                    id=uuid.uuid4(),
+                    monthly_review=bundle.monthly_review,
+                    regional_review=bundle.regional_review,
+                    market_review=bundle.market_review,
+                    code_manifest_sha256=revision.code_manifest_sha256,
+                    compatibility_report_sha256="3" * 64,
+                    typed_fact_set_sha256="4" * 64,
+                    series_count=1,
+                    monthly_fact_count=36,
+                    regional_fact_count=1,
+                    market_fact_count=1,
+                    month_min=revision.month_min,
+                    month_max=revision.month_max,
+                    date_min=revision.date_min,
+                    date_max=revision.date_max,
+                )
+            ]
+        )


