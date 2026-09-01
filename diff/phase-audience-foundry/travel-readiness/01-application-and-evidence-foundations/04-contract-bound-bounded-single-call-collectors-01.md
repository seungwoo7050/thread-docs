# 출처 계약에 묶인 제한형 단일 호출 수집기

## `feat(sources): register approved source contracts`

diff --git a/sources/management/__init__.py b/sources/management/__init__.py
new file mode 100644
index 0000000..0af8d4a
--- /dev/null
+++ b/sources/management/__init__.py
@@ -0,0 +1 @@
+"""Management commands for approved source lifecycle operations."""
diff --git a/sources/management/commands/__init__.py b/sources/management/commands/__init__.py
new file mode 100644
index 0000000..142e6d9
--- /dev/null
+++ b/sources/management/commands/__init__.py
@@ -0,0 +1 @@
+"""Approved source lifecycle commands."""
diff --git a/sources/management/commands/register_approved_sources.py b/sources/management/commands/register_approved_sources.py
new file mode 100644
index 0000000..097cead
--- /dev/null
+++ b/sources/management/commands/register_approved_sources.py
@@ -0,0 +1,410 @@
+from dataclasses import dataclass
+
+from django.core.management.base import BaseCommand, CommandError
+from django.db import DatabaseError, connection, transaction
+
+from sources.models import SourceConfiguration, SourceRightsDecision
+
+
+REGISTRATION_LOCK_NAMESPACE = 1_414_678_614
+REGISTRATION_LOCK_KEY = 20_260_830
+
+
+@dataclass(frozen=True)
+class ApprovedSourceSpec:
+    code: str
+    revision: str
+    module: str
+    owner: str
+    official_locator: str
+    secret_reference_name: str
+    access_mode: str
+    field_scope_code: str
+    contract_fingerprint_sha256: str
+    decision_basis_code: str
+    connect_timeout_seconds: int = 5
+    read_timeout_seconds: int = 15
+    max_retries: int = 2
+
+    def configuration_values(self) -> dict[str, object]:
+        return {
+            "code": self.code,
+            "revision": self.revision,
+            "module": self.module,
+            "owner": self.owner,
+            "official_locator": self.official_locator,
+            "connect_timeout_seconds": self.connect_timeout_seconds,
+            "read_timeout_seconds": self.read_timeout_seconds,
+            "max_retries": self.max_retries,
+            "secret_reference_name": self.secret_reference_name,
+        }
+
+    def rights_values(self) -> dict[str, object]:
+        return {
+            "source_revision": self.revision,
+            "decision_sequence": 1,
+            "decision": SourceRightsDecision.Decision.APPROVED,
+            "access_mode": self.access_mode,
+            "access_allowed": True,
+            "automated_collection_allowed": True,
+            "typed_field_storage_allowed": True,
+            "raw_body_storage_allowed": False,
+            "typed_republication_allowed": True,
+            "raw_retention_seconds": 0,
+            "typed_retention": SourceRightsDecision.Retention.PRODUCT_HISTORY,
+            "evidence_retention": SourceRightsDecision.Retention.PRODUCT_HISTORY,
+            "field_scope_code": self.field_scope_code,
+            "attribution_text": "외교부|공공데이터포털",
+            "contract_fingerprint_sha256": self.contract_fingerprint_sha256,
+            "decided_by": "PROJECT_OWNER_REQUEST",
+            "decision_basis_code": self.decision_basis_code,
+        }
+
+
+APPROVED_SOURCE_SPECS = (
+    ApprovedSourceSpec(
+        code="MOFA_ENTRY_CSV",
+        revision="rights-v1",
+        module=SourceConfiguration.Module.ENTRY,
+        owner="대한민국 외교부 정보화담당관실",
+        official_locator=(
+            "https://www.data.go.kr/cmm/cmm/fileDownload.do?"
+            "atchFileId=FILE_000000003090472&fileDetailSn=1&insertDataPrcus=N"
+        ),
+        secret_reference_name="",
+        access_mode=SourceRightsDecision.AccessMode.ANONYMOUS_PUBLIC,
+        field_scope_code="JP_ORDINARY_TEXT_V1",
+        contract_fingerprint_sha256=(
+            "622a399317ab730f9e9780f51b3ac837073cd99939f07f248323636e97676021"
+        ),
+        decision_basis_code="USER_DIRECTIVE_20260830",
+    ),
+    ApprovedSourceSpec(
+        code="MOFA_TRAVEL_ALARM_API_JP",
+        revision="rights-v1",
+        module=SourceConfiguration.Module.TRAVEL_WARNING,
+        owner="대한민국 외교부",
+        official_locator=(
+            "https://apis.data.go.kr/1262000/TravelAlarmService2/"
+            "getTravelAlarmList2"
+        ),
+        secret_reference_name="MOFA_TRAVEL_ALARM_SERVICE_KEY",
+        access_mode=SourceRightsDecision.AccessMode.CREDENTIAL_REFERENCE,
+        field_scope_code="JP_WARNING_V1",
+        contract_fingerprint_sha256=(
+            "c43c5f7e7e6f37f13ba7e5d9e2448b29b24d5524012b41fc5e851abb907c55f6"
+        ),
+        decision_basis_code="USER_FOLLOWUP_20260830",
+    ),
+)
+
+
+CONFIGURATION_COMPARE_FIELDS = (
+    "revision",
+    "module",
+    "owner",
+    "official_locator",
+    "connect_timeout_seconds",
+    "read_timeout_seconds",
+    "max_retries",
+    "secret_reference_name",
+)
+RIGHTS_COMPARE_FIELDS = (
+    "source_revision",
+    "decision_sequence",
+    "decision",
+    "access_mode",
+    "access_allowed",
+    "automated_collection_allowed",
+    "typed_field_storage_allowed",
+    "raw_body_storage_allowed",
+    "typed_republication_allowed",
+    "raw_retention_seconds",
+    "typed_retention",
+    "evidence_retention",
+    "field_scope_code",
+    "attribution_text",
+    "contract_fingerprint_sha256",
+    "decided_by",
+    "decision_basis_code",
+)
+
+
+class RegistrationConflict(Exception):
+    def __init__(self, code: str, source_code: str):
+        self.code = code
+        self.source_code = source_code
+        super().__init__(code)
+
+
+@dataclass(frozen=True)
+class RegistrationPlan:
+    spec: ApprovedSourceSpec
+    source: SourceConfiguration | None
+    approval_exists: bool
+
+
+@dataclass(frozen=True)
+class RegistrationOutcome:
+    source_code: str
+    revision: str
+    result: str
+
+
+def _raise_conflict(code: str, source_code: str) -> None:
+    raise RegistrationConflict(code, source_code)
+
+
+def _acquire_registration_lock() -> None:
+    if connection.vendor != "postgresql":
+        _raise_conflict("POSTGRESQL_REQUIRED", "REGISTRY")
+    with connection.cursor() as cursor:
+        cursor.execute(
+            "SELECT pg_advisory_xact_lock(%s, %s)",
+            [REGISTRATION_LOCK_NAMESPACE, REGISTRATION_LOCK_KEY],
+        )
+
+
+def _validate_aviation_boundary() -> None:
+    aviation_sources = list(
+        SourceConfiguration.objects.select_for_update()
+        .filter(module=SourceConfiguration.Module.AVIATION)
+        .only("state", "enabled")
+    )
+    if any(
+        source.state != SourceConfiguration.State.DRAFT or source.enabled
+        for source in aviation_sources
+    ):
+        _raise_conflict("AVIATION_BOUNDARY_VIOLATION", "AVIATION")
+
+
+def _validate_configuration(
+    source: SourceConfiguration, spec: ApprovedSourceSpec
+) -> None:
+    expected = spec.configuration_values()
+    if source.revision != spec.revision:
+        _raise_conflict("SOURCE_REVISION_CONFLICT", spec.code)
+    if any(
+        getattr(source, field) != expected[field]
+        for field in CONFIGURATION_COMPARE_FIELDS
+    ):
+        _raise_conflict("SOURCE_CONFIGURATION_CONFLICT", spec.code)
+
+
+def _validate_rights_history(
+    source: SourceConfiguration, spec: ApprovedSourceSpec
+) -> bool:
+    decisions = list(
+        SourceRightsDecision.objects.select_for_update()
+        .filter(source=source)
+        .order_by("source_revision", "decision_sequence", "id")
+    )
+    if not decisions:
+        return False
+    if any(
+        decision.decision == SourceRightsDecision.Decision.REJECTED
+        for decision in decisions
+    ):
+        _raise_conflict("RIGHTS_REJECTION_HISTORY", spec.code)
+    if any(
+        decision.contract_fingerprint_sha256
+        != spec.contract_fingerprint_sha256
+        for decision in decisions
+    ):
+        _raise_conflict("RIGHTS_FINGERPRINT_MISMATCH", spec.code)
+    if len(decisions) != 1 or decisions[0].source_revision != spec.revision:
+        _raise_conflict("RIGHTS_HISTORY_CONFLICT", spec.code)
+
+    expected = spec.rights_values()
+    decision = decisions[0]
+    if any(
+        getattr(decision, field) != expected[field]
+        for field in RIGHTS_COMPARE_FIELDS
+    ):
+        _raise_conflict("RIGHTS_HISTORY_CONFLICT", spec.code)
+    return True
+
+
+def _inspect_spec(spec: ApprovedSourceSpec) -> RegistrationPlan:
+    locator_conflict = (
+        SourceConfiguration.objects.select_for_update()
+        .filter(official_locator=spec.official_locator)
+        .exclude(code=spec.code)
+        .exists()
+    )
+    if locator_conflict:
+        _raise_conflict("SOURCE_LOCATOR_CONFLICT", spec.code)
+
+    source = (
+        SourceConfiguration.objects.select_for_update()
+        .filter(code=spec.code)
+        .first()
+    )
+    if source is None:
+        return RegistrationPlan(spec=spec, source=None, approval_exists=False)
+
+    _validate_configuration(source, spec)
+    approval_exists = _validate_rights_history(source, spec)
+
+    if source.state == SourceConfiguration.State.PAUSED:
+        _raise_conflict("SOURCE_PAUSED", spec.code)
+    if source.state == SourceConfiguration.State.REJECTED:
+        _raise_conflict("SOURCE_REJECTED", spec.code)
+    if source.state == SourceConfiguration.State.ACTIVE:
+        if not source.enabled or not approval_exists:
+            _raise_conflict("SOURCE_STATE_CONFLICT", spec.code)
+    elif source.state in (
+        SourceConfiguration.State.DRAFT,
+        SourceConfiguration.State.RIGHTS_APPROVED,
+    ):
+        if source.enabled:
+            _raise_conflict("SOURCE_STATE_CONFLICT", spec.code)
+        if (
+            source.state == SourceConfiguration.State.RIGHTS_APPROVED
+            and not approval_exists
+        ):
+            _raise_conflict("SOURCE_STATE_CONFLICT", spec.code)
+    else:
+        _raise_conflict("SOURCE_STATE_CONFLICT", spec.code)
+
+    return RegistrationPlan(
+        spec=spec,
+        source=source,
+        approval_exists=approval_exists,
+    )
+
+
+def _check_outcome(plan: RegistrationPlan) -> RegistrationOutcome:
+    if plan.source is None:
+        result = "WOULD_CREATE_AND_ACTIVATE"
+    elif plan.source.state == SourceConfiguration.State.ACTIVE:
+        result = "ALREADY_ACTIVE"
+    elif plan.approval_exists:
+        result = "WOULD_ACTIVATE"
+    else:
+        result = "WOULD_APPROVE_AND_ACTIVATE"
+    return RegistrationOutcome(plan.spec.code, plan.spec.revision, result)
+
+
+def _apply_plan(plan: RegistrationPlan) -> RegistrationOutcome:
+    spec = plan.spec
+    source = plan.source
+    initial_state = source.state if source is not None else None
+    created = source is None
+    approval_created = not plan.approval_exists
+
+    if source is None:
+        source = SourceConfiguration.objects.create(
+            **spec.configuration_values(),
+            state=SourceConfiguration.State.DRAFT,
+            enabled=False,
+        )
+
+    if approval_created:
+        SourceRightsDecision.objects.create(
+            source=source,
+            **spec.rights_values(),
+        )
+
+    if source.state == SourceConfiguration.State.DRAFT:
+        updated = SourceConfiguration.objects.filter(
+            pk=source.pk,
+            state=SourceConfiguration.State.DRAFT,
+            enabled=False,
+        ).update(state=SourceConfiguration.State.RIGHTS_APPROVED)
+        if updated != 1:
+            _raise_conflict("SOURCE_STATE_CONFLICT", spec.code)
+        source.state = SourceConfiguration.State.RIGHTS_APPROVED
+
+    if source.state == SourceConfiguration.State.RIGHTS_APPROVED:
+        updated = SourceConfiguration.objects.filter(
+            pk=source.pk,
+            state=SourceConfiguration.State.RIGHTS_APPROVED,
+            enabled=False,
+        ).update(state=SourceConfiguration.State.ACTIVE, enabled=True)
+        if updated != 1:
+            _raise_conflict("SOURCE_STATE_CONFLICT", spec.code)
+        source.state = SourceConfiguration.State.ACTIVE
+        source.enabled = True
+
+    if source.state != SourceConfiguration.State.ACTIVE or not source.enabled:
+        _raise_conflict("SOURCE_STATE_CONFLICT", spec.code)
+
+    if created:
+        result = "CREATED_AND_ACTIVATED"
+    elif approval_created:
+        result = "APPROVED_AND_ACTIVATED"
+    elif initial_state == SourceConfiguration.State.ACTIVE:
+        result = "ALREADY_ACTIVE"
+    else:
+        result = "ACTIVATED"
+    return RegistrationOutcome(spec.code, spec.revision, result)
+
+
+def _verify_active_spec(spec: ApprovedSourceSpec) -> None:
+    source = SourceConfiguration.objects.select_for_update().get(code=spec.code)
+    _validate_configuration(source, spec)
+    if source.state != SourceConfiguration.State.ACTIVE or not source.enabled:
+        _raise_conflict("SOURCE_STATE_CONFLICT", spec.code)
+    if not _validate_rights_history(source, spec):
+        _raise_conflict("RIGHTS_HISTORY_CONFLICT", spec.code)
+
+
+def register_approved_sources(*, apply: bool) -> tuple[RegistrationOutcome, ...]:
+    with transaction.atomic(durable=True):
+        _acquire_registration_lock()
+        _validate_aviation_boundary()
+        plans = tuple(_inspect_spec(spec) for spec in APPROVED_SOURCE_SPECS)
+
+        if not apply:
+            return tuple(_check_outcome(plan) for plan in plans)
+
+        outcomes = tuple(_apply_plan(plan) for plan in plans)
+        for spec in APPROVED_SOURCE_SPECS:
+            _verify_active_spec(spec)
+        _validate_aviation_boundary()
+        return outcomes
+
+
+class Command(BaseCommand):
+    help = "Check or register the two approved source contracts."
+    requires_migrations_checks = True
+
+    def add_arguments(self, parser):
+        mode = parser.add_mutually_exclusive_group()
+        mode.add_argument(
+            "--check",
+            action="store_true",
+            help="Validate without changing source or rights rows (default).",
+        )
+        mode.add_argument(
+            "--apply",
+            action="store_true",
+            help="Register and activate both exact approved source contracts.",
+        )
+
+    def handle(self, *args, **options):
+        apply = bool(options["apply"])
+        mode = "apply" if apply else "check"
+        try:
+            outcomes = register_approved_sources(apply=apply)
+        except RegistrationConflict as exc:
+            raise CommandError(
+                f"mode={mode} result=blocked code={exc.code} "
+                f"source={exc.source_code}"
+            ) from None
+        except DatabaseError:
+            raise CommandError(
+                f"mode={mode} result=blocked code=DATABASE_ERROR"
+            ) from None
+        except Exception:
+            raise CommandError(
+                f"mode={mode} result=blocked code=INTERNAL_ERROR"
+            ) from None
+
+        for outcome in outcomes:
+            self.stdout.write(
+                f"source={outcome.source_code} revision={outcome.revision} "
+                f"result={outcome.result}"
+            )
+        self.stdout.write(f"mode={mode} result=ok")
diff --git a/sources/tests/test_register_approved_sources.py b/sources/tests/test_register_approved_sources.py
new file mode 100644
index 0000000..ac167cf
--- /dev/null
+++ b/sources/tests/test_register_approved_sources.py
@@ -0,0 +1,531 @@
+import inspect
+import os
+import threading
+from io import StringIO
+from unittest import mock
+
+from django.core.management import call_command
+from django.core.management.base import CommandError
+from django.db import close_old_connections
+from django.test import TransactionTestCase
+from django.utils import timezone
+
+from sources.management.commands import register_approved_sources as registration
+from sources.models import SourceConfiguration, SourceRightsDecision
+
+
+SECRET_REFERENCE_NAME = "MOFA_TRAVEL_ALARM_SERVICE_KEY"
+SECRET_SENTINEL = "never-disclose-source-secret-sentinel-20260830"
+
+
+class ApprovedSourceRegistrationCommandTests(TransactionTestCase):
+    def call_registration(self, *arguments):
+        stdout = StringIO()
+        stderr = StringIO()
+        call_command(
+            "register_approved_sources",
+            *arguments,
+            stdout=stdout,
+            stderr=stderr,
+        )
+        output = stdout.getvalue() + stderr.getvalue()
+        self.assert_safe_output(output)
+        return output
+
+    def call_registration_failure(self, *arguments):
+        stdout = StringIO()
+        stderr = StringIO()
+        with self.assertRaises(CommandError) as caught:
+            call_command(
+                "register_approved_sources",
+                *arguments,
+                stdout=stdout,
+                stderr=stderr,
+            )
+        output = stdout.getvalue() + stderr.getvalue() + str(caught.exception)
+        self.assert_safe_output(output)
+        return str(caught.exception), stdout.getvalue(), stderr.getvalue()
+
+    def assert_safe_output(self, output):
+        self.assertNotIn("https://", output)
+        self.assertNotIn(SECRET_REFERENCE_NAME, output)
+        self.assertNotIn(SECRET_SENTINEL, output)
+
+    def make_exact_source(self, spec):
+        return SourceConfiguration.objects.create(
+            **spec.configuration_values(),
+            state=SourceConfiguration.State.DRAFT,
+            enabled=False,
+        )
+
+    def make_exact_approval(self, source, spec, **overrides):
+        values = spec.rights_values()
+        values.update(overrides)
+        return SourceRightsDecision.objects.create(source=source, **values)
+
+    def make_aviation_source(self):
+        return SourceConfiguration.objects.create(
+            code="AVIATION_UNAPPROVED_SOURCE",
+            revision="draft-v1",
+            module=SourceConfiguration.Module.AVIATION,
+            owner="Synthetic owner",
+            official_locator="https://example.invalid/aviation",
+        )
+
+    def test_default_check_is_read_only(self):
+        output = self.call_registration()
+
+        self.assertEqual(SourceConfiguration.objects.count(), 0)
+        self.assertEqual(SourceRightsDecision.objects.count(), 0)
+        self.assertEqual(output.count("result=WOULD_CREATE_AND_ACTIVATE"), 2)
+        self.assertIn("mode=check result=ok", output)
+
+    def test_apply_creates_the_two_exact_active_sources_and_rights(self):
+        output = self.call_registration("--apply")
+
+        self.assertEqual(SourceConfiguration.objects.count(), 2)
+        self.assertEqual(SourceRightsDecision.objects.count(), 2)
+        self.assertEqual(
+            set(
+                SourceConfiguration.objects.values_list(
+                    "code",
+                    "revision",
+                    "module",
+                    "owner",
+                    "official_locator",
+                    "state",
+                    "enabled",
+                    "connect_timeout_seconds",
+                    "read_timeout_seconds",
+                    "max_retries",
+                    "secret_reference_name",
+                )
+            ),
+            {
+                (
+                    "MOFA_ENTRY_CSV",
+                    "rights-v1",
+                    "ENTRY",
+                    "대한민국 외교부 정보화담당관실",
+                    "https://www.data.go.kr/cmm/cmm/fileDownload.do?"
+                    "atchFileId=FILE_000000003090472&fileDetailSn=1&insertDataPrcus=N",
+                    "ACTIVE",
+                    True,
+                    5,
+                    15,
+                    2,
+                    "",
+                ),
+                (
+                    "MOFA_TRAVEL_ALARM_API_JP",
+                    "rights-v1",
+                    "TRAVEL_WARNING",
+                    "대한민국 외교부",
+                    "https://apis.data.go.kr/1262000/TravelAlarmService2/"
+                    "getTravelAlarmList2",
+                    "ACTIVE",
+                    True,
+                    5,
+                    15,
+                    2,
+                    SECRET_REFERENCE_NAME,
+                ),
+            },
+        )
+        self.assertEqual(
+            set(
+                SourceRightsDecision.objects.select_related("source").values_list(
+                    "source__code",
+                    "source_revision",
+                    "decision_sequence",
+                    "decision",
+                    "access_mode",
+                    "access_allowed",
+                    "automated_collection_allowed",
+                    "typed_field_storage_allowed",
+                    "raw_body_storage_allowed",
+                    "typed_republication_allowed",
+                    "raw_retention_seconds",
+                    "typed_retention",
+                    "evidence_retention",
+                    "field_scope_code",
+                    "attribution_text",
+                    "contract_fingerprint_sha256",
+                    "decided_by",
+                    "decision_basis_code",
+                )
+            ),
+            {
+                (
+                    "MOFA_ENTRY_CSV",
+                    "rights-v1",
+                    1,
+                    "APPROVED",
+                    "ANONYMOUS_PUBLIC",
+                    True,
+                    True,
+                    True,
+                    False,
+                    True,
+                    0,
+                    "PRODUCT_HISTORY",
+                    "PRODUCT_HISTORY",
+                    "JP_ORDINARY_TEXT_V1",
+                    "외교부|공공데이터포털",
+                    "622a399317ab730f9e9780f51b3ac837073cd99939f07f248323636e97676021",
+                    "PROJECT_OWNER_REQUEST",
+                    "USER_DIRECTIVE_20260830",
+                ),
+                (
+                    "MOFA_TRAVEL_ALARM_API_JP",
+                    "rights-v1",
+                    1,
+                    "APPROVED",
+                    "CREDENTIAL_REFERENCE",
+                    True,
+                    True,
+                    True,
+                    False,
+                    True,
+                    0,
+                    "PRODUCT_HISTORY",
+                    "PRODUCT_HISTORY",
+                    "JP_WARNING_V1",
+                    "외교부|공공데이터포털",
+                    "c43c5f7e7e6f37f13ba7e5d9e2448b29b24d5524012b41fc5e851abb907c55f6",
+                    "PROJECT_OWNER_REQUEST",
+                    "USER_FOLLOWUP_20260830",
+                ),
+            },
+        )
+        self.assertTrue(
+            all(
+                timezone.is_aware(value)
+                for value in SourceRightsDecision.objects.values_list(
+                    "decided_at", flat=True
+                )
+            )
+        )
+        self.assertEqual(output.count("result=CREATED_AND_ACTIVATED"), 2)
+        self.assertEqual(
+            SourceConfiguration.objects.filter(
+                module=SourceConfiguration.Module.AVIATION
+            ).count(),
+            0,
+        )
+
+    def test_apply_rerun_is_a_noop_and_preserves_immutable_times(self):
+        self.call_registration("--apply")
+        configuration_identity = list(
+            SourceConfiguration.objects.order_by("code").values_list(
+                "id", "code", "created_at"
+            )
+        )
+        rights_identity = list(
+            SourceRightsDecision.objects.order_by("source__code").values_list(
+                "id", "source_id", "decided_at"
+            )
+        )
+
+        output = self.call_registration("--apply")
+
+        self.assertEqual(
+            list(
+                SourceConfiguration.objects.order_by("code").values_list(
+                    "id", "code", "created_at"
+                )
+            ),
+            configuration_identity,
+        )
+        self.assertEqual(
+            list(
+                SourceRightsDecision.objects.order_by("source__code").values_list(
+                    "id", "source_id", "decided_at"
+                )
+            ),
+            rights_identity,
+        )
+        self.assertEqual(output.count("result=ALREADY_ACTIVE"), 2)
+
+    def test_exact_partial_rows_resume_without_replacement(self):
+        entry_spec, warning_spec = registration.APPROVED_SOURCE_SPECS
+        entry = self.make_exact_source(entry_spec)
+        warning = self.make_exact_source(warning_spec)
+        warning_approval = self.make_exact_approval(warning, warning_spec)
+        SourceConfiguration.objects.filter(pk=warning.pk).update(
+            state=SourceConfiguration.State.RIGHTS_APPROVED
+        )
+
+        output = self.call_registration("--apply")
+
+        entry.refresh_from_db()
+        warning.refresh_from_db()
+        self.assertEqual(entry.state, SourceConfiguration.State.ACTIVE)
+        self.assertEqual(warning.state, SourceConfiguration.State.ACTIVE)
+        self.assertTrue(entry.enabled)
+        self.assertTrue(warning.enabled)
+        self.assertEqual(entry.rights_decisions.count(), 1)
+        self.assertEqual(warning.rights_decisions.get().pk, warning_approval.pk)
+        self.assertIn("result=APPROVED_AND_ACTIVATED", output)
+        self.assertIn("result=ACTIVATED", output)
+
+    def test_forced_failure_after_first_apply_rolls_back_every_change(self):
+        original_apply = registration._apply_plan
+
+        def fail_before_warning(plan):
+            if plan.spec.code == "MOFA_TRAVEL_ALARM_API_JP":
+                raise RuntimeError("synthetic registration interruption")
+            return original_apply(plan)
+
+        with mock.patch.object(
+            registration, "_apply_plan", side_effect=fail_before_warning
+        ):
+            error, stdout, stderr = self.call_registration_failure("--apply")
+
+        self.assertIn("code=INTERNAL_ERROR", error)
+        self.assertEqual(stdout, "")
+        self.assertEqual(stderr, "")
+        self.assertEqual(SourceConfiguration.objects.count(), 0)
+        self.assertEqual(SourceRightsDecision.objects.count(), 0)
+
+    def test_existing_configuration_conflict_does_not_partially_register(self):
+        warning_spec = registration.APPROVED_SOURCE_SPECS[1]
+        existing = SourceConfiguration.objects.create(
+            **{
+                **warning_spec.configuration_values(),
+                "owner": "Conflicting provider",
+            },
+            state=SourceConfiguration.State.DRAFT,
+            enabled=False,
+        )
+
+        error, _, _ = self.call_registration_failure("--apply")
+
+        self.assertIn("code=SOURCE_CONFIGURATION_CONFLICT", error)
+        self.assertFalse(
+            SourceConfiguration.objects.filter(code="MOFA_ENTRY_CSV").exists()
+        )
+        existing.refresh_from_db()
+        self.assertEqual(existing.owner, "Conflicting provider")
+        self.assertEqual(existing.rights_decisions.count(), 0)
+
+    def test_revision_conflict_never_reuses_or_rolls_back_identity(self):
+        warning_spec = registration.APPROVED_SOURCE_SPECS[1]
+        existing = SourceConfiguration.objects.create(
+            **{
+                **warning_spec.configuration_values(),
+                "revision": "rights-v2",
+            },
+            state=SourceConfiguration.State.DRAFT,
+            enabled=False,
+        )
+
+        error, _, _ = self.call_registration_failure("--apply")
+
+        self.assertIn("code=SOURCE_REVISION_CONFLICT", error)
+        self.assertFalse(
+            SourceConfiguration.objects.filter(code="MOFA_ENTRY_CSV").exists()
+        )
+        existing.refresh_from_db()
+        self.assertEqual(existing.revision, "rights-v2")
+        self.assertEqual(existing.state, SourceConfiguration.State.DRAFT)
+        self.assertEqual(existing.rights_decisions.count(), 0)
+
+    def test_fingerprint_conflict_fails_closed(self):
+        warning_spec = registration.APPROVED_SOURCE_SPECS[1]
+        source = self.make_exact_source(warning_spec)
+        self.make_exact_approval(
+            source,
+            warning_spec,
+            contract_fingerprint_sha256="f" * 64,
+        )
+
+        error, _, _ = self.call_registration_failure("--apply")
+
+        self.assertIn("code=RIGHTS_FINGERPRINT_MISMATCH", error)
+        self.assertFalse(
+            SourceConfiguration.objects.filter(code="MOFA_ENTRY_CSV").exists()
+        )
+        self.assertEqual(source.rights_decisions.count(), 1)
+
+    def test_capability_conflict_fails_closed(self):
+        warning_spec = registration.APPROVED_SOURCE_SPECS[1]
+        source = self.make_exact_source(warning_spec)
+        self.make_exact_approval(
+            source,
+            warning_spec,
+            field_scope_code="JP_WARNING_UNAPPROVED_SCOPE",
+        )
+
+        error, _, _ = self.call_registration_failure("--apply")
+
+        self.assertIn("code=RIGHTS_HISTORY_CONFLICT", error)
+        self.assertFalse(
+            SourceConfiguration.objects.filter(code="MOFA_ENTRY_CSV").exists()
+        )
+        self.assertEqual(source.rights_decisions.count(), 1)
+
+    def test_paused_source_is_not_silently_resumed(self):
+        self.call_registration("--apply")
+        warning = SourceConfiguration.objects.get(
+            code="MOFA_TRAVEL_ALARM_API_JP"
+        )
+        SourceConfiguration.objects.filter(pk=warning.pk).update(
+            state=SourceConfiguration.State.PAUSED,
+            enabled=False,
+        )
+        before_counts = (
+            SourceConfiguration.objects.count(),
+            SourceRightsDecision.objects.count(),
+        )
+
+        error, _, _ = self.call_registration_failure("--apply")
+
+        self.assertIn("code=SOURCE_PAUSED", error)
+        warning.refresh_from_db()
+        self.assertEqual(warning.state, SourceConfiguration.State.PAUSED)
+        self.assertFalse(warning.enabled)
+        self.assertEqual(
+            (
+                SourceConfiguration.objects.count(),
+                SourceRightsDecision.objects.count(),
+            ),
+            before_counts,
+        )
+
+    def test_rejection_history_is_never_reapproved(self):
+        self.call_registration("--apply")
+        warning = SourceConfiguration.objects.get(
+            code="MOFA_TRAVEL_ALARM_API_JP"
+        )
+        approval = warning.rights_decisions.get()
+        SourceRightsDecision.objects.create(
+            source=warning,
+            source_revision=warning.revision,
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
+            evidence_retention=SourceRightsDecision.Retention.PRODUCT_HISTORY,
+            field_scope_code="",
+            attribution_text="",
+            contract_fingerprint_sha256=approval.contract_fingerprint_sha256,
+            decided_by="PROJECT_OWNER_REQUEST",
+            decision_basis_code="SYNTHETIC_REJECTION_TEST",
+        )
+
+        error, _, _ = self.call_registration_failure("--apply")
+
+        self.assertIn("code=RIGHTS_REJECTION_HISTORY", error)
+        warning.refresh_from_db()
+        self.assertEqual(warning.state, SourceConfiguration.State.REJECTED)
+        self.assertFalse(warning.enabled)
+        self.assertEqual(warning.rights_decisions.count(), 2)
+
+    def test_draft_aviation_is_untouched_and_no_aviation_is_created(self):
+        aviation = self.make_aviation_source()
+        original = (aviation.pk, aviation.revision, aviation.created_at)
+
+        self.call_registration("--apply")
+
+        aviation.refresh_from_db()
+        self.assertEqual(
+            (aviation.pk, aviation.revision, aviation.created_at), original
+        )
+        self.assertEqual(aviation.state, SourceConfiguration.State.DRAFT)
+        self.assertFalse(aviation.enabled)
+        self.assertEqual(
+            SourceConfiguration.objects.filter(
+                module=SourceConfiguration.Module.AVIATION
+            ).count(),
+            1,
+        )
+
+    def test_secret_environment_is_not_read_or_disclosed(self):
+        module_source = inspect.getsource(registration).lower()
+        self.assertNotIn("import os", module_source)
+        self.assertNotIn("os.environ", module_source)
+        self.assertNotIn("getenv", module_source)
+        self.assertNotIn("dotenv", module_source)
+        self.assertNotIn(".env", module_source)
+
+        original_get = os.environ.get
+        original_getitem = type(os.environ).__getitem__
+
+        def guarded_get(name, default=None):
+            if name == SECRET_REFERENCE_NAME:
+                raise AssertionError("source secret environment read")
+            return original_get(name, default)
+
+        def guarded_getitem(environment, name):
+            if name == SECRET_REFERENCE_NAME:
+                raise AssertionError("source secret environment read")
+            return original_getitem(environment, name)
+
+        with mock.patch.dict(
+            os.environ, {SECRET_REFERENCE_NAME: SECRET_SENTINEL}
+        ), mock.patch.object(os.environ, "get", side_effect=guarded_get), mock.patch.object(
+            type(os.environ), "__getitem__", guarded_getitem
+        ):
+            output = self.call_registration("--apply")
+
+        database_text = repr(
+            list(SourceConfiguration.objects.values())
+            + list(SourceRightsDecision.objects.values())
+        )
+        self.assertNotIn(SECRET_SENTINEL, output)
+        self.assertNotIn(SECRET_SENTINEL, database_text)
+        self.assertNotIn(SECRET_REFERENCE_NAME, output)
+
+    def test_concurrent_apply_serializes_to_one_exact_registration(self):
+        barrier = threading.Barrier(2)
+        results = []
+        errors = []
+        result_lock = threading.Lock()
+
+        def worker():
+            close_old_connections()
+            stdout = StringIO()
+            stderr = StringIO()
+            try:
+                barrier.wait()
+                call_command(
+                    "register_approved_sources",
+                    "--apply",
+                    stdout=stdout,
+                    stderr=stderr,
+                )
+                with result_lock:
+                    results.append(stdout.getvalue() + stderr.getvalue())
+            except Exception as exc:
+                with result_lock:
+                    errors.append(type(exc).__name__)
+            finally:
+                close_old_connections()
+
+        threads = [threading.Thread(target=worker) for _ in range(2)]
+        for thread in threads:
+            thread.start()
+        for thread in threads:
+            thread.join(timeout=20)
+
+        self.assertFalse(any(thread.is_alive() for thread in threads))
+        self.assertEqual(errors, [])
+        self.assertEqual(len(results), 2)
+        for output in results:
+            self.assert_safe_output(output)
+        self.assertEqual(SourceConfiguration.objects.count(), 2)
+        self.assertEqual(SourceRightsDecision.objects.count(), 2)
+        self.assertTrue(
+            all(
+                source.state == SourceConfiguration.State.ACTIVE
+                and source.enabled
+                for source in SourceConfiguration.objects.all()
+            )
+        )


