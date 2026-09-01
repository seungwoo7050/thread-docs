## `feat(web): present official warning snapshots`

diff --git a/e2e/browser_acceptance.py b/e2e/browser_acceptance.py
index 4a0791b..bbecd42 100644
--- a/e2e/browser_acceptance.py
+++ b/e2e/browser_acceptance.py
@@ -185,8 +185,8 @@ WARNING_PUBLIC_SOURCE_LOCATOR: Final = (
 )
 ENTRY_FRESHNESS_MINUTES: Final = 36 * 60
 WARNING_FRESHNESS_MINUTES: Final = 8 * 60
-SITE_CSS_SHA256: Final = "30310db2f9c4070fc754972c8196ace469a7dc5ae2a65e0dcc04d274739ddd46"
-SITE_CSS_BYTES: Final = 17_870
+SITE_CSS_SHA256: Final = "021afa2b4502f425014fb32b8177f06c746cbf9064261aeb48a2a80c977fe3a4"
+SITE_CSS_BYTES: Final = 18_262
 SITE_JS_SHA256: Final = "1a5a95b286d6e12f1765351d5a0b1528c833fb46af16d747b469b61323cc7f44"
 SITE_JS_BYTES: Final = 1_628
 _SIGNAL_INTERRUPTED = False
diff --git a/e2e/browser_scenario_server.py b/e2e/browser_scenario_server.py
index 54732ca..eda3e54 100644
--- a/e2e/browser_scenario_server.py
+++ b/e2e/browser_scenario_server.py
@@ -57,8 +57,8 @@ SITE_ASSETS: Final = {
     "/static/public_web/site.css": (
         REPOSITORY_ROOT / "public_web" / "static" / "public_web" / "site.css",
         "text/css",
-        17_870,
-        "30310db2f9c4070fc754972c8196ace469a7dc5ae2a65e0dcc04d274739ddd46",
+        18_262,
+        "021afa2b4502f425014fb32b8177f06c746cbf9064261aeb48a2a80c977fe3a4",
     ),
     "/static/public_web/site.js": (
         REPOSITORY_ROOT / "public_web" / "static" / "public_web" / "site.js",
diff --git a/public_web/results.py b/public_web/results.py
index 4c25d78..f37ae5e 100644
--- a/public_web/results.py
+++ b/public_web/results.py
@@ -34,11 +34,25 @@ from reviews.models import (
 )
 from sources.models import (
     FetchAttempt,
+    ParseRun,
     SourceArtifact,
     SourceConfiguration,
     SourceRightsDecision,
 )
-from sources.transport import ENTRY_SOURCE_LOCATOR, WARNING_SOURCE_LOCATOR
+from sources.transport import (
+    ENTRY_SOURCE_LOCATOR,
+    WARNING_SOURCE_LOCATOR,
+    warning_snapshot_request_fingerprint,
+)
+from travel_warnings.contracts import (
+    WARNING_SNAPSHOT_DECISION_BASIS,
+    WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256,
+    WARNING_SNAPSHOT_FIELD_SCOPE,
+    WARNING_SNAPSHOT_MAX_FACTS,
+    WARNING_SNAPSHOT_PARSER_CONTRACT_FINGERPRINT_SHA256,
+    WARNING_SNAPSHOT_SOURCE_CODE,
+    WARNING_SNAPSHOT_SOURCE_REVISION,
+)
 from travel_warnings.ingestion import (
     LEGACY_WARNING_CONTRACT_FINGERPRINT_SHA256,
     LEGACY_WARNING_FIELD_SCOPE,
@@ -54,6 +68,7 @@ from travel_warnings.ingestion import (
     WARNING_SOURCE_OWNER,
     WARNING_SOURCE_REVISION,
 )
+from travel_warnings.models import TravelWarningFact
 from travel_warnings.parser import PARSER_CONTRACT_FINGERPRINT_SHA256
 
 
@@ -129,6 +144,23 @@ _FRESHNESS_CONTRACTS = {
         provider_code=FetchAttempt.ProviderCode.MOFA_SUCCESS_0,
         max_success_age=timedelta(hours=8),
     ),
+    "warning_snapshot": _FreshnessContract(
+        source_code=WARNING_SNAPSHOT_SOURCE_CODE,
+        source_revision=WARNING_SNAPSHOT_SOURCE_REVISION,
+        source_module=WARNING_SOURCE_MODULE,
+        source_owner=WARNING_SOURCE_OWNER,
+        source_locator=WARNING_SOURCE_LOCATOR,
+        access_mode=SourceRightsDecision.AccessMode.CREDENTIAL_REFERENCE,
+        field_scope=WARNING_SNAPSHOT_FIELD_SCOPE,
+        contract_fingerprint_sha256=(
+            WARNING_SNAPSHOT_PARSER_CONTRACT_FINGERPRINT_SHA256
+        ),
+        attribution=WARNING_ATTRIBUTION,
+        decided_by=WARNING_DECIDED_BY,
+        decision_basis=WARNING_SNAPSHOT_DECISION_BASIS,
+        provider_code=FetchAttempt.ProviderCode.MOFA_SUCCESS_0,
+        max_success_age=timedelta(hours=8),
+    ),
     "warning_legacy": _FreshnessContract(
         source_code=LEGACY_WARNING_SOURCE_CODE,
         source_revision=LEGACY_WARNING_SOURCE_REVISION,
@@ -167,6 +199,8 @@ def _freshness_contract_for_publication(
             return _FRESHNESS_CONTRACTS["entry_legacy"]
         return None
     if module == "warning":
+        if source_code == WARNING_SNAPSHOT_SOURCE_CODE:
+            return _FRESHNESS_CONTRACTS["warning_snapshot"]
         if source_code == MULTI_COUNTRY_WARNING_SOURCE_CODE:
             return _FRESHNESS_CONTRACTS["warning"]
         if (
@@ -236,6 +270,14 @@ def _iso_date(value: date | None) -> str | None:
     return value.isoformat() if value is not None else None
 
 
+def _is_sha256(value: object) -> bool:
+    return (
+        isinstance(value, str)
+        and len(value) == 64
+        and all(character in "0123456789abcdef" for character in value)
+    )
+
+
 def _utc_minute(value: datetime | None) -> str | None:
     if value is None:
         return None
@@ -460,7 +502,7 @@ def _entry_row(country_id: UUID) -> dict | None:
 
 
 def _warning_row(country_id: UUID) -> dict | None:
-    return (
+    row = (
         PublishedTravelWarning.objects.filter(country_id=country_id)
         .values(
             "current_publication_id",
@@ -473,11 +515,30 @@ def _warning_row(country_id: UUID) -> dict | None:
             "current_publication__source_locator_snapshot",
             "current_publication__attribution_text_snapshot",
             "current_publication__source_contract_fingerprint_sha256",
+            "current_publication__parser_name",
+            "current_publication__parser_version",
+            "current_publication__parser_contract_fingerprint_sha256",
+            "current_publication__schema_fingerprint_sha256",
+            "current_publication__typed_fingerprint_sha256",
+            "current_publication__source_first_observed_at",
+            "current_publication__travel_warning_revision_id",
+            "current_publication__travel_warning_revision__country_id",
             "current_publication__travel_warning_revision__source_alarm_level_code",
             "current_publication__travel_warning_revision__source_scope_type",
             "current_publication__travel_warning_revision__source_scope_text",
             "current_publication__travel_warning_revision__source_written_date",
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
             "current_publication__travel_warning_revision__parse_run__artifact_id",
+            "current_publication__travel_warning_revision__parse_run__artifact__body_sha256",
             "current_publication__travel_warning_revision__parse_run__artifact__source_id",
             "current_publication__travel_warning_revision__parse_run__artifact__source__code",
             "current_publication__travel_warning_revision__parse_run__artifact__source__revision",
@@ -488,9 +549,184 @@ def _warning_row(country_id: UUID) -> dict | None:
             "current_publication__travel_warning_revision__parse_run__artifact__source__enabled",
             "current_publication__travel_warning_revision__parse_run__artifact__first_successful_attempt__request_fingerprint_revision",
             "current_publication__travel_warning_revision__parse_run__artifact__first_successful_attempt__normalized_request_sha256",
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
         .first()
     )
+    if row is None:
+        return None
+    revision_id = row[
+        "current_publication__travel_warning_revision_id"
+    ]
+    row["warning_facts"] = tuple(
+        TravelWarningFact.objects.filter(revision_id=revision_id)
+        .order_by("source_position")
+        .values(
+            "source_position",
+            "source_alarm_level_code",
+            "source_scope_type",
+            "source_scope_text",
+            "source_written_date",
+        )
+    ) if revision_id is not None else ()
+    return row
+
+
+def _is_warning_snapshot_row(row: dict[str, object]) -> bool:
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
+def _snapshot_warning_facts(
+    row: dict[str, object],
+    *,
+    country_id: UUID,
+) -> tuple[dict[str, object], ...]:
+    prefix = "current_publication__travel_warning_revision"
+    artifact_prefix = f"{prefix}__parse_run__artifact"
+    observation_prefix = f"{prefix}__observation_attempt"
+    source_item_count = row.get(f"{prefix}__source_item_count")
+    raw_facts = row.get("warning_facts")
+    request_fingerprint = warning_snapshot_request_fingerprint(
+        row.get("country__iso_alpha2")
+    )
+    exact_identity = (
+        row.get("current_publication__source_code_snapshot")
+        == WARNING_SNAPSHOT_SOURCE_CODE
+        and row.get("current_publication__source_revision")
+        == WARNING_SNAPSHOT_SOURCE_REVISION
+        and row.get("current_publication__source_owner_snapshot")
+        == WARNING_SOURCE_OWNER
+        and row.get("current_publication__source_locator_snapshot")
+        == WARNING_SOURCE_LOCATOR
+        and row.get("current_publication__attribution_text_snapshot")
+        == WARNING_ATTRIBUTION
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
+    if (
+        not exact_identity
+        or type(source_item_count) is not int
+        or not 0 <= source_item_count <= WARNING_SNAPSHOT_MAX_FACTS
+        or not isinstance(raw_facts, (tuple, list))
+        or len(raw_facts) != source_item_count
+    ):
+        raise ValueError("invalid warning snapshot publication lineage")
+
+    facts: list[dict[str, object]] = []
+    for expected_position, raw_fact in enumerate(raw_facts, start=1):
+        written_date = raw_fact.get("source_written_date")
+        if (
+            raw_fact.get("source_position") != expected_position
+            or not isinstance(raw_fact.get("source_alarm_level_code"), str)
+            or not raw_fact["source_alarm_level_code"].strip()
+            or not isinstance(raw_fact.get("source_scope_type"), str)
+            or not raw_fact["source_scope_type"].strip()
+            or not isinstance(raw_fact.get("source_scope_text"), str)
+            or not raw_fact["source_scope_text"].strip()
+            or (written_date is not None and type(written_date) is not date)
+        ):
+            raise ValueError("invalid warning snapshot fact order or shape")
+        facts.append(
+            {
+                "source_position": expected_position,
+                "alarm_level_code": raw_fact["source_alarm_level_code"],
+                "scope_type": raw_fact["source_scope_type"],
+                "scope_text": raw_fact["source_scope_text"],
+                "written_date": _iso_date(written_date),
+            }
+        )
+    return tuple(facts)
 
 
 def _load_entry_card(country_id: UUID) -> dict[str, object]:
@@ -582,6 +818,29 @@ def _load_warning_card(country_id: UUID) -> dict[str, object]:
         return _state_card("warning", CARD_UNAVAILABLE)
     if row["current_publication_id"] is None:
         return _state_card("warning", CARD_EMPTY)
+    is_snapshot = _is_warning_snapshot_row(row)
+    if is_snapshot:
+        facts = _snapshot_warning_facts(row, country_id=country_id)
+    else:
+        facts = (
+            {
+                "source_position": 1,
+                "alarm_level_code": row[
+                    "current_publication__travel_warning_revision__source_alarm_level_code"
+                ],
+                "scope_type": row[
+                    "current_publication__travel_warning_revision__source_scope_type"
+                ],
+                "scope_text": row[
+                    "current_publication__travel_warning_revision__source_scope_text"
+                ],
+                "written_date": _iso_date(
+                    row[
+                        "current_publication__travel_warning_revision__source_written_date"
+                    ]
+                ),
+            },
+        )
     contract = _freshness_contract_for_publication(
         module="warning",
         country_id=country_id,
@@ -593,14 +852,19 @@ def _load_warning_card(country_id: UUID) -> dict[str, object]:
         prefix = (
             "current_publication__travel_warning_revision__parse_run__artifact"
         )
+        request_prefix = (
+            "current_publication__travel_warning_revision__observation_attempt"
+            if is_snapshot
+            else f"{prefix}__first_successful_attempt"
+        )
         state, checked_at = _freshness(
             contract=contract,
             artifact_id=row[f"{prefix}_id"],
             artifact_request_fingerprint_revision=row[
-                f"{prefix}__first_successful_attempt__request_fingerprint_revision"
+                f"{request_prefix}__request_fingerprint_revision"
             ],
             artifact_normalized_request_sha256=row[
-                f"{prefix}__first_successful_attempt__normalized_request_sha256"
+                f"{request_prefix}__normalized_request_sha256"
             ],
             source_id=row[f"{prefix}__source_id"],
             publication_source_code=row[
@@ -630,17 +894,28 @@ def _load_warning_card(country_id: UUID) -> dict[str, object]:
             source_enabled=row[f"{prefix}__source__enabled"],
         )
     stale = state == CARD_STALE
-    return {
+    verified_empty = is_snapshot and not facts
+    card = {
         "module": "warning",
         "heading": "여행경보",
         "state": state,
         "status_label": "재확인 필요" if stale else "현재 게시본",
         "message": (
-            "마지막으로 검수·게시된 여행경보입니다. 공식 출처에서 최신 정보를 재확인해 주세요."
-            if stale
-            else "입국요건과 별도로 공식 출처에서 수집해 검수·게시한 여행경보입니다."
+            (
+                "공식 출처의 이 국가 전체 조회에서 게시할 여행경보 항목이 0건으로 "
+                "확인되어 검수·게시됐습니다. 이는 안전함이나 여행 가능 여부를 뜻하지 "
+                "않으므로 공식 출처를 다시 확인해 주세요."
+            )
+            if verified_empty
+            else (
+                "마지막으로 검수·게시된 여행경보입니다. 공식 출처에서 최신 정보를 재확인해 주세요."
+                if stale
+                else "입국요건과 별도로 공식 출처에서 수집해 검수·게시한 여행경보입니다."
+            )
         ),
         "has_publication": True,
+        "verified_empty": verified_empty,
+        "facts": facts,
         "country_name": row["current_publication__scope_country__name_ko"],
         "generation": row["current_publication__generation"],
         "published_at": _utc_minute(row["current_publication__published_at"]),
@@ -649,21 +924,17 @@ def _load_warning_card(country_id: UUID) -> dict[str, object]:
         "source_locator": row["current_publication__source_locator_snapshot"],
         "attribution": row["current_publication__attribution_text_snapshot"],
         "checked_at": _utc_minute(checked_at),
-        "alarm_level_code": row[
-            "current_publication__travel_warning_revision__source_alarm_level_code"
-        ],
-        "scope_type": row[
-            "current_publication__travel_warning_revision__source_scope_type"
-        ],
-        "scope_text": row[
-            "current_publication__travel_warning_revision__source_scope_text"
-        ],
-        "written_date": _iso_date(
-            row[
-                "current_publication__travel_warning_revision__source_written_date"
-            ]
-        ),
     }
+    if not is_snapshot:
+        card.update(
+            {
+                "alarm_level_code": facts[0]["alarm_level_code"],
+                "scope_type": facts[0]["scope_type"],
+                "scope_text": facts[0]["scope_text"],
+                "written_date": facts[0]["written_date"],
+            }
+        )
+    return card
 
 
 def _safe_card(
diff --git a/public_web/static/public_web/site.css b/public_web/static/public_web/site.css
index 07bebf9..6219834 100644
--- a/public_web/static/public_web/site.css
+++ b/public_web/static/public_web/site.css
@@ -797,6 +797,27 @@ h3 {
   color: var(--muted-ink);
 }
 
+.warning-fact-list {
+  margin: var(--space-4) 0 0;
+  padding-left: 1.5rem;
+}
+
+.warning-fact-list > li + li {
+  margin-top: var(--space-4);
+}
+
+.warning-fact-list .brief-facts {
+  margin-top: 0;
+}
+
+.verified-empty-note {
+  margin: var(--space-4) 0 0;
+  padding: var(--space-3) var(--space-4);
+  background: var(--surface-muted);
+  border-left: 0.25rem solid var(--line);
+  font-weight: 800;
+}
+
 .brief-facts,
 .source-history dl {
   min-width: 0;
diff --git a/public_web/templates/public_web/partials/warning_card.html b/public_web/templates/public_web/partials/warning_card.html
index b27c645..e751e61 100644
--- a/public_web/templates/public_web/partials/warning_card.html
+++ b/public_web/templates/public_web/partials/warning_card.html
@@ -7,12 +7,29 @@
   </header>
   <p class="publication-message">{{ warning_card.message }}</p>
   {% if warning_card.has_publication %}
-    <dl class="brief-facts">
-      <div><dt>출처 경보 단계 코드</dt><dd>{{ warning_card.alarm_level_code }}</dd></div>
-      <div><dt>출처 범위 유형</dt><dd>{{ warning_card.scope_type }}</dd></div>
-      <div><dt>출처 범위</dt><dd>{{ warning_card.scope_text }}</dd></div>
-      <div><dt>출처 작성일</dt><dd>{{ warning_card.written_date|default:"출처가 제공하지 않음" }}</dd></div>
-    </dl>
+    {% if warning_card.verified_empty %}
+      <p class="verified-empty-note">공식 출처의 국가 전체 조회에서 확인된 항목 수: 0건</p>
+    {% elif warning_card.facts %}
+      <ol class="warning-fact-list" aria-label="출처 제공 순서의 여행경보 항목">
+        {% for fact in warning_card.facts %}
+          <li>
+            <dl class="brief-facts">
+              <div><dt>출처 경보 단계 코드</dt><dd>{{ fact.alarm_level_code }}</dd></div>
+              <div><dt>출처 범위 유형</dt><dd>{{ fact.scope_type }}</dd></div>
+              <div><dt>출처 범위</dt><dd>{{ fact.scope_text }}</dd></div>
+              <div><dt>출처 작성일</dt><dd>{{ fact.written_date|default:"출처가 제공하지 않음" }}</dd></div>
+            </dl>
+          </li>
+        {% endfor %}
+      </ol>
+    {% else %}
+      <dl class="brief-facts">
+        <div><dt>출처 경보 단계 코드</dt><dd>{{ warning_card.alarm_level_code }}</dd></div>
+        <div><dt>출처 범위 유형</dt><dd>{{ warning_card.scope_type }}</dd></div>
+        <div><dt>출처 범위</dt><dd>{{ warning_card.scope_text }}</dd></div>
+        <div><dt>출처 작성일</dt><dd>{{ warning_card.written_date|default:"출처가 제공하지 않음" }}</dd></div>
+      </dl>
+    {% endif %}
     <details class="source-history">
       <summary>출처와 확인 이력 보기</summary>
       <dl>
diff --git a/public_web/tests/test_accessibility_contract.py b/public_web/tests/test_accessibility_contract.py
index 6da149d..016d1ba 100644
--- a/public_web/tests/test_accessibility_contract.py
+++ b/public_web/tests/test_accessibility_contract.py
@@ -1,3 +1,4 @@
+from copy import deepcopy
 from pathlib import Path
 from unittest.mock import patch
 
@@ -57,6 +58,23 @@ class AccessiblePresentationContractTests(SimpleTestCase):
                     "scope_type": "국가",
                     "scope_text": "대만",
                     "written_date": "2026-08-30",
+                    "verified_empty": False,
+                    "facts": (
+                        {
+                            "source_position": 1,
+                            "alarm_level_code": "1",
+                            "scope_type": "국가",
+                            "scope_text": "첫 번째 제공 범위",
+                            "written_date": "2026-08-30",
+                        },
+                        {
+                            "source_position": 2,
+                            "alarm_level_code": "2",
+                            "scope_type": "일부",
+                            "scope_text": "두 번째 제공 범위",
+                            "written_date": None,
+                        },
+                    ),
                 }
             )
         return card
@@ -230,12 +248,46 @@ class AccessiblePresentationContractTests(SimpleTestCase):
         self.assertContains(response, "상태: 현재 게시본")
         self.assertContains(response, "상태: 재확인 필요")
         self.assertContains(response, 'rel="noopener noreferrer"')
+        self.assertContains(
+            response,
+            'ol class="warning-fact-list" aria-label="출처 제공 순서의 여행경보 항목"',
+        )
+        self.assertLess(
+            body.index("첫 번째 제공 범위"),
+            body.index("두 번째 제공 범위"),
+        )
         self.assertLess(body.index("검색 결과"), body.index('id="destination-1"'))
         self.assertLess(
             body.index('data-module="entry"'),
             body.index('data-module="warning"'),
         )
 
+    @patch("public_web.views.search_travel_opportunities")
+    def test_verified_empty_keeps_history_without_an_empty_fact_list(
+        self,
+        search,
+    ):
+        opportunity = deepcopy(self._opportunity())
+        warning = opportunity["warning_card"]
+        warning.update(
+            {
+                "verified_empty": True,
+                "facts": (),
+                "message": (
+                    "공식 출처의 국가 전체 조회 결과 0건이며 안전 여부를 뜻하지 않습니다."
+                ),
+            }
+        )
+        search.return_value = self._search_result(opportunities=[opportunity])
+
+        response = self.client.post(reverse("public_web:index"), self.valid_data())
+        body = response.content.decode("utf-8")
+
+        self.assertContains(response, "확인된 항목 수: 0건")
+        self.assertNotIn('class="warning-fact-list"', body)
+        self.assertContains(response, "출처와 확인 이력 보기")
+        self.assertContains(response, "여행경보 공식 출처 열기")
+
     @patch("public_web.views.search_travel_opportunities")
     def test_ready_search_without_matches_explains_how_to_try_again(self, search):
         search.return_value = self._search_result(opportunities=[])
@@ -311,3 +363,8 @@ class AccessiblePresentationContractTests(SimpleTestCase):
             ".status-symbol",
         ):
             self.assertNotIn(forbidden, self.css)
+        empty_rule = self.css.partition(".verified-empty-note {")[2].partition(
+            "}"
+        )[0]
+        self.assertIn("border-left: 0.25rem solid var(--line)", empty_rule)
+        self.assertNotIn("var(--stale)", empty_rule)
diff --git a/public_web/tests/test_travel_opportunity_web.py b/public_web/tests/test_travel_opportunity_web.py
index 0dd6193..d1e80c0 100644
--- a/public_web/tests/test_travel_opportunity_web.py
+++ b/public_web/tests/test_travel_opportunity_web.py
@@ -24,9 +24,21 @@ from public_web.results import (
     _load_warning_card,
     load_publication_cards_for_country,
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
     LEGACY_WARNING_SOURCE_CODE,
     MULTI_COUNTRY_WARNING_SOURCE_CODE,
+    WARNING_SOURCE_MODULE,
+    WARNING_SOURCE_OWNER,
 )
 
 
@@ -224,6 +236,114 @@ def _publication_base_row() -> dict[str, object]:
     }
 
 
+def _warning_snapshot_row(*, facts, source_item_count=None):
+    country_id = SUPPORTED_COUNTRY_IDS["TW"]
+    revision_id = UUID("381f627d-3ec7-449a-aae0-ffbda69b1f57")
+    source_id = UUID("5ca7e28c-81df-40ea-b77b-69b95860949f")
+    observation_id = UUID("42359986-f422-44cc-8699-2fa3898dd9c1")
+    rights_id = UUID("b4261766-5d04-46e4-a362-d8b6e7a244db")
+    typed_fingerprint = "d" * 64
+    body_sha = "e" * 64
+    request = warning_snapshot_request_fingerprint("TW")
+    prefix = "current_publication__travel_warning_revision"
+    artifact_prefix = f"{prefix}__parse_run__artifact"
+    observation_prefix = f"{prefix}__observation_attempt"
+    row = _publication_base_row()
+    row.update(
+        {
+            "current_publication__source_code_snapshot": (
+                WARNING_SNAPSHOT_SOURCE_CODE
+            ),
+            "current_publication__source_revision": (
+                WARNING_SNAPSHOT_SOURCE_REVISION
+            ),
+            "current_publication__source_owner_snapshot": (
+                WARNING_SOURCE_OWNER
+            ),
+            "current_publication__source_locator_snapshot": (
+                WARNING_SOURCE_LOCATOR
+            ),
+            "current_publication__attribution_text_snapshot": (
+                "외교부|공공데이터포털"
+            ),
+            "current_publication__source_contract_fingerprint_sha256": (
+                WARNING_SNAPSHOT_PARSER_CONTRACT_FINGERPRINT_SHA256
+            ),
+            "current_publication__parser_name": "MOFA_TRAVEL_ALARM_JSON",
+            "current_publication__parser_version": "V2",
+            "current_publication__parser_contract_fingerprint_sha256": (
+                WARNING_SNAPSHOT_PARSER_CONTRACT_FINGERPRINT_SHA256
+            ),
+            "current_publication__schema_fingerprint_sha256": (
+                WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256
+            ),
+            "current_publication__typed_fingerprint_sha256": (
+                typed_fingerprint
+            ),
+            "current_publication__source_first_observed_at": NOW,
+            "current_publication__travel_warning_revision_id": revision_id,
+            f"{prefix}__country_id": country_id,
+            f"{prefix}__source_alarm_level_code": None,
+            f"{prefix}__source_scope_type": None,
+            f"{prefix}__source_scope_text": None,
+            f"{prefix}__source_written_date": None,
+            f"{prefix}__source_item_count": (
+                len(facts) if source_item_count is None else source_item_count
+            ),
+            f"{prefix}__observation_attempt_id": observation_id,
+            f"{prefix}__first_observed_at": NOW,
+            f"{prefix}__typed_fingerprint_sha256": typed_fingerprint,
+            f"{prefix}__parse_run__outcome": "VALIDATED",
+            f"{prefix}__parse_run__parser_name": "MOFA_TRAVEL_ALARM_JSON",
+            f"{prefix}__parse_run__parser_version": "V2",
+            f"{prefix}__parse_run__parser_contract_fingerprint_sha256": (
+                WARNING_SNAPSHOT_PARSER_CONTRACT_FINGERPRINT_SHA256
+            ),
+            f"{prefix}__parse_run__expected_schema_fingerprint_sha256": (
+                WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256
+            ),
+            f"{prefix}__parse_run__observed_schema_fingerprint_sha256": (
+                WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256
+            ),
+            f"{artifact_prefix}_id": UUID(
+                "57a68d56-4e7c-4a06-a8bd-f106e0ed4a61"
+            ),
+            f"{artifact_prefix}__body_sha256": body_sha,
+            f"{artifact_prefix}__source_id": source_id,
+            f"{artifact_prefix}__source__code": WARNING_SNAPSHOT_SOURCE_CODE,
+            f"{artifact_prefix}__source__revision": (
+                WARNING_SNAPSHOT_SOURCE_REVISION
+            ),
+            f"{artifact_prefix}__source__module": WARNING_SOURCE_MODULE,
+            f"{artifact_prefix}__source__owner": WARNING_SOURCE_OWNER,
+            f"{artifact_prefix}__source__official_locator": (
+                WARNING_SOURCE_LOCATOR
+            ),
+            f"{artifact_prefix}__source__state": "ACTIVE",
+            f"{artifact_prefix}__source__enabled": True,
+            f"{observation_prefix}__source_id": source_id,
+            f"{observation_prefix}__source_revision": (
+                WARNING_SNAPSHOT_SOURCE_REVISION
+            ),
+            f"{observation_prefix}__rights_decision_id": rights_id,
+            f"{observation_prefix}__request_fingerprint_revision": (
+                request.revision
+            ),
+            f"{observation_prefix}__normalized_request_sha256": (
+                request.normalized_request_sha256
+            ),
+            f"{observation_prefix}__result": "SUCCEEDED",
+            f"{observation_prefix}__http_status": 200,
+            f"{observation_prefix}__provider_code": "MOFA_SUCCESS_0",
+            f"{observation_prefix}__completed_at": NOW,
+            f"{observation_prefix}__response_sha256": body_sha,
+            "country__iso_alpha2": "TW",
+            "warning_facts": facts,
+        }
+    )
+    return row
+
+
 class CountryPublicationCardTests(SimpleTestCase):
     @patch("public_web.results._freshness", return_value=(CARD_READY, NOW))
     @patch("public_web.results._entry_row")
@@ -296,6 +416,90 @@ class CountryPublicationCardTests(SimpleTestCase):
         self.assertEqual(card["scope_type"], "일부")
         self.assertEqual(card["scope_text"], "합성 범위")
         self.assertEqual(card["written_date"], "2026-08-02")
+        self.assertEqual(card["facts"][0]["scope_text"], "합성 범위")
+        self.assertFalse(card["verified_empty"])
+
+    @patch("public_web.results._freshness", return_value=(CARD_READY, NOW))
+    @patch("public_web.results._warning_row")
+    def test_warning_snapshot_keeps_all_provider_ordered_facts(
+        self,
+        warning_row,
+        _freshness,
+    ):
+        warning_row.return_value = _warning_snapshot_row(
+            facts=(
+                {
+                    "source_position": 1,
+                    "source_alarm_level_code": "1",
+                    "source_scope_type": "일부",
+                    "source_scope_text": "첫 번째 범위",
+                    "source_written_date": date(2026, 8, 1),
+                },
+                {
+                    "source_position": 2,
+                    "source_alarm_level_code": "2",
+                    "source_scope_type": "전역",
+                    "source_scope_text": "두 번째 범위",
+                    "source_written_date": None,
+                },
+            )
+        )
+
+        card = _load_warning_card(SUPPORTED_COUNTRY_IDS["TW"])
+
+        self.assertEqual(
+            [fact["scope_text"] for fact in card["facts"]],
+            ["첫 번째 범위", "두 번째 범위"],
+        )
+        self.assertEqual(card["facts"][0]["written_date"], "2026-08-01")
+        self.assertIsNone(card["facts"][1]["written_date"])
+        self.assertFalse(card["verified_empty"])
+
+    @patch("public_web.results._freshness", return_value=(CARD_READY, NOW))
+    @patch("public_web.results._warning_row")
+    def test_verified_empty_is_a_published_card_without_safety_inference(
+        self,
+        warning_row,
+        _freshness,
+    ):
+        warning_row.return_value = _warning_snapshot_row(facts=())
+        _freshness.side_effect = ((CARD_READY, NOW), (CARD_STALE, NOW))
+
+        ready = _load_warning_card(SUPPORTED_COUNTRY_IDS["TW"])
+        stale = _load_warning_card(SUPPORTED_COUNTRY_IDS["TW"])
+
+        self.assertEqual((ready["state"], stale["state"]), (CARD_READY, CARD_STALE))
+        for card in (ready, stale):
+            self.assertTrue(card["has_publication"])
+            self.assertTrue(card["verified_empty"])
+            self.assertEqual(card["facts"], ())
+            self.assertIn("0건", card["message"])
+            self.assertIn(
+                "안전함이나 여행 가능 여부를 뜻하지",
+                card["message"],
+            )
+
+    @patch("public_web.results._freshness", return_value=(CARD_READY, NOW))
+    @patch("public_web.results._warning_row")
+    @patch("public_web.results._load_entry_card")
+    def test_invalid_snapshot_count_isolated_to_warning_server_error(
+        self,
+        entry_card,
+        warning_row,
+        _freshness,
+    ):
+        entry_card.return_value = {"module": "entry", "state": CARD_READY}
+        warning_row.return_value = _warning_snapshot_row(
+            facts=(),
+            source_item_count=1,
+        )
+
+        cards = load_publication_cards_for_country(
+            SUPPORTED_COUNTRY_IDS["TW"]
+        )
+
+        self.assertEqual(cards["entry"]["state"], CARD_READY)
+        self.assertEqual(cards["warning"]["state"], CARD_SERVER_ERROR)
 
     @patch("public_web.results._load_warning_card")
     @patch("public_web.results._load_entry_card", side_effect=RuntimeError)
@@ -318,6 +522,11 @@ class CountryPublicationCardTests(SimpleTestCase):
             country_id=JP_COUNTRY_ID,
             source_code=LEGACY_WARNING_SOURCE_CODE,
         )
+        snapshot = _freshness_contract_for_publication(
+            module="warning",
+            country_id=SUPPORTED_COUNTRY_IDS["TW"],
+            source_code=WARNING_SNAPSHOT_SOURCE_CODE,
+        )
 
         self.assertIsNotNone(entry)
         self.assertIsNotNone(warning)
@@ -325,6 +534,7 @@ class CountryPublicationCardTests(SimpleTestCase):
         self.assertEqual(entry.decision_basis, "USER_DIRECTIVE_20260830")
         self.assertEqual(warning.source_code, LEGACY_WARNING_SOURCE_CODE)
         self.assertEqual(warning.decision_basis, "USER_FOLLOWUP_20260830")
+        self.assertEqual(snapshot.source_code, WARNING_SNAPSHOT_SOURCE_CODE)
         self.assertIsNone(
             _freshness_contract_for_publication(
                 module="entry",
