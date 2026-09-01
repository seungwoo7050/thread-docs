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


## `build(compose): 엄격한 설정 검사 추가`

diff --git a/Makefile b/Makefile
index b51d679..a631f27 100644
--- a/Makefile
+++ b/Makefile
@@ -9,7 +9,7 @@ DESTROY_CONFIRM ?=
 
 COMPOSE_RUN := $(COMPOSE) --project-name $(PROJECT_NAME) --env-file $(ENV_FILE) -f $(COMPOSE_FILE)
 
-.PHONY: up start-database start-application down build logs ps clean fclean test config smoke bootstrap-test e2e persistence backup restore backup-restore-test rotate-secrets rotation-test
+.PHONY: up start-database start-application down build logs ps clean fclean test config smoke bootstrap-test e2e persistence backup restore backup-restore-test rotate-secrets rotation-test config-strict
 
 up:
 	python3 tools/start_stack.py start --project "$(PROJECT_NAME)" --env-file "$(ENV_FILE)" --wait-timeout "$(WAIT_TIMEOUT)"
@@ -44,6 +44,11 @@ fclean:
 config:
 	$(COMPOSE_RUN) config
 
+config-strict:
+	@command -v docker >/dev/null 2>&1 || { echo "docker 명령을 찾을 수 없습니다." >&2; exit 2; }
+	@docker compose version >/dev/null 2>&1 || { echo "Docker Compose v2를 사용할 수 없습니다." >&2; exit 2; }
+	$(COMPOSE_RUN) config --quiet
+
 test:
 	python3 tests/validate_stack.py
 	@if command -v docker >/dev/null 2>&1 && docker compose version >/dev/null 2>&1; then \


## `test(runtime): 프로세스·비밀값·정리 제어 흐름 강화`

diff --git a/tests/runtime_stack.py b/tests/runtime_stack.py
index ab38e5c..bac892b 100644
--- a/tests/runtime_stack.py
+++ b/tests/runtime_stack.py
@@ -15,6 +15,7 @@ import stat
 import subprocess
 import sys
 import tempfile
+from typing import Any
 import time
 
 
@@ -54,13 +55,23 @@ def write_private(path: Path, value: str) -> None:
         stream.write(value)
         stream.write("\n")
     if stat.S_IMODE(path.stat().st_mode) != 0o600:
-        raise StackError(f"비밀 파일 권한이 0600이 아닙니다: {path}")
+        raise StackError(f"비공개 파일 권한이 0600이 아닙니다: {path}")
 
 
 def replace_private(path: Path, value: str) -> None:
-    temporary = path.with_name(f".{path.name}.{secrets.token_hex(4)}")
-    write_private(temporary, value.rstrip("\n"))
-    os.replace(temporary, path)
+    temporary = path.with_name(f".{path.name}.{secrets.token_hex(6)}")
+    descriptor = os.open(temporary, os.O_WRONLY | os.O_CREAT | os.O_EXCL, 0o600)
+    try:
+        with os.fdopen(descriptor, "w", encoding="utf-8") as stream:
+            descriptor = -1
+            stream.write(value)
+            stream.flush()
+            os.fsync(stream.fileno())
+        os.replace(temporary, path)
+    finally:
+        if descriptor >= 0:
+            os.close(descriptor)
+        temporary.unlink(missing_ok=True)
 
 
 class RuntimeStack:
@@ -166,17 +177,19 @@ class RuntimeStack:
                 timeout=timeout,
             )
         except subprocess.TimeoutExpired as error:
+            operation = arguments[0] if arguments else "command"
             raise StackError(
-                f"Compose 명령이 {timeout}초 안에 끝나지 않았습니다"
+                f"Compose {operation} 명령이 {timeout}초 안에 끝나지 않았습니다"
             ) from error
 
-    def _run_start(
+    def _start_command(
         self,
         action: str,
         *,
         build: bool = False,
-        check: bool = True,
-    ) -> subprocess.CompletedProcess[str]:
+        pause_after: str | None = None,
+        pause_ready_file: Path | None = None,
+    ) -> list[str]:
         command = [
             sys.executable,
             str(ROOT / "tools" / "start_stack.py"),
@@ -190,9 +203,21 @@ class RuntimeStack:
         ]
         if build:
             command.append("--build")
+        if pause_after is not None and pause_ready_file is not None:
+            command.extend(("--pause-after", pause_after))
+            command.extend(("--pause-ready-file", str(pause_ready_file)))
+        return command
+
+    def _run_start(
+        self,
+        action: str,
+        *,
+        build: bool = False,
+        check: bool = True,
+    ) -> subprocess.CompletedProcess[str]:
         try:
             result = subprocess.run(
-                command,
+                self._start_command(action, build=build),
                 cwd=ROOT,
                 text=True,
                 capture_output=True,
@@ -600,7 +625,8 @@ class RuntimeStack:
 
         self.run_compose("down", "--remove-orphans", "--timeout", "20")
         self.run_compose("up", "--detach", "--wait", "--wait-timeout", "240")
-        if self.project_volumes() != initial_volumes:
+        recreated_volumes = self.project_volumes()
+        if recreated_volumes != initial_volumes:
             raise StackError("서비스 재생성 과정에서 영구 볼륨이 교체되었습니다")
         self._verify_persistent_values(
             post_id=post_id,
@@ -1599,23 +1625,28 @@ if ($text === false || !preg_match($pattern, $text, $matches) || !hash_equals($p
         print(f"진단 자료: {destination}", file=sys.stderr)
         return destination
 
-    def close(self, *, failed: bool) -> None:
+    def close(self, *, failed: bool) -> list[str]:
+        failures: list[str] = []
         if failed:
             try:
                 self.collect_diagnostics()
-            except (OSError, subprocess.CalledProcessError) as error:
-                print(f"진단 자료를 저장하지 못했습니다: {error}", file=sys.stderr)
+            except Exception as error:
+                failures.append(f"진단 자료: {error}")
         if self.started and not self.keep:
-            self.run_compose(
-                "down", "--volumes", "--remove-orphans", "--timeout", "20", check=False
-            )
+            try:
+                result = self.run_compose(
+                    "down", "--volumes", "--remove-orphans", "--timeout", "20",
+                    capture=True, check=False,
+                )
+                if result.returncode != 0:
+                    failures.append(result.stderr.strip() or "Compose 자원 정리 실패")
+            except Exception as error:
+                failures.append(f"Compose 자원 정리: {error}")
         if self.keep:
-            print(
-                f"검사용 프로젝트를 유지합니다: {self.project} ({self.temp})",
-                file=sys.stderr,
-            )
+            print(f"검사용 프로젝트를 유지합니다: {self.project} ({self.temp})", file=sys.stderr)
         else:
             shutil.rmtree(self.temp, ignore_errors=True)
+        return failures
 
 
 def parse_arguments() -> argparse.Namespace:
@@ -1648,19 +1679,23 @@ def main() -> int:
             stdout=subprocess.DEVNULL,
             timeout=PROCESS_TIMEOUT_SECONDS,
         )
-        stack = RuntimeStack(keep=args.keep, diagnostics_dir=args.diagnostics_dir)
+        stack = RuntimeStack(
+            keep=args.keep,
+            diagnostics_dir=args.diagnostics_dir,
+        )
     except (OSError, StackError, subprocess.SubprocessError) as error:
         print(f"검증 환경을 준비하지 못했습니다: {error}", file=sys.stderr)
         return 2
 
     failed = True
+    result = 0
     try:
         if args.scenario == "bootstrap":
             stack.verify_bootstrap()
-        elif args.scenario == "persistence":
-            stack.verify_persistence()
         elif args.scenario == "e2e":
             stack.verify_e2e()
+        elif args.scenario == "persistence":
+            stack.verify_persistence()
         elif args.scenario == "backup-restore":
             stack.verify_backup_restore()
         elif args.scenario == "rotation":
@@ -1668,12 +1703,19 @@ def main() -> int:
         else:
             stack.verify_operations()
         failed = False
-        return 0
     except (OSError, StackError, subprocess.SubprocessError) as error:
         print(f"{args.scenario} 검증 실패: {error}", file=sys.stderr)
-        return 1
+        result = 1
     finally:
-        stack.close(failed=failed)
+        cleanup_failures = stack.close(failed=failed)
+        if cleanup_failures:
+            print(
+                "검증 정리 중 오류가 발생했습니다: " + "; ".join(cleanup_failures),
+                file=sys.stderr,
+            )
+            if result == 0:
+                result = 1
+    return result
 
 
 if __name__ == "__main__":


