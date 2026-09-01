# WordPress 재조정 초기화와 설정 격리

## `feat(wordpress): Debian PHP-FPM 이미지 추가`

diff --git a/srcs/requirements/wordpress/Dockerfile b/srcs/requirements/wordpress/Dockerfile
new file mode 100644
index 0000000..368aa37
--- /dev/null
+++ b/srcs/requirements/wordpress/Dockerfile
@@ -0,0 +1,31 @@
+FROM debian:bookworm-slim
+
+RUN apt-get update \
+    && apt-get install -y --no-install-recommends \
+        ca-certificates \
+        curl \
+        default-mysql-client \
+        libfcgi-bin \
+        php8.2-cli \
+        php8.2-curl \
+        php8.2-fpm \
+        php8.2-gd \
+        php8.2-intl \
+        php8.2-mbstring \
+        php8.2-mysql \
+        php8.2-xml \
+        php8.2-zip \
+        unzip \
+    && rm -rf /var/lib/apt/lists/*
+
+RUN curl -fsSL https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar \
+        -o /usr/local/bin/wp \
+    && chmod +x /usr/local/bin/wp \
+    && mkdir -p /run/php /var/www/html \
+    && chown -R www-data:www-data /run/php /var/www/html
+
+WORKDIR /var/www/html
+
+EXPOSE 9000
+
+CMD ["php-fpm8.2", "-F"]


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


## `feat(wordpress): 사이트와 사용자 계정 초기화`

diff --git a/srcs/requirements/wordpress/Dockerfile b/srcs/requirements/wordpress/Dockerfile
index 1958703..722104b 100644
--- a/srcs/requirements/wordpress/Dockerfile
+++ b/srcs/requirements/wordpress/Dockerfile
@@ -27,7 +27,11 @@ RUN curl -fsSL https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-
 WORKDIR /var/www/html
 
 COPY conf/www.conf /etc/php/8.2/fpm/pool.d/www.conf
+COPY tools/docker-entrypoint.sh /usr/local/bin/docker-entrypoint.sh
+
+RUN chmod +x /usr/local/bin/docker-entrypoint.sh
 
 EXPOSE 9000
 
+ENTRYPOINT ["docker-entrypoint.sh"]
 CMD ["php-fpm8.2", "-F"]
diff --git a/srcs/requirements/wordpress/tools/docker-entrypoint.sh b/srcs/requirements/wordpress/tools/docker-entrypoint.sh
new file mode 100755
index 0000000..fd01d68
--- /dev/null
+++ b/srcs/requirements/wordpress/tools/docker-entrypoint.sh
@@ -0,0 +1,89 @@
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
+file_env WORDPRESS_DB_PASSWORD
+file_env WORDPRESS_ADMIN_PASSWORD
+file_env WORDPRESS_USER_PASSWORD
+
+: "${WORDPRESS_DB_HOST:=mariadb}"
+: "${MYSQL_DATABASE:?MYSQL_DATABASE is required}"
+: "${MYSQL_USER:?MYSQL_USER is required}"
+: "${WORDPRESS_DB_PASSWORD:?WORDPRESS_DB_PASSWORD is required}"
+: "${DOMAIN_NAME:?DOMAIN_NAME is required}"
+: "${WORDPRESS_TITLE:?WORDPRESS_TITLE is required}"
+: "${WORDPRESS_ADMIN_USER:?WORDPRESS_ADMIN_USER is required}"
+: "${WORDPRESS_ADMIN_PASSWORD:?WORDPRESS_ADMIN_PASSWORD is required}"
+: "${WORDPRESS_ADMIN_EMAIL:?WORDPRESS_ADMIN_EMAIL is required}"
+: "${WORDPRESS_USER:?WORDPRESS_USER is required}"
+: "${WORDPRESS_USER_PASSWORD:?WORDPRESS_USER_PASSWORD is required}"
+: "${WORDPRESS_USER_EMAIL:?WORDPRESS_USER_EMAIL is required}"
+
+install -d -m 0755 -o www-data -g www-data /run/php /var/www/html
+
+for _ in $(seq 1 60); do
+    if mysqladmin ping -h"$WORDPRESS_DB_HOST" -u"$MYSQL_USER" -p"$WORDPRESS_DB_PASSWORD" --silent; then
+        break
+    fi
+    sleep 2
+done
+
+if [ ! -f /var/www/html/wp-includes/version.php ]; then
+    wp core download --allow-root --path=/var/www/html
+fi
+
+if [ ! -f /var/www/html/wp-config.php ]; then
+    wp config create --allow-root \
+        --path=/var/www/html \
+        --dbname="$MYSQL_DATABASE" \
+        --dbuser="$MYSQL_USER" \
+        --dbpass="$WORDPRESS_DB_PASSWORD" \
+        --dbhost="$WORDPRESS_DB_HOST" \
+        --skip-check
+
+    wp config set --allow-root --path=/var/www/html FS_METHOD direct --type=constant
+    wp config set --allow-root --path=/var/www/html WP_HOME "https://${DOMAIN_NAME}" --type=constant
+    wp config set --allow-root --path=/var/www/html WP_SITEURL "https://${DOMAIN_NAME}" --type=constant
+fi
+
+if ! wp core is-installed --allow-root --path=/var/www/html >/dev/null 2>&1; then
+    wp core install --allow-root \
+        --path=/var/www/html \
+        --url="https://${DOMAIN_NAME}" \
+        --title="$WORDPRESS_TITLE" \
+        --admin_user="$WORDPRESS_ADMIN_USER" \
+        --admin_password="$WORDPRESS_ADMIN_PASSWORD" \
+        --admin_email="$WORDPRESS_ADMIN_EMAIL" \
+        --skip-email
+fi
+
+if ! wp user get "$WORDPRESS_USER" --allow-root --path=/var/www/html >/dev/null 2>&1; then
+    wp user create "$WORDPRESS_USER" "$WORDPRESS_USER_EMAIL" \
+        --allow-root \
+        --path=/var/www/html \
+        --role=author \
+        --user_pass="$WORDPRESS_USER_PASSWORD"
+fi
+
+chown -R www-data:www-data /var/www/html /run/php
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


