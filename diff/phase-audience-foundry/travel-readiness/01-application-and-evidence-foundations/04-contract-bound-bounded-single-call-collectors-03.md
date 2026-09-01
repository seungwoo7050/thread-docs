## `feat(source): expose per-call aviation receipts`

diff --git a/sources/tests/test_transport.py b/sources/tests/test_transport.py
index 566f321..918cc3a 100644
--- a/sources/tests/test_transport.py
+++ b/sources/tests/test_transport.py
@@ -19,6 +19,7 @@ from sources.transport import (
     FAILURE_TIMEOUT,
     FAILURE_TRANSPORT,
     FAILURE_UPSTREAM_5XX,
+    PROVIDER_DATA_GO_SUCCESS,
     PROVIDER_GATEWAY_20,
     PROVIDER_OTHER_ERROR,
     PROVIDER_SUCCESS_0,
@@ -26,8 +27,19 @@ from sources.transport import (
     WARNING_REQUEST_FINGERPRINT,
     WARNING_SECRET_REFERENCE,
     WARNING_SOURCE_LOCATOR,
+    ROLE_DESTINATION_CITY,
+    ROLE_LEGACY_ARRIVAL,
+    ROLE_SCHEDULE_ARRIVAL,
+    ROLE_SCHEDULE_DEPARTURE,
+    RequestFingerprint,
+    SingleAttemptResult,
+    destination_city_request_fingerprint,
+    fetch_data_go_reference_evidence,
+    fetch_data_go_schedule_pages,
     fetch_entry_attachment,
     fetch_travel_alarm_jp,
+    legacy_arrival_request_fingerprint,
+    schedule_page_request_fingerprint,
     warning_request_fingerprint,
 )
 
@@ -125,6 +137,23 @@ def warning_body(result_code="0"):
     ).encode("utf-8")
 
 
+def data_go_page(*, page=1, page_size=100, total=1):
+    return json.dumps(
+        {
+            "response": {
+                "header": {"resultCode": "00", "resultMsg": "NORMAL"},
+                "body": {
+                    "items": {"item": []},
+                    "pageNo": page,
+                    "numOfRows": page_size,
+                    "totalCount": total,
+                },
+            }
+        },
+        separators=(",", ":"),
+    ).encode("utf-8")
+
+
 class TransportTestCase(unittest.TestCase):
     def entry_fetch(self, connection):
         return fetch_entry_attachment(
@@ -587,6 +616,233 @@ class TransportTestCase(unittest.TestCase):
                 self.assertEqual(result.failure_code, expected_failure)
                 self.assertFalse(connection.connected)
 
+    def test_aviation_executor_wraps_each_actual_page_with_redacted_identity(self):
+        connections = [
+            FakeConnection(FakeResponse(200, data_go_page())),
+            FakeConnection(FakeResponse(200, data_go_page())),
+        ]
+        factory_calls = []
+
+        def factory(host, port, timeout):
+            factory_calls.append((host, port, timeout))
+            return connections.pop(0)
+
+        executor_calls = []
+
+        def executor(*, source_code, request_fingerprint, call_once):
+            executor_calls.append(
+                (source_code, request_fingerprint, len(factory_calls))
+            )
+            return call_once()
+
+        result = fetch_data_go_schedule_pages(
+            secret_reference_name="DATA_GO_KR_SERVICE_KEY",
+            secret_value="synthetic-sensitive-key",
+            season="S26",
+            connect_timeout_seconds=3,
+            read_timeout_seconds=8,
+            connection_factory=factory,
+            attempt_executor=executor,
+        )
+
+        self.assertTrue(result.succeeded)
+        self.assertEqual([row[2] for row in executor_calls], [0, 1])
+        self.assertEqual(
+            [(row.role, row.role_sequence) for row in result.lineage],
+            [
+                (ROLE_SCHEDULE_DEPARTURE, 1),
+                (ROLE_SCHEDULE_ARRIVAL, 1),
+            ],
+        )
+        self.assertEqual(
+            [row.attempt.provider_code for row in result.lineage],
+            [PROVIDER_DATA_GO_SUCCESS, PROVIDER_DATA_GO_SUCCESS],
+        )
+        self.assertNotEqual(
+            result.lineage[0].attempt.request_fingerprint,
+            result.lineage[1].attempt.request_fingerprint,
+        )
+        self.assertNotIn("synthetic-sensitive-key", repr(result))
+
+    def test_aviation_multi_page_lineage_is_ordered_and_hashes_each_body(self):
+        bodies = (
+            data_go_page(page=1, total=101),
+            data_go_page(page=2, total=101),
+            data_go_page(),
+        )
+        connections = [
+            FakeConnection(FakeResponse(200, body)) for body in bodies
+        ]
+
+        def factory(_host, _port, _timeout):
+            return connections.pop(0)
+
+        result = fetch_data_go_schedule_pages(
+            secret_reference_name="DATA_GO_KR_SERVICE_KEY",
+            secret_value="synthetic-sensitive-key",
+            season="S26",
+            connect_timeout_seconds=3,
+            read_timeout_seconds=8,
+            connection_factory=factory,
+        )
+
+        self.assertTrue(result.succeeded)
+        self.assertEqual(
+            [(row.role, row.role_sequence) for row in result.lineage],
+            [
+                (ROLE_SCHEDULE_DEPARTURE, 1),
+                (ROLE_SCHEDULE_DEPARTURE, 2),
+                (ROLE_SCHEDULE_ARRIVAL, 1),
+            ],
+        )
+        self.assertEqual(
+            [row.attempt.response_sha256 for row in result.lineage],
+            [hashlib.sha256(body).hexdigest() for body in bodies],
+        )
+
+    def test_aviation_failed_call_retains_actual_failure_and_prior_lineage(self):
+        first_body = data_go_page(page=1, total=101)
+        connections = [
+            FakeConnection(FakeResponse(200, first_body)),
+            FakeConnection(FakeResponse(503, b"upstream unavailable")),
+        ]
+
+        def factory(_host, _port, _timeout):
+            return connections.pop(0)
+
+        result = fetch_data_go_schedule_pages(
+            secret_reference_name="DATA_GO_KR_SERVICE_KEY",
+            secret_value="Q7vR4mP2zN8kL6xT",
+            season="S26",
+            connect_timeout_seconds=3,
+            read_timeout_seconds=8,
+            connection_factory=factory,
+        )
+
+        self.assertFalse(result.succeeded)
+        self.assertEqual(result.departure_pages, (first_body,))
+        self.assertEqual(len(result.lineage), 2)
+        failed = result.lineage[-1].attempt
+        self.assertEqual(failed.attempt_result, ATTEMPT_RETRYABLE_FAILED)
+        self.assertEqual(failed.http_status, 503)
+        self.assertEqual(failed.failure_code, FAILURE_UPSTREAM_5XX)
+        self.assertEqual(failed.body, b"")
+
+    def test_aviation_schema_drift_keeps_the_actual_success_receipt(self):
+        drift_body = b'{"unexpected":"schema"}'
+        connection = FakeConnection(FakeResponse(200, drift_body))
+
+        result = fetch_data_go_schedule_pages(
+            secret_reference_name="DATA_GO_KR_SERVICE_KEY",
+            secret_value="synthetic-sensitive-key",
+            season="S26",
+            connect_timeout_seconds=3,
+            read_timeout_seconds=8,
+            connection_factory=FakeFactory(connection),
+        )
+
+        self.assertFalse(result.succeeded)
+        self.assertEqual(result.departure_pages, (drift_body,))
+        self.assertEqual(len(result.lineage), 1)
+        receipt = result.lineage[0].attempt
+        self.assertTrue(receipt.succeeded)
+        self.assertEqual(receipt.provider_code, "")
+        self.assertEqual(
+            receipt.response_sha256,
+            hashlib.sha256(drift_body).hexdigest(),
+        )
+
+    def test_aviation_reference_lineage_names_city_and_legacy_calls(self):
+        city_body = b'{"response":{"header":{"resultCode":"00"},"body":{}}}'
+        legacy_body = data_go_page()
+        connections = [
+            FakeConnection(FakeResponse(200, city_body)),
+            FakeConnection(FakeResponse(200, legacy_body)),
+        ]
+
+        def factory(_host, _port, _timeout):
+            return connections.pop(0)
+
+        result = fetch_data_go_reference_evidence(
+            secret_reference_name="DATA_GO_KR_SERVICE_KEY",
+            secret_value="synthetic-sensitive-key",
+            connect_timeout_seconds=3,
+            read_timeout_seconds=8,
+            connection_factory=factory,
+        )
+
+        self.assertTrue(result.succeeded)
+        self.assertEqual(
+            [(row.role, row.role_sequence) for row in result.lineage],
+            [(ROLE_DESTINATION_CITY, 1), (ROLE_LEGACY_ARRIVAL, 1)],
+        )
+        self.assertEqual(result.city_payload, city_body)
+        self.assertEqual(result.legacy_arrival_pages, (legacy_body,))
+
+    def test_aviation_fingerprints_are_page_scoped_and_secret_free(self):
+        first = schedule_page_request_fingerprint(
+            role=ROLE_SCHEDULE_DEPARTURE,
+            role_sequence=1,
+            season="S26",
+        )
+        second = schedule_page_request_fingerprint(
+            role=ROLE_SCHEDULE_DEPARTURE,
+            role_sequence=2,
+            season="S26",
+        )
+        arrival = schedule_page_request_fingerprint(
+            role=ROLE_SCHEDULE_ARRIVAL,
+            role_sequence=1,
+            season="S26",
+        )
+
+        self.assertEqual(first.revision, "ICN_SCHEDULE_DEPARTURE_V1")
+        self.assertEqual(arrival.revision, "ICN_SCHEDULE_ARRIVAL_V1")
+        self.assertEqual(len({first, second, arrival}), 3)
+        self.assertEqual(
+            destination_city_request_fingerprint().revision,
+            "ICN_DESTINATION_CITY_V1",
+        )
+        self.assertEqual(
+            legacy_arrival_request_fingerprint(1).revision,
+            "ICN_LEGACY_ARRIVAL_V1",
+        )
+
+    def test_aviation_executor_cannot_replace_an_actual_call_with_bytes(self):
+        connection = FakeConnection(FakeResponse(200, data_go_page()))
+
+        def executor(*, source_code, request_fingerprint, call_once):
+            del source_code, call_once
+            forged = b"forged bytes-only receipt"
+            return SingleAttemptResult(
+                request_fingerprint=RequestFingerprint(
+                    request_fingerprint.revision,
+                    request_fingerprint.normalized_request_sha256,
+                ),
+                attempt_result=ATTEMPT_SUCCEEDED,
+                http_status=200,
+                response_sha256=hashlib.sha256(forged).hexdigest(),
+                byte_count=len(forged),
+                body=forged,
+            )
+
+        result = fetch_data_go_schedule_pages(
+            secret_reference_name="DATA_GO_KR_SERVICE_KEY",
+            secret_value="Q7vR4mP2zN8kL6xT",
+            season="S26",
+            connect_timeout_seconds=3,
+            read_timeout_seconds=8,
+            connection_factory=FakeFactory(connection),
+            attempt_executor=executor,
+        )
+
+        self.assertFalse(result.succeeded)
+        self.assertFalse(connection.connected)
+        self.assertEqual(
+            result.lineage[0].attempt.failure_code,
+            FAILURE_TRANSPORT,
+        )
+
 
 if __name__ == "__main__":
     unittest.main()
diff --git a/sources/transport.py b/sources/transport.py
index bee9576..4065da3 100644
--- a/sources/transport.py
+++ b/sources/transport.py
@@ -20,9 +20,12 @@ from urllib.parse import parse_qsl, quote, quote_plus, unquote, urlencode, urlsp
 from xml.etree import ElementTree
 
 from travel_windows.contracts import (
+    CITY_SOURCE_CODE,
     CITY_SOURCE_LOCATOR,
     DATA_GO_SECRET_REFERENCE,
+    LEGACY_SCHEDULE_SOURCE_CODE,
     LEGACY_SCHEDULE_ARRIVALS_LOCATOR,
+    SCHEDULE_SOURCE_CODE,
     SCHEDULE_ARRIVALS_LOCATOR,
     SCHEDULE_DEPARTURES_LOCATOR,
     load_data_go_service_key,
@@ -63,6 +66,13 @@ FAILURE_RESPONSE_TOO_LARGE = "RESPONSE_TOO_LARGE"
 PROVIDER_SUCCESS_0 = "MOFA_SUCCESS_0"
 PROVIDER_GATEWAY_20 = "MOFA_GATEWAY_20"
 PROVIDER_OTHER_ERROR = "MOFA_OTHER_ERROR"
+PROVIDER_DATA_GO_SUCCESS = "DATA_GO_SUCCESS"
+PROVIDER_DATA_GO_ERROR = "DATA_GO_ERROR"
+
+ROLE_SCHEDULE_DEPARTURE = "SCHEDULE_DEPARTURE"
+ROLE_SCHEDULE_ARRIVAL = "SCHEDULE_ARRIVAL"
+ROLE_DESTINATION_CITY = "DESTINATION_CITY"
+ROLE_LEGACY_ARRIVAL = "LEGACY_ARRIVAL"
 
 
 def load_aviation_secret_reference(
@@ -81,6 +91,10 @@ class SchedulePageFetchResult:
     departure_pages: tuple[bytes, ...] = field(default=(), repr=False)
     arrival_pages: tuple[bytes, ...] = field(default=(), repr=False)
     failure_code: str = ""
+    lineage: tuple[AviationResponseLineage, ...] = field(
+        default=(),
+        repr=False,
+    )
 
 
 @dataclass(frozen=True, slots=True)
@@ -92,6 +106,10 @@ class AviationReferenceFetchResult:
         repr=False,
     )
     failure_code: str = ""
+    lineage: tuple[AviationResponseLineage, ...] = field(
+        default=(),
+        repr=False,
+    )
 
 
 @dataclass(frozen=True, slots=True)
@@ -118,6 +136,42 @@ class SingleAttemptResult:
         return self.attempt_result == ATTEMPT_SUCCEEDED
 
 
+@dataclass(frozen=True, slots=True)
+class AviationResponseLineage:
+    """One ordered source response and its exact single-call receipt shape."""
+
+    source_code: str
+    role: str
+    role_sequence: int
+    attempt: SingleAttemptResult
+
+    @property
+    def succeeded(self) -> bool:
+        return self.attempt.succeeded
+
+
+class AviationAttemptExecutor(Protocol):
+    """Persistence boundary that wraps exactly one actual HTTP call."""
+
+    def __call__(
+        self,
+        *,
+        source_code: str,
+        request_fingerprint: RequestFingerprint,
+        call_once: Callable[[], SingleAttemptResult],
+    ) -> SingleAttemptResult: ...
+
+
+def _direct_attempt_executor(
+    *,
+    source_code: str,
+    request_fingerprint: RequestFingerprint,
+    call_once: Callable[[], SingleAttemptResult],
+) -> SingleAttemptResult:
+    del source_code, request_fingerprint
+    return call_once()
+
+
 class _SocketLike(Protocol):
     def settimeout(self, timeout: float) -> None: ...
 
@@ -469,6 +523,198 @@ def _contains_secret_reflection(
     return False
 
 
+def schedule_page_request_fingerprint(
+    *,
+    role: str,
+    role_sequence: int,
+    season: str,
+) -> RequestFingerprint:
+    if role == ROLE_SCHEDULE_DEPARTURE:
+        locator = SCHEDULE_DEPARTURES_LOCATOR
+        revision = "ICN_SCHEDULE_DEPARTURE_V1"
+    elif role == ROLE_SCHEDULE_ARRIVAL:
+        locator = SCHEDULE_ARRIVALS_LOCATOR
+        revision = "ICN_SCHEDULE_ARRIVAL_V1"
+    else:
+        raise ValueError("unknown schedule response role")
+    if (
+        type(role_sequence) is not int
+        or not 1 <= role_sequence <= SCHEDULE_MAX_PAGES_PER_DIRECTION
+        or type(season) is not str
+        or not re.fullmatch(r"[A-Za-z0-9가-힣_-]{2,32}", season)
+    ):
+        raise ValueError("invalid schedule page identity")
+    parts = urlsplit(locator)
+    return RequestFingerprint(
+        revision=revision,
+        normalized_request_sha256=_request_hash(
+            parts.hostname or "",
+            parts.path,
+            [
+                ("serviceKey", "<redacted>"),
+                ("pageNo", str(role_sequence)),
+                ("numOfRows", str(SCHEDULE_PAGE_SIZE)),
+                ("season", season),
+                ("type", "json"),
+            ],
+        ),
+    )
+
+
+def destination_city_request_fingerprint() -> RequestFingerprint:
+    parts = urlsplit(CITY_SOURCE_LOCATOR)
+    return RequestFingerprint(
+        revision="ICN_DESTINATION_CITY_V1",
+        normalized_request_sha256=_request_hash(
+            parts.hostname or "",
+            parts.path,
+            [("serviceKey", "<redacted>"), ("type", "json")],
+        ),
+    )
+
+
+def legacy_arrival_request_fingerprint(
+    role_sequence: int,
+) -> RequestFingerprint:
+    if (
+        type(role_sequence) is not int
+        or not 1 <= role_sequence <= SCHEDULE_MAX_PAGES_PER_DIRECTION
+    ):
+        raise ValueError("invalid legacy arrival page identity")
+    parts = urlsplit(LEGACY_SCHEDULE_ARRIVALS_LOCATOR)
+    return RequestFingerprint(
+        revision="ICN_LEGACY_ARRIVAL_V1",
+        normalized_request_sha256=_request_hash(
+            parts.hostname or "",
+            parts.path,
+            [
+                ("serviceKey", "<redacted>"),
+                ("numOfRows", str(SCHEDULE_PAGE_SIZE)),
+                ("pageNo", str(role_sequence)),
+                ("lang", "K"),
+                ("type", "json"),
+            ],
+        ),
+    )
+
+
+def _execute_aviation_attempt(
+    *,
+    source_code: str,
+    fingerprint: RequestFingerprint,
+    call_once: Callable[[], SingleAttemptResult],
+    attempt_executor: AviationAttemptExecutor,
+) -> SingleAttemptResult:
+    call_count = 0
+    actual_result: SingleAttemptResult | None = None
+
+    def guarded_call_once() -> SingleAttemptResult:
+        nonlocal call_count, actual_result
+        call_count += 1
+        if call_count != 1:
+            return _failed(fingerprint, FAILURE_TRANSPORT)
+        actual_result = call_once()
+        return actual_result
+
+    try:
+        result = attempt_executor(
+            source_code=source_code,
+            request_fingerprint=fingerprint,
+            call_once=guarded_call_once,
+        )
+    except Exception:
+        return _failed(fingerprint, FAILURE_TRANSPORT)
+    if (
+        call_count != 1
+        or result is not actual_result
+        or type(result) is not SingleAttemptResult
+        or result.request_fingerprint != fingerprint
+    ):
+        return _failed(fingerprint, FAILURE_TRANSPORT)
+    return result
+
+
+def _fetch_data_go_json_once(
+    *,
+    locator: str,
+    query: list[tuple[str, str]],
+    original_secret: str,
+    decoded_secret: str,
+    fingerprint: RequestFingerprint,
+    connect_timeout_seconds: int | float,
+    read_timeout_seconds: int | float,
+    connection_factory: ConnectionFactory,
+) -> SingleAttemptResult:
+    parts = urlsplit(locator)
+    wire_result = _read_once(
+        host=parts.hostname or "",
+        request_target=f"{parts.path}?{urlencode(query, doseq=False)}",
+        connect_timeout_seconds=connect_timeout_seconds,
+        read_timeout_seconds=read_timeout_seconds,
+        max_response_bytes=SCHEDULE_PAGE_MAX_RESPONSE_BYTES,
+        request_headers={
+            **_COMMON_REQUEST_HEADERS,
+            "Accept": "application/json",
+        },
+        connection_factory=connection_factory,
+    )
+    if isinstance(wire_result, _WireFailure):
+        if wire_result.http_status is not None:
+            http_failure = _classify_http_failure(
+                fingerprint,
+                wire_result.http_status,
+            )
+            if http_failure is not None:
+                return http_failure
+        return _failed(
+            fingerprint,
+            wire_result.failure_code,
+            http_status=(
+                wire_result.http_status
+                if wire_result.failure_code == FAILURE_RESPONSE_TOO_LARGE
+                else None
+            ),
+        )
+    if _contains_secret_reflection(
+        wire_result.body,
+        original_secret,
+        decoded_secret,
+    ):
+        return _failed(
+            fingerprint,
+            FAILURE_SECRET_REFLECTION,
+            http_status=wire_result.status,
+        )
+    result_code = _provider_result_code(wire_result.body)
+    provider_code = (
+        PROVIDER_DATA_GO_ERROR
+        if result_code is not None and result_code not in {"0", "00"}
+        else ""
+    )
+    http_failure = _classify_http_failure(
+        fingerprint,
+        wire_result.status,
+        provider_code=provider_code,
+    )
+    if http_failure is not None:
+        return http_failure
+    if provider_code:
+        return _failed(
+            fingerprint,
+            FAILURE_PROVIDER_ERROR,
+            http_status=wire_result.status,
+            provider_code=provider_code,
+        )
+    return _succeeded(
+        fingerprint,
+        http_status=wire_result.status,
+        body=wire_result.body,
+        provider_code=(
+            PROVIDER_DATA_GO_SUCCESS if result_code in {"0", "00"} else ""
+        ),
+    )
+
+
 def fetch_entry_attachment(
     *,
     official_locator: str,
@@ -654,65 +900,67 @@ def _schedule_page_count(body: bytes) -> int | None:
 def _fetch_schedule_direction(
     *,
     locator: str,
+    role: str,
     decoded_secret: str,
+    original_secret: str,
     season: str,
     connect_timeout_seconds: int | float,
     read_timeout_seconds: int | float,
     connection_factory: ConnectionFactory,
-) -> tuple[tuple[bytes, ...], str]:
-    parts = urlsplit(locator)
+    attempt_executor: AviationAttemptExecutor,
+) -> tuple[tuple[bytes, ...], tuple[AviationResponseLineage, ...], str]:
     pages: list[bytes] = []
+    lineage: list[AviationResponseLineage] = []
     expected_pages: int | None = None
     for page_number in range(1, SCHEDULE_MAX_PAGES_PER_DIRECTION + 1):
-        query = urlencode(
-            (
-                ("serviceKey", decoded_secret),
-                ("pageNo", str(page_number)),
-                ("numOfRows", str(SCHEDULE_PAGE_SIZE)),
-                ("season", season),
-                ("type", "json"),
+        query = [
+            ("serviceKey", decoded_secret),
+            ("pageNo", str(page_number)),
+            ("numOfRows", str(SCHEDULE_PAGE_SIZE)),
+            ("season", season),
+            ("type", "json"),
+        ]
+        fingerprint = schedule_page_request_fingerprint(
+            role=role,
+            role_sequence=page_number,
+            season=season,
+        )
+        result = _execute_aviation_attempt(
+            source_code=SCHEDULE_SOURCE_CODE,
+            fingerprint=fingerprint,
+            call_once=lambda: _fetch_data_go_json_once(
+                locator=locator,
+                query=query,
+                original_secret=original_secret,
+                decoded_secret=decoded_secret,
+                fingerprint=fingerprint,
+                connect_timeout_seconds=connect_timeout_seconds,
+                read_timeout_seconds=read_timeout_seconds,
+                connection_factory=connection_factory,
             ),
-            doseq=False,
+            attempt_executor=attempt_executor,
         )
-        wire_result = _read_once(
-            host=parts.hostname or "",
-            request_target=f"{parts.path}?{query}",
-            connect_timeout_seconds=connect_timeout_seconds,
-            read_timeout_seconds=read_timeout_seconds,
-            max_response_bytes=SCHEDULE_PAGE_MAX_RESPONSE_BYTES,
-            request_headers={**_COMMON_REQUEST_HEADERS, "Accept": "application/json"},
-            connection_factory=connection_factory,
+        lineage.append(
+            AviationResponseLineage(
+                source_code=SCHEDULE_SOURCE_CODE,
+                role=role,
+                role_sequence=page_number,
+                attempt=result,
+            )
         )
-        if isinstance(wire_result, _WireFailure):
-            return (), wire_result.failure_code
-        if wire_result.status != 200:
-            if wire_result.status in {401, 403}:
-                return (), FAILURE_AUTHENTICATION
-            if wire_result.status == 429:
-                return (), FAILURE_RATE_LIMITED
-            if 500 <= wire_result.status <= 599:
-                return (), FAILURE_UPSTREAM_5XX
-            return (), FAILURE_HTTP_CLIENT
-        if _contains_secret_reflection(
-            wire_result.body,
-            decoded_secret,
-            decoded_secret,
-        ):
-            return (), FAILURE_SECRET_REFLECTION
-        result_code = _provider_result_code(wire_result.body)
-        if result_code not in {"0", "00"}:
-            return (), FAILURE_PROVIDER_ERROR
-        page_count = _schedule_page_count(wire_result.body)
+        if not result.succeeded:
+            return tuple(pages), tuple(lineage), result.failure_code
+        pages.append(result.body)
+        page_count = _schedule_page_count(result.body)
         if page_count is None or page_count > SCHEDULE_MAX_PAGES_PER_DIRECTION:
-            return (), FAILURE_PROVIDER_ERROR
+            return tuple(pages), tuple(lineage), FAILURE_PROVIDER_ERROR
         if expected_pages is None:
             expected_pages = page_count
         elif expected_pages != page_count:
-            return (), FAILURE_PROVIDER_ERROR
-        pages.append(wire_result.body)
+            return tuple(pages), tuple(lineage), FAILURE_PROVIDER_ERROR
         if page_number == expected_pages:
-            return tuple(pages), ""
-    return (), FAILURE_RESPONSE_TOO_LARGE
+            return tuple(pages), tuple(lineage), ""
+    return tuple(pages), tuple(lineage), FAILURE_RESPONSE_TOO_LARGE
 
 
 def fetch_data_go_schedule_pages(
@@ -723,6 +971,7 @@ def fetch_data_go_schedule_pages(
     connect_timeout_seconds: int | float,
     read_timeout_seconds: int | float,
     connection_factory: ConnectionFactory = _default_connection_factory,
+    attempt_executor: AviationAttemptExecutor = _direct_attempt_executor,
 ) -> SchedulePageFetchResult:
     """Fetch complete official departure/arrival pages without persistence."""
 
@@ -741,49 +990,51 @@ def fetch_data_go_schedule_pages(
         decoded_secret = unquote(secret_value, encoding="utf-8", errors="strict")
     except (UnicodeError, TypeError, ValueError):
         return SchedulePageFetchResult(False, failure_code=FAILURE_AUTHENTICATION)
-    departures, failure = _fetch_schedule_direction(
+    departures, departure_lineage, failure = _fetch_schedule_direction(
         locator=SCHEDULE_DEPARTURES_LOCATOR,
+        role=ROLE_SCHEDULE_DEPARTURE,
         decoded_secret=decoded_secret,
+        original_secret=secret_value,
         season=season,
         connect_timeout_seconds=connect_timeout_seconds,
         read_timeout_seconds=read_timeout_seconds,
         connection_factory=connection_factory,
+        attempt_executor=attempt_executor,
     )
     if failure:
-        return SchedulePageFetchResult(False, failure_code=failure)
-    arrivals, failure = _fetch_schedule_direction(
+        return SchedulePageFetchResult(
+            False,
+            departure_pages=departures,
+            failure_code=failure,
+            lineage=departure_lineage,
+        )
+    arrivals, arrival_lineage, failure = _fetch_schedule_direction(
         locator=SCHEDULE_ARRIVALS_LOCATOR,
+        role=ROLE_SCHEDULE_ARRIVAL,
         decoded_secret=decoded_secret,
+        original_secret=secret_value,
         season=season,
         connect_timeout_seconds=connect_timeout_seconds,
         read_timeout_seconds=read_timeout_seconds,
         connection_factory=connection_factory,
+        attempt_executor=attempt_executor,
     )
     if failure:
-        return SchedulePageFetchResult(False, failure_code=failure)
+        return SchedulePageFetchResult(
+            False,
+            departure_pages=departures,
+            arrival_pages=arrivals,
+            failure_code=failure,
+            lineage=(*departure_lineage, *arrival_lineage),
+        )
     return SchedulePageFetchResult(
         True,
         departure_pages=departures,
         arrival_pages=arrivals,
+        lineage=(*departure_lineage, *arrival_lineage),
     )
 
 
-def _data_go_wire_failure_code(
-    wire_result: _WireResponse | _WireFailure,
-) -> str:
-    if isinstance(wire_result, _WireFailure):
-        return wire_result.failure_code
-    if wire_result.status in {401, 403}:
-        return FAILURE_AUTHENTICATION
-    if wire_result.status == 429:
-        return FAILURE_RATE_LIMITED
-    if 500 <= wire_result.status <= 599:
-        return FAILURE_UPSTREAM_5XX
-    if wire_result.status != 200:
-        return FAILURE_HTTP_CLIENT
-    return ""
-
-
 def _fetch_destination_city_reference(
     *,
     original_secret: str,
@@ -791,38 +1042,33 @@ def _fetch_destination_city_reference(
     connect_timeout_seconds: int | float,
     read_timeout_seconds: int | float,
     connection_factory: ConnectionFactory,
-) -> tuple[bytes, str]:
-    parts = urlsplit(CITY_SOURCE_LOCATOR)
-    query = urlencode(
-        (("serviceKey", decoded_secret), ("type", "json")),
-        doseq=False,
+    attempt_executor: AviationAttemptExecutor,
+) -> tuple[bytes, AviationResponseLineage, str]:
+    fingerprint = destination_city_request_fingerprint()
+    result = _execute_aviation_attempt(
+        source_code=CITY_SOURCE_CODE,
+        fingerprint=fingerprint,
+        call_once=lambda: _fetch_data_go_json_once(
+            locator=CITY_SOURCE_LOCATOR,
+            query=[("serviceKey", decoded_secret), ("type", "json")],
+            original_secret=original_secret,
+            decoded_secret=decoded_secret,
+            fingerprint=fingerprint,
+            connect_timeout_seconds=connect_timeout_seconds,
+            read_timeout_seconds=read_timeout_seconds,
+            connection_factory=connection_factory,
+        ),
+        attempt_executor=attempt_executor,
     )
-    wire_result = _read_once(
-        host=parts.hostname or "",
-        request_target=f"{parts.path}?{query}",
-        connect_timeout_seconds=connect_timeout_seconds,
-        read_timeout_seconds=read_timeout_seconds,
-        max_response_bytes=SCHEDULE_PAGE_MAX_RESPONSE_BYTES,
-        request_headers={
-            **_COMMON_REQUEST_HEADERS,
-            "Accept": "application/json",
-        },
-        connection_factory=connection_factory,
+    lineage = AviationResponseLineage(
+        source_code=CITY_SOURCE_CODE,
+        role=ROLE_DESTINATION_CITY,
+        role_sequence=1,
+        attempt=result,
     )
-    failure = _data_go_wire_failure_code(wire_result)
-    if failure:
-        return b"", failure
-    if not isinstance(wire_result, _WireResponse):
-        return b"", FAILURE_TRANSPORT
-    if _contains_secret_reflection(
-        wire_result.body,
-        original_secret,
-        decoded_secret,
-    ):
-        return b"", FAILURE_SECRET_REFLECTION
-    if _provider_result_code(wire_result.body) not in {"0", "00"}:
-        return b"", FAILURE_PROVIDER_ERROR
-    return wire_result.body, ""
+    if not result.succeeded:
+        return b"", lineage, result.failure_code
+    return result.body, lineage, ""
 
 
 def _fetch_legacy_arrival_pages(
@@ -832,60 +1078,59 @@ def _fetch_legacy_arrival_pages(
     connect_timeout_seconds: int | float,
     read_timeout_seconds: int | float,
     connection_factory: ConnectionFactory,
-) -> tuple[tuple[bytes, ...], str]:
-    parts = urlsplit(LEGACY_SCHEDULE_ARRIVALS_LOCATOR)
+    attempt_executor: AviationAttemptExecutor,
+) -> tuple[tuple[bytes, ...], tuple[AviationResponseLineage, ...], str]:
     pages: list[bytes] = []
+    lineage: list[AviationResponseLineage] = []
     expected_pages: int | None = None
     for page_number in range(1, SCHEDULE_MAX_PAGES_PER_DIRECTION + 1):
-        query = urlencode(
-            (
-                ("serviceKey", decoded_secret),
-                ("numOfRows", str(SCHEDULE_PAGE_SIZE)),
-                ("pageNo", str(page_number)),
-                ("lang", "K"),
-                ("type", "json"),
+        query = [
+            ("serviceKey", decoded_secret),
+            ("numOfRows", str(SCHEDULE_PAGE_SIZE)),
+            ("pageNo", str(page_number)),
+            ("lang", "K"),
+            ("type", "json"),
+        ]
+        fingerprint = legacy_arrival_request_fingerprint(page_number)
+        result = _execute_aviation_attempt(
+            source_code=LEGACY_SCHEDULE_SOURCE_CODE,
+            fingerprint=fingerprint,
+            call_once=lambda: _fetch_data_go_json_once(
+                locator=LEGACY_SCHEDULE_ARRIVALS_LOCATOR,
+                query=query,
+                original_secret=original_secret,
+                decoded_secret=decoded_secret,
+                fingerprint=fingerprint,
+                connect_timeout_seconds=connect_timeout_seconds,
+                read_timeout_seconds=read_timeout_seconds,
+                connection_factory=connection_factory,
             ),
-            doseq=False,
+            attempt_executor=attempt_executor,
         )
-        wire_result = _read_once(
-            host=parts.hostname or "",
-            request_target=f"{parts.path}?{query}",
-            connect_timeout_seconds=connect_timeout_seconds,
-            read_timeout_seconds=read_timeout_seconds,
-            max_response_bytes=SCHEDULE_PAGE_MAX_RESPONSE_BYTES,
-            request_headers={
-                **_COMMON_REQUEST_HEADERS,
-                "Accept": "application/json",
-            },
-            connection_factory=connection_factory,
+        lineage.append(
+            AviationResponseLineage(
+                source_code=LEGACY_SCHEDULE_SOURCE_CODE,
+                role=ROLE_LEGACY_ARRIVAL,
+                role_sequence=page_number,
+                attempt=result,
+            )
         )
-        failure = _data_go_wire_failure_code(wire_result)
-        if failure:
-            return (), failure
-        if not isinstance(wire_result, _WireResponse):
-            return (), FAILURE_TRANSPORT
-        if _contains_secret_reflection(
-            wire_result.body,
-            original_secret,
-            decoded_secret,
-        ):
-            return (), FAILURE_SECRET_REFLECTION
-        if _provider_result_code(wire_result.body) not in {"0", "00"}:
-            return (), FAILURE_PROVIDER_ERROR
-        page_count = _schedule_page_count(wire_result.body)
+        if not result.succeeded:
+            return tuple(pages), tuple(lineage), result.failure_code
+        pages.append(result.body)
+        page_count = _schedule_page_count(result.body)
         if (
             page_count is None
             or page_count > SCHEDULE_MAX_PAGES_PER_DIRECTION
         ):
-            return (), FAILURE_PROVIDER_ERROR
+            return tuple(pages), tuple(lineage), FAILURE_PROVIDER_ERROR
         if expected_pages is None:
             expected_pages = page_count
         elif expected_pages != page_count:
-            return (), FAILURE_PROVIDER_ERROR
-        pages.append(wire_result.body)
+            return tuple(pages), tuple(lineage), FAILURE_PROVIDER_ERROR
         if page_number == expected_pages:
-            return tuple(pages), ""
-    return (), FAILURE_RESPONSE_TOO_LARGE
+            return tuple(pages), tuple(lineage), ""
+    return tuple(pages), tuple(lineage), FAILURE_RESPONSE_TOO_LARGE
 
 
 def fetch_data_go_reference_evidence(
@@ -895,6 +1140,7 @@ def fetch_data_go_reference_evidence(
     connect_timeout_seconds: int | float,
     read_timeout_seconds: int | float,
     connection_factory: ConnectionFactory = _default_connection_factory,
+    attempt_executor: AviationAttemptExecutor = _direct_attempt_executor,
 ) -> AviationReferenceFetchResult:
     """Fetch 15095067 cities and complete 15095059 arrival pages in memory."""
 
@@ -921,26 +1167,40 @@ def fetch_data_go_reference_evidence(
             False,
             failure_code=FAILURE_AUTHENTICATION,
         )
-    city_payload, failure = _fetch_destination_city_reference(
+    city_payload, city_lineage, failure = _fetch_destination_city_reference(
         original_secret=secret_value,
         decoded_secret=decoded_secret,
         connect_timeout_seconds=connect_timeout_seconds,
         read_timeout_seconds=read_timeout_seconds,
         connection_factory=connection_factory,
+        attempt_executor=attempt_executor,
     )
     if failure:
-        return AviationReferenceFetchResult(False, failure_code=failure)
-    legacy_pages, failure = _fetch_legacy_arrival_pages(
+        return AviationReferenceFetchResult(
+            False,
+            city_payload=city_payload,
+            failure_code=failure,
+            lineage=(city_lineage,),
+        )
+    legacy_pages, legacy_lineage, failure = _fetch_legacy_arrival_pages(
         original_secret=secret_value,
         decoded_secret=decoded_secret,
         connect_timeout_seconds=connect_timeout_seconds,
         read_timeout_seconds=read_timeout_seconds,
         connection_factory=connection_factory,
+        attempt_executor=attempt_executor,
     )
     if failure:
-        return AviationReferenceFetchResult(False, failure_code=failure)
+        return AviationReferenceFetchResult(
+            False,
+            city_payload=city_payload,
+            legacy_arrival_pages=legacy_pages,
+            failure_code=failure,
+            lineage=(city_lineage, *legacy_lineage),
+        )
     return AviationReferenceFetchResult(
         True,
         city_payload=city_payload,
         legacy_arrival_pages=legacy_pages,
+        lineage=(city_lineage, *legacy_lineage),
     )
