## `test(supply-chain): 검토된 runtime 최소 버전 검증`

diff --git a/tests/runtime_stack.py b/tests/runtime_stack.py
index cc1ba98..d99b51d 100644
--- a/tests/runtime_stack.py
+++ b/tests/runtime_stack.py
@@ -6,6 +6,7 @@ from __future__ import annotations
 import argparse
 import json
 import os
+import re
 from pathlib import Path
 import secrets
 import shutil
@@ -25,6 +26,20 @@ CONTROL_TIMEOUT_SECONDS = 600
 BUILD_TIMEOUT_SECONDS = 1200
 BACKUP_TOOL_TIMEOUT_SECONDS = 1200
 PORT_RETRY_LIMIT = 3
+
+DEBIAN_PACKAGE_MINIMUMS = (
+    ("nginx", "nginx", "1.22.1-9+deb12u9"),
+    ("nginx", "openssl", "3.0.20-1~deb12u2"),
+    ("nginx", "libssl3", "3.0.20-1~deb12u2"),
+    ("wordpress", "php8.2-fpm", "8.2.33-1~deb12u1"),
+    ("wordpress", "php8.2-cli", "8.2.33-1~deb12u1"),
+    ("wordpress", "libssl3", "3.0.20-1~deb12u2"),
+    ("mariadb", "mariadb-server", "1:10.11.18-0+deb12u1"),
+    ("mariadb", "libssl3", "3.0.20-1~deb12u2"),
+)
+WORDPRESS_REQUIRED_PHP = (7, 2, 24)
+WORDPRESS_REQUIRED_MYSQL = (5, 5, 5)
+
 PORT_CONFLICT_MARKERS = (
     "address already in use",
     "bind: address already in use",
@@ -480,6 +495,22 @@ class RuntimeStack:
         )
         return result.stdout
 
+    def assert_debian_package_minimum(
+        self, service: str, package: str, minimum: str
+    ) -> str:
+        result = self.run_compose(
+            "exec", "--no-TTY", service, "sh", "-ceu",
+            "installed=\"$(dpkg-query -W -f='${Version}' -- \"$1\")\"; "
+            "dpkg --compare-versions \"$installed\" ge \"$2\"; printf '%s' \"$installed\"",
+            "sh", package, minimum, capture=True, check=False,
+        )
+        installed = result.stdout.strip()
+        if result.returncode != 0:
+            raise StackError(
+                f"{service} {package} 버전 {installed!r}이 최소선 {minimum}보다 낮습니다"
+            )
+        return installed
+
     def verify_e2e(self) -> None:
         blocked_port = self.port
         with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as listener:
@@ -495,6 +526,19 @@ class RuntimeStack:
             raise StackError("고정한 WordPress 코어 버전과 실행 버전이 다릅니다")
         if "WP-CLI 2.11.0" not in self.wordpress("cli", "version", capture=True):
             raise StackError("고정한 WP-CLI 버전과 실행 버전이 다릅니다")
+        installed = [
+            f"{service}/{package}={self.assert_debian_package_minimum(service, package, minimum)}"
+            for service, package, minimum in DEBIAN_PACKAGE_MINIMUMS
+        ]
+        php = self.wordpress("eval", "echo PHP_VERSION;", capture=True)
+        database = self.wordpress("db", "query", "SELECT VERSION()", "--skip-column-names", capture=True)
+        php_match = re.match(r"^(\d+)\.(\d+)\.(\d+)", php)
+        database_match = re.match(r"^(\d+)\.(\d+)\.(\d+)", database)
+        if php_match is None or tuple(map(int, php_match.groups())) < WORDPRESS_REQUIRED_PHP:
+            raise StackError(f"WordPress PHP runtime이 최소선보다 낮습니다: {php}")
+        if (database_match is None or tuple(map(int, database_match.groups())) < WORDPRESS_REQUIRED_MYSQL or "MariaDB" not in database):
+            raise StackError(f"WordPress DB runtime이 최소선보다 낮습니다: {database}")
+        print("verified runtime versions: " + ", ".join(installed + [f"php={php}", f"database={database}"]))
         if self.fetch("/healthz").strip() != "ok":
             raise StackError("nginx 상태 응답이 예상과 다릅니다")
 
diff --git a/tests/validate_stack.py b/tests/validate_stack.py
index 425de60..5548781 100755
--- a/tests/validate_stack.py
+++ b/tests/validate_stack.py
@@ -304,6 +304,20 @@ def validate_dockerfiles() -> None:
         if required not in entrypoint:
             fail(f"WordPress atomic artifact publication is missing {required!r}")
 
+    runtime = require_file("tests/runtime_stack.py").read_text()
+    for required in (
+        "DEBIAN_PACKAGE_MINIMUMS",
+        "1.22.1-9+deb12u9",
+        "3.0.20-1~deb12u2",
+        "8.2.33-1~deb12u1",
+        "1:10.11.18-0+deb12u1",
+        "dpkg --compare-versions",
+        "WORDPRESS_REQUIRED_PHP = (7, 2, 24)",
+        "WORDPRESS_REQUIRED_MYSQL = (5, 5, 5)",
+    ):
+        if required not in runtime:
+            fail(f"runtime supply-chain check is missing {required!r}")
+
 
 def validate_configs() -> None:
     require_text(
