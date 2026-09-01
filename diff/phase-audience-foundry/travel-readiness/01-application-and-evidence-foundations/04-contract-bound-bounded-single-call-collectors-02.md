## `feat(sources): add bounded approved transports`

diff --git a/sources/tests/test_transport.py b/sources/tests/test_transport.py
new file mode 100644
index 0000000..824bd59
--- /dev/null
+++ b/sources/tests/test_transport.py
@@ -0,0 +1,561 @@
+import hashlib
+import json
+import unittest
+from urllib.parse import parse_qs, quote, urlsplit
+
+from sources.transport import (
+    ATTEMPT_RETRYABLE_FAILED,
+    ATTEMPT_SUCCEEDED,
+    ATTEMPT_TERMINAL_FAILED,
+    ENTRY_MAX_RESPONSE_BYTES,
+    ENTRY_REQUEST_FINGERPRINT,
+    ENTRY_SOURCE_LOCATOR,
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
+    fetch_entry_attachment,
+    fetch_travel_alarm_jp,
+)
+
+
+class FakeSocket:
+    def __init__(self):
+        self.timeouts = []
+
+    def settimeout(self, timeout):
+        self.timeouts.append(timeout)
+
+
+class FakeResponse:
+    def __init__(self, status, body, *, content_length=None, read_error=None):
+        self.status = status
+        self._body = body
+        self._content_length = content_length
+        self._read_error = read_error
+        self.read_amounts = []
+
+    def getheader(self, name, default=None):
+        if name == "Content-Length":
+            return self._content_length
+        return default
+
+    def read(self, amount=None):
+        self.read_amounts.append(amount)
+        if self._read_error is not None:
+            raise self._read_error
+        if amount is None:
+            return self._body
+        return self._body[:amount]
+
+
+class FakeConnection:
+    def __init__(self, response, *, connect_error=None, request_error=None):
+        self.response = response
+        self.connect_error = connect_error
+        self.request_error = request_error
+        self.sock = FakeSocket()
+        self.requests = []
+        self.connected = False
+        self.closed = False
+        self.debug_levels = []
+
+    def set_debuglevel(self, level):
+        self.debug_levels.append(level)
+
+    def connect(self):
+        if self.connect_error is not None:
+            raise self.connect_error
+        self.connected = True
+
+    def request(self, method, url, body=None, headers=None):
+        if self.request_error is not None:
+            raise self.request_error
+        self.requests.append((method, url, body, headers))
+
+    def getresponse(self):
+        return self.response
+
+    def close(self):
+        self.closed = True
+
+
+class FakeFactory:
+    def __init__(self, connection):
+        self.connection = connection
+        self.calls = []
+
+    def __call__(self, host, port, timeout):
+        self.calls.append((host, port, timeout))
+        return self.connection
+
+
+def warning_body(result_code="0"):
+    return json.dumps(
+        {
+            "response": {
+                "header": {
+                    "resultCode": result_code,
+                    "resultMsg": "normal",
+                },
+                "body": {
+                    "dataType": "JSON",
+                    "items": {"item": []},
+                    "numOfRows": 1,
+                    "pageNo": 1,
+                    "totalCount": 0,
+                },
+            }
+        },
+        ensure_ascii=False,
+        separators=(",", ":"),
+    ).encode("utf-8")
+
+
+class TransportTestCase(unittest.TestCase):
+    def entry_fetch(self, connection):
+        return fetch_entry_attachment(
+            official_locator=ENTRY_SOURCE_LOCATOR,
+            connect_timeout_seconds=4,
+            read_timeout_seconds=9,
+            connection_factory=FakeFactory(connection),
+        )
+
+    def warning_fetch(self, connection, *, secret=None, factory=None):
+        if secret is None:
+            secret = "%2B".join(("Q7vR4mP2", "zN8kL6xT")) + "%3D"
+        return fetch_travel_alarm_jp(
+            official_locator=WARNING_SOURCE_LOCATOR,
+            secret_reference_name=WARNING_SECRET_REFERENCE,
+            secret_value=secret,
+            connect_timeout_seconds=3,
+            read_timeout_seconds=8,
+            connection_factory=factory or FakeFactory(connection),
+        )
+
+    def assert_sensitive_absent(self, result, *values):
+        rendered = repr(result)
+        if any(value in rendered for value in values):
+            self.fail("SENSITIVE_VALUE_EXPOSED")
+
+    def test_fixed_fingerprints_are_stable_and_redacted(self):
+        self.assertEqual(ENTRY_REQUEST_FINGERPRINT.revision, "MOFA_ENTRY_V1")
+        self.assertEqual(
+            ENTRY_REQUEST_FINGERPRINT.normalized_request_sha256,
+            "8fe7d01a5e67549a41f27068b6dd4f001e091a6ed781d73444466172e8ca0b00",
+        )
+        self.assertEqual(WARNING_REQUEST_FINGERPRINT.revision, "MOFA_WARNING_V1")
+        self.assertEqual(
+            WARNING_REQUEST_FINGERPRINT.normalized_request_sha256,
+            "e9ed083968ebdcd1341df0fbb0da3014ce62f61ba8114a558446b30cfe21f913",
+        )
+
+    def test_entry_success_is_exact_bounded_and_body_is_repr_redacted(self):
+        raw_marker = b"synthetic-entry-response"
+        response = FakeResponse(200, raw_marker)
+        connection = FakeConnection(response)
+        factory = FakeFactory(connection)
+
+        result = fetch_entry_attachment(
+            official_locator=ENTRY_SOURCE_LOCATOR,
+            connect_timeout_seconds=4,
+            read_timeout_seconds=9,
+            connection_factory=factory,
+        )
+
+        self.assertEqual(result.attempt_result, ATTEMPT_SUCCEEDED)
+        self.assertEqual(result.body, raw_marker)
+        self.assertEqual(result.response_sha256, hashlib.sha256(raw_marker).hexdigest())
+        self.assertEqual(result.byte_count, len(raw_marker))
+        self.assertEqual(factory.calls, [("www.data.go.kr", 443, 4.0)])
+        self.assertEqual(connection.debug_levels, [0])
+        self.assertEqual(connection.sock.timeouts, [9.0])
+        self.assertEqual(response.read_amounts, [ENTRY_MAX_RESPONSE_BYTES + 1])
+        method, target, request_body, headers = connection.requests[0]
+        self.assertEqual(method, "GET")
+        self.assertEqual(
+            target,
+            "/cmm/cmm/fileDownload.do?"
+            "atchFileId=FILE_000000003090472&fileDetailSn=1&insertDataPrcus=N",
+        )
+        self.assertIsNone(request_body)
+        self.assertEqual(
+            headers,
+            {
+                "Accept": "text/csv, application/octet-stream",
+                "Connection": "close",
+                "User-Agent": "travel-readiness-source-fetch/1",
+            },
+        )
+        self.assertNotIn("Authorization", headers)
+        self.assertNotIn("Cookie", headers)
+        self.assertTrue(connection.closed)
+        self.assert_sensitive_absent(result, raw_marker.decode("ascii"))
+
+    def test_entry_does_not_follow_redirect_and_rejects_other_locator(self):
+        connection = FakeConnection(FakeResponse(302, b"synthetic redirect"))
+        result = self.entry_fetch(connection)
+        self.assertEqual(result.attempt_result, ATTEMPT_TERMINAL_FAILED)
+        self.assertEqual(result.failure_code, FAILURE_HTTP_CLIENT)
+        self.assertEqual(result.http_status, 302)
+        self.assertEqual(len(connection.requests), 1)
+        self.assertEqual(result.body, b"")
+
+        for status in (204, 206):
+            with self.subTest(status=status):
+                incomplete = self.entry_fetch(
+                    FakeConnection(FakeResponse(status, b"synthetic partial"))
+                )
+                self.assertEqual(incomplete.attempt_result, ATTEMPT_TERMINAL_FAILED)
+                self.assertEqual(incomplete.failure_code, FAILURE_HTTP_CLIENT)
+                self.assertEqual(incomplete.http_status, status)
+                self.assertEqual(incomplete.body, b"")
+
+        unused = FakeConnection(FakeResponse(200, b"unused"))
+        rejected = fetch_entry_attachment(
+            official_locator="https://example.invalid/other",
+            connect_timeout_seconds=4,
+            read_timeout_seconds=9,
+            connection_factory=FakeFactory(unused),
+        )
+        self.assertEqual(rejected.failure_code, FAILURE_HTTP_CLIENT)
+        self.assertFalse(unused.connected)
+
+    def test_entry_timeout_transport_http_and_oversize_are_closed(self):
+        timeout = self.entry_fetch(
+            FakeConnection(FakeResponse(200, b"unused"), connect_error=TimeoutError())
+        )
+        self.assertEqual(timeout.attempt_result, ATTEMPT_RETRYABLE_FAILED)
+        self.assertEqual(timeout.failure_code, FAILURE_TIMEOUT)
+        self.assertIsNone(timeout.http_status)
+
+        transport = self.entry_fetch(
+            FakeConnection(FakeResponse(200, b"unused"), request_error=OSError())
+        )
+        self.assertEqual(transport.attempt_result, ATTEMPT_TERMINAL_FAILED)
+        self.assertEqual(transport.failure_code, FAILURE_TRANSPORT)
+
+        server_error = self.entry_fetch(FakeConnection(FakeResponse(503, b"busy")))
+        self.assertEqual(server_error.attempt_result, ATTEMPT_RETRYABLE_FAILED)
+        self.assertEqual(server_error.failure_code, FAILURE_UPSTREAM_5XX)
+        self.assertEqual(server_error.http_status, 503)
+
+        declared = self.entry_fetch(
+            FakeConnection(
+                FakeResponse(
+                    200,
+                    b"not-read",
+                    content_length=str(ENTRY_MAX_RESPONSE_BYTES + 1),
+                )
+            )
+        )
+        self.assertEqual(declared.failure_code, FAILURE_RESPONSE_TOO_LARGE)
+
+        actual = self.entry_fetch(
+            FakeConnection(FakeResponse(200, b"x" * (ENTRY_MAX_RESPONSE_BYTES + 1)))
+        )
+        self.assertEqual(actual.failure_code, FAILURE_RESPONSE_TOO_LARGE)
+        self.assertEqual(actual.body, b"")
+
+    def test_warning_unquotes_once_then_query_encoder_encodes_once(self):
+        secret = "%2B".join(("A9cD7eF5", "G3hJ1kL8")) + "%2FM6nP4qR2%3D"
+        response_body = warning_body()
+        response = FakeResponse(200, response_body)
+        connection = FakeConnection(response)
+        factory = FakeFactory(connection)
+
+        result = self.warning_fetch(connection, secret=secret, factory=factory)
+
+        self.assertEqual(result.attempt_result, ATTEMPT_SUCCEEDED)
+        self.assertEqual(result.provider_code, PROVIDER_SUCCESS_0)
+        self.assertEqual(response_body, result.body)
+        self.assertEqual(factory.calls, [("apis.data.go.kr", 443, 3.0)])
+        self.assertEqual(connection.debug_levels, [0])
+        self.assertEqual(connection.sock.timeouts, [8.0])
+        self.assertEqual(response.read_amounts, [WARNING_MAX_RESPONSE_BYTES + 1])
+
+        method, target, request_body, headers = connection.requests[0]
+        parsed = urlsplit(target)
+        parameters = parse_qs(parsed.query, keep_blank_values=True)
+        expected_decoded = "A9cD7eF5+G3hJ1kL8/M6nP4qR2="
+        request_is_exact = (
+            method == "GET"
+            and parsed.path
+            == "/1262000/TravelAlarmService2/getTravelAlarmList2"
+            and request_body is None
+            and parameters.get("ServiceKey") == [expected_decoded]
+            and parameters.get("returnType") == ["JSON"]
+            and parameters.get("numOfRows") == ["1"]
+            and parameters.get("pageNo") == ["1"]
+            and parameters.get("cond[country_iso_alp2::EQ]") == ["JP"]
+            and len(parameters) == 5
+            and "%252B" not in parsed.query
+        )
+        if not request_is_exact:
+            self.fail("WARNING_REQUEST_SHAPE_MISMATCH")
+        self.assertEqual(
+            headers,
+            {
+                "Accept": "application/json",
+                "Connection": "close",
+                "User-Agent": "travel-readiness-source-fetch/1",
+            },
+        )
+        self.assertNotIn("Authorization", headers)
+        self.assertNotIn("Cookie", headers)
+        self.assert_sensitive_absent(
+            result, secret, expected_decoded, response_body.decode("utf-8")
+        )
+
+    def test_warning_accepts_a_decoded_key_without_transforming_its_value(self):
+        secret = "D8mQ4zN7+P2vK6rT/J9wX3cL="
+        connection = FakeConnection(FakeResponse(200, warning_body()))
+
+        result = self.warning_fetch(connection, secret=secret)
+
+        self.assertEqual(result.attempt_result, ATTEMPT_SUCCEEDED)
+        target = connection.requests[0][1]
+        parameters = parse_qs(urlsplit(target).query, keep_blank_values=True)
+        self.assertEqual(parameters.get("ServiceKey"), [secret])
+        self.assertNotIn("%252B", urlsplit(target).query)
+        self.assert_sensitive_absent(result, secret)
+
+    def test_warning_rejects_redirect_auth_rate_limit_and_server_error(self):
+        cases = (
+            (204, ATTEMPT_TERMINAL_FAILED, FAILURE_HTTP_CLIENT),
+            (206, ATTEMPT_TERMINAL_FAILED, FAILURE_HTTP_CLIENT),
+            (302, ATTEMPT_TERMINAL_FAILED, FAILURE_HTTP_CLIENT),
+            (401, ATTEMPT_TERMINAL_FAILED, FAILURE_AUTHENTICATION),
+            (403, ATTEMPT_TERMINAL_FAILED, FAILURE_AUTHENTICATION),
+            (429, ATTEMPT_RETRYABLE_FAILED, FAILURE_RATE_LIMITED),
+            (500, ATTEMPT_RETRYABLE_FAILED, FAILURE_UPSTREAM_5XX),
+        )
+        for status, expected_result, expected_failure in cases:
+            with self.subTest(status=status):
+                connection = FakeConnection(FakeResponse(status, b"synthetic"))
+                result = self.warning_fetch(connection)
+                self.assertEqual(result.attempt_result, expected_result)
+                self.assertEqual(result.failure_code, expected_failure)
+                self.assertEqual(result.http_status, status)
+                self.assertEqual(len(connection.requests), 1)
+                self.assertEqual(result.body, b"")
+
+    def test_warning_classifies_json_and_gateway_provider_envelopes(self):
+        gateway = self.warning_fetch(
+            FakeConnection(FakeResponse(200, warning_body("20")))
+        )
+        self.assertEqual(gateway.attempt_result, ATTEMPT_TERMINAL_FAILED)
+        self.assertEqual(gateway.failure_code, FAILURE_PROVIDER_ERROR)
+        self.assertEqual(gateway.provider_code, PROVIDER_GATEWAY_20)
+        self.assertEqual(gateway.body, b"")
+
+        other = self.warning_fetch(
+            FakeConnection(FakeResponse(200, warning_body("99")))
+        )
+        self.assertEqual(other.failure_code, FAILURE_PROVIDER_ERROR)
+        self.assertEqual(other.provider_code, PROVIDER_OTHER_ERROR)
+
+        xml_envelope = (
+            b"<OpenAPI_ServiceResponse><cmmMsgHeader>"
+            b"<returnReasonCode>20</returnReasonCode>"
+            b"</cmmMsgHeader></OpenAPI_ServiceResponse>"
+        )
+        http_auth = self.warning_fetch(
+            FakeConnection(FakeResponse(401, xml_envelope))
+        )
+        self.assertEqual(http_auth.failure_code, FAILURE_AUTHENTICATION)
+        self.assertEqual(http_auth.provider_code, PROVIDER_GATEWAY_20)
+
+    def test_warning_detects_reflection_before_returning_any_body(self):
+        secret = "%2F".join(("R8fL2qW6", "T4nK9vC3")) + "%3D"
+        decoded = "R8fL2qW6/T4nK9vC3="
+        reflected_body = b'{"error":"' + decoded.encode("ascii") + b'"}'
+        result = self.warning_fetch(
+            FakeConnection(FakeResponse(401, reflected_body)), secret=secret
+        )
+        self.assertEqual(result.attempt_result, ATTEMPT_TERMINAL_FAILED)
+        self.assertEqual(result.failure_code, FAILURE_SECRET_REFLECTION)
+        self.assertEqual(result.provider_code, "")
+        self.assertEqual(result.body, b"")
+        self.assertEqual(result.response_sha256, "")
+        self.assert_sensitive_absent(result, secret, decoded, reflected_body.decode())
+
+    def test_warning_detects_alternate_and_partial_secret_reflections(self):
+        secret = "R8fL2qW6%2FT4nK9vC3%2BZ7mP5dH1%3D"
+        decoded = "R8fL2qW6/T4nK9vC3+Z7mP5dH1="
+        reflected_values = (
+            secret.replace("%2F", "%2f").replace("%2B", "%2b"),
+            quote(secret, safe=""),
+            decoded.replace("/", r"\/"),
+            decoded[9:17],
+        )
+        for index, reflected in enumerate(reflected_values):
+            with self.subTest(case=index):
+                body = b'{"error":"' + reflected.encode("ascii") + b'"}'
+                result = self.warning_fetch(
+                    FakeConnection(FakeResponse(401, body)), secret=secret
+                )
+                self.assertEqual(result.failure_code, FAILURE_SECRET_REFLECTION)
+                self.assertEqual(result.body, b"")
+                self.assert_sensitive_absent(result, secret, decoded, reflected)
+
+    def test_warning_scans_the_entire_accepted_key_for_partial_reflection(self):
+        secret = "A" * 500 + "Z9y8X7w6Q5p4"
+        reflected_suffix = secret[-12:-4]
+        body = b'{"error":"' + reflected_suffix.encode("ascii") + b'"}'
+
+        result = self.warning_fetch(
+            FakeConnection(FakeResponse(200, body)),
+            secret=secret,
+        )
+
+        self.assertEqual(result.failure_code, FAILURE_SECRET_REFLECTION)
+        self.assertEqual(result.body, b"")
+        self.assert_sensitive_absent(result, secret, reflected_suffix)
+
+    def test_warning_timeout_oversize_and_raw_exception_are_redacted(self):
+        timeout = self.warning_fetch(
+            FakeConnection(
+                FakeResponse(200, b"unused", read_error=TimeoutError())
+            )
+        )
+        self.assertEqual(timeout.attempt_result, ATTEMPT_RETRYABLE_FAILED)
+        self.assertEqual(timeout.failure_code, FAILURE_TIMEOUT)
+        self.assertIsNone(timeout.http_status)
+
+        oversize = self.warning_fetch(
+            FakeConnection(FakeResponse(200, b"x" * (WARNING_MAX_RESPONSE_BYTES + 1)))
+        )
+        self.assertEqual(oversize.failure_code, FAILURE_RESPONSE_TOO_LARGE)
+        self.assertEqual(oversize.body, b"")
+
+        for status, expected_failure in (
+            (429, FAILURE_RATE_LIMITED),
+            (503, FAILURE_UPSTREAM_5XX),
+        ):
+            with self.subTest(status=status, size="declared"):
+                status_result = self.warning_fetch(
+                    FakeConnection(
+                        FakeResponse(
+                            status,
+                            b"not-read",
+                            content_length=str(WARNING_MAX_RESPONSE_BYTES + 1),
+                        )
+                    )
+                )
+                self.assertEqual(status_result.failure_code, expected_failure)
+                self.assertEqual(status_result.http_status, status)
+                self.assertEqual(status_result.body, b"")
+            with self.subTest(status=status, size="actual"):
+                status_result = self.warning_fetch(
+                    FakeConnection(
+                        FakeResponse(
+                            status,
+                            b"x" * (WARNING_MAX_RESPONSE_BYTES + 1),
+                        )
+                    )
+                )
+                self.assertEqual(status_result.failure_code, expected_failure)
+                self.assertEqual(status_result.http_status, status)
+                self.assertEqual(status_result.body, b"")
+            with self.subTest(status=status, size="read-error"):
+                status_result = self.warning_fetch(
+                    FakeConnection(
+                        FakeResponse(
+                            status,
+                            b"unused",
+                            read_error=OSError("synthetic read interruption"),
+                        )
+                    )
+                )
+                self.assertEqual(status_result.failure_code, expected_failure)
+                self.assertEqual(status_result.http_status, status)
+                self.assertEqual(status_result.body, b"")
+
+        secret = "%2B".join(("X6qM8rD4", "V2kN7pL5"))
+        raw_exception_marker = "raw-upstream-detail"
+        connection = FakeConnection(
+            FakeResponse(200, b"unused"),
+            request_error=RuntimeError(raw_exception_marker + secret),
+        )
+        result = self.warning_fetch(connection, secret=secret)
+        self.assertEqual(result.failure_code, FAILURE_TRANSPORT)
+        self.assert_sensitive_absent(result, secret, raw_exception_marker)
+
+    def test_warning_requires_exact_locator_reference_secret_and_timeouts(self):
+        invalid_calls = (
+            (
+                FAILURE_HTTP_CLIENT,
+                {
+                    "official_locator": "https://example.invalid/other",
+                    "secret_reference_name": WARNING_SECRET_REFERENCE,
+                    "secret_value": "synthetic",
+                    "connect_timeout_seconds": 3,
+                    "read_timeout_seconds": 8,
+                },
+            ),
+            (
+                FAILURE_AUTHENTICATION,
+                {
+                    "official_locator": WARNING_SOURCE_LOCATOR,
+                    "secret_reference_name": "OTHER_REFERENCE",
+                    "secret_value": "synthetic",
+                    "connect_timeout_seconds": 3,
+                    "read_timeout_seconds": 8,
+                },
+            ),
+            (
+                FAILURE_AUTHENTICATION,
+                {
+                    "official_locator": WARNING_SOURCE_LOCATOR,
+                    "secret_reference_name": WARNING_SECRET_REFERENCE,
+                    "secret_value": "",
+                    "connect_timeout_seconds": 3,
+                    "read_timeout_seconds": 8,
+                },
+            ),
+            (
+                FAILURE_AUTHENTICATION,
+                {
+                    "official_locator": WARNING_SOURCE_LOCATOR,
+                    "secret_reference_name": WARNING_SECRET_REFERENCE,
+                    "secret_value": "x" * 513,
+                    "connect_timeout_seconds": 3,
+                    "read_timeout_seconds": 8,
+                },
+            ),
+            (
+                FAILURE_HTTP_CLIENT,
+                {
+                    "official_locator": WARNING_SOURCE_LOCATOR,
+                    "secret_reference_name": WARNING_SECRET_REFERENCE,
+                    "secret_value": "synthetic",
+                    "connect_timeout_seconds": 0,
+                    "read_timeout_seconds": 8,
+                },
+            ),
+        )
+        for expected_failure, values in invalid_calls:
+            with self.subTest(reference=values["secret_reference_name"]):
+                connection = FakeConnection(FakeResponse(200, warning_body()))
+                result = fetch_travel_alarm_jp(
+                    **values, connection_factory=FakeFactory(connection)
+                )
+                self.assertEqual(result.failure_code, expected_failure)
+                self.assertFalse(connection.connected)
+
+
+if __name__ == "__main__":
+    unittest.main()
diff --git a/sources/transport.py b/sources/transport.py
new file mode 100644
index 0000000..643d713
--- /dev/null
+++ b/sources/transport.py
@@ -0,0 +1,565 @@
+"""Bounded, single-attempt transports for the two approved source paths.
+
+This module deliberately has no persistence or retry behavior.  Callers create
+and close durable attempt receipts around one invocation and decide whether a
+retry is allowed.  Successful response bytes exist only in the returned
+in-memory value; failure results never carry response bytes.
+"""
+
+from __future__ import annotations
+
+import hashlib
+import http.client
+import json
+import re
+import socket
+from dataclasses import dataclass, field
+from typing import Callable, Protocol
+from urllib.parse import parse_qsl, quote, quote_plus, unquote, urlencode, urlsplit
+from xml.etree import ElementTree
+
+
+ENTRY_SOURCE_LOCATOR = (
+    "https://www.data.go.kr/cmm/cmm/fileDownload.do?"
+    "atchFileId=FILE_000000003090472&fileDetailSn=1&insertDataPrcus=N"
+)
+WARNING_SOURCE_LOCATOR = (
+    "https://apis.data.go.kr/1262000/TravelAlarmService2/"
+    "getTravelAlarmList2"
+)
+WARNING_SECRET_REFERENCE = "MOFA_TRAVEL_ALARM_SERVICE_KEY"
+
+ENTRY_MAX_RESPONSE_BYTES = 262_144
+WARNING_MAX_RESPONSE_BYTES = 4_096
+
+ATTEMPT_SUCCEEDED = "SUCCEEDED"
+ATTEMPT_RETRYABLE_FAILED = "RETRYABLE_FAILED"
+ATTEMPT_TERMINAL_FAILED = "TERMINAL_FAILED"
+
+FAILURE_TIMEOUT = "TIMEOUT"
+FAILURE_RATE_LIMITED = "RATE_LIMITED"
+FAILURE_UPSTREAM_5XX = "UPSTREAM_5XX"
+FAILURE_AUTHENTICATION = "AUTHENTICATION"
+FAILURE_PROVIDER_ERROR = "PROVIDER_ERROR"
+FAILURE_SECRET_REFLECTION = "SECRET_REFLECTION"
+FAILURE_HTTP_CLIENT = "HTTP_CLIENT"
+FAILURE_TRANSPORT = "TRANSPORT"
+FAILURE_RESPONSE_TOO_LARGE = "RESPONSE_TOO_LARGE"
+
+PROVIDER_SUCCESS_0 = "MOFA_SUCCESS_0"
+PROVIDER_GATEWAY_20 = "MOFA_GATEWAY_20"
+PROVIDER_OTHER_ERROR = "MOFA_OTHER_ERROR"
+
+
+@dataclass(frozen=True, slots=True)
+class RequestFingerprint:
+    revision: str
+    normalized_request_sha256: str
+
+
+@dataclass(frozen=True, slots=True)
+class SingleAttemptResult:
+    """A redacted result shaped for a durable ``FetchAttempt`` transition."""
+
+    request_fingerprint: RequestFingerprint
+    attempt_result: str
+    http_status: int | None = None
+    provider_code: str = ""
+    response_sha256: str = ""
+    byte_count: int = 0
+    failure_code: str = ""
+    body: bytes = field(default=b"", repr=False, compare=False)
+
+    @property
+    def succeeded(self) -> bool:
+        return self.attempt_result == ATTEMPT_SUCCEEDED
+
+
+class _SocketLike(Protocol):
+    def settimeout(self, timeout: float) -> None: ...
+
+
+class _ResponseLike(Protocol):
+    status: int
+
+    def getheader(self, name: str, default: str | None = None) -> str | None: ...
+
+    def read(self, amount: int | None = None) -> bytes: ...
+
+
+class _ConnectionLike(Protocol):
+    sock: _SocketLike | None
+
+    def connect(self) -> None: ...
+
+    def request(
+        self,
+        method: str,
+        url: str,
+        body: bytes | None = None,
+        headers: dict[str, str] | None = None,
+    ) -> None: ...
+
+    def getresponse(self) -> _ResponseLike: ...
+
+    def close(self) -> None: ...
+
+
+ConnectionFactory = Callable[[str, int, float], _ConnectionLike]
+
+
+def _default_connection_factory(
+    host: str, port: int, timeout: float
+) -> _ConnectionLike:
+    return http.client.HTTPSConnection(host, port=port, timeout=timeout)
+
+
+def _request_hash(host: str, path: str, query: list[tuple[str, str]]) -> str:
+    canonical = "\n".join(
+        ("GET", "https", host, path, urlencode(query, doseq=False))
+    ).encode("utf-8")
+    return hashlib.sha256(canonical).hexdigest()
+
+
+_entry_parts = urlsplit(ENTRY_SOURCE_LOCATOR)
+ENTRY_REQUEST_FINGERPRINT = RequestFingerprint(
+    revision="MOFA_ENTRY_V1",
+    normalized_request_sha256=_request_hash(
+        _entry_parts.hostname or "",
+        _entry_parts.path,
+        parse_qsl(_entry_parts.query, keep_blank_values=True),
+    ),
+)
+
+_WARNING_FIXED_QUERY = (
+    ("returnType", "JSON"),
+    ("numOfRows", "1"),
+    ("pageNo", "1"),
+    ("cond[country_iso_alp2::EQ]", "JP"),
+)
+_warning_parts = urlsplit(WARNING_SOURCE_LOCATOR)
+WARNING_REQUEST_FINGERPRINT = RequestFingerprint(
+    revision="MOFA_WARNING_V1",
+    normalized_request_sha256=_request_hash(
+        _warning_parts.hostname or "",
+        _warning_parts.path,
+        [("ServiceKey", "<redacted>"), *_WARNING_FIXED_QUERY],
+    ),
+)
+
+_COMMON_REQUEST_HEADERS = {
+    "Connection": "close",
+    "User-Agent": "travel-readiness-source-fetch/1",
+}
+_ENTRY_REQUEST_HEADERS = {
+    **_COMMON_REQUEST_HEADERS,
+    "Accept": "text/csv, application/octet-stream",
+}
+_WARNING_REQUEST_HEADERS = {
+    **_COMMON_REQUEST_HEADERS,
+    "Accept": "application/json",
+}
+_PERCENT_ESCAPE = re.compile(rb"%([0-9a-fA-F]{2})")
+_MIN_REFLECTION_FRAGMENT_CHARACTERS = 8
+_MAX_SECRET_CHARACTERS = 512
+
+
+@dataclass(frozen=True, slots=True)
+class _WireResponse:
+    status: int
+    body: bytes = field(repr=False)
+
+
+@dataclass(frozen=True, slots=True)
+class _WireFailure:
+    failure_code: str
+    http_status: int | None = None
+
+
+def _valid_timeout(value: object) -> bool:
+    return type(value) in (int, float) and 1 <= value <= 60
+
+
+def _safe_close(connection: _ConnectionLike | None) -> None:
+    if connection is None:
+        return
+    try:
+        connection.close()
+    except Exception:
+        pass
+
+
+def _read_once(
+    *,
+    host: str,
+    request_target: str,
+    connect_timeout_seconds: int | float,
+    read_timeout_seconds: int | float,
+    max_response_bytes: int,
+    request_headers: dict[str, str],
+    connection_factory: ConnectionFactory,
+) -> _WireResponse | _WireFailure:
+    connection: _ConnectionLike | None = None
+    observed_status: int | None = None
+    try:
+        connection = connection_factory(host, 443, float(connect_timeout_seconds))
+        set_debuglevel = getattr(connection, "set_debuglevel", None)
+        if callable(set_debuglevel):
+            set_debuglevel(0)
+        connection.connect()
+        if connection.sock is None:
+            return _WireFailure(FAILURE_TRANSPORT)
+        connection.sock.settimeout(float(read_timeout_seconds))
+        connection.request(
+            "GET",
+            request_target,
+            body=None,
+            headers=dict(request_headers),
+        )
+        response = connection.getresponse()
+        if type(response.status) is not int or not 100 <= response.status <= 599:
+            return _WireFailure(FAILURE_TRANSPORT)
+        observed_status = response.status
+
+        content_length = response.getheader("Content-Length")
+        if content_length is not None:
+            try:
+                declared_length = int(content_length, 10)
+            except (TypeError, ValueError):
+                declared_length = -1
+            if declared_length > max_response_bytes:
+                return _WireFailure(
+                    FAILURE_RESPONSE_TOO_LARGE, http_status=response.status
+                )
+
+        response_body = response.read(max_response_bytes + 1)
+        if not isinstance(response_body, bytes):
+            return _WireFailure(FAILURE_TRANSPORT)
+        if len(response_body) > max_response_bytes:
+            return _WireFailure(
+                FAILURE_RESPONSE_TOO_LARGE, http_status=response.status
+            )
+        return _WireResponse(response.status, response_body)
+    except (TimeoutError, socket.timeout):
+        return _WireFailure(FAILURE_TIMEOUT, http_status=observed_status)
+    except (OSError, http.client.HTTPException):
+        return _WireFailure(FAILURE_TRANSPORT, http_status=observed_status)
+    except Exception:
+        # Adapter/plugin exceptions are deliberately collapsed to a closed code.
+        return _WireFailure(FAILURE_TRANSPORT, http_status=observed_status)
+    finally:
+        _safe_close(connection)
+
+
+def _failed(
+    fingerprint: RequestFingerprint,
+    failure_code: str,
+    *,
+    http_status: int | None = None,
+    provider_code: str = "",
+) -> SingleAttemptResult:
+    retryable = failure_code in {
+        FAILURE_TIMEOUT,
+        FAILURE_RATE_LIMITED,
+        FAILURE_UPSTREAM_5XX,
+    }
+    return SingleAttemptResult(
+        request_fingerprint=fingerprint,
+        attempt_result=(
+            ATTEMPT_RETRYABLE_FAILED if retryable else ATTEMPT_TERMINAL_FAILED
+        ),
+        http_status=http_status,
+        provider_code=provider_code,
+        failure_code=failure_code,
+    )
+
+
+def _succeeded(
+    fingerprint: RequestFingerprint,
+    *,
+    http_status: int,
+    body: bytes,
+    provider_code: str = "",
+) -> SingleAttemptResult:
+    return SingleAttemptResult(
+        request_fingerprint=fingerprint,
+        attempt_result=ATTEMPT_SUCCEEDED,
+        http_status=http_status,
+        provider_code=provider_code,
+        response_sha256=hashlib.sha256(body).hexdigest(),
+        byte_count=len(body),
+        body=body,
+    )
+
+
+def _classify_http_failure(
+    fingerprint: RequestFingerprint,
+    status: int,
+    *,
+    provider_code: str = "",
+) -> SingleAttemptResult | None:
+    if status in (401, 403):
+        return _failed(
+            fingerprint,
+            FAILURE_AUTHENTICATION,
+            http_status=status,
+            provider_code=provider_code,
+        )
+    if status == 429:
+        return _failed(
+            fingerprint,
+            FAILURE_RATE_LIMITED,
+            http_status=status,
+            provider_code=provider_code,
+        )
+    if 500 <= status <= 599:
+        return _failed(
+            fingerprint,
+            FAILURE_UPSTREAM_5XX,
+            http_status=status,
+            provider_code=provider_code,
+        )
+    if status != 200:
+        return _failed(
+            fingerprint,
+            FAILURE_HTTP_CLIENT,
+            http_status=status,
+            provider_code=provider_code,
+        )
+    return None
+
+
+def _provider_result_code(body: bytes) -> str | None:
+    try:
+        document = json.loads(body.decode("utf-8"))
+    except (UnicodeError, json.JSONDecodeError, RecursionError):
+        document = None
+
+    if isinstance(document, dict):
+        response = document.get("response")
+        if isinstance(response, dict):
+            header = response.get("header")
+            if isinstance(header, dict):
+                result_code = header.get("resultCode")
+                if isinstance(result_code, str):
+                    return result_code
+
+    try:
+        root = ElementTree.fromstring(body)
+    except (ElementTree.ParseError, RecursionError, ValueError):
+        return None
+    for element in root.iter():
+        local_name = element.tag.rsplit("}", 1)[-1]
+        if local_name in {"returnReasonCode", "resultCode"}:
+            value = element.text
+            if isinstance(value, str):
+                return value.strip()
+    return None
+
+
+def _closed_provider_code(result_code: str | None) -> str:
+    if result_code == "0":
+        return PROVIDER_SUCCESS_0
+    if result_code == "20":
+        return PROVIDER_GATEWAY_20
+    if result_code is not None:
+        return PROVIDER_OTHER_ERROR
+    return ""
+
+
+def _normalize_percent_hex(value: bytes) -> bytes:
+    return _PERCENT_ESCAPE.sub(lambda match: b"%" + match.group(1).upper(), value)
+
+
+def _contains_secret_reflection(
+    response_body: bytes,
+    original: str,
+    decoded: str,
+) -> bool:
+    normalized_response = _normalize_percent_hex(response_body)
+    json_escaped = json.dumps(decoded, ensure_ascii=True)[1:-1].replace(
+        "/", r"\/"
+    )
+    full_variants = (
+        original,
+        decoded,
+        quote(decoded, safe=""),
+        quote_plus(decoded, safe=""),
+        quote(original, safe=""),
+        quote_plus(original, safe=""),
+        json_escaped,
+    )
+    for variant in full_variants:
+        normalized = _normalize_percent_hex(variant.encode("utf-8"))
+        if normalized and normalized in normalized_response:
+            return True
+        if len(variant) < _MIN_REFLECTION_FRAGMENT_CHARACTERS:
+            continue
+        for start in range(
+            len(variant) - _MIN_REFLECTION_FRAGMENT_CHARACTERS + 1
+        ):
+            fragment = variant[
+                start : start + _MIN_REFLECTION_FRAGMENT_CHARACTERS
+            ]
+            normalized_fragment = _normalize_percent_hex(
+                fragment.encode("utf-8")
+            )
+            if normalized_fragment in normalized_response:
+                return True
+    return False
+
+
+def fetch_entry_attachment(
+    *,
+    official_locator: str,
+    connect_timeout_seconds: int | float,
+    read_timeout_seconds: int | float,
+    connection_factory: ConnectionFactory = _default_connection_factory,
+) -> SingleAttemptResult:
+    """Fetch exactly the approved anonymous entry attachment once."""
+
+    fingerprint = ENTRY_REQUEST_FINGERPRINT
+    if (
+        official_locator != ENTRY_SOURCE_LOCATOR
+        or not _valid_timeout(connect_timeout_seconds)
+        or not _valid_timeout(read_timeout_seconds)
+    ):
+        return _failed(fingerprint, FAILURE_HTTP_CLIENT)
+
+    parts = urlsplit(ENTRY_SOURCE_LOCATOR)
+    wire_result = _read_once(
+        host=parts.hostname or "",
+        request_target=f"{parts.path}?{parts.query}",
+        connect_timeout_seconds=connect_timeout_seconds,
+        read_timeout_seconds=read_timeout_seconds,
+        max_response_bytes=ENTRY_MAX_RESPONSE_BYTES,
+        request_headers=_ENTRY_REQUEST_HEADERS,
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
+    http_failure = _classify_http_failure(fingerprint, wire_result.status)
+    if http_failure is not None:
+        return http_failure
+    return _succeeded(
+        fingerprint,
+        http_status=wire_result.status,
+        body=wire_result.body,
+    )
+
+
+def fetch_travel_alarm_jp(
+    *,
+    official_locator: str,
+    secret_reference_name: str,
+    secret_value: str,
+    connect_timeout_seconds: int | float,
+    read_timeout_seconds: int | float,
+    connection_factory: ConnectionFactory = _default_connection_factory,
+) -> SingleAttemptResult:
+    """Fetch the approved JP/page-one TravelAlarmService2 request once."""
+
+    fingerprint = WARNING_REQUEST_FINGERPRINT
+    if (
+        official_locator != WARNING_SOURCE_LOCATOR
+        or not _valid_timeout(connect_timeout_seconds)
+        or not _valid_timeout(read_timeout_seconds)
+    ):
+        return _failed(fingerprint, FAILURE_HTTP_CLIENT)
+    if (
+        secret_reference_name != WARNING_SECRET_REFERENCE
+        or not isinstance(secret_value, str)
+        or not secret_value
+        or len(secret_value) > _MAX_SECRET_CHARACTERS
+    ):
+        return _failed(fingerprint, FAILURE_AUTHENTICATION)
+
+    try:
+        decoded_secret = unquote(secret_value, encoding="utf-8", errors="strict")
+        query = urlencode(
+            [("ServiceKey", decoded_secret), *_WARNING_FIXED_QUERY], doseq=False
+        )
+    except (UnicodeError, TypeError, ValueError):
+        return _failed(fingerprint, FAILURE_AUTHENTICATION)
+
+    parts = urlsplit(WARNING_SOURCE_LOCATOR)
+    wire_result = _read_once(
+        host=parts.hostname or "",
+        request_target=f"{parts.path}?{query}",
+        connect_timeout_seconds=connect_timeout_seconds,
+        read_timeout_seconds=read_timeout_seconds,
+        max_response_bytes=WARNING_MAX_RESPONSE_BYTES,
+        request_headers=_WARNING_REQUEST_HEADERS,
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
+
+    if _contains_secret_reflection(
+        wire_result.body,
+        secret_value,
+        decoded_secret,
+    ):
+        return _failed(
+            fingerprint,
+            FAILURE_SECRET_REFLECTION,
+            http_status=wire_result.status,
+        )
+
+    provider_result_code = _provider_result_code(wire_result.body)
+    provider_code = _closed_provider_code(provider_result_code)
+    http_provider_code = (
+        provider_code if provider_result_code not in (None, "0") else ""
+    )
+    http_failure = _classify_http_failure(
+        fingerprint, wire_result.status, provider_code=http_provider_code
+    )
+    if http_failure is not None:
+        return http_failure
+
+    if provider_result_code is not None and provider_result_code != "0":
+        return _failed(
+            fingerprint,
+            FAILURE_PROVIDER_ERROR,
+            http_status=wire_result.status,
+            provider_code=provider_code,
+        )
+
+    return _succeeded(
+        fingerprint,
+        http_status=wire_result.status,
+        body=wire_result.body,
+        provider_code=(
+            PROVIDER_SUCCESS_0 if provider_result_code == "0" else ""
+        ),
+    )


