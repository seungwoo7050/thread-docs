## `feat(history): record source collections`

diff --git a/grocery/historical_collection_models.py b/grocery/historical_collection_models.py
new file mode 100644
index 0000000..e404074
--- /dev/null
+++ b/grocery/historical_collection_models.py
@@ -0,0 +1,226 @@
+"""Auditable partition collections for historical KAMIS source generations."""
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
+from django.utils import timezone
+
+from grocery.historical_identity_models import YEAR_MONTH_PATTERN
+from grocery.models import (
+    SHA256_PATTERN,
+    ParseRun,
+    SourceConfiguration,
+    sha256_validator,
+)
+
+
+class HistoricalSourceCollection(models.Model):
+    class Kind(models.TextChoices):
+        MONTHLY = "MONTHLY", "Monthly regional retail"
+        REGIONAL_DAILY = "REGIONAL_DAILY", "Regional daily retail"
+        MARKET_DAILY = "MARKET_DAILY", "Market daily retail"
+
+    class State(models.TextChoices):
+        STARTED = "STARTED", "Started"
+        VALIDATED = "VALIDATED", "Validated"
+        FAILED = "FAILED", "Failed"
+
+    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
+    kind = models.CharField(max_length=24, choices=Kind.choices)
+    source_configuration = models.ForeignKey(
+        SourceConfiguration,
+        on_delete=models.PROTECT,
+        related_name="historical_collections",
+    )
+    state = models.CharField(max_length=16, choices=State.choices, default=State.STARTED)
+    code_manifest_sha256 = models.CharField(max_length=64, validators=[sha256_validator])
+    partition_manifest_sha256 = models.CharField(max_length=64, validators=[sha256_validator])
+    expected_part_count = models.PositiveIntegerField(validators=[MinValueValidator(1)])
+    month_min = models.CharField(
+        max_length=6,
+        blank=True,
+        validators=[RegexValidator(YEAR_MONTH_PATTERN)],
+    )
+    month_max = models.CharField(
+        max_length=6,
+        blank=True,
+        validators=[RegexValidator(YEAR_MONTH_PATTERN)],
+    )
+    date_min = models.DateField(null=True, blank=True)
+    date_max = models.DateField(null=True, blank=True)
+    accepted_row_count = models.PositiveIntegerField(default=0)
+    out_of_scope_row_count = models.PositiveIntegerField(default=0)
+    quarantined_row_count = models.PositiveIntegerField(default=0)
+    result_sha256 = models.CharField(
+        max_length=64,
+        blank=True,
+        default="",
+        validators=[sha256_validator],
+    )
+    failure_code = models.CharField(
+        max_length=64,
+        blank=True,
+        default="",
+        validators=[RegexValidator(r"^[A-Z][A-Z0-9_]*$")],
+    )
+    started_at = models.DateTimeField(default=timezone.now)
+    completed_at = models.DateTimeField(null=True, blank=True)
+
+    class Meta:
+        constraints = [
+            models.CheckConstraint(
+                condition=Q(kind__in=("MONTHLY", "REGIONAL_DAILY", "MARKET_DAILY")),
+                name="grocery_history_collection_kind_valid",
+            ),
+            models.CheckConstraint(
+                condition=Q(state__in=("STARTED", "VALIDATED", "FAILED")),
+                name="grocery_history_collection_state_valid",
+            ),
+            models.CheckConstraint(
+                condition=Q(code_manifest_sha256__regex=SHA256_PATTERN)
+                & Q(partition_manifest_sha256__regex=SHA256_PATTERN),
+                name="grocery_history_collection_hashes_valid",
+            ),
+            models.CheckConstraint(
+                condition=Q(expected_part_count__gt=0),
+                name="grocery_history_collection_parts_positive",
+            ),
+            models.CheckConstraint(
+                condition=(
+                    Q(
+                        kind="MONTHLY",
+                        month_min__regex=YEAR_MONTH_PATTERN,
+                        month_max__regex=YEAR_MONTH_PATTERN,
+                        date_min__isnull=True,
+                        date_max__isnull=True,
+                    )
+                    | (
+                        Q(kind__in=("REGIONAL_DAILY", "MARKET_DAILY"), month_min="", month_max="")
+                        & Q(date_min__isnull=False, date_max__isnull=False)
+                    )
+                ),
+                name="grocery_history_collection_window_valid",
+            ),
+            models.CheckConstraint(
+                condition=(Q(kind="MONTHLY", month_max__gte=F("month_min")) | ~Q(kind="MONTHLY"))
+                & (Q(date_max__gte=F("date_min")) | Q(date_min__isnull=True)),
+                name="grocery_history_collection_window_order",
+            ),
+            models.CheckConstraint(
+                condition=Q(result_sha256="") | Q(result_sha256__regex=SHA256_PATTERN),
+                name="grocery_history_collection_result_valid",
+            ),
+            models.CheckConstraint(
+                condition=(
+                    Q(
+                        state="STARTED",
+                        completed_at__isnull=True,
+                        result_sha256="",
+                        failure_code="",
+                    )
+                    | (
+                        Q(
+                            state="VALIDATED",
+                            completed_at__isnull=False,
+                            failure_code="",
+                            quarantined_row_count=0,
+                        )
+                        & ~Q(result_sha256="")
+                    )
+                    | (
+                        Q(state="FAILED", completed_at__isnull=False, result_sha256="")
+                        & ~Q(failure_code="")
+                    )
+                ),
+                name="grocery_history_collection_outcome_valid",
+            ),
+        ]
+
+    def __str__(self) -> str:
+        return f"{self.kind}:{self.id}:{self.state}"
+
+    def save(self, *args: Any, **kwargs: Any) -> None:
+        if not self._state.adding:
+            persisted = type(self).objects.filter(pk=self.pk).first()
+            if persisted is not None and persisted.state != self.State.STARTED:
+                raise ValidationError("Completed historical collections are immutable.")
+            immutable = (
+                "kind",
+                "source_configuration_id",
+                "code_manifest_sha256",
+                "partition_manifest_sha256",
+                "expected_part_count",
+                "month_min",
+                "month_max",
+                "date_min",
+                "date_max",
+                "started_at",
+            )
+            if persisted is not None and any(
+                getattr(self, field) != getattr(persisted, field) for field in immutable
+            ):
+                raise ValidationError("Historical collection identity is immutable.")
+        self.full_clean()
+        super().save(*args, **kwargs)
+
+
+class HistoricalSourceCollectionPart(models.Model):
+    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
+    collection = models.ForeignKey(
+        HistoricalSourceCollection,
+        on_delete=models.PROTECT,
+        related_name="parts",
+    )
+    ordinal = models.PositiveIntegerField(validators=[MinValueValidator(1)])
+    partition_scope_sha256 = models.CharField(max_length=64, validators=[sha256_validator])
+    parse_run = models.OneToOneField(
+        ParseRun,
+        on_delete=models.PROTECT,
+        related_name="historical_collection_part",
+    )
+    fact_count = models.PositiveIntegerField()
+
+    class Meta:
+        constraints = [
+            models.UniqueConstraint(
+                fields=("collection", "ordinal"),
+                name="grocery_history_part_ordinal_uniq",
+            ),
+            models.UniqueConstraint(
+                fields=("collection", "partition_scope_sha256"),
+                name="grocery_history_part_scope_uniq",
+            ),
+            models.CheckConstraint(
+                condition=Q(ordinal__gt=0),
+                name="grocery_history_part_ordinal_positive",
+            ),
+            models.CheckConstraint(
+                condition=Q(partition_scope_sha256__regex=SHA256_PATTERN),
+                name="grocery_history_part_scope_valid",
+            ),
+        ]
+
+    def __str__(self) -> str:
+        return f"{self.collection_id}:{self.ordinal}"
+
+    def save(self, *args: Any, **kwargs: Any) -> None:
+        if not self._state.adding:
+            raise ValidationError("Historical collection parts are immutable.")
+        self.full_clean()
+        super().save(*args, **kwargs)
+
+    def delete(self, *args: Any, **kwargs: Any) -> tuple[int, dict[str, int]]:
+        raise ValidationError("Historical collection parts are immutable.")
+
+    def clean(self) -> None:
+        super().clean()
+        if self.collection_id and self.collection.state != HistoricalSourceCollection.State.STARTED:
+            raise ValidationError("Parts can only be attached to a started collection.")
+        if self.parse_run_id and self.parse_run.status != ParseRun.Status.VALIDATED:
+            raise ValidationError("Collection parts require a validated parse run.")
diff --git a/grocery/migrations/0013_historical_source_collections.py b/grocery/migrations/0013_historical_source_collections.py
new file mode 100644
index 0000000..4789bcd
--- /dev/null
+++ b/grocery/migrations/0013_historical_source_collections.py
@@ -0,0 +1,322 @@
+# Generated by Django 5.2.17 on 2026-08-31 04:15
+
+import uuid
+
+import django.core.validators
+import django.db.models.deletion
+import django.utils.timezone
+from django.db import migrations, models
+
+
+class Migration(migrations.Migration):
+    dependencies = [
+        ("grocery", "0012_historical_source_identities"),
+    ]
+
+    operations = [
+        migrations.CreateModel(
+            name="HistoricalSourceCollection",
+            fields=[
+                (
+                    "id",
+                    models.UUIDField(
+                        default=uuid.uuid4, editable=False, primary_key=True, serialize=False
+                    ),
+                ),
+                (
+                    "kind",
+                    models.CharField(
+                        choices=[
+                            ("MONTHLY", "Monthly regional retail"),
+                            ("REGIONAL_DAILY", "Regional daily retail"),
+                            ("MARKET_DAILY", "Market daily retail"),
+                        ],
+                        max_length=24,
+                    ),
+                ),
+                (
+                    "state",
+                    models.CharField(
+                        choices=[
+                            ("STARTED", "Started"),
+                            ("VALIDATED", "Validated"),
+                            ("FAILED", "Failed"),
+                        ],
+                        default="STARTED",
+                        max_length=16,
+                    ),
+                ),
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
+                (
+                    "partition_manifest_sha256",
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
+                (
+                    "expected_part_count",
+                    models.PositiveIntegerField(
+                        validators=[django.core.validators.MinValueValidator(1)]
+                    ),
+                ),
+                (
+                    "month_min",
+                    models.CharField(
+                        blank=True,
+                        max_length=6,
+                        validators=[
+                            django.core.validators.RegexValidator("^[0-9]{4}(0[1-9]|1[0-2])$")
+                        ],
+                    ),
+                ),
+                (
+                    "month_max",
+                    models.CharField(
+                        blank=True,
+                        max_length=6,
+                        validators=[
+                            django.core.validators.RegexValidator("^[0-9]{4}(0[1-9]|1[0-2])$")
+                        ],
+                    ),
+                ),
+                ("date_min", models.DateField(blank=True, null=True)),
+                ("date_max", models.DateField(blank=True, null=True)),
+                ("accepted_row_count", models.PositiveIntegerField(default=0)),
+                ("out_of_scope_row_count", models.PositiveIntegerField(default=0)),
+                ("quarantined_row_count", models.PositiveIntegerField(default=0)),
+                (
+                    "result_sha256",
+                    models.CharField(
+                        blank=True,
+                        default="",
+                        max_length=64,
+                        validators=[
+                            django.core.validators.RegexValidator(
+                                message="Enter a lowercase 64-character SHA-256 digest.",
+                                regex="^[0-9a-f]{64}$",
+                            )
+                        ],
+                    ),
+                ),
+                (
+                    "failure_code",
+                    models.CharField(
+                        blank=True,
+                        default="",
+                        max_length=64,
+                        validators=[django.core.validators.RegexValidator("^[A-Z][A-Z0-9_]*$")],
+                    ),
+                ),
+                ("started_at", models.DateTimeField(default=django.utils.timezone.now)),
+                ("completed_at", models.DateTimeField(blank=True, null=True)),
+                (
+                    "source_configuration",
+                    models.ForeignKey(
+                        on_delete=django.db.models.deletion.PROTECT,
+                        related_name="historical_collections",
+                        to="grocery.sourceconfiguration",
+                    ),
+                ),
+            ],
+        ),
+        migrations.CreateModel(
+            name="HistoricalSourceCollectionPart",
+            fields=[
+                (
+                    "id",
+                    models.UUIDField(
+                        default=uuid.uuid4, editable=False, primary_key=True, serialize=False
+                    ),
+                ),
+                (
+                    "ordinal",
+                    models.PositiveIntegerField(
+                        validators=[django.core.validators.MinValueValidator(1)]
+                    ),
+                ),
+                (
+                    "partition_scope_sha256",
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
+                ("fact_count", models.PositiveIntegerField()),
+                (
+                    "collection",
+                    models.ForeignKey(
+                        on_delete=django.db.models.deletion.PROTECT,
+                        related_name="parts",
+                        to="grocery.historicalsourcecollection",
+                    ),
+                ),
+                (
+                    "parse_run",
+                    models.OneToOneField(
+                        on_delete=django.db.models.deletion.PROTECT,
+                        related_name="historical_collection_part",
+                        to="grocery.parserun",
+                    ),
+                ),
+            ],
+        ),
+        migrations.AddConstraint(
+            model_name="historicalsourcecollection",
+            constraint=models.CheckConstraint(
+                condition=models.Q(("kind__in", ("MONTHLY", "REGIONAL_DAILY", "MARKET_DAILY"))),
+                name="grocery_history_collection_kind_valid",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="historicalsourcecollection",
+            constraint=models.CheckConstraint(
+                condition=models.Q(("state__in", ("STARTED", "VALIDATED", "FAILED"))),
+                name="grocery_history_collection_state_valid",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="historicalsourcecollection",
+            constraint=models.CheckConstraint(
+                condition=models.Q(
+                    ("code_manifest_sha256__regex", "^[0-9a-f]{64}$"),
+                    ("partition_manifest_sha256__regex", "^[0-9a-f]{64}$"),
+                ),
+                name="grocery_history_collection_hashes_valid",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="historicalsourcecollection",
+            constraint=models.CheckConstraint(
+                condition=models.Q(("expected_part_count__gt", 0)),
+                name="grocery_history_collection_parts_positive",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="historicalsourcecollection",
+            constraint=models.CheckConstraint(
+                condition=models.Q(
+                    models.Q(
+                        ("date_max__isnull", True),
+                        ("date_min__isnull", True),
+                        ("kind", "MONTHLY"),
+                        ("month_max__regex", "^[0-9]{4}(0[1-9]|1[0-2])$"),
+                        ("month_min__regex", "^[0-9]{4}(0[1-9]|1[0-2])$"),
+                    ),
+                    models.Q(
+                        ("kind__in", ("REGIONAL_DAILY", "MARKET_DAILY")),
+                        ("month_max", ""),
+                        ("month_min", ""),
+                        ("date_max__isnull", False),
+                        ("date_min__isnull", False),
+                    ),
+                    _connector="OR",
+                ),
+                name="grocery_history_collection_window_valid",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="historicalsourcecollection",
+            constraint=models.CheckConstraint(
+                condition=models.Q(
+                    models.Q(
+                        models.Q(("kind", "MONTHLY"), ("month_max__gte", models.F("month_min"))),
+                        models.Q(("kind", "MONTHLY"), _negated=True),
+                        _connector="OR",
+                    ),
+                    models.Q(
+                        ("date_max__gte", models.F("date_min")),
+                        ("date_min__isnull", True),
+                        _connector="OR",
+                    ),
+                ),
+                name="grocery_history_collection_window_order",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="historicalsourcecollection",
+            constraint=models.CheckConstraint(
+                condition=models.Q(
+                    ("result_sha256", ""),
+                    ("result_sha256__regex", "^[0-9a-f]{64}$"),
+                    _connector="OR",
+                ),
+                name="grocery_history_collection_result_valid",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="historicalsourcecollection",
+            constraint=models.CheckConstraint(
+                condition=models.Q(
+                    models.Q(
+                        ("completed_at__isnull", True),
+                        ("failure_code", ""),
+                        ("result_sha256", ""),
+                        ("state", "STARTED"),
+                    ),
+                    models.Q(
+                        ("completed_at__isnull", False),
+                        ("failure_code", ""),
+                        ("quarantined_row_count", 0),
+                        ("state", "VALIDATED"),
+                        models.Q(("result_sha256", ""), _negated=True),
+                    ),
+                    models.Q(
+                        ("completed_at__isnull", False),
+                        ("result_sha256", ""),
+                        ("state", "FAILED"),
+                        models.Q(("failure_code", ""), _negated=True),
+                    ),
+                    _connector="OR",
+                ),
+                name="grocery_history_collection_outcome_valid",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="historicalsourcecollectionpart",
+            constraint=models.UniqueConstraint(
+                fields=("collection", "ordinal"), name="grocery_history_part_ordinal_uniq"
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="historicalsourcecollectionpart",
+            constraint=models.UniqueConstraint(
+                fields=("collection", "partition_scope_sha256"),
+                name="grocery_history_part_scope_uniq",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="historicalsourcecollectionpart",
+            constraint=models.CheckConstraint(
+                condition=models.Q(("ordinal__gt", 0)), name="grocery_history_part_ordinal_positive"
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="historicalsourcecollectionpart",
+            constraint=models.CheckConstraint(
+                condition=models.Q(("partition_scope_sha256__regex", "^[0-9a-f]{64}$")),
+                name="grocery_history_part_scope_valid",
+            ),
+        ),
+    ]
diff --git a/grocery/models.py b/grocery/models.py
index fece962..3de8b06 100644
--- a/grocery/models.py
+++ b/grocery/models.py
@@ -2704,4 +2704,5 @@ def build_source_artifact(attempt_id: uuid.UUID) -> tuple[SourceArtifact, bool]:
 
 
 # Load separately bounded vNext models into Django's app registry.
+from grocery import historical_collection_models as _historical_collection_models  # noqa: E402,F401
 from grocery import historical_identity_models as _historical_identity_models  # noqa: E402,F401
diff --git a/grocery/tests/test_historical_collections.py b/grocery/tests/test_historical_collections.py
new file mode 100644
index 0000000..85c4a57
--- /dev/null
+++ b/grocery/tests/test_historical_collections.py
@@ -0,0 +1,78 @@
+import pytest
+from django.core.exceptions import ValidationError
+from django.utils import timezone
+
+from grocery.historical_collection_models import (
+    HistoricalSourceCollection,
+    HistoricalSourceCollectionPart,
+)
+from grocery.models import ParseRun, SourceConfiguration
+from grocery.tests.test_acquisition_models import create_source_configuration
+from grocery.tests.test_artifact_parse_models import create_artifact
+
+
+def _validated_parse_run() -> ParseRun:
+    completed_at = timezone.now()
+    return ParseRun.objects.create(
+        artifact=create_artifact(),
+        parser_revision="historical-monthly-v1",
+        configuration_hash="c" * 64,
+        result_hash="d" * 64,
+        status=ParseRun.Status.VALIDATED,
+        started_at=completed_at,
+        completed_at=completed_at,
+        total_row_count=1,
+        accepted_row_count=1,
+    )
+
+
+def test_collection_part_is_complete_then_terminally_immutable(db: None) -> None:
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
+        month_min="202301",
+        month_max="202512",
+    )
+    part = HistoricalSourceCollectionPart.objects.create(
+        collection=collection,
+        ordinal=1,
+        partition_scope_sha256="e" * 64,
+        parse_run=_validated_parse_run(),
+        fact_count=1,
+    )
+
+    assert part.collection_id == collection.id
+    collection.state = HistoricalSourceCollection.State.VALIDATED
+    collection.completed_at = timezone.now()
+    collection.accepted_row_count = 1
+    collection.result_sha256 = "f" * 64
+    collection.save()
+
+    collection.accepted_row_count = 2
+    with pytest.raises(ValidationError, match="immutable"):
+        collection.save()
+
+
+def test_collection_window_kind_fails_closed(db: None) -> None:
+    source = create_source_configuration(
+        dataset_id="15156062",
+        publication_mode=SourceConfiguration.PublicationMode.HISTORICAL_REGIONAL,
+    )
+
+    with pytest.raises(ValidationError):
+        HistoricalSourceCollection.objects.create(
+            kind=HistoricalSourceCollection.Kind.REGIONAL_DAILY,
+            source_configuration=source,
+            code_manifest_sha256="a" * 64,
+            partition_manifest_sha256="b" * 64,
+            expected_part_count=1,
+            month_min="202301",
+            month_max="202512",
+        )


