## `test(sources): verify live parser replay`

diff --git a/operations/management/commands/check_live_parser_replay.py b/operations/management/commands/check_live_parser_replay.py
new file mode 100644
index 0000000..bb80b93
--- /dev/null
+++ b/operations/management/commands/check_live_parser_replay.py
@@ -0,0 +1,104 @@
+from django.core.management.base import BaseCommand, CommandError
+
+from operations.source_replay import (
+    ALLOWED_FAILURE_CODES,
+    ALLOWED_PROVIDER_CODES,
+    ALLOWED_SOURCES,
+    FAILED,
+    SOURCE_ENTRY,
+    SOURCE_INTERNAL,
+    SOURCE_WARNING,
+    VERIFIED,
+    LiveParserReplayReceipt,
+    verify_live_parser_replay,
+)
+from sources.transport import (
+    ENTRY_REQUEST_FINGERPRINT,
+    WARNING_REQUEST_FINGERPRINT,
+)
+
+
+def _invoke_without_exception_context():
+    try:
+        return verify_live_parser_replay()
+    except BaseException:
+        return None
+
+
+def _closed_failed_receipt(receipt) -> bool:
+    if not isinstance(receipt, LiveParserReplayReceipt):
+        return False
+    if (
+        receipt.result != FAILED
+        or receipt.source not in ALLOWED_SOURCES
+        or receipt.failure_code not in ALLOWED_FAILURE_CODES
+        or receipt.provider_code not in ALLOWED_PROVIDER_CODES
+    ):
+        return False
+    if receipt.source == SOURCE_INTERNAL:
+        return (
+            receipt.failure_code == "INTERNAL"
+            and receipt.provider_code == ""
+            and receipt.request_revision == ""
+            and receipt.request_sha256 == ""
+        )
+    if receipt.source == SOURCE_ENTRY and (
+        receipt.failure_code == "SECRET_UNAVAILABLE"
+        or receipt.provider_code != ""
+    ):
+        return False
+    if (
+        receipt.provider_code != ""
+        and receipt.failure_code not in {"FETCH_FAILED", "RESPONSE_INTEGRITY"}
+    ):
+        return False
+    expected = {
+        SOURCE_ENTRY: ENTRY_REQUEST_FINGERPRINT,
+        SOURCE_WARNING: WARNING_REQUEST_FINGERPRINT,
+    }[receipt.source]
+    return (
+        receipt.request_revision == expected.revision
+        and receipt.request_sha256 == expected.normalized_request_sha256
+    )
+
+
+def _closed_success_receipt(receipt) -> bool:
+    return isinstance(receipt, LiveParserReplayReceipt) and receipt == (
+        LiveParserReplayReceipt(
+            result=VERIFIED,
+            source="",
+            failure_code="",
+            provider_code="",
+            request_revision="",
+            request_sha256="",
+        )
+    )
+
+
+class Command(BaseCommand):
+    help = "Ephemerally replay each approved live source through its parser."
+
+    def handle(self, *args, **options):
+        receipt = _invoke_without_exception_context()
+        if not _closed_success_receipt(receipt) and not _closed_failed_receipt(
+            receipt
+        ):
+            raise CommandError(
+                "live_parser_replay_result=FAILED source=INTERNAL "
+                "failure=INTERNAL"
+            ) from None
+        if _closed_success_receipt(receipt):
+            self.stdout.write(
+                "live_parser_replay_result=VERIFIED "
+                "sources=2 requests=2 parses=4"
+            )
+            return
+
+        provider = receipt.provider_code or "NONE"
+        revision = receipt.request_revision or "NONE"
+        request_hash = receipt.request_sha256 or "NONE"
+        raise CommandError(
+            f"live_parser_replay_result=FAILED source={receipt.source} "
+            f"failure={receipt.failure_code} provider_code={provider} "
+            f"request_revision={revision} request_sha256={request_hash}"
+        ) from None
diff --git a/operations/source_replay.py b/operations/source_replay.py
new file mode 100644
index 0000000..370c9ac
--- /dev/null
+++ b/operations/source_replay.py
@@ -0,0 +1,273 @@
+"""Ephemeral live replay of the two approved source parsers.
+
+The verifier makes one bounded request per source, keeps response bytes in
+process memory only, and runs the production parser twice.  Its result is a
+closed receipt that cannot carry response bytes, source facts, locators, or a
+secret value.
+"""
+
+from __future__ import annotations
+
+import hashlib
+import os
+from dataclasses import dataclass
+from typing import Callable
+
+from entry_requirements.parser import parse_entry_csv
+from sources.models import ParseRun
+from sources.transport import (
+    ENTRY_REQUEST_FINGERPRINT,
+    ENTRY_SOURCE_LOCATOR,
+    PROVIDER_GATEWAY_20,
+    PROVIDER_OTHER_ERROR,
+    PROVIDER_SUCCESS_0,
+    WARNING_REQUEST_FINGERPRINT,
+    WARNING_SECRET_REFERENCE,
+    WARNING_SOURCE_LOCATOR,
+    RequestFingerprint,
+    SingleAttemptResult,
+    fetch_entry_attachment,
+    fetch_travel_alarm_jp,
+)
+from travel_warnings.parser import (
+    TravelWarningParseSuccess,
+    parse_travel_alarm_jp,
+)
+
+
+VERIFIED = "VERIFIED"
+FAILED = "FAILED"
+
+SOURCE_ENTRY = "ENTRY"
+SOURCE_WARNING = "WARNING"
+SOURCE_INTERNAL = "INTERNAL"
+
+FAILURE_SECRET_UNAVAILABLE = "SECRET_UNAVAILABLE"
+FAILURE_FETCH = "FETCH_FAILED"
+FAILURE_RESPONSE_INTEGRITY = "RESPONSE_INTEGRITY"
+FAILURE_PARSE = "PARSE_FAILED"
+FAILURE_NONDETERMINISTIC = "NONDETERMINISTIC"
+FAILURE_INTERNAL = "INTERNAL"
+
+_ALLOWED_PROVIDER_CODES = frozenset(
+    {"", PROVIDER_SUCCESS_0, PROVIDER_GATEWAY_20, PROVIDER_OTHER_ERROR}
+)
+ALLOWED_PROVIDER_CODES = _ALLOWED_PROVIDER_CODES
+ALLOWED_SOURCES = frozenset({SOURCE_ENTRY, SOURCE_WARNING, SOURCE_INTERNAL})
+ALLOWED_FAILURE_CODES = frozenset(
+    {
+        FAILURE_SECRET_UNAVAILABLE,
+        FAILURE_FETCH,
+        FAILURE_RESPONSE_INTEGRITY,
+        FAILURE_PARSE,
+        FAILURE_NONDETERMINISTIC,
+        FAILURE_INTERNAL,
+    }
+)
+_CONNECT_TIMEOUT_SECONDS = 5
+_READ_TIMEOUT_SECONDS = 10
+
+EntryTransport = Callable[..., SingleAttemptResult]
+WarningTransport = Callable[..., SingleAttemptResult]
+Parser = Callable[[bytes], object]
+SecretLoader = Callable[[str], object]
+
+
+@dataclass(frozen=True, slots=True)
+class LiveParserReplayReceipt:
+    result: str
+    source: str
+    failure_code: str
+    provider_code: str
+    request_revision: str
+    request_sha256: str
+
+
+def _closed_provider_code(value: object) -> str:
+    return value if type(value) is str and value in _ALLOWED_PROVIDER_CODES else ""
+
+
+def _failure(
+    source: str,
+    failure_code: str,
+    fingerprint: RequestFingerprint | None = None,
+    provider_code: object = "",
+) -> LiveParserReplayReceipt:
+    return LiveParserReplayReceipt(
+        result=FAILED,
+        source=source,
+        failure_code=failure_code,
+        provider_code=_closed_provider_code(provider_code),
+        request_revision=fingerprint.revision if fingerprint is not None else "",
+        request_sha256=(
+            fingerprint.normalized_request_sha256
+            if fingerprint is not None
+            else ""
+        ),
+    )
+
+
+def _success() -> LiveParserReplayReceipt:
+    return LiveParserReplayReceipt(
+        result=VERIFIED,
+        source="",
+        failure_code="",
+        provider_code="",
+        request_revision="",
+        request_sha256="",
+    )
+
+
+def _response_is_intact(
+    result: SingleAttemptResult,
+    expected_fingerprint: RequestFingerprint,
+) -> bool:
+    return (
+        result.request_fingerprint == expected_fingerprint
+        and result.succeeded
+        and result.http_status == 200
+        and type(result.body) is bytes
+        and bool(result.body)
+        and result.byte_count == len(result.body)
+        and result.response_sha256 == hashlib.sha256(result.body).hexdigest()
+    )
+
+
+def _entry_parse_is_valid(result: object) -> bool:
+    return (
+        getattr(result, "outcome", None) == ParseRun.Outcome.VALIDATED
+        and getattr(result, "projection", None) is not None
+    )
+
+
+def _warning_parse_is_valid(result: object) -> bool:
+    return isinstance(result, TravelWarningParseSuccess)
+
+
+def _replay_one(
+    *,
+    source: str,
+    response: SingleAttemptResult,
+    fingerprint: RequestFingerprint,
+    parser: Parser,
+    validator: Callable[[object], bool],
+) -> LiveParserReplayReceipt | None:
+    if not _response_is_intact(response, fingerprint):
+        failure_code = (
+            FAILURE_FETCH
+            if not response.succeeded
+            else FAILURE_RESPONSE_INTEGRITY
+        )
+        return _failure(
+            source,
+            failure_code,
+            fingerprint,
+            response.provider_code,
+        )
+
+    try:
+        first = parser(response.body)
+        second = parser(response.body)
+    except BaseException:
+        return _failure(source, FAILURE_PARSE, fingerprint)
+
+    if not validator(first) or not validator(second):
+        return _failure(source, FAILURE_PARSE, fingerprint)
+    if first != second:
+        return _failure(source, FAILURE_NONDETERMINISTIC, fingerprint)
+    return None
+
+
+def verify_live_parser_replay(
+    *,
+    entry_transport: EntryTransport = fetch_entry_attachment,
+    warning_transport: WarningTransport = fetch_travel_alarm_jp,
+    entry_parser: Parser = parse_entry_csv,
+    warning_parser: Parser = parse_travel_alarm_jp,
+    secret_loader: SecretLoader = os.environ.get,
+) -> LiveParserReplayReceipt:
+    """Verify two live bodies without persistence, retries, or detailed output."""
+
+    secret_value: object = None
+    entry_response: SingleAttemptResult | None = None
+    warning_response: SingleAttemptResult | None = None
+    try:
+        try:
+            entry_response = entry_transport(
+                official_locator=ENTRY_SOURCE_LOCATOR,
+                connect_timeout_seconds=_CONNECT_TIMEOUT_SECONDS,
+                read_timeout_seconds=_READ_TIMEOUT_SECONDS,
+            )
+        except BaseException:
+            return _failure(
+                SOURCE_ENTRY,
+                FAILURE_FETCH,
+                ENTRY_REQUEST_FINGERPRINT,
+            )
+        if not isinstance(entry_response, SingleAttemptResult):
+            return _failure(
+                SOURCE_ENTRY,
+                FAILURE_INTERNAL,
+                ENTRY_REQUEST_FINGERPRINT,
+            )
+        entry_failure = _replay_one(
+            source=SOURCE_ENTRY,
+            response=entry_response,
+            fingerprint=ENTRY_REQUEST_FINGERPRINT,
+            parser=entry_parser,
+            validator=_entry_parse_is_valid,
+        )
+        if entry_failure is not None:
+            return entry_failure
+
+        try:
+            secret_value = secret_loader(WARNING_SECRET_REFERENCE)
+        except BaseException:
+            return _failure(
+                SOURCE_WARNING,
+                FAILURE_SECRET_UNAVAILABLE,
+                WARNING_REQUEST_FINGERPRINT,
+            )
+        if type(secret_value) is not str or not secret_value:
+            return _failure(
+                SOURCE_WARNING,
+                FAILURE_SECRET_UNAVAILABLE,
+                WARNING_REQUEST_FINGERPRINT,
+            )
+
+        try:
+            warning_response = warning_transport(
+                official_locator=WARNING_SOURCE_LOCATOR,
+                secret_reference_name=WARNING_SECRET_REFERENCE,
+                secret_value=secret_value,
+                connect_timeout_seconds=_CONNECT_TIMEOUT_SECONDS,
+                read_timeout_seconds=_READ_TIMEOUT_SECONDS,
+            )
+        except BaseException:
+            return _failure(
+                SOURCE_WARNING,
+                FAILURE_FETCH,
+                WARNING_REQUEST_FINGERPRINT,
+            )
+        if not isinstance(warning_response, SingleAttemptResult):
+            return _failure(
+                SOURCE_WARNING,
+                FAILURE_INTERNAL,
+                WARNING_REQUEST_FINGERPRINT,
+            )
+        warning_failure = _replay_one(
+            source=SOURCE_WARNING,
+            response=warning_response,
+            fingerprint=WARNING_REQUEST_FINGERPRINT,
+            parser=warning_parser,
+            validator=_warning_parse_is_valid,
+        )
+        if warning_failure is not None:
+            return warning_failure
+        return _success()
+    except BaseException:
+        return _failure(SOURCE_INTERNAL, FAILURE_INTERNAL)
+    finally:
+        secret_value = None
+        entry_response = None
+        warning_response = None
diff --git a/operations/tests/test_live_parser_replay.py b/operations/tests/test_live_parser_replay.py
new file mode 100644
index 0000000..598a3bc
--- /dev/null
+++ b/operations/tests/test_live_parser_replay.py
@@ -0,0 +1,283 @@
+import hashlib
+import io
+from dataclasses import replace
+from datetime import date
+from unittest.mock import patch
+
+from django.core.management import call_command
+from django.core.management.base import CommandError
+from django.test import SimpleTestCase
+
+from entry_requirements.parser import EntryParseResult, EntryProjection
+from operations.source_replay import (
+    FAILED,
+    FAILURE_FETCH,
+    FAILURE_NONDETERMINISTIC,
+    FAILURE_PARSE,
+    FAILURE_SECRET_UNAVAILABLE,
+    SOURCE_ENTRY,
+    SOURCE_WARNING,
+    VERIFIED,
+    verify_live_parser_replay,
+)
+from sources.models import ParseRun
+from sources.transport import (
+    ATTEMPT_SUCCEEDED,
+    ENTRY_REQUEST_FINGERPRINT,
+    FAILURE_AUTHENTICATION,
+    PROVIDER_GATEWAY_20,
+    WARNING_REQUEST_FINGERPRINT,
+    SingleAttemptResult,
+)
+from travel_warnings.parser import (
+    ParsedTravelWarning,
+    TravelWarningParseSuccess,
+)
+
+
+SECRET_MARKER = "private-live-replay-secret-marker"
+ENTRY_BODY = b"entry-live-body"
+WARNING_BODY = b"warning-live-body"
+
+
+def _response(fingerprint, body, provider_code=""):
+    return SingleAttemptResult(
+        request_fingerprint=fingerprint,
+        attempt_result=ATTEMPT_SUCCEEDED,
+        http_status=200,
+        provider_code=provider_code,
+        response_sha256=hashlib.sha256(body).hexdigest(),
+        byte_count=len(body),
+        body=body,
+    )
+
+
+def _entry_success(period="90일"):
+    return EntryParseResult(
+        outcome=ParseRun.Outcome.VALIDATED,
+        failure_code="",
+        observed_schema_fingerprint_sha256="a" * 64,
+        projection=EntryProjection(
+            country_iso2="JP",
+            passport_policy_code="KR_ORDINARY_SHORT_TOURISM",
+            passport_policy_revision="V1",
+            ordinary_passport_period_text=period,
+            basis_text="일반여권 소지자: unit-test fact",
+            snapshot_date=date(2026, 8, 1),
+            typed_fingerprint_sha256="b" * 64,
+        ),
+    )
+
+
+def _warning_success(level="1"):
+    return TravelWarningParseSuccess(
+        warning=ParsedTravelWarning(
+            country_iso2="JP",
+            country_name_ko="일본",
+            country_name_en="Japan",
+            source_alarm_level_code=level,
+            source_scope_type="일부",
+            source_scope_text="unit-test scope",
+            source_written_date=date(2026, 8, 1),
+            typed_fingerprint_sha256="c" * 64,
+        ),
+        observed_schema_fingerprint_sha256="d" * 64,
+    )
+
+
+class LiveParserReplayTests(SimpleTestCase):
+    def _verify(self, **overrides):
+        values = {
+            "entry_transport": lambda **kwargs: _response(
+                ENTRY_REQUEST_FINGERPRINT, ENTRY_BODY
+            ),
+            "warning_transport": lambda **kwargs: _response(
+                WARNING_REQUEST_FINGERPRINT, WARNING_BODY
+            ),
+            "entry_parser": lambda body: _entry_success(),
+            "warning_parser": lambda body: _warning_success(),
+            "secret_loader": lambda name: SECRET_MARKER,
+        }
+        values.update(overrides)
+        return verify_live_parser_replay(**values)
+
+    def test_one_request_and_two_identical_parses_per_source(self):
+        calls = {
+            "entry_fetch": 0,
+            "warning_fetch": 0,
+            "entry_parse": 0,
+            "warning_parse": 0,
+        }
+
+        def entry_transport(**kwargs):
+            calls["entry_fetch"] += 1
+            return _response(ENTRY_REQUEST_FINGERPRINT, ENTRY_BODY)
+
+        def warning_transport(**kwargs):
+            calls["warning_fetch"] += 1
+            self.assertEqual(kwargs["secret_value"], SECRET_MARKER)
+            calls["warning_parse"] += 0
+            return _response(WARNING_REQUEST_FINGERPRINT, WARNING_BODY)
+
+        def entry_parser(body):
+            self.assertEqual(body, ENTRY_BODY)
+            calls["entry_parse"] += 1
+            return _entry_success()
+
+        def warning_parser(body):
+            self.assertEqual(body, WARNING_BODY)
+            calls["warning_parse"] += 1
+            return _warning_success()
+
+        receipt = self._verify(
+            entry_transport=entry_transport,
+            warning_transport=warning_transport,
+            entry_parser=entry_parser,
+            warning_parser=warning_parser,
+        )
+
+        self.assertEqual(receipt.result, VERIFIED)
+        self.assertEqual(
+            calls,
+            {
+                "entry_fetch": 1,
+                "warning_fetch": 1,
+                "entry_parse": 2,
+                "warning_parse": 2,
+            },
+        )
+        self.assertNotIn(SECRET_MARKER, repr(receipt))
+        self.assertNotIn("live-body", repr(receipt))
+
+    def test_secret_is_loaded_only_after_entry_validation(self):
+        secret_calls = []
+        receipt = self._verify(
+            entry_parser=lambda body: replace(
+                _entry_success(),
+                outcome=ParseRun.Outcome.QUARANTINED,
+                projection=None,
+            ),
+            secret_loader=lambda name: secret_calls.append(name) or SECRET_MARKER,
+        )
+        self.assertEqual(receipt.result, FAILED)
+        self.assertEqual(receipt.source, SOURCE_ENTRY)
+        self.assertEqual(receipt.failure_code, FAILURE_PARSE)
+        self.assertEqual(secret_calls, [])
+
+    def test_missing_secret_fails_without_calling_warning_transport(self):
+        warning_calls = []
+        receipt = self._verify(
+            secret_loader=lambda name: None,
+            warning_transport=lambda **kwargs: warning_calls.append(kwargs),
+        )
+        self.assertEqual(receipt.result, FAILED)
+        self.assertEqual(receipt.source, SOURCE_WARNING)
+        self.assertEqual(receipt.failure_code, FAILURE_SECRET_UNAVAILABLE)
+        self.assertEqual(warning_calls, [])
+
+    def test_parse_exception_detail_and_body_never_enter_receipt(self):
+        marker = "private-parser-exception-marker"
+
+        def failed_parser(body):
+            raise RuntimeError(f"{marker}:{body!r}:{SECRET_MARKER}")
+
+        receipt = self._verify(entry_parser=failed_parser)
+
+        self.assertEqual(receipt.failure_code, FAILURE_PARSE)
+        self.assertNotIn(marker, repr(receipt))
+        self.assertNotIn(SECRET_MARKER, repr(receipt))
+        self.assertNotIn("live-body", repr(receipt))
+
+    def test_parser_drift_is_fail_closed(self):
+        results = iter((_entry_success("90일"), _entry_success("30일")))
+        receipt = self._verify(entry_parser=lambda body: next(results))
+        self.assertEqual(receipt.failure_code, FAILURE_NONDETERMINISTIC)
+
+    def test_warning_provider_failure_is_allowlisted_and_redacted(self):
+        failed = SingleAttemptResult(
+            request_fingerprint=WARNING_REQUEST_FINGERPRINT,
+            attempt_result="TERMINAL_FAILED",
+            http_status=403,
+            provider_code=PROVIDER_GATEWAY_20,
+            failure_code=FAILURE_AUTHENTICATION,
+        )
+        receipt = self._verify(warning_transport=lambda **kwargs: failed)
+        self.assertEqual(receipt.failure_code, FAILURE_FETCH)
+        self.assertEqual(receipt.provider_code, PROVIDER_GATEWAY_20)
+        rendered = repr(receipt)
+        self.assertIn(WARNING_REQUEST_FINGERPRINT.revision, rendered)
+        self.assertIn(WARNING_REQUEST_FINGERPRINT.normalized_request_sha256, rendered)
+        self.assertNotIn(SECRET_MARKER, rendered)
+
+    def test_invalid_transport_object_fails_with_fixed_receipt(self):
+        marker = "private-invalid-transport-marker"
+        receipt = self._verify(entry_transport=lambda **kwargs: marker)
+        self.assertEqual(receipt.result, FAILED)
+        self.assertEqual(receipt.source, SOURCE_ENTRY)
+        self.assertNotIn(marker, repr(receipt))
+
+    def test_management_command_success_is_a_fixed_receipt(self):
+        stdout = io.StringIO()
+        with patch(
+            "operations.management.commands.check_live_parser_replay."
+            "verify_live_parser_replay",
+            return_value=self._verify(),
+        ):
+            call_command("check_live_parser_replay", stdout=stdout)
+        self.assertEqual(
+            stdout.getvalue().strip(),
+            "live_parser_replay_result=VERIFIED sources=2 requests=2 parses=4",
+        )
+
+    def test_management_command_failure_has_only_redacted_shape(self):
+        receipt = self._verify(
+            warning_transport=lambda **kwargs: SingleAttemptResult(
+                request_fingerprint=WARNING_REQUEST_FINGERPRINT,
+                attempt_result="TERMINAL_FAILED",
+                http_status=403,
+                provider_code=PROVIDER_GATEWAY_20,
+                failure_code=FAILURE_AUTHENTICATION,
+            )
+        )
+        stderr = io.StringIO()
+        with patch(
+            "operations.management.commands.check_live_parser_replay."
+            "verify_live_parser_replay",
+            return_value=receipt,
+        ):
+            with self.assertRaises(CommandError) as raised:
+                call_command("check_live_parser_replay", stderr=stderr)
+        rendered = str(raised.exception)
+        self.assertIn("provider_code=MOFA_GATEWAY_20", rendered)
+        self.assertIn("request_revision=MOFA_WARNING_V1", rendered)
+        self.assertNotIn(SECRET_MARKER, rendered)
+        self.assertNotIn("https://", rendered)
+        self.assertEqual(stderr.getvalue(), "")
+
+    def test_management_command_rejects_an_unclosed_receipt(self):
+        injected = replace(
+            self._verify(
+                warning_transport=lambda **kwargs: SingleAttemptResult(
+                    request_fingerprint=WARNING_REQUEST_FINGERPRINT,
+                    attempt_result="TERMINAL_FAILED",
+                    failure_code=FAILURE_AUTHENTICATION,
+                )
+            ),
+            failure_code=SECRET_MARKER,
+            request_revision=SECRET_MARKER,
+            request_sha256=SECRET_MARKER,
+        )
+        with patch(
+            "operations.management.commands.check_live_parser_replay."
+            "verify_live_parser_replay",
+            return_value=injected,
+        ):
+            with self.assertRaises(CommandError) as raised:
+                call_command("check_live_parser_replay")
+        rendered = str(raised.exception)
+        self.assertEqual(
+            rendered,
+            "live_parser_replay_result=FAILED source=INTERNAL "
+            "failure=INTERNAL",
+        )
+        self.assertNotIn(SECRET_MARKER, rendered)


