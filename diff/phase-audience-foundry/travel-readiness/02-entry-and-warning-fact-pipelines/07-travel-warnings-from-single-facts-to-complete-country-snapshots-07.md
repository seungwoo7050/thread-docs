## `feat(warnings): ingest official country snapshots`

diff --git a/operations/management/commands/ingest_travel_warning.py b/operations/management/commands/ingest_travel_warning.py
index 6e73fd2..47048b7 100644
--- a/operations/management/commands/ingest_travel_warning.py
+++ b/operations/management/commands/ingest_travel_warning.py
@@ -5,7 +5,7 @@ from travel_warnings.ingestion import (
     SUCCESS_CODES,
     TravelWarningIngestionCode,
     TravelWarningIngestionOutcome,
-    ingest_travel_warning,
+    ingest_travel_warning_snapshot as ingest_travel_warning,
 )
 
 
@@ -36,7 +36,10 @@ def _invoke_ingestion_without_exception_context(country_iso2):
 
 
 class Command(BaseCommand):
-    help = "Ingest one supported-country warning from the approved MOFA API."
+    help = (
+        "Ingest one complete supported-country warning snapshot from the "
+        "approved MOFA API."
+    )
 
     def add_arguments(self, parser):
         parser.add_argument(
diff --git a/travel_warnings/ingestion.py b/travel_warnings/ingestion.py
index 3e74c67..bb0f62d 100644
--- a/travel_warnings/ingestion.py
+++ b/travel_warnings/ingestion.py
@@ -43,30 +43,49 @@ from sources.transport import (
     PROVIDER_GATEWAY_20,
     PROVIDER_OTHER_ERROR,
     PROVIDER_SUCCESS_0,
+    WARNING_SNAPSHOT_MAX_RESPONSE_BYTES,
     WARNING_MAX_RESPONSE_BYTES,
     WARNING_REQUEST_FINGERPRINT,
     WARNING_SECRET_REFERENCE,
     WARNING_SOURCE_LOCATOR,
+    RequestFingerprint,
     SingleAttemptResult,
     fetch_travel_alarm_jp,
+    fetch_travel_alarm_snapshot,
     warning_request_fingerprint,
+    warning_snapshot_request_fingerprint,
 )
 from travel_windows.contracts import load_data_go_service_key
 
+from .contracts import (
+    WARNING_SNAPSHOT_DECISION_BASIS,
+    WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256,
+    WARNING_SNAPSHOT_FIELD_SCOPE,
+    WARNING_SNAPSHOT_MAX_FACTS,
+    WARNING_SNAPSHOT_PARSER_CONTRACT_FINGERPRINT_SHA256,
+    WARNING_SNAPSHOT_SOURCE_CODE,
+    WARNING_SNAPSHOT_SOURCE_REVISION,
+)
 from .models import (
     SOURCE_ALARM_LEVEL_CODE_MAX_LENGTH,
     SOURCE_SCOPE_TEXT_MAX_LENGTH,
     SOURCE_SCOPE_TYPE_MAX_LENGTH,
+    TravelWarningFact,
     TravelWarningRevision,
 )
 from .parser import (
     EXPECTED_SCHEMA_FINGERPRINT_SHA256,
     PARSER_CONTRACT_FINGERPRINT_SHA256,
     ParsedTravelWarning,
+    ParsedTravelWarningSnapshot,
     TravelWarningParseFailure,
     TravelWarningParseResult,
     TravelWarningParseSuccess,
+    TravelWarningSnapshotParseResult,
+    TravelWarningSnapshotParseSuccess,
     parse_travel_alarm_jp,
+    parse_travel_alarm_snapshot,
+    warning_snapshot_typed_fingerprint,
 )
 
 
@@ -99,6 +118,10 @@ _SUPPORTED_COUNTRY_IDENTITIES = {
     for _country_id, iso_alpha2, name_ko, name_en, _is_public
     in SUPPORTED_COUNTRY_ROWS
 }
+_SUPPORTED_SOURCE_COUNTRY_IDENTITIES = {
+    **_SUPPORTED_COUNTRY_IDENTITIES,
+    "HK": ("홍콩", "Hongkong"),
+}
 
 
 class TravelWarningIngestionCode:
@@ -132,6 +155,47 @@ class TravelWarningIngestionOutcome:
         return self.code in SUCCESS_CODES
 
 
+@dataclass(frozen=True, slots=True)
+class _WarningIngestionContract:
+    source_code: str
+    source_revision: str
+    field_scope: str
+    decision_basis: str
+    parser_version: str
+    parser_contract_fingerprint_sha256: str
+    expected_schema_fingerprint_sha256: str
+    max_response_bytes: int
+    request_fingerprint: Callable[[str], RequestFingerprint]
+
+
+_V1_WARNING_CONTRACT = _WarningIngestionContract(
+    source_code=WARNING_SOURCE_CODE,
+    source_revision=WARNING_SOURCE_REVISION,
+    field_scope=WARNING_FIELD_SCOPE,
+    decision_basis=WARNING_DECISION_BASIS,
+    parser_version=WARNING_PARSER_VERSION,
+    parser_contract_fingerprint_sha256=PARSER_CONTRACT_FINGERPRINT_SHA256,
+    expected_schema_fingerprint_sha256=EXPECTED_SCHEMA_FINGERPRINT_SHA256,
+    max_response_bytes=WARNING_MAX_RESPONSE_BYTES,
+    request_fingerprint=warning_request_fingerprint,
+)
+_V2_WARNING_CONTRACT = _WarningIngestionContract(
+    source_code=WARNING_SNAPSHOT_SOURCE_CODE,
+    source_revision=WARNING_SNAPSHOT_SOURCE_REVISION,
+    field_scope=WARNING_SNAPSHOT_FIELD_SCOPE,
+    decision_basis=WARNING_SNAPSHOT_DECISION_BASIS,
+    parser_version=ParseRun.ParserVersion.V2,
+    parser_contract_fingerprint_sha256=(
+        WARNING_SNAPSHOT_PARSER_CONTRACT_FINGERPRINT_SHA256
+    ),
+    expected_schema_fingerprint_sha256=(
+        WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256
+    ),
+    max_response_bytes=WARNING_SNAPSHOT_MAX_RESPONSE_BYTES,
+    request_fingerprint=warning_snapshot_request_fingerprint,
+)
+
+
 @dataclass(frozen=True, slots=True)
 class _FetchConfiguration:
     source_id: uuid.UUID
@@ -166,6 +230,14 @@ class _VerifiedParseResult:
     warning: ParsedTravelWarning | None
 
 
+@dataclass(frozen=True, slots=True)
+class _VerifiedSnapshotParseResult:
+    outcome: str
+    failure_code: str
+    observed_schema_fingerprint_sha256: str
+    snapshot: ParsedTravelWarningSnapshot | None
+
+
 @dataclass(frozen=True, slots=True)
 class _SanitizedInterruption:
     kind: str
@@ -264,12 +336,14 @@ def _discard_warning_ingestion_connection() -> None:
         pass
 
 
-def _close_abandoned_started_attempts() -> None:
+def _close_abandoned_started_attempts(
+    contract: _WarningIngestionContract = _V1_WARNING_CONTRACT,
+) -> None:
     with transaction.atomic(durable=True):
         abandoned = list(
             FetchAttempt.objects.select_for_update()
             .filter(
-                source__code=WARNING_SOURCE_CODE,
+                source__code=contract.source_code,
                 result=FetchAttempt.Result.STARTED,
             )
             .only("id", "started_at")
@@ -306,10 +380,13 @@ def _latest_rights(source: SourceConfiguration) -> SourceRightsDecision | None:
     )
 
 
-def _rights_are_exact(rights: SourceRightsDecision | None) -> bool:
+def _rights_are_exact(
+    rights: SourceRightsDecision | None,
+    contract: _WarningIngestionContract = _V1_WARNING_CONTRACT,
+) -> bool:
     return bool(
         rights is not None
-        and rights.source_revision == WARNING_SOURCE_REVISION
+        and rights.source_revision == contract.source_revision
         and rights.decision_sequence == 1
         and rights.decision == SourceRightsDecision.Decision.APPROVED
         and rights.access_mode
@@ -324,19 +401,22 @@ def _rights_are_exact(rights: SourceRightsDecision | None) -> bool:
         == SourceRightsDecision.Retention.PRODUCT_HISTORY
         and rights.evidence_retention
         == SourceRightsDecision.Retention.PRODUCT_HISTORY
-        and rights.field_scope_code == WARNING_FIELD_SCOPE
+        and rights.field_scope_code == contract.field_scope
         and rights.attribution_text == WARNING_ATTRIBUTION
         and rights.contract_fingerprint_sha256
-        == PARSER_CONTRACT_FINGERPRINT_SHA256
+        == contract.parser_contract_fingerprint_sha256
         and rights.decided_by == WARNING_DECIDED_BY
-        and rights.decision_basis_code == WARNING_DECISION_BASIS
+        and rights.decision_basis_code == contract.decision_basis
     )
 
 
-def _source_is_exact(source: SourceConfiguration) -> bool:
+def _source_is_exact(
+    source: SourceConfiguration,
+    contract: _WarningIngestionContract = _V1_WARNING_CONTRACT,
+) -> bool:
     return bool(
-        source.code == WARNING_SOURCE_CODE
-        and source.revision == WARNING_SOURCE_REVISION
+        source.code == contract.source_code
+        and source.revision == contract.source_revision
         and source.module == WARNING_SOURCE_MODULE
         and source.owner == WARNING_SOURCE_OWNER
         and source.official_locator == WARNING_SOURCE_LOCATOR
@@ -349,18 +429,20 @@ def _source_is_exact(source: SourceConfiguration) -> bool:
     )
 
 
-def _locked_approved_source() -> tuple[SourceConfiguration, SourceRightsDecision]:
+def _locked_approved_source(
+    contract: _WarningIngestionContract = _V1_WARNING_CONTRACT,
+) -> tuple[SourceConfiguration, SourceRightsDecision]:
     source = (
         SourceConfiguration.objects.select_for_update()
-        .filter(code=WARNING_SOURCE_CODE)
+        .filter(code=contract.source_code)
         .first()
     )
-    if source is None or not _source_is_exact(source):
+    if source is None or not _source_is_exact(source, contract):
         raise _ClosedPersistenceFailure(
             TravelWarningIngestionCode.SOURCE_GATE_FAILED
         )
     rights = _latest_rights(source)
-    if not _rights_are_exact(rights):
+    if not _rights_are_exact(rights, contract):
         raise _ClosedPersistenceFailure(
             TravelWarningIngestionCode.SOURCE_GATE_FAILED
         )
@@ -371,10 +453,11 @@ def _create_started_attempt(
     operation_id: uuid.UUID,
     attempt_number: int,
     country_iso2: str,
+    contract: _WarningIngestionContract = _V1_WARNING_CONTRACT,
 ) -> tuple[FetchAttempt, _FetchConfiguration]:
     with transaction.atomic(durable=True):
-        source, rights = _locked_approved_source()
-        request_fingerprint = warning_request_fingerprint(country_iso2)
+        source, rights = _locked_approved_source(contract)
+        request_fingerprint = contract.request_fingerprint(country_iso2)
         attempt = FetchAttempt.objects.create(
             source=source,
             source_revision=source.revision,
@@ -409,9 +492,12 @@ def _worker_interrupted_result() -> _VerifiedTransportResult:
     )
 
 
-def _missing_secret_result(country_iso2: str) -> SingleAttemptResult:
+def _missing_secret_result(
+    country_iso2: str,
+    contract: _WarningIngestionContract = _V1_WARNING_CONTRACT,
+) -> SingleAttemptResult:
     return SingleAttemptResult(
-        request_fingerprint=warning_request_fingerprint(country_iso2),
+        request_fingerprint=contract.request_fingerprint(country_iso2),
         attempt_result=ATTEMPT_TERMINAL_FAILED,
         failure_code=FAILURE_AUTHENTICATION,
     )
@@ -420,6 +506,7 @@ def _missing_secret_result(country_iso2: str) -> SingleAttemptResult:
 def _verify_transport_result(
     result: object,
     expected_fingerprint=WARNING_REQUEST_FINGERPRINT,
+    max_response_bytes: int = WARNING_MAX_RESPONSE_BYTES,
 ) -> _VerifiedTransportResult:
     if type(result) is not SingleAttemptResult:
         return _worker_interrupted_result()
@@ -461,7 +548,7 @@ def _verify_transport_result(
             or result.provider_code != PROVIDER_SUCCESS_0
             or result.failure_code != ""
             or result.byte_count != len(result.body)
-            or result.byte_count > WARNING_MAX_RESPONSE_BYTES
+            or result.byte_count > max_response_bytes
             or result.response_sha256
             != hashlib.sha256(result.body).hexdigest()
         ):
@@ -724,6 +811,152 @@ def _safe_parse(
     return _internal_parse_failure()
 
 
+def _internal_snapshot_parse_failure() -> _VerifiedSnapshotParseResult:
+    return _VerifiedSnapshotParseResult(
+        outcome=ParseRun.Outcome.FAILED,
+        failure_code=ParseRun.FailureCode.INTERNAL_PARSER_ERROR,
+        observed_schema_fingerprint_sha256="",
+        snapshot=None,
+    )
+
+
+def _safe_parse_snapshot(
+    body: bytes,
+    parser: Callable[..., TravelWarningSnapshotParseResult],
+    country_iso2: str,
+) -> _VerifiedSnapshotParseResult:
+    try:
+        result = parser(body, country_iso2=country_iso2)
+        if type(result) is TravelWarningSnapshotParseSuccess:
+            snapshot = result.snapshot
+            expected_name_ko, expected_name_en = (
+                _SUPPORTED_SOURCE_COUNTRY_IDENTITIES[country_iso2]
+            )
+            if (
+                type(snapshot) is not ParsedTravelWarningSnapshot
+                or result.observed_schema_fingerprint_sha256
+                != WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256
+                or type(snapshot.country_iso2) is not str
+                or snapshot.country_iso2 != country_iso2
+                or type(snapshot.country_name_ko) is not str
+                or snapshot.country_name_ko != expected_name_ko
+                or type(snapshot.country_name_en) is not str
+                or snapshot.country_name_en != expected_name_en
+                or type(snapshot.facts) is not tuple
+                or not 0 <= len(snapshot.facts) <= WARNING_SNAPSHOT_MAX_FACTS
+                or type(snapshot.typed_fingerprint_sha256) is not str
+            ):
+                return _internal_snapshot_parse_failure()
+
+            observed_fact_fingerprints: set[str] = set()
+            for warning in snapshot.facts:
+                if (
+                    type(warning) is not ParsedTravelWarning
+                    or type(warning.country_iso2) is not str
+                    or warning.country_iso2 != country_iso2
+                    or type(warning.country_name_ko) is not str
+                    or warning.country_name_ko != expected_name_ko
+                    or type(warning.country_name_en) is not str
+                    or warning.country_name_en != expected_name_en
+                    or type(warning.source_alarm_level_code) is not str
+                    or not warning.source_alarm_level_code.strip()
+                    or len(warning.source_alarm_level_code)
+                    > SOURCE_ALARM_LEVEL_CODE_MAX_LENGTH
+                    or type(warning.source_scope_type) is not str
+                    or not warning.source_scope_type.strip()
+                    or len(warning.source_scope_type)
+                    > SOURCE_SCOPE_TYPE_MAX_LENGTH
+                    or type(warning.source_scope_text) is not str
+                    or not warning.source_scope_text.strip()
+                    or len(warning.source_scope_text)
+                    > SOURCE_SCOPE_TEXT_MAX_LENGTH
+                    or (
+                        warning.source_written_date is not None
+                        and type(warning.source_written_date) is not date
+                    )
+                    or type(warning.typed_fingerprint_sha256) is not str
+                    or warning.typed_fingerprint_sha256
+                    != _canonical_typed_fingerprint(warning)
+                    or warning.typed_fingerprint_sha256
+                    in observed_fact_fingerprints
+                ):
+                    return _internal_snapshot_parse_failure()
+                observed_fact_fingerprints.add(
+                    warning.typed_fingerprint_sha256
+                )
+
+            if snapshot.typed_fingerprint_sha256 != (
+                warning_snapshot_typed_fingerprint(
+                    country_iso2=snapshot.country_iso2,
+                    country_name_ko=snapshot.country_name_ko,
+                    country_name_en=snapshot.country_name_en,
+                    facts=snapshot.facts,
+                )
+            ):
+                return _internal_snapshot_parse_failure()
+            return _VerifiedSnapshotParseResult(
+                outcome=ParseRun.Outcome.VALIDATED,
+                failure_code="",
+                observed_schema_fingerprint_sha256=(
+                    WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256
+                ),
+                snapshot=snapshot,
+            )
+        if type(result) is TravelWarningParseFailure:
+            observed_schema = result.observed_schema_fingerprint_sha256
+            failure_shapes = {
+                ParseRun.FailureCode.SCHEMA_MISMATCH: bool(
+                    _is_sha256(observed_schema)
+                    and observed_schema
+                    != WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256
+                ),
+                ParseRun.FailureCode.IDENTITY_MISMATCH: (
+                    observed_schema
+                    == WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256
+                ),
+                ParseRun.FailureCode.REQUIRED_VALUE_MISSING: (
+                    observed_schema
+                    == WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256
+                ),
+                ParseRun.FailureCode.DUPLICATE_RECORD: (
+                    observed_schema
+                    == WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256
+                ),
+                ParseRun.FailureCode.ENCODING_ERROR: observed_schema == "",
+                ParseRun.FailureCode.SYNTAX_ERROR: observed_schema == "",
+                ParseRun.FailureCode.INVALID_VALUE: (
+                    observed_schema
+                    == WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256
+                ),
+                ParseRun.FailureCode.CONFLICTING_VALUE: (
+                    observed_schema
+                    == WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256
+                ),
+                ParseRun.FailureCode.INTERNAL_PARSER_ERROR: (
+                    observed_schema == ""
+                ),
+            }
+            if (
+                type(result.failure_code) is str
+                and type(observed_schema) is str
+                and failure_shapes.get(result.failure_code, False)
+            ):
+                return _VerifiedSnapshotParseResult(
+                    outcome=(
+                        ParseRun.Outcome.FAILED
+                        if result.failure_code
+                        == ParseRun.FailureCode.INTERNAL_PARSER_ERROR
+                        else ParseRun.Outcome.QUARANTINED
+                    ),
+                    failure_code=result.failure_code,
+                    observed_schema_fingerprint_sha256=observed_schema,
+                    snapshot=None,
+                )
+    except Exception:
+        pass
+    return _internal_snapshot_parse_failure()
+
+
 def _replay_is_consistent(
     artifact: SourceArtifact,
     byte_count: int,
@@ -932,13 +1165,333 @@ def _persist_success(
         ) from None
 
 
+def _locked_snapshot_parse_run(
+    artifact: SourceArtifact,
+    byte_count: int,
+) -> ParseRun | None:
+    if (
+        artifact.byte_count != byte_count
+        or artifact.state != SourceArtifact.State.REVIEW_REQUIRED
+    ):
+        return None
+    runs = list(
+        ParseRun.objects.select_for_update().filter(
+            artifact=artifact,
+            parser_name=WARNING_PARSER_NAME,
+            parser_version=ParseRun.ParserVersion.V2,
+        )
+    )
+    if len(runs) != 1:
+        return None
+    run = runs[0]
+    if (
+        run.outcome != ParseRun.Outcome.VALIDATED
+        or run.parser_contract_fingerprint_sha256
+        != WARNING_SNAPSHOT_PARSER_CONTRACT_FINGERPRINT_SHA256
+        or run.expected_schema_fingerprint_sha256
+        != WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256
+        or run.observed_schema_fingerprint_sha256
+        != WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256
+        or run.completed_at is None
+    ):
+        return None
+    return run
+
+
+def _snapshot_revision_is_consistent(
+    revision: TravelWarningRevision,
+    snapshot: ParsedTravelWarningSnapshot,
+) -> bool:
+    if (
+        revision.country.iso_alpha2 != snapshot.country_iso2
+        or revision.observation_attempt_id is None
+        or revision.source_item_count != len(snapshot.facts)
+        or revision.source_alarm_level_code is not None
+        or revision.source_scope_type is not None
+        or revision.source_scope_text is not None
+        or revision.source_written_date is not None
+        or revision.typed_fingerprint_sha256
+        != snapshot.typed_fingerprint_sha256
+    ):
+        return False
+    expected_request = warning_snapshot_request_fingerprint(
+        snapshot.country_iso2
+    )
+    observation = revision.observation_attempt
+    if (
+        observation.request_fingerprint_revision
+        != expected_request.revision
+        or observation.normalized_request_sha256
+        != expected_request.normalized_request_sha256
+        or observation.result != FetchAttempt.Result.SUCCEEDED
+        or observation.http_status != 200
+        or observation.provider_code
+        != FetchAttempt.ProviderCode.MOFA_SUCCESS_0
+    ):
+        return False
+    facts = list(revision.facts.all())
+    if len(facts) != len(snapshot.facts):
+        return False
+    return all(
+        fact.source_position == position
+        and fact.source_alarm_level_code
+        == warning.source_alarm_level_code
+        and fact.source_scope_type == warning.source_scope_type
+        and fact.source_scope_text == warning.source_scope_text
+        and fact.source_written_date == warning.source_written_date
+        and fact.typed_fingerprint_sha256
+        == warning.typed_fingerprint_sha256
+        for position, (fact, warning) in enumerate(
+            zip(facts, snapshot.facts, strict=True),
+            start=1,
+        )
+    )
+
+
+def _persist_snapshot_success(
+    attempt: FetchAttempt,
+    verified: _VerifiedTransportResult,
+    parser: Callable[..., TravelWarningSnapshotParseResult],
+    country_iso2: str,
+) -> str:
+    try:
+        with transaction.atomic(durable=True):
+            source, rights = _locked_approved_source(_V2_WARNING_CONTRACT)
+            locked_attempt = FetchAttempt.objects.select_for_update().get(
+                pk=attempt.id
+            )
+            expected_request = warning_snapshot_request_fingerprint(
+                country_iso2
+            )
+            if (
+                locked_attempt.source_id != source.id
+                or locked_attempt.rights_decision_id != rights.id
+                or locked_attempt.source_revision != source.revision
+                or locked_attempt.request_fingerprint_revision
+                != expected_request.revision
+                or locked_attempt.normalized_request_sha256
+                != expected_request.normalized_request_sha256
+                or locked_attempt.result != FetchAttempt.Result.SUCCEEDED
+                or locked_attempt.http_status != 200
+                or locked_attempt.provider_code
+                != FetchAttempt.ProviderCode.MOFA_SUCCESS_0
+                or locked_attempt.completed_at is None
+                or locked_attempt.response_sha256
+                != verified.response_sha256
+                or verified.provider_code != PROVIDER_SUCCESS_0
+                or verified.byte_count != len(verified.body)
+                or not 1 <= verified.byte_count <= (
+                    WARNING_SNAPSHOT_MAX_RESPONSE_BYTES
+                )
+                or verified.response_sha256
+                != hashlib.sha256(verified.body).hexdigest()
+            ):
+                raise _ClosedPersistenceFailure(
+                    TravelWarningIngestionCode.EVIDENCE_CONFLICT
+                )
+
+            with connection.cursor() as cursor:
+                cursor.execute(
+                    "SELECT pg_advisory_xact_lock(hashtextextended(%s, 1))",
+                    [f"{source.id}:{verified.response_sha256}"],
+                )
+            artifact = (
+                SourceArtifact.objects.select_for_update()
+                .filter(
+                    source=source,
+                    body_sha256=verified.response_sha256,
+                )
+                .first()
+            )
+            parse_run: ParseRun | None = None
+            if artifact is not None:
+                parse_run = _locked_snapshot_parse_run(
+                    artifact,
+                    verified.byte_count,
+                )
+                if parse_run is None:
+                    raise _ClosedPersistenceFailure(
+                        TravelWarningIngestionCode.EVIDENCE_CONFLICT
+                    )
+
+            parse_result = _safe_parse_snapshot(
+                verified.body,
+                parser,
+                country_iso2,
+            )
+            if artifact is not None:
+                if (
+                    parse_result.outcome != ParseRun.Outcome.VALIDATED
+                    or parse_result.snapshot is None
+                ):
+                    raise _ClosedPersistenceFailure(
+                        TravelWarningIngestionCode.EVIDENCE_CONFLICT
+                    )
+            else:
+                anchor = (
+                    FetchAttempt.objects.select_for_update()
+                    .filter(
+                        source=source,
+                        result=FetchAttempt.Result.SUCCEEDED,
+                        response_sha256=verified.response_sha256,
+                    )
+                    .order_by("completed_at", "started_at", "id")
+                    .first()
+                )
+                if (
+                    anchor is None
+                    or anchor.source_revision != source.revision
+                    or anchor.rights_decision_id != rights.id
+                    or anchor.http_status != 200
+                    or anchor.provider_code
+                    != FetchAttempt.ProviderCode.MOFA_SUCCESS_0
+                    or anchor.response_sha256 != verified.response_sha256
+                    or anchor.completed_at is None
+                ):
+                    raise _ClosedPersistenceFailure(
+                        TravelWarningIngestionCode.EVIDENCE_CONFLICT
+                    )
+                artifact = SourceArtifact.objects.create(
+                    source=source,
+                    body_sha256=verified.response_sha256,
+                    byte_count=verified.byte_count,
+                    first_successful_attempt=anchor,
+                )
+                parse_run = ParseRun.objects.create(
+                    artifact=artifact,
+                    parser_name=WARNING_PARSER_NAME,
+                    parser_version=ParseRun.ParserVersion.V2,
+                    parser_contract_fingerprint_sha256=(
+                        WARNING_SNAPSHOT_PARSER_CONTRACT_FINGERPRINT_SHA256
+                    ),
+                    expected_schema_fingerprint_sha256=(
+                        WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256
+                    ),
+                    started_at=max(
+                        timezone.now(),
+                        anchor.completed_at,
+                        locked_attempt.completed_at,
+                    ),
+                )
+                completed_at = max(timezone.now(), parse_run.started_at)
+                updated = ParseRun.objects.filter(
+                    pk=parse_run.id,
+                    outcome=ParseRun.Outcome.STARTED,
+                ).update(
+                    outcome=parse_result.outcome,
+                    completed_at=completed_at,
+                    failure_code=parse_result.failure_code,
+                    observed_schema_fingerprint_sha256=(
+                        parse_result.observed_schema_fingerprint_sha256
+                    ),
+                )
+                if updated != 1:
+                    raise _ClosedPersistenceFailure(
+                        TravelWarningIngestionCode.PERSISTENCE_FAILED
+                    )
+                parse_run.refresh_from_db()
+                if parse_result.outcome != ParseRun.Outcome.VALIDATED:
+                    updated = SourceArtifact.objects.filter(
+                        pk=artifact.id,
+                        state=SourceArtifact.State.RECEIVED,
+                    ).update(state=SourceArtifact.State.REJECTED)
+                    if updated != 1:
+                        raise _ClosedPersistenceFailure(
+                            TravelWarningIngestionCode.PERSISTENCE_FAILED
+                        )
+                    if parse_result.outcome == ParseRun.Outcome.QUARANTINED:
+                        return TravelWarningIngestionCode.PARSE_QUARANTINED
+                    return TravelWarningIngestionCode.PARSE_FAILED
+                updated = SourceArtifact.objects.filter(
+                    pk=artifact.id,
+                    state=SourceArtifact.State.RECEIVED,
+                ).update(state=SourceArtifact.State.REVIEW_REQUIRED)
+                if updated != 1:
+                    raise _ClosedPersistenceFailure(
+                        TravelWarningIngestionCode.PERSISTENCE_FAILED
+                    )
+
+            snapshot = parse_result.snapshot
+            if snapshot is None or parse_run is None or artifact is None:
+                raise _ClosedPersistenceFailure(
+                    TravelWarningIngestionCode.PERSISTENCE_FAILED
+                )
+            current_source, current_rights = _locked_approved_source(
+                _V2_WARNING_CONTRACT
+            )
+            if current_source.id != source.id or current_rights.id != rights.id:
+                raise _ClosedPersistenceFailure(
+                    TravelWarningIngestionCode.SOURCE_GATE_FAILED
+                )
+            country = Country.objects.get(
+                pk=SUPPORTED_COUNTRY_IDS[country_iso2],
+                iso_alpha2=country_iso2,
+            )
+            existing_revision = (
+                TravelWarningRevision.objects.select_for_update(of=("self",))
+                .select_related("country", "observation_attempt")
+                .prefetch_related("facts")
+                .filter(parse_run=parse_run, country=country)
+                .first()
+            )
+            if existing_revision is not None:
+                if not _snapshot_revision_is_consistent(
+                    existing_revision,
+                    snapshot,
+                ):
+                    raise _ClosedPersistenceFailure(
+                        TravelWarningIngestionCode.EVIDENCE_CONFLICT
+                    )
+                return TravelWarningIngestionCode.REPLAY_OBSERVED
+
+            revision = TravelWarningRevision.objects.create(
+                country=country,
+                parse_run=parse_run,
+                observation_attempt=locked_attempt,
+                source_item_count=len(snapshot.facts),
+                first_observed_at=locked_attempt.completed_at,
+                typed_fingerprint_sha256=(
+                    snapshot.typed_fingerprint_sha256
+                ),
+            )
+            for source_position, warning in enumerate(
+                snapshot.facts,
+                start=1,
+            ):
+                TravelWarningFact.objects.create(
+                    revision=revision,
+                    source_position=source_position,
+                    source_alarm_level_code=(
+                        warning.source_alarm_level_code
+                    ),
+                    source_scope_type=warning.source_scope_type,
+                    source_scope_text=warning.source_scope_text,
+                    source_written_date=warning.source_written_date,
+                    typed_fingerprint_sha256=(
+                        warning.typed_fingerprint_sha256
+                    ),
+                )
+            return TravelWarningIngestionCode.REVIEW_REQUIRED
+    except _ClosedPersistenceFailure:
+        raise
+    except Exception:
+        raise _ClosedPersistenceFailure(
+            TravelWarningIngestionCode.PERSISTENCE_FAILED
+        ) from None
+
+
 def _run_travel_warning_ingestion(
     *,
     country_iso2: str,
     transport: Callable[..., SingleAttemptResult],
-    parser: Callable[[bytes], TravelWarningParseResult],
+    parser: Callable[..., object],
     retry_wait: Callable[[int], None],
     secret_loader: SecretLoader,
+    contract: _WarningIngestionContract = _V1_WARNING_CONTRACT,
+    persist_success: Callable[
+        [FetchAttempt, _VerifiedTransportResult, Callable[..., object], str],
+        str,
+    ] = _persist_success,
 ) -> TravelWarningIngestionOutcome | _SanitizedInterruption:
     """Run one operation and return only a closed outcome or kind sentinel."""
 
@@ -962,7 +1515,7 @@ def _run_travel_warning_ingestion(
         )
     try:
         try:
-            _close_abandoned_started_attempts()
+            _close_abandoned_started_attempts(contract)
         except Exception:
             return TravelWarningIngestionOutcome(
                 TravelWarningIngestionCode.PERSISTENCE_FAILED,
@@ -979,6 +1532,7 @@ def _run_travel_warning_ingestion(
                     operation_id,
                     attempt_count,
                     country_iso2,
+                    contract,
                 )
             except _ClosedPersistenceFailure as failure:
                 return TravelWarningIngestionOutcome(
@@ -1004,7 +1558,10 @@ def _run_travel_warning_ingestion(
                     secret_loader,
                 )
                 if loaded_secret is None:
-                    raw_result: object = _missing_secret_result(country_iso2)
+                    raw_result: object = _missing_secret_result(
+                        country_iso2,
+                        contract,
+                    )
                 else:
                     raw_result = transport(
                         country_iso2=country_iso2,
@@ -1032,7 +1589,8 @@ def _run_travel_warning_ingestion(
             try:
                 verified = _verify_transport_result(
                     raw_result,
-                    warning_request_fingerprint(country_iso2),
+                    contract.request_fingerprint(country_iso2),
+                    contract.max_response_bytes,
                 )
             except Exception:
                 verified = _worker_interrupted_result()
@@ -1044,7 +1602,7 @@ def _run_travel_warning_ingestion(
                 closed_attempt = _close_attempt(attempt.id, verified)
             except Exception:
                 try:
-                    _close_abandoned_started_attempts()
+                    _close_abandoned_started_attempts(contract)
                 except BaseException:
                     pass
                 loaded_secret = None
@@ -1059,7 +1617,7 @@ def _run_travel_warning_ingestion(
                 exception_kind = _base_exception_kind(exception)
                 del exception
                 try:
-                    _close_abandoned_started_attempts()
+                    _close_abandoned_started_attempts(contract)
                 except BaseException:
                     pass
                 loaded_secret = None
@@ -1076,7 +1634,7 @@ def _run_travel_warning_ingestion(
 
             if verified.attempt_result == ATTEMPT_SUCCEEDED:
                 try:
-                    code = _persist_success(
+                    code = persist_success(
                         closed_attempt,
                         verified,
                         parser,
@@ -1107,7 +1665,7 @@ def _run_travel_warning_ingestion(
         exception_kind = _base_exception_kind(exception)
         del exception
         try:
-            _close_abandoned_started_attempts()
+            _close_abandoned_started_attempts(contract)
         except BaseException:
             pass
         loaded_secret = None
@@ -1155,3 +1713,46 @@ def ingest_travel_warning(
         secret_loader = None
         _raise_sanitized_base_exception(interruption_kind)
     return result
+
+
+def ingest_travel_warning_snapshot(
+    *,
+    country_iso2: str = "JP",
+    transport: Callable[..., SingleAttemptResult] = (
+        fetch_travel_alarm_snapshot
+    ),
+    parser: Callable[..., TravelWarningSnapshotParseResult] = (
+        parse_travel_alarm_snapshot
+    ),
+    retry_wait: Callable[[int], None] = _bounded_retry_wait,
+    secret_loader: SecretLoader = _environment_secret_loader,
+) -> TravelWarningIngestionOutcome:
+    """Ingest one complete 0..100-row country warning snapshot."""
+
+    if (
+        type(country_iso2) is not str
+        or country_iso2 not in SUPPORTED_COUNTRY_IDS
+    ):
+        return TravelWarningIngestionOutcome(
+            TravelWarningIngestionCode.SOURCE_GATE_FAILED,
+            0,
+        )
+
+    result = _run_travel_warning_ingestion(
+        country_iso2=country_iso2,
+        transport=transport,
+        parser=parser,
+        retry_wait=retry_wait,
+        secret_loader=secret_loader,
+        contract=_V2_WARNING_CONTRACT,
+        persist_success=_persist_snapshot_success,
+    )
+    if isinstance(result, _SanitizedInterruption):
+        interruption_kind = result.kind
+        result = None
+        transport = None
+        parser = None
+        retry_wait = None
+        secret_loader = None
+        _raise_sanitized_base_exception(interruption_kind)
+    return result
diff --git a/travel_warnings/tests/test_snapshot_ingestion.py b/travel_warnings/tests/test_snapshot_ingestion.py
new file mode 100644
index 0000000..5582e8e
--- /dev/null
+++ b/travel_warnings/tests/test_snapshot_ingestion.py
@@ -0,0 +1,248 @@
+import hashlib
+import json
+from io import StringIO
+from unittest.mock import patch
+
+from django.core.management import call_command
+from django.test import SimpleTestCase, TransactionTestCase
+
+from countries.models import SUPPORTED_COUNTRY_ROWS, Country
+from sources.models import FetchAttempt, ParseRun, SourceArtifact
+from sources.transport import (
+    ATTEMPT_SUCCEEDED,
+    PROVIDER_SUCCESS_0,
+    SingleAttemptResult,
+    warning_snapshot_request_fingerprint,
+)
+from travel_warnings.contracts import WARNING_SNAPSHOT_SOURCE_CODE
+from travel_warnings.ingestion import (
+    TravelWarningIngestionCode,
+    TravelWarningIngestionOutcome,
+    ingest_travel_warning_snapshot,
+)
+from travel_warnings.models import TravelWarningFact, TravelWarningRevision
+
+
+SOURCE_IDENTITIES = {
+    "JP": ("일본", "Japan"),
+    "TW": ("대만", "Taiwan"),
+    "HK": ("홍콩", "Hongkong"),
+    "MO": ("마카오", "Macao"),
+    "VN": ("베트남", "Vietnam"),
+    "TH": ("태국", "Thailand"),
+}
+RAW_MARKER = "RAW_SNAPSHOT_MUST_NOT_PERSIST"
+
+
+def snapshot_item(country_iso2, position):
+    name_ko, name_en = SOURCE_IDENTITIES[country_iso2]
+    return {
+        "alarm_lvl": str(position),
+        "continent_cd": "1000",
+        "continent_eng_nm": "Asia",
+        "continent_nm": "아시아",
+        "country_eng_nm": name_en,
+        "country_iso_alp2": country_iso2,
+        "country_nm": name_ko,
+        "dang_map_download_url": "",
+        "flag_download_url": "",
+        "map_download_url": None,
+        "org_country_idx": str(position),
+        "region_ty": f"범위 {position}",
+        "remark": f"제공 순서 {position}",
+        "written_dt": None,
+    }
+
+
+def snapshot_payload(country_iso2, item_count):
+    return json.dumps(
+        {
+            "response": {
+                "header": {
+                    "resultCode": "0",
+                    "resultMsg": RAW_MARKER,
+                },
+                "body": {
+                    "dataType": "JSON",
+                    "items": {
+                        "item": [
+                            snapshot_item(country_iso2, position)
+                            for position in range(1, item_count + 1)
+                        ]
+                    },
+                    "numOfRows": 100,
+                    "pageNo": 1,
+                    "totalCount": item_count,
+                },
+            }
+        },
+        ensure_ascii=False,
+        separators=(",", ":"),
+    ).encode("utf-8")
+
+
+def successful_snapshot_result(payload, country_iso2):
+    return SingleAttemptResult(
+        request_fingerprint=warning_snapshot_request_fingerprint(country_iso2),
+        attempt_result=ATTEMPT_SUCCEEDED,
+        http_status=200,
+        provider_code=PROVIDER_SUCCESS_0,
+        response_sha256=hashlib.sha256(payload).hexdigest(),
+        byte_count=len(payload),
+        body=payload,
+    )
+
+
+class SnapshotIngestionTests(TransactionTestCase):
+    def setUp(self):
+        for country_id, iso2, name_ko, name_en, is_public in (
+            SUPPORTED_COUNTRY_ROWS
+        ):
+            Country.objects.get_or_create(
+                id=country_id,
+                defaults={
+                    "iso_alpha2": iso2,
+                    "name_ko": name_ko,
+                    "name_en": name_en,
+                    "is_public": is_public,
+                },
+            )
+        call_command("register_approved_sources", "--apply", stdout=StringIO())
+
+    def ingest(self, country_iso2, payload, *, response_country=None):
+        result = successful_snapshot_result(
+            payload,
+            response_country or country_iso2,
+        )
+
+        def transport(**kwargs):
+            self.assertEqual(kwargs["country_iso2"], country_iso2)
+            return result
+
+        return ingest_travel_warning_snapshot(
+            country_iso2=country_iso2,
+            transport=transport,
+            retry_wait=lambda _attempt: None,
+            secret_loader=lambda _reference: "synthetic-credential",
+        )
+
+    def test_empty_countries_share_artifact_with_distinct_observations(self):
+        payload = snapshot_payload("TW", 0)
+        taiwan = self.ingest("TW", payload)
+        macau = self.ingest("MO", payload)
+
+        self.assertEqual(taiwan.code, TravelWarningIngestionCode.REVIEW_REQUIRED)
+        self.assertEqual(macau.code, TravelWarningIngestionCode.REVIEW_REQUIRED)
+        source = SourceArtifact.objects.get().source
+        self.assertEqual(source.code, WARNING_SNAPSHOT_SOURCE_CODE)
+        self.assertEqual(SourceArtifact.objects.count(), 1)
+        self.assertEqual(ParseRun.objects.filter(parser_version="V2").count(), 1)
+        roots = list(
+            TravelWarningRevision.objects.select_related(
+                "country", "observation_attempt"
+            ).order_by("country__iso_alpha2")
+        )
+        self.assertEqual([root.country.iso_alpha2 for root in roots], ["MO", "TW"])
+        self.assertEqual([root.source_item_count for root in roots], [0, 0])
+        self.assertEqual(TravelWarningFact.objects.count(), 0)
+        self.assertNotEqual(
+            roots[0].observation_attempt_id,
+            roots[1].observation_attempt_id,
+        )
+        for root in roots:
+            expected = warning_snapshot_request_fingerprint(
+                root.country.iso_alpha2
+            )
+            self.assertEqual(
+                root.observation_attempt.request_fingerprint_revision,
+                expected.revision,
+            )
+            self.assertEqual(
+                root.observation_attempt.normalized_request_sha256,
+                expected.normalized_request_sha256,
+            )
+        for rows in (
+            FetchAttempt.objects.values(),
+            SourceArtifact.objects.values(),
+            ParseRun.objects.values(),
+            TravelWarningRevision.objects.values(),
+            TravelWarningFact.objects.values(),
+        ):
+            self.assertNotIn(RAW_MARKER, repr(list(rows)))
+        self.assertNotIn(RAW_MARKER, repr(taiwan))
+
+    def test_one_and_three_facts_are_ordered_and_replay_is_idempotent(self):
+        japan_payload = snapshot_payload("JP", 1)
+        first = self.ingest("JP", japan_payload)
+        replay = self.ingest("JP", japan_payload)
+        thailand = self.ingest("TH", snapshot_payload("TH", 3))
+
+        self.assertEqual(first.code, TravelWarningIngestionCode.REVIEW_REQUIRED)
+        self.assertEqual(replay.code, TravelWarningIngestionCode.REPLAY_OBSERVED)
+        self.assertEqual(thailand.code, TravelWarningIngestionCode.REVIEW_REQUIRED)
+        japan = TravelWarningRevision.objects.get(country__iso_alpha2="JP")
+        thai = TravelWarningRevision.objects.get(country__iso_alpha2="TH")
+        self.assertEqual(japan.source_item_count, 1)
+        self.assertEqual(thai.source_item_count, 3)
+        self.assertEqual(
+            list(thai.facts.values_list("source_position", flat=True)),
+            [1, 2, 3],
+        )
+        self.assertEqual(
+            list(thai.facts.values_list("source_scope_text", flat=True)),
+            ["제공 순서 1", "제공 순서 2", "제공 순서 3"],
+        )
+        self.assertEqual(SourceArtifact.objects.count(), 2)
+        self.assertEqual(ParseRun.objects.filter(parser_version="V2").count(), 2)
+        self.assertEqual(TravelWarningRevision.objects.count(), 2)
+        self.assertEqual(TravelWarningFact.objects.count(), 4)
+        self.assertEqual(FetchAttempt.objects.count(), 3)
+
+    def test_country_request_fingerprint_mismatch_is_closed(self):
+        outcome = self.ingest(
+            "TW",
+            snapshot_payload("TW", 0),
+            response_country="MO",
+        )
+
+        self.assertEqual(
+            outcome.code,
+            TravelWarningIngestionCode.FETCH_TERMINAL_FAILED,
+        )
+        attempt = FetchAttempt.objects.get()
+        self.assertEqual(attempt.result, FetchAttempt.Result.TERMINAL_FAILED)
+        self.assertEqual(
+            attempt.failure_code,
+            FetchAttempt.FailureCode.WORKER_INTERRUPTED,
+        )
+        self.assertFalse(SourceArtifact.objects.exists())
+        self.assertFalse(TravelWarningRevision.objects.exists())
+
+    def test_hongkong_official_identity_persists_one_fact(self):
+        outcome = self.ingest("HK", snapshot_payload("HK", 1))
+
+        self.assertEqual(outcome.code, TravelWarningIngestionCode.REVIEW_REQUIRED)
+        root = TravelWarningRevision.objects.get(country__iso_alpha2="HK")
+        self.assertEqual(root.source_item_count, 1)
+        self.assertEqual(root.facts.count(), 1)
+
+
+class SnapshotIngestionCommandTests(SimpleTestCase):
+    def test_command_uses_snapshot_boundary_and_redacted_output(self):
+        output = StringIO()
+        with patch(
+            "operations.management.commands.ingest_travel_warning."
+            "ingest_travel_warning",
+            return_value=TravelWarningIngestionOutcome(
+                TravelWarningIngestionCode.REVIEW_REQUIRED,
+                1,
+            ),
+        ) as ingestion:
+            call_command("ingest_travel_warning", "--country", "TW", stdout=output)
+
+        ingestion.assert_called_once_with(country_iso2="TW")
+        self.assertEqual(
+            output.getvalue().strip(),
+            "travel_warning_ingestion_result=REVIEW_REQUIRED "
+            "attempts=1 country=TW",
+        )


