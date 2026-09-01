## `feat(history): persist monthly provider ranges`

diff --git a/grocery/historical_fact_base.py b/grocery/historical_fact_base.py
new file mode 100644
index 0000000..a2cee47
--- /dev/null
+++ b/grocery/historical_fact_base.py
@@ -0,0 +1,48 @@
+"""Shared immutable identity fields for typed historical price facts."""
+
+from __future__ import annotations
+
+import uuid
+from typing import Any
+
+from django.core.exceptions import ValidationError
+from django.db import models
+
+from grocery.historical_collection_models import (
+    HistoricalSourceCollection,
+    HistoricalSourceCollectionPart,
+)
+from grocery.historical_identity_models import HistoricalRetailSeriesKey, RetailRegionKey
+from grocery.models import sha256_validator
+
+
+class HistoricalPriceFact(models.Model):
+    class Currency(models.TextChoices):
+        KRW = "KRW", "South Korean won"
+
+    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
+    collection = models.ForeignKey(HistoricalSourceCollection, on_delete=models.PROTECT)
+    collection_part = models.ForeignKey(HistoricalSourceCollectionPart, on_delete=models.PROTECT)
+    series = models.ForeignKey(HistoricalRetailSeriesKey, on_delete=models.PROTECT)
+    region = models.ForeignKey(RetailRegionKey, on_delete=models.PROTECT)
+    currency = models.CharField(max_length=3, choices=Currency.choices, default=Currency.KRW)
+    source_row_sha256 = models.CharField(max_length=64, validators=[sha256_validator])
+    source_contract_revision = models.CharField(max_length=64)
+    created_at = models.DateTimeField(auto_now_add=True)
+
+    class Meta:
+        abstract = True
+
+    def save(self, *args: Any, **kwargs: Any) -> None:
+        if not self._state.adding:
+            raise ValidationError("Historical price facts are immutable.")
+        self.full_clean()
+        super().save(*args, **kwargs)
+
+    def delete(self, *args: Any, **kwargs: Any) -> tuple[int, dict[str, int]]:
+        raise ValidationError("Historical price facts are immutable.")
+
+    def clean(self) -> None:
+        super().clean()
+        if self.collection_part_id and self.collection_part.collection_id != self.collection_id:
+            raise ValidationError("Historical fact collection and part do not match.")
diff --git a/grocery/historical_monthly_models.py b/grocery/historical_monthly_models.py
new file mode 100644
index 0000000..8ec3062
--- /dev/null
+++ b/grocery/historical_monthly_models.py
@@ -0,0 +1,66 @@
+"""Immutable provider monthly ranges from KAMIS dataset 15156060."""
+
+from __future__ import annotations
+
+from decimal import Decimal
+
+from django.core.exceptions import ValidationError
+from django.core.validators import MinValueValidator, RegexValidator
+from django.db import models
+from django.db.models import F, Q
+
+from grocery.historical_collection_models import HistoricalSourceCollection
+from grocery.historical_fact_base import HistoricalPriceFact
+from grocery.historical_identity_models import YEAR_MONTH_PATTERN
+from grocery.models import SHA256_PATTERN
+
+
+class MonthlyRegionalRetailPrice(HistoricalPriceFact):
+    year_month = models.CharField(max_length=6, validators=[RegexValidator(YEAR_MONTH_PATTERN)])
+    provider_mean = models.DecimalField(
+        max_digits=12,
+        decimal_places=0,
+        validators=[MinValueValidator(Decimal("1"))],
+    )
+    provider_low = models.DecimalField(
+        max_digits=12,
+        decimal_places=0,
+        validators=[MinValueValidator(Decimal("1"))],
+    )
+    provider_high = models.DecimalField(
+        max_digits=12,
+        decimal_places=0,
+        validators=[MinValueValidator(Decimal("1"))],
+    )
+    source_recorded_at = models.DateTimeField(null=True, blank=True)
+
+    class Meta:
+        constraints = [
+            models.UniqueConstraint(
+                fields=("collection", "series", "region", "year_month"),
+                name="grocery_monthly_series_region_month_uniq",
+            ),
+            models.CheckConstraint(
+                condition=Q(year_month__regex=YEAR_MONTH_PATTERN),
+                name="grocery_monthly_year_month_valid",
+            ),
+            models.CheckConstraint(
+                condition=Q(provider_low__gt=0)
+                & Q(provider_mean__gte=F("provider_low"))
+                & Q(provider_high__gte=F("provider_mean")),
+                name="grocery_monthly_price_range_valid",
+            ),
+            models.CheckConstraint(condition=Q(currency="KRW"), name="grocery_monthly_currency"),
+            models.CheckConstraint(
+                condition=Q(source_row_sha256__regex=SHA256_PATTERN),
+                name="grocery_monthly_row_hash_valid",
+            ),
+        ]
+
+    def __str__(self) -> str:
+        return f"{self.series_id}:{self.region_id}:{self.year_month}"
+
+    def clean(self) -> None:
+        super().clean()
+        if self.collection_id and self.collection.kind != HistoricalSourceCollection.Kind.MONTHLY:
+            raise ValidationError("Monthly facts require a monthly collection.")
diff --git a/grocery/migrations/0014_historical_monthly_facts.py b/grocery/migrations/0014_historical_monthly_facts.py
new file mode 100644
index 0000000..6433f18
--- /dev/null
+++ b/grocery/migrations/0014_historical_monthly_facts.py
@@ -0,0 +1,136 @@
+# Generated by Django 5.2.17 on 2026-08-31 04:22
+
+import uuid
+from decimal import Decimal
+
+import django.core.validators
+import django.db.models.deletion
+from django.db import migrations, models
+
+
+class Migration(migrations.Migration):
+    dependencies = [
+        ("grocery", "0013_historical_source_collections"),
+    ]
+
+    operations = [
+        migrations.CreateModel(
+            name="MonthlyRegionalRetailPrice",
+            fields=[
+                (
+                    "id",
+                    models.UUIDField(
+                        default=uuid.uuid4, editable=False, primary_key=True, serialize=False
+                    ),
+                ),
+                (
+                    "currency",
+                    models.CharField(
+                        choices=[("KRW", "South Korean won")], default="KRW", max_length=3
+                    ),
+                ),
+                (
+                    "source_row_sha256",
+                    models.CharField(
+                        max_length=64,
+                        validators=[
+                            django.core.validators.RegexValidator(
+                                message="Enter a lowercase 64-character SHA-256 digest.",
+                                regex="^[0-9a-f]{64}$",
+                            )
+                        ],
+                    ),
+                ),
+                ("source_contract_revision", models.CharField(max_length=64)),
+                ("created_at", models.DateTimeField(auto_now_add=True)),
+                (
+                    "year_month",
+                    models.CharField(
+                        max_length=6,
+                        validators=[
+                            django.core.validators.RegexValidator("^[0-9]{4}(0[1-9]|1[0-2])$")
+                        ],
+                    ),
+                ),
+                (
+                    "provider_mean",
+                    models.DecimalField(
+                        decimal_places=0,
+                        max_digits=12,
+                        validators=[django.core.validators.MinValueValidator(Decimal("1"))],
+                    ),
+                ),
+                (
+                    "provider_low",
+                    models.DecimalField(
+                        decimal_places=0,
+                        max_digits=12,
+                        validators=[django.core.validators.MinValueValidator(Decimal("1"))],
+                    ),
+                ),
+                (
+                    "provider_high",
+                    models.DecimalField(
+                        decimal_places=0,
+                        max_digits=12,
+                        validators=[django.core.validators.MinValueValidator(Decimal("1"))],
+                    ),
+                ),
+                ("source_recorded_at", models.DateTimeField(blank=True, null=True)),
+                (
+                    "collection",
+                    models.ForeignKey(
+                        on_delete=django.db.models.deletion.PROTECT,
+                        to="grocery.historicalsourcecollection",
+                    ),
+                ),
+                (
+                    "collection_part",
+                    models.ForeignKey(
+                        on_delete=django.db.models.deletion.PROTECT,
+                        to="grocery.historicalsourcecollectionpart",
+                    ),
+                ),
+                (
+                    "region",
+                    models.ForeignKey(
+                        on_delete=django.db.models.deletion.PROTECT, to="grocery.retailregionkey"
+                    ),
+                ),
+                (
+                    "series",
+                    models.ForeignKey(
+                        on_delete=django.db.models.deletion.PROTECT,
+                        to="grocery.historicalretailserieskey",
+                    ),
+                ),
+            ],
+            options={
+                "constraints": [
+                    models.UniqueConstraint(
+                        fields=("collection", "series", "region", "year_month"),
+                        name="grocery_monthly_series_region_month_uniq",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(("year_month__regex", "^[0-9]{4}(0[1-9]|1[0-2])$")),
+                        name="grocery_monthly_year_month_valid",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(
+                            ("provider_low__gt", 0),
+                            ("provider_mean__gte", models.F("provider_low")),
+                            ("provider_high__gte", models.F("provider_mean")),
+                        ),
+                        name="grocery_monthly_price_range_valid",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(("currency", "KRW")), name="grocery_monthly_currency"
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(("source_row_sha256__regex", "^[0-9a-f]{64}$")),
+                        name="grocery_monthly_row_hash_valid",
+                    ),
+                ],
+            },
+        ),
+    ]
diff --git a/grocery/models.py b/grocery/models.py
index 3de8b06..5f66379 100644
--- a/grocery/models.py
+++ b/grocery/models.py
@@ -2706,3 +2706,4 @@ def build_source_artifact(attempt_id: uuid.UUID) -> tuple[SourceArtifact, bool]:
 # Load separately bounded vNext models into Django's app registry.
 from grocery import historical_collection_models as _historical_collection_models  # noqa: E402,F401
 from grocery import historical_identity_models as _historical_identity_models  # noqa: E402,F401
+from grocery import historical_monthly_models as _historical_monthly_models  # noqa: E402,F401
diff --git a/grocery/tests/test_historical_monthly_facts.py b/grocery/tests/test_historical_monthly_facts.py
new file mode 100644
index 0000000..f2cb9b5
--- /dev/null
+++ b/grocery/tests/test_historical_monthly_facts.py
@@ -0,0 +1,83 @@
+from decimal import Decimal
+
+import pytest
+from django.core.exceptions import ValidationError
+
+from grocery.historical_collection_models import (
+    HistoricalSourceCollection,
+    HistoricalSourceCollectionPart,
+)
+from grocery.historical_identity_models import (
+    HistoricalRetailSeriesKey,
+    RetailRegionKey,
+    price_series_identity_sha256,
+)
+from grocery.historical_monthly_models import MonthlyRegionalRetailPrice
+from grocery.models import SourceConfiguration
+from grocery.tests.test_acquisition_models import create_source_configuration
+from grocery.tests.test_historical_collections import _validated_parse_run
+from grocery.tests.test_price_series_key_models import create_series
+
+
+def test_monthly_fact_preserves_provider_range_and_is_immutable(db: None) -> None:
+    source = create_source_configuration(
+        dataset_id="15156060",
+        publication_mode=SourceConfiguration.PublicationMode.HISTORICAL_MONTHLY,
+    )
+    collection = HistoricalSourceCollection.objects.create(
+        kind=HistoricalSourceCollection.Kind.MONTHLY,
+        source_configuration=source,
+        code_manifest_sha256="a" * 64,
+        partition_manifest_sha256="b" * 64,
+        expected_part_count=1,
+        month_min="202512",
+        month_max="202512",
+    )
+    part = HistoricalSourceCollectionPart.objects.create(
+        collection=collection,
+        ordinal=1,
+        partition_scope_sha256="c" * 64,
+        parse_run=_validated_parse_run(),
+        fact_count=1,
+    )
+    recent = create_series()
+    series = HistoricalRetailSeriesKey.objects.create(
+        recent_series=recent,
+        series_identity_sha256=price_series_identity_sha256(recent),
+        cross_source_evidence_revision="cross-v1",
+        code_manifest_sha256="a" * 64,
+    )
+    region = RetailRegionKey.objects.create(
+        region_code="1101", region_name="서울", identity_evidence_revision="codes-v1"
+    )
+    fact = MonthlyRegionalRetailPrice.objects.create(
+        collection=collection,
+        collection_part=part,
+        series=series,
+        region=region,
+        year_month="202512",
+        provider_mean=Decimal("1200"),
+        provider_low=Decimal("1000"),
+        provider_high=Decimal("1500"),
+        source_row_sha256="d" * 64,
+        source_contract_revision="15156060-v1",
+    )
+
+    assert fact.provider_mean == Decimal("1200")
+    fact.provider_mean = Decimal("1300")
+    with pytest.raises(ValidationError, match="immutable"):
+        fact.save()
+
+    with pytest.raises(ValidationError):
+        MonthlyRegionalRetailPrice.objects.create(
+            collection=collection,
+            collection_part=part,
+            series=series,
+            region=region,
+            year_month="202511",
+            provider_mean=Decimal("900"),
+            provider_low=Decimal("1000"),
+            provider_high=Decimal("1500"),
+            source_row_sha256="e" * 64,
+            source_contract_revision="15156060-v1",
+        )


