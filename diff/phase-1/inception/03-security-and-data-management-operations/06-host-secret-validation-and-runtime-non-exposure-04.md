## `test(secrets): 회전 후 런타임 비밀 경계 고정`

diff --git a/tests/validate_stack.py b/tests/validate_stack.py
index fccbbdf..f994042 100755
--- a/tests/validate_stack.py
+++ b/tests/validate_stack.py
@@ -403,6 +403,25 @@ def validate_bootstrap_recovery() -> None:
     )
 
 
+def validate_rotation_runtime_boundary() -> None:
+    runtime = require_file("tests/runtime_stack.py").read_text()
+    forbidden = (
+        "def _mounted_secret_matches",
+        "/run/secrets/db_root_password",
+        "/run/secrets/wp_admin_password",
+    )
+    for fragment in forbidden:
+        if fragment in runtime:
+            fail(f"rotation validation depends on an obsolete secret mount: {fragment}")
+    for required in (
+        "find /var/www/config -maxdepth 1 -type f",
+        ".wp-config.rotate.*",
+        "self.assert_runtime_secret_boundary(expected)",
+    ):
+        if required not in runtime:
+            fail(f"rotation runtime boundary is missing {required!r}")
+
+
 def main() -> None:
     validate_source_only()
     validate_compose()
@@ -411,6 +430,7 @@ def main() -> None:
     validate_env_policy()
     validate_tools()
     validate_bootstrap_recovery()
+    validate_rotation_runtime_boundary()
     print("static stack validation passed")
 
 


