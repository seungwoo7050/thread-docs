## `feat(source): orchestrate recent ingestion`

diff --git a/grocery/management/__init__.py b/grocery/management/__init__.py
new file mode 100644
index 0000000..024515b
--- /dev/null
+++ b/grocery/management/__init__.py
@@ -0,0 +1 @@
+"""Project management commands."""
diff --git a/grocery/management/commands/__init__.py b/grocery/management/commands/__init__.py
new file mode 100644
index 0000000..c61a410
--- /dev/null
+++ b/grocery/management/commands/__init__.py
@@ -0,0 +1 @@
+"""Grocery management commands."""
diff --git a/grocery/management/commands/ingest_kamis_recent.py b/grocery/management/commands/ingest_kamis_recent.py
new file mode 100644
index 0000000..fc5c15f
--- /dev/null
+++ b/grocery/management/commands/ingest_kamis_recent.py
@@ -0,0 +1,424 @@
+"""Run one bounded, secret-safe KAMIS recent-price ingestion generation."""
+
+from __future__ import annotations
+
+import logging
+import uuid
+from argparse import ArgumentTypeError
+from enum import StrEnum
+from typing import Final, NoReturn
+
+from django.core.management.base import BaseCommand, CommandError, CommandParser
+
+from grocery.models import FetchAttempt, ParseRun
+from grocery.observability import Severity, log_event
+from grocery.source.client import (
+    DEFAULT_PAGE_SIZE,
+    MAX_PAGE_SIZE,
+    KamisHttpClient,
+    KamisTransportError,
+)
+from grocery.source.configuration import bootstrap_kamis_source_configuration
+from grocery.source.generation import (
+    ParseGenerationError,
+    ParseGenerationErrorCode,
+    ParseGenerationFailureCode,
+    complete_kamis_parse_generation,
+    fail_kamis_parse_run,
+    start_or_get_kamis_parse_run,
+)
+from grocery.source.kamis import KamisParseError, parse_recent_price_rows
+from grocery.source.persistence import complete_kamis_fetch, fail_kamis_fetch, start_kamis_fetch
+from grocery.source.registry import INITIAL_RETAIL_IDENTITY_REGISTRY
+from grocery.source.secrets import SecretLoadError, load_kamis_api_key
+
+_LOGGER: Final = logging.getLogger("grocery.audit")
+
+_IDENTITY_PARSE_CODES: Final = frozenset(
+    {
+        "category_name_drift",
+        "coverage_identity_drift",
+        "grade_code_name_drift",
+        "item_code_name_drift",
+        "product_class_name_drift",
+        "unit_identity_drift",
+        "unsupported_category",
+        "unsupported_product_class",
+        "variety_code_name_drift",
+    }
+)
+
+
+class _CommandFailureCode(StrEnum):
+    SOURCE_START_FAILED = "INGEST_SOURCE_START_FAILED"
+    SECRET_UNAVAILABLE = "INGEST_SECRET_UNAVAILABLE"  # noqa: S105 - safe status code
+    FETCH_FAILED = "INGEST_FETCH_FAILED"
+    FETCH_FINALIZATION_FAILED = "INGEST_FETCH_FINALIZATION_FAILED"
+    FETCH_PERSISTENCE_FAILED = "INGEST_FETCH_PERSISTENCE_FAILED"
+    PARSE_START_FAILED = "INGEST_PARSE_START_FAILED"
+    PARSE_FAILED = "INGEST_PARSE_FAILED"
+    GENERATION_FAILED = "INGEST_GENERATION_FAILED"
+
+
+def _page_size(value: str) -> int:
+    try:
+        parsed = int(value)
+    except ValueError:
+        raise ArgumentTypeError("page_size_invalid") from None
+    if parsed < 1 or parsed > MAX_PAGE_SIZE:
+        raise ArgumentTypeError("page_size_invalid")
+    return parsed
+
+
+def _safe_option_page_size(value: object) -> int:
+    if not isinstance(value, int) or isinstance(value, bool) or not 1 <= value <= MAX_PAGE_SIZE:
+        raise CommandError("code=INGEST_PAGE_SIZE_INVALID")
+    return value
+
+
+def _failure_receipt(
+    code: _CommandFailureCode,
+    *,
+    attempt: FetchAttempt | None = None,
+    parse_run: ParseRun | None = None,
+) -> str:
+    fields = [f"code={code.value}"]
+    if attempt is not None:
+        fields.extend(
+            (
+                f"attempt_id={attempt.id}",
+                f"status={attempt.state}",
+                f"pages={attempt.received_page_count}",
+                f"rows={attempt.received_row_count}",
+            )
+        )
+    if parse_run is not None:
+        fields.extend(
+            (
+                f"parse_run_id={parse_run.id}",
+                f"parse_status={parse_run.status}",
+            )
+        )
+    return " ".join(fields)
+
+
+def _log_fetch(
+    command_run_id: uuid.UUID,
+    attempt: FetchAttempt,
+    *,
+    message_code: str,
+    lifecycle_event: str,
+    severity: Severity = "INFO",
+) -> None:
+    log_event(
+        _LOGGER,
+        severity,
+        message_code,
+        command_run_id=command_run_id,
+        lifecycle_id=attempt.id,
+        lifecycle_status=str(attempt.state),
+        lifecycle_event=lifecycle_event,
+    )
+
+
+def _log_parse(
+    command_run_id: uuid.UUID,
+    parse_run: ParseRun,
+    *,
+    message_code: str,
+    lifecycle_event: str,
+    severity: Severity = "INFO",
+) -> None:
+    log_event(
+        _LOGGER,
+        severity,
+        message_code,
+        command_run_id=command_run_id,
+        lifecycle_id=parse_run.id,
+        lifecycle_status=str(parse_run.status),
+        lifecycle_event=lifecycle_event,
+    )
+
+
+def _finalize_fetch_failure(
+    command_run_id: uuid.UUID,
+    attempt: FetchAttempt,
+    error: KamisTransportError,
+    *,
+    command_code: _CommandFailureCode,
+) -> NoReturn:
+    try:
+        failed = fail_kamis_fetch(attempt.id, error)
+    except Exception:
+        log_event(
+            _LOGGER,
+            "ERROR",
+            "ingest.fetch.finalization_failed",
+            command_run_id=command_run_id,
+            lifecycle_id=attempt.id,
+            lifecycle_status=FetchAttempt.State.STARTED,
+            lifecycle_event="FETCH_FINALIZATION_FAILED",
+        )
+        raise CommandError(
+            _failure_receipt(_CommandFailureCode.FETCH_FINALIZATION_FAILED, attempt=attempt)
+        ) from None
+    _log_fetch(
+        command_run_id,
+        failed,
+        message_code="ingest.fetch.failed",
+        lifecycle_event="FETCH_FAILED",
+        severity="ERROR",
+    )
+    raise CommandError(_failure_receipt(command_code, attempt=failed)) from None
+
+
+def _parse_failure_code(error: KamisParseError) -> ParseGenerationFailureCode:
+    if error.code in _IDENTITY_PARSE_CODES:
+        return ParseGenerationFailureCode.IDENTITY_DRIFT
+    if error.code == "duplicate_semantic_identity":
+        return ParseGenerationFailureCode.RECONCILIATION_FAILED
+    return ParseGenerationFailureCode.SCHEMA_INVALID
+
+
+def _generation_failure_code(error: ParseGenerationError) -> ParseGenerationFailureCode:
+    if error.code in {
+        ParseGenerationErrorCode.NONDETERMINISTIC_REPLAY,
+        ParseGenerationErrorCode.RESULT_RECONCILIATION_FAILED,
+    }:
+        return ParseGenerationFailureCode.RECONCILIATION_FAILED
+    return ParseGenerationFailureCode.PERSISTENCE_FAILED
+
+
+def _finalize_parse_failure(
+    command_run_id: uuid.UUID,
+    parse_run: ParseRun,
+    failure_code: ParseGenerationFailureCode,
+    *,
+    command_code: _CommandFailureCode,
+) -> NoReturn:
+    failed: ParseRun | None = None
+    try:
+        failed = fail_kamis_parse_run(parse_run.id, failure_code)
+    except Exception:
+        # A replay may already be terminal. Never replace the original safe failure
+        # with exception text from the ORM or parser boundary.
+        try:
+            candidate = ParseRun.objects.only("id", "status").get(pk=parse_run.id)
+        except Exception:
+            candidate = parse_run
+        if candidate.status in {
+            ParseRun.Status.VALIDATED,
+            ParseRun.Status.QUARANTINED,
+            ParseRun.Status.FAILED,
+        }:
+            failed = candidate
+    if failed is None:
+        _log_parse(
+            command_run_id,
+            parse_run,
+            message_code="ingest.parse.finalization_failed",
+            lifecycle_event="PARSE_FINALIZATION_FAILED",
+            severity="ERROR",
+        )
+        raise CommandError(
+            _failure_receipt(_CommandFailureCode.GENERATION_FAILED, parse_run=parse_run)
+        ) from None
+    _log_parse(
+        command_run_id,
+        failed,
+        message_code="ingest.parse.failed",
+        lifecycle_event="PARSE_FAILED",
+        severity="ERROR",
+    )
+    raise CommandError(_failure_receipt(command_code, parse_run=failed)) from None
+
+
+class Command(BaseCommand):
+    help = "Fetch and persist one bounded KAMIS recent-price generation."
+
+    def add_arguments(self, parser: CommandParser) -> None:
+        parser.add_argument(
+            "--page-size",
+            default=DEFAULT_PAGE_SIZE,
+            type=_page_size,
+            help=f"Rows per source page (1..{MAX_PAGE_SIZE}).",
+        )
+
+    def handle(self, *args: object, **options: object) -> None:
+        del args
+        page_size = _safe_option_page_size(options.get("page_size", DEFAULT_PAGE_SIZE))
+        command_run_id = uuid.uuid4()
+
+        try:
+            source = bootstrap_kamis_source_configuration()
+            attempt = start_kamis_fetch(source.id, acquisition_run_id=command_run_id)
+        except Exception:
+            log_event(
+                _LOGGER,
+                "ERROR",
+                "ingest.source.start_failed",
+                command_run_id=command_run_id,
+                lifecycle_event="SOURCE_START_FAILED",
+            )
+            raise CommandError(_failure_receipt(_CommandFailureCode.SOURCE_START_FAILED)) from None
+
+        _log_fetch(
+            command_run_id,
+            attempt,
+            message_code="ingest.fetch.started",
+            lifecycle_event="FETCH_STARTED",
+        )
+
+        try:
+            secret = load_kamis_api_key()
+        except SecretLoadError:
+            _finalize_fetch_failure(
+                command_run_id,
+                attempt,
+                KamisTransportError("service_key_missing"),
+                command_code=_CommandFailureCode.SECRET_UNAVAILABLE,
+            )
+        except Exception:
+            _finalize_fetch_failure(
+                command_run_id,
+                attempt,
+                KamisTransportError("service_key_missing"),
+                command_code=_CommandFailureCode.SECRET_UNAVAILABLE,
+            )
+
+        try:
+            result = KamisHttpClient().fetch_recent_prices(
+                secret.reveal(),
+                page_size=page_size,
+            )
+        except KamisTransportError as error:
+            _finalize_fetch_failure(
+                command_run_id,
+                attempt,
+                error,
+                command_code=_CommandFailureCode.FETCH_FAILED,
+            )
+        except Exception:
+            _finalize_fetch_failure(
+                command_run_id,
+                attempt,
+                KamisTransportError("transport_internal_error"),
+                command_code=_CommandFailureCode.FETCH_FAILED,
+            )
+        finally:
+            del secret
+
+        try:
+            completed_fetch = complete_kamis_fetch(attempt.id, result)
+        except Exception:
+            _finalize_fetch_failure(
+                command_run_id,
+                attempt,
+                KamisTransportError("transport_internal_error"),
+                command_code=_CommandFailureCode.FETCH_PERSISTENCE_FAILED,
+            )
+
+        _log_fetch(
+            command_run_id,
+            completed_fetch.attempt,
+            message_code="ingest.fetch.succeeded",
+            lifecycle_event="FETCH_COMPLETED",
+        )
+
+        try:
+            started_parse = start_or_get_kamis_parse_run(completed_fetch.artifact.id)
+        except Exception:
+            log_event(
+                _LOGGER,
+                "ERROR",
+                "ingest.parse.start_failed",
+                command_run_id=command_run_id,
+                lifecycle_event="PARSE_START_FAILED",
+            )
+            raise CommandError(
+                _failure_receipt(
+                    _CommandFailureCode.PARSE_START_FAILED,
+                    attempt=completed_fetch.attempt,
+                )
+            ) from None
+
+        if started_parse.created:
+            parse_start_message = "ingest.parse.started"
+            parse_start_event = "PARSE_STARTED"
+        elif started_parse.parse_run.status == ParseRun.Status.STARTED:
+            parse_start_message = "ingest.parse.resumed"
+            parse_start_event = "PARSE_RESUMED"
+        else:
+            parse_start_message = "ingest.parse.replay_started"
+            parse_start_event = "PARSE_REPLAY_STARTED"
+        _log_parse(
+            command_run_id,
+            started_parse.parse_run,
+            message_code=parse_start_message,
+            lifecycle_event=parse_start_event,
+        )
+
+        try:
+            parsed = parse_recent_price_rows(
+                result.rows,
+                identity_registry=INITIAL_RETAIL_IDENTITY_REGISTRY,
+            )
+        except KamisParseError as error:
+            _finalize_parse_failure(
+                command_run_id,
+                started_parse.parse_run,
+                _parse_failure_code(error),
+                command_code=_CommandFailureCode.PARSE_FAILED,
+            )
+        except Exception:
+            _finalize_parse_failure(
+                command_run_id,
+                started_parse.parse_run,
+                ParseGenerationFailureCode.SCHEMA_INVALID,
+                command_code=_CommandFailureCode.PARSE_FAILED,
+            )
+
+        try:
+            completed_parse = complete_kamis_parse_generation(started_parse.parse_run.id, parsed)
+        except ParseGenerationError as error:
+            _finalize_parse_failure(
+                command_run_id,
+                started_parse.parse_run,
+                _generation_failure_code(error),
+                command_code=_CommandFailureCode.GENERATION_FAILED,
+            )
+        except Exception:
+            _finalize_parse_failure(
+                command_run_id,
+                started_parse.parse_run,
+                ParseGenerationFailureCode.PERSISTENCE_FAILED,
+                command_code=_CommandFailureCode.GENERATION_FAILED,
+            )
+
+        _log_parse(
+            command_run_id,
+            completed_parse.parse_run,
+            message_code="ingest.parse.validated",
+            lifecycle_event="PARSE_VALIDATED",
+        )
+        log_event(
+            _LOGGER,
+            "INFO",
+            "ingest.command.succeeded",
+            command_run_id=command_run_id,
+            lifecycle_event="COMMAND_SUCCEEDED",
+        )
+        receipt = " ".join(
+            (
+                f"status={completed_parse.parse_run.status}",
+                f"attempt_id={completed_fetch.attempt.id}",
+                f"artifact_id={completed_fetch.artifact.id}",
+                f"parse_run_id={completed_parse.parse_run.id}",
+                f"pages={completed_fetch.attempt.received_page_count}",
+                f"rows={parsed.input_row_count}",
+                f"accepted={parsed.accepted_row_count}",
+                f"out_of_scope={parsed.out_of_scope_row_count}",
+                f"replayed={'yes' if completed_parse.replayed else 'no'}",
+            )
+        )
+        self.stdout.write(receipt)
diff --git a/grocery/tests/test_ingest_kamis_recent_command.py b/grocery/tests/test_ingest_kamis_recent_command.py
new file mode 100644
index 0000000..164f773
--- /dev/null
+++ b/grocery/tests/test_ingest_kamis_recent_command.py
@@ -0,0 +1,374 @@
+"""Secret-safe orchestration tests for the KAMIS ingestion command."""
+
+from __future__ import annotations
+
+import hashlib
+import io
+import json
+import traceback
+import uuid
+from collections.abc import Iterator
+from copy import deepcopy
+from types import SimpleNamespace
+from typing import cast
+from unittest.mock import ANY, DEFAULT, MagicMock, call, patch
+
+import pytest
+from django.core.management import call_command
+from django.core.management.base import CommandError
+
+from grocery.models import (
+    FetchAttempt,
+    ParseRun,
+    PriceChangeFact,
+    PriceSeriesKey,
+    ReferencePrice,
+    RetailPriceSnapshot,
+    SourceArtifact,
+)
+from grocery.source.client import JsonObject, KamisFetchResult, KamisTransportError, PageReceipt
+from grocery.source.generation import ParseGenerationFailureCode
+from grocery.source.kamis import KamisParseError
+from grocery.source.registry import INITIAL_RETAIL_IDENTITY_REGISTRY
+from grocery.source.secrets import SecretValue
+from grocery.tests.test_kamis_generation import _synthetic_contract_row
+
+_COMMAND_MODULE = "grocery.management.commands.ingest_kamis_recent"
+_SYNTHETIC_CREDENTIAL = "unit-test-credential-marker"
+
+
+def _live_shaped_client_result() -> KamisFetchResult:
+    rows = [
+        _synthetic_contract_row(series_key, ordinal=ordinal)
+        for ordinal, series_key in enumerate(
+            sorted(INITIAL_RETAIL_IDENTITY_REGISTRY.units),
+            start=1,
+        )
+    ]
+    out_of_scope = deepcopy(rows[0])
+    out_of_scope["se_cd"] = "02"
+    out_of_scope["se_nm"] = "합성 중도매"
+    ordered_rows = tuple([*reversed(rows), out_of_scope])
+    body_hash = "a" * 64
+    manifest = hashlib.sha256(
+        json.dumps([body_hash], ensure_ascii=True, separators=(",", ":")).encode("ascii")
+    ).hexdigest()
+    return KamisFetchResult(
+        rows=cast(tuple[JsonObject, ...], ordered_rows),
+        page_receipts=(
+            PageReceipt(
+                ordinal=1,
+                requested_page_number=1,
+                declared_page_number=1,
+                declared_page_size=100,
+                declared_total_count=len(ordered_rows),
+                row_count=len(ordered_rows),
+                http_status=200,
+                provider_result_code="0",
+                byte_length=1_024,
+                body_sha256=body_hash,
+            ),
+        ),
+        ordered_manifest_sha256=manifest,
+        call_count=1,
+    )
+
+
+@pytest.fixture
+def orchestration() -> Iterator[dict[str, MagicMock]]:
+    source_id = uuid.uuid4()
+    attempt_id = uuid.uuid4()
+    artifact_id = uuid.uuid4()
+    parse_run_id = uuid.uuid4()
+    started_attempt = SimpleNamespace(
+        id=attempt_id,
+        state=FetchAttempt.State.STARTED,
+        received_page_count=0,
+        received_row_count=0,
+    )
+    succeeded_attempt = SimpleNamespace(
+        id=attempt_id,
+        state=FetchAttempt.State.SUCCEEDED,
+        received_page_count=2,
+        received_row_count=11,
+    )
+    failed_attempt = SimpleNamespace(
+        id=attempt_id,
+        state=FetchAttempt.State.TERMINAL_FAILED,
+        received_page_count=0,
+        received_row_count=0,
+    )
+    started_parse_run = SimpleNamespace(id=parse_run_id, status=ParseRun.Status.STARTED)
+    validated_parse_run = SimpleNamespace(id=parse_run_id, status=ParseRun.Status.VALIDATED)
+    failed_parse_run = SimpleNamespace(id=parse_run_id, status=ParseRun.Status.FAILED)
+    fetch_result = SimpleNamespace(rows=(object(),))
+    parsed_result = SimpleNamespace(
+        input_row_count=11,
+        accepted_row_count=10,
+        out_of_scope_row_count=1,
+    )
+
+    names = {
+        "bootstrap_kamis_source_configuration": DEFAULT,
+        "start_kamis_fetch": DEFAULT,
+        "load_kamis_api_key": DEFAULT,
+        "KamisHttpClient": DEFAULT,
+        "complete_kamis_fetch": DEFAULT,
+        "fail_kamis_fetch": DEFAULT,
+        "start_or_get_kamis_parse_run": DEFAULT,
+        "parse_recent_price_rows": DEFAULT,
+        "complete_kamis_parse_generation": DEFAULT,
+        "fail_kamis_parse_run": DEFAULT,
+        "log_event": DEFAULT,
+    }
+    with patch.multiple(_COMMAND_MODULE, **names) as doubles:
+        doubles["bootstrap_kamis_source_configuration"].return_value = SimpleNamespace(id=source_id)
+        doubles["start_kamis_fetch"].return_value = started_attempt
+        doubles["load_kamis_api_key"].return_value = SecretValue(_SYNTHETIC_CREDENTIAL)
+        doubles["KamisHttpClient"].return_value.fetch_recent_prices.return_value = fetch_result
+        doubles["complete_kamis_fetch"].return_value = SimpleNamespace(
+            attempt=succeeded_attempt,
+            artifact=SimpleNamespace(id=artifact_id),
+        )
+        doubles["fail_kamis_fetch"].return_value = failed_attempt
+        doubles["start_or_get_kamis_parse_run"].return_value = SimpleNamespace(
+            parse_run=started_parse_run,
+            created=True,
+        )
+        doubles["parse_recent_price_rows"].return_value = parsed_result
+        doubles["complete_kamis_parse_generation"].return_value = SimpleNamespace(
+            parse_run=validated_parse_run,
+            replayed=False,
+        )
+        doubles["fail_kamis_parse_run"].return_value = failed_parse_run
+        doubles.update(
+            {
+                "source_id": source_id,
+                "attempt_id": attempt_id,
+                "artifact_id": artifact_id,
+                "parse_run_id": parse_run_id,
+                "started_attempt": started_attempt,
+                "succeeded_attempt": succeeded_attempt,
+                "failed_attempt": failed_attempt,
+                "started_parse_run": started_parse_run,
+                "validated_parse_run": validated_parse_run,
+                "failed_parse_run": failed_parse_run,
+                "fetch_result": fetch_result,
+                "parsed_result": parsed_result,
+            }
+        )
+        yield doubles
+
+
+def _run_command(*, page_size: int = 47) -> str:
+    output = io.StringIO()
+    call_command("ingest_kamis_recent", page_size=page_size, stdout=output)
+    return output.getvalue().strip()
+
+
+def _safe_traceback(error: BaseException) -> str:
+    return "".join(traceback.format_exception(type(error), error, error.__traceback__))
+
+
+def test_positive_run_uses_only_mocked_secret_and_client_then_emits_safe_receipt(
+    orchestration: dict[str, MagicMock],
+) -> None:
+    receipt = _run_command(page_size=47)
+
+    orchestration["load_kamis_api_key"].assert_called_once_with()
+    orchestration["KamisHttpClient"].assert_called_once_with()
+    orchestration["KamisHttpClient"].return_value.fetch_recent_prices.assert_called_once_with(
+        _SYNTHETIC_CREDENTIAL,
+        page_size=47,
+    )
+    orchestration["complete_kamis_fetch"].assert_called_once_with(
+        orchestration["attempt_id"],
+        orchestration["fetch_result"],
+    )
+    orchestration["parse_recent_price_rows"].assert_called_once_with(
+        orchestration["fetch_result"].rows,
+        identity_registry=INITIAL_RETAIL_IDENTITY_REGISTRY,
+    )
+    orchestration["complete_kamis_parse_generation"].assert_called_once_with(
+        orchestration["parse_run_id"],
+        orchestration["parsed_result"],
+    )
+    assert receipt == " ".join(
+        (
+            "status=VALIDATED",
+            f"attempt_id={orchestration['attempt_id']}",
+            f"artifact_id={orchestration['artifact_id']}",
+            f"parse_run_id={orchestration['parse_run_id']}",
+            "pages=2",
+            "rows=11",
+            "accepted=10",
+            "out_of_scope=1",
+            "replayed=no",
+        )
+    )
+    assert _SYNTHETIC_CREDENTIAL not in receipt
+
+
+def test_command_correlates_lifecycle_events_with_one_command_run_id(
+    orchestration: dict[str, MagicMock],
+) -> None:
+    _run_command()
+
+    start_call = orchestration["start_kamis_fetch"].call_args
+    command_run_id = start_call.kwargs["acquisition_run_id"]
+    assert isinstance(command_run_id, uuid.UUID)
+    assert orchestration["log_event"].call_count == 5
+    assert all(
+        audit_call.kwargs["command_run_id"] == command_run_id
+        for audit_call in orchestration["log_event"].call_args_list
+    )
+    assert orchestration["log_event"].call_args_list[0] == call(
+        ANY,
+        "INFO",
+        "ingest.fetch.started",
+        command_run_id=command_run_id,
+        lifecycle_id=orchestration["attempt_id"],
+        lifecycle_status="STARTED",
+        lifecycle_event="FETCH_STARTED",
+    )
+
+
+def test_transport_failure_is_finalized_and_never_starts_parse(
+    orchestration: dict[str, MagicMock],
+) -> None:
+    transport_error = KamisTransportError("retry_exhausted", http_status=429)
+    orchestration["KamisHttpClient"].return_value.fetch_recent_prices.side_effect = transport_error
+    orchestration["failed_attempt"].state = FetchAttempt.State.RETRYABLE_FAILED
+
+    with pytest.raises(CommandError) as caught:
+        _run_command()
+
+    orchestration["fail_kamis_fetch"].assert_called_once_with(
+        orchestration["attempt_id"], transport_error
+    )
+    orchestration["complete_kamis_fetch"].assert_not_called()
+    orchestration["start_or_get_kamis_parse_run"].assert_not_called()
+    assert str(caught.value).startswith("code=INGEST_FETCH_FAILED ")
+    assert "status=RETRYABLE_FAILED" in str(caught.value)
+    assert "429" not in str(caught.value)
+    assert _SYNTHETIC_CREDENTIAL not in _safe_traceback(caught.value)
+
+
+def test_parser_identity_failure_closes_started_parse_run(
+    orchestration: dict[str, MagicMock],
+) -> None:
+    orchestration["parse_recent_price_rows"].side_effect = KamisParseError(
+        "unit_identity_drift",
+        row_index=7,
+        field="unit",
+    )
+
+    with pytest.raises(CommandError) as caught:
+        _run_command()
+
+    orchestration["fail_kamis_parse_run"].assert_called_once_with(
+        orchestration["parse_run_id"],
+        ParseGenerationFailureCode.IDENTITY_DRIFT,
+    )
+    orchestration["complete_kamis_parse_generation"].assert_not_called()
+    assert str(caught.value) == " ".join(
+        (
+            "code=INGEST_PARSE_FAILED",
+            f"parse_run_id={orchestration['parse_run_id']}",
+            "parse_status=FAILED",
+        )
+    )
+    assert "unit" not in str(caught.value)
+
+
+def test_arbitrary_transport_exception_and_credential_are_not_reflected(
+    orchestration: dict[str, MagicMock],
+) -> None:
+    arbitrary_marker = f"arbitrary-{_SYNTHETIC_CREDENTIAL}"
+    orchestration["KamisHttpClient"].return_value.fetch_recent_prices.side_effect = RuntimeError(
+        arbitrary_marker
+    )
+
+    with pytest.raises(CommandError) as caught:
+        _run_command()
+
+    finalized_error = orchestration["fail_kamis_fetch"].call_args.args[1]
+    assert isinstance(finalized_error, KamisTransportError)
+    assert finalized_error.code == "transport_internal_error"
+    visible = "\n".join(
+        (
+            str(caught.value),
+            _safe_traceback(caught.value),
+            str(orchestration["log_event"].mock_calls),
+        )
+    )
+    assert arbitrary_marker not in visible
+    assert _SYNTHETIC_CREDENTIAL not in visible
+
+
+def test_replay_receipt_is_explicit_without_creating_a_second_contract(
+    orchestration: dict[str, MagicMock],
+) -> None:
+    orchestration["start_or_get_kamis_parse_run"].return_value.created = False
+    orchestration["start_or_get_kamis_parse_run"].return_value.parse_run = orchestration[
+        "validated_parse_run"
+    ]
+    orchestration["complete_kamis_parse_generation"].return_value.replayed = True
+
+    receipt = _run_command()
+
+    assert receipt.endswith("replayed=yes")
+    orchestration["start_or_get_kamis_parse_run"].assert_called_once_with(
+        orchestration["artifact_id"]
+    )
+    orchestration["complete_kamis_parse_generation"].assert_called_once()
+
+
+def test_invalid_programmatic_page_size_fails_before_secret_or_database_access(
+    orchestration: dict[str, MagicMock],
+) -> None:
+    with pytest.raises(CommandError, match=r"^code=INGEST_PAGE_SIZE_INVALID$"):
+        _run_command(page_size=0)
+
+    orchestration["bootstrap_kamis_source_configuration"].assert_not_called()
+    orchestration["load_kamis_api_key"].assert_not_called()
+    orchestration["KamisHttpClient"].assert_not_called()
+
+
+@pytest.mark.django_db
+def test_real_persistence_generation_replays_with_mocked_secret_and_network() -> None:
+    result = _live_shaped_client_result()
+    output = io.StringIO()
+    replay_output = io.StringIO()
+
+    with (
+        patch(
+            f"{_COMMAND_MODULE}.load_kamis_api_key",
+            return_value=SecretValue(_SYNTHETIC_CREDENTIAL),
+        ) as secret_loader,
+        patch(f"{_COMMAND_MODULE}.KamisHttpClient") as client_class,
+        patch(f"{_COMMAND_MODULE}.log_event") as audit_log,
+        patch("grocery.source.secrets._read_secret_file") as secret_file_reader,
+    ):
+        client_class.return_value.fetch_recent_prices.return_value = result
+        call_command("ingest_kamis_recent", stdout=output)
+        call_command("ingest_kamis_recent", stdout=replay_output)
+
+    secret_loader.assert_has_calls([call(), call()])
+    secret_file_reader.assert_not_called()
+    assert client_class.return_value.fetch_recent_prices.call_args_list == [
+        call(_SYNTHETIC_CREDENTIAL, page_size=100),
+        call(_SYNTHETIC_CREDENTIAL, page_size=100),
+    ]
+    assert "replayed=no" in output.getvalue()
+    assert "replayed=yes" in replay_output.getvalue()
+    assert FetchAttempt.objects.count() == 2
+    assert SourceArtifact.objects.count() == 1
+    assert ParseRun.objects.count() == 1
+    assert PriceSeriesKey.objects.count() == 10
+    assert RetailPriceSnapshot.objects.count() == 10
+    assert ReferencePrice.objects.count() == 30
+    assert PriceChangeFact.objects.count() == 30
+    assert _SYNTHETIC_CREDENTIAL not in output.getvalue()
+    assert _SYNTHETIC_CREDENTIAL not in replay_output.getvalue()
+    assert _SYNTHETIC_CREDENTIAL not in str(audit_log.mock_calls)
