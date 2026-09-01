## `fix(search): enforce country publication contracts`

diff --git a/public_web/results.py b/public_web/results.py
index 967a2fd..de4cd0d 100644
--- a/public_web/results.py
+++ b/public_web/results.py
@@ -21,8 +21,13 @@ from entry_requirements.ingestion import (
     ENTRY_SOURCE_MODULE,
     ENTRY_SOURCE_OWNER,
     ENTRY_SOURCE_REVISION,
+    LEGACY_ENTRY_CONTRACT_FINGERPRINT_SHA256,
+    LEGACY_ENTRY_FIELD_SCOPE,
+    LEGACY_ENTRY_SOURCE_CODE,
+    LEGACY_ENTRY_SOURCE_REVISION,
+    MULTI_COUNTRY_ENTRY_SOURCE_CODE,
 )
-from countries.models import SUPPORTED_COUNTRY_IDS
+from countries.models import JP_COUNTRY_ID, SUPPORTED_COUNTRY_IDS
 from reviews.models import (
     PublishedEntryFacts,
     PublishedTravelWarning,
@@ -35,6 +40,11 @@ from sources.models import (
 )
 from sources.transport import ENTRY_SOURCE_LOCATOR, WARNING_SOURCE_LOCATOR
 from travel_warnings.ingestion import (
+    LEGACY_WARNING_CONTRACT_FINGERPRINT_SHA256,
+    LEGACY_WARNING_FIELD_SCOPE,
+    LEGACY_WARNING_SOURCE_CODE,
+    LEGACY_WARNING_SOURCE_REVISION,
+    MULTI_COUNTRY_WARNING_SOURCE_CODE,
     WARNING_ATTRIBUTION,
     WARNING_DECIDED_BY,
     WARNING_DECISION_BASIS,
@@ -87,6 +97,23 @@ _FRESHNESS_CONTRACTS = {
         provider_code="",
         max_success_age=timedelta(hours=36),
     ),
+    "entry_legacy": _FreshnessContract(
+        source_code=LEGACY_ENTRY_SOURCE_CODE,
+        source_revision=LEGACY_ENTRY_SOURCE_REVISION,
+        source_module=ENTRY_SOURCE_MODULE,
+        source_owner=ENTRY_SOURCE_OWNER,
+        source_locator=ENTRY_SOURCE_LOCATOR,
+        access_mode=SourceRightsDecision.AccessMode.ANONYMOUS_PUBLIC,
+        field_scope=LEGACY_ENTRY_FIELD_SCOPE,
+        contract_fingerprint_sha256=(
+            LEGACY_ENTRY_CONTRACT_FINGERPRINT_SHA256
+        ),
+        attribution=ENTRY_ATTRIBUTION,
+        decided_by="PROJECT_OWNER_REQUEST",
+        decision_basis="USER_FOLLOWUP_20260830",
+        provider_code="",
+        max_success_age=timedelta(hours=36),
+    ),
     "warning": _FreshnessContract(
         source_code=WARNING_SOURCE_CODE,
         source_revision=WARNING_SOURCE_REVISION,
@@ -102,9 +129,54 @@ _FRESHNESS_CONTRACTS = {
         provider_code=FetchAttempt.ProviderCode.MOFA_SUCCESS_0,
         max_success_age=timedelta(hours=8),
     ),
+    "warning_legacy": _FreshnessContract(
+        source_code=LEGACY_WARNING_SOURCE_CODE,
+        source_revision=LEGACY_WARNING_SOURCE_REVISION,
+        source_module=WARNING_SOURCE_MODULE,
+        source_owner=WARNING_SOURCE_OWNER,
+        source_locator=WARNING_SOURCE_LOCATOR,
+        access_mode=SourceRightsDecision.AccessMode.CREDENTIAL_REFERENCE,
+        field_scope=LEGACY_WARNING_FIELD_SCOPE,
+        contract_fingerprint_sha256=(
+            LEGACY_WARNING_CONTRACT_FINGERPRINT_SHA256
+        ),
+        attribution=WARNING_ATTRIBUTION,
+        decided_by="PROJECT_OWNER_REQUEST",
+        decision_basis="USER_FOLLOWUP_20260830",
+        provider_code=FetchAttempt.ProviderCode.MOFA_SUCCESS_0,
+        max_success_age=timedelta(hours=8),
+    ),
 }
 
 
+def _freshness_contract_for_publication(
+    *,
+    module: str,
+    country_id: UUID,
+    source_code: object,
+) -> _FreshnessContract | None:
+    """Select only an approved country/source contract without rewriting history."""
+
+    if module == "entry":
+        if source_code == MULTI_COUNTRY_ENTRY_SOURCE_CODE:
+            return _FRESHNESS_CONTRACTS["entry"]
+        if (
+            country_id == JP_COUNTRY_ID
+            and source_code == LEGACY_ENTRY_SOURCE_CODE
+        ):
+            return _FRESHNESS_CONTRACTS["entry_legacy"]
+        return None
+    if module == "warning":
+        if source_code == MULTI_COUNTRY_WARNING_SOURCE_CODE:
+            return _FRESHNESS_CONTRACTS["warning"]
+        if (
+            country_id == JP_COUNTRY_ID
+            and source_code == LEGACY_WARNING_SOURCE_CODE
+        ):
+            return _FRESHNESS_CONTRACTS["warning_legacy"]
+    return None
+
+
 def _fixed_redirect(route_name: str) -> HttpResponse:
     return HttpResponse(
         status=303,
@@ -198,6 +270,8 @@ def _freshness(
     *,
     contract: _FreshnessContract,
     artifact_id,
+    artifact_request_fingerprint_revision: str,
+    artifact_normalized_request_sha256: str,
     source_id,
     publication_source_code: str,
     publication_source_revision: str,
@@ -290,6 +364,10 @@ def _freshness(
     terminal_attempts = FetchAttempt.objects.filter(
         source_id=source_id,
         source_revision=publication_source_revision,
+        request_fingerprint_revision=(
+            artifact_request_fingerprint_revision
+        ),
+        normalized_request_sha256=artifact_normalized_request_sha256,
         completed_at__isnull=False,
     ).annotate(matches_publication=Exists(matching_body))
     latest = (
@@ -374,6 +452,8 @@ def _entry_row(country_id: UUID) -> dict | None:
             "current_publication__entry_fact_revision__parse_run__artifact__source__official_locator",
             "current_publication__entry_fact_revision__parse_run__artifact__source__state",
             "current_publication__entry_fact_revision__parse_run__artifact__source__enabled",
+            "current_publication__entry_fact_revision__parse_run__artifact__first_successful_attempt__request_fingerprint_revision",
+            "current_publication__entry_fact_revision__parse_run__artifact__first_successful_attempt__normalized_request_sha256",
         )
         .first()
     )
@@ -406,6 +486,8 @@ def _warning_row(country_id: UUID) -> dict | None:
             "current_publication__travel_warning_revision__parse_run__artifact__source__official_locator",
             "current_publication__travel_warning_revision__parse_run__artifact__source__state",
             "current_publication__travel_warning_revision__parse_run__artifact__source__enabled",
+            "current_publication__travel_warning_revision__parse_run__artifact__first_successful_attempt__request_fingerprint_revision",
+            "current_publication__travel_warning_revision__parse_run__artifact__first_successful_attempt__normalized_request_sha256",
         )
         .first()
     )
@@ -417,35 +499,51 @@ def _load_entry_card(country_id: UUID) -> dict[str, object]:
         return _state_card("entry", CARD_UNAVAILABLE)
     if row["current_publication_id"] is None:
         return _state_card("entry", CARD_EMPTY)
-    prefix = "current_publication__entry_fact_revision__parse_run__artifact"
-    state, checked_at = _freshness(
-        contract=_FRESHNESS_CONTRACTS["entry"],
-        artifact_id=row[f"{prefix}_id"],
-        source_id=row[f"{prefix}__source_id"],
-        publication_source_code=row[
-            "current_publication__source_code_snapshot"
-        ],
-        publication_source_revision=row["current_publication__source_revision"],
-        publication_source_owner=row[
-            "current_publication__source_owner_snapshot"
-        ],
-        publication_source_locator=row[
-            "current_publication__source_locator_snapshot"
-        ],
-        publication_contract_fingerprint_sha256=row[
-            "current_publication__source_contract_fingerprint_sha256"
-        ],
-        publication_attribution=row[
-            "current_publication__attribution_text_snapshot"
-        ],
-        source_code=row[f"{prefix}__source__code"],
-        source_revision=row[f"{prefix}__source__revision"],
-        source_module=row[f"{prefix}__source__module"],
-        source_owner=row[f"{prefix}__source__owner"],
-        source_locator=row[f"{prefix}__source__official_locator"],
-        source_state=row[f"{prefix}__source__state"],
-        source_enabled=row[f"{prefix}__source__enabled"],
+    contract = _freshness_contract_for_publication(
+        module="entry",
+        country_id=country_id,
+        source_code=row["current_publication__source_code_snapshot"],
     )
+    if contract is None:
+        state, checked_at = CARD_STALE, None
+    else:
+        prefix = "current_publication__entry_fact_revision__parse_run__artifact"
+        state, checked_at = _freshness(
+            contract=contract,
+            artifact_id=row[f"{prefix}_id"],
+            artifact_request_fingerprint_revision=row[
+                f"{prefix}__first_successful_attempt__request_fingerprint_revision"
+            ],
+            artifact_normalized_request_sha256=row[
+                f"{prefix}__first_successful_attempt__normalized_request_sha256"
+            ],
+            source_id=row[f"{prefix}__source_id"],
+            publication_source_code=row[
+                "current_publication__source_code_snapshot"
+            ],
+            publication_source_revision=row[
+                "current_publication__source_revision"
+            ],
+            publication_source_owner=row[
+                "current_publication__source_owner_snapshot"
+            ],
+            publication_source_locator=row[
+                "current_publication__source_locator_snapshot"
+            ],
+            publication_contract_fingerprint_sha256=row[
+                "current_publication__source_contract_fingerprint_sha256"
+            ],
+            publication_attribution=row[
+                "current_publication__attribution_text_snapshot"
+            ],
+            source_code=row[f"{prefix}__source__code"],
+            source_revision=row[f"{prefix}__source__revision"],
+            source_module=row[f"{prefix}__source__module"],
+            source_owner=row[f"{prefix}__source__owner"],
+            source_locator=row[f"{prefix}__source__official_locator"],
+            source_state=row[f"{prefix}__source__state"],
+            source_enabled=row[f"{prefix}__source__enabled"],
+        )
     stale = state == CARD_STALE
     return {
         "module": "entry",
@@ -484,35 +582,53 @@ def _load_warning_card(country_id: UUID) -> dict[str, object]:
         return _state_card("warning", CARD_UNAVAILABLE)
     if row["current_publication_id"] is None:
         return _state_card("warning", CARD_EMPTY)
-    prefix = "current_publication__travel_warning_revision__parse_run__artifact"
-    state, checked_at = _freshness(
-        contract=_FRESHNESS_CONTRACTS["warning"],
-        artifact_id=row[f"{prefix}_id"],
-        source_id=row[f"{prefix}__source_id"],
-        publication_source_code=row[
-            "current_publication__source_code_snapshot"
-        ],
-        publication_source_revision=row["current_publication__source_revision"],
-        publication_source_owner=row[
-            "current_publication__source_owner_snapshot"
-        ],
-        publication_source_locator=row[
-            "current_publication__source_locator_snapshot"
-        ],
-        publication_contract_fingerprint_sha256=row[
-            "current_publication__source_contract_fingerprint_sha256"
-        ],
-        publication_attribution=row[
-            "current_publication__attribution_text_snapshot"
-        ],
-        source_code=row[f"{prefix}__source__code"],
-        source_revision=row[f"{prefix}__source__revision"],
-        source_module=row[f"{prefix}__source__module"],
-        source_owner=row[f"{prefix}__source__owner"],
-        source_locator=row[f"{prefix}__source__official_locator"],
-        source_state=row[f"{prefix}__source__state"],
-        source_enabled=row[f"{prefix}__source__enabled"],
+    contract = _freshness_contract_for_publication(
+        module="warning",
+        country_id=country_id,
+        source_code=row["current_publication__source_code_snapshot"],
     )
+    if contract is None:
+        state, checked_at = CARD_STALE, None
+    else:
+        prefix = (
+            "current_publication__travel_warning_revision__parse_run__artifact"
+        )
+        state, checked_at = _freshness(
+            contract=contract,
+            artifact_id=row[f"{prefix}_id"],
+            artifact_request_fingerprint_revision=row[
+                f"{prefix}__first_successful_attempt__request_fingerprint_revision"
+            ],
+            artifact_normalized_request_sha256=row[
+                f"{prefix}__first_successful_attempt__normalized_request_sha256"
+            ],
+            source_id=row[f"{prefix}__source_id"],
+            publication_source_code=row[
+                "current_publication__source_code_snapshot"
+            ],
+            publication_source_revision=row[
+                "current_publication__source_revision"
+            ],
+            publication_source_owner=row[
+                "current_publication__source_owner_snapshot"
+            ],
+            publication_source_locator=row[
+                "current_publication__source_locator_snapshot"
+            ],
+            publication_contract_fingerprint_sha256=row[
+                "current_publication__source_contract_fingerprint_sha256"
+            ],
+            publication_attribution=row[
+                "current_publication__attribution_text_snapshot"
+            ],
+            source_code=row[f"{prefix}__source__code"],
+            source_revision=row[f"{prefix}__source__revision"],
+            source_module=row[f"{prefix}__source__module"],
+            source_owner=row[f"{prefix}__source__owner"],
+            source_locator=row[f"{prefix}__source__official_locator"],
+            source_state=row[f"{prefix}__source__state"],
+            source_enabled=row[f"{prefix}__source__enabled"],
+        )
     stale = state == CARD_STALE
     return {
         "module": "warning",
diff --git a/public_web/tests/test_travel_opportunity_web.py b/public_web/tests/test_travel_opportunity_web.py
index 7b69b78..c4267ea 100644
--- a/public_web/tests/test_travel_opportunity_web.py
+++ b/public_web/tests/test_travel_opportunity_web.py
@@ -1,5 +1,5 @@
 from datetime import date, datetime
-from unittest.mock import patch
+from unittest.mock import MagicMock, patch
 from uuid import UUID
 from zoneinfo import ZoneInfo
 
@@ -7,15 +7,27 @@ from django.http import HttpResponse
 from django.test import SimpleTestCase
 from django.urls import reverse
 
-from countries.models import SUPPORTED_COUNTRY_IDS
+from countries.models import JP_COUNTRY_ID, SUPPORTED_COUNTRY_IDS
+from entry_requirements.ingestion import (
+    LEGACY_ENTRY_SOURCE_CODE,
+    MULTI_COUNTRY_ENTRY_SOURCE_CODE,
+)
 from public_web.forms import TripForm
 from public_web.results import (
     CARD_READY,
     CARD_SERVER_ERROR,
+    CARD_STALE,
+    _FRESHNESS_CONTRACTS,
+    _freshness,
+    _freshness_contract_for_publication,
     _load_entry_card,
     _load_warning_card,
     load_publication_cards_for_country,
 )
+from travel_warnings.ingestion import (
+    LEGACY_WARNING_SOURCE_CODE,
+    MULTI_COUNTRY_WARNING_SOURCE_CODE,
+)
 
 
 SEOUL = ZoneInfo("Asia/Seoul")
@@ -217,6 +229,9 @@ class CountryPublicationCardTests(SimpleTestCase):
     @patch("public_web.results._entry_row")
     def test_entry_card_contains_complete_typed_facts(self, entry_row, _freshness):
         row = _publication_base_row()
+        row["current_publication__source_code_snapshot"] = (
+            MULTI_COUNTRY_ENTRY_SOURCE_CODE
+        )
         prefix = "current_publication__entry_fact_revision__parse_run__artifact"
         row.update(
             {
@@ -232,6 +247,8 @@ class CountryPublicationCardTests(SimpleTestCase):
                 f"{prefix}__source__official_locator": "https://example.invalid/source",
                 f"{prefix}__source__state": "ACTIVE",
                 f"{prefix}__source__enabled": True,
+                f"{prefix}__first_successful_attempt__request_fingerprint_revision": "ENTRY_REQUEST_V1",
+                f"{prefix}__first_successful_attempt__normalized_request_sha256": "b" * 64,
             }
         )
         entry_row.return_value = row
@@ -247,6 +264,9 @@ class CountryPublicationCardTests(SimpleTestCase):
     @patch("public_web.results._warning_row")
     def test_warning_card_contains_complete_typed_facts(self, warning_row, _freshness):
         row = _publication_base_row()
+        row["current_publication__source_code_snapshot"] = (
+            MULTI_COUNTRY_WARNING_SOURCE_CODE
+        )
         prefix = "current_publication__travel_warning_revision__parse_run__artifact"
         row.update(
             {
@@ -263,6 +283,8 @@ class CountryPublicationCardTests(SimpleTestCase):
                 f"{prefix}__source__official_locator": "https://example.invalid/source",
                 f"{prefix}__source__state": "ACTIVE",
                 f"{prefix}__source__enabled": True,
+                f"{prefix}__first_successful_attempt__request_fingerprint_revision": "WARNING_TW_V1",
+                f"{prefix}__first_successful_attempt__normalized_request_sha256": "c" * 64,
             }
         )
         warning_row.return_value = row
@@ -284,3 +306,117 @@ class CountryPublicationCardTests(SimpleTestCase):
 
         self.assertEqual(cards["entry"]["state"], CARD_SERVER_ERROR)
         self.assertEqual(cards["warning"]["state"], CARD_READY)
+
+    def test_legacy_contracts_are_eligible_only_for_japan(self):
+        entry = _freshness_contract_for_publication(
+            module="entry",
+            country_id=JP_COUNTRY_ID,
+            source_code=LEGACY_ENTRY_SOURCE_CODE,
+        )
+        warning = _freshness_contract_for_publication(
+            module="warning",
+            country_id=JP_COUNTRY_ID,
+            source_code=LEGACY_WARNING_SOURCE_CODE,
+        )
+
+        self.assertIsNotNone(entry)
+        self.assertIsNotNone(warning)
+        self.assertEqual(entry.source_code, LEGACY_ENTRY_SOURCE_CODE)
+        self.assertEqual(warning.source_code, LEGACY_WARNING_SOURCE_CODE)
+        self.assertIsNone(
+            _freshness_contract_for_publication(
+                module="entry",
+                country_id=SUPPORTED_COUNTRY_IDS["TW"],
+                source_code=LEGACY_ENTRY_SOURCE_CODE,
+            )
+        )
+        self.assertIsNone(
+            _freshness_contract_for_publication(
+                module="warning",
+                country_id=SUPPORTED_COUNTRY_IDS["TW"],
+                source_code=LEGACY_WARNING_SOURCE_CODE,
+            )
+        )
+
+    @patch("public_web.results._freshness")
+    @patch("public_web.results._entry_row")
+    def test_wrong_scope_non_japan_publication_is_stale(
+        self,
+        entry_row,
+        freshness,
+    ):
+        row = _publication_base_row()
+        row["current_publication__source_code_snapshot"] = (
+            LEGACY_ENTRY_SOURCE_CODE
+        )
+        row.update(
+            {
+                "current_publication__entry_fact_revision__ordinary_passport_period_text": "90일",
+                "current_publication__entry_fact_revision__basis_text": "일반여권 소지자: 90일",
+                "current_publication__entry_fact_revision__snapshot_date": date(2026, 8, 1),
+            }
+        )
+        entry_row.return_value = row
+
+        card = _load_entry_card(SUPPORTED_COUNTRY_IDS["TW"])
+
+        self.assertEqual(card["state"], CARD_STALE)
+        self.assertIsNone(card["checked_at"])
+        freshness.assert_not_called()
+
+    @patch("public_web.results.FetchAttempt")
+    @patch("public_web.results.SourceArtifact")
+    @patch("public_web.results.SourceRightsDecision")
+    def test_freshness_scopes_latest_attempt_to_publication_request_identity(
+        self,
+        rights_model,
+        artifact_model,
+        attempt_model,
+    ):
+        rights_values = MagicMock()
+        rights_model.objects.filter.return_value.values.return_value = (
+            rights_values
+        )
+        rights_values.filter.return_value.first.return_value = None
+        rights_values.order_by.return_value.first.return_value = None
+        artifact_model.objects.filter.return_value = MagicMock()
+        attempts = MagicMock()
+        attempt_model.objects.filter.return_value.annotate.return_value = (
+            attempts
+        )
+        attempts.order_by.return_value.values.return_value.first.return_value = (
+            None
+        )
+        request_hash = "d" * 64
+
+        state, checked_at = _freshness(
+            contract=_FRESHNESS_CONTRACTS["warning"],
+            artifact_id=UUID("57a68d56-4e7c-4a06-a8bd-f106e0ed4a61"),
+            artifact_request_fingerprint_revision="MOFA_WARNING_V1",
+            artifact_normalized_request_sha256=request_hash,
+            source_id=UUID("5ca7e28c-81df-40ea-b77b-69b95860949f"),
+            publication_source_code="wrong",
+            publication_source_revision="wrong",
+            publication_source_owner="wrong",
+            publication_source_locator="https://example.invalid/wrong",
+            publication_contract_fingerprint_sha256="e" * 64,
+            publication_attribution="wrong",
+            source_code="wrong",
+            source_revision="wrong",
+            source_module="TRAVEL_WARNING",
+            source_owner="wrong",
+            source_locator="https://example.invalid/wrong",
+            source_state="ACTIVE",
+            source_enabled=True,
+        )
+
+        self.assertEqual((state, checked_at), (CARD_STALE, None))
+        filter_kwargs = attempt_model.objects.filter.call_args.kwargs
+        self.assertEqual(
+            filter_kwargs["request_fingerprint_revision"],
+            "MOFA_WARNING_V1",
+        )
+        self.assertEqual(
+            filter_kwargs["normalized_request_sha256"],
+            request_hash,
+        )
diff --git a/reviews/admin.py b/reviews/admin.py
index 6eaf9d4..628d069 100644
--- a/reviews/admin.py
+++ b/reviews/admin.py
@@ -121,15 +121,61 @@ def _single_primary_key(queryset):
     return selected[0]
 
 
-def _single_publication_identity(queryset):
-    selected = list(queryset.values_list("pk", "module")[:2])
+def _single_subject_identity(queryset, module: str):
+    fields = (
+        ("pk", "country_id", "passport_policy_id")
+        if module == PublicationModule.ENTRY
+        else ("pk", "country_id")
+    )
+    selected = list(queryset.values_list(*fields)[:2])
     if len(selected) != 1:
         return None
-    return selected[0]
+    row = selected[0]
+    if len(row) != len(fields) or row[1] is None:
+        return None
+    passport_policy_id = row[2] if len(row) == 3 else None
+    if module == PublicationModule.ENTRY and passport_policy_id is None:
+        return None
+    return row[0], row[1], passport_policy_id
 
 
-def _current_pointer_version(pointer_model) -> int:
-    version = pointer_model.objects.values_list("version", flat=True).get()
+def _single_publication_identity(queryset):
+    selected = list(
+        queryset.values_list(
+            "pk",
+            "module",
+            "scope_country_id",
+            "scope_passport_policy_id",
+        )[:2]
+    )
+    if len(selected) != 1 or len(selected[0]) != 4:
+        return None
+    publication_id, module, country_id, passport_policy_id = selected[0]
+    if country_id is None:
+        return None
+    if module == PublicationModule.ENTRY and passport_policy_id is None:
+        return None
+    if module == PublicationModule.TRAVEL_WARNING:
+        passport_policy_id = None
+    return publication_id, module, country_id, passport_policy_id
+
+
+def _current_pointer_version(
+    pointer_model,
+    *,
+    country_id,
+    passport_policy_id=None,
+) -> int:
+    scope = {"country_id": country_id}
+    if pointer_model is PublishedEntryFacts:
+        if passport_policy_id is None:
+            raise RuntimeError("entry publication pointer requires policy")
+        scope["passport_policy_id"] = passport_policy_id
+    version = (
+        pointer_model.objects.filter(**scope)
+        .values_list("version", flat=True)
+        .get()
+    )
     if type(version) is not int or version < 0:
         raise RuntimeError("invalid publication pointer")
     return version
@@ -147,14 +193,19 @@ def _fixed_outcome_code(outcome) -> str:
 
 def _run_publish_action(*, queryset, module, actor, pointer_model):
     try:
-        revision_id = _single_primary_key(queryset)
-        if revision_id is None:
+        identity = _single_subject_identity(queryset, module)
+        if identity is None:
             return _SELECTION_REQUIRED
+        revision_id, country_id, passport_policy_id = identity
         outcome = publish_candidate(
             module=module,
             typed_revision_id=revision_id,
             actor=actor,
-            expected_pointer_version=_current_pointer_version(pointer_model),
+            expected_pointer_version=_current_pointer_version(
+                pointer_model,
+                country_id=country_id,
+                passport_policy_id=passport_policy_id,
+            ),
         )
         return _fixed_outcome_code(outcome)
     except Exception:
@@ -183,14 +234,23 @@ def _run_rollback_action(
         identity = _single_publication_identity(queryset)
         if identity is None:
             return _SELECTION_REQUIRED
-        publication_id, selected_module = identity
+        (
+            publication_id,
+            selected_module,
+            country_id,
+            passport_policy_id,
+        ) = identity
         if selected_module != module:
             return PublicationCode.INVALID_TARGET
         outcome = rollback_publication(
             module=module,
             target_publication_id=publication_id,
             actor=actor,
-            expected_pointer_version=_current_pointer_version(pointer_model),
+            expected_pointer_version=_current_pointer_version(
+                pointer_model,
+                country_id=country_id,
+                passport_policy_id=passport_policy_id,
+            ),
         )
         return _fixed_outcome_code(outcome)
     except Exception:
diff --git a/reviews/tests/test_admin.py b/reviews/tests/test_admin.py
index 81344e4..a5746e5 100644
--- a/reviews/tests/test_admin.py
+++ b/reviews/tests/test_admin.py
@@ -1,6 +1,6 @@
 import uuid
 from types import SimpleNamespace
-from unittest.mock import patch
+from unittest.mock import MagicMock, call, patch
 
 from django.contrib import admin, messages
 from django.core.exceptions import ImproperlyConfigured
@@ -18,12 +18,14 @@ from reviews.admin import (
     TravelWarningRevisionAdmin,
     _GENERIC_FAILURE_MESSAGE,
     _OUTCOME_MESSAGES,
+    _current_pointer_version,
     operator_admin_site,
 )
 from reviews.models import (
     AuditEvent,
     PublicationModule,
     PublicationRevision,
+    PublishedEntryFacts,
     ReviewDecision,
 )
 from reviews.publication import PublicationCode, PublicationOutcome
@@ -222,6 +224,8 @@ class AdminBoundaryTests(SimpleTestCase):
 
     def test_entry_publish_passes_only_typed_id_actor_and_pointer_version(self):
         revision_id = uuid.uuid4()
+        country_id = uuid.uuid4()
+        policy_id = uuid.uuid4()
         request = self.request("reviews.publish_entry")
         outcome = PublicationOutcome(
             PublicationCode.PUBLISHED,
@@ -236,7 +240,7 @@ class AdminBoundaryTests(SimpleTestCase):
         ):
             self.entry_admin.publish_entry_candidate(
                 request,
-                _SelectedRows([(revision_id,)]),
+                _SelectedRows([(revision_id, country_id, policy_id)]),
             )
 
         call.assert_called_once_with(
@@ -272,6 +276,8 @@ class AdminBoundaryTests(SimpleTestCase):
 
     def test_rollback_is_single_target_and_exact_module(self):
         publication_id = uuid.uuid4()
+        country_id = uuid.uuid4()
+        policy_id = uuid.uuid4()
         request = self.request("reviews.rollback_entry")
         outcome = PublicationOutcome(
             PublicationCode.ROLLED_BACK,
@@ -286,7 +292,16 @@ class AdminBoundaryTests(SimpleTestCase):
         ):
             self.publication_admin.rollback_entry_publication(
                 request,
-                _SelectedRows([(publication_id, PublicationModule.ENTRY)]),
+                _SelectedRows(
+                    [
+                        (
+                            publication_id,
+                            PublicationModule.ENTRY,
+                            country_id,
+                            policy_id,
+                        )
+                    ]
+                ),
             )
 
         call.assert_called_once_with(
@@ -307,14 +322,26 @@ class AdminBoundaryTests(SimpleTestCase):
                 _SelectedRows(
                     [
                         (uuid.uuid4(), PublicationModule.ENTRY),
-                        (uuid.uuid4(), PublicationModule.ENTRY),
+                        (
+                            uuid.uuid4(),
+                            PublicationModule.ENTRY,
+                            uuid.uuid4(),
+                            uuid.uuid4(),
+                        ),
                     ]
                 ),
             )
             self.publication_admin.rollback_entry_publication(
                 request,
                 _SelectedRows(
-                    [(uuid.uuid4(), PublicationModule.TRAVEL_WARNING)]
+                    [
+                        (
+                            uuid.uuid4(),
+                            PublicationModule.TRAVEL_WARNING,
+                            uuid.uuid4(),
+                            None,
+                        )
+                    ]
                 ),
             )
 
@@ -329,7 +356,7 @@ class AdminBoundaryTests(SimpleTestCase):
         ):
             self.entry_admin.publish_entry_candidate(
                 request,
-                _SelectedRows([(uuid.uuid4(),)]),
+                _SelectedRows([(uuid.uuid4(), uuid.uuid4(), uuid.uuid4())]),
             )
 
         call.assert_not_called()
@@ -358,7 +385,7 @@ class AdminBoundaryTests(SimpleTestCase):
         ):
             self.entry_admin.publish_entry_candidate(
                 request,
-                _SelectedRows([(uuid.uuid4(),)]),
+                _SelectedRows([(uuid.uuid4(), uuid.uuid4(), uuid.uuid4())]),
             )
         rendered = message.call_args.args[1]
         self.assertEqual(rendered, _GENERIC_FAILURE_MESSAGE)
@@ -390,7 +417,9 @@ class AdminBoundaryTests(SimpleTestCase):
             try:
                 self.entry_admin.publish_entry_candidate(
                     request,
-                    _SelectedRows([(uuid.uuid4(),)]),
+                    _SelectedRows(
+                        [(uuid.uuid4(), uuid.uuid4(), uuid.uuid4())]
+                    ),
                 )
             except RuntimeError as exception:
                 caught_exception = exception
@@ -430,6 +459,48 @@ class AdminBoundaryTests(SimpleTestCase):
                     with self.assertRaises(exception_type):
                         self.entry_admin.publish_entry_candidate(
                             request,
-                            _SelectedRows([(uuid.uuid4(),)]),
+                            _SelectedRows(
+                                [(uuid.uuid4(), uuid.uuid4(), uuid.uuid4())]
+                            ),
                         )
                 message.assert_not_called()
+
+    def test_pointer_versions_are_resolved_by_country_and_policy(self):
+        first_country = uuid.uuid4()
+        second_country = uuid.uuid4()
+        policy_id = uuid.uuid4()
+        first_query = MagicMock()
+        second_query = MagicMock()
+        first_query.values_list.return_value.get.return_value = 2
+        second_query.values_list.return_value.get.return_value = 7
+
+        with patch.object(
+            PublishedEntryFacts.objects,
+            "filter",
+            side_effect=(first_query, second_query),
+        ) as scoped:
+            first = _current_pointer_version(
+                PublishedEntryFacts,
+                country_id=first_country,
+                passport_policy_id=policy_id,
+            )
+            second = _current_pointer_version(
+                PublishedEntryFacts,
+                country_id=second_country,
+                passport_policy_id=policy_id,
+            )
+
+        self.assertEqual((first, second), (2, 7))
+        self.assertEqual(
+            scoped.call_args_list,
+            [
+                call(
+                    country_id=first_country,
+                    passport_policy_id=policy_id,
+                ),
+                call(
+                    country_id=second_country,
+                    passport_policy_id=policy_id,
+                ),
+            ],
+        )
diff --git a/travel_windows/search.py b/travel_windows/search.py
index fe4899e..b0edd1d 100644
--- a/travel_windows/search.py
+++ b/travel_windows/search.py
@@ -9,12 +9,30 @@ from zoneinfo import ZoneInfo, ZoneInfoNotFoundError
 from django.db.models import F
 from django.utils import timezone
 
+from countries.models import JP_COUNTRY_ID
+from entry_requirements.ingestion import (
+    LEGACY_ENTRY_CONTRACT_FINGERPRINT_SHA256,
+    LEGACY_ENTRY_SOURCE_CODE,
+    LEGACY_ENTRY_SOURCE_REVISION,
+    MULTI_COUNTRY_ENTRY_CONTRACT_FINGERPRINT_SHA256,
+    MULTI_COUNTRY_ENTRY_SOURCE_CODE,
+    MULTI_COUNTRY_ENTRY_SOURCE_REVISION,
+)
 from entry_requirements.models import PASSPORT_POLICY_ID
 from reviews.models import (
+    PublicationModule,
     PublicationRevision,
     PublishedEntryFacts,
     PublishedTravelWarning,
 )
+from travel_warnings.ingestion import (
+    LEGACY_WARNING_CONTRACT_FINGERPRINT_SHA256,
+    LEGACY_WARNING_SOURCE_CODE,
+    LEGACY_WARNING_SOURCE_REVISION,
+    MULTI_COUNTRY_WARNING_CONTRACT_FINGERPRINT_SHA256,
+    MULTI_COUNTRY_WARNING_SOURCE_CODE,
+    MULTI_COUNTRY_WARNING_SOURCE_REVISION,
+)
 from travel_windows.models import (
     FLIGHT_POINTER_ID,
     FlightPublication,
@@ -37,7 +55,9 @@ FLIGHT_EMPTY = "empty"
 FLIGHT_UNAVAILABLE = "unavailable"
 FLIGHT_STALE = "stale"
 FLIGHT_SERVER_ERROR = "server-error"
-ELIGIBLE_PUBLICATION_CARD_STATES = frozenset((FLIGHT_READY, FLIGHT_STALE))
+ELIGIBLE_PUBLICATION_CARD_STATES = frozenset(
+    (FLIGHT_READY, FLIGHT_STALE, FLIGHT_SERVER_ERROR)
+)
 
 _CALCULATION_NOTICE = (
     "공식 인천 출발·도착 시각에 검수된 노선별 비행시간을 더하고 빼서 계산한 "
@@ -116,24 +136,100 @@ PublicationCardLoader = Callable[[UUID], dict[str, dict[str, object]]]
 
 
 def _load_eligible_country_ids() -> frozenset[UUID]:
-    entry_country_ids = set(
+    entry_rows = (
         PublishedEntryFacts.objects.filter(
             passport_policy_id=PASSPORT_POLICY_ID,
             current_publication__isnull=False,
+            current_publication__module=PublicationModule.ENTRY,
             current_publication__state=PublicationRevision.State.PUBLISHED,
             current_publication__scope_country_id=F("country_id"),
-        ).values_list("country_id", flat=True)
+        ).values(
+            "country_id",
+            "current_publication__source_code_snapshot",
+            "current_publication__source_revision",
+            "current_publication__source_contract_fingerprint_sha256",
+            "current_publication__parser_contract_fingerprint_sha256",
+        )
     )
-    warning_country_ids = set(
+    warning_rows = (
         PublishedTravelWarning.objects.filter(
             current_publication__isnull=False,
+            current_publication__module=PublicationModule.TRAVEL_WARNING,
             current_publication__state=PublicationRevision.State.PUBLISHED,
             current_publication__scope_country_id=F("country_id"),
-        ).values_list("country_id", flat=True)
+        ).values(
+            "country_id",
+            "current_publication__source_code_snapshot",
+            "current_publication__source_revision",
+            "current_publication__source_contract_fingerprint_sha256",
+            "current_publication__parser_contract_fingerprint_sha256",
+        )
     )
+    entry_country_ids = {
+        row["country_id"]
+        for row in entry_rows
+        if _publication_contract_allowed(row, module=PublicationModule.ENTRY)
+    }
+    warning_country_ids = {
+        row["country_id"]
+        for row in warning_rows
+        if _publication_contract_allowed(
+            row,
+            module=PublicationModule.TRAVEL_WARNING,
+        )
+    }
     return frozenset(entry_country_ids & warning_country_ids)
 
 
+def _publication_contract_allowed(
+    row: dict[str, object], *, module: str
+) -> bool:
+    country_id = row.get("country_id")
+    if not isinstance(country_id, UUID):
+        return False
+    if module == PublicationModule.ENTRY:
+        legacy = (
+            LEGACY_ENTRY_SOURCE_CODE,
+            LEGACY_ENTRY_SOURCE_REVISION,
+            LEGACY_ENTRY_CONTRACT_FINGERPRINT_SHA256,
+        )
+        multi_country = (
+            MULTI_COUNTRY_ENTRY_SOURCE_CODE,
+            MULTI_COUNTRY_ENTRY_SOURCE_REVISION,
+            MULTI_COUNTRY_ENTRY_CONTRACT_FINGERPRINT_SHA256,
+        )
+    elif module == PublicationModule.TRAVEL_WARNING:
+        legacy = (
+            LEGACY_WARNING_SOURCE_CODE,
+            LEGACY_WARNING_SOURCE_REVISION,
+            LEGACY_WARNING_CONTRACT_FINGERPRINT_SHA256,
+        )
+        multi_country = (
+            MULTI_COUNTRY_WARNING_SOURCE_CODE,
+            MULTI_COUNTRY_WARNING_SOURCE_REVISION,
+            MULTI_COUNTRY_WARNING_CONTRACT_FINGERPRINT_SHA256,
+        )
+    else:
+        return False
+    observed = (
+        row.get("current_publication__source_code_snapshot"),
+        row.get("current_publication__source_revision"),
+        row.get(
+            "current_publication__source_contract_fingerprint_sha256"
+        ),
+    )
+    parser_fingerprint = row.get(
+        "current_publication__parser_contract_fingerprint_sha256"
+    )
+    if observed == multi_country:
+        return parser_fingerprint == multi_country[2]
+    return (
+        country_id == JP_COUNTRY_ID
+        and observed == legacy
+        and parser_fingerprint == legacy[2]
+    )
+
+
 def _load_current_flight_evidence() -> _CurrentFlightEvidence | None:
     pointer = (
         PublishedFlightSchedule.objects.select_related(
@@ -151,6 +247,8 @@ def _load_current_flight_evidence() -> _CurrentFlightEvidence | None:
         publication.state != FlightPublication.State.PUBLISHED
         or revision.state != FlightScheduleRevision.State.VALIDATED
         or revision.completeness != FlightScheduleRevision.Completeness.COMPLETE
+        or revision.city_reference_parse_run_id is None
+        or revision.legacy_arrivals_parse_run_id is None
         or timezone.is_naive(publication.source_checked_at)
     ):
         raise _FlightEvidenceUnavailable
@@ -361,7 +459,13 @@ def _calculate_candidates(
     by_city: dict[str, list[_Candidate]] = {}
     for candidate in candidates:
         city_rows = by_city.setdefault(candidate.airport.city_code, [])
-        if len(city_rows) < MAXIMUM_ITINERARIES_PER_CITY:
+        if (
+            len(city_rows) < MAXIMUM_ITINERARIES_PER_CITY
+            and (
+                not city_rows
+                or candidate.airport.id == city_rows[0].airport.id
+            )
+        ):
             city_rows.append(candidate)
     ranked_cities = sorted(by_city.values(), key=lambda rows: _candidate_sort_key(rows[0]))
     return tuple(tuple(rows) for rows in ranked_cities[:MAXIMUM_CITY_RESULTS])
diff --git a/travel_windows/tests/test_search.py b/travel_windows/tests/test_search.py
index d2a7d1e..42a3f36 100644
--- a/travel_windows/tests/test_search.py
+++ b/travel_windows/tests/test_search.py
@@ -1,15 +1,29 @@
 import json
 from datetime import date, datetime, time
+from types import SimpleNamespace
 from unittest.mock import patch
 from zoneinfo import ZoneInfo
 
 from django.test import SimpleTestCase, TransactionTestCase
 
-from countries.models import SUPPORTED_COUNTRY_IDS, Country
+from countries.models import (
+    SUPPORTED_COUNTRY_IDS,
+    SUPPORTED_COUNTRY_ROWS,
+    Country,
+)
+from entry_requirements.ingestion import (
+    LEGACY_ENTRY_CONTRACT_FINGERPRINT_SHA256,
+    LEGACY_ENTRY_SOURCE_CODE,
+    LEGACY_ENTRY_SOURCE_REVISION,
+    MULTI_COUNTRY_ENTRY_CONTRACT_FINGERPRINT_SHA256,
+    MULTI_COUNTRY_ENTRY_SOURCE_CODE,
+    MULTI_COUNTRY_ENTRY_SOURCE_REVISION,
+)
 from operations.tests import migration_helpers as _migration_helpers  # noqa: F401
 from sources.management.commands.register_approved_sources import (
     register_approved_sources,
 )
+from travel_windows.contracts import DURATION_SOURCE_LOCATOR
 from travel_windows.ingestion import (
     FlightIngestionCode,
     ingest_and_publish_flight_evidence,
@@ -27,10 +41,25 @@ from travel_windows.search import (
     _AirportEvidence,
     _CurrentFlightEvidence,
     _DurationEvidence,
+    _FlightEvidenceUnavailable,
     _ScheduleEvidence,
+    _load_current_flight_evidence,
     _load_eligible_country_ids,
     search_travel_opportunities,
 )
+from travel_windows.tests.test_source_publication import (
+    city_reference_payload,
+    legacy_arrival_page,
+    legacy_arrival_row,
+)
+from travel_warnings.ingestion import (
+    LEGACY_WARNING_CONTRACT_FINGERPRINT_SHA256,
+    LEGACY_WARNING_SOURCE_CODE,
+    LEGACY_WARNING_SOURCE_REVISION,
+    MULTI_COUNTRY_WARNING_CONTRACT_FINGERPRINT_SHA256,
+    MULTI_COUNTRY_WARNING_SOURCE_CODE,
+    MULTI_COUNTRY_WARNING_SOURCE_REVISION,
+)
 
 
 SEOUL = ZoneInfo("Asia/Seoul")
@@ -322,6 +351,55 @@ class TravelOpportunityCalculationTests(SimpleTestCase):
             [1, 2, 3, 4, 5, 6],
         )
 
+    def test_city_alternatives_never_switch_the_displayed_airport(self):
+        nrt = _airport_evidence("NRT")
+        hnd = _airport_evidence("HND")
+        schedules = (
+            _schedule(
+                nrt,
+                direction="OUTBOUND",
+                event_time=time(8),
+                flight_number="KE701",
+                weekday=1,
+            ),
+            _schedule(
+                nrt,
+                direction="INBOUND",
+                event_time=time(7),
+                flight_number="KE704",
+                weekday=3,
+            ),
+            _schedule(
+                hnd,
+                direction="OUTBOUND",
+                event_time=time(9),
+                flight_number="KE719",
+                weekday=1,
+            ),
+            _schedule(
+                hnd,
+                direction="INBOUND",
+                event_time=time(6),
+                flight_number="KE720",
+                weekday=3,
+            ),
+        )
+
+        result = _search(
+            _evidence(
+                schedules=schedules,
+                durations=(_duration(nrt), _duration(hnd)),
+            ),
+            {SUPPORTED_COUNTRY_IDS["JP"]},
+            departure_at=datetime(2026, 9, 1, 7, tzinfo=SEOUL),
+            return_by=datetime(2026, 9, 3, 8, tzinfo=SEOUL),
+        )
+
+        self.assertEqual(len(result["opportunities"]), 1)
+        opportunity = result["opportunities"][0]
+        self.assertEqual(opportunity["destination"]["airport_code"], "NRT")
+        self.assertEqual(opportunity["alternatives"], [])
+
     def test_sorting_uses_stay_then_slack_then_departure_instant(self):
         hkg = _airport_evidence("HKG")
         nrt = _airport_evidence("NRT")
@@ -419,7 +497,7 @@ class TravelOpportunityCalculationTests(SimpleTestCase):
             self.assertEqual(result["flight_state"], FLIGHT_STALE)
             self.assertEqual(len(result["opportunities"]), 1)
 
-    def test_city_requires_two_usable_independent_publication_cards(self):
+    def test_entry_loader_error_keeps_flight_and_warning_result(self):
         nrt = _airport_evidence("NRT")
         evidence = _evidence(
             schedules=(
@@ -461,30 +539,81 @@ class TravelOpportunityCalculationTests(SimpleTestCase):
             )
 
         self.assertEqual(result["flight_state"], FLIGHT_READY)
-        self.assertEqual(result["opportunities"], [])
+        self.assertEqual(len(result["opportunities"]), 1)
+        opportunity = result["opportunities"][0]
+        self.assertEqual(opportunity["entry_card"]["state"], "server-error")
+        self.assertEqual(opportunity["warning_card"]["state"], "ready")
 
 
 class PublicationEligibilityTests(SimpleTestCase):
-    def test_country_requires_both_current_entry_and_warning_publications(self):
+    def test_country_requires_two_exact_country_approved_contracts(self):
         japan = SUPPORTED_COUNTRY_IDS["JP"]
         taiwan = SUPPORTED_COUNTRY_IDS["TW"]
         vietnam = SUPPORTED_COUNTRY_IDS["VN"]
+
+        def contract_row(
+            country_id,
+            *,
+            source_code,
+            source_revision,
+            fingerprint,
+        ):
+            return {
+                "country_id": country_id,
+                "current_publication__source_code_snapshot": source_code,
+                "current_publication__source_revision": source_revision,
+                "current_publication__source_contract_fingerprint_sha256": fingerprint,
+                "current_publication__parser_contract_fingerprint_sha256": fingerprint,
+            }
+
         with (
             patch("travel_windows.search.PublishedEntryFacts") as entry_model,
             patch("travel_windows.search.PublishedTravelWarning") as warning_model,
         ):
-            entry_model.objects.filter.return_value.values_list.return_value = (
-                japan,
-                taiwan,
+            entry_model.objects.filter.return_value.values.return_value = (
+                contract_row(
+                    japan,
+                    source_code=LEGACY_ENTRY_SOURCE_CODE,
+                    source_revision=LEGACY_ENTRY_SOURCE_REVISION,
+                    fingerprint=LEGACY_ENTRY_CONTRACT_FINGERPRINT_SHA256,
+                ),
+                contract_row(
+                    taiwan,
+                    source_code=LEGACY_ENTRY_SOURCE_CODE,
+                    source_revision=LEGACY_ENTRY_SOURCE_REVISION,
+                    fingerprint=LEGACY_ENTRY_CONTRACT_FINGERPRINT_SHA256,
+                ),
+                contract_row(
+                    taiwan,
+                    source_code=MULTI_COUNTRY_ENTRY_SOURCE_CODE,
+                    source_revision=MULTI_COUNTRY_ENTRY_SOURCE_REVISION,
+                    fingerprint=MULTI_COUNTRY_ENTRY_CONTRACT_FINGERPRINT_SHA256,
+                ),
+                contract_row(
+                    vietnam,
+                    source_code=MULTI_COUNTRY_ENTRY_SOURCE_CODE,
+                    source_revision=MULTI_COUNTRY_ENTRY_SOURCE_REVISION,
+                    fingerprint=MULTI_COUNTRY_ENTRY_CONTRACT_FINGERPRINT_SHA256,
+                ),
             )
-            warning_model.objects.filter.return_value.values_list.return_value = (
-                japan,
-                vietnam,
+            warning_model.objects.filter.return_value.values.return_value = (
+                contract_row(
+                    japan,
+                    source_code=LEGACY_WARNING_SOURCE_CODE,
+                    source_revision=LEGACY_WARNING_SOURCE_REVISION,
+                    fingerprint=LEGACY_WARNING_CONTRACT_FINGERPRINT_SHA256,
+                ),
+                contract_row(
+                    taiwan,
+                    source_code=MULTI_COUNTRY_WARNING_SOURCE_CODE,
+                    source_revision=MULTI_COUNTRY_WARNING_SOURCE_REVISION,
+                    fingerprint=MULTI_COUNTRY_WARNING_CONTRACT_FINGERPRINT_SHA256,
+                ),
             )
 
             eligible = _load_eligible_country_ids()
 
-        self.assertEqual(eligible, frozenset({japan}))
+        self.assertEqual(eligible, frozenset({japan, taiwan}))
         entry_filters = entry_model.objects.filter.call_args.kwargs
         warning_filters = warning_model.objects.filter.call_args.kwargs
         self.assertFalse(entry_filters["current_publication__isnull"])
@@ -496,6 +625,35 @@ class PublicationEligibilityTests(SimpleTestCase):
             warning_filters["current_publication__state"], "PUBLISHED"
         )
 
+    def test_pre_crosscheck_current_revision_is_unavailable(self):
+        revision = SimpleNamespace(
+            state="VALIDATED",
+            completeness="COMPLETE",
+            city_reference_parse_run_id=None,
+            legacy_arrivals_parse_run_id=(
+                "57a68d56-4e7c-4a06-a8bd-f106e0ed4a61"
+            ),
+        )
+        publication = SimpleNamespace(
+            state="PUBLISHED",
+            schedule_revision=revision,
+            source_checked_at=datetime(2026, 8, 31, tzinfo=SEOUL),
+        )
+        pointer = SimpleNamespace(
+            current_publication_id=(
+                "6fc3778f-53a5-4dc0-9177-ac6b7f03299e"
+            ),
+            current_publication=publication,
+        )
+        with patch(
+            "travel_windows.search.PublishedFlightSchedule.objects"
+        ) as objects:
+            objects.select_related.return_value.filter.return_value.first.return_value = (
+                pointer
+            )
+            with self.assertRaises(_FlightEvidenceUnavailable):
+                _load_current_flight_evidence()
+
 
 def _official_row(
     *, airport_code, airport_name, flight_number, event_time, weekday
@@ -556,11 +714,25 @@ class CurrentFlightPublicationSearchTests(TransactionTestCase):
     def setUp(self):
         register_approved_sources(apply=True)
         row = next(row for row in CURATED_AIRPORT_ROWS if row[1] == "NRT")
+        country_row = next(
+            country
+            for country in SUPPORTED_COUNTRY_ROWS
+            if country[1] == row[2]
+        )
+        country, _ = Country.objects.get_or_create(
+            id=country_row[0],
+            defaults={
+                "iso_alpha2": country_row[1],
+                "name_ko": country_row[2],
+                "name_en": country_row[3],
+                "is_public": country_row[4],
+            },
+        )
         Airport.objects.get_or_create(
             id=row[0],
             defaults={
                 "iata_code": row[1],
-                "country": Country.objects.get(iso_alpha2=row[2]),
+                "country": country,
                 "city_code": row[3],
                 "city_name_ko": row[4],
                 "name_ko": row[5],
@@ -599,10 +771,28 @@ class CurrentFlightPublicationSearchTests(TransactionTestCase):
                     ]
                 ),
             ),
+            city_payload=city_reference_payload(),
+            legacy_arrival_pages=(
+                legacy_arrival_page(
+                    [
+                        legacy_arrival_row(
+                            st="0500",
+                            monday="N",
+                            tuesday="N",
+                            wednesday="N",
+                            thursday="Y",
+                            friday="N",
+                            saturday="N",
+                            sunday="N",
+                        )
+                    ]
+                ),
+            ),
             duration_csv=(
                 b"destination_iata,outbound_minutes,inbound_minutes,reference_date,"
                 b"reference_locator\r\nNRT,120,120,2026-08-01,"
-                b"https://www.ulip.go.kr/\r\n"
+                + DURATION_SOURCE_LOCATOR.encode("ascii")
+                + b"\r\n"
             ),
             source_date=SOURCE_DATE,
             published_by="synthetic-search-reviewer",


