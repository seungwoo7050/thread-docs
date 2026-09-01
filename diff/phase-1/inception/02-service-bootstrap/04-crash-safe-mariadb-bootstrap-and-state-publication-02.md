## `fix(init): 중단된 단계별 초기화를 수렴`

diff --git a/Makefile b/Makefile
index 7906780..d6aded2 100644
--- a/Makefile
+++ b/Makefile
@@ -1,13 +1,21 @@
 COMPOSE := docker compose
 COMPOSE_FILE := srcs/docker-compose.yml
 ENV_FILE ?= .env
+PROJECT_NAME ?= container-stack
+WAIT_TIMEOUT ?= 300
 
-COMPOSE_RUN := $(COMPOSE) --env-file $(ENV_FILE) -f $(COMPOSE_FILE)
+COMPOSE_RUN := $(COMPOSE) --project-name $(PROJECT_NAME) --env-file $(ENV_FILE) -f $(COMPOSE_FILE)
 
-.PHONY: up down build logs ps clean fclean test config smoke
+.PHONY: up start-database start-application down build logs ps clean fclean test config smoke
 
 up:
-	$(COMPOSE_RUN) up -d
+	python3 tools/start_stack.py start --project "$(PROJECT_NAME)" --env-file "$(ENV_FILE)" --wait-timeout "$(WAIT_TIMEOUT)"
+
+start-database:
+	python3 tools/start_stack.py database --project "$(PROJECT_NAME)" --env-file "$(ENV_FILE)" --wait-timeout "$(WAIT_TIMEOUT)"
+
+start-application:
+	python3 tools/start_stack.py application --project "$(PROJECT_NAME)" --env-file "$(ENV_FILE)" --wait-timeout "$(WAIT_TIMEOUT)"
 
 down:
 	$(COMPOSE_RUN) down --remove-orphans
diff --git a/srcs/docker-compose.yml b/srcs/docker-compose.yml
index 31180f6..01d4174 100644
--- a/srcs/docker-compose.yml
+++ b/srcs/docker-compose.yml
@@ -1,3 +1,9 @@
+x-secret-files:
+  db_root_password: ${DB_ROOT_PASSWORD_FILE:-../secrets/db_root_password.txt}
+  db_password: ${DB_PASSWORD_FILE:-../secrets/db_password.txt}
+  wp_admin_password: ${WP_ADMIN_PASSWORD_FILE:-../secrets/wp_admin_password.txt}
+  wp_user_password: ${WP_USER_PASSWORD_FILE:-../secrets/wp_user_password.txt}
+
 services:
   nginx:
     build:
@@ -32,17 +38,12 @@ services:
     environment:
       MYSQL_DATABASE: ${MYSQL_DATABASE:?set MYSQL_DATABASE}
       MYSQL_USER: ${MYSQL_USER:?set MYSQL_USER}
-      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/db_root_password
-      MYSQL_PASSWORD_FILE: /run/secrets/db_password
-    secrets:
-      - db_root_password
-      - db_password
     volumes:
-      - mariadb_data:/var/lib/mysql
+      - mariadb_data:/var/lib/mysql-volume
     networks:
       - inception
     healthcheck:
-      test: ["CMD-SHELL", "mysqladmin --socket=/run/mysqld/mysqld.sock -uroot -p\"$$(cat /run/secrets/db_root_password)\" ping --silent"]
+      test: ["CMD-SHELL", "test -f /var/lib/mysql-volume/data/.container-stack-initialized && test -S /run/mysqld/mysqld.sock && kill -0 1"]
       interval: 10s
       timeout: 5s
       retries: 8
@@ -59,27 +60,21 @@ services:
       WORDPRESS_DB_HOST: mariadb
       MYSQL_DATABASE: ${MYSQL_DATABASE:?set MYSQL_DATABASE}
       MYSQL_USER: ${MYSQL_USER:?set MYSQL_USER}
-      WORDPRESS_DB_PASSWORD_FILE: /run/secrets/db_password
       WORDPRESS_TITLE: ${WORDPRESS_TITLE:?set WORDPRESS_TITLE}
       WORDPRESS_ADMIN_USER: ${WORDPRESS_ADMIN_USER:?set WORDPRESS_ADMIN_USER}
-      WORDPRESS_ADMIN_PASSWORD_FILE: /run/secrets/wp_admin_password
       WORDPRESS_ADMIN_EMAIL: ${WORDPRESS_ADMIN_EMAIL:?set WORDPRESS_ADMIN_EMAIL}
       WORDPRESS_USER: ${WORDPRESS_USER:?set WORDPRESS_USER}
-      WORDPRESS_USER_PASSWORD_FILE: /run/secrets/wp_user_password
       WORDPRESS_USER_EMAIL: ${WORDPRESS_USER_EMAIL:?set WORDPRESS_USER_EMAIL}
-    secrets:
-      - db_password
-      - wp_admin_password
-      - wp_user_password
     volumes:
       - wordpress_data:/var/www/html
+      - wordpress_config:/var/www/config
     depends_on:
       mariadb:
         condition: service_healthy
     networks:
       - inception
     healthcheck:
-      test: ["CMD-SHELL", "REQUEST_METHOD=GET SCRIPT_NAME=/ping SCRIPT_FILENAME=/ping cgi-fcgi -bind -connect 127.0.0.1:9000 | grep -q pong"]
+      test: ["CMD-SHELL", "test -f /var/www/html/.container-stack-initialized && REQUEST_METHOD=GET SCRIPT_NAME=/ping SCRIPT_FILENAME=/ping cgi-fcgi -bind -connect 127.0.0.1:9000 | grep -q pong"]
       interval: 10s
       timeout: 5s
       retries: 8
@@ -92,13 +87,4 @@ networks:
 volumes:
   mariadb_data:
   wordpress_data:
-
-secrets:
-  db_root_password:
-    file: ${DB_ROOT_PASSWORD_FILE:-../secrets/db_root_password.txt}
-  db_password:
-    file: ${DB_PASSWORD_FILE:-../secrets/db_password.txt}
-  wp_admin_password:
-    file: ${WP_ADMIN_PASSWORD_FILE:-../secrets/wp_admin_password.txt}
-  wp_user_password:
-    file: ${WP_USER_PASSWORD_FILE:-../secrets/wp_user_password.txt}
+  wordpress_config:
diff --git a/srcs/requirements/mariadb/Dockerfile b/srcs/requirements/mariadb/Dockerfile
index 91d4744..07ad95e 100644
--- a/srcs/requirements/mariadb/Dockerfile
+++ b/srcs/requirements/mariadb/Dockerfile
@@ -8,9 +8,9 @@ RUN apt-get update \
         mariadb-server \
     && rm -rf /var/lib/apt/lists/*
 
-RUN rm -rf /var/lib/mysql/* \
-    && mkdir -p /run/mysqld /var/lib/mysql \
-    && chown -R mysql:mysql /run/mysqld /var/lib/mysql
+RUN rm -rf /var/lib/mysql \
+    && mkdir -p /run/mysqld /var/lib/mysql-volume \
+    && chown -R mysql:mysql /run/mysqld /var/lib/mysql-volume
 
 COPY conf/50-server.cnf /etc/mysql/mariadb.conf.d/50-server.cnf
 COPY tools/docker-entrypoint.sh /usr/local/bin/docker-entrypoint.sh
diff --git a/srcs/requirements/mariadb/conf/50-server.cnf b/srcs/requirements/mariadb/conf/50-server.cnf
index 7978f87..7c473ee 100644
--- a/srcs/requirements/mariadb/conf/50-server.cnf
+++ b/srcs/requirements/mariadb/conf/50-server.cnf
@@ -1,7 +1,7 @@
 [mysqld]
 bind-address=0.0.0.0
 port=3306
-datadir=/var/lib/mysql
+datadir=/var/lib/mysql-volume/data
 socket=/run/mysqld/mysqld.sock
 pid-file=/run/mysqld/mysqld.pid
 
diff --git a/srcs/requirements/mariadb/tools/docker-entrypoint.sh b/srcs/requirements/mariadb/tools/docker-entrypoint.sh
index 1a79473..0737ae4 100755
--- a/srcs/requirements/mariadb/tools/docker-entrypoint.sh
+++ b/srcs/requirements/mariadb/tools/docker-entrypoint.sh
@@ -1,82 +1,209 @@
 #!/bin/sh
 set -eu
 
-file_env() {
-    var="$1"
-    file_var="${var}_FILE"
-    value="${2:-}"
-    eval current="\${$var:-}"
-    eval file_path="\${$file_var:-}"
-
-    if [ -n "$current" ] && [ -n "$file_path" ]; then
-        echo "$var and $file_var are mutually exclusive" >&2
-        exit 1
-    fi
-    if [ -n "$file_path" ]; then
-        value="$(cat "$file_path")"
-    elif [ -n "$current" ]; then
-        value="$current"
-    fi
-    export "$var=$value"
-    unset "$file_var"
+volume_dir="${MARIADB_VOLUME_DIR:-/var/lib/mysql-volume}"
+data_dir="${volume_dir}/data"
+staging_dir="${volume_dir}/.container-stack-bootstrap"
+run_dir="${MARIADB_RUN_DIR:-/run/mysqld}"
+socket="${run_dir}/mysqld.sock"
+marker="${data_dir}/.container-stack-initialized"
+wait_retries="${MARIADB_INIT_WAIT_RETRIES:-60}"
+wait_delay="${MARIADB_INIT_WAIT_DELAY:-1}"
+temporary_pid=""
+root_option_file=""
+app_option_file=""
+
+fail() {
+    echo "$*" >&2
+    exit 1
 }
 
 require_name() {
     case "$2" in
         *[!A-Za-z0-9_]*|"")
-            echo "$1 must contain only letters, numbers, and underscores" >&2
-            exit 1
+            fail "$1 must contain only letters, numbers, and underscores"
             ;;
     esac
 }
 
-sql_escape() {
-    printf "%s" "$1" | sed "s/'/''/g"
+require_password() {
+    case "$2" in
+        *[!A-Za-z0-9_.~!@#%^+=,-]*|"")
+            fail "$1 has an invalid format"
+            ;;
+    esac
+    length="${#2}"
+    if [ "$length" -lt 24 ] || [ "$length" -gt 128 ]; then
+        fail "$1 must contain 24 to 128 characters"
+    fi
 }
 
-file_env MYSQL_ROOT_PASSWORD
-file_env MYSQL_PASSWORD
-
-: "${MYSQL_DATABASE:?MYSQL_DATABASE is required}"
-: "${MYSQL_USER:?MYSQL_USER is required}"
-: "${MYSQL_ROOT_PASSWORD:?MYSQL_ROOT_PASSWORD is required}"
-: "${MYSQL_PASSWORD:?MYSQL_PASSWORD is required}"
-
-require_name MYSQL_DATABASE "$MYSQL_DATABASE"
-require_name MYSQL_USER "$MYSQL_USER"
+require_positive_integer() {
+    case "$2" in
+        *[!0-9]*|"") fail "$1 must be a positive integer" ;;
+    esac
+    [ "$2" -gt 0 ] || fail "$1 must be a positive integer"
+}
 
-install -d -m 0755 -o mysql -g mysql /run/mysqld /var/lib/mysql
+pause_after() {
+    stage="$1"
+    [ "${CONTAINER_STACK_PAUSE_AFTER:-}" = "$stage" ] || return 0
+    ready_name="${CONTAINER_STACK_PAUSE_READY_FILE:-ready}"
+    case "$ready_name" in
+        ""|.|..|*/*) fail "invalid pause ready filename" ;;
+    esac
+    install -d -m 0700 /run/container-stack-test
+    ready="/run/container-stack-test/${ready_name}"
+    (umask 077; printf '%s\n' "$stage" >"$ready")
+    while :; do
+        sleep 3600
+    done
+}
 
-if [ ! -d /var/lib/mysql/mysql ]; then
-    mariadb-install-db --user=mysql --datadir=/var/lib/mysql --skip-test-db >/dev/null
+cleanup() {
+    if [ -n "$temporary_pid" ] && kill -0 "$temporary_pid" 2>/dev/null; then
+        kill -TERM "$temporary_pid" 2>/dev/null || true
+        wait "$temporary_pid" 2>/dev/null || true
+    fi
+    temporary_pid=""
+    [ -z "$root_option_file" ] || rm -f -- "$root_option_file"
+    [ -z "$app_option_file" ] || rm -f -- "$app_option_file"
+}
 
-    mariadbd --user=mysql --datadir=/var/lib/mysql --skip-networking \
-        --socket=/run/mysqld/mysqld.sock &
-    pid="$!"
+start_temporary_server() {
+    server_data_dir="$1"
+    rm -f -- "$socket"
+    mariadbd --user=mysql --datadir="$server_data_dir" --skip-networking \
+        --socket="$socket" --pid-file="${run_dir}/bootstrap.pid" &
+    temporary_pid="$!"
 
-    for _ in $(seq 1 60); do
-        if mysqladmin --socket=/run/mysqld/mysqld.sock ping --silent; then
-            break
+    remaining="$wait_retries"
+    while [ "$remaining" -gt 0 ]; do
+        if mysqladmin --socket="$socket" ping --silent >/dev/null 2>&1; then
+            return 0
+        fi
+        if ! kill -0 "$temporary_pid" 2>/dev/null; then
+            wait "$temporary_pid" || true
+            temporary_pid=""
+            fail "temporary MariaDB server exited during bootstrap"
         fi
-        sleep 1
+        remaining=$((remaining - 1))
+        [ "$remaining" -gt 0 ] || fail "timed out waiting for temporary MariaDB server"
+        sleep "$wait_delay"
     done
+}
+
+stop_temporary_server() {
+    mysqladmin --defaults-extra-file="$root_option_file" shutdown
+    wait "$temporary_pid"
+    temporary_pid=""
+}
+
+write_option_file() {
+    target="$1"
+    user="$2"
+    password="$3"
+    (
+        umask 077
+        printf '[client]\nuser=%s\npassword="%s"\nsocket=%s\n' \
+            "$user" "$password" "$socket" >"$target"
+    )
+}
+
+verify_database() {
+    mariadb --defaults-extra-file="$root_option_file" --batch --skip-column-names \
+        --execute="SELECT COUNT(*) FROM mysql.user WHERE User='${MYSQL_USER}' AND Host='%'" \
+        | grep -qx 1
+    mariadb --defaults-extra-file="$app_option_file" "$MYSQL_DATABASE" \
+        --batch --skip-column-names --execute='SELECT 1' | grep -qx 1
+}
+
+runtime() {
+    [ -d "${data_dir}/mysql" ] || fail "MariaDB data is not bootstrapped; run tools/start_stack.py"
+    [ -f "$marker" ] && [ ! -L "$marker" ] \
+        || fail "MariaDB completion marker is missing; rerun bootstrap"
+    exec "$@"
+}
 
-    root_password="$(sql_escape "$MYSQL_ROOT_PASSWORD")"
-    user_password="$(sql_escape "$MYSQL_PASSWORD")"
+bootstrap() {
+    : "${MYSQL_DATABASE:?MYSQL_DATABASE is required}"
+    : "${MYSQL_USER:?MYSQL_USER is required}"
+    require_name MYSQL_DATABASE "$MYSQL_DATABASE"
+    require_name MYSQL_USER "$MYSQL_USER"
+    require_positive_integer MARIADB_INIT_WAIT_RETRIES "$wait_retries"
+    require_positive_integer MARIADB_INIT_WAIT_DELAY "$wait_delay"
 
-    mariadb --socket=/run/mysqld/mysqld.sock <<SQL
+    IFS= read -r root_password || fail "missing root password on standard input"
+    IFS= read -r app_password || fail "missing application password on standard input"
+    if IFS= read -r _unexpected; then
+        fail "unexpected extra bootstrap input"
+    fi
+    require_password MYSQL_ROOT_PASSWORD "$root_password"
+    require_password MYSQL_PASSWORD "$app_password"
+
+    install -d -m 0755 -o mysql -g mysql "$run_dir" "$volume_dir"
+    root_option_file="$(mktemp "${run_dir}/root-client.XXXXXX")"
+    app_option_file="$(mktemp "${run_dir}/app-client.XXXXXX")"
+    chmod 0600 "$root_option_file" "$app_option_file"
+    write_option_file "$root_option_file" root "$root_password"
+    write_option_file "$app_option_file" "$MYSQL_USER" "$app_password"
+
+    if [ -e "$data_dir" ]; then
+        [ -d "${data_dir}/mysql" ] || fail "MariaDB data path is not a valid data directory"
+        [ -f "$marker" ] && [ ! -L "$marker" ] \
+            || fail "MariaDB data exists without a completion marker"
+        start_temporary_server "$data_dir"
+        pause_after temporary-server
+        verify_database
+        stop_temporary_server
+        return 0
+    fi
+
+    rm -rf -- "$staging_dir"
+    install -d -m 0700 -o mysql -g mysql "$staging_dir"
+    mariadb-install-db --user=mysql --datadir="$staging_dir" --skip-test-db >/dev/null
+    pause_after system-tables
+
+    start_temporary_server "$staging_dir"
+    pause_after temporary-server
+    mariadb --socket="$socket" -uroot <<SQL
+SET SESSION sql_mode='NO_BACKSLASH_ESCAPES';
 ALTER USER 'root'@'localhost' IDENTIFIED BY '${root_password}';
 DELETE FROM mysql.user WHERE User='';
 DELETE FROM mysql.user WHERE User='root' AND Host NOT IN ('localhost');
 DROP DATABASE IF EXISTS test;
 CREATE DATABASE IF NOT EXISTS \`${MYSQL_DATABASE}\` CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
-CREATE USER IF NOT EXISTS '${MYSQL_USER}'@'%' IDENTIFIED BY '${user_password}';
+CREATE USER IF NOT EXISTS '${MYSQL_USER}'@'%' IDENTIFIED BY '${app_password}';
+ALTER USER '${MYSQL_USER}'@'%' IDENTIFIED BY '${app_password}';
 GRANT ALL PRIVILEGES ON \`${MYSQL_DATABASE}\`.* TO '${MYSQL_USER}'@'%';
 FLUSH PRIVILEGES;
 SQL
+    pause_after database-state
+    verify_database
+    stop_temporary_server
+
+    staging_marker="${staging_dir}/.container-stack-initialized"
+    (umask 077; : >"$staging_marker")
+    chown mysql:mysql "$staging_marker"
+    sync -f "$staging_marker"
+    sync -f "$staging_dir"
+    pause_after database-marker
+    mv -- "$staging_dir" "$data_dir"
+    sync -f "$volume_dir"
+    pause_after database-publish
+}
+
+trap cleanup EXIT
+trap 'exit 129' HUP
+trap 'exit 130' INT
+trap 'exit 143' TERM
 
-    mysqladmin --socket=/run/mysqld/mysqld.sock -uroot -p"$MYSQL_ROOT_PASSWORD" shutdown
-    wait "$pid"
+if [ "${1:-}" = "bootstrap" ]; then
+    shift
+    [ "$#" -eq 0 ] || fail "bootstrap does not accept arguments"
+    bootstrap
+    exit 0
 fi
 
-exec "$@"
+trap - EXIT HUP INT TERM
+runtime "$@"
diff --git a/srcs/requirements/wordpress/Dockerfile b/srcs/requirements/wordpress/Dockerfile
index 722104b..0bdcde8 100644
--- a/srcs/requirements/wordpress/Dockerfile
+++ b/srcs/requirements/wordpress/Dockerfile
@@ -21,8 +21,8 @@ RUN apt-get update \
 RUN curl -fsSL https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar \
         -o /usr/local/bin/wp \
     && chmod +x /usr/local/bin/wp \
-    && mkdir -p /run/php /var/www/html \
-    && chown -R www-data:www-data /run/php /var/www/html
+    && mkdir -p /run/php /var/www/html /var/www/config \
+    && chown -R www-data:www-data /run/php /var/www/html /var/www/config
 
 WORKDIR /var/www/html
 
diff --git a/srcs/requirements/wordpress/conf/www.conf b/srcs/requirements/wordpress/conf/www.conf
index 5a357f8..8f0e998 100644
--- a/srcs/requirements/wordpress/conf/www.conf
+++ b/srcs/requirements/wordpress/conf/www.conf
@@ -12,7 +12,7 @@ pm.start_servers = 3
 pm.min_spare_servers = 2
 pm.max_spare_servers = 5
 
-clear_env = no
+clear_env = yes
 catch_workers_output = yes
 
 ping.path = /ping
diff --git a/srcs/requirements/wordpress/tools/docker-entrypoint.sh b/srcs/requirements/wordpress/tools/docker-entrypoint.sh
index fd01d68..5c6d7b0 100755
--- a/srcs/requirements/wordpress/tools/docker-entrypoint.sh
+++ b/srcs/requirements/wordpress/tools/docker-entrypoint.sh
@@ -1,89 +1,459 @@
 #!/bin/sh
 set -eu
 
-file_env() {
-    var="$1"
-    file_var="${var}_FILE"
-    value="${2:-}"
-    eval current="\${$var:-}"
-    eval file_path="\${$file_var:-}"
-
-    if [ -n "$current" ] && [ -n "$file_path" ]; then
-        echo "$var and $file_var are mutually exclusive" >&2
-        exit 1
+wordpress_dir="${WORDPRESS_DATA_DIR:-/var/www/html}"
+config_dir="${WORDPRESS_CONFIG_DIR:-/var/www/config}"
+config_path="${config_dir}/wp-config.php"
+config_link="${wordpress_dir}/wp-config.php"
+marker="${wordpress_dir}/.container-stack-initialized"
+wait_retries="${WORDPRESS_DB_WAIT_RETRIES:-60}"
+wait_delay="${WORDPRESS_DB_WAIT_DELAY:-2}"
+db_option_file=""
+
+fail() {
+    echo "$*" >&2
+    exit 1
+}
+
+require_name() {
+    case "$2" in
+        *[!A-Za-z0-9_]*|"") fail "$1 has an invalid format" ;;
+    esac
+}
+
+require_password() {
+    case "$2" in
+        *[!A-Za-z0-9_.~!@#%^+=,-]*|"") fail "$1 has an invalid format" ;;
+    esac
+    length="${#2}"
+    if [ "$length" -lt 24 ] || [ "$length" -gt 128 ]; then
+        fail "$1 must contain 24 to 128 characters"
+    fi
+}
+
+require_positive_integer() {
+    case "$2" in
+        *[!0-9]*|"") fail "$1 must be a positive integer" ;;
+    esac
+    [ "$2" -gt 0 ] || fail "$1 must be a positive integer"
+}
+
+require_runtime_value() {
+    case "$2" in
+        ""|*[!A-Za-z0-9._:/-]*) fail "$1 has an invalid format" ;;
+    esac
+}
+
+pause_after() {
+    stage="$1"
+    [ "${CONTAINER_STACK_PAUSE_AFTER:-}" = "$stage" ] || return 0
+    ready_name="${CONTAINER_STACK_PAUSE_READY_FILE:-ready}"
+    case "$ready_name" in
+        ""|.|..|*/*) fail "invalid pause ready filename" ;;
+    esac
+    install -d -m 0700 /run/container-stack-test
+    ready="/run/container-stack-test/${ready_name}"
+    (umask 077; printf '%s\n' "$stage" >"$ready")
+    while :; do
+        sleep 3600
+    done
+}
+
+cleanup() {
+    [ -z "$db_option_file" ] || rm -f -- "$db_option_file"
+}
+
+wait_for_database() {
+    remaining="$wait_retries"
+    while [ "$remaining" -gt 0 ]; do
+        if mariadb --defaults-extra-file="$db_option_file" "$MYSQL_DATABASE" \
+            --batch --skip-column-names --execute='SELECT 1' >/dev/null 2>&1; then
+            return 0
+        fi
+        remaining=$((remaining - 1))
+        [ "$remaining" -gt 0 ] || fail "timed out waiting for authenticated MariaDB access"
+        sleep "$wait_delay"
+    done
+}
+
+install_core_files() {
+    if find "$wordpress_dir" -path "${wordpress_dir}/wp-content" -prune \
+        -o -path "$config_link" -prune \
+        -o -type l -print | grep -q .; then
+        fail "WordPress core path contains a symbolic link"
     fi
-    if [ -n "$file_path" ]; then
-        value="$(cat "$file_path")"
-    elif [ -n "$current" ]; then
-        value="$current"
+    if [ ! -f "${wordpress_dir}/wp-includes/version.php" ]; then
+        wp core download --allow-root --path="$wordpress_dir"
     fi
-    export "$var=$value"
-    unset "$file_var"
-}
-
-file_env WORDPRESS_DB_PASSWORD
-file_env WORDPRESS_ADMIN_PASSWORD
-file_env WORDPRESS_USER_PASSWORD
-
-: "${WORDPRESS_DB_HOST:=mariadb}"
-: "${MYSQL_DATABASE:?MYSQL_DATABASE is required}"
-: "${MYSQL_USER:?MYSQL_USER is required}"
-: "${WORDPRESS_DB_PASSWORD:?WORDPRESS_DB_PASSWORD is required}"
-: "${DOMAIN_NAME:?DOMAIN_NAME is required}"
-: "${WORDPRESS_TITLE:?WORDPRESS_TITLE is required}"
-: "${WORDPRESS_ADMIN_USER:?WORDPRESS_ADMIN_USER is required}"
-: "${WORDPRESS_ADMIN_PASSWORD:?WORDPRESS_ADMIN_PASSWORD is required}"
-: "${WORDPRESS_ADMIN_EMAIL:?WORDPRESS_ADMIN_EMAIL is required}"
-: "${WORDPRESS_USER:?WORDPRESS_USER is required}"
-: "${WORDPRESS_USER_PASSWORD:?WORDPRESS_USER_PASSWORD is required}"
-: "${WORDPRESS_USER_EMAIL:?WORDPRESS_USER_EMAIL is required}"
-
-install -d -m 0755 -o www-data -g www-data /run/php /var/www/html
-
-for _ in $(seq 1 60); do
-    if mysqladmin ping -h"$WORDPRESS_DB_HOST" -u"$MYSQL_USER" -p"$WORDPRESS_DB_PASSWORD" --silent; then
-        break
+    [ -f "${wordpress_dir}/wp-includes/version.php" ] \
+        || fail "WordPress core files are incomplete"
+}
+
+install_content_files() {
+    :
+}
+
+publish_config_link() {
+    temporary="${wordpress_dir}/.wp-config-link.$$"
+    rm -f -- "$temporary"
+    ln -s "$config_path" "$temporary"
+    mv -f -- "$temporary" "$config_link"
+    sync -f "$wordpress_dir"
+}
+
+prepare_config_location() {
+    install -d -m 0700 -o www-data -g www-data "$config_dir"
+    if [ -L "$config_link" ]; then
+        [ "$(readlink "$config_link")" = "$config_path" ] \
+            || fail "WordPress configuration link has an unexpected target"
+        if [ -e "$config_path" ]; then
+            [ -f "$config_path" ] && [ ! -L "$config_path" ] \
+                || fail "WordPress configuration is not a regular file"
+        fi
+        return 0
     fi
-    sleep 2
-done
 
-if [ ! -f /var/www/html/wp-includes/version.php ]; then
-    wp core download --allow-root --path=/var/www/html
-fi
+    if [ -e "$config_link" ]; then
+        [ -f "$config_link" ] \
+            || fail "WordPress configuration path is not a regular file"
+        if [ -e "$config_path" ]; then
+            [ -f "$config_path" ] && [ ! -L "$config_path" ] \
+                || fail "WordPress configuration is not a regular file"
+            cmp -s "$config_link" "$config_path" \
+                || fail "WordPress configuration locations disagree"
+        else
+            temporary="${config_dir}/.wp-config.migrate.$$"
+            rm -f -- "$temporary"
+            cp -p -- "$config_link" "$temporary"
+            chmod 0600 "$temporary"
+            chown www-data:www-data "$temporary"
+            sync -f "$temporary"
+            mv -- "$temporary" "$config_path"
+            sync -f "$config_dir"
+        fi
+        publish_config_link
+        return 0
+    fi
 
-if [ ! -f /var/www/html/wp-config.php ]; then
-    wp config create --allow-root \
-        --path=/var/www/html \
-        --dbname="$MYSQL_DATABASE" \
-        --dbuser="$MYSQL_USER" \
-        --dbpass="$WORDPRESS_DB_PASSWORD" \
-        --dbhost="$WORDPRESS_DB_HOST" \
-        --skip-check
-
-    wp config set --allow-root --path=/var/www/html FS_METHOD direct --type=constant
-    wp config set --allow-root --path=/var/www/html WP_HOME "https://${DOMAIN_NAME}" --type=constant
-    wp config set --allow-root --path=/var/www/html WP_SITEURL "https://${DOMAIN_NAME}" --type=constant
-fi
+    if [ -e "$config_path" ]; then
+        [ -f "$config_path" ] && [ ! -L "$config_path" ] \
+            || fail "WordPress configuration is not a regular file"
+        publish_config_link
+    fi
+}
 
-if ! wp core is-installed --allow-root --path=/var/www/html >/dev/null 2>&1; then
-    wp core install --allow-root \
-        --path=/var/www/html \
-        --url="https://${DOMAIN_NAME}" \
-        --title="$WORDPRESS_TITLE" \
-        --admin_user="$WORDPRESS_ADMIN_USER" \
-        --admin_password="$WORDPRESS_ADMIN_PASSWORD" \
-        --admin_email="$WORDPRESS_ADMIN_EMAIL" \
-        --skip-email
-fi
+write_wordpress_config() {
+    target="$config_path"
+    temporary="${config_dir}/.wp-config.bootstrap.$$"
+    salts="$(od -An -N128 -tx1 /dev/urandom | tr -d ' \n')"
+    (
+        umask 077
+        {
+            printf '%s\n' '<?php'
+            printf "define('DB_NAME', '%s');\n" "$MYSQL_DATABASE"
+            printf "define('DB_USER', '%s');\n" "$MYSQL_USER"
+            printf "define('DB_PASSWORD', '%s');\n" "$db_password"
+            printf "define('DB_HOST', '%s');\n" "$WORDPRESS_DB_HOST"
+            printf '%s\n' \
+                "define('DB_CHARSET', 'utf8mb4');" \
+                "define('DB_COLLATE', '');" \
+                "define('AUTH_KEY', '${salts}01');" \
+                "define('SECURE_AUTH_KEY', '${salts}02');" \
+                "define('LOGGED_IN_KEY', '${salts}03');" \
+                "define('NONCE_KEY', '${salts}04');" \
+                "define('AUTH_SALT', '${salts}05');" \
+                "define('SECURE_AUTH_SALT', '${salts}06');" \
+                "define('LOGGED_IN_SALT', '${salts}07');" \
+                "define('NONCE_SALT', '${salts}08');" \
+                "define('FS_METHOD', 'direct');"
+            printf "define('WP_HOME', '%s');\n" "$WORDPRESS_URL"
+            printf "define('WP_SITEURL', '%s');\n" "$WORDPRESS_URL"
+            printf '%s\n' \
+                "\$table_prefix = 'wp_';" \
+                "if (!defined('ABSPATH')) { define('ABSPATH', __DIR__ . '/'); }" \
+                "require_once ABSPATH . 'wp-settings.php';"
+        } >"$temporary"
+    )
+    chown www-data:www-data "$temporary"
+    sync -f "$temporary"
+    mv -f -- "$temporary" "$target"
+    sync -f "$config_dir"
+    publish_config_link
+}
 
-if ! wp user get "$WORDPRESS_USER" --allow-root --path=/var/www/html >/dev/null 2>&1; then
-    wp user create "$WORDPRESS_USER" "$WORDPRESS_USER_EMAIL" \
-        --allow-root \
-        --path=/var/www/html \
-        --role=author \
-        --user_pass="$WORDPRESS_USER_PASSWORD"
-fi
+config_value() {
+    name="$1"
+    kind="${2:-constant}"
+    wp config get "$name" --allow-root --path="$wordpress_dir" --type="$kind" 2>/dev/null
+}
+
+validate_wordpress_config() {
+    target="$config_path"
+    [ -L "$config_link" ] || return 1
+    [ "$(readlink "$config_link")" = "$config_path" ] || return 1
+    [ -f "$target" ] && [ ! -L "$target" ] || return 1
+    php -l "$target" >/dev/null 2>&1 || return 1
+    actual_db_name="$(config_value DB_NAME)" || return 1
+    actual_db_user="$(config_value DB_USER)" || return 1
+    actual_db_password="$(config_value DB_PASSWORD)" || return 1
+    actual_db_host="$(config_value DB_HOST)" || return 1
+    actual_table_prefix="$(config_value table_prefix variable)" || return 1
+    [ -n "$actual_table_prefix" ] || return 1
+    [ "$actual_db_name" = "$MYSQL_DATABASE" ] || return 2
+    [ "$actual_db_user" = "$MYSQL_USER" ] || return 2
+    [ "$actual_db_password" = "$db_password" ] || return 2
+    [ "$actual_db_host" = "$WORDPRESS_DB_HOST" ] || return 2
+}
+
+update_config_urls() {
+    updater=/run/container-stack-update-config.php
+    cat >"$updater" <<'PHP'
+<?php
+$path = getenv('CONTAINER_STACK_CONFIG_PATH');
+$url = getenv('CONTAINER_STACK_WORDPRESS_URL');
+$text = file_get_contents($path);
+if ($text === false || is_link($path) || !is_file($path)) {
+    fwrite(STDERR, "WordPress configuration read failed\n");
+    exit(1);
+}
+foreach (['WP_HOME', 'WP_SITEURL'] as $name) {
+    $pattern = "/define\\(\\s*['\"]" . preg_quote($name, '/') . "['\"]\\s*,\\s*.*?\\);/";
+    $replacement = "define('" . $name . "', " . var_export($url, true) . ");";
+    $text = preg_replace($pattern, $replacement, $text, 1, $count);
+    if ($text === null || $count !== 1) {
+        fwrite(STDERR, "WordPress URL setting is missing: " . $name . "\n");
+        exit(1);
+    }
+}
+umask(0077);
+$temporary = tempnam(dirname($path), '.wp-config.url.');
+if ($temporary === false) {
+    fwrite(STDERR, "WordPress configuration temporary file failed\n");
+    exit(1);
+}
+$published = false;
+try {
+    $written = file_put_contents($temporary, $text, LOCK_EX);
+    if ($written !== strlen($text)) {
+        throw new RuntimeException('WordPress configuration write failed');
+    }
+    if (!chmod($temporary, 0600)
+        || !chown($temporary, fileowner($path))
+        || !chgrp($temporary, filegroup($path))) {
+        throw new RuntimeException('WordPress configuration ownership failed');
+    }
+    $handle = fopen($temporary, 'rb');
+    if ($handle === false) {
+        throw new RuntimeException('WordPress configuration reopen failed');
+    }
+    try {
+        if (function_exists('fsync') && !fsync($handle)) {
+            throw new RuntimeException('WordPress configuration fsync failed');
+        }
+    } finally {
+        fclose($handle);
+    }
+    if (!rename($temporary, $path)) {
+        throw new RuntimeException('WordPress configuration publish failed');
+    }
+    $published = true;
+} finally {
+    if (!$published) {
+        @unlink($temporary);
+    }
+}
+PHP
+    chmod 0600 "$updater"
+    if ! CONTAINER_STACK_CONFIG_PATH="$config_path" \
+        CONTAINER_STACK_WORDPRESS_URL="$WORDPRESS_URL" \
+        php "$updater"; then
+        rm -f -- "$updater"
+        fail "WordPress URL configuration update failed"
+    fi
+    rm -f -- "$updater"
+}
+
+converge_wordpress_config() {
+    prepare_config_location
+    config_status=0
+    validate_wordpress_config || config_status="$?"
+    case "$config_status" in
+        0)
+            ;;
+        1)
+            if [ -f "$marker" ]; then
+                fail "completed WordPress configuration is invalid"
+            fi
+            write_wordpress_config
+            validate_wordpress_config \
+                || fail "generated WordPress configuration is invalid"
+            ;;
+        2)
+            fail "WordPress database credentials differ; use the secret rotation command"
+            ;;
+        *)
+            fail "cannot validate WordPress configuration"
+            ;;
+    esac
+    update_config_urls
+    chmod 0600 "$config_path"
+    chown www-data:www-data "$config_path"
+    sync -f "$config_path"
+    sync -f "$config_dir"
+}
+
+install_wordpress() {
+    if wp core is-installed --allow-root --path="$wordpress_dir" >/dev/null 2>&1; then
+        return 0
+    fi
+    command_log="$(mktemp /run/wp-core-install.XXXXXX)"
+    chmod 0600 "$command_log"
+    if ! printf '%s\n' "$admin_password" \
+        | wp core install --allow-root --path="$wordpress_dir" \
+            --url="$WORDPRESS_URL" \
+            --title="$WORDPRESS_TITLE" \
+            --admin_user="$WORDPRESS_ADMIN_USER" \
+            --admin_email="$WORDPRESS_ADMIN_EMAIL" \
+            --prompt=admin_password \
+            --skip-email >"$command_log" 2>&1; then
+        rm -f -- "$command_log"
+        fail "WordPress core installation failed"
+    fi
+    rm -f -- "$command_log"
+}
+
+ensure_author() {
+    if wp user get "$WORDPRESS_USER" --allow-root --path="$wordpress_dir" >/dev/null 2>&1; then
+        return 0
+    fi
+    command_log="$(mktemp /run/wp-user-create.XXXXXX)"
+    chmod 0600 "$command_log"
+    if ! printf '%s\n' "$user_password" \
+        | wp user create "$WORDPRESS_USER" "$WORDPRESS_USER_EMAIL" \
+            --allow-root --path="$wordpress_dir" --role=author \
+            --prompt=user_pass >"$command_log" 2>&1; then
+        rm -f -- "$command_log"
+        fail "WordPress author creation failed"
+    fi
+    rm -f -- "$command_log"
+}
+
+verify_user_password() {
+    login="$1"
+    password="$2"
+    verifier=/run/container-stack-verify-password.php
+    cat >"$verifier" <<'PHP'
+<?php
+$password = rtrim(stream_get_contents(STDIN), "\r\n");
+$login = getenv('CONTAINER_STACK_VERIFY_USER');
+$account = get_user_by('login', $login);
+if (!$account || !wp_check_password($password, $account->user_pass, $account->ID)) {
+    exit(1);
+}
+PHP
+    chmod 0600 "$verifier"
+    if ! printf '%s\n' "$password" \
+        | CONTAINER_STACK_VERIFY_USER="$login" \
+            wp eval-file "$verifier" --allow-root --path="$wordpress_dir" >/dev/null; then
+        rm -f -- "$verifier"
+        fail "WordPress account password verification failed: $login"
+    fi
+    rm -f -- "$verifier"
+}
 
-chown -R www-data:www-data /var/www/html /run/php
+runtime() {
+    [ -f "${wordpress_dir}/wp-includes/version.php" ] \
+        || fail "WordPress core is not bootstrapped; run tools/start_stack.py"
+    [ -L "$config_link" ] \
+        && [ "$(readlink "$config_link")" = "$config_path" ] \
+        && [ -f "$config_path" ] && [ ! -L "$config_path" ] \
+        || fail "WordPress configuration is missing or exposed in the web volume"
+    [ -f "$marker" ] && [ ! -L "$marker" ] \
+        || fail "WordPress completion marker is missing; rerun bootstrap"
+    install -d -m 0755 -o www-data -g www-data /run/php
+    exec "$@"
+}
+
+bootstrap() {
+    : "${WORDPRESS_DB_HOST:=mariadb}"
+    : "${MYSQL_DATABASE:?MYSQL_DATABASE is required}"
+    : "${MYSQL_USER:?MYSQL_USER is required}"
+    : "${DOMAIN_NAME:?DOMAIN_NAME is required}"
+    : "${WORDPRESS_URL:=https://${DOMAIN_NAME}}"
+    : "${WORDPRESS_TITLE:?WORDPRESS_TITLE is required}"
+    : "${WORDPRESS_ADMIN_USER:?WORDPRESS_ADMIN_USER is required}"
+    : "${WORDPRESS_ADMIN_EMAIL:?WORDPRESS_ADMIN_EMAIL is required}"
+    : "${WORDPRESS_USER:?WORDPRESS_USER is required}"
+    : "${WORDPRESS_USER_EMAIL:?WORDPRESS_USER_EMAIL is required}"
+    require_name MYSQL_DATABASE "$MYSQL_DATABASE"
+    require_name MYSQL_USER "$MYSQL_USER"
+    require_name WORDPRESS_ADMIN_USER "$WORDPRESS_ADMIN_USER"
+    require_name WORDPRESS_USER "$WORDPRESS_USER"
+    require_runtime_value WORDPRESS_DB_HOST "$WORDPRESS_DB_HOST"
+    require_runtime_value WORDPRESS_URL "$WORDPRESS_URL"
+    case "$WORDPRESS_URL" in
+        https://*) ;;
+        *) fail "WORDPRESS_URL must use https" ;;
+    esac
+    require_positive_integer WORDPRESS_DB_WAIT_RETRIES "$wait_retries"
+    require_positive_integer WORDPRESS_DB_WAIT_DELAY "$wait_delay"
+
+    IFS= read -r db_password || fail "missing database password on standard input"
+    IFS= read -r admin_password || fail "missing administrator password on standard input"
+    IFS= read -r user_password || fail "missing author password on standard input"
+    if IFS= read -r _unexpected; then
+        fail "unexpected extra bootstrap input"
+    fi
+    require_password WORDPRESS_DB_PASSWORD "$db_password"
+    require_password WORDPRESS_ADMIN_PASSWORD "$admin_password"
+    require_password WORDPRESS_USER_PASSWORD "$user_password"
+
+    install -d -m 0755 -o www-data -g www-data /run/php "$wordpress_dir"
+    install -d -m 0700 -o www-data -g www-data "$config_dir"
+    db_option_file="$(mktemp /run/db-client.XXXXXX)"
+    chmod 0600 "$db_option_file"
+    printf '[client]\nhost=%s\nuser=%s\npassword="%s"\n' \
+        "$WORDPRESS_DB_HOST" "$MYSQL_USER" "$db_password" >"$db_option_file"
+    wait_for_database
+
+    install_core_files
+    install_content_files
+    pause_after core-files
+    converge_wordpress_config
+    pause_after wordpress-config
+    install_wordpress
+    pause_after wordpress-core
+    ensure_author
+    pause_after wordpress-users
+
+    wp option update home "$WORDPRESS_URL" --allow-root --path="$wordpress_dir" >/dev/null
+    wp option update siteurl "$WORDPRESS_URL" --allow-root --path="$wordpress_dir" >/dev/null
+    wp user get "$WORDPRESS_ADMIN_USER" --allow-root --path="$wordpress_dir" >/dev/null
+    wp user get "$WORDPRESS_USER" --allow-root --path="$wordpress_dir" >/dev/null
+    verify_user_password "$WORDPRESS_ADMIN_USER" "$admin_password"
+    verify_user_password "$WORDPRESS_USER" "$user_password"
+
+    marker_tmp="${wordpress_dir}/.container-stack-initialized.tmp.$$"
+    (umask 077; : >"$marker_tmp")
+    chown www-data:www-data "$marker_tmp"
+    sync -f "$marker_tmp"
+    mv -f -- "$marker_tmp" "$marker"
+    sync -f "$wordpress_dir"
+    pause_after wordpress-marker
+    chown -R www-data:www-data "$wordpress_dir" "$config_dir" /run/php
+}
+
+trap cleanup EXIT
+trap 'exit 129' HUP
+trap 'exit 130' INT
+trap 'exit 143' TERM
+
+if [ "${1:-}" = "bootstrap" ]; then
+    shift
+    [ "$#" -eq 0 ] || fail "bootstrap does not accept arguments"
+    bootstrap
+    exit 0
+fi
 
-exec "$@"
+trap - EXIT HUP INT TERM
+runtime "$@"
diff --git a/tests/validate_stack.py b/tests/validate_stack.py
index 42aa70f..856154c 100755
--- a/tests/validate_stack.py
+++ b/tests/validate_stack.py
@@ -65,9 +65,10 @@ def validate_compose() -> None:
             r"\"443:443\"",
             r"condition: service_healthy",
             r"healthcheck:",
-            r"secrets:",
+            r"x-secret-files:",
             r"mariadb_data:",
             r"wordpress_data:",
+            r"wordpress_config:",
         ],
     )
     if re.search(r"(^|\s)-\s*[\"']?80:", text):
@@ -75,15 +76,23 @@ def validate_compose() -> None:
     if "mysqladmin ping -h127.0.0.1 -uroot" in text:
         fail("mariadb healthcheck must not require TCP root login")
     if not re.search(
-        r"mysqladmin\s+--socket=/run/mysqld/mysqld\.sock\s+-uroot\s+-p.+\s+ping\s+--silent",
+        r"test -f /var/lib/mysql-volume/data/\.container-stack-initialized.+test -S /run/mysqld/mysqld\.sock.+kill -0 1",
         text,
     ):
-        fail("mariadb healthcheck must use the local socket as root")
+        fail("mariadb healthcheck must require the completed bootstrap marker")
+    if "/run/secrets" in text or re.search(r"^\s+secrets:", text, re.MULTILINE):
+        fail("runtime services must not mount secret files")
+    if re.search(r"^\s{6}[A-Z0-9_]*PASSWORD(?:_FILE)?:", text, re.MULTILINE):
+        fail("runtime service environments must not contain passwords")
+    if "/var/www/config" in re.search(
+        r"(?ms)^\s+nginx:.*?(?=^\s{2}[a-z])", text
+    ).group(0):
+        fail("nginx must not mount the WordPress configuration volume")
     if not re.search(
-        r"REQUEST_METHOD=GET\s+SCRIPT_NAME=/ping\s+SCRIPT_FILENAME=/ping\s+cgi-fcgi",
+        r"test -f /var/www/html/\.container-stack-initialized.+REQUEST_METHOD=GET\s+SCRIPT_NAME=/ping\s+SCRIPT_FILENAME=/ping\s+cgi-fcgi",
         text,
     ):
-        fail("wordpress healthcheck must call the FPM ping endpoint as a GET request")
+        fail("wordpress healthcheck must require bootstrap completion before FPM ping")
     for image in ("wordpress:", "mariadb:", "nginx:"):
         if re.search(rf"image:\s*{image}", text):
             fail(f"compose must not use the official {image.rstrip(':')} image directly")
@@ -100,7 +109,7 @@ def validate_dockerfiles() -> None:
         "mariadb": [
             r"FROM\s+debian:bookworm-slim|FROM\s+alpine:",
             r"mariadb-server",
-            r"rm -rf /var/lib/mysql/\*",
+            r"rm -rf /var/lib/mysql",
             r"COPY conf/50-server\.cnf",
             r"ENTRYPOINT",
         ],
@@ -135,7 +144,7 @@ def validate_configs() -> None:
     )
     require_text(
         "srcs/requirements/wordpress/conf/www.conf",
-        [r"listen = 0\.0\.0\.0:9000", r"ping\.path = /ping"],
+        [r"listen = 0\.0\.0\.0:9000", r"ping\.path = /ping", r"clear_env = yes"],
     )
 
 
@@ -158,7 +167,18 @@ def validate_env_policy() -> None:
 
 def validate_tools() -> None:
     require_executable("tools/smoke_https.sh")
-    require_text("Makefile", [r"^smoke:", r"tools/smoke_https\.sh"])
+    require_executable("tools/start_stack.py")
+    require_file("tools/stack_runtime.py")
+    require_text(
+        "Makefile",
+        [
+            r"^up:\n\s+python3 tools/start_stack\.py start",
+            r"^start-database:",
+            r"^start-application:",
+            r"^smoke:",
+            r"tools/smoke_https\.sh",
+        ],
+    )
 
 
 def main() -> None:
diff --git a/tools/start_stack.py b/tools/start_stack.py
new file mode 100755
index 0000000..7386e05
--- /dev/null
+++ b/tools/start_stack.py
@@ -0,0 +1,331 @@
+#!/usr/bin/env python3
+"""비밀값을 런타임 컨테이너에 남기지 않고 Compose 스택을 시작합니다."""
+
+from __future__ import annotations
+
+import argparse
+from contextlib import nullcontext
+import json
+from pathlib import Path
+import subprocess
+import sys
+
+from stack_runtime import (
+    ComposeProject,
+    DEFAULT_COMPOSE_FILE,
+    StackRuntimeError,
+    load_secret_values,
+    project_operation_lock,
+    secret_payload,
+)
+
+
+DATABASE_STAGES = {
+    "system-tables",
+    "temporary-server",
+    "database-state",
+    "database-marker",
+    "database-publish",
+}
+APPLICATION_STAGES = {
+    "core-files",
+    "wordpress-config",
+    "wordpress-core",
+    "wordpress-users",
+    "wordpress-marker",
+}
+BOOTSTRAP_LABEL = "com.container-stack.bootstrap"
+BUILD_TIMEOUT_SECONDS = 900
+
+
+def parser() -> argparse.ArgumentParser:
+    result = argparse.ArgumentParser(
+        description="초기화용 컨테이너에만 표준 입력으로 비밀값을 전달해 스택을 시작합니다."
+    )
+    result.add_argument("action", choices=("start", "database", "application"))
+    result.add_argument("--project", required=True)
+    result.add_argument("--env-file", type=Path, required=True)
+    result.add_argument(
+        "--compose-file",
+        type=Path,
+        default=DEFAULT_COMPOSE_FILE,
+    )
+    result.add_argument("--build", action="store_true")
+    result.add_argument("--wait-timeout", type=int, default=300)
+    result.add_argument(
+        "--pause-after",
+        choices=sorted(DATABASE_STAGES | APPLICATION_STAGES),
+        help=argparse.SUPPRESS,
+    )
+    result.add_argument(
+        "--pause-ready-file",
+        type=Path,
+        help=argparse.SUPPRESS,
+    )
+    return result
+
+
+def docker(
+    *arguments: str,
+    capture: bool = False,
+    check: bool = True,
+    timeout: int = 30,
+) -> subprocess.CompletedProcess[bytes]:
+    try:
+        return subprocess.run(
+            ["docker", *arguments],
+            stdout=subprocess.PIPE if capture else None,
+            stderr=subprocess.PIPE if capture else None,
+            check=check,
+            timeout=timeout,
+        )
+    except subprocess.TimeoutExpired as error:
+        raise StackRuntimeError(
+            f"Docker 명령이 {timeout}초 안에 끝나지 않았습니다"
+        ) from error
+
+
+def remove_stale_bootstrap(project: ComposeProject, service: str) -> None:
+    name = f"{project.project}-{service}-bootstrap"
+    inspected = docker(
+        "container",
+        "inspect",
+        name,
+        capture=True,
+        check=False,
+    )
+    if inspected.returncode != 0:
+        return
+    try:
+        entries = json.loads(inspected.stdout)
+        labels = entries[0]["Config"]["Labels"]
+    except (IndexError, KeyError, TypeError, json.JSONDecodeError) as error:
+        raise StackRuntimeError(
+            f"초기화 컨테이너의 소유권을 확인할 수 없습니다: {name}"
+        ) from error
+    if (
+        labels.get("com.docker.compose.project") != project.project
+        or labels.get(BOOTSTRAP_LABEL) != service
+    ):
+        raise StackRuntimeError(
+            f"다른 컨테이너가 초기화 이름을 사용 중입니다: {name}"
+        )
+    docker("container", "rm", "--force", name)
+
+
+def pause_arguments(
+    service: str,
+    stage: str | None,
+    ready_file: Path | None,
+) -> list[str]:
+    if stage is None:
+        if ready_file is not None:
+            raise StackRuntimeError(
+                "--pause-ready-file은 --pause-after와 함께 사용해야 합니다"
+            )
+        return []
+    service_stages = DATABASE_STAGES if service == "mariadb" else APPLICATION_STAGES
+    if stage not in service_stages:
+        return []
+    if ready_file is None:
+        raise StackRuntimeError("--pause-after에는 --pause-ready-file이 필요합니다")
+    path = ready_file.expanduser().resolve()
+    if path.exists():
+        raise StackRuntimeError(f"일시정지 준비 파일이 이미 존재합니다: {path}")
+    if not path.parent.is_dir():
+        raise StackRuntimeError(
+            f"일시정지 준비 파일 디렉터리가 없습니다: {path.parent}"
+        )
+    return [
+        "--volume",
+        f"{path.parent}:/run/container-stack-test",
+        "--env",
+        f"CONTAINER_STACK_PAUSE_AFTER={stage}",
+        "--env",
+        f"CONTAINER_STACK_PAUSE_READY_FILE={path.name}",
+    ]
+
+
+def run_bootstrap(
+    project: ComposeProject,
+    service: str,
+    payload: bytes,
+    *,
+    pause_after_stage: str | None,
+    pause_ready_file: Path | None,
+) -> None:
+    remove_stale_bootstrap(project, service)
+    name = f"{project.project}-{service}-bootstrap"
+    extra = pause_arguments(
+        service,
+        pause_after_stage,
+        pause_ready_file,
+    )
+    project.run(
+        "run",
+        "--rm",
+        "--no-deps",
+        "--no-TTY",
+        "--name",
+        name,
+        "--label",
+        f"{BOOTSTRAP_LABEL}={service}",
+        *extra,
+        service,
+        "bootstrap",
+        input_data=payload,
+        timeout=project.timeout,
+    )
+
+
+def wait_for_services(project: ComposeProject, *services: str) -> None:
+    project.run(
+        "up",
+        "--detach",
+        "--wait",
+        "--wait-timeout",
+        str(project.timeout),
+        *services,
+        timeout=project.timeout + 30,
+    )
+
+
+def start_database(
+    project: ComposeProject,
+    secrets: dict[str, str],
+    *,
+    build: bool,
+    pause_after_stage: str | None,
+    pause_ready_file: Path | None,
+) -> None:
+    if build:
+        project.run(
+            "build",
+            "mariadb",
+            timeout=BUILD_TIMEOUT_SECONDS,
+        )
+    if "mariadb" not in project.running_services():
+        run_bootstrap(
+            project,
+            "mariadb",
+            secret_payload(
+                secrets["db_root_password"],
+                secrets["db_password"],
+            ),
+            pause_after_stage=pause_after_stage,
+            pause_ready_file=pause_ready_file,
+        )
+    wait_for_services(project, "mariadb")
+
+
+def start_application(
+    project: ComposeProject,
+    secrets: dict[str, str],
+    *,
+    build: bool,
+    pause_after_stage: str | None,
+    pause_ready_file: Path | None,
+) -> None:
+    if "mariadb" not in project.running_services():
+        raise StackRuntimeError(
+            "MariaDB가 실행 중이 아닙니다. database 단계부터 실행하십시오"
+        )
+    if build:
+        project.run(
+            "build",
+            "wordpress",
+            "nginx",
+            timeout=BUILD_TIMEOUT_SECONDS,
+        )
+    project.run("stop", "nginx", "wordpress")
+    run_bootstrap(
+        project,
+        "wordpress",
+        secret_payload(
+            secrets["db_password"],
+            secrets["wp_admin_password"],
+            secrets["wp_user_password"],
+        ),
+        pause_after_stage=pause_after_stage,
+        pause_ready_file=pause_ready_file,
+    )
+    wait_for_services(project, "wordpress", "nginx")
+
+
+def run_action(
+    project: ComposeProject,
+    action: str,
+    *,
+    secrets: dict[str, str] | None = None,
+    build: bool = False,
+    pause_after_stage: str | None = None,
+    pause_ready_file: Path | None = None,
+    acquire_lock: bool = True,
+) -> None:
+    if action not in {"start", "database", "application"}:
+        raise StackRuntimeError(f"알 수 없는 시작 단계입니다: {action}")
+    if (
+        pause_after_stage in DATABASE_STAGES
+        and action == "application"
+    ) or (
+        pause_after_stage in APPLICATION_STAGES
+        and action == "database"
+    ):
+        raise StackRuntimeError(
+            "요청한 일시정지 단계가 선택한 시작 단계에 속하지 않습니다"
+        )
+    lock = project_operation_lock(project.project) if acquire_lock else nullcontext()
+    with lock:
+        resolved_secrets = (
+            secrets if secrets is not None else load_secret_values(project)
+        )
+        if action in {"start", "database"}:
+            start_database(
+                project,
+                resolved_secrets,
+                build=build,
+                pause_after_stage=pause_after_stage,
+                pause_ready_file=pause_ready_file,
+            )
+        if action in {"start", "application"}:
+            start_application(
+                project,
+                resolved_secrets,
+                build=build,
+                pause_after_stage=pause_after_stage,
+                pause_ready_file=pause_ready_file,
+            )
+
+
+def execute(arguments: argparse.Namespace) -> None:
+    project = ComposeProject(
+        arguments.project,
+        arguments.env_file,
+        arguments.compose_file,
+        timeout=arguments.wait_timeout,
+    )
+    run_action(
+        project,
+        arguments.action,
+        build=arguments.build,
+        pause_after_stage=arguments.pause_after,
+        pause_ready_file=arguments.pause_ready_file,
+    )
+
+
+def main() -> int:
+    arguments = parser().parse_args()
+    try:
+        execute(arguments)
+    except (
+        StackRuntimeError,
+        OSError,
+        subprocess.CalledProcessError,
+    ) as error:
+        print(f"start-stack: {error}", file=sys.stderr)
+        return 1
+    return 0
+
+
+if __name__ == "__main__":
+    raise SystemExit(main())


