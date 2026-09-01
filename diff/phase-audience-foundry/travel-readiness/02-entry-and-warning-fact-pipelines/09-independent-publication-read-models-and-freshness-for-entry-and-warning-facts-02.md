## `fix(web): bind freshness to exact source receipts`

diff --git a/public_web/results.py b/public_web/results.py
index f859077..b0f0ab6 100644
--- a/public_web/results.py
+++ b/public_web/results.py
@@ -1,5 +1,6 @@
 from __future__ import annotations
 
+from dataclasses import dataclass
 from datetime import UTC, date, datetime
 from typing import Callable
 
@@ -9,13 +10,41 @@ from django.shortcuts import render
 from django.urls import reverse
 from django.views.decorators.http import require_GET
 
+from entry_requirements.ingestion import (
+    ENTRY_ATTRIBUTION,
+    ENTRY_CONTRACT_FINGERPRINT_SHA256,
+    ENTRY_DECIDED_BY,
+    ENTRY_DECISION_BASIS,
+    ENTRY_FIELD_SCOPE,
+    ENTRY_SOURCE_CODE,
+    ENTRY_SOURCE_MODULE,
+    ENTRY_SOURCE_OWNER,
+    ENTRY_SOURCE_REVISION,
+)
 from reviews.models import (
     ENTRY_POINTER_ID,
     WARNING_POINTER_ID,
     PublishedEntryFacts,
     PublishedTravelWarning,
 )
-from sources.models import FetchAttempt, SourceArtifact, SourceConfiguration
+from sources.models import (
+    FetchAttempt,
+    SourceArtifact,
+    SourceConfiguration,
+    SourceRightsDecision,
+)
+from sources.transport import ENTRY_SOURCE_LOCATOR, WARNING_SOURCE_LOCATOR
+from travel_warnings.ingestion import (
+    WARNING_ATTRIBUTION,
+    WARNING_DECIDED_BY,
+    WARNING_DECISION_BASIS,
+    WARNING_FIELD_SCOPE,
+    WARNING_SOURCE_CODE,
+    WARNING_SOURCE_MODULE,
+    WARNING_SOURCE_OWNER,
+    WARNING_SOURCE_REVISION,
+)
+from travel_warnings.parser import PARSER_CONTRACT_FINGERPRINT_SHA256
 
 
 CARD_READY = "ready"
@@ -25,6 +54,54 @@ CARD_STALE = "stale"
 CARD_SERVER_ERROR = "server-error"
 
 
+@dataclass(frozen=True, slots=True)
+class _FreshnessContract:
+    source_code: str
+    source_revision: str
+    source_module: str
+    source_owner: str
+    source_locator: str
+    access_mode: str
+    field_scope: str
+    contract_fingerprint_sha256: str
+    attribution: str
+    decided_by: str
+    decision_basis: str
+    provider_code: str
+
+
+_FRESHNESS_CONTRACTS = {
+    "entry": _FreshnessContract(
+        source_code=ENTRY_SOURCE_CODE,
+        source_revision=ENTRY_SOURCE_REVISION,
+        source_module=ENTRY_SOURCE_MODULE,
+        source_owner=ENTRY_SOURCE_OWNER,
+        source_locator=ENTRY_SOURCE_LOCATOR,
+        access_mode=SourceRightsDecision.AccessMode.ANONYMOUS_PUBLIC,
+        field_scope=ENTRY_FIELD_SCOPE,
+        contract_fingerprint_sha256=ENTRY_CONTRACT_FINGERPRINT_SHA256,
+        attribution=ENTRY_ATTRIBUTION,
+        decided_by=ENTRY_DECIDED_BY,
+        decision_basis=ENTRY_DECISION_BASIS,
+        provider_code="",
+    ),
+    "warning": _FreshnessContract(
+        source_code=WARNING_SOURCE_CODE,
+        source_revision=WARNING_SOURCE_REVISION,
+        source_module=WARNING_SOURCE_MODULE,
+        source_owner=WARNING_SOURCE_OWNER,
+        source_locator=WARNING_SOURCE_LOCATOR,
+        access_mode=SourceRightsDecision.AccessMode.CREDENTIAL_REFERENCE,
+        field_scope=WARNING_FIELD_SCOPE,
+        contract_fingerprint_sha256=PARSER_CONTRACT_FINGERPRINT_SHA256,
+        attribution=WARNING_ATTRIBUTION,
+        decided_by=WARNING_DECIDED_BY,
+        decision_basis=WARNING_DECISION_BASIS,
+        provider_code=FetchAttempt.ProviderCode.MOFA_SUCCESS_0,
+    ),
+}
+
+
 def _fixed_redirect(route_name: str) -> HttpResponse:
     return HttpResponse(
         status=303,
@@ -75,39 +152,139 @@ def _utc_minute(value: datetime | None) -> str | None:
 
 def _freshness(
     *,
+    contract: _FreshnessContract,
     artifact_id,
     source_id,
+    publication_source_code: str,
+    publication_source_revision: str,
+    publication_source_owner: str,
+    publication_source_locator: str,
+    publication_contract_fingerprint_sha256: str,
+    publication_attribution: str,
+    source_code: str,
     source_revision: str,
+    source_module: str,
+    source_owner: str,
+    source_locator: str,
     source_state: str,
     source_enabled: bool,
 ) -> tuple[str, datetime | None]:
+    rights = (
+        SourceRightsDecision.objects.filter(
+            source_id=source_id,
+            source_revision=publication_source_revision,
+        )
+        .order_by("-decision_sequence", "-id")
+        .values(
+            "id",
+            "decision_sequence",
+            "decision",
+            "access_mode",
+            "access_allowed",
+            "automated_collection_allowed",
+            "typed_field_storage_allowed",
+            "raw_body_storage_allowed",
+            "typed_republication_allowed",
+            "raw_retention_seconds",
+            "typed_retention",
+            "evidence_retention",
+            "field_scope_code",
+            "attribution_text",
+            "contract_fingerprint_sha256",
+            "decided_by",
+            "decision_basis_code",
+        )
+        .first()
+    )
+    publication_contract_exact = (
+        publication_source_code == contract.source_code == source_code
+        and publication_source_revision
+        == contract.source_revision
+        == source_revision
+        and publication_source_owner == contract.source_owner == source_owner
+        and publication_source_locator
+        == contract.source_locator
+        == source_locator
+        and source_module == contract.source_module
+        and publication_contract_fingerprint_sha256
+        == contract.contract_fingerprint_sha256
+        and publication_attribution == contract.attribution
+    )
+    rights_exact = (
+        rights is not None
+        and rights["decision_sequence"] == 1
+        and rights["decision"] == SourceRightsDecision.Decision.APPROVED
+        and rights["access_mode"] == contract.access_mode
+        and rights["access_allowed"] is True
+        and rights["automated_collection_allowed"] is True
+        and rights["typed_field_storage_allowed"] is True
+        and rights["raw_body_storage_allowed"] is False
+        and rights["typed_republication_allowed"] is True
+        and rights["raw_retention_seconds"] == 0
+        and rights["typed_retention"]
+        == SourceRightsDecision.Retention.PRODUCT_HISTORY
+        and rights["evidence_retention"]
+        == SourceRightsDecision.Retention.PRODUCT_HISTORY
+        and rights["field_scope_code"] == contract.field_scope
+        and rights["attribution_text"]
+        == publication_attribution
+        == contract.attribution
+        and rights["contract_fingerprint_sha256"]
+        == publication_contract_fingerprint_sha256
+        == contract.contract_fingerprint_sha256
+        and rights["decided_by"] == contract.decided_by
+        and rights["decision_basis_code"] == contract.decision_basis
+    )
     matching_body = SourceArtifact.objects.filter(
         id=artifact_id,
         body_sha256=OuterRef("response_sha256"),
     )
     terminal_attempts = FetchAttempt.objects.filter(
         source_id=source_id,
-        source_revision=source_revision,
+        source_revision=publication_source_revision,
         completed_at__isnull=False,
     ).annotate(matches_publication=Exists(matching_body))
     latest = (
         terminal_attempts.order_by("-completed_at", "-started_at", "-id")
-        .values("id", "result", "matches_publication")
+        .values(
+            "id",
+            "result",
+            "http_status",
+            "provider_code",
+            "rights_decision_id",
+            "matches_publication",
+        )
         .first()
     )
-    last_matching_success = (
-        terminal_attempts.filter(
-            result=FetchAttempt.Result.SUCCEEDED,
-            matches_publication=True,
+    last_matching_success = None
+    if rights is not None:
+        last_matching_success = (
+            terminal_attempts.filter(
+                result=FetchAttempt.Result.SUCCEEDED,
+                http_status=200,
+                provider_code=contract.provider_code,
+                rights_decision_id=rights["id"],
+                matches_publication=True,
+            )
+            .order_by("-completed_at", "-started_at", "-id")
+            .values("id", "completed_at")
+            .first()
         )
-        .order_by("-completed_at", "-started_at", "-id")
-        .values("id", "completed_at")
-        .first()
+    latest_exact = (
+        latest is not None
+        and rights is not None
+        and latest["result"] == FetchAttempt.Result.SUCCEEDED
+        and latest["http_status"] == 200
+        and latest["provider_code"] == contract.provider_code
+        and latest["rights_decision_id"] == rights["id"]
+        and latest["matches_publication"] is True
     )
     stale = (
-        source_state != SourceConfiguration.State.ACTIVE
+        not publication_contract_exact
+        or not rights_exact
+        or source_state != SourceConfiguration.State.ACTIVE
         or not source_enabled
-        or latest is None
+        or not latest_exact
         or last_matching_success is None
         or latest["id"] != last_matching_success["id"]
     )
@@ -127,16 +304,22 @@ def _entry_row() -> dict | None:
             "current_publication__generation",
             "current_publication__published_at",
             "current_publication__scope_country__name_ko",
+            "current_publication__source_code_snapshot",
             "current_publication__source_revision",
             "current_publication__source_owner_snapshot",
             "current_publication__source_locator_snapshot",
             "current_publication__attribution_text_snapshot",
+            "current_publication__source_contract_fingerprint_sha256",
             "current_publication__entry_fact_revision__ordinary_passport_period_text",
             "current_publication__entry_fact_revision__basis_text",
             "current_publication__entry_fact_revision__snapshot_date",
             "current_publication__entry_fact_revision__parse_run__artifact_id",
             "current_publication__entry_fact_revision__parse_run__artifact__source_id",
+            "current_publication__entry_fact_revision__parse_run__artifact__source__code",
             "current_publication__entry_fact_revision__parse_run__artifact__source__revision",
+            "current_publication__entry_fact_revision__parse_run__artifact__source__module",
+            "current_publication__entry_fact_revision__parse_run__artifact__source__owner",
+            "current_publication__entry_fact_revision__parse_run__artifact__source__official_locator",
             "current_publication__entry_fact_revision__parse_run__artifact__source__state",
             "current_publication__entry_fact_revision__parse_run__artifact__source__enabled",
         )
@@ -152,17 +335,23 @@ def _warning_row() -> dict | None:
             "current_publication__generation",
             "current_publication__published_at",
             "current_publication__scope_country__name_ko",
+            "current_publication__source_code_snapshot",
             "current_publication__source_revision",
             "current_publication__source_owner_snapshot",
             "current_publication__source_locator_snapshot",
             "current_publication__attribution_text_snapshot",
+            "current_publication__source_contract_fingerprint_sha256",
             "current_publication__travel_warning_revision__source_alarm_level_code",
             "current_publication__travel_warning_revision__source_scope_type",
             "current_publication__travel_warning_revision__source_scope_text",
             "current_publication__travel_warning_revision__source_written_date",
             "current_publication__travel_warning_revision__parse_run__artifact_id",
             "current_publication__travel_warning_revision__parse_run__artifact__source_id",
+            "current_publication__travel_warning_revision__parse_run__artifact__source__code",
             "current_publication__travel_warning_revision__parse_run__artifact__source__revision",
+            "current_publication__travel_warning_revision__parse_run__artifact__source__module",
+            "current_publication__travel_warning_revision__parse_run__artifact__source__owner",
+            "current_publication__travel_warning_revision__parse_run__artifact__source__official_locator",
             "current_publication__travel_warning_revision__parse_run__artifact__source__state",
             "current_publication__travel_warning_revision__parse_run__artifact__source__enabled",
         )
@@ -178,9 +367,30 @@ def _load_entry_card() -> dict[str, object]:
         return _state_card("entry", CARD_EMPTY)
     prefix = "current_publication__entry_fact_revision__parse_run__artifact"
     state, checked_at = _freshness(
+        contract=_FRESHNESS_CONTRACTS["entry"],
         artifact_id=row[f"{prefix}_id"],
         source_id=row[f"{prefix}__source_id"],
+        publication_source_code=row[
+            "current_publication__source_code_snapshot"
+        ],
+        publication_source_revision=row["current_publication__source_revision"],
+        publication_source_owner=row[
+            "current_publication__source_owner_snapshot"
+        ],
+        publication_source_locator=row[
+            "current_publication__source_locator_snapshot"
+        ],
+        publication_contract_fingerprint_sha256=row[
+            "current_publication__source_contract_fingerprint_sha256"
+        ],
+        publication_attribution=row[
+            "current_publication__attribution_text_snapshot"
+        ],
+        source_code=row[f"{prefix}__source__code"],
         source_revision=row[f"{prefix}__source__revision"],
+        source_module=row[f"{prefix}__source__module"],
+        source_owner=row[f"{prefix}__source__owner"],
+        source_locator=row[f"{prefix}__source__official_locator"],
         source_state=row[f"{prefix}__source__state"],
         source_enabled=row[f"{prefix}__source__enabled"],
     )
@@ -224,9 +434,30 @@ def _load_warning_card() -> dict[str, object]:
         return _state_card("warning", CARD_EMPTY)
     prefix = "current_publication__travel_warning_revision__parse_run__artifact"
     state, checked_at = _freshness(
+        contract=_FRESHNESS_CONTRACTS["warning"],
         artifact_id=row[f"{prefix}_id"],
         source_id=row[f"{prefix}__source_id"],
+        publication_source_code=row[
+            "current_publication__source_code_snapshot"
+        ],
+        publication_source_revision=row["current_publication__source_revision"],
+        publication_source_owner=row[
+            "current_publication__source_owner_snapshot"
+        ],
+        publication_source_locator=row[
+            "current_publication__source_locator_snapshot"
+        ],
+        publication_contract_fingerprint_sha256=row[
+            "current_publication__source_contract_fingerprint_sha256"
+        ],
+        publication_attribution=row[
+            "current_publication__attribution_text_snapshot"
+        ],
+        source_code=row[f"{prefix}__source__code"],
         source_revision=row[f"{prefix}__source__revision"],
+        source_module=row[f"{prefix}__source__module"],
+        source_owner=row[f"{prefix}__source__owner"],
+        source_locator=row[f"{prefix}__source__official_locator"],
         source_state=row[f"{prefix}__source__state"],
         source_enabled=row[f"{prefix}__source__enabled"],
     )
diff --git a/public_web/tests/test_results.py b/public_web/tests/test_results.py
index 1091639..f05511f 100644
--- a/public_web/tests/test_results.py
+++ b/public_web/tests/test_results.py
@@ -1,3 +1,4 @@
+import uuid
 from unittest.mock import patch
 
 from django.contrib.auth import get_user_model
@@ -5,6 +6,7 @@ from django.contrib.auth.models import Permission
 from django.db import DatabaseError
 from django.test import TransactionTestCase
 from django.urls import reverse
+from django.utils import timezone
 
 from entry_requirements.models import EntryFactRevision
 from reviews.models import (
@@ -18,7 +20,11 @@ from reviews.publication import (
     rollback_publication,
 )
 from reviews.tests.test_publication import PublicationFixtureMixin
-from sources.models import SourceConfiguration
+from sources.models import (
+    FetchAttempt,
+    SourceConfiguration,
+    SourceRightsDecision,
+)
 
 
 class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
@@ -67,6 +73,42 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
         self.assertEqual(outcome.code, PublicationCode.PUBLISHED)
         return warning
 
+    def append_same_body_success(
+        self,
+        subject,
+        *,
+        source,
+        rights,
+        http_status=200,
+        provider_code="",
+    ):
+        artifact = subject.parse_run.artifact
+        first_attempt = artifact.first_successful_attempt
+        attempt = FetchAttempt.objects.create(
+            source=source,
+            source_revision=source.revision,
+            rights_decision=rights,
+            operation_id=uuid.uuid4(),
+            attempt_number=1,
+            request_fingerprint_revision=(
+                first_attempt.request_fingerprint_revision
+            ),
+            normalized_request_sha256=(
+                first_attempt.normalized_request_sha256
+            ),
+            started_at=timezone.now(),
+        )
+        FetchAttempt.objects.filter(pk=attempt.pk).update(
+            completed_at=timezone.now(),
+            result=FetchAttempt.Result.SUCCEEDED,
+            http_status=http_status,
+            provider_code=provider_code,
+            response_sha256=artifact.body_sha256,
+            failure_code="",
+        )
+        attempt.refresh_from_db()
+        return attempt
+
     def test_empty_cards_are_independent_semantic_states(self):
         response = self.client.get(reverse("public_web:results"))
 
@@ -140,6 +182,85 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
         self.assertContains(response, "재확인 필요")
         self.assertContains(response, "90일")
 
+    def test_new_contract_same_body_does_not_refresh_old_publication(self):
+        entry = self.publish_entry()
+        artifact = entry.parse_run.artifact
+        source = artifact.source
+        approval = source.rights_decisions.get(
+            source_revision=source.revision,
+            decision_sequence=1,
+        )
+        SourceConfiguration.objects.filter(pk=source.pk).update(
+            revision="rights-v2",
+            state=SourceConfiguration.State.DRAFT,
+            enabled=False,
+        )
+        source.refresh_from_db()
+        replacement_rights = SourceRightsDecision.objects.create(
+            source=source,
+            source_revision=source.revision,
+            decision_sequence=1,
+            decision=SourceRightsDecision.Decision.APPROVED,
+            access_mode=approval.access_mode,
+            access_allowed=True,
+            automated_collection_allowed=True,
+            typed_field_storage_allowed=True,
+            raw_body_storage_allowed=False,
+            typed_republication_allowed=True,
+            raw_retention_seconds=0,
+            typed_retention=SourceRightsDecision.Retention.PRODUCT_HISTORY,
+            evidence_retention=(
+                SourceRightsDecision.Retention.PRODUCT_HISTORY
+            ),
+            field_scope_code=approval.field_scope_code,
+            attribution_text=approval.attribution_text,
+            contract_fingerprint_sha256="f" * 64,
+            decided_by="PROJECT_OWNER_REQUEST",
+            decision_basis_code="SYNTHETIC_NEW_CONTRACT",
+        )
+        SourceConfiguration.objects.filter(pk=source.pk).update(
+            state=SourceConfiguration.State.RIGHTS_APPROVED,
+            enabled=False,
+        )
+        SourceConfiguration.objects.filter(pk=source.pk).update(
+            state=SourceConfiguration.State.ACTIVE,
+            enabled=True,
+        )
+        source.refresh_from_db()
+        self.append_same_body_success(
+            entry,
+            source=source,
+            rights=replacement_rights,
+        )
+
+        response = self.client.get(reverse("public_web:results"))
+
+        self.assertContains(response, 'id="entry-card" data-state="stale"')
+        self.assertContains(response, "source revision</dt><dd>rights-v1")
+        self.assertContains(response, 'id="warning-card" data-state="empty"')
+
+    def test_warning_nonexact_success_receipt_is_stale(self):
+        warning = self.publish_warning()
+        artifact = warning.parse_run.artifact
+        source = artifact.source
+        rights = source.rights_decisions.get(
+            source_revision=source.revision,
+            decision_sequence=1,
+        )
+        self.append_same_body_success(
+            warning,
+            source=source,
+            rights=rights,
+            http_status=204,
+            provider_code="",
+        )
+
+        response = self.client.get(reverse("public_web:results"))
+
+        self.assertContains(response, 'id="warning-card" data-state="stale"')
+        self.assertContains(response, "재확인 필요")
+        self.assertContains(response, 'id="entry-card" data-state="empty"')
+
     def test_unavailable_and_server_error_are_isolated(self):
         self.publish_warning()
         with (


