## `feat(duration): seal official route duration derivations`

diff --git a/travel_windows/ingestion.py b/travel_windows/ingestion.py
index b74cfd6..a5f1320 100644
--- a/travel_windows/ingestion.py
+++ b/travel_windows/ingestion.py
@@ -9,6 +9,7 @@ parsing. Raw response bytes are never written to a model, log, or exception.
 from __future__ import annotations
 
 import hashlib
+import json
 import re
 import uuid
 from dataclasses import dataclass, field
@@ -33,11 +34,16 @@ from sources.transport import (
     ATTEMPT_TERMINAL_FAILED,
     ROLE_DESTINATION_CITY,
     ROLE_LEGACY_ARRIVAL,
+    ROLE_DURATION_ARCHIVE,
+    ROLE_DURATION_LIMIT,
+    ROLE_DURATION_METADATA,
+    ROLE_DURATION_PAGE,
     ROLE_SCHEDULE_ARRIVAL,
     ROLE_SCHEDULE_DEPARTURE,
     AviationReferenceFetchResult,
     AviationResponseLineage,
     RequestFingerprint,
+    RouteDurationReferenceFetchResult,
     SchedulePageFetchResult,
     SingleAttemptResult,
 )
@@ -50,10 +56,18 @@ from .contracts import (
     CITY_SOURCE_REVISION,
     DATA_GO_SECRET_REFERENCE,
     DURATION_CONTRACT_FINGERPRINT_SHA256,
+    DURATION_DERIVATION_FINGERPRINT_SHA256,
     DURATION_FIELD_SCOPE,
     DURATION_SCHEMA_FINGERPRINT_SHA256,
+    DURATION_REFERENCE_CONTRACT_FINGERPRINT_SHA256,
+    DURATION_REFERENCE_SCHEMA_FINGERPRINT_SHA256,
     DURATION_SOURCE_CODE,
+    DURATION_SOURCE_LOCATOR,
     DURATION_SOURCE_REVISION,
+    DURATION_V1_CONTRACT_FINGERPRINT_SHA256,
+    DURATION_V1_FIELD_SCOPE,
+    DURATION_V1_SCHEMA_FINGERPRINT_SHA256,
+    DURATION_V1_SOURCE_REVISION,
     LEGACY_SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
     LEGACY_SCHEDULE_FIELD_SCOPE,
     LEGACY_SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
@@ -65,7 +79,18 @@ from .contracts import (
     SCHEDULE_SOURCE_CODE,
     SCHEDULE_SOURCE_REVISION,
 )
-from .models import ReviewedDurationReceipt
+from .duration_reference import (
+    DurationReferenceFailure,
+    DurationReferenceSuccess,
+    derive_route_durations,
+)
+from .models import (
+    CURATED_AIRPORT_ROWS,
+    Airport,
+    ReviewedDurationReceipt,
+    RouteDurationDerivation,
+    RouteDurationDerivedRoute,
+)
 from .parser import (
     AviationParseFailure,
     CityReferenceParseSuccess,
@@ -97,6 +122,16 @@ class FlightIngestionOutcome:
         return self.code == FlightIngestionCode.STAGED
 
 
+@dataclass(frozen=True, slots=True)
+class DurationIngestionOutcome:
+    code: str
+    derivation_id: str | None = None
+
+    @property
+    def succeeded(self) -> bool:
+        return self.code == FlightIngestionCode.STAGED
+
+
 class _IngestionRejected(Exception):
     def __init__(self, code: str):
         self.code = code
@@ -132,13 +167,18 @@ _SOURCE_CONTRACTS = {
     ),
     DURATION_SOURCE_CODE: _SourceContract(
         DURATION_SOURCE_REVISION,
-        DURATION_CONTRACT_FINGERPRINT_SHA256,
+        DURATION_REFERENCE_CONTRACT_FINGERPRINT_SHA256,
         DURATION_FIELD_SCOPE,
         "",
     ),
 }
 _REMOTE_SOURCE_CODES = frozenset(
-    (SCHEDULE_SOURCE_CODE, CITY_SOURCE_CODE, LEGACY_SCHEDULE_SOURCE_CODE)
+    (
+        SCHEDULE_SOURCE_CODE,
+        CITY_SOURCE_CODE,
+        LEGACY_SCHEDULE_SOURCE_CODE,
+        DURATION_SOURCE_CODE,
+    )
 )
 _SHA256_HEX = re.compile(r"^[0-9a-f]{64}$")
 _REVIEWER = re.compile(r"^[A-Z][A-Z0-9_.:-]{2,99}$")
@@ -192,6 +232,55 @@ def _locked_source(
     return source, rights
 
 
+def _locked_v1_duration_source(
+) -> tuple[SourceConfiguration, SourceRightsDecision]:
+    """Validate the immutable V1 decision used by historical manual receipts."""
+
+    source = (
+        SourceConfiguration.objects.select_for_update()
+        .filter(
+            code=DURATION_SOURCE_CODE,
+            revision__in=(
+                DURATION_V1_SOURCE_REVISION,
+                DURATION_SOURCE_REVISION,
+            ),
+            module=SourceConfiguration.Module.AVIATION,
+            state=SourceConfiguration.State.ACTIVE,
+            enabled=True,
+            secret_reference_name="",
+            official_locator=DURATION_SOURCE_LOCATOR,
+        )
+        .first()
+    )
+    if source is None:
+        raise _IngestionRejected(FlightIngestionCode.SOURCE_GATE_FAILED)
+    rights = (
+        SourceRightsDecision.objects.select_for_update()
+        .filter(source=source, source_revision=DURATION_V1_SOURCE_REVISION)
+        .order_by("-decision_sequence", "-id")
+        .first()
+    )
+    if (
+        rights is None
+        or rights.decision != SourceRightsDecision.Decision.APPROVED
+        or not rights.access_allowed
+        or not rights.automated_collection_allowed
+        or not rights.typed_field_storage_allowed
+        or rights.raw_body_storage_allowed
+        or not rights.typed_republication_allowed
+        or rights.raw_retention_seconds != 0
+        or rights.typed_retention
+        != SourceRightsDecision.Retention.PRODUCT_HISTORY
+        or rights.evidence_retention
+        != SourceRightsDecision.Retention.PRODUCT_HISTORY
+        or rights.field_scope_code != DURATION_V1_FIELD_SCOPE
+        or rights.contract_fingerprint_sha256
+        != DURATION_V1_CONTRACT_FINGERPRINT_SHA256
+    ):
+        raise _IngestionRejected(FlightIngestionCode.SOURCE_GATE_FAILED)
+    return source, rights
+
+
 @dataclass(frozen=True, slots=True)
 class _DurableAttemptReceipt:
     result: SingleAttemptResult = field(repr=False)
@@ -556,6 +645,56 @@ def _reference_inputs(
     return city, legacy
 
 
+def _duration_archive_input(
+    result: RouteDurationReferenceFetchResult,
+    recorder: DurableAviationAttemptRecorder,
+) -> tuple[_CollectedInput, date]:
+    if type(result) is not RouteDurationReferenceFetchResult:
+        raise _IngestionRejected(FlightIngestionCode.PERSISTENCE_FAILED)
+    lineages = tuple(result.lineage)
+    expected_roles = (
+        ROLE_DURATION_PAGE,
+        ROLE_DURATION_METADATA,
+        ROLE_DURATION_LIMIT,
+        ROLE_DURATION_ARCHIVE,
+    )
+    if (
+        not result.succeeded
+        or result.failure_code
+        or result.unchanged
+        or not result.archive
+        or len(lineages) != len(expected_roles)
+        or any(
+            lineage.source_code != DURATION_SOURCE_CODE
+            or lineage.role != role
+            or lineage.role_sequence != 1
+            or not lineage.succeeded
+            for lineage, role in zip(lineages, expected_roles, strict=True)
+        )
+    ):
+        raise _IngestionRejected(FlightIngestionCode.SOURCE_GATE_FAILED)
+    try:
+        reference_date = date.fromisoformat(result.reference_date)
+        if reference_date.isoformat() != result.reference_date:
+            raise ValueError
+    except (TypeError, ValueError):
+        raise _IngestionRejected(
+            FlightIngestionCode.PERSISTENCE_FAILED
+        ) from None
+    for lineage in lineages[:-1]:
+        recorder.receipt_for(
+            source_code=DURATION_SOURCE_CODE,
+            result=lineage.attempt,
+        )
+    archive = _collected_input(
+        recorder=recorder,
+        lineage=lineages[-1],
+        source_code=DURATION_SOURCE_CODE,
+        body=result.archive,
+    )
+    return archive, reference_date
+
+
 def _parse_failure_shape(
     result: AviationParseFailure,
     *,
@@ -680,7 +819,7 @@ def _persist_parse_run(
                     FlightIngestionCode.PERSISTENCE_FAILED
                 )
 
-    if isinstance(parsed, AviationParseFailure):
+    if isinstance(parsed, (AviationParseFailure, DurationReferenceFailure)):
         outcome, failure_code, observed_schema = _parse_failure_shape(
             parsed,
             expected_schema=schema_fingerprint,
@@ -723,7 +862,9 @@ def _persist_parse_run(
             run.refresh_from_db()
         expected_outcome = (
             ParseRun.Outcome.VALIDATED
-            if not isinstance(parsed, AviationParseFailure)
+            if not isinstance(
+                parsed, (AviationParseFailure, DurationReferenceFailure)
+            )
             and getattr(parsed, "observed_schema_fingerprint_sha256", "")
             == schema_fingerprint
             else outcome
@@ -733,6 +874,256 @@ def _persist_parse_run(
         return run
 
 
+def _typed_fingerprint(value: object) -> str:
+    payload = json.dumps(
+        value,
+        ensure_ascii=False,
+        separators=(",", ":"),
+        sort_keys=True,
+    ).encode("utf-8")
+    return hashlib.sha256(payload).hexdigest()
+
+
+def _derived_route_payload(row: object) -> dict[str, object]:
+    destination_iata = getattr(row, "destination_iata", None)
+    if destination_iata is None:
+        destination_iata = row.destination_airport.iata_code
+    return {
+        "destination_iata": destination_iata,
+        "outbound_minutes": row.outbound_minutes,
+        "inbound_minutes": row.inbound_minutes,
+        "outbound": {
+            "source_rows": row.outbound_source_rows,
+            "invalid_rows": row.outbound_invalid_rows,
+            "cancelled_rows": row.outbound_cancelled_rows,
+            "occurrences": row.outbound_occurrences,
+            "outliers": row.outbound_outliers,
+            "retained": row.outbound_retained,
+        },
+        "inbound": {
+            "source_rows": row.inbound_source_rows,
+            "invalid_rows": row.inbound_invalid_rows,
+            "cancelled_rows": row.inbound_cancelled_rows,
+            "occurrences": row.inbound_occurrences,
+            "outliers": row.inbound_outliers,
+            "retained": row.inbound_retained,
+        },
+    }
+
+
+def _duration_derivation_payload(
+    parsed: DurationReferenceSuccess,
+    *,
+    reference_version_sha256: str,
+) -> dict[str, object]:
+    return {
+        "reference_date": parsed.reference_date.isoformat(),
+        "reference_locator": parsed.reference_locator,
+        "reference_version_sha256": reference_version_sha256,
+        "source_row_count": parsed.source_row_count,
+        "derivation_contract": parsed.derivation_fingerprint_sha256,
+        "routes": [
+            _derived_route_payload(row)
+            for row in sorted(
+                parsed.routes,
+                key=lambda value: value.destination_iata,
+            )
+        ],
+    }
+
+
+def _persist_duration_derivation(
+    *,
+    parse_run: ParseRun,
+    parsed: DurationReferenceSuccess,
+    reference_version_sha256: str,
+) -> RouteDurationDerivation:
+    expected_codes = {row[1] for row in CURATED_AIRPORT_ROWS}
+    parsed_codes = {row.destination_iata for row in parsed.routes}
+    if (
+        parsed_codes != expected_codes
+        or len(parsed.routes) != len(expected_codes)
+        or parsed.reference_locator != DURATION_SOURCE_LOCATOR
+        or parsed.derivation_fingerprint_sha256
+        != DURATION_DERIVATION_FINGERPRINT_SHA256
+        or _SHA256_HEX.fullmatch(reference_version_sha256 or "") is None
+    ):
+        raise _IngestionRejected(FlightIngestionCode.PARSE_QUARANTINED)
+
+    with transaction.atomic(durable=True):
+        locked_run = (
+            ParseRun.objects.select_for_update()
+            .select_related("artifact__source")
+            .get(pk=parse_run.pk)
+        )
+        if (
+            locked_run.outcome != ParseRun.Outcome.VALIDATED
+            or locked_run.parser_name
+            != ParseRun.ParserName.ROUTE_DURATION_REFERENCE_FILE
+            or locked_run.parser_version != ParseRun.ParserVersion.V2
+            or locked_run.parser_contract_fingerprint_sha256
+            != DURATION_REFERENCE_CONTRACT_FINGERPRINT_SHA256
+            or locked_run.expected_schema_fingerprint_sha256
+            != DURATION_REFERENCE_SCHEMA_FINGERPRINT_SHA256
+            or locked_run.observed_schema_fingerprint_sha256
+            != DURATION_REFERENCE_SCHEMA_FINGERPRINT_SHA256
+            or locked_run.artifact.source.code != DURATION_SOURCE_CODE
+            or locked_run.artifact.source.revision != DURATION_SOURCE_REVISION
+        ):
+            raise _IngestionRejected(FlightIngestionCode.SOURCE_GATE_FAILED)
+        derivation = RouteDurationDerivation.objects.filter(
+            parse_run=locked_run
+        ).first()
+        expected_payload = _duration_derivation_payload(
+            parsed,
+            reference_version_sha256=reference_version_sha256,
+        )
+        expected_fingerprint = _typed_fingerprint(expected_payload)
+        if derivation is not None:
+            routes = list(
+                locked_run.derived_duration_routes.select_related(
+                    "destination_airport"
+                ).order_by("destination_airport__iata_code")
+            )
+            observed_payload = {
+                "reference_date": derivation.reference_date.isoformat(),
+                "reference_locator": derivation.reference_locator,
+                "reference_version_sha256": (
+                    derivation.reference_version_sha256
+                ),
+                "source_row_count": derivation.source_row_count,
+                "derivation_contract": (
+                    derivation.derivation_contract_fingerprint_sha256
+                ),
+                "routes": [
+                    _derived_route_payload(row)
+                    for row in routes
+                ],
+            }
+            if (
+                derivation.route_count != len(routes)
+                or derivation.typed_fingerprint_sha256
+                != expected_fingerprint
+                or observed_payload != expected_payload
+            ):
+                raise _IngestionRejected(
+                    FlightIngestionCode.PERSISTENCE_FAILED
+                )
+            return derivation
+
+        airports = {
+            row.iata_code: row
+            for row in Airport.objects.select_for_update().filter(
+                iata_code__in=expected_codes,
+                is_public=True,
+            )
+        }
+        if set(airports) != expected_codes:
+            raise _IngestionRejected(FlightIngestionCode.PERSISTENCE_FAILED)
+        created_at = max(timezone.now(), locked_run.completed_at)
+        RouteDurationDerivedRoute.objects.bulk_create(
+            [
+                RouteDurationDerivedRoute(
+                    parse_run=locked_run,
+                    destination_airport=airports[row.destination_iata],
+                    outbound_minutes=row.outbound_minutes,
+                    inbound_minutes=row.inbound_minutes,
+                    outbound_source_rows=row.outbound_source_rows,
+                    outbound_invalid_rows=row.outbound_invalid_rows,
+                    outbound_cancelled_rows=row.outbound_cancelled_rows,
+                    outbound_occurrences=row.outbound_occurrences,
+                    outbound_outliers=row.outbound_outliers,
+                    outbound_retained=row.outbound_retained,
+                    inbound_source_rows=row.inbound_source_rows,
+                    inbound_invalid_rows=row.inbound_invalid_rows,
+                    inbound_cancelled_rows=row.inbound_cancelled_rows,
+                    inbound_occurrences=row.inbound_occurrences,
+                    inbound_outliers=row.inbound_outliers,
+                    inbound_retained=row.inbound_retained,
+                    typed_fingerprint_sha256=_typed_fingerprint(
+                        _derived_route_payload(row)
+                    ),
+                    created_at=created_at,
+                )
+                for row in parsed.routes
+            ]
+        )
+        return RouteDurationDerivation.objects.create(
+            parse_run=locked_run,
+            reference_date=parsed.reference_date,
+            reference_locator=parsed.reference_locator,
+            reference_version_sha256=reference_version_sha256,
+            derivation_contract_fingerprint_sha256=(
+                parsed.derivation_fingerprint_sha256
+            ),
+            source_row_count=parsed.source_row_count,
+            route_count=len(parsed.routes),
+            typed_fingerprint_sha256=expected_fingerprint,
+            created_at=created_at,
+        )
+
+
+def ingest_route_duration_reference(
+    *,
+    fetch: RouteDurationReferenceFetchResult,
+    attempt_recorder: DurableAviationAttemptRecorder,
+    source_checked_at: datetime | None = None,
+) -> DurationIngestionOutcome:
+    checked_at = source_checked_at or timezone.now()
+    if (
+        type(attempt_recorder) is not DurableAviationAttemptRecorder
+        or not isinstance(checked_at, datetime)
+        or not timezone.is_aware(checked_at)
+    ):
+        return DurationIngestionOutcome(
+            FlightIngestionCode.PERSISTENCE_FAILED
+        )
+    try:
+        archive_input, reference_date = _duration_archive_input(
+            fetch, attempt_recorder
+        )
+        required_codes = frozenset(row[1] for row in CURATED_AIRPORT_ROWS)
+        parsed = derive_route_durations(
+            fetch.archive,
+            reference_date,
+            required_codes,
+        )
+        parse_run = _persist_parse_run(
+            inputs=(archive_input,),
+            parser_name=ParseRun.ParserName.ROUTE_DURATION_REFERENCE_FILE,
+            parser_version=ParseRun.ParserVersion.V2,
+            contract_fingerprint=(
+                DURATION_REFERENCE_CONTRACT_FINGERPRINT_SHA256
+            ),
+            schema_fingerprint=DURATION_REFERENCE_SCHEMA_FINGERPRINT_SHA256,
+            parsed=parsed,
+            completed_at=checked_at,
+        )
+        if isinstance(parsed, DurationReferenceFailure):
+            return DurationIngestionOutcome(
+                FlightIngestionCode.PARSE_QUARANTINED
+            )
+        if not isinstance(parsed, DurationReferenceSuccess):
+            return DurationIngestionOutcome(
+                FlightIngestionCode.PERSISTENCE_FAILED
+            )
+        derivation = _persist_duration_derivation(
+            parse_run=parse_run,
+            parsed=parsed,
+            reference_version_sha256=fetch.reference_version_sha256,
+        )
+    except _IngestionRejected as exc:
+        return DurationIngestionOutcome(exc.code)
+    except Exception:
+        return DurationIngestionOutcome(
+            FlightIngestionCode.PERSISTENCE_FAILED
+        )
+    return DurationIngestionOutcome(
+        FlightIngestionCode.STAGED,
+        str(derivation.pk),
+    )
+
+
 def _duration_failure_shape(
     parsed: AviationParseFailure,
 ) -> tuple[str, str]:
@@ -740,15 +1131,15 @@ def _duration_failure_shape(
         observed = parsed.observed_schema_fingerprint_sha256
         if (
             _SHA256_HEX.fullmatch(observed or "") is None
-            or observed == DURATION_SCHEMA_FINGERPRINT_SHA256
+            or observed == DURATION_V1_SCHEMA_FINGERPRINT_SHA256
         ):
             observed = "f" * 64
         return "SCHEMA_MISMATCH", observed
     if parsed.failure_code == "INCOMPLETE_COVERAGE":
-        return "REQUIRED_VALUE_MISSING", DURATION_SCHEMA_FINGERPRINT_SHA256
+        return "REQUIRED_VALUE_MISSING", DURATION_V1_SCHEMA_FINGERPRINT_SHA256
     if parsed.failure_code == "INVALID_SIZE":
         return "SYNTAX_ERROR", ""
-    return "INVALID_VALUE", DURATION_SCHEMA_FINGERPRINT_SHA256
+    return "INVALID_VALUE", DURATION_V1_SCHEMA_FINGERPRINT_SHA256
 
 
 def _record_duration_receipt(
@@ -774,26 +1165,26 @@ def _record_duration_receipt(
     elif (
         isinstance(parsed, DurationParseSuccess)
         and parsed.observed_schema_fingerprint_sha256
-        == DURATION_SCHEMA_FINGERPRINT_SHA256
+        == DURATION_V1_SCHEMA_FINGERPRINT_SHA256
     ):
         outcome = ReviewedDurationReceipt.Outcome.VALIDATED
         failure_code = ""
-        observed_schema = DURATION_SCHEMA_FINGERPRINT_SHA256
+        observed_schema = DURATION_V1_SCHEMA_FINGERPRINT_SHA256
     else:
         raise _IngestionRejected(FlightIngestionCode.PARSE_QUARANTINED)
     with transaction.atomic(durable=True):
-        source, rights = _locked_source(DURATION_SOURCE_CODE)
+        source, rights = _locked_v1_duration_source()
         return ReviewedDurationReceipt.objects.create(
             source=source,
-            source_revision=source.revision,
+            source_revision=DURATION_V1_SOURCE_REVISION,
             rights_decision=rights,
             body_sha256=hashlib.sha256(duration_csv).hexdigest(),
             byte_count=len(duration_csv),
             parser_contract_fingerprint_sha256=(
-                DURATION_CONTRACT_FINGERPRINT_SHA256
+                DURATION_V1_CONTRACT_FINGERPRINT_SHA256
             ),
             expected_schema_fingerprint_sha256=(
-                DURATION_SCHEMA_FINGERPRINT_SHA256
+                DURATION_V1_SCHEMA_FINGERPRINT_SHA256
             ),
             observed_schema_fingerprint_sha256=observed_schema,
             outcome=outcome,
@@ -805,6 +1196,94 @@ def _record_duration_receipt(
         )
 
 
+@dataclass(frozen=True, slots=True)
+class _ParsedFlightEvidence:
+    schedule_run: ParseRun
+    schedule: ScheduleParseSuccess | AviationParseFailure
+    city_reference_run: ParseRun
+    city_reference: CityReferenceParseSuccess | AviationParseFailure
+    legacy_arrivals_run: ParseRun
+    legacy_arrivals: LegacyArrivalParseSuccess | AviationParseFailure
+
+
+def _collect_flight_source_evidence(
+    *,
+    schedule_fetch: SchedulePageFetchResult,
+    reference_fetch: AviationReferenceFetchResult,
+    attempt_recorder: DurableAviationAttemptRecorder,
+    source_date: date,
+    checked_at: datetime,
+) -> _ParsedFlightEvidence:
+    schedule_inputs = _schedule_inputs(schedule_fetch, attempt_recorder)
+    city_input, legacy_inputs = _reference_inputs(
+        reference_fetch, attempt_recorder
+    )
+    schedule = adapt_data_go_schedule_pages(
+        departure_pages=schedule_fetch.departure_pages,
+        arrival_pages=schedule_fetch.arrival_pages,
+        source_date=source_date,
+    )
+    city_reference = parse_destination_city_reference(
+        reference_fetch.city_payload
+    )
+    legacy_arrivals = parse_legacy_arrival_pages(
+        reference_fetch.legacy_arrival_pages
+    )
+    schedule_run = _persist_parse_run(
+        inputs=schedule_inputs,
+        parser_name=ParseRun.ParserName.ICN_FLIGHT_SCHEDULE_JSON,
+        parser_version=ParseRun.ParserVersion.V2,
+        contract_fingerprint=SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
+        schema_fingerprint=SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
+        parsed=schedule,
+        completed_at=checked_at,
+    )
+    city_reference_run = _persist_parse_run(
+        inputs=(city_input,),
+        parser_name=ParseRun.ParserName.ICN_DESTINATION_CITY_JSON,
+        parser_version=ParseRun.ParserVersion.V3,
+        contract_fingerprint=CITY_CONTRACT_FINGERPRINT_SHA256,
+        schema_fingerprint=CITY_SCHEMA_FINGERPRINT_SHA256,
+        parsed=city_reference,
+        completed_at=checked_at,
+    )
+    legacy_arrivals_run = _persist_parse_run(
+        inputs=legacy_inputs,
+        parser_name=ParseRun.ParserName.ICN_LEGACY_ARRIVALS_JSON,
+        parser_version=ParseRun.ParserVersion.V2,
+        contract_fingerprint=(
+            LEGACY_SCHEDULE_CONTRACT_FINGERPRINT_SHA256
+        ),
+        schema_fingerprint=LEGACY_SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
+        parsed=legacy_arrivals,
+        completed_at=checked_at,
+    )
+    return _ParsedFlightEvidence(
+        schedule_run=schedule_run,
+        schedule=schedule,
+        city_reference_run=city_reference_run,
+        city_reference=city_reference,
+        legacy_arrivals_run=legacy_arrivals_run,
+        legacy_arrivals=legacy_arrivals,
+    )
+
+
+def _flight_publication_outcome(
+    outcome: object,
+) -> FlightIngestionOutcome:
+    code = getattr(outcome, "code", None)
+    if code == FlightPublicationCode.SOURCE_GATE_FAILED:
+        return FlightIngestionOutcome(FlightIngestionCode.SOURCE_GATE_FAILED)
+    if code == FlightPublicationCode.INVALID_EVIDENCE:
+        return FlightIngestionOutcome(FlightIngestionCode.PARSE_QUARANTINED)
+    if code != FlightPublicationCode.STAGED:
+        return FlightIngestionOutcome(FlightIngestionCode.PERSISTENCE_FAILED)
+    return FlightIngestionOutcome(
+        FlightIngestionCode.STAGED,
+        getattr(outcome, "schedule_revision_id", None),
+    )
+
+
 def ingest_flight_evidence_candidate(
     *,
     schedule_fetch: SchedulePageFetchResult,
@@ -831,52 +1310,14 @@ def ingest_flight_evidence_candidate(
     ):
         return FlightIngestionOutcome(FlightIngestionCode.PERSISTENCE_FAILED)
     try:
-        schedule_inputs = _schedule_inputs(schedule_fetch, attempt_recorder)
-        city_input, legacy_inputs = _reference_inputs(
-            reference_fetch, attempt_recorder
-        )
-        schedule = adapt_data_go_schedule_pages(
-            departure_pages=schedule_fetch.departure_pages,
-            arrival_pages=schedule_fetch.arrival_pages,
+        evidence = _collect_flight_source_evidence(
+            schedule_fetch=schedule_fetch,
+            reference_fetch=reference_fetch,
+            attempt_recorder=attempt_recorder,
             source_date=source_date,
-        )
-        city_reference = parse_destination_city_reference(
-            reference_fetch.city_payload
-        )
-        legacy_arrivals = parse_legacy_arrival_pages(
-            reference_fetch.legacy_arrival_pages
+            checked_at=checked_at,
         )
         durations = parse_route_durations(duration_csv)
-
-        schedule_run = _persist_parse_run(
-            inputs=schedule_inputs,
-            parser_name=ParseRun.ParserName.ICN_FLIGHT_SCHEDULE_JSON,
-            parser_version=ParseRun.ParserVersion.V2,
-            contract_fingerprint=SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
-            schema_fingerprint=SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
-            parsed=schedule,
-            completed_at=checked_at,
-        )
-        city_reference_run = _persist_parse_run(
-            inputs=(city_input,),
-            parser_name=ParseRun.ParserName.ICN_DESTINATION_CITY_JSON,
-            parser_version=ParseRun.ParserVersion.V3,
-            contract_fingerprint=CITY_CONTRACT_FINGERPRINT_SHA256,
-            schema_fingerprint=CITY_SCHEMA_FINGERPRINT_SHA256,
-            parsed=city_reference,
-            completed_at=checked_at,
-        )
-        legacy_arrivals_run = _persist_parse_run(
-            inputs=legacy_inputs,
-            parser_name=ParseRun.ParserName.ICN_LEGACY_ARRIVALS_JSON,
-            parser_version=ParseRun.ParserVersion.V2,
-            contract_fingerprint=(
-                LEGACY_SCHEDULE_CONTRACT_FINGERPRINT_SHA256
-            ),
-            schema_fingerprint=LEGACY_SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
-            parsed=legacy_arrivals,
-            completed_at=checked_at,
-        )
         duration_receipt = _record_duration_receipt(
             duration_csv=duration_csv,
             parsed=durations,
@@ -887,9 +1328,9 @@ def ingest_flight_evidence_candidate(
         if any(
             isinstance(result, AviationParseFailure)
             for result in (
-                schedule,
-                city_reference,
-                legacy_arrivals,
+                evidence.schedule,
+                evidence.city_reference,
+                evidence.legacy_arrivals,
                 durations,
             )
         ):
@@ -897,9 +1338,13 @@ def ingest_flight_evidence_candidate(
                 FlightIngestionCode.PARSE_QUARANTINED
             )
         if (
-            not isinstance(schedule, ScheduleParseSuccess)
-            or not isinstance(city_reference, CityReferenceParseSuccess)
-            or not isinstance(legacy_arrivals, LegacyArrivalParseSuccess)
+            not isinstance(evidence.schedule, ScheduleParseSuccess)
+            or not isinstance(
+                evidence.city_reference, CityReferenceParseSuccess
+            )
+            or not isinstance(
+                evidence.legacy_arrivals, LegacyArrivalParseSuccess
+            )
             or not isinstance(durations, DurationParseSuccess)
             or duration_receipt.outcome
             != ReviewedDurationReceipt.Outcome.VALIDATED
@@ -908,12 +1353,12 @@ def ingest_flight_evidence_candidate(
                 FlightIngestionCode.PARSE_QUARANTINED
             )
         outcome = stage_flight_evidence(
-            schedule_run=schedule_run,
-            schedule=schedule,
-            city_reference_run=city_reference_run,
-            city_reference=city_reference,
-            legacy_arrivals_run=legacy_arrivals_run,
-            legacy_arrivals=legacy_arrivals,
+            schedule_run=evidence.schedule_run,
+            schedule=evidence.schedule,
+            city_reference_run=evidence.city_reference_run,
+            city_reference=evidence.city_reference,
+            legacy_arrivals_run=evidence.legacy_arrivals_run,
+            legacy_arrivals=evidence.legacy_arrivals,
             duration_receipt=duration_receipt,
             durations=durations,
             source_checked_at=checked_at,
@@ -922,13 +1367,83 @@ def ingest_flight_evidence_candidate(
         return FlightIngestionOutcome(exc.code)
     except Exception:
         return FlightIngestionOutcome(FlightIngestionCode.PERSISTENCE_FAILED)
-    if outcome.code == FlightPublicationCode.SOURCE_GATE_FAILED:
-        return FlightIngestionOutcome(FlightIngestionCode.SOURCE_GATE_FAILED)
-    if outcome.code == FlightPublicationCode.INVALID_EVIDENCE:
-        return FlightIngestionOutcome(FlightIngestionCode.PARSE_QUARANTINED)
-    if outcome.code != FlightPublicationCode.STAGED:
+    return _flight_publication_outcome(outcome)
+
+
+def ingest_flight_evidence_candidate_with_derivation(
+    *,
+    schedule_fetch: SchedulePageFetchResult,
+    reference_fetch: AviationReferenceFetchResult,
+    attempt_recorder: DurableAviationAttemptRecorder,
+    duration_derivation_id: str | uuid.UUID,
+    source_date: date,
+    source_checked_at: datetime | None = None,
+) -> FlightIngestionOutcome:
+    """Stage live schedule evidence against one exact official derivation."""
+
+    checked_at = source_checked_at or timezone.now()
+    try:
+        derivation_id = uuid.UUID(str(duration_derivation_id))
+    except (AttributeError, TypeError, ValueError):
         return FlightIngestionOutcome(FlightIngestionCode.PERSISTENCE_FAILED)
-    return FlightIngestionOutcome(
-        FlightIngestionCode.STAGED,
-        outcome.schedule_revision_id,
-    )
+    if (
+        type(attempt_recorder) is not DurableAviationAttemptRecorder
+        or not isinstance(source_date, date)
+        or not isinstance(checked_at, datetime)
+        or not timezone.is_aware(checked_at)
+    ):
+        return FlightIngestionOutcome(FlightIngestionCode.PERSISTENCE_FAILED)
+    try:
+        evidence = _collect_flight_source_evidence(
+            schedule_fetch=schedule_fetch,
+            reference_fetch=reference_fetch,
+            attempt_recorder=attempt_recorder,
+            source_date=source_date,
+            checked_at=checked_at,
+        )
+        if any(
+            isinstance(result, AviationParseFailure)
+            for result in (
+                evidence.schedule,
+                evidence.city_reference,
+                evidence.legacy_arrivals,
+            )
+        ):
+            return FlightIngestionOutcome(
+                FlightIngestionCode.PARSE_QUARANTINED
+            )
+        if (
+            not isinstance(evidence.schedule, ScheduleParseSuccess)
+            or not isinstance(
+                evidence.city_reference, CityReferenceParseSuccess
+            )
+            or not isinstance(
+                evidence.legacy_arrivals, LegacyArrivalParseSuccess
+            )
+        ):
+            return FlightIngestionOutcome(
+                FlightIngestionCode.PARSE_QUARANTINED
+            )
+        try:
+            duration_derivation = RouteDurationDerivation.objects.get(
+                pk=derivation_id
+            )
+        except RouteDurationDerivation.DoesNotExist:
+            raise _IngestionRejected(
+                FlightIngestionCode.SOURCE_GATE_FAILED
+            ) from None
+        outcome = stage_flight_evidence(
+            schedule_run=evidence.schedule_run,
+            schedule=evidence.schedule,
+            city_reference_run=evidence.city_reference_run,
+            city_reference=evidence.city_reference,
+            legacy_arrivals_run=evidence.legacy_arrivals_run,
+            legacy_arrivals=evidence.legacy_arrivals,
+            duration_derivation=duration_derivation,
+            source_checked_at=checked_at,
+        )
+    except _IngestionRejected as exc:
+        return FlightIngestionOutcome(exc.code)
+    except Exception:
+        return FlightIngestionOutcome(FlightIngestionCode.PERSISTENCE_FAILED)
+    return _flight_publication_outcome(outcome)
diff --git a/travel_windows/management/commands/collect_route_duration_reference.py b/travel_windows/management/commands/collect_route_duration_reference.py
new file mode 100644
index 0000000..3957730
--- /dev/null
+++ b/travel_windows/management/commands/collect_route_duration_reference.py
@@ -0,0 +1,74 @@
+from django.core.management.base import BaseCommand, CommandError
+
+from sources.models import SourceConfiguration
+from sources.transport import fetch_route_duration_reference
+from travel_windows.contracts import (
+    DURATION_SOURCE_CODE,
+    DURATION_SOURCE_LOCATOR,
+    DURATION_SOURCE_REVISION,
+)
+from travel_windows.ingestion import (
+    DurableAviationAttemptRecorder,
+    ingest_route_duration_reference,
+)
+from travel_windows.models import RouteDurationDerivation
+
+
+class Command(BaseCommand):
+    help = (
+        "Check the official route-duration reference and stage a new sealed "
+        "derivation only when the published file identity changes."
+    )
+    requires_migrations_checks = True
+
+    def handle(self, *args, **options):
+        try:
+            source = SourceConfiguration.objects.get(
+                code=DURATION_SOURCE_CODE,
+                revision=DURATION_SOURCE_REVISION,
+                state=SourceConfiguration.State.ACTIVE,
+                enabled=True,
+                official_locator=DURATION_SOURCE_LOCATOR,
+                secret_reference_name="",
+            )
+        except SourceConfiguration.DoesNotExist:
+            raise CommandError(
+                "result=blocked code=SOURCE_GATE_FAILED"
+            ) from None
+
+        latest = RouteDurationDerivation.objects.order_by(
+            "-created_at", "-id"
+        ).first()
+        known_version = (
+            latest.reference_version_sha256 if latest is not None else ""
+        )
+        recorder = DurableAviationAttemptRecorder()
+        fetched = fetch_route_duration_reference(
+            official_locator=source.official_locator,
+            connect_timeout_seconds=source.connect_timeout_seconds,
+            read_timeout_seconds=source.read_timeout_seconds,
+            known_reference_version_sha256=known_version,
+            attempt_executor=recorder,
+        )
+        if not fetched.succeeded:
+            raise CommandError(
+                f"result=blocked code={fetched.failure_code}"
+            )
+        if fetched.unchanged:
+            self.stdout.write(
+                "result=unchanged "
+                f"reference_date={fetched.reference_date}"
+            )
+            return
+
+        outcome = ingest_route_duration_reference(
+            fetch=fetched,
+            attempt_recorder=recorder,
+        )
+        if not outcome.succeeded:
+            raise CommandError(f"result=blocked code={outcome.code}")
+        self.stdout.write(
+            "result=review-required "
+            f"duration_derivation={outcome.derivation_id} "
+            f"reference_date={fetched.reference_date}"
+        )
diff --git a/travel_windows/migrations/0010_route_duration_derivation.py b/travel_windows/migrations/0010_route_duration_derivation.py
new file mode 100644
index 0000000..1dbdf11
--- /dev/null
+++ b/travel_windows/migrations/0010_route_duration_derivation.py
@@ -0,0 +1,547 @@
+# Generated by Django 5.2.17 on 2026-08-31
+
+import uuid
+
+import django.db.models.deletion
+import django.utils.timezone
+from django.db import migrations, models
+
+
+ROUTE_DURATION_DERIVATION_SQL = r"""
+CREATE OR REPLACE FUNCTION travel_windows_guard_duration_receipt()
+RETURNS trigger LANGUAGE plpgsql AS $$
+DECLARE
+    source_code text;
+    locked_source_revision text;
+    source_module text;
+    source_state text;
+    source_enabled boolean;
+    source_locator text;
+    source_secret_reference text;
+    rights_source_id uuid;
+    rights_source_revision text;
+    rights_decision text;
+    rights_access_allowed boolean;
+    rights_collection_allowed boolean;
+    rights_storage_allowed boolean;
+    rights_raw_allowed boolean;
+    rights_republication_allowed boolean;
+    rights_raw_retention integer;
+    rights_typed_retention text;
+    rights_evidence_retention text;
+    rights_scope text;
+    rights_contract text;
+    rights_decided_at timestamptz;
+    latest_rights_id uuid;
+    source_and_rights_found boolean;
+    latest_rights_found boolean;
+BEGIN
+    IF TG_OP <> 'INSERT' THEN
+        RAISE EXCEPTION 'reviewed duration receipts are append-only'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    SELECT source.code, source.revision, source.module, source.state,
+           source.enabled, source.official_locator,
+           source.secret_reference_name, rights.source_id,
+           rights.source_revision, rights.decision, rights.access_allowed,
+           rights.automated_collection_allowed,
+           rights.typed_field_storage_allowed,
+           rights.raw_body_storage_allowed,
+           rights.typed_republication_allowed, rights.raw_retention_seconds,
+           rights.typed_retention, rights.evidence_retention,
+           rights.field_scope_code, rights.contract_fingerprint_sha256,
+           rights.decided_at
+      INTO source_code, locked_source_revision, source_module, source_state,
+           source_enabled, source_locator, source_secret_reference,
+           rights_source_id, rights_source_revision, rights_decision,
+           rights_access_allowed, rights_collection_allowed,
+           rights_storage_allowed, rights_raw_allowed,
+           rights_republication_allowed, rights_raw_retention,
+           rights_typed_retention, rights_evidence_retention, rights_scope,
+           rights_contract, rights_decided_at
+      FROM sources_sourceconfiguration AS source
+      JOIN sources_sourcerightsdecision AS rights
+        ON rights.id = NEW.rights_decision_id
+     WHERE source.id = NEW.source_id
+     FOR UPDATE OF source, rights;
+    source_and_rights_found := FOUND;
+    SELECT id INTO latest_rights_id
+      FROM sources_sourcerightsdecision AS latest_rights
+     WHERE latest_rights.source_id = NEW.source_id
+       AND latest_rights.source_revision = 'travel-v1'
+     ORDER BY latest_rights.decision_sequence DESC, latest_rights.id DESC
+     LIMIT 1;
+    latest_rights_found := FOUND;
+    IF NOT source_and_rights_found
+       OR NOT latest_rights_found
+       OR source_code <> 'PORT_LOGISTICS_ROUTE_DURATION_15151728'
+       OR locked_source_revision NOT IN ('travel-v1', 'travel-v2')
+       OR source_module <> 'AVIATION'
+       OR source_state <> 'ACTIVE'
+       OR NOT source_enabled
+       OR source_locator <> 'https://www.data.go.kr/data/15151728/fileData.do'
+       OR source_secret_reference <> ''
+       OR NEW.source_revision <> 'travel-v1'
+       OR rights_source_id IS DISTINCT FROM NEW.source_id
+       OR rights_source_revision <> 'travel-v1'
+       OR latest_rights_id IS DISTINCT FROM NEW.rights_decision_id
+       OR rights_decision <> 'APPROVED'
+       OR NOT rights_access_allowed
+       OR NOT rights_collection_allowed
+       OR NOT rights_storage_allowed
+       OR rights_raw_allowed
+       OR NOT rights_republication_allowed
+       OR rights_raw_retention <> 0
+       OR rights_typed_retention <> 'PRODUCT_HISTORY'
+       OR rights_evidence_retention <> 'PRODUCT_HISTORY'
+       OR rights_scope <> 'ROUTE_DURATION_V1'
+       OR rights_contract <>
+          '430903be2d56367dfc1ca9617e69e6317ff76dadb54912fd02ca1e34d88fee01'
+       OR NEW.parser_contract_fingerprint_sha256 IS DISTINCT FROM rights_contract
+       OR NEW.expected_schema_fingerprint_sha256 <>
+          '0fe301b62df9abd8b449aeeb1e6ea62cbca80ab7c61e35b6cee552aa58278307'
+       OR NEW.reviewed_at < rights_decided_at THEN
+        RAISE EXCEPTION 'reviewed duration receipt is outside the approved source contract'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    RETURN NEW;
+END;
+$$;
+
+CREATE TRIGGER travel_windows_duration_derived_route_immutable
+BEFORE UPDATE OR DELETE ON travel_windows_routedurationderivedroute
+FOR EACH ROW EXECUTE FUNCTION travel_windows_reject_revision_mutation();
+
+CREATE TRIGGER travel_windows_duration_derivation_immutable
+BEFORE UPDATE OR DELETE ON travel_windows_routedurationderivation
+FOR EACH ROW EXECUTE FUNCTION travel_windows_reject_revision_mutation();
+
+CREATE FUNCTION travel_windows_guard_duration_derived_route_insert()
+RETURNS trigger LANGUAGE plpgsql AS $$
+DECLARE
+    run_outcome text;
+    run_parser_name text;
+    run_parser_version text;
+    run_completed_at timestamptz;
+    run_exists boolean;
+BEGIN
+    IF EXISTS (
+        SELECT 1 FROM travel_windows_routedurationderivation
+         WHERE parse_run_id = NEW.parse_run_id
+    ) THEN
+        RAISE EXCEPTION 'sealed route duration membership is immutable'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    SELECT outcome, parser_name, parser_version, completed_at
+      INTO run_outcome, run_parser_name, run_parser_version, run_completed_at
+      FROM sources_parserun
+     WHERE id = NEW.parse_run_id
+     FOR UPDATE;
+    run_exists := FOUND;
+    IF NOT run_exists
+       OR run_outcome <> 'VALIDATED'
+       OR run_parser_name <> 'ROUTE_DURATION_REFERENCE_FILE'
+       OR run_parser_version <> 'V2'
+       OR run_completed_at IS NULL
+       OR NEW.created_at < run_completed_at THEN
+        RAISE EXCEPTION 'derived route requires validated official reference parse'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    RETURN NEW;
+END;
+$$;
+CREATE TRIGGER travel_windows_duration_derived_route_insert_guard
+BEFORE INSERT ON travel_windows_routedurationderivedroute
+FOR EACH ROW EXECUTE FUNCTION travel_windows_guard_duration_derived_route_insert();
+
+CREATE FUNCTION travel_windows_guard_duration_derivation_insert()
+RETURNS trigger LANGUAGE plpgsql AS $$
+DECLARE
+    run_outcome text;
+    run_parser_name text;
+    run_parser_version text;
+    run_completed_at timestamptz;
+    run_exists boolean;
+    observed_route_count bigint;
+    observed_source_rows bigint;
+BEGIN
+    SELECT outcome, parser_name, parser_version, completed_at
+      INTO run_outcome, run_parser_name, run_parser_version, run_completed_at
+      FROM sources_parserun
+     WHERE id = NEW.parse_run_id
+     FOR UPDATE;
+    run_exists := FOUND;
+    SELECT COUNT(*),
+           COALESCE(SUM(outbound_source_rows + inbound_source_rows), 0)
+      INTO observed_route_count, observed_source_rows
+      FROM travel_windows_routedurationderivedroute
+     WHERE parse_run_id = NEW.parse_run_id;
+    IF NOT run_exists
+       OR run_outcome <> 'VALIDATED'
+       OR run_parser_name <> 'ROUTE_DURATION_REFERENCE_FILE'
+       OR run_parser_version <> 'V2'
+       OR run_completed_at IS NULL
+       OR NEW.created_at < run_completed_at
+       OR NEW.reference_locator <>
+          'https://www.data.go.kr/data/15151728/fileData.do'
+       OR NEW.derivation_contract_fingerprint_sha256 <>
+          '40de0cd89650b144cd7a8bd8bf6af664378e066ca1b45defb96d932d923bd9d9'
+       OR NEW.route_count IS DISTINCT FROM observed_route_count
+       OR NEW.source_row_count < observed_source_rows THEN
+        RAISE EXCEPTION 'route duration derivation requires exact sealed membership'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    RETURN NEW;
+END;
+$$;
+CREATE TRIGGER travel_windows_duration_derivation_insert_guard
+BEFORE INSERT ON travel_windows_routedurationderivation
+FOR EACH ROW EXECUTE FUNCTION travel_windows_guard_duration_derivation_insert();
+"""
+
+
+ROUTE_DURATION_DERIVATION_REVERSE_SQL = r"""
+DO $$
+BEGIN
+    IF EXISTS (SELECT 1 FROM travel_windows_routedurationderivation)
+       OR EXISTS (SELECT 1 FROM travel_windows_routedurationderivedroute) THEN
+        RAISE EXCEPTION 'route duration derivation rollback requires empty evidence'
+            USING ERRCODE = 'check_violation';
+    END IF;
+END;
+$$;
+CREATE OR REPLACE FUNCTION travel_windows_guard_duration_receipt()
+RETURNS trigger LANGUAGE plpgsql AS $$
+DECLARE
+    source_code text;
+    locked_source_revision text;
+    source_module text;
+    source_state text;
+    source_enabled boolean;
+    source_locator text;
+    source_secret_reference text;
+    rights_source_id uuid;
+    rights_source_revision text;
+    rights_decision text;
+    rights_access_allowed boolean;
+    rights_collection_allowed boolean;
+    rights_storage_allowed boolean;
+    rights_raw_allowed boolean;
+    rights_republication_allowed boolean;
+    rights_raw_retention integer;
+    rights_typed_retention text;
+    rights_evidence_retention text;
+    rights_scope text;
+    rights_contract text;
+    rights_decided_at timestamptz;
+    latest_rights_id uuid;
+BEGIN
+    IF TG_OP <> 'INSERT' THEN
+        RAISE EXCEPTION 'reviewed duration receipts are append-only'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    SELECT source.code, source.revision, source.module, source.state,
+           source.enabled, source.official_locator,
+           source.secret_reference_name, rights.source_id,
+           rights.source_revision, rights.decision, rights.access_allowed,
+           rights.automated_collection_allowed,
+           rights.typed_field_storage_allowed,
+           rights.raw_body_storage_allowed,
+           rights.typed_republication_allowed, rights.raw_retention_seconds,
+           rights.typed_retention, rights.evidence_retention,
+           rights.field_scope_code, rights.contract_fingerprint_sha256,
+           rights.decided_at
+      INTO source_code, locked_source_revision, source_module, source_state,
+           source_enabled, source_locator, source_secret_reference,
+           rights_source_id, rights_source_revision, rights_decision,
+           rights_access_allowed, rights_collection_allowed,
+           rights_storage_allowed, rights_raw_allowed,
+           rights_republication_allowed, rights_raw_retention,
+           rights_typed_retention, rights_evidence_retention, rights_scope,
+           rights_contract, rights_decided_at
+      FROM sources_sourceconfiguration AS source
+      JOIN sources_sourcerightsdecision AS rights
+        ON rights.id = NEW.rights_decision_id
+     WHERE source.id = NEW.source_id
+     FOR UPDATE OF source, rights;
+    SELECT id INTO latest_rights_id
+      FROM sources_sourcerightsdecision AS latest_rights
+     WHERE latest_rights.source_id = NEW.source_id
+       AND latest_rights.source_revision = locked_source_revision
+     ORDER BY latest_rights.decision_sequence DESC, latest_rights.id DESC
+     LIMIT 1;
+    IF NOT FOUND
+       OR source_code <> 'PORT_LOGISTICS_ROUTE_DURATION_15151728'
+       OR locked_source_revision <> 'travel-v1'
+       OR source_module <> 'AVIATION'
+       OR source_state <> 'ACTIVE'
+       OR NOT source_enabled
+       OR source_locator <> 'https://www.data.go.kr/data/15151728/fileData.do'
+       OR source_secret_reference <> ''
+       OR NEW.source_revision IS DISTINCT FROM locked_source_revision
+       OR rights_source_id IS DISTINCT FROM NEW.source_id
+       OR rights_source_revision IS DISTINCT FROM locked_source_revision
+       OR latest_rights_id IS DISTINCT FROM NEW.rights_decision_id
+       OR rights_decision <> 'APPROVED'
+       OR NOT rights_access_allowed
+       OR NOT rights_collection_allowed
+       OR NOT rights_storage_allowed
+       OR rights_raw_allowed
+       OR NOT rights_republication_allowed
+       OR rights_raw_retention <> 0
+       OR rights_typed_retention <> 'PRODUCT_HISTORY'
+       OR rights_evidence_retention <> 'PRODUCT_HISTORY'
+       OR rights_scope <> 'ROUTE_DURATION_V1'
+       OR rights_contract <>
+          '430903be2d56367dfc1ca9617e69e6317ff76dadb54912fd02ca1e34d88fee01'
+       OR NEW.parser_contract_fingerprint_sha256 IS DISTINCT FROM rights_contract
+       OR NEW.expected_schema_fingerprint_sha256 <>
+          '0fe301b62df9abd8b449aeeb1e6ea62cbca80ab7c61e35b6cee552aa58278307'
+       OR NEW.reviewed_at < rights_decided_at THEN
+        RAISE EXCEPTION 'reviewed duration receipt is outside the approved source contract'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    RETURN NEW;
+END;
+$$;
+DROP TRIGGER IF EXISTS travel_windows_duration_derivation_insert_guard
+    ON travel_windows_routedurationderivation;
+DROP FUNCTION IF EXISTS travel_windows_guard_duration_derivation_insert();
+DROP TRIGGER IF EXISTS travel_windows_duration_derived_route_insert_guard
+    ON travel_windows_routedurationderivedroute;
+DROP FUNCTION IF EXISTS travel_windows_guard_duration_derived_route_insert();
+DROP TRIGGER IF EXISTS travel_windows_duration_derivation_immutable
+    ON travel_windows_routedurationderivation;
+DROP TRIGGER IF EXISTS travel_windows_duration_derived_route_immutable
+    ON travel_windows_routedurationderivedroute;
+"""
+
+
+class Migration(migrations.Migration):
+
+    dependencies = [
+        ("sources", "0016_route_duration_reference_contract"),
+        ("travel_windows", "0009_aviation_collection_receipts"),
+    ]
+
+    operations = [
+        migrations.CreateModel(
+            name="RouteDurationDerivedRoute",
+            fields=[
+                (
+                    "id",
+                    models.UUIDField(
+                        default=uuid.uuid4,
+                        editable=False,
+                        primary_key=True,
+                        serialize=False,
+                    ),
+                ),
+                ("outbound_minutes", models.PositiveSmallIntegerField()),
+                ("inbound_minutes", models.PositiveSmallIntegerField()),
+                ("outbound_source_rows", models.PositiveIntegerField()),
+                ("outbound_invalid_rows", models.PositiveIntegerField()),
+                ("outbound_cancelled_rows", models.PositiveIntegerField()),
+                ("outbound_occurrences", models.PositiveIntegerField()),
+                ("outbound_outliers", models.PositiveIntegerField()),
+                ("outbound_retained", models.PositiveIntegerField()),
+                ("inbound_source_rows", models.PositiveIntegerField()),
+                ("inbound_invalid_rows", models.PositiveIntegerField()),
+                ("inbound_cancelled_rows", models.PositiveIntegerField()),
+                ("inbound_occurrences", models.PositiveIntegerField()),
+                ("inbound_outliers", models.PositiveIntegerField()),
+                ("inbound_retained", models.PositiveIntegerField()),
+                ("typed_fingerprint_sha256", models.CharField(max_length=64)),
+                (
+                    "created_at",
+                    models.DateTimeField(default=django.utils.timezone.now),
+                ),
+                (
+                    "destination_airport",
+                    models.ForeignKey(
+                        on_delete=django.db.models.deletion.PROTECT,
+                        related_name="derived_duration_routes",
+                        to="travel_windows.airport",
+                    ),
+                ),
+                (
+                    "parse_run",
+                    models.ForeignKey(
+                        on_delete=django.db.models.deletion.PROTECT,
+                        related_name="derived_duration_routes",
+                        to="sources.parserun",
+                    ),
+                ),
+            ],
+            options={"ordering": ("destination_airport__iata_code",)},
+        ),
+        migrations.CreateModel(
+            name="RouteDurationDerivation",
+            fields=[
+                (
+                    "id",
+                    models.UUIDField(
+                        default=uuid.uuid4,
+                        editable=False,
+                        primary_key=True,
+                        serialize=False,
+                    ),
+                ),
+                ("reference_date", models.DateField()),
+                ("reference_locator", models.URLField(max_length=500)),
+                ("reference_version_sha256", models.CharField(max_length=64)),
+                (
+                    "derivation_contract_fingerprint_sha256",
+                    models.CharField(max_length=64),
+                ),
+                ("source_row_count", models.PositiveIntegerField()),
+                ("route_count", models.PositiveSmallIntegerField()),
+                ("typed_fingerprint_sha256", models.CharField(max_length=64)),
+                (
+                    "created_at",
+                    models.DateTimeField(default=django.utils.timezone.now),
+                ),
+                (
+                    "parse_run",
+                    models.OneToOneField(
+                        on_delete=django.db.models.deletion.PROTECT,
+                        related_name="duration_derivation",
+                        to="sources.parserun",
+                    ),
+                ),
+            ],
+        ),
+        migrations.AddConstraint(
+            model_name="routedurationderivedroute",
+            constraint=models.UniqueConstraint(
+                fields=("parse_run", "destination_airport"),
+                name="duration_derived_run_airport_unique",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="routedurationderivedroute",
+            constraint=models.CheckConstraint(
+                condition=models.Q(
+                    ("inbound_minutes__gte", 30),
+                    ("inbound_minutes__lte", 720),
+                    ("outbound_minutes__gte", 30),
+                    ("outbound_minutes__lte", 720),
+                ),
+                name="duration_derived_minutes_bounded",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="routedurationderivedroute",
+            constraint=models.CheckConstraint(
+                condition=models.Q(
+                    ("outbound_occurrences__gte", 5),
+                    ("outbound_retained__gte", 5),
+                    ("inbound_occurrences__gte", 5),
+                    ("inbound_retained__gte", 5),
+                    (
+                        "outbound_retained__lte",
+                        models.F("outbound_occurrences"),
+                    ),
+                    (
+                        "inbound_retained__lte",
+                        models.F("inbound_occurrences"),
+                    ),
+                    (
+                        "outbound_outliers__lte",
+                        models.F("outbound_occurrences"),
+                    ),
+                    (
+                        "inbound_outliers__lte",
+                        models.F("inbound_occurrences"),
+                    ),
+                ),
+                name="duration_derived_samples_consistent",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="routedurationderivedroute",
+            constraint=models.CheckConstraint(
+                condition=(
+                    models.Q(
+                        (
+                            "outbound_source_rows__gte",
+                            models.F("outbound_invalid_rows")
+                            + models.F("outbound_cancelled_rows"),
+                        )
+                    )
+                    & models.Q(
+                        (
+                            "inbound_source_rows__gte",
+                            models.F("inbound_invalid_rows")
+                            + models.F("inbound_cancelled_rows"),
+                        )
+                    )
+                ),
+                name="duration_derived_row_counts_consistent",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="routedurationderivedroute",
+            constraint=models.CheckConstraint(
+                condition=models.Q(
+                    ("typed_fingerprint_sha256__regex", "^[0-9a-f]{64}$")
+                ),
+                name="duration_derived_route_hash",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="routedurationderivation",
+            constraint=models.CheckConstraint(
+                condition=models.Q(("reference_locator__startswith", "https://")),
+                name="duration_derivation_locator_https",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="routedurationderivation",
+            constraint=models.CheckConstraint(
+                condition=models.Q(
+                    ("reference_version_sha256__regex", "^[0-9a-f]{64}$")
+                ),
+                name="duration_derivation_version_hash",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="routedurationderivation",
+            constraint=models.CheckConstraint(
+                condition=models.Q(("route_count", 13)),
+                name="duration_derivation_route_count_exact",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="routedurationderivation",
+            constraint=models.CheckConstraint(
+                condition=models.Q(("source_row_count__gte", 1)),
+                name="duration_derivation_source_rows_positive",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="routedurationderivation",
+            constraint=models.CheckConstraint(
+                condition=models.Q(
+                    (
+                        "derivation_contract_fingerprint_sha256__regex",
+                        "^[0-9a-f]{64}$",
+                    )
+                ),
+                name="duration_derivation_contract_hash",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="routedurationderivation",
+            constraint=models.CheckConstraint(
+                condition=models.Q(
+                    ("typed_fingerprint_sha256__regex", "^[0-9a-f]{64}$")
+                ),
+                name="duration_derivation_typed_hash",
+            ),
+        ),
+        migrations.RunSQL(
+            ROUTE_DURATION_DERIVATION_SQL,
+            ROUTE_DURATION_DERIVATION_REVERSE_SQL,
+        ),
+    ]
diff --git a/travel_windows/models.py b/travel_windows/models.py
index 247346f..4fb5cb1 100644
--- a/travel_windows/models.py
+++ b/travel_windows/models.py
@@ -392,6 +392,141 @@ class ReviewedDurationReceipt(models.Model):
         ]
 
 
+class RouteDurationDerivedRoute(models.Model):
+    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
+    parse_run = models.ForeignKey(
+        "sources.ParseRun",
+        on_delete=models.PROTECT,
+        related_name="derived_duration_routes",
+    )
+    destination_airport = models.ForeignKey(
+        Airport,
+        on_delete=models.PROTECT,
+        related_name="derived_duration_routes",
+    )
+    outbound_minutes = models.PositiveSmallIntegerField()
+    inbound_minutes = models.PositiveSmallIntegerField()
+    outbound_source_rows = models.PositiveIntegerField()
+    outbound_invalid_rows = models.PositiveIntegerField()
+    outbound_cancelled_rows = models.PositiveIntegerField()
+    outbound_occurrences = models.PositiveIntegerField()
+    outbound_outliers = models.PositiveIntegerField()
+    outbound_retained = models.PositiveIntegerField()
+    inbound_source_rows = models.PositiveIntegerField()
+    inbound_invalid_rows = models.PositiveIntegerField()
+    inbound_cancelled_rows = models.PositiveIntegerField()
+    inbound_occurrences = models.PositiveIntegerField()
+    inbound_outliers = models.PositiveIntegerField()
+    inbound_retained = models.PositiveIntegerField()
+    typed_fingerprint_sha256 = models.CharField(max_length=64)
+    created_at = models.DateTimeField(default=timezone.now)
+
+    class Meta:
+        ordering = ("destination_airport__iata_code",)
+        constraints = [
+            models.UniqueConstraint(
+                fields=("parse_run", "destination_airport"),
+                name="duration_derived_run_airport_unique",
+            ),
+            models.CheckConstraint(
+                condition=Q(
+                    outbound_minutes__gte=30,
+                    outbound_minutes__lte=720,
+                    inbound_minutes__gte=30,
+                    inbound_minutes__lte=720,
+                ),
+                name="duration_derived_minutes_bounded",
+            ),
+            models.CheckConstraint(
+                condition=(
+                    Q(outbound_occurrences__gte=5, outbound_retained__gte=5)
+                    & Q(inbound_occurrences__gte=5, inbound_retained__gte=5)
+                    & Q(outbound_retained__lte=F("outbound_occurrences"))
+                    & Q(inbound_retained__lte=F("inbound_occurrences"))
+                    & Q(outbound_outliers__lte=F("outbound_occurrences"))
+                    & Q(inbound_outliers__lte=F("inbound_occurrences"))
+                ),
+                name="duration_derived_samples_consistent",
+            ),
+            models.CheckConstraint(
+                condition=(
+                    Q(
+                        outbound_source_rows__gte=(
+                            F("outbound_invalid_rows")
+                            + F("outbound_cancelled_rows")
+                        )
+                    )
+                    & Q(
+                        inbound_source_rows__gte=(
+                            F("inbound_invalid_rows")
+                            + F("inbound_cancelled_rows")
+                        )
+                    )
+                ),
+                name="duration_derived_row_counts_consistent",
+            ),
+            models.CheckConstraint(
+                condition=Q(
+                    typed_fingerprint_sha256__regex=r"^[0-9a-f]{64}$"
+                ),
+                name="duration_derived_route_hash",
+            ),
+        ]
+
+
+class RouteDurationDerivation(models.Model):
+    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
+    parse_run = models.OneToOneField(
+        "sources.ParseRun",
+        on_delete=models.PROTECT,
+        related_name="duration_derivation",
+    )
+    reference_date = models.DateField()
+    reference_locator = models.URLField(max_length=500)
+    reference_version_sha256 = models.CharField(max_length=64)
+    derivation_contract_fingerprint_sha256 = models.CharField(max_length=64)
+    source_row_count = models.PositiveIntegerField()
+    route_count = models.PositiveSmallIntegerField()
+    typed_fingerprint_sha256 = models.CharField(max_length=64)
+    created_at = models.DateTimeField(default=timezone.now)
+
+    class Meta:
+        constraints = [
+            models.CheckConstraint(
+                condition=Q(reference_locator__startswith="https://"),
+                name="duration_derivation_locator_https",
+            ),
+            models.CheckConstraint(
+                condition=Q(
+                    reference_version_sha256__regex=r"^[0-9a-f]{64}$"
+                ),
+                name="duration_derivation_version_hash",
+            ),
+            models.CheckConstraint(
+                condition=Q(route_count=13),
+                name="duration_derivation_route_count_exact",
+            ),
+            models.CheckConstraint(
+                condition=Q(source_row_count__gte=1),
+                name="duration_derivation_source_rows_positive",
+            ),
+            models.CheckConstraint(
+                condition=Q(
+                    derivation_contract_fingerprint_sha256__regex=(
+                        r"^[0-9a-f]{64}$"
+                    )
+                ),
+                name="duration_derivation_contract_hash",
+            ),
+            models.CheckConstraint(
+                condition=Q(
+                    typed_fingerprint_sha256__regex=r"^[0-9a-f]{64}$"
+                ),
+                name="duration_derivation_typed_hash",
+            ),
+        ]
+
+
 class RouteDurationRevision(models.Model):
     class State(models.TextChoices):
         VALIDATED = "VALIDATED", "Validated"
diff --git a/travel_windows/tests/test_duration_ingestion.py b/travel_windows/tests/test_duration_ingestion.py
new file mode 100644
index 0000000..07c6e96
--- /dev/null
+++ b/travel_windows/tests/test_duration_ingestion.py
@@ -0,0 +1,123 @@
+import json
+
+from django.test import TransactionTestCase
+
+from sources.management.commands.register_approved_sources import (
+    register_approved_sources,
+)
+from sources.models import ParseRun, SourceArtifact
+from sources.tests.test_transport import FakeConnection, FakeResponse
+from sources.transport import fetch_route_duration_reference
+from travel_windows.contracts import DURATION_SOURCE_LOCATOR
+from travel_windows.ingestion import (
+    DurableAviationAttemptRecorder,
+    FlightIngestionCode,
+    ingest_route_duration_reference,
+)
+from travel_windows.models import (
+    CURATED_AIRPORT_ROWS,
+    RouteDurationDerivation,
+    RouteDurationDerivedRoute,
+)
+from travel_windows.tests.test_duration_reference import _archive, _row
+
+
+def _duration_page():
+    return (
+        '<meta name="title" property="og:title" '
+        'content="해양수산부_항공 스케줄 기본정보_20241212">'
+        '<button onclick="fileDetailObj.fn_fileDataDown('
+        "'15151728', 'uddi:ebe570de-68fb-42d4-ad47-9a5d565d6c5a',"
+        "'','1', '2')\">다운로드</button>"
+    ).encode("utf-8")
+
+
+def _duration_metadata():
+    return json.dumps(
+        {
+            "status": True,
+            "atchFileId": "FILE_000000003520744",
+            "fileDetailSn": "1",
+            "dpk": "uddi:ebe570de-68fb-42d4-ad47-9a5d565d6c5a",
+            "fileDataRegistVO": {
+                "atchFileId": "FILE_000000003520744",
+                "fileDetailSn": "1",
+                "orginlFileNm": "수출입 물류 플랫폼_항공 스케줄 기본정보.zip",
+            },
+            "dataSetFileDetailInfo": {
+                "publicDataPk": "15151728",
+                "publicDataDetailPk": (
+                    "uddi:ebe570de-68fb-42d4-ad47-9a5d565d6c5a"
+                ),
+                "stdrDe": "12/12/2024",
+                "dataNm": "해양수산부_항공 스케줄 기본정보_20241212",
+            },
+        },
+        ensure_ascii=False,
+        separators=(",", ":"),
+    ).encode("utf-8")
+
+
+def _all_route_archive():
+    rows = []
+    for route_index, (_id, iata, *_rest) in enumerate(CURATED_AIRPORT_ROWS):
+        for direction, minutes in (("OUTBOUND", 120), ("INBOUND", 150)):
+            for day in range(5):
+                row = _row(
+                    direction=direction,
+                    day=route_index * 20 + day,
+                    duration=str(minutes + route_index),
+                )
+                if direction == "OUTBOUND":
+                    row[4] = iata
+                else:
+                    row[3] = iata
+                rows.append(row)
+    return _archive(rows)
+
+
+class RouteDurationIngestionTests(TransactionTestCase):
+    def test_official_archive_receipt_derives_and_seals_all_curated_routes(self):
+        register_approved_sources(apply=True)
+        archive = _all_route_archive()
+        connections = [
+            FakeConnection(FakeResponse(200, _duration_page())),
+            FakeConnection(FakeResponse(200, _duration_metadata())),
+            FakeConnection(FakeResponse(200, b'{"needCaptcha":false}')),
+            FakeConnection(
+                FakeResponse(
+                    200,
+                    archive,
+                    content_type="application/octet-stream",
+                    content_disposition='attachment; filename="duration.zip"',
+                )
+            ),
+        ]
+        recorder = DurableAviationAttemptRecorder()
+        fetched = fetch_route_duration_reference(
+            official_locator=DURATION_SOURCE_LOCATOR,
+            connect_timeout_seconds=4,
+            read_timeout_seconds=12,
+            connection_factory=lambda *_args: connections.pop(0),
+            attempt_executor=recorder,
+        )
+
+        outcome = ingest_route_duration_reference(
+            fetch=fetched,
+            attempt_recorder=recorder,
+        )
+
+        self.assertEqual(outcome.code, FlightIngestionCode.STAGED)
+        derivation = RouteDurationDerivation.objects.get(pk=outcome.derivation_id)
+        self.assertEqual(derivation.route_count, 13)
+        self.assertEqual(derivation.source_row_count, 130)
+        self.assertEqual(RouteDurationDerivedRoute.objects.count(), 13)
+        self.assertEqual(
+            derivation.parse_run.outcome,
+            ParseRun.Outcome.VALIDATED,
+        )
+        self.assertEqual(
+            derivation.parse_run.artifact.state,
+            SourceArtifact.State.REVIEW_REQUIRED,
+        )
+        self.assertNotIn(archive[:64], repr(derivation).encode("utf-8"))


