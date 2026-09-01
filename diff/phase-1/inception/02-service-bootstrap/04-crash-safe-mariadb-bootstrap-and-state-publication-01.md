# MariaDB 충돌 안전 초기화와 상태 게시

## `feat(mariadb): Debian 서버 이미지 추가`

diff --git a/srcs/requirements/mariadb/Dockerfile b/srcs/requirements/mariadb/Dockerfile
new file mode 100644
index 0000000..c796913
--- /dev/null
+++ b/srcs/requirements/mariadb/Dockerfile
@@ -0,0 +1,17 @@
+FROM debian:bookworm-slim
+
+RUN apt-get update \
+    && apt-get install -y --no-install-recommends \
+        ca-certificates \
+        gosu \
+        mariadb-client \
+        mariadb-server \
+    && rm -rf /var/lib/apt/lists/*
+
+RUN rm -rf /var/lib/mysql/* \
+    && mkdir -p /run/mysqld /var/lib/mysql \
+    && chown -R mysql:mysql /run/mysqld /var/lib/mysql
+
+EXPOSE 3306
+
+CMD ["mariadbd", "--user=mysql", "--console"]


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


## `feat(mariadb): DB와 애플리케이션 계정 초기화`

diff --git a/srcs/requirements/mariadb/Dockerfile b/srcs/requirements/mariadb/Dockerfile
index b1f4d18..91d4744 100644
--- a/srcs/requirements/mariadb/Dockerfile
+++ b/srcs/requirements/mariadb/Dockerfile
@@ -13,7 +13,11 @@ RUN rm -rf /var/lib/mysql/* \
     && chown -R mysql:mysql /run/mysqld /var/lib/mysql
 
 COPY conf/50-server.cnf /etc/mysql/mariadb.conf.d/50-server.cnf
+COPY tools/docker-entrypoint.sh /usr/local/bin/docker-entrypoint.sh
+
+RUN chmod +x /usr/local/bin/docker-entrypoint.sh
 
 EXPOSE 3306
 
+ENTRYPOINT ["docker-entrypoint.sh"]
 CMD ["mariadbd", "--user=mysql", "--console"]
diff --git a/srcs/requirements/mariadb/tools/docker-entrypoint.sh b/srcs/requirements/mariadb/tools/docker-entrypoint.sh
new file mode 100755
index 0000000..1a79473
--- /dev/null
+++ b/srcs/requirements/mariadb/tools/docker-entrypoint.sh
@@ -0,0 +1,82 @@
+#!/bin/sh
+set -eu
+
+file_env() {
+    var="$1"
+    file_var="${var}_FILE"
+    value="${2:-}"
+    eval current="\${$var:-}"
+    eval file_path="\${$file_var:-}"
+
+    if [ -n "$current" ] && [ -n "$file_path" ]; then
+        echo "$var and $file_var are mutually exclusive" >&2
+        exit 1
+    fi
+    if [ -n "$file_path" ]; then
+        value="$(cat "$file_path")"
+    elif [ -n "$current" ]; then
+        value="$current"
+    fi
+    export "$var=$value"
+    unset "$file_var"
+}
+
+require_name() {
+    case "$2" in
+        *[!A-Za-z0-9_]*|"")
+            echo "$1 must contain only letters, numbers, and underscores" >&2
+            exit 1
+            ;;
+    esac
+}
+
+sql_escape() {
+    printf "%s" "$1" | sed "s/'/''/g"
+}
+
+file_env MYSQL_ROOT_PASSWORD
+file_env MYSQL_PASSWORD
+
+: "${MYSQL_DATABASE:?MYSQL_DATABASE is required}"
+: "${MYSQL_USER:?MYSQL_USER is required}"
+: "${MYSQL_ROOT_PASSWORD:?MYSQL_ROOT_PASSWORD is required}"
+: "${MYSQL_PASSWORD:?MYSQL_PASSWORD is required}"
+
+require_name MYSQL_DATABASE "$MYSQL_DATABASE"
+require_name MYSQL_USER "$MYSQL_USER"
+
+install -d -m 0755 -o mysql -g mysql /run/mysqld /var/lib/mysql
+
+if [ ! -d /var/lib/mysql/mysql ]; then
+    mariadb-install-db --user=mysql --datadir=/var/lib/mysql --skip-test-db >/dev/null
+
+    mariadbd --user=mysql --datadir=/var/lib/mysql --skip-networking \
+        --socket=/run/mysqld/mysqld.sock &
+    pid="$!"
+
+    for _ in $(seq 1 60); do
+        if mysqladmin --socket=/run/mysqld/mysqld.sock ping --silent; then
+            break
+        fi
+        sleep 1
+    done
+
+    root_password="$(sql_escape "$MYSQL_ROOT_PASSWORD")"
+    user_password="$(sql_escape "$MYSQL_PASSWORD")"
+
+    mariadb --socket=/run/mysqld/mysqld.sock <<SQL
+ALTER USER 'root'@'localhost' IDENTIFIED BY '${root_password}';
+DELETE FROM mysql.user WHERE User='';
+DELETE FROM mysql.user WHERE User='root' AND Host NOT IN ('localhost');
+DROP DATABASE IF EXISTS test;
+CREATE DATABASE IF NOT EXISTS \`${MYSQL_DATABASE}\` CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
+CREATE USER IF NOT EXISTS '${MYSQL_USER}'@'%' IDENTIFIED BY '${user_password}';
+GRANT ALL PRIVILEGES ON \`${MYSQL_DATABASE}\`.* TO '${MYSQL_USER}'@'%';
+FLUSH PRIVILEGES;
+SQL
+
+    mysqladmin --socket=/run/mysqld/mysqld.sock -uroot -p"$MYSQL_ROOT_PASSWORD" shutdown
+    wait "$pid"
+fi
+
+exec "$@"


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


