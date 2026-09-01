## `build: scaffold django project`

diff --git a/.env.example b/.env.example
index 5ec6723..ecb08bd 100644
--- a/.env.example
+++ b/.env.example
@@ -2,6 +2,6 @@
 DJANGO_DEBUG=0
 DJANGO_SECRET_KEY=replace-in-secret-store
 DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1
-DATABASE_URL=postgresql://grocery:local-grocery-only@127.0.0.1:55432/grocery
+DATABASE_URL=postgresql://grocery:local-grocery-only@127.0.0.1:55434/grocery
 KAMIS_API_KEY=
 DEPLOY_VERSION=dev
diff --git a/compose.yaml b/compose.yaml
index f04d8ac..a1739e5 100644
--- a/compose.yaml
+++ b/compose.yaml
@@ -13,7 +13,7 @@ services:
       timeout: 3s
       retries: 15
     ports:
-      - "127.0.0.1:55432:5432"
+      - "127.0.0.1:55434:5432"
     volumes:
       - grocery-postgres:/var/lib/postgresql
 
diff --git a/config/__init__.py b/config/__init__.py
new file mode 100644
index 0000000..e69de29
diff --git a/config/settings.py b/config/settings.py
new file mode 100644
index 0000000..c25a074
--- /dev/null
+++ b/config/settings.py
@@ -0,0 +1,114 @@
+import os
+from pathlib import Path
+from urllib.parse import parse_qsl, unquote, urlparse
+
+BASE_DIR = Path(__file__).resolve().parent.parent
+
+
+def env_bool(name: str, default: bool) -> bool:
+    value = os.environ.get(name)
+    if value is None:
+        return default
+    return value.strip().lower() in {"1", "true", "yes", "on"}
+
+
+def env_list(name: str, default: str) -> list[str]:
+    return [item.strip() for item in os.environ.get(name, default).split(",") if item.strip()]
+
+
+def database_config() -> dict[str, object]:
+    value = os.environ.get(
+        "DATABASE_URL",
+        "postgresql://grocery:local-grocery-only@127.0.0.1:55434/grocery",
+    )
+    parsed = urlparse(value)
+    if parsed.scheme not in {"postgres", "postgresql"}:
+        raise ValueError("DATABASE_URL must use PostgreSQL")
+    if not all((parsed.hostname, parsed.path.removeprefix("/"), parsed.username)):
+        raise ValueError("DATABASE_URL is incomplete")
+    allowed_options = {"sslmode", "target_session_attrs"}
+    options = {key: val for key, val in parse_qsl(parsed.query) if key in allowed_options}
+    return {
+        "ENGINE": "django.db.backends.postgresql",
+        "NAME": unquote(parsed.path.removeprefix("/")),
+        "USER": unquote(parsed.username or ""),
+        "PASSWORD": unquote(parsed.password or ""),
+        "HOST": parsed.hostname,
+        "PORT": parsed.port or 5432,
+        "CONN_MAX_AGE": int(os.environ.get("DATABASE_CONN_MAX_AGE", "0")),
+        "CONN_HEALTH_CHECKS": True,
+        "OPTIONS": options,
+    }
+
+
+DEBUG = env_bool("DJANGO_DEBUG", True)
+SECRET_KEY = os.environ.get("DJANGO_SECRET_KEY", "local-development-only-not-for-production")
+ALLOWED_HOSTS = env_list("DJANGO_ALLOWED_HOSTS", "localhost,127.0.0.1,testserver")
+CSRF_TRUSTED_ORIGINS = env_list("DJANGO_CSRF_TRUSTED_ORIGINS", "")
+
+INSTALLED_APPS = [
+    "django.contrib.admin",
+    "django.contrib.auth",
+    "django.contrib.contenttypes",
+    "django.contrib.sessions",
+    "django.contrib.messages",
+    "django.contrib.staticfiles",
+    "grocery.apps.GroceryConfig",
+]
+
+MIDDLEWARE = [
+    "django.middleware.security.SecurityMiddleware",
+    "django.contrib.sessions.middleware.SessionMiddleware",
+    "django.middleware.common.CommonMiddleware",
+    "django.middleware.csrf.CsrfViewMiddleware",
+    "django.contrib.auth.middleware.AuthenticationMiddleware",
+    "django.contrib.messages.middleware.MessageMiddleware",
+    "django.middleware.clickjacking.XFrameOptionsMiddleware",
+]
+
+ROOT_URLCONF = "config.urls"
+TEMPLATES = [
+    {
+        "BACKEND": "django.template.backends.django.DjangoTemplates",
+        "DIRS": [BASE_DIR / "templates"],
+        "APP_DIRS": True,
+        "OPTIONS": {
+            "context_processors": [
+                "django.template.context_processors.request",
+                "django.contrib.auth.context_processors.auth",
+                "django.contrib.messages.context_processors.messages",
+            ],
+        },
+    }
+]
+WSGI_APPLICATION = "config.wsgi.application"
+
+DATABASES = {"default": database_config()}
+
+AUTH_PASSWORD_VALIDATORS = [
+    {"NAME": "django.contrib.auth.password_validation.UserAttributeSimilarityValidator"},
+    {"NAME": "django.contrib.auth.password_validation.MinimumLengthValidator"},
+    {"NAME": "django.contrib.auth.password_validation.CommonPasswordValidator"},
+    {"NAME": "django.contrib.auth.password_validation.NumericPasswordValidator"},
+]
+
+LANGUAGE_CODE = "ko-kr"
+TIME_ZONE = "Asia/Seoul"
+USE_I18N = True
+USE_TZ = True
+
+STATIC_URL = "/static/"
+STATIC_ROOT = BASE_DIR / "staticfiles"
+DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"
+
+SECURE_SSL_REDIRECT = env_bool("DJANGO_SECURE_SSL_REDIRECT", not DEBUG)
+SESSION_COOKIE_SECURE = not DEBUG
+CSRF_COOKIE_SECURE = not DEBUG
+SECURE_HSTS_SECONDS = int(
+    os.environ.get("DJANGO_SECURE_HSTS_SECONDS", "0" if DEBUG else "31536000")
+)
+SECURE_HSTS_INCLUDE_SUBDOMAINS = not DEBUG
+SECURE_HSTS_PRELOAD = not DEBUG
+SECURE_CONTENT_TYPE_NOSNIFF = True
+SECURE_REFERRER_POLICY = "same-origin"
+X_FRAME_OPTIONS = "DENY"
diff --git a/config/urls.py b/config/urls.py
new file mode 100644
index 0000000..4096fa2
--- /dev/null
+++ b/config/urls.py
@@ -0,0 +1,4 @@
+from django.contrib import admin
+from django.urls import path
+
+urlpatterns = [path("admin/", admin.site.urls)]
diff --git a/config/wsgi.py b/config/wsgi.py
new file mode 100644
index 0000000..8509335
--- /dev/null
+++ b/config/wsgi.py
@@ -0,0 +1,7 @@
+import os
+
+from django.core.wsgi import get_wsgi_application
+
+os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings")
+
+application = get_wsgi_application()
diff --git a/grocery/__init__.py b/grocery/__init__.py
new file mode 100644
index 0000000..e69de29
diff --git a/grocery/apps.py b/grocery/apps.py
new file mode 100644
index 0000000..a47c1b2
--- /dev/null
+++ b/grocery/apps.py
@@ -0,0 +1,6 @@
+from django.apps import AppConfig
+
+
+class GroceryConfig(AppConfig):
+    default_auto_field = "django.db.models.BigAutoField"
+    name = "grocery"
diff --git a/grocery/migrations/__init__.py b/grocery/migrations/__init__.py
new file mode 100644
index 0000000..e69de29
diff --git a/grocery/models.py b/grocery/models.py
new file mode 100644
index 0000000..e8f8527
--- /dev/null
+++ b/grocery/models.py
@@ -0,0 +1 @@
+# Models are introduced in additive, reversible migrations.
diff --git a/grocery/tests/__init__.py b/grocery/tests/__init__.py
new file mode 100644
index 0000000..e69de29
diff --git a/manage.py b/manage.py
new file mode 100644
index 0000000..7f082c5
--- /dev/null
+++ b/manage.py
@@ -0,0 +1,14 @@
+#!/usr/bin/env python
+import os
+import sys
+
+
+def main() -> None:
+    os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings")
+    from django.core.management import execute_from_command_line
+
+    execute_from_command_line(sys.argv)
+
+
+if __name__ == "__main__":
+    main()


## `build: add repeatable verification targets`

diff --git a/Makefile b/Makefile
new file mode 100644
index 0000000..53cb194
--- /dev/null
+++ b/Makefile
@@ -0,0 +1,34 @@
+UV_RUN := env UV_CACHE_DIR=.cache/uv UV_TOOL_DIR=.cache/uv-tools UV_PYTHON_INSTALL_DIR=.cache/python uvx --from uv==0.12.6 uv
+PYTHON := .venv/bin/python
+
+.PHONY: check db-up format-check lint migrate migration-check serve sync test type
+
+sync:
+	$(UV_RUN) sync --frozen
+
+db-up:
+	docker compose up -d db
+
+migrate:
+	$(PYTHON) manage.py migrate --noinput
+
+migration-check:
+	$(PYTHON) manage.py makemigrations --check --dry-run
+
+format-check:
+	.venv/bin/ruff format --check .
+
+lint:
+	.venv/bin/ruff check .
+
+type:
+	.venv/bin/mypy config grocery manage.py
+
+test:
+	.venv/bin/pytest
+
+check: format-check lint type migration-check test
+	$(PYTHON) manage.py check
+
+serve:
+	.venv/bin/gunicorn config.wsgi:application --bind 127.0.0.1:8000 --workers 2 --threads 4
