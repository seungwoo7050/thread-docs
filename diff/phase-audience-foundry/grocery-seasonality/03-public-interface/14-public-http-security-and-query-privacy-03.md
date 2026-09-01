## `fix(security): suppress public query referrers`

diff --git a/grocery/security.py b/grocery/security.py
index df38a30..3d95c5c 100644
--- a/grocery/security.py
+++ b/grocery/security.py
@@ -29,7 +29,7 @@ SECURITY_HEADERS: Final[dict[str, str]] = {
     "Permissions-Policy": "camera=(), geolocation=(), microphone=(), payment=()",
     "Cross-Origin-Opener-Policy": "same-origin",
     "Cross-Origin-Resource-Policy": "same-origin",
-    "Referrer-Policy": "same-origin",
+    "Referrer-Policy": "no-referrer",
     "X-Content-Type-Options": "nosniff",
     "X-Frame-Options": "DENY",
 }
