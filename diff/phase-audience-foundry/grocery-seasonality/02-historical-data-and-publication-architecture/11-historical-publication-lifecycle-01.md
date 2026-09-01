# 히스토리 게시본 수명 주기

## `feat(history): bind reviews to collections`

diff --git a/grocery/historical_review_models.py b/grocery/historical_review_models.py
new file mode 100644
index 0000000..0ec820d
--- /dev/null
+++ b/grocery/historical_review_models.py
@@ -0,0 +1,154 @@
+"""Append-only human review decisions for complete historical collections."""
+
+from __future__ import annotations
+
+import uuid
+from typing import Any
+
+from django.conf import settings
+from django.core.exceptions import ValidationError
+from django.core.validators import RegexValidator
+from django.db import models
+from django.db.models import F, Q
+from django.utils import timezone
+
+from grocery.historical_collection_models import HistoricalSourceCollection
+from grocery.models import SHA256_PATTERN, SourceConfiguration, sha256_validator
+
+
+class HistoricalCollectionReviewDecision(models.Model):
+    class Decision(models.TextChoices):
+        APPROVE = "APPROVE", "Approve"
+        REJECT = "REJECT", "Reject"
+
+    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
+    collection = models.ForeignKey(
+        HistoricalSourceCollection,
+        on_delete=models.PROTECT,
+        related_name="review_decisions",
+    )
+    decision = models.CharField(max_length=8, choices=Decision.choices)
+    reviewer = models.ForeignKey(
+        settings.AUTH_USER_MODEL,
+        on_delete=models.PROTECT,
+        related_name="grocery_historical_review_decisions",
+    )
+    decided_at = models.DateTimeField(default=timezone.now)
+    reconciliation_report_sha256 = models.CharField(max_length=64, validators=[sha256_validator])
+    acceptance_evidence_sha256 = models.CharField(max_length=64, validators=[sha256_validator])
+    reason_code = models.CharField(
+        max_length=64,
+        validators=[RegexValidator(r"^[A-Z][A-Z0-9_]*$")],
+    )
+    approved_result_sha256 = models.CharField(
+        max_length=64,
+        blank=True,
+        default="",
+        validators=[sha256_validator],
+    )
+    approved_partition_manifest_sha256 = models.CharField(
+        max_length=64,
+        blank=True,
+        default="",
+        validators=[sha256_validator],
+    )
+    supersedes = models.OneToOneField(
+        "self",
+        on_delete=models.PROTECT,
+        related_name="superseding_decision",
+        null=True,
+        blank=True,
+    )
+
+    class Meta:
+        permissions = [("review_historical_collection", "Can review historical collections")]
+        constraints = [
+            models.CheckConstraint(
+                condition=Q(decision__in=("APPROVE", "REJECT")),
+                name="grocery_history_review_decision_valid",
+            ),
+            models.CheckConstraint(
+                condition=Q(reconciliation_report_sha256__regex=SHA256_PATTERN)
+                & Q(acceptance_evidence_sha256__regex=SHA256_PATTERN),
+                name="grocery_history_review_evidence_valid",
+            ),
+            models.CheckConstraint(
+                condition=Q(reason_code__regex=r"^[A-Z][A-Z0-9_]*$"),
+                name="grocery_history_review_reason_valid",
+            ),
+            models.CheckConstraint(
+                condition=(
+                    Q(
+                        decision="APPROVE",
+                        approved_result_sha256__regex=SHA256_PATTERN,
+                        approved_partition_manifest_sha256__regex=SHA256_PATTERN,
+                    )
+                    | Q(
+                        decision="REJECT",
+                        approved_result_sha256="",
+                        approved_partition_manifest_sha256="",
+                    )
+                ),
+                name="grocery_history_review_approval_valid",
+            ),
+            models.CheckConstraint(
+                condition=Q(supersedes__isnull=True) | ~Q(id=F("supersedes_id")),
+                name="grocery_history_review_not_self",
+            ),
+            models.UniqueConstraint(
+                fields=("collection",),
+                condition=Q(supersedes__isnull=True),
+                name="grocery_history_review_root_uniq",
+            ),
+        ]
+
+    def __str__(self) -> str:
+        return f"{self.collection_id}:{self.decision}:{self.id}"
+
+    def save(self, *args: Any, **kwargs: Any) -> None:
+        if not self._state.adding:
+            raise ValidationError("Historical review decisions are append-only.")
+        self.full_clean()
+        super().save(*args, **kwargs)
+
+    def delete(self, *args: Any, **kwargs: Any) -> tuple[int, dict[str, int]]:
+        raise ValidationError("Historical review decisions are append-only.")
+
+    def clean(self) -> None:
+        super().clean()
+        if not self.collection_id:
+            return
+        collection = self.collection
+        expected_mode = {
+            HistoricalSourceCollection.Kind.MONTHLY: (
+                SourceConfiguration.PublicationMode.HISTORICAL_MONTHLY
+            ),
+            HistoricalSourceCollection.Kind.REGIONAL_DAILY: (
+                SourceConfiguration.PublicationMode.HISTORICAL_REGIONAL
+            ),
+            HistoricalSourceCollection.Kind.MARKET_DAILY: (
+                SourceConfiguration.PublicationMode.HISTORICAL_MARKET
+            ),
+        }[collection.kind]
+        if (
+            collection.source_configuration.state != SourceConfiguration.State.ACTIVE
+            or collection.source_configuration.publication_mode != expected_mode
+        ):
+            raise ValidationError("Historical review requires the matching active source.")
+        if collection.completed_at is None or collection.completed_at > self.decided_at:
+            raise ValidationError("Historical review requires an already completed collection.")
+        if self.supersedes_id:
+            previous = type(self).objects.filter(pk=self.supersedes_id).first()
+            if previous is None or previous.collection_id != collection.id:
+                raise ValidationError("A review can only supersede this collection's current tail.")
+            if type(self).objects.filter(supersedes_id=previous.id).exists():
+                raise ValidationError("Only the current historical review tail can be superseded.")
+        if self.decision == self.Decision.APPROVE:
+            if collection.state != HistoricalSourceCollection.State.VALIDATED:
+                raise ValidationError("Approval requires a validated historical collection.")
+            if (
+                self.approved_result_sha256 != collection.result_sha256
+                or self.approved_partition_manifest_sha256
+                != collection.partition_manifest_sha256
+            ):
+                raise ValidationError("Approved hashes must match the reviewed collection.")
diff --git a/grocery/migrations/0016_historical_collection_reviews.py b/grocery/migrations/0016_historical_collection_reviews.py
new file mode 100644
index 0000000..78ff30d
--- /dev/null
+++ b/grocery/migrations/0016_historical_collection_reviews.py
@@ -0,0 +1,40 @@
+# Generated by Django 5.2.17 on 2026-08-31 04:28
+
+import uuid
+
+import django.core.validators
+import django.db.models.deletion
+import django.utils.timezone
+from django.conf import settings
+from django.db import migrations, models
+
+
+class Migration(migrations.Migration):
+
+    dependencies = [
+        ('grocery', '0015_historical_daily_facts'),
+        migrations.swappable_dependency(settings.AUTH_USER_MODEL),
+    ]
+
+    operations = [
+        migrations.CreateModel(
+            name='HistoricalCollectionReviewDecision',
+            fields=[
+                ('id', models.UUIDField(default=uuid.uuid4, editable=False, primary_key=True, serialize=False)),
+                ('decision', models.CharField(choices=[('APPROVE', 'Approve'), ('REJECT', 'Reject')], max_length=8)),
+                ('decided_at', models.DateTimeField(default=django.utils.timezone.now)),
+                ('reconciliation_report_sha256', models.CharField(max_length=64, validators=[django.core.validators.RegexValidator(message='Enter a lowercase 64-character SHA-256 digest.', regex='^[0-9a-f]{64}$')])),
+                ('acceptance_evidence_sha256', models.CharField(max_length=64, validators=[django.core.validators.RegexValidator(message='Enter a lowercase 64-character SHA-256 digest.', regex='^[0-9a-f]{64}$')])),
+                ('reason_code', models.CharField(max_length=64, validators=[django.core.validators.RegexValidator('^[A-Z][A-Z0-9_]*$')])),
+                ('approved_result_sha256', models.CharField(blank=True, default='', max_length=64, validators=[django.core.validators.RegexValidator(message='Enter a lowercase 64-character SHA-256 digest.', regex='^[0-9a-f]{64}$')])),
+                ('approved_partition_manifest_sha256', models.CharField(blank=True, default='', max_length=64, validators=[django.core.validators.RegexValidator(message='Enter a lowercase 64-character SHA-256 digest.', regex='^[0-9a-f]{64}$')])),
+                ('collection', models.ForeignKey(on_delete=django.db.models.deletion.PROTECT, related_name='review_decisions', to='grocery.historicalsourcecollection')),
+                ('reviewer', models.ForeignKey(on_delete=django.db.models.deletion.PROTECT, related_name='grocery_historical_review_decisions', to=settings.AUTH_USER_MODEL)),
+                ('supersedes', models.OneToOneField(blank=True, null=True, on_delete=django.db.models.deletion.PROTECT, related_name='superseding_decision', to='grocery.historicalcollectionreviewdecision')),
+            ],
+            options={
+                'permissions': [('review_historical_collection', 'Can review historical collections')],
+                'constraints': [models.CheckConstraint(condition=models.Q(('decision__in', ('APPROVE', 'REJECT'))), name='grocery_history_review_decision_valid'), models.CheckConstraint(condition=models.Q(('reconciliation_report_sha256__regex', '^[0-9a-f]{64}$'), ('acceptance_evidence_sha256__regex', '^[0-9a-f]{64}$')), name='grocery_history_review_evidence_valid'), models.CheckConstraint(condition=models.Q(('reason_code__regex', '^[A-Z][A-Z0-9_]*$')), name='grocery_history_review_reason_valid'), models.CheckConstraint(condition=models.Q(models.Q(('approved_partition_manifest_sha256__regex', '^[0-9a-f]{64}$'), ('approved_result_sha256__regex', '^[0-9a-f]{64}$'), ('decision', 'APPROVE')), models.Q(('approved_partition_manifest_sha256', ''), ('approved_result_sha256', ''), ('decision', 'REJECT')), _connector='OR'), name='grocery_history_review_approval_valid'), models.CheckConstraint(condition=models.Q(('supersedes__isnull', True), models.Q(('id', models.F('supersedes_id')), _negated=True), _connector='OR'), name='grocery_history_review_not_self'), models.UniqueConstraint(condition=models.Q(('supersedes__isnull', True)), fields=('collection',), name='grocery_history_review_root_uniq')],
+            },
+        ),
+    ]
diff --git a/grocery/models.py b/grocery/models.py
index fb6ec63..56a2c7e 100644
--- a/grocery/models.py
+++ b/grocery/models.py
@@ -2708,3 +2708,4 @@ from grocery import historical_collection_models as _historical_collection_model
 from grocery import historical_daily_models as _historical_daily_models  # noqa: E402,F401
 from grocery import historical_identity_models as _historical_identity_models  # noqa: E402,F401
 from grocery import historical_monthly_models as _historical_monthly_models  # noqa: E402,F401
+from grocery import historical_review_models as _historical_review_models  # noqa: E402,F401
diff --git a/grocery/tests/test_historical_review_models.py b/grocery/tests/test_historical_review_models.py
new file mode 100644
index 0000000..b177872
--- /dev/null
+++ b/grocery/tests/test_historical_review_models.py
@@ -0,0 +1,47 @@
+import pytest
+from django.contrib.auth import get_user_model
+from django.core.exceptions import ValidationError
+from django.utils import timezone
+
+from grocery.historical_collection_models import HistoricalSourceCollection
+from grocery.historical_review_models import HistoricalCollectionReviewDecision
+from grocery.models import SourceConfiguration
+from grocery.tests.test_acquisition_models import create_source_configuration
+
+
+def test_approval_is_bound_to_the_validated_collection_hashes(db: None) -> None:
+    source = create_source_configuration(
+        dataset_id="15156060",
+        publication_mode=SourceConfiguration.PublicationMode.HISTORICAL_MONTHLY,
+    )
+    collection = HistoricalSourceCollection.objects.create(
+        kind=HistoricalSourceCollection.Kind.MONTHLY,
+        source_configuration=source,
+        state=HistoricalSourceCollection.State.VALIDATED,
+        code_manifest_sha256="a" * 64,
+        partition_manifest_sha256="b" * 64,
+        expected_part_count=1,
+        month_min="202301",
+        month_max="202512",
+        accepted_row_count=1,
+        result_sha256="c" * 64,
+        completed_at=timezone.now(),
+    )
+    actor = get_user_model().objects.create_user(username="historical-reviewer")
+    values = {
+        "collection": collection,
+        "decision": HistoricalCollectionReviewDecision.Decision.APPROVE,
+        "reviewer": actor,
+        "reconciliation_report_sha256": "d" * 64,
+        "acceptance_evidence_sha256": "e" * 64,
+        "reason_code": "RECONCILED",
+        "approved_result_sha256": collection.result_sha256,
+        "approved_partition_manifest_sha256": collection.partition_manifest_sha256,
+    }
+
+    decision = HistoricalCollectionReviewDecision.objects.create(**values)
+    assert decision.collection_id == collection.id
+
+    values["approved_result_sha256"] = "f" * 64
+    with pytest.raises(ValidationError, match="hashes"):
+        HistoricalCollectionReviewDecision.objects.create(**values)


## `feat(history): authorize collection reviews`

diff --git a/grocery/historical_reviews.py b/grocery/historical_reviews.py
new file mode 100644
index 0000000..e8bb0fb
--- /dev/null
+++ b/grocery/historical_reviews.py
@@ -0,0 +1,68 @@
+"""Authorized, idempotent historical collection review recording."""
+
+from __future__ import annotations
+
+import uuid
+from typing import Any
+
+from django.core.exceptions import PermissionDenied, ValidationError
+from django.db import transaction
+
+from grocery.historical_collection_models import HistoricalSourceCollection
+from grocery.historical_review_models import HistoricalCollectionReviewDecision
+
+
+@transaction.atomic
+def record_historical_review_decision(
+    *,
+    decision_id: uuid.UUID,
+    actor: Any,
+    collection_id: uuid.UUID,
+    decision: str,
+    reconciliation_report_sha256: str,
+    acceptance_evidence_sha256: str,
+    reason_code: str,
+    approved_result_sha256: str = "",
+    approved_partition_manifest_sha256: str = "",
+    supersedes_id: uuid.UUID | None = None,
+) -> tuple[HistoricalCollectionReviewDecision, bool]:
+    has_permission = getattr(actor, "has_perm", None)
+    if (
+        getattr(actor, "pk", None) is None
+        or not bool(getattr(actor, "is_authenticated", False))
+        or not bool(getattr(actor, "is_active", False))
+        or not callable(has_permission)
+        or not has_permission("grocery.review_historical_collection")
+    ):
+        raise PermissionDenied("An active historical collection reviewer is required.")
+
+    fields: dict[str, object] = {
+        "collection_id": collection_id,
+        "decision": decision,
+        "reviewer_id": actor.pk,
+        "reconciliation_report_sha256": reconciliation_report_sha256,
+        "acceptance_evidence_sha256": acceptance_evidence_sha256,
+        "reason_code": reason_code,
+        "approved_result_sha256": approved_result_sha256,
+        "approved_partition_manifest_sha256": approved_partition_manifest_sha256,
+        "supersedes_id": supersedes_id,
+    }
+    existing = (
+        HistoricalCollectionReviewDecision.objects.select_for_update()
+        .filter(pk=decision_id)
+        .first()
+    )
+    if existing is not None:
+        if any(getattr(existing, key) != value for key, value in fields.items()):
+            raise ValidationError("Historical review UUID replay conflicts with stored evidence.")
+        return existing, False
+
+    HistoricalSourceCollection.objects.select_for_update().get(pk=collection_id)
+    list(
+        HistoricalCollectionReviewDecision.objects.select_for_update().filter(
+            collection_id=collection_id
+        )
+    )
+    candidate = HistoricalCollectionReviewDecision(id=decision_id, **fields)
+    candidate.save()
+    return candidate, True
diff --git a/grocery/tests/test_historical_reviews.py b/grocery/tests/test_historical_reviews.py
new file mode 100644
index 0000000..960b6c2
--- /dev/null
+++ b/grocery/tests/test_historical_reviews.py
@@ -0,0 +1,49 @@
+import uuid
+
+from django.contrib.auth import get_user_model
+from django.contrib.auth.models import Permission
+from django.utils import timezone
+
+from grocery.historical_collection_models import HistoricalSourceCollection
+from grocery.historical_review_models import HistoricalCollectionReviewDecision
+from grocery.historical_reviews import record_historical_review_decision
+from grocery.models import SourceConfiguration
+from grocery.tests.test_acquisition_models import create_source_configuration
+
+
+def test_authorized_review_uuid_replay_is_idempotent(db: None) -> None:
+    source = create_source_configuration(
+        dataset_id="15156060",
+        publication_mode=SourceConfiguration.PublicationMode.HISTORICAL_MONTHLY,
+    )
+    collection = HistoricalSourceCollection.objects.create(
+        kind=HistoricalSourceCollection.Kind.MONTHLY,
+        source_configuration=source,
+        state=HistoricalSourceCollection.State.VALIDATED,
+        code_manifest_sha256="a" * 64,
+        partition_manifest_sha256="b" * 64,
+        expected_part_count=1,
+        month_min="202301",
+        month_max="202512",
+        accepted_row_count=1,
+        result_sha256="c" * 64,
+        completed_at=timezone.now(),
+    )
+    actor = get_user_model().objects.create_user(username="authorized-history-reviewer")
+    actor.user_permissions.add(Permission.objects.get(codename="review_historical_collection"))
+    values = {
+        "decision_id": uuid.uuid4(),
+        "actor": actor,
+        "collection_id": collection.id,
+        "decision": HistoricalCollectionReviewDecision.Decision.APPROVE,
+        "reconciliation_report_sha256": "d" * 64,
+        "acceptance_evidence_sha256": "e" * 64,
+        "reason_code": "RECONCILED",
+        "approved_result_sha256": collection.result_sha256,
+        "approved_partition_manifest_sha256": collection.partition_manifest_sha256,
+    }
+
+    decision, created = record_historical_review_decision(**values)
+    replay, replay_created = record_historical_review_decision(**values)
+
+    assert created is True and replay_created is False and replay.id == decision.id


