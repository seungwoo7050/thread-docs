## `Record E04 session lifecycle verification`

diff --git a/evidence/E04/browser.json b/evidence/E04/browser.json
new file mode 100644
index 0000000..4e7aac0
--- /dev/null
+++ b/evidence/E04/browser.json
@@ -0,0 +1,19 @@
+{
+  "wrongLoginStatus": 401,
+  "aliceLoginStatus": 200,
+  "reloadAuthenticated": true,
+  "cookieFlags": {
+    "httpOnly": true,
+    "sameSite": "Lax",
+    "secure": false,
+    "path": "/",
+    "hostOnly": true
+  },
+  "cookieInvisibleToJavaScript": true,
+  "credentialStorageEntries": 0,
+  "logoutStatus": 200,
+  "logoutClearedCookie": true,
+  "revokedStatus": 401,
+  "bobLoginStatus": 200,
+  "durationMs": 4486
+}
diff --git a/evidence/E04/schema.json b/evidence/E04/schema.json
new file mode 100644
index 0000000..2129ba3
--- /dev/null
+++ b/evidence/E04/schema.json
@@ -0,0 +1,56 @@
+{
+  "results": [
+    {
+      "mutation": "missing_password_hash",
+      "startupRejected": true,
+      "listening": false
+    },
+    {
+      "mutation": "nullable_username",
+      "startupRejected": true,
+      "listening": false
+    },
+    {
+      "mutation": "missing_user_key",
+      "startupRejected": true,
+      "listening": false
+    },
+    {
+      "mutation": "missing_unique_username",
+      "startupRejected": true,
+      "listening": false
+    },
+    {
+      "mutation": "unexpected_user_column",
+      "startupRejected": true,
+      "listening": false
+    },
+    {
+      "mutation": "nullable_expiry",
+      "startupRejected": true,
+      "listening": false
+    },
+    {
+      "mutation": "wrong_expiry_type",
+      "startupRejected": true,
+      "listening": false
+    },
+    {
+      "mutation": "wrong_expiry_precision",
+      "startupRejected": true,
+      "listening": false
+    },
+    {
+      "mutation": "missing_session_key",
+      "startupRejected": true,
+      "listening": false
+    },
+    {
+      "mutation": "missing_session_parent",
+      "startupRejected": true,
+      "listening": false
+    }
+  ],
+  "count": 10,
+  "durationMs": 364
+}
diff --git a/evidence/E04/sessions.json b/evidence/E04/sessions.json
new file mode 100644
index 0000000..7cf6dea
--- /dev/null
+++ b/evidence/E04/sessions.json
@@ -0,0 +1,49 @@
+{
+  "passwordRepresentation": {
+    "count": 2,
+    "saltedScrypt": true,
+    "independentSalts": true,
+    "noPlaintext": true
+  },
+  "anonymousStatus": 401,
+  "cookieFlags": {
+    "name": "wse_session",
+    "httpOnly": true,
+    "sameSite": "Lax",
+    "secure": false,
+    "path": "/",
+    "hostOnly": true,
+    "maxAgeSeconds": 3600
+  },
+  "aliceLoginStatus": 200,
+  "wrongPasswordStatus": 401,
+  "authenticatedStatus": 200,
+  "rotation": {
+    "changed": true,
+    "oldStatus": 401,
+    "newStatus": 200
+  },
+  "logout": {
+    "status": 200,
+    "oldStatus": 401,
+    "cleared": true
+  },
+  "expiry": {
+    "beforeStatus": 200,
+    "atExpiryStatus": 401,
+    "sleepMs": 0
+  },
+  "bobLoginStatus": 200,
+  "accountExistenceHidden": true,
+  "sessionStorage": {
+    "hashOnly": true,
+    "finiteTtl": true,
+    "revokedRowsAbsent": true
+  },
+  "newInstance": {
+    "validStatus": 200,
+    "revokedStatus": 401
+  },
+  "protectedRejections": 40,
+  "durationMs": 2308
+}
diff --git a/evidence/E04/verification.json b/evidence/E04/verification.json
new file mode 100644
index 0000000..b07d811
--- /dev/null
+++ b/evidence/E04/verification.json
@@ -0,0 +1,80 @@
+{
+  "thread": "E04",
+  "attempt": 1,
+  "specRevision": "0a006589477f8ae47bad3faa5510c999cff85ee4",
+  "start": "a2b51405f406a86b7d81b09c00905a25be05c8ec",
+  "runtime": {
+    "node": "24.19.0",
+    "npm": "11.17.0"
+  },
+  "database": "Existing wse-fundamentals PostgreSQL on loopback 15431; only e03_ and e04_ test schemas recreated",
+  "baseline": {
+    "httpRequests": 1,
+    "status": 200,
+    "requiredStatus": 401,
+    "durationMs": 132
+  },
+  "preflight": {
+    "failedBeforeHttpRequests": 1,
+    "reason": "The initial disposable schema label contained digits forbidden by the existing e03_ test helper. Only the label was corrected before the first HTTP request; application code and scenario inputs were unchanged."
+  },
+  "runs": [
+    {
+      "command": "fnm exec --using 24.19.0 npm run typecheck",
+      "exitCode": 0,
+      "durationMs": 4064
+    },
+    {
+      "command": "fnm exec --using 24.19.0 npm run test:unit",
+      "exitCode": 0,
+      "tests": 11,
+      "passed": 11,
+      "durationMs": 2331
+    },
+    {
+      "command": "fnm exec --using 24.19.0 npm run test:functional",
+      "exitCode": 0,
+      "tests": 14,
+      "passed": 14,
+      "durationMs": 6781
+    },
+    {
+      "command": "fnm exec --using 24.19.0 npm run test:integration",
+      "exitCode": 0,
+      "tests": 7,
+      "passed": 7,
+      "durationMs": 3997
+    },
+    {
+      "command": "fnm exec --using 24.19.0 npm run typecheck",
+      "phase": "browser implementation",
+      "exitCode": 0,
+      "durationMs": 2209
+    },
+    {
+      "command": "fnm exec --using 24.19.0 npm run test:e2e",
+      "exitCode": 0,
+      "tests": 6,
+      "passed": 6,
+      "durationMs": 21200,
+      "workers": 1,
+      "retries": 0
+    },
+    {
+      "command": "fnm exec --using 24.19.0 npm run build",
+      "exitCode": 0,
+      "durationMs": 9220
+    }
+  ],
+  "schemaMutations": 10,
+  "protectedRouteRejections": 40,
+  "browserWorkers": 1,
+  "retries": 0,
+  "loadRuns": 0,
+  "benchmarkRuns": 0,
+  "parameterSweeps": 0,
+  "earlierScenarioFilesChanged": false,
+  "earlierMigrationFilesChanged": false,
+  "dependencyPinsChanged": false,
+  "hostedCiRunClaimed": false
+}
