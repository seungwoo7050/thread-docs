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
 


## `test(persistence): 재시작·재생성 뒤 상태 보존 검증`

diff --git a/Makefile b/Makefile
index 2848a7d..05cc1a5 100644
--- a/Makefile
+++ b/Makefile
@@ -6,7 +6,7 @@ WAIT_TIMEOUT ?= 300
 
 COMPOSE_RUN := $(COMPOSE) --project-name $(PROJECT_NAME) --env-file $(ENV_FILE) -f $(COMPOSE_FILE)
 
-.PHONY: up start-database start-application down build logs ps clean fclean test config smoke bootstrap-test e2e
+.PHONY: up start-database start-application down build logs ps clean fclean test config smoke bootstrap-test e2e persistence
 
 up:
 	python3 tools/start_stack.py start --project "$(PROJECT_NAME)" --env-file "$(ENV_FILE)" --wait-timeout "$(WAIT_TIMEOUT)"
@@ -54,3 +54,6 @@ bootstrap-test:
 
 e2e:
 	python3 tests/runtime_stack.py e2e
+
+persistence:
+	python3 tests/runtime_stack.py persistence
diff --git a/tests/runtime_stack.py b/tests/runtime_stack.py
index fdb2739..d9eef12 100644
--- a/tests/runtime_stack.py
+++ b/tests/runtime_stack.py
@@ -233,6 +233,24 @@ class RuntimeStack:
         )
         return result.stdout.strip() if capture else ""
 
+    def project_volumes(self) -> set[str]:
+        result = subprocess.run(
+            [
+                "docker",
+                "volume",
+                "ls",
+                "--filter",
+                f"label=com.docker.compose.project={self.project}",
+                "--format",
+                "{{.Name}}",
+            ],
+            check=True,
+            text=True,
+            capture_output=True,
+            timeout=PROCESS_TIMEOUT_SECONDS,
+        )
+        return {line for line in result.stdout.splitlines() if line}
+
     def inspect_service(self, service: str) -> dict[str, object]:
         container_id = self.run_compose(
             "ps", "--quiet", service, capture=True
@@ -447,6 +465,78 @@ class RuntimeStack:
         if migrated.returncode != 0:
             raise StackError("기존 WordPress 설정을 전용 볼륨으로 옮기지 못했습니다")
 
+    def _verify_persistent_values(
+        self, *, post_id: str, title: str, content: str, filename: str, file_value: str
+    ) -> None:
+        page = self.fetch(f"/?p={post_id}")
+        if title not in page or content not in page:
+            raise StackError("재기동 뒤 게시물 내용이 보존되지 않았습니다")
+        option = self.wordpress(
+            "option", "get", "container_stack_persistence", capture=True
+        )
+        if option != content:
+            raise StackError("재기동 뒤 WordPress 옵션 값이 보존되지 않았습니다")
+        if self.fetch(f"/wp-content/uploads/{filename}") != file_value:
+            raise StackError("재기동 뒤 업로드 파일이 보존되지 않았습니다")
+
+    def verify_persistence(self) -> None:
+        self.start()
+        nonce = secrets.token_hex(8)
+        title = f"영속성 검증 {nonce}"
+        content = f"persistent-database-{nonce}"
+        filename = f"persistence-{nonce}.txt"
+        file_value = f"persistent-volume-{nonce}\n"
+        post_id = self.wordpress(
+            "post",
+            "create",
+            f"--post_title={title}",
+            f"--post_content={content}",
+            "--post_status=publish",
+            "--porcelain",
+            capture=True,
+        )
+        self.wordpress("option", "update", "container_stack_persistence", content)
+        php_value = file_value.replace("\\", "\\\\").replace('"', '\\"').replace("$", "\\$")
+        php_file = filename.replace('"', '\\"')
+        self.wordpress(
+            "eval",
+            "wp_mkdir_p(WP_CONTENT_DIR . '/uploads'); "
+            f'file_put_contents(WP_CONTENT_DIR . "/uploads/{php_file}", "{php_value}");',
+        )
+        initial_volumes = self.project_volumes()
+        if len(initial_volumes) != 3:
+            raise StackError(f"예상한 영구 볼륨 세 개를 찾지 못했습니다: {initial_volumes}")
+        self._verify_persistent_values(
+            post_id=post_id,
+            title=title,
+            content=content,
+            filename=filename,
+            file_value=file_value,
+        )
+
+        self.run_compose("restart", "mariadb", "wordpress", "nginx")
+        self.run_compose("up", "--detach", "--wait", "--wait-timeout", "240")
+        self._verify_persistent_values(
+            post_id=post_id,
+            title=title,
+            content=content,
+            filename=filename,
+            file_value=file_value,
+        )
+
+        self.run_compose("down", "--remove-orphans", "--timeout", "20")
+        self.run_compose("up", "--detach", "--wait", "--wait-timeout", "240")
+        if self.project_volumes() != initial_volumes:
+            raise StackError("서비스 재생성 과정에서 영구 볼륨이 교체되었습니다")
+        self._verify_persistent_values(
+            post_id=post_id,
+            title=title,
+            content=content,
+            filename=filename,
+            file_value=file_value,
+        )
+        print(f"restart and recreation persistence passed: project={self.project}")
+
     def collect_diagnostics(self) -> Path:
         destination = self.diagnostics_dir
         if destination is None:
@@ -488,7 +578,7 @@ class RuntimeStack:
 
 def parse_arguments() -> argparse.Namespace:
     parser = argparse.ArgumentParser(description="격리된 컨테이너 스택 검증")
-    parser.add_argument("scenario", choices=("bootstrap", "e2e"))
+    parser.add_argument("scenario", choices=("bootstrap", "e2e", "persistence"))
     parser.add_argument("--keep", action="store_true", help="검사 뒤 프로젝트를 유지합니다")
     parser.add_argument("--diagnostics-dir", type=Path)
     return parser.parse_args()
@@ -514,6 +604,8 @@ def main() -> int:
     try:
         if args.scenario == "bootstrap":
             stack.verify_bootstrap()
+        elif args.scenario == "persistence":
+            stack.verify_persistence()
         else:
             stack.verify_e2e()
         failed = False
diff --git a/tests/validate_stack.py b/tests/validate_stack.py
index f02e68f..3151c40 100755
--- a/tests/validate_stack.py
+++ b/tests/validate_stack.py
@@ -182,6 +182,8 @@ def validate_tools() -> None:
             r"runtime_stack\.py bootstrap",
             r"^e2e:",
             r"runtime_stack\.py e2e",
+            r"^persistence:",
+            r"runtime_stack\.py persistence",
         ],
     )
     require_text(
@@ -192,6 +194,8 @@ def validate_tools() -> None:
             r'"post",\s*\n\s*"create"',
             r"tools.+start_stack\.py",
             r'"bootstrap",\s*"e2e"',
+            r"def verify_persistence",
+            r"len\(initial_volumes\) != 3",
         ],
     )
 
