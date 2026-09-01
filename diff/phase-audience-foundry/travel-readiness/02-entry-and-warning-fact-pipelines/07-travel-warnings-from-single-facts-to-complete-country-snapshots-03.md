## `feat(warnings): ingest approved alarm evidence`

diff --git a/operations/management/commands/ingest_travel_warning.py b/operations/management/commands/ingest_travel_warning.py
new file mode 100644
index 0000000..cb456c4
--- /dev/null
+++ b/operations/management/commands/ingest_travel_warning.py
@@ -0,0 +1,54 @@
+from django.core.management.base import BaseCommand, CommandError
+
+from travel_warnings.ingestion import (
+    SUCCESS_CODES,
+    TravelWarningIngestionCode,
+    TravelWarningIngestionOutcome,
+    ingest_travel_warning,
+)
+
+
+KNOWN_CODES = frozenset(
+    {
+        TravelWarningIngestionCode.REVIEW_REQUIRED,
+        TravelWarningIngestionCode.REPLAY_OBSERVED,
+        TravelWarningIngestionCode.PARSE_QUARANTINED,
+        TravelWarningIngestionCode.PARSE_FAILED,
+        TravelWarningIngestionCode.FETCH_TERMINAL_FAILED,
+        TravelWarningIngestionCode.FETCH_RETRIES_EXHAUSTED,
+        TravelWarningIngestionCode.SOURCE_GATE_FAILED,
+        TravelWarningIngestionCode.EVIDENCE_CONFLICT,
+        TravelWarningIngestionCode.PERSISTENCE_FAILED,
+        TravelWarningIngestionCode.ALREADY_RUNNING,
+    }
+)
+
+
+def _invoke_ingestion_without_exception_context():
+    try:
+        return ingest_travel_warning()
+    except BaseException:
+        return None
+
+
+class Command(BaseCommand):
+    help = "Ingest the exact approved MOFA JP travel warning into review."
+
+    def handle(self, *args, **options):
+        outcome = _invoke_ingestion_without_exception_context()
+        if (
+            not isinstance(outcome, TravelWarningIngestionOutcome)
+            or outcome.code not in KNOWN_CODES
+            or type(outcome.attempt_count) is not int
+            or outcome.attempt_count < 0
+        ):
+            raise CommandError(
+                "travel_warning_ingestion_result=PERSISTENCE_FAILED attempts=0"
+            ) from None
+        message = (
+            f"travel_warning_ingestion_result={outcome.code} "
+            f"attempts={outcome.attempt_count}"
+        )
+        if outcome.code not in SUCCESS_CODES:
+            raise CommandError(message)
+        self.stdout.write(message)
diff --git a/travel_warnings/ingestion.py b/travel_warnings/ingestion.py
new file mode 100644
index 0000000..1d876b9
--- /dev/null
+++ b/travel_warnings/ingestion.py
@@ -0,0 +1,1081 @@
+"""Closed ingestion boundary for the approved JP travel-warning source.
+
+Every network attempt is surrounded by a durable redacted receipt. Successful
+bytes remain in memory only until their hash, schema, and typed projection have
+been checked; no raw response or credential value is persisted or returned.
+"""
+
+from __future__ import annotations
+
+import hashlib
+import json
+import os
+import time
+import uuid
+from dataclasses import dataclass, field
+from datetime import date
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
+    FAILURE_AUTHENTICATION,
+    FAILURE_HTTP_CLIENT,
+    FAILURE_PROVIDER_ERROR,
+    FAILURE_RATE_LIMITED,
+    FAILURE_RESPONSE_TOO_LARGE,
+    FAILURE_SECRET_REFLECTION,
+    FAILURE_TIMEOUT,
+    FAILURE_TRANSPORT,
+    FAILURE_UPSTREAM_5XX,
+    PROVIDER_GATEWAY_20,
+    PROVIDER_OTHER_ERROR,
+    PROVIDER_SUCCESS_0,
+    WARNING_MAX_RESPONSE_BYTES,
+    WARNING_REQUEST_FINGERPRINT,
+    WARNING_SECRET_REFERENCE,
+    WARNING_SOURCE_LOCATOR,
+    SingleAttemptResult,
+    fetch_travel_alarm_jp,
+)
+
+from .models import (
+    SOURCE_ALARM_LEVEL_CODE_MAX_LENGTH,
+    SOURCE_SCOPE_TEXT_MAX_LENGTH,
+    SOURCE_SCOPE_TYPE_MAX_LENGTH,
+    TravelWarningRevision,
+)
+from .parser import (
+    EXPECTED_SCHEMA_FINGERPRINT_SHA256,
+    PARSER_CONTRACT_FINGERPRINT_SHA256,
+    ParsedTravelWarning,
+    TravelWarningParseFailure,
+    TravelWarningParseResult,
+    TravelWarningParseSuccess,
+    parse_travel_alarm_jp,
+)
+
+
+WARNING_SOURCE_CODE = "MOFA_TRAVEL_ALARM_API_JP"
+WARNING_SOURCE_REVISION = "rights-v1"
+WARNING_SOURCE_MODULE = SourceConfiguration.Module.TRAVEL_WARNING
+WARNING_SOURCE_OWNER = "대한민국 외교부"
+WARNING_FIELD_SCOPE = "JP_WARNING_V1"
+WARNING_ATTRIBUTION = "외교부|공공데이터포털"
+WARNING_DECIDED_BY = "PROJECT_OWNER_REQUEST"
+WARNING_DECISION_BASIS = "USER_FOLLOWUP_20260830"
+WARNING_PARSER_NAME = ParseRun.ParserName.MOFA_TRAVEL_ALARM_JSON
+WARNING_PARSER_VERSION = ParseRun.ParserVersion.V1
+WARNING_INGESTION_LOCK_NAMESPACE = 1_414_678_614
+WARNING_INGESTION_LOCK_KEY = 1_006_002
+
+
+class TravelWarningIngestionCode:
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
+        TravelWarningIngestionCode.REVIEW_REQUIRED,
+        TravelWarningIngestionCode.REPLAY_OBSERVED,
+    }
+)
+
+
+@dataclass(frozen=True, slots=True)
+class TravelWarningIngestionOutcome:
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
+    secret_reference_name: str
+    connect_timeout_seconds: int
+    read_timeout_seconds: int
+    max_retries: int
+
+
+@dataclass(frozen=True, slots=True)
+class _LoadedSecret:
+    value: str = field(repr=False, compare=False)
+
+
+@dataclass(frozen=True, slots=True)
+class _VerifiedTransportResult:
+    attempt_result: str
+    http_status: int | None
+    provider_code: str
+    failure_code: str
+    response_sha256: str
+    byte_count: int
+    body: bytes = field(repr=False, compare=False)
+
+
+@dataclass(frozen=True, slots=True)
+class _VerifiedParseResult:
+    outcome: str
+    failure_code: str
+    observed_schema_fingerprint_sha256: str
+    warning: ParsedTravelWarning | None
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
+SecretLoader = Callable[[str], object]
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
+        raise SystemExit() from None
+    raise BaseException() from None
+
+
+def _environment_secret_loader(reference_name: str) -> object:
+    """Read only the named process environment variable; never a dotenv file."""
+
+    return os.environ.get(reference_name)
+
+
+def _load_secret(
+    reference_name: str,
+    loader: SecretLoader,
+) -> _LoadedSecret | None:
+    value = loader(reference_name)
+    if type(value) is not str or value == "":
+        return None
+    return _LoadedSecret(value=value)
+
+
+def _try_acquire_warning_ingestion_lock() -> bool:
+    if (
+        connection.vendor != "postgresql"
+        or connection.get_autocommit() is not True
+        or connection.in_atomic_block
+    ):
+        raise RuntimeError(
+            "travel warning ingestion requires an autocommit PostgreSQL session"
+        )
+    with connection.cursor() as cursor:
+        cursor.execute(
+            "SELECT pg_try_advisory_lock(%s, %s)",
+            [
+                WARNING_INGESTION_LOCK_NAMESPACE,
+                WARNING_INGESTION_LOCK_KEY,
+            ],
+        )
+        row = cursor.fetchone()
+    if not row or type(row[0]) is not bool:
+        raise RuntimeError("invalid advisory lock result")
+    return row[0]
+
+
+def _release_warning_ingestion_lock() -> None:
+    try:
+        with connection.cursor() as cursor:
+            cursor.execute(
+                "SELECT pg_advisory_unlock(%s, %s)",
+                [
+                    WARNING_INGESTION_LOCK_NAMESPACE,
+                    WARNING_INGESTION_LOCK_KEY,
+                ],
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
+def _discard_warning_ingestion_connection() -> None:
+    try:
+        connection.close()
+    except BaseException:
+        pass
+
+
+def _close_abandoned_started_attempts() -> None:
+    with transaction.atomic(durable=True):
+        abandoned = list(
+            FetchAttempt.objects.select_for_update()
+            .filter(
+                source__code=WARNING_SOURCE_CODE,
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
+                    TravelWarningIngestionCode.PERSISTENCE_FAILED
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
+        and rights.source_revision == WARNING_SOURCE_REVISION
+        and rights.decision_sequence == 1
+        and rights.decision == SourceRightsDecision.Decision.APPROVED
+        and rights.access_mode
+        == SourceRightsDecision.AccessMode.CREDENTIAL_REFERENCE
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
+        and rights.field_scope_code == WARNING_FIELD_SCOPE
+        and rights.attribution_text == WARNING_ATTRIBUTION
+        and rights.contract_fingerprint_sha256
+        == PARSER_CONTRACT_FINGERPRINT_SHA256
+        and rights.decided_by == WARNING_DECIDED_BY
+        and rights.decision_basis_code == WARNING_DECISION_BASIS
+    )
+
+
+def _source_is_exact(source: SourceConfiguration) -> bool:
+    return bool(
+        source.code == WARNING_SOURCE_CODE
+        and source.revision == WARNING_SOURCE_REVISION
+        and source.module == WARNING_SOURCE_MODULE
+        and source.owner == WARNING_SOURCE_OWNER
+        and source.official_locator == WARNING_SOURCE_LOCATOR
+        and source.state == SourceConfiguration.State.ACTIVE
+        and source.enabled
+        and source.secret_reference_name == WARNING_SECRET_REFERENCE
+        and source.connect_timeout_seconds == 5
+        and source.read_timeout_seconds == 15
+        and source.max_retries == 2
+    )
+
+
+def _locked_approved_source() -> tuple[SourceConfiguration, SourceRightsDecision]:
+    source = (
+        SourceConfiguration.objects.select_for_update()
+        .filter(code=WARNING_SOURCE_CODE)
+        .first()
+    )
+    if source is None or not _source_is_exact(source):
+        raise _ClosedPersistenceFailure(
+            TravelWarningIngestionCode.SOURCE_GATE_FAILED
+        )
+    rights = _latest_rights(source)
+    if not _rights_are_exact(rights):
+        raise _ClosedPersistenceFailure(
+            TravelWarningIngestionCode.SOURCE_GATE_FAILED
+        )
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
+            request_fingerprint_revision=WARNING_REQUEST_FINGERPRINT.revision,
+            normalized_request_sha256=(
+                WARNING_REQUEST_FINGERPRINT.normalized_request_sha256
+            ),
+        )
+        configuration = _FetchConfiguration(
+            source_id=source.id,
+            rights_id=rights.id,
+            secret_reference_name=source.secret_reference_name,
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
+        provider_code="",
+        failure_code=FetchAttempt.FailureCode.WORKER_INTERRUPTED,
+        response_sha256="",
+        byte_count=0,
+        body=b"",
+    )
+
+
+def _missing_secret_result() -> SingleAttemptResult:
+    return SingleAttemptResult(
+        request_fingerprint=WARNING_REQUEST_FINGERPRINT,
+        attempt_result=ATTEMPT_TERMINAL_FAILED,
+        failure_code=FAILURE_AUTHENTICATION,
+    )
+
+
+def _verify_transport_result(result: object) -> _VerifiedTransportResult:
+    if type(result) is not SingleAttemptResult:
+        return _worker_interrupted_result()
+    if (
+        type(result.request_fingerprint) is not type(WARNING_REQUEST_FINGERPRINT)
+        or result.request_fingerprint.revision
+        != WARNING_REQUEST_FINGERPRINT.revision
+        or result.request_fingerprint.normalized_request_sha256
+        != WARNING_REQUEST_FINGERPRINT.normalized_request_sha256
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
+
+    known_provider_codes = {
+        "",
+        PROVIDER_SUCCESS_0,
+        PROVIDER_GATEWAY_20,
+        PROVIDER_OTHER_ERROR,
+    }
+    if result.provider_code not in known_provider_codes:
+        return _worker_interrupted_result()
+
+    if result.attempt_result == ATTEMPT_SUCCEEDED:
+        if (
+            result.http_status != 200
+            or result.provider_code != PROVIDER_SUCCESS_0
+            or result.failure_code != ""
+            or result.byte_count != len(result.body)
+            or result.byte_count > WARNING_MAX_RESPONSE_BYTES
+            or result.response_sha256
+            != hashlib.sha256(result.body).hexdigest()
+        ):
+            return _worker_interrupted_result()
+        return _VerifiedTransportResult(
+            attempt_result=result.attempt_result,
+            http_status=result.http_status,
+            provider_code=result.provider_code,
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
+        or result.provider_code == PROVIDER_SUCCESS_0
+    ):
+        return _worker_interrupted_result()
+
+    retry_shape = False
+    if result.failure_code == FAILURE_TIMEOUT:
+        retry_shape = result.http_status is None and result.provider_code == ""
+    elif result.failure_code == FAILURE_RATE_LIMITED:
+        retry_shape = result.http_status == 429
+    elif result.failure_code == FAILURE_UPSTREAM_5XX:
+        retry_shape = bool(
+            type(result.http_status) is int
+            and 500 <= result.http_status <= 599
+        )
+
+    if result.attempt_result == ATTEMPT_RETRYABLE_FAILED:
+        if not retry_shape:
+            return _worker_interrupted_result()
+    elif result.attempt_result == ATTEMPT_TERMINAL_FAILED:
+        terminal_shape = False
+        if result.failure_code == FAILURE_AUTHENTICATION:
+            terminal_shape = result.http_status in (None, 401, 403)
+        elif result.failure_code == FAILURE_PROVIDER_ERROR:
+            terminal_shape = (
+                result.http_status == 200
+                and result.provider_code
+                in {PROVIDER_GATEWAY_20, PROVIDER_OTHER_ERROR}
+            )
+        elif result.failure_code == FAILURE_SECRET_REFLECTION:
+            terminal_shape = (
+                result.http_status is not None and result.provider_code == ""
+            )
+        elif result.failure_code == FAILURE_HTTP_CLIENT:
+            terminal_shape = bool(
+                type(result.http_status) is int
+                and result.http_status not in (200, 401, 403, 429)
+                and not 500 <= result.http_status <= 599
+            )
+        elif result.failure_code == FAILURE_TRANSPORT:
+            terminal_shape = (
+                result.http_status is None and result.provider_code == ""
+            )
+        elif result.failure_code == FAILURE_RESPONSE_TOO_LARGE:
+            terminal_shape = (
+                result.http_status == 200 and result.provider_code == ""
+            )
+        if not terminal_shape:
+            return _worker_interrupted_result()
+    else:
+        return _worker_interrupted_result()
+
+    return _VerifiedTransportResult(
+        attempt_result=result.attempt_result,
+        http_status=result.http_status,
+        provider_code=result.provider_code,
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
+                TravelWarningIngestionCode.PERSISTENCE_FAILED
+            )
+        updated = FetchAttempt.objects.filter(
+            pk=attempt.id,
+            result=FetchAttempt.Result.STARTED,
+        ).update(
+            result=result.attempt_result,
+            completed_at=timezone.now(),
+            http_status=result.http_status,
+            provider_code=result.provider_code,
+            response_sha256=result.response_sha256,
+            failure_code=result.failure_code,
+        )
+        if updated != 1:
+            raise _ClosedPersistenceFailure(
+                TravelWarningIngestionCode.PERSISTENCE_FAILED
+            )
+        attempt.refresh_from_db()
+    return attempt
+
+
+def _canonical_typed_fingerprint(warning: ParsedTravelWarning) -> str:
+    canonical = json.dumps(
+        {
+            "country_iso2": warning.country_iso2,
+            "country_name_en": warning.country_name_en,
+            "country_name_ko": warning.country_name_ko,
+            "source_alarm_level_code": warning.source_alarm_level_code,
+            "source_scope_text": warning.source_scope_text,
+            "source_scope_type": warning.source_scope_type,
+            "source_written_date": (
+                warning.source_written_date.isoformat()
+                if warning.source_written_date is not None
+                else None
+            ),
+        },
+        ensure_ascii=False,
+        sort_keys=True,
+        separators=(",", ":"),
+    ).encode("utf-8")
+    return hashlib.sha256(canonical).hexdigest()
+
+
+def _internal_parse_failure() -> _VerifiedParseResult:
+    return _VerifiedParseResult(
+        outcome=ParseRun.Outcome.FAILED,
+        failure_code=ParseRun.FailureCode.INTERNAL_PARSER_ERROR,
+        observed_schema_fingerprint_sha256="",
+        warning=None,
+    )
+
+
+def _is_sha256(value: object) -> bool:
+    return bool(
+        type(value) is str
+        and len(value) == 64
+        and all(character in "0123456789abcdef" for character in value)
+    )
+
+
+def _safe_parse(
+    body: bytes,
+    parser: Callable[[bytes], TravelWarningParseResult],
+) -> _VerifiedParseResult:
+    try:
+        result = parser(body)
+        if type(result) is TravelWarningParseSuccess:
+            warning = result.warning
+            if (
+                type(warning) is ParsedTravelWarning
+                and result.observed_schema_fingerprint_sha256
+                == EXPECTED_SCHEMA_FINGERPRINT_SHA256
+                and type(warning.country_iso2) is str
+                and warning.country_iso2 == "JP"
+                and type(warning.country_name_ko) is str
+                and warning.country_name_ko == "일본"
+                and type(warning.country_name_en) is str
+                and warning.country_name_en == "Japan"
+                and type(warning.source_alarm_level_code) is str
+                and bool(warning.source_alarm_level_code.strip())
+                and len(warning.source_alarm_level_code)
+                <= SOURCE_ALARM_LEVEL_CODE_MAX_LENGTH
+                and type(warning.source_scope_type) is str
+                and bool(warning.source_scope_type.strip())
+                and len(warning.source_scope_type)
+                <= SOURCE_SCOPE_TYPE_MAX_LENGTH
+                and type(warning.source_scope_text) is str
+                and bool(warning.source_scope_text.strip())
+                and len(warning.source_scope_text)
+                <= SOURCE_SCOPE_TEXT_MAX_LENGTH
+                and (
+                    warning.source_written_date is None
+                    or type(warning.source_written_date) is date
+                )
+                and type(warning.typed_fingerprint_sha256) is str
+                and warning.typed_fingerprint_sha256
+                == _canonical_typed_fingerprint(warning)
+            ):
+                return _VerifiedParseResult(
+                    outcome=ParseRun.Outcome.VALIDATED,
+                    failure_code="",
+                    observed_schema_fingerprint_sha256=(
+                        EXPECTED_SCHEMA_FINGERPRINT_SHA256
+                    ),
+                    warning=warning,
+                )
+        elif type(result) is TravelWarningParseFailure:
+            failure_shapes = {
+                ParseRun.FailureCode.SCHEMA_MISMATCH: bool(
+                    _is_sha256(result.observed_schema_fingerprint_sha256)
+                    and result.observed_schema_fingerprint_sha256
+                    != EXPECTED_SCHEMA_FINGERPRINT_SHA256
+                ),
+                ParseRun.FailureCode.IDENTITY_MISMATCH: (
+                    result.observed_schema_fingerprint_sha256
+                    == EXPECTED_SCHEMA_FINGERPRINT_SHA256
+                ),
+                ParseRun.FailureCode.REQUIRED_VALUE_MISSING: (
+                    result.observed_schema_fingerprint_sha256
+                    == EXPECTED_SCHEMA_FINGERPRINT_SHA256
+                ),
+                ParseRun.FailureCode.DUPLICATE_RECORD: (
+                    result.observed_schema_fingerprint_sha256
+                    == EXPECTED_SCHEMA_FINGERPRINT_SHA256
+                ),
+                ParseRun.FailureCode.ENCODING_ERROR: (
+                    result.observed_schema_fingerprint_sha256 == ""
+                ),
+                ParseRun.FailureCode.SYNTAX_ERROR: (
+                    result.observed_schema_fingerprint_sha256 == ""
+                ),
+                ParseRun.FailureCode.INVALID_VALUE: (
+                    result.observed_schema_fingerprint_sha256
+                    == EXPECTED_SCHEMA_FINGERPRINT_SHA256
+                ),
+                ParseRun.FailureCode.CONFLICTING_VALUE: (
+                    result.observed_schema_fingerprint_sha256
+                    == EXPECTED_SCHEMA_FINGERPRINT_SHA256
+                ),
+                ParseRun.FailureCode.INTERNAL_PARSER_ERROR: (
+                    result.observed_schema_fingerprint_sha256 == ""
+                ),
+            }
+            if (
+                isinstance(result.failure_code, str)
+                and type(result.observed_schema_fingerprint_sha256) is str
+                and failure_shapes.get(result.failure_code, False)
+            ):
+                return _VerifiedParseResult(
+                    outcome=(
+                        ParseRun.Outcome.FAILED
+                        if result.failure_code
+                        == ParseRun.FailureCode.INTERNAL_PARSER_ERROR
+                        else ParseRun.Outcome.QUARANTINED
+                    ),
+                    failure_code=result.failure_code,
+                    observed_schema_fingerprint_sha256=(
+                        result.observed_schema_fingerprint_sha256
+                    ),
+                    warning=None,
+                )
+    except Exception:
+        pass
+    return _internal_parse_failure()
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
+            parser_name=WARNING_PARSER_NAME,
+            parser_version=WARNING_PARSER_VERSION,
+        )
+    )
+    if len(runs) != 1:
+        return False
+    run = runs[0]
+    return bool(
+        run.outcome == ParseRun.Outcome.VALIDATED
+        and run.parser_contract_fingerprint_sha256
+        == PARSER_CONTRACT_FINGERPRINT_SHA256
+        and run.expected_schema_fingerprint_sha256
+        == EXPECTED_SCHEMA_FINGERPRINT_SHA256
+        and run.observed_schema_fingerprint_sha256
+        == EXPECTED_SCHEMA_FINGERPRINT_SHA256
+        and TravelWarningRevision.objects.filter(parse_run=run).count() == 1
+    )
+
+
+def _persist_success(
+    attempt: FetchAttempt,
+    verified: _VerifiedTransportResult,
+    parser: Callable[[bytes], TravelWarningParseResult],
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
+                or locked_attempt.http_status != 200
+                or locked_attempt.provider_code
+                != FetchAttempt.ProviderCode.MOFA_SUCCESS_0
+                or locked_attempt.response_sha256 != verified.response_sha256
+                or verified.provider_code != PROVIDER_SUCCESS_0
+                or verified.byte_count != len(verified.body)
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
+                        TravelWarningIngestionCode.EVIDENCE_CONFLICT
+                    )
+                return TravelWarningIngestionCode.REPLAY_OBSERVED
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
+            if (
+                anchor is None
+                or anchor.source_revision != source.revision
+                or anchor.rights_decision_id != rights.id
+                or anchor.http_status != 200
+                or anchor.provider_code
+                != FetchAttempt.ProviderCode.MOFA_SUCCESS_0
+                or anchor.response_sha256 != verified.response_sha256
+                or anchor.completed_at is None
+            ):
+                raise _ClosedPersistenceFailure(
+                    TravelWarningIngestionCode.EVIDENCE_CONFLICT
+                )
+            artifact = SourceArtifact.objects.create(
+                source=source,
+                body_sha256=verified.response_sha256,
+                byte_count=verified.byte_count,
+                first_successful_attempt=anchor,
+            )
+            parse_run = ParseRun.objects.create(
+                artifact=artifact,
+                parser_name=WARNING_PARSER_NAME,
+                parser_version=WARNING_PARSER_VERSION,
+                parser_contract_fingerprint_sha256=(
+                    PARSER_CONTRACT_FINGERPRINT_SHA256
+                ),
+                expected_schema_fingerprint_sha256=(
+                    EXPECTED_SCHEMA_FINGERPRINT_SHA256
+                ),
+                started_at=max(timezone.now(), anchor.completed_at),
+            )
+            parse_result = _safe_parse(verified.body, parser)
+            completed_at = max(timezone.now(), parse_run.started_at)
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
+                    TravelWarningIngestionCode.PERSISTENCE_FAILED
+                )
+
+            if parse_result.outcome != ParseRun.Outcome.VALIDATED:
+                updated = SourceArtifact.objects.filter(
+                    pk=artifact.id,
+                    state=SourceArtifact.State.RECEIVED,
+                ).update(state=SourceArtifact.State.REJECTED)
+                if updated != 1:
+                    raise _ClosedPersistenceFailure(
+                        TravelWarningIngestionCode.PERSISTENCE_FAILED
+                    )
+                if parse_result.outcome == ParseRun.Outcome.QUARANTINED:
+                    return TravelWarningIngestionCode.PARSE_QUARANTINED
+                return TravelWarningIngestionCode.PARSE_FAILED
+
+            warning = parse_result.warning
+            if warning is None:
+                raise _ClosedPersistenceFailure(
+                    TravelWarningIngestionCode.PERSISTENCE_FAILED
+                )
+            updated = SourceArtifact.objects.filter(
+                pk=artifact.id,
+                state=SourceArtifact.State.RECEIVED,
+            ).update(state=SourceArtifact.State.REVIEW_REQUIRED)
+            if updated != 1:
+                raise _ClosedPersistenceFailure(
+                    TravelWarningIngestionCode.PERSISTENCE_FAILED
+                )
+
+            current_source, current_rights = _locked_approved_source()
+            if current_source.id != source.id or current_rights.id != rights.id:
+                raise _ClosedPersistenceFailure(
+                    TravelWarningIngestionCode.SOURCE_GATE_FAILED
+                )
+            country = Country.objects.get(pk=JP_COUNTRY_ID)
+            TravelWarningRevision.objects.create(
+                country=country,
+                parse_run=parse_run,
+                source_alarm_level_code=warning.source_alarm_level_code,
+                source_scope_type=warning.source_scope_type,
+                source_scope_text=warning.source_scope_text,
+                source_written_date=warning.source_written_date,
+                first_observed_at=anchor.completed_at,
+                typed_fingerprint_sha256=warning.typed_fingerprint_sha256,
+            )
+            return TravelWarningIngestionCode.REVIEW_REQUIRED
+    except _ClosedPersistenceFailure:
+        raise
+    except Exception:
+        raise _ClosedPersistenceFailure(
+            TravelWarningIngestionCode.PERSISTENCE_FAILED
+        ) from None
+
+
+def _run_travel_warning_ingestion(
+    *,
+    transport: Callable[..., SingleAttemptResult],
+    parser: Callable[[bytes], TravelWarningParseResult],
+    retry_wait: Callable[[int], None],
+    secret_loader: SecretLoader,
+) -> TravelWarningIngestionOutcome | _SanitizedInterruption:
+    """Run one operation and return only a closed outcome or kind sentinel."""
+
+    try:
+        lock_acquired = _try_acquire_warning_ingestion_lock()
+    except Exception:
+        _discard_warning_ingestion_connection()
+        return TravelWarningIngestionOutcome(
+            TravelWarningIngestionCode.PERSISTENCE_FAILED,
+            0,
+        )
+    except BaseException as exception:
+        exception_kind = _base_exception_kind(exception)
+        del exception
+        _discard_warning_ingestion_connection()
+        return _SanitizedInterruption(exception_kind)
+    if not lock_acquired:
+        return TravelWarningIngestionOutcome(
+            TravelWarningIngestionCode.ALREADY_RUNNING,
+            0,
+        )
+    try:
+        try:
+            _close_abandoned_started_attempts()
+        except Exception:
+            return TravelWarningIngestionOutcome(
+                TravelWarningIngestionCode.PERSISTENCE_FAILED,
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
+                return TravelWarningIngestionOutcome(
+                    failure.code,
+                    attempt_count - 1,
+                )
+            except Exception:
+                return TravelWarningIngestionOutcome(
+                    TravelWarningIngestionCode.PERSISTENCE_FAILED,
+                    attempt_count - 1,
+                )
+            if max_retries is None:
+                max_retries = configuration.max_retries
+
+            interruption_kind: str | None = None
+            loaded_secret: _LoadedSecret | None = None
+            raw_result: object = None
+            verified: _VerifiedTransportResult | None = None
+            closed_attempt: FetchAttempt | None = None
+            try:
+                loaded_secret = _load_secret(
+                    configuration.secret_reference_name,
+                    secret_loader,
+                )
+                if loaded_secret is None:
+                    raw_result: object = _missing_secret_result()
+                else:
+                    raw_result = transport(
+                        official_locator=WARNING_SOURCE_LOCATOR,
+                        secret_reference_name=(
+                            configuration.secret_reference_name
+                        ),
+                        secret_value=loaded_secret.value,
+                        connect_timeout_seconds=(
+                            configuration.connect_timeout_seconds
+                        ),
+                        read_timeout_seconds=(
+                            configuration.read_timeout_seconds
+                        ),
+                    )
+            except Exception:
+                raw_result = None
+            except BaseException as exception:
+                interruption_kind = _base_exception_kind(exception)
+                del exception
+                raw_result = None
+            finally:
+                loaded_secret = None
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
+            except Exception:
+                try:
+                    _close_abandoned_started_attempts()
+                except BaseException:
+                    pass
+                loaded_secret = None
+                raw_result = None
+                verified = None
+                closed_attempt = None
+                return TravelWarningIngestionOutcome(
+                    TravelWarningIngestionCode.PERSISTENCE_FAILED,
+                    attempt_count,
+                )
+            except BaseException as exception:
+                exception_kind = _base_exception_kind(exception)
+                del exception
+                try:
+                    _close_abandoned_started_attempts()
+                except BaseException:
+                    pass
+                loaded_secret = None
+                raw_result = None
+                verified = None
+                closed_attempt = None
+                return _SanitizedInterruption(exception_kind)
+            if interruption_kind is not None:
+                loaded_secret = None
+                raw_result = None
+                verified = None
+                closed_attempt = None
+                return _SanitizedInterruption(interruption_kind)
+
+            if verified.attempt_result == ATTEMPT_SUCCEEDED:
+                try:
+                    code = _persist_success(closed_attempt, verified, parser)
+                except _ClosedPersistenceFailure as failure:
+                    code = failure.code
+                return TravelWarningIngestionOutcome(code, attempt_count)
+
+            if verified.attempt_result != ATTEMPT_RETRYABLE_FAILED:
+                return TravelWarningIngestionOutcome(
+                    TravelWarningIngestionCode.FETCH_TERMINAL_FAILED,
+                    attempt_count,
+                )
+            if max_retries is None or attempt_count > max_retries:
+                return TravelWarningIngestionOutcome(
+                    TravelWarningIngestionCode.FETCH_RETRIES_EXHAUSTED,
+                    attempt_count,
+                )
+            try:
+                retry_wait(attempt_count)
+            except Exception:
+                return TravelWarningIngestionOutcome(
+                    TravelWarningIngestionCode.PERSISTENCE_FAILED,
+                    attempt_count,
+                )
+    except BaseException as exception:
+        exception_kind = _base_exception_kind(exception)
+        del exception
+        try:
+            _close_abandoned_started_attempts()
+        except BaseException:
+            pass
+        loaded_secret = None
+        raw_result = None
+        verified = None
+        closed_attempt = None
+        interruption_kind = None
+        return _SanitizedInterruption(exception_kind)
+    finally:
+        _release_warning_ingestion_lock()
+
+
+def ingest_travel_warning(
+    *,
+    transport: Callable[..., SingleAttemptResult] = fetch_travel_alarm_jp,
+    parser: Callable[[bytes], TravelWarningParseResult] = parse_travel_alarm_jp,
+    retry_wait: Callable[[int], None] = _bounded_retry_wait,
+    secret_loader: SecretLoader = _environment_secret_loader,
+) -> TravelWarningIngestionOutcome:
+    """Run one redacted operation, including only its permitted retries."""
+
+    result = _run_travel_warning_ingestion(
+        transport=transport,
+        parser=parser,
+        retry_wait=retry_wait,
+        secret_loader=secret_loader,
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
diff --git a/travel_warnings/tests/test_ingestion.py b/travel_warnings/tests/test_ingestion.py
new file mode 100644
index 0000000..f435e7d
--- /dev/null
+++ b/travel_warnings/tests/test_ingestion.py
@@ -0,0 +1,1191 @@
+import hashlib
+import inspect
+import io
+import json
+import uuid
+from unittest.mock import patch
+
+import psycopg
+from django.core.management import call_command
+from django.core.management.base import CommandError
+from django.db import connection, models
+from django.test import SimpleTestCase, TransactionTestCase
+
+from countries.models import JP_COUNTRY_ID, Country
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
+    FAILURE_AUTHENTICATION,
+    FAILURE_PROVIDER_ERROR,
+    FAILURE_RATE_LIMITED,
+    FAILURE_TIMEOUT,
+    FAILURE_TRANSPORT,
+    FAILURE_UPSTREAM_5XX,
+    PROVIDER_OTHER_ERROR,
+    PROVIDER_SUCCESS_0,
+    WARNING_REQUEST_FINGERPRINT,
+    WARNING_SECRET_REFERENCE,
+    WARNING_SOURCE_LOCATOR,
+    SingleAttemptResult,
+)
+import travel_warnings.ingestion as warning_ingestion
+from travel_warnings.ingestion import (
+    WARNING_INGESTION_LOCK_KEY,
+    WARNING_INGESTION_LOCK_NAMESPACE,
+    TravelWarningIngestionCode,
+    TravelWarningIngestionOutcome,
+    _LoadedSecret,
+    _environment_secret_loader,
+    _verify_transport_result,
+    ingest_travel_warning,
+)
+from travel_warnings.models import TravelWarningRevision
+from travel_warnings.parser import parse_travel_alarm_jp
+
+
+SYNTHETIC_CREDENTIAL = "synthetic-credential-value"
+ALARM_LEVEL = "3"
+SCOPE_TYPE = "일부"
+SCOPE_TEXT = "합성 검증 범위"
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
+                "/travel_warnings/ingestion.py",
+                "/operations/management/commands/ingest_travel_warning.py",
+            )
+        ):
+            for value in traceback.tb_frame.f_locals.values():
+                testcase.assertNotIn(marker, repr(value))
+        traceback = traceback.tb_next
+
+
+def warning_item(**overrides):
+    item = {
+        "alarm_lvl": ALARM_LEVEL,
+        "continent_cd": "1000",
+        "continent_eng_nm": "Asia",
+        "continent_nm": "아시아",
+        "country_eng_nm": "Japan",
+        "country_iso_alp2": "JP",
+        "country_nm": "일본",
+        "dang_map_download_url": "",
+        "flag_download_url": "",
+        "map_download_url": "",
+        "org_country_idx": "1",
+        "region_ty": SCOPE_TYPE,
+        "remark": SCOPE_TEXT,
+        "written_dt": None,
+    }
+    item.update(overrides)
+    return item
+
+
+def warning_payload(*, item=None) -> bytes:
+    document = {
+        "response": {
+            "header": {"resultCode": "0", "resultMsg": "SYNTHETIC"},
+            "body": {
+                "dataType": "JSON",
+                "items": {
+                    "item": [item if item is not None else warning_item()]
+                },
+                "numOfRows": 1,
+                "pageNo": 1,
+                "totalCount": 1,
+            },
+        }
+    }
+    return json.dumps(
+        document,
+        ensure_ascii=False,
+        separators=(",", ":"),
+    ).encode("utf-8")
+
+
+def successful_result(
+    payload: bytes,
+    *,
+    provider_code: str = PROVIDER_SUCCESS_0,
+) -> SingleAttemptResult:
+    return SingleAttemptResult(
+        request_fingerprint=WARNING_REQUEST_FINGERPRINT,
+        attempt_result=ATTEMPT_SUCCEEDED,
+        http_status=200,
+        provider_code=provider_code,
+        response_sha256=hashlib.sha256(payload).hexdigest(),
+        byte_count=len(payload),
+        body=payload,
+    )
+
+
+def retryable_result(
+    failure_code: str,
+    status: int | None,
+) -> SingleAttemptResult:
+    return SingleAttemptResult(
+        request_fingerprint=WARNING_REQUEST_FINGERPRINT,
+        attempt_result=ATTEMPT_RETRYABLE_FAILED,
+        http_status=status,
+        failure_code=failure_code,
+    )
+
+
+def terminal_result() -> SingleAttemptResult:
+    return SingleAttemptResult(
+        request_fingerprint=WARNING_REQUEST_FINGERPRINT,
+        attempt_result=ATTEMPT_TERMINAL_FAILED,
+        failure_code=FAILURE_TRANSPORT,
+    )
+
+
+def provider_error_result() -> SingleAttemptResult:
+    return SingleAttemptResult(
+        request_fingerprint=WARNING_REQUEST_FINGERPRINT,
+        attempt_result=ATTEMPT_TERMINAL_FAILED,
+        http_status=200,
+        provider_code=PROVIDER_OTHER_ERROR,
+        failure_code=FAILURE_PROVIDER_ERROR,
+    )
+
+
+class RecordingSecretLoader:
+    def __init__(self, value=SYNTHETIC_CREDENTIAL):
+        self.value = value
+        self.reference_names = []
+
+    def __call__(self, reference_name):
+        if connection.in_atomic_block:
+            raise AssertionError("credential lookup called inside a transaction")
+        if not FetchAttempt.objects.filter(
+            result=FetchAttempt.Result.STARTED
+        ).exists():
+            raise AssertionError("STARTED receipt was not durable before lookup")
+        self.reference_names.append(reference_name)
+        return self.value
+
+    def __repr__(self):
+        return "RecordingSecretLoader(redacted=True)"
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
+        if type(kwargs.get("secret_value")) is not str:
+            raise AssertionError("credential value boundary was not injected")
+        self.calls.append((frozenset(kwargs), tuple(started)))
+        return self.results.pop(0)
+
+
+class TravelWarningIngestionTests(TransactionTestCase):
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
+        register_approved_sources(apply=True)
+
+    def loader(self, value=SYNTHETIC_CREDENTIAL):
+        return RecordingSecretLoader(value)
+
+    def open_second_session(self):
+        return psycopg.connect(**connection.get_connection_params())
+
+    def assert_warning_lock_available(self):
+        contender = self.open_second_session()
+        try:
+            with contender.cursor() as cursor:
+                cursor.execute(
+                    "SELECT pg_try_advisory_lock(%s, %s)",
+                    [
+                        WARNING_INGESTION_LOCK_NAMESPACE,
+                        WARNING_INGESTION_LOCK_KEY,
+                    ],
+                )
+                self.assertIs(cursor.fetchone()[0], True)
+                cursor.execute(
+                    "SELECT pg_advisory_unlock(%s, %s)",
+                    [
+                        WARNING_INGESTION_LOCK_NAMESPACE,
+                        WARNING_INGESTION_LOCK_KEY,
+                    ],
+                )
+                self.assertIs(cursor.fetchone()[0], True)
+        finally:
+            contender.close()
+
+    def reject_warning_rights(self):
+        source = SourceConfiguration.objects.get(
+            code="MOFA_TRAVEL_ALARM_API_JP"
+        )
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
+    def test_positive_closes_receipt_and_creates_atomic_review_candidate(self):
+        payload = warning_payload()
+        transport = SequenceTransport(successful_result(payload))
+        loader = self.loader()
+
+        outcome = ingest_travel_warning(
+            transport=transport,
+            secret_loader=loader,
+        )
+
+        self.assertEqual(
+            outcome,
+            TravelWarningIngestionOutcome(
+                TravelWarningIngestionCode.REVIEW_REQUIRED,
+                1,
+            ),
+        )
+        self.assertTrue(outcome.succeeded)
+        self.assertEqual(loader.reference_names, [WARNING_SECRET_REFERENCE])
+        self.assertEqual(len(transport.calls), 1)
+        names, receipts_at_call = transport.calls[0]
+        self.assertEqual(
+            names,
+            {
+                "official_locator",
+                "secret_reference_name",
+                "secret_value",
+                "connect_timeout_seconds",
+                "read_timeout_seconds",
+            },
+        )
+        self.assertEqual(receipts_at_call, ((1, FetchAttempt.Result.STARTED),))
+
+        attempt = FetchAttempt.objects.get()
+        artifact = SourceArtifact.objects.get()
+        run = ParseRun.objects.get()
+        revision = TravelWarningRevision.objects.get()
+        self.assertEqual(attempt.result, FetchAttempt.Result.SUCCEEDED)
+        self.assertEqual(
+            attempt.provider_code,
+            FetchAttempt.ProviderCode.MOFA_SUCCESS_0,
+        )
+        self.assertEqual(
+            attempt.response_sha256,
+            hashlib.sha256(payload).hexdigest(),
+        )
+        self.assertEqual(artifact.first_successful_attempt_id, attempt.id)
+        self.assertEqual(artifact.byte_count, len(payload))
+        self.assertEqual(artifact.state, SourceArtifact.State.REVIEW_REQUIRED)
+        self.assertEqual(run.outcome, ParseRun.Outcome.VALIDATED)
+        self.assertEqual(revision.parse_run_id, run.id)
+        self.assertEqual(revision.first_observed_at, attempt.completed_at)
+        self.assertEqual(revision.source_alarm_level_code, ALARM_LEVEL)
+        self.assertEqual(revision.source_scope_type, SCOPE_TYPE)
+        self.assertEqual(revision.source_scope_text, SCOPE_TEXT)
+        self.assert_warning_lock_available()
+
+    def test_same_body_replay_adds_only_fresh_attempt_evidence(self):
+        payload = warning_payload()
+        first = ingest_travel_warning(
+            transport=SequenceTransport(successful_result(payload)),
+            secret_loader=self.loader(),
+        )
+        artifact = SourceArtifact.objects.get()
+        run = ParseRun.objects.get()
+        revision = TravelWarningRevision.objects.get()
+
+        second = ingest_travel_warning(
+            transport=SequenceTransport(successful_result(payload)),
+            secret_loader=self.loader(),
+        )
+
+        self.assertEqual(first.code, TravelWarningIngestionCode.REVIEW_REQUIRED)
+        self.assertEqual(second.code, TravelWarningIngestionCode.REPLAY_OBSERVED)
+        self.assertEqual(FetchAttempt.objects.count(), 2)
+        self.assertEqual(SourceArtifact.objects.count(), 1)
+        self.assertEqual(ParseRun.objects.count(), 1)
+        self.assertEqual(TravelWarningRevision.objects.count(), 1)
+        self.assertEqual(SourceArtifact.objects.get().id, artifact.id)
+        self.assertEqual(ParseRun.objects.get().id, run.id)
+        self.assertEqual(TravelWarningRevision.objects.get().id, revision.id)
+        self.assert_warning_lock_available()
+
+    def test_retry_allowlist_is_contiguous_and_bounded(self):
+        payload = warning_payload()
+        transport = SequenceTransport(
+            retryable_result(FAILURE_TIMEOUT, None),
+            retryable_result(FAILURE_UPSTREAM_5XX, 503),
+            successful_result(payload),
+        )
+        waits = []
+
+        outcome = ingest_travel_warning(
+            transport=transport,
+            retry_wait=waits.append,
+            secret_loader=self.loader(),
+        )
+
+        self.assertEqual(
+            outcome,
+            TravelWarningIngestionOutcome(
+                TravelWarningIngestionCode.REVIEW_REQUIRED,
+                3,
+            ),
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
+        self.assertEqual(waits, [1, 2])
+
+    def test_retryable_outage_stops_after_max_plus_one(self):
+        transport = SequenceTransport(
+            retryable_result(FAILURE_RATE_LIMITED, 429),
+            retryable_result(FAILURE_TIMEOUT, None),
+            retryable_result(FAILURE_UPSTREAM_5XX, 500),
+        )
+        waits = []
+
+        outcome = ingest_travel_warning(
+            transport=transport,
+            retry_wait=waits.append,
+            secret_loader=self.loader(),
+        )
+
+        self.assertEqual(
+            outcome,
+            TravelWarningIngestionOutcome(
+                TravelWarningIngestionCode.FETCH_RETRIES_EXHAUSTED,
+                3,
+            ),
+        )
+        self.assertEqual(FetchAttempt.objects.count(), 3)
+        self.assertFalse(
+            FetchAttempt.objects.filter(result=FetchAttempt.Result.STARTED).exists()
+        )
+        self.assertFalse(SourceArtifact.objects.exists())
+        self.assertEqual(waits, [1, 2])
+        self.assert_warning_lock_available()
+
+    def test_missing_secret_and_provider_error_are_closed_negative_receipts(self):
+        missing = ingest_travel_warning(
+            transport=lambda **kwargs: self.fail("transport must not run"),
+            secret_loader=self.loader(None),
+        )
+        provider_error = ingest_travel_warning(
+            transport=SequenceTransport(provider_error_result()),
+            secret_loader=self.loader(),
+        )
+
+        self.assertEqual(
+            missing.code,
+            TravelWarningIngestionCode.FETCH_TERMINAL_FAILED,
+        )
+        self.assertEqual(
+            provider_error.code,
+            TravelWarningIngestionCode.FETCH_TERMINAL_FAILED,
+        )
+        attempts = list(FetchAttempt.objects.order_by("started_at", "id"))
+        self.assertEqual(
+            {attempt.failure_code for attempt in attempts},
+            {FAILURE_AUTHENTICATION, FAILURE_PROVIDER_ERROR},
+        )
+        self.assertEqual(
+            {attempt.provider_code for attempt in attempts},
+            {"", PROVIDER_OTHER_ERROR},
+        )
+        self.assertFalse(SourceArtifact.objects.exists())
+        self.assert_warning_lock_available()
+
+    def test_schema_drift_is_quarantined_with_no_typed_revision(self):
+        drifted_item = warning_item()
+        drifted_item["synthetic_extra"] = "value"
+        payload = warning_payload(item=drifted_item)
+
+        outcome = ingest_travel_warning(
+            transport=SequenceTransport(successful_result(payload)),
+            secret_loader=self.loader(),
+        )
+
+        self.assertEqual(
+            outcome.code,
+            TravelWarningIngestionCode.PARSE_QUARANTINED,
+        )
+        artifact = SourceArtifact.objects.get()
+        run = ParseRun.objects.get()
+        self.assertEqual(artifact.state, SourceArtifact.State.REJECTED)
+        self.assertEqual(run.outcome, ParseRun.Outcome.QUARANTINED)
+        self.assertEqual(run.failure_code, ParseRun.FailureCode.SCHEMA_MISMATCH)
+        self.assertFalse(TravelWarningRevision.objects.exists())
+        self.assert_warning_lock_available()
+
+    def test_parser_exception_is_closed_as_failed_and_releases_lock(self):
+        payload = warning_payload()
+
+        def failed_parser(body):
+            self.assertEqual(body, payload)
+            raise RuntimeError("private parser detail")
+
+        outcome = ingest_travel_warning(
+            transport=SequenceTransport(successful_result(payload)),
+            parser=failed_parser,
+            secret_loader=self.loader(),
+        )
+
+        self.assertEqual(outcome.code, TravelWarningIngestionCode.PARSE_FAILED)
+        artifact = SourceArtifact.objects.get()
+        parse_run = ParseRun.objects.get()
+        self.assertEqual(artifact.state, SourceArtifact.State.REJECTED)
+        self.assertEqual(parse_run.outcome, ParseRun.Outcome.FAILED)
+        self.assertEqual(
+            parse_run.failure_code,
+            ParseRun.FailureCode.INTERNAL_PARSER_ERROR,
+        )
+        self.assertFalse(TravelWarningRevision.objects.exists())
+        self.assert_warning_lock_available()
+
+    def test_replay_of_rejected_artifact_fails_closed(self):
+        drifted_item = warning_item()
+        drifted_item["synthetic_extra"] = "value"
+        payload = warning_payload(item=drifted_item)
+        first = ingest_travel_warning(
+            transport=SequenceTransport(successful_result(payload)),
+            secret_loader=self.loader(),
+        )
+        second = ingest_travel_warning(
+            transport=SequenceTransport(successful_result(payload)),
+            secret_loader=self.loader(),
+        )
+
+        self.assertEqual(first.code, TravelWarningIngestionCode.PARSE_QUARANTINED)
+        self.assertEqual(second.code, TravelWarningIngestionCode.EVIDENCE_CONFLICT)
+        self.assertEqual(FetchAttempt.objects.count(), 2)
+        self.assertEqual(SourceArtifact.objects.count(), 1)
+        self.assertEqual(ParseRun.objects.count(), 1)
+        self.assertFalse(TravelWarningRevision.objects.exists())
+        self.assert_warning_lock_available()
+
+    def test_rights_are_checked_before_fetch_and_again_after_network(self):
+        self.reject_warning_rights()
+        before = ingest_travel_warning(
+            transport=lambda **kwargs: self.fail("transport must not run"),
+            secret_loader=lambda name: self.fail("loader must not run"),
+        )
+
+        self.assertEqual(before.code, TravelWarningIngestionCode.SOURCE_GATE_FAILED)
+        self.assertEqual(before.attempt_count, 0)
+        self.assertFalse(FetchAttempt.objects.exists())
+        self.assert_warning_lock_available()
+
+    def test_midflight_rights_rejection_blocks_candidate_persistence(self):
+        payload = warning_payload()
+
+        def revoke_then_succeed(**kwargs):
+            self.assertFalse(connection.in_atomic_block)
+            self.reject_warning_rights()
+            return successful_result(payload)
+
+        outcome = ingest_travel_warning(
+            transport=revoke_then_succeed,
+            secret_loader=self.loader(),
+        )
+
+        self.assertEqual(outcome.code, TravelWarningIngestionCode.SOURCE_GATE_FAILED)
+        self.assertEqual(FetchAttempt.objects.get().result, FetchAttempt.Result.SUCCEEDED)
+        self.assertFalse(SourceArtifact.objects.exists())
+        self.assertFalse(ParseRun.objects.exists())
+        self.assertFalse(TravelWarningRevision.objects.exists())
+        self.assert_warning_lock_available()
+
+    def test_rights_are_rechecked_immediately_before_the_typed_insert(self):
+        payload = warning_payload()
+
+        def reject_during_parse(body):
+            parsed = parse_travel_alarm_jp(body)
+            self.reject_warning_rights()
+            return parsed
+
+        outcome = ingest_travel_warning(
+            transport=SequenceTransport(successful_result(payload)),
+            parser=reject_during_parse,
+            secret_loader=self.loader(),
+        )
+
+        self.assertEqual(outcome.code, TravelWarningIngestionCode.SOURCE_GATE_FAILED)
+        self.assertEqual(FetchAttempt.objects.get().result, FetchAttempt.Result.SUCCEEDED)
+        self.assertFalse(SourceArtifact.objects.exists())
+        self.assertFalse(ParseRun.objects.exists())
+        self.assertFalse(TravelWarningRevision.objects.exists())
+        source = SourceConfiguration.objects.get(
+            code="MOFA_TRAVEL_ALARM_API_JP"
+        )
+        self.assertEqual(source.state, SourceConfiguration.State.ACTIVE)
+        self.assertEqual(source.rights_decisions.count(), 1)
+
+    def test_typed_insert_failure_rolls_back_the_candidate_boundary(self):
+        payload = warning_payload()
+        with patch.object(
+            TravelWarningRevision.objects,
+            "create",
+            side_effect=RuntimeError("private rollback detail"),
+        ):
+            outcome = ingest_travel_warning(
+                transport=SequenceTransport(successful_result(payload)),
+                secret_loader=self.loader(),
+            )
+
+        self.assertEqual(outcome.code, TravelWarningIngestionCode.PERSISTENCE_FAILED)
+        self.assertEqual(FetchAttempt.objects.get().result, FetchAttempt.Result.SUCCEEDED)
+        self.assertFalse(SourceArtifact.objects.exists())
+        self.assertFalse(ParseRun.objects.exists())
+        self.assertFalse(TravelWarningRevision.objects.exists())
+        self.assert_warning_lock_available()
+
+    def test_abandoned_receipt_is_closed_before_a_new_operation(self):
+        source = SourceConfiguration.objects.get(
+            code="MOFA_TRAVEL_ALARM_API_JP"
+        )
+        rights = source.rights_decisions.get(decision_sequence=1)
+        abandoned = FetchAttempt.objects.create(
+            source=source,
+            source_revision=source.revision,
+            rights_decision=rights,
+            operation_id=uuid.uuid4(),
+            attempt_number=1,
+            request_fingerprint_revision=WARNING_REQUEST_FINGERPRINT.revision,
+            normalized_request_sha256=(
+                WARNING_REQUEST_FINGERPRINT.normalized_request_sha256
+            ),
+        )
+
+        outcome = ingest_travel_warning(
+            transport=SequenceTransport(terminal_result()),
+            secret_loader=self.loader(),
+        )
+
+        self.assertEqual(
+            outcome.code,
+            TravelWarningIngestionCode.FETCH_TERMINAL_FAILED,
+        )
+        abandoned.refresh_from_db()
+        self.assertEqual(abandoned.result, FetchAttempt.Result.TERMINAL_FAILED)
+        self.assertEqual(
+            abandoned.failure_code,
+            FetchAttempt.FailureCode.WORKER_INTERRUPTED,
+        )
+        self.assertFalse(
+            FetchAttempt.objects.filter(result=FetchAttempt.Result.STARTED).exists()
+        )
+
+    def test_overlap_stops_before_receipt_secret_lookup_or_transport(self):
+        with patch(
+            "travel_warnings.ingestion._try_acquire_warning_ingestion_lock",
+            return_value=False,
+        ):
+            outcome = ingest_travel_warning(
+                transport=lambda **kwargs: self.fail("transport must not run"),
+                secret_loader=lambda name: self.fail("loader must not run"),
+            )
+
+        self.assertEqual(
+            outcome,
+            TravelWarningIngestionOutcome(
+                TravelWarningIngestionCode.ALREADY_RUNNING,
+                0,
+            ),
+        )
+        self.assertFalse(FetchAttempt.objects.exists())
+
+    def test_lock_database_failure_is_not_reported_as_overlap(self):
+        with patch(
+            "travel_warnings.ingestion._try_acquire_warning_ingestion_lock",
+            side_effect=RuntimeError("private database detail"),
+        ):
+            outcome = ingest_travel_warning(
+                transport=lambda **kwargs: self.fail("transport must not run"),
+                secret_loader=lambda name: self.fail("loader must not run"),
+            )
+
+        self.assertEqual(
+            outcome,
+            TravelWarningIngestionOutcome(
+                TravelWarningIngestionCode.PERSISTENCE_FAILED,
+                0,
+            ),
+        )
+        self.assertFalse(FetchAttempt.objects.exists())
+
+    def test_non_autocommit_session_fails_before_receipt_or_external_boundary(self):
+        connection.set_autocommit(False)
+        try:
+            outcome = ingest_travel_warning(
+                transport=lambda **kwargs: self.fail("transport must not run"),
+                secret_loader=lambda name: self.fail("loader must not run"),
+            )
+        finally:
+            connection.close()
+            connection.ensure_connection()
+
+        self.assertIs(connection.get_autocommit(), True)
+        self.assertEqual(
+            outcome,
+            TravelWarningIngestionOutcome(
+                TravelWarningIngestionCode.PERSISTENCE_FAILED,
+                0,
+            ),
+        )
+        self.assertFalse(FetchAttempt.objects.exists())
+
+    def test_postgresql_session_lock_blocks_a_second_ingestion_session(self):
+        contender = self.open_second_session()
+        try:
+            with contender.cursor() as cursor:
+                cursor.execute(
+                    "SELECT pg_try_advisory_lock(%s, %s)",
+                    [
+                        WARNING_INGESTION_LOCK_NAMESPACE,
+                        WARNING_INGESTION_LOCK_KEY,
+                    ],
+                )
+                self.assertIs(cursor.fetchone()[0], True)
+
+            outcome = ingest_travel_warning(
+                transport=lambda **kwargs: self.fail("transport must not run"),
+                secret_loader=lambda name: self.fail("loader must not run"),
+            )
+
+            self.assertEqual(
+                outcome,
+                TravelWarningIngestionOutcome(
+                    TravelWarningIngestionCode.ALREADY_RUNNING,
+                    0,
+                ),
+            )
+            self.assertFalse(FetchAttempt.objects.exists())
+            with contender.cursor() as cursor:
+                cursor.execute(
+                    "SELECT pg_advisory_unlock(%s, %s)",
+                    [
+                        WARNING_INGESTION_LOCK_NAMESPACE,
+                        WARNING_INGESTION_LOCK_KEY,
+                    ],
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
+                            WARNING_INGESTION_LOCK_NAMESPACE,
+                            WARNING_INGESTION_LOCK_KEY,
+                        ],
+                    )
+                    observed.append(cursor.fetchone()[0])
+            finally:
+                contender.close()
+            return terminal_result()
+
+        outcome = ingest_travel_warning(
+            transport=inspect_lock_then_fail,
+            secret_loader=self.loader(),
+        )
+
+        self.assertEqual(
+            outcome.code,
+            TravelWarningIngestionCode.FETCH_TERMINAL_FAILED,
+        )
+        self.assertEqual(observed, [False])
+        self.assert_warning_lock_available()
+
+    def test_unlock_database_failure_discards_the_session(self):
+        with patch.object(
+            warning_ingestion.connection,
+            "cursor",
+            side_effect=RuntimeError("private unlock detail"),
+        ), patch.object(warning_ingestion.connection, "close") as close:
+            warning_ingestion._release_warning_ingestion_lock()
+
+        close.assert_called_once_with()
+
+    def test_transport_base_exception_scrubs_secret_raw_and_callback_frames(self):
+        secret_marker = "private-secret-value-marker"
+        raw_marker = "private-transport-interruption-marker"
+
+        class SensitiveLoader:
+            def __call__(self, reference_name):
+                return secret_marker
+
+            def __repr__(self):
+                return secret_marker
+
+        class InterruptedTransport:
+            def __call__(self, **kwargs):
+                self.assertion(kwargs["secret_value"], secret_marker)
+                raise KeyboardInterrupt(raw_marker)
+
+            def assertion(self, actual, expected):
+                if actual != expected:
+                    raise AssertionError("injected credential changed")
+
+            def __repr__(self):
+                return raw_marker
+
+        with self.assertRaises(KeyboardInterrupt) as raised:
+            ingest_travel_warning(
+                transport=InterruptedTransport(),
+                secret_loader=SensitiveLoader(),
+            )
+
+        self.assertEqual(raised.exception.args, ())
+        assert_exception_boundary_is_redacted(
+            self,
+            raised.exception,
+            secret_marker,
+        )
+        assert_exception_boundary_is_redacted(
+            self,
+            raised.exception,
+            raw_marker,
+        )
+
+        first = FetchAttempt.objects.get()
+        self.assertEqual(first.result, FetchAttempt.Result.TERMINAL_FAILED)
+        self.assertEqual(
+            first.failure_code,
+            FetchAttempt.FailureCode.WORKER_INTERRUPTED,
+        )
+        self.assert_warning_lock_available()
+
+    def test_create_interruption_reconciles_committed_started_receipt(self):
+        marker = "private-create-interruption-marker"
+        original_create = warning_ingestion._create_started_attempt
+
+        def create_then_interrupt(*args, **kwargs):
+            original_create(*args, **kwargs)
+            raise KeyboardInterrupt(marker)
+
+        with patch(
+            "travel_warnings.ingestion._create_started_attempt",
+            side_effect=create_then_interrupt,
+        ):
+            with self.assertRaises(KeyboardInterrupt) as raised:
+                ingest_travel_warning(
+                    transport=lambda **kwargs: self.fail("transport must not run"),
+                    secret_loader=lambda name: self.fail("loader must not run"),
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
+        self.assert_warning_lock_available()
+
+    def test_secret_loader_base_exception_is_sanitized_after_terminal_receipt(self):
+        marker = "private-loader-interruption-marker"
+
+        class InterruptedLoader:
+            def __call__(self, reference_name):
+                raise KeyboardInterrupt(marker)
+
+            def __repr__(self):
+                return marker
+
+        with self.assertRaises(KeyboardInterrupt) as raised:
+            ingest_travel_warning(
+                transport=lambda **kwargs: self.fail("transport must not run"),
+                secret_loader=InterruptedLoader(),
+            )
+
+        self.assertEqual(raised.exception.args, ())
+        assert_exception_boundary_is_redacted(self, raised.exception, marker)
+        attempt = FetchAttempt.objects.get()
+        self.assertEqual(attempt.result, FetchAttempt.Result.TERMINAL_FAILED)
+        self.assertEqual(
+            attempt.failure_code,
+            FetchAttempt.FailureCode.WORKER_INTERRUPTED,
+        )
+        self.assert_warning_lock_available()
+
+    def test_parser_base_exception_scrubs_raw_and_rolls_back_candidate(self):
+        marker = "private-parser-raw-body-marker"
+        payload = warning_payload(item=warning_item(remark=marker))
+
+        class InterruptedParser:
+            def __call__(self, body):
+                if body != payload:
+                    raise AssertionError("parser body changed")
+                raise KeyboardInterrupt(marker)
+
+            def __repr__(self):
+                return marker
+
+        with self.assertRaises(KeyboardInterrupt) as raised:
+            ingest_travel_warning(
+                transport=SequenceTransport(successful_result(payload)),
+                parser=InterruptedParser(),
+                secret_loader=self.loader(),
+            )
+
+        self.assertEqual(raised.exception.args, ())
+        assert_exception_boundary_is_redacted(self, raised.exception, marker)
+        self.assertEqual(
+            FetchAttempt.objects.get().result,
+            FetchAttempt.Result.SUCCEEDED,
+        )
+        self.assertFalse(SourceArtifact.objects.exists())
+        self.assertFalse(ParseRun.objects.exists())
+        self.assertFalse(TravelWarningRevision.objects.exists())
+        self.assert_warning_lock_available()
+
+    def test_close_base_exception_is_sanitized_after_receipt_reconciliation(self):
+        marker = "private-close-interruption-marker"
+        with patch(
+            "travel_warnings.ingestion._close_attempt",
+            side_effect=BaseException(marker),
+        ):
+            with self.assertRaises(BaseException) as raised:
+                ingest_travel_warning(
+                    transport=SequenceTransport(terminal_result()),
+                    secret_loader=self.loader(),
+                )
+
+        self.assertIs(type(raised.exception), BaseException)
+        self.assertEqual(raised.exception.args, ())
+        assert_exception_boundary_is_redacted(self, raised.exception, marker)
+        attempt = FetchAttempt.objects.get()
+        self.assertEqual(attempt.result, FetchAttempt.Result.TERMINAL_FAILED)
+        self.assertEqual(
+            attempt.failure_code,
+            FetchAttempt.FailureCode.WORKER_INTERRUPTED,
+        )
+        self.assert_warning_lock_available()
+
+    def test_close_failure_is_reconciled_to_a_terminal_receipt(self):
+        with patch(
+            "travel_warnings.ingestion._close_attempt",
+            side_effect=RuntimeError("private close detail"),
+        ):
+            outcome = ingest_travel_warning(
+                transport=SequenceTransport(terminal_result()),
+                secret_loader=self.loader(),
+            )
+
+        self.assertEqual(outcome.code, TravelWarningIngestionCode.PERSISTENCE_FAILED)
+        attempt = FetchAttempt.objects.get()
+        self.assertEqual(attempt.result, FetchAttempt.Result.TERMINAL_FAILED)
+        self.assertEqual(
+            attempt.failure_code,
+            FetchAttempt.FailureCode.WORKER_INTERRUPTED,
+        )
+
+    def test_malformed_success_provider_shape_is_terminal_and_not_parsed(self):
+        payload = warning_payload()
+        outcome = ingest_travel_warning(
+            transport=SequenceTransport(
+                successful_result(payload, provider_code="")
+            ),
+            secret_loader=self.loader(),
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
+        self.assertEqual(attempt.response_sha256, "")
+        self.assertFalse(SourceArtifact.objects.exists())
+
+    def test_malformed_status_and_result_type_each_close_terminal_receipt(self):
+        malformed_status = SingleAttemptResult(
+            request_fingerprint=WARNING_REQUEST_FINGERPRINT,
+            attempt_result=ATTEMPT_RETRYABLE_FAILED,
+            http_status="429",
+            failure_code=FAILURE_RATE_LIMITED,
+        )
+
+        status_outcome = ingest_travel_warning(
+            transport=SequenceTransport(malformed_status),
+            secret_loader=self.loader(),
+        )
+        type_outcome = ingest_travel_warning(
+            transport=SequenceTransport(object()),
+            secret_loader=self.loader(),
+        )
+
+        self.assertEqual(
+            status_outcome.code,
+            TravelWarningIngestionCode.FETCH_TERMINAL_FAILED,
+        )
+        self.assertEqual(
+            type_outcome.code,
+            TravelWarningIngestionCode.FETCH_TERMINAL_FAILED,
+        )
+        attempts = list(FetchAttempt.objects.order_by("started_at", "id"))
+        self.assertEqual(len(attempts), 2)
+        self.assertEqual(
+            {attempt.result for attempt in attempts},
+            {FetchAttempt.Result.TERMINAL_FAILED},
+        )
+        self.assertEqual(
+            {attempt.failure_code for attempt in attempts},
+            {FetchAttempt.FailureCode.WORKER_INTERRUPTED},
+        )
+        self.assert_warning_lock_available()
+
+    def test_secret_loader_exception_is_redacted_and_closes_the_receipt(self):
+        marker = "private-loader-exception-marker"
+
+        def raising_loader(reference_name):
+            raise RuntimeError(marker)
+
+        outcome = ingest_travel_warning(
+            transport=lambda **kwargs: self.fail("transport must not run"),
+            secret_loader=raising_loader,
+        )
+
+        self.assertEqual(
+            outcome.code,
+            TravelWarningIngestionCode.FETCH_TERMINAL_FAILED,
+        )
+        self.assertNotIn(marker, repr(outcome))
+        attempt = FetchAttempt.objects.get()
+        self.assertEqual(attempt.result, FetchAttempt.Result.TERMINAL_FAILED)
+        self.assertEqual(
+            attempt.failure_code,
+            FetchAttempt.FailureCode.WORKER_INTERRUPTED,
+        )
+
+    def test_secret_and_raw_bytes_are_absent_from_repr_and_persistence(self):
+        payload = warning_payload()
+        raw_marker = b"private-raw-response-plaintext-marker"
+        secret_marker = "private-secret-plaintext-marker"
+        outcome = ingest_travel_warning(
+            transport=SequenceTransport(successful_result(payload)),
+            secret_loader=self.loader(),
+        )
+
+        self.assertNotIn(SYNTHETIC_CREDENTIAL, repr(outcome))
+        self.assertNotIn(payload.hex(), repr(outcome))
+        self.assertNotIn(SYNTHETIC_CREDENTIAL, repr(_LoadedSecret(SYNTHETIC_CREDENTIAL)))
+        self.assertNotIn(secret_marker, repr(_LoadedSecret(secret_marker)))
+        verified = _verify_transport_result(successful_result(payload))
+        self.assertNotIn(payload.hex(), repr(verified))
+        raw_verified = _verify_transport_result(successful_result(raw_marker))
+        self.assertNotIn(raw_marker.decode("ascii"), repr(raw_verified))
+        self.assertEqual(
+            set(inspect.signature(ingest_travel_warning).parameters),
+            {"transport", "parser", "retry_wait", "secret_loader"},
+        )
+        banned = {
+            "raw",
+            "secret_value",
+            "credential_value",
+            "destination",
+            "departure",
+            "return_date",
+            "session",
+            "purpose",
+        }
+        for model in (
+            FetchAttempt,
+            SourceArtifact,
+            ParseRun,
+            TravelWarningRevision,
+        ):
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
+        forbidden_fragments = {
+            SYNTHETIC_CREDENTIAL[index : index + 8]
+            for index in range(len(SYNTHETIC_CREDENTIAL) - 7)
+        }
+        raw_text = payload.decode("utf-8")
+        for model in (
+            FetchAttempt,
+            SourceArtifact,
+            ParseRun,
+            TravelWarningRevision,
+        ):
+            for row in model.objects.values():
+                for value in row.values():
+                    if not isinstance(value, str):
+                        continue
+                    self.assertNotEqual(value, raw_text)
+                    self.assertFalse(
+                        any(fragment in value for fragment in forbidden_fragments)
+                    )
+
+
+class TravelWarningSecretLoaderTests(SimpleTestCase):
+    def test_default_loader_reads_only_the_requested_environment_name(self):
+        class EnvironmentBoundary:
+            def __init__(self):
+                self.requested = []
+
+            def get(self, name):
+                self.requested.append(name)
+                return SYNTHETIC_CREDENTIAL
+
+        boundary = EnvironmentBoundary()
+        with patch("travel_warnings.ingestion.os.environ", boundary):
+            value = _environment_secret_loader(WARNING_SECRET_REFERENCE)
+
+        self.assertEqual(value, SYNTHETIC_CREDENTIAL)
+        self.assertEqual(boundary.requested, [WARNING_SECRET_REFERENCE])
+
+
+class TravelWarningIngestionCommandTests(SimpleTestCase):
+    def test_command_output_contains_only_fixed_code_and_attempt_count(self):
+        output = io.StringIO()
+        with patch(
+            "operations.management.commands.ingest_travel_warning."
+            "ingest_travel_warning",
+            return_value=TravelWarningIngestionOutcome(
+                TravelWarningIngestionCode.REVIEW_REQUIRED,
+                1,
+            ),
+        ):
+            call_command("ingest_travel_warning", stdout=output)
+        self.assertEqual(
+            output.getvalue(),
+            "travel_warning_ingestion_result=REVIEW_REQUIRED attempts=1\n",
+        )
+
+    def test_command_failure_contains_only_fixed_code_and_attempt_count(self):
+        with patch(
+            "operations.management.commands.ingest_travel_warning."
+            "ingest_travel_warning",
+            return_value=TravelWarningIngestionOutcome(
+                TravelWarningIngestionCode.FETCH_TERMINAL_FAILED,
+                1,
+            ),
+        ):
+            with self.assertRaises(CommandError) as raised:
+                call_command("ingest_travel_warning")
+        self.assertEqual(
+            str(raised.exception),
+            "travel_warning_ingestion_result=FETCH_TERMINAL_FAILED attempts=1",
+        )
+
+    def test_command_never_exposes_exception_text(self):
+        marker = "private-credential-and-body-marker"
+        with patch(
+            "operations.management.commands.ingest_travel_warning."
+            "ingest_travel_warning",
+            side_effect=RuntimeError(marker),
+        ):
+            with self.assertRaises(CommandError) as raised:
+                call_command("ingest_travel_warning")
+        self.assertEqual(
+            str(raised.exception),
+            "travel_warning_ingestion_result=PERSISTENCE_FAILED attempts=0",
+        )
+        self.assertNotIn(marker, str(raised.exception))
+        assert_exception_boundary_is_redacted(self, raised.exception, marker)
+
+    def test_command_never_exposes_base_exception_text_or_chain(self):
+        marker = "private-command-base-exception-marker"
+        with patch(
+            "operations.management.commands.ingest_travel_warning."
+            "ingest_travel_warning",
+            side_effect=KeyboardInterrupt(marker),
+        ):
+            with self.assertRaises(CommandError) as raised:
+                call_command("ingest_travel_warning")
+        self.assertEqual(
+            str(raised.exception),
+            "travel_warning_ingestion_result=PERSISTENCE_FAILED attempts=0",
+        )
+        self.assertNotIn(marker, str(raised.exception))
+        assert_exception_boundary_is_redacted(self, raised.exception, marker)


