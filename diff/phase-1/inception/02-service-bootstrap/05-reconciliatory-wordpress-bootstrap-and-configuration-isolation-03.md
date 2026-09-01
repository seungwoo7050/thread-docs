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


## `test(init): 안정 단계별 초기화 중단 복구 검증`

diff --git a/tests/runtime_stack.py b/tests/runtime_stack.py
index bac892b..e5080d3 100644
--- a/tests/runtime_stack.py
+++ b/tests/runtime_stack.py
@@ -463,26 +463,6 @@ class RuntimeStack:
         )
         return result.stdout
 
-    def verify_bootstrap(self) -> None:
-        self.start()
-        self.assert_runtime_secret_boundary()
-        for service, marker in (
-            ("mariadb", "/var/lib/mysql-volume/data/.container-stack-initialized"),
-            ("wordpress", "/var/www/html/.container-stack-initialized"),
-        ):
-            result = self.run_compose(
-                "exec",
-                "--no-TTY",
-                service,
-                "test",
-                "-f",
-                marker,
-                capture=True,
-                check=False,
-            )
-            if result.returncode != 0:
-                raise StackError(f"{service} 초기화 완료 표식이 없습니다")
-        print("bootstrap completion and secret boundary passed")
 
     def verify_e2e(self) -> None:
         blocked_port = self.port
@@ -727,6 +707,177 @@ class RuntimeStack:
                 )
             time.sleep(0.1)
 
+    def _interrupt_bootstrap(
+        self, *, action: str, service: str, stage: str
+    ) -> None:
+        ready_file = self.temp / f"bootstrap-{service}-{stage}.ready"
+        command = self._start_command(
+            action,
+            pause_after=stage,
+            pause_ready_file=ready_file,
+        )
+        process = subprocess.Popen(
+            command,
+            cwd=ROOT,
+            text=True,
+            stdout=subprocess.PIPE,
+            stderr=subprocess.PIPE,
+        )
+        try:
+            self._wait_for_ready_file(
+                process,
+                ready_file,
+                f"{service} {stage} 초기화 단계",
+            )
+            container_name = f"{self.project}-{service}-bootstrap"
+            inspection = subprocess.run(
+                ["docker", "container", "inspect", container_name],
+                check=True,
+                text=True,
+                capture_output=True,
+                timeout=PROCESS_TIMEOUT_SECONDS,
+            )
+            containers = json.loads(inspection.stdout)
+            labels = containers[0]["Config"]["Labels"]
+            if (
+                labels.get("com.docker.compose.project") != self.project
+                or labels.get("com.container-stack.bootstrap") != service
+            ):
+                raise StackError("초기화 컨테이너의 소유권 라벨이 예상과 다릅니다")
+            container_id = str(containers[0]["Id"])
+            killed = subprocess.run(
+                ["docker", "kill", "--signal", "KILL", container_id],
+                text=True,
+                capture_output=True,
+                timeout=PROCESS_TIMEOUT_SECONDS,
+            )
+            if killed.returncode != 0:
+                state = containers[0].get("State", {})
+                raise StackError(
+                    f"{service} {stage} 초기화 컨테이너를 강제 종료하지 못했습니다 "
+                    f"(state={state}): "
+                    f"{killed.stderr.strip() or killed.stdout.strip()}"
+                )
+            stdout, stderr = process.communicate(timeout=PROCESS_TIMEOUT_SECONDS)
+        finally:
+            if process.poll() is None:
+                self._terminate_process(process)
+            ready_file.unlink(missing_ok=True)
+        if process.returncode == 0:
+            raise StackError(
+                f"{service} {stage} 강제 종료가 실패로 전달되지 않았습니다: "
+                f"{stderr.strip() or stdout.strip()}"
+            )
+
+    def _clear_wordpress_volume(self) -> None:
+        self.run_compose("stop", "nginx", "wordpress", check=False)
+        self.run_compose(
+            "run",
+            "--rm",
+            "--no-TTY",
+            "--no-deps",
+            "--entrypoint",
+            "sh",
+            "wordpress",
+            "-ceu",
+            "find /var/www/html -mindepth 1 -maxdepth 1 -exec rm -rf -- {} +; "
+            "find /var/www/config -mindepth 1 -maxdepth 1 -exec rm -rf -- {} +",
+        )
+
+    def verify_bootstrap_recovery(self) -> None:
+        self.started = True
+        self.run_compose(
+            "build",
+            "mariadb",
+            "wordpress",
+            "nginx",
+            timeout=BUILD_TIMEOUT_SECONDS,
+        )
+        database_stages = (
+            "system-tables",
+            "temporary-server",
+            "database-state",
+            "database-marker",
+            "database-publish",
+        )
+        for index, stage in enumerate(database_stages):
+            if index:
+                self.run_compose(
+                    "down",
+                    "--volumes",
+                    "--remove-orphans",
+                    "--timeout",
+                    "20",
+                )
+            self._interrupt_bootstrap(
+                action="database",
+                service="mariadb",
+                stage=stage,
+            )
+            self._run_start("database")
+            state = self.run_compose(
+                "exec",
+                "--no-TTY",
+                "mariadb",
+                "sh",
+                "-ceu",
+                "test -f /var/lib/mysql-volume/data/.container-stack-initialized; "
+                "test ! -e /var/lib/mysql-volume/.container-stack-bootstrap",
+                capture=True,
+                check=False,
+            )
+            if state.returncode != 0:
+                raise StackError(f"MariaDB {stage} 재실행 뒤 상태가 수렴하지 않았습니다")
+
+        application_stages = (
+            "core-files",
+            "wordpress-config",
+            "wordpress-core",
+            "wordpress-users",
+            "wordpress-marker",
+        )
+        for index, stage in enumerate(application_stages):
+            if index:
+                self._clear_wordpress_volume()
+            self._interrupt_bootstrap(
+                action="application",
+                service="wordpress",
+                stage=stage,
+            )
+            self._run_start("application")
+            state = self.run_compose(
+                "exec",
+                "--no-TTY",
+                "wordpress",
+                "sh",
+                "-ceu",
+                "test -f /var/www/html/.container-stack-initialized; "
+                "test -L /var/www/html/wp-config.php; "
+                "test -f /var/www/config/wp-config.php; "
+                "test -z \"$(find /var/www/html -type f "
+                "\\( -name '*.bootstrap.*' -o -name '*.tmp.*' \\) -print -quit)\"; "
+                "test -z \"$(find /var/www/config -type f "
+                "\\( -name '*.bootstrap.*' -o -name '*.tmp.*' \\) -print -quit)\"",
+                capture=True,
+                check=False,
+            )
+            if state.returncode != 0:
+                raise StackError(
+                    f"WordPress {stage} 재실행 뒤 상태가 수렴하지 않았습니다"
+                )
+            if not self._wordpress_password_works(
+                "admin", self.credential_values["wp_admin_password.txt"]
+            ) or not self._wordpress_password_works(
+                "user", self.credential_values["wp_user_password.txt"]
+            ):
+                raise StackError(
+                    f"WordPress {stage} 재실행 뒤 사용자 인증이 복구되지 않았습니다"
+                )
+
+        self.assert_runtime_secret_boundary()
+        self.verify_services_running()
+        print("bootstrap SIGKILL recovery and secret boundary passed")
+
     def _interrupt_backup_tool(
         self,
         operation: str,
@@ -1691,7 +1842,7 @@ def main() -> int:
     result = 0
     try:
         if args.scenario == "bootstrap":
-            stack.verify_bootstrap()
+            stack.verify_bootstrap_recovery()
         elif args.scenario == "e2e":
             stack.verify_e2e()
         elif args.scenario == "persistence":
