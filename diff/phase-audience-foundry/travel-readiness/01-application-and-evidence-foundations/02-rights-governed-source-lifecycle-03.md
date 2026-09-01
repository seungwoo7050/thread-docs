## `feat(sources): enforce the rights activation gate`

diff --git a/sources/migrations/0003_rights_activation_gate.py b/sources/migrations/0003_rights_activation_gate.py
new file mode 100644
index 0000000..f3eb551
--- /dev/null
+++ b/sources/migrations/0003_rights_activation_gate.py
@@ -0,0 +1,157 @@
+from django.db import migrations
+
+
+ACTIVATION_GATE_SQL = """
+CREATE OR REPLACE FUNCTION sources_guard_configuration_change() RETURNS trigger
+LANGUAGE plpgsql AS $$
+DECLARE
+    latest_decision text;
+BEGIN
+    IF TG_OP = 'INSERT' THEN
+        IF NEW.state <> 'DRAFT' OR NEW.enabled THEN
+            RAISE EXCEPTION 'new source configuration must be draft and disabled'
+                USING ERRCODE = 'check_violation';
+        END IF;
+        RETURN NEW;
+    END IF;
+    IF NEW.id IS DISTINCT FROM OLD.id OR NEW.code IS DISTINCT FROM OLD.code THEN
+        RAISE EXCEPTION 'source configuration identity is immutable'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    IF (NEW.module, NEW.owner, NEW.official_locator, NEW.connect_timeout_seconds,
+        NEW.read_timeout_seconds, NEW.max_retries, NEW.secret_reference_name)
+       IS DISTINCT FROM
+       (OLD.module, OLD.owner, OLD.official_locator, OLD.connect_timeout_seconds,
+        OLD.read_timeout_seconds, OLD.max_retries, OLD.secret_reference_name)
+       AND NEW.revision IS NOT DISTINCT FROM OLD.revision THEN
+        RAISE EXCEPTION 'contract changes require a new source revision'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    IF NEW.revision IS DISTINCT FROM OLD.revision THEN
+        IF NEW.state <> 'DRAFT' OR NEW.enabled THEN
+            RAISE EXCEPTION 'new source revision must reset to draft and disabled'
+                USING ERRCODE = 'check_violation';
+        END IF;
+        RETURN NEW;
+    END IF;
+    IF NEW.state IS DISTINCT FROM OLD.state
+       AND NOT (
+           (OLD.state = 'DRAFT' AND NEW.state IN ('RIGHTS_APPROVED', 'REJECTED'))
+           OR (OLD.state = 'RIGHTS_APPROVED' AND NEW.state IN ('ACTIVE', 'REJECTED'))
+           OR (OLD.state = 'ACTIVE' AND NEW.state IN ('PAUSED', 'REJECTED'))
+           OR (OLD.state = 'PAUSED' AND NEW.state IN ('ACTIVE', 'REJECTED'))
+       ) THEN
+        RAISE EXCEPTION 'invalid source lifecycle transition'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    IF NEW.state IN ('RIGHTS_APPROVED', 'ACTIVE') THEN
+        SELECT decision INTO latest_decision
+        FROM sources_sourcerightsdecision
+        WHERE source_id = NEW.id AND source_revision = NEW.revision
+        ORDER BY decision_sequence DESC
+        LIMIT 1;
+        IF latest_decision IS DISTINCT FROM 'APPROVED' THEN
+            RAISE EXCEPTION 'source activation requires the latest matching approval'
+                USING ERRCODE = 'check_violation';
+        END IF;
+    ELSIF NEW.state = 'REJECTED' THEN
+        SELECT decision INTO latest_decision
+        FROM sources_sourcerightsdecision
+        WHERE source_id = NEW.id AND source_revision = NEW.revision
+        ORDER BY decision_sequence DESC
+        LIMIT 1;
+        IF latest_decision IS DISTINCT FROM 'REJECTED' THEN
+            RAISE EXCEPTION 'source rejection requires the latest matching rejection'
+                USING ERRCODE = 'check_violation';
+        END IF;
+    END IF;
+    RETURN NEW;
+END;
+$$;
+
+CREATE FUNCTION sources_apply_rights_rejection() RETURNS trigger
+LANGUAGE plpgsql AS $$
+BEGIN
+    IF NEW.decision = 'REJECTED' THEN
+        UPDATE sources_sourceconfiguration
+        SET state = 'REJECTED', enabled = false
+        WHERE id = NEW.source_id AND revision = NEW.source_revision;
+        IF NOT FOUND THEN
+            RAISE EXCEPTION 'rights rejection could not close its exact source revision'
+                USING ERRCODE = 'check_violation';
+        END IF;
+    END IF;
+    RETURN NULL;
+END;
+$$;
+CREATE TRIGGER sources_rights_rejection_gate
+AFTER INSERT ON sources_sourcerightsdecision
+FOR EACH ROW EXECUTE FUNCTION sources_apply_rights_rejection();
+"""
+
+
+ACTIVATION_GATE_REVERSE_SQL = """
+LOCK TABLE sources_sourceconfiguration IN ACCESS EXCLUSIVE MODE;
+DO $$
+BEGIN
+    IF EXISTS (
+        SELECT 1 FROM sources_sourceconfiguration
+        WHERE state <> 'DRAFT' OR enabled
+    ) OR EXISTS (
+        SELECT 1 FROM sources_sourcerightsdecision
+    ) THEN
+        RAISE EXCEPTION 'activation gate rollback requires an empty draft-only database'
+            USING ERRCODE = 'check_violation';
+    END IF;
+END;
+$$;
+
+DROP TRIGGER IF EXISTS sources_rights_rejection_gate ON sources_sourcerightsdecision;
+DROP FUNCTION IF EXISTS sources_apply_rights_rejection();
+
+CREATE OR REPLACE FUNCTION sources_guard_configuration_change() RETURNS trigger
+LANGUAGE plpgsql AS $$
+BEGIN
+    IF TG_OP = 'INSERT' THEN
+        IF NEW.state <> 'DRAFT' OR NEW.enabled THEN
+            RAISE EXCEPTION 'new source configuration must be draft and disabled'
+                USING ERRCODE = 'check_violation';
+        END IF;
+        RETURN NEW;
+    END IF;
+    IF NEW.id IS DISTINCT FROM OLD.id OR NEW.code IS DISTINCT FROM OLD.code THEN
+        RAISE EXCEPTION 'source configuration identity is immutable'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    IF (NEW.module, NEW.owner, NEW.official_locator, NEW.connect_timeout_seconds,
+        NEW.read_timeout_seconds, NEW.max_retries, NEW.secret_reference_name)
+       IS DISTINCT FROM
+       (OLD.module, OLD.owner, OLD.official_locator, OLD.connect_timeout_seconds,
+        OLD.read_timeout_seconds, OLD.max_retries, OLD.secret_reference_name)
+       AND NEW.revision IS NOT DISTINCT FROM OLD.revision THEN
+        RAISE EXCEPTION 'contract changes require a new source revision'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    IF NEW.revision IS DISTINCT FROM OLD.revision
+       AND (NEW.state <> 'DRAFT' OR NEW.enabled) THEN
+        RAISE EXCEPTION 'new source revision must reset to draft and disabled'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    IF OLD.state = 'DRAFT' AND NEW.state <> 'DRAFT' THEN
+        RAISE EXCEPTION 'source activation requires the rights decision guard'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    RETURN NEW;
+END;
+$$;
+"""
+
+
+class Migration(migrations.Migration):
+    dependencies = [
+        ("sources", "0002_source_rights_decision"),
+    ]
+
+    operations = [
+        migrations.RunSQL(ACTIVATION_GATE_SQL, ACTIVATION_GATE_REVERSE_SQL),
+    ]
diff --git a/sources/tests/test_activation.py b/sources/tests/test_activation.py
new file mode 100644
index 0000000..c007fa6
--- /dev/null
+++ b/sources/tests/test_activation.py
@@ -0,0 +1,292 @@
+import threading
+
+from django.db import IntegrityError, close_old_connections, transaction
+from django.test import TestCase, TransactionTestCase
+
+from sources.models import SourceConfiguration, SourceRightsDecision
+
+
+class SourceActivationGateTests(TestCase):
+    def setUp(self):
+        self.source = SourceConfiguration.objects.create(
+            code="MOFA_ENTRY_CSV",
+            revision="rights-v1",
+            module=SourceConfiguration.Module.ENTRY,
+            owner="MOFA",
+            official_locator="https://example.invalid/source.csv",
+        )
+
+    def rights_values(self, **overrides):
+        values = {
+            "source": self.source,
+            "source_revision": self.source.revision,
+            "decision_sequence": 1,
+            "decision": SourceRightsDecision.Decision.APPROVED,
+            "access_mode": SourceRightsDecision.AccessMode.ANONYMOUS_PUBLIC,
+            "access_allowed": True,
+            "automated_collection_allowed": True,
+            "typed_field_storage_allowed": True,
+            "raw_body_storage_allowed": False,
+            "typed_republication_allowed": True,
+            "raw_retention_seconds": 0,
+            "typed_retention": SourceRightsDecision.Retention.PRODUCT_HISTORY,
+            "evidence_retention": SourceRightsDecision.Retention.PRODUCT_HISTORY,
+            "field_scope_code": "JP_ORDINARY_TEXT_V1",
+            "attribution_text": "Synthetic attribution",
+            "contract_fingerprint_sha256": "a" * 64,
+            "decided_by": "PROJECT_OWNER_REQUEST",
+            "decision_basis_code": "USER_DIRECTIVE_20260830",
+        }
+        values.update(overrides)
+        return values
+
+    def approve(self):
+        return SourceRightsDecision.objects.create(**self.rights_values())
+
+    def reject(self, **overrides):
+        values = self.rights_values(
+            decision_sequence=2,
+            decision=SourceRightsDecision.Decision.REJECTED,
+            access_mode=SourceRightsDecision.AccessMode.NO_ACCESS,
+            access_allowed=False,
+            automated_collection_allowed=False,
+            typed_field_storage_allowed=False,
+            raw_body_storage_allowed=False,
+            typed_republication_allowed=False,
+            raw_retention_seconds=0,
+            typed_retention=SourceRightsDecision.Retention.NONE,
+            field_scope_code="",
+            attribution_text="",
+        )
+        values.update(overrides)
+        return SourceRightsDecision.objects.create(**values)
+
+    def activate(self):
+        self.approve()
+        SourceConfiguration.objects.filter(pk=self.source.pk).update(
+            state=SourceConfiguration.State.RIGHTS_APPROVED
+        )
+        SourceConfiguration.objects.filter(pk=self.source.pk).update(
+            state=SourceConfiguration.State.ACTIVE,
+            enabled=True,
+        )
+        self.source.refresh_from_db()
+
+    def assert_update_rejected(self, **changes):
+        with self.assertRaises(IntegrityError), transaction.atomic():
+            SourceConfiguration.objects.filter(pk=self.source.pk).update(**changes)
+
+    def test_activation_requires_an_ordered_matching_approval(self):
+        self.assert_update_rejected(state=SourceConfiguration.State.RIGHTS_APPROVED)
+        self.assert_update_rejected(
+            state=SourceConfiguration.State.ACTIVE,
+            enabled=True,
+        )
+        self.approve()
+        self.assert_update_rejected(
+            state=SourceConfiguration.State.ACTIVE,
+            enabled=True,
+        )
+        SourceConfiguration.objects.filter(pk=self.source.pk).update(
+            state=SourceConfiguration.State.RIGHTS_APPROVED
+        )
+        SourceConfiguration.objects.filter(pk=self.source.pk).update(
+            state=SourceConfiguration.State.ACTIVE,
+            enabled=True,
+        )
+        self.source.refresh_from_db()
+        self.assertEqual(self.source.state, SourceConfiguration.State.ACTIVE)
+        self.assertTrue(self.source.enabled)
+
+    def test_active_source_can_pause_and_reactivate(self):
+        self.activate()
+        SourceConfiguration.objects.filter(pk=self.source.pk).update(
+            state=SourceConfiguration.State.PAUSED,
+            enabled=False,
+        )
+        SourceConfiguration.objects.filter(pk=self.source.pk).update(
+            state=SourceConfiguration.State.ACTIVE,
+            enabled=True,
+        )
+        self.source.refresh_from_db()
+        self.assertEqual(self.source.state, SourceConfiguration.State.ACTIVE)
+
+    def test_rejection_immediately_disables_and_requires_a_new_revision(self):
+        self.activate()
+        self.reject()
+        self.source.refresh_from_db()
+        self.assertEqual(self.source.state, SourceConfiguration.State.REJECTED)
+        self.assertFalse(self.source.enabled)
+        self.assert_update_rejected(state=SourceConfiguration.State.DRAFT)
+
+        SourceConfiguration.objects.filter(pk=self.source.pk).update(
+            revision="rights-v2",
+            state=SourceConfiguration.State.DRAFT,
+            enabled=False,
+        )
+        self.source.refresh_from_db()
+        SourceRightsDecision.objects.create(
+            **self.rights_values(
+                source_revision="rights-v2",
+                contract_fingerprint_sha256="b" * 64,
+            )
+        )
+        SourceConfiguration.objects.filter(pk=self.source.pk).update(
+            state=SourceConfiguration.State.RIGHTS_APPROVED
+        )
+        self.source.refresh_from_db()
+        self.assertEqual(self.source.state, SourceConfiguration.State.RIGHTS_APPROVED)
+
+    def test_initial_rejection_closes_a_draft_source(self):
+        SourceRightsDecision.objects.create(
+            **self.rights_values(
+                decision=SourceRightsDecision.Decision.REJECTED,
+                access_mode=SourceRightsDecision.AccessMode.NO_ACCESS,
+                access_allowed=False,
+                automated_collection_allowed=False,
+                typed_field_storage_allowed=False,
+                raw_body_storage_allowed=False,
+                typed_republication_allowed=False,
+                typed_retention=SourceRightsDecision.Retention.NONE,
+                field_scope_code="",
+                attribution_text="",
+            )
+        )
+        self.source.refresh_from_db()
+        self.assertEqual(self.source.state, SourceConfiguration.State.REJECTED)
+        self.assertFalse(self.source.enabled)
+
+    def test_rejection_and_disable_roll_back_together(self):
+        self.activate()
+        with self.assertRaises(RuntimeError):
+            with transaction.atomic():
+                self.reject()
+                raise RuntimeError("force transaction rollback")
+        self.source.refresh_from_db()
+        self.assertEqual(self.source.state, SourceConfiguration.State.ACTIVE)
+        self.assertTrue(self.source.enabled)
+        self.assertEqual(self.source.rights_decisions.count(), 1)
+
+    def test_direct_rejection_and_invalid_graph_edges_are_blocked(self):
+        self.approve()
+        self.assert_update_rejected(state=SourceConfiguration.State.REJECTED)
+        self.assert_update_rejected(state=SourceConfiguration.State.PAUSED)
+        SourceConfiguration.objects.filter(pk=self.source.pk).update(
+            state=SourceConfiguration.State.RIGHTS_APPROVED
+        )
+        self.assert_update_rejected(state=SourceConfiguration.State.DRAFT)
+        self.assert_update_rejected(state=SourceConfiguration.State.PAUSED)
+
+
+class SourceActivationRaceTests(TransactionTestCase):
+    def setUp(self):
+        self.source = SourceConfiguration.objects.create(
+            code="MOFA_ENTRY_RACE",
+            revision="rights-v1",
+            module=SourceConfiguration.Module.ENTRY,
+            owner="MOFA",
+            official_locator="https://example.invalid/source.csv",
+        )
+        SourceRightsDecision.objects.create(
+            source=self.source,
+            source_revision="rights-v1",
+            decision_sequence=1,
+            decision=SourceRightsDecision.Decision.APPROVED,
+            access_mode=SourceRightsDecision.AccessMode.ANONYMOUS_PUBLIC,
+            access_allowed=True,
+            automated_collection_allowed=True,
+            typed_field_storage_allowed=True,
+            raw_body_storage_allowed=False,
+            typed_republication_allowed=True,
+            raw_retention_seconds=0,
+            typed_retention=SourceRightsDecision.Retention.PRODUCT_HISTORY,
+            evidence_retention=SourceRightsDecision.Retention.PRODUCT_HISTORY,
+            field_scope_code="JP_ORDINARY_TEXT_V1",
+            attribution_text="Synthetic attribution",
+            contract_fingerprint_sha256="a" * 64,
+            decided_by="PROJECT_OWNER_REQUEST",
+            decision_basis_code="USER_DIRECTIVE_20260830",
+        )
+        SourceConfiguration.objects.filter(pk=self.source.pk).update(
+            state=SourceConfiguration.State.RIGHTS_APPROVED
+        )
+        SourceConfiguration.objects.filter(pk=self.source.pk).update(
+            state=SourceConfiguration.State.ACTIVE,
+            enabled=True,
+        )
+        SourceConfiguration.objects.filter(pk=self.source.pk).update(
+            state=SourceConfiguration.State.PAUSED,
+            enabled=False,
+        )
+
+    def test_concurrent_reactivation_cannot_outrun_rejection(self):
+        barrier = threading.Barrier(2)
+        outcomes = []
+        errors = []
+        result_lock = threading.Lock()
+
+        def record(collection, value):
+            with result_lock:
+                collection.append(value)
+
+        def reactivate():
+            close_old_connections()
+            try:
+                barrier.wait()
+                with transaction.atomic():
+                    SourceConfiguration.objects.filter(pk=self.source.pk).update(
+                        state=SourceConfiguration.State.ACTIVE,
+                        enabled=True,
+                    )
+                record(outcomes, "reactivated")
+            except IntegrityError:
+                record(outcomes, "reactivation_blocked")
+            except BaseException as exc:
+                record(errors, type(exc).__name__)
+            finally:
+                close_old_connections()
+
+        def reject():
+            close_old_connections()
+            try:
+                barrier.wait()
+                with transaction.atomic():
+                    SourceRightsDecision.objects.create(
+                        source_id=self.source.pk,
+                        source_revision="rights-v1",
+                        decision_sequence=2,
+                        decision=SourceRightsDecision.Decision.REJECTED,
+                        access_mode=SourceRightsDecision.AccessMode.NO_ACCESS,
+                        access_allowed=False,
+                        automated_collection_allowed=False,
+                        typed_field_storage_allowed=False,
+                        raw_body_storage_allowed=False,
+                        typed_republication_allowed=False,
+                        raw_retention_seconds=0,
+                        typed_retention=SourceRightsDecision.Retention.NONE,
+                        evidence_retention=SourceRightsDecision.Retention.PRODUCT_HISTORY,
+                        field_scope_code="",
+                        attribution_text="",
+                        contract_fingerprint_sha256="a" * 64,
+                        decided_by="PROJECT_OWNER_REQUEST",
+                        decision_basis_code="USER_DIRECTIVE_20260830",
+                    )
+                record(outcomes, "rejected")
+            except BaseException as exc:
+                record(errors, type(exc).__name__)
+            finally:
+                close_old_connections()
+
+        threads = [threading.Thread(target=reactivate), threading.Thread(target=reject)]
+        for worker in threads:
+            worker.start()
+        for worker in threads:
+            worker.join(timeout=10)
+
+        self.assertFalse(any(worker.is_alive() for worker in threads))
+        self.assertEqual(errors, [])
+        self.assertIn("rejected", outcomes)
+        self.assertEqual(len(outcomes), 2)
+        self.source.refresh_from_db()
+        self.assertEqual(self.source.state, SourceConfiguration.State.REJECTED)
+        self.assertFalse(self.source.enabled)


