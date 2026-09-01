## `feat(history): guard publication activation state`

diff --git a/grocery/historical_activation_models.py b/grocery/historical_activation_models.py
new file mode 100644
index 0000000..8db3d43
--- /dev/null
+++ b/grocery/historical_activation_models.py
@@ -0,0 +1,141 @@
+"""Independent current pointer and append-only audit events for historical retail."""
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
+from grocery.historical_publication_models import HistoricalRetailPublicationRevision
+from grocery.models import SHA256_PATTERN, sha256_validator
+
+
+class HistoricalRetailPublicationChannel(models.Model):
+    CHANNEL = "HISTORICAL_RETAIL"
+
+    channel = models.CharField(max_length=32, primary_key=True, default=CHANNEL, editable=False)
+    current_revision = models.ForeignKey(
+        HistoricalRetailPublicationRevision,
+        on_delete=models.PROTECT,
+        null=True,
+        blank=True,
+        related_name="current_for_channels",
+    )
+    version = models.PositiveBigIntegerField(default=0)
+    updated_at = models.DateTimeField(default=timezone.now)
+
+    class Meta:
+        permissions = [
+            ("publish_historical_publication", "Can publish historical retail revisions")
+        ]
+        constraints = [
+            models.CheckConstraint(
+                condition=Q(channel="HISTORICAL_RETAIL"),
+                name="grocery_history_channel_fixed",
+            ),
+            models.CheckConstraint(
+                condition=Q(version=0, current_revision__isnull=True) | Q(version__gt=0),
+                name="grocery_history_channel_initial_valid",
+            ),
+        ]
+
+    def __str__(self) -> str:
+        return f"{self.channel}:{self.version}:{self.current_revision_id or 'WITHDRAWN'}"
+
+    def save(self, *args: Any, **kwargs: Any) -> None:
+        if not self._state.adding:
+            raise ValidationError("Historical publication channels change only through events.")
+        self.full_clean()
+        super().save(*args, **kwargs)
+
+    def delete(self, *args: Any, **kwargs: Any) -> tuple[int, dict[str, int]]:
+        raise ValidationError("Historical publication channels cannot be deleted.")
+
+
+class HistoricalRetailPublicationActivation(models.Model):
+    class Operation(models.TextChoices):
+        ACTIVATE = "ACTIVATE", "Activate"
+        ROLLBACK = "ROLLBACK", "Rollback"
+        WITHDRAW = "WITHDRAW", "Withdraw"
+
+    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
+    channel = models.ForeignKey(
+        HistoricalRetailPublicationChannel,
+        on_delete=models.PROTECT,
+        related_name="activations",
+    )
+    operation = models.CharField(max_length=16, choices=Operation.choices)
+    sequence = models.PositiveBigIntegerField()
+    previous_revision = models.ForeignKey(
+        HistoricalRetailPublicationRevision,
+        on_delete=models.PROTECT,
+        null=True,
+        blank=True,
+        related_name="historical_transition_sources",
+    )
+    target_revision = models.ForeignKey(
+        HistoricalRetailPublicationRevision,
+        on_delete=models.PROTECT,
+        null=True,
+        blank=True,
+        related_name="historical_transition_targets",
+    )
+    publisher = models.ForeignKey(
+        settings.AUTH_USER_MODEL,
+        on_delete=models.PROTECT,
+        related_name="grocery_historical_publication_activations",
+    )
+    reason_code = models.CharField(
+        max_length=64,
+        validators=[RegexValidator(r"^[A-Z][A-Z0-9_]*$")],
+    )
+    acceptance_evidence_sha256 = models.CharField(max_length=64, validators=[sha256_validator])
+    created_at = models.DateTimeField(auto_now_add=True)
+
+    class Meta:
+        constraints = [
+            models.UniqueConstraint(
+                fields=("channel", "sequence"),
+                name="grocery_history_activation_sequence_uniq",
+            ),
+            models.CheckConstraint(
+                condition=Q(sequence__gt=0),
+                name="grocery_history_activation_sequence_positive",
+            ),
+            models.CheckConstraint(
+                condition=Q(operation__in=("ACTIVATE", "ROLLBACK", "WITHDRAW")),
+                name="grocery_history_activation_operation_valid",
+            ),
+            models.CheckConstraint(
+                condition=Q(reason_code__regex=r"^[A-Z][A-Z0-9_]*$")
+                & Q(acceptance_evidence_sha256__regex=SHA256_PATTERN),
+                name="grocery_history_activation_evidence_valid",
+            ),
+            models.CheckConstraint(
+                condition=(
+                    Q(operation="WITHDRAW", target_revision__isnull=True)
+                    & Q(previous_revision__isnull=False)
+                    | Q(operation__in=("ACTIVATE", "ROLLBACK"), target_revision__isnull=False)
+                    & ~Q(previous_revision=F("target_revision"))
+                ),
+                name="grocery_history_activation_shape_valid",
+            ),
+        ]
+
+    def __str__(self) -> str:
+        return f"{self.channel_id}:{self.sequence}:{self.operation}"
+
+    def save(self, *args: Any, **kwargs: Any) -> None:
+        if not self._state.adding or not getattr(self, "_transition_write", False):
+            raise ValidationError("Historical publication events use the transition service.")
+        self.full_clean()
+        super().save(*args, **kwargs)
+
+    def delete(self, *args: Any, **kwargs: Any) -> tuple[int, dict[str, int]]:
+        raise ValidationError("Historical publication events are append-only.")
diff --git a/grocery/migrations/0018_historical_publication_activation.py b/grocery/migrations/0018_historical_publication_activation.py
new file mode 100644
index 0000000..9a015f3
--- /dev/null
+++ b/grocery/migrations/0018_historical_publication_activation.py
@@ -0,0 +1,151 @@
+# Generated by Django 5.2.17 on 2026-08-31 04:36
+
+import uuid
+
+import django.core.validators
+import django.db.models.deletion
+import django.utils.timezone
+from django.conf import settings
+from django.db import migrations, models
+
+CREATE_GUARDS = r"""
+CREATE OR REPLACE FUNCTION grocery_guard_historical_activation()
+RETURNS trigger AS $$
+DECLARE
+    channel_record grocery_historicalretailpublicationchannel%ROWTYPE;
+BEGIN
+    IF TG_OP <> 'INSERT' THEN
+        RAISE EXCEPTION 'historical publication events are append-only';
+    END IF;
+    SELECT * INTO channel_record
+      FROM grocery_historicalretailpublicationchannel
+     WHERE channel = NEW.channel_id
+     FOR KEY SHARE;
+    IF NOT FOUND
+       OR NEW.sequence <> channel_record.version + 1
+       OR NEW.previous_revision_id IS DISTINCT FROM channel_record.current_revision_id THEN
+        RAISE EXCEPTION 'historical publication event does not match current state';
+    END IF;
+    IF NEW.target_revision_id IS NOT NULL AND NOT EXISTS (
+        SELECT 1 FROM grocery_historicalretailpublicationrevision
+         WHERE id = NEW.target_revision_id AND sealed_at IS NOT NULL
+    ) THEN
+        RAISE EXCEPTION 'historical publication target is not sealed';
+    END IF;
+    RETURN NEW;
+END;
+$$ LANGUAGE plpgsql;
+
+CREATE OR REPLACE FUNCTION grocery_guard_historical_channel()
+RETURNS trigger AS $$
+BEGIN
+    IF TG_OP = 'DELETE' THEN
+        RAISE EXCEPTION 'historical publication channel cannot be deleted';
+    END IF;
+    IF TG_OP = 'INSERT' THEN
+        IF NEW.channel <> 'HISTORICAL_RETAIL'
+           OR NEW.version <> 0
+           OR NEW.current_revision_id IS NOT NULL THEN
+            RAISE EXCEPTION 'historical publication channel must start withdrawn';
+        END IF;
+        RETURN NEW;
+    END IF;
+    IF NEW.channel IS DISTINCT FROM OLD.channel
+       OR NEW.version <> OLD.version + 1
+       OR NOT EXISTS (
+           SELECT 1 FROM grocery_historicalretailpublicationactivation
+            WHERE channel_id = OLD.channel
+              AND sequence = NEW.version
+              AND previous_revision_id IS NOT DISTINCT FROM OLD.current_revision_id
+              AND target_revision_id IS NOT DISTINCT FROM NEW.current_revision_id
+       ) THEN
+        RAISE EXCEPTION 'historical publication channel requires a matching event';
+    END IF;
+    RETURN NEW;
+END;
+$$ LANGUAGE plpgsql;
+
+CREATE TRIGGER grocery_history_activation_append_only
+BEFORE INSERT OR UPDATE OR DELETE ON grocery_historicalretailpublicationactivation
+FOR EACH ROW EXECUTE FUNCTION grocery_guard_historical_activation();
+
+CREATE TRIGGER grocery_history_channel_transition_only
+BEFORE INSERT OR UPDATE OR DELETE ON grocery_historicalretailpublicationchannel
+FOR EACH ROW EXECUTE FUNCTION grocery_guard_historical_channel();
+"""
+
+DROP_GUARDS = r"""
+DROP TRIGGER IF EXISTS grocery_history_channel_transition_only
+    ON grocery_historicalretailpublicationchannel;
+DROP TRIGGER IF EXISTS grocery_history_activation_append_only
+    ON grocery_historicalretailpublicationactivation;
+DROP FUNCTION IF EXISTS grocery_guard_historical_channel();
+DROP FUNCTION IF EXISTS grocery_guard_historical_activation();
+"""
+
+
+class Migration(migrations.Migration):
+
+    dependencies = [
+        ('grocery', '0017_historical_publication_revision'),
+        migrations.swappable_dependency(settings.AUTH_USER_MODEL),
+    ]
+
+    operations = [
+        migrations.CreateModel(
+            name='HistoricalRetailPublicationChannel',
+            fields=[
+                ('channel', models.CharField(default='HISTORICAL_RETAIL', editable=False, max_length=32, primary_key=True, serialize=False)),
+                ('version', models.PositiveBigIntegerField(default=0)),
+                ('updated_at', models.DateTimeField(default=django.utils.timezone.now)),
+                ('current_revision', models.ForeignKey(blank=True, null=True, on_delete=django.db.models.deletion.PROTECT, related_name='current_for_channels', to='grocery.historicalretailpublicationrevision')),
+            ],
+            options={
+                'permissions': [('publish_historical_publication', 'Can publish historical retail revisions')],
+            },
+        ),
+        migrations.CreateModel(
+            name='HistoricalRetailPublicationActivation',
+            fields=[
+                ('id', models.UUIDField(default=uuid.uuid4, editable=False, primary_key=True, serialize=False)),
+                ('operation', models.CharField(choices=[('ACTIVATE', 'Activate'), ('ROLLBACK', 'Rollback'), ('WITHDRAW', 'Withdraw')], max_length=16)),
+                ('sequence', models.PositiveBigIntegerField()),
+                ('reason_code', models.CharField(max_length=64, validators=[django.core.validators.RegexValidator('^[A-Z][A-Z0-9_]*$')])),
+                ('acceptance_evidence_sha256', models.CharField(max_length=64, validators=[django.core.validators.RegexValidator(message='Enter a lowercase 64-character SHA-256 digest.', regex='^[0-9a-f]{64}$')])),
+                ('created_at', models.DateTimeField(auto_now_add=True)),
+                ('previous_revision', models.ForeignKey(blank=True, null=True, on_delete=django.db.models.deletion.PROTECT, related_name='historical_transition_sources', to='grocery.historicalretailpublicationrevision')),
+                ('publisher', models.ForeignKey(on_delete=django.db.models.deletion.PROTECT, related_name='grocery_historical_publication_activations', to=settings.AUTH_USER_MODEL)),
+                ('target_revision', models.ForeignKey(blank=True, null=True, on_delete=django.db.models.deletion.PROTECT, related_name='historical_transition_targets', to='grocery.historicalretailpublicationrevision')),
+                ('channel', models.ForeignKey(on_delete=django.db.models.deletion.PROTECT, related_name='activations', to='grocery.historicalretailpublicationchannel')),
+            ],
+        ),
+        migrations.AddConstraint(
+            model_name='historicalretailpublicationchannel',
+            constraint=models.CheckConstraint(condition=models.Q(('channel', 'HISTORICAL_RETAIL')), name='grocery_history_channel_fixed'),
+        ),
+        migrations.AddConstraint(
+            model_name='historicalretailpublicationchannel',
+            constraint=models.CheckConstraint(condition=models.Q(models.Q(('current_revision__isnull', True), ('version', 0)), ('version__gt', 0), _connector='OR'), name='grocery_history_channel_initial_valid'),
+        ),
+        migrations.AddConstraint(
+            model_name='historicalretailpublicationactivation',
+            constraint=models.UniqueConstraint(fields=('channel', 'sequence'), name='grocery_history_activation_sequence_uniq'),
+        ),
+        migrations.AddConstraint(
+            model_name='historicalretailpublicationactivation',
+            constraint=models.CheckConstraint(condition=models.Q(('sequence__gt', 0)), name='grocery_history_activation_sequence_positive'),
+        ),
+        migrations.AddConstraint(
+            model_name='historicalretailpublicationactivation',
+            constraint=models.CheckConstraint(condition=models.Q(('operation__in', ('ACTIVATE', 'ROLLBACK', 'WITHDRAW'))), name='grocery_history_activation_operation_valid'),
+        ),
+        migrations.AddConstraint(
+            model_name='historicalretailpublicationactivation',
+            constraint=models.CheckConstraint(condition=models.Q(('reason_code__regex', '^[A-Z][A-Z0-9_]*$'), ('acceptance_evidence_sha256__regex', '^[0-9a-f]{64}$')), name='grocery_history_activation_evidence_valid'),
+        ),
+        migrations.AddConstraint(
+            model_name='historicalretailpublicationactivation',
+            constraint=models.CheckConstraint(condition=models.Q(models.Q(('operation', 'WITHDRAW'), ('target_revision__isnull', True), ('previous_revision__isnull', False)), models.Q(('operation__in', ('ACTIVATE', 'ROLLBACK')), ('target_revision__isnull', False), models.Q(('previous_revision', models.F('target_revision')), _negated=True)), _connector='OR'), name='grocery_history_activation_shape_valid'),
+        ),
+        migrations.RunSQL(CREATE_GUARDS, DROP_GUARDS),
+    ]
diff --git a/grocery/models.py b/grocery/models.py
index b43c2d1..67a3a99 100644
--- a/grocery/models.py
+++ b/grocery/models.py
@@ -2704,6 +2704,9 @@ def build_source_artifact(attempt_id: uuid.UUID) -> tuple[SourceArtifact, bool]:
 
 
 # Load separately bounded vNext models into Django's app registry.
+from grocery import (  # noqa: E402
+    historical_activation_models as _historical_activation_models,  # noqa: F401
+)
 from grocery import historical_collection_models as _historical_collection_models  # noqa: E402,F401
 from grocery import historical_daily_models as _historical_daily_models  # noqa: E402,F401
 from grocery import historical_identity_models as _historical_identity_models  # noqa: E402,F401
diff --git a/grocery/tests/test_historical_activation_models.py b/grocery/tests/test_historical_activation_models.py
new file mode 100644
index 0000000..3d1110b
--- /dev/null
+++ b/grocery/tests/test_historical_activation_models.py
@@ -0,0 +1,29 @@
+import pytest
+from django.core.exceptions import ValidationError
+from django.db import DatabaseError, connection, transaction
+
+from grocery.historical_activation_models import HistoricalRetailPublicationChannel
+
+
+def test_historical_channel_is_fixed_and_rejects_direct_model_transition(db: None) -> None:
+    channel = HistoricalRetailPublicationChannel.objects.create()
+    assert (channel.channel, channel.version, channel.current_revision_id) == (
+        "HISTORICAL_RETAIL",
+        0,
+        None,
+    )
+
+    channel.version = 1
+    with pytest.raises(ValidationError, match="events"):
+        channel.save()
+
+
+def test_database_rejects_a_channel_update_without_matching_event(db: None) -> None:
+    HistoricalRetailPublicationChannel.objects.create()
+
+    with pytest.raises(DatabaseError, match="matching event"), transaction.atomic():
+        with connection.cursor() as cursor:
+            cursor.execute(
+                "UPDATE grocery_historicalretailpublicationchannel "
+                "SET version = 1 WHERE channel = 'HISTORICAL_RETAIL'"
+            )


