## `feat(operations): expose bounded health signals`

diff --git a/operations/management/commands/check_freshness.py b/operations/management/commands/check_freshness.py
new file mode 100644
index 0000000..30f47c7
--- /dev/null
+++ b/operations/management/commands/check_freshness.py
@@ -0,0 +1,34 @@
+from django.core.management.base import BaseCommand, CommandError
+
+from operations.views import publication_freshness_snapshot
+from public_web.results import CARD_READY
+
+
+class Command(BaseCommand):
+    help = "Exit nonzero unless both independent publications are fresh."
+
+    def handle(self, *args, **options):
+        try:
+            states = publication_freshness_snapshot()
+        except Exception:
+            states = {"entry": "server-error", "warning": "server-error"}
+        entry = states.get("entry", "server-error")
+        warning = states.get("warning", "server-error")
+        known_states = {
+            "ready",
+            "empty",
+            "unavailable",
+            "stale",
+            "server-error",
+        }
+        if entry not in known_states or warning not in known_states:
+            entry = "server-error"
+            warning = "server-error"
+        if entry == CARD_READY and warning == CARD_READY:
+            self.stdout.write(
+                "freshness=OK entry=ready warning=ready"
+            )
+            return
+        raise CommandError(
+            f"freshness=ALERT entry={entry} warning={warning}"
+        ) from None
diff --git a/operations/middleware.py b/operations/middleware.py
new file mode 100644
index 0000000..5a2738e
--- /dev/null
+++ b/operations/middleware.py
@@ -0,0 +1,68 @@
+"""Fixed-key request telemetry without request identity or input material."""
+
+from __future__ import annotations
+
+import json
+import logging
+import time
+import uuid
+
+
+_LOGGER = logging.getLogger("travel_readiness.request")
+_METHODS = {"GET", "POST", "HEAD"}
+
+
+def _duration_bucket(elapsed_ns: int) -> str:
+    elapsed_ms = max(elapsed_ns, 0) / 1_000_000
+    if elapsed_ms < 50:
+        return "LT_50_MS"
+    if elapsed_ms < 250:
+        return "LT_250_MS"
+    if elapsed_ms < 1_000:
+        return "LT_1000_MS"
+    return "GTE_1000_MS"
+
+
+class RequestTelemetryMiddleware:
+    def __init__(self, get_response):
+        self.get_response = get_response
+
+    def __call__(self, request):
+        correlation_id = str(uuid.uuid4())
+        request.correlation_id = correlation_id
+        started_ns = time.monotonic_ns()
+        response = self.get_response(request)
+        elapsed_ns = time.monotonic_ns() - started_ns
+        match = getattr(request, "resolver_match", None)
+        route = getattr(match, "route", None)
+        if type(route) is not str or route not in {
+            "",
+            "results/",
+            "healthz",
+            "readyz",
+            "freshnessz",
+        }:
+            route = "unmatched"
+        method = getattr(request, "method", None)
+        method = (
+            method
+            if type(method) is str and method in _METHODS
+            else "OTHER"
+        )
+        status = response.status_code
+        if type(status) is not int or not 100 <= status <= 599:
+            status = 500
+        payload = {
+            "event": "request",
+            "route": route,
+            "method": method,
+            "status": status,
+            "duration_bucket": _duration_bucket(elapsed_ns),
+            "correlation_id": correlation_id,
+            "redacted": True,
+        }
+        try:
+            _LOGGER.info(json.dumps(payload, separators=(",", ":")))
+        except Exception:
+            pass
+        return response
diff --git a/operations/tests/test_health.py b/operations/tests/test_health.py
index 08bbd7e..435a214 100644
--- a/operations/tests/test_health.py
+++ b/operations/tests/test_health.py
@@ -1,4 +1,8 @@
-from django.test import SimpleTestCase
+import json
+from unittest.mock import patch
+
+from django.db import DatabaseError
+from django.test import SimpleTestCase, TestCase
 
 
 class HealthTests(SimpleTestCase):
@@ -7,12 +11,59 @@ class HealthTests(SimpleTestCase):
 
         response = self.client.get(f"/healthz?destination={marker}", secure=True)
 
-        self.assertEqual(response.status_code, 200)
-        self.assertEqual(response.content, b"ok\n")
-        self.assertNotContains(response, marker)
+        self.assertEqual(response.status_code, 303)
+        self.assertNotIn(marker, response.content.decode("utf-8"))
+        self.assertEqual(response.headers["Location"], "/healthz")
         self.assertEqual(response.headers["Cache-Control"], "no-store")
         self.assertEqual(response.headers["X-Content-Type-Options"], "nosniff")
         self.assertEqual(response.headers["X-Frame-Options"], "DENY")
         self.assertEqual(response.headers["Referrer-Policy"], "no-referrer")
         self.assertIn("max-age=31536000", response.headers["Strict-Transport-Security"])
         self.assertFalse(response.cookies)
+
+    def test_health_is_get_only_and_queryless(self):
+        response = self.client.get("/healthz", secure=True)
+        self.assertEqual(response.status_code, 200)
+        self.assertEqual(response.content, b"ok\n")
+        rejected = self.client.post("/healthz", secure=True)
+        self.assertEqual(rejected.status_code, 405)
+        self.assertEqual(rejected.headers["Cache-Control"], "no-store")
+
+
+class ReadinessTests(TestCase):
+    def test_readyz_reads_only_the_database(self):
+        response = self.client.get("/readyz", secure=True)
+        self.assertEqual(response.status_code, 200)
+        self.assertEqual(json.loads(response.content), {"status": "ready"})
+        self.assertEqual(response.headers["Cache-Control"], "no-store")
+
+    def test_readyz_closes_database_errors_to_a_fixed_shape(self):
+        with patch(
+            "operations.views.connection.cursor",
+            side_effect=DatabaseError("private-database-detail"),
+        ):
+            response = self.client.get("/readyz", secure=True)
+        self.assertEqual(response.status_code, 503)
+        self.assertEqual(
+            json.loads(response.content),
+            {"status": "unavailable"},
+        )
+        self.assertNotIn(
+            "private-database-detail",
+            response.content.decode("utf-8"),
+        )
+
+    def test_freshness_endpoint_has_only_module_states(self):
+        with patch(
+            "operations.views.publication_freshness_snapshot",
+            return_value={"entry": "ready", "warning": "stale"},
+        ):
+            response = self.client.get("/freshnessz", secure=True)
+        self.assertEqual(response.status_code, 200)
+        self.assertEqual(
+            json.loads(response.content),
+            {
+                "status": "alert",
+                "modules": {"entry": "ready", "warning": "stale"},
+            },
+        )
diff --git a/operations/tests/test_observability.py b/operations/tests/test_observability.py
new file mode 100644
index 0000000..d263458
--- /dev/null
+++ b/operations/tests/test_observability.py
@@ -0,0 +1,154 @@
+import io
+import json
+import logging
+import uuid
+from unittest.mock import patch
+
+from django.core.management import call_command
+from django.core.management.base import CommandError
+from django.test import RequestFactory, SimpleTestCase, override_settings
+
+from operations.middleware import RequestTelemetryMiddleware
+from operations.views import publication_freshness_snapshot
+
+
+@override_settings(SECURE_SSL_REDIRECT=False, ALLOWED_HOSTS=["testserver"])
+class RequestTelemetryTests(SimpleTestCase):
+    def test_event_has_only_fixed_keys_and_no_request_material(self):
+        marker = "synthetic-destination-date-ip-cookie-marker"
+        request = RequestFactory().get(
+            f"/unknown?destination={marker}",
+            HTTP_COOKIE=marker,
+            REMOTE_ADDR=marker,
+        )
+        request.resolver_match = None
+        middleware = RequestTelemetryMiddleware(
+            lambda incoming: type(
+                "Response",
+                (),
+                {"status_code": 404},
+            )()
+        )
+
+        with self.assertLogs("travel_readiness.request", level="INFO") as logs:
+            middleware(request)
+
+        payload = json.loads(logs.records[0].getMessage())
+        self.assertEqual(
+            set(payload),
+            {
+                "event",
+                "route",
+                "method",
+                "status",
+                "duration_bucket",
+                "correlation_id",
+                "redacted",
+            },
+        )
+        self.assertEqual(payload["route"], "unmatched")
+        self.assertEqual(payload["status"], 404)
+        self.assertTrue(payload["redacted"])
+        uuid.UUID(payload["correlation_id"])
+        self.assertNotIn(marker, logs.output[0])
+
+    def test_logging_backend_failure_does_not_change_response(self):
+        request = RequestFactory().get("/healthz")
+        response = type("Response", (), {"status_code": 200})()
+        middleware = RequestTelemetryMiddleware(lambda incoming: response)
+        with patch.object(logging.Logger, "info", side_effect=RuntimeError):
+            self.assertIs(middleware(request), response)
+
+    def test_unknown_method_is_bounded(self):
+        request = RequestFactory().get("/healthz")
+        request.method = "private-method-marker"
+        response = type("Response", (), {"status_code": 405})()
+        middleware = RequestTelemetryMiddleware(lambda incoming: response)
+
+        with self.assertLogs("travel_readiness.request", level="INFO") as logs:
+            middleware(request)
+
+        payload = json.loads(logs.records[0].getMessage())
+        self.assertEqual(payload["method"], "OTHER")
+        self.assertNotIn("private-method-marker", logs.output[0])
+
+    def test_installed_middleware_covers_query_stripping(self):
+        marker = "private-query-cookie-ip-marker"
+        with self.assertLogs("travel_readiness.request", level="INFO") as logs:
+            response = self.client.get(
+                f"/healthz?destination={marker}",
+                HTTP_COOKIE=marker,
+                REMOTE_ADDR=marker,
+            )
+
+        self.assertEqual(response.status_code, 303)
+        request_events = [
+            json.loads(record.getMessage())
+            for record in logs.records
+            if json.loads(record.getMessage()).get("event") == "request"
+        ]
+        self.assertEqual(len(request_events), 1)
+        self.assertEqual(request_events[0]["route"], "unmatched")
+        self.assertNotIn(marker, "\n".join(logs.output))
+
+    def test_installed_middleware_never_logs_body_or_credentials(self):
+        marker = "private-body-authorization-source-marker"
+        with self.assertLogs("travel_readiness.request", level="INFO") as logs:
+            response = self.client.post(
+                "/healthz",
+                data={"destination": marker, "raw": marker},
+                HTTP_AUTHORIZATION=f"Bearer {marker}",
+            )
+
+        self.assertEqual(response.status_code, 405)
+        self.assertNotIn(marker, "\n".join(logs.output))
+
+
+class FreshnessIsolationTests(SimpleTestCase):
+    def test_one_loader_failure_preserves_the_other_module_state(self):
+        with (
+            patch(
+                "public_web.results._load_entry_card",
+                side_effect=RuntimeError("private-entry-detail"),
+            ),
+            patch(
+                "public_web.results._load_warning_card",
+                return_value={"state": "ready"},
+            ),
+        ):
+            states = publication_freshness_snapshot()
+
+        self.assertEqual(
+            states,
+            {"entry": "server-error", "warning": "ready"},
+        )
+
+
+class FreshnessCommandBoundaryTests(SimpleTestCase):
+    def test_ready_output_is_fixed(self):
+        output = io.StringIO()
+        with patch(
+            "operations.management.commands.check_freshness."
+            "publication_freshness_snapshot",
+            return_value={"entry": "ready", "warning": "ready"},
+        ):
+            call_command("check_freshness", stdout=output)
+        self.assertEqual(
+            output.getvalue(),
+            "freshness=OK entry=ready warning=ready\n",
+        )
+
+    def test_alert_output_is_fixed_and_nonzero(self):
+        marker = "private-upstream-detail"
+        with patch(
+            "operations.management.commands.check_freshness."
+            "publication_freshness_snapshot",
+            return_value={"entry": "stale", "warning": "unavailable"},
+        ):
+            with self.assertRaises(CommandError) as raised:
+                call_command("check_freshness")
+        self.assertEqual(
+            str(raised.exception),
+            "freshness=ALERT entry=stale warning=unavailable",
+        )
+        self.assertNotIn(marker, str(raised.exception))
diff --git a/operations/urls.py b/operations/urls.py
index 632e590..31999c8 100644
--- a/operations/urls.py
+++ b/operations/urls.py
@@ -1,5 +1,9 @@
 from django.urls import path
 
-from .views import healthz
+from .views import freshnessz, healthz, readyz
 
-urlpatterns = [path("healthz", healthz, name="healthz")]
+urlpatterns = [
+    path("healthz", healthz, name="healthz"),
+    path("readyz", readyz, name="readyz"),
+    path("freshnessz", freshnessz, name="freshnessz"),
+]
diff --git a/operations/views.py b/operations/views.py
index 5393e62..c1e8fa2 100644
--- a/operations/views.py
+++ b/operations/views.py
@@ -1,9 +1,86 @@
-from django.http import HttpResponse
+import json
 
+from django.db import DatabaseError, connection
+from django.http import HttpRequest, HttpResponse
+from django.views.decorators.http import require_GET
 
-def healthz(request) -> HttpResponse:
+from public_web.results import (
+    CARD_EMPTY,
+    CARD_READY,
+    CARD_SERVER_ERROR,
+    CARD_STALE,
+    CARD_UNAVAILABLE,
+    load_publication_cards,
+)
+
+
+def _json_response(payload: dict, *, status: int) -> HttpResponse:
+    return HttpResponse(
+        json.dumps(payload, separators=(",", ":")),
+        status=status,
+        content_type="application/json; charset=utf-8",
+        headers={"Cache-Control": "no-store"},
+    )
+
+
+@require_GET
+def healthz(request: HttpRequest) -> HttpResponse:
     return HttpResponse(
         "ok\n",
         content_type="text/plain; charset=utf-8",
         headers={"Cache-Control": "no-store"},
     )
+
+
+@require_GET
+def readyz(request: HttpRequest) -> HttpResponse:
+    try:
+        with connection.cursor() as cursor:
+            cursor.execute("SELECT 1")
+            row = cursor.fetchone()
+        if row != (1,):
+            raise DatabaseError("unexpected readiness result")
+    except Exception:
+        return _json_response({"status": "unavailable"}, status=503)
+    return _json_response({"status": "ready"}, status=200)
+
+
+def publication_freshness_snapshot() -> dict[str, str]:
+    try:
+        cards = load_publication_cards()
+        states = {
+            "entry": cards["entry"].get("state"),
+            "warning": cards["warning"].get("state"),
+        }
+    except Exception:
+        states = {
+            "entry": CARD_SERVER_ERROR,
+            "warning": CARD_SERVER_ERROR,
+        }
+    allowed_states = {
+        CARD_READY,
+        CARD_EMPTY,
+        CARD_UNAVAILABLE,
+        CARD_STALE,
+        CARD_SERVER_ERROR,
+    }
+    if any(type(value) is not str or value not in allowed_states for value in states.values()):
+        return {
+            "entry": CARD_SERVER_ERROR,
+            "warning": CARD_SERVER_ERROR,
+        }
+    return states
+
+
+@require_GET
+def freshnessz(request: HttpRequest) -> HttpResponse:
+    states = publication_freshness_snapshot()
+    aggregate = (
+        "ready"
+        if all(state == CARD_READY for state in states.values())
+        else "alert"
+    )
+    return _json_response(
+        {"status": aggregate, "modules": states},
+        status=200,
+    )
diff --git a/public_web/middleware.py b/public_web/middleware.py
index 132b31a..e2bbeda 100644
--- a/public_web/middleware.py
+++ b/public_web/middleware.py
@@ -7,6 +7,15 @@ _PUBLIC_CANONICAL_PATHS = {
     "/results": "/results/",
     "/results/": "/results/",
 }
+_OPERATIONS_CANONICAL_PATHS = {
+    "/healthz": "/healthz",
+    "/readyz": "/readyz",
+    "/freshnessz": "/freshnessz",
+}
+_CANONICAL_PATHS = {
+    **_PUBLIC_CANONICAL_PATHS,
+    **_OPERATIONS_CANONICAL_PATHS,
+}
 _CSP = (
     "default-src 'self'; base-uri 'none'; form-action 'self'; "
     "frame-ancestors 'none'; object-src 'none'"
@@ -41,7 +50,7 @@ class PublicPrivacyMiddleware:
         self.get_response = get_response
 
     def __call__(self, request: HttpRequest) -> HttpResponse:
-        canonical_path = _PUBLIC_CANONICAL_PATHS.get(request.path_info)
+        canonical_path = _CANONICAL_PATHS.get(request.path_info)
         if canonical_path is not None and request.META.get("QUERY_STRING"):
             return _protect(
                 HttpResponse(
@@ -54,7 +63,10 @@ class PublicPrivacyMiddleware:
         response = self.get_response(request)
         if canonical_path is None:
             return response
-        if response.status_code >= 500:
+        if (
+            request.path_info in _PUBLIC_CANONICAL_PATHS
+            and response.status_code >= 500
+        ):
             response = HttpResponse(
                 "일시적으로 요청을 처리할 수 없습니다.\n",
                 status=response.status_code,
diff --git a/public_web/results.py b/public_web/results.py
index bd815c9..db37b50 100644
--- a/public_web/results.py
+++ b/public_web/results.py
@@ -512,18 +512,26 @@ def _safe_card(
         return _state_card(module, CARD_SERVER_ERROR)
 
 
+def load_publication_cards() -> dict[str, dict[str, object]]:
+    """Return the two fixed, independent read models without external I/O."""
+
+    return {
+        "entry": _safe_card("entry", _load_entry_card),
+        "warning": _safe_card("warning", _load_warning_card),
+    }
+
+
 @require_GET
 def results(request: HttpRequest) -> HttpResponse:
     if request.GET:
         return _fixed_redirect("public_web:results")
-    entry_card = _safe_card("entry", _load_entry_card)
-    warning_card = _safe_card("warning", _load_warning_card)
+    cards = load_publication_cards()
     response = render(
         request,
         "public_web/results.html",
         {
-            "entry_card": entry_card,
-            "warning_card": warning_card,
+            "entry_card": cards["entry"],
+            "warning_card": cards["warning"],
         },
     )
     response.headers["Cache-Control"] = "no-store"
diff --git a/travel_readiness/settings.py b/travel_readiness/settings.py
index 919eb5a..ab9a3f5 100644
--- a/travel_readiness/settings.py
+++ b/travel_readiness/settings.py
@@ -164,6 +164,7 @@ AUTH_PASSWORD_VALIDATORS = [
 ]
 
 MIDDLEWARE = [
+    "operations.middleware.RequestTelemetryMiddleware",
     "public_web.middleware.PublicPrivacyMiddleware",
     "django.middleware.security.SecurityMiddleware",
     "django.contrib.sessions.middleware.SessionMiddleware",
@@ -245,12 +246,24 @@ LOGGING = {
     "version": 1,
     "disable_existing_loggers": False,
     "filters": {"redact_django": {"()": "django.utils.log.CallbackFilter", "callback": redact_django_record}},
-    "handlers": {"null": {"class": "logging.NullHandler"}, "redacted": {"class": "logging.StreamHandler", "filters": ["redact_django"]}},
+    "handlers": {
+        "null": {"class": "logging.NullHandler"},
+        "redacted": {
+            "class": "logging.StreamHandler",
+            "filters": ["redact_django"],
+        },
+        "request": {"class": "logging.StreamHandler"},
+    },
     "loggers": {
         "django.db.backends": {"handlers": ["null"], "propagate": False},
         "django.request": {"handlers": ["redacted"], "propagate": False},
         "django.security": {"handlers": ["redacted"], "propagate": False},
         "django.server": {"handlers": ["redacted"], "propagate": False},
+        "travel_readiness.request": {
+            "handlers": ["request"],
+            "level": "INFO",
+            "propagate": False,
+        },
     },
 }
 


