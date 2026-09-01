## `fix(operations): fail closed before deployment`

diff --git a/operations/checks.py b/operations/checks.py
new file mode 100644
index 0000000..f002f55
--- /dev/null
+++ b/operations/checks.py
@@ -0,0 +1,81 @@
+from django.conf import settings
+from django.core.checks import Error, Tags, register
+
+
+def _error(message: str, identifier: str) -> Error:
+    return Error(message, id=identifier)
+
+
+@register(Tags.security, deploy=True)
+def deployment_settings_check(app_configs, **kwargs):
+    errors = []
+    if settings.BUILD_MODE:
+        errors.append(
+            _error(
+                "Static build mode cannot be used as a deployed runtime.",
+                "operations.E001",
+            )
+        )
+    if settings.DEBUG:
+        errors.append(
+            _error(
+                "DEBUG must be false for a deployed runtime.",
+                "operations.E002",
+            )
+        )
+    if not settings.ALLOWED_HOSTS or any(
+        host == "*" or host.startswith(".") for host in settings.ALLOWED_HOSTS
+    ):
+        errors.append(
+            _error(
+                "Exact deployment hosts are required.",
+                "operations.E003",
+            )
+        )
+    if (
+        settings.SECURE_PROXY_SSL_HEADER is not None
+        or settings.USE_X_FORWARDED_HOST
+        or settings.USE_X_FORWARDED_PORT
+    ):
+        errors.append(
+            _error(
+                "Forwarded proxy headers must remain untrusted before approval.",
+                "operations.E004",
+            )
+        )
+    required_flags = (
+        settings.SECURE_SSL_REDIRECT,
+        settings.SECURE_HSTS_SECONDS == 31_536_000,
+        settings.SECURE_HSTS_INCLUDE_SUBDOMAINS,
+        settings.SECURE_HSTS_PRELOAD,
+        settings.SECURE_CONTENT_TYPE_NOSNIFF,
+        settings.SECURE_REFERRER_POLICY == "no-referrer",
+        settings.SECURE_CROSS_ORIGIN_OPENER_POLICY == "same-origin",
+        settings.SESSION_COOKIE_SECURE,
+        settings.SESSION_COOKIE_HTTPONLY,
+        settings.SESSION_COOKIE_SAMESITE == "Strict",
+        settings.CSRF_COOKIE_SECURE,
+        settings.CSRF_COOKIE_HTTPONLY,
+        settings.CSRF_COOKIE_SAMESITE == "Strict",
+        settings.X_FRAME_OPTIONS == "DENY",
+    )
+    if not all(required_flags):
+        errors.append(
+            _error(
+                "The strict transport, cookie and browser flags are required.",
+                "operations.E005",
+            )
+        )
+    if (
+        settings.DATA_UPLOAD_MAX_MEMORY_SIZE != 16 * 1024
+        or settings.FILE_UPLOAD_MAX_MEMORY_SIZE != 16 * 1024
+        or settings.DATA_UPLOAD_MAX_NUMBER_FIELDS != 10
+        or settings.DATA_UPLOAD_MAX_NUMBER_FILES != 0
+    ):
+        errors.append(
+            _error(
+                "The bounded request upload limits are required.",
+                "operations.E006",
+            )
+        )
+    return errors
diff --git a/operations/error_handlers.py b/operations/error_handlers.py
new file mode 100644
index 0000000..e01543a
--- /dev/null
+++ b/operations/error_handlers.py
@@ -0,0 +1,48 @@
+from django.http import HttpResponse
+
+
+_MESSAGES = {
+    400: "요청을 처리할 수 없습니다.\n",
+    403: "이 요청을 처리할 권한이 없습니다.\n",
+    404: "요청한 페이지를 찾을 수 없습니다.\n",
+    500: "일시적으로 요청을 처리할 수 없습니다.\n",
+}
+_HEADERS = {
+    "Cache-Control": "no-store",
+    "Pragma": "no-cache",
+    "Content-Security-Policy": (
+        "default-src 'none'; base-uri 'none'; frame-ancestors 'none'"
+    ),
+    "Referrer-Policy": "no-referrer",
+    "X-Content-Type-Options": "nosniff",
+    "X-Frame-Options": "DENY",
+}
+
+
+def _generic_error(status: int) -> HttpResponse:
+    return HttpResponse(
+        _MESSAGES[status],
+        status=status,
+        content_type="text/plain; charset=utf-8",
+        headers=_HEADERS,
+    )
+
+
+def bad_request(request, exception) -> HttpResponse:
+    return _generic_error(400)
+
+
+def permission_denied(request, exception) -> HttpResponse:
+    return _generic_error(403)
+
+
+def csrf_failure(request, reason="") -> HttpResponse:
+    return _generic_error(403)
+
+
+def page_not_found(request, exception) -> HttpResponse:
+    return _generic_error(404)
+
+
+def server_error(request) -> HttpResponse:
+    return _generic_error(500)
diff --git a/operations/tests/test_deployment_safety.py b/operations/tests/test_deployment_safety.py
new file mode 100644
index 0000000..084a97c
--- /dev/null
+++ b/operations/tests/test_deployment_safety.py
@@ -0,0 +1,373 @@
+import json
+import logging
+import os
+import subprocess
+import sys
+import uuid
+from pathlib import Path
+from types import SimpleNamespace
+
+from django.core.exceptions import PermissionDenied, SuspiciousOperation
+from django.http import HttpResponse
+from django.test import Client, SimpleTestCase, override_settings
+from django.urls import NoReverseMatch, path, reverse
+
+from travel_readiness.settings import redact_django_record
+
+
+_PRIVATE_MARKER = "private-input-exception-marker"
+
+
+def bad_request_view(request):
+    raise SuspiciousOperation(_PRIVATE_MARKER)
+
+
+def permission_denied_view(request):
+    raise PermissionDenied(_PRIVATE_MARKER)
+
+
+def server_error_view(request):
+    raise RuntimeError(_PRIVATE_MARKER)
+
+
+def csrf_target(request):
+    return HttpResponse("not reached")
+
+
+def request_body_target(request):
+    _ = request.body
+    return HttpResponse("not reached")
+
+
+def request_fields_target(request):
+    _ = len(request.POST)
+    return HttpResponse("not reached")
+
+
+urlpatterns = [
+    path("__errors__/400", bad_request_view),
+    path("__errors__/403", permission_denied_view),
+    path("__errors__/500", server_error_view),
+    path("__errors__/csrf", csrf_target),
+    path("__errors__/body", request_body_target),
+    path("__errors__/fields", request_fields_target),
+]
+handler400 = "operations.error_handlers.bad_request"
+handler403 = "operations.error_handlers.permission_denied"
+handler404 = "operations.error_handlers.page_not_found"
+handler500 = "operations.error_handlers.server_error"
+
+
+class SettingsProcessMixin:
+    def settings_environment(self, **overrides):
+        environment = {
+            "PATH": os.environ.get("PATH", ""),
+            "PYTHONPATH": str(Path(__file__).resolve().parents[2]),
+            "DJANGO_SETTINGS_MODULE": "travel_readiness.settings",
+            "TRAVEL_READINESS_SECRET_KEY": (
+                "test-only-not-a-credential-" * 3
+            ),
+            "TRAVEL_READINESS_DB_PASSWORD": "unused-candidate-database-value",
+            "TRAVEL_READINESS_ALLOWED_HOSTS": "candidate.invalid",
+        }
+        environment.update(overrides)
+        return environment
+
+    def run_python(self, code, **overrides):
+        return subprocess.run(
+            [sys.executable, "-c", code],
+            cwd=Path(__file__).resolve().parents[2],
+            env=self.settings_environment(**overrides),
+            capture_output=True,
+            text=True,
+            check=False,
+        )
+
+
+class DeploymentSettingsTests(SettingsProcessMixin, SimpleTestCase):
+    def test_runtime_requires_exact_nonwildcard_hosts(self):
+        invalid_hosts = (
+            "",
+            "*",
+            ".candidate.invalid",
+            "https://candidate.invalid",
+            "candidate.invalid:443",
+            "candidate.invalid/path",
+            "candidate.invalid,,second.invalid",
+            " candidate.invalid",
+            "candidate_invalid",
+            "candidate.invalid,CANDIDATE.INVALID",
+        )
+        for invalid_host in invalid_hosts:
+            with self.subTest(invalid_host=invalid_host):
+                result = self.run_python(
+                    "import travel_readiness.settings",
+                    TRAVEL_READINESS_ALLOWED_HOSTS=invalid_host,
+                )
+                self.assertNotEqual(result.returncode, 0)
+                self.assertIn(
+                    "Invalid TRAVEL_READINESS_ALLOWED_HOSTS configuration",
+                    result.stderr,
+                )
+                if invalid_host:
+                    self.assertNotIn(invalid_host, result.stderr)
+
+        accepted = self.run_python(
+            "from django.conf import settings; print(settings.ALLOWED_HOSTS)",
+            TRAVEL_READINESS_ALLOWED_HOSTS=(
+                "candidate.invalid,127.0.0.1,[::1]"
+            ),
+        )
+        self.assertEqual(accepted.returncode, 0, accepted.stderr)
+        self.assertEqual(
+            accepted.stdout.strip(),
+            "['candidate.invalid', '127.0.0.1', '[::1]']",
+        )
+
+    def test_environment_flags_reject_non_boolean_values(self):
+        for name in (
+            "TRAVEL_READINESS_BUILD",
+            "TRAVEL_READINESS_DEBUG",
+            "TRAVEL_READINESS_HTTPS",
+        ):
+            with self.subTest(name=name):
+                result = self.run_python(
+                    "import travel_readiness.settings",
+                    **{name: "yes"},
+                )
+                self.assertNotEqual(result.returncode, 0)
+                self.assertIn(
+                    "Environment variable must be 0 or 1",
+                    result.stderr,
+                )
+
+    def test_safe_reserved_host_passes_every_deployment_check(self):
+        result = subprocess.run(
+            [
+                sys.executable,
+                str(Path(__file__).resolve().parents[2] / "manage.py"),
+                "check",
+                "--deploy",
+                "--fail-level",
+                "WARNING",
+            ],
+            cwd=Path(__file__).resolve().parents[2],
+            env=self.settings_environment(),
+            capture_output=True,
+            text=True,
+            check=False,
+        )
+        self.assertEqual(result.returncode, 0, result.stderr)
+        self.assertIn("System check identified no issues", result.stdout)
+
+    def test_deployment_check_rejects_debug_runtime(self):
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
+            env=self.settings_environment(TRAVEL_READINESS_DEBUG="1"),
+            capture_output=True,
+            text=True,
+            check=False,
+        )
+        self.assertNotEqual(result.returncode, 0)
+        self.assertIn("operations.E002", result.stderr)
+
+    def test_strict_flags_limits_and_proxy_defaults_are_exact(self):
+        result = self.run_python(
+            "import json; from django.conf import settings; "
+            "print(json.dumps({"
+            "'debug': settings.DEBUG, "
+            "'ssl_redirect': settings.SECURE_SSL_REDIRECT, "
+            "'hsts_seconds': settings.SECURE_HSTS_SECONDS, "
+            "'hsts_subdomains': settings.SECURE_HSTS_INCLUDE_SUBDOMAINS, "
+            "'hsts_preload': settings.SECURE_HSTS_PRELOAD, "
+            "'referrer': settings.SECURE_REFERRER_POLICY, "
+            "'proxy': settings.SECURE_PROXY_SSL_HEADER, "
+            "'forwarded_host': settings.USE_X_FORWARDED_HOST, "
+            "'forwarded_port': settings.USE_X_FORWARDED_PORT, "
+            "'body': settings.DATA_UPLOAD_MAX_MEMORY_SIZE, "
+            "'file': settings.FILE_UPLOAD_MAX_MEMORY_SIZE, "
+            "'fields': settings.DATA_UPLOAD_MAX_NUMBER_FIELDS, "
+            "'files': settings.DATA_UPLOAD_MAX_NUMBER_FILES, "
+            "'origins': settings.CSRF_TRUSTED_ORIGINS}))"
+        )
+        self.assertEqual(result.returncode, 0, result.stderr)
+        self.assertEqual(
+            json.loads(result.stdout),
+            {
+                "debug": False,
+                "ssl_redirect": True,
+                "hsts_seconds": 31_536_000,
+                "hsts_subdomains": True,
+                "hsts_preload": True,
+                "referrer": "no-referrer",
+                "proxy": None,
+                "forwarded_host": False,
+                "forwarded_port": False,
+                "body": 16 * 1024,
+                "file": 16 * 1024,
+                "fields": 10,
+                "files": 0,
+                "origins": [],
+            },
+        )
+
+    def test_operator_admin_has_no_route(self):
+        with self.assertRaises(NoReverseMatch):
+            reverse("admin:index")
+        response = self.client.get("/admin/", secure=True)
+        self.assertEqual(response.status_code, 404)
+
+    def test_django_log_record_uses_only_bounded_request_dimensions(self):
+        marker = "private-method-route-ip-cookie-exception-marker"
+        record = logging.LogRecord(
+            f"django.security.{marker}",
+            123,
+            __file__,
+            1,
+            marker,
+            (marker,),
+            (RuntimeError, RuntimeError(marker), None),
+        )
+        record.request = SimpleNamespace(
+            method=marker,
+            resolver_match=SimpleNamespace(route=marker),
+            correlation_id=marker,
+            META={"REMOTE_ADDR": marker},
+            COOKIES={marker: marker},
+        )
+        record.status_code = marker
+        record.stack_info = marker
+
+        self.assertTrue(redact_django_record(record))
+        payload = json.loads(record.getMessage())
+
+        self.assertEqual(
+            payload,
+            {
+                "event": "django",
+                "logger": "security",
+                "level": "OTHER",
+                "status": 500,
+                "method": "OTHER",
+                "route": "unmatched",
+                "correlation_id": None,
+                "redacted": True,
+            },
+        )
+        self.assertNotIn(marker, record.getMessage())
+        self.assertNotIn(marker, repr(record.__dict__))
+        self.assertEqual(record.args, ())
+        self.assertIsNone(record.exc_info)
+        self.assertIsNone(record.stack_info)
+        self.assertIsNone(record.request)
+
+    def test_django_log_record_preserves_only_known_values(self):
+        correlation_id = str(uuid.uuid4())
+        record = logging.LogRecord(
+            "django.request",
+            logging.WARNING,
+            __file__,
+            1,
+            "discarded",
+            (),
+            None,
+        )
+        record.request = SimpleNamespace(
+            method="GET",
+            resolver_match=SimpleNamespace(route="results/"),
+            correlation_id=correlation_id,
+        )
+        record.status_code = 404
+
+        redact_django_record(record)
+
+        self.assertEqual(
+            json.loads(record.getMessage()),
+            {
+                "event": "django",
+                "logger": "request",
+                "level": "WARNING",
+                "status": 404,
+                "method": "GET",
+                "route": "results/",
+                "correlation_id": correlation_id,
+                "redacted": True,
+            },
+        )
+
+
+@override_settings(
+    DEBUG=False,
+    SECURE_SSL_REDIRECT=False,
+    ROOT_URLCONF=__name__,
+    ALLOWED_HOSTS=["testserver"],
+)
+class GenericErrorHandlerTests(SimpleTestCase):
+    def test_global_errors_are_fixed_no_store_and_redacted(self):
+        cases = (
+            ("/__errors__/400", 400, "요청을 처리할 수 없습니다."),
+            ("/__errors__/403", 403, "이 요청을 처리할 권한이 없습니다."),
+            ("/__errors__/missing", 404, "요청한 페이지를 찾을 수 없습니다."),
+            ("/__errors__/500", 500, "일시적으로 요청을 처리할 수 없습니다."),
+        )
+        client = Client()
+        client.raise_request_exception = False
+        for route, status, message in cases:
+            with self.subTest(status=status):
+                response = client.get(f"{route}?destination={_PRIVATE_MARKER}")
+                self.assertEqual(response.status_code, status)
+                self.assertEqual(response.content.decode().strip(), message)
+                self.assertEqual(response.headers["Cache-Control"], "no-store")
+                self.assertEqual(response.headers["Pragma"], "no-cache")
+                self.assertEqual(
+                    response.headers["Content-Security-Policy"],
+                    "default-src 'none'; base-uri 'none'; frame-ancestors 'none'",
+                )
+                self.assertNotIn(_PRIVATE_MARKER, response.content.decode())
+
+    def test_csrf_failure_drops_post_body_and_reason(self):
+        client = Client(enforce_csrf_checks=True)
+        response = client.post(
+            "/__errors__/csrf",
+            {"destination": _PRIVATE_MARKER},
+        )
+
+        self.assertEqual(response.status_code, 403)
+        self.assertEqual(
+            response.content,
+            "이 요청을 처리할 권한이 없습니다.\n".encode(),
+        )
+        self.assertEqual(response.headers["Cache-Control"], "no-store")
+        self.assertNotIn(_PRIVATE_MARKER, response.content.decode())
+
+    def test_oversize_body_and_field_count_fail_as_generic_400(self):
+        client = Client()
+        client.raise_request_exception = False
+        responses = (
+            client.post(
+                "/__errors__/body",
+                data=_PRIVATE_MARKER * 2_000,
+                content_type="text/plain",
+            ),
+            client.post(
+                "/__errors__/fields",
+                {f"field-{index}": _PRIVATE_MARKER for index in range(11)},
+            ),
+        )
+
+        for response in responses:
+            self.assertEqual(response.status_code, 400)
+            self.assertEqual(
+                response.content,
+                "요청을 처리할 수 없습니다.\n".encode(),
+            )
+            self.assertEqual(response.headers["Cache-Control"], "no-store")
+            self.assertNotIn(_PRIVATE_MARKER, response.content.decode())
diff --git a/travel_readiness/settings.py b/travel_readiness/settings.py
index 63325c8..919eb5a 100644
--- a/travel_readiness/settings.py
+++ b/travel_readiness/settings.py
@@ -1,5 +1,8 @@
+import ipaddress
 import json
 import os
+import re
+import uuid
 from pathlib import Path
 
 from django.core.exceptions import ImproperlyConfigured
@@ -25,14 +28,106 @@ def strict_env_flag(name: str, *, default: bool = False) -> bool:
     return value == "1"
 
 
+_DNS_HOST_RE = re.compile(
+    r"[a-z0-9](?:[a-z0-9-]{0,61}[a-z0-9])?"
+    r"(?:\.[a-z0-9](?:[a-z0-9-]{0,61}[a-z0-9])?)*",
+    re.IGNORECASE | re.ASCII,
+)
+
+
+def _is_exact_allowed_host(host: str) -> bool:
+    if not host or len(host) > 253 or host != host.strip():
+        return False
+    if host.startswith("[") and host.endswith("]"):
+        try:
+            return ipaddress.ip_address(host[1:-1]).version == 6
+        except ValueError:
+            return False
+    if ":" in host:
+        return False
+    try:
+        return ipaddress.ip_address(host).version == 4
+    except ValueError:
+        return _DNS_HOST_RE.fullmatch(host) is not None
+
+
+def required_allowed_hosts() -> list[str]:
+    raw = os.environ.get("TRAVEL_READINESS_ALLOWED_HOSTS", "")
+    candidates = raw.split(",") if raw else []
+    if (
+        not candidates
+        or any(not _is_exact_allowed_host(host) for host in candidates)
+        or len({host.lower() for host in candidates}) != len(candidates)
+    ):
+        raise ImproperlyConfigured(
+            "Invalid TRAVEL_READINESS_ALLOWED_HOSTS configuration"
+        )
+    return [host.lower() for host in candidates]
+
+
 def redact_django_record(record) -> bool:
     request = getattr(record, "request", None)
     match = getattr(request, "resolver_match", None)
-    payload = {"event": "django", "logger": record.name, "level": record.levelname, "status": getattr(record, "status_code", None), "method": getattr(request, "method", None), "route": getattr(match, "route", None), "correlation_id": getattr(request, "correlation_id", None), "redacted": True}
+    logger_name = getattr(record, "name", "")
+    if type(logger_name) is not str:
+        logger_name = ""
+    if logger_name == "django.request":
+        logger = "request"
+    elif logger_name == "django.server":
+        logger = "server"
+    elif logger_name == "django.security" or logger_name.startswith(
+        "django.security."
+    ):
+        logger = "security"
+    else:
+        logger = "django"
+    level = getattr(record, "levelname", "")
+    if type(level) is not str or level not in {
+        "DEBUG",
+        "INFO",
+        "WARNING",
+        "ERROR",
+        "CRITICAL",
+    }:
+        level = "OTHER"
+    status = getattr(record, "status_code", None)
+    if type(status) is not int or not 100 <= status <= 599:
+        status = 500
+    method = getattr(request, "method", None)
+    if type(method) is not str or method not in {"GET", "POST", "HEAD"}:
+        method = "OTHER"
+    route = getattr(match, "route", None)
+    if type(route) is not str or route not in {
+        "",
+        "results/",
+        "healthz",
+        "readyz",
+        "freshnessz",
+    }:
+        route = "unmatched"
+    correlation_id = getattr(request, "correlation_id", None)
+    try:
+        correlation_id = str(uuid.UUID(correlation_id))
+    except (AttributeError, TypeError, ValueError):
+        correlation_id = None
+    payload = {
+        "event": "django",
+        "logger": logger,
+        "level": level,
+        "status": status,
+        "method": method,
+        "route": route,
+        "correlation_id": correlation_id,
+        "redacted": True,
+    }
+    record.name = "django" if logger == "django" else f"django.{logger}"
+    record.levelname = level
     record.msg = json.dumps(payload, separators=(",", ":"))
     record.args = ()
     record.exc_info = record.exc_text = None
     record.stack_info = None
+    record.request = None
+    record.status_code = status
     return True
 
 
@@ -43,11 +138,7 @@ SECRET_KEY = (
     else required_env("TRAVEL_READINESS_SECRET_KEY")
 )
 DEBUG = strict_env_flag("TRAVEL_READINESS_DEBUG")
-ALLOWED_HOSTS = [
-    host.strip()
-    for host in os.environ.get("TRAVEL_READINESS_ALLOWED_HOSTS", "").split(",")
-    if host.strip()
-]
+ALLOWED_HOSTS = [] if BUILD_MODE else required_allowed_hosts()
 
 INSTALLED_APPS = [
     "operations",
@@ -125,18 +216,29 @@ STATIC_URL = "/static/"
 STATIC_ROOT = BASE_DIR / "staticfiles"
 DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"
 
-SECURE_SSL_REDIRECT = os.environ.get("TRAVEL_READINESS_HTTPS", "1") != "0"
+DATA_UPLOAD_MAX_MEMORY_SIZE = 16 * 1024
+DATA_UPLOAD_MAX_NUMBER_FIELDS = 10
+DATA_UPLOAD_MAX_NUMBER_FILES = 0
+FILE_UPLOAD_MAX_MEMORY_SIZE = 16 * 1024
+
+SECURE_SSL_REDIRECT = strict_env_flag("TRAVEL_READINESS_HTTPS", default=True)
 SECURE_HSTS_SECONDS = 31_536_000
 SECURE_HSTS_INCLUDE_SUBDOMAINS = True
 SECURE_HSTS_PRELOAD = True
 SECURE_CONTENT_TYPE_NOSNIFF = True
 SECURE_REFERRER_POLICY = "no-referrer"
+SECURE_CROSS_ORIGIN_OPENER_POLICY = "same-origin"
+SECURE_PROXY_SSL_HEADER = None
+USE_X_FORWARDED_HOST = False
+USE_X_FORWARDED_PORT = False
 SESSION_COOKIE_SECURE = True
 SESSION_COOKIE_HTTPONLY = True
 SESSION_COOKIE_SAMESITE = "Strict"
 CSRF_COOKIE_SECURE = True
 CSRF_COOKIE_HTTPONLY = True
 CSRF_COOKIE_SAMESITE = "Strict"
+CSRF_TRUSTED_ORIGINS = []
+CSRF_FAILURE_VIEW = "operations.error_handlers.csrf_failure"
 X_FRAME_OPTIONS = "DENY"
 
 LOGGING = {
@@ -151,3 +253,7 @@ LOGGING = {
         "django.server": {"handlers": ["redacted"], "propagate": False},
     },
 }
+
+# Registration is intentionally explicit because deployment checks are not an
+# application runtime dependency and must also run before a database exists.
+from operations import checks as _operations_checks  # noqa: E402,F401
diff --git a/travel_readiness/urls.py b/travel_readiness/urls.py
index a147719..e32565b 100644
--- a/travel_readiness/urls.py
+++ b/travel_readiness/urls.py
@@ -1,5 +1,10 @@
 from django.urls import include, path
 
+handler400 = "operations.error_handlers.bad_request"
+handler403 = "operations.error_handlers.permission_denied"
+handler404 = "operations.error_handlers.page_not_found"
+handler500 = "operations.error_handlers.server_error"
+
 urlpatterns = [
     path("", include("public_web.urls")),
     path("", include("operations.urls")),


