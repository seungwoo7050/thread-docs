## `fix(operations): expire stale source observations`

diff --git a/public_web/results.py b/public_web/results.py
index db37b50..1dceeac 100644
--- a/public_web/results.py
+++ b/public_web/results.py
@@ -1,13 +1,14 @@
 from __future__ import annotations
 
 from dataclasses import dataclass
-from datetime import UTC, date, datetime
+from datetime import UTC, date, datetime, timedelta
 from typing import Callable
 
 from django.db.models import Exists, OuterRef
 from django.http import HttpRequest, HttpResponse
 from django.shortcuts import render
 from django.urls import reverse
+from django.utils import timezone
 from django.views.decorators.http import require_GET
 
 from entry_requirements.ingestion import (
@@ -68,6 +69,7 @@ class _FreshnessContract:
     decided_by: str
     decision_basis: str
     provider_code: str
+    max_success_age: timedelta
 
 
 _FRESHNESS_CONTRACTS = {
@@ -84,6 +86,7 @@ _FRESHNESS_CONTRACTS = {
         decided_by=ENTRY_DECIDED_BY,
         decision_basis=ENTRY_DECISION_BASIS,
         provider_code="",
+        max_success_age=timedelta(hours=36),
     ),
     "warning": _FreshnessContract(
         source_code=WARNING_SOURCE_CODE,
@@ -98,6 +101,7 @@ _FRESHNESS_CONTRACTS = {
         decided_by=WARNING_DECIDED_BY,
         decision_basis=WARNING_DECISION_BASIS,
         provider_code=FetchAttempt.ProviderCode.MOFA_SUCCESS_0,
+        max_success_age=timedelta(hours=8),
     ),
 }
 
@@ -150,6 +154,30 @@ def _utc_minute(value: datetime | None) -> str | None:
     return value.astimezone(UTC).strftime("%Y-%m-%d %H:%M UTC")
 
 
+def _aware_utc(value: object) -> datetime | None:
+    try:
+        if not isinstance(value, datetime) or timezone.is_naive(value):
+            return None
+        return value.astimezone(UTC)
+    except (AttributeError, OverflowError, TypeError, ValueError):
+        return None
+
+
+def _matching_success_is_current(
+    completed_at: object,
+    *,
+    max_age: timedelta,
+) -> bool:
+    """Fail closed unless a successful observation has a valid current age."""
+
+    observation_time = _aware_utc(completed_at)
+    current_time = _aware_utc(timezone.now())
+    if observation_time is None or current_time is None:
+        return False
+    age = current_time - observation_time
+    return timedelta(0) <= age < max_age
+
+
 def _freshness(
     *,
     contract: _FreshnessContract,
@@ -291,8 +319,12 @@ def _freshness(
         or not latest_exact
         or last_matching_success is None
         or latest["id"] != last_matching_success["id"]
+        or not _matching_success_is_current(
+            last_matching_success["completed_at"],
+            max_age=contract.max_success_age,
+        )
     )
-    checked_at = (
+    checked_at = _aware_utc(
         None
         if last_matching_success is None
         else last_matching_success["completed_at"]
diff --git a/public_web/tests/test_results.py b/public_web/tests/test_results.py
index 0624454..7ac7f0f 100644
--- a/public_web/tests/test_results.py
+++ b/public_web/tests/test_results.py
@@ -1,5 +1,5 @@
 import uuid
-from datetime import UTC, timedelta
+from datetime import UTC, datetime, timedelta
 from unittest.mock import patch
 
 from django.contrib.auth import get_user_model
@@ -21,6 +21,7 @@ from reviews.publication import (
     rollback_publication,
 )
 from reviews.tests.test_publication import PublicationFixtureMixin
+from public_web.results import _matching_success_is_current
 from sources.models import (
     FetchAttempt,
     SourceConfiguration,
@@ -167,6 +168,120 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
         ):
             self.assertNotIn(sensitive_name, body)
 
+    def test_entry_matching_success_age_boundaries_fail_closed(self):
+        entry = self.publish_entry(period="90일")
+        completed_at = (
+            entry.parse_run.artifact.first_successful_attempt.completed_at
+        )
+        cases = (
+            ("just-before", timedelta(hours=36) - timedelta(seconds=1), "ready"),
+            ("boundary", timedelta(hours=36), "stale"),
+            ("old", timedelta(hours=37), "stale"),
+            ("future", -timedelta(seconds=1), "stale"),
+        )
+
+        for label, age, expected_state in cases:
+            with self.subTest(label=label):
+                with patch(
+                    "public_web.results.timezone.now",
+                    return_value=completed_at + age,
+                ):
+                    response = self.client.get(reverse("public_web:results"))
+
+                card = self.card_html(response, "entry-card")
+                self.assertIn(f'data-state="{expected_state}"', card)
+                self.assertIn("90일", card)
+                if expected_state == "stale":
+                    self.assertIn("재확인 필요", card)
+
+    def test_warning_matching_success_age_boundaries_fail_closed(self):
+        warning = self.publish_warning(scope_text="합성 검증 범위")
+        completed_at = (
+            warning.parse_run.artifact.first_successful_attempt.completed_at
+        )
+        cases = (
+            ("just-before", timedelta(hours=8) - timedelta(seconds=1), "ready"),
+            ("boundary", timedelta(hours=8), "stale"),
+            ("old", timedelta(hours=9), "stale"),
+            ("future", -timedelta(seconds=1), "stale"),
+        )
+
+        for label, age, expected_state in cases:
+            with self.subTest(label=label):
+                with patch(
+                    "public_web.results.timezone.now",
+                    return_value=completed_at + age,
+                ):
+                    response = self.client.get(reverse("public_web:results"))
+
+                card = self.card_html(response, "warning-card")
+                self.assertIn(f'data-state="{expected_state}"', card)
+                self.assertIn("합성 검증 범위", card)
+                if expected_state == "stale":
+                    self.assertIn("재확인 필요", card)
+
+    def test_warning_age_boundary_drives_freshness_endpoint_alert(self):
+        self.publish_entry()
+        warning = self.publish_warning()
+        completed_at = (
+            warning.parse_run.artifact.first_successful_attempt.completed_at
+        )
+
+        with patch(
+            "public_web.results.timezone.now",
+            return_value=(
+                completed_at
+                + timedelta(hours=8)
+                - timedelta(seconds=1)
+            ),
+        ):
+            just_before = self.client.get("/freshnessz")
+
+        self.assertEqual(just_before.status_code, 200)
+        self.assertEqual(
+            just_before.json(),
+            {
+                "status": "ready",
+                "modules": {"entry": "ready", "warning": "ready"},
+            },
+        )
+
+        with patch(
+            "public_web.results.timezone.now",
+            return_value=completed_at + timedelta(hours=8),
+        ):
+            boundary = self.client.get("/freshnessz")
+
+        self.assertEqual(boundary.status_code, 200)
+        self.assertEqual(
+            boundary.json(),
+            {
+                "status": "alert",
+                "modules": {"entry": "ready", "warning": "stale"},
+            },
+        )
+
+    def test_matching_success_invalid_timestamp_shapes_fail_closed(self):
+        current_time = datetime(2026, 8, 30, 12, 0, tzinfo=UTC)
+        invalid_values = (
+            None,
+            "2026-08-30T11:00:00Z",
+            datetime(2026, 8, 30, 11, 0),
+        )
+
+        with patch(
+            "public_web.results.timezone.now",
+            return_value=current_time,
+        ):
+            for completed_at in invalid_values:
+                with self.subTest(completed_at=completed_at):
+                    self.assertFalse(
+                        _matching_success_is_current(
+                            completed_at,
+                            max_age=timedelta(hours=36),
+                        )
+                    )
+
     def test_warning_text_is_template_escaped(self):
         marker = '<script data-marker="unsafe">alert(1)</script>'
         self.publish_warning(scope_text=marker)


## `feat(operations): expose release-aware health signals`

diff --git a/operations/checks.py b/operations/checks.py
index f002f55..8262f95 100644
--- a/operations/checks.py
+++ b/operations/checks.py
@@ -78,4 +78,11 @@ def deployment_settings_check(app_configs, **kwargs):
                 "operations.E006",
             )
         )
+    if settings.RELEASE_SHA == "unavailable":
+        errors.append(
+            _error(
+                "An exact release SHA is required for a deployed runtime.",
+                "operations.E007",
+            )
+        )
     return errors
diff --git a/operations/middleware.py b/operations/middleware.py
index 5a2738e..38379f5 100644
--- a/operations/middleware.py
+++ b/operations/middleware.py
@@ -7,6 +7,10 @@ import logging
 import time
 import uuid
 
+from django.conf import settings
+
+from .telemetry import bounded_release_sha, utc_timestamp
+
 
 _LOGGER = logging.getLogger("travel_readiness.request")
 _METHODS = {"GET", "POST", "HEAD"}
@@ -41,6 +45,7 @@ class RequestTelemetryMiddleware:
             "healthz",
             "readyz",
             "freshnessz",
+            "releasez",
         }:
             route = "unmatched"
         method = getattr(request, "method", None)
@@ -54,6 +59,8 @@ class RequestTelemetryMiddleware:
             status = 500
         payload = {
             "event": "request",
+            "timestamp": utc_timestamp(),
+            "release_sha": bounded_release_sha(settings.RELEASE_SHA),
             "route": route,
             "method": method,
             "status": status,
diff --git a/operations/tests/test_deployment_safety.py b/operations/tests/test_deployment_safety.py
index b9ce3b8..22c9c97 100644
--- a/operations/tests/test_deployment_safety.py
+++ b/operations/tests/test_deployment_safety.py
@@ -1,12 +1,14 @@
 import json
 import logging
 import os
+import re
 import subprocess
 import sys
 import uuid
 from pathlib import Path
 from types import SimpleNamespace
 
+from django.conf import settings
 from django.core.exceptions import PermissionDenied, SuspiciousOperation
 from django.http import HttpResponse
 from django.test import Client, SimpleTestCase, override_settings
@@ -69,6 +71,7 @@ class SettingsProcessMixin:
             ),
             "TRAVEL_READINESS_DB_PASSWORD": "unused-candidate-database-value",
             "TRAVEL_READINESS_ALLOWED_HOSTS": "candidate.invalid",
+            "TRAVEL_READINESS_RELEASE_SHA": "a" * 40,
         }
         environment.update(overrides)
         return environment
@@ -141,6 +144,24 @@ class DeploymentSettingsTests(SettingsProcessMixin, SimpleTestCase):
                     result.stderr,
                 )
 
+    def test_release_identity_rejects_invalid_values_without_reflection(self):
+        marker = "private-invalid-release-marker"
+        result = self.run_python(
+            "import travel_readiness.settings",
+            TRAVEL_READINESS_RELEASE_SHA=marker,
+        )
+        self.assertNotEqual(result.returncode, 0)
+        self.assertIn("Invalid TRAVEL_READINESS_RELEASE_SHA", result.stderr)
+        self.assertNotIn(marker, result.stderr)
+
+    def test_wsgi_runtime_requires_release_identity(self):
+        result = self.run_python(
+            "import travel_readiness.wsgi",
+            TRAVEL_READINESS_RELEASE_SHA="",
+        )
+        self.assertNotEqual(result.returncode, 0)
+        self.assertIn("exact release SHA is required", result.stderr)
+
     def test_safe_reserved_host_passes_every_deployment_check(self):
         result = subprocess.run(
             [
@@ -179,6 +200,25 @@ class DeploymentSettingsTests(SettingsProcessMixin, SimpleTestCase):
         self.assertNotEqual(result.returncode, 0)
         self.assertIn("operations.E002", result.stderr)
 
+    def test_deployment_check_requires_release_identity(self):
+        result = subprocess.run(
+            [
+                sys.executable,
+                str(Path(__file__).resolve().parents[2] / "manage.py"),
+                "check",
+                "--deploy",
+                "--fail-level",
+                "ERROR",
+            ],
+            cwd=Path(__file__).resolve().parents[2],
+            env=self.settings_environment(TRAVEL_READINESS_RELEASE_SHA=""),
+            capture_output=True,
+            text=True,
+            check=False,
+        )
+        self.assertNotEqual(result.returncode, 0)
+        self.assertIn("operations.E007", result.stderr)
+
     def test_strict_flags_limits_and_proxy_defaults_are_exact(self):
         result = self.run_python(
             "import json; from django.conf import settings; "
@@ -273,10 +313,18 @@ class DeploymentSettingsTests(SettingsProcessMixin, SimpleTestCase):
         self.assertTrue(redact_django_record(record))
         payload = json.loads(record.getMessage())
 
+        timestamp = payload.pop("timestamp")
+        self.assertIsNotNone(
+            re.fullmatch(
+                r"\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}\.\d{3}Z",
+                timestamp,
+            )
+        )
         self.assertEqual(
             payload,
             {
                 "event": "django",
+                "release_sha": settings.RELEASE_SHA,
                 "logger": "security",
                 "level": "OTHER",
                 "status": 500,
@@ -313,10 +361,17 @@ class DeploymentSettingsTests(SettingsProcessMixin, SimpleTestCase):
 
         redact_django_record(record)
 
+        payload = json.loads(record.getMessage())
+        timestamp = payload.pop("timestamp")
+        self.assertRegex(
+            timestamp,
+            r"\A\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}\.\d{3}Z\Z",
+        )
         self.assertEqual(
-            json.loads(record.getMessage()),
+            payload,
             {
                 "event": "django",
+                "release_sha": settings.RELEASE_SHA,
                 "logger": "request",
                 "level": "WARNING",
                 "status": 404,
diff --git a/operations/tests/test_health.py b/operations/tests/test_health.py
index 435a214..c8c913b 100644
--- a/operations/tests/test_health.py
+++ b/operations/tests/test_health.py
@@ -2,7 +2,7 @@ import json
 from unittest.mock import patch
 
 from django.db import DatabaseError
-from django.test import SimpleTestCase, TestCase
+from django.test import SimpleTestCase, TestCase, override_settings
 
 
 class HealthTests(SimpleTestCase):
@@ -29,6 +29,31 @@ class HealthTests(SimpleTestCase):
         self.assertEqual(rejected.status_code, 405)
         self.assertEqual(rejected.headers["Cache-Control"], "no-store")
 
+    @override_settings(RELEASE_SHA="a" * 40)
+    def test_release_endpoint_returns_only_the_exact_release_identity(self):
+        response = self.client.get("/releasez", secure=True)
+        self.assertEqual(response.status_code, 200)
+        self.assertEqual(json.loads(response.content), {"release_sha": "a" * 40})
+        self.assertEqual(response.headers["Cache-Control"], "no-store")
+
+    @override_settings(RELEASE_SHA="unavailable")
+    def test_release_endpoint_fails_closed_without_an_identity(self):
+        response = self.client.get("/releasez", secure=True)
+        self.assertEqual(response.status_code, 503)
+        self.assertEqual(
+            json.loads(response.content), {"release_sha": "unavailable"}
+        )
+
+    @override_settings(RELEASE_SHA="a" * 40)
+    def test_release_endpoint_strips_queries(self):
+        marker = "private-release-query-marker"
+        response = self.client.get(
+            f"/releasez?destination={marker}", secure=True
+        )
+        self.assertEqual(response.status_code, 303)
+        self.assertEqual(response.headers["Location"], "/releasez")
+        self.assertNotIn(marker, response.content.decode("utf-8"))
+
 
 class ReadinessTests(TestCase):
     def test_readyz_reads_only_the_database(self):
diff --git a/operations/tests/test_observability.py b/operations/tests/test_observability.py
index d263458..1bbfd81 100644
--- a/operations/tests/test_observability.py
+++ b/operations/tests/test_observability.py
@@ -1,14 +1,17 @@
 import io
 import json
 import logging
+import re
 import uuid
 from unittest.mock import patch
 
+from django.conf import settings
 from django.core.management import call_command
 from django.core.management.base import CommandError
 from django.test import RequestFactory, SimpleTestCase, override_settings
 
 from operations.middleware import RequestTelemetryMiddleware
+from operations.telemetry import utc_timestamp
 from operations.views import publication_freshness_snapshot
 
 
@@ -38,6 +41,8 @@ class RequestTelemetryTests(SimpleTestCase):
             set(payload),
             {
                 "event",
+                "timestamp",
+                "release_sha",
                 "route",
                 "method",
                 "status",
@@ -49,6 +54,11 @@ class RequestTelemetryTests(SimpleTestCase):
         self.assertEqual(payload["route"], "unmatched")
         self.assertEqual(payload["status"], 404)
         self.assertTrue(payload["redacted"])
+        self.assertEqual(payload["release_sha"], settings.RELEASE_SHA)
+        self.assertRegex(
+            payload["timestamp"],
+            r"\A\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}\.\d{3}Z\Z",
+        )
         uuid.UUID(payload["correlation_id"])
         self.assertNotIn(marker, logs.output[0])
 
@@ -91,6 +101,20 @@ class RequestTelemetryTests(SimpleTestCase):
         self.assertEqual(request_events[0]["route"], "unmatched")
         self.assertNotIn(marker, "\n".join(logs.output))
 
+    @override_settings(RELEASE_SHA="a" * 40)
+    def test_release_identity_is_an_exact_bounded_dimension(self):
+        request = RequestFactory().get("/releasez")
+        request.resolver_match = type("Match", (), {"route": "releasez"})()
+        response = type("Response", (), {"status_code": 200})()
+        middleware = RequestTelemetryMiddleware(lambda incoming: response)
+
+        with self.assertLogs("travel_readiness.request", level="INFO") as logs:
+            middleware(request)
+
+        payload = json.loads(logs.records[0].getMessage())
+        self.assertEqual(payload["release_sha"], "a" * 40)
+        self.assertEqual(payload["route"], "releasez")
+
     def test_installed_middleware_never_logs_body_or_credentials(self):
         marker = "private-body-authorization-source-marker"
         with self.assertLogs("travel_readiness.request", level="INFO") as logs:
@@ -104,6 +128,17 @@ class RequestTelemetryTests(SimpleTestCase):
         self.assertNotIn(marker, "\n".join(logs.output))
 
 
+class TelemetryTimestampTests(SimpleTestCase):
+    def test_timestamp_is_fixed_precision_rfc3339_utc(self):
+        timestamp = utc_timestamp()
+        self.assertIsNotNone(
+            re.fullmatch(
+                r"\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}\.\d{3}Z",
+                timestamp,
+            )
+        )
+
+
 class FreshnessIsolationTests(SimpleTestCase):
     def test_one_loader_failure_preserves_the_other_module_state(self):
         with (
diff --git a/operations/urls.py b/operations/urls.py
index 31999c8..927454d 100644
--- a/operations/urls.py
+++ b/operations/urls.py
@@ -1,9 +1,10 @@
 from django.urls import path
 
-from .views import freshnessz, healthz, readyz
+from .views import freshnessz, healthz, readyz, releasez
 
 urlpatterns = [
     path("healthz", healthz, name="healthz"),
     path("readyz", readyz, name="readyz"),
     path("freshnessz", freshnessz, name="freshnessz"),
+    path("releasez", releasez, name="releasez"),
 ]
diff --git a/operations/views.py b/operations/views.py
index c1e8fa2..efd7459 100644
--- a/operations/views.py
+++ b/operations/views.py
@@ -1,5 +1,6 @@
 import json
 
+from django.conf import settings
 from django.db import DatabaseError, connection
 from django.http import HttpRequest, HttpResponse
 from django.views.decorators.http import require_GET
@@ -84,3 +85,11 @@ def freshnessz(request: HttpRequest) -> HttpResponse:
         {"status": aggregate, "modules": states},
         status=200,
     )
+
+
+@require_GET
+def releasez(request: HttpRequest) -> HttpResponse:
+    release_sha = settings.RELEASE_SHA
+    if release_sha == "unavailable":
+        return _json_response({"release_sha": "unavailable"}, status=503)
+    return _json_response({"release_sha": release_sha}, status=200)
diff --git a/public_web/middleware.py b/public_web/middleware.py
index e2bbeda..07f8f2e 100644
--- a/public_web/middleware.py
+++ b/public_web/middleware.py
@@ -11,6 +11,7 @@ _OPERATIONS_CANONICAL_PATHS = {
     "/healthz": "/healthz",
     "/readyz": "/readyz",
     "/freshnessz": "/freshnessz",
+    "/releasez": "/releasez",
 }
 _CANONICAL_PATHS = {
     **_PUBLIC_CANONICAL_PATHS,
diff --git a/travel_readiness/settings.py b/travel_readiness/settings.py
index a1c3301..c496787 100644
--- a/travel_readiness/settings.py
+++ b/travel_readiness/settings.py
@@ -7,6 +7,8 @@ from pathlib import Path
 
 from django.core.exceptions import ImproperlyConfigured
 
+from operations.telemetry import bounded_release_sha, utc_timestamp
+
 BASE_DIR = Path(__file__).resolve().parent.parent
 
 
@@ -103,6 +105,7 @@ def redact_django_record(record) -> bool:
         "healthz",
         "readyz",
         "freshnessz",
+        "releasez",
     }:
         route = "unmatched"
     correlation_id = getattr(request, "correlation_id", None)
@@ -112,6 +115,8 @@ def redact_django_record(record) -> bool:
         correlation_id = None
     payload = {
         "event": "django",
+        "timestamp": utc_timestamp(),
+        "release_sha": RELEASE_SHA,
         "logger": logger,
         "level": level,
         "status": status,
@@ -132,6 +137,10 @@ def redact_django_record(record) -> bool:
 
 
 BUILD_MODE = strict_env_flag("TRAVEL_READINESS_BUILD")
+_RELEASE_SHA_RAW = os.environ.get("TRAVEL_READINESS_RELEASE_SHA")
+RELEASE_SHA = bounded_release_sha(_RELEASE_SHA_RAW)
+if _RELEASE_SHA_RAW and RELEASE_SHA == "unavailable":
+    raise ImproperlyConfigured("Invalid TRAVEL_READINESS_RELEASE_SHA configuration")
 SECRET_KEY = (
     "credential-free-static-build-only-not-for-runtime"
     if BUILD_MODE
diff --git a/travel_readiness/wsgi.py b/travel_readiness/wsgi.py
index 5a665ca..a1f3987 100644
--- a/travel_readiness/wsgi.py
+++ b/travel_readiness/wsgi.py
@@ -8,5 +8,7 @@ os.environ.setdefault("DJANGO_SETTINGS_MODULE", "travel_readiness.settings")
 
 if settings.BUILD_MODE:
     raise ImproperlyConfigured("Static build mode cannot start the WSGI runtime")
+if settings.RELEASE_SHA == "unavailable":
+    raise ImproperlyConfigured("An exact release SHA is required for WSGI runtime")
 
 application = get_wsgi_application()


