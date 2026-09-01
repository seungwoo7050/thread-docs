## `feat(source): persist reconciled fetch receipts`

diff --git a/grocery/source/persistence.py b/grocery/source/persistence.py
new file mode 100644
index 0000000..c650922
--- /dev/null
+++ b/grocery/source/persistence.py
@@ -0,0 +1,188 @@
+"""Persist one successful KAMIS acquisition without retaining its raw payload."""
+
+from __future__ import annotations
+
+import uuid
+from dataclasses import dataclass
+
+from django.core.exceptions import ValidationError
+from django.db import transaction
+from django.utils import timezone
+
+from grocery.models import (
+    FetchAttempt,
+    PageReceipt,
+    SourceArtifact,
+    SourceConfiguration,
+    build_source_artifact,
+    ordered_page_manifest_sha256,
+)
+from grocery.source.client import REDACTED_REQUEST_SHAPE, KamisFetchResult
+
+
+@dataclass(frozen=True, slots=True)
+class CompletedKamisFetch:
+    attempt: FetchAttempt
+    artifact: SourceArtifact
+    artifact_created: bool
+
+
+@transaction.atomic
+def start_kamis_fetch(
+    source_configuration_id: uuid.UUID,
+    *,
+    acquisition_run_id: uuid.UUID | None = None,
+    attempt_ordinal: int = 1,
+) -> FetchAttempt:
+    """Start one logical acquisition against an approved active source revision."""
+
+    source = SourceConfiguration.objects.select_for_update().get(pk=source_configuration_id)
+    if source.state != SourceConfiguration.State.ACTIVE:
+        raise ValidationError("KAMIS fetches require an active source configuration.")
+    if source.publication_mode != SourceConfiguration.PublicationMode.RECENT_COMPARISON:
+        raise ValidationError("KAMIS recent fetches require the recent-comparison mode.")
+    return FetchAttempt.objects.create(
+        source_configuration=source,
+        acquisition_run_id=acquisition_run_id or uuid.uuid4(),
+        attempt_ordinal=attempt_ordinal,
+        redacted_request_shape=REDACTED_REQUEST_SHAPE,
+    )
+
+
+@transaction.atomic
+def complete_kamis_fetch(
+    attempt_id: uuid.UUID,
+    result: KamisFetchResult,
+) -> CompletedKamisFetch:
+    """Reconcile redacted receipts and atomically attach a content-addressed artifact."""
+
+    attempt = (
+        FetchAttempt.objects.select_for_update()
+        .select_related("source_configuration")
+        .get(pk=attempt_id)
+    )
+    if attempt.state == FetchAttempt.State.SUCCEEDED:
+        return _validate_completed_replay(attempt, result)
+    if attempt.state != FetchAttempt.State.STARTED:
+        raise ValidationError("Only a started fetch attempt can be completed.")
+
+    source = attempt.source_configuration
+    _validate_result_budget(source, result)
+    receipts = _receipt_candidates(attempt, result)
+    if ordered_page_manifest_sha256(receipts) != result.ordered_manifest_sha256:
+        raise ValidationError("The client and persistence ordered manifests differ.")
+
+    PageReceipt.objects.bulk_create(receipts)
+    attempt.state = FetchAttempt.State.SUCCEEDED
+    attempt.completed_at = timezone.now()
+    attempt.received_page_count = len(receipts)
+    attempt.received_row_count = len(result.rows)
+    attempt.received_byte_count = sum(receipt.body_byte_length for receipt in receipts)
+    attempt.save()
+
+    artifact, created = build_source_artifact(attempt.id)
+    if artifact.ordered_manifest_sha256 != result.ordered_manifest_sha256:
+        raise ValidationError("The persisted artifact manifest differs from the client result.")
+    attempt.refresh_from_db()
+    return CompletedKamisFetch(attempt=attempt, artifact=artifact, artifact_created=created)
+
+
+def _validate_result_budget(
+    source: SourceConfiguration,
+    result: KamisFetchResult,
+) -> None:
+    if not result.page_receipts:
+        raise ValidationError("A successful fetch requires at least one page receipt.")
+    if result.call_count < len(result.page_receipts):
+        raise ValidationError("The call count cannot be smaller than the page count.")
+    if result.call_count > source.max_requests_per_attempt:
+        raise ValidationError("The fetch exceeded its configured request budget.")
+    if len(result.page_receipts) > source.max_pages_per_attempt:
+        raise ValidationError("The fetch exceeded its configured page budget.")
+    if any(receipt.byte_length > source.max_page_bytes for receipt in result.page_receipts):
+        raise ValidationError("A fetch page exceeded its configured byte budget.")
+
+
+def _receipt_candidates(
+    attempt: FetchAttempt,
+    result: KamisFetchResult,
+) -> list[PageReceipt]:
+    expected_ordinals = list(range(1, len(result.page_receipts) + 1))
+    if [receipt.ordinal for receipt in result.page_receipts] != expected_ordinals:
+        raise ValidationError("Client page receipt ordinals must be contiguous from one.")
+    if [receipt.requested_page_number for receipt in result.page_receipts] != expected_ordinals:
+        raise ValidationError("Client requested page numbers must be contiguous from one.")
+
+    declared_totals = {receipt.declared_total_count for receipt in result.page_receipts}
+    if len(declared_totals) != 1 or sum(
+        receipt.row_count for receipt in result.page_receipts
+    ) != len(result.rows):
+        raise ValidationError("Client page rows do not reconcile with the result row count.")
+    if declared_totals != {len(result.rows)}:
+        raise ValidationError("Client declared total does not match the result row count.")
+
+    candidates: list[PageReceipt] = []
+    for receipt in result.page_receipts:
+        if receipt.requested_page_number != receipt.declared_page_number:
+            raise ValidationError("Client requested and declared page numbers differ.")
+        if receipt.http_status != 200 or receipt.provider_result_code != "0":
+            raise ValidationError("A successful result contains a failed page receipt.")
+        candidates.append(
+            PageReceipt(
+                fetch_attempt=attempt,
+                request_ordinal=receipt.ordinal,
+                page_number=receipt.requested_page_number,
+                http_status=receipt.http_status,
+                provider_result_code=receipt.provider_result_code,
+                declared_total_count=receipt.declared_total_count,
+                received_row_count=receipt.row_count,
+                body_state=PageReceipt.BodyState.RECEIVED,
+                body_byte_length=receipt.byte_length,
+                body_sha256=receipt.body_sha256,
+                media_type=PageReceipt.MediaType.JSON,
+                encoding=PageReceipt.Encoding.UTF_8,
+            )
+        )
+    return candidates
+
+
+def _validate_completed_replay(
+    attempt: FetchAttempt,
+    result: KamisFetchResult,
+) -> CompletedKamisFetch:
+    source = attempt.source_configuration
+    _validate_result_budget(source, result)
+    persisted_receipts = list(attempt.page_receipts.order_by("request_ordinal"))
+    candidate_receipts = _receipt_candidates(attempt, result)
+    persisted_shape = [
+        (
+            receipt.request_ordinal,
+            receipt.page_number,
+            receipt.declared_total_count,
+            receipt.received_row_count,
+            receipt.body_byte_length,
+            receipt.body_sha256,
+        )
+        for receipt in persisted_receipts
+    ]
+    candidate_shape = [
+        (
+            receipt.request_ordinal,
+            receipt.page_number,
+            receipt.declared_total_count,
+            receipt.received_row_count,
+            receipt.body_byte_length,
+            receipt.body_sha256,
+        )
+        for receipt in candidate_receipts
+    ]
+    if persisted_shape != candidate_shape:
+        raise ValidationError("A completed fetch replay conflicts with its receipts.")
+    artifact = attempt.artifact
+    if artifact is None or artifact.ordered_manifest_sha256 != result.ordered_manifest_sha256:
+        raise ValidationError("A completed fetch replay conflicts with its artifact.")
+    return CompletedKamisFetch(
+        attempt=attempt,
+        artifact=artifact,
+        artifact_created=False,
+    )
diff --git a/grocery/tests/test_source_persistence.py b/grocery/tests/test_source_persistence.py
new file mode 100644
index 0000000..8deed68
--- /dev/null
+++ b/grocery/tests/test_source_persistence.py
@@ -0,0 +1,158 @@
+import hashlib
+import json
+import uuid
+
+import pytest
+from django.core.exceptions import ValidationError
+
+from grocery.models import FetchAttempt, PageReceipt, SourceArtifact, SourceConfiguration
+from grocery.source.client import KamisFetchResult
+from grocery.source.client import PageReceipt as ClientPageReceipt
+from grocery.source.persistence import complete_kamis_fetch, start_kamis_fetch
+from grocery.tests.test_acquisition_models import create_source_configuration
+
+pytestmark = pytest.mark.django_db
+
+
+def _manifest(*hashes: str) -> str:
+    encoded = json.dumps(list(hashes), ensure_ascii=True, separators=(",", ":")).encode("ascii")
+    return hashlib.sha256(encoded).hexdigest()
+
+
+def make_result(*, second_hash: str = "b" * 64, call_count: int = 2) -> KamisFetchResult:
+    first_hash = "a" * 64
+    receipts = (
+        ClientPageReceipt(
+            ordinal=1,
+            requested_page_number=1,
+            declared_page_number=1,
+            declared_page_size=2,
+            declared_total_count=3,
+            row_count=2,
+            http_status=200,
+            provider_result_code="0",
+            byte_length=20,
+            body_sha256=first_hash,
+        ),
+        ClientPageReceipt(
+            ordinal=2,
+            requested_page_number=2,
+            declared_page_number=2,
+            declared_page_size=2,
+            declared_total_count=3,
+            row_count=1,
+            http_status=200,
+            provider_result_code="0",
+            byte_length=10,
+            body_sha256=second_hash,
+        ),
+    )
+    return KamisFetchResult(
+        rows=({"row": 1}, {"row": 2}, {"row": 3}),
+        page_receipts=receipts,
+        ordered_manifest_sha256=_manifest(first_hash, second_hash),
+        call_count=call_count,
+    )
+
+
+def test_start_records_only_a_redacted_shape_for_an_active_recent_source() -> None:
+    source = create_source_configuration()
+    run_id = uuid.uuid4()
+
+    attempt = start_kamis_fetch(source.id, acquisition_run_id=run_id)
+
+    assert attempt.state == FetchAttempt.State.STARTED
+    assert attempt.acquisition_run_id == run_id
+    assert "<redacted>" in attempt.redacted_request_shape
+    assert "://" not in attempt.redacted_request_shape
+    assert "?" not in attempt.redacted_request_shape
+
+
+def test_success_reconciles_receipts_and_builds_a_hash_only_artifact() -> None:
+    source = create_source_configuration()
+    attempt = start_kamis_fetch(source.id)
+    result = make_result()
+
+    completed = complete_kamis_fetch(attempt.id, result)
+
+    assert completed.artifact_created is True
+    assert completed.attempt.state == FetchAttempt.State.SUCCEEDED
+    assert completed.attempt.received_page_count == 2
+    assert completed.attempt.received_row_count == 3
+    assert completed.attempt.received_byte_count == 30
+    assert completed.artifact.ordered_manifest_sha256 == result.ordered_manifest_sha256
+    assert completed.artifact.retention_mode == SourceArtifact.RetentionMode.HASH_ONLY
+    assert list(
+        PageReceipt.objects.filter(fetch_attempt=attempt).values_list(
+            "request_ordinal", "page_number", "received_row_count", "body_byte_length"
+        )
+    ) == [(1, 1, 2, 20), (2, 2, 1, 10)]
+
+
+def test_same_attempt_replay_is_idempotent_and_new_attempt_deduplicates_artifact() -> None:
+    source = create_source_configuration()
+    result = make_result()
+    first_attempt = start_kamis_fetch(source.id)
+    first = complete_kamis_fetch(first_attempt.id, result)
+    replay = complete_kamis_fetch(first_attempt.id, result)
+    second_attempt = start_kamis_fetch(source.id)
+    second = complete_kamis_fetch(second_attempt.id, result)
+
+    assert replay.attempt.id == first.attempt.id
+    assert replay.artifact_created is False
+    assert second.artifact.id == first.artifact.id
+    assert second.artifact_created is False
+    assert FetchAttempt.objects.count() == 2
+    assert SourceArtifact.objects.count() == 1
+    assert PageReceipt.objects.count() == 4
+
+
+def test_manifest_or_budget_mismatch_rolls_back_without_partial_receipts() -> None:
+    source = create_source_configuration(max_requests_per_attempt=1)
+    attempt = start_kamis_fetch(source.id)
+
+    with pytest.raises(ValidationError, match="request budget"):
+        complete_kamis_fetch(attempt.id, make_result(call_count=2))
+
+    attempt.refresh_from_db()
+    assert attempt.state == FetchAttempt.State.STARTED
+    assert not attempt.page_receipts.exists()
+    assert not SourceArtifact.objects.exists()
+
+    source_two = create_source_configuration()
+    attempt_two = start_kamis_fetch(source_two.id)
+    invalid = make_result()
+    invalid = KamisFetchResult(
+        rows=invalid.rows,
+        page_receipts=invalid.page_receipts,
+        ordered_manifest_sha256="f" * 64,
+        call_count=invalid.call_count,
+    )
+    with pytest.raises(ValidationError, match="ordered manifests"):
+        complete_kamis_fetch(attempt_two.id, invalid)
+    assert not attempt_two.page_receipts.exists()
+
+
+@pytest.mark.parametrize(
+    "overrides",
+    [
+        {"state": SourceConfiguration.State.PAUSED},
+        {"publication_mode": SourceConfiguration.PublicationMode.CURRENT_ONLY},
+    ],
+)
+def test_start_rejects_non_active_or_wrong_mode_source(overrides: dict[str, str]) -> None:
+    source = create_source_configuration(**overrides)
+
+    with pytest.raises(ValidationError):
+        start_kamis_fetch(source.id)
+
+
+def test_completed_replay_conflict_fails_closed_without_echoing_rows() -> None:
+    source = create_source_configuration()
+    attempt = start_kamis_fetch(source.id)
+    complete_kamis_fetch(attempt.id, make_result())
+
+    with pytest.raises(ValidationError, match="conflicts") as caught:
+        complete_kamis_fetch(attempt.id, make_result(second_hash="c" * 64))
+
+    assert "row" not in str(caught.value)


## `feat(source): record redacted fetch failures`

diff --git a/grocery/source/persistence.py b/grocery/source/persistence.py
index c650922..8ebb4b1 100644
--- a/grocery/source/persistence.py
+++ b/grocery/source/persistence.py
@@ -17,7 +17,7 @@ from grocery.models import (
     build_source_artifact,
     ordered_page_manifest_sha256,
 )
-from grocery.source.client import REDACTED_REQUEST_SHAPE, KamisFetchResult
+from grocery.source.client import REDACTED_REQUEST_SHAPE, KamisFetchResult, KamisTransportError
 
 
 @dataclass(frozen=True, slots=True)
@@ -27,6 +27,69 @@ class CompletedKamisFetch:
     artifact_created: bool
 
 
+_RESPONSE_LIMIT_CODES = frozenset(
+    {
+        "call_budget_exceeded",
+        "page_budget_exceeded",
+        "page_too_large",
+    }
+)
+_RECONCILIATION_CODES = frozenset(
+    {
+        "declared_page_mismatch",
+        "declared_page_size_mismatch",
+        "declared_total_changed",
+        "page_row_count_mismatch",
+        "row_total_exceeded",
+    }
+)
+_SCHEMA_CODES = frozenset(
+    {
+        "invalid_json",
+        "invalid_envelope",
+        "invalid_header",
+        "invalid_provider_header",
+        "invalid_body",
+        "unexpected_data_type",
+        "invalid_declared_page",
+        "invalid_declared_page_size",
+        "invalid_declared_total",
+        "invalid_items_envelope",
+        "items_not_array",
+        "item_not_object",
+        "unexpected_envelope_keys",
+        "unexpected_content_type",
+        "missing_content_type",
+        "unexpected_charset",
+        "missing_charset",
+        "invalid_content_length",
+        "response_body_not_bytes",
+    }
+)
+_KNOWN_TRANSPORT_CODES = (
+    _RESPONSE_LIMIT_CODES
+    | _RECONCILIATION_CODES
+    | _SCHEMA_CODES
+    | frozenset(
+        {
+            "invalid_http_status",
+            "invalid_page_size",
+            "invalid_response_state",
+            "invalid_retry_state",
+            "redirect_not_allowed",
+            "request_parameter_allowlist_violation",
+            "retry_exhausted",
+            "service_key_missing",
+            "terminal_http_status",
+            "terminal_provider_error",
+            "tls_error",
+            "tls_verification_failed",
+            "transport_internal_error",
+        }
+    )
+)
+
+
 @transaction.atomic
 def start_kamis_fetch(
     source_configuration_id: uuid.UUID,
@@ -87,6 +150,61 @@ def complete_kamis_fetch(
     return CompletedKamisFetch(attempt=attempt, artifact=artifact, artifact_created=created)
 
 
+@transaction.atomic
+def fail_kamis_fetch(
+    attempt_id: uuid.UUID,
+    error: KamisTransportError,
+) -> FetchAttempt:
+    """Finalize a failed attempt using only the transport's redacted error fields."""
+
+    attempt = FetchAttempt.objects.select_for_update().get(pk=attempt_id)
+    if attempt.state != FetchAttempt.State.STARTED:
+        raise ValidationError("Only a started fetch attempt can be failed.")
+
+    state, failure_class = _classify_transport_failure(error)
+    attempt.state = state
+    attempt.completed_at = timezone.now()
+    attempt.failure_class = failure_class
+    attempt.failure_code = (
+        error.code.upper()
+        if error.code in _KNOWN_TRANSPORT_CODES
+        else "UNCLASSIFIED_TRANSPORT_ERROR"
+    )
+    attempt.save()
+    return attempt
+
+
+def _classify_transport_failure(
+    error: KamisTransportError,
+) -> tuple[FetchAttempt.State, FetchAttempt.FailureClass]:
+    if error.code == "retry_exhausted":
+        if error.http_status == 429:
+            failure_class = FetchAttempt.FailureClass.HTTP_429
+        elif error.http_status is not None and 500 <= error.http_status < 600:
+            failure_class = FetchAttempt.FailureClass.HTTP_5XX
+        elif error.provider_result_code is not None:
+            failure_class = FetchAttempt.FailureClass.PROVIDER_TRANSIENT
+        else:
+            failure_class = FetchAttempt.FailureClass.NETWORK
+        return FetchAttempt.State.RETRYABLE_FAILED, failure_class
+
+    if error.code in _RESPONSE_LIMIT_CODES:
+        failure_class = FetchAttempt.FailureClass.RESPONSE_LIMIT
+    elif error.code in _RECONCILIATION_CODES:
+        failure_class = FetchAttempt.FailureClass.RECONCILIATION
+    elif error.code in _SCHEMA_CODES:
+        failure_class = FetchAttempt.FailureClass.SCHEMA
+    elif error.code in {"tls_error", "tls_verification_failed"}:
+        failure_class = FetchAttempt.FailureClass.NETWORK
+    elif error.code == "terminal_http_status" and error.http_status in {401, 403}:
+        failure_class = FetchAttempt.FailureClass.AUTHENTICATION
+    elif error.code == "terminal_provider_error" and error.provider_result_code == "-3":
+        failure_class = FetchAttempt.FailureClass.AUTHENTICATION
+    else:
+        failure_class = FetchAttempt.FailureClass.INVALID_REQUEST
+    return FetchAttempt.State.TERMINAL_FAILED, failure_class
+
+
 def _validate_result_budget(
     source: SourceConfiguration,
     result: KamisFetchResult,
diff --git a/grocery/tests/test_source_persistence.py b/grocery/tests/test_source_persistence.py
index 8deed68..0653ac9 100644
--- a/grocery/tests/test_source_persistence.py
+++ b/grocery/tests/test_source_persistence.py
@@ -6,9 +6,13 @@ import pytest
 from django.core.exceptions import ValidationError
 
 from grocery.models import FetchAttempt, PageReceipt, SourceArtifact, SourceConfiguration
-from grocery.source.client import KamisFetchResult
+from grocery.source.client import KamisFetchResult, KamisTransportError
 from grocery.source.client import PageReceipt as ClientPageReceipt
-from grocery.source.persistence import complete_kamis_fetch, start_kamis_fetch
+from grocery.source.persistence import (
+    complete_kamis_fetch,
+    fail_kamis_fetch,
+    start_kamis_fetch,
+)
 from grocery.tests.test_acquisition_models import create_source_configuration
 
 pytestmark = pytest.mark.django_db
@@ -156,3 +160,81 @@ def test_completed_replay_conflict_fails_closed_without_echoing_rows() -> None:
         complete_kamis_fetch(attempt.id, make_result(second_hash="c" * 64))
 
     assert "row" not in str(caught.value)
+
+
+@pytest.mark.parametrize(
+    ("error", "state", "failure_class"),
+    [
+        (
+            KamisTransportError("retry_exhausted", http_status=429),
+            FetchAttempt.State.RETRYABLE_FAILED,
+            FetchAttempt.FailureClass.HTTP_429,
+        ),
+        (
+            KamisTransportError("retry_exhausted", http_status=503),
+            FetchAttempt.State.RETRYABLE_FAILED,
+            FetchAttempt.FailureClass.HTTP_5XX,
+        ),
+        (
+            KamisTransportError("retry_exhausted", provider_result_code="-5"),
+            FetchAttempt.State.RETRYABLE_FAILED,
+            FetchAttempt.FailureClass.PROVIDER_TRANSIENT,
+        ),
+        (
+            KamisTransportError("terminal_http_status", http_status=401),
+            FetchAttempt.State.TERMINAL_FAILED,
+            FetchAttempt.FailureClass.AUTHENTICATION,
+        ),
+        (
+            KamisTransportError("page_too_large"),
+            FetchAttempt.State.TERMINAL_FAILED,
+            FetchAttempt.FailureClass.RESPONSE_LIMIT,
+        ),
+        (
+            KamisTransportError("unexpected_envelope_keys"),
+            FetchAttempt.State.TERMINAL_FAILED,
+            FetchAttempt.FailureClass.SCHEMA,
+        ),
+        (
+            KamisTransportError("declared_total_changed"),
+            FetchAttempt.State.TERMINAL_FAILED,
+            FetchAttempt.FailureClass.RECONCILIATION,
+        ),
+    ],
+)
+def test_failure_finalization_records_only_safe_codes(
+    error: KamisTransportError,
+    state: str,
+    failure_class: str,
+) -> None:
+    source = create_source_configuration()
+    attempt = start_kamis_fetch(source.id)
+
+    failed = fail_kamis_fetch(attempt.id, error)
+
+    assert failed.state == state
+    assert failed.failure_class == failure_class
+    assert failed.failure_code == error.code.upper()
+    assert failed.artifact_id is None
+    assert not failed.page_receipts.exists()
+    assert "serviceKey" not in failed.failure_code
+
+
+def test_failure_finalization_cannot_overwrite_a_terminal_attempt() -> None:
+    source = create_source_configuration()
+    attempt = start_kamis_fetch(source.id)
+    fail_kamis_fetch(attempt.id, KamisTransportError("tls_verification_failed"))
+
+    with pytest.raises(ValidationError, match="started"):
+        fail_kamis_fetch(attempt.id, KamisTransportError("retry_exhausted"))
+
+
+def test_unknown_failure_code_is_not_copied_to_a_receipt_or_error_field() -> None:
+    source = create_source_configuration()
+    attempt = start_kamis_fetch(source.id)
+    marker = "KAMIS_API_KEY_synthetic_marker"
+
+    failed = fail_kamis_fetch(attempt.id, KamisTransportError(marker))
+
+    assert failed.failure_code == "UNCLASSIFIED_TRANSPORT_ERROR"
+    assert marker not in failed.failure_code


