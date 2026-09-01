## `feat(source): bind flight publication to collection receipts`

diff --git a/travel_windows/ingestion.py b/travel_windows/ingestion.py
index 9ca77aa..ebc41f4 100644
--- a/travel_windows/ingestion.py
+++ b/travel_windows/ingestion.py
@@ -1,16 +1,19 @@
-"""Offline ingestion bridge from documented source pages to publication.
+"""Durable offline collection and staging for scheduled-flight evidence.
 
-Raw response bytes are accepted in memory and are never written to a model,
-log, audit row, or exception.  Only bounded receipt hashes and typed revisions
-survive the call.
+The HTTP transport owns response bytes only in memory. This module wraps each
+actual call with a durable receipt, links every successful response hash to an
+immutable artifact, and accepts only those exact in-process receipts for typed
+parsing. Raw response bytes are never written to a model, log, or exception.
 """
 
 from __future__ import annotations
 
 import hashlib
+import re
 import uuid
-from dataclasses import dataclass
+from dataclasses import dataclass, field
 from datetime import date, datetime
+from typing import Callable
 
 from django.db import transaction
 from django.utils import timezone
@@ -18,29 +21,51 @@ from django.utils import timezone
 from sources.models import (
     FetchAttempt,
     ParseRun,
+    ParseRunInput,
     SourceArtifact,
     SourceConfiguration,
     SourceRightsDecision,
+    aviation_parse_input_identity,
+)
+from sources.transport import (
+    ATTEMPT_RETRYABLE_FAILED,
+    ATTEMPT_SUCCEEDED,
+    ATTEMPT_TERMINAL_FAILED,
+    ROLE_DESTINATION_CITY,
+    ROLE_LEGACY_ARRIVAL,
+    ROLE_SCHEDULE_ARRIVAL,
+    ROLE_SCHEDULE_DEPARTURE,
+    AviationReferenceFetchResult,
+    AviationResponseLineage,
+    RequestFingerprint,
+    SchedulePageFetchResult,
+    SingleAttemptResult,
 )
 
 from .contracts import (
     CITY_CONTRACT_FINGERPRINT_SHA256,
+    CITY_FIELD_SCOPE,
     CITY_SCHEMA_FINGERPRINT_SHA256,
     CITY_SOURCE_CODE,
-    CITY_SOURCE_LOCATOR,
+    CITY_SOURCE_REVISION,
+    DATA_GO_SECRET_REFERENCE,
     DURATION_CONTRACT_FINGERPRINT_SHA256,
+    DURATION_FIELD_SCOPE,
     DURATION_SCHEMA_FINGERPRINT_SHA256,
     DURATION_SOURCE_CODE,
-    LEGACY_SCHEDULE_ARRIVALS_LOCATOR,
+    DURATION_SOURCE_REVISION,
     LEGACY_SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
+    LEGACY_SCHEDULE_FIELD_SCOPE,
     LEGACY_SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
     LEGACY_SCHEDULE_SOURCE_CODE,
-    SCHEDULE_ARRIVALS_LOCATOR,
+    LEGACY_SCHEDULE_SOURCE_REVISION,
     SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
-    SCHEDULE_DEPARTURES_LOCATOR,
+    SCHEDULE_FIELD_SCOPE,
     SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
     SCHEDULE_SOURCE_CODE,
+    SCHEDULE_SOURCE_REVISION,
 )
+from .models import ReviewedDurationReceipt
 from .parser import (
     AviationParseFailure,
     CityReferenceParseSuccess,
@@ -78,30 +103,63 @@ class _IngestionRejected(Exception):
         super().__init__(code)
 
 
-def _digest(parts: tuple[bytes, ...]) -> tuple[str, int]:
-    digest = hashlib.sha256()
-    byte_count = 0
-    for part in parts:
-        if not isinstance(part, bytes):
-            raise _IngestionRejected(FlightIngestionCode.PARSE_QUARANTINED)
-        digest.update(len(part).to_bytes(8, "big"))
-        digest.update(part)
-        byte_count += len(part)
-    return digest.hexdigest(), byte_count
+@dataclass(frozen=True, slots=True)
+class _SourceContract:
+    revision: str
+    contract_fingerprint: str
+    field_scope: str
+    secret_reference_name: str
 
 
-def _request_hash(value: str) -> str:
-    return hashlib.sha256(value.encode("utf-8")).hexdigest()
+_SOURCE_CONTRACTS = {
+    SCHEDULE_SOURCE_CODE: _SourceContract(
+        SCHEDULE_SOURCE_REVISION,
+        SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
+        SCHEDULE_FIELD_SCOPE,
+        DATA_GO_SECRET_REFERENCE,
+    ),
+    CITY_SOURCE_CODE: _SourceContract(
+        CITY_SOURCE_REVISION,
+        CITY_CONTRACT_FINGERPRINT_SHA256,
+        CITY_FIELD_SCOPE,
+        DATA_GO_SECRET_REFERENCE,
+    ),
+    LEGACY_SCHEDULE_SOURCE_CODE: _SourceContract(
+        LEGACY_SCHEDULE_SOURCE_REVISION,
+        LEGACY_SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
+        LEGACY_SCHEDULE_FIELD_SCOPE,
+        DATA_GO_SECRET_REFERENCE,
+    ),
+    DURATION_SOURCE_CODE: _SourceContract(
+        DURATION_SOURCE_REVISION,
+        DURATION_CONTRACT_FINGERPRINT_SHA256,
+        DURATION_FIELD_SCOPE,
+        "",
+    ),
+}
+_REMOTE_SOURCE_CODES = frozenset(
+    (SCHEDULE_SOURCE_CODE, CITY_SOURCE_CODE, LEGACY_SCHEDULE_SOURCE_CODE)
+)
+_SHA256_HEX = re.compile(r"^[0-9a-f]{64}$")
+_REVIEWER = re.compile(r"^[A-Z][A-Z0-9_.:-]{2,99}$")
+_REVIEW_BASIS = re.compile(r"^[A-Z][A-Z0-9_]{2,99}$")
 
 
-def _locked_source(code: str) -> tuple[SourceConfiguration, SourceRightsDecision]:
+def _locked_source(
+    code: str,
+) -> tuple[SourceConfiguration, SourceRightsDecision]:
+    contract = _SOURCE_CONTRACTS.get(code)
+    if contract is None:
+        raise _IngestionRejected(FlightIngestionCode.SOURCE_GATE_FAILED)
     source = (
         SourceConfiguration.objects.select_for_update()
         .filter(
             code=code,
+            revision=contract.revision,
             module=SourceConfiguration.Module.AVIATION,
             state=SourceConfiguration.State.ACTIVE,
             enabled=True,
+            secret_reference_name=contract.secret_reference_name,
         )
         .first()
     )
@@ -119,193 +177,732 @@ def _locked_source(code: str) -> tuple[SourceConfiguration, SourceRightsDecision
         or not rights.access_allowed
         or not rights.automated_collection_allowed
         or not rights.typed_field_storage_allowed
-        or not rights.typed_republication_allowed
         or rights.raw_body_storage_allowed
+        or not rights.typed_republication_allowed
+        or rights.raw_retention_seconds != 0
+        or rights.typed_retention
+        != SourceRightsDecision.Retention.PRODUCT_HISTORY
+        or rights.evidence_retention
+        != SourceRightsDecision.Retention.PRODUCT_HISTORY
+        or rights.field_scope_code != contract.field_scope
+        or rights.contract_fingerprint_sha256
+        != contract.contract_fingerprint
     ):
         raise _IngestionRejected(FlightIngestionCode.SOURCE_GATE_FAILED)
     return source, rights
 
 
-def _record_validated_parse(
+@dataclass(frozen=True, slots=True)
+class _DurableAttemptReceipt:
+    result: SingleAttemptResult = field(repr=False)
+    source_code: str
+    attempt_id: uuid.UUID
+    artifact_id: uuid.UUID | None
+
+
+def _valid_attempt_result(
+    result: object,
+    request_fingerprint: RequestFingerprint,
+) -> bool:
+    if (
+        type(result) is not SingleAttemptResult
+        or result.request_fingerprint != request_fingerprint
+        or result.attempt_result
+        not in {
+            ATTEMPT_SUCCEEDED,
+            ATTEMPT_RETRYABLE_FAILED,
+            ATTEMPT_TERMINAL_FAILED,
+        }
+        or result.provider_code
+        not in {choice for choice, _label in FetchAttempt.ProviderCode.choices}
+        | {""}
+    ):
+        return False
+    if result.attempt_result == ATTEMPT_SUCCEEDED:
+        return (
+            isinstance(result.body, bytes)
+            and result.byte_count == len(result.body)
+            and result.http_status is not None
+            and 200 <= result.http_status <= 299
+            and result.failure_code == ""
+            and _SHA256_HEX.fullmatch(result.response_sha256) is not None
+            and hashlib.sha256(result.body).hexdigest()
+            == result.response_sha256
+        )
+    allowed_failures = {
+        choice for choice, _label in FetchAttempt.FailureCode.choices
+    }
+    return (
+        result.body == b""
+        and result.byte_count == 0
+        and result.response_sha256 == ""
+        and result.failure_code in allowed_failures
+    )
+
+
+class DurableAviationAttemptRecorder:
+    """Transport callback that durably wraps exactly one actual HTTP call."""
+
+    def __init__(self) -> None:
+        self._receipts: dict[int, _DurableAttemptReceipt] = {}
+
+    def _close_interrupted(self, attempt_id: uuid.UUID) -> None:
+        try:
+            with transaction.atomic(durable=True):
+                attempt = FetchAttempt.objects.select_for_update().get(
+                    pk=attempt_id
+                )
+                if attempt.result != FetchAttempt.Result.STARTED:
+                    return
+                FetchAttempt.objects.filter(pk=attempt.pk).update(
+                    result=FetchAttempt.Result.TERMINAL_FAILED,
+                    completed_at=max(timezone.now(), attempt.started_at),
+                    http_status=None,
+                    provider_code="",
+                    response_sha256="",
+                    failure_code=FetchAttempt.FailureCode.WORKER_INTERRUPTED,
+                )
+        except Exception:
+            # The original persistence failure remains fail-closed. No raw
+            # response detail is added to the exception or any log record.
+            return
+
+    def _close(
+        self,
+        *,
+        attempt_id: uuid.UUID,
+        result: SingleAttemptResult,
+    ) -> uuid.UUID | None:
+        with transaction.atomic(durable=True):
+            attempt = (
+                FetchAttempt.objects.select_for_update()
+                .select_related("source")
+                .get(pk=attempt_id)
+            )
+            if attempt.result != FetchAttempt.Result.STARTED:
+                raise _IngestionRejected(
+                    FlightIngestionCode.PERSISTENCE_FAILED
+                )
+            completed_at = max(timezone.now(), attempt.started_at)
+            FetchAttempt.objects.filter(pk=attempt.pk).update(
+                result=result.attempt_result,
+                completed_at=completed_at,
+                http_status=result.http_status,
+                provider_code=result.provider_code,
+                response_sha256=result.response_sha256,
+                failure_code=result.failure_code,
+            )
+            if not result.succeeded:
+                return None
+
+            artifact = (
+                SourceArtifact.objects.select_for_update()
+                .filter(
+                    source=attempt.source,
+                    body_sha256=result.response_sha256,
+                )
+                .first()
+            )
+            if artifact is not None:
+                if artifact.byte_count != result.byte_count:
+                    raise _IngestionRejected(
+                        FlightIngestionCode.PERSISTENCE_FAILED
+                    )
+                return artifact.pk
+            canonical_attempt = (
+                FetchAttempt.objects.filter(
+                    source=attempt.source,
+                    result=FetchAttempt.Result.SUCCEEDED,
+                    response_sha256=result.response_sha256,
+                )
+                .order_by("completed_at", "started_at", "id")
+                .first()
+            )
+            if canonical_attempt is None:
+                raise _IngestionRejected(
+                    FlightIngestionCode.PERSISTENCE_FAILED
+                )
+            artifact = SourceArtifact.objects.create(
+                source=attempt.source,
+                body_sha256=result.response_sha256,
+                byte_count=result.byte_count,
+                first_successful_attempt=canonical_attempt,
+            )
+            return artifact.pk
+
+    def __call__(
+        self,
+        *,
+        source_code: str,
+        request_fingerprint: RequestFingerprint,
+        call_once: Callable[[], SingleAttemptResult],
+    ) -> SingleAttemptResult:
+        if (
+            source_code not in _REMOTE_SOURCE_CODES
+            or type(request_fingerprint) is not RequestFingerprint
+            or not callable(call_once)
+        ):
+            raise _IngestionRejected(FlightIngestionCode.SOURCE_GATE_FAILED)
+        with transaction.atomic(durable=True):
+            source, rights = _locked_source(source_code)
+            attempt = FetchAttempt.objects.create(
+                source=source,
+                source_revision=source.revision,
+                rights_decision=rights,
+                operation_id=uuid.uuid4(),
+                attempt_number=1,
+                request_fingerprint_revision=request_fingerprint.revision,
+                normalized_request_sha256=(
+                    request_fingerprint.normalized_request_sha256
+                ),
+            )
+        try:
+            result = call_once()
+            if not _valid_attempt_result(result, request_fingerprint):
+                raise _IngestionRejected(
+                    FlightIngestionCode.PERSISTENCE_FAILED
+                )
+            artifact_id = self._close(attempt_id=attempt.pk, result=result)
+        except Exception:
+            self._close_interrupted(attempt.pk)
+            raise
+        self._receipts[id(result)] = _DurableAttemptReceipt(
+            result=result,
+            source_code=source_code,
+            attempt_id=attempt.pk,
+            artifact_id=artifact_id,
+        )
+        return result
+
+    def receipt_for(
+        self,
+        *,
+        source_code: str,
+        result: SingleAttemptResult,
+    ) -> _DurableAttemptReceipt:
+        receipt = self._receipts.get(id(result))
+        if (
+            receipt is None
+            or receipt.result is not result
+            or receipt.source_code != source_code
+        ):
+            raise _IngestionRejected(
+                FlightIngestionCode.PERSISTENCE_FAILED
+            )
+        return receipt
+
+
+@dataclass(frozen=True, slots=True)
+class _CollectedInput:
+    artifact: SourceArtifact
+    role: str
+    role_sequence: int
+    body: bytes = field(repr=False)
+
+
+def _collected_input(
     *,
+    recorder: DurableAviationAttemptRecorder,
+    lineage: AviationResponseLineage,
     source_code: str,
-    payload_parts: tuple[bytes, ...],
-    request_fingerprint_revision: str,
-    normalized_request_sha256: str,
-    provider_code: str,
+    body: bytes,
+) -> _CollectedInput:
+    if (
+        type(lineage) is not AviationResponseLineage
+        or lineage.source_code != source_code
+        or not lineage.succeeded
+        or lineage.attempt.body != body
+    ):
+        raise _IngestionRejected(FlightIngestionCode.PERSISTENCE_FAILED)
+    receipt = recorder.receipt_for(
+        source_code=source_code,
+        result=lineage.attempt,
+    )
+    if receipt.artifact_id is None:
+        raise _IngestionRejected(FlightIngestionCode.PERSISTENCE_FAILED)
+    try:
+        attempt = FetchAttempt.objects.select_related("source").get(
+            pk=receipt.attempt_id
+        )
+        artifact = SourceArtifact.objects.select_related(
+            "source", "first_successful_attempt"
+        ).get(pk=receipt.artifact_id)
+    except (FetchAttempt.DoesNotExist, SourceArtifact.DoesNotExist):
+        raise _IngestionRejected(
+            FlightIngestionCode.PERSISTENCE_FAILED
+        ) from None
+    if (
+        attempt.source.code != source_code
+        or attempt.result != FetchAttempt.Result.SUCCEEDED
+        or attempt.response_sha256 != hashlib.sha256(body).hexdigest()
+        or artifact.source_id != attempt.source_id
+        or artifact.body_sha256 != attempt.response_sha256
+        or artifact.byte_count != len(body)
+        or artifact.state
+        not in {
+            SourceArtifact.State.RECEIVED,
+            SourceArtifact.State.REVIEW_REQUIRED,
+        }
+    ):
+        raise _IngestionRejected(FlightIngestionCode.PERSISTENCE_FAILED)
+    return _CollectedInput(
+        artifact=artifact,
+        role=lineage.role,
+        role_sequence=lineage.role_sequence,
+        body=body,
+    )
+
+
+def _schedule_inputs(
+    result: SchedulePageFetchResult,
+    recorder: DurableAviationAttemptRecorder,
+) -> tuple[_CollectedInput, ...]:
+    if type(result) is not SchedulePageFetchResult:
+        raise _IngestionRejected(FlightIngestionCode.PERSISTENCE_FAILED)
+    lineages = tuple(result.lineage)
+    if not result.succeeded or result.failure_code or not lineages:
+        raise _IngestionRejected(FlightIngestionCode.SOURCE_GATE_FAILED)
+    if any(
+        not lineage.succeeded
+        or lineage.source_code != SCHEDULE_SOURCE_CODE
+        or lineage.role
+        not in {ROLE_SCHEDULE_DEPARTURE, ROLE_SCHEDULE_ARRIVAL}
+        for lineage in lineages
+    ):
+        raise _IngestionRejected(FlightIngestionCode.PERSISTENCE_FAILED)
+    departures = tuple(
+        lineage
+        for lineage in lineages
+        if lineage.role == ROLE_SCHEDULE_DEPARTURE
+    )
+    arrivals = tuple(
+        lineage
+        for lineage in lineages
+        if lineage.role == ROLE_SCHEDULE_ARRIVAL
+    )
+    if (
+        not departures
+        or not arrivals
+        or tuple(row.role_sequence for row in departures)
+        != tuple(range(1, len(departures) + 1))
+        or tuple(row.role_sequence for row in arrivals)
+        != tuple(range(1, len(arrivals) + 1))
+        or len(departures) != len(result.departure_pages)
+        or len(arrivals) != len(result.arrival_pages)
+    ):
+        raise _IngestionRejected(FlightIngestionCode.PERSISTENCE_FAILED)
+    collected = []
+    pairs = (
+        *zip(departures, result.departure_pages, strict=True),
+        *zip(arrivals, result.arrival_pages, strict=True),
+    )
+    for lineage, body in pairs:
+        collected.append(
+            _collected_input(
+                recorder=recorder,
+                lineage=lineage,
+                source_code=SCHEDULE_SOURCE_CODE,
+                body=body,
+            )
+        )
+    return tuple(collected)
+
+
+def _reference_inputs(
+    result: AviationReferenceFetchResult,
+    recorder: DurableAviationAttemptRecorder,
+) -> tuple[_CollectedInput, tuple[_CollectedInput, ...]]:
+    if type(result) is not AviationReferenceFetchResult:
+        raise _IngestionRejected(FlightIngestionCode.PERSISTENCE_FAILED)
+    lineages = tuple(result.lineage)
+    if not result.succeeded or result.failure_code or len(lineages) < 2:
+        raise _IngestionRejected(FlightIngestionCode.SOURCE_GATE_FAILED)
+    city_lineage = lineages[0]
+    legacy_lineages = lineages[1:]
+    if (
+        city_lineage.source_code != CITY_SOURCE_CODE
+        or city_lineage.role != ROLE_DESTINATION_CITY
+        or city_lineage.role_sequence != 1
+        or not city_lineage.succeeded
+        or any(
+            lineage.source_code != LEGACY_SCHEDULE_SOURCE_CODE
+            or lineage.role != ROLE_LEGACY_ARRIVAL
+            or lineage.role_sequence != index
+            or not lineage.succeeded
+            for index, lineage in enumerate(legacy_lineages, 1)
+        )
+        or len(legacy_lineages) != len(result.legacy_arrival_pages)
+    ):
+        raise _IngestionRejected(FlightIngestionCode.PERSISTENCE_FAILED)
+    city = _collected_input(
+        recorder=recorder,
+        lineage=city_lineage,
+        source_code=CITY_SOURCE_CODE,
+        body=result.city_payload,
+    )
+    legacy = tuple(
+        _collected_input(
+            recorder=recorder,
+            lineage=lineage,
+            source_code=LEGACY_SCHEDULE_SOURCE_CODE,
+            body=body,
+        )
+        for lineage, body in zip(
+            legacy_lineages,
+            result.legacy_arrival_pages,
+            strict=True,
+        )
+    )
+    return city, legacy
+
+
+def _parse_failure_shape(
+    result: AviationParseFailure,
+    *,
+    expected_schema: str,
+) -> tuple[str, str, str]:
+    observed = result.observed_schema_fingerprint_sha256
+    if result.failure_code == "SCHEMA_MISMATCH":
+        if (
+            _SHA256_HEX.fullmatch(observed or "") is None
+            or observed == expected_schema
+        ):
+            observed = "f" * 64 if expected_schema != "f" * 64 else "e" * 64
+        return ParseRun.Outcome.QUARANTINED, "SCHEMA_MISMATCH", observed
+    if result.failure_code == "INCOMPLETE_COVERAGE":
+        return (
+            ParseRun.Outcome.QUARANTINED,
+            "REQUIRED_VALUE_MISSING",
+            expected_schema,
+        )
+    if result.failure_code == "INVALID_SIZE":
+        return ParseRun.Outcome.QUARANTINED, "SYNTAX_ERROR", ""
+    if result.failure_code in {
+        "IDENTITY_MISMATCH",
+        "REQUIRED_VALUE_MISSING",
+        "DUPLICATE_RECORD",
+        "INVALID_VALUE",
+        "CONFLICTING_VALUE",
+    }:
+        return ParseRun.Outcome.QUARANTINED, result.failure_code, expected_schema
+    if result.failure_code in {"ENCODING_ERROR", "SYNTAX_ERROR"}:
+        return ParseRun.Outcome.QUARANTINED, result.failure_code, ""
+    return ParseRun.Outcome.FAILED, "INTERNAL_PARSER_ERROR", ""
+
+
+def _persist_parse_run(
+    *,
+    inputs: tuple[_CollectedInput, ...],
     parser_name: str,
     contract_fingerprint: str,
     schema_fingerprint: str,
+    parsed: object,
     completed_at: datetime,
 ) -> ParseRun:
-    body_sha256, byte_count = _digest(payload_parts)
-    with transaction.atomic(durable=True):
-        source, rights = _locked_source(source_code)
-        operation_id = uuid.uuid4()
-        attempt = FetchAttempt.objects.create(
-            source=source,
-            source_revision=source.revision,
-            rights_decision=rights,
-            operation_id=operation_id,
-            attempt_number=1,
-            request_fingerprint_revision=request_fingerprint_revision,
-            normalized_request_sha256=normalized_request_sha256,
-        )
-        closed_at = max(completed_at, attempt.started_at)
-        FetchAttempt.objects.filter(pk=attempt.pk).update(
-            result=FetchAttempt.Result.SUCCEEDED,
-            completed_at=closed_at,
-            http_status=200,
-            provider_code=provider_code,
-            response_sha256=body_sha256,
-            failure_code="",
+    if not inputs:
+        raise _IngestionRejected(FlightIngestionCode.PERSISTENCE_FAILED)
+    identities = tuple(
+        (
+            ordinal,
+            row.role,
+            row.role_sequence,
+            row.artifact.body_sha256,
         )
-        attempt.refresh_from_db()
-        artifact = SourceArtifact.objects.filter(
-            source=source,
-            body_sha256=body_sha256,
-        ).first()
-        if artifact is None:
-            artifact = SourceArtifact.objects.create(
-                source=source,
-                body_sha256=body_sha256,
-                byte_count=byte_count,
-                first_successful_attempt=attempt,
+        for ordinal, row in enumerate(inputs, 1)
+    )
+    input_identity = aviation_parse_input_identity(identities)
+    with transaction.atomic(durable=True):
+        artifacts = {
+            artifact.pk: artifact
+            for artifact in SourceArtifact.objects.select_for_update().filter(
+                pk__in={row.artifact.pk for row in inputs}
             )
-            parse_run = ParseRun.objects.create(
-                artifact=artifact,
+        }
+        if len(artifacts) != len({row.artifact.pk for row in inputs}) or any(
+            artifacts[row.artifact.pk].body_sha256
+            != row.artifact.body_sha256
+            or artifacts[row.artifact.pk].state
+            not in {
+                SourceArtifact.State.RECEIVED,
+                SourceArtifact.State.REVIEW_REQUIRED,
+            }
+            for row in inputs
+        ):
+            raise _IngestionRejected(FlightIngestionCode.PERSISTENCE_FAILED)
+        run = (
+            ParseRun.objects.select_for_update()
+            .filter(
+                artifact=artifacts[inputs[0].artifact.pk],
                 parser_name=parser_name,
                 parser_version=ParseRun.ParserVersion.V1,
-                parser_contract_fingerprint_sha256=contract_fingerprint,
-                expected_schema_fingerprint_sha256=schema_fingerprint,
-            )
-            parse_completed_at = max(closed_at, parse_run.started_at)
-            ParseRun.objects.filter(pk=parse_run.pk).update(
-                completed_at=parse_completed_at,
-                outcome=ParseRun.Outcome.VALIDATED,
-                failure_code="",
-                observed_schema_fingerprint_sha256=schema_fingerprint,
+                input_identity_sha256=input_identity,
             )
-            SourceArtifact.objects.filter(pk=artifact.pk).update(
-                state=SourceArtifact.State.REVIEW_REQUIRED
-            )
-            parse_run.refresh_from_db()
-            return parse_run
-        if artifact.byte_count != byte_count:
-            raise _IngestionRejected(FlightIngestionCode.PERSISTENCE_FAILED)
-        try:
-            parse_run = artifact.parse_runs.get(
+            .first()
+        )
+        if run is None:
+            run = ParseRun.objects.create(
+                artifact=artifacts[inputs[0].artifact.pk],
                 parser_name=parser_name,
                 parser_version=ParseRun.ParserVersion.V1,
                 parser_contract_fingerprint_sha256=contract_fingerprint,
                 expected_schema_fingerprint_sha256=schema_fingerprint,
-                observed_schema_fingerprint_sha256=schema_fingerprint,
-                outcome=ParseRun.Outcome.VALIDATED,
+                input_identity_sha256=input_identity,
+            )
+            for ordinal, row in enumerate(inputs, 1):
+                ParseRunInput.objects.create(
+                    parse_run=run,
+                    artifact=artifacts[row.artifact.pk],
+                    ordinal=ordinal,
+                    role=row.role,
+                    role_sequence=row.role_sequence,
+                )
+        else:
+            observed_inputs = tuple(
+                (
+                    row.ordinal,
+                    row.role,
+                    row.role_sequence,
+                    row.artifact.body_sha256,
+                )
+                for row in run.ordered_inputs.select_related("artifact").order_by(
+                    "ordinal"
+                )
+            )
+            if (
+                run.parser_contract_fingerprint_sha256
+                != contract_fingerprint
+                or run.expected_schema_fingerprint_sha256
+                != schema_fingerprint
+                or observed_inputs != identities
+            ):
+                raise _IngestionRejected(
+                    FlightIngestionCode.PERSISTENCE_FAILED
+                )
+
+    if isinstance(parsed, AviationParseFailure):
+        outcome, failure_code, observed_schema = _parse_failure_shape(
+            parsed,
+            expected_schema=schema_fingerprint,
+        )
+    else:
+        observed_schema = getattr(
+            parsed, "observed_schema_fingerprint_sha256", ""
+        )
+        if observed_schema != schema_fingerprint:
+            outcome, failure_code, observed_schema = (
+                ParseRun.Outcome.FAILED,
+                ParseRun.FailureCode.INTERNAL_PARSER_ERROR,
+                "",
             )
-        except ParseRun.DoesNotExist:
-            raise _IngestionRejected(FlightIngestionCode.PERSISTENCE_FAILED) from None
-        return parse_run
+        else:
+            outcome, failure_code = ParseRun.Outcome.VALIDATED, ""
+
+    with transaction.atomic(durable=True):
+        run = ParseRun.objects.select_for_update().get(pk=run.pk)
+        if run.outcome == ParseRun.Outcome.STARTED:
+            closed_at = max(timezone.now(), completed_at, run.started_at)
+            ParseRun.objects.filter(pk=run.pk).update(
+                completed_at=closed_at,
+                outcome=outcome,
+                failure_code=failure_code,
+                observed_schema_fingerprint_sha256=observed_schema,
+            )
+            target_state = (
+                SourceArtifact.State.REVIEW_REQUIRED
+                if outcome == ParseRun.Outcome.VALIDATED
+                else SourceArtifact.State.REJECTED
+            )
+            for artifact_id in dict.fromkeys(
+                row.artifact.pk for row in inputs
+            ):
+                SourceArtifact.objects.filter(
+                    pk=artifact_id,
+                    state=SourceArtifact.State.RECEIVED,
+                ).update(state=target_state)
+            run.refresh_from_db()
+        expected_outcome = (
+            ParseRun.Outcome.VALIDATED
+            if not isinstance(parsed, AviationParseFailure)
+            and getattr(parsed, "observed_schema_fingerprint_sha256", "")
+            == schema_fingerprint
+            else outcome
+        )
+        if run.outcome != expected_outcome:
+            raise _IngestionRejected(FlightIngestionCode.PERSISTENCE_FAILED)
+        return run
+
+
+def _duration_failure_shape(
+    parsed: AviationParseFailure,
+) -> tuple[str, str]:
+    if parsed.failure_code == "SCHEMA_MISMATCH":
+        observed = parsed.observed_schema_fingerprint_sha256
+        if (
+            _SHA256_HEX.fullmatch(observed or "") is None
+            or observed == DURATION_SCHEMA_FINGERPRINT_SHA256
+        ):
+            observed = "f" * 64
+        return "SCHEMA_MISMATCH", observed
+    if parsed.failure_code == "INCOMPLETE_COVERAGE":
+        return "REQUIRED_VALUE_MISSING", DURATION_SCHEMA_FINGERPRINT_SHA256
+    if parsed.failure_code == "INVALID_SIZE":
+        return "SYNTAX_ERROR", ""
+    return "INVALID_VALUE", DURATION_SCHEMA_FINGERPRINT_SHA256
+
+
+def _record_duration_receipt(
+    *,
+    duration_csv: bytes,
+    parsed: DurationParseSuccess | AviationParseFailure,
+    reviewed_by: str,
+    reviewer_basis_code: str,
+    reviewed_at: datetime,
+) -> ReviewedDurationReceipt:
+    if (
+        not isinstance(duration_csv, bytes)
+        or not duration_csv
+        or _REVIEWER.fullmatch(reviewed_by or "") is None
+        or _REVIEW_BASIS.fullmatch(reviewer_basis_code or "") is None
+        or not isinstance(reviewed_at, datetime)
+        or not timezone.is_aware(reviewed_at)
+    ):
+        raise _IngestionRejected(FlightIngestionCode.PARSE_QUARANTINED)
+    if isinstance(parsed, AviationParseFailure):
+        outcome = ReviewedDurationReceipt.Outcome.QUARANTINED
+        failure_code, observed_schema = _duration_failure_shape(parsed)
+    elif (
+        isinstance(parsed, DurationParseSuccess)
+        and parsed.observed_schema_fingerprint_sha256
+        == DURATION_SCHEMA_FINGERPRINT_SHA256
+    ):
+        outcome = ReviewedDurationReceipt.Outcome.VALIDATED
+        failure_code = ""
+        observed_schema = DURATION_SCHEMA_FINGERPRINT_SHA256
+    else:
+        raise _IngestionRejected(FlightIngestionCode.PARSE_QUARANTINED)
+    with transaction.atomic(durable=True):
+        source, rights = _locked_source(DURATION_SOURCE_CODE)
+        return ReviewedDurationReceipt.objects.create(
+            source=source,
+            source_revision=source.revision,
+            rights_decision=rights,
+            body_sha256=hashlib.sha256(duration_csv).hexdigest(),
+            byte_count=len(duration_csv),
+            parser_contract_fingerprint_sha256=(
+                DURATION_CONTRACT_FINGERPRINT_SHA256
+            ),
+            expected_schema_fingerprint_sha256=(
+                DURATION_SCHEMA_FINGERPRINT_SHA256
+            ),
+            observed_schema_fingerprint_sha256=observed_schema,
+            outcome=outcome,
+            failure_code=failure_code,
+            reviewed_by=reviewed_by,
+            reviewer_basis_code=reviewer_basis_code,
+            reviewed_at=reviewed_at,
+            created_at=max(timezone.now(), reviewed_at),
+        )
 
 
 def ingest_flight_evidence_candidate(
     *,
-    departure_pages: tuple[bytes, ...],
-    arrival_pages: tuple[bytes, ...],
-    city_payload: bytes,
-    legacy_arrival_pages: tuple[bytes, ...],
+    schedule_fetch: SchedulePageFetchResult,
+    reference_fetch: AviationReferenceFetchResult,
+    attempt_recorder: DurableAviationAttemptRecorder,
     duration_csv: bytes,
     source_date: date,
+    duration_reviewed_by: str,
+    duration_reviewer_basis_code: str,
     source_checked_at: datetime | None = None,
 ) -> FlightIngestionOutcome:
-    """Validate source pages and persist a review-required typed candidate."""
+    """Parse exact durable collection receipts and stage a typed candidate.
+
+    The former bytes-only API is intentionally gone: callers must supply the
+    aggregate objects produced with this call's ``attempt_recorder``.
+    """
 
     checked_at = source_checked_at or timezone.now()
-    schedule = adapt_data_go_schedule_pages(
-        departure_pages=departure_pages,
-        arrival_pages=arrival_pages,
-        source_date=source_date,
-    )
-    city_reference = parse_destination_city_reference(city_payload)
-    legacy_arrivals = parse_legacy_arrival_pages(legacy_arrival_pages)
-    durations = parse_route_durations(duration_csv)
-    if any(
-        isinstance(result, AviationParseFailure)
-        for result in (
-            schedule,
-            city_reference,
-            legacy_arrivals,
-            durations,
-        )
-    ):
-        return FlightIngestionOutcome(FlightIngestionCode.PARSE_QUARANTINED)
     if (
-        not isinstance(schedule, ScheduleParseSuccess)
-        or not isinstance(city_reference, CityReferenceParseSuccess)
-        or not isinstance(legacy_arrivals, LegacyArrivalParseSuccess)
-        or not isinstance(durations, DurationParseSuccess)
+        type(attempt_recorder) is not DurableAviationAttemptRecorder
+        or not isinstance(source_date, date)
+        or not isinstance(checked_at, datetime)
+        or not timezone.is_aware(checked_at)
     ):
-        return FlightIngestionOutcome(FlightIngestionCode.PARSE_QUARANTINED)
+        return FlightIngestionOutcome(FlightIngestionCode.PERSISTENCE_FAILED)
     try:
-        schedule_run = _record_validated_parse(
-            source_code=SCHEDULE_SOURCE_CODE,
-            payload_parts=(*departure_pages, *arrival_pages),
-            request_fingerprint_revision="ICN_SCHEDULE_V1",
-            normalized_request_sha256=_request_hash(
-                f"{SCHEDULE_DEPARTURES_LOCATOR}\n{SCHEDULE_ARRIVALS_LOCATOR}\n"
-                "serviceKey=<redacted>&pageNo=1..N&type=json&"
-                f"season={schedule.season}"
-            ),
-            provider_code=FetchAttempt.ProviderCode.DATA_GO_SUCCESS,
+        schedule_inputs = _schedule_inputs(schedule_fetch, attempt_recorder)
+        city_input, legacy_inputs = _reference_inputs(
+            reference_fetch, attempt_recorder
+        )
+        schedule = adapt_data_go_schedule_pages(
+            departure_pages=schedule_fetch.departure_pages,
+            arrival_pages=schedule_fetch.arrival_pages,
+            source_date=source_date,
+        )
+        city_reference = parse_destination_city_reference(
+            reference_fetch.city_payload
+        )
+        legacy_arrivals = parse_legacy_arrival_pages(
+            reference_fetch.legacy_arrival_pages
+        )
+        durations = parse_route_durations(duration_csv)
+
+        schedule_run = _persist_parse_run(
+            inputs=schedule_inputs,
             parser_name=ParseRun.ParserName.ICN_FLIGHT_SCHEDULE_JSON,
             contract_fingerprint=SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
             schema_fingerprint=SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
+            parsed=schedule,
             completed_at=checked_at,
         )
-        city_reference_run = _record_validated_parse(
-            source_code=CITY_SOURCE_CODE,
-            payload_parts=(city_payload,),
-            request_fingerprint_revision="ICN_DESTINATION_CITY_V1",
-            normalized_request_sha256=_request_hash(
-                f"{CITY_SOURCE_LOCATOR}\nserviceKey=<redacted>&type=json"
-            ),
-            provider_code=FetchAttempt.ProviderCode.DATA_GO_SUCCESS,
+        city_reference_run = _persist_parse_run(
+            inputs=(city_input,),
             parser_name=ParseRun.ParserName.ICN_DESTINATION_CITY_JSON,
             contract_fingerprint=CITY_CONTRACT_FINGERPRINT_SHA256,
             schema_fingerprint=CITY_SCHEMA_FINGERPRINT_SHA256,
+            parsed=city_reference,
             completed_at=checked_at,
         )
-        legacy_arrivals_run = _record_validated_parse(
-            source_code=LEGACY_SCHEDULE_SOURCE_CODE,
-            payload_parts=legacy_arrival_pages,
-            request_fingerprint_revision="ICN_LEGACY_ARRIVALS_V1",
-            normalized_request_sha256=_request_hash(
-                f"{LEGACY_SCHEDULE_ARRIVALS_LOCATOR}\n"
-                "serviceKey=<redacted>&pageNo=1..N&numOfRows=100&"
-                "lang=K&type=json"
-            ),
-            provider_code=FetchAttempt.ProviderCode.DATA_GO_SUCCESS,
+        legacy_arrivals_run = _persist_parse_run(
+            inputs=legacy_inputs,
             parser_name=ParseRun.ParserName.ICN_LEGACY_ARRIVALS_JSON,
             contract_fingerprint=(
                 LEGACY_SCHEDULE_CONTRACT_FINGERPRINT_SHA256
             ),
-            schema_fingerprint=(
-                LEGACY_SCHEDULE_SCHEMA_FINGERPRINT_SHA256
-            ),
+            schema_fingerprint=LEGACY_SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
+            parsed=legacy_arrivals,
             completed_at=checked_at,
         )
-        duration_run = _record_validated_parse(
-            source_code=DURATION_SOURCE_CODE,
-            payload_parts=(duration_csv,),
-            request_fingerprint_revision="ROUTE_DURATION_V1",
-            normalized_request_sha256=_request_hash(
-                "reviewed-route-duration-csv-v1"
-            ),
-            provider_code="",
-            parser_name=ParseRun.ParserName.ROUTE_DURATION_CSV,
-            contract_fingerprint=DURATION_CONTRACT_FINGERPRINT_SHA256,
-            schema_fingerprint=DURATION_SCHEMA_FINGERPRINT_SHA256,
-            completed_at=checked_at,
+        duration_receipt = _record_duration_receipt(
+            duration_csv=duration_csv,
+            parsed=durations,
+            reviewed_by=duration_reviewed_by,
+            reviewer_basis_code=duration_reviewer_basis_code,
+            reviewed_at=checked_at,
         )
+        if any(
+            isinstance(result, AviationParseFailure)
+            for result in (
+                schedule,
+                city_reference,
+                legacy_arrivals,
+                durations,
+            )
+        ):
+            return FlightIngestionOutcome(
+                FlightIngestionCode.PARSE_QUARANTINED
+            )
+        if (
+            not isinstance(schedule, ScheduleParseSuccess)
+            or not isinstance(city_reference, CityReferenceParseSuccess)
+            or not isinstance(legacy_arrivals, LegacyArrivalParseSuccess)
+            or not isinstance(durations, DurationParseSuccess)
+            or duration_receipt.outcome
+            != ReviewedDurationReceipt.Outcome.VALIDATED
+        ):
+            return FlightIngestionOutcome(
+                FlightIngestionCode.PARSE_QUARANTINED
+            )
         outcome = stage_flight_evidence(
             schedule_run=schedule_run,
             schedule=schedule,
@@ -313,7 +910,7 @@ def ingest_flight_evidence_candidate(
             city_reference=city_reference,
             legacy_arrivals_run=legacy_arrivals_run,
             legacy_arrivals=legacy_arrivals,
-            duration_run=duration_run,
+            duration_receipt=duration_receipt,
             durations=durations,
             source_checked_at=checked_at,
         )
diff --git a/travel_windows/management/commands/publish_scheduled_flights.py b/travel_windows/management/commands/publish_scheduled_flights.py
index df8cecb..bc97efe 100644
--- a/travel_windows/management/commands/publish_scheduled_flights.py
+++ b/travel_windows/management/commands/publish_scheduled_flights.py
@@ -1,3 +1,4 @@
+import re
 from datetime import date
 from pathlib import Path
 
@@ -12,10 +13,14 @@ from sources.transport import (
 from travel_windows.contracts import (
     CITY_SOURCE_CODE,
     DATA_GO_SECRET_REFERENCE,
+    DURATION_SOURCE_CODE,
     LEGACY_SCHEDULE_SOURCE_CODE,
     SCHEDULE_SOURCE_CODE,
 )
-from travel_windows.ingestion import ingest_flight_evidence_candidate
+from travel_windows.ingestion import (
+    DurableAviationAttemptRecorder,
+    ingest_flight_evidence_candidate,
+)
 
 
 class Command(BaseCommand):
@@ -29,10 +34,27 @@ class Command(BaseCommand):
         parser.add_argument("--duration-csv", required=True)
         parser.add_argument("--source-date", required=True)
         parser.add_argument("--season", required=True)
+        parser.add_argument("--duration-reviewed-by", required=True)
+        parser.add_argument("--duration-review-basis", required=True)
 
     def handle(self, *args, **options):
         try:
             source_date = date.fromisoformat(options["source_date"])
+            duration_reviewed_by = options["duration_reviewed_by"]
+            duration_review_basis = options["duration_review_basis"]
+            if (
+                re.fullmatch(
+                    r"[A-Z][A-Z0-9_.:-]{2,99}",
+                    duration_reviewed_by,
+                )
+                is None
+                or re.fullmatch(
+                    r"[A-Z][A-Z0-9_]{2,99}",
+                    duration_review_basis,
+                )
+                is None
+            ):
+                raise ValueError
             duration_csv = Path(options["duration_csv"]).read_bytes()
             source = SourceConfiguration.objects.get(
                 code=SCHEDULE_SOURCE_CODE,
@@ -53,6 +75,12 @@ class Command(BaseCommand):
             )
             if len(reference_sources) != 2:
                 raise SourceConfiguration.DoesNotExist
+            SourceConfiguration.objects.get(
+                code=DURATION_SOURCE_CODE,
+                state=SourceConfiguration.State.ACTIVE,
+                enabled=True,
+                secret_reference_name="",
+            )
         except (OSError, TypeError, ValueError):
             raise CommandError("result=blocked code=INVALID_INPUT") from None
         except SourceConfiguration.DoesNotExist:
@@ -61,12 +89,14 @@ class Command(BaseCommand):
         secret_value = load_aviation_secret_reference()
         if secret_value is None:
             raise CommandError("result=blocked code=SECRET_UNAVAILABLE")
+        attempt_recorder = DurableAviationAttemptRecorder()
         fetched = fetch_data_go_schedule_pages(
             secret_reference_name=DATA_GO_SECRET_REFERENCE,
             secret_value=secret_value,
             season=options["season"],
             connect_timeout_seconds=source.connect_timeout_seconds,
             read_timeout_seconds=source.read_timeout_seconds,
+            attempt_executor=attempt_recorder,
         )
         if not fetched.succeeded:
             raise CommandError(f"result=blocked code={fetched.failure_code}")
@@ -79,6 +109,7 @@ class Command(BaseCommand):
             read_timeout_seconds=min(
                 row.read_timeout_seconds for row in reference_sources
             ),
+            attempt_executor=attempt_recorder,
         )
         if not reference_evidence.succeeded:
             raise CommandError(
@@ -87,14 +118,13 @@ class Command(BaseCommand):
             )
 
         outcome = ingest_flight_evidence_candidate(
-            departure_pages=fetched.departure_pages,
-            arrival_pages=fetched.arrival_pages,
-            city_payload=reference_evidence.city_payload,
-            legacy_arrival_pages=(
-                reference_evidence.legacy_arrival_pages
-            ),
+            schedule_fetch=fetched,
+            reference_fetch=reference_evidence,
+            attempt_recorder=attempt_recorder,
             duration_csv=duration_csv,
             source_date=source_date,
+            duration_reviewed_by=duration_reviewed_by,
+            duration_reviewer_basis_code=duration_review_basis,
         )
         if not outcome.succeeded:
             raise CommandError(f"result=blocked code={outcome.code}")
diff --git a/travel_windows/migrations/0009_aviation_collection_receipts.py b/travel_windows/migrations/0009_aviation_collection_receipts.py
new file mode 100644
index 0000000..29fee48
--- /dev/null
+++ b/travel_windows/migrations/0009_aviation_collection_receipts.py
@@ -0,0 +1,280 @@
+# Generated by Django 5.2.17 on 2026-08-31 04:13
+
+import django.db.models.deletion
+import django.utils.timezone
+import uuid
+from django.db import migrations, models
+
+
+AVIATION_COLLECTION_RECEIPT_SQL = r"""
+CREATE FUNCTION travel_windows_guard_duration_receipt() RETURNS trigger
+LANGUAGE plpgsql AS $$
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
+
+    SELECT source.code,
+           source.revision,
+           source.module,
+           source.state,
+           source.enabled,
+           source.official_locator,
+           source.secret_reference_name,
+           rights.source_id,
+           rights.source_revision,
+           rights.decision,
+           rights.access_allowed,
+           rights.automated_collection_allowed,
+           rights.typed_field_storage_allowed,
+           rights.raw_body_storage_allowed,
+           rights.typed_republication_allowed,
+           rights.raw_retention_seconds,
+           rights.typed_retention,
+           rights.evidence_retention,
+           rights.field_scope_code,
+           rights.contract_fingerprint_sha256,
+           rights.decided_at
+      INTO source_code,
+           locked_source_revision,
+           source_module,
+           source_state,
+           source_enabled,
+           source_locator,
+           source_secret_reference,
+           rights_source_id,
+           rights_source_revision,
+           rights_decision,
+           rights_access_allowed,
+           rights_collection_allowed,
+           rights_storage_allowed,
+           rights_raw_allowed,
+           rights_republication_allowed,
+           rights_raw_retention,
+           rights_typed_retention,
+           rights_evidence_retention,
+           rights_scope,
+           rights_contract,
+           rights_decided_at
+      FROM sources_sourceconfiguration AS source
+      JOIN sources_sourcerightsdecision AS rights
+        ON rights.id = NEW.rights_decision_id
+     WHERE source.id = NEW.source_id
+     FOR UPDATE OF source, rights;
+
+    SELECT id INTO latest_rights_id
+      FROM sources_sourcerightsdecision AS latest_rights
+     WHERE latest_rights.source_id = NEW.source_id
+       AND latest_rights.source_revision = locked_source_revision
+     ORDER BY latest_rights.decision_sequence DESC, latest_rights.id DESC
+     LIMIT 1;
+
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
+CREATE TRIGGER travel_windows_duration_receipt_guard
+BEFORE INSERT OR UPDATE OR DELETE
+ON travel_windows_revieweddurationreceipt
+FOR EACH ROW EXECUTE FUNCTION travel_windows_guard_duration_receipt();
+
+CREATE FUNCTION travel_windows_guard_duration_lineage() RETURNS trigger
+LANGUAGE plpgsql AS $$
+DECLARE
+    receipt_outcome text;
+    receipt_reviewer text;
+    receipt_reviewed_at timestamptz;
+    receipt_created_at timestamptz;
+BEGIN
+    IF NEW.reviewed_duration_receipt_id IS NULL THEN
+        RETURN NEW;
+    END IF;
+    SELECT outcome, reviewed_by, reviewed_at, created_at
+      INTO receipt_outcome,
+           receipt_reviewer,
+           receipt_reviewed_at,
+           receipt_created_at
+      FROM travel_windows_revieweddurationreceipt
+     WHERE id = NEW.reviewed_duration_receipt_id;
+    IF NOT FOUND
+       OR receipt_outcome <> 'VALIDATED'
+       OR NEW.parse_run_id IS NOT NULL
+       OR NEW.state <> 'VALIDATED'
+       OR NEW.reviewed_by IS DISTINCT FROM receipt_reviewer
+       OR NEW.reviewed_at IS DISTINCT FROM receipt_reviewed_at
+       OR NEW.created_at < receipt_created_at THEN
+        RAISE EXCEPTION 'route duration revision does not match its reviewed receipt'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    RETURN NEW;
+END;
+$$;
+CREATE TRIGGER travel_windows_duration_lineage_guard
+BEFORE INSERT ON travel_windows_routedurationrevision
+FOR EACH ROW EXECUTE FUNCTION travel_windows_guard_duration_lineage();
+"""
+
+
+AVIATION_COLLECTION_RECEIPT_REVERSE_SQL = r"""
+DO $$
+BEGIN
+    IF EXISTS (SELECT 1 FROM travel_windows_revieweddurationreceipt) THEN
+        RAISE EXCEPTION 'aviation receipt rollback requires empty reviewed duration receipts'
+            USING ERRCODE = 'check_violation';
+    END IF;
+END;
+$$;
+DROP TRIGGER IF EXISTS travel_windows_duration_lineage_guard
+    ON travel_windows_routedurationrevision;
+DROP FUNCTION IF EXISTS travel_windows_guard_duration_lineage();
+DROP TRIGGER IF EXISTS travel_windows_duration_receipt_guard
+    ON travel_windows_revieweddurationreceipt;
+DROP FUNCTION IF EXISTS travel_windows_guard_duration_receipt();
+"""
+
+
+class Migration(migrations.Migration):
+
+    dependencies = [
+        ('sources', '0013_aviation_parse_inputs'),
+        ('travel_windows', '0008_seal_flight_candidates'),
+    ]
+
+    operations = [
+        migrations.AlterField(
+            model_name='routedurationrevision',
+            name='parse_run',
+            field=models.ForeignKey(blank=True, null=True, on_delete=django.db.models.deletion.PROTECT, related_name='route_duration_revisions', to='sources.parserun'),
+        ),
+        migrations.CreateModel(
+            name='ReviewedDurationReceipt',
+            fields=[
+                ('id', models.UUIDField(default=uuid.uuid4, editable=False, primary_key=True, serialize=False)),
+                ('source_revision', models.CharField(max_length=64)),
+                ('body_sha256', models.CharField(max_length=64)),
+                ('byte_count', models.PositiveBigIntegerField()),
+                ('parser_contract_fingerprint_sha256', models.CharField(max_length=64)),
+                ('expected_schema_fingerprint_sha256', models.CharField(max_length=64)),
+                ('observed_schema_fingerprint_sha256', models.CharField(blank=True, max_length=64)),
+                ('outcome', models.CharField(choices=[('VALIDATED', 'Validated'), ('QUARANTINED', 'Quarantined')], max_length=16)),
+                ('failure_code', models.CharField(blank=True, choices=[('SCHEMA_MISMATCH', 'Schema mismatch'), ('REQUIRED_VALUE_MISSING', 'Required value missing'), ('SYNTAX_ERROR', 'Syntax error'), ('INVALID_VALUE', 'Invalid value')], max_length=32)),
+                ('reviewed_by', models.CharField(max_length=100)),
+                ('reviewer_basis_code', models.CharField(max_length=100)),
+                ('reviewed_at', models.DateTimeField()),
+                ('created_at', models.DateTimeField(default=django.utils.timezone.now)),
+                ('rights_decision', models.ForeignKey(on_delete=django.db.models.deletion.PROTECT, related_name='reviewed_duration_receipts', to='sources.sourcerightsdecision')),
+                ('source', models.ForeignKey(on_delete=django.db.models.deletion.PROTECT, related_name='reviewed_duration_receipts', to='sources.sourceconfiguration')),
+            ],
+        ),
+        migrations.AddField(
+            model_name='routedurationrevision',
+            name='reviewed_duration_receipt',
+            field=models.ForeignKey(blank=True, null=True, on_delete=django.db.models.deletion.PROTECT, related_name='route_duration_revisions', to='travel_windows.revieweddurationreceipt'),
+        ),
+        migrations.AddConstraint(
+            model_name='routedurationrevision',
+            constraint=models.UniqueConstraint(fields=('reviewed_duration_receipt', 'destination_airport'), name='route_duration_receipt_airport_unique'),
+        ),
+        migrations.AddConstraint(
+            model_name='routedurationrevision',
+            constraint=models.CheckConstraint(condition=models.Q(models.Q(('parse_run__isnull', False), ('reviewed_duration_receipt__isnull', True)), models.Q(('parse_run__isnull', True), ('reviewed_duration_receipt__isnull', False)), _connector='OR'), name='route_duration_lineage_xor'),
+        ),
+        migrations.AddConstraint(
+            model_name='revieweddurationreceipt',
+            constraint=models.CheckConstraint(condition=models.Q(('source_revision__regex', '^[A-Za-z0-9][A-Za-z0-9._-]{0,63}$')), name='duration_receipt_source_revision'),
+        ),
+        migrations.AddConstraint(
+            model_name='revieweddurationreceipt',
+            constraint=models.CheckConstraint(condition=models.Q(('body_sha256__regex', '^[0-9a-f]{64}$')), name='duration_receipt_body_hash'),
+        ),
+        migrations.AddConstraint(
+            model_name='revieweddurationreceipt',
+            constraint=models.CheckConstraint(condition=models.Q(('byte_count__gte', 1)), name='duration_receipt_bytes_positive'),
+        ),
+        migrations.AddConstraint(
+            model_name='revieweddurationreceipt',
+            constraint=models.CheckConstraint(condition=models.Q(('parser_contract_fingerprint_sha256__regex', '^[0-9a-f]{64}$')), name='duration_receipt_contract_hash'),
+        ),
+        migrations.AddConstraint(
+            model_name='revieweddurationreceipt',
+            constraint=models.CheckConstraint(condition=models.Q(('expected_schema_fingerprint_sha256__regex', '^[0-9a-f]{64}$')), name='duration_receipt_expected_schema'),
+        ),
+        migrations.AddConstraint(
+            model_name='revieweddurationreceipt',
+            constraint=models.CheckConstraint(condition=models.Q(('observed_schema_fingerprint_sha256', ''), ('observed_schema_fingerprint_sha256__regex', '^[0-9a-f]{64}$'), _connector='OR'), name='duration_receipt_observed_schema'),
+        ),
+        migrations.AddConstraint(
+            model_name='revieweddurationreceipt',
+            constraint=models.CheckConstraint(condition=models.Q(('reviewed_by__regex', '^[A-Z][A-Z0-9_.:-]{2,99}$')), name='duration_receipt_reviewer_format'),
+        ),
+        migrations.AddConstraint(
+            model_name='revieweddurationreceipt',
+            constraint=models.CheckConstraint(condition=models.Q(('reviewer_basis_code__regex', '^[A-Z][A-Z0-9_]{2,99}$')), name='duration_receipt_basis_format'),
+        ),
+        migrations.AddConstraint(
+            model_name='revieweddurationreceipt',
+            constraint=models.CheckConstraint(condition=models.Q(('created_at__gte', models.F('reviewed_at'))), name='duration_receipt_created_after_review'),
+        ),
+        migrations.AddConstraint(
+            model_name='revieweddurationreceipt',
+            constraint=models.CheckConstraint(condition=models.Q(models.Q(('failure_code', ''), ('observed_schema_fingerprint_sha256', models.F('expected_schema_fingerprint_sha256')), ('outcome', 'VALIDATED')), models.Q(('failure_code', 'SCHEMA_MISMATCH'), ('outcome', 'QUARANTINED'), models.Q(('observed_schema_fingerprint_sha256', ''), _negated=True), models.Q(('observed_schema_fingerprint_sha256', models.F('expected_schema_fingerprint_sha256')), _negated=True)), models.Q(('failure_code', 'SYNTAX_ERROR'), ('observed_schema_fingerprint_sha256', ''), ('outcome', 'QUARANTINED')), models.Q(('failure_code__in', ('REQUIRED_VALUE_MISSING', 'INVALID_VALUE')), ('observed_schema_fingerprint_sha256', models.F('expected_schema_fingerprint_sha256')), ('outcome', 'QUARANTINED')), _connector='OR'), name='duration_receipt_outcome_shape'),
+        ),
+        migrations.RunSQL(
+            AVIATION_COLLECTION_RECEIPT_SQL,
+            AVIATION_COLLECTION_RECEIPT_REVERSE_SQL,
+        ),
+    ]
diff --git a/travel_windows/models.py b/travel_windows/models.py
index 8209745..247346f 100644
--- a/travel_windows/models.py
+++ b/travel_windows/models.py
@@ -247,6 +247,151 @@ class FlightSchedule(models.Model):
         ]
 
 
+class ReviewedDurationReceipt(models.Model):
+    """Immutable receipt for an operator-reviewed, non-HTTP duration file."""
+
+    class Outcome(models.TextChoices):
+        VALIDATED = "VALIDATED", "Validated"
+        QUARANTINED = "QUARANTINED", "Quarantined"
+
+    class FailureCode(models.TextChoices):
+        SCHEMA_MISMATCH = "SCHEMA_MISMATCH", "Schema mismatch"
+        REQUIRED_VALUE_MISSING = (
+            "REQUIRED_VALUE_MISSING",
+            "Required value missing",
+        )
+        SYNTAX_ERROR = "SYNTAX_ERROR", "Syntax error"
+        INVALID_VALUE = "INVALID_VALUE", "Invalid value"
+
+    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
+    source = models.ForeignKey(
+        "sources.SourceConfiguration",
+        on_delete=models.PROTECT,
+        related_name="reviewed_duration_receipts",
+    )
+    source_revision = models.CharField(max_length=64)
+    rights_decision = models.ForeignKey(
+        "sources.SourceRightsDecision",
+        on_delete=models.PROTECT,
+        related_name="reviewed_duration_receipts",
+    )
+    body_sha256 = models.CharField(max_length=64)
+    byte_count = models.PositiveBigIntegerField()
+    parser_contract_fingerprint_sha256 = models.CharField(max_length=64)
+    expected_schema_fingerprint_sha256 = models.CharField(max_length=64)
+    observed_schema_fingerprint_sha256 = models.CharField(max_length=64, blank=True)
+    outcome = models.CharField(max_length=16, choices=Outcome.choices)
+    failure_code = models.CharField(
+        max_length=32,
+        choices=FailureCode.choices,
+        blank=True,
+    )
+    reviewed_by = models.CharField(max_length=100)
+    reviewer_basis_code = models.CharField(max_length=100)
+    reviewed_at = models.DateTimeField()
+    created_at = models.DateTimeField(default=timezone.now)
+
+    class Meta:
+        constraints = [
+            models.CheckConstraint(
+                condition=Q(
+                    source_revision__regex=(
+                        r"^[A-Za-z0-9][A-Za-z0-9._-]{0,63}$"
+                    )
+                ),
+                name="duration_receipt_source_revision",
+            ),
+            models.CheckConstraint(
+                condition=Q(body_sha256__regex=r"^[0-9a-f]{64}$"),
+                name="duration_receipt_body_hash",
+            ),
+            models.CheckConstraint(
+                condition=Q(byte_count__gte=1),
+                name="duration_receipt_bytes_positive",
+            ),
+            models.CheckConstraint(
+                condition=Q(
+                    parser_contract_fingerprint_sha256__regex=(
+                        r"^[0-9a-f]{64}$"
+                    )
+                ),
+                name="duration_receipt_contract_hash",
+            ),
+            models.CheckConstraint(
+                condition=Q(
+                    expected_schema_fingerprint_sha256__regex=(
+                        r"^[0-9a-f]{64}$"
+                    )
+                ),
+                name="duration_receipt_expected_schema",
+            ),
+            models.CheckConstraint(
+                condition=(
+                    Q(observed_schema_fingerprint_sha256="")
+                    | Q(
+                        observed_schema_fingerprint_sha256__regex=(
+                            r"^[0-9a-f]{64}$"
+                        )
+                    )
+                ),
+                name="duration_receipt_observed_schema",
+            ),
+            models.CheckConstraint(
+                condition=Q(
+                    reviewed_by__regex=r"^[A-Z][A-Z0-9_.:-]{2,99}$"
+                ),
+                name="duration_receipt_reviewer_format",
+            ),
+            models.CheckConstraint(
+                condition=Q(
+                    reviewer_basis_code__regex=r"^[A-Z][A-Z0-9_]{2,99}$"
+                ),
+                name="duration_receipt_basis_format",
+            ),
+            models.CheckConstraint(
+                condition=Q(created_at__gte=F("reviewed_at")),
+                name="duration_receipt_created_after_review",
+            ),
+            models.CheckConstraint(
+                condition=(
+                    Q(
+                        outcome="VALIDATED",
+                        failure_code="",
+                        observed_schema_fingerprint_sha256=F(
+                            "expected_schema_fingerprint_sha256"
+                        ),
+                    )
+                    | Q(
+                        outcome="QUARANTINED",
+                        failure_code="SCHEMA_MISMATCH",
+                    )
+                    & ~Q(observed_schema_fingerprint_sha256="")
+                    & ~Q(
+                        observed_schema_fingerprint_sha256=F(
+                            "expected_schema_fingerprint_sha256"
+                        )
+                    )
+                    | Q(
+                        outcome="QUARANTINED",
+                        failure_code="SYNTAX_ERROR",
+                        observed_schema_fingerprint_sha256="",
+                    )
+                    | Q(
+                        outcome="QUARANTINED",
+                        failure_code__in=(
+                            "REQUIRED_VALUE_MISSING",
+                            "INVALID_VALUE",
+                        ),
+                        observed_schema_fingerprint_sha256=F(
+                            "expected_schema_fingerprint_sha256"
+                        ),
+                    )
+                ),
+                name="duration_receipt_outcome_shape",
+            ),
+        ]
+
+
 class RouteDurationRevision(models.Model):
     class State(models.TextChoices):
         VALIDATED = "VALIDATED", "Validated"
@@ -257,6 +402,15 @@ class RouteDurationRevision(models.Model):
         "sources.ParseRun",
         on_delete=models.PROTECT,
         related_name="route_duration_revisions",
+        null=True,
+        blank=True,
+    )
+    reviewed_duration_receipt = models.ForeignKey(
+        ReviewedDurationReceipt,
+        on_delete=models.PROTECT,
+        related_name="route_duration_revisions",
+        null=True,
+        blank=True,
     )
     destination_airport = models.ForeignKey(
         Airport,
@@ -304,6 +458,23 @@ class RouteDurationRevision(models.Model):
                 fields=("parse_run", "destination_airport"),
                 name="route_duration_parse_airport_unique",
             ),
+            models.UniqueConstraint(
+                fields=("reviewed_duration_receipt", "destination_airport"),
+                name="route_duration_receipt_airport_unique",
+            ),
+            models.CheckConstraint(
+                condition=(
+                    Q(
+                        parse_run__isnull=False,
+                        reviewed_duration_receipt__isnull=True,
+                    )
+                    | Q(
+                        parse_run__isnull=True,
+                        reviewed_duration_receipt__isnull=False,
+                    )
+                ),
+                name="route_duration_lineage_xor",
+            ),
         ]
 
 
diff --git a/travel_windows/publication.py b/travel_windows/publication.py
index 0d4196a..8116917 100644
--- a/travel_windows/publication.py
+++ b/travel_windows/publication.py
@@ -19,7 +19,14 @@ from django.db import connection, transaction
 from django.db.models import F, Max
 from django.utils import timezone
 
-from sources.models import ParseRun, SourceConfiguration, SourceRightsDecision
+from sources.models import (
+    FetchAttempt,
+    ParseRun,
+    SourceArtifact,
+    SourceConfiguration,
+    SourceRightsDecision,
+    aviation_parse_input_identity,
+)
 
 from .contracts import (
     CITY_CONTRACT_FINGERPRINT_SHA256,
@@ -54,6 +61,7 @@ from .models import (
     FlightSchedule,
     FlightScheduleRevision,
     PublishedFlightSchedule,
+    ReviewedDurationReceipt,
     RouteDurationRevision,
 )
 from .parser import (
@@ -226,6 +234,7 @@ def _approved_run(
     parser_name: str,
     contract_fingerprint: str,
     schema_fingerprint: str,
+    allow_legacy_unordered: bool = False,
 ) -> ParseRun:
     try:
         locked = (
@@ -250,17 +259,134 @@ def _approved_run(
         or not source.enabled
     ):
         raise _PublicationRejected(FlightPublicationCode.SOURCE_GATE_FAILED)
-    approved = SourceRightsDecision.objects.select_for_update().filter(
-        source=source,
-        source_revision=source_revision,
-        decision=SourceRightsDecision.Decision.APPROVED,
-        access_allowed=True,
-        typed_field_storage_allowed=True,
-        typed_republication_allowed=True,
-        raw_body_storage_allowed=False,
-        contract_fingerprint_sha256=contract_fingerprint,
+    approved = (
+        SourceRightsDecision.objects.select_for_update()
+        .filter(source=source, source_revision=source_revision)
+        .order_by("-decision_sequence", "-id")
+        .first()
+    )
+    if (
+        approved is None
+        or approved.decision != SourceRightsDecision.Decision.APPROVED
+        or not approved.access_allowed
+        or not approved.typed_field_storage_allowed
+        or not approved.typed_republication_allowed
+        or approved.raw_body_storage_allowed
+        or approved.contract_fingerprint_sha256 != contract_fingerprint
+    ):
+        raise _PublicationRejected(FlightPublicationCode.SOURCE_GATE_FAILED)
+
+    ordered_parser = parser_name in {
+        ParseRun.ParserName.ICN_FLIGHT_SCHEDULE_JSON,
+        ParseRun.ParserName.ICN_DESTINATION_CITY_JSON,
+        ParseRun.ParserName.ICN_LEGACY_ARRIVALS_JSON,
+    }
+    if ordered_parser:
+        inputs = list(
+            locked.ordered_inputs.select_for_update()
+            .select_related(
+                "artifact__source",
+                "artifact__first_successful_attempt__rights_decision",
+            )
+            .order_by("ordinal")
+        )
+        if not inputs:
+            if allow_legacy_unordered and locked.input_identity_sha256 == "":
+                return locked
+            raise _PublicationRejected(FlightPublicationCode.SOURCE_GATE_FAILED)
+        identities = tuple(
+            (
+                row.ordinal,
+                row.role,
+                row.role_sequence,
+                row.artifact.body_sha256,
+            )
+            for row in inputs
+        )
+        try:
+            canonical_identity = aviation_parse_input_identity(identities)
+        except ValueError:
+            raise _PublicationRejected(
+                FlightPublicationCode.SOURCE_GATE_FAILED
+            ) from None
+        expected_roles: tuple[str, ...]
+        if parser_name == ParseRun.ParserName.ICN_FLIGHT_SCHEDULE_JSON:
+            expected_roles = ("SCHEDULE_DEPARTURE", "SCHEDULE_ARRIVAL")
+        elif parser_name == ParseRun.ParserName.ICN_DESTINATION_CITY_JSON:
+            expected_roles = ("DESTINATION_CITY",)
+        else:
+            expected_roles = ("LEGACY_ARRIVAL",)
+        observed_roles = tuple(dict.fromkeys(row.role for row in inputs))
+        if (
+            locked.input_identity_sha256 != canonical_identity
+            or locked.artifact_id != inputs[0].artifact_id
+            or observed_roles != expected_roles
+            or any(
+                row.artifact.source_id != source.pk
+                or row.artifact.state != SourceArtifact.State.REVIEW_REQUIRED
+                or row.artifact.first_successful_attempt.result
+                != FetchAttempt.Result.SUCCEEDED
+                or row.artifact.first_successful_attempt.response_sha256
+                != row.artifact.body_sha256
+                or row.artifact.first_successful_attempt.source_revision
+                != source_revision
+                or row.artifact.first_successful_attempt.rights_decision_id
+                != approved.pk
+                for row in inputs
+            )
+        ):
+            raise _PublicationRejected(FlightPublicationCode.SOURCE_GATE_FAILED)
+    return locked
+
+
+def _approved_duration_receipt(
+    receipt: ReviewedDurationReceipt,
+) -> ReviewedDurationReceipt:
+    try:
+        locked = (
+            ReviewedDurationReceipt.objects.select_for_update()
+            .select_related("source", "rights_decision")
+            .get(pk=receipt.pk)
+        )
+    except (ReviewedDurationReceipt.DoesNotExist, AttributeError, TypeError):
+        raise _PublicationRejected(FlightPublicationCode.INVALID_EVIDENCE) from None
+    source = locked.source
+    rights = locked.rights_decision
+    latest_rights = (
+        SourceRightsDecision.objects.select_for_update()
+        .filter(source=source, source_revision=source.revision)
+        .order_by("-decision_sequence", "-id")
+        .first()
     )
-    if approved.count() != 1:
+    if (
+        locked.outcome != ReviewedDurationReceipt.Outcome.VALIDATED
+        or locked.failure_code
+        or locked.source_revision != DURATION_SOURCE_REVISION
+        or locked.parser_contract_fingerprint_sha256
+        != DURATION_CONTRACT_FINGERPRINT_SHA256
+        or locked.expected_schema_fingerprint_sha256
+        != DURATION_SCHEMA_FINGERPRINT_SHA256
+        or locked.observed_schema_fingerprint_sha256
+        != DURATION_SCHEMA_FINGERPRINT_SHA256
+        or locked.byte_count < 1
+        or not _aware(locked.reviewed_at)
+        or source.code != DURATION_SOURCE_CODE
+        or source.revision != DURATION_SOURCE_REVISION
+        or source.module != SourceConfiguration.Module.AVIATION
+        or source.state != SourceConfiguration.State.ACTIVE
+        or not source.enabled
+        or latest_rights is None
+        or latest_rights.pk != rights.pk
+        or rights.source_id != source.pk
+        or rights.source_revision != source.revision
+        or rights.decision != SourceRightsDecision.Decision.APPROVED
+        or not rights.access_allowed
+        or not rights.typed_field_storage_allowed
+        or not rights.typed_republication_allowed
+        or rights.raw_body_storage_allowed
+        or rights.contract_fingerprint_sha256
+        != DURATION_CONTRACT_FINGERPRINT_SHA256
+    ):
         raise _PublicationRejected(FlightPublicationCode.SOURCE_GATE_FAILED)
     return locked
 
@@ -359,7 +485,7 @@ def stage_flight_evidence(
     city_reference: CityReferenceParseSuccess,
     legacy_arrivals_run: ParseRun,
     legacy_arrivals: LegacyArrivalParseSuccess,
-    duration_run: ParseRun,
+    duration_receipt: ReviewedDurationReceipt,
     durations: DurationParseSuccess,
     source_checked_at: datetime,
 ) -> FlightPublicationOutcome:
@@ -407,20 +533,15 @@ def stage_flight_evidence(
                     LEGACY_SCHEDULE_SCHEMA_FINGERPRINT_SHA256
                 ),
             )
-            locked_duration_run = _approved_run(
-                duration_run,
-                source_code=DURATION_SOURCE_CODE,
-                source_revision=DURATION_SOURCE_REVISION,
-                parser_name=ParseRun.ParserName.ROUTE_DURATION_CSV,
-                contract_fingerprint=DURATION_CONTRACT_FINGERPRINT_SHA256,
-                schema_fingerprint=DURATION_SCHEMA_FINGERPRINT_SHA256,
+            locked_duration_receipt = _approved_duration_receipt(
+                duration_receipt
             )
             effective_checked_at = max(
                 source_checked_at,
                 locked_schedule_run.completed_at,
                 locked_city_reference_run.completed_at,
                 locked_legacy_arrivals_run.completed_at,
-                locked_duration_run.completed_at,
+                locked_duration_receipt.reviewed_at,
             )
             if (
                 schedule.observed_schema_fingerprint_sha256
@@ -497,18 +618,23 @@ def stage_flight_evidence(
                     continue
                 duration_revisions.append(
                     RouteDurationRevision.objects.create(
-                        parse_run=locked_duration_run,
+                        parse_run=None,
+                        reviewed_duration_receipt=locked_duration_receipt,
                         destination_airport=airports[row.destination_iata],
                         outbound_minutes=row.outbound_minutes,
                         inbound_minutes=row.inbound_minutes,
                         reference_date=row.reference_date,
                         reference_locator=row.reference_locator,
                         state=RouteDurationRevision.State.VALIDATED,
-                        reviewed_by="AVIATION_TYPED_VALIDATOR_V1",
-                        reviewed_at=effective_checked_at,
+                        reviewed_by=locked_duration_receipt.reviewed_by,
+                        reviewed_at=locked_duration_receipt.reviewed_at,
                         typed_fingerprint_sha256=_fingerprint(
                             _duration_payload_from_parsed(row)
                         ),
+                        created_at=max(
+                            timezone.now(),
+                            locked_duration_receipt.created_at,
+                        ),
                     )
                 )
             FlightCandidateDuration.objects.bulk_create(
@@ -608,6 +734,8 @@ def _load_valid_candidate(
     ):
         raise _PublicationRejected(FlightPublicationCode.INVALID_EVIDENCE)
 
+    allow_legacy_lineage = revision.publications.exists()
+
     _approved_run(
         revision.parse_run,
         source_code=SCHEDULE_SOURCE_CODE,
@@ -615,6 +743,7 @@ def _load_valid_candidate(
         parser_name=ParseRun.ParserName.ICN_FLIGHT_SCHEDULE_JSON,
         contract_fingerprint=SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
         schema_fingerprint=SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
+        allow_legacy_unordered=allow_legacy_lineage,
     )
     _approved_run(
         revision.city_reference_parse_run,
@@ -623,6 +752,7 @@ def _load_valid_candidate(
         parser_name=ParseRun.ParserName.ICN_DESTINATION_CITY_JSON,
         contract_fingerprint=CITY_CONTRACT_FINGERPRINT_SHA256,
         schema_fingerprint=CITY_SCHEMA_FINGERPRINT_SHA256,
+        allow_legacy_unordered=allow_legacy_lineage,
     )
     _approved_run(
         revision.legacy_arrivals_parse_run,
@@ -631,6 +761,7 @@ def _load_valid_candidate(
         parser_name=ParseRun.ParserName.ICN_LEGACY_ARRIVALS_JSON,
         contract_fingerprint=LEGACY_SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
         schema_fingerprint=LEGACY_SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
+        allow_legacy_unordered=allow_legacy_lineage,
     )
 
     schedules = list(
@@ -653,9 +784,10 @@ def _load_valid_candidate(
     )
 
     links = list(
-        FlightCandidateDuration.objects.select_for_update()
+        FlightCandidateDuration.objects.select_for_update(of=("self",))
         .select_related(
             "duration_revision__parse_run",
+            "duration_revision__reviewed_duration_receipt",
             "duration_revision__destination_airport",
             "destination_airport",
         )
@@ -664,6 +796,7 @@ def _load_valid_candidate(
     if {link.destination_airport_id for link in links} != set(directions):
         raise _PublicationRejected(FlightPublicationCode.INVALID_EVIDENCE)
     validated_runs: set[object] = set()
+    validated_receipts: set[object] = set()
     duration_identities: list[tuple[str, str]] = []
     duration_fingerprints: list[tuple[RouteDurationRevision, str]] = []
     for link in links:
@@ -673,6 +806,8 @@ def _load_valid_candidate(
             or duration.state != RouteDurationRevision.State.VALIDATED
             or not 1 <= duration.outbound_minutes <= 1440
             or not 1 <= duration.inbound_minutes <= 1440
+            or (duration.parse_run_id is None)
+            == (duration.reviewed_duration_receipt_id is None)
         ):
             raise _PublicationRejected(FlightPublicationCode.INVALID_EVIDENCE)
         duration_fingerprint = _fingerprint(
@@ -685,7 +820,25 @@ def _load_valid_candidate(
                 duration_fingerprint,
             )
         )
-        if duration.parse_run_id not in validated_runs:
+        if duration.reviewed_duration_receipt_id is not None:
+            if (
+                duration.reviewed_duration_receipt_id
+                not in validated_receipts
+            ):
+                receipt = _approved_duration_receipt(
+                    duration.reviewed_duration_receipt
+                )
+                if (
+                    duration.reviewed_by != receipt.reviewed_by
+                    or duration.reviewed_at != receipt.reviewed_at
+                ):
+                    raise _PublicationRejected(
+                        FlightPublicationCode.INVALID_EVIDENCE
+                    )
+                validated_receipts.add(receipt.pk)
+        elif not allow_legacy_lineage:
+            raise _PublicationRejected(FlightPublicationCode.INVALID_EVIDENCE)
+        elif duration.parse_run_id not in validated_runs:
             _approved_run(
                 duration.parse_run,
                 source_code=DURATION_SOURCE_CODE,
diff --git a/travel_windows/tests/test_publication_sealing.py b/travel_windows/tests/test_publication_sealing.py
index 98d72b5..1c77ab9 100644
--- a/travel_windows/tests/test_publication_sealing.py
+++ b/travel_windows/tests/test_publication_sealing.py
@@ -10,10 +10,7 @@ from sources.management.commands.register_approved_sources import (
     register_approved_sources,
 )
 from travel_windows.contracts import DURATION_SOURCE_LOCATOR
-from travel_windows.ingestion import (
-    FlightIngestionCode,
-    ingest_flight_evidence_candidate,
-)
+from travel_windows.ingestion import FlightIngestionCode
 from travel_windows.models import (
     Airport,
     FlightPublicationDuration,
@@ -27,6 +24,7 @@ from travel_windows.publication import (
 )
 from travel_windows.tests.test_source_publication import (
     city_reference_payload,
+    collect_and_stage_fixture,
     legacy_arrival_page,
     legacy_arrival_row,
     official_page,
@@ -37,7 +35,7 @@ from travel_windows.tests.test_source_publication import (
 class PublishedFlightEvidenceSealingTests(TransactionTestCase):
     def setUp(self):
         register_approved_sources(apply=True)
-        outcome = ingest_flight_evidence_candidate(
+        outcome = collect_and_stage_fixture(
             departure_pages=(official_page([official_row()]),),
             arrival_pages=(
                 official_page(
@@ -85,17 +83,27 @@ class PublishedFlightEvidenceSealingTests(TransactionTestCase):
         )
         reviewed_at = timezone.now()
         return RouteDurationRevision.objects.create(
-            parse_run=existing.parse_run,
+            parse_run=None,
+            reviewed_duration_receipt=(
+                existing.reviewed_duration_receipt
+            ),
             destination_airport=Airport.objects.get(iata_code=airport_code),
             outbound_minutes=120,
             inbound_minutes=120,
             reference_date=date(2026, 8, 1),
             reference_locator=DURATION_SOURCE_LOCATOR,
             state=RouteDurationRevision.State.VALIDATED,
-            reviewed_by="synthetic-sealing-reviewer",
-            reviewed_at=reviewed_at,
+            reviewed_by=(
+                existing.reviewed_duration_receipt.reviewed_by
+            ),
+            reviewed_at=(
+                existing.reviewed_duration_receipt.reviewed_at
+            ),
             typed_fingerprint_sha256=fingerprint_character * 64,
-            created_at=reviewed_at,
+            created_at=max(
+                reviewed_at,
+                existing.reviewed_duration_receipt.created_at,
+            ),
         )
 
     def test_pointer_advance_seals_schedule_and_duration_membership(self):
@@ -106,7 +114,7 @@ class PublishedFlightEvidenceSealingTests(TransactionTestCase):
         schedule_count = FlightSchedule.objects.count()
         with self.assertRaisesRegex(
             IntegrityError,
-            "published flight schedule membership is immutable",
+            "(published|sealed) flight schedule membership is immutable",
         ), transaction.atomic():
             FlightSchedule.objects.create(
                 revision=first_publication.schedule_revision,
diff --git a/travel_windows/tests/test_review_lifecycle.py b/travel_windows/tests/test_review_lifecycle.py
index 25d7f56..abb1a8a 100644
--- a/travel_windows/tests/test_review_lifecycle.py
+++ b/travel_windows/tests/test_review_lifecycle.py
@@ -20,10 +20,7 @@ from travel_windows.contracts import (
     SCHEDULE_SOURCE_REVISION,
 )
 from travel_windows.admin import _expected_version, operator_admin_site
-from travel_windows.ingestion import (
-    FlightIngestionCode,
-    ingest_flight_evidence_candidate,
-)
+from travel_windows.ingestion import FlightIngestionCode
 from travel_windows.models import (
     Airport,
     CURATED_AIRPORT_ROWS,
@@ -46,6 +43,7 @@ from travel_windows.publication import (
 )
 from travel_windows.tests.test_source_publication import (
     city_reference_payload,
+    collect_and_stage_fixture,
     legacy_arrival_page,
     legacy_arrival_row,
     official_page,
@@ -129,7 +127,7 @@ class FlightReviewLifecycleTests(TransactionTestCase):
                     "airportCode": "AMS",
                 },
             )
-        outcome = ingest_flight_evidence_candidate(
+        outcome = collect_and_stage_fixture(
             departure_pages=(
                 official_page([official_row(st=outbound_time)]),
             ),
diff --git a/travel_windows/tests/test_review_lifecycle_migration.py b/travel_windows/tests/test_review_lifecycle_migration.py
index 4a9e0ad..06b13cf 100644
--- a/travel_windows/tests/test_review_lifecycle_migration.py
+++ b/travel_windows/tests/test_review_lifecycle_migration.py
@@ -50,6 +50,10 @@ class FlightReviewLifecycleMigrationTests(TransactionTestCase):
     ]
     reviewed = [("travel_windows", "0007_flight_review_lifecycle")]
     sealed = [("travel_windows", "0008_seal_flight_candidates")]
+    current = [
+        ("sources", "0013_aviation_parse_inputs"),
+        ("travel_windows", "0009_aviation_collection_receipts"),
+    ]
 
     def setUp(self):
         self.addCleanup(self._restore_head)
@@ -78,6 +82,7 @@ class FlightReviewLifecycleMigrationTests(TransactionTestCase):
                 "travel_windows_flightcandidateduration",
                 "travel_windows_flightschedule",
                 "travel_windows_routedurationrevision",
+                "travel_windows_revieweddurationreceipt",
                 "travel_windows_flightschedulerevision",
                 "travel_windows_publishedflightschedule",
             )
@@ -352,18 +357,23 @@ class FlightReviewLifecycleMigrationTests(TransactionTestCase):
                 legacy_hashes[duration_id],
             )
 
+        MigrationExecutor(connection).migrate(self.current)
+        current_apps = MigrationExecutor(connection).loader.project_state(
+            self.current
+        ).apps
+        CurrentDuration = current_apps.get_model(
+            "travel_windows", "RouteDurationRevision"
+        )
+        for duration_id in (first_duration.pk, second_duration.pk):
+            upgraded = CurrentDuration.objects.get(pk=duration_id)
+            self.assertIsNotNone(upgraded.parse_run_id)
+            self.assertIsNone(upgraded.reviewed_duration_receipt_id)
+
         evidence = _load_current_flight_evidence()
         self.assertIsNotNone(evidence)
         self.assertEqual(evidence.generation, 2)
         self.assertEqual(len(evidence.schedules), 2)
         self.assertEqual(len(evidence.durations), 1)
-
-        MigrationExecutor(connection).migrate(
-            [
-                ("sources", "0013_aviation_parse_inputs"),
-                ("travel_windows", "0008_seal_flight_candidates"),
-            ]
-        )
         actor = get_user_model().objects.create_user(
             username="legacy-rollback-operator",
             password="synthetic-password",
diff --git a/travel_windows/tests/test_search.py b/travel_windows/tests/test_search.py
index 819cc7c..0de9b8b 100644
--- a/travel_windows/tests/test_search.py
+++ b/travel_windows/tests/test_search.py
@@ -25,10 +25,7 @@ from sources.management.commands.register_approved_sources import (
     register_approved_sources,
 )
 from travel_windows.contracts import DURATION_SOURCE_LOCATOR
-from travel_windows.ingestion import (
-    FlightIngestionCode,
-    ingest_flight_evidence_candidate,
-)
+from travel_windows.ingestion import FlightIngestionCode
 from travel_windows.models import (
     CURATED_AIRPORT_ROWS,
     TIMEZONE_EVIDENCE_LOCATOR,
@@ -58,6 +55,7 @@ from travel_windows.publication import (
 )
 from travel_windows.tests.test_source_publication import (
     city_reference_payload,
+    collect_and_stage_fixture,
     legacy_arrival_page,
     legacy_arrival_row,
 )
@@ -781,7 +779,7 @@ class CurrentFlightPublicationSearchTests(TransactionTestCase):
 
     def test_search_reads_current_publication_without_persisting_search_values(self):
         nrt = Airport.objects.get(iata_code="NRT")
-        outcome = ingest_flight_evidence_candidate(
+        outcome = collect_and_stage_fixture(
             departure_pages=(
                 _official_page(
                     [
diff --git a/travel_windows/tests/test_source_publication.py b/travel_windows/tests/test_source_publication.py
index 17bfdf1..0a66bb8 100644
--- a/travel_windows/tests/test_source_publication.py
+++ b/travel_windows/tests/test_source_publication.py
@@ -6,6 +6,7 @@ from types import SimpleNamespace
 from unittest.mock import patch
 
 from django.core.management import call_command
+from django.db import DatabaseError, transaction
 from django.test import SimpleTestCase, TransactionTestCase
 from django.utils import timezone
 
@@ -13,7 +14,15 @@ from countries.models import Country, SUPPORTED_COUNTRY_ROWS
 from sources.management.commands.register_approved_sources import (
     register_approved_sources,
 )
-from sources.models import FetchAttempt, ParseRun, SourceArtifact
+from sources.models import (
+    FetchAttempt,
+    ParseRun,
+    ParseRunInput,
+    SourceArtifact,
+    SourceConfiguration,
+    SourceRightsDecision,
+    aviation_parse_input_identity,
+)
 from sources.tests.test_transport import FakeConnection, FakeResponse
 from sources.transport import (
     fetch_data_go_reference_evidence,
@@ -21,14 +30,17 @@ from sources.transport import (
     load_aviation_secret_reference,
 )
 from travel_windows.ingestion import (
+    DurableAviationAttemptRecorder,
     FlightIngestionCode,
     ingest_flight_evidence_candidate,
 )
 from travel_windows.management.commands.publish_scheduled_flights import Command
 from travel_windows.contracts import (
+    CITY_SOURCE_CODE,
+    DURATION_CONTRACT_FINGERPRINT_SHA256,
+    DURATION_SCHEMA_FINGERPRINT_SHA256,
+    DURATION_SOURCE_CODE,
     DURATION_SOURCE_LOCATOR,
-    SCHEDULE_ARRIVALS_LOCATOR,
-    SCHEDULE_DEPARTURES_LOCATOR,
     SCHEDULE_SOURCE_CODE,
 )
 from travel_windows.models import (
@@ -37,6 +49,8 @@ from travel_windows.models import (
     FlightSchedule,
     FlightScheduleRevision,
     PublishedFlightSchedule,
+    ReviewedDurationReceipt,
+    RouteDurationRevision,
     TIMEZONE_EVIDENCE_LOCATOR,
 )
 from travel_windows.parser import (
@@ -182,6 +196,67 @@ def legacy_arrival_page(rows, *, page=1, page_size=None, total=None):
     ).encode("utf-8")
 
 
+def collect_and_stage_fixture(
+    *,
+    departure_pages,
+    arrival_pages,
+    city_payload,
+    legacy_arrival_pages,
+    duration_csv,
+    source_date,
+    source_checked_at=None,
+):
+    """Exercise the production receipt boundary with synthetic HTTP peers."""
+
+    recorder = DurableAviationAttemptRecorder()
+    schedule_connections = [
+        FakeConnection(FakeResponse(200, body))
+        for body in (*departure_pages, *arrival_pages)
+    ]
+
+    def schedule_factory(_host, _port, _timeout):
+        return schedule_connections.pop(0)
+
+    schedule_fetch = fetch_data_go_schedule_pages(
+        secret_reference_name="DATA_GO_KR_SERVICE_KEY",
+        secret_value="synthetic-collection-key",
+        season="S26",
+        connect_timeout_seconds=3,
+        read_timeout_seconds=8,
+        connection_factory=schedule_factory,
+        attempt_executor=recorder,
+    )
+    reference_connections = [
+        FakeConnection(FakeResponse(200, city_payload)),
+        *(
+            FakeConnection(FakeResponse(200, body))
+            for body in legacy_arrival_pages
+        ),
+    ]
+
+    def reference_factory(_host, _port, _timeout):
+        return reference_connections.pop(0)
+
+    reference_fetch = fetch_data_go_reference_evidence(
+        secret_reference_name="DATA_GO_KR_SERVICE_KEY",
+        secret_value="synthetic-collection-key",
+        connect_timeout_seconds=3,
+        read_timeout_seconds=8,
+        connection_factory=reference_factory,
+        attempt_executor=recorder,
+    )
+    return ingest_flight_evidence_candidate(
+        schedule_fetch=schedule_fetch,
+        reference_fetch=reference_fetch,
+        attempt_recorder=recorder,
+        duration_csv=duration_csv,
+        source_date=source_date,
+        duration_reviewed_by="TEST_DURATION_REVIEWER",
+        duration_reviewer_basis_code="TEST_DURATION_FILE_REVIEW_V1",
+        source_checked_at=source_checked_at,
+    )
+
+
 class DataGoScheduleAdapterTests(SimpleTestCase):
     def test_complete_pages_normalize_and_dedupe_codeshare_by_master(self):
         departure = official_page(
@@ -444,13 +519,25 @@ class FlightPublicationCommandTests(SimpleTestCase):
             duration_csv="reviewed.csv",
             source_date="2026-08-31",
             season="S26",
+            duration_reviewed_by="TEST_DURATION_REVIEWER",
+            duration_review_basis="TEST_DURATION_FILE_REVIEW_V1",
             stdout=output,
         )
 
         reference_fetch.assert_called_once()
+        self.assertIs(
+            schedule_fetch.call_args.kwargs["attempt_executor"],
+            reference_fetch.call_args.kwargs["attempt_executor"],
+        )
         kwargs = ingest.call_args.kwargs
-        self.assertEqual(kwargs["city_payload"], b"city")
-        self.assertEqual(kwargs["legacy_arrival_pages"], (b"legacy",))
+        self.assertIs(kwargs["schedule_fetch"], schedule_fetch.return_value)
+        self.assertIs(
+            kwargs["reference_fetch"], reference_fetch.return_value
+        )
+        self.assertIs(
+            kwargs["attempt_recorder"],
+            schedule_fetch.call_args.kwargs["attempt_executor"],
+        )
         self.assertEqual(
             output.getvalue(),
             "result=staged "
@@ -509,7 +596,7 @@ class FlightEvidencePublicationTests(TransactionTestCase):
             + DURATION_SOURCE_LOCATOR.encode("ascii")
             + b"\r\n"
         )
-        outcome = ingest_flight_evidence_candidate(
+        outcome = collect_and_stage_fixture(
             departure_pages=(departure,),
             arrival_pages=(arrival,),
             city_payload=city_reference_payload(),
@@ -529,26 +616,85 @@ class FlightEvidencePublicationTests(TransactionTestCase):
         self.assertEqual(FlightSchedule.objects.count(), 2)
         self.assertEqual(FetchAttempt.objects.count(), 4)
         self.assertEqual(SourceArtifact.objects.count(), 4)
-        self.assertEqual(ParseRun.objects.count(), 4)
-        schedule_attempt = FetchAttempt.objects.get(
-            source__code=SCHEDULE_SOURCE_CODE
-        )
+        self.assertEqual(ParseRun.objects.count(), 3)
+        self.assertEqual(ReviewedDurationReceipt.objects.count(), 1)
         self.assertEqual(
-            schedule_attempt.normalized_request_sha256,
-            hashlib.sha256(
-                (
-                    f"{SCHEDULE_DEPARTURES_LOCATOR}\n"
-                    f"{SCHEDULE_ARRIVALS_LOCATOR}\n"
-                    "serviceKey=<redacted>&pageNo=1..N&type=json&"
-                    "season=S26"
-                ).encode("utf-8")
-            ).hexdigest(),
+            FetchAttempt.objects.filter(
+                source__code=SCHEDULE_SOURCE_CODE
+            ).count(),
+            2,
+        )
+        self.assertFalse(
+            FetchAttempt.objects.filter(
+                source__code=DURATION_SOURCE_CODE
+            ).exists()
+        )
+        self.assertFalse(
+            SourceArtifact.objects.filter(
+                source__code=DURATION_SOURCE_CODE
+            ).exists()
         )
         revision = FlightScheduleRevision.objects.get(
             pk=outcome.schedule_revision_id
         )
+        ordered = list(
+            revision.parse_run.ordered_inputs.select_related("artifact").order_by(
+                "ordinal"
+            )
+        )
+        identities = tuple(
+            (
+                row.ordinal,
+                row.role,
+                row.role_sequence,
+                row.artifact.body_sha256,
+            )
+            for row in ordered
+        )
+        self.assertEqual(
+            [(row.ordinal, row.role, row.role_sequence) for row in ordered],
+            [
+                (1, ParseRunInput.Role.SCHEDULE_DEPARTURE, 1),
+                (2, ParseRunInput.Role.SCHEDULE_ARRIVAL, 1),
+            ],
+        )
+        self.assertEqual(
+            revision.parse_run.input_identity_sha256,
+            aviation_parse_input_identity(identities),
+        )
+        city_payload = city_reference_payload()
+        city_artifact = SourceArtifact.objects.get(
+            source__code=CITY_SOURCE_CODE
+        )
+        self.assertEqual(
+            city_artifact.body_sha256,
+            hashlib.sha256(city_payload).hexdigest(),
+        )
+        self.assertEqual(city_artifact.byte_count, len(city_payload))
         self.assertIsNotNone(revision.city_reference_parse_run_id)
         self.assertIsNotNone(revision.legacy_arrivals_parse_run_id)
+        duration_revision = RouteDurationRevision.objects.get()
+        self.assertIsNone(duration_revision.parse_run_id)
+        self.assertIsNotNone(
+            duration_revision.reviewed_duration_receipt_id
+        )
+        duration_receipt = duration_revision.reviewed_duration_receipt
+        self.assertEqual(
+            duration_receipt.body_sha256,
+            hashlib.sha256(duration).hexdigest(),
+        )
+        self.assertEqual(duration_receipt.byte_count, len(duration))
+        self.assertEqual(
+            duration_receipt.outcome,
+            ReviewedDurationReceipt.Outcome.VALIDATED,
+        )
+        self.assertEqual(
+            duration_receipt.reviewed_by, "TEST_DURATION_REVIEWER"
+        )
+        self.assertEqual(
+            duration_receipt.reviewer_basis_code,
+            "TEST_DURATION_FILE_REVIEW_V1",
+        )
         model_fields = {
             field.name
             for model in (FetchAttempt, SourceArtifact, ParseRun)
@@ -557,6 +703,229 @@ class FlightEvidencePublicationTests(TransactionTestCase):
         self.assertNotIn("raw_body", model_fields)
         self.assertNotIn("response_body", model_fields)
 
+    def test_failed_real_call_keeps_the_closed_attempt(self):
+        recorder = DurableAviationAttemptRecorder()
+        failed_connection = FakeConnection(FakeResponse(503, b"upstream"))
+
+        result = fetch_data_go_schedule_pages(
+            secret_reference_name="DATA_GO_KR_SERVICE_KEY",
+            secret_value="synthetic-collection-key",
+            season="S26",
+            connect_timeout_seconds=3,
+            read_timeout_seconds=8,
+            connection_factory=lambda _host, _port, _timeout: (
+                failed_connection
+            ),
+            attempt_executor=recorder,
+        )
+
+        self.assertFalse(result.succeeded)
+        attempt = FetchAttempt.objects.get()
+        self.assertEqual(
+            attempt.result, FetchAttempt.Result.RETRYABLE_FAILED
+        )
+        self.assertEqual(attempt.http_status, 503)
+        self.assertEqual(
+            attempt.failure_code, FetchAttempt.FailureCode.UPSTREAM_5XX
+        )
+        self.assertEqual(attempt.response_sha256, "")
+        self.assertFalse(SourceArtifact.objects.exists())
+
+    def test_duration_receipt_requires_latest_exact_rights_and_is_immutable(self):
+        source = SourceConfiguration.objects.get(code=DURATION_SOURCE_CODE)
+        prior = SourceRightsDecision.objects.get(
+            source=source,
+            decision_sequence=1,
+        )
+        reviewed_at = max(timezone.now(), prior.decided_at)
+        values = {
+            "source": source,
+            "source_revision": source.revision,
+            "body_sha256": "a" * 64,
+            "byte_count": 1,
+            "parser_contract_fingerprint_sha256": (
+                DURATION_CONTRACT_FINGERPRINT_SHA256
+            ),
+            "expected_schema_fingerprint_sha256": (
+                DURATION_SCHEMA_FINGERPRINT_SHA256
+            ),
+            "observed_schema_fingerprint_sha256": (
+                DURATION_SCHEMA_FINGERPRINT_SHA256
+            ),
+            "outcome": ReviewedDurationReceipt.Outcome.VALIDATED,
+            "failure_code": "",
+            "reviewed_by": "TEST_DURATION_REVIEWER",
+            "reviewer_basis_code": "TEST_DURATION_FILE_REVIEW_V1",
+            "reviewed_at": reviewed_at,
+            "created_at": reviewed_at,
+        }
+        receipt = ReviewedDurationReceipt.objects.create(
+            rights_decision=prior,
+            **values,
+        )
+        with self.assertRaisesRegex(
+            DatabaseError,
+            "append-only",
+        ), transaction.atomic():
+            ReviewedDurationReceipt.objects.filter(pk=receipt.pk).update(
+                byte_count=2
+            )
+
+        SourceRightsDecision.objects.create(
+            source=source,
+            source_revision=prior.source_revision,
+            decision_sequence=2,
+            decision=SourceRightsDecision.Decision.REJECTED,
+            access_mode=SourceRightsDecision.AccessMode.NO_ACCESS,
+            access_allowed=False,
+            automated_collection_allowed=False,
+            typed_field_storage_allowed=False,
+            raw_body_storage_allowed=False,
+            typed_republication_allowed=False,
+            raw_retention_seconds=0,
+            typed_retention=SourceRightsDecision.Retention.NONE,
+            evidence_retention=(
+                SourceRightsDecision.Retention.PRODUCT_HISTORY
+            ),
+            field_scope_code="",
+            attribution_text="",
+            contract_fingerprint_sha256=(
+                prior.contract_fingerprint_sha256
+            ),
+            decided_by="TEST_DURATION_RIGHTS_REVIEWER",
+            decision_basis_code="TEST_DURATION_RIGHTS_REVOKED_V1",
+            decided_at=timezone.now(),
+        )
+        with self.assertRaisesRegex(
+            DatabaseError,
+            "outside the approved source contract",
+        ), transaction.atomic():
+            ReviewedDurationReceipt.objects.create(
+                rights_decision=prior,
+                **values,
+            )
+
+    def test_city_schema_drift_is_quarantined_with_exact_artifact(self):
+        city_document = json.loads(city_reference_payload())
+        city_document["response"]["body"]["items"]["item"][0][
+            "schemaDrift"
+        ] = "unexpected"
+        city_drift = json.dumps(
+            city_document,
+            ensure_ascii=False,
+            separators=(",", ":"),
+        ).encode("utf-8")
+        outcome = collect_and_stage_fixture(
+            departure_pages=(official_page([official_row()]),),
+            arrival_pages=(
+                official_page(
+                    [
+                        official_row(
+                            flightId="KE704",
+                            masterFlightId="KE704",
+                            st="2015",
+                        )
+                    ]
+                ),
+            ),
+            city_payload=city_drift,
+            legacy_arrival_pages=(
+                legacy_arrival_page([legacy_arrival_row()]),
+            ),
+            duration_csv=(
+                b"destination_iata,outbound_minutes,inbound_minutes,reference_date,"
+                b"reference_locator\r\nNRT,150,165,2026-08-01,"
+                + DURATION_SOURCE_LOCATOR.encode("ascii")
+                + b"\r\n"
+            ),
+            source_date=date(2026, 8, 31),
+            source_checked_at=timezone.now(),
+        )
+
+        self.assertEqual(
+            outcome.code, FlightIngestionCode.PARSE_QUARANTINED
+        )
+        city_artifact = SourceArtifact.objects.get(
+            source__code=CITY_SOURCE_CODE
+        )
+        self.assertEqual(
+            city_artifact.body_sha256,
+            hashlib.sha256(city_drift).hexdigest(),
+        )
+        self.assertEqual(city_artifact.state, SourceArtifact.State.REJECTED)
+        city_run = ParseRun.objects.get(
+            parser_name=ParseRun.ParserName.ICN_DESTINATION_CITY_JSON
+        )
+        self.assertEqual(city_run.outcome, ParseRun.Outcome.QUARANTINED)
+        self.assertEqual(
+            city_run.failure_code, ParseRun.FailureCode.SCHEMA_MISMATCH
+        )
+        self.assertFalse(FlightScheduleRevision.objects.exists())
+        self.assertFalse(
+            PublishedFlightSchedule.objects.filter(
+                current_publication__isnull=False
+            ).exists()
+        )
+
+    def test_schedule_typed_schema_drift_closes_the_ordered_parse_run(self):
+        departure_document = json.loads(official_page([official_row()]))
+        departure_document["response"]["body"]["items"]["item"][0][
+            "schemaDrift"
+        ] = "unexpected"
+        departure_drift = json.dumps(
+            departure_document,
+            ensure_ascii=False,
+            separators=(",", ":"),
+        ).encode("utf-8")
+
+        outcome = collect_and_stage_fixture(
+            departure_pages=(departure_drift,),
+            arrival_pages=(
+                official_page(
+                    [
+                        official_row(
+                            flightId="KE704",
+                            masterFlightId="KE704",
+                            st="2015",
+                        )
+                    ]
+                ),
+            ),
+            city_payload=city_reference_payload(),
+            legacy_arrival_pages=(
+                legacy_arrival_page([legacy_arrival_row()]),
+            ),
+            duration_csv=(
+                b"destination_iata,outbound_minutes,inbound_minutes,reference_date,"
+                b"reference_locator\r\nNRT,150,165,2026-08-01,"
+                + DURATION_SOURCE_LOCATOR.encode("ascii")
+                + b"\r\n"
+            ),
+            source_date=date(2026, 8, 31),
+            source_checked_at=timezone.now(),
+        )
+
+        self.assertEqual(
+            outcome.code, FlightIngestionCode.PARSE_QUARANTINED
+        )
+        schedule_run = ParseRun.objects.get(
+            parser_name=ParseRun.ParserName.ICN_FLIGHT_SCHEDULE_JSON
+        )
+        self.assertEqual(schedule_run.outcome, ParseRun.Outcome.QUARANTINED)
+        self.assertEqual(
+            schedule_run.ordered_inputs.count(),
+            2,
+        )
+        self.assertEqual(
+            set(
+                SourceArtifact.objects.filter(
+                    source__code=SCHEDULE_SOURCE_CODE
+                ).values_list("state", flat=True)
+            ),
+            {SourceArtifact.State.REJECTED},
+        )
+        self.assertFalse(FlightScheduleRevision.objects.exists())
+
     def test_unknown_airport_schedule_is_quarantined_without_pointer_change(self):
         departure = official_page(
             [official_row(airportCode="AAA", airport="알 수 없는 공항")]
@@ -577,7 +946,7 @@ class FlightEvidencePublicationTests(TransactionTestCase):
             + DURATION_SOURCE_LOCATOR.encode("ascii")
             + b"\r\n"
         )
-        outcome = ingest_flight_evidence_candidate(
+        outcome = collect_and_stage_fixture(
             departure_pages=(departure,),
             arrival_pages=(arrival,),
             city_payload=city_reference_payload(),
@@ -627,7 +996,7 @@ class FlightEvidencePublicationTests(TransactionTestCase):
         )
         for city_payload, legacy_page in cases:
             with self.subTest(city_sha=len(city_payload), legacy_sha=len(legacy_page)):
-                outcome = ingest_flight_evidence_candidate(
+                outcome = collect_and_stage_fixture(
                     departure_pages=(departure,),
                     arrival_pages=(arrival,),
                     city_payload=city_payload,
@@ -682,7 +1051,7 @@ class FlightEvidencePublicationTests(TransactionTestCase):
             + DURATION_SOURCE_LOCATOR.encode("ascii")
             + b"\r\n"
         )
-        outcome = ingest_flight_evidence_candidate(
+        outcome = collect_and_stage_fixture(
             departure_pages=(departure,),
             arrival_pages=(arrival,),
             city_payload=city_reference_payload(
@@ -710,7 +1079,7 @@ class FlightEvidencePublicationTests(TransactionTestCase):
         self.assertEqual(FlightSchedule.objects.count(), 2)
 
     def test_curated_route_without_both_directions_is_quarantined(self):
-        outcome = ingest_flight_evidence_candidate(
+        outcome = collect_and_stage_fixture(
             departure_pages=(official_page([official_row()]),),
             arrival_pages=(
                 official_page(


