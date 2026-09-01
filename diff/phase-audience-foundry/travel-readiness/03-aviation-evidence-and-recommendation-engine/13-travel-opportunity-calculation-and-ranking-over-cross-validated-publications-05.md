## `feat(search): accept verified warning snapshots`

diff --git a/travel_windows/search.py b/travel_windows/search.py
index dda6b80..400204f 100644
--- a/travel_windows/search.py
+++ b/travel_windows/search.py
@@ -9,7 +9,7 @@ from zoneinfo import ZoneInfo, ZoneInfoNotFoundError
 from django.db.models import F
 from django.utils import timezone
 
-from countries.models import JP_COUNTRY_ID
+from countries.models import JP_COUNTRY_ID, SUPPORTED_COUNTRY_IDS
 from entry_requirements.ingestion import (
     LEGACY_ENTRY_CONTRACT_FINGERPRINT_SHA256,
     LEGACY_ENTRY_SOURCE_CODE,
@@ -25,6 +25,18 @@ from reviews.models import (
     PublishedEntryFacts,
     PublishedTravelWarning,
 )
+from sources.models import FetchAttempt, ParseRun
+from sources.transport import (
+    WARNING_SOURCE_LOCATOR,
+    warning_snapshot_request_fingerprint,
+)
+from travel_warnings.contracts import (
+    WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256,
+    WARNING_SNAPSHOT_MAX_FACTS,
+    WARNING_SNAPSHOT_PARSER_CONTRACT_FINGERPRINT_SHA256,
+    WARNING_SNAPSHOT_SOURCE_CODE,
+    WARNING_SNAPSHOT_SOURCE_REVISION,
+)
 from travel_warnings.ingestion import (
     LEGACY_WARNING_CONTRACT_FINGERPRINT_SHA256,
     LEGACY_WARNING_SOURCE_CODE,
@@ -32,7 +44,10 @@ from travel_warnings.ingestion import (
     MULTI_COUNTRY_WARNING_CONTRACT_FINGERPRINT_SHA256,
     MULTI_COUNTRY_WARNING_SOURCE_CODE,
     MULTI_COUNTRY_WARNING_SOURCE_REVISION,
+    WARNING_SOURCE_MODULE,
+    WARNING_SOURCE_OWNER,
 )
+from travel_warnings.models import TravelWarningFact
 from travel_windows.models import (
     FLIGHT_POINTER_ID,
     FlightPublication,
@@ -135,6 +150,41 @@ class _Candidate:
 PublicationCardLoader = Callable[[UUID], dict[str, dict[str, object]]]
 
 
+def _is_sha256(value: object) -> bool:
+    return (
+        isinstance(value, str)
+        and len(value) == 64
+        and all(character in "0123456789abcdef" for character in value)
+    )
+
+
+def _is_warning_snapshot_contract_row(row: dict[str, object]) -> bool:
+    return (
+        row.get("current_publication__source_code_snapshot")
+        == WARNING_SNAPSHOT_SOURCE_CODE
+        or row.get("current_publication__parser_version")
+        == ParseRun.ParserVersion.V2
+        or row.get(
+            "current_publication__travel_warning_revision__parse_run__parser_version"
+        )
+        == ParseRun.ParserVersion.V2
+        or row.get(
+            "current_publication__travel_warning_revision__observation_attempt_id"
+        )
+        is not None
+    )
+
+
+def _warning_fact_positions(revision_id: object) -> tuple[int, ...] | None:
+    if not isinstance(revision_id, UUID):
+        return None
+    return tuple(
+        TravelWarningFact.objects.filter(revision_id=revision_id)
+        .order_by("source_position")
+        .values_list("source_position", flat=True)
+    )
+
+
 def _load_eligible_country_ids() -> frozenset[UUID]:
     entry_rows = (
         PublishedEntryFacts.objects.filter(
@@ -161,8 +211,50 @@ def _load_eligible_country_ids() -> frozenset[UUID]:
             "country_id",
             "current_publication__source_code_snapshot",
             "current_publication__source_revision",
+            "current_publication__source_owner_snapshot",
+            "current_publication__source_locator_snapshot",
+            "current_publication__attribution_text_snapshot",
             "current_publication__source_contract_fingerprint_sha256",
+            "current_publication__parser_name",
+            "current_publication__parser_version",
             "current_publication__parser_contract_fingerprint_sha256",
+            "current_publication__schema_fingerprint_sha256",
+            "current_publication__typed_fingerprint_sha256",
+            "current_publication__source_first_observed_at",
+            "current_publication__travel_warning_revision_id",
+            "current_publication__travel_warning_revision__country_id",
+            "current_publication__travel_warning_revision__source_alarm_level_code",
+            "current_publication__travel_warning_revision__source_scope_type",
+            "current_publication__travel_warning_revision__source_scope_text",
+            "current_publication__travel_warning_revision__source_written_date",
+            "current_publication__travel_warning_revision__source_item_count",
+            "current_publication__travel_warning_revision__observation_attempt_id",
+            "current_publication__travel_warning_revision__first_observed_at",
+            "current_publication__travel_warning_revision__typed_fingerprint_sha256",
+            "current_publication__travel_warning_revision__parse_run__outcome",
+            "current_publication__travel_warning_revision__parse_run__parser_name",
+            "current_publication__travel_warning_revision__parse_run__parser_version",
+            "current_publication__travel_warning_revision__parse_run__parser_contract_fingerprint_sha256",
+            "current_publication__travel_warning_revision__parse_run__expected_schema_fingerprint_sha256",
+            "current_publication__travel_warning_revision__parse_run__observed_schema_fingerprint_sha256",
+            "current_publication__travel_warning_revision__parse_run__artifact__body_sha256",
+            "current_publication__travel_warning_revision__parse_run__artifact__source_id",
+            "current_publication__travel_warning_revision__parse_run__artifact__source__code",
+            "current_publication__travel_warning_revision__parse_run__artifact__source__revision",
+            "current_publication__travel_warning_revision__parse_run__artifact__source__module",
+            "current_publication__travel_warning_revision__parse_run__artifact__source__owner",
+            "current_publication__travel_warning_revision__parse_run__artifact__source__official_locator",
+            "current_publication__travel_warning_revision__observation_attempt__source_id",
+            "current_publication__travel_warning_revision__observation_attempt__source_revision",
+            "current_publication__travel_warning_revision__observation_attempt__rights_decision_id",
+            "current_publication__travel_warning_revision__observation_attempt__request_fingerprint_revision",
+            "current_publication__travel_warning_revision__observation_attempt__normalized_request_sha256",
+            "current_publication__travel_warning_revision__observation_attempt__result",
+            "current_publication__travel_warning_revision__observation_attempt__http_status",
+            "current_publication__travel_warning_revision__observation_attempt__provider_code",
+            "current_publication__travel_warning_revision__observation_attempt__completed_at",
+            "current_publication__travel_warning_revision__observation_attempt__response_sha256",
+            "country__iso_alpha2",
         )
     )
     entry_country_ids = {
@@ -176,13 +268,25 @@ def _load_eligible_country_ids() -> frozenset[UUID]:
         if _publication_contract_allowed(
             row,
             module=PublicationModule.TRAVEL_WARNING,
+            warning_fact_positions=(
+                _warning_fact_positions(
+                    row.get(
+                        "current_publication__travel_warning_revision_id"
+                    )
+                )
+                if _is_warning_snapshot_contract_row(row)
+                else None
+            ),
         )
     }
     return frozenset(entry_country_ids & warning_country_ids)
 
 
 def _publication_contract_allowed(
-    row: dict[str, object], *, module: str
+    row: dict[str, object],
+    *,
+    module: str,
+    warning_fact_positions: tuple[int, ...] | None = None,
 ) -> bool:
     country_id = row.get("country_id")
     if not isinstance(country_id, UUID):
@@ -199,6 +303,11 @@ def _publication_contract_allowed(
             MULTI_COUNTRY_ENTRY_CONTRACT_FINGERPRINT_SHA256,
         )
     elif module == PublicationModule.TRAVEL_WARNING:
+        if _is_warning_snapshot_contract_row(row):
+            return _warning_snapshot_contract_allowed(
+                row,
+                fact_positions=warning_fact_positions,
+            )
         legacy = (
             LEGACY_WARNING_SOURCE_CODE,
             LEGACY_WARNING_SOURCE_REVISION,
@@ -230,6 +339,110 @@ def _publication_contract_allowed(
     )
 
 
+def _warning_snapshot_contract_allowed(
+    row: dict[str, object],
+    *,
+    fact_positions: tuple[int, ...] | None,
+) -> bool:
+    prefix = "current_publication__travel_warning_revision"
+    artifact_prefix = f"{prefix}__parse_run__artifact"
+    observation_prefix = f"{prefix}__observation_attempt"
+    country_id = row.get("country_id")
+    source_item_count = row.get(f"{prefix}__source_item_count")
+    try:
+        request_fingerprint = warning_snapshot_request_fingerprint(
+            row.get("country__iso_alpha2")
+        )
+    except ValueError:
+        return False
+    return (
+        isinstance(country_id, UUID)
+        and type(source_item_count) is int
+        and 0 <= source_item_count <= WARNING_SNAPSHOT_MAX_FACTS
+        and fact_positions == tuple(range(1, source_item_count + 1))
+        and row.get("current_publication__source_code_snapshot")
+        == WARNING_SNAPSHOT_SOURCE_CODE
+        and row.get("current_publication__source_revision")
+        == WARNING_SNAPSHOT_SOURCE_REVISION
+        and row.get("current_publication__source_owner_snapshot")
+        == WARNING_SOURCE_OWNER
+        and row.get("current_publication__source_locator_snapshot")
+        == WARNING_SOURCE_LOCATOR
+        and row.get("current_publication__attribution_text_snapshot")
+        == "외교부|공공데이터포털"
+        and row.get("current_publication__source_contract_fingerprint_sha256")
+        == WARNING_SNAPSHOT_PARSER_CONTRACT_FINGERPRINT_SHA256
+        and row.get("current_publication__parser_name")
+        == ParseRun.ParserName.MOFA_TRAVEL_ALARM_JSON
+        and row.get("current_publication__parser_version")
+        == ParseRun.ParserVersion.V2
+        and row.get("current_publication__parser_contract_fingerprint_sha256")
+        == WARNING_SNAPSHOT_PARSER_CONTRACT_FINGERPRINT_SHA256
+        and row.get("current_publication__schema_fingerprint_sha256")
+        == WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256
+        and SUPPORTED_COUNTRY_IDS.get(row.get("country__iso_alpha2"))
+        == country_id
+        and row.get(f"{prefix}__country_id") == country_id
+        and isinstance(
+            row.get(f"{prefix}__observation_attempt_id"),
+            UUID,
+        )
+        and row.get(f"{prefix}__source_alarm_level_code") is None
+        and row.get(f"{prefix}__source_scope_type") is None
+        and row.get(f"{prefix}__source_scope_text") is None
+        and row.get(f"{prefix}__source_written_date") is None
+        and row.get(f"{prefix}__parse_run__outcome")
+        == ParseRun.Outcome.VALIDATED
+        and row.get(f"{prefix}__parse_run__parser_name")
+        == ParseRun.ParserName.MOFA_TRAVEL_ALARM_JSON
+        and row.get(f"{prefix}__parse_run__parser_version")
+        == ParseRun.ParserVersion.V2
+        and row.get(f"{prefix}__parse_run__parser_contract_fingerprint_sha256")
+        == WARNING_SNAPSHOT_PARSER_CONTRACT_FINGERPRINT_SHA256
+        and row.get(f"{prefix}__parse_run__expected_schema_fingerprint_sha256")
+        == WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256
+        and row.get(f"{prefix}__parse_run__observed_schema_fingerprint_sha256")
+        == WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256
+        and row.get(f"{artifact_prefix}__source__code")
+        == WARNING_SNAPSHOT_SOURCE_CODE
+        and row.get(f"{artifact_prefix}__source__revision")
+        == WARNING_SNAPSHOT_SOURCE_REVISION
+        and row.get(f"{artifact_prefix}__source__module")
+        == WARNING_SOURCE_MODULE
+        and row.get(f"{artifact_prefix}__source__owner")
+        == WARNING_SOURCE_OWNER
+        and row.get(f"{artifact_prefix}__source__official_locator")
+        == WARNING_SOURCE_LOCATOR
+        and isinstance(row.get(f"{artifact_prefix}__source_id"), UUID)
+        and row.get(f"{observation_prefix}__source_id")
+        == row.get(f"{artifact_prefix}__source_id")
+        and row.get(f"{observation_prefix}__source_revision")
+        == WARNING_SNAPSHOT_SOURCE_REVISION
+        and isinstance(
+            row.get(f"{observation_prefix}__rights_decision_id"),
+            UUID,
+        )
+        and row.get(f"{observation_prefix}__request_fingerprint_revision")
+        == request_fingerprint.revision
+        and row.get(f"{observation_prefix}__normalized_request_sha256")
+        == request_fingerprint.normalized_request_sha256
+        and row.get(f"{observation_prefix}__result")
+        == FetchAttempt.Result.SUCCEEDED
+        and row.get(f"{observation_prefix}__http_status") == 200
+        and row.get(f"{observation_prefix}__provider_code")
+        == FetchAttempt.ProviderCode.MOFA_SUCCESS_0
+        and row.get(f"{observation_prefix}__completed_at")
+        == row.get(f"{prefix}__first_observed_at")
+        == row.get("current_publication__source_first_observed_at")
+        and row.get(f"{observation_prefix}__response_sha256")
+        == row.get(f"{artifact_prefix}__body_sha256")
+        and _is_sha256(row.get(f"{artifact_prefix}__body_sha256"))
+        and row.get("current_publication__typed_fingerprint_sha256")
+        == row.get(f"{prefix}__typed_fingerprint_sha256")
+        and _is_sha256(row.get(f"{prefix}__typed_fingerprint_sha256"))
+    )
+
+
 def _load_current_flight_evidence() -> _CurrentFlightEvidence | None:
     pointer = (
         PublishedFlightSchedule.objects.select_related(
diff --git a/travel_windows/tests/test_search.py b/travel_windows/tests/test_search.py
index 38b9878..bd34fc3 100644
--- a/travel_windows/tests/test_search.py
+++ b/travel_windows/tests/test_search.py
@@ -2,6 +2,7 @@ import json
 from datetime import date, datetime, time
 from types import SimpleNamespace
 from unittest.mock import patch
+from uuid import UUID
 from zoneinfo import ZoneInfo
 
 from django.contrib.auth import get_user_model
@@ -57,6 +58,16 @@ from travel_windows.tests.test_source_publication import (
     legacy_arrival_row,
     register_sources_with_duration_v1_history,
 )
+from sources.transport import (
+    WARNING_SOURCE_LOCATOR,
+    warning_snapshot_request_fingerprint,
+)
+from travel_warnings.contracts import (
+    WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256,
+    WARNING_SNAPSHOT_PARSER_CONTRACT_FINGERPRINT_SHA256,
+    WARNING_SNAPSHOT_SOURCE_CODE,
+    WARNING_SNAPSHOT_SOURCE_REVISION,
+)
 from travel_warnings.ingestion import (
     LEGACY_WARNING_CONTRACT_FINGERPRINT_SHA256,
     LEGACY_WARNING_SOURCE_CODE,
@@ -64,6 +75,8 @@ from travel_warnings.ingestion import (
     MULTI_COUNTRY_WARNING_CONTRACT_FINGERPRINT_SHA256,
     MULTI_COUNTRY_WARNING_SOURCE_CODE,
     MULTI_COUNTRY_WARNING_SOURCE_REVISION,
+    WARNING_SOURCE_MODULE,
+    WARNING_SOURCE_OWNER,
 )
 
 
@@ -658,6 +671,175 @@ class PublicationEligibilityTests(SimpleTestCase):
             warning_filters["current_publication__state"], "PUBLISHED"
         )
 
+    def test_warning_snapshot_zero_and_ordered_facts_are_eligible(self):
+        taiwan = SUPPORTED_COUNTRY_IDS["TW"]
+        hong_kong = SUPPORTED_COUNTRY_IDS["HK"]
+        macau = SUPPORTED_COUNTRY_IDS["MO"]
+        vietnam = SUPPORTED_COUNTRY_IDS["VN"]
+        observed_at = datetime(2026, 8, 31, tzinfo=SEOUL)
+        source_id = UUID("5ca7e28c-81df-40ea-b77b-69b95860949f")
+        rights_id = UUID("b4261766-5d04-46e4-a362-d8b6e7a244db")
+
+        def entry_row(country_id):
+            return {
+                "country_id": country_id,
+                "current_publication__source_code_snapshot": (
+                    MULTI_COUNTRY_ENTRY_SOURCE_CODE
+                ),
+                "current_publication__source_revision": (
+                    MULTI_COUNTRY_ENTRY_SOURCE_REVISION
+                ),
+                "current_publication__source_contract_fingerprint_sha256": (
+                    MULTI_COUNTRY_ENTRY_CONTRACT_FINGERPRINT_SHA256
+                ),
+                "current_publication__parser_contract_fingerprint_sha256": (
+                    MULTI_COUNTRY_ENTRY_CONTRACT_FINGERPRINT_SHA256
+                ),
+            }
+
+        def snapshot_row(country_id, iso_alpha2, revision_id, item_count):
+            request = warning_snapshot_request_fingerprint(iso_alpha2)
+            typed_fingerprint = "d" * 64
+            body_sha = "e" * 64
+            prefix = "current_publication__travel_warning_revision"
+            artifact_prefix = f"{prefix}__parse_run__artifact"
+            observation_prefix = f"{prefix}__observation_attempt"
+            return {
+                "country_id": country_id,
+                "country__iso_alpha2": iso_alpha2,
+                "current_publication__source_code_snapshot": (
+                    WARNING_SNAPSHOT_SOURCE_CODE
+                ),
+                "current_publication__source_revision": (
+                    WARNING_SNAPSHOT_SOURCE_REVISION
+                ),
+                "current_publication__source_owner_snapshot": (
+                    WARNING_SOURCE_OWNER
+                ),
+                "current_publication__source_locator_snapshot": (
+                    WARNING_SOURCE_LOCATOR
+                ),
+                "current_publication__attribution_text_snapshot": (
+                    "외교부|공공데이터포털"
+                ),
+                "current_publication__source_contract_fingerprint_sha256": (
+                    WARNING_SNAPSHOT_PARSER_CONTRACT_FINGERPRINT_SHA256
+                ),
+                "current_publication__parser_name": "MOFA_TRAVEL_ALARM_JSON",
+                "current_publication__parser_version": "V2",
+                "current_publication__parser_contract_fingerprint_sha256": (
+                    WARNING_SNAPSHOT_PARSER_CONTRACT_FINGERPRINT_SHA256
+                ),
+                "current_publication__schema_fingerprint_sha256": (
+                    WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256
+                ),
+                "current_publication__typed_fingerprint_sha256": (
+                    typed_fingerprint
+                ),
+                "current_publication__source_first_observed_at": observed_at,
+                "current_publication__travel_warning_revision_id": (
+                    revision_id
+                ),
+                f"{prefix}__country_id": country_id,
+                f"{prefix}__source_alarm_level_code": None,
+                f"{prefix}__source_scope_type": None,
+                f"{prefix}__source_scope_text": None,
+                f"{prefix}__source_written_date": None,
+                f"{prefix}__source_item_count": item_count,
+                f"{prefix}__observation_attempt_id": UUID(
+                    "42359986-f422-44cc-8699-2fa3898dd9c1"
+                ),
+                f"{prefix}__first_observed_at": observed_at,
+                f"{prefix}__typed_fingerprint_sha256": typed_fingerprint,
+                f"{prefix}__parse_run__outcome": "VALIDATED",
+                f"{prefix}__parse_run__parser_name": (
+                    "MOFA_TRAVEL_ALARM_JSON"
+                ),
+                f"{prefix}__parse_run__parser_version": "V2",
+                f"{prefix}__parse_run__parser_contract_fingerprint_sha256": (
+                    WARNING_SNAPSHOT_PARSER_CONTRACT_FINGERPRINT_SHA256
+                ),
+                f"{prefix}__parse_run__expected_schema_fingerprint_sha256": (
+                    WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256
+                ),
+                f"{prefix}__parse_run__observed_schema_fingerprint_sha256": (
+                    WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256
+                ),
+                f"{artifact_prefix}__body_sha256": body_sha,
+                f"{artifact_prefix}__source_id": source_id,
+                f"{artifact_prefix}__source__code": (
+                    WARNING_SNAPSHOT_SOURCE_CODE
+                ),
+                f"{artifact_prefix}__source__revision": (
+                    WARNING_SNAPSHOT_SOURCE_REVISION
+                ),
+                f"{artifact_prefix}__source__module": WARNING_SOURCE_MODULE,
+                f"{artifact_prefix}__source__owner": WARNING_SOURCE_OWNER,
+                f"{artifact_prefix}__source__official_locator": (
+                    WARNING_SOURCE_LOCATOR
+                ),
+                f"{observation_prefix}__source_id": source_id,
+                f"{observation_prefix}__source_revision": (
+                    WARNING_SNAPSHOT_SOURCE_REVISION
+                ),
+                f"{observation_prefix}__rights_decision_id": rights_id,
+                f"{observation_prefix}__request_fingerprint_revision": (
+                    request.revision
+                ),
+                f"{observation_prefix}__normalized_request_sha256": (
+                    request.normalized_request_sha256
+                ),
+                f"{observation_prefix}__result": "SUCCEEDED",
+                f"{observation_prefix}__http_status": 200,
+                f"{observation_prefix}__provider_code": "MOFA_SUCCESS_0",
+                f"{observation_prefix}__completed_at": observed_at,
+                f"{observation_prefix}__response_sha256": body_sha,
+            }
+
+        revision_ids = {
+            "TW": UUID("381f627d-3ec7-449a-aae0-ffbda69b1f57"),
+            "HK": UUID("9c9c00f0-8f94-42cf-a5d7-d2dddc2cc328"),
+            "MO": UUID("de9719c8-d48a-43b8-a8bd-02330efb5a7b"),
+            "VN": UUID("990e9ad6-fd4d-4561-a007-3f167fa1e0ca"),
+        }
+        warning_rows = [
+            snapshot_row(taiwan, "TW", revision_ids["TW"], 0),
+            snapshot_row(hong_kong, "HK", revision_ids["HK"], 2),
+            snapshot_row(macau, "MO", revision_ids["MO"], 1),
+            snapshot_row(vietnam, "VN", revision_ids["VN"], 0),
+        ]
+        warning_rows[-1][
+            "current_publication__travel_warning_revision__observation_attempt__normalized_request_sha256"
+        ] = "f" * 64
+        positions = {
+            revision_ids["TW"]: (),
+            revision_ids["HK"]: (1, 2),
+            revision_ids["MO"]: (2,),
+            revision_ids["VN"]: (),
+        }
+
+        with (
+            patch("travel_windows.search.PublishedEntryFacts") as entry_model,
+            patch(
+                "travel_windows.search.PublishedTravelWarning"
+            ) as warning_model,
+            patch(
+                "travel_windows.search._warning_fact_positions",
+                side_effect=lambda revision_id: positions[revision_id],
+            ),
+        ):
+            entry_model.objects.filter.return_value.values.return_value = tuple(
+                entry_row(country_id)
+                for country_id in (taiwan, hong_kong, macau, vietnam)
+            )
+            warning_model.objects.filter.return_value.values.return_value = (
+                warning_rows
+            )
+
+            eligible = _load_eligible_country_ids()
+
+        self.assertEqual(eligible, frozenset({taiwan, hong_kong}))
+
     def test_pre_crosscheck_current_revision_is_unavailable(self):
         revision = SimpleNamespace(
             state="VALIDATED",
