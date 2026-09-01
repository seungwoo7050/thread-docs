## `feat(ops): correlate logs with release SHA`

diff --git a/config/settings.py b/config/settings.py
index 2524aff..8f50c8c 100644
--- a/config/settings.py
+++ b/config/settings.py
@@ -62,6 +62,7 @@ ADMIN_ENABLED = env_bool("ADMIN_ENABLED", DEBUG)
 SECRET_KEY = os.environ.get("DJANGO_SECRET_KEY", "local-development-only-not-for-production")
 ALLOWED_HOSTS = env_list("DJANGO_ALLOWED_HOSTS", "localhost,127.0.0.1,testserver")
 CSRF_TRUSTED_ORIGINS = env_list("DJANGO_CSRF_TRUSTED_ORIGINS", "")
+DEPLOY_VERSION = os.environ.get("DEPLOY_VERSION", "0000000")
 
 
 def validate_production_environment(
@@ -72,6 +73,7 @@ def validate_production_environment(
     secret_key: str,
     allowed_hosts: Sequence[str],
     csrf_trusted_origins: Sequence[str],
+    deploy_version: str,
 ) -> None:
     """Reject incomplete production settings without reflecting any supplied value."""
 
@@ -95,6 +97,12 @@ def validate_production_environment(
             or parsed_origin.path not in ("", "/")
         ):
             raise ImproperlyConfigured("production_csrf_origin_invalid")
+    if (
+        "DEPLOY_VERSION" not in environment
+        or len(deploy_version) != 40
+        or any(character not in "0123456789abcdef" for character in deploy_version)
+    ):
+        raise ImproperlyConfigured("production_deploy_version_required")
     if admin_enabled:
         raise ImproperlyConfigured("production_admin_strong_auth_not_configured")
 
@@ -106,6 +114,7 @@ validate_production_environment(
     secret_key=SECRET_KEY,
     allowed_hosts=ALLOWED_HOSTS,
     csrf_trusted_origins=CSRF_TRUSTED_ORIGINS,
+    deploy_version=DEPLOY_VERSION,
 )
 
 INSTALLED_APPS = [
@@ -178,7 +187,6 @@ SECURE_CONTENT_TYPE_NOSNIFF = True
 SECURE_REFERRER_POLICY = "same-origin"
 X_FRAME_OPTIONS = "DENY"
 
-DEPLOY_VERSION = os.environ.get("DEPLOY_VERSION", "0000000")
 KAMIS_CONFIRMATION_MAX_AGE_HOURS = env_positive_int(
     "KAMIS_CONFIRMATION_MAX_AGE_HOURS",
     36,
diff --git a/grocery/observability.py b/grocery/observability.py
index d624dd6..823dcbc 100644
--- a/grocery/observability.py
+++ b/grocery/observability.py
@@ -17,6 +17,7 @@ from datetime import UTC, datetime
 from typing import Final, Literal, cast
 from uuid import UUID, uuid4
 
+from django.conf import settings
 from django.http import HttpRequest, HttpResponseBase
 
 type Severity = Literal["DEBUG", "INFO", "WARNING", "ERROR", "CRITICAL"]
@@ -256,6 +257,14 @@ def log_event(
 
     if severity not in _LEVEL_BY_SEVERITY:
         raise ObservabilityValidationError("severity has an invalid value")
+    configured_deploy_version = getattr(settings, "DEPLOY_VERSION", None)
+    if (
+        deploy_version is None
+        and type(configured_deploy_version) is str
+        and _DEPLOY_VERSION_PATTERN.fullmatch(configured_deploy_version) is not None
+    ):
+        deploy_version = configured_deploy_version
+
     event = make_observability_event(
         message_code,
         request_id=request_id if request_id is not None else current_request_id(),
diff --git a/grocery/tests/test_observability.py b/grocery/tests/test_observability.py
index 8a4d54e..e8c3239 100644
--- a/grocery/tests/test_observability.py
+++ b/grocery/tests/test_observability.py
@@ -6,7 +6,7 @@ from uuid import UUID
 
 import pytest
 from django.http import HttpResponse
-from django.test import RequestFactory
+from django.test import RequestFactory, override_settings
 
 from grocery.observability import (
     OBSERVABILITY_KEYS,
@@ -156,6 +156,7 @@ def test_filter_rejects_invalid_event_and_formatter_uses_safe_fallback() -> None
     assert "\n" not in line
 
 
+@override_settings(DEPLOY_VERSION=DEPLOY_VERSION)
 def test_log_event_emits_one_line_without_normal_log_message_data() -> None:
     output = io.StringIO()
     handler = logging.StreamHandler(output)
@@ -183,6 +184,7 @@ def test_log_event_emits_one_line_without_normal_log_message_data() -> None:
     assert len(lines) == 1
     payload = json.loads(lines[0])
     assert payload["message_code"] == "review.decision.recorded"
+    assert payload["deploy_version"] == DEPLOY_VERSION
     assert payload["command_run_id"] == COMMAND_RUN_ID
     assert payload["lifecycle_id"] == LIFECYCLE_ID
     assert set(payload).issubset(OBSERVABILITY_KEYS)
diff --git a/grocery/tests/test_production_settings.py b/grocery/tests/test_production_settings.py
index f56c5d1..1135745 100644
--- a/grocery/tests/test_production_settings.py
+++ b/grocery/tests/test_production_settings.py
@@ -8,6 +8,7 @@ from config.settings import env_positive_int, validate_production_environment
 _SAFE_SECRET = "x" * 50
 _SAFE_HOSTS = ("prices.example",)
 _SAFE_ORIGINS = ("https://prices.example",)
+_SAFE_DEPLOY_VERSION = "a" * 40
 
 
 def validate(
@@ -18,6 +19,7 @@ def validate(
     secret_key: str = _SAFE_SECRET,
     allowed_hosts: Sequence[str] = _SAFE_HOSTS,
     csrf_trusted_origins: Sequence[str] = _SAFE_ORIGINS,
+    deploy_version: str = _SAFE_DEPLOY_VERSION,
 ) -> None:
     validate_production_environment(
         environment,
@@ -26,6 +28,7 @@ def validate(
         secret_key=secret_key,
         allowed_hosts=allowed_hosts,
         csrf_trusted_origins=csrf_trusted_origins,
+        deploy_version=deploy_version,
     )
 
 
@@ -34,6 +37,7 @@ def production_environment() -> dict[str, str]:
         "DJANGO_SECRET_KEY": "present",
         "DJANGO_ALLOWED_HOSTS": "present",
         "DJANGO_CSRF_TRUSTED_ORIGINS": "present",
+        "DEPLOY_VERSION": "present",
     }
 
 
@@ -45,6 +49,7 @@ def test_debug_environment_keeps_local_development_defaults() -> None:
         secret_key="local",
         allowed_hosts=(),
         csrf_trusted_origins=(),
+        deploy_version="0000000",
     )
 
 
@@ -156,6 +161,22 @@ def test_validation_error_never_reflects_supplied_values() -> None:
     assert marker not in str(caught.value)
 
 
+def test_production_requires_exact_full_lowercase_release_sha() -> None:
+    missing = production_environment()
+    missing.pop("DEPLOY_VERSION")
+    with pytest.raises(
+        ImproperlyConfigured,
+        match="^production_deploy_version_required$",
+    ):
+        validate(missing)
+
+    with pytest.raises(
+        ImproperlyConfigured,
+        match="^production_deploy_version_required$",
+    ):
+        validate(production_environment(), deploy_version="G" * 40)
+
+
 def test_positive_integer_environment_bound_is_explicit() -> None:
     assert env_positive_int("MISSING_TEST_VALUE", 36, maximum=168) == 36
 


