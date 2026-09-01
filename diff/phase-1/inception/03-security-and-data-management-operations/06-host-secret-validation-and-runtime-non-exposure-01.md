# 호스트 Secret 검증과 런타임 비노출 경계

## `chore(repo): 컨테이너 스택 저장소 경계 설정`

diff --git a/.gitignore b/.gitignore
new file mode 100644
index 0000000..4bba7ea
--- /dev/null
+++ b/.gitignore
@@ -0,0 +1,5 @@
+.env
+secrets/*.txt
+*.log
+*.pid
+.DS_Store


## `feat(compose): 준비 상태에 따라 영속 서비스 연결`

diff --git a/.env.example b/.env.example
index 2deceb8..072c9d8 100644
--- a/.env.example
+++ b/.env.example
@@ -2,9 +2,13 @@ DOMAIN_NAME=localhost
 
 MYSQL_DATABASE=wordpress
 MYSQL_USER=wpuser
+MYSQL_PASSWORD=change-me-db-password
+MYSQL_ROOT_PASSWORD=change-me-root-password
 
 WORDPRESS_TITLE=Inception
 WORDPRESS_ADMIN_USER=admin
+WORDPRESS_ADMIN_PASSWORD=change-me-admin-password
 WORDPRESS_ADMIN_EMAIL=admin@example.com
 WORDPRESS_USER=author
+WORDPRESS_USER_PASSWORD=change-me-author-password
 WORDPRESS_USER_EMAIL=author@example.com
diff --git a/srcs/docker-compose.yml b/srcs/docker-compose.yml
index d5fca56..699d7ea 100644
--- a/srcs/docker-compose.yml
+++ b/srcs/docker-compose.yml
@@ -9,8 +9,19 @@ services:
       DOMAIN_NAME: ${DOMAIN_NAME:?set DOMAIN_NAME}
     ports:
       - "443:443"
+    volumes:
+      - wordpress_data:/var/www/html:ro
+    depends_on:
+      wordpress:
+        condition: service_healthy
     networks:
       - inception
+    healthcheck:
+      test: ["CMD-SHELL", "curl -kfsS https://127.0.0.1/healthz >/dev/null"]
+      interval: 10s
+      timeout: 5s
+      retries: 6
+      start_period: 20s
 
   mariadb:
     build:
@@ -21,8 +32,18 @@ services:
     environment:
       MYSQL_DATABASE: ${MYSQL_DATABASE:?set MYSQL_DATABASE}
       MYSQL_USER: ${MYSQL_USER:?set MYSQL_USER}
+      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD:?set MYSQL_ROOT_PASSWORD}
+      MYSQL_PASSWORD: ${MYSQL_PASSWORD:?set MYSQL_PASSWORD}
+    volumes:
+      - mariadb_data:/var/lib/mysql
     networks:
       - inception
+    healthcheck:
+      test: ["CMD-SHELL", "mysqladmin ping -h127.0.0.1 -uroot -p\"$${MYSQL_ROOT_PASSWORD}\" --silent"]
+      interval: 10s
+      timeout: 5s
+      retries: 8
+      start_period: 30s
 
   wordpress:
     build:
@@ -32,15 +53,30 @@ services:
     restart: unless-stopped
     environment:
       DOMAIN_NAME: ${DOMAIN_NAME:?set DOMAIN_NAME}
+      WORDPRESS_DB_HOST: mariadb
       MYSQL_DATABASE: ${MYSQL_DATABASE:?set MYSQL_DATABASE}
       MYSQL_USER: ${MYSQL_USER:?set MYSQL_USER}
+      WORDPRESS_DB_PASSWORD: ${MYSQL_PASSWORD:?set MYSQL_PASSWORD}
       WORDPRESS_TITLE: ${WORDPRESS_TITLE:?set WORDPRESS_TITLE}
       WORDPRESS_ADMIN_USER: ${WORDPRESS_ADMIN_USER:?set WORDPRESS_ADMIN_USER}
+      WORDPRESS_ADMIN_PASSWORD: ${WORDPRESS_ADMIN_PASSWORD:?set WORDPRESS_ADMIN_PASSWORD}
       WORDPRESS_ADMIN_EMAIL: ${WORDPRESS_ADMIN_EMAIL:?set WORDPRESS_ADMIN_EMAIL}
       WORDPRESS_USER: ${WORDPRESS_USER:?set WORDPRESS_USER}
+      WORDPRESS_USER_PASSWORD: ${WORDPRESS_USER_PASSWORD:?set WORDPRESS_USER_PASSWORD}
       WORDPRESS_USER_EMAIL: ${WORDPRESS_USER_EMAIL:?set WORDPRESS_USER_EMAIL}
+    volumes:
+      - wordpress_data:/var/www/html
+    depends_on:
+      mariadb:
+        condition: service_healthy
     networks:
       - inception
+    healthcheck:
+      test: ["CMD-SHELL", "REQUEST_METHOD=GET SCRIPT_NAME=/ping SCRIPT_FILENAME=/ping cgi-fcgi -bind -connect 127.0.0.1:9000 | grep -q pong"]
+      interval: 10s
+      timeout: 5s
+      retries: 8
+      start_period: 40s
 
 networks:
   inception:


## `feat(secrets): 비밀번호를 비밀 파일에서 로드`

diff --git a/.env.example b/.env.example
index 072c9d8..7ee8ef2 100644
--- a/.env.example
+++ b/.env.example
@@ -2,13 +2,14 @@ DOMAIN_NAME=localhost
 
 MYSQL_DATABASE=wordpress
 MYSQL_USER=wpuser
-MYSQL_PASSWORD=change-me-db-password
-MYSQL_ROOT_PASSWORD=change-me-root-password
 
 WORDPRESS_TITLE=Inception
 WORDPRESS_ADMIN_USER=admin
-WORDPRESS_ADMIN_PASSWORD=change-me-admin-password
 WORDPRESS_ADMIN_EMAIL=admin@example.com
 WORDPRESS_USER=author
-WORDPRESS_USER_PASSWORD=change-me-author-password
 WORDPRESS_USER_EMAIL=author@example.com
+
+DB_ROOT_PASSWORD_FILE=../secrets/db_root_password.txt
+DB_PASSWORD_FILE=../secrets/db_password.txt
+WP_ADMIN_PASSWORD_FILE=../secrets/wp_admin_password.txt
+WP_USER_PASSWORD_FILE=../secrets/wp_user_password.txt
diff --git a/srcs/docker-compose.yml b/srcs/docker-compose.yml
index 699d7ea..31180f6 100644
--- a/srcs/docker-compose.yml
+++ b/srcs/docker-compose.yml
@@ -32,14 +32,17 @@ services:
     environment:
       MYSQL_DATABASE: ${MYSQL_DATABASE:?set MYSQL_DATABASE}
       MYSQL_USER: ${MYSQL_USER:?set MYSQL_USER}
-      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD:?set MYSQL_ROOT_PASSWORD}
-      MYSQL_PASSWORD: ${MYSQL_PASSWORD:?set MYSQL_PASSWORD}
+      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/db_root_password
+      MYSQL_PASSWORD_FILE: /run/secrets/db_password
+    secrets:
+      - db_root_password
+      - db_password
     volumes:
       - mariadb_data:/var/lib/mysql
     networks:
       - inception
     healthcheck:
-      test: ["CMD-SHELL", "mysqladmin ping -h127.0.0.1 -uroot -p\"$${MYSQL_ROOT_PASSWORD}\" --silent"]
+      test: ["CMD-SHELL", "mysqladmin --socket=/run/mysqld/mysqld.sock -uroot -p\"$$(cat /run/secrets/db_root_password)\" ping --silent"]
       interval: 10s
       timeout: 5s
       retries: 8
@@ -56,14 +59,18 @@ services:
       WORDPRESS_DB_HOST: mariadb
       MYSQL_DATABASE: ${MYSQL_DATABASE:?set MYSQL_DATABASE}
       MYSQL_USER: ${MYSQL_USER:?set MYSQL_USER}
-      WORDPRESS_DB_PASSWORD: ${MYSQL_PASSWORD:?set MYSQL_PASSWORD}
+      WORDPRESS_DB_PASSWORD_FILE: /run/secrets/db_password
       WORDPRESS_TITLE: ${WORDPRESS_TITLE:?set WORDPRESS_TITLE}
       WORDPRESS_ADMIN_USER: ${WORDPRESS_ADMIN_USER:?set WORDPRESS_ADMIN_USER}
-      WORDPRESS_ADMIN_PASSWORD: ${WORDPRESS_ADMIN_PASSWORD:?set WORDPRESS_ADMIN_PASSWORD}
+      WORDPRESS_ADMIN_PASSWORD_FILE: /run/secrets/wp_admin_password
       WORDPRESS_ADMIN_EMAIL: ${WORDPRESS_ADMIN_EMAIL:?set WORDPRESS_ADMIN_EMAIL}
       WORDPRESS_USER: ${WORDPRESS_USER:?set WORDPRESS_USER}
-      WORDPRESS_USER_PASSWORD: ${WORDPRESS_USER_PASSWORD:?set WORDPRESS_USER_PASSWORD}
+      WORDPRESS_USER_PASSWORD_FILE: /run/secrets/wp_user_password
       WORDPRESS_USER_EMAIL: ${WORDPRESS_USER_EMAIL:?set WORDPRESS_USER_EMAIL}
+    secrets:
+      - db_password
+      - wp_admin_password
+      - wp_user_password
     volumes:
       - wordpress_data:/var/www/html
     depends_on:
@@ -85,3 +92,13 @@ networks:
 volumes:
   mariadb_data:
   wordpress_data:
+
+secrets:
+  db_root_password:
+    file: ${DB_ROOT_PASSWORD_FILE:-../secrets/db_root_password.txt}
+  db_password:
+    file: ${DB_PASSWORD_FILE:-../secrets/db_password.txt}
+  wp_admin_password:
+    file: ${WP_ADMIN_PASSWORD_FILE:-../secrets/wp_admin_password.txt}
+  wp_user_password:
+    file: ${WP_USER_PASSWORD_FILE:-../secrets/wp_user_password.txt}


## `refactor(secrets): 비밀 파일 로딩 경계 공통화`

diff --git a/tools/stack_runtime.py b/tools/stack_runtime.py
index c7abbbc..b4f3b0f 100755
--- a/tools/stack_runtime.py
+++ b/tools/stack_runtime.py
@@ -1,11 +1,13 @@
 #!/usr/bin/env python3
-"""Compose 관리 도구가 공유하는 프로젝트 실행 경계를 제공합니다."""
+"""Compose 관리 도구가 공유하는 프로젝트·비밀 파일 경계를 제공합니다."""
 
 from __future__ import annotations
 
 import json
+import os
 from pathlib import Path
 import re
+import stat
 import subprocess
 from typing import BinaryIO
 
@@ -13,6 +15,15 @@ from typing import BinaryIO
 ROOT = Path(__file__).resolve().parents[1]
 DEFAULT_COMPOSE_FILE = ROOT / "srcs" / "docker-compose.yml"
 PROJECT_PATTERN = re.compile(r"^[a-z0-9][a-z0-9_-]{2,62}$")
+PASSWORD_PATTERN = re.compile(r"^[A-Za-z0-9_.~!@#%^+=,-]{24,128}$")
+SECRET_FILENAMES = {
+    "db_root_password": "db_root_password.txt",
+    "db_password": "db_password.txt",
+    "wp_admin_password": "wp_admin_password.txt",
+    "wp_user_password": "wp_user_password.txt",
+}
+NOFOLLOW = getattr(os, "O_NOFOLLOW", 0)
+NONBLOCK = getattr(os, "O_NONBLOCK", 0)
 
 
 class StackRuntimeError(RuntimeError):
@@ -104,3 +115,109 @@ class ComposeProject:
             for line in result.stdout.decode(errors="replace").splitlines()
             if line
         }
+
+
+def _private_directory(path: Path, description: str) -> None:
+    try:
+        info = os.lstat(path)
+    except OSError as error:
+        raise StackRuntimeError(f"{description}을 확인할 수 없습니다: {path}") from error
+    if (
+        not stat.S_ISDIR(info.st_mode)
+        or info.st_uid != os.getuid()
+        or stat.S_IMODE(info.st_mode) & 0o077
+    ):
+        raise StackRuntimeError(
+            f"{description}은 현재 사용자만 접근하는 일반 디렉터리여야 합니다: {path}"
+        )
+
+
+def read_private_secret(path: Path) -> str:
+    path = path.expanduser()
+    _private_directory(path.parent, "비밀 파일 상위 디렉터리")
+    try:
+        descriptor = os.open(path, os.O_RDONLY | NOFOLLOW | NONBLOCK)
+    except OSError as error:
+        raise StackRuntimeError(f"비밀 파일을 안전하게 열 수 없습니다: {path}") from error
+    try:
+        info = os.fstat(descriptor)
+        if (
+            not stat.S_ISREG(info.st_mode)
+            or info.st_nlink != 1
+            or info.st_uid != os.getuid()
+            or stat.S_IMODE(info.st_mode) != 0o600
+        ):
+            raise StackRuntimeError(
+                f"비밀 파일은 현재 사용자가 소유한 0600 단일 링크 일반 파일이어야 합니다: {path}"
+            )
+        with os.fdopen(descriptor, "r", encoding="utf-8") as stream:
+            descriptor = -1
+            value = stream.read(1025)
+            if len(value) > 1024 or stream.read(1):
+                raise StackRuntimeError(f"비밀 파일이 허용 크기를 넘었습니다: {path}")
+    finally:
+        if descriptor >= 0:
+            os.close(descriptor)
+    value = value.removesuffix("\n")
+    if not PASSWORD_PATTERN.fullmatch(value):
+        raise StackRuntimeError(
+            f"비밀값은 허용 문자로 작성한 24~128자 한 줄이어야 합니다: {path.name}"
+        )
+    return value
+
+
+def secret_source_paths(
+    config: dict[str, object],
+    *,
+    compose_directory: Path | None = None,
+) -> dict[str, Path]:
+    configured = config.get("x-secret-files")
+    if not isinstance(configured, dict):
+        raise StackRuntimeError("Compose x-secret-files 설정을 찾을 수 없습니다")
+    paths: dict[str, Path] = {}
+    for name in SECRET_FILENAMES:
+        entry = configured.get(name)
+        if not isinstance(entry, str):
+            raise StackRuntimeError(f"Compose secret 원본을 찾을 수 없습니다: {name}")
+        path = Path(entry)
+        if not path.is_absolute():
+            if compose_directory is None:
+                raise StackRuntimeError(
+                    f"상대 secret 경로의 기준 디렉터리가 없습니다: {name}"
+                )
+            path = compose_directory / path
+        paths[name] = Path(os.path.abspath(path))
+    canonical = [path.resolve(strict=True) for path in paths.values()]
+    if len(set(canonical)) != len(canonical):
+        raise StackRuntimeError("Compose secret 원본 경로는 서로 달라야 합니다")
+    return paths
+
+
+def load_secret_values(project: ComposeProject) -> dict[str, str]:
+    paths = secret_source_paths(
+        project.config(),
+        compose_directory=project.compose_file.parent,
+    )
+    return {name: read_private_secret(path) for name, path in paths.items()}
+
+
+def service_environment(
+    config: dict[str, object], service: str
+) -> dict[str, str]:
+    services = config.get("services")
+    if not isinstance(services, dict):
+        raise StackRuntimeError("Compose 서비스 설정을 찾을 수 없습니다")
+    configured = services.get(service)
+    if not isinstance(configured, dict):
+        raise StackRuntimeError(f"Compose 서비스를 찾을 수 없습니다: {service}")
+    environment = configured.get("environment")
+    if not isinstance(environment, dict):
+        raise StackRuntimeError(f"서비스 환경 변수를 찾을 수 없습니다: {service}")
+    return {str(key): str(value) for key, value in environment.items()}
+
+
+def secret_payload(*values: str) -> bytes:
+    for value in values:
+        if not PASSWORD_PATTERN.fullmatch(value):
+            raise StackRuntimeError("표준 입력 비밀값의 형식이 올바르지 않습니다")
+    return ("\n".join(values) + "\n").encode()


