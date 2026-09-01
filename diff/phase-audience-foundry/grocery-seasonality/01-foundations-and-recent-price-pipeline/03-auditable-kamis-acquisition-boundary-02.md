## `feat(source): fetch kamis within strict bounds`

diff --git a/grocery/source/client.py b/grocery/source/client.py
new file mode 100644
index 0000000..8e470a9
--- /dev/null
+++ b/grocery/source/client.py
@@ -0,0 +1,602 @@
+"""Secret-safe HTTPS transport for the KAMIS recent-price endpoint.
+
+The transport deliberately retains only normalized rows and redacted receipts. Raw
+response bodies and request URLs exist only inside a single call and are never put in
+exceptions, logs, return values, or object representations.
+"""
+
+from __future__ import annotations
+
+import hashlib
+import json
+import re
+import ssl
+import time
+from collections.abc import Callable, Mapping
+from dataclasses import dataclass
+from email.message import Message
+from http.client import HTTPMessage
+from typing import Protocol, cast
+from urllib.error import HTTPError, URLError
+from urllib.parse import unquote, urlencode
+from urllib.request import (
+    HTTPRedirectHandler,
+    HTTPSHandler,
+    OpenerDirector,
+    Request,
+    build_opener,
+)
+
+KAMIS_ENDPOINT = "https://apis.data.go.kr/B552845/recent/price"
+REDACTED_REQUEST_SHAPE = (
+    "GET /B552845/recent/price parameters=[numOfRows,pageNo,returnType,serviceKey:<redacted>]"
+)
+CONNECT_READ_TIMEOUT_SECONDS = 10.0
+MAX_PAGE_BYTES = 4 * 1024 * 1024
+MAX_PAGES = 12
+MAX_CALLS = 12
+MAX_ATTEMPTS_PER_PAGE = 3
+DEFAULT_PAGE_SIZE = 100
+MAX_PAGE_SIZE = 1_000
+
+_REQUEST_PARAMETER_NAMES = frozenset({"serviceKey", "returnType", "pageNo", "numOfRows"})
+_SUCCESS_TOP_LEVEL_KEYS = frozenset({"response"})
+_SUCCESS_RESPONSE_KEYS = frozenset({"header", "body"})
+_HEADER_KEYS = frozenset({"resultCode", "resultMsg"})
+_BODY_KEYS = frozenset({"dataType", "items", "numOfRows", "pageNo", "totalCount"})
+_ITEMS_KEYS = frozenset({"item"})
+_SAFE_PROVIDER_CODE = re.compile(r"-?[0-9]{1,3}\Z")
+_RETRYABLE_PROVIDER_CODES = frozenset({"-1", "-5", "-10", "22", "23"})
+_RETRY_DELAYS_SECONDS = (0.25, 1.0)
+
+type JsonValue = None | bool | int | float | str | list[JsonValue] | dict[str, JsonValue]
+type JsonObject = dict[str, JsonValue]
+type OpenUrl = Callable[[Request, float], ResponseLike]
+type Sleep = Callable[[float], None]
+
+
+class ResponseLike(Protocol):
+    """Small response surface used by urllib and deterministic tests."""
+
+    @property
+    def status(self) -> int: ...
+
+    @property
+    def headers(self) -> HTTPMessage | Mapping[str, str]: ...
+
+    def read(self, amount: int | None = None) -> bytes: ...
+
+    def close(self) -> None: ...
+
+
+class _NoRedirectHandler(HTTPRedirectHandler):
+    def redirect_request(
+        self,
+        req: Request,
+        fp: object,
+        code: int,
+        msg: str,
+        headers: HTTPMessage,
+        newurl: str,
+    ) -> Request | None:
+        del req, fp, code, msg, headers, newurl
+        return None
+
+
+class KamisTransportError(RuntimeError):
+    """A redacted, operationally safe transport failure."""
+
+    def __init__(
+        self,
+        code: str,
+        *,
+        page_number: int | None = None,
+        attempt: int | None = None,
+        http_status: int | None = None,
+        provider_result_code: str | None = None,
+    ) -> None:
+        self.code = code
+        self.page_number = page_number
+        self.attempt = attempt
+        self.http_status = http_status
+        self.provider_result_code = provider_result_code
+        self.request_shape = REDACTED_REQUEST_SHAPE
+        details = [f"code={code}"]
+        if page_number is not None:
+            details.append(f"page={page_number}")
+        if attempt is not None:
+            details.append(f"attempt={attempt}")
+        if http_status is not None:
+            details.append(f"http_status={http_status}")
+        if provider_result_code is not None:
+            details.append(f"provider_result_code={provider_result_code}")
+        details.append(f"request={REDACTED_REQUEST_SHAPE}")
+        super().__init__(" ".join(details))
+
+
+@dataclass(frozen=True, slots=True)
+class PageReceipt:
+    """Raw-free evidence for one ordered source page."""
+
+    ordinal: int
+    requested_page_number: int
+    declared_page_number: int
+    declared_page_size: int
+    declared_total_count: int
+    row_count: int
+    http_status: int
+    provider_result_code: str
+    byte_length: int
+    body_sha256: str
+
+
+@dataclass(frozen=True, slots=True)
+class KamisFetchResult:
+    """In-memory source rows plus their deterministic ordered manifest."""
+
+    rows: tuple[JsonObject, ...]
+    page_receipts: tuple[PageReceipt, ...]
+    ordered_manifest_sha256: str
+    call_count: int
+
+
+@dataclass(frozen=True, slots=True)
+class _DecodedPage:
+    items: tuple[JsonObject, ...]
+    declared_page_number: int
+    declared_page_size: int
+    declared_total_count: int
+    provider_result_code: str
+    byte_length: int
+    body_sha256: str
+
+
+@dataclass(frozen=True, slots=True)
+class _RetrySignal:
+    code: str
+    http_status: int | None = None
+    provider_result_code: str | None = None
+
+
+class _CallBudget:
+    __slots__ = ("count",)
+
+    def __init__(self) -> None:
+        self.count = 0
+
+    def consume(self, *, page_number: int, attempt: int) -> None:
+        if self.count >= MAX_CALLS:
+            raise KamisTransportError(
+                "call_budget_exceeded", page_number=page_number, attempt=attempt
+            )
+        self.count += 1
+
+
+class KamisHttpClient:
+    """Fetch KAMIS pages without ever retaining a credential on the client."""
+
+    __slots__ = ("_open_url", "_sleep")
+
+    def __init__(
+        self,
+        *,
+        open_url: OpenUrl | None = None,
+        sleep: Sleep = time.sleep,
+    ) -> None:
+        self._open_url = open_url if open_url is not None else _default_open_url()
+        self._sleep = sleep
+
+    def __repr__(self) -> str:
+        return "KamisHttpClient(endpoint=/B552845/recent/price, credential=<not-retained>)"
+
+    def fetch_recent_prices(
+        self,
+        service_key: str,
+        *,
+        page_size: int = DEFAULT_PAGE_SIZE,
+    ) -> KamisFetchResult:
+        """Fetch every ordered page, keeping raw bytes only within this call."""
+
+        normalized_key = _normalize_service_key(service_key)
+        _validate_page_size(page_size)
+        budget = _CallBudget()
+        rows: list[JsonObject] = []
+        receipts: list[PageReceipt] = []
+        expected_total: int | None = None
+        page_number = 1
+
+        while True:
+            page = self._fetch_page(
+                normalized_key,
+                page_number=page_number,
+                page_size=page_size,
+                budget=budget,
+            )
+
+            if expected_total is None:
+                expected_total = page.declared_total_count
+                required_pages = max(1, (expected_total + page_size - 1) // page_size)
+                if required_pages > MAX_PAGES:
+                    raise KamisTransportError("page_budget_exceeded", page_number=page_number)
+            elif page.declared_total_count != expected_total:
+                raise KamisTransportError("declared_total_changed", page_number=page_number)
+
+            remaining = expected_total - len(rows)
+            expected_row_count = min(page_size, max(0, remaining))
+            if len(page.items) != expected_row_count:
+                raise KamisTransportError("page_row_count_mismatch", page_number=page_number)
+
+            rows.extend(page.items)
+            receipts.append(
+                PageReceipt(
+                    ordinal=len(receipts) + 1,
+                    requested_page_number=page_number,
+                    declared_page_number=page.declared_page_number,
+                    declared_page_size=page.declared_page_size,
+                    declared_total_count=page.declared_total_count,
+                    row_count=len(page.items),
+                    http_status=200,
+                    provider_result_code=page.provider_result_code,
+                    byte_length=page.byte_length,
+                    body_sha256=page.body_sha256,
+                )
+            )
+
+            if len(rows) == expected_total:
+                break
+            if len(rows) > expected_total:
+                raise KamisTransportError("row_total_exceeded", page_number=page_number)
+            page_number += 1
+            if page_number > MAX_PAGES:
+                raise KamisTransportError("page_budget_exceeded", page_number=page_number)
+
+        frozen_receipts = tuple(receipts)
+        return KamisFetchResult(
+            rows=tuple(rows),
+            page_receipts=frozen_receipts,
+            ordered_manifest_sha256=_ordered_manifest_sha256(frozen_receipts),
+            call_count=budget.count,
+        )
+
+    def _fetch_page(
+        self,
+        normalized_key: str,
+        *,
+        page_number: int,
+        page_size: int,
+        budget: _CallBudget,
+    ) -> _DecodedPage:
+        last_retry: _RetrySignal | None = None
+
+        for attempt in range(1, MAX_ATTEMPTS_PER_PAGE + 1):
+            budget.consume(page_number=page_number, attempt=attempt)
+            outcome = self._request_once(
+                normalized_key,
+                page_number=page_number,
+                page_size=page_size,
+            )
+            if isinstance(outcome, _DecodedPage):
+                return outcome
+
+            last_retry = outcome
+            if attempt < MAX_ATTEMPTS_PER_PAGE:
+                self._sleep(_RETRY_DELAYS_SECONDS[attempt - 1])
+
+        if last_retry is None:
+            raise KamisTransportError("invalid_retry_state", page_number=page_number)
+        raise KamisTransportError(
+            "retry_exhausted",
+            page_number=page_number,
+            attempt=MAX_ATTEMPTS_PER_PAGE,
+            http_status=last_retry.http_status,
+            provider_result_code=last_retry.provider_result_code,
+        )
+
+    def _request_once(
+        self,
+        normalized_key: str,
+        *,
+        page_number: int,
+        page_size: int,
+    ) -> _DecodedPage | _RetrySignal:
+        request = _build_request(
+            normalized_key,
+            page_number=page_number,
+            page_size=page_size,
+        )
+        response: ResponseLike | None = None
+        safe_error: KamisTransportError | None = None
+        retry: _RetrySignal | None = None
+        raw_body: bytes | None = None
+        status: int | None = None
+        content_type: str | None = None
+
+        try:
+            response = self._open_url(request, CONNECT_READ_TIMEOUT_SECONDS)
+            status = response.status
+            if not isinstance(status, int) or isinstance(status, bool):
+                safe_error = KamisTransportError("invalid_http_status", page_number=page_number)
+            elif 300 <= status < 400:
+                safe_error = KamisTransportError(
+                    "redirect_not_allowed", page_number=page_number, http_status=status
+                )
+            elif status == 429 or 500 <= status < 600:
+                retry = _RetrySignal("retryable_http_status", http_status=status)
+            elif status != 200:
+                safe_error = KamisTransportError(
+                    "terminal_http_status", page_number=page_number, http_status=status
+                )
+            else:
+                content_type = _validated_content_type(response.headers)
+                declared_length = _content_length(response.headers)
+                if declared_length is not None and declared_length > MAX_PAGE_BYTES:
+                    safe_error = KamisTransportError(
+                        "page_too_large", page_number=page_number, http_status=status
+                    )
+                else:
+                    candidate_body = response.read(MAX_PAGE_BYTES + 1)
+                    if isinstance(candidate_body, bytes):
+                        raw_body = candidate_body
+                    else:
+                        safe_error = KamisTransportError(
+                            "response_body_not_bytes", page_number=page_number
+                        )
+        except HTTPError as error:
+            status = error.code if isinstance(error.code, int) else None
+            if status is not None and 300 <= status < 400:
+                safe_error = KamisTransportError(
+                    "redirect_not_allowed", page_number=page_number, http_status=status
+                )
+            elif status == 429 or (status is not None and 500 <= status < 600):
+                retry = _RetrySignal("retryable_http_status", http_status=status)
+            else:
+                safe_error = KamisTransportError(
+                    "terminal_http_status", page_number=page_number, http_status=status
+                )
+            try:
+                error.close()
+            except Exception:  # noqa: S110 - dependency errors can include request URLs.
+                pass
+        except KamisTransportError as error:
+            safe_error = error
+        except ssl.SSLCertVerificationError:
+            safe_error = KamisTransportError("tls_verification_failed", page_number=page_number)
+        except ssl.SSLError:
+            safe_error = KamisTransportError("tls_error", page_number=page_number)
+        except TimeoutError:
+            retry = _RetrySignal("timeout")
+        except URLError, ConnectionError, OSError:
+            retry = _RetrySignal("network_error")
+        except Exception:
+            safe_error = KamisTransportError("transport_internal_error", page_number=page_number)
+        finally:
+            if response is not None:
+                try:
+                    response.close()
+                except Exception:  # noqa: S110 - dependency errors can include request URLs.
+                    pass
+
+        if safe_error is not None:
+            raise safe_error
+        if retry is not None:
+            return retry
+        if status != 200 or content_type != "application/json" or raw_body is None:
+            raise KamisTransportError("invalid_response_state", page_number=page_number)
+        if len(raw_body) > MAX_PAGE_BYTES:
+            raise KamisTransportError("page_too_large", page_number=page_number, http_status=status)
+
+        return _decode_page(
+            raw_body,
+            requested_page_number=page_number,
+            requested_page_size=page_size,
+        )
+
+
+def _default_open_url() -> OpenUrl:
+    context = ssl.create_default_context()
+    opener = build_opener(HTTPSHandler(context=context), _NoRedirectHandler())
+    director: OpenerDirector = opener
+
+    def open_url(request: Request, timeout: float) -> ResponseLike:
+        return cast(ResponseLike, director.open(request, timeout=timeout))
+
+    return open_url
+
+
+def _normalize_service_key(service_key: str) -> str:
+    if not isinstance(service_key, str) or not service_key:
+        raise KamisTransportError("service_key_missing")
+    normalized = unquote(service_key)
+    if not normalized:
+        raise KamisTransportError("service_key_missing")
+    return normalized
+
+
+def _validate_page_size(page_size: int) -> None:
+    if (
+        not isinstance(page_size, int)
+        or isinstance(page_size, bool)
+        or page_size < 1
+        or page_size > MAX_PAGE_SIZE
+    ):
+        raise KamisTransportError("invalid_page_size")
+
+
+def _build_request(normalized_key: str, *, page_number: int, page_size: int) -> Request:
+    parameters = {
+        "serviceKey": normalized_key,
+        "returnType": "json",
+        "pageNo": str(page_number),
+        "numOfRows": str(page_size),
+    }
+    if frozenset(parameters) != _REQUEST_PARAMETER_NAMES:
+        raise KamisTransportError("request_parameter_allowlist_violation")
+    query = urlencode(parameters, doseq=False, safe="")
+    return Request(  # noqa: S310 - the endpoint is a fixed HTTPS constant.
+        f"{KAMIS_ENDPOINT}?{query}",
+        headers={"Accept": "application/json"},
+        method="GET",
+    )
+
+
+def _validated_content_type(headers: HTTPMessage | Mapping[str, str]) -> str:
+    raw_content_type = headers.get("Content-Type")
+    if not isinstance(raw_content_type, str):
+        raise KamisTransportError("missing_content_type")
+    message = Message()
+    message["Content-Type"] = raw_content_type
+    media_type = message.get_content_type().lower()
+    charset = message.get_content_charset()
+    if media_type != "application/json":
+        raise KamisTransportError("unexpected_content_type")
+    if charset is None:
+        raise KamisTransportError("missing_charset")
+    try:
+        normalized_charset = charset.encode("ascii").decode("ascii").lower().replace("_", "-")
+    except UnicodeError:
+        raise KamisTransportError("unexpected_charset") from None
+    if normalized_charset not in {"utf-8", "utf8"}:
+        raise KamisTransportError("unexpected_charset")
+    return media_type
+
+
+def _content_length(headers: HTTPMessage | Mapping[str, str]) -> int | None:
+    raw_length = headers.get("Content-Length")
+    if raw_length is None:
+        return None
+    if not isinstance(raw_length, str) or not raw_length.isascii() or not raw_length.isdigit():
+        raise KamisTransportError("invalid_content_length")
+    return int(raw_length)
+
+
+def _decode_page(
+    raw_body: bytes,
+    *,
+    requested_page_number: int,
+    requested_page_size: int,
+) -> _DecodedPage | _RetrySignal:
+    decoded: JsonValue | None = None
+    decode_failed = False
+    try:
+        text = raw_body.decode("utf-8", errors="strict")
+        decoded = cast(JsonValue, json.loads(text, object_pairs_hook=_reject_duplicate_keys))
+    except UnicodeDecodeError, json.JSONDecodeError, _DuplicateJsonKeyError:
+        decode_failed = True
+    if decode_failed:
+        raise KamisTransportError("invalid_json", page_number=requested_page_number)
+
+    top = _require_object(decoded, "invalid_envelope", page_number=requested_page_number)
+    _require_exact_keys(top, _SUCCESS_TOP_LEVEL_KEYS, page_number=requested_page_number)
+    response = _require_object(
+        top["response"], "invalid_envelope", page_number=requested_page_number
+    )
+    _require_exact_keys(response, _SUCCESS_RESPONSE_KEYS, page_number=requested_page_number)
+    header = _require_object(
+        response["header"], "invalid_header", page_number=requested_page_number
+    )
+    _require_exact_keys(header, _HEADER_KEYS, page_number=requested_page_number)
+
+    provider_code_value = header["resultCode"]
+    provider_message = header["resultMsg"]
+    if (
+        not isinstance(provider_code_value, str)
+        or _SAFE_PROVIDER_CODE.fullmatch(provider_code_value) is None
+        or not isinstance(provider_message, str)
+    ):
+        raise KamisTransportError("invalid_provider_header", page_number=requested_page_number)
+    provider_code = provider_code_value
+    if provider_code != "0":
+        if provider_code in _RETRYABLE_PROVIDER_CODES:
+            return _RetrySignal("retryable_provider_error", provider_result_code=provider_code)
+        raise KamisTransportError(
+            "terminal_provider_error",
+            page_number=requested_page_number,
+            provider_result_code=provider_code,
+        )
+
+    body = _require_object(response["body"], "invalid_body", page_number=requested_page_number)
+    _require_exact_keys(body, _BODY_KEYS, page_number=requested_page_number)
+    if body["dataType"] != "JSON":
+        raise KamisTransportError("unexpected_data_type", page_number=requested_page_number)
+
+    declared_page_number = _require_nonnegative_int(
+        body["pageNo"], "invalid_declared_page", page_number=requested_page_number
+    )
+    declared_page_size = _require_nonnegative_int(
+        body["numOfRows"], "invalid_declared_page_size", page_number=requested_page_number
+    )
+    declared_total_count = _require_nonnegative_int(
+        body["totalCount"], "invalid_declared_total", page_number=requested_page_number
+    )
+    if declared_page_number != requested_page_number:
+        raise KamisTransportError("declared_page_mismatch", page_number=requested_page_number)
+    if declared_page_size != requested_page_size:
+        raise KamisTransportError("declared_page_size_mismatch", page_number=requested_page_number)
+
+    items_container = _require_object(
+        body["items"], "invalid_items_envelope", page_number=requested_page_number
+    )
+    _require_exact_keys(items_container, _ITEMS_KEYS, page_number=requested_page_number)
+    raw_items = items_container["item"]
+    if not isinstance(raw_items, list):
+        raise KamisTransportError("items_not_array", page_number=requested_page_number)
+    items: list[JsonObject] = []
+    for raw_item in raw_items:
+        if not isinstance(raw_item, dict):
+            raise KamisTransportError("item_not_object", page_number=requested_page_number)
+        items.append(raw_item)
+
+    return _DecodedPage(
+        items=tuple(items),
+        declared_page_number=declared_page_number,
+        declared_page_size=declared_page_size,
+        declared_total_count=declared_total_count,
+        provider_result_code=provider_code,
+        byte_length=len(raw_body),
+        body_sha256=hashlib.sha256(raw_body).hexdigest(),
+    )
+
+
+class _DuplicateJsonKeyError(ValueError):
+    pass
+
+
+def _reject_duplicate_keys(pairs: list[tuple[str, JsonValue]]) -> JsonObject:
+    result: JsonObject = {}
+    for key, value in pairs:
+        if key in result:
+            raise _DuplicateJsonKeyError
+        result[key] = value
+    return result
+
+
+def _require_object(value: JsonValue, code: str, *, page_number: int) -> JsonObject:
+    if not isinstance(value, dict):
+        raise KamisTransportError(code, page_number=page_number)
+    return value
+
+
+def _require_exact_keys(
+    value: JsonObject,
+    expected: frozenset[str],
+    *,
+    page_number: int,
+) -> None:
+    if frozenset(value) != expected:
+        raise KamisTransportError("unexpected_envelope_keys", page_number=page_number)
+
+
+def _require_nonnegative_int(value: JsonValue, code: str, *, page_number: int) -> int:
+    if not isinstance(value, int) or isinstance(value, bool) or value < 0:
+        raise KamisTransportError(code, page_number=page_number)
+    return value
+
+
+def _ordered_manifest_sha256(receipts: tuple[PageReceipt, ...]) -> str:
+    manifest = [receipt.body_sha256 for receipt in receipts]
+    canonical = json.dumps(
+        manifest,
+        ensure_ascii=True,
+        separators=(",", ":"),
+    ).encode("ascii")
+    return hashlib.sha256(canonical).hexdigest()
diff --git a/grocery/tests/test_kamis_client.py b/grocery/tests/test_kamis_client.py
new file mode 100644
index 0000000..4608406
--- /dev/null
+++ b/grocery/tests/test_kamis_client.py
@@ -0,0 +1,529 @@
+"""Synthetic transport tests; no request in this module reaches the network."""
+
+from __future__ import annotations
+
+import json
+import ssl
+from collections.abc import Mapping
+from email.message import Message
+from urllib.error import HTTPError, URLError
+from urllib.parse import parse_qs, urlsplit
+from urllib.request import Request
+
+import pytest
+
+from grocery.source.client import (
+    CONNECT_READ_TIMEOUT_SECONDS,
+    KAMIS_ENDPOINT,
+    MAX_ATTEMPTS_PER_PAGE,
+    MAX_PAGE_BYTES,
+    REDACTED_REQUEST_SHAPE,
+    JsonObject,
+    JsonValue,
+    KamisHttpClient,
+    KamisTransportError,
+)
+
+
+class FakeResponse:
+    def __init__(
+        self,
+        body: bytes,
+        *,
+        status: int = 200,
+        content_type: str | None = "application/json; charset=utf-8",
+        include_length: bool = True,
+    ) -> None:
+        self.body = body
+        self.status = status
+        headers: dict[str, str] = {}
+        if content_type is not None:
+            headers["Content-Type"] = content_type
+        if include_length:
+            headers["Content-Length"] = str(len(body))
+        self.headers: Mapping[str, str] = headers
+        self.read_amounts: list[int | None] = []
+        self.closed = False
+
+    def read(self, amount: int | None = None) -> bytes:
+        self.read_amounts.append(amount)
+        return self.body if amount is None else self.body[:amount]
+
+    def close(self) -> None:
+        self.closed = True
+
+
+class FakeOpener:
+    def __init__(self, scripted: list[FakeResponse | Exception]) -> None:
+        self.scripted = list(scripted)
+        self.requests: list[Request] = []
+        self.timeouts: list[float] = []
+
+    def __call__(self, request: Request, timeout: float) -> FakeResponse:
+        self.requests.append(request)
+        self.timeouts.append(timeout)
+        if not self.scripted:
+            raise AssertionError("unexpected synthetic request")
+        result = self.scripted.pop(0)
+        if isinstance(result, Exception):
+            raise result
+        return result
+
+
+class LeakyNetworkOpener:
+    """Simulates a dependency exception that contains the complete secret URL."""
+
+    def __init__(self) -> None:
+        self.requests: list[Request] = []
+
+    def __call__(self, request: Request, timeout: float) -> FakeResponse:
+        del timeout
+        self.requests.append(request)
+        raise URLError(request.full_url)
+
+    def __repr__(self) -> str:
+        return "LeakyNetworkOpener(secret=synthetic-secret-material)"
+
+
+def _item(identity: int) -> JsonObject:
+    return {"synthetic_id": identity}
+
+
+def _page_bytes(
+    *,
+    page_number: int = 1,
+    page_size: int = 100,
+    total_count: int = 1,
+    items: list[JsonObject] | None = None,
+    result_code: str = "0",
+    result_message: str = "NORMAL_SERVICE",
+) -> bytes:
+    item_values: list[JsonValue] = list(items if items is not None else [_item(1)])
+    payload: JsonObject = {
+        "response": {
+            "header": {"resultCode": result_code, "resultMsg": result_message},
+            "body": {
+                "dataType": "JSON",
+                "items": {"item": item_values},
+                "numOfRows": page_size,
+                "pageNo": page_number,
+                "totalCount": total_count,
+            },
+        }
+    }
+    return json.dumps(payload, ensure_ascii=False, separators=(",", ":")).encode()
+
+
+def _http_error(status: int) -> HTTPError:
+    return HTTPError(
+        "https://redacted.invalid/provider",
+        status,
+        "synthetic",
+        Message(),
+        None,
+    )
+
+
+def test_success_uses_only_fixed_https_parameters_and_encodes_decoding_key_once() -> None:
+    encoded_key = "synthetic%2Bkey%2Fsegment%3D"
+    decoded_key = "synthetic+key/segment="
+    response = FakeResponse(_page_bytes(items=[_item(7)]))
+    opener = FakeOpener([response])
+    client = KamisHttpClient(open_url=opener, sleep=lambda _: None)
+
+    result = client.fetch_recent_prices(encoded_key)
+
+    assert result.rows == (_item(7),)
+    assert result.call_count == 1
+    assert opener.timeouts == [CONNECT_READ_TIMEOUT_SECONDS]
+    assert response.read_amounts == [MAX_PAGE_BYTES + 1]
+    assert response.closed is True
+
+    request = opener.requests[0]
+    split = urlsplit(request.full_url)
+    endpoint = urlsplit(KAMIS_ENDPOINT)
+    assert request.get_method() == "GET"
+    assert split.scheme == "https"
+    assert split.netloc == endpoint.netloc
+    assert split.path == endpoint.path
+    assert request.get_header("Accept") == "application/json"
+    parameters = parse_qs(split.query, strict_parsing=True)
+    assert set(parameters) == {"serviceKey", "returnType", "pageNo", "numOfRows"}
+    assert parameters == {
+        "serviceKey": [decoded_key],
+        "returnType": ["json"],
+        "pageNo": ["1"],
+        "numOfRows": ["100"],
+    }
+    assert "serviceKey=synthetic%2Bkey%2Fsegment%3D" in split.query
+    assert "%252B" not in split.query
+
+
+def test_plain_decoding_key_is_encoded_by_the_http_client_once() -> None:
+    opener = FakeOpener([FakeResponse(_page_bytes())])
+
+    KamisHttpClient(open_url=opener, sleep=lambda _: None).fetch_recent_prices(
+        "synthetic+key/segment="
+    )
+
+    query = urlsplit(opener.requests[0].full_url).query
+    assert "serviceKey=synthetic%2Bkey%2Fsegment%3D" in query
+    assert "%252B" not in query
+
+
+def test_pagination_reconciles_order_counts_and_stable_manifest() -> None:
+    bodies = [
+        _page_bytes(page_number=1, page_size=2, total_count=5, items=[_item(1), _item(2)]),
+        _page_bytes(page_number=2, page_size=2, total_count=5, items=[_item(3), _item(4)]),
+        _page_bytes(page_number=3, page_size=2, total_count=5, items=[_item(5)]),
+    ]
+
+    first = KamisHttpClient(
+        open_url=FakeOpener([FakeResponse(body) for body in bodies]), sleep=lambda _: None
+    ).fetch_recent_prices("synthetic-key", page_size=2)
+    second = KamisHttpClient(
+        open_url=FakeOpener([FakeResponse(body) for body in bodies]), sleep=lambda _: None
+    ).fetch_recent_prices("synthetic-key", page_size=2)
+
+    assert [row["synthetic_id"] for row in first.rows] == [1, 2, 3, 4, 5]
+    assert [receipt.ordinal for receipt in first.page_receipts] == [1, 2, 3]
+    assert [receipt.requested_page_number for receipt in first.page_receipts] == [1, 2, 3]
+    assert [receipt.row_count for receipt in first.page_receipts] == [2, 2, 1]
+    assert all(receipt.declared_total_count == 5 for receipt in first.page_receipts)
+    assert first.call_count == 3
+    assert first.page_receipts == second.page_receipts
+    assert first.ordered_manifest_sha256 == second.ordered_manifest_sha256
+    assert len(first.ordered_manifest_sha256) == 64
+
+
+def test_zero_total_is_one_empty_reconciled_page() -> None:
+    opener = FakeOpener([FakeResponse(_page_bytes(total_count=0, items=[], page_size=100))])
+
+    result = KamisHttpClient(open_url=opener, sleep=lambda _: None).fetch_recent_prices(
+        "synthetic-key"
+    )
+
+    assert result.rows == ()
+    assert len(result.page_receipts) == 1
+    assert result.page_receipts[0].row_count == 0
+
+
+def test_declared_page_budget_is_enforced_before_a_second_call() -> None:
+    opener = FakeOpener([FakeResponse(_page_bytes(total_count=13, page_size=1, items=[_item(1)]))])
+
+    with pytest.raises(KamisTransportError, match="page_budget_exceeded"):
+        KamisHttpClient(open_url=opener, sleep=lambda _: None).fetch_recent_prices(
+            "synthetic-key", page_size=1
+        )
+
+    assert len(opener.requests) == 1
+
+
+def test_total_network_call_budget_includes_retries() -> None:
+    scripted: list[FakeResponse | Exception] = []
+    for page_number in range(1, 7):
+        scripted.extend(
+            [
+                _http_error(429),
+                FakeResponse(
+                    _page_bytes(
+                        page_number=page_number,
+                        page_size=1,
+                        total_count=7,
+                        items=[_item(page_number)],
+                    )
+                ),
+            ]
+        )
+    opener = FakeOpener(scripted)
+
+    with pytest.raises(KamisTransportError, match="call_budget_exceeded"):
+        KamisHttpClient(open_url=opener, sleep=lambda _: None).fetch_recent_prices(
+            "synthetic-key", page_size=1
+        )
+
+    assert len(opener.requests) == 12
+
+
+def test_http_429_retries_with_bounded_backoff_then_succeeds() -> None:
+    opener = FakeOpener([_http_error(429), FakeResponse(_page_bytes())])
+    sleeps: list[float] = []
+
+    result = KamisHttpClient(open_url=opener, sleep=sleeps.append).fetch_recent_prices(
+        "synthetic-key"
+    )
+
+    assert result.call_count == 2
+    assert sleeps == [0.25]
+
+
+@pytest.mark.parametrize("status", [500, 503])
+def test_retryable_http_status_exhausts_after_fixed_attempts(status: int) -> None:
+    opener = FakeOpener([_http_error(status) for _ in range(MAX_ATTEMPTS_PER_PAGE)])
+    sleeps: list[float] = []
+
+    with pytest.raises(KamisTransportError, match="retry_exhausted") as raised:
+        KamisHttpClient(open_url=opener, sleep=sleeps.append).fetch_recent_prices("synthetic-key")
+
+    assert raised.value.http_status == status
+    assert len(opener.requests) == MAX_ATTEMPTS_PER_PAGE
+    assert sleeps == [0.25, 1.0]
+
+
+@pytest.mark.parametrize("status", [301, 302, 307, 308])
+def test_redirect_is_terminal_and_never_followed(status: int) -> None:
+    opener = FakeOpener([_http_error(status)])
+
+    with pytest.raises(KamisTransportError, match="redirect_not_allowed") as raised:
+        KamisHttpClient(open_url=opener, sleep=lambda _: None).fetch_recent_prices("synthetic-key")
+
+    assert raised.value.http_status == status
+    assert len(opener.requests) == 1
+
+
+def test_retryable_provider_error_retries_without_using_provider_message() -> None:
+    provider_failure = _page_bytes(
+        result_code="-10",
+        result_message="synthetic-sensitive-message-must-not-appear",
+    )
+    opener = FakeOpener([FakeResponse(provider_failure), FakeResponse(_page_bytes())])
+
+    result = KamisHttpClient(open_url=opener, sleep=lambda _: None).fetch_recent_prices(
+        "synthetic-key"
+    )
+
+    assert result.call_count == 2
+
+
+@pytest.mark.parametrize("provider_code", ["-1", "-5", "-10", "22", "23"])
+def test_every_documented_transient_provider_code_is_bounded(provider_code: str) -> None:
+    opener = FakeOpener(
+        [FakeResponse(_page_bytes(result_code=provider_code)) for _ in range(MAX_ATTEMPTS_PER_PAGE)]
+    )
+
+    with pytest.raises(KamisTransportError, match="retry_exhausted") as raised:
+        KamisHttpClient(open_url=opener, sleep=lambda _: None).fetch_recent_prices("synthetic-key")
+
+    assert raised.value.provider_result_code == provider_code
+    assert len(opener.requests) == MAX_ATTEMPTS_PER_PAGE
+
+
+def test_terminal_provider_error_is_not_retried_or_echoed() -> None:
+    provider_message = "synthetic-sensitive-message-must-not-appear"
+    opener = FakeOpener(
+        [FakeResponse(_page_bytes(result_code="-3", result_message=provider_message))]
+    )
+
+    with pytest.raises(KamisTransportError, match="terminal_provider_error") as raised:
+        KamisHttpClient(open_url=opener, sleep=lambda _: None).fetch_recent_prices("synthetic-key")
+
+    assert raised.value.provider_result_code == "-3"
+    assert provider_message not in str(raised.value)
+    assert len(opener.requests) == 1
+
+
+@pytest.mark.parametrize(
+    ("content_type", "error_code"),
+    [
+        ("text/html; charset=utf-8", "unexpected_content_type"),
+        ("application/json", "missing_charset"),
+        ("application/json; charset=euc-kr", "unexpected_charset"),
+        (None, "missing_content_type"),
+    ],
+)
+def test_content_type_and_utf8_are_strict(
+    content_type: str | None,
+    error_code: str,
+) -> None:
+    opener = FakeOpener([FakeResponse(_page_bytes(), content_type=content_type)])
+
+    with pytest.raises(KamisTransportError, match=error_code):
+        KamisHttpClient(open_url=opener, sleep=lambda _: None).fetch_recent_prices("synthetic-key")
+
+
+def test_stream_larger_than_four_mib_is_rejected_without_parsing() -> None:
+    response = FakeResponse(b"x" * (MAX_PAGE_BYTES + 1), include_length=False)
+    opener = FakeOpener([response])
+
+    with pytest.raises(KamisTransportError, match="page_too_large"):
+        KamisHttpClient(open_url=opener, sleep=lambda _: None).fetch_recent_prices("synthetic-key")
+
+    assert response.read_amounts == [MAX_PAGE_BYTES + 1]
+
+
+def test_declared_oversize_is_rejected_before_reading() -> None:
+    response = FakeResponse(b"{}")
+    response.headers = {
+        "Content-Type": "application/json; charset=utf-8",
+        "Content-Length": str(MAX_PAGE_BYTES + 1),
+    }
+    opener = FakeOpener([response])
+
+    with pytest.raises(KamisTransportError, match="page_too_large"):
+        KamisHttpClient(open_url=opener, sleep=lambda _: None).fetch_recent_prices("synthetic-key")
+
+    assert response.read_amounts == []
+
+
+@pytest.mark.parametrize(
+    "mutation",
+    [
+        "extra_top_key",
+        "extra_body_key",
+        "wrong_data_type",
+        "items_not_array",
+        "wrong_declared_page",
+        "wrong_declared_size",
+    ],
+)
+def test_success_envelope_and_pagination_schema_are_exact(mutation: str) -> None:
+    payload = json.loads(_page_bytes())
+    assert isinstance(payload, dict)
+    response = payload["response"]
+    assert isinstance(response, dict)
+    body = response["body"]
+    assert isinstance(body, dict)
+    if mutation == "extra_top_key":
+        payload["unexpected"] = True
+    elif mutation == "extra_body_key":
+        body["unexpected"] = True
+    elif mutation == "wrong_data_type":
+        body["dataType"] = "XML"
+    elif mutation == "items_not_array":
+        body["items"] = {"item": {}}
+    elif mutation == "wrong_declared_page":
+        body["pageNo"] = 2
+    elif mutation == "wrong_declared_size":
+        body["numOfRows"] = 99
+    raw = json.dumps(payload, separators=(",", ":")).encode()
+
+    with pytest.raises(KamisTransportError):
+        KamisHttpClient(
+            open_url=FakeOpener([FakeResponse(raw)]), sleep=lambda _: None
+        ).fetch_recent_prices("synthetic-key")
+
+
+def test_duplicate_json_keys_are_terminal_schema_failure() -> None:
+    raw = (
+        b'{"response":{"header":{"resultCode":"0","resultCode":"0",'
+        b'"resultMsg":"NORMAL_SERVICE"},"body":{}}}'
+    )
+
+    with pytest.raises(KamisTransportError, match="invalid_json"):
+        KamisHttpClient(
+            open_url=FakeOpener([FakeResponse(raw)]), sleep=lambda _: None
+        ).fetch_recent_prices("synthetic-key")
+
+
+def test_declared_total_change_is_terminal() -> None:
+    opener = FakeOpener(
+        [
+            FakeResponse(_page_bytes(page_number=1, page_size=1, total_count=2, items=[_item(1)])),
+            FakeResponse(_page_bytes(page_number=2, page_size=1, total_count=3, items=[_item(2)])),
+        ]
+    )
+
+    with pytest.raises(KamisTransportError, match="declared_total_changed"):
+        KamisHttpClient(open_url=opener, sleep=lambda _: None).fetch_recent_prices(
+            "synthetic-key", page_size=1
+        )
+
+
+def test_short_page_is_terminal() -> None:
+    opener = FakeOpener([FakeResponse(_page_bytes(page_size=2, total_count=2, items=[_item(1)]))])
+
+    with pytest.raises(KamisTransportError, match="page_row_count_mismatch"):
+        KamisHttpClient(open_url=opener, sleep=lambda _: None).fetch_recent_prices(
+            "synthetic-key", page_size=2
+        )
+
+
+def test_network_dependency_cannot_leak_key_url_or_opener_repr(
+    caplog: pytest.LogCaptureFixture,
+) -> None:
+    encoded_key = "synthetic%2Bsecret%2Fmaterial%3D"
+    decoded_key = "synthetic+secret/material="
+    opener = LeakyNetworkOpener()
+    client = KamisHttpClient(open_url=opener, sleep=lambda _: None)
+
+    with pytest.raises(KamisTransportError, match="retry_exhausted") as raised:
+        client.fetch_recent_prices(encoded_key)
+
+    full_url = opener.requests[0].full_url
+    visible_error = f"{raised.value!s} {raised.value!r}"
+    assert encoded_key not in visible_error
+    assert decoded_key not in visible_error
+    assert full_url not in visible_error
+    assert "synthetic-secret-material" not in repr(client)
+    assert raised.value.request_shape == REDACTED_REQUEST_SHAPE
+    assert "<redacted>" in visible_error
+    assert raised.value.__context__ is None
+    assert caplog.records == []
+
+
+def test_tls_failure_is_terminal_and_not_retried() -> None:
+    opener = FakeOpener([ssl.SSLError("synthetic TLS failure")])
+
+    with pytest.raises(KamisTransportError, match="tls_error"):
+        KamisHttpClient(open_url=opener, sleep=lambda _: None).fetch_recent_prices("synthetic-key")
+
+    assert len(opener.requests) == 1
+
+
+@pytest.mark.parametrize("service_key", ["", None])
+def test_missing_key_is_rejected_before_any_request(service_key: str | None) -> None:
+    opener = FakeOpener([])
+
+    with pytest.raises(KamisTransportError, match="service_key_missing"):
+        KamisHttpClient(open_url=opener, sleep=lambda _: None).fetch_recent_prices(
+            service_key  # type: ignore[arg-type]
+        )
+
+    assert opener.requests == []
+
+
+@pytest.mark.parametrize("page_size", [0, -1, 1001, True])
+def test_invalid_page_size_is_terminal_before_any_request(page_size: int) -> None:
+    opener = FakeOpener([])
+
+    with pytest.raises(KamisTransportError, match="invalid_page_size"):
+        KamisHttpClient(open_url=opener, sleep=lambda _: None).fetch_recent_prices(
+            "synthetic-key", page_size=page_size
+        )
+
+    assert opener.requests == []
+
+
+def test_non_retryable_http_auth_failure_is_redacted_and_terminal() -> None:
+    opener = FakeOpener([_http_error(401)])
+
+    with pytest.raises(KamisTransportError, match="terminal_http_status") as raised:
+        KamisHttpClient(open_url=opener, sleep=lambda _: None).fetch_recent_prices("synthetic-key")
+
+    assert raised.value.http_status == 401
+    assert len(opener.requests) == 1
+
+
+def test_unsafe_provider_code_is_schema_error_not_echoed() -> None:
+    unsafe_code = "synthetic-key-material"
+    opener = FakeOpener([FakeResponse(_page_bytes(result_code=unsafe_code))])
+
+    with pytest.raises(KamisTransportError, match="invalid_provider_header") as raised:
+        KamisHttpClient(open_url=opener, sleep=lambda _: None).fetch_recent_prices("synthetic-key")
+
+    assert unsafe_code not in str(raised.value)
+
+
+def test_json_values_are_kept_in_memory_without_raw_response_artifact() -> None:
+    nested: JsonValue = {"name": "합성 품목", "missing": None, "flags": [True, 1]}
+    item: JsonObject = {"nested": nested}
+    body = _page_bytes(items=[item])
+    opener = FakeOpener([FakeResponse(body)])
+
+    result = KamisHttpClient(open_url=opener, sleep=lambda _: None).fetch_recent_prices(
+        "synthetic-key"
+    )
+
+    assert result.rows == (item,)
+    assert not hasattr(result, "raw_body")
+    assert not hasattr(result.page_receipts[0], "raw_body")


