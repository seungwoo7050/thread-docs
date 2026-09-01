## `fix(release): pin uv interpreter discovery`

diff --git a/operations/tests/test_release_artifact.py b/operations/tests/test_release_artifact.py
index efed315..87bcd6e 100644
--- a/operations/tests/test_release_artifact.py
+++ b/operations/tests/test_release_artifact.py
@@ -145,22 +145,23 @@ class ReleaseArtifactTests(unittest.TestCase):
         builder.chmod(0o755)
         self.uv = self._write(
             "tools/uv",
-            """
+            f"""
             #!/bin/sh
-            [ -z "${MOFA_TRAVEL_ALARM_SERVICE_KEY:-}" ] || exit 97
-            [ -z "${UNRELATED_RELEASE_PARENT_MARKER:-}" ] || exit 97
-            [ "${PATH:-}" = "/usr/bin:/bin" ] || exit 97
-            if [ "${1:-}" = "--version" ]; then
+            [ -z "${{MOFA_TRAVEL_ALARM_SERVICE_KEY:-}}" ] || exit 97
+            [ -z "${{UNRELATED_RELEASE_PARENT_MARKER:-}}" ] || exit 97
+            [ "${{PATH:-}}" = "/usr/bin:/bin" ] || exit 97
+            [ "${{UV_PYTHON:-}}" = "{sys.executable}" ] || exit 97
+            if [ "${{1:-}}" = "--version" ]; then
                 printf '%s\\n' 'uv 0.12.6 (fixture build metadata)'
                 exit 0
             fi
-            if [ "${1:-}" = "lock" ]; then
-                [ "${2:-}" = "--check" ] || exit 98
-                [ "${3:-}" = "--offline" ] || exit 98
+            if [ "${{1:-}}" = "lock" ]; then
+                [ "${{2:-}}" = "--check" ] || exit 98
+                [ "${{3:-}}" = "--offline" ] || exit 98
                 exit 0
             fi
-            [ "${1:-}" = "pip" ] || exit 98
-            [ "${2:-}" = "check" ] || exit 98
+            [ "${{1:-}}" = "pip" ] || exit 98
+            [ "${{2:-}}" = "check" ] || exit 98
             exit 0
             """,
             executable=True,
diff --git a/scripts/build-release b/scripts/build-release
index 3707432..43a55d8 100755
--- a/scripts/build-release
+++ b/scripts/build-release
@@ -489,6 +489,7 @@ def check_uv(
             "UV_CACHE_DIR": str(cache),
             "UV_NO_CONFIG": "1",
             "UV_OFFLINE": "1",
+            "UV_PYTHON": sys.executable,
             "UV_PYTHON_DOWNLOADS": "never",
         }
     )


