## `feat(history): expose bundle seal command`

diff --git a/grocery/management/commands/seal_historical_publication.py b/grocery/management/commands/seal_historical_publication.py
new file mode 100644
index 0000000..1045019
--- /dev/null
+++ b/grocery/management/commands/seal_historical_publication.py
@@ -0,0 +1,111 @@
+"""Seal three independently reviewed historical collections."""
+
+from __future__ import annotations
+
+import uuid
+
+from django.conf import settings
+from django.core.management.base import BaseCommand, CommandError, CommandParser
+from django.db import transaction
+
+from grocery.historical_publication_models import HistoricalRetailPublicationRevision
+from grocery.historical_publications import seal_historical_publication
+from grocery.management.control_plane import (
+    ControlPlaneCode,
+    ControlPlaneError,
+    preflight_operation,
+    resolve_operation_actor,
+)
+from grocery.management.local_phase0 import (
+    LocalPhase0Code,
+    LocalPhase0Error,
+    require_sha256,
+    require_uuid,
+)
+
+
+class Command(BaseCommand):
+    help = (
+        "Seal one exact historical retail bundle. Production requires an external-MFA "
+        "private job; the control-plane flag is not authentication."
+    )
+
+    def add_arguments(self, parser: CommandParser) -> None:
+        parser.add_argument("--monthly-review-id", required=True)
+        parser.add_argument("--regional-review-id", required=True)
+        parser.add_argument("--market-review-id", required=True)
+        parser.add_argument("--compatibility-report-sha256", required=True)
+        parser.add_argument("--expected-release-sha")
+
+    def handle(self, *args: object, **options: object) -> None:
+        del args
+        expected_release_sha = options.get("expected_release_sha")
+        production = (
+            getattr(settings, "CONTROL_PLANE_OPERATIONS_ENABLED", False) is True
+            or expected_release_sha is not None
+        )
+        try:
+            preflight_operation(expected_release_sha)
+            revision, created, actor_id = self._seal(
+                monthly_review_id=require_uuid(options.get("monthly_review_id")),
+                regional_review_id=require_uuid(options.get("regional_review_id")),
+                market_review_id=require_uuid(options.get("market_review_id")),
+                compatibility_hash=require_sha256(
+                    options.get("compatibility_report_sha256")
+                ),
+                expected_release_sha=expected_release_sha,
+            )
+        except ControlPlaneError as error:
+            raise CommandError(f"code={error.code.value}") from None
+        except LocalPhase0Error as error:
+            raise CommandError(f"code={error.code.value}") from None
+        except Exception:
+            code = (
+                ControlPlaneCode.PUBLICATION_FAILED.value
+                if production
+                else LocalPhase0Code.PUBLICATION_FAILED.value
+            )
+            raise CommandError(f"code={code}") from None
+
+        receipt = [
+            "status=SEALED",
+            f"publication_id={revision.id}",
+            f"public_copy_revision={revision.public_copy_revision}",
+        ]
+        if not production:
+            receipt.append(f"actor_id={actor_id}")
+        receipt.append(f"created={'yes' if created else 'no'}")
+        self.stdout.write(" ".join(receipt))
+
+    @staticmethod
+    @transaction.atomic
+    def _seal(
+        *,
+        monthly_review_id: uuid.UUID,
+        regional_review_id: uuid.UUID,
+        market_review_id: uuid.UUID,
+        compatibility_hash: str,
+        expected_release_sha: object = None,
+    ) -> tuple[HistoricalRetailPublicationRevision, bool, int]:
+        authority = resolve_operation_actor(
+            role="publisher",
+            expected_release_sha=expected_release_sha,
+            lock=True,
+        )
+        existing_ids = set(
+            HistoricalRetailPublicationRevision.objects.select_for_update()
+            .filter(
+                monthly_review_id=monthly_review_id,
+                regional_review_id=regional_review_id,
+                market_review_id=market_review_id,
+                public_copy_revision=HistoricalRetailPublicationRevision.COPY_REVISION,
+            )
+            .values_list("id", flat=True)
+        )
+        revision = seal_historical_publication(
+            monthly_review_id=monthly_review_id,
+            regional_review_id=regional_review_id,
+            market_review_id=market_review_id,
+            compatibility_report_sha256=compatibility_hash,
+        )
+        return revision, revision.id not in existing_ids, authority.actor_id
diff --git a/grocery/tests/test_historical_publication_commands.py b/grocery/tests/test_historical_publication_commands.py
index 92ba6d2..4812d19 100644
--- a/grocery/tests/test_historical_publication_commands.py
+++ b/grocery/tests/test_historical_publication_commands.py
@@ -8,10 +8,12 @@ from django.core.management import call_command
 from django.core.management.base import CommandError
 from django.test import override_settings
 
+from grocery.historical_activation_models import HistoricalRetailPublicationChannel
 from grocery.historical_publication_models import HistoricalRetailPublicationRevision
 from grocery.management.commands.approve_historical_collection import (
     Command as ApproveCommand,
 )
+from grocery.management.commands.seal_historical_publication import Command as SealCommand
 from grocery.tests.historical_bundle_factory import create_reviewed_historical_bundle
 
 _HASH = "8" * 64
@@ -46,6 +48,14 @@ def test_historical_review_command_is_default_off_and_has_no_actor_override() ->
             acceptance_evidence_sha256=_HASH,
         )
 
+
+def test_historical_seal_command_has_release_lock_and_no_actor_override() -> None:
+    parser = SealCommand().create_parser("manage.py", "seal_historical_publication")
+    destinations = {action.dest for action in parser._actions}
+    assert "expected_release_sha" in destinations
+    assert not {"actor", "actor_id", "username"} & destinations
+
+
 @pytest.mark.django_db
 @override_settings(
     DEBUG=True,
@@ -70,3 +80,29 @@ def test_local_review_command_records_only_one_explicit_collection_decision() ->
     assert f"decision_id={replacement_id}" in approved
     assert "status=APPROVED" in approved and "created=yes" in approved
     assert not HistoricalRetailPublicationRevision.objects.exists()
+
+
+@pytest.mark.django_db
+@override_settings(
+    DEBUG=True,
+    ADMIN_ENABLED=False,
+    QA_STATE_PREVIEWS_ENABLED=False,
+    CONTROL_PLANE_OPERATIONS_ENABLED=False,
+)
+def test_local_seal_command_binds_three_reviews_without_activating() -> None:
+    _run("bootstrap_local_phase0_operator")
+    bundle = create_reviewed_historical_bundle()
+
+    sealed = _run(
+        "seal_historical_publication",
+        monthly_review_id=bundle.monthly_review.id,
+        regional_review_id=bundle.regional_review.id,
+        market_review_id=bundle.market_review.id,
+        compatibility_report_sha256="a" * 64,
+    )
+
+    revision = HistoricalRetailPublicationRevision.objects.get()
+    assert f"publication_id={revision.id}" in sealed
+    assert "status=SEALED" in sealed and "created=yes" in sealed
+    assert revision.sealed_at is not None
+    assert not HistoricalRetailPublicationChannel.objects.exists()


## `feat(history): expose CAS publication transitions`

diff --git a/grocery/management/commands/transition_historical_publication.py b/grocery/management/commands/transition_historical_publication.py
new file mode 100644
index 0000000..d6825a4
--- /dev/null
+++ b/grocery/management/commands/transition_historical_publication.py
@@ -0,0 +1,186 @@
+"""CAS transition for the independent historical retail publication channel."""
+
+from __future__ import annotations
+
+import re
+import uuid
+from enum import StrEnum
+
+from django.conf import settings
+from django.core.management.base import BaseCommand, CommandError, CommandParser
+from django.db import transaction
+
+from grocery.historical_activation_models import HistoricalRetailPublicationActivation
+from grocery.historical_activations import transition_historical_publication
+from grocery.management.control_plane import (
+    ControlPlaneCode,
+    ControlPlaneError,
+    preflight_operation,
+    resolve_operation_actor,
+)
+from grocery.management.local_phase0 import (
+    LocalPhase0Error,
+    require_sha256,
+    require_uuid,
+)
+
+_NONE = "NONE"
+_NONNEGATIVE_INTEGER = re.compile(r"(?:0|[1-9][0-9]*)\Z")
+_REASON_CODES = {
+    False: {
+        "ACTIVATE": "LOCAL_HISTORICAL_PUBLICATION_ACTIVATED",
+        "ROLLBACK": "LOCAL_HISTORICAL_PUBLICATION_ROLLED_BACK",
+        "WITHDRAW": "LOCAL_HISTORICAL_PUBLICATION_WITHDRAWN",
+    },
+    True: {
+        "ACTIVATE": "CONTROL_PLANE_HISTORICAL_PUBLICATION_ACTIVATED",
+        "ROLLBACK": "CONTROL_PLANE_HISTORICAL_PUBLICATION_ROLLED_BACK",
+        "WITHDRAW": "CONTROL_PLANE_HISTORICAL_PUBLICATION_WITHDRAWN",
+    },
+}
+_STATUS_CODES = {
+    "ACTIVATE": "ACTIVATED",
+    "ROLLBACK": "ROLLED_BACK",
+    "WITHDRAW": "WITHDRAWN",
+}
+
+
+class _Code(StrEnum):
+    INPUT_INVALID = "HISTORICAL_PUBLICATION_INPUT_INVALID"
+    TRANSITION_FAILED = "HISTORICAL_PUBLICATION_TRANSITION_FAILED"
+
+
+class _CommandFailure(RuntimeError):
+    def __init__(self, code: _Code) -> None:
+        self.code = code
+        super().__init__(code.value)
+
+
+def _operation(value: object) -> str:
+    if isinstance(value, str) and value in _STATUS_CODES:
+        return value
+    raise _CommandFailure(_Code.INPUT_INVALID)
+
+
+def _expected_version(value: object) -> int:
+    if isinstance(value, int) and not isinstance(value, bool):
+        parsed = value
+    elif isinstance(value, str) and _NONNEGATIVE_INTEGER.fullmatch(value):
+        parsed = int(value)
+    else:
+        raise _CommandFailure(_Code.INPUT_INVALID)
+    if parsed < 0 or parsed > (2**63) - 2:
+        raise _CommandFailure(_Code.INPUT_INVALID)
+    return parsed
+
+
+def _revision(value: object) -> uuid.UUID | None:
+    return None if value == _NONE else require_uuid(value)
+
+
+def _target(operation: str, value: object) -> uuid.UUID | None:
+    if operation == HistoricalRetailPublicationActivation.Operation.WITHDRAW:
+        if value not in (None, _NONE):
+            raise _CommandFailure(_Code.INPUT_INVALID)
+        return None
+    if value in (None, _NONE):
+        raise _CommandFailure(_Code.INPUT_INVALID)
+    return require_uuid(value)
+
+
+class Command(BaseCommand):
+    help = (
+        "Activate, roll back, or withdraw the historical retail publication. Production "
+        "requires an external-MFA private job; the control-plane flag is not authentication."
+    )
+
+    def add_arguments(self, parser: CommandParser) -> None:
+        parser.add_argument("--operation", required=True)
+        parser.add_argument("--operation-id", required=True)
+        parser.add_argument("--acceptance-evidence-sha256", required=True)
+        parser.add_argument("--expected-version", required=True)
+        parser.add_argument("--expected-current-revision", required=True)
+        parser.add_argument("--target-revision")
+        parser.add_argument("--expected-release-sha")
+
+    def handle(self, *args: object, **options: object) -> None:
+        del args
+        expected_release_sha = options.get("expected_release_sha")
+        production = (
+            getattr(settings, "CONTROL_PLANE_OPERATIONS_ENABLED", False) is True
+            or expected_release_sha is not None
+        )
+        try:
+            preflight_operation(expected_release_sha)
+            operation = _operation(options.get("operation"))
+            operation_id = require_uuid(options.get("operation_id"))
+            evidence_hash = require_sha256(options.get("acceptance_evidence_sha256"))
+            expected_version = _expected_version(options.get("expected_version"))
+            expected_current = _revision(options.get("expected_current_revision"))
+            target_revision = _target(operation, options.get("target_revision"))
+            activation, created, actor_id = self._transition(
+                operation_id=operation_id,
+                operation=operation,
+                target_revision_id=target_revision,
+                expected_current_revision_id=expected_current,
+                expected_version=expected_version,
+                reason_code=_REASON_CODES[production][operation],
+                evidence_hash=evidence_hash,
+                expected_release_sha=expected_release_sha,
+            )
+        except ControlPlaneError as error:
+            raise CommandError(f"code={error.code.value}") from None
+        except LocalPhase0Error as error:
+            raise CommandError(f"code={error.code.value}") from None
+        except _CommandFailure as error:
+            raise CommandError(f"code={error.code.value}") from None
+        except Exception:
+            code = (
+                ControlPlaneCode.TRANSITION_FAILED.value
+                if production
+                else _Code.TRANSITION_FAILED.value
+            )
+            raise CommandError(f"code={code}") from None
+
+        receipt = [
+            f"status={_STATUS_CODES[operation]}",
+            f"operation_id={activation.id}",
+            f"previous_revision_id={activation.previous_revision_id or _NONE}",
+            f"target_revision_id={activation.target_revision_id or _NONE}",
+        ]
+        if not production:
+            receipt.append(f"actor_id={actor_id}")
+        receipt.extend(
+            (f"resulting_version={activation.sequence}", f"created={'yes' if created else 'no'}")
+        )
+        self.stdout.write(" ".join(receipt))
+
+    @staticmethod
+    @transaction.atomic
+    def _transition(
+        *,
+        operation_id: uuid.UUID,
+        operation: str,
+        target_revision_id: uuid.UUID | None,
+        expected_current_revision_id: uuid.UUID | None,
+        expected_version: int,
+        reason_code: str,
+        evidence_hash: str,
+        expected_release_sha: object = None,
+    ) -> tuple[HistoricalRetailPublicationActivation, bool, int]:
+        authority = resolve_operation_actor(
+            role="publisher",
+            expected_release_sha=expected_release_sha,
+            lock=True,
+        )
+        activation, created = transition_historical_publication(
+            operation_id=operation_id,
+            actor=authority.actor,
+            operation=operation,
+            target_revision_id=target_revision_id,
+            expected_current_revision_id=expected_current_revision_id,
+            expected_version=expected_version,
+            reason_code=reason_code,
+            acceptance_evidence_sha256=evidence_hash,
+        )
+        return activation, created, authority.actor_id
diff --git a/grocery/tests/test_historical_publication_commands.py b/grocery/tests/test_historical_publication_commands.py
index 4812d19..14d10c2 100644
--- a/grocery/tests/test_historical_publication_commands.py
+++ b/grocery/tests/test_historical_publication_commands.py
@@ -14,6 +14,9 @@ from grocery.management.commands.approve_historical_collection import (
     Command as ApproveCommand,
 )
 from grocery.management.commands.seal_historical_publication import Command as SealCommand
+from grocery.management.commands.transition_historical_publication import (
+    Command as TransitionCommand,
+)
 from grocery.tests.historical_bundle_factory import create_reviewed_historical_bundle
 
 _HASH = "8" * 64
@@ -56,6 +59,32 @@ def test_historical_seal_command_has_release_lock_and_no_actor_override() -> Non
     assert not {"actor", "actor_id", "username"} & destinations
 
 
+@override_settings(
+    DEBUG=True,
+    ADMIN_ENABLED=False,
+    QA_STATE_PREVIEWS_ENABLED=False,
+    CONTROL_PLANE_OPERATIONS_ENABLED=False,
+)
+def test_historical_transition_command_rejects_negative_cas_version() -> None:
+    parser = TransitionCommand().create_parser(
+        "manage.py", "transition_historical_publication"
+    )
+    destinations = {action.dest for action in parser._actions}
+    assert "expected_release_sha" in destinations
+    assert not {"actor", "actor_id", "username"} & destinations
+
+    with pytest.raises(CommandError, match="HISTORICAL_PUBLICATION_INPUT_INVALID"):
+        _run(
+            "transition_historical_publication",
+            operation="ACTIVATE",
+            operation_id=uuid.uuid4(),
+            acceptance_evidence_sha256=_HASH,
+            expected_version=-1,
+            expected_current_revision="NONE",
+            target_revision=uuid.uuid4(),
+        )
+
+
 @pytest.mark.django_db
 @override_settings(
     DEBUG=True,
@@ -106,3 +135,37 @@ def test_local_seal_command_binds_three_reviews_without_activating() -> None:
     assert "status=SEALED" in sealed and "created=yes" in sealed
     assert revision.sealed_at is not None
     assert not HistoricalRetailPublicationChannel.objects.exists()
+
+
+@pytest.mark.django_db
+@override_settings(
+    DEBUG=True,
+    ADMIN_ENABLED=False,
+    QA_STATE_PREVIEWS_ENABLED=False,
+    CONTROL_PLANE_OPERATIONS_ENABLED=False,
+)
+def test_local_transition_command_activates_sealed_bundle_by_exact_cas() -> None:
+    _run("bootstrap_local_phase0_operator")
+    bundle = create_reviewed_historical_bundle()
+    _run(
+        "seal_historical_publication",
+        monthly_review_id=bundle.monthly_review.id,
+        regional_review_id=bundle.regional_review.id,
+        market_review_id=bundle.market_review.id,
+        compatibility_report_sha256="a" * 64,
+    )
+    revision = HistoricalRetailPublicationRevision.objects.get()
+
+    activated = _run(
+        "transition_historical_publication",
+        operation="ACTIVATE",
+        operation_id=uuid.uuid4(),
+        acceptance_evidence_sha256="b" * 64,
+        expected_version=0,
+        expected_current_revision="NONE",
+        target_revision=revision.id,
+    )
+
+    channel = HistoricalRetailPublicationChannel.objects.get()
+    assert "status=ACTIVATED" in activated and "resulting_version=1" in activated
+    assert channel.current_revision_id == revision.id


## `docs(history): record human publication checkpoint`

diff --git a/docs/OPERATIONS-RUNBOOK.md b/docs/OPERATIONS-RUNBOOK.md
index 25dbcd8..cc10861 100644
--- a/docs/OPERATIONS-RUNBOOK.md
+++ b/docs/OPERATIONS-RUNBOOK.md
@@ -505,8 +505,9 @@ job별 role-specific database credential·grant와 immutable audit를 별도로
 ingestion과 scheduler process에서는 항상 `0`이다.
 
 actor provisioning credential은 승인된 change에서 한 번만 다음 명령을 실행한다. 두 actor는
-로그인할 수 없고 PII·staff·superuser·group이 없으며 reviewer와 publisher가 각각 정확히 하나의
-Django permission만 가진다. 기존 actor에 drift가 있으면 둘 중 어느 것도 부분 수정하지 않는다.
+로그인할 수 없고 PII·staff·superuser·group이 없으며 reviewer와 publisher가 각각 recent와
+historical에서 자기 역할에 해당하는 Django permission만 가진다. 기존 actor에 drift가 있으면
+둘 중 어느 것도 부분 수정하지 않는다.
 
 ```sh
 .venv/bin/python manage.py bootstrap_control_plane_actors \
@@ -539,6 +540,65 @@ rehearsal reason과 분리된다.
   --expected-release-sha "$RELEASE_SHA"
 ```
 
+### historical collection review와 첫 publication
+
+`ingest_kamis_monthly`, `ingest_kamis_regional_daily`, `ingest_kamis_market_daily`는 각각 사람이
+승인한 exact source configuration·code manifest·partition 범위로 실행한다. 각 명령은 하나의
+완전한 `VALIDATED` collection에서 멈춘다. scheduler나 같은 job에서 review, seal 또는 activation을
+이어 실행하지 않는다. code manifest와 cross-source series·region·market mapping 등록은 독립적인
+사람 검토 checkpoint이며 command가 새 mapping을 추측하지 않는다.
+
+외부 MFA reviewer job은 세 collection을 각각 아래 명령으로 승인한다. 첫 decision의
+`--supersedes-decision`은 생략하고, 재검수 decision만 현재 tail UUID를 명시한다. evidence 원문은
+private 경계에 두고 canonical SHA-256만 전달한다.
+
+```sh
+.venv/bin/python manage.py approve_historical_collection \
+  --collection-id "$HISTORICAL_COLLECTION_ID" \
+  --decision-id "$HISTORICAL_REVIEW_ID" \
+  --reconciliation-report-sha256 "$RECONCILIATION_REPORT_SHA256" \
+  --acceptance-evidence-sha256 "$REVIEW_EVIDENCE_SHA256" \
+  --expected-release-sha "$RELEASE_SHA"
+```
+
+세 current APPROVE review가 준비된 뒤 publisher job이 별도 change에서 봉인하고, 결과 revision을
+검사한 뒤 다시 별도 change에서 exact current/version CAS로 활성화한다. 이 두 명령을 하나의
+자동 chain으로 묶지 않는다.
+
+```sh
+.venv/bin/python manage.py seal_historical_publication \
+  --monthly-review-id "$MONTHLY_REVIEW_ID" \
+  --regional-review-id "$REGIONAL_REVIEW_ID" \
+  --market-review-id "$MARKET_REVIEW_ID" \
+  --compatibility-report-sha256 "$COMPATIBILITY_REPORT_SHA256" \
+  --expected-release-sha "$RELEASE_SHA"
+
+.venv/bin/python manage.py transition_historical_publication \
+  --operation ACTIVATE \
+  --operation-id "$HISTORICAL_PUBLICATION_OPERATION_ID" \
+  --acceptance-evidence-sha256 "$HISTORICAL_PUBLICATION_EVIDENCE_SHA256" \
+  --expected-version "$EXPECTED_HISTORICAL_VERSION" \
+  --expected-current-revision "$EXPECTED_HISTORICAL_REVISION_OR_NONE" \
+  --target-revision "$HISTORICAL_PUBLICATION_REVISION_ID" \
+  --expected-release-sha "$RELEASE_SHA"
+```
+
+최초 activation은 version `0`과 current literal `NONE`을 사용한다. 같은 logical retry에만 같은
+decision·operation UUID를 재사용한다. `ACTIVATE`는 current review tail을 요구하지만,
+`ROLLBACK`은 activation history에서 previously-current였음이 확인된 sealed last-known-good를
+대상으로 한다. production actor provisioning, 첫 review·seal·activation, traffic 전환과 rollback은
+모두 사람 전용 checkpoint다.
+
+browser evidence fixture는 production command가 아니다. `DEBUG=1`, Admin 비활성, QA preview
+활성, loopback PostgreSQL, `grocery_vnext_`로 시작하는 비어 있는 DB를 모두 확인한 뒤에만 실행된다.
+핵심 source·domain·publication row가 하나라도 있으면 실패 폐쇄한다.
+
+```sh
+DJANGO_DEBUG=1 ADMIN_ENABLED=0 QA_STATE_PREVIEWS_ENABLED=1 \
+  DATABASE_URL="$DISPOSABLE_VNEXT_DATABASE_URL" \
+  .venv/bin/python scripts/build_vnext_browser_fixture.py
+```
+
 `PUBLIC_COPY_REVISION`은 현재 `ko-v1`, `ko-v2`, `ko-v3` 또는 `ko-v4`만 허용한다. `ko-v3`는
 초록장부 frontend redesign copy이고 `ko-v4`는 historical consumer 확장 copy다. 기존 revision
 row를 수정하지 않고 새 sealed revision으로만 만든다. production의 `ko-v4` seal·activation,
