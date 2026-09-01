# 교차 소스 정체성과 타입 사실

## `feat(history): identify exact retail sources`

diff --git a/grocery/historical_identity_models.py b/grocery/historical_identity_models.py
new file mode 100644
index 0000000..0019b02
--- /dev/null
+++ b/grocery/historical_identity_models.py
@@ -0,0 +1,172 @@
+"""Reviewed exact identities shared by recent and historical retail sources."""
+
+from __future__ import annotations
+
+import hashlib
+import json
+import uuid
+from typing import Any
+
+from django.core.exceptions import ValidationError
+from django.core.validators import RegexValidator
+from django.db import models
+from django.db.models import Q
+
+from grocery.models import (
+    DIGIT_CODE_PATTERN,
+    SHA256_PATTERN,
+    PriceSeriesKey,
+    sha256_validator,
+)
+
+YEAR_MONTH_PATTERN = r"^[0-9]{4}(0[1-9]|1[0-2])$"
+
+
+def price_series_identity_sha256(series: PriceSeriesKey) -> str:
+    """Hash exact source identity without the recent aggregate coverage."""
+
+    value = {
+        "product_class_code": series.product_class_code,
+        "category_code": series.category_code,
+        "item_code": series.item_code,
+        "variety_code": series.variety_code,
+        "grade_code": series.grade_code,
+        "raw_unit": series.raw_unit,
+        "raw_unit_size": series.raw_unit_size,
+    }
+    canonical = json.dumps(value, sort_keys=True, separators=(",", ":")).encode("utf-8")
+    return hashlib.sha256(canonical).hexdigest()
+
+
+class HistoricalRetailSeriesKey(models.Model):
+    """Reviewed bridge from a recent series to the three historical APIs."""
+
+    recent_series = models.OneToOneField(
+        PriceSeriesKey,
+        on_delete=models.PROTECT,
+        primary_key=True,
+        related_name="historical_identity",
+    )
+    series_identity_sha256 = models.CharField(
+        max_length=64,
+        unique=True,
+        validators=[sha256_validator],
+    )
+    cross_source_evidence_revision = models.CharField(max_length=128)
+    code_manifest_sha256 = models.CharField(max_length=64, validators=[sha256_validator])
+    created_at = models.DateTimeField(auto_now_add=True)
+
+    class Meta:
+        constraints = [
+            models.CheckConstraint(
+                condition=Q(series_identity_sha256__regex=SHA256_PATTERN),
+                name="grocery_history_series_hash_valid",
+            ),
+            models.CheckConstraint(
+                condition=Q(code_manifest_sha256__regex=SHA256_PATTERN),
+                name="grocery_history_series_manifest_valid",
+            ),
+            models.CheckConstraint(
+                condition=~Q(cross_source_evidence_revision=""),
+                name="grocery_history_series_evidence_nonempty",
+            ),
+        ]
+
+    def __str__(self) -> str:
+        return self.series_identity_sha256
+
+    def save(self, *args: Any, **kwargs: Any) -> None:
+        if not self._state.adding:
+            raise ValidationError("Historical retail series keys are immutable.")
+        self.full_clean()
+        super().save(*args, **kwargs)
+
+    def delete(self, *args: Any, **kwargs: Any) -> tuple[int, dict[str, int]]:
+        raise ValidationError("Historical retail series keys are immutable.")
+
+    def clean(self) -> None:
+        super().clean()
+        if self.recent_series_id and self.series_identity_sha256 != price_series_identity_sha256(
+            self.recent_series
+        ):
+            raise ValidationError("Historical series hash does not match its recent exact series.")
+
+
+class RetailRegionKey(models.Model):
+    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
+    region_code = models.CharField(
+        max_length=32,
+        unique=True,
+        validators=[RegexValidator(DIGIT_CODE_PATTERN)],
+    )
+    region_name = models.CharField(max_length=200)
+    identity_evidence_revision = models.CharField(max_length=128)
+    created_at = models.DateTimeField(auto_now_add=True)
+
+    class Meta:
+        constraints = [
+            models.CheckConstraint(
+                condition=Q(region_code__regex=DIGIT_CODE_PATTERN),
+                name="grocery_region_code_valid",
+            ),
+            models.CheckConstraint(
+                condition=~Q(region_name="") & ~Q(identity_evidence_revision=""),
+                name="grocery_region_identity_complete",
+            ),
+        ]
+
+    def __str__(self) -> str:
+        return f"{self.region_code}:{self.region_name}"
+
+    def save(self, *args: Any, **kwargs: Any) -> None:
+        if not self._state.adding:
+            raise ValidationError("Retail region keys are immutable.")
+        self.full_clean()
+        super().save(*args, **kwargs)
+
+    def delete(self, *args: Any, **kwargs: Any) -> tuple[int, dict[str, int]]:
+        raise ValidationError("Retail region keys are immutable.")
+
+
+class RetailMarketKey(models.Model):
+    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
+    region = models.ForeignKey(
+        RetailRegionKey,
+        on_delete=models.PROTECT,
+        related_name="markets",
+    )
+    market_code = models.CharField(
+        max_length=32,
+        validators=[RegexValidator(DIGIT_CODE_PATTERN)],
+    )
+    market_name = models.CharField(max_length=200)
+    identity_evidence_revision = models.CharField(max_length=128)
+    created_at = models.DateTimeField(auto_now_add=True)
+
+    class Meta:
+        constraints = [
+            models.UniqueConstraint(
+                fields=("region", "market_code"),
+                name="grocery_market_region_code_uniq",
+            ),
+            models.CheckConstraint(
+                condition=Q(market_code__regex=DIGIT_CODE_PATTERN),
+                name="grocery_market_code_valid",
+            ),
+            models.CheckConstraint(
+                condition=~Q(market_name="") & ~Q(identity_evidence_revision=""),
+                name="grocery_market_identity_complete",
+            ),
+        ]
+
+    def __str__(self) -> str:
+        return f"{self.region_id}:{self.market_code}:{self.market_name}"
+
+    def save(self, *args: Any, **kwargs: Any) -> None:
+        if not self._state.adding:
+            raise ValidationError("Retail market keys are immutable.")
+        self.full_clean()
+        super().save(*args, **kwargs)
+
+    def delete(self, *args: Any, **kwargs: Any) -> tuple[int, dict[str, int]]:
+        raise ValidationError("Retail market keys are immutable.")
diff --git a/grocery/migrations/0012_historical_source_identities.py b/grocery/migrations/0012_historical_source_identities.py
new file mode 100644
index 0000000..d2b6567
--- /dev/null
+++ b/grocery/migrations/0012_historical_source_identities.py
@@ -0,0 +1,158 @@
+# Generated by Django 5.2.17 on 2026-08-31 04:14
+
+import uuid
+
+import django.core.validators
+import django.db.models.deletion
+from django.db import migrations, models
+
+
+class Migration(migrations.Migration):
+    dependencies = [
+        ("grocery", "0011_historical_source_scope"),
+    ]
+
+    operations = [
+        migrations.CreateModel(
+            name="HistoricalRetailSeriesKey",
+            fields=[
+                (
+                    "recent_series",
+                    models.OneToOneField(
+                        on_delete=django.db.models.deletion.PROTECT,
+                        primary_key=True,
+                        related_name="historical_identity",
+                        serialize=False,
+                        to="grocery.priceserieskey",
+                    ),
+                ),
+                (
+                    "series_identity_sha256",
+                    models.CharField(
+                        max_length=64,
+                        unique=True,
+                        validators=[
+                            django.core.validators.RegexValidator(
+                                message="Enter a lowercase 64-character SHA-256 digest.",
+                                regex="^[0-9a-f]{64}$",
+                            )
+                        ],
+                    ),
+                ),
+                ("cross_source_evidence_revision", models.CharField(max_length=128)),
+                (
+                    "code_manifest_sha256",
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
+                ("created_at", models.DateTimeField(auto_now_add=True)),
+            ],
+            options={
+                "constraints": [
+                    models.CheckConstraint(
+                        condition=models.Q(("series_identity_sha256__regex", "^[0-9a-f]{64}$")),
+                        name="grocery_history_series_hash_valid",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(("code_manifest_sha256__regex", "^[0-9a-f]{64}$")),
+                        name="grocery_history_series_manifest_valid",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(("cross_source_evidence_revision", ""), _negated=True),
+                        name="grocery_history_series_evidence_nonempty",
+                    ),
+                ],
+            },
+        ),
+        migrations.CreateModel(
+            name="RetailRegionKey",
+            fields=[
+                (
+                    "id",
+                    models.UUIDField(
+                        default=uuid.uuid4, editable=False, primary_key=True, serialize=False
+                    ),
+                ),
+                (
+                    "region_code",
+                    models.CharField(
+                        max_length=32,
+                        unique=True,
+                        validators=[django.core.validators.RegexValidator("^[0-9]+$")],
+                    ),
+                ),
+                ("region_name", models.CharField(max_length=200)),
+                ("identity_evidence_revision", models.CharField(max_length=128)),
+                ("created_at", models.DateTimeField(auto_now_add=True)),
+            ],
+            options={
+                "constraints": [
+                    models.CheckConstraint(
+                        condition=models.Q(("region_code__regex", "^[0-9]+$")),
+                        name="grocery_region_code_valid",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(
+                            models.Q(("region_name", ""), _negated=True),
+                            models.Q(("identity_evidence_revision", ""), _negated=True),
+                        ),
+                        name="grocery_region_identity_complete",
+                    ),
+                ],
+            },
+        ),
+        migrations.CreateModel(
+            name="RetailMarketKey",
+            fields=[
+                (
+                    "id",
+                    models.UUIDField(
+                        default=uuid.uuid4, editable=False, primary_key=True, serialize=False
+                    ),
+                ),
+                (
+                    "market_code",
+                    models.CharField(
+                        max_length=32,
+                        validators=[django.core.validators.RegexValidator("^[0-9]+$")],
+                    ),
+                ),
+                ("market_name", models.CharField(max_length=200)),
+                ("identity_evidence_revision", models.CharField(max_length=128)),
+                ("created_at", models.DateTimeField(auto_now_add=True)),
+                (
+                    "region",
+                    models.ForeignKey(
+                        on_delete=django.db.models.deletion.PROTECT,
+                        related_name="markets",
+                        to="grocery.retailregionkey",
+                    ),
+                ),
+            ],
+            options={
+                "constraints": [
+                    models.UniqueConstraint(
+                        fields=("region", "market_code"), name="grocery_market_region_code_uniq"
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(("market_code__regex", "^[0-9]+$")),
+                        name="grocery_market_code_valid",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(
+                            models.Q(("market_name", ""), _negated=True),
+                            models.Q(("identity_evidence_revision", ""), _negated=True),
+                        ),
+                        name="grocery_market_identity_complete",
+                    ),
+                ],
+            },
+        ),
+    ]
diff --git a/grocery/models.py b/grocery/models.py
index f5bfeaa..fece962 100644
--- a/grocery/models.py
+++ b/grocery/models.py
@@ -2701,3 +2701,7 @@ def build_source_artifact(attempt_id: uuid.UUID) -> tuple[SourceArtifact, bool]:
         attempt.artifact = artifact
         attempt.save(update_fields=["artifact"])
     return artifact, created
+
+
+# Load separately bounded vNext models into Django's app registry.
+from grocery import historical_identity_models as _historical_identity_models  # noqa: E402,F401
diff --git a/grocery/tests/test_historical_identities.py b/grocery/tests/test_historical_identities.py
new file mode 100644
index 0000000..5464f19
--- /dev/null
+++ b/grocery/tests/test_historical_identities.py
@@ -0,0 +1,45 @@
+import pytest
+from django.core.exceptions import ValidationError
+
+from grocery.historical_identity_models import (
+    HistoricalRetailSeriesKey,
+    RetailMarketKey,
+    RetailRegionKey,
+    price_series_identity_sha256,
+)
+from grocery.tests.test_price_series_key_models import create_series
+
+
+def test_cross_source_identity_excludes_recent_coverage_but_rejects_drift(db: None) -> None:
+    series = create_series()
+    identity = HistoricalRetailSeriesKey.objects.create(
+        recent_series=series,
+        series_identity_sha256=price_series_identity_sha256(series),
+        cross_source_evidence_revision="cross-source-v1",
+        code_manifest_sha256="a" * 64,
+    )
+
+    assert identity.recent_series_id == series.id
+    identity.series_identity_sha256 = "b" * 64
+    with pytest.raises(ValidationError, match="immutable"):
+        identity.save()
+
+
+def test_region_and_market_preserve_official_leading_zero_codes(db: None) -> None:
+    region = RetailRegionKey.objects.create(
+        region_code="1101",
+        region_name="서울",
+        identity_evidence_revision="codebook-v1",
+    )
+    market = RetailMarketKey.objects.create(
+        region=region,
+        market_code="0110253",
+        market_name="양곡도매",
+        identity_evidence_revision="codebook-v1",
+    )
+
+    assert market.market_code == "0110253"
+    assert market.region_id == region.id
+    market.market_name = "changed"
+    with pytest.raises(ValidationError, match="immutable"):
+        market.save()


