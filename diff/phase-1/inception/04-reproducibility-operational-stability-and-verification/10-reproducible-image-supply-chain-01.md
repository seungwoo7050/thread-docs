# 재현 가능한 이미지 공급망

## `build(docker): 임시 파일을 빌드 컨텍스트에서 제외`

diff --git a/srcs/requirements/mariadb/.dockerignore b/srcs/requirements/mariadb/.dockerignore
new file mode 100644
index 0000000..4aa3157
--- /dev/null
+++ b/srcs/requirements/mariadb/.dockerignore
@@ -0,0 +1,3 @@
+.git
+*.log
+*.pid
diff --git a/srcs/requirements/nginx/.dockerignore b/srcs/requirements/nginx/.dockerignore
new file mode 100644
index 0000000..4aa3157
--- /dev/null
+++ b/srcs/requirements/nginx/.dockerignore
@@ -0,0 +1,3 @@
+.git
+*.log
+*.pid
diff --git a/srcs/requirements/wordpress/.dockerignore b/srcs/requirements/wordpress/.dockerignore
new file mode 100644
index 0000000..4aa3157
--- /dev/null
+++ b/srcs/requirements/wordpress/.dockerignore
@@ -0,0 +1,3 @@
+.git
+*.log
+*.pid


## `test(docker): 서비스별 빌드 필터 검사`

diff --git a/tests/validate_stack.py b/tests/validate_stack.py
index 5e5b38c..42aa70f 100755
--- a/tests/validate_stack.py
+++ b/tests/validate_stack.py
@@ -112,6 +112,7 @@ def validate_dockerfiles() -> None:
         ],
     }
     for service, patterns in services.items():
+        require_file(f"srcs/requirements/{service}/.dockerignore")
         require_text(f"srcs/requirements/{service}/Dockerfile", patterns)
         require_executable(f"srcs/requirements/{service}/tools/docker-entrypoint.sh")
 


## `build(images): Debian 이미지와 패키지 입력 고정`

diff --git a/srcs/requirements/mariadb/Dockerfile b/srcs/requirements/mariadb/Dockerfile
index 07ad95e..1823381 100644
--- a/srcs/requirements/mariadb/Dockerfile
+++ b/srcs/requirements/mariadb/Dockerfile
@@ -1,6 +1,13 @@
-FROM debian:bookworm-slim
+FROM debian:bookworm-20241202-slim@sha256:1537a6a1cbc4b4fd401da800ee9480207e7dc1f23560c21259f681db56768f63
 
-RUN apt-get update \
+RUN printf '%s\n' \
+        'deb [check-valid-until=no] http://snapshot.debian.org/archive/debian/20241214T000000Z bookworm main' \
+        'deb [check-valid-until=no] http://snapshot.debian.org/archive/debian/20241214T000000Z bookworm-updates main' \
+        'deb [check-valid-until=no] http://snapshot.debian.org/archive/debian-security/20241214T000000Z bookworm-security main' \
+        > /etc/apt/sources.list \
+    && rm -f /etc/apt/sources.list.d/debian.sources
+
+RUN apt-get -o Acquire::Check-Valid-Until=false update \
     && apt-get install -y --no-install-recommends \
         ca-certificates \
         gosu \
diff --git a/srcs/requirements/nginx/Dockerfile b/srcs/requirements/nginx/Dockerfile
index 0c0a91a..ab18499 100644
--- a/srcs/requirements/nginx/Dockerfile
+++ b/srcs/requirements/nginx/Dockerfile
@@ -1,6 +1,13 @@
-FROM debian:bookworm-slim
+FROM debian:bookworm-20241202-slim@sha256:1537a6a1cbc4b4fd401da800ee9480207e7dc1f23560c21259f681db56768f63
 
-RUN apt-get update \
+RUN printf '%s\n' \
+        'deb [check-valid-until=no] http://snapshot.debian.org/archive/debian/20241214T000000Z bookworm main' \
+        'deb [check-valid-until=no] http://snapshot.debian.org/archive/debian/20241214T000000Z bookworm-updates main' \
+        'deb [check-valid-until=no] http://snapshot.debian.org/archive/debian-security/20241214T000000Z bookworm-security main' \
+        > /etc/apt/sources.list \
+    && rm -f /etc/apt/sources.list.d/debian.sources
+
+RUN apt-get -o Acquire::Check-Valid-Until=false update \
     && apt-get install -y --no-install-recommends \
         ca-certificates \
         curl \
diff --git a/srcs/requirements/wordpress/Dockerfile b/srcs/requirements/wordpress/Dockerfile
index 0bdcde8..9827316 100644
--- a/srcs/requirements/wordpress/Dockerfile
+++ b/srcs/requirements/wordpress/Dockerfile
@@ -1,6 +1,13 @@
-FROM debian:bookworm-slim
+FROM debian:bookworm-20241202-slim@sha256:1537a6a1cbc4b4fd401da800ee9480207e7dc1f23560c21259f681db56768f63
 
-RUN apt-get update \
+RUN printf '%s\n' \
+        'deb [check-valid-until=no] http://snapshot.debian.org/archive/debian/20241214T000000Z bookworm main' \
+        'deb [check-valid-until=no] http://snapshot.debian.org/archive/debian/20241214T000000Z bookworm-updates main' \
+        'deb [check-valid-until=no] http://snapshot.debian.org/archive/debian-security/20241214T000000Z bookworm-security main' \
+        > /etc/apt/sources.list \
+    && rm -f /etc/apt/sources.list.d/debian.sources
+
+RUN apt-get -o Acquire::Check-Valid-Until=false update \
     && apt-get install -y --no-install-recommends \
         ca-certificates \
         curl \
@@ -15,7 +22,6 @@ RUN apt-get update \
         php8.2-mysql \
         php8.2-xml \
         php8.2-zip \
-        unzip \
     && rm -rf /var/lib/apt/lists/*
 
 RUN curl -fsSL https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar \
diff --git a/tests/validate_stack.py b/tests/validate_stack.py
index 08c1bea..985d747 100755
--- a/tests/validate_stack.py
+++ b/tests/validate_stack.py
@@ -102,20 +102,20 @@ def validate_compose() -> None:
 def validate_dockerfiles() -> None:
     services = {
         "nginx": [
-            r"FROM\s+debian:bookworm-slim|FROM\s+alpine:",
+            r"FROM\s+debian:bookworm(?:-\d{8})?-slim|FROM\s+alpine:",
             r"apt-get install|apk add",
             r"COPY conf/nginx\.conf",
             r"EXPOSE 443",
         ],
         "mariadb": [
-            r"FROM\s+debian:bookworm-slim|FROM\s+alpine:",
+            r"FROM\s+debian:bookworm(?:-\d{8})?-slim|FROM\s+alpine:",
             r"mariadb-server",
             r"rm -rf /var/lib/mysql",
             r"COPY conf/50-server\.cnf",
             r"ENTRYPOINT",
         ],
         "wordpress": [
-            r"FROM\s+debian:bookworm-slim|FROM\s+alpine:",
+            r"FROM\s+debian:bookworm(?:-\d{8})?-slim|FROM\s+alpine:",
             r"php8\.2-fpm|php-fpm",
             r"wp-cli\.phar",
             r"EXPOSE 9000",


## `build(wordpress): WordPress 산출물을 고정해 게시`

diff --git a/srcs/requirements/wordpress/Dockerfile b/srcs/requirements/wordpress/Dockerfile
index 9827316..28a8f31 100644
--- a/srcs/requirements/wordpress/Dockerfile
+++ b/srcs/requirements/wordpress/Dockerfile
@@ -24,11 +24,28 @@ RUN apt-get -o Acquire::Check-Valid-Until=false update \
         php8.2-zip \
     && rm -rf /var/lib/apt/lists/*
 
-RUN curl -fsSL https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar \
+ENV WP_CLI_VERSION=2.11.0 \
+    WORDPRESS_VERSION=6.7.1
+
+RUN curl -fsSL \
+        "https://github.com/wp-cli/wp-cli/releases/download/v${WP_CLI_VERSION}/wp-cli-${WP_CLI_VERSION}.phar" \
         -o /usr/local/bin/wp \
-    && chmod +x /usr/local/bin/wp \
-    && mkdir -p /run/php /var/www/html /var/www/config \
-    && chown -R www-data:www-data /run/php /var/www/html /var/www/config
+    && echo 'a39021ac809530ea607580dbf93afbc46ba02f86b6cffd03de4b126ca53079f6  /usr/local/bin/wp' \
+        | sha256sum -c - \
+    && chmod 0755 /usr/local/bin/wp \
+    && curl -fsSL "https://wordpress.org/wordpress-${WORDPRESS_VERSION}.tar.gz" \
+        -o /tmp/wordpress.tar.gz \
+    && echo '33529cd638c845007e8e0d26c91d60c9c16b822c849c8deead03d0c851a26deb  /tmp/wordpress.tar.gz' \
+        | sha256sum -c - \
+    && mkdir -p /usr/src/wordpress /run/php /var/www/html /var/www/config \
+    && tar -xzf /tmp/wordpress.tar.gz --strip-components=1 -C /usr/src/wordpress \
+    && rm /tmp/wordpress.tar.gz \
+    && cd /usr/src/wordpress \
+    && find . -type f ! -path './wp-content/*' -print0 \
+        | sort -z \
+        | xargs -0 sha256sum > /usr/src/wordpress-core.sha256 \
+    && chown -R www-data:www-data \
+        /usr/src/wordpress /run/php /var/www/html /var/www/config
 
 WORKDIR /var/www/html
 
diff --git a/srcs/requirements/wordpress/tools/docker-entrypoint.sh b/srcs/requirements/wordpress/tools/docker-entrypoint.sh
index c415b1e..4776cb9 100755
--- a/srcs/requirements/wordpress/tools/docker-entrypoint.sh
+++ b/srcs/requirements/wordpress/tools/docker-entrypoint.sh
@@ -82,15 +82,78 @@ install_core_files() {
         -o -type l -print | grep -q .; then
         fail "WordPress core path contains a symbolic link"
     fi
-    if [ ! -f "${wordpress_dir}/wp-includes/version.php" ]; then
-        wp core download --allow-root --path="$wordpress_dir"
-    fi
+    while IFS= read -r manifest_line; do
+        digest="${manifest_line%% *}"
+        relative="${manifest_line#*  ./}"
+        case "$relative" in
+            ""|/*|../*|*/../*|*/..) fail "invalid WordPress core manifest path" ;;
+        esac
+        source="/usr/src/wordpress/${relative}"
+        target="${wordpress_dir}/${relative}"
+        parent="${target%/*}"
+        install -d -m 0755 -o www-data -g www-data "$parent"
+        if [ -L "$target" ]; then
+            fail "WordPress core target is a symbolic link: $relative"
+        fi
+        if [ -f "$target" ] \
+            && printf '%s  %s\n' "$digest" "$target" | sha256sum -c - >/dev/null 2>&1; then
+            continue
+        fi
+        if [ -e "$target" ] && [ ! -f "$target" ]; then
+            fail "WordPress core target is not a regular file: $relative"
+        fi
+        base="${target##*/}"
+        temporary="${parent}/.${base}.bootstrap.$$"
+        rm -f -- "$temporary"
+        cp -p -- "$source" "$temporary"
+        chown www-data:www-data "$temporary"
+        sync -f "$temporary"
+        mv -f -- "$temporary" "$target"
+        sync -f "$parent"
+    done < /usr/src/wordpress-core.sha256
+    (
+        cd "$wordpress_dir"
+        sha256sum -c /usr/src/wordpress-core.sha256 >/dev/null
+    ) || fail "WordPress core checksum verification failed"
     [ -f "${wordpress_dir}/wp-includes/version.php" ] \
         || fail "WordPress core files are incomplete"
 }
 
 install_content_files() {
-    :
+    source_root=/usr/src/wordpress/wp-content
+    target_root="${wordpress_dir}/wp-content"
+    if [ -L "$target_root" ]; then
+        fail "WordPress content directory must not be a symbolic link"
+    fi
+    install -d -m 0755 -o www-data -g www-data "$target_root"
+    (
+        cd "$source_root"
+        find . -mindepth 1 -type d -print | sort
+    ) | while IFS= read -r relative; do
+        target="${target_root}/${relative#./}"
+        if [ -L "$target" ] || { [ -e "$target" ] && [ ! -d "$target" ]; }; then
+            fail "WordPress content path is not a regular directory: $relative"
+        fi
+        install -d -m 0755 -o www-data -g www-data "$target"
+    done
+    (
+        cd "$source_root"
+        find . -type f -print | sort
+    ) | while IFS= read -r relative; do
+        source="${source_root}/${relative#./}"
+        target="${target_root}/${relative#./}"
+        if [ -e "$target" ] || [ -L "$target" ]; then
+            continue
+        fi
+        parent="${target%/*}"
+        base="${target##*/}"
+        temporary="${parent}/.${base}.bootstrap.$$"
+        cp -p -- "$source" "$temporary"
+        chown www-data:www-data "$temporary"
+        sync -f "$temporary"
+        mv -- "$temporary" "$target"
+        sync -f "$parent"
+    done
 }
 
 publish_config_link() {
@@ -169,6 +232,8 @@ write_wordpress_config() {
             printf "define('WP_HOME', '%s');\n" "$WORDPRESS_URL"
             printf "define('WP_SITEURL', '%s');\n" "$WORDPRESS_URL"
             printf '%s\n' \
+                "define('WP_AUTO_UPDATE_CORE', false);" \
+                "define('AUTOMATIC_UPDATER_DISABLED', true);" \
                 "\$table_prefix = 'wp_';" \
                 "if (!defined('ABSPATH')) { define('ABSPATH', __DIR__ . '/'); }" \
                 "require_once ABSPATH . 'wp-settings.php';"
@@ -378,7 +443,6 @@ bootstrap() {
     : "${WORDPRESS_DB_HOST:=mariadb}"
     : "${MYSQL_DATABASE:?MYSQL_DATABASE is required}"
     : "${MYSQL_USER:?MYSQL_USER is required}"
-    : "${DOMAIN_NAME:?DOMAIN_NAME is required}"
     : "${WORDPRESS_URL:?WORDPRESS_URL is required}"
     : "${WORDPRESS_TITLE:?WORDPRESS_TITLE is required}"
     : "${WORDPRESS_ADMIN_USER:?WORDPRESS_ADMIN_USER is required}"
diff --git a/tests/validate_stack.py b/tests/validate_stack.py
index 985d747..c5f4391 100755
--- a/tests/validate_stack.py
+++ b/tests/validate_stack.py
@@ -117,7 +117,7 @@ def validate_dockerfiles() -> None:
         "wordpress": [
             r"FROM\s+debian:bookworm(?:-\d{8})?-slim|FROM\s+alpine:",
             r"php8\.2-fpm|php-fpm",
-            r"wp-cli\.phar",
+            r"wp-cli-\$\{WP_CLI_VERSION\}\.phar",
             r"EXPOSE 9000",
         ],
     }


## `test(supply-chain): 불변 image 입력 검증`

diff --git a/tests/runtime_stack.py b/tests/runtime_stack.py
index 4602e99..7a35b3d 100644
--- a/tests/runtime_stack.py
+++ b/tests/runtime_stack.py
@@ -470,6 +470,10 @@ class RuntimeStack:
             raise StackError("HTTPS 포트 충돌 뒤 새 포트를 선택하지 않았습니다")
         self._verify_legacy_config_migration()
         self.assert_runtime_secret_boundary()
+        if self.wordpress("core", "version", capture=True) != "6.7.1":
+            raise StackError("고정한 WordPress 코어 버전과 실행 버전이 다릅니다")
+        if "WP-CLI 2.11.0" not in self.wordpress("cli", "version", capture=True):
+            raise StackError("고정한 WP-CLI 버전과 실행 버전이 다릅니다")
         if self.fetch("/healthz").strip() != "ok":
             raise StackError("nginx 상태 응답이 예상과 다릅니다")
 
diff --git a/tests/validate_stack.py b/tests/validate_stack.py
index c5f4391..69fc346 100755
--- a/tests/validate_stack.py
+++ b/tests/validate_stack.py
@@ -123,9 +123,34 @@ def validate_dockerfiles() -> None:
     }
     for service, patterns in services.items():
         require_file(f"srcs/requirements/{service}/.dockerignore")
-        require_text(f"srcs/requirements/{service}/Dockerfile", patterns)
+        dockerfile = require_text(f"srcs/requirements/{service}/Dockerfile", patterns)
+        if "bookworm-20241202-slim@sha256:1537a6a1cbc4b4fd401da800ee9480207e7dc1f23560c21259f681db56768f63" not in dockerfile:
+            fail(f"{service} must pin the Debian base image digest")
+        if "snapshot.debian.org/archive/debian/20241214T000000Z" not in dockerfile:
+            fail(f"{service} must use the immutable Debian package snapshot")
         require_executable(f"srcs/requirements/{service}/tools/docker-entrypoint.sh")
 
+    wordpress = require_file("srcs/requirements/wordpress/Dockerfile").read_text()
+    for required in (
+        "WP_CLI_VERSION=2.11.0",
+        "WORDPRESS_VERSION=6.7.1",
+        "a39021ac809530ea607580dbf93afbc46ba02f86b6cffd03de4b126ca53079f6",
+        "33529cd638c845007e8e0d26c91d60c9c16b822c849c8deead03d0c851a26deb",
+        "sha256sum -c -",
+        "/usr/src/wordpress-core.sha256",
+    ):
+        if required not in wordpress:
+            fail(f"wordpress image is missing pinned artifact data: {required}")
+    entrypoint = require_file(
+        "srcs/requirements/wordpress/tools/docker-entrypoint.sh"
+    ).read_text()
+    if (
+        "wp core download" in entrypoint
+        or "/usr/src/wordpress-core.sha256" not in entrypoint
+        or 'cp -p -- "$source" "$temporary"' not in entrypoint
+    ):
+        fail("WordPress must copy the verified image artifact instead of downloading at runtime")
+
 
 def validate_configs() -> None:
     require_text(


## `fix(supply-chain): 보안 지원 runtime pin 갱신`

diff --git a/srcs/requirements/mariadb/Dockerfile b/srcs/requirements/mariadb/Dockerfile
index 1823381..fbc9ed1 100644
--- a/srcs/requirements/mariadb/Dockerfile
+++ b/srcs/requirements/mariadb/Dockerfile
@@ -1,9 +1,9 @@
-FROM debian:bookworm-20241202-slim@sha256:1537a6a1cbc4b4fd401da800ee9480207e7dc1f23560c21259f681db56768f63
+FROM debian:bookworm-20260803-slim@sha256:abd67ffcfa541b485a3dff59865ab629aa048a6c613e639d36e7456b0b229241
 
 RUN printf '%s\n' \
-        'deb [check-valid-until=no] http://snapshot.debian.org/archive/debian/20241214T000000Z bookworm main' \
-        'deb [check-valid-until=no] http://snapshot.debian.org/archive/debian/20241214T000000Z bookworm-updates main' \
-        'deb [check-valid-until=no] http://snapshot.debian.org/archive/debian-security/20241214T000000Z bookworm-security main' \
+        'deb [check-valid-until=no] http://snapshot.debian.org/archive/debian/20260812T000000Z bookworm main' \
+        'deb [check-valid-until=no] http://snapshot.debian.org/archive/debian/20260812T000000Z bookworm-updates main' \
+        'deb [check-valid-until=no] http://snapshot.debian.org/archive/debian-security/20260812T000000Z bookworm-security main' \
         > /etc/apt/sources.list \
     && rm -f /etc/apt/sources.list.d/debian.sources
 
diff --git a/srcs/requirements/nginx/Dockerfile b/srcs/requirements/nginx/Dockerfile
index ab18499..7266e81 100644
--- a/srcs/requirements/nginx/Dockerfile
+++ b/srcs/requirements/nginx/Dockerfile
@@ -1,9 +1,9 @@
-FROM debian:bookworm-20241202-slim@sha256:1537a6a1cbc4b4fd401da800ee9480207e7dc1f23560c21259f681db56768f63
+FROM debian:bookworm-20260803-slim@sha256:abd67ffcfa541b485a3dff59865ab629aa048a6c613e639d36e7456b0b229241
 
 RUN printf '%s\n' \
-        'deb [check-valid-until=no] http://snapshot.debian.org/archive/debian/20241214T000000Z bookworm main' \
-        'deb [check-valid-until=no] http://snapshot.debian.org/archive/debian/20241214T000000Z bookworm-updates main' \
-        'deb [check-valid-until=no] http://snapshot.debian.org/archive/debian-security/20241214T000000Z bookworm-security main' \
+        'deb [check-valid-until=no] http://snapshot.debian.org/archive/debian/20260812T000000Z bookworm main' \
+        'deb [check-valid-until=no] http://snapshot.debian.org/archive/debian/20260812T000000Z bookworm-updates main' \
+        'deb [check-valid-until=no] http://snapshot.debian.org/archive/debian-security/20260812T000000Z bookworm-security main' \
         > /etc/apt/sources.list \
     && rm -f /etc/apt/sources.list.d/debian.sources
 
diff --git a/srcs/requirements/wordpress/Dockerfile b/srcs/requirements/wordpress/Dockerfile
index 28a8f31..fb09bfc 100644
--- a/srcs/requirements/wordpress/Dockerfile
+++ b/srcs/requirements/wordpress/Dockerfile
@@ -1,9 +1,9 @@
-FROM debian:bookworm-20241202-slim@sha256:1537a6a1cbc4b4fd401da800ee9480207e7dc1f23560c21259f681db56768f63
+FROM debian:bookworm-20260803-slim@sha256:abd67ffcfa541b485a3dff59865ab629aa048a6c613e639d36e7456b0b229241
 
 RUN printf '%s\n' \
-        'deb [check-valid-until=no] http://snapshot.debian.org/archive/debian/20241214T000000Z bookworm main' \
-        'deb [check-valid-until=no] http://snapshot.debian.org/archive/debian/20241214T000000Z bookworm-updates main' \
-        'deb [check-valid-until=no] http://snapshot.debian.org/archive/debian-security/20241214T000000Z bookworm-security main' \
+        'deb [check-valid-until=no] http://snapshot.debian.org/archive/debian/20260812T000000Z bookworm main' \
+        'deb [check-valid-until=no] http://snapshot.debian.org/archive/debian/20260812T000000Z bookworm-updates main' \
+        'deb [check-valid-until=no] http://snapshot.debian.org/archive/debian-security/20260812T000000Z bookworm-security main' \
         > /etc/apt/sources.list \
     && rm -f /etc/apt/sources.list.d/debian.sources
 
@@ -25,7 +25,7 @@ RUN apt-get -o Acquire::Check-Valid-Until=false update \
     && rm -rf /var/lib/apt/lists/*
 
 ENV WP_CLI_VERSION=2.11.0 \
-    WORDPRESS_VERSION=6.7.1
+    WORDPRESS_VERSION=6.7.7
 
 RUN curl -fsSL \
         "https://github.com/wp-cli/wp-cli/releases/download/v${WP_CLI_VERSION}/wp-cli-${WP_CLI_VERSION}.phar" \
@@ -35,7 +35,7 @@ RUN curl -fsSL \
     && chmod 0755 /usr/local/bin/wp \
     && curl -fsSL "https://wordpress.org/wordpress-${WORDPRESS_VERSION}.tar.gz" \
         -o /tmp/wordpress.tar.gz \
-    && echo '33529cd638c845007e8e0d26c91d60c9c16b822c849c8deead03d0c851a26deb  /tmp/wordpress.tar.gz' \
+    && echo 'dadac21d0fd6f54f7c0565d8935d3e4baea5e649486ac6f40fe89792da498350  /tmp/wordpress.tar.gz' \
         | sha256sum -c - \
     && mkdir -p /usr/src/wordpress /run/php /var/www/html /var/www/config \
     && tar -xzf /tmp/wordpress.tar.gz --strip-components=1 -C /usr/src/wordpress \
diff --git a/tests/runtime_stack.py b/tests/runtime_stack.py
index 471942c..cc1ba98 100644
--- a/tests/runtime_stack.py
+++ b/tests/runtime_stack.py
@@ -491,7 +491,7 @@ class RuntimeStack:
             raise StackError("HTTPS 포트 충돌 뒤 새 포트를 선택하지 않았습니다")
         self._verify_legacy_config_migration()
         self.assert_runtime_secret_boundary()
-        if self.wordpress("core", "version", capture=True) != "6.7.1":
+        if self.wordpress("core", "version", capture=True) != "6.7.7":
             raise StackError("고정한 WordPress 코어 버전과 실행 버전이 다릅니다")
         if "WP-CLI 2.11.0" not in self.wordpress("cli", "version", capture=True):
             raise StackError("고정한 WP-CLI 버전과 실행 버전이 다릅니다")
diff --git a/tests/validate_stack.py b/tests/validate_stack.py
index fa3c469..425de60 100755
--- a/tests/validate_stack.py
+++ b/tests/validate_stack.py
@@ -275,18 +275,18 @@ def validate_dockerfiles() -> None:
     for service, patterns in services.items():
         require_file(f"srcs/requirements/{service}/.dockerignore")
         dockerfile = require_text(f"srcs/requirements/{service}/Dockerfile", patterns)
-        if "bookworm-20241202-slim@sha256:1537a6a1cbc4b4fd401da800ee9480207e7dc1f23560c21259f681db56768f63" not in dockerfile:
+        if "bookworm-20260803-slim@sha256:abd67ffcfa541b485a3dff59865ab629aa048a6c613e639d36e7456b0b229241" not in dockerfile:
             fail(f"{service} must pin the Debian base image digest")
-        if "snapshot.debian.org/archive/debian/20241214T000000Z" not in dockerfile:
+        if "snapshot.debian.org/archive/debian/20260812T000000Z" not in dockerfile:
             fail(f"{service} must use the immutable Debian package snapshot")
         require_executable(f"srcs/requirements/{service}/tools/docker-entrypoint.sh")
 
     wordpress = require_file("srcs/requirements/wordpress/Dockerfile").read_text()
     for required in (
         "WP_CLI_VERSION=2.11.0",
-        "WORDPRESS_VERSION=6.7.1",
+        "WORDPRESS_VERSION=6.7.7",
         "a39021ac809530ea607580dbf93afbc46ba02f86b6cffd03de4b126ca53079f6",
-        "33529cd638c845007e8e0d26c91d60c9c16b822c849c8deead03d0c851a26deb",
+        "dadac21d0fd6f54f7c0565d8935d3e4baea5e649486ac6f40fe89792da498350",
         "sha256sum -c -",
     ):
         if required not in wordpress:


