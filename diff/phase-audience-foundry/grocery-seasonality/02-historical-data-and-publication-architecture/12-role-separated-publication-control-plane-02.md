## `feat(ops): gate production publication commands`

diff --git a/.env.example b/.env.example
index 5ea1ce6..f947ed8 100644
--- a/.env.example
+++ b/.env.example
@@ -14,4 +14,5 @@ DATABASE_CONN_MAX_AGE=60
 KAMIS_API_KEY=
 KAMIS_CONFIRMATION_MAX_AGE_HOURS=36
 QA_STATE_PREVIEWS_ENABLED=0
+CONTROL_PLANE_OPERATIONS_ENABLED=0
 DEPLOY_VERSION=0000000
diff --git a/config/settings.py b/config/settings.py
index e545cea..8da0f19 100644
--- a/config/settings.py
+++ b/config/settings.py
@@ -63,6 +63,7 @@ SECRET_KEY = os.environ.get("DJANGO_SECRET_KEY", "local-development-only-not-for
 ALLOWED_HOSTS = env_list("DJANGO_ALLOWED_HOSTS", "localhost,127.0.0.1,testserver")
 CSRF_TRUSTED_ORIGINS = env_list("DJANGO_CSRF_TRUSTED_ORIGINS", "")
 DEPLOY_VERSION = os.environ.get("DEPLOY_VERSION", "0000000")
+CONTROL_PLANE_OPERATIONS_ENABLED = env_bool("CONTROL_PLANE_OPERATIONS_ENABLED", False)
 
 
 def validate_production_environment(
diff --git a/grocery/management/commands/approve_recent_generation.py b/grocery/management/commands/approve_recent_generation.py
index fecd8e8..0b26bc1 100644
--- a/grocery/management/commands/approve_recent_generation.py
+++ b/grocery/management/commands/approve_recent_generation.py
@@ -4,15 +4,21 @@ from __future__ import annotations
 
 import uuid
 
+from django.conf import settings
 from django.core.management.base import BaseCommand, CommandError, CommandParser
 from django.db import transaction
 
+from grocery.management.control_plane import (
+    CONTROL_APPROVAL_REASON_CODE,
+    ControlPlaneCode,
+    ControlPlaneError,
+    preflight_operation,
+    resolve_operation_actor,
+)
 from grocery.management.local_phase0 import (
     LOCAL_APPROVAL_REASON_CODE,
     LocalPhase0Code,
     LocalPhase0Error,
-    canonical_actor_id,
-    get_local_operator,
     require_sha256,
     require_uuid,
 )
@@ -27,42 +33,66 @@ from grocery.models import (
 
 
 class Command(BaseCommand):
-    help = "Approve one validated KAMIS recent-price generation for local Phase 0."
+    help = (
+        "Approve one validated KAMIS recent-price generation. Production execution requires "
+        "an external-MFA private job; the control-plane flag is not authentication."
+    )
 
     def add_arguments(self, parser: CommandParser) -> None:
         parser.add_argument("--parse-run-id", required=True)
         parser.add_argument("--decision-id", required=True)
         parser.add_argument("--acceptance-evidence-sha256", required=True)
+        parser.add_argument("--expected-release-sha")
 
     def handle(self, *args: object, **options: object) -> None:
         del args
+        expected_release_sha = options.get("expected_release_sha")
+        production = (
+            getattr(settings, "CONTROL_PLANE_OPERATIONS_ENABLED", False) is True
+            or expected_release_sha is not None
+        )
         try:
+            if production:
+                preflight_operation(expected_release_sha)
             parse_run_id = require_uuid(options.get("parse_run_id"))
             decision_id = require_uuid(options.get("decision_id"))
             acceptance_hash = require_sha256(options.get("acceptance_evidence_sha256"))
-            decision, created, actor_id = self._approve(
-                parse_run_id=parse_run_id,
-                decision_id=decision_id,
-                acceptance_hash=acceptance_hash,
-            )
+            if production:
+                decision, created, actor_id = self._approve(
+                    parse_run_id=parse_run_id,
+                    decision_id=decision_id,
+                    acceptance_hash=acceptance_hash,
+                    expected_release_sha=expected_release_sha,
+                )
+            else:
+                decision, created, actor_id = self._approve(
+                    parse_run_id=parse_run_id,
+                    decision_id=decision_id,
+                    acceptance_hash=acceptance_hash,
+                )
+        except ControlPlaneError as error:
+            raise CommandError(f"code={error.code.value}") from None
         except LocalPhase0Error as error:
             raise CommandError(f"code={error.code.value}") from None
         except Exception:
-            raise CommandError(f"code={LocalPhase0Code.REVIEW_FAILED.value}") from None
-
-        self.stdout.write(
-            " ".join(
-                (
-                    "status=APPROVED",
-                    f"decision_id={decision.id}",
-                    f"parse_run_id={decision.parse_run_id}",
-                    f"artifact_id={decision.source_artifact_id}",
-                    f"source_configuration_id={decision.source_configuration_id}",
-                    f"actor_id={actor_id}",
-                    f"created={'yes' if created else 'no'}",
-                )
+            code = (
+                ControlPlaneCode.REVIEW_FAILED.value
+                if production
+                else LocalPhase0Code.REVIEW_FAILED.value
             )
-        )
+            raise CommandError(f"code={code}") from None
+
+        receipt = [
+            "status=APPROVED",
+            f"decision_id={decision.id}",
+            f"parse_run_id={decision.parse_run_id}",
+            f"artifact_id={decision.source_artifact_id}",
+            f"source_configuration_id={decision.source_configuration_id}",
+        ]
+        if not production:
+            receipt.append(f"actor_id={actor_id}")
+        receipt.append(f"created={'yes' if created else 'no'}")
+        self.stdout.write(" ".join(receipt))
 
     @staticmethod
     @transaction.atomic
@@ -71,8 +101,14 @@ class Command(BaseCommand):
         parse_run_id: uuid.UUID,
         decision_id: uuid.UUID,
         acceptance_hash: str,
+        expected_release_sha: object = None,
     ) -> tuple[ReviewDecision, bool, int]:
-        actor = get_local_operator(lock=True)
+        authority = resolve_operation_actor(
+            role="reviewer",
+            expected_release_sha=expected_release_sha,
+            lock=True,
+        )
+        actor = authority.actor
         try:
             parse_run = ParseRun.objects.select_for_update().get(pk=parse_run_id)
             artifact = SourceArtifact.objects.select_for_update().get(pk=parse_run.artifact_id)
@@ -120,11 +156,17 @@ class Command(BaseCommand):
                 parse_run_id=parse_run.id,
                 reconciliation_report_sha256=parse_run.result_hash,
                 acceptance_evidence_sha256=acceptance_hash,
-                reason_code=LOCAL_APPROVAL_REASON_CODE,
+                reason_code=(
+                    CONTROL_APPROVAL_REASON_CODE
+                    if authority.production
+                    else LOCAL_APPROVAL_REASON_CODE
+                ),
                 approved_mode=SourceConfiguration.PublicationMode.RECENT_COMPARISON,
                 approved_coverage_identity=source.coverage_identity,
                 approved_coverage_evidence_revision=source.coverage_evidence_revision,
             )
         except Exception:
+            if authority.production:
+                raise ControlPlaneError(ControlPlaneCode.REVIEW_FAILED) from None
             raise LocalPhase0Error(LocalPhase0Code.REVIEW_FAILED) from None
-        return decision, created, canonical_actor_id(actor)
+        return decision, created, authority.actor_id
diff --git a/grocery/management/commands/bootstrap_control_plane_actors.py b/grocery/management/commands/bootstrap_control_plane_actors.py
new file mode 100644
index 0000000..ba6521b
--- /dev/null
+++ b/grocery/management/commands/bootstrap_control_plane_actors.py
@@ -0,0 +1,42 @@
+"""Bootstrap fixed actors for an externally authenticated private production job."""
+
+from __future__ import annotations
+
+from django.core.management.base import BaseCommand, CommandError, CommandParser
+
+from grocery.management.control_plane import (
+    ControlPlaneCode,
+    ControlPlaneError,
+    bootstrap_control_plane_actors,
+)
+
+
+class Command(BaseCommand):
+    help = (
+        "Create or validate fixed production control-plane actors. The flag is not "
+        "authentication; run only behind external MFA/IAM and role-specific DB credentials."
+    )
+
+    def add_arguments(self, parser: CommandParser) -> None:
+        parser.add_argument("--expected-release-sha", required=True)
+
+    def handle(self, *args: object, **options: object) -> None:
+        del args
+        try:
+            actors = bootstrap_control_plane_actors(options.get("expected_release_sha"))
+        except ControlPlaneError as error:
+            raise CommandError(f"code={error.code.value}") from None
+        except Exception:
+            raise CommandError(f"code={ControlPlaneCode.PERSISTENCE_FAILED.value}") from None
+
+        self.stdout.write(
+            " ".join(
+                (
+                    "status=READY",
+                    "review_actor=READY",
+                    f"review_created={'yes' if actors.reviewer_created else 'no'}",
+                    "publication_actor=READY",
+                    f"publication_created={'yes' if actors.publisher_created else 'no'}",
+                )
+            )
+        )
diff --git a/grocery/management/commands/seal_recent_publication.py b/grocery/management/commands/seal_recent_publication.py
index c13c693..00ee509 100644
--- a/grocery/management/commands/seal_recent_publication.py
+++ b/grocery/management/commands/seal_recent_publication.py
@@ -4,14 +4,19 @@ from __future__ import annotations
 
 import uuid
 
+from django.conf import settings
 from django.core.management.base import BaseCommand, CommandError, CommandParser
 from django.db import transaction
 
+from grocery.management.control_plane import (
+    ControlPlaneCode,
+    ControlPlaneError,
+    preflight_operation,
+    resolve_operation_actor,
+)
 from grocery.management.local_phase0 import (
     LocalPhase0Code,
     LocalPhase0Error,
-    canonical_actor_id,
-    get_local_operator,
     require_copy_revision,
     require_uuid,
 )
@@ -19,45 +24,75 @@ from grocery.models import PublicationRevision, seal_recent_publication
 
 
 class Command(BaseCommand):
-    help = "Seal one approved recent-price generation for local Phase 0."
+    help = (
+        "Seal one approved recent-price generation. Production execution requires an "
+        "external-MFA private job; the control-plane flag is not authentication."
+    )
 
     def add_arguments(self, parser: CommandParser) -> None:
         parser.add_argument("--decision-id", required=True)
         parser.add_argument("--public-copy-revision", required=True)
+        parser.add_argument("--expected-release-sha")
 
     def handle(self, *args: object, **options: object) -> None:
         del args
+        expected_release_sha = options.get("expected_release_sha")
+        production = (
+            getattr(settings, "CONTROL_PLANE_OPERATIONS_ENABLED", False) is True
+            or expected_release_sha is not None
+        )
         try:
+            if production:
+                preflight_operation(expected_release_sha)
             decision_id = require_uuid(options.get("decision_id"))
             copy_revision = require_copy_revision(options.get("public_copy_revision"))
-            revision, created, actor_id = self._seal(
-                decision_id=decision_id,
-                copy_revision=copy_revision,
-            )
+            if production:
+                revision, created, actor_id = self._seal(
+                    decision_id=decision_id,
+                    copy_revision=copy_revision,
+                    expected_release_sha=expected_release_sha,
+                )
+            else:
+                revision, created, actor_id = self._seal(
+                    decision_id=decision_id,
+                    copy_revision=copy_revision,
+                )
+        except ControlPlaneError as error:
+            raise CommandError(f"code={error.code.value}") from None
         except LocalPhase0Error as error:
             raise CommandError(f"code={error.code.value}") from None
         except Exception:
-            raise CommandError(f"code={LocalPhase0Code.PUBLICATION_FAILED.value}") from None
-
-        self.stdout.write(
-            " ".join(
-                (
-                    "status=SEALED",
-                    f"publication_id={revision.id}",
-                    f"decision_id={revision.review_decision_id}",
-                    f"parse_run_id={revision.generation_id}",
-                    f"actor_id={actor_id}",
-                    f"created={'yes' if created else 'no'}",
-                )
+            code = (
+                ControlPlaneCode.PUBLICATION_FAILED.value
+                if production
+                else LocalPhase0Code.PUBLICATION_FAILED.value
             )
-        )
+            raise CommandError(f"code={code}") from None
+
+        receipt = [
+            "status=SEALED",
+            f"publication_id={revision.id}",
+            f"decision_id={revision.review_decision_id}",
+            f"parse_run_id={revision.generation_id}",
+        ]
+        if not production:
+            receipt.append(f"actor_id={actor_id}")
+        receipt.append(f"created={'yes' if created else 'no'}")
+        self.stdout.write(" ".join(receipt))
 
     @staticmethod
     @transaction.atomic
     def _seal(
-        *, decision_id: uuid.UUID, copy_revision: str
+        *,
+        decision_id: uuid.UUID,
+        copy_revision: str,
+        expected_release_sha: object = None,
     ) -> tuple[PublicationRevision, bool, int]:
-        actor = get_local_operator(lock=True)
+        authority = resolve_operation_actor(
+            role="publisher",
+            expected_release_sha=expected_release_sha,
+            lock=True,
+        )
         existing_ids = set(
             PublicationRevision.objects.select_for_update()
             .filter(
@@ -69,5 +104,7 @@ class Command(BaseCommand):
         try:
             revision = seal_recent_publication(decision_id, copy_revision)
         except Exception:
+            if authority.production:
+                raise ControlPlaneError(ControlPlaneCode.PUBLICATION_FAILED) from None
             raise LocalPhase0Error(LocalPhase0Code.PUBLICATION_FAILED) from None
-        return revision, revision.id not in existing_ids, canonical_actor_id(actor)
+        return revision, revision.id not in existing_ids, authority.actor_id
diff --git a/grocery/management/commands/transition_recent_publication.py b/grocery/management/commands/transition_recent_publication.py
index 12ffe66..5b3b7ee 100644
--- a/grocery/management/commands/transition_recent_publication.py
+++ b/grocery/management/commands/transition_recent_publication.py
@@ -7,13 +7,19 @@ import uuid
 from enum import StrEnum
 from typing import Final
 
+from django.conf import settings
 from django.core.management.base import BaseCommand, CommandError, CommandParser
 from django.db import transaction
 
+from grocery.management.control_plane import (
+    CONTROL_TRANSITION_REASON_CODES,
+    ControlPlaneCode,
+    ControlPlaneError,
+    preflight_operation,
+    resolve_operation_actor,
+)
 from grocery.management.local_phase0 import (
     LocalPhase0Error,
-    canonical_actor_id,
-    get_local_operator,
     require_sha256,
     require_uuid,
 )
@@ -97,7 +103,10 @@ def _revision_text(value: uuid.UUID | None) -> str:
 
 
 class Command(BaseCommand):
-    help = "Activate, roll back, or withdraw the local recent-retail publication pointer."
+    help = (
+        "Activate, roll back, or withdraw the recent-retail publication pointer. Production "
+        "requires an external-MFA private job; the control-plane flag is not authentication."
+    )
 
     def add_arguments(self, parser: CommandParser) -> None:
         parser.add_argument("--operation", required=True)
@@ -106,10 +115,18 @@ class Command(BaseCommand):
         parser.add_argument("--expected-version", required=True)
         parser.add_argument("--expected-current-revision", required=True)
         parser.add_argument("--target-revision")
+        parser.add_argument("--expected-release-sha")
 
     def handle(self, *args: object, **options: object) -> None:
         del args
+        expected_release_sha = options.get("expected_release_sha")
+        production = (
+            getattr(settings, "CONTROL_PLANE_OPERATIONS_ENABLED", False) is True
+            or expected_release_sha is not None
+        )
         try:
+            if production:
+                preflight_operation(expected_release_sha)
             operation = _operation(options.get("operation"))
             operation_id = require_uuid(options.get("operation_id"))
             evidence_hash = require_sha256(options.get("acceptance_evidence_sha256"))
@@ -119,35 +136,56 @@ class Command(BaseCommand):
                 invalid_code=_TransitionCode.EXPECTED_CURRENT_INVALID,
             )
             target = _target_revision(operation, options.get("target_revision"))
-            activation, created, actor_id = self._transition(
-                operation_id=operation_id,
-                operation=operation,
-                target_revision_id=target,
-                expected_current_revision_id=expected_current,
-                expected_version=expected_version,
-                reason_code=_REASON_CODES[operation],
-                evidence_hash=evidence_hash,
-            )
+            if production:
+                activation, created, actor_id = self._transition(
+                    operation_id=operation_id,
+                    operation=operation,
+                    target_revision_id=target,
+                    expected_current_revision_id=expected_current,
+                    expected_version=expected_version,
+                    reason_code=CONTROL_TRANSITION_REASON_CODES[operation],
+                    evidence_hash=evidence_hash,
+                    expected_release_sha=expected_release_sha,
+                )
+            else:
+                activation, created, actor_id = self._transition(
+                    operation_id=operation_id,
+                    operation=operation,
+                    target_revision_id=target,
+                    expected_current_revision_id=expected_current,
+                    expected_version=expected_version,
+                    reason_code=_REASON_CODES[operation],
+                    evidence_hash=evidence_hash,
+                )
+        except ControlPlaneError as error:
+            raise CommandError(f"code={error.code.value}") from None
         except LocalPhase0Error as error:
             raise CommandError(f"code={error.code.value}") from None
         except _TransitionError as error:
             raise CommandError(f"code={error.code.value}") from None
         except Exception:
-            raise CommandError(f"code={_TransitionCode.TRANSITION_FAILED.value}") from None
-
-        self.stdout.write(
-            " ".join(
-                (
-                    f"status={_STATUS_CODES[operation]}",
-                    f"operation_id={activation.id}",
-                    f"previous_revision_id={_revision_text(activation.previous_revision_id)}",
-                    f"target_revision_id={_revision_text(activation.target_revision_id)}",
-                    f"actor_id={actor_id}",
-                    f"resulting_version={activation.sequence}",
-                    f"created={'yes' if created else 'no'}",
-                )
+            code = (
+                ControlPlaneCode.TRANSITION_FAILED.value
+                if production
+                else _TransitionCode.TRANSITION_FAILED.value
+            )
+            raise CommandError(f"code={code}") from None
+
+        receipt = [
+            f"status={_STATUS_CODES[operation]}",
+            f"operation_id={activation.id}",
+            f"previous_revision_id={_revision_text(activation.previous_revision_id)}",
+            f"target_revision_id={_revision_text(activation.target_revision_id)}",
+        ]
+        if not production:
+            receipt.append(f"actor_id={actor_id}")
+        receipt.extend(
+            (
+                f"resulting_version={activation.sequence}",
+                f"created={'yes' if created else 'no'}",
             )
         )
+        self.stdout.write(" ".join(receipt))
 
     @staticmethod
     @transaction.atomic
@@ -160,12 +198,17 @@ class Command(BaseCommand):
         expected_version: int,
         reason_code: str,
         evidence_hash: str,
+        expected_release_sha: object = None,
     ) -> tuple[PublicationActivation, bool, int]:
-        actor = get_local_operator(lock=True)
+        authority = resolve_operation_actor(
+            role="publisher",
+            expected_release_sha=expected_release_sha,
+            lock=True,
+        )
         try:
             activation, created = transition_recent_publication(
                 operation_id=operation_id,
-                actor=actor,
+                actor=authority.actor,
                 operation=operation,
                 target_revision_id=target_revision_id,
                 expected_current_revision_id=expected_current_revision_id,
@@ -174,5 +217,7 @@ class Command(BaseCommand):
                 acceptance_evidence_sha256=evidence_hash,
             )
         except Exception:
+            if authority.production:
+                raise ControlPlaneError(ControlPlaneCode.TRANSITION_FAILED) from None
             raise _TransitionError(_TransitionCode.TRANSITION_FAILED) from None
-        return activation, created, canonical_actor_id(actor)
+        return activation, created, authority.actor_id
diff --git a/grocery/management/control_plane.py b/grocery/management/control_plane.py
new file mode 100644
index 0000000..710d6fc
--- /dev/null
+++ b/grocery/management/control_plane.py
@@ -0,0 +1,251 @@
+"""Fail-closed identities and release lock for private production operations.
+
+This module is not an authentication boundary.  The enable flag and expected-release
+check only prevent accidental execution by the wrong release.  A production platform
+must put these commands behind external MFA/IAM and provision separate role-specific
+database credentials.  The complete database grant matrix remains a production-platform
+checkpoint; application-level permission checks here are defense in depth.
+"""
+
+from __future__ import annotations
+
+import re
+from dataclasses import dataclass
+from enum import StrEnum
+from typing import Any, Final, Literal
+
+from django.conf import settings
+from django.contrib.auth import get_user_model
+from django.contrib.auth.models import Permission
+from django.db import transaction
+
+from grocery.management.local_phase0 import (
+    canonical_actor_id,
+    get_local_operator,
+    require_local_phase0_environment,
+)
+
+CONTROL_REVIEWER_USERNAME: Final = "grocery-control-reviewer"
+CONTROL_PUBLISHER_USERNAME: Final = "grocery-control-publisher"
+CONTROL_APPROVAL_REASON_CODE: Final = "CONTROL_PLANE_SOURCE_GATE_APPROVED"
+CONTROL_TRANSITION_REASON_CODES: Final[dict[str, str]] = {
+    "ACTIVATE": "CONTROL_PLANE_PUBLICATION_ACTIVATED",
+    "ROLLBACK": "CONTROL_PLANE_PUBLICATION_ROLLED_BACK",
+    "WITHDRAW": "CONTROL_PLANE_PUBLICATION_WITHDRAWN",
+}
+
+ControlRole = Literal["reviewer", "publisher"]
+
+_RELEASE_SHA: Final = re.compile(r"[0-9a-f]{40}\Z")
+_ROLE_CONTRACTS: Final[dict[ControlRole, tuple[str, tuple[str, str, str]]]] = {
+    "reviewer": (
+        CONTROL_REVIEWER_USERNAME,
+        ("grocery", "reviewdecision", "review_generation"),
+    ),
+    "publisher": (
+        CONTROL_PUBLISHER_USERNAME,
+        ("grocery", "publicationactivation", "publish_publication"),
+    ),
+}
+
+
+class ControlPlaneCode(StrEnum):
+    ENVIRONMENT_DENIED = "CONTROL_PLANE_ENVIRONMENT_DENIED"
+    RELEASE_SHA_INVALID = "CONTROL_PLANE_RELEASE_SHA_INVALID"
+    RELEASE_SHA_MISMATCH = "CONTROL_PLANE_RELEASE_SHA_MISMATCH"
+    PERMISSION_MISSING = "CONTROL_PLANE_PERMISSION_MISSING"
+    ACTOR_MISSING = "CONTROL_PLANE_ACTOR_MISSING"
+    ACTOR_CONFLICT = "CONTROL_PLANE_ACTOR_CONFLICT"
+    ACTOR_ID_INVALID = "CONTROL_PLANE_ACTOR_ID_INVALID"
+    REVIEW_FAILED = "CONTROL_PLANE_REVIEW_FAILED"
+    PUBLICATION_FAILED = "CONTROL_PLANE_PUBLICATION_FAILED"
+    TRANSITION_FAILED = "CONTROL_PLANE_TRANSITION_FAILED"
+    PERSISTENCE_FAILED = "CONTROL_PLANE_PERSISTENCE_FAILED"
+
+
+class ControlPlaneError(RuntimeError):
+    """A production command failure containing only one fixed operational code."""
+
+    def __init__(self, code: ControlPlaneCode) -> None:
+        self.code = code
+        super().__init__(code.value)
+
+
+@dataclass(frozen=True, slots=True)
+class OperationActor:
+    actor: Any
+    actor_id: int
+    production: bool
+
+
+@dataclass(frozen=True, slots=True)
+class BootstrappedActors:
+    reviewer: Any
+    reviewer_created: bool
+    publisher: Any
+    publisher_created: bool
+
+
+def require_production_operation_environment(expected_release_sha: object) -> None:
+    """Require the explicit private-job flag and an exact running release lock."""
+
+    if (
+        settings.DEBUG is not False
+        or getattr(settings, "ADMIN_ENABLED", None) is not False
+        or getattr(settings, "QA_STATE_PREVIEWS_ENABLED", None) is not False
+        or getattr(settings, "CONTROL_PLANE_OPERATIONS_ENABLED", None) is not True
+    ):
+        raise ControlPlaneError(ControlPlaneCode.ENVIRONMENT_DENIED)
+
+    deploy_version = getattr(settings, "DEPLOY_VERSION", None)
+    if (
+        not isinstance(deploy_version, str)
+        or _RELEASE_SHA.fullmatch(deploy_version) is None
+        or not isinstance(expected_release_sha, str)
+        or _RELEASE_SHA.fullmatch(expected_release_sha) is None
+    ):
+        raise ControlPlaneError(ControlPlaneCode.RELEASE_SHA_INVALID)
+    if expected_release_sha != deploy_version:
+        raise ControlPlaneError(ControlPlaneCode.RELEASE_SHA_MISMATCH)
+
+
+def preflight_operation(expected_release_sha: object) -> bool:
+    """Validate the active local or production command boundary without touching data."""
+
+    production_requested = (
+        getattr(settings, "CONTROL_PLANE_OPERATIONS_ENABLED", False) is True
+        or expected_release_sha is not None
+    )
+    if production_requested:
+        require_production_operation_environment(expected_release_sha)
+        return True
+    require_local_phase0_environment()
+    return False
+
+
+def _required_permission(role: ControlRole, *, lock: bool) -> Permission:
+    _username, (app_label, model, codename) = _ROLE_CONTRACTS[role]
+    query = Permission.objects.select_related("content_type").filter(
+        content_type__app_label=app_label,
+        content_type__model=model,
+        codename=codename,
+    )
+    if lock:
+        query = query.select_for_update()
+    permissions = tuple(query)
+    if len(permissions) != 1:
+        raise ControlPlaneError(ControlPlaneCode.PERMISSION_MISSING)
+    return permissions[0]
+
+
+def _validate_actor_shape(actor: Any, *, role: ControlRole, permission: Permission) -> None:
+    username, _permission_spec = _ROLE_CONTRACTS[role]
+    if (
+        getattr(actor, "username", None) != username
+        or getattr(actor, "email", None) != ""
+        or getattr(actor, "first_name", None) != ""
+        or getattr(actor, "last_name", None) != ""
+        or getattr(actor, "is_active", None) is not True
+        or getattr(actor, "is_staff", None) is not False
+        or getattr(actor, "is_superuser", None) is not False
+        or actor.has_usable_password()
+        or actor.groups.exists()
+        or set(actor.user_permissions.values_list("id", flat=True)) != {permission.id}
+    ):
+        raise ControlPlaneError(ControlPlaneCode.ACTOR_CONFLICT)
+
+
+def _control_actor_id(actor: Any) -> int:
+    actor_id = getattr(actor, "pk", None)
+    if not isinstance(actor_id, int) or isinstance(actor_id, bool) or actor_id < 1:
+        raise ControlPlaneError(ControlPlaneCode.ACTOR_ID_INVALID)
+    return actor_id
+
+
+def _get_control_actor(*, role: ControlRole, lock: bool) -> Any:
+    permission = _required_permission(role, lock=lock)
+    username, _permission_spec = _ROLE_CONTRACTS[role]
+    user_model = get_user_model()
+    query = user_model._default_manager.all()
+    if lock:
+        query = query.select_for_update()
+    actor = query.filter(username=username).first()
+    if actor is None:
+        raise ControlPlaneError(ControlPlaneCode.ACTOR_MISSING)
+    _validate_actor_shape(actor, role=role, permission=permission)
+    _control_actor_id(actor)
+    return actor
+
+
+def resolve_operation_actor(
+    *,
+    role: ControlRole,
+    expected_release_sha: object,
+    lock: bool,
+) -> OperationActor:
+    """Select only the fixed actor for this environment and operation role."""
+
+    production = preflight_operation(expected_release_sha)
+    if production:
+        actor = _get_control_actor(role=role, lock=lock)
+        actor_id = _control_actor_id(actor)
+    else:
+        actor = get_local_operator(lock=lock)
+        actor_id = canonical_actor_id(actor)
+    return OperationActor(actor=actor, actor_id=actor_id, production=production)
+
+
+def _create_actor(*, role: ControlRole, permission: Permission) -> Any:
+    username, _permission_spec = _ROLE_CONTRACTS[role]
+    user_model = get_user_model()
+    actor = user_model(
+        username=username,
+        email="",
+        first_name="",
+        last_name="",
+        is_active=True,
+        is_staff=False,
+        is_superuser=False,
+    )
+    actor.set_unusable_password()
+    actor.full_clean()
+    actor.save()
+    actor.user_permissions.set((permission,))
+    _validate_actor_shape(actor, role=role, permission=permission)
+    _control_actor_id(actor)
+    return actor
+
+
+@transaction.atomic
+def bootstrap_control_plane_actors(expected_release_sha: object) -> BootstrappedActors:
+    """Create both fixed actors once; any existing semantic drift fails atomically."""
+
+    require_production_operation_environment(expected_release_sha)
+    permissions = {role: _required_permission(role, lock=True) for role in _ROLE_CONTRACTS}
+    user_model = get_user_model()
+    existing = {
+        actor.username: actor
+        for actor in user_model._default_manager.select_for_update().filter(
+            username__in=tuple(contract[0] for contract in _ROLE_CONTRACTS.values())
+        )
+    }
+
+    actors: dict[ControlRole, Any] = {}
+    created: dict[ControlRole, bool] = {}
+    for role, (username, _permission_spec) in _ROLE_CONTRACTS.items():
+        actor = existing.get(username)
+        was_created = actor is None
+        if actor is None:
+            actor = _create_actor(role=role, permission=permissions[role])
+        else:
+            _validate_actor_shape(actor, role=role, permission=permissions[role])
+            _control_actor_id(actor)
+        actors[role] = actor
+        created[role] = was_created
+
+    return BootstrappedActors(
+        reviewer=actors["reviewer"],
+        reviewer_created=created["reviewer"],
+        publisher=actors["publisher"],
+        publisher_created=created["publisher"],
+    )
diff --git a/grocery/tests/test_control_plane_commands.py b/grocery/tests/test_control_plane_commands.py
new file mode 100644
index 0000000..d87c93f
--- /dev/null
+++ b/grocery/tests/test_control_plane_commands.py
@@ -0,0 +1,488 @@
+"""Focused production control-plane boundary and lifecycle command tests."""
+
+from __future__ import annotations
+
+import io
+import uuid
+from collections.abc import Iterator
+from contextlib import contextmanager
+from typing import Any
+from unittest.mock import patch
+
+import pytest
+from django.contrib.auth import get_user_model
+from django.contrib.auth.models import Permission
+from django.core.management import call_command
+from django.core.management.base import CommandError
+from django.test import override_settings
+
+from grocery.management.commands.approve_recent_generation import Command as ApproveCommand
+from grocery.management.commands.seal_recent_publication import Command as SealCommand
+from grocery.management.commands.transition_recent_publication import (
+    Command as TransitionCommand,
+)
+from grocery.management.control_plane import (
+    CONTROL_APPROVAL_REASON_CODE,
+    CONTROL_PUBLISHER_USERNAME,
+    CONTROL_REVIEWER_USERNAME,
+    CONTROL_TRANSITION_REASON_CODES,
+    ControlPlaneCode,
+    ControlPlaneError,
+    require_production_operation_environment,
+)
+from grocery.models import (
+    PublicationActivation,
+    PublicationChannel,
+    PublicationRevision,
+    ReviewDecision,
+)
+from grocery.tests.test_review_decision_models import complete_generation
+
+_RELEASE_SHA = "a" * 40
+_OTHER_RELEASE_SHA = "b" * 40
+_ACCEPTANCE_HASH = "7" * 64
+_TRANSITION_HASH = "8" * 64
+_PRODUCTION_SETTINGS: dict[str, object] = {
+    "DEBUG": False,
+    "ADMIN_ENABLED": False,
+    "QA_STATE_PREVIEWS_ENABLED": False,
+    "CONTROL_PLANE_OPERATIONS_ENABLED": True,
+    "DEPLOY_VERSION": _RELEASE_SHA,
+}
+
+
+def _run(name: str, **options: object) -> str:
+    output = io.StringIO()
+    call_command(name, stdout=output, **options)
+    return output.getvalue().strip()
+
+
+def _user(username: str) -> Any:
+    return get_user_model()._default_manager.get(username=username)
+
+
+def _permission_contract(actor: Any) -> set[tuple[str, str, str]]:
+    return set(
+        actor.user_permissions.values_list(
+            "content_type__app_label",
+            "content_type__model",
+            "codename",
+        )
+    )
+
+
+@contextmanager
+def _production_settings(**changes: object) -> Iterator[None]:
+    values = {**_PRODUCTION_SETTINGS, **changes}
+    with override_settings(**values):
+        yield
+
+
+def test_production_precondition_requires_all_flags_and_exact_running_release() -> None:
+    with _production_settings():
+        require_production_operation_environment(_RELEASE_SHA)
+
+    denied_settings = (
+        {"DEBUG": True},
+        {"ADMIN_ENABLED": True},
+        {"QA_STATE_PREVIEWS_ENABLED": True},
+        {"CONTROL_PLANE_OPERATIONS_ENABLED": False},
+    )
+    for changes in denied_settings:
+        with (
+            _production_settings(**changes),
+            pytest.raises(ControlPlaneError) as caught,
+        ):
+            require_production_operation_environment(_RELEASE_SHA)
+        assert caught.value.code is ControlPlaneCode.ENVIRONMENT_DENIED
+
+    invalid_release_marker = "private-release-marker"
+    with _production_settings(), pytest.raises(ControlPlaneError) as caught:
+        require_production_operation_environment(invalid_release_marker)
+    assert caught.value.code is ControlPlaneCode.RELEASE_SHA_INVALID
+    assert invalid_release_marker not in str(caught.value)
+
+    with _production_settings(), pytest.raises(ControlPlaneError) as caught:
+        require_production_operation_environment(_OTHER_RELEASE_SHA)
+    assert caught.value.code is ControlPlaneCode.RELEASE_SHA_MISMATCH
+    assert _OTHER_RELEASE_SHA not in str(caught.value)
+
+
+def test_write_commands_expose_release_lock_but_no_actor_override() -> None:
+    commands = (
+        ("approve_recent_generation", ApproveCommand()),
+        ("seal_recent_publication", SealCommand()),
+        ("transition_recent_publication", TransitionCommand()),
+    )
+    for name, command in commands:
+        parser = command.create_parser("manage.py", name)
+        destinations = {action.dest for action in parser._actions}
+        assert "expected_release_sha" in destinations
+        assert "actor" not in destinations
+        assert "actor_id" not in destinations
+        assert "username" not in destinations
+
+
+@override_settings(**_PRODUCTION_SETTINGS)
+def test_production_write_commands_require_release_before_service_dispatch() -> None:
+    calls = (
+        (
+            "approve_recent_generation",
+            "grocery.management.commands.approve_recent_generation.Command._approve",
+            {
+                "parse_run_id": uuid.uuid4(),
+                "decision_id": uuid.uuid4(),
+                "acceptance_evidence_sha256": _ACCEPTANCE_HASH,
+            },
+        ),
+        (
+            "seal_recent_publication",
+            "grocery.management.commands.seal_recent_publication.Command._seal",
+            {"decision_id": uuid.uuid4(), "public_copy_revision": "ko-v1"},
+        ),
+        (
+            "transition_recent_publication",
+            "grocery.management.commands.transition_recent_publication.Command._transition",
+            {
+                "operation": "ACTIVATE",
+                "operation_id": uuid.uuid4(),
+                "acceptance_evidence_sha256": _TRANSITION_HASH,
+                "expected_version": 0,
+                "expected_current_revision": "NONE",
+                "target_revision": uuid.uuid4(),
+            },
+        ),
+    )
+    for command_name, service_path, options in calls:
+        with patch(service_path) as service, pytest.raises(CommandError) as caught:
+            _run(command_name, **options)
+        service.assert_not_called()
+        assert str(caught.value) == "code=CONTROL_PLANE_RELEASE_SHA_INVALID"
+
+
+@pytest.mark.django_db
+@override_settings(**_PRODUCTION_SETTINGS)
+def test_bootstrap_creates_exact_nonlogin_roles_and_replays_idempotently() -> None:
+    first = _run("bootstrap_control_plane_actors", expected_release_sha=_RELEASE_SHA)
+    second = _run("bootstrap_control_plane_actors", expected_release_sha=_RELEASE_SHA)
+
+    reviewer = _user(CONTROL_REVIEWER_USERNAME)
+    publisher = _user(CONTROL_PUBLISHER_USERNAME)
+    assert first == " ".join(
+        (
+            "status=READY",
+            "review_actor=READY",
+            "review_created=yes",
+            "publication_actor=READY",
+            "publication_created=yes",
+        )
+    )
+    assert second == first.replace("created=yes", "created=no")
+    assert CONTROL_REVIEWER_USERNAME not in first
+    assert CONTROL_PUBLISHER_USERNAME not in first
+    assert _RELEASE_SHA not in first
+
+    for actor in (reviewer, publisher):
+        assert actor.email == actor.first_name == actor.last_name == ""
+        assert actor.is_active is True
+        assert actor.is_staff is False
+        assert actor.is_superuser is False
+        assert actor.has_usable_password() is False
+        assert actor.groups.count() == 0
+    assert _permission_contract(reviewer) == {("grocery", "reviewdecision", "review_generation")}
+    assert _permission_contract(publisher) == {
+        ("grocery", "publicationactivation", "publish_publication")
+    }
+
+
+@pytest.mark.django_db
+@override_settings(**_PRODUCTION_SETTINGS)
+def test_bootstrap_release_mismatch_and_existing_drift_never_partially_mutate() -> None:
+    with pytest.raises(CommandError) as caught:
+        _run("bootstrap_control_plane_actors", expected_release_sha=_OTHER_RELEASE_SHA)
+    assert str(caught.value) == "code=CONTROL_PLANE_RELEASE_SHA_MISMATCH"
+    assert _OTHER_RELEASE_SHA not in str(caught.value)
+    assert (
+        not get_user_model()
+        ._default_manager.filter(
+            username__in=(CONTROL_REVIEWER_USERNAME, CONTROL_PUBLISHER_USERNAME)
+        )
+        .exists()
+    )
+
+    drift_marker = "private-actor-drift-marker@example.invalid"
+    reviewer = get_user_model()._default_manager.create_user(
+        username=CONTROL_REVIEWER_USERNAME,
+        email=drift_marker,
+        password=None,
+        is_active=True,
+        is_staff=False,
+        is_superuser=False,
+    )
+    reviewer.user_permissions.add(
+        Permission.objects.get(
+            content_type__app_label="grocery",
+            content_type__model="reviewdecision",
+            codename="review_generation",
+        )
+    )
+
+    with pytest.raises(CommandError) as caught:
+        _run("bootstrap_control_plane_actors", expected_release_sha=_RELEASE_SHA)
+
+    reviewer.refresh_from_db()
+    assert str(caught.value) == "code=CONTROL_PLANE_ACTOR_CONFLICT"
+    assert drift_marker not in str(caught.value)
+    assert reviewer.email == drift_marker
+    assert (
+        not get_user_model()._default_manager.filter(username=CONTROL_PUBLISHER_USERNAME).exists()
+    )
+
+
+@pytest.mark.django_db
+@override_settings(**_PRODUCTION_SETTINGS)
+def test_production_lifecycle_uses_separate_roles_reasons_and_idempotent_services() -> None:
+    _run("bootstrap_control_plane_actors", expected_release_sha=_RELEASE_SHA)
+    source, parse_run = complete_generation()
+    decision_id = uuid.uuid4()
+
+    approval_first = _run(
+        "approve_recent_generation",
+        parse_run_id=parse_run.id,
+        decision_id=decision_id,
+        acceptance_evidence_sha256=_ACCEPTANCE_HASH,
+        expected_release_sha=_RELEASE_SHA,
+    )
+    approval_replay = _run(
+        "approve_recent_generation",
+        parse_run_id=parse_run.id,
+        decision_id=decision_id,
+        acceptance_evidence_sha256=_ACCEPTANCE_HASH,
+        expected_release_sha=_RELEASE_SHA,
+    )
+    seal_first = _run(
+        "seal_recent_publication",
+        decision_id=decision_id,
+        public_copy_revision="ko-v1",
+        expected_release_sha=_RELEASE_SHA,
+    )
+    seal_replay = _run(
+        "seal_recent_publication",
+        decision_id=decision_id,
+        public_copy_revision="ko-v1",
+        expected_release_sha=_RELEASE_SHA,
+    )
+    revision = PublicationRevision.objects.get(review_decision_id=decision_id)
+    operation_id = uuid.uuid4()
+    activation_first = _run(
+        "transition_recent_publication",
+        operation="ACTIVATE",
+        operation_id=operation_id,
+        acceptance_evidence_sha256=_TRANSITION_HASH,
+        expected_version=0,
+        expected_current_revision="NONE",
+        target_revision=revision.id,
+        expected_release_sha=_RELEASE_SHA,
+    )
+    activation_replay = _run(
+        "transition_recent_publication",
+        operation="ACTIVATE",
+        operation_id=operation_id,
+        acceptance_evidence_sha256=_TRANSITION_HASH,
+        expected_version=0,
+        expected_current_revision="NONE",
+        target_revision=revision.id,
+        expected_release_sha=_RELEASE_SHA,
+    )
+
+    reviewer = _user(CONTROL_REVIEWER_USERNAME)
+    publisher = _user(CONTROL_PUBLISHER_USERNAME)
+    decision = ReviewDecision.objects.get(pk=decision_id)
+    activation = PublicationActivation.objects.get(pk=operation_id)
+    assert decision.reviewer_id == reviewer.pk
+    assert decision.reason_code == CONTROL_APPROVAL_REASON_CODE
+    assert decision.source_configuration_id == source.id
+    assert activation.publisher_id == publisher.pk
+    assert activation.reason_code == CONTROL_TRANSITION_REASON_CODES["ACTIVATE"]
+    assert approval_first.endswith("created=yes")
+    assert approval_replay == approval_first.removesuffix("created=yes") + "created=no"
+    assert seal_first.endswith("created=yes")
+    assert seal_replay == seal_first.removesuffix("created=yes") + "created=no"
+    assert activation_first.endswith("created=yes")
+    assert activation_replay == activation_first.removesuffix("created=yes") + "created=no"
+
+    all_receipts = "\n".join(
+        (
+            approval_first,
+            approval_replay,
+            seal_first,
+            seal_replay,
+            activation_first,
+            activation_replay,
+        )
+    )
+    for forbidden in (
+        CONTROL_REVIEWER_USERNAME,
+        CONTROL_PUBLISHER_USERNAME,
+        "actor_id=",
+        _RELEASE_SHA,
+        _ACCEPTANCE_HASH,
+        _TRANSITION_HASH,
+    ):
+        assert forbidden not in all_receipts
+
+
+@pytest.mark.django_db
+@override_settings(**_PRODUCTION_SETTINGS)
+def test_release_mismatch_blocks_each_write_command_before_lifecycle_mutation() -> None:
+    _run("bootstrap_control_plane_actors", expected_release_sha=_RELEASE_SHA)
+    _source, parse_run = complete_generation()
+    decision_id = uuid.uuid4()
+
+    with pytest.raises(CommandError) as caught:
+        _run(
+            "approve_recent_generation",
+            parse_run_id=parse_run.id,
+            decision_id=decision_id,
+            acceptance_evidence_sha256=_ACCEPTANCE_HASH,
+            expected_release_sha=_OTHER_RELEASE_SHA,
+        )
+    assert str(caught.value) == "code=CONTROL_PLANE_RELEASE_SHA_MISMATCH"
+    assert not ReviewDecision.objects.exists()
+
+    _run(
+        "approve_recent_generation",
+        parse_run_id=parse_run.id,
+        decision_id=decision_id,
+        acceptance_evidence_sha256=_ACCEPTANCE_HASH,
+        expected_release_sha=_RELEASE_SHA,
+    )
+    with pytest.raises(CommandError) as caught:
+        _run(
+            "seal_recent_publication",
+            decision_id=decision_id,
+            public_copy_revision="ko-v1",
+            expected_release_sha=_OTHER_RELEASE_SHA,
+        )
+    assert str(caught.value) == "code=CONTROL_PLANE_RELEASE_SHA_MISMATCH"
+    assert not PublicationRevision.objects.exists()
+
+    revision_receipt = _run(
+        "seal_recent_publication",
+        decision_id=decision_id,
+        public_copy_revision="ko-v1",
+        expected_release_sha=_RELEASE_SHA,
+    )
+    assert "status=SEALED" in revision_receipt
+    revision = PublicationRevision.objects.get()
+    with pytest.raises(CommandError) as caught:
+        _run(
+            "transition_recent_publication",
+            operation="ACTIVATE",
+            operation_id=uuid.uuid4(),
+            acceptance_evidence_sha256=_TRANSITION_HASH,
+            expected_version=0,
+            expected_current_revision="NONE",
+            target_revision=revision.id,
+            expected_release_sha=_OTHER_RELEASE_SHA,
+        )
+    assert str(caught.value) == "code=CONTROL_PLANE_RELEASE_SHA_MISMATCH"
+    assert not PublicationChannel.objects.exists()
+    assert not PublicationActivation.objects.exists()
+    assert _OTHER_RELEASE_SHA not in str(caught.value)
+
+
+@pytest.mark.django_db
+@override_settings(**_PRODUCTION_SETTINGS)
+def test_publisher_is_never_substituted_for_missing_or_drifted_reviewer() -> None:
+    _run("bootstrap_control_plane_actors", expected_release_sha=_RELEASE_SHA)
+    _user(CONTROL_REVIEWER_USERNAME).delete()
+    _source, parse_run = complete_generation()
+
+    with pytest.raises(CommandError) as caught:
+        _run(
+            "approve_recent_generation",
+            parse_run_id=parse_run.id,
+            decision_id=uuid.uuid4(),
+            acceptance_evidence_sha256=_ACCEPTANCE_HASH,
+            expected_release_sha=_RELEASE_SHA,
+        )
+    assert str(caught.value) == "code=CONTROL_PLANE_ACTOR_MISSING"
+    assert _user(CONTROL_PUBLISHER_USERNAME).has_perm("grocery.review_generation") is False
+    assert not ReviewDecision.objects.exists()
+
+
+@pytest.mark.django_db
+@override_settings(**_PRODUCTION_SETTINGS)
+def test_reviewer_is_never_substituted_for_missing_publisher_on_seal_or_transition() -> None:
+    _run("bootstrap_control_plane_actors", expected_release_sha=_RELEASE_SHA)
+    _source, parse_run = complete_generation()
+    decision_id = uuid.uuid4()
+    _run(
+        "approve_recent_generation",
+        parse_run_id=parse_run.id,
+        decision_id=decision_id,
+        acceptance_evidence_sha256=_ACCEPTANCE_HASH,
+        expected_release_sha=_RELEASE_SHA,
+    )
+    _run(
+        "seal_recent_publication",
+        decision_id=decision_id,
+        public_copy_revision="ko-v1",
+        expected_release_sha=_RELEASE_SHA,
+    )
+    revision = PublicationRevision.objects.get()
+    _user(CONTROL_PUBLISHER_USERNAME).delete()
+
+    with pytest.raises(CommandError) as seal_error:
+        _run(
+            "seal_recent_publication",
+            decision_id=decision_id,
+            public_copy_revision="ko-v2",
+            expected_release_sha=_RELEASE_SHA,
+        )
+    with pytest.raises(CommandError) as transition_error:
+        _run(
+            "transition_recent_publication",
+            operation="ACTIVATE",
+            operation_id=uuid.uuid4(),
+            acceptance_evidence_sha256=_TRANSITION_HASH,
+            expected_version=0,
+            expected_current_revision="NONE",
+            target_revision=revision.id,
+            expected_release_sha=_RELEASE_SHA,
+        )
+
+    assert str(seal_error.value) == "code=CONTROL_PLANE_ACTOR_MISSING"
+    assert str(transition_error.value) == "code=CONTROL_PLANE_ACTOR_MISSING"
+    assert _user(CONTROL_REVIEWER_USERNAME).has_perm("grocery.publish_publication") is False
+    assert PublicationRevision.objects.count() == 1
+    assert not PublicationChannel.objects.exists()
+    assert not PublicationActivation.objects.exists()
+
+
+@pytest.mark.django_db
+@override_settings(**_PRODUCTION_SETTINGS)
+def test_actor_drift_error_is_redacted_and_approval_does_not_mutate() -> None:
+    _run("bootstrap_control_plane_actors", expected_release_sha=_RELEASE_SHA)
+    drift_marker = "private-review-drift@example.invalid"
+    get_user_model()._default_manager.filter(username=CONTROL_REVIEWER_USERNAME).update(
+        email=drift_marker
+    )
+    _source, parse_run = complete_generation()
+
+    with pytest.raises(CommandError) as caught:
+        _run(
+            "approve_recent_generation",
+            parse_run_id=parse_run.id,
+            decision_id=uuid.uuid4(),
+            acceptance_evidence_sha256=_ACCEPTANCE_HASH,
+            expected_release_sha=_RELEASE_SHA,
+        )
+
+    assert str(caught.value) == "code=CONTROL_PLANE_ACTOR_CONFLICT"
+    assert drift_marker not in str(caught.value)
+    assert CONTROL_REVIEWER_USERNAME not in str(caught.value)
+    assert _RELEASE_SHA not in str(caught.value)
+    assert _ACCEPTANCE_HASH not in str(caught.value)
+    assert not ReviewDecision.objects.exists()


