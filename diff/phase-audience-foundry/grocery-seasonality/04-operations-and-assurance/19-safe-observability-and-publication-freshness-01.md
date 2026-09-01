# 안전한 관측성과 자료 신선도

## `feat(ops): add redacted structured events`

diff --git a/grocery/observability.py b/grocery/observability.py
new file mode 100644
index 0000000..d624dd6
--- /dev/null
+++ b/grocery/observability.py
@@ -0,0 +1,311 @@
+"""Bounded structured observability without source or user data payloads.
+
+The formatter deliberately ignores ``LogRecord.msg``, arguments, exception text,
+and every attribute outside the explicit event envelope.  This keeps request
+queries, search terms, response bodies, credentials, and provider identifiers out
+of logs by construction.
+"""
+
+from __future__ import annotations
+
+import json
+import logging
+import re
+from collections.abc import Callable
+from contextvars import ContextVar, Token
+from datetime import UTC, datetime
+from typing import Final, Literal, cast
+from uuid import UUID, uuid4
+
+from django.http import HttpRequest, HttpResponseBase
+
+type Severity = Literal["DEBUG", "INFO", "WARNING", "ERROR", "CRITICAL"]
+
+OBSERVABILITY_KEYS: Final[tuple[str, ...]] = (
+    "timestamp",
+    "severity",
+    "message_code",
+    "request_id",
+    "deploy_version",
+    "command_run_id",
+    "lifecycle_id",
+    "lifecycle_status",
+    "lifecycle_event",
+)
+
+_EVENT_ATTRIBUTE: Final = "_grocery_observability_event"
+_NORMALIZED_ATTRIBUTE: Final = "_grocery_normalized_observability_event"
+_INVALID_MESSAGE_CODE: Final = "observability.invalid_event"
+_REQUEST_ID_HEADER: Final = "X-Request-ID"
+
+_UUID_PATTERN: Final = re.compile(
+    r"^[0-9a-f]{8}-[0-9a-f]{4}-[1-8][0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$"
+)
+_MESSAGE_CODE_PATTERN: Final = re.compile(r"^[a-z][a-z0-9]*(?:[._-][a-z0-9]+){1,7}$")
+_DEPLOY_VERSION_PATTERN: Final = re.compile(r"^[0-9a-f]{7,40}$")
+_LIFECYCLE_TOKEN_PATTERN: Final = re.compile(r"^[A-Z][A-Z0-9_]{0,63}$")
+
+_LEVEL_BY_SEVERITY: Final[dict[Severity, int]] = {
+    "DEBUG": logging.DEBUG,
+    "INFO": logging.INFO,
+    "WARNING": logging.WARNING,
+    "ERROR": logging.ERROR,
+    "CRITICAL": logging.CRITICAL,
+}
+_SEVERITY_BY_LEVEL: Final[dict[int, Severity]] = {
+    value: key for key, value in _LEVEL_BY_SEVERITY.items()
+}
+
+_request_id_context: ContextVar[str | None] = ContextVar(
+    "grocery_request_id",
+    default=None,
+)
+
+
+class ObservabilityValidationError(ValueError):
+    """An event did not fit the deliberately narrow logging contract."""
+
+
+def _canonical_uuid(value: str | UUID, *, field_name: str) -> str:
+    if type(value) is UUID:
+        return str(value)
+    if type(value) is not str or _UUID_PATTERN.fullmatch(value) is None:
+        raise ObservabilityValidationError(f"{field_name} must be a canonical UUID")
+    try:
+        parsed = UUID(value)
+    except ValueError as exc:
+        raise ObservabilityValidationError(f"{field_name} must be a canonical UUID") from exc
+    canonical = str(parsed)
+    if canonical != value:
+        raise ObservabilityValidationError(f"{field_name} must be a canonical UUID")
+    return canonical
+
+
+def _validated_text(value: str, *, field_name: str, pattern: re.Pattern[str]) -> str:
+    if type(value) is not str or pattern.fullmatch(value) is None:
+        raise ObservabilityValidationError(f"{field_name} has an invalid value")
+    return value
+
+
+def make_observability_event(
+    message_code: str,
+    *,
+    request_id: str | UUID | None = None,
+    deploy_version: str | None = None,
+    command_run_id: str | UUID | None = None,
+    lifecycle_id: str | UUID | None = None,
+    lifecycle_status: str | None = None,
+    lifecycle_event: str | None = None,
+) -> dict[str, str]:
+    """Build the only payload shape accepted by the JSON formatter.
+
+    There is intentionally no arbitrary ``extra`` mapping.  Callers can record
+    correlation and lifecycle state, but cannot attach source payloads or user
+    input to an event.
+    """
+
+    event = {
+        "message_code": _validated_text(
+            message_code,
+            field_name="message_code",
+            pattern=_MESSAGE_CODE_PATTERN,
+        )
+    }
+    if request_id is not None:
+        event["request_id"] = _canonical_uuid(request_id, field_name="request_id")
+    if deploy_version is not None:
+        event["deploy_version"] = _validated_text(
+            deploy_version,
+            field_name="deploy_version",
+            pattern=_DEPLOY_VERSION_PATTERN,
+        )
+    if command_run_id is not None:
+        event["command_run_id"] = _canonical_uuid(
+            command_run_id,
+            field_name="command_run_id",
+        )
+    if lifecycle_id is not None:
+        event["lifecycle_id"] = _canonical_uuid(lifecycle_id, field_name="lifecycle_id")
+    if lifecycle_status is not None:
+        event["lifecycle_status"] = _validated_text(
+            lifecycle_status,
+            field_name="lifecycle_status",
+            pattern=_LIFECYCLE_TOKEN_PATTERN,
+        )
+    if lifecycle_event is not None:
+        event["lifecycle_event"] = _validated_text(
+            lifecycle_event,
+            field_name="lifecycle_event",
+            pattern=_LIFECYCLE_TOKEN_PATTERN,
+        )
+    return event
+
+
+def _normalize_event(raw_event: object) -> dict[str, str] | None:
+    """Copy known, valid fields without iterating or stringifying unknown values."""
+
+    if type(raw_event) is not dict:
+        return None
+
+    event_mapping = cast(dict[object, object], raw_event)
+    message_code = event_mapping.get("message_code")
+    if type(message_code) is not str:
+        return None
+
+    def optional_text(field_name: str) -> str | None:
+        value = event_mapping.get(field_name)
+        if value is None:
+            return None
+        if type(value) is not str:
+            raise ObservabilityValidationError(f"{field_name} has an invalid value")
+        return value
+
+    try:
+        return make_observability_event(
+            message_code,
+            request_id=optional_text("request_id"),
+            deploy_version=optional_text("deploy_version"),
+            command_run_id=optional_text("command_run_id"),
+            lifecycle_id=optional_text("lifecycle_id"),
+            lifecycle_status=optional_text("lifecycle_status"),
+            lifecycle_event=optional_text("lifecycle_event"),
+        )
+    except ObservabilityValidationError:
+        return None
+
+
+def _timestamp_text(timestamp: datetime) -> str:
+    if type(timestamp) is not datetime or timestamp.tzinfo is None:
+        raise ObservabilityValidationError("timestamp must be timezone-aware")
+    return timestamp.astimezone(UTC).isoformat(timespec="milliseconds").replace("+00:00", "Z")
+
+
+def format_observability_event(
+    event: dict[str, str],
+    *,
+    timestamp: datetime,
+    severity: Severity,
+) -> str:
+    """Serialize one validated event as deterministic, single-line JSON."""
+
+    if severity not in _LEVEL_BY_SEVERITY:
+        raise ObservabilityValidationError("severity has an invalid value")
+    normalized = _normalize_event(event)
+    if normalized is None:
+        raise ObservabilityValidationError("event has an invalid value")
+
+    payload: dict[str, str] = {
+        "timestamp": _timestamp_text(timestamp),
+        "severity": severity,
+        "message_code": normalized["message_code"],
+    }
+    for key in OBSERVABILITY_KEYS[3:]:
+        value = normalized.get(key)
+        if value is not None:
+            payload[key] = value
+    return json.dumps(payload, ensure_ascii=True, separators=(",", ":"))
+
+
+class ObservabilityAllowlistFilter(logging.Filter):
+    """Admit only records containing a valid bounded event envelope."""
+
+    def filter(self, record: logging.LogRecord) -> bool:
+        normalized = _normalize_event(record.__dict__.get(_EVENT_ATTRIBUTE))
+        if normalized is None:
+            return False
+        setattr(record, _NORMALIZED_ATTRIBUTE, normalized)
+        return True
+
+
+class StructuredJsonFormatter(logging.Formatter):
+    """Emit one allowlisted JSON object and ignore normal log message content."""
+
+    def format(self, record: logging.LogRecord) -> str:
+        raw_normalized = record.__dict__.get(_NORMALIZED_ATTRIBUTE)
+        normalized = _normalize_event(raw_normalized)
+        if normalized is None:
+            normalized = _normalize_event(record.__dict__.get(_EVENT_ATTRIBUTE))
+        if normalized is None:
+            normalized = make_observability_event(_INVALID_MESSAGE_CODE)
+
+        severity = _SEVERITY_BY_LEVEL.get(record.levelno, "WARNING")
+        try:
+            timestamp = datetime.fromtimestamp(record.created, tz=UTC)
+        except OverflowError, OSError, ValueError:
+            timestamp = datetime.fromtimestamp(0, tz=UTC)
+        return format_observability_event(
+            normalized,
+            timestamp=timestamp,
+            severity=severity,
+        )
+
+
+def log_event(
+    logger: logging.Logger,
+    severity: Severity,
+    message_code: str,
+    *,
+    request_id: str | UUID | None = None,
+    deploy_version: str | None = None,
+    command_run_id: str | UUID | None = None,
+    lifecycle_id: str | UUID | None = None,
+    lifecycle_status: str | None = None,
+    lifecycle_event: str | None = None,
+) -> None:
+    """Log one structured event without accepting message text or arbitrary extras."""
+
+    if severity not in _LEVEL_BY_SEVERITY:
+        raise ObservabilityValidationError("severity has an invalid value")
+    event = make_observability_event(
+        message_code,
+        request_id=request_id if request_id is not None else current_request_id(),
+        deploy_version=deploy_version,
+        command_run_id=command_run_id,
+        lifecycle_id=lifecycle_id,
+        lifecycle_status=lifecycle_status,
+        lifecycle_event=lifecycle_event,
+    )
+    logger.log(
+        _LEVEL_BY_SEVERITY[severity],
+        "structured-event",
+        extra={_EVENT_ATTRIBUTE: event},
+    )
+
+
+def current_request_id() -> str | None:
+    """Return the current request correlation UUID, if running inside middleware."""
+
+    return _request_id_context.get()
+
+
+def _request_id_from_header(value: object) -> str | None:
+    if type(value) is not str or _UUID_PATTERN.fullmatch(value) is None:
+        return None
+    try:
+        return _canonical_uuid(value, field_name="request_id")
+    except ObservabilityValidationError:
+        return None
+
+
+class RequestIdMiddleware:
+    """Propagate a valid correlation UUID or generate one at the HTTP boundary."""
+
+    sync_capable = True
+    async_capable = False
+
+    def __init__(self, get_response: Callable[[HttpRequest], HttpResponseBase]) -> None:
+        self.get_response = get_response
+
+    def __call__(self, request: HttpRequest) -> HttpResponseBase:
+        request_id = _request_id_from_header(request.headers.get(_REQUEST_ID_HEADER))
+        if request_id is None:
+            request_id = str(uuid4())
+
+        token: Token[str | None] = _request_id_context.set(request_id)
+        request.__dict__["request_id"] = request_id
+        try:
+            response = self.get_response(request)
+            response.headers[_REQUEST_ID_HEADER] = request_id
+            return response
+        finally:
+            _request_id_context.reset(token)
diff --git a/grocery/tests/test_observability.py b/grocery/tests/test_observability.py
new file mode 100644
index 0000000..8a4d54e
--- /dev/null
+++ b/grocery/tests/test_observability.py
@@ -0,0 +1,229 @@
+import io
+import json
+import logging
+from datetime import UTC, datetime
+from uuid import UUID
+
+import pytest
+from django.http import HttpResponse
+from django.test import RequestFactory
+
+from grocery.observability import (
+    OBSERVABILITY_KEYS,
+    ObservabilityAllowlistFilter,
+    ObservabilityValidationError,
+    RequestIdMiddleware,
+    StructuredJsonFormatter,
+    current_request_id,
+    format_observability_event,
+    log_event,
+    make_observability_event,
+)
+
+REQUEST_ID = "018f47d2-f9b2-7cc4-8ddf-fce39c000001"
+COMMAND_RUN_ID = "018f47d2-f9b2-7cc4-8ddf-fce39c000002"
+LIFECYCLE_ID = "018f47d2-f9b2-7cc4-8ddf-fce39c000003"
+DEPLOY_VERSION = "0123456789abcdef0123456789abcdef01234567"
+
+
+class MustNotBeStringified:
+    def __str__(self) -> str:
+        raise AssertionError("an arbitrary object was stringified")
+
+
+def test_event_formatter_has_deterministic_allowlisted_schema() -> None:
+    event = make_observability_event(
+        "publication.activation.completed",
+        request_id=REQUEST_ID,
+        deploy_version=DEPLOY_VERSION,
+        command_run_id=COMMAND_RUN_ID,
+        lifecycle_id=LIFECYCLE_ID,
+        lifecycle_status="ACTIVE",
+        lifecycle_event="PUBLICATION_ACTIVATED",
+    )
+
+    line = format_observability_event(
+        event,
+        timestamp=datetime(2026, 8, 30, 1, 2, 3, 456789, tzinfo=UTC),
+        severity="INFO",
+    )
+
+    assert line == (
+        '{"timestamp":"2026-08-30T01:02:03.456Z","severity":"INFO",'
+        '"message_code":"publication.activation.completed",'
+        f'"request_id":"{REQUEST_ID}","deploy_version":"{DEPLOY_VERSION}",'
+        f'"command_run_id":"{COMMAND_RUN_ID}","lifecycle_id":"{LIFECYCLE_ID}",'
+        '"lifecycle_status":"ACTIVE","lifecycle_event":"PUBLICATION_ACTIVATED"}'
+    )
+    assert tuple(json.loads(line)) == OBSERVABILITY_KEYS
+    assert "\n" not in line
+    assert "\r" not in line
+
+
+@pytest.mark.parametrize(
+    ("field", "value"),
+    [
+        ("message_code", "source.fetch\ncredential"),
+        ("message_code", "https://provider.example/path?query=secret"),
+        ("request_id", "search=한우"),
+        ("deploy_version", "provider-dataset-15156063"),
+        ("lifecycle_status", "ACTIVE\r\ncredential"),
+        ("lifecycle_event", "RAW_BODY={secret}"),
+    ],
+)
+def test_event_helper_rejects_unbounded_or_injectable_values(field: str, value: str) -> None:
+    kwargs = {field: value}
+    if field == "message_code":
+        with pytest.raises(ObservabilityValidationError):
+            make_observability_event(value)
+    else:
+        with pytest.raises(ObservabilityValidationError):
+            make_observability_event("source.fetch.completed", **kwargs)
+
+
+def test_filter_and_formatter_drop_every_unknown_sensitive_field() -> None:
+    raw_event: dict[str, object] = {
+        "message_code": "source.fetch.completed",
+        "lifecycle_status": "SUCCEEDED",
+        "query_string": "serviceKey=secret-value",
+        "search_term": "민감한 검색어",
+        "url": "https://provider.example/path?serviceKey=secret-value",
+        "raw_body": '{"credential":"secret-value"}',
+        "credentials": "secret-value",
+        "secret_value": "secret-value",
+        "provider_identifier": "provider-dataset-15156063",
+        "exception": MustNotBeStringified(),
+    }
+    record = logging.LogRecord(
+        name="test.observability",
+        level=logging.INFO,
+        pathname=__file__,
+        lineno=1,
+        msg=MustNotBeStringified(),
+        args=(),
+        exc_info=None,
+    )
+    record.created = 0
+    record.__dict__["_grocery_observability_event"] = raw_event
+
+    assert ObservabilityAllowlistFilter().filter(record)
+    line = StructuredJsonFormatter().format(record)
+
+    assert json.loads(line) == {
+        "timestamp": "1970-01-01T00:00:00.000Z",
+        "severity": "INFO",
+        "message_code": "source.fetch.completed",
+        "lifecycle_status": "SUCCEEDED",
+    }
+    for forbidden in (
+        "serviceKey",
+        "secret-value",
+        "민감한 검색어",
+        "https://",
+        "provider-dataset",
+        "raw_body",
+        "query_string",
+    ):
+        assert forbidden not in line
+
+
+def test_filter_rejects_invalid_event_and_formatter_uses_safe_fallback() -> None:
+    record = logging.LogRecord(
+        name="test.observability",
+        level=logging.ERROR,
+        pathname=__file__,
+        lineno=1,
+        msg="https://provider.example/?credential=secret-value\nforged-line",
+        args=(),
+        exc_info=None,
+    )
+    record.created = 0
+    record.__dict__["_grocery_observability_event"] = {
+        "message_code": "invalid\nsecret-value",
+        "raw_body": "secret-value",
+    }
+
+    assert not ObservabilityAllowlistFilter().filter(record)
+    line = StructuredJsonFormatter().format(record)
+
+    assert json.loads(line) == {
+        "timestamp": "1970-01-01T00:00:00.000Z",
+        "severity": "ERROR",
+        "message_code": "observability.invalid_event",
+    }
+    assert "secret-value" not in line
+    assert "forged-line" not in line
+    assert "\n" not in line
+
+
+def test_log_event_emits_one_line_without_normal_log_message_data() -> None:
+    output = io.StringIO()
+    handler = logging.StreamHandler(output)
+    handler.addFilter(ObservabilityAllowlistFilter())
+    handler.setFormatter(StructuredJsonFormatter())
+    logger = logging.getLogger("grocery.tests.structured-event")
+    logger.handlers = [handler]
+    logger.propagate = False
+    logger.setLevel(logging.INFO)
+    try:
+        log_event(
+            logger,
+            "INFO",
+            "review.decision.recorded",
+            command_run_id=UUID(COMMAND_RUN_ID),
+            lifecycle_id=UUID(LIFECYCLE_ID),
+            lifecycle_status="APPROVED",
+            lifecycle_event="REVIEW_RECORDED",
+        )
+    finally:
+        logger.handlers = []
+        logger.propagate = True
+
+    lines = output.getvalue().splitlines()
+    assert len(lines) == 1
+    payload = json.loads(lines[0])
+    assert payload["message_code"] == "review.decision.recorded"
+    assert payload["command_run_id"] == COMMAND_RUN_ID
+    assert payload["lifecycle_id"] == LIFECYCLE_ID
+    assert set(payload).issubset(OBSERVABILITY_KEYS)
+
+
+def test_request_id_middleware_propagates_valid_uuid_and_clears_context() -> None:
+    observed: list[str | None] = []
+
+    def get_response(request: object) -> HttpResponse:
+        del request
+        observed.append(current_request_id())
+        return HttpResponse(status=204)
+
+    request = RequestFactory().get("/health")
+    request.META["HTTP_X_REQUEST_ID"] = REQUEST_ID
+    response = RequestIdMiddleware(get_response)(request)
+
+    assert observed == [REQUEST_ID]
+    assert request.__dict__["request_id"] == REQUEST_ID
+    assert response.headers["X-Request-ID"] == REQUEST_ID
+    assert current_request_id() is None
+
+
+@pytest.mark.parametrize(
+    "untrusted_header",
+    [
+        "not-a-uuid",
+        "https://provider.example/?serviceKey=secret-value",
+        "018f47d2-f9b2-7cc4-8ddf-fce39c000001\r\nX-Forged: true",
+    ],
+)
+def test_request_id_middleware_replaces_untrusted_header(untrusted_header: str) -> None:
+    request = RequestFactory().get("/health")
+    request.META["HTTP_X_REQUEST_ID"] = untrusted_header
+
+    response = RequestIdMiddleware(lambda unused: HttpResponse(status=204))(request)
+    generated = response.headers["X-Request-ID"]
+
+    assert generated != untrusted_header
+    assert str(UUID(generated)) == generated
+    assert request.__dict__["request_id"] == generated
+    assert "secret-value" not in generated
+    assert "\r" not in generated
+    assert "\n" not in generated


## `feat(ops): wire safe request logging`

diff --git a/.env.example b/.env.example
index ecb08bd..bf6881a 100644
--- a/.env.example
+++ b/.env.example
@@ -4,4 +4,4 @@ DJANGO_SECRET_KEY=replace-in-secret-store
 DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1
 DATABASE_URL=postgresql://grocery:local-grocery-only@127.0.0.1:55434/grocery
 KAMIS_API_KEY=
-DEPLOY_VERSION=dev
+DEPLOY_VERSION=0000000
diff --git a/config/settings.py b/config/settings.py
index c25a074..7f2652b 100644
--- a/config/settings.py
+++ b/config/settings.py
@@ -58,6 +58,7 @@ INSTALLED_APPS = [
 
 MIDDLEWARE = [
     "django.middleware.security.SecurityMiddleware",
+    "grocery.observability.RequestIdMiddleware",
     "django.contrib.sessions.middleware.SessionMiddleware",
     "django.middleware.common.CommonMiddleware",
     "django.middleware.csrf.CsrfViewMiddleware",
@@ -112,3 +113,37 @@ SECURE_HSTS_PRELOAD = not DEBUG
 SECURE_CONTENT_TYPE_NOSNIFF = True
 SECURE_REFERRER_POLICY = "same-origin"
 X_FRAME_OPTIONS = "DENY"
+
+DEPLOY_VERSION = os.environ.get("DEPLOY_VERSION", "0000000")
+
+LOGGING = {
+    "version": 1,
+    "disable_existing_loggers": False,
+    "filters": {
+        "observability_allowlist": {
+            "()": "grocery.observability.ObservabilityAllowlistFilter",
+        }
+    },
+    "formatters": {
+        "structured_json": {
+            "()": "grocery.observability.StructuredJsonFormatter",
+        }
+    },
+    "handlers": {
+        "null": {"class": "logging.NullHandler"},
+        "structured_console": {
+            "class": "logging.StreamHandler",
+            "filters": ["observability_allowlist"],
+            "formatter": "structured_json",
+        },
+    },
+    "loggers": {
+        "django.request": {"handlers": ["null"], "propagate": False},
+        "django.server": {"handlers": ["null"], "propagate": False},
+        "grocery.audit": {
+            "handlers": ["structured_console"],
+            "level": "INFO",
+            "propagate": False,
+        },
+    },
+}
diff --git a/grocery/tests/test_logging_settings.py b/grocery/tests/test_logging_settings.py
new file mode 100644
index 0000000..5e99bda
--- /dev/null
+++ b/grocery/tests/test_logging_settings.py
@@ -0,0 +1,31 @@
+from typing import Any, cast
+
+from django.conf import settings
+from django.test import Client, SimpleTestCase
+from django.urls import reverse
+
+
+class LoggingSettingsTests(SimpleTestCase):
+    def test_request_id_middleware_is_active(self) -> None:
+        response = Client().get(reverse("admin:login"))
+
+        self.assertEqual(response.status_code, 200)
+        self.assertIn("X-Request-ID", response.headers)
+
+    def test_query_bearing_framework_access_logs_are_disabled(self) -> None:
+        logging_config = cast(dict[str, Any], settings.LOGGING)
+        loggers = logging_config["loggers"]
+
+        self.assertEqual(loggers["django.request"]["handlers"], ["null"])
+        self.assertEqual(loggers["django.server"]["handlers"], ["null"])
+        self.assertFalse(loggers["django.request"]["propagate"])
+        self.assertFalse(loggers["django.server"]["propagate"])
+
+    def test_audit_logger_uses_only_structured_allowlisted_output(self) -> None:
+        logging_config = cast(dict[str, Any], settings.LOGGING)
+        logger = logging_config["loggers"]["grocery.audit"]
+        handler = logging_config["handlers"]["structured_console"]
+
+        self.assertEqual(logger["handlers"], ["structured_console"])
+        self.assertEqual(handler["filters"], ["observability_allowlist"])
+        self.assertEqual(handler["formatter"], "structured_json")


