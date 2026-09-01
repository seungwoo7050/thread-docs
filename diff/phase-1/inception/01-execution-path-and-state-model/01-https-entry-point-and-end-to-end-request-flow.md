# HTTPS 진입점과 종단 요청 경로

## `feat(mariadb): 네트워크 DB 서버 설정`

diff --git a/srcs/requirements/mariadb/Dockerfile b/srcs/requirements/mariadb/Dockerfile
index c796913..b1f4d18 100644
--- a/srcs/requirements/mariadb/Dockerfile
+++ b/srcs/requirements/mariadb/Dockerfile
@@ -12,6 +12,8 @@ RUN rm -rf /var/lib/mysql/* \
     && mkdir -p /run/mysqld /var/lib/mysql \
     && chown -R mysql:mysql /run/mysqld /var/lib/mysql
 
+COPY conf/50-server.cnf /etc/mysql/mariadb.conf.d/50-server.cnf
+
 EXPOSE 3306
 
 CMD ["mariadbd", "--user=mysql", "--console"]
diff --git a/srcs/requirements/mariadb/conf/50-server.cnf b/srcs/requirements/mariadb/conf/50-server.cnf
new file mode 100644
index 0000000..7978f87
--- /dev/null
+++ b/srcs/requirements/mariadb/conf/50-server.cnf
@@ -0,0 +1,16 @@
+[mysqld]
+bind-address=0.0.0.0
+port=3306
+datadir=/var/lib/mysql
+socket=/run/mysqld/mysqld.sock
+pid-file=/run/mysqld/mysqld.pid
+
+skip-name-resolve
+character-set-server=utf8mb4
+collation-server=utf8mb4_unicode_ci
+innodb_buffer_pool_size=128M
+max_connections=80
+
+[client]
+socket=/run/mysqld/mysqld.sock
+default-character-set=utf8mb4


## `feat(wordpress): PHP-FPM 풀 설정`

diff --git a/srcs/requirements/wordpress/Dockerfile b/srcs/requirements/wordpress/Dockerfile
index 368aa37..1958703 100644
--- a/srcs/requirements/wordpress/Dockerfile
+++ b/srcs/requirements/wordpress/Dockerfile
@@ -26,6 +26,8 @@ RUN curl -fsSL https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-
 
 WORKDIR /var/www/html
 
+COPY conf/www.conf /etc/php/8.2/fpm/pool.d/www.conf
+
 EXPOSE 9000
 
 CMD ["php-fpm8.2", "-F"]
diff --git a/srcs/requirements/wordpress/conf/www.conf b/srcs/requirements/wordpress/conf/www.conf
new file mode 100644
index 0000000..5a357f8
--- /dev/null
+++ b/srcs/requirements/wordpress/conf/www.conf
@@ -0,0 +1,23 @@
+[www]
+user = www-data
+group = www-data
+
+listen = 0.0.0.0:9000
+listen.owner = www-data
+listen.group = www-data
+
+pm = dynamic
+pm.max_children = 12
+pm.start_servers = 3
+pm.min_spare_servers = 2
+pm.max_spare_servers = 5
+
+clear_env = no
+catch_workers_output = yes
+
+ping.path = /ping
+ping.response = pong
+
+access.log = /proc/self/fd/2
+php_admin_value[error_log] = /proc/self/fd/2
+php_admin_flag[log_errors] = on


## `feat(nginx): TLS 프런트엔드 이미지 추가`

diff --git a/srcs/requirements/nginx/Dockerfile b/srcs/requirements/nginx/Dockerfile
new file mode 100644
index 0000000..5e81669
--- /dev/null
+++ b/srcs/requirements/nginx/Dockerfile
@@ -0,0 +1,20 @@
+FROM debian:bookworm-slim
+
+RUN apt-get update \
+    && apt-get install -y --no-install-recommends \
+        ca-certificates \
+        curl \
+        nginx \
+        openssl \
+    && rm -rf /var/lib/apt/lists/*
+
+RUN mkdir -p /etc/nginx/ssl /var/www/html /run/nginx
+
+COPY tools/docker-entrypoint.sh /usr/local/bin/docker-entrypoint.sh
+
+RUN chmod +x /usr/local/bin/docker-entrypoint.sh
+
+EXPOSE 443
+
+ENTRYPOINT ["docker-entrypoint.sh"]
+CMD ["nginx", "-g", "daemon off;"]
diff --git a/srcs/requirements/nginx/tools/docker-entrypoint.sh b/srcs/requirements/nginx/tools/docker-entrypoint.sh
new file mode 100755
index 0000000..3deb5e7
--- /dev/null
+++ b/srcs/requirements/nginx/tools/docker-entrypoint.sh
@@ -0,0 +1,19 @@
+#!/bin/sh
+set -eu
+
+: "${DOMAIN_NAME:=localhost}"
+
+cert_dir=/etc/nginx/ssl
+cert_file="${cert_dir}/inception.crt"
+key_file="${cert_dir}/inception.key"
+
+mkdir -p "$cert_dir" /run/nginx
+
+if [ ! -s "$cert_file" ] || [ ! -s "$key_file" ]; then
+    openssl req -x509 -nodes -newkey rsa:2048 -days 365 \
+        -subj "/CN=${DOMAIN_NAME}" \
+        -keyout "$key_file" \
+        -out "$cert_file" >/dev/null 2>&1
+fi
+
+exec "$@"


## `feat(nginx): PHP 요청을 WordPress로 전달`

diff --git a/srcs/requirements/nginx/Dockerfile b/srcs/requirements/nginx/Dockerfile
index 5e81669..0c0a91a 100644
--- a/srcs/requirements/nginx/Dockerfile
+++ b/srcs/requirements/nginx/Dockerfile
@@ -10,6 +10,7 @@ RUN apt-get update \
 
 RUN mkdir -p /etc/nginx/ssl /var/www/html /run/nginx
 
+COPY conf/nginx.conf /etc/nginx/sites-available/default
 COPY tools/docker-entrypoint.sh /usr/local/bin/docker-entrypoint.sh
 
 RUN chmod +x /usr/local/bin/docker-entrypoint.sh
diff --git a/srcs/requirements/nginx/conf/nginx.conf b/srcs/requirements/nginx/conf/nginx.conf
new file mode 100644
index 0000000..1a151a6
--- /dev/null
+++ b/srcs/requirements/nginx/conf/nginx.conf
@@ -0,0 +1,40 @@
+server {
+    listen 443 ssl http2;
+    listen [::]:443 ssl http2;
+
+    server_name _;
+    root /var/www/html;
+    index index.php index.html;
+
+    ssl_certificate /etc/nginx/ssl/inception.crt;
+    ssl_certificate_key /etc/nginx/ssl/inception.key;
+    ssl_protocols TLSv1.2 TLSv1.3;
+    ssl_prefer_server_ciphers off;
+
+    client_max_body_size 64m;
+
+    add_header X-Content-Type-Options nosniff always;
+    add_header X-Frame-Options SAMEORIGIN always;
+    add_header Referrer-Policy strict-origin-when-cross-origin always;
+
+    location = /healthz {
+        access_log off;
+        add_header Content-Type text/plain;
+        return 200 "ok\n";
+    }
+
+    location / {
+        try_files $uri $uri/ /index.php?$args;
+    }
+
+    location ~ \.php$ {
+        include snippets/fastcgi-php.conf;
+        fastcgi_pass wordpress:9000;
+        fastcgi_param SCRIPT_FILENAME /var/www/html$fastcgi_script_name;
+        fastcgi_param HTTPS on;
+    }
+
+    location ~ /\. {
+        deny all;
+    }
+}


## `feat(compose): 세 서비스 토폴로지 구성`

diff --git a/srcs/docker-compose.yml b/srcs/docker-compose.yml
new file mode 100644
index 0000000..52cf8b7
--- /dev/null
+++ b/srcs/docker-compose.yml
@@ -0,0 +1,37 @@
+services:
+  nginx:
+    build:
+      context: ./requirements/nginx
+    image: inception-nginx:local
+    container_name: inception-nginx
+    restart: unless-stopped
+    ports:
+      - "443:443"
+    networks:
+      - inception
+
+  mariadb:
+    build:
+      context: ./requirements/mariadb
+    image: inception-mariadb:local
+    container_name: inception-mariadb
+    restart: unless-stopped
+    networks:
+      - inception
+
+  wordpress:
+    build:
+      context: ./requirements/wordpress
+    image: inception-wordpress:local
+    container_name: inception-wordpress
+    restart: unless-stopped
+    networks:
+      - inception
+
+networks:
+  inception:
+    driver: bridge
+
+volumes:
+  mariadb_data:
+  wordpress_data:


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


## `test(smoke): HTTPS 상태 엔드포인트 검사`

diff --git a/Makefile b/Makefile
index 4a1ecf0..7906780 100644
--- a/Makefile
+++ b/Makefile
@@ -4,7 +4,7 @@ ENV_FILE ?= .env
 
 COMPOSE_RUN := $(COMPOSE) --env-file $(ENV_FILE) -f $(COMPOSE_FILE)
 
-.PHONY: up down build logs ps clean fclean test config
+.PHONY: up down build logs ps clean fclean test config smoke
 
 up:
 	$(COMPOSE_RUN) up -d
@@ -37,3 +37,6 @@ test:
 	else \
 		echo "docker compose not available; skipped compose config"; \
 	fi
+
+smoke:
+	tools/smoke_https.sh
diff --git a/tests/validate_stack.py b/tests/validate_stack.py
index 4c73640..5e5b38c 100755
--- a/tests/validate_stack.py
+++ b/tests/validate_stack.py
@@ -155,12 +155,18 @@ def validate_env_policy() -> None:
         fail(".env.example must point to secret files instead of embedding passwords")
 
 
+def validate_tools() -> None:
+    require_executable("tools/smoke_https.sh")
+    require_text("Makefile", [r"^smoke:", r"tools/smoke_https\.sh"])
+
+
 def main() -> None:
     validate_source_only()
     validate_compose()
     validate_dockerfiles()
     validate_configs()
     validate_env_policy()
+    validate_tools()
     print("static stack validation passed")
 
 
diff --git a/tools/smoke_https.sh b/tools/smoke_https.sh
new file mode 100755
index 0000000..af6f22f
--- /dev/null
+++ b/tools/smoke_https.sh
@@ -0,0 +1,20 @@
+#!/bin/sh
+set -eu
+
+url="${SMOKE_URL:-https://localhost/healthz}"
+
+if ! command -v curl >/dev/null 2>&1; then
+    echo "curl is required for smoke checks" >&2
+    exit 1
+fi
+
+for _ in $(seq 1 "${SMOKE_RETRIES:-30}"); do
+    if curl -kfsS "$url" >/dev/null; then
+        echo "https smoke passed: $url"
+        exit 0
+    fi
+    sleep "${SMOKE_DELAY:-2}"
+done
+
+echo "https smoke failed: $url" >&2
+exit 1


## `test(e2e): HTTPS와 MariaDB를 잇는 WordPress 데이터 검증`

diff --git a/Makefile b/Makefile
index d211751..2848a7d 100644
--- a/Makefile
+++ b/Makefile
@@ -6,7 +6,7 @@ WAIT_TIMEOUT ?= 300
 
 COMPOSE_RUN := $(COMPOSE) --project-name $(PROJECT_NAME) --env-file $(ENV_FILE) -f $(COMPOSE_FILE)
 
-.PHONY: up start-database start-application down build logs ps clean fclean test config smoke bootstrap-test
+.PHONY: up start-database start-application down build logs ps clean fclean test config smoke bootstrap-test e2e
 
 up:
 	python3 tools/start_stack.py start --project "$(PROJECT_NAME)" --env-file "$(ENV_FILE)" --wait-timeout "$(WAIT_TIMEOUT)"
@@ -51,3 +51,6 @@ smoke:
 
 bootstrap-test:
 	python3 tests/runtime_stack.py bootstrap
+
+e2e:
+	python3 tests/runtime_stack.py e2e
diff --git a/tests/runtime_stack.py b/tests/runtime_stack.py
index a88c538..fdb2739 100644
--- a/tests/runtime_stack.py
+++ b/tests/runtime_stack.py
@@ -328,6 +328,27 @@ class RuntimeStack:
             if self.credential_values[filename] in config:
                 raise StackError(f"wp-config.php에 불필요한 비밀값이 남았습니다: {filename}")
 
+    def fetch(self, path: str) -> str:
+        url = f"https://{self.domain}:{self.port}{path}"
+        result = subprocess.run(
+            [
+                "curl",
+                "--fail",
+                "--silent",
+                "--show-error",
+                "--insecure",
+                "--noproxy",
+                "*",
+                "--resolve",
+                f"{self.domain}:{self.port}:127.0.0.1",
+                url,
+            ],
+            check=True,
+            text=True,
+            capture_output=True,
+            timeout=PROCESS_TIMEOUT_SECONDS,
+        )
+        return result.stdout
 
     def verify_bootstrap(self) -> None:
         self.start()
@@ -350,6 +371,81 @@ class RuntimeStack:
                 raise StackError(f"{service} 초기화 완료 표식이 없습니다")
         print("bootstrap completion and secret boundary passed")
 
+    def verify_e2e(self) -> None:
+        blocked_port = self.port
+        with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as listener:
+            listener.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
+            listener.bind(("127.0.0.1", blocked_port))
+            listener.listen()
+            self.start()
+        if self.port == blocked_port:
+            raise StackError("HTTPS 포트 충돌 뒤 새 포트를 선택하지 않았습니다")
+        self._verify_legacy_config_migration()
+        self.assert_runtime_secret_boundary()
+        if self.fetch("/healthz").strip() != "ok":
+            raise StackError("nginx 상태 응답이 예상과 다릅니다")
+
+        nonce = secrets.token_hex(8)
+        title = f"종단 검증 {nonce}"
+        content = f"nginx-fpm-wordpress-mariadb-{nonce}"
+        post_id = self.wordpress(
+            "post",
+            "create",
+            f"--post_title={title}",
+            f"--post_content={content}",
+            "--post_status=publish",
+            "--porcelain",
+            capture=True,
+        )
+        if not post_id.isdigit():
+            raise StackError(f"WordPress가 유효한 글 번호를 반환하지 않았습니다: {post_id!r}")
+        page = self.fetch(f"/?p={post_id}")
+        if title not in page or content not in page:
+            raise StackError("HTTPS 응답에서 방금 저장한 글을 찾지 못했습니다")
+
+        database_value = self.wordpress(
+            "db",
+            "query",
+            f"SELECT post_content FROM wp_posts WHERE ID={post_id}",
+            "--skip-column-names",
+            capture=True,
+        )
+        if content not in database_value:
+            raise StackError("MariaDB 조회 결과가 WordPress 입력과 다릅니다")
+        print(f"isolated end-to-end check passed: project={self.project} port={self.port}")
+
+    def _verify_legacy_config_migration(self) -> None:
+        self.run_compose("stop", "nginx", "wordpress")
+        self.run_compose(
+            "run",
+            "--rm",
+            "--no-TTY",
+            "--no-deps",
+            "--entrypoint",
+            "sh",
+            "wordpress",
+            "-ceu",
+            "cp -p /var/www/config/wp-config.php /var/www/html/.wp-config.legacy; "
+            "rm -f /var/www/html/wp-config.php /var/www/config/wp-config.php; "
+            "mv /var/www/html/.wp-config.legacy /var/www/html/wp-config.php",
+        )
+        self._run_start("application")
+        migrated = self.run_compose(
+            "exec",
+            "--no-TTY",
+            "wordpress",
+            "sh",
+            "-ceu",
+            "test -L /var/www/html/wp-config.php; "
+            "test \"$(readlink /var/www/html/wp-config.php)\" = "
+            "/var/www/config/wp-config.php; "
+            "test -f /var/www/config/wp-config.php; "
+            "test \"$(stat -c %a /var/www/config/wp-config.php)\" = 600",
+            capture=True,
+            check=False,
+        )
+        if migrated.returncode != 0:
+            raise StackError("기존 WordPress 설정을 전용 볼륨으로 옮기지 못했습니다")
 
     def collect_diagnostics(self) -> Path:
         destination = self.diagnostics_dir
@@ -392,7 +488,7 @@ class RuntimeStack:
 
 def parse_arguments() -> argparse.Namespace:
     parser = argparse.ArgumentParser(description="격리된 컨테이너 스택 검증")
-    parser.add_argument("scenario", choices=("bootstrap",))
+    parser.add_argument("scenario", choices=("bootstrap", "e2e"))
     parser.add_argument("--keep", action="store_true", help="검사 뒤 프로젝트를 유지합니다")
     parser.add_argument("--diagnostics-dir", type=Path)
     return parser.parse_args()
@@ -416,7 +512,10 @@ def main() -> int:
 
     failed = True
     try:
-        stack.verify_bootstrap()
+        if args.scenario == "bootstrap":
+            stack.verify_bootstrap()
+        else:
+            stack.verify_e2e()
         failed = False
         return 0
     except (OSError, StackError, subprocess.SubprocessError) as error:
diff --git a/tests/validate_stack.py b/tests/validate_stack.py
index e345099..f02e68f 100755
--- a/tests/validate_stack.py
+++ b/tests/validate_stack.py
@@ -180,13 +180,18 @@ def validate_tools() -> None:
             r"tools/smoke_https\.sh",
             r"^bootstrap-test:",
             r"runtime_stack\.py bootstrap",
+            r"^e2e:",
+            r"runtime_stack\.py e2e",
         ],
     )
     require_text(
         "tests/runtime_stack.py",
         [
             r"--project-name",
+            r"--resolve",
+            r'"post",\s*\n\s*"create"',
             r"tools.+start_stack\.py",
+            r'"bootstrap",\s*"e2e"',
         ],
     )
 


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


## `fix(smoke): HTTPS 연결과 응답 대기시간 제한`

diff --git a/tools/smoke_https.sh b/tools/smoke_https.sh
index af6f22f..017da23 100755
--- a/tools/smoke_https.sh
+++ b/tools/smoke_https.sh
@@ -9,7 +9,7 @@ if ! command -v curl >/dev/null 2>&1; then
 fi
 
 for _ in $(seq 1 "${SMOKE_RETRIES:-30}"); do
-    if curl -kfsS "$url" >/dev/null; then
+    if curl -kfsS --connect-timeout 5 --max-time 15 "$url" >/dev/null; then
         echo "https smoke passed: $url"
         exit 0
     fi


## `test(smoke): HTTPS timeout 계약 검사`

diff --git a/tests/validate_stack.py b/tests/validate_stack.py
index 3fca706..fccbbdf 100755
--- a/tests/validate_stack.py
+++ b/tests/validate_stack.py
@@ -214,6 +214,7 @@ def validate_env_policy() -> None:
 
 def validate_tools() -> None:
     require_executable("tools/smoke_https.sh")
+    require_text("tools/smoke_https.sh", [r"curl .+--connect-timeout", r"curl .+--max-time"])
     require_executable("tools/start_stack.py")
     require_file("tools/stack_runtime.py")
     require_text(
