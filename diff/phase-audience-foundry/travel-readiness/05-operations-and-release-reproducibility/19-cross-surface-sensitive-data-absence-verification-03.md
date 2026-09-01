## `test(ops): isolate pinned subprocess runtime`

diff --git a/operations/tests/subprocess_support.py b/operations/tests/subprocess_support.py
new file mode 100644
index 0000000..90fed6f
--- /dev/null
+++ b/operations/tests/subprocess_support.py
@@ -0,0 +1,58 @@
+from __future__ import annotations
+
+import os
+from pathlib import Path
+import sys
+
+
+REPOSITORY_ROOT = Path(__file__).resolve().parents[2]
+
+
+def dependency_site_packages() -> Path:
+    candidate = (
+        REPOSITORY_ROOT
+        / ".venv"
+        / "lib"
+        / f"python{sys.version_info.major}.{sys.version_info.minor}"
+        / "site-packages"
+    )
+    try:
+        resolved = candidate.resolve(strict=True)
+    except OSError as error:
+        raise AssertionError(
+            "pinned dependency site-packages are unavailable"
+        ) from error
+    if not resolved.is_dir():
+        raise AssertionError("pinned dependency site-packages are unavailable")
+    return resolved
+
+
+def dependency_pythonpath(project_root: Path) -> str:
+    try:
+        resolved_project = project_root.resolve(strict=True)
+    except OSError as error:
+        raise AssertionError("subprocess project root is unavailable") from error
+    if not resolved_project.is_dir():
+        raise AssertionError("subprocess project root is unavailable")
+    return os.pathsep.join(
+        (str(dependency_site_packages()), str(resolved_project))
+    )
+
+
+def dependency_venv_python() -> Path:
+    candidate = (
+        REPOSITORY_ROOT
+        / ".venv"
+        / "bin"
+        / f"python{sys.version_info.major}.{sys.version_info.minor}"
+    )
+    try:
+        resolved = candidate.resolve(strict=True)
+        active = Path(sys.executable).resolve(strict=True)
+    except OSError as error:
+        raise AssertionError("pinned dependency Python is unavailable") from error
+    if not candidate.is_file() or resolved != active:
+        raise AssertionError(
+            "pinned dependency Python does not match the test runner"
+        )
+    return candidate
diff --git a/operations/tests/test_browser_acceptance_harness.py b/operations/tests/test_browser_acceptance_harness.py
index c98e97f..efa4bbe 100644
--- a/operations/tests/test_browser_acceptance_harness.py
+++ b/operations/tests/test_browser_acceptance_harness.py
@@ -1214,7 +1214,7 @@ class BrowserAcceptanceHarnessTests(unittest.TestCase):
             f"os.write(2,b'e'*{payload_size})"
         )
         returncode, stdout, stderr = acceptance._bounded_process(
-            [sys.executable, "-I", "-c", code],
+            [sys.executable, "-I", "-B", "-c", code],
             cwd=acceptance.REPOSITORY_ROOT,
             environment={"PATH": "/usr/bin:/bin"},
             timeout=10,
@@ -1228,7 +1228,7 @@ class BrowserAcceptanceHarnessTests(unittest.TestCase):
         code = f"import os;os.write(1,b'x'*{acceptance.MAX_CLI_OUTPUT_BYTES + 1})"
         with self.assertRaises(acceptance.AcceptanceFailure) as caught:
             acceptance._bounded_process(
-                [sys.executable, "-I", "-c", code],
+                [sys.executable, "-I", "-B", "-c", code],
                 cwd=acceptance.REPOSITORY_ROOT,
                 environment={"PATH": "/usr/bin:/bin"},
                 timeout=10,
diff --git a/operations/tests/test_deployment_safety.py b/operations/tests/test_deployment_safety.py
index b097cdd..57a550c 100644
--- a/operations/tests/test_deployment_safety.py
+++ b/operations/tests/test_deployment_safety.py
@@ -14,6 +14,7 @@ from django.http import HttpResponse
 from django.test import Client, SimpleTestCase, override_settings
 from django.urls import NoReverseMatch, path, reverse
 
+from operations.tests.subprocess_support import dependency_pythonpath
 from travel_readiness.settings import redact_django_record
 
 
@@ -62,9 +63,11 @@ handler500 = "operations.error_handlers.server_error"
 
 class SettingsProcessMixin:
     def settings_environment(self, **overrides):
+        repository_root = Path(__file__).resolve().parents[2]
         environment = {
-            "PATH": os.environ.get("PATH", ""),
-            "PYTHONPATH": str(Path(__file__).resolve().parents[2]),
+            "PATH": "/usr/bin:/bin",
+            "PYTHONDONTWRITEBYTECODE": "1",
+            "PYTHONPATH": dependency_pythonpath(repository_root),
             "DJANGO_SETTINGS_MODULE": "travel_readiness.settings",
             "TRAVEL_READINESS_SECRET_KEY": (
                 "test-only-not-a-credential-" * 3
@@ -78,7 +81,7 @@ class SettingsProcessMixin:
 
     def run_python(self, code, **overrides):
         return subprocess.run(
-            [sys.executable, "-c", code],
+            [sys.executable, "-B", "-c", code],
             cwd=Path(__file__).resolve().parents[2],
             env=self.settings_environment(**overrides),
             capture_output=True,
@@ -166,6 +169,7 @@ class DeploymentSettingsTests(SettingsProcessMixin, SimpleTestCase):
         result = subprocess.run(
             [
                 sys.executable,
+                "-B",
                 str(Path(__file__).resolve().parents[2] / "manage.py"),
                 "check",
                 "--deploy",
@@ -185,6 +189,7 @@ class DeploymentSettingsTests(SettingsProcessMixin, SimpleTestCase):
         result = subprocess.run(
             [
                 sys.executable,
+                "-B",
                 str(Path(__file__).resolve().parents[2] / "manage.py"),
                 "check",
                 "--deploy",
@@ -204,6 +209,7 @@ class DeploymentSettingsTests(SettingsProcessMixin, SimpleTestCase):
         result = subprocess.run(
             [
                 sys.executable,
+                "-B",
                 str(Path(__file__).resolve().parents[2] / "manage.py"),
                 "check",
                 "--deploy",
diff --git a/operations/tests/test_postgresql_backup_restore.py b/operations/tests/test_postgresql_backup_restore.py
index aeabf5c..3ead59c 100644
--- a/operations/tests/test_postgresql_backup_restore.py
+++ b/operations/tests/test_postgresql_backup_restore.py
@@ -604,14 +604,20 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
         leader = (
             "import signal,subprocess,sys,time; "
             "signal.signal(signal.SIGTERM, signal.SIG_IGN); "
-            f"subprocess.Popen([sys.executable, '-c', {grandchild!r}], "
+            f"subprocess.Popen([sys.executable, '-B', '-c', {grandchild!r}], "
             "stdin=subprocess.DEVNULL, stdout=subprocess.DEVNULL, "
             "stderr=subprocess.DEVNULL); "
             "print('ready', flush=True); "
             "time.sleep(60)"
         )
         process = subprocess.Popen(
-            [sys.executable, "-c", leader],
+            [sys.executable, "-B", "-c", leader],
+            env={
+                "LANG": "C",
+                "LC_ALL": "C",
+                "PATH": "/usr/bin:/bin",
+                "PYTHONDONTWRITEBYTECODE": "1",
+            },
             stdin=subprocess.DEVNULL,
             stdout=subprocess.PIPE,
             stderr=subprocess.PIPE,
diff --git a/operations/tests/test_release_artifact.py b/operations/tests/test_release_artifact.py
index 9e0dab2..2c829ce 100644
--- a/operations/tests/test_release_artifact.py
+++ b/operations/tests/test_release_artifact.py
@@ -14,6 +14,11 @@ import tempfile
 import textwrap
 import unittest
 
+from operations.tests.subprocess_support import (
+    dependency_pythonpath,
+    dependency_venv_python,
+)
+
 
 class ReleaseArtifactTests(unittest.TestCase):
     @staticmethod
@@ -37,6 +42,7 @@ class ReleaseArtifactTests(unittest.TestCase):
         self.temporary = tempfile.TemporaryDirectory()
         self.repository = Path(self.temporary.name) / "candidate"
         self.repository.mkdir()
+        self.child_python = dependency_venv_python()
         self.source_builder = (
             Path(__file__).resolve().parents[2] / "scripts" / "build-release"
         )
@@ -153,7 +159,7 @@ class ReleaseArtifactTests(unittest.TestCase):
             [ -z "${{MOFA_TRAVEL_ALARM_SERVICE_KEY:-}}" ] || exit 97
             [ -z "${{UNRELATED_RELEASE_PARENT_MARKER:-}}" ] || exit 97
             [ "${{PATH:-}}" = "/usr/bin:/bin" ] || exit 97
-            [ "${{UV_PYTHON:-}}" = "{sys.executable}" ] || exit 97
+            [ "${{UV_PYTHON:-}}" = "{self.child_python}" ] || exit 97
             if [ "${{1:-}}" = "--version" ]; then
                 printf '%s\\n' 'uv 0.12.6 (fixture build metadata)'
                 exit 0
@@ -180,12 +186,15 @@ class ReleaseArtifactTests(unittest.TestCase):
             "DATA_GO_KR_SERVICE_KEY": self.secret_marker,
             "MOFA_TRAVEL_ALARM_SERVICE_KEY": self.secret_marker,
             "PATH": "/private/untrusted-parent-path",
+            "PYTHONDONTWRITEBYTECODE": "1",
+            "PYTHONPATH": dependency_pythonpath(self.repository),
             "TZ": "UTC",
             "UNRELATED_RELEASE_PARENT_MARKER": self.secret_marker,
         }
         return subprocess.run(
             [
-                sys.executable,
+                str(self.child_python),
+                "-B",
                 str(self.repository / "scripts" / "build-release"),
                 "--output-dir",
                 f"output/{output_name}",
diff --git a/operations/tests/test_runtime_config.py b/operations/tests/test_runtime_config.py
index 98a70e0..0bb72a8 100644
--- a/operations/tests/test_runtime_config.py
+++ b/operations/tests/test_runtime_config.py
@@ -14,7 +14,6 @@ import ssl
 import stat
 import subprocess
 import sys
-import sysconfig
 import tempfile
 import tomllib
 from unittest.mock import patch
@@ -27,6 +26,10 @@ from operations.gunicorn_logging import (
     FixedGunicornLogger,
     _RedactedGunicornFilter,
 )
+from operations.tests.subprocess_support import (
+    dependency_pythonpath,
+    dependency_site_packages,
+)
 from operations.telemetry import bounded_release_sha
 
 
@@ -416,7 +419,7 @@ class _RuntimeReleaseFixture:
             record.write_text(
                 "\n".join(record_rows) + "\n", encoding="utf-8"
             )
-        actual_site_packages = Path(sysconfig.get_path("purelib")).resolve()
+        actual_site_packages = dependency_site_packages()
         for package_name in (
             "asgiref",
             "django",
@@ -1011,6 +1014,9 @@ class RuntimeConfigTests(SimpleTestCase):
                     "LANG": "C",
                     "LC_ALL": "C",
                     "PATH": "/usr/bin:/bin",
+                    "PYTHONPYCACHEPREFIX": str(
+                        Path(temporary).resolve() / "pycache"
+                    ),
                     "TZ": "UTC",
                 },
                 capture_output=True,
@@ -2680,12 +2686,15 @@ CSRF_COOKIE_SECURE = True
 
     def test_collectstatic_build_mode_needs_no_runtime_credentials(self):
         environment = {
-            "PATH": os.environ.get("PATH", ""),
+            "PATH": "/usr/bin:/bin",
+            "PYTHONDONTWRITEBYTECODE": "1",
+            "PYTHONPATH": dependency_pythonpath(settings.BASE_DIR),
             "TRAVEL_READINESS_BUILD": "1",
         }
         result = subprocess.run(
             [
                 sys.executable,
+                "-B",
                 str(settings.BASE_DIR / "manage.py"),
                 "collectstatic",
                 "--noinput",
@@ -2704,13 +2713,14 @@ CSRF_COOKIE_SECURE = True
 
     def test_build_mode_cannot_start_wsgi(self):
         environment = {
-            "PATH": os.environ.get("PATH", ""),
-            "PYTHONPATH": str(settings.BASE_DIR),
+            "PATH": "/usr/bin:/bin",
+            "PYTHONDONTWRITEBYTECODE": "1",
+            "PYTHONPATH": dependency_pythonpath(settings.BASE_DIR),
             "DJANGO_SETTINGS_MODULE": "travel_readiness.settings",
             "TRAVEL_READINESS_BUILD": "1",
         }
         result = subprocess.run(
-            [sys.executable, "-c", "import travel_readiness.wsgi"],
+            [sys.executable, "-B", "-c", "import travel_readiness.wsgi"],
             cwd="/private/tmp",
             env=environment,
             capture_output=True,
