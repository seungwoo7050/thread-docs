## `feat(sources): support reviewed contract revisions`

diff --git a/sources/management/commands/register_approved_sources.py b/sources/management/commands/register_approved_sources.py
index bbd4541..2f97b34 100644
--- a/sources/management/commands/register_approved_sources.py
+++ b/sources/management/commands/register_approved_sources.py
@@ -57,6 +57,7 @@ class ApprovedSourceSpec:
     connect_timeout_seconds: int = 5
     read_timeout_seconds: int = 15
     max_retries: int = 2
+    prior_contracts: tuple["ApprovedSourceSpec", ...] = ()
 
     def configuration_values(self) -> dict[str, object]:
         return {
@@ -266,6 +267,7 @@ class RegistrationPlan:
     spec: ApprovedSourceSpec
     source: SourceConfiguration | None
     approval_exists: bool
+    upgrade_from: ApprovedSourceSpec | None = None
 
 
 @dataclass(frozen=True)
@@ -320,8 +322,41 @@ def _validate_configuration(
         _raise_conflict("SOURCE_CONFIGURATION_CONFLICT", spec.code)
 
 
-def _validate_rights_history(
+def _known_contracts(spec: ApprovedSourceSpec) -> tuple[ApprovedSourceSpec, ...]:
+    contracts = (*spec.prior_contracts, spec)
+    revisions = [contract.revision for contract in contracts]
+    if (
+        any(
+            contract.code != spec.code
+            or contract.module != spec.module
+            or contract.prior_contracts
+            for contract in spec.prior_contracts
+        )
+        or len(revisions) != len(set(revisions))
+    ):
+        _raise_conflict("SOURCE_CONTRACT_HISTORY_INVALID", spec.code)
+    return contracts
+
+
+def _configuration_contract(
     source: SourceConfiguration, spec: ApprovedSourceSpec
+) -> ApprovedSourceSpec:
+    contracts = _known_contracts(spec)
+    matches = tuple(
+        contract for contract in contracts
+        if contract.revision == source.revision
+    )
+    if not matches:
+        _raise_conflict("SOURCE_REVISION_CONFLICT", spec.code)
+    contract = matches[0]
+    _validate_configuration(source, contract)
+    return contract
+
+
+def _validate_rights_history(
+    source: SourceConfiguration,
+    spec: ApprovedSourceSpec,
+    configuration_contract: ApprovedSourceSpec,
 ) -> bool:
     decisions = list(
         SourceRightsDecision.objects.select_for_update()
@@ -329,32 +364,50 @@ def _validate_rights_history(
         .order_by("source_revision", "decision_sequence", "id")
     )
     if not decisions:
-        return False
+        if configuration_contract == spec:
+            return False
+        _raise_conflict("RIGHTS_HISTORY_CONFLICT", spec.code)
     if any(
         decision.decision == SourceRightsDecision.Decision.REJECTED
         for decision in decisions
     ):
         _raise_conflict("RIGHTS_REJECTION_HISTORY", spec.code)
-    if any(
-        decision.contract_fingerprint_sha256
-        != spec.contract_fingerprint_sha256
-        for decision in decisions
-    ):
-        _raise_conflict("RIGHTS_FINGERPRINT_MISMATCH", spec.code)
-    if len(decisions) != 1 or decisions[0].source_revision != spec.revision:
-        _raise_conflict("RIGHTS_HISTORY_CONFLICT", spec.code)
+    contracts = _known_contracts(spec)
+    contracts_by_revision = {
+        contract.revision: contract for contract in contracts
+    }
+    configuration_index = contracts.index(configuration_contract)
+    seen_revisions: set[str] = set()
+    for decision in decisions:
+        contract = contracts_by_revision.get(decision.source_revision)
+        if (
+            contract is None
+            or contracts.index(contract) > configuration_index
+            or decision.source_revision in seen_revisions
+        ):
+            _raise_conflict("RIGHTS_HISTORY_CONFLICT", spec.code)
+        if (
+            decision.contract_fingerprint_sha256
+            != contract.contract_fingerprint_sha256
+        ):
+            _raise_conflict("RIGHTS_FINGERPRINT_MISMATCH", spec.code)
+        expected = contract.rights_values()
+        if any(
+            getattr(decision, field) != expected[field]
+            for field in RIGHTS_COMPARE_FIELDS
+        ):
+            _raise_conflict("RIGHTS_HISTORY_CONFLICT", spec.code)
+        seen_revisions.add(decision.source_revision)
 
-    expected = spec.rights_values()
-    decision = decisions[0]
-    if any(
-        getattr(decision, field) != expected[field]
-        for field in RIGHTS_COMPARE_FIELDS
-    ):
+    if configuration_contract.revision not in seen_revisions:
+        if configuration_contract == spec and not decisions:
+            return False
         _raise_conflict("RIGHTS_HISTORY_CONFLICT", spec.code)
-    return True
+    return spec.revision in seen_revisions
 
 
 def _inspect_spec(spec: ApprovedSourceSpec) -> RegistrationPlan:
+    _known_contracts(spec)
     approved_locator_siblings = {
         candidate.code
         for candidate in APPROVED_SOURCE_SPECS
@@ -377,13 +430,29 @@ def _inspect_spec(spec: ApprovedSourceSpec) -> RegistrationPlan:
     if source is None:
         return RegistrationPlan(spec=spec, source=None, approval_exists=False)
 
-    _validate_configuration(source, spec)
-    approval_exists = _validate_rights_history(source, spec)
+    configuration_contract = _configuration_contract(source, spec)
+    approval_exists = _validate_rights_history(
+        source,
+        spec,
+        configuration_contract,
+    )
 
     if source.state == SourceConfiguration.State.PAUSED:
         _raise_conflict("SOURCE_PAUSED", spec.code)
     if source.state == SourceConfiguration.State.REJECTED:
         _raise_conflict("SOURCE_REJECTED", spec.code)
+    upgrade_from = (
+        configuration_contract if configuration_contract != spec else None
+    )
+    if upgrade_from is not None:
+        if source.state != SourceConfiguration.State.ACTIVE or not source.enabled:
+            _raise_conflict("SOURCE_STATE_CONFLICT", spec.code)
+        return RegistrationPlan(
+            spec=spec,
+            source=source,
+            approval_exists=False,
+            upgrade_from=upgrade_from,
+        )
     if source.state == SourceConfiguration.State.ACTIVE:
         if not source.enabled or not approval_exists:
             _raise_conflict("SOURCE_STATE_CONFLICT", spec.code)
@@ -411,6 +480,8 @@ def _inspect_spec(spec: ApprovedSourceSpec) -> RegistrationPlan:
 def _check_outcome(plan: RegistrationPlan) -> RegistrationOutcome:
     if plan.source is None:
         result = "WOULD_CREATE_AND_ACTIVATE"
+    elif plan.upgrade_from is not None:
+        result = "WOULD_UPGRADE_AND_ACTIVATE"
     elif plan.source.state == SourceConfiguration.State.ACTIVE:
         result = "ALREADY_ACTIVE"
     elif plan.approval_exists:
@@ -434,6 +505,28 @@ def _apply_plan(plan: RegistrationPlan) -> RegistrationOutcome:
             enabled=False,
         )
 
+    if plan.upgrade_from is not None:
+        prior_values = plan.upgrade_from.configuration_values()
+        updated = SourceConfiguration.objects.filter(
+            pk=source.pk,
+            state=SourceConfiguration.State.ACTIVE,
+            enabled=True,
+            **{
+                field: prior_values[field]
+                for field in CONFIGURATION_COMPARE_FIELDS
+            },
+        ).update(
+            **spec.configuration_values(),
+            state=SourceConfiguration.State.DRAFT,
+            enabled=False,
+        )
+        if updated != 1:
+            _raise_conflict("SOURCE_STATE_CONFLICT", spec.code)
+        for field, value in spec.configuration_values().items():
+            setattr(source, field, value)
+        source.state = SourceConfiguration.State.DRAFT
+        source.enabled = False
+
     if approval_created:
         SourceRightsDecision.objects.create(
             source=source,
@@ -464,7 +557,9 @@ def _apply_plan(plan: RegistrationPlan) -> RegistrationOutcome:
     if source.state != SourceConfiguration.State.ACTIVE or not source.enabled:
         _raise_conflict("SOURCE_STATE_CONFLICT", spec.code)
 
-    if created:
+    if plan.upgrade_from is not None:
+        result = "UPGRADED_AND_ACTIVATED"
+    elif created:
         result = "CREATED_AND_ACTIVATED"
     elif approval_created:
         result = "APPROVED_AND_ACTIVATED"
@@ -480,7 +575,7 @@ def _verify_active_spec(spec: ApprovedSourceSpec) -> None:
     _validate_configuration(source, spec)
     if source.state != SourceConfiguration.State.ACTIVE or not source.enabled:
         _raise_conflict("SOURCE_STATE_CONFLICT", spec.code)
-    if not _validate_rights_history(source, spec):
+    if not _validate_rights_history(source, spec, spec):
         _raise_conflict("RIGHTS_HISTORY_CONFLICT", spec.code)
 
 
diff --git a/sources/tests/test_register_approved_sources.py b/sources/tests/test_register_approved_sources.py
index b16beb9..9d271ac 100644
--- a/sources/tests/test_register_approved_sources.py
+++ b/sources/tests/test_register_approved_sources.py
@@ -1,6 +1,7 @@
 import inspect
 import os
 import threading
+from dataclasses import replace
 from io import StringIO
 from unittest import mock
 
@@ -74,6 +75,37 @@ class ApprovedSourceRegistrationCommandTests(TransactionTestCase):
             official_locator="https://example.invalid/aviation",
         )
 
+    def make_upgrade_contracts(self, suffix=""):
+        code_suffix = f"_{suffix}" if suffix else ""
+        locator_suffix = suffix.lower() if suffix else "default"
+        prior = replace(
+            registration.APPROVED_SOURCE_SPECS[0],
+            code=f"SYNTHETIC_ENTRY_SOURCE{code_suffix}",
+            official_locator=f"https://example.invalid/{locator_suffix}",
+            prior_contracts=(),
+        )
+        current = replace(
+            prior,
+            revision="contract-v2",
+            contract_fingerprint_sha256="2" * 64,
+            decision_basis_code="SYNTHETIC_CURRENT_CONTRACT",
+            prior_contracts=(prior,),
+        )
+        return prior, current
+
+    def activate_source(self, source):
+        for old_state, new_state, enabled in (
+            (SourceConfiguration.State.DRAFT,
+             SourceConfiguration.State.RIGHTS_APPROVED, False),
+            (SourceConfiguration.State.RIGHTS_APPROVED,
+             SourceConfiguration.State.ACTIVE, True),
+        ):
+            updated = SourceConfiguration.objects.filter(
+                pk=source.pk, state=old_state, enabled=False
+            ).update(state=new_state, enabled=enabled)
+            self.assertEqual(updated, 1)
+        source.refresh_from_db()
+
     def test_default_check_is_read_only(self):
         output = self.call_registration()
 
@@ -339,6 +371,132 @@ class ApprovedSourceRegistrationCommandTests(TransactionTestCase):
             len(registration.APPROVED_SOURCE_SPECS),
         )
 
+    def test_exact_prior_contract_is_checked_upgraded_and_idempotent(self):
+        prior, current = self.make_upgrade_contracts()
+        source = self.make_exact_source(prior)
+        prior_approval = self.make_exact_approval(source, prior)
+        self.activate_source(source)
+        prior_identity = (prior_approval.pk, prior_approval.decided_at)
+
+        with mock.patch.object(
+            registration,
+            "APPROVED_SOURCE_SPECS",
+            (current,),
+        ):
+            output = self.call_registration("--check")
+
+            source.refresh_from_db()
+            self.assertEqual(source.revision, prior.revision)
+            self.assertEqual(source.rights_decisions.count(), 1)
+            self.assertIn("result=WOULD_UPGRADE_AND_ACTIVATE", output)
+
+            output = self.call_registration("--apply")
+
+            source.refresh_from_db()
+            prior_approval.refresh_from_db()
+            self.assertEqual(
+                (source.revision, source.state, source.enabled),
+                (current.revision, SourceConfiguration.State.ACTIVE, True),
+            )
+            self.assertEqual(
+                (prior_approval.pk, prior_approval.decided_at),
+                prior_identity,
+            )
+            self.assertEqual(
+                set(source.rights_decisions.values_list(
+                    "source_revision", "contract_fingerprint_sha256"
+                )),
+                {
+                    (prior.revision, prior.contract_fingerprint_sha256),
+                    (current.revision, current.contract_fingerprint_sha256),
+                },
+            )
+            self.assertIn("result=UPGRADED_AND_ACTIVATED", output)
+
+            rights_identity = set(
+                source.rights_decisions.values_list("id", "decided_at")
+            )
+            output = self.call_registration("--apply")
+
+        self.assertEqual(
+            set(source.rights_decisions.values_list("id", "decided_at")),
+            rights_identity,
+        )
+        self.assertIn("result=ALREADY_ACTIVE", output)
+
+    def test_upgrade_rejects_mismatched_prior_fingerprint(self):
+        prior, current = self.make_upgrade_contracts("MISMATCH")
+        source = self.make_exact_source(prior)
+        self.make_exact_approval(
+            source,
+            prior,
+            contract_fingerprint_sha256="f" * 64,
+        )
+        self.activate_source(source)
+
+        with mock.patch.object(
+            registration,
+            "APPROVED_SOURCE_SPECS",
+            (current,),
+        ):
+            error, _, _ = self.call_registration_failure("--apply")
+
+        self.assertIn("code=RIGHTS_FINGERPRINT_MISMATCH", error)
+        source.refresh_from_db()
+        self.assertEqual(source.revision, prior.revision)
+        self.assertEqual(source.rights_decisions.count(), 1)
+
+    def test_upgrade_rejects_unknown_and_rejected_history(self):
+        prior, current = self.make_upgrade_contracts("HISTORY")
+        source = self.make_exact_source(prior)
+        self.make_exact_approval(source, prior)
+        self.activate_source(source)
+
+        unknown_decision = mock.Mock(
+            decision=SourceRightsDecision.Decision.APPROVED,
+            source_revision="unknown-v0",
+        )
+        locked = mock.Mock()
+        locked.filter.return_value.order_by.return_value = [unknown_decision]
+        with mock.patch.object(
+            SourceRightsDecision.objects,
+            "select_for_update",
+            return_value=locked,
+        ), self.assertRaises(registration.RegistrationConflict) as caught:
+            registration._validate_rights_history(source, current, prior)
+        self.assertEqual(caught.exception.code, "RIGHTS_HISTORY_CONFLICT")
+
+        rejection = prior.rights_values()
+        rejection.update(
+            decision_sequence=2,
+            decision=SourceRightsDecision.Decision.REJECTED,
+            access_mode=SourceRightsDecision.AccessMode.NO_ACCESS,
+            access_allowed=False,
+            automated_collection_allowed=False,
+            typed_field_storage_allowed=False,
+            raw_body_storage_allowed=False,
+            typed_republication_allowed=False,
+            typed_retention=SourceRightsDecision.Retention.NONE,
+            field_scope_code="",
+            attribution_text="",
+            decision_basis_code="SYNTHETIC_REJECTION_TEST",
+        )
+        SourceRightsDecision.objects.create(source=source, **rejection)
+
+        with mock.patch.object(
+            registration,
+            "APPROVED_SOURCE_SPECS",
+            (current,),
+        ):
+            error, _, _ = self.call_registration_failure("--apply")
+
+        self.assertIn("code=RIGHTS_REJECTION_HISTORY", error)
+        source.refresh_from_db()
+        self.assertEqual(source.revision, prior.revision)
+        self.assertEqual(source.state, SourceConfiguration.State.REJECTED)
+        self.assertFalse(source.enabled)
+        self.assertEqual(source.rights_decisions.count(), 2)
+
     def test_exact_partial_rows_resume_without_replacement(self):
         entry_spec, warning_spec = registration.APPROVED_SOURCE_SPECS[:2]
         entry = self.make_exact_source(entry_spec)
