## `build(web): pin the production process`

diff --git a/.gitignore b/.gitignore
index 7629239..21bd0da 100644
--- a/.gitignore
+++ b/.gitignore
@@ -11,10 +11,10 @@ coverage/
 htmlcov/
 dist/
 build/
+staticfiles/
 node_modules/
 tmp/
 .cache/
 .mypy_cache/
 .pytest_cache/
 .ruff_cache/
-
diff --git a/THIRD_PARTY_NOTICES.md b/THIRD_PARTY_NOTICES.md
index 6fd3de0..d165d36 100644
--- a/THIRD_PARTY_NOTICES.md
+++ b/THIRD_PARTY_NOTICES.md
@@ -14,14 +14,17 @@ vendoring하지 않습니다.
 
 uv macOS arm64 archive SHA-256은
 `14b459d51ea2e71eeba28c45a268c922bdf8607fc6455e3f40b4e082895d160d`이며 release
-attestation을 GitHub CLI와 public Sigstore Rekor로 검증했습니다. PostgreSQL package/image,
-production Linux target과 WSGI server는 아직 선택하거나 내려받지 않았습니다.
+attestation을 GitHub CLI와 public Sigstore Rekor로 검증했습니다. PostgreSQL package/image와
+production Linux target은 아직 선택하거나 내려받지 않았습니다. WSGI process는 Python
+3.10+를 지원하는 Gunicorn 26.2.0 sync worker로 고정했으며 실제 platform worker 수와
+resource limit은 배포 checkpoint에 남깁니다.
 
 ## lock에 포함된 Python distribution
 
 | dependency | 관계 | license | upstream | 사용 목적 |
 | --- | --- | --- | --- | --- |
 | Django 5.2.17 | direct | BSD-3-Clause | `https://github.com/django/django` | server-rendered web, models, migrations, Admin |
+| Gunicorn 26.2.0 | direct | MIT | `https://github.com/benoitc/gunicorn` | production WSGI process; access log disabled |
 | psycopg 3.3.4 | direct | LGPL-3.0-only | `https://github.com/psycopg/psycopg` | Django PostgreSQL adapter |
 | psycopg-binary 3.3.4 | direct extra | LGPL-3.0-only | `https://github.com/psycopg/psycopg` | pinned CPython adapter binary for development verification |
 | asgiref 3.12.1 | Django transitive | BSD-3-Clause | `https://github.com/django/asgiref` | Django sync/async compatibility |
@@ -33,7 +36,8 @@ Django universal wheel은
 `f04fb3b36ee119e1af4fa1d397d5fd6cf12700f49321e84d4f4c642c5b1973db`, psycopg
 universal wheel은 `b6bbc25ccf05c8fad3b061d9db2ef0909a555171b84b07f29458a447253d679a`,
 psycopg-binary CPython 3.14 macOS arm64 wheel은
-`1fbaa292a3c8bb61b45df1ad3da1908ccee7cb889db9425e3557d9e34e2a4829`입니다.
+`1fbaa292a3c8bb61b45df1ad3da1908ccee7cb889db9425e3557d9e34e2a4829`, Gunicorn universal wheel은
+`bd249d0b3f7972f7432f0a6b6ff3b3ee2d129f70cd1ff6c09a9dd9e29a2b88e3`입니다.
 
 BSD와 Apache 재배포 시 해당 copyright/license/notice를 유지해야 합니다. LGPL 구성요소를
 배포하면 LGPL/GPL license 사본, 고지, 수정·재링크 조건을 충족해야 합니다. 설치된 macOS
diff --git a/operations/gunicorn_logging.py b/operations/gunicorn_logging.py
new file mode 100644
index 0000000..8c8811f
--- /dev/null
+++ b/operations/gunicorn_logging.py
@@ -0,0 +1,37 @@
+"""Fixed-shape Gunicorn lifecycle logging without request material."""
+
+from __future__ import annotations
+
+import json
+import logging
+
+from gunicorn.glogging import Logger
+
+
+class _RedactedGunicornFilter(logging.Filter):
+    def filter(self, record: logging.LogRecord) -> bool:
+        record.msg = json.dumps(
+            {
+                "event": "gunicorn",
+                "level": record.levelname,
+                "redacted": True,
+            },
+            separators=(",", ":"),
+        )
+        record.args = ()
+        record.exc_info = None
+        record.exc_text = None
+        record.stack_info = None
+        return True
+
+
+class FixedGunicornLogger(Logger):
+    """Retain process severity while discarding URI, IP, header and errors."""
+
+    def setup(self, cfg):
+        super().setup(cfg)
+        redaction_filter = _RedactedGunicornFilter()
+        for handler in self.error_log.handlers:
+            if getattr(handler, "_gunicorn", False):
+                handler.setFormatter(logging.Formatter("%(message)s"))
+                handler.addFilter(redaction_filter)
diff --git a/operations/tests/test_runtime_config.py b/operations/tests/test_runtime_config.py
new file mode 100644
index 0000000..35dbd3f
--- /dev/null
+++ b/operations/tests/test_runtime_config.py
@@ -0,0 +1,200 @@
+import io
+import json
+import logging
+import os
+from pathlib import Path
+import runpy
+import stat
+import subprocess
+import sys
+from unittest.mock import patch
+
+from django.conf import settings
+from django.test import SimpleTestCase
+from gunicorn.app.wsgiapp import WSGIApplication
+
+from operations.gunicorn_logging import (
+    FixedGunicornLogger,
+    _RedactedGunicornFilter,
+)
+
+
+class RuntimeConfigTests(SimpleTestCase):
+    def setUp(self):
+        self.config_path = settings.BASE_DIR / "runtime" / "gunicorn.conf.py"
+
+    def test_static_root_is_outside_tracked_source_assets(self):
+        self.assertEqual(settings.STATIC_ROOT, settings.BASE_DIR / "staticfiles")
+        self.assertFalse(
+            Path(settings.STATIC_ROOT).is_relative_to(
+                settings.BASE_DIR / "public_web" / "static"
+            )
+        )
+
+    def test_gunicorn_configuration_disables_access_and_proxy_trust(self):
+        arguments = [
+            "gunicorn",
+            "--config",
+            str(self.config_path),
+            "travel_readiness.wsgi:application",
+        ]
+        with patch("sys.argv", arguments):
+            application = WSGIApplication()
+        config = application.cfg
+
+        self.assertIsNone(config.accesslog)
+        self.assertEqual(config.forwarded_allow_ips, [])
+        self.assertEqual(config.secure_scheme_headers, {})
+        self.assertEqual(config.worker_class_str, "sync")
+        self.assertEqual(config.workers, 2)
+        self.assertEqual(config.threads, 1)
+        self.assertEqual(config.limit_request_line, 2048)
+        self.assertEqual(config.limit_request_fields, 50)
+        self.assertIs(config.logger_class, FixedGunicornLogger)
+
+    def test_gunicorn_error_records_drop_request_and_exception_material(self):
+        marker = "private-query-ip-header-and-exception-marker"
+        record = logging.LogRecord(
+            "gunicorn.error",
+            logging.ERROR,
+            __file__,
+            1,
+            "Error handling request %s from %s",
+            (marker, marker),
+            (RuntimeError, RuntimeError(marker), None),
+        )
+        record.stack_info = marker
+
+        self.assertTrue(_RedactedGunicornFilter().filter(record))
+        payload = json.loads(record.getMessage())
+        self.assertEqual(
+            payload,
+            {"event": "gunicorn", "level": "ERROR", "redacted": True},
+        )
+        self.assertNotIn(marker, repr(record.__dict__))
+
+    def test_configured_gunicorn_handler_emits_only_fixed_json(self):
+        marker = "synthetic-uri-ip-header-marker"
+        arguments = [
+            "gunicorn",
+            "--config",
+            str(self.config_path),
+            "travel_readiness.wsgi:application",
+        ]
+        captured = io.StringIO()
+        with patch("sys.argv", arguments), patch("sys.stderr", captured):
+            application = WSGIApplication()
+            logger = application.cfg.logger_class(application.cfg)
+            logger.warning("Invalid request from %s: %s", marker, marker)
+        self.assertEqual(
+            captured.getvalue(),
+            '{"event":"gunicorn","level":"WARNING","redacted":true}\n',
+        )
+        self.assertNotIn(marker, captured.getvalue())
+
+    def test_direct_tls_requires_an_exact_pair_and_keeps_proxy_trust_empty(self):
+        cert_name = "/private/tmp/synthetic-cert.pem"
+        key_name = "/private/tmp/synthetic-key.pem"
+        with patch.dict(
+            os.environ,
+            {
+                "TRAVEL_READINESS_TLS_CERT_FILE": cert_name,
+                "TRAVEL_READINESS_TLS_KEY_FILE": key_name,
+            },
+            clear=False,
+        ):
+            config = runpy.run_path(str(self.config_path))
+        self.assertEqual(config["certfile"], cert_name)
+        self.assertEqual(config["keyfile"], key_name)
+        self.assertEqual(config["forwarded_allow_ips"], "")
+        self.assertEqual(config["secure_scheme_headers"], {})
+
+        with patch.dict(
+            os.environ,
+            {"TRAVEL_READINESS_TLS_CERT_FILE": cert_name},
+            clear=False,
+        ):
+            os.environ.pop("TRAVEL_READINESS_TLS_KEY_FILE", None)
+            with self.assertRaisesRegex(
+                RuntimeError,
+                "TLS certificate and key file must be configured together",
+            ):
+                runpy.run_path(str(self.config_path))
+
+    def test_startup_script_uses_only_the_pinned_config_and_wsgi_app(self):
+        script_path = settings.BASE_DIR / "scripts" / "run-production"
+        script = script_path.read_text(encoding="utf-8")
+        self.assertEqual(stat.S_IMODE(script_path.stat().st_mode), 0o755)
+        self.assertIn('"$python_bin" -I -m gunicorn', script)
+        self.assertIn('--chdir "$project_dir"', script)
+        self.assertIn('"$project_dir/runtime/gunicorn.conf.py"', script)
+        self.assertIn("travel_readiness.wsgi:application", script)
+        self.assertIn("DJANGO_SETTINGS_MODULE=travel_readiness.settings", script)
+        self.assertIn("GUNICORN_CMD_ARGS_FORBIDDEN", script)
+        self.assertIn('version("gunicorn") == "26.2.0"', script)
+        for forbidden in ("--access-logfile", "--reload", "--preload", "eval"):
+            self.assertNotIn(forbidden, script)
+
+    def test_startup_rejects_inherited_gunicorn_arguments_from_any_cwd(self):
+        script_path = settings.BASE_DIR / "scripts" / "run-production"
+        environment = os.environ.copy()
+        environment["GUNICORN_CMD_ARGS"] = "--access-logfile=-"
+        result = subprocess.run(
+            [str(script_path)],
+            cwd="/private/tmp",
+            env=environment,
+            capture_output=True,
+            text=True,
+            check=False,
+        )
+        self.assertEqual(result.returncode, 64)
+        self.assertEqual(
+            result.stderr,
+            "startup_error=GUNICORN_CMD_ARGS_FORBIDDEN\n",
+        )
+        self.assertEqual(result.stdout, "")
+
+    def test_collectstatic_build_mode_needs_no_runtime_credentials(self):
+        environment = {
+            "PATH": os.environ.get("PATH", ""),
+            "TRAVEL_READINESS_BUILD": "1",
+        }
+        result = subprocess.run(
+            [
+                sys.executable,
+                str(settings.BASE_DIR / "manage.py"),
+                "collectstatic",
+                "--noinput",
+                "--dry-run",
+                "--verbosity",
+                "0",
+            ],
+            cwd=settings.BASE_DIR,
+            env=environment,
+            capture_output=True,
+            text=True,
+            check=False,
+        )
+        self.assertEqual(result.returncode, 0, result.stderr)
+        self.assertEqual(result.stderr, "")
+
+    def test_build_mode_cannot_start_wsgi(self):
+        environment = {
+            "PATH": os.environ.get("PATH", ""),
+            "PYTHONPATH": str(settings.BASE_DIR),
+            "DJANGO_SETTINGS_MODULE": "travel_readiness.settings",
+            "TRAVEL_READINESS_BUILD": "1",
+        }
+        result = subprocess.run(
+            [sys.executable, "-c", "import travel_readiness.wsgi"],
+            cwd="/private/tmp",
+            env=environment,
+            capture_output=True,
+            text=True,
+            check=False,
+        )
+        self.assertNotEqual(result.returncode, 0)
+        self.assertNotIn(
+            "credential-free-static-build-only",
+            result.stdout + result.stderr,
+        )
diff --git a/pyproject.toml b/pyproject.toml
index dfe238c..42c2e15 100644
--- a/pyproject.toml
+++ b/pyproject.toml
@@ -5,6 +5,7 @@ description = "Evidence-backed travel-readiness publication service"
 requires-python = "==3.14.7"
 dependencies = [
     "Django==5.2.17",
+    "gunicorn==26.2.0",
     "psycopg[binary]==3.3.4",
 ]
 
diff --git a/runtime/gunicorn.conf.py b/runtime/gunicorn.conf.py
new file mode 100644
index 0000000..f3974d6
--- /dev/null
+++ b/runtime/gunicorn.conf.py
@@ -0,0 +1,29 @@
+"""Credential-free, query-free Gunicorn process defaults."""
+
+import os
+
+
+tls_cert_file = os.environ.get("TRAVEL_READINESS_TLS_CERT_FILE", "")
+tls_key_file = os.environ.get("TRAVEL_READINESS_TLS_KEY_FILE", "")
+if bool(tls_cert_file) != bool(tls_key_file):
+    raise RuntimeError("TLS certificate and key file must be configured together")
+
+bind = os.environ.get("TRAVEL_READINESS_BIND", "127.0.0.1:8000")
+workers = 2
+worker_class = "sync"
+threads = 1
+timeout = 30
+graceful_timeout = 30
+keepalive = 2
+accesslog = None
+errorlog = "-"
+loglevel = "info"
+logger_class = "operations.gunicorn_logging.FixedGunicornLogger"
+capture_output = False
+limit_request_line = 2048
+limit_request_fields = 50
+limit_request_field_size = 8190
+forwarded_allow_ips = ""
+secure_scheme_headers = {}
+certfile = tls_cert_file or None
+keyfile = tls_key_file or None
diff --git a/runtime/versions.toml b/runtime/versions.toml
index 2295e2e..e2bd57e 100644
--- a/runtime/versions.toml
+++ b/runtime/versions.toml
@@ -5,6 +5,7 @@ postgresql = "18.6"
 uv = "0.12.6"
 psycopg = "3.3.4"
 psycopg_distribution = "binary-wheel"
+gunicorn = "26.2.0"
 
 [integrity]
 uv_release_commit = "7938ca5d53dbb9c614a4a030df406e41ff101ab9"
@@ -12,7 +13,7 @@ uv_macos_arm64_archive_sha256 = "14b459d51ea2e71eeba28c45a268c922bdf8607fc6455e3
 uv_attestation = "verified-with-github-cli-and-public-sigstore-rekor"
 
 [delivery]
-production_wsgi = "UNRESOLVED"
+production_wsgi = "gunicorn-26.2.0-sync"
 production_postgresql_package_or_image = "UNRESOLVED"
 production_linux_target = "UNRESOLVED"
-status = "development-only"
+status = "runtime-pinned-awaiting-acceptance"
diff --git a/scripts/run-production b/scripts/run-production
new file mode 100755
index 0000000..194b962
--- /dev/null
+++ b/scripts/run-production
@@ -0,0 +1,32 @@
+#!/bin/sh
+set -eu
+
+if [ "${GUNICORN_CMD_ARGS+x}" = x ]; then
+  printf '%s\n' 'startup_error=GUNICORN_CMD_ARGS_FORBIDDEN' >&2
+  exit 64
+fi
+
+script_dir=$(CDPATH= cd -- "$(dirname -- "$0")" && pwd)
+project_dir=$(dirname -- "$script_dir")
+python_bin="$project_dir/.venv/bin/python"
+
+if [ ! -x "$python_bin" ]; then
+  printf '%s\n' 'startup_error=FROZEN_ENV_MISSING' >&2
+  exit 64
+fi
+if ! "$python_bin" -I -c 'import sys; raise SystemExit(0 if sys.version_info[:3] == (3, 14, 7) else 1)'; then
+  printf '%s\n' 'startup_error=PYTHON_VERSION_MISMATCH' >&2
+  exit 64
+fi
+if ! "$python_bin" -I -c 'from importlib.metadata import version; raise SystemExit(0 if version("gunicorn") == "26.2.0" else 1)'; then
+  printf '%s\n' 'startup_error=GUNICORN_VERSION_MISMATCH' >&2
+  exit 64
+fi
+
+cd "$project_dir"
+DJANGO_SETTINGS_MODULE=travel_readiness.settings
+export DJANGO_SETTINGS_MODULE
+exec "$python_bin" -I -m gunicorn \
+  --chdir "$project_dir" \
+  --config "$project_dir/runtime/gunicorn.conf.py" \
+  travel_readiness.wsgi:application
diff --git a/travel_readiness/settings.py b/travel_readiness/settings.py
index 0699270..63325c8 100644
--- a/travel_readiness/settings.py
+++ b/travel_readiness/settings.py
@@ -14,6 +14,17 @@ def required_env(name: str) -> str:
     return value
 
 
+def strict_env_flag(name: str, *, default: bool = False) -> bool:
+    value = os.environ.get(name)
+    if value is None:
+        return default
+    if value not in {"0", "1"}:
+        raise ImproperlyConfigured(
+            f"Environment variable must be 0 or 1: {name}"
+        )
+    return value == "1"
+
+
 def redact_django_record(record) -> bool:
     request = getattr(record, "request", None)
     match = getattr(request, "resolver_match", None)
@@ -25,8 +36,13 @@ def redact_django_record(record) -> bool:
     return True
 
 
-SECRET_KEY = required_env("TRAVEL_READINESS_SECRET_KEY")
-DEBUG = os.environ.get("TRAVEL_READINESS_DEBUG") == "1"
+BUILD_MODE = strict_env_flag("TRAVEL_READINESS_BUILD")
+SECRET_KEY = (
+    "credential-free-static-build-only-not-for-runtime"
+    if BUILD_MODE
+    else required_env("TRAVEL_READINESS_SECRET_KEY")
+)
+DEBUG = strict_env_flag("TRAVEL_READINESS_DEBUG")
 ALLOWED_HOSTS = [
     host.strip()
     for host in os.environ.get("TRAVEL_READINESS_ALLOWED_HOSTS", "").split(",")
@@ -89,7 +105,11 @@ DATABASES = {
         "ENGINE": "django.db.backends.postgresql",
         "NAME": os.environ.get("TRAVEL_READINESS_DB_NAME", "travel_readiness"),
         "USER": os.environ.get("TRAVEL_READINESS_DB_USER", "travel_readiness"),
-        "PASSWORD": required_env("TRAVEL_READINESS_DB_PASSWORD"),
+        "PASSWORD": (
+            "credential-free-static-build-only"
+            if BUILD_MODE
+            else required_env("TRAVEL_READINESS_DB_PASSWORD")
+        ),
         "HOST": os.environ.get("TRAVEL_READINESS_DB_HOST", "127.0.0.1"),
         "PORT": os.environ.get("TRAVEL_READINESS_DB_PORT", "5432"),
         "CONN_MAX_AGE": 0,
@@ -102,6 +122,7 @@ TIME_ZONE = "UTC"
 USE_I18N = True
 USE_TZ = True
 STATIC_URL = "/static/"
+STATIC_ROOT = BASE_DIR / "staticfiles"
 DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"
 
 SECURE_SSL_REDIRECT = os.environ.get("TRAVEL_READINESS_HTTPS", "1") != "0"
diff --git a/travel_readiness/wsgi.py b/travel_readiness/wsgi.py
index 215dc01..5a665ca 100644
--- a/travel_readiness/wsgi.py
+++ b/travel_readiness/wsgi.py
@@ -1,7 +1,12 @@
 import os
 
+from django.conf import settings
+from django.core.exceptions import ImproperlyConfigured
 from django.core.wsgi import get_wsgi_application
 
 os.environ.setdefault("DJANGO_SETTINGS_MODULE", "travel_readiness.settings")
 
+if settings.BUILD_MODE:
+    raise ImproperlyConfigured("Static build mode cannot start the WSGI runtime")
+
 application = get_wsgi_application()
diff --git a/uv.lock b/uv.lock
index 94ea44a..3654b2c 100644
--- a/uv.lock
+++ b/uv.lock
@@ -17,12 +17,14 @@ version = "0.1.0"
 source = { virtual = "." }
 dependencies = [
     { name = "django" },
+    { name = "gunicorn" },
     { name = "psycopg", extra = ["binary"] },
 ]
 
 [package.metadata]
 requires-dist = [
     { name = "django", specifier = "==5.2.17" },
+    { name = "gunicorn", specifier = "==26.2.0" },
     { name = "psycopg", extras = ["binary"], specifier = "==3.3.4" },
 ]
 
@@ -40,6 +42,15 @@ wheels = [
     { url = "https://files.pythonhosted.org/packages/df/f8/ce120525ca78f12b07daf65786679c5d0b54a75285a8958d3ae55e39da35/django-5.2.17-py3-none-any.whl", hash = "sha256:f04fb3b36ee119e1af4fa1d397d5fd6cf12700f49321e84d4f4c642c5b1973db", size = 8315563, upload-time = "2026-08-04T15:03:59.1Z" },
 ]
 
+[[package]]
+name = "gunicorn"
+version = "26.2.0"
+source = { registry = "https://pypi.org/simple" }
+sdist = { url = "https://files.pythonhosted.org/packages/d9/8a/e4ef6ee11701b6cd64702848415ffb69eeff85cb388a3c6c7fe86f22f3f8/gunicorn-26.2.0.tar.gz", hash = "sha256:62b864895d9ebff0b2f9867ba04fe811c93121596540830c9c916d0769668447", size = 787921, upload-time = "2026-08-24T15:05:59.3Z" }
+wheels = [
+    { url = "https://files.pythonhosted.org/packages/fe/85/7522a52e5e2f42faf1a129113ab63e548c42e103e9af395b7bfe65e403e2/gunicorn-26.2.0-py3-none-any.whl", hash = "sha256:bd249d0b3f7972f7432f0a6b6ff3b3ee2d129f70cd1ff6c09a9dd9e29a2b88e3", size = 228389, upload-time = "2026-08-24T15:05:57.67Z" },
+]
+
 [[package]]
 name = "psycopg"
 version = "3.3.4"


