## `feat(history): define publication bundles`

diff --git a/grocery/historical_publication_models.py b/grocery/historical_publication_models.py
new file mode 100644
index 0000000..6fe716a
--- /dev/null
+++ b/grocery/historical_publication_models.py
@@ -0,0 +1,107 @@
+"""Immutable publication bundle over three independently reviewed source collections."""
+
+from __future__ import annotations
+
+import uuid
+from typing import Any
+
+from django.core.exceptions import ValidationError
+from django.core.validators import MinValueValidator, RegexValidator
+from django.db import models
+from django.db.models import F, Q
+
+from grocery.historical_identity_models import YEAR_MONTH_PATTERN
+from grocery.historical_review_models import HistoricalCollectionReviewDecision
+from grocery.models import SHA256_PATTERN, sha256_validator
+
+
+class HistoricalRetailPublicationRevision(models.Model):
+    FACT_HASH_VERSION = "historical-retail-bundle-v1"
+    COPY_REVISION = "ko-v4"
+
+    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
+    monthly_review = models.ForeignKey(
+        HistoricalCollectionReviewDecision,
+        on_delete=models.PROTECT,
+        related_name="monthly_publication_revisions",
+    )
+    regional_review = models.ForeignKey(
+        HistoricalCollectionReviewDecision,
+        on_delete=models.PROTECT,
+        related_name="regional_publication_revisions",
+    )
+    market_review = models.ForeignKey(
+        HistoricalCollectionReviewDecision,
+        on_delete=models.PROTECT,
+        related_name="market_publication_revisions",
+    )
+    code_manifest_sha256 = models.CharField(max_length=64, validators=[sha256_validator])
+    compatibility_report_sha256 = models.CharField(max_length=64, validators=[sha256_validator])
+    fact_hash_version = models.CharField(max_length=64, default=FACT_HASH_VERSION, editable=False)
+    typed_fact_set_sha256 = models.CharField(max_length=64, validators=[sha256_validator])
+    series_count = models.PositiveIntegerField(validators=[MinValueValidator(1)])
+    monthly_fact_count = models.PositiveIntegerField(validators=[MinValueValidator(1)])
+    regional_fact_count = models.PositiveIntegerField(validators=[MinValueValidator(1)])
+    market_fact_count = models.PositiveIntegerField(validators=[MinValueValidator(1)])
+    month_min = models.CharField(max_length=6, validators=[RegexValidator(YEAR_MONTH_PATTERN)])
+    month_max = models.CharField(max_length=6, validators=[RegexValidator(YEAR_MONTH_PATTERN)])
+    date_min = models.DateField()
+    date_max = models.DateField()
+    public_copy_revision = models.CharField(max_length=16, default=COPY_REVISION)
+    created_at = models.DateTimeField(auto_now_add=True)
+    sealed_at = models.DateTimeField(null=True, blank=True)
+
+    class Meta:
+        constraints = [
+            models.UniqueConstraint(
+                fields=("typed_fact_set_sha256", "public_copy_revision"),
+                name="grocery_history_publication_set_copy_uniq",
+            ),
+            models.CheckConstraint(
+                condition=Q(code_manifest_sha256__regex=SHA256_PATTERN)
+                & Q(compatibility_report_sha256__regex=SHA256_PATTERN)
+                & Q(typed_fact_set_sha256__regex=SHA256_PATTERN),
+                name="grocery_history_publication_hashes_valid",
+            ),
+            models.CheckConstraint(
+                condition=Q(fact_hash_version="historical-retail-bundle-v1"),
+                name="grocery_history_publication_hash_version",
+            ),
+            models.CheckConstraint(
+                condition=Q(series_count__gt=0)
+                & Q(monthly_fact_count__gt=0)
+                & Q(regional_fact_count__gt=0)
+                & Q(market_fact_count__gt=0),
+                name="grocery_history_publication_counts_positive",
+            ),
+            models.CheckConstraint(
+                condition=Q(month_min__regex=YEAR_MONTH_PATTERN)
+                & Q(month_max__regex=YEAR_MONTH_PATTERN)
+                & Q(month_max__gte=F("month_min")),
+                name="grocery_history_publication_months_valid",
+            ),
+            models.CheckConstraint(
+                condition=Q(date_max__gte=F("date_min")),
+                name="grocery_history_publication_dates_valid",
+            ),
+            models.CheckConstraint(
+                condition=Q(public_copy_revision="ko-v4"),
+                name="grocery_history_publication_copy_valid",
+            ),
+            models.CheckConstraint(
+                condition=Q(sealed_at__isnull=True) | Q(sealed_at__gte=F("created_at")),
+                name="grocery_history_publication_seal_time",
+            ),
+        ]
+
+    def __str__(self) -> str:
+        return f"HISTORICAL_RETAIL:{self.typed_fact_set_sha256}:{self.public_copy_revision}"
+
+    def save(self, *args: Any, **kwargs: Any) -> None:
+        if not self._state.adding:
+            raise ValidationError("Historical publication revisions are immutable.")
+        self.full_clean()
+        super().save(*args, **kwargs)
+
+    def delete(self, *args: Any, **kwargs: Any) -> tuple[int, dict[str, int]]:
+        raise ValidationError("Historical publication revisions are immutable.")
diff --git a/grocery/migrations/0017_historical_publication_revision.py b/grocery/migrations/0017_historical_publication_revision.py
new file mode 100644
index 0000000..2cbceb3
--- /dev/null
+++ b/grocery/migrations/0017_historical_publication_revision.py
@@ -0,0 +1,44 @@
+# Generated by Django 5.2.17 on 2026-08-31 04:31
+
+import uuid
+
+import django.core.validators
+import django.db.models.deletion
+from django.db import migrations, models
+
+
+class Migration(migrations.Migration):
+
+    dependencies = [
+        ('grocery', '0016_historical_collection_reviews'),
+    ]
+
+    operations = [
+        migrations.CreateModel(
+            name='HistoricalRetailPublicationRevision',
+            fields=[
+                ('id', models.UUIDField(default=uuid.uuid4, editable=False, primary_key=True, serialize=False)),
+                ('code_manifest_sha256', models.CharField(max_length=64, validators=[django.core.validators.RegexValidator(message='Enter a lowercase 64-character SHA-256 digest.', regex='^[0-9a-f]{64}$')])),
+                ('compatibility_report_sha256', models.CharField(max_length=64, validators=[django.core.validators.RegexValidator(message='Enter a lowercase 64-character SHA-256 digest.', regex='^[0-9a-f]{64}$')])),
+                ('fact_hash_version', models.CharField(default='historical-retail-bundle-v1', editable=False, max_length=64)),
+                ('typed_fact_set_sha256', models.CharField(max_length=64, validators=[django.core.validators.RegexValidator(message='Enter a lowercase 64-character SHA-256 digest.', regex='^[0-9a-f]{64}$')])),
+                ('series_count', models.PositiveIntegerField(validators=[django.core.validators.MinValueValidator(1)])),
+                ('monthly_fact_count', models.PositiveIntegerField(validators=[django.core.validators.MinValueValidator(1)])),
+                ('regional_fact_count', models.PositiveIntegerField(validators=[django.core.validators.MinValueValidator(1)])),
+                ('market_fact_count', models.PositiveIntegerField(validators=[django.core.validators.MinValueValidator(1)])),
+                ('month_min', models.CharField(max_length=6, validators=[django.core.validators.RegexValidator('^[0-9]{4}(0[1-9]|1[0-2])$')])),
+                ('month_max', models.CharField(max_length=6, validators=[django.core.validators.RegexValidator('^[0-9]{4}(0[1-9]|1[0-2])$')])),
+                ('date_min', models.DateField()),
+                ('date_max', models.DateField()),
+                ('public_copy_revision', models.CharField(default='ko-v4', max_length=16)),
+                ('created_at', models.DateTimeField(auto_now_add=True)),
+                ('sealed_at', models.DateTimeField(blank=True, null=True)),
+                ('market_review', models.ForeignKey(on_delete=django.db.models.deletion.PROTECT, related_name='market_publication_revisions', to='grocery.historicalcollectionreviewdecision')),
+                ('monthly_review', models.ForeignKey(on_delete=django.db.models.deletion.PROTECT, related_name='monthly_publication_revisions', to='grocery.historicalcollectionreviewdecision')),
+                ('regional_review', models.ForeignKey(on_delete=django.db.models.deletion.PROTECT, related_name='regional_publication_revisions', to='grocery.historicalcollectionreviewdecision')),
+            ],
+            options={
+                'constraints': [models.UniqueConstraint(fields=('typed_fact_set_sha256', 'public_copy_revision'), name='grocery_history_publication_set_copy_uniq'), models.CheckConstraint(condition=models.Q(('code_manifest_sha256__regex', '^[0-9a-f]{64}$'), ('compatibility_report_sha256__regex', '^[0-9a-f]{64}$'), ('typed_fact_set_sha256__regex', '^[0-9a-f]{64}$')), name='grocery_history_publication_hashes_valid'), models.CheckConstraint(condition=models.Q(('fact_hash_version', 'historical-retail-bundle-v1')), name='grocery_history_publication_hash_version'), models.CheckConstraint(condition=models.Q(('series_count__gt', 0), ('monthly_fact_count__gt', 0), ('regional_fact_count__gt', 0), ('market_fact_count__gt', 0)), name='grocery_history_publication_counts_positive'), models.CheckConstraint(condition=models.Q(('month_min__regex', '^[0-9]{4}(0[1-9]|1[0-2])$'), ('month_max__regex', '^[0-9]{4}(0[1-9]|1[0-2])$'), ('month_max__gte', models.F('month_min'))), name='grocery_history_publication_months_valid'), models.CheckConstraint(condition=models.Q(('date_max__gte', models.F('date_min'))), name='grocery_history_publication_dates_valid'), models.CheckConstraint(condition=models.Q(('public_copy_revision', 'ko-v4')), name='grocery_history_publication_copy_valid'), models.CheckConstraint(condition=models.Q(('sealed_at__isnull', True), ('sealed_at__gte', models.F('created_at')), _connector='OR'), name='grocery_history_publication_seal_time')],
+            },
+        ),
+    ]
diff --git a/grocery/models.py b/grocery/models.py
index 56a2c7e..b43c2d1 100644
--- a/grocery/models.py
+++ b/grocery/models.py
@@ -2708,4 +2708,7 @@ from grocery import historical_collection_models as _historical_collection_model
 from grocery import historical_daily_models as _historical_daily_models  # noqa: E402,F401
 from grocery import historical_identity_models as _historical_identity_models  # noqa: E402,F401
 from grocery import historical_monthly_models as _historical_monthly_models  # noqa: E402,F401
+from grocery import (  # noqa: E402
+    historical_publication_models as _historical_publication_models,  # noqa: F401
+)
 from grocery import historical_review_models as _historical_review_models  # noqa: E402,F401
diff --git a/grocery/tests/test_historical_publication_models.py b/grocery/tests/test_historical_publication_models.py
new file mode 100644
index 0000000..b6fff01
--- /dev/null
+++ b/grocery/tests/test_historical_publication_models.py
@@ -0,0 +1,13 @@
+from django.db.models import PROTECT
+
+from grocery.historical_publication_models import HistoricalRetailPublicationRevision
+
+
+def test_historical_revision_owns_three_protected_review_boundaries() -> None:
+    revision = HistoricalRetailPublicationRevision
+
+    assert revision.FACT_HASH_VERSION == "historical-retail-bundle-v1"
+    assert revision.COPY_REVISION == "ko-v4"
+    for field_name in ("monthly_review", "regional_review", "market_review"):
+        field = revision._meta.get_field(field_name)
+        assert field.remote_field.on_delete is PROTECT


