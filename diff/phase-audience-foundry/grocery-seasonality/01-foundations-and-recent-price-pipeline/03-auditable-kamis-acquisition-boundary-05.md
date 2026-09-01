## `feat(source): retain partial failure receipts`

diff --git a/grocery/source/client.py b/grocery/source/client.py
index 8e470a9..3061400 100644
--- a/grocery/source/client.py
+++ b/grocery/source/client.py
@@ -48,6 +48,51 @@ _ITEMS_KEYS = frozenset({"item"})
 _SAFE_PROVIDER_CODE = re.compile(r"-?[0-9]{1,3}\Z")
 _RETRYABLE_PROVIDER_CODES = frozenset({"-1", "-5", "-10", "22", "23"})
 _RETRY_DELAYS_SECONDS = (0.25, 1.0)
+_SAFE_TRANSPORT_ERROR_CODES = frozenset(
+    {
+        "call_budget_exceeded",
+        "declared_page_mismatch",
+        "declared_page_size_mismatch",
+        "declared_total_changed",
+        "invalid_body",
+        "invalid_content_length",
+        "invalid_declared_page",
+        "invalid_declared_page_size",
+        "invalid_declared_total",
+        "invalid_envelope",
+        "invalid_header",
+        "invalid_http_status",
+        "invalid_items_envelope",
+        "invalid_json",
+        "invalid_page_size",
+        "invalid_provider_header",
+        "invalid_response_state",
+        "invalid_retry_state",
+        "item_not_object",
+        "items_not_array",
+        "missing_charset",
+        "missing_content_type",
+        "page_budget_exceeded",
+        "page_row_count_mismatch",
+        "page_too_large",
+        "redirect_not_allowed",
+        "request_parameter_allowlist_violation",
+        "response_body_not_bytes",
+        "retry_exhausted",
+        "row_total_exceeded",
+        "service_key_missing",
+        "terminal_http_status",
+        "terminal_provider_error",
+        "tls_error",
+        "tls_verification_failed",
+        "transport_internal_error",
+        "unexpected_charset",
+        "unexpected_content_type",
+        "unexpected_data_type",
+        "unexpected_envelope_keys",
+    }
+)
+_UNCLASSIFIED_TRANSPORT_ERROR = "unclassified_transport_error"
 
 type JsonValue = None | bool | int | float | str | list[JsonValue] | dict[str, JsonValue]
 type JsonObject = dict[str, JsonValue]
@@ -94,25 +139,68 @@ class KamisTransportError(RuntimeError):
         attempt: int | None = None,
         http_status: int | None = None,
         provider_result_code: str | None = None,
+        partial_page_receipts: tuple[PageReceipt, ...] = (),
     ) -> None:
-        self.code = code
-        self.page_number = page_number
-        self.attempt = attempt
-        self.http_status = http_status
-        self.provider_result_code = provider_result_code
+        if not isinstance(partial_page_receipts, tuple) or any(
+            not isinstance(receipt, PageReceipt) for receipt in partial_page_receipts
+        ):
+            raise TypeError("partial_page_receipts must be a tuple of PageReceipt values")
+        safe_code = (
+            code
+            if isinstance(code, str) and code in _SAFE_TRANSPORT_ERROR_CODES
+            else _UNCLASSIFIED_TRANSPORT_ERROR
+        )
+        safe_page_number = (
+            page_number
+            if (
+                isinstance(page_number, int)
+                and not isinstance(page_number, bool)
+                and page_number > 0
+            )
+            else None
+        )
+        safe_attempt = (
+            attempt
+            if isinstance(attempt, int) and not isinstance(attempt, bool) and attempt > 0
+            else None
+        )
+        safe_http_status = (
+            http_status
+            if isinstance(http_status, int)
+            and not isinstance(http_status, bool)
+            and 100 <= http_status <= 599
+            else None
+        )
+        safe_provider_result_code = (
+            provider_result_code
+            if isinstance(provider_result_code, str)
+            and _SAFE_PROVIDER_CODE.fullmatch(provider_result_code) is not None
+            else None
+        )
+        self.code = safe_code
+        self.page_number = safe_page_number
+        self.attempt = safe_attempt
+        self.http_status = safe_http_status
+        self.provider_result_code = safe_provider_result_code
+        self.partial_page_receipts = partial_page_receipts
         self.request_shape = REDACTED_REQUEST_SHAPE
-        details = [f"code={code}"]
-        if page_number is not None:
-            details.append(f"page={page_number}")
-        if attempt is not None:
-            details.append(f"attempt={attempt}")
-        if http_status is not None:
-            details.append(f"http_status={http_status}")
-        if provider_result_code is not None:
-            details.append(f"provider_result_code={provider_result_code}")
+        details = [f"code={safe_code}"]
+        if safe_page_number is not None:
+            details.append(f"page={safe_page_number}")
+        if safe_attempt is not None:
+            details.append(f"attempt={safe_attempt}")
+        if safe_http_status is not None:
+            details.append(f"http_status={safe_http_status}")
+        if safe_provider_result_code is not None:
+            details.append(f"provider_result_code={safe_provider_result_code}")
         details.append(f"request={REDACTED_REQUEST_SHAPE}")
         super().__init__(" ".join(details))
 
+    def _retain_completed_pages(self, receipts: tuple[PageReceipt, ...]) -> None:
+        """Attach only raw-free evidence collected by this client invocation."""
+
+        self.partial_page_receipts = receipts
+
 
 @dataclass(frozen=True, slots=True)
 class PageReceipt:
@@ -205,50 +293,54 @@ class KamisHttpClient:
         expected_total: int | None = None
         page_number = 1
 
-        while True:
-            page = self._fetch_page(
-                normalized_key,
-                page_number=page_number,
-                page_size=page_size,
-                budget=budget,
-            )
+        try:
+            while True:
+                page = self._fetch_page(
+                    normalized_key,
+                    page_number=page_number,
+                    page_size=page_size,
+                    budget=budget,
+                )
 
-            if expected_total is None:
-                expected_total = page.declared_total_count
-                required_pages = max(1, (expected_total + page_size - 1) // page_size)
-                if required_pages > MAX_PAGES:
-                    raise KamisTransportError("page_budget_exceeded", page_number=page_number)
-            elif page.declared_total_count != expected_total:
-                raise KamisTransportError("declared_total_changed", page_number=page_number)
-
-            remaining = expected_total - len(rows)
-            expected_row_count = min(page_size, max(0, remaining))
-            if len(page.items) != expected_row_count:
-                raise KamisTransportError("page_row_count_mismatch", page_number=page_number)
-
-            rows.extend(page.items)
-            receipts.append(
-                PageReceipt(
-                    ordinal=len(receipts) + 1,
-                    requested_page_number=page_number,
-                    declared_page_number=page.declared_page_number,
-                    declared_page_size=page.declared_page_size,
-                    declared_total_count=page.declared_total_count,
-                    row_count=len(page.items),
-                    http_status=200,
-                    provider_result_code=page.provider_result_code,
-                    byte_length=page.byte_length,
-                    body_sha256=page.body_sha256,
+                if expected_total is None:
+                    expected_total = page.declared_total_count
+                    required_pages = max(1, (expected_total + page_size - 1) // page_size)
+                    if required_pages > MAX_PAGES:
+                        raise KamisTransportError("page_budget_exceeded", page_number=page_number)
+                elif page.declared_total_count != expected_total:
+                    raise KamisTransportError("declared_total_changed", page_number=page_number)
+
+                remaining = expected_total - len(rows)
+                expected_row_count = min(page_size, max(0, remaining))
+                if len(page.items) != expected_row_count:
+                    raise KamisTransportError("page_row_count_mismatch", page_number=page_number)
+
+                rows.extend(page.items)
+                receipts.append(
+                    PageReceipt(
+                        ordinal=len(receipts) + 1,
+                        requested_page_number=page_number,
+                        declared_page_number=page.declared_page_number,
+                        declared_page_size=page.declared_page_size,
+                        declared_total_count=page.declared_total_count,
+                        row_count=len(page.items),
+                        http_status=200,
+                        provider_result_code=page.provider_result_code,
+                        byte_length=page.byte_length,
+                        body_sha256=page.body_sha256,
+                    )
                 )
-            )
 
-            if len(rows) == expected_total:
-                break
-            if len(rows) > expected_total:
-                raise KamisTransportError("row_total_exceeded", page_number=page_number)
-            page_number += 1
-            if page_number > MAX_PAGES:
-                raise KamisTransportError("page_budget_exceeded", page_number=page_number)
+                if len(rows) == expected_total:
+                    break
+                if len(rows) > expected_total:
+                    raise KamisTransportError("row_total_exceeded", page_number=page_number)
+                page_number += 1
+                if page_number > MAX_PAGES:
+                    raise KamisTransportError("page_budget_exceeded", page_number=page_number)
+        except KamisTransportError as error:
+            error._retain_completed_pages(tuple(receipts))
+            raise
 
         frozen_receipts = tuple(receipts)
         return KamisFetchResult(
diff --git a/grocery/source/persistence.py b/grocery/source/persistence.py
index 8ebb4b1..757a6c5 100644
--- a/grocery/source/persistence.py
+++ b/grocery/source/persistence.py
@@ -1,7 +1,8 @@
-"""Persist one successful KAMIS acquisition without retaining its raw payload."""
+"""Persist KAMIS acquisition evidence without retaining its raw payload."""
 
 from __future__ import annotations
 
+import re
 import uuid
 from dataclasses import dataclass
 
@@ -18,6 +19,7 @@ from grocery.models import (
     ordered_page_manifest_sha256,
 )
 from grocery.source.client import REDACTED_REQUEST_SHAPE, KamisFetchResult, KamisTransportError
+from grocery.source.client import PageReceipt as ClientPageReceipt
 
 
 @dataclass(frozen=True, slots=True)
@@ -88,6 +90,7 @@ _KNOWN_TRANSPORT_CODES = (
         }
     )
 )
+_SHA256 = re.compile(r"[0-9a-f]{64}\Z")
 
 
 @transaction.atomic
@@ -157,9 +160,18 @@ def fail_kamis_fetch(
 ) -> FetchAttempt:
     """Finalize a failed attempt using only the transport's redacted error fields."""
 
-    attempt = FetchAttempt.objects.select_for_update().get(pk=attempt_id)
+    attempt = (
+        FetchAttempt.objects.select_for_update()
+        .select_related("source_configuration")
+        .get(pk=attempt_id)
+    )
     if attempt.state != FetchAttempt.State.STARTED:
         raise ValidationError("Only a started fetch attempt can be failed.")
+    if attempt.page_receipts.exists():
+        raise ValidationError("A started fetch attempt already has page receipts.")
+
+    receipts = _partial_receipt_candidates(attempt, error)
+    PageReceipt.objects.bulk_create(receipts)
 
     state, failure_class = _classify_transport_failure(error)
     attempt.state = state
@@ -170,6 +182,9 @@ def fail_kamis_fetch(
         if error.code in _KNOWN_TRANSPORT_CODES
         else "UNCLASSIFIED_TRANSPORT_ERROR"
     )
+    attempt.received_page_count = len(receipts)
+    attempt.received_row_count = sum(receipt.received_row_count for receipt in receipts)
+    attempt.received_byte_count = sum(receipt.body_byte_length for receipt in receipts)
     attempt.save()
     return attempt
 
@@ -264,6 +279,108 @@ def _receipt_candidates(
     return candidates
 
 
+def _partial_receipt_candidates(
+    attempt: FetchAttempt,
+    error: KamisTransportError,
+) -> list[PageReceipt]:
+    receipts = error.partial_page_receipts
+    if not isinstance(receipts, tuple) or any(
+        not isinstance(receipt, ClientPageReceipt) for receipt in receipts
+    ):
+        raise ValidationError("Partial receipt evidence has an invalid container.")
+    source = attempt.source_configuration
+    if error.page_number is not None and error.page_number > source.max_pages_per_attempt:
+        raise ValidationError("The failed page exceeds the configured page budget.")
+    if error.attempt is not None and error.attempt > source.max_retries + 1:
+        raise ValidationError("The failed page exceeds the configured retry budget.")
+    if error.attempt is not None and error.attempt > source.max_requests_per_attempt:
+        raise ValidationError("The failed page exceeds the configured request budget.")
+    if not receipts:
+        return []
+
+    expected = list(range(1, len(receipts) + 1))
+    if len(receipts) > source.max_pages_per_attempt:
+        raise ValidationError("Partial receipts exceed the configured page budget.")
+    if len(receipts) > source.max_requests_per_attempt:
+        raise ValidationError("Partial receipts exceed the configured request budget.")
+    if [receipt.ordinal for receipt in receipts] != expected:
+        raise ValidationError("Partial receipt ordinals must be contiguous from one.")
+    if [receipt.requested_page_number for receipt in receipts] != expected:
+        raise ValidationError("Partial receipt page numbers must be contiguous from one.")
+    if error.page_number != len(receipts) + 1:
+        raise ValidationError("Partial receipts do not precede the failed page.")
+    minimum_request_count = len(receipts) + (error.attempt or 1)
+    if minimum_request_count > source.max_requests_per_attempt:
+        raise ValidationError("Partial evidence exceeds the configured request budget.")
+
+    if any(not _has_safe_partial_shape(receipt) for receipt in receipts):
+        raise ValidationError("Partial receipt evidence has an invalid safe shape.")
+    declared_totals = {receipt.declared_total_count for receipt in receipts}
+    declared_sizes = {receipt.declared_page_size for receipt in receipts}
+    if len(declared_totals) != 1 or len(declared_sizes) != 1:
+        raise ValidationError("Partial receipt source pagination changed between pages.")
+
+    declared_total = next(iter(declared_totals))
+    declared_page_size = next(iter(declared_sizes))
+    required_pages = max(1, (declared_total + declared_page_size - 1) // declared_page_size)
+    if required_pages > source.max_pages_per_attempt:
+        raise ValidationError("Partial receipts declare a total beyond the source page budget.")
+    if required_pages > source.max_requests_per_attempt:
+        raise ValidationError("Partial receipts declare a total beyond the source request budget.")
+    if sum(receipt.row_count for receipt in receipts) >= declared_total:
+        raise ValidationError("Partial receipt rows must be an incomplete prefix.")
+    if any(receipt.byte_length > source.max_page_bytes for receipt in receipts):
+        raise ValidationError("A partial receipt exceeds the configured byte budget.")
+
+    return [_page_receipt_candidate(attempt, receipt) for receipt in receipts]
+
+
+def _has_safe_partial_shape(receipt: ClientPageReceipt) -> bool:
+    integer_values = (
+        receipt.ordinal,
+        receipt.requested_page_number,
+        receipt.declared_page_number,
+        receipt.declared_page_size,
+        receipt.declared_total_count,
+        receipt.row_count,
+        receipt.http_status,
+        receipt.byte_length,
+    )
+    return (
+        all(isinstance(value, int) and not isinstance(value, bool) for value in integer_values)
+        and receipt.ordinal > 0
+        and receipt.requested_page_number == receipt.declared_page_number
+        and receipt.declared_page_size > 0
+        and receipt.declared_total_count > 0
+        and receipt.row_count == receipt.declared_page_size
+        and receipt.http_status == 200
+        and receipt.provider_result_code == "0"
+        and receipt.byte_length > 0
+        and isinstance(receipt.body_sha256, str)
+        and _SHA256.fullmatch(receipt.body_sha256) is not None
+    )
+
+
+def _page_receipt_candidate(
+    attempt: FetchAttempt,
+    receipt: ClientPageReceipt,
+) -> PageReceipt:
+    return PageReceipt(
+        fetch_attempt=attempt,
+        request_ordinal=receipt.ordinal,
+        page_number=receipt.requested_page_number,
+        http_status=receipt.http_status,
+        provider_result_code=receipt.provider_result_code,
+        declared_total_count=receipt.declared_total_count,
+        received_row_count=receipt.row_count,
+        body_state=PageReceipt.BodyState.RECEIVED,
+        body_byte_length=receipt.byte_length,
+        body_sha256=receipt.body_sha256,
+        media_type=PageReceipt.MediaType.JSON,
+        encoding=PageReceipt.Encoding.UTF_8,
+    )
+
+
 def _validate_completed_replay(
     attempt: FetchAttempt,
     result: KamisFetchResult,
diff --git a/grocery/tests/test_kamis_client.py b/grocery/tests/test_kamis_client.py
index 4608406..a60d8f8 100644
--- a/grocery/tests/test_kamis_client.py
+++ b/grocery/tests/test_kamis_client.py
@@ -22,6 +22,7 @@ from grocery.source.client import (
     JsonValue,
     KamisHttpClient,
     KamisTransportError,
+    PageReceipt,
 )
 
 
@@ -423,11 +424,75 @@ def test_declared_total_change_is_terminal() -> None:
         ]
     )
 
-    with pytest.raises(KamisTransportError, match="declared_total_changed"):
+    with pytest.raises(KamisTransportError, match="declared_total_changed") as raised:
         KamisHttpClient(open_url=opener, sleep=lambda _: None).fetch_recent_prices(
             "synthetic-key", page_size=1
         )
 
+    assert len(raised.value.partial_page_receipts) == 1
+    assert raised.value.partial_page_receipts[0].requested_page_number == 1
+
+
+@pytest.mark.parametrize(
+    ("failed_calls", "error_code", "http_status"),
+    [
+        ([TimeoutError() for _ in range(MAX_ATTEMPTS_PER_PAGE)], "retry_exhausted", None),
+        ([_http_error(429) for _ in range(MAX_ATTEMPTS_PER_PAGE)], "retry_exhausted", 429),
+        ([FakeResponse(b"{")], "invalid_json", None),
+    ],
+)
+def test_later_page_failure_retains_only_completed_page_receipts(
+    failed_calls: list[FakeResponse | Exception],
+    error_code: str,
+    http_status: int | None,
+) -> None:
+    first_body = _page_bytes(
+        page_number=1,
+        page_size=1,
+        total_count=2,
+        items=[_item(1)],
+    )
+    opener = FakeOpener([FakeResponse(first_body), *failed_calls])
+
+    with pytest.raises(KamisTransportError, match=error_code) as raised:
+        KamisHttpClient(open_url=opener, sleep=lambda _: None).fetch_recent_prices(
+            "synthetic-key", page_size=1
+        )
+
+    error = raised.value
+    assert error.page_number == 2
+    assert error.http_status == http_status
+    assert isinstance(error.partial_page_receipts, tuple)
+    assert error.partial_page_receipts == (
+        PageReceipt(
+            ordinal=1,
+            requested_page_number=1,
+            declared_page_number=1,
+            declared_page_size=1,
+            declared_total_count=2,
+            row_count=1,
+            http_status=200,
+            provider_result_code="0",
+            byte_length=len(first_body),
+            body_sha256=error.partial_page_receipts[0].body_sha256,
+        ),
+    )
+    assert not hasattr(error, "rows")
+    visible_error = f"{error!s} {error!r}"
+    assert error.partial_page_receipts[0].body_sha256 not in visible_error
+    assert "synthetic_id" not in visible_error
+
+
+def test_unknown_error_metadata_is_not_reflected_by_exception_text_or_repr() -> None:
+    marker = "KAMIS_API_KEY_synthetic_marker"
+
+    error = KamisTransportError(marker, provider_result_code=marker)
+
+    assert error.code == "unclassified_transport_error"
+    assert error.provider_result_code is None
+    assert marker not in str(error)
+    assert marker not in repr(error)
+
 
 def test_short_page_is_terminal() -> None:
     opener = FakeOpener([FakeResponse(_page_bytes(page_size=2, total_count=2, items=[_item(1)]))])
diff --git a/grocery/tests/test_source_persistence.py b/grocery/tests/test_source_persistence.py
index 0653ac9..449e48f 100644
--- a/grocery/tests/test_source_persistence.py
+++ b/grocery/tests/test_source_persistence.py
@@ -59,6 +59,23 @@ def make_result(*, second_hash: str = "b" * 64, call_count: int = 2) -> KamisFet
     )
 
 
+def make_partial_receipt(**overrides: object) -> ClientPageReceipt:
+    values: dict[str, object] = {
+        "ordinal": 1,
+        "requested_page_number": 1,
+        "declared_page_number": 1,
+        "declared_page_size": 2,
+        "declared_total_count": 3,
+        "row_count": 2,
+        "http_status": 200,
+        "provider_result_code": "0",
+        "byte_length": 20,
+        "body_sha256": "a" * 64,
+    }
+    values.update(overrides)
+    return ClientPageReceipt(**values)  # type: ignore[arg-type]
+
+
 def test_start_records_only_a_redacted_shape_for_an_active_recent_source() -> None:
     source = create_source_configuration()
     run_id = uuid.uuid4()
@@ -229,12 +246,180 @@ def test_failure_finalization_cannot_overwrite_a_terminal_attempt() -> None:
         fail_kamis_fetch(attempt.id, KamisTransportError("retry_exhausted"))
 
 
+@pytest.mark.parametrize(
+    ("error", "state", "failure_class"),
+    [
+        (
+            KamisTransportError(
+                "retry_exhausted",
+                page_number=2,
+                attempt=3,
+                partial_page_receipts=(make_partial_receipt(),),
+            ),
+            FetchAttempt.State.RETRYABLE_FAILED,
+            FetchAttempt.FailureClass.NETWORK,
+        ),
+        (
+            KamisTransportError(
+                "retry_exhausted",
+                page_number=2,
+                attempt=3,
+                http_status=429,
+                partial_page_receipts=(make_partial_receipt(),),
+            ),
+            FetchAttempt.State.RETRYABLE_FAILED,
+            FetchAttempt.FailureClass.HTTP_429,
+        ),
+        (
+            KamisTransportError(
+                "invalid_json",
+                page_number=2,
+                partial_page_receipts=(make_partial_receipt(),),
+            ),
+            FetchAttempt.State.TERMINAL_FAILED,
+            FetchAttempt.FailureClass.SCHEMA,
+        ),
+    ],
+)
+def test_failure_persists_only_completed_received_page_evidence(
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
+    assert failed.received_page_count == 1
+    assert failed.received_row_count == 2
+    assert failed.received_byte_count == 20
+    assert failed.artifact_id is None
+    receipt = PageReceipt.objects.get(fetch_attempt=attempt)
+    assert receipt.body_state == PageReceipt.BodyState.RECEIVED
+    assert receipt.body_absence_reason == ""
+    assert receipt.request_ordinal == receipt.page_number == 1
+    assert receipt.received_row_count == 2
+    assert not PageReceipt.objects.filter(
+        fetch_attempt=attempt,
+        body_state=PageReceipt.BodyState.NOT_RECEIVED,
+    ).exists()
+    assert not SourceArtifact.objects.exists()
+
+
+@pytest.mark.parametrize(
+    ("receipt_overrides", "source_overrides"),
+    [
+        ({"ordinal": 2}, {}),
+        ({"requested_page_number": 2, "declared_page_number": 2}, {}),
+        ({"declared_total_count": 2}, {}),
+        ({"body_sha256": "not-a-hash"}, {}),
+        ({"byte_length": 21}, {"max_page_bytes": 20}),
+        ({}, {"max_pages_per_attempt": 1}),
+        ({}, {"max_requests_per_attempt": 1}),
+    ],
+)
+def test_partial_receipt_or_source_budget_drift_rolls_back_atomically(
+    receipt_overrides: dict[str, object],
+    source_overrides: dict[str, object],
+) -> None:
+    source = create_source_configuration(**source_overrides)
+    attempt = start_kamis_fetch(source.id)
+    error = KamisTransportError(
+        "invalid_json",
+        page_number=2,
+        partial_page_receipts=(make_partial_receipt(**receipt_overrides),),
+    )
+
+    with pytest.raises(ValidationError):
+        fail_kamis_fetch(attempt.id, error)
+
+    attempt.refresh_from_db()
+    assert attempt.state == FetchAttempt.State.STARTED
+    assert attempt.received_page_count == 0
+    assert not attempt.page_receipts.exists()
+    assert not SourceArtifact.objects.exists()
+
+
+def test_partial_retry_attempts_are_included_in_the_minimum_request_budget() -> None:
+    source = create_source_configuration(max_requests_per_attempt=3)
+    attempt = start_kamis_fetch(source.id)
+    error = KamisTransportError(
+        "retry_exhausted",
+        page_number=2,
+        attempt=3,
+        partial_page_receipts=(make_partial_receipt(),),
+    )
+
+    with pytest.raises(ValidationError, match="request budget"):
+        fail_kamis_fetch(attempt.id, error)
+
+    attempt.refresh_from_db()
+    assert attempt.state == FetchAttempt.State.STARTED
+    assert not attempt.page_receipts.exists()
+
+
+@pytest.mark.parametrize(
+    ("source_overrides", "page_number", "retry_attempt"),
+    [
+        ({"max_pages_per_attempt": 1}, 2, None),
+        ({"max_retries": 1}, None, 3),
+        ({"max_requests_per_attempt": 2}, None, 3),
+    ],
+)
+def test_failure_coordinates_cannot_exceed_source_budgets(
+    source_overrides: dict[str, object],
+    page_number: int | None,
+    retry_attempt: int | None,
+) -> None:
+    source = create_source_configuration(**source_overrides)
+    attempt = start_kamis_fetch(source.id)
+
+    with pytest.raises(ValidationError):
+        fail_kamis_fetch(
+            attempt.id,
+            KamisTransportError(
+                "retry_exhausted",
+                page_number=page_number,
+                attempt=retry_attempt,
+            ),
+        )
+
+    attempt.refresh_from_db()
+    assert attempt.state == FetchAttempt.State.STARTED
+    assert not attempt.page_receipts.exists()
+
+
+def test_partial_failure_cannot_be_finalized_or_completed_twice() -> None:
+    source = create_source_configuration()
+    attempt = start_kamis_fetch(source.id)
+    error = KamisTransportError(
+        "invalid_json",
+        page_number=2,
+        partial_page_receipts=(make_partial_receipt(),),
+    )
+    fail_kamis_fetch(attempt.id, error)
+
+    with pytest.raises(ValidationError, match="started"):
+        fail_kamis_fetch(attempt.id, error)
+    with pytest.raises(ValidationError, match="started"):
+        complete_kamis_fetch(attempt.id, make_result())
+
+    assert PageReceipt.objects.filter(fetch_attempt=attempt).count() == 1
+    assert not SourceArtifact.objects.exists()
+
+
 def test_unknown_failure_code_is_not_copied_to_a_receipt_or_error_field() -> None:
     source = create_source_configuration()
     attempt = start_kamis_fetch(source.id)
     marker = "KAMIS_API_KEY_synthetic_marker"
 
-    failed = fail_kamis_fetch(attempt.id, KamisTransportError(marker))
+    error = KamisTransportError(marker)
+    failed = fail_kamis_fetch(attempt.id, error)
 
     assert failed.failure_code == "UNCLASSIFIED_TRANSPORT_ERROR"
     assert marker not in failed.failure_code
+    assert marker not in str(error)
+    assert marker not in repr(error)


