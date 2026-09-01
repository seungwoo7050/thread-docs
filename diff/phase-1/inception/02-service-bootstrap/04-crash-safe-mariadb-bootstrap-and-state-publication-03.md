## `test(init): 단계별 초기화 계약 검사`

diff --git a/tests/validate_stack.py b/tests/validate_stack.py
index 856154c..cf5898f 100755
--- a/tests/validate_stack.py
+++ b/tests/validate_stack.py
@@ -181,6 +181,28 @@ def validate_tools() -> None:
     )
 
 
+def validate_bootstrap_recovery() -> None:
+    require_text(
+        "srcs/requirements/mariadb/tools/docker-entrypoint.sh",
+        [
+            r"\.container-stack-initialized",
+            r"timed out waiting for temporary MariaDB server",
+            r"staging_dir",
+            r"database-publish",
+            r"ALTER USER '\$\{MYSQL_USER\}'@'%'",
+        ],
+    )
+    require_text(
+        "srcs/requirements/wordpress/tools/docker-entrypoint.sh",
+        [
+            r"\.container-stack-initialized",
+            r"timed out waiting for authenticated MariaDB access",
+            r"wp core is-installed",
+            r"config_dir=.*?/var/www/config",
+        ],
+    )
+
+
 def main() -> None:
     validate_source_only()
     validate_compose()
@@ -188,6 +210,7 @@ def main() -> None:
     validate_configs()
     validate_env_policy()
     validate_tools()
+    validate_bootstrap_recovery()
     print("static stack validation passed")
 
 


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
