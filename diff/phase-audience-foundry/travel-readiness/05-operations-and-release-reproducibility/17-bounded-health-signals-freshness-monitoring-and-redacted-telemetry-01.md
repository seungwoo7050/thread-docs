# 제한된 상태 신호·신선도 감시·비식별 텔레메트리

## `feat(operations): add no-record health boundary`

diff --git a/operations/__init__.py b/operations/__init__.py
new file mode 100644
index 0000000..22751f9
--- /dev/null
+++ b/operations/__init__.py
@@ -0,0 +1 @@
+"""Operational endpoints and commands."""
diff --git a/operations/apps.py b/operations/apps.py
new file mode 100644
index 0000000..1a16b61
--- /dev/null
+++ b/operations/apps.py
@@ -0,0 +1,6 @@
+from django.apps import AppConfig
+
+
+class OperationsConfig(AppConfig):
+    default_auto_field = "django.db.models.BigAutoField"
+    name = "operations"
diff --git a/operations/tests/__init__.py b/operations/tests/__init__.py
new file mode 100644
index 0000000..ba26c08
--- /dev/null
+++ b/operations/tests/__init__.py
@@ -0,0 +1 @@
+"""Operations tests."""
diff --git a/operations/tests/test_health.py b/operations/tests/test_health.py
new file mode 100644
index 0000000..08bbd7e
--- /dev/null
+++ b/operations/tests/test_health.py
@@ -0,0 +1,18 @@
+from django.test import SimpleTestCase
+
+
+class HealthTests(SimpleTestCase):
+    def test_health_is_queryless_and_stateless(self):
+        marker = "must-not-be-retained"
+
+        response = self.client.get(f"/healthz?destination={marker}", secure=True)
+
+        self.assertEqual(response.status_code, 200)
+        self.assertEqual(response.content, b"ok\n")
+        self.assertNotContains(response, marker)
+        self.assertEqual(response.headers["Cache-Control"], "no-store")
+        self.assertEqual(response.headers["X-Content-Type-Options"], "nosniff")
+        self.assertEqual(response.headers["X-Frame-Options"], "DENY")
+        self.assertEqual(response.headers["Referrer-Policy"], "no-referrer")
+        self.assertIn("max-age=31536000", response.headers["Strict-Transport-Security"])
+        self.assertFalse(response.cookies)
diff --git a/operations/urls.py b/operations/urls.py
new file mode 100644
index 0000000..632e590
--- /dev/null
+++ b/operations/urls.py
@@ -0,0 +1,5 @@
+from django.urls import path
+
+from .views import healthz
+
+urlpatterns = [path("healthz", healthz, name="healthz")]
diff --git a/operations/views.py b/operations/views.py
new file mode 100644
index 0000000..5393e62
--- /dev/null
+++ b/operations/views.py
@@ -0,0 +1,9 @@
+from django.http import HttpResponse
+
+
+def healthz(request) -> HttpResponse:
+    return HttpResponse(
+        "ok\n",
+        content_type="text/plain; charset=utf-8",
+        headers={"Cache-Control": "no-store"},
+    )
diff --git a/travel_readiness/settings.py b/travel_readiness/settings.py
index 6996bc3..8eafedd 100644
--- a/travel_readiness/settings.py
+++ b/travel_readiness/settings.py
@@ -34,6 +34,7 @@ ALLOWED_HOSTS = [
 ]
 
 INSTALLED_APPS = [
+    "operations",
     "django.contrib.admin",
     "django.contrib.auth",
     "django.contrib.contenttypes",
diff --git a/travel_readiness/urls.py b/travel_readiness/urls.py
index 3929a43..8650156 100644
--- a/travel_readiness/urls.py
+++ b/travel_readiness/urls.py
@@ -1,3 +1,3 @@
-from django.urls import path
+from django.urls import include, path
 
-urlpatterns: list = []
+urlpatterns = [path("", include("operations.urls"))]


