# 런타임 격리와 운영 보호 장치

## `feat(runtime): 프로젝트·이미지·포트·URL 격리`

diff --git a/.env.example b/.env.example
index 7ee8ef2..a525de8 100644
--- a/.env.example
+++ b/.env.example
@@ -1,9 +1,14 @@
 DOMAIN_NAME=localhost
+WORDPRESS_URL=https://localhost
+HTTPS_BIND_ADDRESS=127.0.0.1
+HTTPS_PORT=443
+STACK_IMAGE_PREFIX=container-stack
+STACK_IMAGE_TAG=local
 
 MYSQL_DATABASE=wordpress
 MYSQL_USER=wpuser
 
-WORDPRESS_TITLE=Inception
+WORDPRESS_TITLE=Container Stack
 WORDPRESS_ADMIN_USER=admin
 WORDPRESS_ADMIN_EMAIL=admin@example.com
 WORDPRESS_USER=author
diff --git a/srcs/docker-compose.yml b/srcs/docker-compose.yml
index 01d4174..096e4cd 100644
--- a/srcs/docker-compose.yml
+++ b/srcs/docker-compose.yml
@@ -8,13 +8,12 @@ services:
   nginx:
     build:
       context: ./requirements/nginx
-    image: inception-nginx:local
-    container_name: inception-nginx
+    image: ${STACK_IMAGE_PREFIX:-container-stack}-nginx:${STACK_IMAGE_TAG:-local}
     restart: unless-stopped
     environment:
       DOMAIN_NAME: ${DOMAIN_NAME:?set DOMAIN_NAME}
     ports:
-      - "443:443"
+      - "${HTTPS_BIND_ADDRESS:-127.0.0.1}:${HTTPS_PORT:-443}:443"
     volumes:
       - wordpress_data:/var/www/html:ro
     depends_on:
@@ -32,8 +31,7 @@ services:
   mariadb:
     build:
       context: ./requirements/mariadb
-    image: inception-mariadb:local
-    container_name: inception-mariadb
+    image: ${STACK_IMAGE_PREFIX:-container-stack}-mariadb:${STACK_IMAGE_TAG:-local}
     restart: unless-stopped
     environment:
       MYSQL_DATABASE: ${MYSQL_DATABASE:?set MYSQL_DATABASE}
@@ -52,11 +50,11 @@ services:
   wordpress:
     build:
       context: ./requirements/wordpress
-    image: inception-wordpress:local
-    container_name: inception-wordpress
+    image: ${STACK_IMAGE_PREFIX:-container-stack}-wordpress:${STACK_IMAGE_TAG:-local}
     restart: unless-stopped
     environment:
       DOMAIN_NAME: ${DOMAIN_NAME:?set DOMAIN_NAME}
+      WORDPRESS_URL: ${WORDPRESS_URL:?set WORDPRESS_URL}
       WORDPRESS_DB_HOST: mariadb
       MYSQL_DATABASE: ${MYSQL_DATABASE:?set MYSQL_DATABASE}
       MYSQL_USER: ${MYSQL_USER:?set MYSQL_USER}
diff --git a/srcs/requirements/wordpress/tools/docker-entrypoint.sh b/srcs/requirements/wordpress/tools/docker-entrypoint.sh
index 5c6d7b0..c415b1e 100755
--- a/srcs/requirements/wordpress/tools/docker-entrypoint.sh
+++ b/srcs/requirements/wordpress/tools/docker-entrypoint.sh
@@ -379,7 +379,7 @@ bootstrap() {
     : "${MYSQL_DATABASE:?MYSQL_DATABASE is required}"
     : "${MYSQL_USER:?MYSQL_USER is required}"
     : "${DOMAIN_NAME:?DOMAIN_NAME is required}"
-    : "${WORDPRESS_URL:=https://${DOMAIN_NAME}}"
+    : "${WORDPRESS_URL:?WORDPRESS_URL is required}"
     : "${WORDPRESS_TITLE:?WORDPRESS_TITLE is required}"
     : "${WORDPRESS_ADMIN_USER:?WORDPRESS_ADMIN_USER is required}"
     : "${WORDPRESS_ADMIN_EMAIL:?WORDPRESS_ADMIN_EMAIL is required}"
diff --git a/tests/validate_stack.py b/tests/validate_stack.py
index cf5898f..cbbfc33 100755
--- a/tests/validate_stack.py
+++ b/tests/validate_stack.py
@@ -62,7 +62,8 @@ def validate_compose() -> None:
             r"^\s+nginx:",
             r"^\s+mariadb:",
             r"^\s+wordpress:",
-            r"\"443:443\"",
+            r"HTTPS_BIND_ADDRESS:-127\.0\.0\.1",
+            r"HTTPS_PORT:-443",
             r"condition: service_healthy",
             r"healthcheck:",
             r"x-secret-files:",


## `feat(network): DB 트래픽을 내부 backend로 격리`

diff --git a/srcs/docker-compose.yml b/srcs/docker-compose.yml
index 096e4cd..c59062f 100644
--- a/srcs/docker-compose.yml
+++ b/srcs/docker-compose.yml
@@ -20,7 +20,7 @@ services:
       wordpress:
         condition: service_healthy
     networks:
-      - inception
+      - frontend
     healthcheck:
       test: ["CMD-SHELL", "curl -kfsS https://127.0.0.1/healthz >/dev/null"]
       interval: 10s
@@ -39,7 +39,7 @@ services:
     volumes:
       - mariadb_data:/var/lib/mysql-volume
     networks:
-      - inception
+      - backend
     healthcheck:
       test: ["CMD-SHELL", "test -f /var/lib/mysql-volume/data/.container-stack-initialized && test -S /run/mysqld/mysqld.sock && kill -0 1"]
       interval: 10s
@@ -70,7 +70,8 @@ services:
       mariadb:
         condition: service_healthy
     networks:
-      - inception
+      - frontend
+      - backend
     healthcheck:
       test: ["CMD-SHELL", "test -f /var/www/html/.container-stack-initialized && REQUEST_METHOD=GET SCRIPT_NAME=/ping SCRIPT_FILENAME=/ping cgi-fcgi -bind -connect 127.0.0.1:9000 | grep -q pong"]
       interval: 10s
@@ -79,8 +80,11 @@ services:
       start_period: 40s
 
 networks:
-  inception:
+  frontend:
     driver: bridge
+  backend:
+    driver: bridge
+    internal: true
 
 volumes:
   mariadb_data:


## `feat(runtime): 서비스 자원과 종료 한계 적용`

diff --git a/srcs/docker-compose.yml b/srcs/docker-compose.yml
index c59062f..fee8b51 100644
--- a/srcs/docker-compose.yml
+++ b/srcs/docker-compose.yml
@@ -10,6 +10,22 @@ services:
       context: ./requirements/nginx
     image: ${STACK_IMAGE_PREFIX:-container-stack}-nginx:${STACK_IMAGE_TAG:-local}
     restart: unless-stopped
+    cpus: ${NGINX_CPUS:-0.50}
+    mem_limit: ${NGINX_MEMORY:-128m}
+    pids_limit: ${NGINX_PIDS_LIMIT:-64}
+    ulimits:
+      nofile:
+        soft: ${NGINX_NOFILE_SOFT:-1024}
+        hard: ${NGINX_NOFILE_HARD:-4096}
+    stop_signal: SIGQUIT
+    stop_grace_period: 15s
+    security_opt:
+      - no-new-privileges:true
+    logging:
+      driver: json-file
+      options:
+        max-size: "10m"
+        max-file: "3"
     environment:
       DOMAIN_NAME: ${DOMAIN_NAME:?set DOMAIN_NAME}
     ports:
@@ -33,6 +49,22 @@ services:
       context: ./requirements/mariadb
     image: ${STACK_IMAGE_PREFIX:-container-stack}-mariadb:${STACK_IMAGE_TAG:-local}
     restart: unless-stopped
+    cpus: ${MARIADB_CPUS:-1.00}
+    mem_limit: ${MARIADB_MEMORY:-512m}
+    pids_limit: ${MARIADB_PIDS_LIMIT:-256}
+    ulimits:
+      nofile:
+        soft: ${MARIADB_NOFILE_SOFT:-4096}
+        hard: ${MARIADB_NOFILE_HARD:-65536}
+    stop_signal: SIGTERM
+    stop_grace_period: 60s
+    security_opt:
+      - no-new-privileges:true
+    logging:
+      driver: json-file
+      options:
+        max-size: "10m"
+        max-file: "3"
     environment:
       MYSQL_DATABASE: ${MYSQL_DATABASE:?set MYSQL_DATABASE}
       MYSQL_USER: ${MYSQL_USER:?set MYSQL_USER}
@@ -52,6 +84,22 @@ services:
       context: ./requirements/wordpress
     image: ${STACK_IMAGE_PREFIX:-container-stack}-wordpress:${STACK_IMAGE_TAG:-local}
     restart: unless-stopped
+    cpus: ${WORDPRESS_CPUS:-1.00}
+    mem_limit: ${WORDPRESS_MEMORY:-512m}
+    pids_limit: ${WORDPRESS_PIDS_LIMIT:-256}
+    ulimits:
+      nofile:
+        soft: ${WORDPRESS_NOFILE_SOFT:-1024}
+        hard: ${WORDPRESS_NOFILE_HARD:-4096}
+    stop_signal: SIGQUIT
+    stop_grace_period: 30s
+    security_opt:
+      - no-new-privileges:true
+    logging:
+      driver: json-file
+      options:
+        max-size: "10m"
+        max-file: "3"
     environment:
       DOMAIN_NAME: ${DOMAIN_NAME:?set DOMAIN_NAME}
       WORDPRESS_URL: ${WORDPRESS_URL:?set WORDPRESS_URL}


## `feat(nginx): 접근·오류 로그를 컨테이너 스트림에 게시`

diff --git a/srcs/requirements/nginx/conf/nginx.conf b/srcs/requirements/nginx/conf/nginx.conf
index 1a151a6..c1806a9 100644
--- a/srcs/requirements/nginx/conf/nginx.conf
+++ b/srcs/requirements/nginx/conf/nginx.conf
@@ -5,6 +5,8 @@ server {
     server_name _;
     root /var/www/html;
     index index.php index.html;
+    access_log /dev/stdout;
+    error_log /dev/stderr warn;
 
     ssl_certificate /etc/nginx/ssl/inception.crt;
     ssl_certificate_key /etc/nginx/ssl/inception.key;


## `fix(make): 볼륨 삭제 전에 확인을 요구`

diff --git a/Makefile b/Makefile
index 1fd6193..b51d679 100644
--- a/Makefile
+++ b/Makefile
@@ -5,6 +5,7 @@ PROJECT_NAME ?= container-stack
 WAIT_TIMEOUT ?= 300
 BACKUP_DIR ?=
 NEW_SECRETS_DIR ?=
+DESTROY_CONFIRM ?=
 
 COMPOSE_RUN := $(COMPOSE) --project-name $(PROJECT_NAME) --env-file $(ENV_FILE) -f $(COMPOSE_FILE)
 
@@ -34,6 +35,10 @@ ps:
 clean: down
 
 fclean:
+	@test -n "$(PROJECT_NAME)" && test "$(DESTROY_CONFIRM)" = "$(PROJECT_NAME)" || { \
+		echo "볼륨과 로컬 이미지를 삭제하려면 DESTROY_CONFIRM=$(PROJECT_NAME)을 지정하십시오." >&2; \
+		exit 2; \
+	}
 	$(COMPOSE_RUN) down -v --rmi local --remove-orphans
 
 config:


