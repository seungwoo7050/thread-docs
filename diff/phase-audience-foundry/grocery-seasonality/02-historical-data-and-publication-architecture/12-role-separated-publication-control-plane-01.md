# 역할 분리형 게시 제어 평면

## `feat(review): authorize local phase zero publication`

diff --git a/grocery/management/commands/approve_recent_generation.py b/grocery/management/commands/approve_recent_generation.py
new file mode 100644
index 0000000..fecd8e8
--- /dev/null
+++ b/grocery/management/commands/approve_recent_generation.py
@@ -0,0 +1,130 @@
+"""Record one exact local Phase 0 approval over a validated recent generation."""
+
+from __future__ import annotations
+
+import uuid
+
+from django.core.management.base import BaseCommand, CommandError, CommandParser
+from django.db import transaction
+
+from grocery.management.local_phase0 import (
+    LOCAL_APPROVAL_REASON_CODE,
+    LocalPhase0Code,
+    LocalPhase0Error,
+    canonical_actor_id,
+    get_local_operator,
+    require_sha256,
+    require_uuid,
+)
+from grocery.models import (
+    FetchAttempt,
+    ParseRun,
+    ReviewDecision,
+    SourceArtifact,
+    SourceConfiguration,
+    record_review_decision,
+)
+
+
+class Command(BaseCommand):
+    help = "Approve one validated KAMIS recent-price generation for local Phase 0."
+
+    def add_arguments(self, parser: CommandParser) -> None:
+        parser.add_argument("--parse-run-id", required=True)
+        parser.add_argument("--decision-id", required=True)
+        parser.add_argument("--acceptance-evidence-sha256", required=True)
+
+    def handle(self, *args: object, **options: object) -> None:
+        del args
+        try:
+            parse_run_id = require_uuid(options.get("parse_run_id"))
+            decision_id = require_uuid(options.get("decision_id"))
+            acceptance_hash = require_sha256(options.get("acceptance_evidence_sha256"))
+            decision, created, actor_id = self._approve(
+                parse_run_id=parse_run_id,
+                decision_id=decision_id,
+                acceptance_hash=acceptance_hash,
+            )
+        except LocalPhase0Error as error:
+            raise CommandError(f"code={error.code.value}") from None
+        except Exception:
+            raise CommandError(f"code={LocalPhase0Code.REVIEW_FAILED.value}") from None
+
+        self.stdout.write(
+            " ".join(
+                (
+                    "status=APPROVED",
+                    f"decision_id={decision.id}",
+                    f"parse_run_id={decision.parse_run_id}",
+                    f"artifact_id={decision.source_artifact_id}",
+                    f"source_configuration_id={decision.source_configuration_id}",
+                    f"actor_id={actor_id}",
+                    f"created={'yes' if created else 'no'}",
+                )
+            )
+        )
+
+    @staticmethod
+    @transaction.atomic
+    def _approve(
+        *,
+        parse_run_id: uuid.UUID,
+        decision_id: uuid.UUID,
+        acceptance_hash: str,
+    ) -> tuple[ReviewDecision, bool, int]:
+        actor = get_local_operator(lock=True)
+        try:
+            parse_run = ParseRun.objects.select_for_update().get(pk=parse_run_id)
+            artifact = SourceArtifact.objects.select_for_update().get(pk=parse_run.artifact_id)
+        except ParseRun.DoesNotExist, SourceArtifact.DoesNotExist:
+            raise LocalPhase0Error(LocalPhase0Code.GENERATION_INVALID) from None
+
+        if (
+            parse_run.status != ParseRun.Status.VALIDATED
+            or parse_run.completed_at is None
+            or not parse_run.result_hash
+            or parse_run.quarantined_row_count != 0
+            or parse_run.artifact_id != artifact.id
+        ):
+            raise LocalPhase0Error(LocalPhase0Code.GENERATION_INVALID)
+
+        attempts = tuple(
+            FetchAttempt.objects.select_for_update()
+            .filter(artifact=artifact, state=FetchAttempt.State.SUCCEEDED)
+            .order_by("source_configuration_id", "started_at", "id")
+        )
+        source_ids = {attempt.source_configuration_id for attempt in attempts}
+        if len(source_ids) != 1:
+            raise LocalPhase0Error(LocalPhase0Code.GENERATION_INVALID)
+        source_id = next(iter(source_ids))
+        try:
+            source = SourceConfiguration.objects.select_for_update().get(pk=source_id)
+        except SourceConfiguration.DoesNotExist:
+            raise LocalPhase0Error(LocalPhase0Code.GENERATION_INVALID) from None
+        if (
+            source.state != SourceConfiguration.State.ACTIVE
+            or source.publication_mode != SourceConfiguration.PublicationMode.RECENT_COMPARISON
+            or artifact.source_identity != source.artifact_source_identity
+            or not source.coverage_identity
+            or not source.coverage_evidence_revision
+        ):
+            raise LocalPhase0Error(LocalPhase0Code.GENERATION_INVALID)
+
+        try:
+            decision, created = record_review_decision(
+                decision_id=decision_id,
+                actor=actor,
+                decision=ReviewDecision.Decision.APPROVE,
+                source_configuration_id=source.id,
+                source_artifact_id=artifact.id,
+                parse_run_id=parse_run.id,
+                reconciliation_report_sha256=parse_run.result_hash,
+                acceptance_evidence_sha256=acceptance_hash,
+                reason_code=LOCAL_APPROVAL_REASON_CODE,
+                approved_mode=SourceConfiguration.PublicationMode.RECENT_COMPARISON,
+                approved_coverage_identity=source.coverage_identity,
+                approved_coverage_evidence_revision=source.coverage_evidence_revision,
+            )
+        except Exception:
+            raise LocalPhase0Error(LocalPhase0Code.REVIEW_FAILED) from None
+        return decision, created, canonical_actor_id(actor)
diff --git a/grocery/management/commands/bootstrap_local_phase0_operator.py b/grocery/management/commands/bootstrap_local_phase0_operator.py
new file mode 100644
index 0000000..346f652
--- /dev/null
+++ b/grocery/management/commands/bootstrap_local_phase0_operator.py
@@ -0,0 +1,36 @@
+"""Bootstrap the fixed, local-only Phase 0 review and publication actor."""
+
+from __future__ import annotations
+
+from django.core.management.base import BaseCommand, CommandError
+
+from grocery.management.local_phase0 import (
+    LocalPhase0Code,
+    LocalPhase0Error,
+    bootstrap_local_operator,
+    canonical_actor_id,
+)
+
+
+class Command(BaseCommand):
+    help = "Create or validate the fixed local-only Phase 0 operator."
+
+    def handle(self, *args: object, **options: object) -> None:
+        del args, options
+        try:
+            actor, created = bootstrap_local_operator()
+            actor_id = canonical_actor_id(actor)
+        except LocalPhase0Error as error:
+            raise CommandError(f"code={error.code.value}") from None
+        except Exception:
+            raise CommandError(f"code={LocalPhase0Code.PERSISTENCE_FAILED.value}") from None
+
+        self.stdout.write(
+            " ".join(
+                (
+                    "status=READY",
+                    f"actor_id={actor_id}",
+                    f"created={'yes' if created else 'no'}",
+                )
+            )
+        )
diff --git a/grocery/management/commands/seal_recent_publication.py b/grocery/management/commands/seal_recent_publication.py
new file mode 100644
index 0000000..c13c693
--- /dev/null
+++ b/grocery/management/commands/seal_recent_publication.py
@@ -0,0 +1,73 @@
+"""Seal one reviewed generation into a bounded local publication revision."""
+
+from __future__ import annotations
+
+import uuid
+
+from django.core.management.base import BaseCommand, CommandError, CommandParser
+from django.db import transaction
+
+from grocery.management.local_phase0 import (
+    LocalPhase0Code,
+    LocalPhase0Error,
+    canonical_actor_id,
+    get_local_operator,
+    require_copy_revision,
+    require_uuid,
+)
+from grocery.models import PublicationRevision, seal_recent_publication
+
+
+class Command(BaseCommand):
+    help = "Seal one approved recent-price generation for local Phase 0."
+
+    def add_arguments(self, parser: CommandParser) -> None:
+        parser.add_argument("--decision-id", required=True)
+        parser.add_argument("--public-copy-revision", required=True)
+
+    def handle(self, *args: object, **options: object) -> None:
+        del args
+        try:
+            decision_id = require_uuid(options.get("decision_id"))
+            copy_revision = require_copy_revision(options.get("public_copy_revision"))
+            revision, created, actor_id = self._seal(
+                decision_id=decision_id,
+                copy_revision=copy_revision,
+            )
+        except LocalPhase0Error as error:
+            raise CommandError(f"code={error.code.value}") from None
+        except Exception:
+            raise CommandError(f"code={LocalPhase0Code.PUBLICATION_FAILED.value}") from None
+
+        self.stdout.write(
+            " ".join(
+                (
+                    "status=SEALED",
+                    f"publication_id={revision.id}",
+                    f"decision_id={revision.review_decision_id}",
+                    f"parse_run_id={revision.generation_id}",
+                    f"actor_id={actor_id}",
+                    f"created={'yes' if created else 'no'}",
+                )
+            )
+        )
+
+    @staticmethod
+    @transaction.atomic
+    def _seal(
+        *, decision_id: uuid.UUID, copy_revision: str
+    ) -> tuple[PublicationRevision, bool, int]:
+        actor = get_local_operator(lock=True)
+        existing_ids = set(
+            PublicationRevision.objects.select_for_update()
+            .filter(
+                review_decision_id=decision_id,
+                public_copy_revision=copy_revision,
+            )
+            .values_list("id", flat=True)
+        )
+        try:
+            revision = seal_recent_publication(decision_id, copy_revision)
+        except Exception:
+            raise LocalPhase0Error(LocalPhase0Code.PUBLICATION_FAILED) from None
+        return revision, revision.id not in existing_ids, canonical_actor_id(actor)
diff --git a/grocery/management/local_phase0.py b/grocery/management/local_phase0.py
new file mode 100644
index 0000000..e16898a
--- /dev/null
+++ b/grocery/management/local_phase0.py
@@ -0,0 +1,190 @@
+"""Fail-closed helpers for the local-only Phase 0 review and publication path."""
+
+from __future__ import annotations
+
+import re
+import uuid
+from enum import StrEnum
+from typing import Any, Final
+
+from django.conf import settings
+from django.contrib.auth import get_user_model
+from django.contrib.auth.models import Permission
+from django.db import transaction
+
+LOCAL_OPERATOR_USERNAME: Final = "phase0-local-operator"
+LOCAL_APPROVAL_REASON_CODE: Final = "LOCAL_PHASE0_SOURCE_GATE_APPROVED"
+PUBLIC_COPY_REVISIONS: Final = frozenset({"ko-v1", "ko-v2"})
+
+_SHA256 = re.compile(r"[0-9a-f]{64}\Z")
+_PERMISSION_SPECS: Final = frozenset(
+    {
+        ("grocery", "reviewdecision", "review_generation"),
+        ("grocery", "publicationactivation", "publish_publication"),
+    }
+)
+
+
+class LocalPhase0Code(StrEnum):
+    ENVIRONMENT_DENIED = "LOCAL_PHASE0_ENVIRONMENT_DENIED"
+    PERMISSION_MISSING = "LOCAL_PHASE0_PERMISSION_MISSING"
+    OPERATOR_MISSING = "LOCAL_PHASE0_OPERATOR_MISSING"
+    OPERATOR_CONFLICT = "LOCAL_PHASE0_OPERATOR_CONFLICT"
+    ACTOR_ID_INVALID = "LOCAL_PHASE0_ACTOR_ID_INVALID"
+    UUID_INVALID = "LOCAL_PHASE0_UUID_INVALID"
+    SHA256_INVALID = "LOCAL_PHASE0_SHA256_INVALID"
+    COPY_REVISION_INVALID = "LOCAL_PHASE0_COPY_REVISION_INVALID"
+    GENERATION_INVALID = "LOCAL_PHASE0_GENERATION_INVALID"
+    REVIEW_FAILED = "LOCAL_PHASE0_REVIEW_FAILED"
+    PUBLICATION_FAILED = "LOCAL_PHASE0_PUBLICATION_FAILED"
+    PERSISTENCE_FAILED = "LOCAL_PHASE0_PERSISTENCE_FAILED"
+
+
+class LocalPhase0Error(RuntimeError):
+    """A local command failure containing only one fixed operational code."""
+
+    def __init__(self, code: LocalPhase0Code) -> None:
+        self.code = code
+        super().__init__(code.value)
+
+
+def require_local_phase0_environment() -> None:
+    if settings.DEBUG is not True or getattr(settings, "ADMIN_ENABLED", None) is not False:
+        raise LocalPhase0Error(LocalPhase0Code.ENVIRONMENT_DENIED)
+
+
+def _parse_uuid(value: str) -> uuid.UUID:
+    try:
+        parsed = uuid.UUID(value)
+    except ValueError, AttributeError, TypeError:
+        raise LocalPhase0Error(LocalPhase0Code.UUID_INVALID) from None
+    if str(parsed) != value:
+        raise LocalPhase0Error(LocalPhase0Code.UUID_INVALID)
+    return parsed
+
+
+def require_uuid(value: object) -> uuid.UUID:
+    if isinstance(value, uuid.UUID):
+        return value
+    if isinstance(value, str):
+        return _parse_uuid(value)
+    raise LocalPhase0Error(LocalPhase0Code.UUID_INVALID)
+
+
+def require_sha256(value: object) -> str:
+    if isinstance(value, str) and _SHA256.fullmatch(value) is not None:
+        return value
+    raise LocalPhase0Error(LocalPhase0Code.SHA256_INVALID)
+
+
+def require_copy_revision(value: object) -> str:
+    if isinstance(value, str) and value in PUBLIC_COPY_REVISIONS:
+        return value
+    raise LocalPhase0Error(LocalPhase0Code.COPY_REVISION_INVALID)
+
+
+def _required_permissions(*, lock: bool) -> tuple[Permission, ...]:
+    query = Permission.objects.select_related("content_type").filter(
+        content_type__app_label="grocery",
+        content_type__model__in=("reviewdecision", "publicationactivation"),
+        codename__in=("review_generation", "publish_publication"),
+    )
+    if lock:
+        query = query.select_for_update()
+    permissions = tuple(query.order_by("content_type__model", "codename"))
+    actual = {
+        (
+            permission.content_type.app_label,
+            permission.content_type.model,
+            permission.codename,
+        )
+        for permission in permissions
+    }
+    if actual != _PERMISSION_SPECS:
+        raise LocalPhase0Error(LocalPhase0Code.PERMISSION_MISSING)
+    return permissions
+
+
+def _validate_operator_shape(actor: Any) -> None:
+    if (
+        getattr(actor, "username", None) != LOCAL_OPERATOR_USERNAME
+        or getattr(actor, "email", None) != ""
+        or getattr(actor, "first_name", None) != ""
+        or getattr(actor, "last_name", None) != ""
+        or getattr(actor, "is_active", None) is not True
+        or getattr(actor, "is_staff", None) is not False
+        or getattr(actor, "is_superuser", None) is not False
+        or actor.has_usable_password()
+        or actor.groups.exists()
+    ):
+        raise LocalPhase0Error(LocalPhase0Code.OPERATOR_CONFLICT)
+
+
+def _validate_exact_permissions(actor: Any, required: tuple[Permission, ...]) -> None:
+    expected_ids = {permission.id for permission in required}
+    actual_ids = set(actor.user_permissions.values_list("id", flat=True))
+    if actual_ids != expected_ids:
+        raise LocalPhase0Error(LocalPhase0Code.OPERATOR_CONFLICT)
+
+
+def canonical_actor_id(actor: Any) -> int:
+    actor_id = getattr(actor, "pk", None)
+    if not isinstance(actor_id, int) or isinstance(actor_id, bool) or actor_id < 1:
+        raise LocalPhase0Error(LocalPhase0Code.ACTOR_ID_INVALID)
+    return actor_id
+
+
+@transaction.atomic
+def bootstrap_local_operator() -> tuple[Any, bool]:
+    """Create or complete the exact non-login local actor without widening authority."""
+
+    require_local_phase0_environment()
+    required = _required_permissions(lock=True)
+    required_ids = {permission.id for permission in required}
+    user_model = get_user_model()
+    actor = (
+        user_model._default_manager.select_for_update()
+        .filter(username=LOCAL_OPERATOR_USERNAME)
+        .first()
+    )
+    created = actor is None
+    if actor is None:
+        actor = user_model(
+            username=LOCAL_OPERATOR_USERNAME,
+            email="",
+            first_name="",
+            last_name="",
+            is_active=True,
+            is_staff=False,
+            is_superuser=False,
+        )
+        actor.set_unusable_password()
+        actor.full_clean()
+        actor.save()
+    else:
+        _validate_operator_shape(actor)
+        current_ids = set(actor.user_permissions.values_list("id", flat=True))
+        if not current_ids.issubset(required_ids):
+            raise LocalPhase0Error(LocalPhase0Code.OPERATOR_CONFLICT)
+
+    actor.user_permissions.set(required)
+    _validate_operator_shape(actor)
+    _validate_exact_permissions(actor, required)
+    canonical_actor_id(actor)
+    return actor, created
+
+
+def get_local_operator(*, lock: bool) -> Any:
+    require_local_phase0_environment()
+    required = _required_permissions(lock=lock)
+    user_model = get_user_model()
+    query = user_model._default_manager.all()
+    if lock:
+        query = query.select_for_update()
+    actor = query.filter(username=LOCAL_OPERATOR_USERNAME).first()
+    if actor is None:
+        raise LocalPhase0Error(LocalPhase0Code.OPERATOR_MISSING)
+    _validate_operator_shape(actor)
+    _validate_exact_permissions(actor, required)
+    canonical_actor_id(actor)
+    return actor
diff --git a/grocery/tests/test_local_phase0_command_unit.py b/grocery/tests/test_local_phase0_command_unit.py
new file mode 100644
index 0000000..b8a6c61
--- /dev/null
+++ b/grocery/tests/test_local_phase0_command_unit.py
@@ -0,0 +1,243 @@
+"""Database-free command-boundary tests for local Phase 0 operations."""
+
+from __future__ import annotations
+
+import io
+import uuid
+from types import SimpleNamespace
+from unittest.mock import patch
+
+import pytest
+from django.core.management import call_command
+from django.core.management.base import CommandError
+from django.test import override_settings
+
+from grocery.management.local_phase0 import (
+    LocalPhase0Code,
+    LocalPhase0Error,
+    require_local_phase0_environment,
+)
+
+_APPROVAL_HASH = "7" * 64
+
+
+@override_settings(DEBUG=True, ADMIN_ENABLED=False)
+def test_local_environment_gate_accepts_only_debug_with_admin_disabled() -> None:
+    require_local_phase0_environment()
+
+
+@pytest.mark.parametrize(
+    ("debug", "admin_enabled"),
+    ((False, False), (True, True), (False, True)),
+)
+def test_local_environment_gate_denies_every_nonlocal_combination(
+    debug: bool,
+    admin_enabled: bool,
+) -> None:
+    with (
+        override_settings(DEBUG=debug, ADMIN_ENABLED=admin_enabled),
+        pytest.raises(LocalPhase0Error) as caught,
+    ):
+        require_local_phase0_environment()
+
+    assert caught.value.code is LocalPhase0Code.ENVIRONMENT_DENIED
+
+
+def test_bootstrap_command_receipt_contains_only_status_actor_id_and_created() -> None:
+    output = io.StringIO()
+    actor = SimpleNamespace(pk=17)
+    with (
+        patch(
+            "grocery.management.commands.bootstrap_local_phase0_operator.bootstrap_local_operator",
+            return_value=(actor, True),
+        ),
+        patch(
+            "grocery.management.commands.bootstrap_local_phase0_operator.canonical_actor_id",
+            return_value=17,
+        ),
+    ):
+        call_command("bootstrap_local_phase0_operator", stdout=output)
+
+    assert output.getvalue().strip() == "status=READY actor_id=17 created=yes"
+    assert "phase0-local-operator" not in output.getvalue()
+
+
+def test_bootstrap_arbitrary_failure_is_not_reflected() -> None:
+    marker = "private-environment-marker"
+    with (
+        patch(
+            "grocery.management.commands.bootstrap_local_phase0_operator.bootstrap_local_operator",
+            side_effect=RuntimeError(marker),
+        ),
+        pytest.raises(CommandError) as caught,
+    ):
+        call_command("bootstrap_local_phase0_operator")
+
+    assert str(caught.value) == "code=LOCAL_PHASE0_PERSISTENCE_FAILED"
+    assert marker not in str(caught.value)
+
+
+def test_approve_command_passes_validated_identifiers_and_emits_bounded_receipt() -> None:
+    output = io.StringIO()
+    parse_run_id = uuid.uuid4()
+    decision_id = uuid.uuid4()
+    artifact_id = uuid.uuid4()
+    source_id = uuid.uuid4()
+    decision = SimpleNamespace(
+        id=decision_id,
+        parse_run_id=parse_run_id,
+        source_artifact_id=artifact_id,
+        source_configuration_id=source_id,
+    )
+    with patch(
+        "grocery.management.commands.approve_recent_generation.Command._approve",
+        return_value=(decision, True, 19),
+    ) as approve:
+        call_command(
+            "approve_recent_generation",
+            parse_run_id=parse_run_id,
+            decision_id=decision_id,
+            acceptance_evidence_sha256=_APPROVAL_HASH,
+            stdout=output,
+        )
+
+    approve.assert_called_once_with(
+        parse_run_id=parse_run_id,
+        decision_id=decision_id,
+        acceptance_hash=_APPROVAL_HASH,
+    )
+    assert output.getvalue().strip() == " ".join(
+        (
+            "status=APPROVED",
+            f"decision_id={decision_id}",
+            f"parse_run_id={parse_run_id}",
+            f"artifact_id={artifact_id}",
+            f"source_configuration_id={source_id}",
+            "actor_id=19",
+            "created=yes",
+        )
+    )
+    assert _APPROVAL_HASH not in output.getvalue()
+
+
+@pytest.mark.parametrize(
+    ("parse_run_id", "decision_id", "acceptance_hash", "expected_code"),
+    (
+        ("private-parse-id", uuid.uuid4(), _APPROVAL_HASH, "LOCAL_PHASE0_UUID_INVALID"),
+        (uuid.uuid4(), "private-decision-id", _APPROVAL_HASH, "LOCAL_PHASE0_UUID_INVALID"),
+        (uuid.uuid4(), uuid.uuid4(), "private-hash", "LOCAL_PHASE0_SHA256_INVALID"),
+    ),
+)
+def test_approve_invalid_input_fails_without_echo_before_service(
+    parse_run_id: object,
+    decision_id: object,
+    acceptance_hash: object,
+    expected_code: str,
+) -> None:
+    with (
+        patch("grocery.management.commands.approve_recent_generation.Command._approve") as approve,
+        pytest.raises(CommandError) as caught,
+    ):
+        call_command(
+            "approve_recent_generation",
+            parse_run_id=parse_run_id,
+            decision_id=decision_id,
+            acceptance_evidence_sha256=acceptance_hash,
+        )
+
+    approve.assert_not_called()
+    assert str(caught.value) == f"code={expected_code}"
+    assert "private" not in str(caught.value)
+
+
+def test_approve_arbitrary_service_failure_is_not_reflected() -> None:
+    marker = "private-review-marker"
+    with (
+        patch(
+            "grocery.management.commands.approve_recent_generation.Command._approve",
+            side_effect=RuntimeError(marker),
+        ),
+        pytest.raises(CommandError) as caught,
+    ):
+        call_command(
+            "approve_recent_generation",
+            parse_run_id=uuid.uuid4(),
+            decision_id=uuid.uuid4(),
+            acceptance_evidence_sha256=_APPROVAL_HASH,
+        )
+
+    assert str(caught.value) == "code=LOCAL_PHASE0_REVIEW_FAILED"
+    assert marker not in str(caught.value)
+
+
+def test_seal_command_accepts_only_allowlisted_copy_and_emits_bounded_receipt() -> None:
+    output = io.StringIO()
+    decision_id = uuid.uuid4()
+    publication_id = uuid.uuid4()
+    parse_run_id = uuid.uuid4()
+    revision = SimpleNamespace(
+        id=publication_id,
+        review_decision_id=decision_id,
+        generation_id=parse_run_id,
+    )
+    with patch(
+        "grocery.management.commands.seal_recent_publication.Command._seal",
+        return_value=(revision, False, 23),
+    ) as seal:
+        call_command(
+            "seal_recent_publication",
+            decision_id=decision_id,
+            public_copy_revision="ko-v2",
+            stdout=output,
+        )
+
+    seal.assert_called_once_with(decision_id=decision_id, copy_revision="ko-v2")
+    assert output.getvalue().strip() == " ".join(
+        (
+            "status=SEALED",
+            f"publication_id={publication_id}",
+            f"decision_id={decision_id}",
+            f"parse_run_id={parse_run_id}",
+            "actor_id=23",
+            "created=no",
+        )
+    )
+    assert "ko-v2" not in output.getvalue()
+
+
+@pytest.mark.parametrize("copy_revision", ("ko-v3", "KO-V1", "private-copy"))
+def test_seal_rejects_nonallowlisted_copy_without_echo(
+    copy_revision: str,
+) -> None:
+    with (
+        patch("grocery.management.commands.seal_recent_publication.Command._seal") as seal,
+        pytest.raises(CommandError) as caught,
+    ):
+        call_command(
+            "seal_recent_publication",
+            decision_id=uuid.uuid4(),
+            public_copy_revision=copy_revision,
+        )
+
+    seal.assert_not_called()
+    assert str(caught.value) == "code=LOCAL_PHASE0_COPY_REVISION_INVALID"
+    assert copy_revision not in str(caught.value)
+
+
+def test_seal_arbitrary_service_failure_is_not_reflected() -> None:
+    marker = "private-publication-marker"
+    with (
+        patch(
+            "grocery.management.commands.seal_recent_publication.Command._seal",
+            side_effect=RuntimeError(marker),
+        ),
+        pytest.raises(CommandError) as caught,
+    ):
+        call_command(
+            "seal_recent_publication",
+            decision_id=uuid.uuid4(),
+            public_copy_revision="ko-v1",
+        )
+
+    assert str(caught.value) == "code=LOCAL_PHASE0_PUBLICATION_FAILED"
+    assert marker not in str(caught.value)
diff --git a/grocery/tests/test_local_phase0_commands.py b/grocery/tests/test_local_phase0_commands.py
new file mode 100644
index 0000000..85b8d5f
--- /dev/null
+++ b/grocery/tests/test_local_phase0_commands.py
@@ -0,0 +1,265 @@
+"""PostgreSQL integration tests for local review and publication commands."""
+
+from __future__ import annotations
+
+import io
+import uuid
+from typing import Any
+
+import pytest
+from django.contrib.auth import get_user_model
+from django.contrib.auth.models import Permission
+from django.core.management import call_command
+from django.core.management.base import CommandError
+from django.test import override_settings
+
+from grocery.management.local_phase0 import (
+    LOCAL_APPROVAL_REASON_CODE,
+    LOCAL_OPERATOR_USERNAME,
+)
+from grocery.models import PublicationRevision, ReviewDecision
+from grocery.tests.test_review_decision_models import complete_generation
+
+pytestmark = pytest.mark.django_db
+
+_ACCEPTANCE_HASH = "7" * 64
+
+
+def _run(name: str, **options: object) -> str:
+    output = io.StringIO()
+    call_command(name, stdout=output, **options)
+    return output.getvalue().strip()
+
+
+def _operator() -> Any:
+    return get_user_model()._default_manager.get(username=LOCAL_OPERATOR_USERNAME)
+
+
+def _permission_contract() -> set[tuple[str, str, str]]:
+    return {
+        ("grocery", "reviewdecision", "review_generation"),
+        ("grocery", "publicationactivation", "publish_publication"),
+    }
+
+
+@override_settings(DEBUG=True, ADMIN_ENABLED=False)
+def test_bootstrap_creates_exact_nonlogin_actor_and_replays_idempotently() -> None:
+    first = _run("bootstrap_local_phase0_operator")
+    second = _run("bootstrap_local_phase0_operator")
+
+    actor = _operator()
+    actual_permissions = set(
+        actor.user_permissions.values_list(
+            "content_type__app_label",
+            "content_type__model",
+            "codename",
+        )
+    )
+    assert first == f"status=READY actor_id={actor.pk} created=yes"
+    assert second == f"status=READY actor_id={actor.pk} created=no"
+    assert LOCAL_OPERATOR_USERNAME not in first
+    assert LOCAL_OPERATOR_USERNAME not in second
+    assert actor.email == actor.first_name == actor.last_name == ""
+    assert actor.is_active is True
+    assert actor.is_staff is False
+    assert actor.is_superuser is False
+    assert actor.has_usable_password() is False
+    assert actor.groups.count() == 0
+    assert actual_permissions == _permission_contract()
+
+
+@override_settings(DEBUG=True, ADMIN_ENABLED=False)
+def test_bootstrap_missing_publish_permission_fails_before_actor_creation() -> None:
+    Permission.objects.filter(
+        content_type__app_label="grocery",
+        content_type__model="publicationactivation",
+        codename="publish_publication",
+    ).delete()
+
+    with pytest.raises(CommandError) as caught:
+        _run("bootstrap_local_phase0_operator")
+
+    assert str(caught.value) == "code=LOCAL_PHASE0_PERMISSION_MISSING"
+    assert not get_user_model()._default_manager.filter(username=LOCAL_OPERATOR_USERNAME).exists()
+
+
+@override_settings(DEBUG=True, ADMIN_ENABLED=False)
+def test_bootstrap_existing_semantic_conflict_is_not_mutated() -> None:
+    actor = get_user_model()._default_manager.create_user(
+        username=LOCAL_OPERATOR_USERNAME,
+        email="",
+        password=None,
+        is_active=True,
+        is_staff=True,
+    )
+
+    with pytest.raises(CommandError) as caught:
+        _run("bootstrap_local_phase0_operator")
+
+    actor.refresh_from_db()
+    assert str(caught.value) == "code=LOCAL_PHASE0_OPERATOR_CONFLICT"
+    assert actor.is_staff is True
+    assert actor.user_permissions.count() == 0
+
+
+@pytest.mark.parametrize(
+    ("debug", "admin_enabled"),
+    ((False, False), (True, True)),
+)
+def test_bootstrap_environment_denial_writes_nothing(
+    debug: bool,
+    admin_enabled: bool,
+) -> None:
+    with (
+        override_settings(DEBUG=debug, ADMIN_ENABLED=admin_enabled),
+        pytest.raises(CommandError) as caught,
+    ):
+        _run("bootstrap_local_phase0_operator")
+
+    assert str(caught.value) == "code=LOCAL_PHASE0_ENVIRONMENT_DENIED"
+    assert not get_user_model()._default_manager.filter(username=LOCAL_OPERATOR_USERNAME).exists()
+
+
+@override_settings(DEBUG=True, ADMIN_ENABLED=False)
+def test_approve_locks_and_records_exact_generation_then_replays() -> None:
+    _run("bootstrap_local_phase0_operator")
+    source, parse_run = complete_generation()
+    decision_id = uuid.uuid4()
+
+    first = _run(
+        "approve_recent_generation",
+        parse_run_id=parse_run.id,
+        decision_id=decision_id,
+        acceptance_evidence_sha256=_ACCEPTANCE_HASH,
+    )
+    second = _run(
+        "approve_recent_generation",
+        parse_run_id=parse_run.id,
+        decision_id=decision_id,
+        acceptance_evidence_sha256=_ACCEPTANCE_HASH,
+    )
+
+    actor = _operator()
+    decision = ReviewDecision.objects.get(pk=decision_id)
+    expected_prefix = " ".join(
+        (
+            "status=APPROVED",
+            f"decision_id={decision_id}",
+            f"parse_run_id={parse_run.id}",
+            f"artifact_id={parse_run.artifact_id}",
+            f"source_configuration_id={source.id}",
+            f"actor_id={actor.pk}",
+        )
+    )
+    assert first == f"{expected_prefix} created=yes"
+    assert second == f"{expected_prefix} created=no"
+    assert LOCAL_OPERATOR_USERNAME not in first
+    assert _ACCEPTANCE_HASH not in first
+    assert decision.reviewer_id == actor.pk
+    assert decision.decision == ReviewDecision.Decision.APPROVE
+    assert decision.reconciliation_report_sha256 == parse_run.result_hash
+    assert decision.acceptance_evidence_sha256 == _ACCEPTANCE_HASH
+    assert decision.reason_code == LOCAL_APPROVAL_REASON_CODE
+    assert decision.approved_mode == source.publication_mode
+    assert decision.approved_coverage_identity == source.coverage_identity
+    assert decision.approved_coverage_evidence_revision == source.coverage_evidence_revision
+    assert ReviewDecision.objects.count() == 1
+
+
+@override_settings(DEBUG=True, ADMIN_ENABLED=False)
+def test_approve_uuid_replay_conflict_fails_without_mutating_decision() -> None:
+    _run("bootstrap_local_phase0_operator")
+    _source, parse_run = complete_generation()
+    decision_id = uuid.uuid4()
+    _run(
+        "approve_recent_generation",
+        parse_run_id=parse_run.id,
+        decision_id=decision_id,
+        acceptance_evidence_sha256=_ACCEPTANCE_HASH,
+    )
+
+    with pytest.raises(CommandError) as caught:
+        _run(
+            "approve_recent_generation",
+            parse_run_id=parse_run.id,
+            decision_id=decision_id,
+            acceptance_evidence_sha256="8" * 64,
+        )
+
+    assert str(caught.value) == "code=LOCAL_PHASE0_REVIEW_FAILED"
+    assert ReviewDecision.objects.get(pk=decision_id).acceptance_evidence_sha256 == (
+        _ACCEPTANCE_HASH
+    )
+
+
+@override_settings(DEBUG=True, ADMIN_ENABLED=False)
+def test_seal_calls_reviewed_publication_service_and_replays() -> None:
+    _run("bootstrap_local_phase0_operator")
+    _source, parse_run = complete_generation()
+    decision_id = uuid.uuid4()
+    _run(
+        "approve_recent_generation",
+        parse_run_id=parse_run.id,
+        decision_id=decision_id,
+        acceptance_evidence_sha256=_ACCEPTANCE_HASH,
+    )
+
+    first = _run(
+        "seal_recent_publication",
+        decision_id=decision_id,
+        public_copy_revision="ko-v1",
+    )
+    second = _run(
+        "seal_recent_publication",
+        decision_id=decision_id,
+        public_copy_revision="ko-v1",
+    )
+
+    actor = _operator()
+    revision = PublicationRevision.objects.get(review_decision_id=decision_id)
+    expected_prefix = " ".join(
+        (
+            "status=SEALED",
+            f"publication_id={revision.id}",
+            f"decision_id={decision_id}",
+            f"parse_run_id={parse_run.id}",
+            f"actor_id={actor.pk}",
+        )
+    )
+    assert first == f"{expected_prefix} created=yes"
+    assert second == f"{expected_prefix} created=no"
+    assert LOCAL_OPERATOR_USERNAME not in first
+    assert "ko-v1" not in first
+    assert revision.sealed_at is not None
+    assert revision.entries.count() == parse_run.accepted_row_count
+    assert PublicationRevision.objects.count() == 1
+
+
+@override_settings(DEBUG=True, ADMIN_ENABLED=False)
+def test_seal_refuses_operator_permission_drift_before_publication() -> None:
+    _run("bootstrap_local_phase0_operator")
+    _source, parse_run = complete_generation()
+    decision_id = uuid.uuid4()
+    _run(
+        "approve_recent_generation",
+        parse_run_id=parse_run.id,
+        decision_id=decision_id,
+        acceptance_evidence_sha256=_ACCEPTANCE_HASH,
+    )
+    actor = _operator()
+    publish_permission = Permission.objects.get(
+        content_type__app_label="grocery",
+        content_type__model="publicationactivation",
+        codename="publish_publication",
+    )
+    actor.user_permissions.remove(publish_permission)
+
+    with pytest.raises(CommandError) as caught:
+        _run(
+            "seal_recent_publication",
+            decision_id=decision_id,
+            public_copy_revision="ko-v1",
+        )
+
+    assert str(caught.value) == "code=LOCAL_PHASE0_OPERATOR_CONFLICT"
+    assert not PublicationRevision.objects.exists()


