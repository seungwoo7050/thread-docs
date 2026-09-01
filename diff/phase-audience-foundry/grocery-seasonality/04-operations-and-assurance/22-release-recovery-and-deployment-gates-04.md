## `fix(release): isolate local assurance database`

diff --git a/Makefile b/Makefile
index 8551a13..6195ba3 100644
--- a/Makefile
+++ b/Makefile
@@ -6,7 +6,7 @@ PYTHON := .venv/bin/python
 # DATABASE_URL, and the exact 40-character lowercase release DEPLOY_VERSION.
 # Its secret-check reads the ignored owner-only .env.local in-process; do not export
 # KAMIS_API_KEY into Make, a command argument, or a child-process environment.
-.PHONY: check db-up dependency-audit format-check license-inventory lint migrate migration-check production-check production-env-check secret-check serve sync test type
+.PHONY: check db-up dependency-audit format-check license-inventory lint local-release-db-check migrate migration-check production-check production-env-check secret-check serve sync test type
 
 sync:
 	$(UV_RUN) sync --frozen
@@ -51,8 +51,11 @@ production-env-check:
 	@test "$${#DEPLOY_VERSION}" -eq 40 || { echo "production_check=failed code=release_sha_required"; exit 2; }
 	@case "$${DEPLOY_VERSION}" in *[!0-9a-f]*) echo "production_check=failed code=release_sha_required"; exit 2;; esac
 
-production-check: production-env-check check secret-check dependency-audit license-inventory
-	$(PYTHON) manage.py check --deploy
+local-release-db-check:
+	$(PYTHON) -m scripts.local_release_database_check
+
+production-check: production-env-check local-release-db-check check secret-check dependency-audit license-inventory
+	$(PYTHON) manage.py check --deploy --fail-level WARNING
 
 serve:
 	.venv/bin/gunicorn config.wsgi:application --bind 127.0.0.1:8000 --workers 2 --threads 4
diff --git a/scripts/local_release_database_check.py b/scripts/local_release_database_check.py
new file mode 100644
index 0000000..2570332
--- /dev/null
+++ b/scripts/local_release_database_check.py
@@ -0,0 +1,42 @@
+"""Fail closed unless a release assurance run targets the fixed local database."""
+
+from __future__ import annotations
+
+import os
+from urllib.parse import urlparse
+
+
+def is_fixed_local_release_database(value: object) -> bool:
+    """Recognize only the repository's loopback Compose PostgreSQL contract."""
+
+    if type(value) is not str:
+        return False
+    try:
+        parsed = urlparse(value)
+        port = parsed.port
+    except ValueError:
+        return False
+    return bool(
+        parsed.scheme in {"postgres", "postgresql"}
+        and parsed.hostname == "127.0.0.1"
+        and port == 55_434
+        and parsed.path == "/grocery"
+        and parsed.username == "grocery"
+        # This is the public Compose-only development credential in compose.yaml.
+        and parsed.password == "local-grocery-only"  # noqa: S105
+        and not parsed.params
+        and not parsed.query
+        and not parsed.fragment
+    )
+
+
+def main() -> int:
+    if not is_fixed_local_release_database(os.environ.get("DATABASE_URL")):
+        print("local_release_database=failed code=fixed_loopback_database_required")
+        return 2
+    print("local_release_database=ok")
+    return 0
+
+
+if __name__ == "__main__":
+    raise SystemExit(main())
diff --git a/scripts/tests/test_local_release_database_check.py b/scripts/tests/test_local_release_database_check.py
new file mode 100644
index 0000000..ca97016
--- /dev/null
+++ b/scripts/tests/test_local_release_database_check.py
@@ -0,0 +1,45 @@
+from __future__ import annotations
+
+import pytest
+
+from scripts.local_release_database_check import is_fixed_local_release_database, main
+
+_FIXED_URL = "postgresql://grocery:local-grocery-only@127.0.0.1:55434/grocery"
+
+
+def test_fixed_loopback_compose_database_is_accepted(
+    monkeypatch: pytest.MonkeyPatch,
+    capsys: pytest.CaptureFixture[str],
+) -> None:
+    monkeypatch.setenv("DATABASE_URL", _FIXED_URL)
+
+    assert is_fixed_local_release_database(_FIXED_URL)
+    assert main() == 0
+    assert capsys.readouterr().out == "local_release_database=ok\n"
+
+
+@pytest.mark.parametrize(
+    "value",
+    (
+        None,
+        "",
+        "postgresql://grocery:private@database.example:5432/grocery",
+        "postgresql://grocery:local-grocery-only@localhost:55434/grocery",
+        "postgresql://grocery:local-grocery-only@127.0.0.1:5432/grocery",
+        "postgresql://grocery:local-grocery-only@127.0.0.1:55434/other",
+        "postgresql://grocery:local-grocery-only@127.0.0.1:55434/grocery?sslmode=require",
+    ),
+)
+def test_every_nonfixed_database_shape_is_rejected_without_reflection(
+    value: str | None,
+    monkeypatch: pytest.MonkeyPatch,
+    capsys: pytest.CaptureFixture[str],
+) -> None:
+    marker = "must-not-be-reflected"
+    monkeypatch.setenv("DATABASE_URL", value or marker)
+
+    assert not is_fixed_local_release_database(value)
+    assert main() == 2
+    output = capsys.readouterr().out
+    assert output == ("local_release_database=failed code=fixed_loopback_database_required\n")
+    assert marker not in output


## `feat(web): serve immutable static assets`

diff --git a/THIRD_PARTY_NOTICES.md b/THIRD_PARTY_NOTICES.md
index e22155a..1739e23 100644
--- a/THIRD_PARTY_NOTICES.md
+++ b/THIRD_PARTY_NOTICES.md
@@ -9,6 +9,7 @@ digest를 함께 고정한다. 이 고지는 각 upstream license 원문을 대
 | Python | `3.14.7` | application runtime | `python.org` | PSF-2.0 |
 | Django | `5.2.17` | SSR, Forms, Auth, ORM, migration | `djangoproject.com` | BSD-3-Clause |
 | Gunicorn | `23.0.0` | 고정 production WSGI process | `gunicorn.org` | MIT |
+| WhiteNoise | `6.12.0` | 해시·압축된 WSGI static asset 제공 | `whitenoise.readthedocs.io` | MIT |
 | Psycopg | `3.3.4` | PostgreSQL driver | `psycopg.org` | LGPL-3.0-only |
 | psycopg-binary | `3.3.4` | local candidate의 self-contained libpq runtime | `psycopg.org` | LGPL-3.0-only 및 wheel 내 고지 |
 | asgiref | `3.12.1` | Django 전이 runtime | `github.com/django/asgiref` | BSD-3-Clause |
diff --git a/config/settings.py b/config/settings.py
index eae511c..e545cea 100644
--- a/config/settings.py
+++ b/config/settings.py
@@ -148,6 +148,7 @@ INSTALLED_APPS = [
 
 MIDDLEWARE = [
     "django.middleware.security.SecurityMiddleware",
+    "whitenoise.middleware.WhiteNoiseMiddleware",
     "grocery.security.SecurityHeadersMiddleware",
     "grocery.observability.RequestIdMiddleware",
     "grocery.security.AdminExposureMiddleware",
@@ -192,6 +193,20 @@ USE_TZ = True
 
 STATIC_URL = "/static/"
 STATIC_ROOT = BASE_DIR / "staticfiles"
+
+
+def staticfiles_storage_backend(*, debug: bool) -> str:
+    if debug:
+        return "django.contrib.staticfiles.storage.StaticFilesStorage"
+    return "whitenoise.storage.CompressedManifestStaticFilesStorage"
+
+
+STORAGES = {
+    "default": {"BACKEND": "django.core.files.storage.FileSystemStorage"},
+    "staticfiles": {
+        "BACKEND": staticfiles_storage_backend(debug=DEBUG),
+    },
+}
 DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"
 
 SECURE_SSL_REDIRECT = env_bool("DJANGO_SECURE_SSL_REDIRECT", not DEBUG)
diff --git a/grocery/tests/test_static_delivery.py b/grocery/tests/test_static_delivery.py
new file mode 100644
index 0000000..5ff5949
--- /dev/null
+++ b/grocery/tests/test_static_delivery.py
@@ -0,0 +1,26 @@
+from django.conf import settings
+from django.test import SimpleTestCase
+
+from config.settings import staticfiles_storage_backend
+
+
+class StaticDeliverySettingsTests(SimpleTestCase):
+    def test_whitenoise_is_inside_security_and_before_application_middleware(self) -> None:
+        middleware = list(settings.MIDDLEWARE)
+
+        security = middleware.index("django.middleware.security.SecurityMiddleware")
+        whitenoise = middleware.index("whitenoise.middleware.WhiteNoiseMiddleware")
+        application = middleware.index("grocery.security.SecurityHeadersMiddleware")
+
+        self.assertEqual(whitenoise, security + 1)
+        self.assertLess(whitenoise, application)
+
+    def test_static_build_uses_compressed_manifest_storage(self) -> None:
+        self.assertEqual(
+            staticfiles_storage_backend(debug=False),
+            "whitenoise.storage.CompressedManifestStaticFilesStorage",
+        )
+        self.assertEqual(
+            staticfiles_storage_backend(debug=True),
+            "django.contrib.staticfiles.storage.StaticFilesStorage",
+        )
diff --git a/pyproject.toml b/pyproject.toml
index 6e61654..4b2d08a 100644
--- a/pyproject.toml
+++ b/pyproject.toml
@@ -6,6 +6,7 @@ dependencies = [
     "django==5.2.17",
     "gunicorn==23.0.0",
     "psycopg[binary]==3.3.4",
+    "whitenoise==6.12.0",
 ]
 
 [tool.uv]
diff --git a/uv.lock b/uv.lock
index d357f94..4ca7782 100644
--- a/uv.lock
+++ b/uv.lock
@@ -82,6 +82,7 @@ dependencies = [
     { name = "django" },
     { name = "gunicorn" },
     { name = "psycopg", extra = ["binary"] },
+    { name = "whitenoise" },
 ]
 
 [package.dev-dependencies]
@@ -101,6 +102,7 @@ requires-dist = [
     { name = "django", specifier = "==5.2.17" },
     { name = "gunicorn", specifier = "==23.0.0" },
     { name = "psycopg", extras = ["binary"], specifier = "==3.3.4" },
+    { name = "whitenoise", specifier = "==6.12.0" },
 ]
 
 [package.metadata.requires-dev]
@@ -888,3 +890,12 @@ sdist = { url = "https://files.pythonhosted.org/packages/36/57/ed58088fafdf4c55a
 wheels = [
     { url = "https://files.pythonhosted.org/packages/c4/0e/57f6bb3024a597b2e8ec4aee710ffe62ddc95af2e2bb1ee7a7abdc22c68c/wcwidth-0.8.3-py3-none-any.whl", hash = "sha256:d5b73dba6158a595ec9370350e7f2637bcac8d6c5e4fde34f30fcffb6103a5e4", size = 331669, upload-time = "2026-08-28T18:10:04.909Z" },
 ]
+
+[[package]]
+name = "whitenoise"
+version = "6.12.0"
+source = { registry = "https://pypi.org/simple" }
+sdist = { url = "https://files.pythonhosted.org/packages/cb/2a/55b3f3a4ec326cd077c1c3defeee656b9298372a69229134d930151acd01/whitenoise-6.12.0.tar.gz", hash = "sha256:f723ebb76a112e98816ff80fcea0a6c9b8ecde835f8ddda25df7a30a3c2db6ad", size = 26841, upload-time = "2026-02-27T00:05:42.028Z" }
+wheels = [
+    { url = "https://files.pythonhosted.org/packages/db/eb/d5583a11486211f3ebd4b385545ae787f32363d453c19fffd81106c9c138/whitenoise-6.12.0-py3-none-any.whl", hash = "sha256:fc5e8c572e33ebf24795b47b6a7da8da3c00cff2349f5b04c02f28d0cc5a3cc2", size = 20302, upload-time = "2026-02-27T00:05:40.086Z" },
+]


