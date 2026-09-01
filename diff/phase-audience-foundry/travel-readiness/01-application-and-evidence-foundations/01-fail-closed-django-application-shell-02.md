## `build(web): add the public SSR boundary`

diff --git a/public_web/__init__.py b/public_web/__init__.py
new file mode 100644
index 0000000..6fb1ce6
--- /dev/null
+++ b/public_web/__init__.py
@@ -0,0 +1 @@
+"""Public, server-rendered, no-retention web boundary."""
diff --git a/public_web/apps.py b/public_web/apps.py
new file mode 100644
index 0000000..a123cc3
--- /dev/null
+++ b/public_web/apps.py
@@ -0,0 +1,6 @@
+from django.apps import AppConfig
+
+
+class PublicWebConfig(AppConfig):
+    default_auto_field = "django.db.models.BigAutoField"
+    name = "public_web"
diff --git a/travel_readiness/settings.py b/travel_readiness/settings.py
index c57d815..dc876ab 100644
--- a/travel_readiness/settings.py
+++ b/travel_readiness/settings.py
@@ -40,6 +40,7 @@ INSTALLED_APPS = [
     "entry_requirements",
     "travel_warnings",
     "reviews",
+    "public_web",
     "django.contrib.admin",
     "django.contrib.auth",
     "django.contrib.contenttypes",


## `fix(runtime): fail closed before serving`

diff --git a/operations/tests/test_runtime_config.py b/operations/tests/test_runtime_config.py
index 35dbd3f..c42213e 100644
--- a/operations/tests/test_runtime_config.py
+++ b/operations/tests/test_runtime_config.py
@@ -4,9 +4,11 @@ import logging
 import os
 from pathlib import Path
 import runpy
+import ssl
 import stat
 import subprocess
 import sys
+import tempfile
 from unittest.mock import patch
 
 from django.conf import settings
@@ -45,6 +47,11 @@ class RuntimeConfigTests(SimpleTestCase):
         self.assertIsNone(config.accesslog)
         self.assertEqual(config.forwarded_allow_ips, [])
         self.assertEqual(config.secure_scheme_headers, {})
+        context = config.ssl_context(
+            None,
+            lambda: ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER),
+        )
+        self.assertEqual(context.minimum_version, ssl.TLSVersion.TLSv1_2)
         self.assertEqual(config.worker_class_str, "sync")
         self.assertEqual(config.workers, 2)
         self.assertEqual(config.threads, 1)
@@ -131,7 +138,19 @@ class RuntimeConfigTests(SimpleTestCase):
         self.assertIn("travel_readiness.wsgi:application", script)
         self.assertIn("DJANGO_SETTINGS_MODULE=travel_readiness.settings", script)
         self.assertIn("GUNICORN_CMD_ARGS_FORBIDDEN", script)
-        self.assertIn('version("gunicorn") == "26.2.0"', script)
+        for dependency in (
+            '"Django": "5.2.17"',
+            '"gunicorn": "26.2.0"',
+            '"psycopg": "3.3.4"',
+            '"psycopg-binary": "3.3.4"',
+        ):
+            self.assertIn(dependency, script)
+        self.assertIn('call_command("check", deploy=True', script)
+        self.assertIn('call_command("migrate", check=True', script)
+        self.assertIn("unset MOFA_TRAVEL_ALARM_SERVICE_KEY", script)
+        self.assertIn("DIRECT_TLS_REQUIRED", script)
+        self.assertIn("TLS_MATERIAL_INVALID", script)
+        self.assertIn("LOOPBACK_BIND_REQUIRED", script)
         for forbidden in ("--access-logfile", "--reload", "--preload", "eval"):
             self.assertNotIn(forbidden, script)
 
@@ -154,6 +173,146 @@ class RuntimeConfigTests(SimpleTestCase):
         )
         self.assertEqual(result.stdout, "")
 
+    def test_startup_refuses_missing_direct_tls_before_runtime_import(self):
+        script_path = settings.BASE_DIR / "scripts" / "run-production"
+        environment = os.environ.copy()
+        environment.pop("GUNICORN_CMD_ARGS", None)
+        environment.pop("TRAVEL_READINESS_TLS_CERT_FILE", None)
+        environment.pop("TRAVEL_READINESS_TLS_KEY_FILE", None)
+        result = subprocess.run(
+            [str(script_path)],
+            cwd="/private/tmp",
+            env=environment,
+            capture_output=True,
+            text=True,
+            check=False,
+        )
+        self.assertEqual(result.returncode, 64)
+        self.assertEqual(result.stdout, "")
+        self.assertEqual(result.stderr, "startup_error=DIRECT_TLS_REQUIRED\n")
+
+    def test_startup_refuses_non_loopback_bind(self):
+        script_path = settings.BASE_DIR / "scripts" / "run-production"
+        environment = os.environ.copy()
+        environment.pop("GUNICORN_CMD_ARGS", None)
+        environment["TRAVEL_READINESS_BIND"] = "0.0.0.0:8000"
+        result = subprocess.run(
+            [str(script_path)],
+            cwd="/private/tmp",
+            env=environment,
+            capture_output=True,
+            text=True,
+            check=False,
+        )
+        self.assertEqual(result.returncode, 64)
+        self.assertEqual(result.stdout, "")
+        self.assertEqual(
+            result.stderr,
+            "startup_error=LOOPBACK_BIND_REQUIRED\n",
+        )
+
+    def test_startup_rejects_invalid_tls_material_with_fixed_error(self):
+        script_path = settings.BASE_DIR / "scripts" / "run-production"
+        with tempfile.TemporaryDirectory() as temporary:
+            temporary_path = Path(temporary)
+            cert_file = temporary_path / "invalid-cert.pem"
+            key_file = temporary_path / "invalid-key.pem"
+            cert_file.write_text("invalid-test-certificate\n")
+            key_file.write_text("invalid-test-key\n")
+            key_file.chmod(0o600)
+            environment = os.environ.copy()
+            environment.pop("GUNICORN_CMD_ARGS", None)
+            environment["TRAVEL_READINESS_TLS_CERT_FILE"] = str(cert_file)
+            environment["TRAVEL_READINESS_TLS_KEY_FILE"] = str(key_file)
+            result = subprocess.run(
+                [str(script_path)],
+                cwd="/private/tmp",
+                env=environment,
+                capture_output=True,
+                text=True,
+                check=False,
+            )
+        self.assertEqual(result.returncode, 64)
+        self.assertEqual(result.stdout, "")
+        self.assertEqual(
+            result.stderr,
+            "startup_error=TLS_MATERIAL_INVALID\n",
+        )
+
+    def test_startup_preflights_deployment_then_current_migrations(self):
+        script_path = settings.BASE_DIR / "scripts" / "run-production"
+        with tempfile.TemporaryDirectory() as temporary:
+            temporary_path = Path(temporary)
+            cert_file = temporary_path / "candidate-cert.pem"
+            key_file = temporary_path / "candidate-key.pem"
+            generated = subprocess.run(
+                [
+                    "openssl",
+                    "req",
+                    "-x509",
+                    "-newkey",
+                    "rsa:2048",
+                    "-nodes",
+                    "-days",
+                    "1",
+                    "-subj",
+                    "/CN=127.0.0.1",
+                    "-keyout",
+                    str(key_file),
+                    "-out",
+                    str(cert_file),
+                ],
+                capture_output=True,
+                text=True,
+                check=False,
+                timeout=15,
+            )
+            self.assertEqual(generated.returncode, 0)
+            key_file.chmod(0o600)
+            base_environment = os.environ.copy()
+            for name in (
+                "GUNICORN_CMD_ARGS",
+                "TRAVEL_READINESS_BUILD",
+                "MOFA_TRAVEL_ALARM_SERVICE_KEY",
+            ):
+                base_environment.pop(name, None)
+            base_environment.update(
+                {
+                    "TRAVEL_READINESS_SECRET_KEY": (
+                        "test-only-startup-secret-0123456789-"
+                        "ABCDEFGHIJKLMNOPQRSTUVWXYZ-not-a-credential"
+                    ),
+                    "TRAVEL_READINESS_DB_PASSWORD": (
+                        "test-only-database-value"
+                    ),
+                    "TRAVEL_READINESS_DB_HOST": "127.0.0.1",
+                    "TRAVEL_READINESS_DB_PORT": "1",
+                    "TRAVEL_READINESS_ALLOWED_HOSTS": "candidate.invalid",
+                    "TRAVEL_READINESS_TLS_CERT_FILE": str(cert_file),
+                    "TRAVEL_READINESS_TLS_KEY_FILE": str(key_file),
+                }
+            )
+            cases = (
+                ("0", "startup_error=DEPLOYMENT_CHECK_FAILED\n"),
+                ("1", "startup_error=MIGRATION_CHECK_FAILED\n"),
+            )
+            for https_flag, expected_error in cases:
+                with self.subTest(https_flag=https_flag):
+                    environment = base_environment.copy()
+                    environment["TRAVEL_READINESS_HTTPS"] = https_flag
+                    result = subprocess.run(
+                        [str(script_path)],
+                        cwd="/private/tmp",
+                        env=environment,
+                        capture_output=True,
+                        text=True,
+                        check=False,
+                        timeout=15,
+                    )
+                    self.assertEqual(result.returncode, 78)
+                    self.assertEqual(result.stdout, "")
+                    self.assertEqual(result.stderr, expected_error)
+
     def test_collectstatic_build_mode_needs_no_runtime_credentials(self):
         environment = {
             "PATH": os.environ.get("PATH", ""),
diff --git a/runtime/gunicorn.conf.py b/runtime/gunicorn.conf.py
index f3974d6..8082bdc 100644
--- a/runtime/gunicorn.conf.py
+++ b/runtime/gunicorn.conf.py
@@ -1,6 +1,7 @@
 """Credential-free, query-free Gunicorn process defaults."""
 
 import os
+import ssl
 
 
 tls_cert_file = os.environ.get("TRAVEL_READINESS_TLS_CERT_FILE", "")
@@ -27,3 +28,9 @@ forwarded_allow_ips = ""
 secure_scheme_headers = {}
 certfile = tls_cert_file or None
 keyfile = tls_key_file or None
+
+
+def ssl_context(conf, default_ssl_context_factory):
+    context = default_ssl_context_factory()
+    context.minimum_version = ssl.TLSVersion.TLSv1_2
+    return context
diff --git a/scripts/run-production b/scripts/run-production
index 194b962..0ed9f4d 100755
--- a/scripts/run-production
+++ b/scripts/run-production
@@ -5,8 +5,45 @@ if [ "${GUNICORN_CMD_ARGS+x}" = x ]; then
   printf '%s\n' 'startup_error=GUNICORN_CMD_ARGS_FORBIDDEN' >&2
   exit 64
 fi
+unset MOFA_TRAVEL_ALARM_SERVICE_KEY
+unset TRAVEL_READINESS_BACKUP_RESTORE_ADMIN_PASSWORD
+unset TRAVEL_READINESS_BACKUP_ROLE_PASSWORD
 
-script_dir=$(CDPATH= cd -- "$(dirname -- "$0")" && pwd)
+bind=${TRAVEL_READINESS_BIND-127.0.0.1:8000}
+case "$bind" in
+  127.0.0.1:*) bind_port=${bind#127.0.0.1:} ;;
+  *)
+    printf '%s\n' 'startup_error=LOOPBACK_BIND_REQUIRED' >&2
+    exit 64
+    ;;
+esac
+case "$bind_port" in ''|*[!0-9]*)
+  printf '%s\n' 'startup_error=LOOPBACK_BIND_REQUIRED' >&2
+  exit 64
+  ;;
+esac
+if [ "${#bind_port}" -gt 5 ] || [ "$bind_port" -lt 1 ] \
+  || [ "$bind_port" -gt 65535 ]; then
+  printf '%s\n' 'startup_error=LOOPBACK_BIND_REQUIRED' >&2
+  exit 64
+fi
+
+tls_cert=${TRAVEL_READINESS_TLS_CERT_FILE-}
+tls_key=${TRAVEL_READINESS_TLS_KEY_FILE-}
+case "$tls_cert:$tls_key" in
+  /*:/*) ;;
+  *)
+    printf '%s\n' 'startup_error=DIRECT_TLS_REQUIRED' >&2
+    exit 64
+    ;;
+esac
+if [ ! -f "$tls_cert" ] || [ -L "$tls_cert" ] \
+  || [ ! -f "$tls_key" ] || [ -L "$tls_key" ]; then
+  printf '%s\n' 'startup_error=DIRECT_TLS_REQUIRED' >&2
+  exit 64
+fi
+
+script_dir=$(CDPATH='' cd -- "$(dirname -- "$0")" && pwd)
 project_dir=$(dirname -- "$script_dir")
 python_bin="$project_dir/.venv/bin/python"
 
@@ -14,18 +51,80 @@ if [ ! -x "$python_bin" ]; then
   printf '%s\n' 'startup_error=FROZEN_ENV_MISSING' >&2
   exit 64
 fi
-if ! "$python_bin" -I -c 'import sys; raise SystemExit(0 if sys.version_info[:3] == (3, 14, 7) else 1)'; then
+if ! "$python_bin" -I -c 'import sys; raise SystemExit(0 if sys.version_info[:3] == (3, 14, 7) else 1)' \
+  >/dev/null 2>&1; then
   printf '%s\n' 'startup_error=PYTHON_VERSION_MISMATCH' >&2
   exit 64
 fi
-if ! "$python_bin" -I -c 'from importlib.metadata import version; raise SystemExit(0 if version("gunicorn") == "26.2.0" else 1)'; then
-  printf '%s\n' 'startup_error=GUNICORN_VERSION_MISMATCH' >&2
+if ! "$python_bin" -I -c '
+import os
+import stat
+import sys
+cert = os.lstat(sys.argv[1])
+key = os.lstat(sys.argv[2])
+valid = (
+    stat.S_ISREG(cert.st_mode)
+    and stat.S_ISREG(key.st_mode)
+    and cert.st_uid == os.geteuid()
+    and key.st_uid == os.geteuid()
+    and key.st_mode & 0o077 == 0
+)
+raise SystemExit(0 if valid else 1)
+' "$tls_cert" "$tls_key" >/dev/null 2>&1; then
+  printf '%s\n' 'startup_error=DIRECT_TLS_REQUIRED' >&2
+  exit 64
+fi
+if ! "$python_bin" -I -c '
+import ssl
+import sys
+context = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
+context.minimum_version = ssl.TLSVersion.TLSv1_2
+context.load_cert_chain(sys.argv[1], sys.argv[2])
+' "$tls_cert" "$tls_key" >/dev/null 2>&1; then
+  printf '%s\n' 'startup_error=TLS_MATERIAL_INVALID' >&2
+  exit 64
+fi
+if ! "$python_bin" -I -c '
+from importlib.metadata import version
+expected = {
+    "Django": "5.2.17",
+    "gunicorn": "26.2.0",
+    "psycopg": "3.3.4",
+    "psycopg-binary": "3.3.4",
+}
+if any(version(name) != wanted for name, wanted in expected.items()):
+    raise SystemExit(1)
+import django, gunicorn, psycopg  # noqa: F401
+' >/dev/null 2>&1; then
+  printf '%s\n' 'startup_error=DEPENDENCY_VERSION_MISMATCH' >&2
   exit 64
 fi
 
 cd "$project_dir"
 DJANGO_SETTINGS_MODULE=travel_readiness.settings
 export DJANGO_SETTINGS_MODULE
+if ! "$python_bin" -I -c '
+import sys
+sys.path.insert(0, sys.argv[1])
+import django
+django.setup()
+from django.core.management import call_command
+call_command("check", deploy=True, fail_level="WARNING", verbosity=0)
+' "$project_dir" >/dev/null 2>&1; then
+  printf '%s\n' 'startup_error=DEPLOYMENT_CHECK_FAILED' >&2
+  exit 78
+fi
+if ! "$python_bin" -I -c '
+import sys
+sys.path.insert(0, sys.argv[1])
+import django
+django.setup()
+from django.core.management import call_command
+call_command("migrate", check=True, interactive=False, verbosity=0)
+' "$project_dir" >/dev/null 2>&1; then
+  printf '%s\n' 'startup_error=MIGRATION_CHECK_FAILED' >&2
+  exit 78
+fi
 exec "$python_bin" -I -m gunicorn \
   --chdir "$project_dir" \
   --config "$project_dir/runtime/gunicorn.conf.py" \
