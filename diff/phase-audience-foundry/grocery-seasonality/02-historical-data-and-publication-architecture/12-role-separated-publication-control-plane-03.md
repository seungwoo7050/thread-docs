## `feat(history): authorize publication operators`

diff --git a/grocery/management/control_plane.py b/grocery/management/control_plane.py
index 710d6fc..34ed1a0 100644
--- a/grocery/management/control_plane.py
+++ b/grocery/management/control_plane.py
@@ -37,14 +37,30 @@ CONTROL_TRANSITION_REASON_CODES: Final[dict[str, str]] = {
 ControlRole = Literal["reviewer", "publisher"]
 
 _RELEASE_SHA: Final = re.compile(r"[0-9a-f]{40}\Z")
-_ROLE_CONTRACTS: Final[dict[ControlRole, tuple[str, tuple[str, str, str]]]] = {
+PermissionSpec = tuple[str, str, str]
+
+_ROLE_CONTRACTS: Final[dict[ControlRole, tuple[str, tuple[PermissionSpec, ...]]]] = {
     "reviewer": (
         CONTROL_REVIEWER_USERNAME,
-        ("grocery", "reviewdecision", "review_generation"),
+        (
+            ("grocery", "reviewdecision", "review_generation"),
+            (
+                "grocery",
+                "historicalcollectionreviewdecision",
+                "review_historical_collection",
+            ),
+        ),
     ),
     "publisher": (
         CONTROL_PUBLISHER_USERNAME,
-        ("grocery", "publicationactivation", "publish_publication"),
+        (
+            ("grocery", "publicationactivation", "publish_publication"),
+            (
+                "grocery",
+                "historicalretailpublicationchannel",
+                "publish_historical_publication",
+            ),
+        ),
     ),
 }
 
@@ -123,22 +139,43 @@ def preflight_operation(expected_release_sha: object) -> bool:
     return False
 
 
-def _required_permission(role: ControlRole, *, lock: bool) -> Permission:
-    _username, (app_label, model, codename) = _ROLE_CONTRACTS[role]
+def _required_permissions(role: ControlRole, *, lock: bool) -> tuple[Permission, ...]:
+    _username, specs = _ROLE_CONTRACTS[role]
     query = Permission.objects.select_related("content_type").filter(
-        content_type__app_label=app_label,
-        content_type__model=model,
-        codename=codename,
+        content_type__app_label="grocery",
+        content_type__model__in=tuple(spec[1] for spec in specs),
+        codename__in=tuple(spec[2] for spec in specs),
     )
     if lock:
         query = query.select_for_update()
-    permissions = tuple(query)
-    if len(permissions) != 1:
+    permissions = tuple(query.order_by("content_type__model", "codename"))
+    actual = {
+        (
+            permission.content_type.app_label,
+            permission.content_type.model,
+            permission.codename,
+        )
+        for permission in permissions
+    }
+    if actual != set(specs):
         raise ControlPlaneError(ControlPlaneCode.PERMISSION_MISSING)
-    return permissions[0]
+    return permissions
+
+
+def _validate_actor_shape(
+    actor: Any,
+    *,
+    role: ControlRole,
+    permissions: tuple[Permission, ...],
+) -> None:
+    _validate_actor_identity(actor, role=role)
+    if set(actor.user_permissions.values_list("id", flat=True)) != {
+        permission.id for permission in permissions
+    }:
+        raise ControlPlaneError(ControlPlaneCode.ACTOR_CONFLICT)
 
 
-def _validate_actor_shape(actor: Any, *, role: ControlRole, permission: Permission) -> None:
+def _validate_actor_identity(actor: Any, *, role: ControlRole) -> None:
     username, _permission_spec = _ROLE_CONTRACTS[role]
     if (
         getattr(actor, "username", None) != username
@@ -150,7 +187,6 @@ def _validate_actor_shape(actor: Any, *, role: ControlRole, permission: Permissi
         or getattr(actor, "is_superuser", None) is not False
         or actor.has_usable_password()
         or actor.groups.exists()
-        or set(actor.user_permissions.values_list("id", flat=True)) != {permission.id}
     ):
         raise ControlPlaneError(ControlPlaneCode.ACTOR_CONFLICT)
 
@@ -163,7 +199,7 @@ def _control_actor_id(actor: Any) -> int:
 
 
 def _get_control_actor(*, role: ControlRole, lock: bool) -> Any:
-    permission = _required_permission(role, lock=lock)
+    permissions = _required_permissions(role, lock=lock)
     username, _permission_spec = _ROLE_CONTRACTS[role]
     user_model = get_user_model()
     query = user_model._default_manager.all()
@@ -172,7 +208,7 @@ def _get_control_actor(*, role: ControlRole, lock: bool) -> Any:
     actor = query.filter(username=username).first()
     if actor is None:
         raise ControlPlaneError(ControlPlaneCode.ACTOR_MISSING)
-    _validate_actor_shape(actor, role=role, permission=permission)
+    _validate_actor_shape(actor, role=role, permissions=permissions)
     _control_actor_id(actor)
     return actor
 
@@ -195,7 +231,7 @@ def resolve_operation_actor(
     return OperationActor(actor=actor, actor_id=actor_id, production=production)
 
 
-def _create_actor(*, role: ControlRole, permission: Permission) -> Any:
+def _create_actor(*, role: ControlRole, permissions: tuple[Permission, ...]) -> Any:
     username, _permission_spec = _ROLE_CONTRACTS[role]
     user_model = get_user_model()
     actor = user_model(
@@ -210,8 +246,8 @@ def _create_actor(*, role: ControlRole, permission: Permission) -> Any:
     actor.set_unusable_password()
     actor.full_clean()
     actor.save()
-    actor.user_permissions.set((permission,))
-    _validate_actor_shape(actor, role=role, permission=permission)
+    actor.user_permissions.set(permissions)
+    _validate_actor_shape(actor, role=role, permissions=permissions)
     _control_actor_id(actor)
     return actor
 
@@ -221,7 +257,7 @@ def bootstrap_control_plane_actors(expected_release_sha: object) -> Bootstrapped
     """Create both fixed actors once; any existing semantic drift fails atomically."""
 
     require_production_operation_environment(expected_release_sha)
-    permissions = {role: _required_permission(role, lock=True) for role in _ROLE_CONTRACTS}
+    permissions = {role: _required_permissions(role, lock=True) for role in _ROLE_CONTRACTS}
     user_model = get_user_model()
     existing = {
         actor.username: actor
@@ -236,9 +272,15 @@ def bootstrap_control_plane_actors(expected_release_sha: object) -> Bootstrapped
         actor = existing.get(username)
         was_created = actor is None
         if actor is None:
-            actor = _create_actor(role=role, permission=permissions[role])
+            actor = _create_actor(role=role, permissions=permissions[role])
         else:
-            _validate_actor_shape(actor, role=role, permission=permissions[role])
+            _validate_actor_identity(actor, role=role)
+            expected_ids = {permission.id for permission in permissions[role]}
+            current_ids = set(actor.user_permissions.values_list("id", flat=True))
+            if not current_ids.issubset(expected_ids):
+                raise ControlPlaneError(ControlPlaneCode.ACTOR_CONFLICT)
+            actor.user_permissions.set(permissions[role])
+            _validate_actor_shape(actor, role=role, permissions=permissions[role])
             _control_actor_id(actor)
         actors[role] = actor
         created[role] = was_created
diff --git a/grocery/management/local_phase0.py b/grocery/management/local_phase0.py
index ab49b36..7d38237 100644
--- a/grocery/management/local_phase0.py
+++ b/grocery/management/local_phase0.py
@@ -21,6 +21,16 @@ _PERMISSION_SPECS: Final = frozenset(
     {
         ("grocery", "reviewdecision", "review_generation"),
         ("grocery", "publicationactivation", "publish_publication"),
+        (
+            "grocery",
+            "historicalcollectionreviewdecision",
+            "review_historical_collection",
+        ),
+        (
+            "grocery",
+            "historicalretailpublicationchannel",
+            "publish_historical_publication",
+        ),
     }
 )
 
@@ -86,8 +96,8 @@ def require_copy_revision(value: object) -> str:
 def _required_permissions(*, lock: bool) -> tuple[Permission, ...]:
     query = Permission.objects.select_related("content_type").filter(
         content_type__app_label="grocery",
-        content_type__model__in=("reviewdecision", "publicationactivation"),
-        codename__in=("review_generation", "publish_publication"),
+        content_type__model__in=tuple(spec[1] for spec in _PERMISSION_SPECS),
+        codename__in=tuple(spec[2] for spec in _PERMISSION_SPECS),
     )
     if lock:
         query = query.select_for_update()
diff --git a/grocery/tests/test_control_plane_commands.py b/grocery/tests/test_control_plane_commands.py
index d87c93f..9d22c9b 100644
--- a/grocery/tests/test_control_plane_commands.py
+++ b/grocery/tests/test_control_plane_commands.py
@@ -189,9 +189,59 @@ def test_bootstrap_creates_exact_nonlogin_roles_and_replays_idempotently() -> No
         assert actor.is_superuser is False
         assert actor.has_usable_password() is False
         assert actor.groups.count() == 0
-    assert _permission_contract(reviewer) == {("grocery", "reviewdecision", "review_generation")}
+    assert _permission_contract(reviewer) == {
+        ("grocery", "reviewdecision", "review_generation"),
+        (
+            "grocery",
+            "historicalcollectionreviewdecision",
+            "review_historical_collection",
+        ),
+    }
     assert _permission_contract(publisher) == {
-        ("grocery", "publicationactivation", "publish_publication")
+        ("grocery", "publicationactivation", "publish_publication"),
+        (
+            "grocery",
+            "historicalretailpublicationchannel",
+            "publish_historical_publication",
+        ),
+    }
+
+
+@pytest.mark.django_db
+@override_settings(**_PRODUCTION_SETTINGS)
+def test_bootstrap_expands_existing_recent_only_actors_to_historical_roles() -> None:
+    contracts = (
+        (CONTROL_REVIEWER_USERNAME, "review_generation"),
+        (CONTROL_PUBLISHER_USERNAME, "publish_publication"),
+    )
+    for username, codename in contracts:
+        actor = get_user_model()._default_manager.create_user(
+            username=username,
+            password=None,
+            is_active=True,
+            is_staff=False,
+            is_superuser=False,
+        )
+        actor.user_permissions.add(Permission.objects.get(codename=codename))
+
+    receipt = _run("bootstrap_control_plane_actors", expected_release_sha=_RELEASE_SHA)
+
+    assert "review_created=no" in receipt and "publication_created=no" in receipt
+    assert _permission_contract(_user(CONTROL_REVIEWER_USERNAME)) == {
+        ("grocery", "reviewdecision", "review_generation"),
+        (
+            "grocery",
+            "historicalcollectionreviewdecision",
+            "review_historical_collection",
+        ),
+    }
+    assert _permission_contract(_user(CONTROL_PUBLISHER_USERNAME)) == {
+        ("grocery", "publicationactivation", "publish_publication"),
+        (
+            "grocery",
+            "historicalretailpublicationchannel",
+            "publish_historical_publication",
+        ),
     }
 
 
diff --git a/grocery/tests/test_local_phase0_commands.py b/grocery/tests/test_local_phase0_commands.py
index 85b8d5f..898ab4e 100644
--- a/grocery/tests/test_local_phase0_commands.py
+++ b/grocery/tests/test_local_phase0_commands.py
@@ -39,6 +39,16 @@ def _permission_contract() -> set[tuple[str, str, str]]:
     return {
         ("grocery", "reviewdecision", "review_generation"),
         ("grocery", "publicationactivation", "publish_publication"),
+        (
+            "grocery",
+            "historicalcollectionreviewdecision",
+            "review_historical_collection",
+        ),
+        (
+            "grocery",
+            "historicalretailpublicationchannel",
+            "publish_historical_publication",
+        ),
     }
 
 


## `feat(history): expose collection approval command`

diff --git a/grocery/management/commands/approve_historical_collection.py b/grocery/management/commands/approve_historical_collection.py
new file mode 100644
index 0000000..2254f1c
--- /dev/null
+++ b/grocery/management/commands/approve_historical_collection.py
@@ -0,0 +1,130 @@
+"""Record one human approval for a validated historical source collection."""
+
+from __future__ import annotations
+
+import uuid
+
+from django.conf import settings
+from django.core.management.base import BaseCommand, CommandError, CommandParser
+from django.db import transaction
+
+from grocery.historical_collection_models import HistoricalSourceCollection
+from grocery.historical_review_models import HistoricalCollectionReviewDecision
+from grocery.historical_reviews import record_historical_review_decision
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
+_NONE = "NONE"
+_LOCAL_REASON = "LOCAL_HISTORICAL_COLLECTION_APPROVED"
+_CONTROL_REASON = "CONTROL_PLANE_HISTORICAL_COLLECTION_APPROVED"
+
+
+def _optional_uuid(value: object) -> uuid.UUID | None:
+    return None if value == _NONE else require_uuid(value)
+
+
+class Command(BaseCommand):
+    help = (
+        "Approve one validated historical collection. Production requires an "
+        "external-MFA private job; the control-plane flag is not authentication."
+    )
+
+    def add_arguments(self, parser: CommandParser) -> None:
+        parser.add_argument("--collection-id", required=True)
+        parser.add_argument("--decision-id", required=True)
+        parser.add_argument("--reconciliation-report-sha256", required=True)
+        parser.add_argument("--acceptance-evidence-sha256", required=True)
+        parser.add_argument("--supersedes-decision", default=_NONE)
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
+            collection_id = require_uuid(options.get("collection_id"))
+            decision_id = require_uuid(options.get("decision_id"))
+            reconciliation_hash = require_sha256(
+                options.get("reconciliation_report_sha256")
+            )
+            acceptance_hash = require_sha256(options.get("acceptance_evidence_sha256"))
+            supersedes_id = _optional_uuid(options.get("supersedes_decision"))
+            decision, created, actor_id = self._approve(
+                collection_id=collection_id,
+                decision_id=decision_id,
+                reconciliation_hash=reconciliation_hash,
+                acceptance_hash=acceptance_hash,
+                supersedes_id=supersedes_id,
+                expected_release_sha=expected_release_sha,
+            )
+        except ControlPlaneError as error:
+            raise CommandError(f"code={error.code.value}") from None
+        except LocalPhase0Error as error:
+            raise CommandError(f"code={error.code.value}") from None
+        except Exception:
+            code = (
+                ControlPlaneCode.REVIEW_FAILED.value
+                if production
+                else LocalPhase0Code.REVIEW_FAILED.value
+            )
+            raise CommandError(f"code={code}") from None
+
+        receipt = [
+            "status=APPROVED",
+            f"decision_id={decision.id}",
+            f"collection_id={decision.collection_id}",
+            f"collection_kind={decision.collection.kind}",
+        ]
+        if not production:
+            receipt.append(f"actor_id={actor_id}")
+        receipt.append(f"created={'yes' if created else 'no'}")
+        self.stdout.write(" ".join(receipt))
+
+    @staticmethod
+    @transaction.atomic
+    def _approve(
+        *,
+        collection_id: uuid.UUID,
+        decision_id: uuid.UUID,
+        reconciliation_hash: str,
+        acceptance_hash: str,
+        supersedes_id: uuid.UUID | None,
+        expected_release_sha: object = None,
+    ) -> tuple[HistoricalCollectionReviewDecision, bool, int]:
+        authority = resolve_operation_actor(
+            role="reviewer",
+            expected_release_sha=expected_release_sha,
+            lock=True,
+        )
+        collection = (
+            HistoricalSourceCollection.objects.select_for_update()
+            .select_related("source_configuration")
+            .get(pk=collection_id)
+        )
+        decision, created = record_historical_review_decision(
+            decision_id=decision_id,
+            actor=authority.actor,
+            collection_id=collection.id,
+            decision=HistoricalCollectionReviewDecision.Decision.APPROVE,
+            reconciliation_report_sha256=reconciliation_hash,
+            acceptance_evidence_sha256=acceptance_hash,
+            reason_code=_CONTROL_REASON if authority.production else _LOCAL_REASON,
+            approved_result_sha256=collection.result_sha256,
+            approved_partition_manifest_sha256=collection.partition_manifest_sha256,
+            supersedes_id=supersedes_id,
+        )
+        return decision, created, authority.actor_id
diff --git a/grocery/tests/test_historical_publication_commands.py b/grocery/tests/test_historical_publication_commands.py
new file mode 100644
index 0000000..92ba6d2
--- /dev/null
+++ b/grocery/tests/test_historical_publication_commands.py
@@ -0,0 +1,72 @@
+from __future__ import annotations
+
+import io
+import uuid
+
+import pytest
+from django.core.management import call_command
+from django.core.management.base import CommandError
+from django.test import override_settings
+
+from grocery.historical_publication_models import HistoricalRetailPublicationRevision
+from grocery.management.commands.approve_historical_collection import (
+    Command as ApproveCommand,
+)
+from grocery.tests.historical_bundle_factory import create_reviewed_historical_bundle
+
+_HASH = "8" * 64
+
+
+def _run(command: str, **options: object) -> str:
+    output = io.StringIO()
+    call_command(command, stdout=output, **options)
+    return output.getvalue().strip()
+
+
+def test_historical_review_command_is_default_off_and_has_no_actor_override() -> None:
+    parser = ApproveCommand().create_parser("manage.py", "approve_historical_collection")
+    destinations = {action.dest for action in parser._actions}
+    assert "expected_release_sha" in destinations
+    assert not {"actor", "actor_id", "username"} & destinations
+
+    with (
+        override_settings(
+            DEBUG=False,
+            ADMIN_ENABLED=False,
+            QA_STATE_PREVIEWS_ENABLED=False,
+            CONTROL_PLANE_OPERATIONS_ENABLED=False,
+        ),
+        pytest.raises(CommandError, match="LOCAL_PHASE0_ENVIRONMENT_DENIED"),
+    ):
+        _run(
+            "approve_historical_collection",
+            collection_id=uuid.uuid4(),
+            decision_id=uuid.uuid4(),
+            reconciliation_report_sha256=_HASH,
+            acceptance_evidence_sha256=_HASH,
+        )
+
+@pytest.mark.django_db
+@override_settings(
+    DEBUG=True,
+    ADMIN_ENABLED=False,
+    QA_STATE_PREVIEWS_ENABLED=False,
+    CONTROL_PLANE_OPERATIONS_ENABLED=False,
+)
+def test_local_review_command_records_only_one_explicit_collection_decision() -> None:
+    _run("bootstrap_local_phase0_operator")
+    bundle = create_reviewed_historical_bundle()
+    collection = bundle.monthly_review.collection
+    replacement_id = uuid.uuid4()
+
+    approved = _run(
+        "approve_historical_collection",
+        collection_id=collection.id,
+        decision_id=replacement_id,
+        reconciliation_report_sha256=_HASH,
+        acceptance_evidence_sha256="9" * 64,
+        supersedes_decision=bundle.monthly_review.id,
+    )
+    assert f"decision_id={replacement_id}" in approved
+    assert "status=APPROVED" in approved and "created=yes" in approved
+    assert not HistoricalRetailPublicationRevision.objects.exists()


