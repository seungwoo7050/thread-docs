## `fix(history): activate publications through CAS`

diff --git a/grocery/historical_activation_models.py b/grocery/historical_activation_models.py
index 8db3d43..b7cd440 100644
--- a/grocery/historical_activation_models.py
+++ b/grocery/historical_activation_models.py
@@ -49,10 +49,7 @@ class HistoricalRetailPublicationChannel(models.Model):
         return f"{self.channel}:{self.version}:{self.current_revision_id or 'WITHDRAWN'}"
 
     def save(self, *args: Any, **kwargs: Any) -> None:
-        if not self._state.adding:
-            raise ValidationError("Historical publication channels change only through events.")
-        self.full_clean()
-        super().save(*args, **kwargs)
+        raise ValidationError("Historical publication channels use the transition service.")
 
     def delete(self, *args: Any, **kwargs: Any) -> tuple[int, dict[str, int]]:
         raise ValidationError("Historical publication channels cannot be deleted.")
diff --git a/grocery/historical_activations.py b/grocery/historical_activations.py
new file mode 100644
index 0000000..3749c42
--- /dev/null
+++ b/grocery/historical_activations.py
@@ -0,0 +1,165 @@
+"""Authorized CAS transitions for the independent historical publication channel."""
+
+from __future__ import annotations
+
+import uuid
+from typing import Any
+
+from django.core.exceptions import PermissionDenied, ValidationError
+from django.db import connection, transaction
+from django.utils import timezone
+
+from grocery.historical_activation_models import (
+    HistoricalRetailPublicationActivation,
+    HistoricalRetailPublicationChannel,
+)
+from grocery.historical_publication_models import HistoricalRetailPublicationRevision
+from grocery.historical_review_models import HistoricalCollectionReviewDecision
+
+
+def _set_historical_transition_token(operation_id: uuid.UUID | None) -> None:
+    token = "" if operation_id is None else str(operation_id)
+    with connection.cursor() as cursor:
+        cursor.execute(
+            "SELECT set_config('grocery.historical_transition_id', %s, true)",
+            [token],
+        )
+
+
+def _bootstrap_historical_channel(operation_id: uuid.UUID) -> None:
+    _set_historical_transition_token(operation_id)
+    HistoricalRetailPublicationChannel.objects.bulk_create(
+        [HistoricalRetailPublicationChannel()],
+        ignore_conflicts=True,
+    )
+
+
+def _target_is_currently_approved(revision: HistoricalRetailPublicationRevision) -> bool:
+    review_ids = (
+        revision.monthly_review_id,
+        revision.regional_review_id,
+        revision.market_review_id,
+    )
+    return revision.sealed_at is not None and not HistoricalCollectionReviewDecision.objects.filter(
+        supersedes_id__in=review_ids
+    ).exists()
+
+
+@transaction.atomic
+def transition_historical_publication(
+    *,
+    operation_id: uuid.UUID,
+    actor: Any,
+    operation: str,
+    target_revision_id: uuid.UUID | None,
+    expected_current_revision_id: uuid.UUID | None,
+    expected_version: int,
+    reason_code: str,
+    acceptance_evidence_sha256: str,
+) -> tuple[HistoricalRetailPublicationActivation, bool]:
+    has_permission = getattr(actor, "has_perm", None)
+    if (
+        getattr(actor, "pk", None) is None
+        or not bool(getattr(actor, "is_authenticated", False))
+        or not bool(getattr(actor, "is_active", False))
+        or not callable(has_permission)
+        or not has_permission("grocery.publish_historical_publication")
+    ):
+        raise PermissionDenied("An active historical publication publisher is required.")
+    if type(operation_id) is not uuid.UUID:
+        raise ValidationError("Historical publication operation ID must be a UUID.")
+    if type(expected_version) is not int or expected_version < 0:
+        raise ValidationError("Expected historical publication version must be non-negative.")
+
+    _bootstrap_historical_channel(operation_id)
+    channel = HistoricalRetailPublicationChannel.objects.select_for_update().get(
+        pk=HistoricalRetailPublicationChannel.CHANNEL
+    )
+    semantic_fields: dict[str, object] = {
+        "channel_id": HistoricalRetailPublicationChannel.CHANNEL,
+        "operation": operation,
+        "sequence": expected_version + 1,
+        "previous_revision_id": expected_current_revision_id,
+        "target_revision_id": target_revision_id,
+        "publisher_id": actor.pk,
+        "reason_code": reason_code,
+        "acceptance_evidence_sha256": acceptance_evidence_sha256,
+    }
+    existing = (
+        HistoricalRetailPublicationActivation.objects.select_for_update()
+        .filter(pk=operation_id)
+        .first()
+    )
+    if existing is not None:
+        if any(getattr(existing, name) != value for name, value in semantic_fields.items()):
+            raise ValidationError(
+                "Historical publication operation UUID conflicts with stored evidence."
+            )
+        _set_historical_transition_token(None)
+        return existing, False
+    if (
+        channel.version != expected_version
+        or channel.current_revision_id != expected_current_revision_id
+    ):
+        raise ValidationError("Historical publication expectation is stale.")
+    if operation not in HistoricalRetailPublicationActivation.Operation.values:
+        raise ValidationError("Historical publication operation is invalid.")
+
+    target: HistoricalRetailPublicationRevision | None
+    if operation == HistoricalRetailPublicationActivation.Operation.WITHDRAW:
+        if target_revision_id is not None or channel.current_revision_id is None:
+            raise ValidationError("Withdrawal requires a current historical revision.")
+        target = None
+    else:
+        if target_revision_id is None or target_revision_id == channel.current_revision_id:
+            raise ValidationError("Historical publication requires a different target.")
+        target = (
+            HistoricalRetailPublicationRevision.objects.select_for_update()
+            .select_related("monthly_review", "regional_review", "market_review")
+            .filter(pk=target_revision_id)
+            .first()
+        )
+        if target is None or not _target_is_currently_approved(target):
+            raise ValidationError("Historical publication target is not a current sealed bundle.")
+        if (
+            operation == HistoricalRetailPublicationActivation.Operation.ROLLBACK
+            and not HistoricalRetailPublicationActivation.objects.filter(
+                channel=channel,
+                sequence__lte=expected_version,
+                operation__in=(
+                    HistoricalRetailPublicationActivation.Operation.ACTIVATE,
+                    HistoricalRetailPublicationActivation.Operation.ROLLBACK,
+                ),
+                target_revision_id=target.id,
+            ).exists()
+        ):
+            raise ValidationError("Historical rollback target was not previously current.")
+
+    activation = HistoricalRetailPublicationActivation(
+        id=operation_id,
+        channel=channel,
+        operation=operation,
+        sequence=expected_version + 1,
+        previous_revision_id=expected_current_revision_id,
+        target_revision=target,
+        publisher_id=actor.pk,
+        reason_code=reason_code,
+        acceptance_evidence_sha256=acceptance_evidence_sha256,
+    )
+    activation._transition_write = True
+    _set_historical_transition_token(operation_id)
+    activation.save()
+    updated = HistoricalRetailPublicationChannel.objects.filter(
+        pk=HistoricalRetailPublicationChannel.CHANNEL,
+        current_revision_id=expected_current_revision_id,
+        version=expected_version,
+    ).update(
+        current_revision_id=target_revision_id,
+        version=expected_version + 1,
+        updated_at=timezone.now(),
+    )
+    if updated != 1:
+        raise ValidationError("Historical publication pointer did not advance exactly once.")
+    _set_historical_transition_token(None)
+    activation.refresh_from_db()
+    return activation, True
diff --git a/grocery/migrations/0026_guard_historical_activation_cas.py b/grocery/migrations/0026_guard_historical_activation_cas.py
new file mode 100644
index 0000000..dc45825
--- /dev/null
+++ b/grocery/migrations/0026_guard_historical_activation_cas.py
@@ -0,0 +1,273 @@
+from django.db import migrations
+
+UPGRADE_GUARDS = r"""
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
+              AND grocery_historical_review_matches(
+                  revision.monthly_review_id, 'MONTHLY', revision.code_manifest_sha256
+              )
+              AND grocery_historical_review_matches(
+                  revision.regional_review_id,
+                  'REGIONAL_DAILY',
+                  revision.code_manifest_sha256
+              )
+              AND grocery_historical_review_matches(
+                  revision.market_review_id,
+                  'MARKET_DAILY',
+                  revision.code_manifest_sha256
+              )
+       ) THEN
+        RAISE EXCEPTION 'historical activation target is not current and sealed'
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
+
+CREATE OR REPLACE FUNCTION grocery_guard_historical_channel()
+RETURNS trigger AS $$
+DECLARE
+    capability text := current_setting('grocery.historical_transition_id', true);
+BEGIN
+    IF TG_OP = 'DELETE' THEN
+        RAISE EXCEPTION 'historical publication channel cannot be deleted'
+            USING ERRCODE = '55000';
+    END IF;
+    IF capability IS NULL
+       OR capability !~ '^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$' THEN
+        RAISE EXCEPTION 'historical channel requires transition capability'
+            USING ERRCODE = '42501';
+    END IF;
+    IF TG_OP = 'INSERT' THEN
+        IF NEW.channel IS DISTINCT FROM 'HISTORICAL_RETAIL'
+           OR NEW.version IS DISTINCT FROM 0::bigint
+           OR NEW.current_revision_id IS NOT NULL THEN
+            RAISE EXCEPTION 'historical channel bootstrap is invalid'
+                USING ERRCODE = '23514';
+        END IF;
+        RETURN NEW;
+    END IF;
+    IF NEW.channel IS DISTINCT FROM OLD.channel
+       OR NEW.version IS DISTINCT FROM OLD.version + 1
+       OR NOT EXISTS (
+           SELECT 1
+             FROM grocery_historicalretailpublicationactivation activation
+            WHERE activation.id::text = capability
+              AND activation.channel_id = OLD.channel
+              AND activation.sequence = NEW.version
+              AND activation.previous_revision_id IS NOT DISTINCT FROM OLD.current_revision_id
+              AND activation.target_revision_id IS NOT DISTINCT FROM NEW.current_revision_id
+       ) THEN
+        RAISE EXCEPTION 'historical channel requires its matching activation'
+            USING ERRCODE = '23514';
+    END IF;
+    RETURN NEW;
+END;
+$$ LANGUAGE plpgsql;
+
+CREATE OR REPLACE FUNCTION grocery_assert_historical_channel_state(channel_key varchar)
+RETURNS void AS $$
+DECLARE
+    channel_record grocery_historicalretailpublicationchannel%ROWTYPE;
+    activation_count bigint;
+    maximum_sequence bigint;
+    latest_target uuid;
+BEGIN
+    SELECT * INTO channel_record
+      FROM grocery_historicalretailpublicationchannel
+     WHERE channel = channel_key;
+    IF NOT FOUND THEN
+        RAISE EXCEPTION 'historical channel state is missing' USING ERRCODE = '23514';
+    END IF;
+    SELECT count(*), coalesce(max(sequence), 0)
+      INTO activation_count, maximum_sequence
+      FROM grocery_historicalretailpublicationactivation
+     WHERE channel_id = channel_key;
+    IF activation_count = 0 THEN
+        IF channel_record.version IS DISTINCT FROM 0::bigint
+           OR channel_record.current_revision_id IS NOT NULL THEN
+            RAISE EXCEPTION 'empty historical channel state is inconsistent'
+                USING ERRCODE = '23514';
+        END IF;
+        RETURN;
+    END IF;
+    SELECT target_revision_id INTO latest_target
+      FROM grocery_historicalretailpublicationactivation
+     WHERE channel_id = channel_key
+     ORDER BY sequence DESC
+     LIMIT 1;
+    IF channel_record.version IS DISTINCT FROM maximum_sequence
+       OR maximum_sequence IS DISTINCT FROM activation_count
+       OR channel_record.current_revision_id IS DISTINCT FROM latest_target
+       OR EXISTS (
+           SELECT 1
+             FROM grocery_historicalretailpublicationactivation current_event
+             LEFT JOIN grocery_historicalretailpublicationactivation previous_event
+               ON previous_event.channel_id = current_event.channel_id
+              AND previous_event.sequence = current_event.sequence - 1
+            WHERE current_event.channel_id = channel_key
+              AND (
+                  (current_event.sequence = 1 AND current_event.previous_revision_id IS NOT NULL)
+                  OR (
+                      current_event.sequence > 1
+                      AND (
+                          previous_event.id IS NULL
+                          OR current_event.previous_revision_id
+                             IS DISTINCT FROM previous_event.target_revision_id
+                      )
+                  )
+              )
+       ) THEN
+        RAISE EXCEPTION 'historical channel event history is inconsistent'
+            USING ERRCODE = '23514';
+    END IF;
+END;
+$$ LANGUAGE plpgsql;
+
+CREATE OR REPLACE FUNCTION grocery_check_historical_activation_deferred()
+RETURNS trigger AS $$
+BEGIN
+    PERFORM grocery_assert_historical_channel_state(NEW.channel_id);
+    RETURN NULL;
+END;
+$$ LANGUAGE plpgsql;
+
+CREATE OR REPLACE FUNCTION grocery_check_historical_channel_deferred()
+RETURNS trigger AS $$
+BEGIN
+    PERFORM grocery_assert_historical_channel_state(NEW.channel);
+    RETURN NULL;
+END;
+$$ LANGUAGE plpgsql;
+
+CREATE CONSTRAINT TRIGGER grocery_history_activation_state_consistent
+AFTER INSERT ON grocery_historicalretailpublicationactivation
+DEFERRABLE INITIALLY DEFERRED
+FOR EACH ROW EXECUTE FUNCTION grocery_check_historical_activation_deferred();
+
+CREATE CONSTRAINT TRIGGER grocery_history_channel_state_consistent
+AFTER INSERT OR UPDATE ON grocery_historicalretailpublicationchannel
+DEFERRABLE INITIALLY DEFERRED
+FOR EACH ROW EXECUTE FUNCTION grocery_check_historical_channel_deferred();
+"""
+
+
+RESTORE_GUARDS = r"""
+DROP TRIGGER IF EXISTS grocery_history_channel_state_consistent
+    ON grocery_historicalretailpublicationchannel;
+DROP TRIGGER IF EXISTS grocery_history_activation_state_consistent
+    ON grocery_historicalretailpublicationactivation;
+DROP FUNCTION IF EXISTS grocery_check_historical_channel_deferred();
+DROP FUNCTION IF EXISTS grocery_check_historical_activation_deferred();
+DROP FUNCTION IF EXISTS grocery_assert_historical_channel_state(varchar);
+
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
+"""
+
+
+class Migration(migrations.Migration):
+    dependencies = [("grocery", "0025_guard_historical_publication_seals")]
+
+    operations = [migrations.RunSQL(UPGRADE_GUARDS, RESTORE_GUARDS)]
diff --git a/grocery/tests/test_historical_activation_models.py b/grocery/tests/test_historical_activation_models.py
index 3d1110b..77f64c9 100644
--- a/grocery/tests/test_historical_activation_models.py
+++ b/grocery/tests/test_historical_activation_models.py
@@ -1,29 +1,99 @@
+import uuid
+
 import pytest
-from django.core.exceptions import ValidationError
-from django.db import DatabaseError, connection, transaction
+from django.contrib.auth import get_user_model
+from django.contrib.auth.models import Permission
+from django.core.exceptions import PermissionDenied, ValidationError
+from django.db import DatabaseError, transaction
 
-from grocery.historical_activation_models import HistoricalRetailPublicationChannel
+from grocery.historical_activation_models import (
+    HistoricalRetailPublicationActivation,
+    HistoricalRetailPublicationChannel,
+)
+from grocery.historical_activations import (
+    _set_historical_transition_token,
+    transition_historical_publication,
+)
+from grocery.historical_publications import seal_historical_publication
+from grocery.tests.historical_bundle_factory import create_reviewed_historical_bundle
 
 
 def test_historical_channel_is_fixed_and_rejects_direct_model_transition(db: None) -> None:
-    channel = HistoricalRetailPublicationChannel.objects.create()
-    assert (channel.channel, channel.version, channel.current_revision_id) == (
-        "HISTORICAL_RETAIL",
-        0,
-        None,
+    with pytest.raises(ValidationError, match="service"):
+        HistoricalRetailPublicationChannel.objects.create()
+    with pytest.raises(DatabaseError), transaction.atomic():
+        HistoricalRetailPublicationChannel.objects.bulk_create(
+            [HistoricalRetailPublicationChannel()]
+        )
+
+
+def test_historical_activation_is_authorized_idempotent_cas(transactional_db: None) -> None:
+    bundle = create_reviewed_historical_bundle()
+    revision = seal_historical_publication(
+        monthly_review_id=bundle.monthly_review.id,
+        regional_review_id=bundle.regional_review.id,
+        market_review_id=bundle.market_review.id,
+        compatibility_report_sha256="2" * 64,
     )
+    publisher = get_user_model().objects.create_user(username="historical-publisher")
+    publisher.user_permissions.add(
+        Permission.objects.get(codename="publish_historical_publication")
+    )
+    values = {
+        "operation_id": uuid.uuid4(),
+        "actor": publisher,
+        "operation": HistoricalRetailPublicationActivation.Operation.ACTIVATE,
+        "target_revision_id": revision.id,
+        "expected_current_revision_id": None,
+        "expected_version": 0,
+        "reason_code": "INITIAL_PUBLICATION",
+        "acceptance_evidence_sha256": "3" * 64,
+    }
+
+    activation, created = transition_historical_publication(**values)
+    replay, replay_created = transition_historical_publication(**values)
+
+    channel = HistoricalRetailPublicationChannel.objects.get()
+    assert created is True and replay_created is False and replay.id == activation.id
+    assert (channel.version, channel.current_revision_id) == (1, revision.id)
+    assert HistoricalRetailPublicationActivation.objects.count() == 1
 
-    channel.version = 1
-    with pytest.raises(ValidationError, match="events"):
-        channel.save()
+    stale_values = dict(values, operation_id=uuid.uuid4())
+    with pytest.raises(ValidationError, match="stale"):
+        transition_historical_publication(**stale_values)
+    assert HistoricalRetailPublicationActivation.objects.count() == 1
 
+    with pytest.raises(DatabaseError), transaction.atomic():
+        _set_historical_transition_token(uuid.uuid4())
+        orphan = HistoricalRetailPublicationActivation(
+            channel=channel,
+            operation=HistoricalRetailPublicationActivation.Operation.WITHDRAW,
+            sequence=2,
+            previous_revision=revision,
+            target_revision=None,
+            publisher=publisher,
+            reason_code="ORPHAN_PROBE",
+            acceptance_evidence_sha256="4" * 64,
+        )
+        orphan._transition_write = True
+        _set_historical_transition_token(orphan.id)
+        orphan.save()
+    assert HistoricalRetailPublicationActivation.objects.count() == 1
 
-def test_database_rejects_a_channel_update_without_matching_event(db: None) -> None:
-    HistoricalRetailPublicationChannel.objects.create()
+    outsider = get_user_model().objects.create_user(username="historical-outsider")
+    with pytest.raises(PermissionDenied):
+        transition_historical_publication(
+            operation_id=uuid.uuid4(),
+            actor=outsider,
+            operation=HistoricalRetailPublicationActivation.Operation.WITHDRAW,
+            target_revision_id=None,
+            expected_current_revision_id=revision.id,
+            expected_version=1,
+            reason_code="UNAUTHORIZED",
+            acceptance_evidence_sha256="5" * 64,
+        )
 
-    with pytest.raises(DatabaseError, match="matching event"), transaction.atomic():
-        with connection.cursor() as cursor:
-            cursor.execute(
-                "UPDATE grocery_historicalretailpublicationchannel "
-                "SET version = 1 WHERE channel = 'HISTORICAL_RETAIL'"
-            )
+    with pytest.raises(DatabaseError), transaction.atomic():
+        HistoricalRetailPublicationChannel.objects.filter(pk=channel.pk).update(
+            version=2,
+        )


