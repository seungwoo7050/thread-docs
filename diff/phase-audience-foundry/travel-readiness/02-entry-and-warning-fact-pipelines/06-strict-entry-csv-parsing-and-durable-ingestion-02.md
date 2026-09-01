## `feat(entry): ingest approved snapshot evidence`

diff --git a/entry_requirements/ingestion.py b/entry_requirements/ingestion.py
new file mode 100644
index 0000000..fab359a
--- /dev/null
+++ b/entry_requirements/ingestion.py
@@ -0,0 +1,834 @@
+"""Closed, redacted ingestion boundary for the approved entry snapshot.
+
+The transport is deliberately invoked only after its STARTED receipt commits.
+Successful bytes remain in local memory just long enough to verify, parse, and
+project them; only hashes, byte counts, parse evidence, and typed facts persist.
+"""
+
+from __future__ import annotations
+
+import hashlib
+import time
+import uuid
+from dataclasses import dataclass, field
+from typing import Callable
+
+from django.db import connection, transaction
+from django.utils import timezone
+
+from countries.models import JP_COUNTRY_ID, Country
+from sources.models import (
+    FetchAttempt,
+    ParseRun,
+    SourceArtifact,
+    SourceConfiguration,
+    SourceRightsDecision,
+)
+from sources.transport import (
+    ATTEMPT_RETRYABLE_FAILED,
+    ATTEMPT_SUCCEEDED,
+    ATTEMPT_TERMINAL_FAILED,
+    ENTRY_MAX_RESPONSE_BYTES,
+    ENTRY_REQUEST_FINGERPRINT,
+    ENTRY_SOURCE_LOCATOR,
+    FAILURE_AUTHENTICATION,
+    FAILURE_HTTP_CLIENT,
+    FAILURE_RATE_LIMITED,
+    FAILURE_RESPONSE_TOO_LARGE,
+    FAILURE_TIMEOUT,
+    FAILURE_TRANSPORT,
+    FAILURE_UPSTREAM_5XX,
+    SingleAttemptResult,
+    fetch_entry_attachment,
+)
+
+from .models import (
+    PASSPORT_POLICY_CODE,
+    PASSPORT_POLICY_ID,
+    PASSPORT_POLICY_REVISION,
+    EntryFactRevision,
+    PassportPolicy,
+)
+from .parser import (
+    ENTRY_SCHEMA_FINGERPRINT_SHA256,
+    EntryParseResult,
+    EntryProjection,
+    parse_entry_csv,
+)
+
+
+ENTRY_SOURCE_CODE = "MOFA_ENTRY_CSV"
+ENTRY_SOURCE_REVISION = "rights-v1"
+ENTRY_SOURCE_MODULE = SourceConfiguration.Module.ENTRY
+ENTRY_SOURCE_OWNER = "대한민국 외교부 정보화담당관실"
+ENTRY_FIELD_SCOPE = "JP_ORDINARY_TEXT_V1"
+ENTRY_ATTRIBUTION = "외교부|공공데이터포털"
+ENTRY_CONTRACT_FINGERPRINT_SHA256 = (
+    "622a399317ab730f9e9780f51b3ac837073cd99939f07f248323636e97676021"
+)
+ENTRY_DECIDED_BY = "PROJECT_OWNER_REQUEST"
+ENTRY_DECISION_BASIS = "USER_DIRECTIVE_20260830"
+ENTRY_PARSER_NAME = ParseRun.ParserName.MOFA_ENTRY_CSV
+ENTRY_PARSER_VERSION = ParseRun.ParserVersion.V1
+ENTRY_INGESTION_LOCK_NAMESPACE = 1_414_678_614
+ENTRY_INGESTION_LOCK_KEY = 1_006_001
+
+
+class EntryIngestionCode:
+    REVIEW_REQUIRED = "REVIEW_REQUIRED"
+    REPLAY_OBSERVED = "REPLAY_OBSERVED"
+    PARSE_QUARANTINED = "PARSE_QUARANTINED"
+    PARSE_FAILED = "PARSE_FAILED"
+    FETCH_TERMINAL_FAILED = "FETCH_TERMINAL_FAILED"
+    FETCH_RETRIES_EXHAUSTED = "FETCH_RETRIES_EXHAUSTED"
+    SOURCE_GATE_FAILED = "SOURCE_GATE_FAILED"
+    EVIDENCE_CONFLICT = "EVIDENCE_CONFLICT"
+    PERSISTENCE_FAILED = "PERSISTENCE_FAILED"
+    ALREADY_RUNNING = "ALREADY_RUNNING"
+
+
+SUCCESS_CODES = frozenset(
+    {
+        EntryIngestionCode.REVIEW_REQUIRED,
+        EntryIngestionCode.REPLAY_OBSERVED,
+    }
+)
+
+
+@dataclass(frozen=True, slots=True)
+class EntryIngestionOutcome:
+    code: str
+    attempt_count: int
+
+    @property
+    def succeeded(self) -> bool:
+        return self.code in SUCCESS_CODES
+
+
+@dataclass(frozen=True, slots=True)
+class _FetchConfiguration:
+    source_id: uuid.UUID
+    rights_id: uuid.UUID
+    connect_timeout_seconds: int
+    read_timeout_seconds: int
+    max_retries: int
+
+
+@dataclass(frozen=True, slots=True)
+class _VerifiedTransportResult:
+    attempt_result: str
+    http_status: int | None
+    failure_code: str
+    response_sha256: str
+    byte_count: int
+    body: bytes = field(repr=False, compare=False)
+
+
+@dataclass(frozen=True, slots=True)
+class _SanitizedInterruption:
+    kind: str
+
+
+class _ClosedPersistenceFailure(Exception):
+    def __init__(self, code: str):
+        self.code = code
+        super().__init__(code)
+
+
+def _try_acquire_entry_ingestion_lock() -> bool:
+    if connection.vendor != "postgresql" or connection.in_atomic_block:
+        raise RuntimeError("entry ingestion requires an autocommit PostgreSQL session")
+    with connection.cursor() as cursor:
+        cursor.execute(
+            "SELECT pg_try_advisory_lock(%s, %s)",
+            [ENTRY_INGESTION_LOCK_NAMESPACE, ENTRY_INGESTION_LOCK_KEY],
+        )
+        row = cursor.fetchone()
+    if not row or type(row[0]) is not bool:
+        raise RuntimeError("invalid advisory lock result")
+    return row[0]
+
+
+def _release_entry_ingestion_lock() -> None:
+    try:
+        with connection.cursor() as cursor:
+            cursor.execute(
+                "SELECT pg_advisory_unlock(%s, %s)",
+                [ENTRY_INGESTION_LOCK_NAMESPACE, ENTRY_INGESTION_LOCK_KEY],
+            )
+            row = cursor.fetchone()
+        if not row or row[0] is not True:
+            connection.close()
+    except BaseException:
+        try:
+            connection.close()
+        except BaseException:
+            pass
+
+
+def _discard_entry_ingestion_connection() -> None:
+    try:
+        connection.close()
+    except BaseException:
+        pass
+
+
+def _base_exception_kind(exception: BaseException) -> str:
+    if isinstance(exception, KeyboardInterrupt):
+        return "KEYBOARD_INTERRUPT"
+    if isinstance(exception, SystemExit):
+        return "SYSTEM_EXIT"
+    return "BASE_EXCEPTION"
+
+
+def _raise_sanitized_base_exception(kind: str) -> None:
+    if kind == "KEYBOARD_INTERRUPT":
+        raise KeyboardInterrupt() from None
+    if kind == "SYSTEM_EXIT":
+        raise SystemExit(1) from None
+    raise BaseException() from None
+
+
+def _close_abandoned_started_attempts() -> None:
+    with transaction.atomic(durable=True):
+        abandoned = list(
+            FetchAttempt.objects.select_for_update()
+            .filter(
+                source__code=ENTRY_SOURCE_CODE,
+                result=FetchAttempt.Result.STARTED,
+            )
+            .only("id", "started_at")
+        )
+        for attempt in abandoned:
+            completed_at = max(timezone.now(), attempt.started_at)
+            updated = FetchAttempt.objects.filter(
+                pk=attempt.id,
+                result=FetchAttempt.Result.STARTED,
+            ).update(
+                result=FetchAttempt.Result.TERMINAL_FAILED,
+                completed_at=completed_at,
+                http_status=None,
+                provider_code="",
+                response_sha256="",
+                failure_code=FetchAttempt.FailureCode.WORKER_INTERRUPTED,
+            )
+            if updated != 1:
+                raise _ClosedPersistenceFailure(
+                    EntryIngestionCode.PERSISTENCE_FAILED
+                )
+
+
+def _bounded_retry_wait(attempt_number: int) -> None:
+    time.sleep(min(2 ** (attempt_number - 1), 4))
+
+
+def _latest_rights(source: SourceConfiguration) -> SourceRightsDecision | None:
+    return (
+        SourceRightsDecision.objects.select_for_update()
+        .filter(source=source, source_revision=source.revision)
+        .order_by("-decision_sequence", "-id")
+        .first()
+    )
+
+
+def _rights_are_exact(rights: SourceRightsDecision | None) -> bool:
+    return bool(
+        rights is not None
+        and rights.source_revision == ENTRY_SOURCE_REVISION
+        and rights.decision_sequence == 1
+        and rights.decision == SourceRightsDecision.Decision.APPROVED
+        and rights.access_mode
+        == SourceRightsDecision.AccessMode.ANONYMOUS_PUBLIC
+        and rights.access_allowed
+        and rights.automated_collection_allowed
+        and rights.typed_field_storage_allowed
+        and not rights.raw_body_storage_allowed
+        and rights.typed_republication_allowed
+        and rights.raw_retention_seconds == 0
+        and rights.typed_retention
+        == SourceRightsDecision.Retention.PRODUCT_HISTORY
+        and rights.evidence_retention
+        == SourceRightsDecision.Retention.PRODUCT_HISTORY
+        and rights.field_scope_code == ENTRY_FIELD_SCOPE
+        and rights.attribution_text == ENTRY_ATTRIBUTION
+        and rights.contract_fingerprint_sha256
+        == ENTRY_CONTRACT_FINGERPRINT_SHA256
+        and rights.decided_by == ENTRY_DECIDED_BY
+        and rights.decision_basis_code == ENTRY_DECISION_BASIS
+    )
+
+
+def _source_is_exact(source: SourceConfiguration) -> bool:
+    return bool(
+        source.code == ENTRY_SOURCE_CODE
+        and source.revision == ENTRY_SOURCE_REVISION
+        and source.module == ENTRY_SOURCE_MODULE
+        and source.owner == ENTRY_SOURCE_OWNER
+        and source.official_locator == ENTRY_SOURCE_LOCATOR
+        and source.state == SourceConfiguration.State.ACTIVE
+        and source.enabled
+        and source.secret_reference_name == ""
+        and source.connect_timeout_seconds == 5
+        and source.read_timeout_seconds == 15
+        and source.max_retries == 2
+    )
+
+
+def _locked_approved_source() -> tuple[SourceConfiguration, SourceRightsDecision]:
+    source = (
+        SourceConfiguration.objects.select_for_update()
+        .filter(code=ENTRY_SOURCE_CODE)
+        .first()
+    )
+    if source is None or not _source_is_exact(source):
+        raise _ClosedPersistenceFailure(EntryIngestionCode.SOURCE_GATE_FAILED)
+    rights = _latest_rights(source)
+    if not _rights_are_exact(rights):
+        raise _ClosedPersistenceFailure(EntryIngestionCode.SOURCE_GATE_FAILED)
+    return source, rights
+
+
+def _create_started_attempt(
+    operation_id: uuid.UUID,
+    attempt_number: int,
+) -> tuple[FetchAttempt, _FetchConfiguration]:
+    with transaction.atomic(durable=True):
+        source, rights = _locked_approved_source()
+        attempt = FetchAttempt.objects.create(
+            source=source,
+            source_revision=source.revision,
+            rights_decision=rights,
+            operation_id=operation_id,
+            attempt_number=attempt_number,
+            request_fingerprint_revision=ENTRY_REQUEST_FINGERPRINT.revision,
+            normalized_request_sha256=(
+                ENTRY_REQUEST_FINGERPRINT.normalized_request_sha256
+            ),
+        )
+        configuration = _FetchConfiguration(
+            source_id=source.id,
+            rights_id=rights.id,
+            connect_timeout_seconds=source.connect_timeout_seconds,
+            read_timeout_seconds=source.read_timeout_seconds,
+            max_retries=source.max_retries,
+        )
+    return attempt, configuration
+
+
+def _worker_interrupted_result() -> _VerifiedTransportResult:
+    return _VerifiedTransportResult(
+        attempt_result=ATTEMPT_TERMINAL_FAILED,
+        http_status=None,
+        failure_code=FetchAttempt.FailureCode.WORKER_INTERRUPTED,
+        response_sha256="",
+        byte_count=0,
+        body=b"",
+    )
+
+
+def _verify_transport_result(result: object) -> _VerifiedTransportResult:
+    if not isinstance(result, SingleAttemptResult):
+        return _worker_interrupted_result()
+    if (
+        type(result.request_fingerprint) is not type(ENTRY_REQUEST_FINGERPRINT)
+        or result.request_fingerprint.revision
+        != ENTRY_REQUEST_FINGERPRINT.revision
+        or result.request_fingerprint.normalized_request_sha256
+        != ENTRY_REQUEST_FINGERPRINT.normalized_request_sha256
+        or type(result.attempt_result) is not str
+        or (
+            result.http_status is not None
+            and (
+                type(result.http_status) is not int
+                or not 100 <= result.http_status <= 599
+            )
+        )
+        or type(result.provider_code) is not str
+        or type(result.response_sha256) is not str
+        or type(result.byte_count) is not int
+        or result.byte_count < 0
+        or type(result.failure_code) is not str
+        or type(result.body) is not bytes
+    ):
+        return _worker_interrupted_result()
+    if result.provider_code != "":
+        return _worker_interrupted_result()
+
+    if result.attempt_result == ATTEMPT_SUCCEEDED:
+        if (
+            result.http_status != 200
+            or result.failure_code != ""
+            or result.byte_count != len(result.body)
+            or result.byte_count > ENTRY_MAX_RESPONSE_BYTES
+            or result.response_sha256
+            != hashlib.sha256(result.body).hexdigest()
+        ):
+            return _worker_interrupted_result()
+        return _VerifiedTransportResult(
+            attempt_result=result.attempt_result,
+            http_status=result.http_status,
+            failure_code="",
+            response_sha256=result.response_sha256,
+            byte_count=result.byte_count,
+            body=result.body,
+        )
+
+    if (
+        result.body != b""
+        or result.byte_count != 0
+        or result.response_sha256 != ""
+    ):
+        return _worker_interrupted_result()
+    retry_shapes = {
+        FAILURE_TIMEOUT: result.http_status is None,
+        FAILURE_RATE_LIMITED: result.http_status == 429,
+        FAILURE_UPSTREAM_5XX: bool(
+            type(result.http_status) is int
+            and 500 <= result.http_status <= 599
+        ),
+    }
+    if result.attempt_result == ATTEMPT_RETRYABLE_FAILED:
+        if not retry_shapes.get(result.failure_code, False):
+            return _worker_interrupted_result()
+    elif result.attempt_result == ATTEMPT_TERMINAL_FAILED:
+        terminal_shapes = {
+            FAILURE_AUTHENTICATION: result.http_status in (401, 403),
+            FAILURE_HTTP_CLIENT: bool(
+                type(result.http_status) is int
+                and result.http_status != 200
+                and result.http_status not in (401, 403, 429)
+                and not 500 <= result.http_status <= 599
+            ),
+            FAILURE_TRANSPORT: result.http_status is None,
+            FAILURE_RESPONSE_TOO_LARGE: result.http_status == 200,
+        }
+        if not terminal_shapes.get(result.failure_code, False):
+            return _worker_interrupted_result()
+    else:
+        return _worker_interrupted_result()
+    return _VerifiedTransportResult(
+        attempt_result=result.attempt_result,
+        http_status=result.http_status,
+        failure_code=result.failure_code,
+        response_sha256="",
+        byte_count=0,
+        body=b"",
+    )
+
+
+def _close_attempt(
+    attempt_id: uuid.UUID,
+    result: _VerifiedTransportResult,
+) -> FetchAttempt:
+    with transaction.atomic(durable=True):
+        attempt = FetchAttempt.objects.select_for_update().get(pk=attempt_id)
+        if attempt.result != FetchAttempt.Result.STARTED:
+            raise _ClosedPersistenceFailure(
+                EntryIngestionCode.PERSISTENCE_FAILED
+            )
+        updated = FetchAttempt.objects.filter(
+            pk=attempt.id,
+            result=FetchAttempt.Result.STARTED,
+        ).update(
+            result=result.attempt_result,
+            completed_at=timezone.now(),
+            http_status=result.http_status,
+            provider_code="",
+            response_sha256=result.response_sha256,
+            failure_code=result.failure_code,
+        )
+        if updated != 1:
+            raise _ClosedPersistenceFailure(
+                EntryIngestionCode.PERSISTENCE_FAILED
+            )
+        attempt.refresh_from_db()
+    return attempt
+
+
+def _safe_parse(
+    body: bytes,
+    parser: Callable[[bytes], EntryParseResult],
+) -> EntryParseResult:
+    try:
+        result = parser(body)
+    except Exception:
+        result = None
+    if not isinstance(result, EntryParseResult):
+        return EntryParseResult(
+            outcome=ParseRun.Outcome.FAILED,
+            failure_code=ParseRun.FailureCode.INTERNAL_PARSER_ERROR,
+            observed_schema_fingerprint_sha256="",
+            projection=None,
+        )
+    if result.outcome == ParseRun.Outcome.VALIDATED:
+        projection = result.projection
+        if (
+            result.failure_code == ""
+            and result.observed_schema_fingerprint_sha256
+            == ENTRY_SCHEMA_FINGERPRINT_SHA256
+            and isinstance(projection, EntryProjection)
+            and projection.country_iso2 == "JP"
+            and projection.passport_policy_code == PASSPORT_POLICY_CODE
+            and projection.passport_policy_revision == PASSPORT_POLICY_REVISION
+        ):
+            return result
+    elif result.outcome == ParseRun.Outcome.QUARANTINED:
+        if result.projection is None and result.failure_code in {
+            ParseRun.FailureCode.SCHEMA_MISMATCH,
+            ParseRun.FailureCode.IDENTITY_MISMATCH,
+            ParseRun.FailureCode.REQUIRED_VALUE_MISSING,
+            ParseRun.FailureCode.DUPLICATE_RECORD,
+            ParseRun.FailureCode.ENCODING_ERROR,
+            ParseRun.FailureCode.SYNTAX_ERROR,
+            ParseRun.FailureCode.INVALID_VALUE,
+            ParseRun.FailureCode.CONFLICTING_VALUE,
+        }:
+            return result
+    elif (
+        result.outcome == ParseRun.Outcome.FAILED
+        and result.failure_code == ParseRun.FailureCode.INTERNAL_PARSER_ERROR
+        and result.observed_schema_fingerprint_sha256 == ""
+        and result.projection is None
+    ):
+        return result
+    return EntryParseResult(
+        outcome=ParseRun.Outcome.FAILED,
+        failure_code=ParseRun.FailureCode.INTERNAL_PARSER_ERROR,
+        observed_schema_fingerprint_sha256="",
+        projection=None,
+    )
+
+
+def _replay_is_consistent(artifact: SourceArtifact, byte_count: int) -> bool:
+    if (
+        artifact.byte_count != byte_count
+        or artifact.state != SourceArtifact.State.REVIEW_REQUIRED
+    ):
+        return False
+    runs = list(
+        ParseRun.objects.select_for_update().filter(
+            artifact=artifact,
+            parser_name=ENTRY_PARSER_NAME,
+            parser_version=ENTRY_PARSER_VERSION,
+        )
+    )
+    if len(runs) != 1:
+        return False
+    run = runs[0]
+    return bool(
+        run.outcome == ParseRun.Outcome.VALIDATED
+        and run.parser_contract_fingerprint_sha256
+        == ENTRY_CONTRACT_FINGERPRINT_SHA256
+        and run.expected_schema_fingerprint_sha256
+        == ENTRY_SCHEMA_FINGERPRINT_SHA256
+        and run.observed_schema_fingerprint_sha256
+        == ENTRY_SCHEMA_FINGERPRINT_SHA256
+        and EntryFactRevision.objects.filter(parse_run=run).count() == 1
+    )
+
+
+def _persist_success(
+    attempt: FetchAttempt,
+    verified: _VerifiedTransportResult,
+    parser: Callable[[bytes], EntryParseResult],
+) -> str:
+    try:
+        with transaction.atomic(durable=True):
+            source, rights = _locked_approved_source()
+            locked_attempt = FetchAttempt.objects.select_for_update().get(
+                pk=attempt.id
+            )
+            if (
+                locked_attempt.source_id != source.id
+                or locked_attempt.rights_decision_id != rights.id
+                or locked_attempt.source_revision != source.revision
+                or locked_attempt.result != FetchAttempt.Result.SUCCEEDED
+                or locked_attempt.response_sha256 != verified.response_sha256
+                or verified.byte_count != len(verified.body)
+                or verified.response_sha256
+                != hashlib.sha256(verified.body).hexdigest()
+            ):
+                raise _ClosedPersistenceFailure(
+                    EntryIngestionCode.EVIDENCE_CONFLICT
+                )
+
+            with connection.cursor() as cursor:
+                cursor.execute(
+                    "SELECT pg_advisory_xact_lock(hashtextextended(%s, 1))",
+                    [f"{source.id}:{verified.response_sha256}"],
+                )
+            existing = (
+                SourceArtifact.objects.select_for_update()
+                .filter(
+                    source=source,
+                    body_sha256=verified.response_sha256,
+                )
+                .first()
+            )
+            if existing is not None:
+                if not _replay_is_consistent(existing, verified.byte_count):
+                    raise _ClosedPersistenceFailure(
+                        EntryIngestionCode.EVIDENCE_CONFLICT
+                    )
+                return EntryIngestionCode.REPLAY_OBSERVED
+
+            anchor = (
+                FetchAttempt.objects.select_for_update()
+                .filter(
+                    source=source,
+                    result=FetchAttempt.Result.SUCCEEDED,
+                    response_sha256=verified.response_sha256,
+                )
+                .order_by("completed_at", "started_at", "id")
+                .first()
+            )
+            if anchor is None:
+                raise _ClosedPersistenceFailure(
+                    EntryIngestionCode.EVIDENCE_CONFLICT
+                )
+            artifact = SourceArtifact.objects.create(
+                source=source,
+                body_sha256=verified.response_sha256,
+                byte_count=verified.byte_count,
+                first_successful_attempt=anchor,
+            )
+            parse_run = ParseRun.objects.create(
+                artifact=artifact,
+                parser_name=ENTRY_PARSER_NAME,
+                parser_version=ENTRY_PARSER_VERSION,
+                parser_contract_fingerprint_sha256=(
+                    ENTRY_CONTRACT_FINGERPRINT_SHA256
+                ),
+                expected_schema_fingerprint_sha256=(
+                    ENTRY_SCHEMA_FINGERPRINT_SHA256
+                ),
+            )
+            parse_result = _safe_parse(verified.body, parser)
+            completed_at = timezone.now()
+            updated = ParseRun.objects.filter(
+                pk=parse_run.id,
+                outcome=ParseRun.Outcome.STARTED,
+            ).update(
+                outcome=parse_result.outcome,
+                completed_at=completed_at,
+                failure_code=parse_result.failure_code,
+                observed_schema_fingerprint_sha256=(
+                    parse_result.observed_schema_fingerprint_sha256
+                ),
+            )
+            if updated != 1:
+                raise _ClosedPersistenceFailure(
+                    EntryIngestionCode.PERSISTENCE_FAILED
+                )
+
+            if parse_result.outcome != ParseRun.Outcome.VALIDATED:
+                updated = SourceArtifact.objects.filter(
+                    pk=artifact.id,
+                    state=SourceArtifact.State.RECEIVED,
+                ).update(state=SourceArtifact.State.REJECTED)
+                if updated != 1:
+                    raise _ClosedPersistenceFailure(
+                        EntryIngestionCode.PERSISTENCE_FAILED
+                    )
+                if parse_result.outcome == ParseRun.Outcome.QUARANTINED:
+                    return EntryIngestionCode.PARSE_QUARANTINED
+                return EntryIngestionCode.PARSE_FAILED
+
+            projection = parse_result.projection
+            if projection is None:
+                raise _ClosedPersistenceFailure(
+                    EntryIngestionCode.PERSISTENCE_FAILED
+                )
+            updated = SourceArtifact.objects.filter(
+                pk=artifact.id,
+                state=SourceArtifact.State.RECEIVED,
+            ).update(state=SourceArtifact.State.REVIEW_REQUIRED)
+            if updated != 1:
+                raise _ClosedPersistenceFailure(
+                    EntryIngestionCode.PERSISTENCE_FAILED
+                )
+
+            # This second exact/latest check sits immediately before the typed
+            # insert.  Holding the source row lock also serializes rejection.
+            current_source, current_rights = _locked_approved_source()
+            if current_source.id != source.id or current_rights.id != rights.id:
+                raise _ClosedPersistenceFailure(
+                    EntryIngestionCode.SOURCE_GATE_FAILED
+                )
+            country = Country.objects.get(pk=JP_COUNTRY_ID)
+            policy = PassportPolicy.objects.get(pk=PASSPORT_POLICY_ID)
+            EntryFactRevision.objects.create(
+                country=country,
+                passport_policy=policy,
+                parse_run=parse_run,
+                ordinary_passport_period_text=(
+                    projection.ordinary_passport_period_text
+                ),
+                basis_text=projection.basis_text,
+                snapshot_date=projection.snapshot_date,
+                first_observed_at=anchor.completed_at,
+                typed_fingerprint_sha256=(
+                    projection.typed_fingerprint_sha256
+                ),
+            )
+            return EntryIngestionCode.REVIEW_REQUIRED
+    except _ClosedPersistenceFailure:
+        raise
+    except Exception:
+        raise _ClosedPersistenceFailure(
+            EntryIngestionCode.PERSISTENCE_FAILED
+        ) from None
+
+
+def _run_entry_snapshot_ingestion(
+    *,
+    transport: Callable[..., SingleAttemptResult],
+    parser: Callable[[bytes], EntryParseResult],
+    retry_wait: Callable[[int], None],
+) -> EntryIngestionOutcome | _SanitizedInterruption:
+    """Run one redacted operation, including only its bounded permitted retries."""
+
+    try:
+        lock_acquired = _try_acquire_entry_ingestion_lock()
+    except Exception:
+        _discard_entry_ingestion_connection()
+        return EntryIngestionOutcome(EntryIngestionCode.PERSISTENCE_FAILED, 0)
+    except BaseException as exception:
+        exception_kind = _base_exception_kind(exception)
+        del exception
+        _discard_entry_ingestion_connection()
+        return _SanitizedInterruption(exception_kind)
+    if not lock_acquired:
+        return EntryIngestionOutcome(EntryIngestionCode.ALREADY_RUNNING, 0)
+    try:
+        try:
+            _close_abandoned_started_attempts()
+        except Exception:
+            return EntryIngestionOutcome(
+                EntryIngestionCode.PERSISTENCE_FAILED,
+                0,
+            )
+
+        operation_id = uuid.uuid4()
+        attempt_count = 0
+        max_retries: int | None = None
+        while True:
+            attempt_count += 1
+            try:
+                attempt, configuration = _create_started_attempt(
+                    operation_id,
+                    attempt_count,
+                )
+            except _ClosedPersistenceFailure as failure:
+                return EntryIngestionOutcome(failure.code, attempt_count - 1)
+            except Exception:
+                return EntryIngestionOutcome(
+                    EntryIngestionCode.PERSISTENCE_FAILED,
+                    attempt_count - 1,
+                )
+            if max_retries is None:
+                max_retries = configuration.max_retries
+
+            interruption_kind: str | None = None
+            try:
+                raw_result = transport(
+                    official_locator=ENTRY_SOURCE_LOCATOR,
+                    connect_timeout_seconds=(
+                        configuration.connect_timeout_seconds
+                    ),
+                    read_timeout_seconds=configuration.read_timeout_seconds,
+                )
+            except Exception:
+                raw_result = None
+            except BaseException as exception:
+                interruption_kind = _base_exception_kind(exception)
+                del exception
+                raw_result = None
+
+            try:
+                verified = _verify_transport_result(raw_result)
+            except Exception:
+                verified = _worker_interrupted_result()
+            except BaseException as exception:
+                interruption_kind = _base_exception_kind(exception)
+                del exception
+                verified = _worker_interrupted_result()
+            try:
+                closed_attempt = _close_attempt(attempt.id, verified)
+            except BaseException as exception:
+                exception_kind = _base_exception_kind(exception)
+                del exception
+                try:
+                    _close_abandoned_started_attempts()
+                except BaseException:
+                    pass
+                if exception_kind != "BASE_EXCEPTION":
+                    raw_result = None
+                    verified = _worker_interrupted_result()
+                    return _SanitizedInterruption(exception_kind)
+                return EntryIngestionOutcome(
+                    EntryIngestionCode.PERSISTENCE_FAILED,
+                    attempt_count,
+                )
+            if interruption_kind is not None:
+                raw_result = None
+                verified = _worker_interrupted_result()
+                return _SanitizedInterruption(interruption_kind)
+
+            if verified.attempt_result == ATTEMPT_SUCCEEDED:
+                try:
+                    code = _persist_success(closed_attempt, verified, parser)
+                except _ClosedPersistenceFailure as failure:
+                    code = failure.code
+                return EntryIngestionOutcome(code, attempt_count)
+
+            if verified.attempt_result != ATTEMPT_RETRYABLE_FAILED:
+                return EntryIngestionOutcome(
+                    EntryIngestionCode.FETCH_TERMINAL_FAILED,
+                    attempt_count,
+                )
+            if max_retries is None or attempt_count > max_retries:
+                return EntryIngestionOutcome(
+                    EntryIngestionCode.FETCH_RETRIES_EXHAUSTED,
+                    attempt_count,
+                )
+            try:
+                retry_wait(attempt_count)
+            except Exception:
+                return EntryIngestionOutcome(
+                    EntryIngestionCode.PERSISTENCE_FAILED,
+                    attempt_count,
+                )
+    except BaseException as exception:
+        exception_kind = _base_exception_kind(exception)
+        del exception
+        try:
+            _close_abandoned_started_attempts()
+        except BaseException:
+            pass
+        raw_result = None
+        verified = None
+        interruption_kind = None
+        return _SanitizedInterruption(exception_kind)
+    finally:
+        _release_entry_ingestion_lock()
+
+
+def ingest_entry_snapshot(
+    *,
+    transport: Callable[..., SingleAttemptResult] = fetch_entry_attachment,
+    parser: Callable[[bytes], EntryParseResult] = parse_entry_csv,
+    retry_wait: Callable[[int], None] = _bounded_retry_wait,
+) -> EntryIngestionOutcome:
+    """Run one redacted operation, including only its bounded permitted retries."""
+
+    result = _run_entry_snapshot_ingestion(
+        transport=transport,
+        parser=parser,
+        retry_wait=retry_wait,
+    )
+    if isinstance(result, _SanitizedInterruption):
+        interruption_kind = result.kind
+        result = None
+        _raise_sanitized_base_exception(interruption_kind)
+    return result
diff --git a/entry_requirements/tests/test_ingestion.py b/entry_requirements/tests/test_ingestion.py
new file mode 100644
index 0000000..a17d7d0
--- /dev/null
+++ b/entry_requirements/tests/test_ingestion.py
@@ -0,0 +1,783 @@
+import csv
+import hashlib
+import inspect
+import io
+import uuid
+from functools import lru_cache
+from unittest.mock import patch
+
+import psycopg
+from django.core.management import call_command
+from django.core.management.base import CommandError
+from django.db import connection, models
+from django.test import SimpleTestCase, TransactionTestCase
+
+from countries.models import JP_COUNTRY_ID, Country
+import entry_requirements.ingestion as entry_ingestion
+from entry_requirements.ingestion import (
+    ENTRY_INGESTION_LOCK_KEY,
+    ENTRY_INGESTION_LOCK_NAMESPACE,
+    EntryIngestionCode,
+    EntryIngestionOutcome,
+    _verify_transport_result,
+    ingest_entry_snapshot,
+)
+from entry_requirements.models import (
+    PASSPORT_POLICY_CODE,
+    PASSPORT_POLICY_ID,
+    PASSPORT_POLICY_REVISION,
+    EntryFactRevision,
+    PassportPolicy,
+)
+from entry_requirements.parser import ENTRY_HEADERS
+from sources.management.commands.register_approved_sources import (
+    register_approved_sources,
+)
+from sources.models import (
+    FetchAttempt,
+    ParseRun,
+    SourceArtifact,
+    SourceConfiguration,
+    SourceRightsDecision,
+)
+from sources.transport import (
+    ATTEMPT_RETRYABLE_FAILED,
+    ATTEMPT_SUCCEEDED,
+    ATTEMPT_TERMINAL_FAILED,
+    ENTRY_REQUEST_FINGERPRINT,
+    ENTRY_SOURCE_LOCATOR,
+    FAILURE_RATE_LIMITED,
+    FAILURE_TIMEOUT,
+    FAILURE_TRANSPORT,
+    FAILURE_UPSTREAM_5XX,
+    SingleAttemptResult,
+)
+
+
+VALIDATION_MARKER_SHA256 = (
+    "18f5384d58bcb1bba0bcd9e6a6781d1a6ac2cc280c330ecbab6cb7931b721552"
+)
+PERIOD_TEXT = "90일"
+BASIS_TEXT = "일반여권 소지자 : synthetic evidence"
+
+
+def assert_exception_boundary_is_redacted(testcase, exception, marker):
+    testcase.assertNotIn(marker, repr(exception))
+    testcase.assertIsNone(exception.__context__)
+    testcase.assertIsNone(exception.__cause__)
+    traceback = exception.__traceback__
+    while traceback is not None:
+        filename = traceback.tb_frame.f_code.co_filename
+        if filename.endswith(
+            (
+                "/entry_requirements/ingestion.py",
+                "/operations/management/commands/ingest_entry_snapshot.py",
+            )
+        ):
+            for value in traceback.tb_frame.f_locals.values():
+                testcase.assertNotIn(marker, repr(value))
+        traceback = traceback.tb_next
+
+
+@lru_cache(maxsize=1)
+def validation_marker() -> str:
+    for codepoint in range(0x110000):
+        if 0xD800 <= codepoint <= 0xDFFF:
+            continue
+        candidate = chr(codepoint)
+        if (
+            hashlib.sha256(candidate.encode("utf-8")).hexdigest()
+            != VALIDATION_MARKER_SHA256
+        ):
+            continue
+        candidate.encode("cp949", errors="strict")
+        return candidate
+    raise AssertionError("approved validation marker could not be reconstructed")
+
+
+def entry_payload(*, headers=ENTRY_HEADERS, period=PERIOD_TEXT) -> bytes:
+    row = [""] * len(ENTRY_HEADERS)
+    row[0] = "일본"
+    row[2] = validation_marker()
+    row[3] = period
+    row[8] = BASIS_TEXT
+    output = io.StringIO(newline="")
+    writer = csv.writer(output, lineterminator="\r\n")
+    writer.writerow(headers)
+    writer.writerow(row)
+    return output.getvalue().encode("cp949", errors="strict")
+
+
+def successful_result(payload: bytes) -> SingleAttemptResult:
+    return SingleAttemptResult(
+        request_fingerprint=ENTRY_REQUEST_FINGERPRINT,
+        attempt_result=ATTEMPT_SUCCEEDED,
+        http_status=200,
+        response_sha256=hashlib.sha256(payload).hexdigest(),
+        byte_count=len(payload),
+        body=payload,
+    )
+
+
+def retryable_result(failure_code: str, status: int | None) -> SingleAttemptResult:
+    return SingleAttemptResult(
+        request_fingerprint=ENTRY_REQUEST_FINGERPRINT,
+        attempt_result=ATTEMPT_RETRYABLE_FAILED,
+        http_status=status,
+        failure_code=failure_code,
+    )
+
+
+def terminal_result() -> SingleAttemptResult:
+    return SingleAttemptResult(
+        request_fingerprint=ENTRY_REQUEST_FINGERPRINT,
+        attempt_result=ATTEMPT_TERMINAL_FAILED,
+        failure_code=FAILURE_TRANSPORT,
+    )
+
+
+class SequenceTransport:
+    def __init__(self, *results):
+        self.results = list(results)
+        self.calls = []
+
+    def __call__(self, **kwargs):
+        if connection.in_atomic_block:
+            raise AssertionError("network adapter called inside a DB transaction")
+        started = list(
+            FetchAttempt.objects.order_by("attempt_number").values_list(
+                "attempt_number", "result"
+            )
+        )
+        if not started or started[-1][1] != FetchAttempt.Result.STARTED:
+            raise AssertionError("STARTED receipt was not durable before transport")
+        self.calls.append((kwargs, tuple(started)))
+        return self.results.pop(0)
+
+
+class EntryIngestionTests(TransactionTestCase):
+    reset_sequences = True
+
+    def setUp(self):
+        Country.objects.get_or_create(
+            id=JP_COUNTRY_ID,
+            defaults={
+                "iso_alpha2": "JP",
+                "name_ko": "일본",
+                "name_en": "Japan",
+                "is_public": True,
+            },
+        )
+        PassportPolicy.objects.get_or_create(
+            id=PASSPORT_POLICY_ID,
+            defaults={
+                "code": PASSPORT_POLICY_CODE,
+                "revision": PASSPORT_POLICY_REVISION,
+            },
+        )
+        register_approved_sources(apply=True)
+
+    def reject_entry_rights(self):
+        source = SourceConfiguration.objects.get(code="MOFA_ENTRY_CSV")
+        approval = source.rights_decisions.get(decision_sequence=1)
+        return SourceRightsDecision.objects.create(
+            source=source,
+            source_revision=source.revision,
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
+            evidence_retention=SourceRightsDecision.Retention.PRODUCT_HISTORY,
+            field_scope_code="",
+            attribution_text="",
+            contract_fingerprint_sha256=(
+                approval.contract_fingerprint_sha256
+            ),
+            decided_by="PROJECT_OWNER_REQUEST",
+            decision_basis_code="SYNTHETIC_REJECTION_TEST",
+        )
+
+    def open_second_session(self):
+        return psycopg.connect(**connection.get_connection_params())
+
+    def test_positive_closes_receipt_and_creates_atomic_review_candidate(self):
+        payload = entry_payload()
+        transport = SequenceTransport(successful_result(payload))
+
+        outcome = ingest_entry_snapshot(transport=transport)
+
+        self.assertEqual(
+            outcome,
+            EntryIngestionOutcome(EntryIngestionCode.REVIEW_REQUIRED, 1),
+        )
+        self.assertTrue(outcome.succeeded)
+        self.assertEqual(len(transport.calls), 1)
+        kwargs, receipts_at_call = transport.calls[0]
+        self.assertEqual(
+            set(kwargs),
+            {
+                "official_locator",
+                "connect_timeout_seconds",
+                "read_timeout_seconds",
+            },
+        )
+        self.assertEqual(kwargs["official_locator"], ENTRY_SOURCE_LOCATOR)
+        self.assertEqual(receipts_at_call, ((1, FetchAttempt.Result.STARTED),))
+
+        attempt = FetchAttempt.objects.get()
+        artifact = SourceArtifact.objects.get()
+        run = ParseRun.objects.get()
+        fact = EntryFactRevision.objects.get()
+        self.assertEqual(attempt.result, FetchAttempt.Result.SUCCEEDED)
+        self.assertEqual(attempt.response_sha256, hashlib.sha256(payload).hexdigest())
+        self.assertEqual(artifact.first_successful_attempt_id, attempt.id)
+        self.assertEqual(artifact.byte_count, len(payload))
+        self.assertEqual(artifact.state, SourceArtifact.State.REVIEW_REQUIRED)
+        self.assertEqual(run.outcome, ParseRun.Outcome.VALIDATED)
+        self.assertEqual(fact.parse_run_id, run.id)
+        self.assertEqual(fact.first_observed_at, attempt.completed_at)
+        self.assertEqual(fact.ordinary_passport_period_text, PERIOD_TEXT)
+        self.assertEqual(fact.basis_text, BASIS_TEXT)
+
+    def test_same_body_replay_adds_only_fresh_attempt_evidence(self):
+        payload = entry_payload()
+        first = ingest_entry_snapshot(
+            transport=SequenceTransport(successful_result(payload))
+        )
+        artifact = SourceArtifact.objects.get()
+        run = ParseRun.objects.get()
+        fact = EntryFactRevision.objects.get()
+
+        second = ingest_entry_snapshot(
+            transport=SequenceTransport(successful_result(payload))
+        )
+
+        self.assertEqual(first.code, EntryIngestionCode.REVIEW_REQUIRED)
+        self.assertEqual(second.code, EntryIngestionCode.REPLAY_OBSERVED)
+        self.assertEqual(FetchAttempt.objects.count(), 2)
+        self.assertEqual(SourceArtifact.objects.count(), 1)
+        self.assertEqual(ParseRun.objects.count(), 1)
+        self.assertEqual(EntryFactRevision.objects.count(), 1)
+        self.assertEqual(SourceArtifact.objects.get().id, artifact.id)
+        self.assertEqual(ParseRun.objects.get().id, run.id)
+        self.assertEqual(EntryFactRevision.objects.get().id, fact.id)
+
+    def test_retry_allowlist_is_contiguous_and_stops_after_success(self):
+        payload = entry_payload()
+        transport = SequenceTransport(
+            retryable_result(FAILURE_TIMEOUT, None),
+            retryable_result(FAILURE_UPSTREAM_5XX, 503),
+            successful_result(payload),
+        )
+
+        waits = []
+        outcome = ingest_entry_snapshot(
+            transport=transport,
+            retry_wait=waits.append,
+        )
+
+        self.assertEqual(
+            outcome,
+            EntryIngestionOutcome(EntryIngestionCode.REVIEW_REQUIRED, 3),
+        )
+        attempts = list(FetchAttempt.objects.order_by("attempt_number"))
+        self.assertEqual([attempt.attempt_number for attempt in attempts], [1, 2, 3])
+        self.assertEqual(len({attempt.operation_id for attempt in attempts}), 1)
+        self.assertEqual(
+            [attempt.result for attempt in attempts],
+            [
+                FetchAttempt.Result.RETRYABLE_FAILED,
+                FetchAttempt.Result.RETRYABLE_FAILED,
+                FetchAttempt.Result.SUCCEEDED,
+            ],
+        )
+        self.assertEqual(
+            [attempt.failure_code for attempt in attempts],
+            [FAILURE_TIMEOUT, FAILURE_UPSTREAM_5XX, ""],
+        )
+        self.assertEqual(waits, [1, 2])
+
+    def test_retryable_outage_is_exhausted_at_max_plus_one(self):
+        transport = SequenceTransport(
+            retryable_result(FAILURE_RATE_LIMITED, 429),
+            retryable_result(FAILURE_TIMEOUT, None),
+            retryable_result(FAILURE_UPSTREAM_5XX, 500),
+        )
+
+        waits = []
+        outcome = ingest_entry_snapshot(
+            transport=transport,
+            retry_wait=waits.append,
+        )
+
+        self.assertEqual(
+            outcome,
+            EntryIngestionOutcome(
+                EntryIngestionCode.FETCH_RETRIES_EXHAUSTED,
+                3,
+            ),
+        )
+        self.assertEqual(FetchAttempt.objects.count(), 3)
+        self.assertFalse(
+            FetchAttempt.objects.filter(result=FetchAttempt.Result.STARTED).exists()
+        )
+        self.assertFalse(SourceArtifact.objects.exists())
+        self.assertEqual(waits, [1, 2])
+
+    def test_overlap_fails_closed_before_receipt_or_transport(self):
+        with patch(
+            "entry_requirements.ingestion._try_acquire_entry_ingestion_lock",
+            return_value=False,
+        ):
+            outcome = ingest_entry_snapshot(
+                transport=lambda **kwargs: self.fail("transport must not run")
+            )
+
+        self.assertEqual(
+            outcome,
+            EntryIngestionOutcome(EntryIngestionCode.ALREADY_RUNNING, 0),
+        )
+        self.assertFalse(FetchAttempt.objects.exists())
+
+    def test_lock_database_failure_is_not_reported_as_overlap(self):
+        with patch(
+            "entry_requirements.ingestion._try_acquire_entry_ingestion_lock",
+            side_effect=RuntimeError("private database detail"),
+        ):
+            outcome = ingest_entry_snapshot(
+                transport=lambda **kwargs: self.fail("transport must not run")
+            )
+
+        self.assertEqual(
+            outcome,
+            EntryIngestionOutcome(EntryIngestionCode.PERSISTENCE_FAILED, 0),
+        )
+
+    def test_postgresql_session_lock_blocks_a_second_ingestion_session(self):
+        contender = self.open_second_session()
+        try:
+            with contender.cursor() as cursor:
+                cursor.execute(
+                    "SELECT pg_try_advisory_lock(%s, %s)",
+                    [ENTRY_INGESTION_LOCK_NAMESPACE, ENTRY_INGESTION_LOCK_KEY],
+                )
+                self.assertIs(cursor.fetchone()[0], True)
+
+            outcome = ingest_entry_snapshot(
+                transport=lambda **kwargs: self.fail("transport must not run")
+            )
+
+            self.assertEqual(
+                outcome,
+                EntryIngestionOutcome(EntryIngestionCode.ALREADY_RUNNING, 0),
+            )
+            self.assertFalse(FetchAttempt.objects.exists())
+            with contender.cursor() as cursor:
+                cursor.execute(
+                    "SELECT pg_advisory_unlock(%s, %s)",
+                    [ENTRY_INGESTION_LOCK_NAMESPACE, ENTRY_INGESTION_LOCK_KEY],
+                )
+                self.assertIs(cursor.fetchone()[0], True)
+        finally:
+            contender.close()
+
+    def test_session_lock_spans_transport_and_is_released_after_outcome(self):
+        observed = []
+
+        def inspect_lock_then_fail(**kwargs):
+            contender = self.open_second_session()
+            try:
+                with contender.cursor() as cursor:
+                    cursor.execute(
+                        "SELECT pg_try_advisory_lock(%s, %s)",
+                        [
+                            ENTRY_INGESTION_LOCK_NAMESPACE,
+                            ENTRY_INGESTION_LOCK_KEY,
+                        ],
+                    )
+                    observed.append(cursor.fetchone()[0])
+            finally:
+                contender.close()
+            return terminal_result()
+
+        outcome = ingest_entry_snapshot(transport=inspect_lock_then_fail)
+
+        self.assertEqual(outcome.code, EntryIngestionCode.FETCH_TERMINAL_FAILED)
+        self.assertEqual(observed, [False])
+        contender = self.open_second_session()
+        try:
+            with contender.cursor() as cursor:
+                cursor.execute(
+                    "SELECT pg_try_advisory_lock(%s, %s)",
+                    [ENTRY_INGESTION_LOCK_NAMESPACE, ENTRY_INGESTION_LOCK_KEY],
+                )
+                self.assertIs(cursor.fetchone()[0], True)
+                cursor.execute(
+                    "SELECT pg_advisory_unlock(%s, %s)",
+                    [ENTRY_INGESTION_LOCK_NAMESPACE, ENTRY_INGESTION_LOCK_KEY],
+                )
+                self.assertIs(cursor.fetchone()[0], True)
+        finally:
+            contender.close()
+
+    def test_prior_started_receipt_is_reconciled_before_new_operation(self):
+        source = SourceConfiguration.objects.get(code="MOFA_ENTRY_CSV")
+        rights = source.rights_decisions.get(decision_sequence=1)
+        abandoned = FetchAttempt.objects.create(
+            source=source,
+            source_revision=source.revision,
+            rights_decision=rights,
+            operation_id=uuid.uuid4(),
+            attempt_number=1,
+            request_fingerprint_revision=ENTRY_REQUEST_FINGERPRINT.revision,
+            normalized_request_sha256=(
+                ENTRY_REQUEST_FINGERPRINT.normalized_request_sha256
+            ),
+        )
+
+        outcome = ingest_entry_snapshot(
+            transport=SequenceTransport(terminal_result())
+        )
+
+        self.assertEqual(outcome.code, EntryIngestionCode.FETCH_TERMINAL_FAILED)
+        abandoned.refresh_from_db()
+        self.assertEqual(abandoned.result, FetchAttempt.Result.TERMINAL_FAILED)
+        self.assertEqual(
+            abandoned.failure_code,
+            FetchAttempt.FailureCode.WORKER_INTERRUPTED,
+        )
+        self.assertIsNotNone(abandoned.completed_at)
+        self.assertFalse(
+            FetchAttempt.objects.filter(result=FetchAttempt.Result.STARTED).exists()
+        )
+
+    def test_malformed_http_status_is_closed_without_leaving_started_receipt(self):
+        malformed = SingleAttemptResult(
+            request_fingerprint=ENTRY_REQUEST_FINGERPRINT,
+            attempt_result=ATTEMPT_RETRYABLE_FAILED,
+            http_status="503",
+            failure_code=FAILURE_UPSTREAM_5XX,
+        )
+
+        outcome = ingest_entry_snapshot(
+            transport=SequenceTransport(malformed)
+        )
+
+        self.assertEqual(outcome.code, EntryIngestionCode.FETCH_TERMINAL_FAILED)
+        attempt = FetchAttempt.objects.get()
+        self.assertEqual(attempt.result, FetchAttempt.Result.TERMINAL_FAILED)
+        self.assertEqual(
+            attempt.failure_code,
+            FetchAttempt.FailureCode.WORKER_INTERRUPTED,
+        )
+
+    def test_base_exception_closes_receipt_and_releases_operation_lock(self):
+        marker = "private-transport-interruption-marker"
+
+        def interrupted_transport(**kwargs):
+            raise KeyboardInterrupt(marker)
+
+        with self.assertRaises(KeyboardInterrupt) as raised:
+            ingest_entry_snapshot(transport=interrupted_transport)
+
+        self.assertEqual(raised.exception.args, ())
+        assert_exception_boundary_is_redacted(self, raised.exception, marker)
+        first = FetchAttempt.objects.get()
+        self.assertEqual(first.result, FetchAttempt.Result.TERMINAL_FAILED)
+        self.assertEqual(
+            first.failure_code,
+            FetchAttempt.FailureCode.WORKER_INTERRUPTED,
+        )
+        outcome = ingest_entry_snapshot(
+            transport=SequenceTransport(terminal_result())
+        )
+        self.assertEqual(outcome.code, EntryIngestionCode.FETCH_TERMINAL_FAILED)
+
+    def test_create_interruption_reconciles_committed_started_receipt(self):
+        marker = "private-create-interruption-marker"
+        original_create = entry_ingestion._create_started_attempt
+
+        def create_then_interrupt(*args, **kwargs):
+            original_create(*args, **kwargs)
+            raise KeyboardInterrupt(marker)
+
+        with patch(
+            "entry_requirements.ingestion._create_started_attempt",
+            side_effect=create_then_interrupt,
+        ):
+            with self.assertRaises(KeyboardInterrupt) as raised:
+                ingest_entry_snapshot(
+                    transport=lambda **kwargs: self.fail("transport must not run")
+                )
+
+        self.assertEqual(raised.exception.args, ())
+        assert_exception_boundary_is_redacted(self, raised.exception, marker)
+        attempt = FetchAttempt.objects.get()
+        self.assertEqual(attempt.result, FetchAttempt.Result.TERMINAL_FAILED)
+        self.assertEqual(
+            attempt.failure_code,
+            FetchAttempt.FailureCode.WORKER_INTERRUPTED,
+        )
+
+    def test_parser_base_exception_is_sanitized_and_rolls_back_candidate(self):
+        payload = entry_payload()
+        marker = "private-parser-interruption-marker"
+
+        def interrupted_parser(body):
+            self.assertEqual(body, payload)
+            raise KeyboardInterrupt(marker)
+
+        with self.assertRaises(KeyboardInterrupt) as raised:
+            ingest_entry_snapshot(
+                transport=SequenceTransport(successful_result(payload)),
+                parser=interrupted_parser,
+            )
+
+        self.assertEqual(raised.exception.args, ())
+        assert_exception_boundary_is_redacted(self, raised.exception, marker)
+        self.assertEqual(FetchAttempt.objects.get().result, FetchAttempt.Result.SUCCEEDED)
+        self.assertFalse(SourceArtifact.objects.exists())
+        self.assertFalse(ParseRun.objects.exists())
+        self.assertFalse(EntryFactRevision.objects.exists())
+
+    def test_close_failure_is_reconciled_before_returning(self):
+        with patch(
+            "entry_requirements.ingestion._close_attempt",
+            side_effect=RuntimeError("private close detail"),
+        ):
+            outcome = ingest_entry_snapshot(
+                transport=SequenceTransport(terminal_result())
+            )
+
+        self.assertEqual(outcome.code, EntryIngestionCode.PERSISTENCE_FAILED)
+        attempt = FetchAttempt.objects.get()
+        self.assertEqual(attempt.result, FetchAttempt.Result.TERMINAL_FAILED)
+        self.assertEqual(
+            attempt.failure_code,
+            FetchAttempt.FailureCode.WORKER_INTERRUPTED,
+        )
+
+    def test_terminal_failure_and_adapter_exception_each_close_their_receipt(self):
+        terminal = ingest_entry_snapshot(
+            transport=SequenceTransport(terminal_result())
+        )
+
+        def raising_transport(**kwargs):
+            self.assertFalse(connection.in_atomic_block)
+            raise RuntimeError("sensitive transport detail")
+
+        interrupted = ingest_entry_snapshot(transport=raising_transport)
+
+        self.assertEqual(terminal.code, EntryIngestionCode.FETCH_TERMINAL_FAILED)
+        self.assertEqual(interrupted.code, EntryIngestionCode.FETCH_TERMINAL_FAILED)
+        attempts = list(FetchAttempt.objects.order_by("started_at", "id"))
+        self.assertEqual(
+            [attempt.result for attempt in attempts],
+            [
+                FetchAttempt.Result.TERMINAL_FAILED,
+                FetchAttempt.Result.TERMINAL_FAILED,
+            ],
+        )
+        self.assertEqual(
+            {attempt.failure_code for attempt in attempts},
+            {FAILURE_TRANSPORT, FetchAttempt.FailureCode.WORKER_INTERRUPTED},
+        )
+        self.assertFalse(SourceArtifact.objects.exists())
+
+    def test_schema_drift_is_quarantined_with_no_typed_revision(self):
+        payload = entry_payload(headers=ENTRY_HEADERS + ("synthetic extra",))
+
+        outcome = ingest_entry_snapshot(
+            transport=SequenceTransport(successful_result(payload))
+        )
+
+        self.assertEqual(outcome.code, EntryIngestionCode.PARSE_QUARANTINED)
+        artifact = SourceArtifact.objects.get()
+        run = ParseRun.objects.get()
+        self.assertEqual(artifact.state, SourceArtifact.State.REJECTED)
+        self.assertEqual(run.outcome, ParseRun.Outcome.QUARANTINED)
+        self.assertEqual(run.failure_code, ParseRun.FailureCode.SCHEMA_MISMATCH)
+        self.assertNotEqual(
+            run.observed_schema_fingerprint_sha256,
+            run.expected_schema_fingerprint_sha256,
+        )
+        self.assertFalse(EntryFactRevision.objects.exists())
+
+    def test_replay_of_rejected_artifact_fails_closed(self):
+        payload = entry_payload(headers=ENTRY_HEADERS + ("synthetic extra",))
+        first = ingest_entry_snapshot(
+            transport=SequenceTransport(successful_result(payload))
+        )
+        second = ingest_entry_snapshot(
+            transport=SequenceTransport(successful_result(payload))
+        )
+
+        self.assertEqual(first.code, EntryIngestionCode.PARSE_QUARANTINED)
+        self.assertEqual(second.code, EntryIngestionCode.EVIDENCE_CONFLICT)
+        self.assertEqual(FetchAttempt.objects.count(), 2)
+        self.assertEqual(SourceArtifact.objects.count(), 1)
+        self.assertEqual(ParseRun.objects.count(), 1)
+        self.assertFalse(EntryFactRevision.objects.exists())
+
+    def test_typed_insert_failure_rolls_back_the_whole_candidate_boundary(self):
+        payload = entry_payload()
+        with patch.object(
+            EntryFactRevision.objects,
+            "create",
+            side_effect=RuntimeError("synthetic rollback"),
+        ):
+            outcome = ingest_entry_snapshot(
+                transport=SequenceTransport(successful_result(payload))
+            )
+
+        self.assertEqual(outcome.code, EntryIngestionCode.PERSISTENCE_FAILED)
+        attempt = FetchAttempt.objects.get()
+        self.assertEqual(attempt.result, FetchAttempt.Result.SUCCEEDED)
+        self.assertFalse(SourceArtifact.objects.exists())
+        self.assertFalse(ParseRun.objects.exists())
+        self.assertFalse(EntryFactRevision.objects.exists())
+
+    def test_rights_are_checked_before_fetch_and_again_before_persistence(self):
+        self.reject_entry_rights()
+        before = ingest_entry_snapshot(
+            transport=lambda **kwargs: self.fail("transport must not be called")
+        )
+        self.assertEqual(before.code, EntryIngestionCode.SOURCE_GATE_FAILED)
+        self.assertEqual(before.attempt_count, 0)
+        self.assertFalse(FetchAttempt.objects.exists())
+
+    def test_midflight_rights_rejection_blocks_artifact_and_typed_storage(self):
+        payload = entry_payload()
+
+        def revoke_then_succeed(**kwargs):
+            self.assertFalse(connection.in_atomic_block)
+            self.reject_entry_rights()
+            return successful_result(payload)
+
+        outcome = ingest_entry_snapshot(transport=revoke_then_succeed)
+
+        self.assertEqual(outcome.code, EntryIngestionCode.SOURCE_GATE_FAILED)
+        self.assertEqual(FetchAttempt.objects.get().result, FetchAttempt.Result.SUCCEEDED)
+        self.assertFalse(SourceArtifact.objects.exists())
+        self.assertFalse(ParseRun.objects.exists())
+        self.assertFalse(EntryFactRevision.objects.exists())
+
+    def test_malformed_success_hash_is_terminal_and_never_parsed(self):
+        payload = entry_payload()
+        malformed = SingleAttemptResult(
+            request_fingerprint=ENTRY_REQUEST_FINGERPRINT,
+            attempt_result=ATTEMPT_SUCCEEDED,
+            http_status=200,
+            response_sha256="0" * 64,
+            byte_count=len(payload),
+            body=payload,
+        )
+        outcome = ingest_entry_snapshot(transport=SequenceTransport(malformed))
+
+        self.assertEqual(outcome.code, EntryIngestionCode.FETCH_TERMINAL_FAILED)
+        attempt = FetchAttempt.objects.get()
+        self.assertEqual(attempt.result, FetchAttempt.Result.TERMINAL_FAILED)
+        self.assertEqual(
+            attempt.failure_code,
+            FetchAttempt.FailureCode.WORKER_INTERRUPTED,
+        )
+        self.assertEqual(attempt.response_sha256, "")
+        self.assertFalse(SourceArtifact.objects.exists())
+
+    def test_boundary_has_no_raw_secret_or_user_trip_persistence_fields(self):
+        payload = entry_payload()
+        outcome = ingest_entry_snapshot(
+            transport=SequenceTransport(successful_result(payload))
+        )
+
+        self.assertNotIn(BASIS_TEXT, repr(outcome))
+        self.assertNotIn(payload.hex(), repr(outcome))
+        verified = _verify_transport_result(successful_result(payload))
+        self.assertNotIn(BASIS_TEXT, repr(verified))
+        self.assertNotIn(payload.hex(), repr(verified))
+        parameters = set(inspect.signature(ingest_entry_snapshot).parameters)
+        self.assertEqual(parameters, {"transport", "parser", "retry_wait"})
+        banned = {
+            "raw",
+            "secret",
+            "key",
+            "destination",
+            "departure",
+            "return_date",
+            "session",
+            "purpose",
+        }
+        for model in (FetchAttempt, SourceArtifact, ParseRun, EntryFactRevision):
+            fields_by_name = {field.name: field for field in model._meta.fields}
+            self.assertFalse(
+                any(
+                    token in field_name
+                    for field_name in fields_by_name
+                    for token in banned
+                )
+            )
+            self.assertFalse(
+                any(
+                    isinstance(
+                        field,
+                        (models.BinaryField, models.JSONField, models.FileField),
+                    )
+                    for field in fields_by_name.values()
+                )
+            )
+
+
+class EntryIngestionCommandTests(SimpleTestCase):
+    def test_command_output_is_a_fixed_redacted_code(self):
+        output = io.StringIO()
+        with patch(
+            "operations.management.commands.ingest_entry_snapshot."
+            "ingest_entry_snapshot",
+            return_value=EntryIngestionOutcome(
+                EntryIngestionCode.REVIEW_REQUIRED,
+                1,
+            ),
+        ):
+            call_command("ingest_entry_snapshot", stdout=output)
+        self.assertEqual(
+            output.getvalue(),
+            "entry_ingestion_result=REVIEW_REQUIRED attempts=1\n",
+        )
+
+    def test_command_never_exposes_internal_exception_text(self):
+        marker = "sensitive-secret-and-raw-marker"
+        with patch(
+            "operations.management.commands.ingest_entry_snapshot."
+            "ingest_entry_snapshot",
+            side_effect=RuntimeError(marker),
+        ):
+            with self.assertRaises(CommandError) as raised:
+                call_command("ingest_entry_snapshot")
+        self.assertEqual(
+            str(raised.exception),
+            "entry_ingestion_result=PERSISTENCE_FAILED",
+        )
+        self.assertNotIn(marker, str(raised.exception))
+        assert_exception_boundary_is_redacted(self, raised.exception, marker)
+
+    def test_command_never_exposes_base_exception_text(self):
+        marker = "sensitive-base-exception-marker"
+        with patch(
+            "operations.management.commands.ingest_entry_snapshot."
+            "ingest_entry_snapshot",
+            side_effect=KeyboardInterrupt(marker),
+        ):
+            with self.assertRaises(CommandError) as raised:
+                call_command("ingest_entry_snapshot")
+        self.assertEqual(
+            str(raised.exception),
+            "entry_ingestion_result=PERSISTENCE_FAILED",
+        )
+        self.assertNotIn(marker, str(raised.exception))
+        assert_exception_boundary_is_redacted(self, raised.exception, marker)
diff --git a/operations/management/__init__.py b/operations/management/__init__.py
new file mode 100644
index 0000000..5929d2e
--- /dev/null
+++ b/operations/management/__init__.py
@@ -0,0 +1 @@
+"""Operational management command package."""
diff --git a/operations/management/commands/__init__.py b/operations/management/commands/__init__.py
new file mode 100644
index 0000000..6adad0f
--- /dev/null
+++ b/operations/management/commands/__init__.py
@@ -0,0 +1 @@
+"""Operational command package."""
diff --git a/operations/management/commands/ingest_entry_snapshot.py b/operations/management/commands/ingest_entry_snapshot.py
new file mode 100644
index 0000000..0fdcc44
--- /dev/null
+++ b/operations/management/commands/ingest_entry_snapshot.py
@@ -0,0 +1,52 @@
+from django.core.management.base import BaseCommand, CommandError
+
+from entry_requirements.ingestion import (
+    SUCCESS_CODES,
+    EntryIngestionCode,
+    EntryIngestionOutcome,
+    ingest_entry_snapshot,
+)
+
+
+KNOWN_CODES = frozenset(
+    {
+        EntryIngestionCode.REVIEW_REQUIRED,
+        EntryIngestionCode.REPLAY_OBSERVED,
+        EntryIngestionCode.PARSE_QUARANTINED,
+        EntryIngestionCode.PARSE_FAILED,
+        EntryIngestionCode.FETCH_TERMINAL_FAILED,
+        EntryIngestionCode.FETCH_RETRIES_EXHAUSTED,
+        EntryIngestionCode.SOURCE_GATE_FAILED,
+        EntryIngestionCode.EVIDENCE_CONFLICT,
+        EntryIngestionCode.PERSISTENCE_FAILED,
+        EntryIngestionCode.ALREADY_RUNNING,
+    }
+)
+
+
+def _invoke_ingestion_without_exception_context():
+    try:
+        return ingest_entry_snapshot()
+    except BaseException:
+        return None
+
+
+class Command(BaseCommand):
+    help = "Ingest the exact approved MOFA entry snapshot into review."
+
+    def handle(self, *args, **options):
+        outcome = _invoke_ingestion_without_exception_context()
+        if (
+            not isinstance(outcome, EntryIngestionOutcome)
+            or outcome.code not in KNOWN_CODES
+        ):
+            raise CommandError(
+                "entry_ingestion_result=PERSISTENCE_FAILED"
+            ) from None
+        message = (
+            f"entry_ingestion_result={outcome.code} "
+            f"attempts={outcome.attempt_count}"
+        )
+        if outcome.code not in SUCCESS_CODES:
+            raise CommandError(message)
+        self.stdout.write(message)


