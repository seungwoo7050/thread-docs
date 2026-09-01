# 공개 HTTP 보안과 질의 프라이버시

## `feat(security): enforce public response boundaries`

diff --git a/grocery/security.py b/grocery/security.py
new file mode 100644
index 0000000..671fdf9
--- /dev/null
+++ b/grocery/security.py
@@ -0,0 +1,70 @@
+"""Fixed HTTP security boundaries for the server-rendered public surface."""
+
+from collections.abc import Callable
+from typing import Final
+
+from django.conf import settings
+from django.http import HttpRequest, HttpResponseNotFound
+from django.http.response import HttpResponseBase
+
+GetResponse = Callable[[HttpRequest], HttpResponseBase]
+
+CONTENT_SECURITY_POLICY: Final = (
+    "default-src 'self'; "
+    "script-src 'none'; "
+    "style-src 'self'; "
+    "img-src 'self' data:; "
+    "font-src 'self'; "
+    "connect-src 'self'; "
+    "frame-src 'none'; "
+    "frame-ancestors 'none'; "
+    "object-src 'none'; "
+    "base-uri 'none'; "
+    "form-action 'self'"
+)
+
+SECURITY_HEADERS: Final[dict[str, str]] = {
+    "Content-Security-Policy": CONTENT_SECURITY_POLICY,
+    "Permissions-Policy": "camera=(), geolocation=(), microphone=(), payment=()",
+    "Cross-Origin-Opener-Policy": "same-origin",
+    "Cross-Origin-Resource-Policy": "same-origin",
+    "Referrer-Policy": "same-origin",
+    "X-Content-Type-Options": "nosniff",
+    "X-Frame-Options": "DENY",
+}
+
+
+class SecurityHeadersMiddleware:
+    """Attach one input-independent header policy to every downstream response."""
+
+    sync_capable = True
+    async_capable = False
+
+    def __init__(self, get_response: GetResponse) -> None:
+        self.get_response = get_response
+
+    def __call__(self, request: HttpRequest) -> HttpResponseBase:
+        response = self.get_response(request)
+        for header_name, header_value in SECURITY_HEADERS.items():
+            response.headers[header_name] = header_value
+        return response
+
+
+class AdminExposureMiddleware:
+    """Fail closed with a generic response when the Django admin is disabled."""
+
+    sync_capable = True
+    async_capable = False
+
+    def __init__(self, get_response: GetResponse) -> None:
+        self.get_response = get_response
+
+    def __call__(self, request: HttpRequest) -> HttpResponseBase:
+        path = request.path_info
+        admin_path = path == "/admin" or path.startswith("/admin/")
+        if admin_path and not getattr(settings, "ADMIN_ENABLED", False):
+            return HttpResponseNotFound(
+                "Not Found",
+                content_type="text/plain; charset=utf-8",
+            )
+        return self.get_response(request)
diff --git a/grocery/tests/test_security.py b/grocery/tests/test_security.py
new file mode 100644
index 0000000..5a21208
--- /dev/null
+++ b/grocery/tests/test_security.py
@@ -0,0 +1,132 @@
+import pytest
+from django.http import HttpRequest, HttpResponse
+from django.test import RequestFactory, override_settings
+
+from grocery.security import (
+    CONTENT_SECURITY_POLICY,
+    SECURITY_HEADERS,
+    AdminExposureMiddleware,
+    SecurityHeadersMiddleware,
+)
+
+
+@pytest.mark.parametrize("status_code", [200, 400, 403, 404, 500])
+def test_security_headers_apply_to_success_and_error_responses(status_code: int) -> None:
+    request = RequestFactory().get("/example")
+    middleware = SecurityHeadersMiddleware(
+        lambda unused_request: HttpResponse(status=status_code)
+    )
+
+    response = middleware(request)
+
+    assert response.status_code == status_code
+    for header_name, header_value in SECURITY_HEADERS.items():
+        assert response.headers[header_name] == header_value
+
+
+def test_content_security_policy_allows_only_required_same_origin_assets() -> None:
+    directives = set(CONTENT_SECURITY_POLICY.split("; "))
+
+    assert directives == {
+        "default-src 'self'",
+        "script-src 'none'",
+        "style-src 'self'",
+        "img-src 'self' data:",
+        "font-src 'self'",
+        "connect-src 'self'",
+        "frame-src 'none'",
+        "frame-ancestors 'none'",
+        "object-src 'none'",
+        "base-uri 'none'",
+        "form-action 'self'",
+    }
+    assert "http:" not in CONTENT_SECURITY_POLICY
+    assert "https:" not in CONTENT_SECURITY_POLICY
+    assert "*" not in CONTENT_SECURITY_POLICY
+    assert "'unsafe-inline'" not in CONTENT_SECURITY_POLICY
+    assert "'unsafe-eval'" not in CONTENT_SECURITY_POLICY
+
+
+def test_privileged_browser_capabilities_and_cross_origin_access_are_denied() -> None:
+    request = RequestFactory().get("/")
+    response = SecurityHeadersMiddleware(lambda unused: HttpResponse())(request)
+
+    assert response.headers["Permissions-Policy"] == (
+        "camera=(), geolocation=(), microphone=(), payment=()"
+    )
+    assert response.headers["Cross-Origin-Opener-Policy"] == "same-origin"
+    assert response.headers["Cross-Origin-Resource-Policy"] == "same-origin"
+    assert response.headers["X-Frame-Options"] == "DENY"
+    assert response.headers["X-Content-Type-Options"] == "nosniff"
+
+
+@pytest.mark.parametrize("path", ["/admin", "/admin/", "/admin/auth/user/"])
+@override_settings(ADMIN_ENABLED=False)
+def test_disabled_admin_returns_generic_404_without_calling_downstream(path: str) -> None:
+    downstream_called = False
+
+    def get_response(request: HttpRequest) -> HttpResponse:
+        nonlocal downstream_called
+        downstream_called = True
+        return HttpResponse(status=200)
+
+    response = AdminExposureMiddleware(get_response)(RequestFactory().get(path))
+
+    assert response.status_code == 404
+    assert isinstance(response, HttpResponse)
+    assert response.content == b"Not Found"
+    assert response.headers["Content-Type"] == "text/plain; charset=utf-8"
+    assert downstream_called is False
+
+
+@override_settings(ADMIN_ENABLED=True)
+def test_enabled_admin_leaves_authentication_and_csrf_flow_downstream() -> None:
+    downstream_response = HttpResponse("admin login", status=401)
+    seen_requests: list[HttpRequest] = []
+
+    def get_response(request: HttpRequest) -> HttpResponse:
+        seen_requests.append(request)
+        return downstream_response
+
+    request = RequestFactory().post("/admin/login/", {"username": "reviewer"})
+    response = AdminExposureMiddleware(get_response)(request)
+
+    assert response is downstream_response
+    assert response.status_code == 401
+    assert seen_requests == [request]
+
+
+@override_settings(ADMIN_ENABLED=False)
+def test_admin_prefix_does_not_hide_unrelated_public_path() -> None:
+    def downstream(unused_request: HttpRequest) -> HttpResponse:
+        del unused_request
+        return HttpResponse(status=204)
+
+    response = AdminExposureMiddleware(downstream)(
+        RequestFactory().get("/administrator/help")
+    )
+
+    assert response.status_code == 204
+
+
+@override_settings(ADMIN_ENABLED=False)
+def test_malicious_query_is_not_reflected_in_headers_or_admin_error() -> None:
+    malicious_value = "https://evil.invalid/'><script>alert(1)</script>"
+    request = RequestFactory().get("/admin/login/", {"next": malicious_value})
+    middleware = SecurityHeadersMiddleware(
+        AdminExposureMiddleware(lambda unused: HttpResponse(status=200))
+    )
+
+    response = middleware(request)
+    serialized_headers = "\n".join(
+        f"{header_name}: {header_value}"
+        for header_name, header_value in response.headers.items()
+    )
+    assert isinstance(response, HttpResponse)
+    body = response.content.decode("utf-8")
+
+    assert response.status_code == 404
+    assert malicious_value not in serialized_headers
+    assert malicious_value not in body
+    assert "evil.invalid" not in serialized_headers
+    assert "evil.invalid" not in body


## `feat(security): close production admin by default`

diff --git a/.env.example b/.env.example
index bf6881a..6f3a22c 100644
--- a/.env.example
+++ b/.env.example
@@ -1,5 +1,6 @@
 # 로컬 예시값뿐이다. production 값이나 KAMIS key를 commit하지 않는다.
 DJANGO_DEBUG=0
+ADMIN_ENABLED=0
 DJANGO_SECRET_KEY=replace-in-secret-store
 DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1
 DATABASE_URL=postgresql://grocery:local-grocery-only@127.0.0.1:55434/grocery
diff --git a/config/settings.py b/config/settings.py
index 7f2652b..4bc3897 100644
--- a/config/settings.py
+++ b/config/settings.py
@@ -42,6 +42,7 @@ def database_config() -> dict[str, object]:
 
 
 DEBUG = env_bool("DJANGO_DEBUG", True)
+ADMIN_ENABLED = env_bool("ADMIN_ENABLED", DEBUG)
 SECRET_KEY = os.environ.get("DJANGO_SECRET_KEY", "local-development-only-not-for-production")
 ALLOWED_HOSTS = env_list("DJANGO_ALLOWED_HOSTS", "localhost,127.0.0.1,testserver")
 CSRF_TRUSTED_ORIGINS = env_list("DJANGO_CSRF_TRUSTED_ORIGINS", "")
@@ -58,7 +59,9 @@ INSTALLED_APPS = [
 
 MIDDLEWARE = [
     "django.middleware.security.SecurityMiddleware",
+    "grocery.security.SecurityHeadersMiddleware",
     "grocery.observability.RequestIdMiddleware",
+    "grocery.security.AdminExposureMiddleware",
     "django.contrib.sessions.middleware.SessionMiddleware",
     "django.middleware.common.CommonMiddleware",
     "django.middleware.csrf.CsrfViewMiddleware",
diff --git a/grocery/tests/test_logging_settings.py b/grocery/tests/test_logging_settings.py
index 5e99bda..bbc3cc0 100644
--- a/grocery/tests/test_logging_settings.py
+++ b/grocery/tests/test_logging_settings.py
@@ -21,6 +21,18 @@ class LoggingSettingsTests(SimpleTestCase):
         self.assertFalse(loggers["django.request"]["propagate"])
         self.assertFalse(loggers["django.server"]["propagate"])
 
+    def test_public_security_middleware_order_is_fail_closed(self) -> None:
+        middleware = list(settings.MIDDLEWARE)
+
+        self.assertLess(
+            middleware.index("grocery.security.SecurityHeadersMiddleware"),
+            middleware.index("grocery.observability.RequestIdMiddleware"),
+        )
+        self.assertLess(
+            middleware.index("grocery.observability.RequestIdMiddleware"),
+            middleware.index("grocery.security.AdminExposureMiddleware"),
+        )
+
     def test_audit_logger_uses_only_structured_allowlisted_output(self) -> None:
         logging_config = cast(dict[str, Any], settings.LOGGING)
         logger = logging_config["loggers"]["grocery.audit"]


## `fix(security): fail closed on production settings`

diff --git a/.env.example b/.env.example
index 6f3a22c..5a85a53 100644
--- a/.env.example
+++ b/.env.example
@@ -3,6 +3,7 @@ DJANGO_DEBUG=0
 ADMIN_ENABLED=0
 DJANGO_SECRET_KEY=replace-in-secret-store
 DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1
+DJANGO_CSRF_TRUSTED_ORIGINS=https://replace-with-approved-domain.example
 DATABASE_URL=postgresql://grocery:local-grocery-only@127.0.0.1:55434/grocery
 KAMIS_API_KEY=
 DEPLOY_VERSION=0000000
diff --git a/config/settings.py b/config/settings.py
index 4bc3897..59d89c8 100644
--- a/config/settings.py
+++ b/config/settings.py
@@ -1,7 +1,10 @@
 import os
+from collections.abc import Mapping, Sequence
 from pathlib import Path
 from urllib.parse import parse_qsl, unquote, urlparse
 
+from django.core.exceptions import ImproperlyConfigured
+
 BASE_DIR = Path(__file__).resolve().parent.parent
 
 
@@ -47,6 +50,51 @@ SECRET_KEY = os.environ.get("DJANGO_SECRET_KEY", "local-development-only-not-for
 ALLOWED_HOSTS = env_list("DJANGO_ALLOWED_HOSTS", "localhost,127.0.0.1,testserver")
 CSRF_TRUSTED_ORIGINS = env_list("DJANGO_CSRF_TRUSTED_ORIGINS", "")
 
+
+def validate_production_environment(
+    environment: Mapping[str, str],
+    *,
+    debug: bool,
+    admin_enabled: bool,
+    secret_key: str,
+    allowed_hosts: Sequence[str],
+    csrf_trusted_origins: Sequence[str],
+) -> None:
+    """Reject incomplete production settings without reflecting any supplied value."""
+
+    if debug:
+        return
+    if "DJANGO_SECRET_KEY" not in environment or len(secret_key) < 50:
+        raise ImproperlyConfigured("production_secret_key_required")
+    if "DJANGO_ALLOWED_HOSTS" not in environment or not allowed_hosts or "*" in allowed_hosts:
+        raise ImproperlyConfigured("production_allowed_hosts_required")
+    if "DJANGO_CSRF_TRUSTED_ORIGINS" not in environment or not csrf_trusted_origins:
+        raise ImproperlyConfigured("production_csrf_origins_required")
+    for origin in csrf_trusted_origins:
+        parsed_origin = urlparse(origin)
+        if (
+            parsed_origin.scheme != "https"
+            or not parsed_origin.hostname
+            or parsed_origin.username is not None
+            or parsed_origin.password is not None
+            or parsed_origin.query
+            or parsed_origin.fragment
+            or parsed_origin.path not in ("", "/")
+        ):
+            raise ImproperlyConfigured("production_csrf_origin_invalid")
+    if admin_enabled:
+        raise ImproperlyConfigured("production_admin_strong_auth_not_configured")
+
+
+validate_production_environment(
+    os.environ,
+    debug=DEBUG,
+    admin_enabled=ADMIN_ENABLED,
+    secret_key=SECRET_KEY,
+    allowed_hosts=ALLOWED_HOSTS,
+    csrf_trusted_origins=CSRF_TRUSTED_ORIGINS,
+)
+
 INSTALLED_APPS = [
     "django.contrib.admin",
     "django.contrib.auth",
diff --git a/grocery/tests/test_production_settings.py b/grocery/tests/test_production_settings.py
new file mode 100644
index 0000000..8fc0001
--- /dev/null
+++ b/grocery/tests/test_production_settings.py
@@ -0,0 +1,156 @@
+from collections.abc import Mapping, Sequence
+
+import pytest
+from django.core.exceptions import ImproperlyConfigured
+
+from config.settings import validate_production_environment
+
+_SAFE_SECRET = "x" * 50
+_SAFE_HOSTS = ("prices.example",)
+_SAFE_ORIGINS = ("https://prices.example",)
+
+
+def validate(
+    environment: Mapping[str, str],
+    *,
+    debug: bool = False,
+    admin_enabled: bool = False,
+    secret_key: str = _SAFE_SECRET,
+    allowed_hosts: Sequence[str] = _SAFE_HOSTS,
+    csrf_trusted_origins: Sequence[str] = _SAFE_ORIGINS,
+) -> None:
+    validate_production_environment(
+        environment,
+        debug=debug,
+        admin_enabled=admin_enabled,
+        secret_key=secret_key,
+        allowed_hosts=allowed_hosts,
+        csrf_trusted_origins=csrf_trusted_origins,
+    )
+
+
+def production_environment() -> dict[str, str]:
+    return {
+        "DJANGO_SECRET_KEY": "present",
+        "DJANGO_ALLOWED_HOSTS": "present",
+        "DJANGO_CSRF_TRUSTED_ORIGINS": "present",
+    }
+
+
+def test_debug_environment_keeps_local_development_defaults() -> None:
+    validate(
+        {},
+        debug=True,
+        admin_enabled=True,
+        secret_key="local",
+        allowed_hosts=(),
+        csrf_trusted_origins=(),
+    )
+
+
+def test_complete_production_environment_is_accepted_with_admin_disabled() -> None:
+    validate(production_environment())
+
+
+@pytest.mark.parametrize(
+    (
+        "environment",
+        "admin_enabled",
+        "secret_key",
+        "allowed_hosts",
+        "csrf_trusted_origins",
+        "code",
+    ),
+    [
+        ({}, False, _SAFE_SECRET, _SAFE_HOSTS, _SAFE_ORIGINS, "production_secret_key_required"),
+        (
+            {"DJANGO_SECRET_KEY": "present"},
+            False,
+            "short",
+            _SAFE_HOSTS,
+            _SAFE_ORIGINS,
+            "production_secret_key_required",
+        ),
+        (
+            {
+                "DJANGO_SECRET_KEY": "present",
+                "DJANGO_CSRF_TRUSTED_ORIGINS": "present",
+            },
+            False,
+            _SAFE_SECRET,
+            _SAFE_HOSTS,
+            _SAFE_ORIGINS,
+            "production_allowed_hosts_required",
+        ),
+        (
+            production_environment(),
+            False,
+            _SAFE_SECRET,
+            ("*",),
+            _SAFE_ORIGINS,
+            "production_allowed_hosts_required",
+        ),
+        (
+            {
+                "DJANGO_SECRET_KEY": "present",
+                "DJANGO_ALLOWED_HOSTS": "present",
+            },
+            False,
+            _SAFE_SECRET,
+            _SAFE_HOSTS,
+            _SAFE_ORIGINS,
+            "production_csrf_origins_required",
+        ),
+        (
+            production_environment(),
+            False,
+            _SAFE_SECRET,
+            _SAFE_HOSTS,
+            ("http://prices.example",),
+            "production_csrf_origin_invalid",
+        ),
+        (
+            production_environment(),
+            False,
+            _SAFE_SECRET,
+            _SAFE_HOSTS,
+            ("https://prices.example/unexpected",),
+            "production_csrf_origin_invalid",
+        ),
+        (
+            production_environment(),
+            True,
+            _SAFE_SECRET,
+            _SAFE_HOSTS,
+            _SAFE_ORIGINS,
+            "production_admin_strong_auth_not_configured",
+        ),
+    ],
+)
+def test_incomplete_or_unsafe_production_environment_fails_closed(
+    environment: Mapping[str, str],
+    admin_enabled: bool,
+    secret_key: str,
+    allowed_hosts: Sequence[str],
+    csrf_trusted_origins: Sequence[str],
+    code: str,
+) -> None:
+    with pytest.raises(ImproperlyConfigured, match=f"^{code}$"):
+        validate(
+            environment,
+            admin_enabled=admin_enabled,
+            secret_key=secret_key,
+            allowed_hosts=allowed_hosts,
+            csrf_trusted_origins=csrf_trusted_origins,
+        )
+
+
+def test_validation_error_never_reflects_supplied_values() -> None:
+    marker = "must-not-be-reflected"
+    with pytest.raises(ImproperlyConfigured) as caught:
+        validate(
+            production_environment(),
+            csrf_trusted_origins=(f"https://prices.example/{marker}",),
+        )
+
+    assert marker not in str(caught.value)


