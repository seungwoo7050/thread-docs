## `Record E05 ownership and browser protection verification`

diff --git a/evidence/E05/browser.json b/evidence/E05/browser.json
new file mode 100644
index 0000000..0ba2d78
--- /dev/null
+++ b/evidence/E05/browser.json
@@ -0,0 +1,16 @@
+{
+  "distinctBrowserSessions": 2,
+  "ownerCreates": 2,
+  "ownerChecks": 2,
+  "isolatedCollections": true,
+  "ownHistoryReads": 2,
+  "foreignReadsRejected": 6,
+  "csrfHeadersPresent": true,
+  "allowedOriginPresent": true,
+  "missingCsrfMutationStatus": 403,
+  "missingCsrfLogoutStatus": 403,
+  "noUnauthorizedChange": true,
+  "browserPreflightBlocked": true,
+  "authorizedLogoutStatus": 200,
+  "durationMs": 3048
+}
diff --git a/evidence/E05/migration.json b/evidence/E05/migration.json
new file mode 100644
index 0000000..20cde4a
--- /dev/null
+++ b/evidence/E05/migration.json
@@ -0,0 +1,15 @@
+{
+  "previousSchema": "001/002/003",
+  "refusedAbsentOwner": true,
+  "refusedUnknownOwner": true,
+  "refusalAtomic": true,
+  "priorStartupRefused": true,
+  "selectedOwner": "bob-e04",
+  "firstUserNotAssumed": true,
+  "preservedMonitors": 2,
+  "preservedChecks": 2,
+  "preservedOldHistory": true,
+  "onlyDesignatedOwnerCanRead": true,
+  "ownerDeletionRestricted": true,
+  "durationMs": 2280
+}
diff --git a/evidence/E05/ownership.json b/evidence/E05/ownership.json
new file mode 100644
index 0000000..e051d6f
--- /dev/null
+++ b/evidence/E05/ownership.json
@@ -0,0 +1,386 @@
+{
+  "observations": [
+    {
+      "operation": "0: owner collection/detail/history/direct CheckRun",
+      "status": 200
+    },
+    {
+      "operation": "0: foreign/absent detail",
+      "status": 404,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "0: foreign/absent history",
+      "status": 404,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "0: foreign/absent direct CheckRun",
+      "status": 404,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "0: foreign/absent update",
+      "status": 404,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "0: foreign/absent pause",
+      "status": 404,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "0: foreign/absent resume",
+      "status": 404,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "0: foreign/absent delete",
+      "status": 404,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "0: foreign/absent manual check",
+      "status": 404,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "1: owner collection/detail/history/direct CheckRun",
+      "status": 200
+    },
+    {
+      "operation": "1: foreign/absent detail",
+      "status": 404,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "1: foreign/absent history",
+      "status": 404,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "1: foreign/absent direct CheckRun",
+      "status": 404,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "1: foreign/absent update",
+      "status": 404,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "1: foreign/absent pause",
+      "status": 404,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "1: foreign/absent resume",
+      "status": 404,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "1: foreign/absent delete",
+      "status": 404,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "1: foreign/absent manual check",
+      "status": 404,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "create: missing CSRF",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "create: incorrect CSRF",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "create: other session CSRF",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "create: foreign Origin",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "create: missing Origin",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "create: null Origin",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "update: missing CSRF",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "update: incorrect CSRF",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "update: other session CSRF",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "update: foreign Origin",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "update: missing Origin",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "update: null Origin",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "pause: missing CSRF",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "pause: incorrect CSRF",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "pause: other session CSRF",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "pause: foreign Origin",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "pause: missing Origin",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "pause: null Origin",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "resume: missing CSRF",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "resume: incorrect CSRF",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "resume: other session CSRF",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "resume: foreign Origin",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "resume: missing Origin",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "resume: null Origin",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "delete: missing CSRF",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "delete: incorrect CSRF",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "delete: other session CSRF",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "delete: foreign Origin",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "delete: missing Origin",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "delete: null Origin",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "check: missing CSRF",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "check: incorrect CSRF",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "check: other session CSRF",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "check: foreign Origin",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "check: missing Origin",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "check: null Origin",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "logout: missing CSRF",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "logout: incorrect CSRF",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "logout: other session CSRF",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "logout: foreign Origin",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "logout: missing Origin",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "logout: null Origin",
+      "status": 403,
+      "noWrite": true,
+      "noOutbound": true
+    },
+    {
+      "operation": "owner update",
+      "status": 200
+    },
+    {
+      "operation": "owner pause",
+      "status": 200
+    },
+    {
+      "operation": "owner resume",
+      "status": 200
+    }
+  ],
+  "isolatedCollections": true,
+  "authoritativeOwnership": true,
+  "headForeignStatus": 404,
+  "ownerDeleteCascade": true,
+  "csrfRejections": 42,
+  "loginOriginRejections": 3,
+  "deniedPreflights": 2,
+  "originCheckedSeparatelyFromCsrf": true,
+  "corsPermissionAbsent": true,
+  "rotationChangesCsrf": true,
+  "oldCsrfRejectedByNewSession": true,
+  "logoutRevoked": true,
+  "csrfAloneUnauthenticated": true,
+  "durationMs": 2183
+}
diff --git a/evidence/E05/schema.json b/evidence/E05/schema.json
new file mode 100644
index 0000000..b2e4e07
--- /dev/null
+++ b/evidence/E05/schema.json
@@ -0,0 +1,25 @@
+{
+  "results": [
+    {
+      "mutation": "missing_owner",
+      "startupRejected": true,
+      "listening": false
+    },
+    {
+      "mutation": "nullable_owner",
+      "startupRejected": true,
+      "listening": false
+    },
+    {
+      "mutation": "missing_owner_fk",
+      "startupRejected": true,
+      "listening": false
+    },
+    {
+      "mutation": "wrong_owner_type",
+      "startupRejected": true,
+      "listening": false
+    }
+  ],
+  "count": 4
+}
diff --git a/evidence/E05/verification.json b/evidence/E05/verification.json
new file mode 100644
index 0000000..7cabde8
--- /dev/null
+++ b/evidence/E05/verification.json
@@ -0,0 +1,57 @@
+{
+  "thread": "E05",
+  "attempt": 1,
+  "specRevision": "0a006589477f8ae47bad3faa5510c999cff85ee4",
+  "start": "c1bace0a0e2de598f11fc9f1fac32d459fc910a6",
+  "scenarioSha256": "a429c0c70afa8365656d64e54d158fcb51fd9a5d8dd6905b6ea5d58db73ac95f",
+  "runtime": { "node": "24.19.0", "npm": "11.17.0", "newDependencies": [], "pinsChanged": false },
+  "baseline": {
+    "command": "fnm exec --using 24.19.0 node evidence/E05/baseline-reproducer.mjs",
+    "exitCode": 0,
+    "unchangedStart": true,
+    "foreignReadRequests": 1,
+    "observedStatus": 200,
+    "requiredStatus": 404,
+    "result": "REPRODUCED",
+    "durationMs": 1913
+  },
+  "commands": [
+    { "command": "fnm exec --using 24.19.0 npm run typecheck", "invocation": 1, "exitCode": 2, "result": "New test collection inference produced TS7022; fixed by one explicit owners-array type annotation. No product input or fixture changed.", "wallSeconds": 3.442 },
+    { "command": "fnm exec --using 24.19.0 npm run typecheck", "invocation": 2, "exitCode": 0, "wallSeconds": 1.356 },
+    { "command": "fnm exec --using 24.19.0 npm run test:unit", "exitCode": 0, "passed": 13, "failed": 0, "runnerDurationMs": 1376.181 },
+    { "command": "fnm exec --using 24.19.0 npm run test:functional", "exitCode": 0, "passed": 15, "failed": 0, "runnerDurationMs": 13707.859 },
+    { "command": "fnm exec --using 24.19.0 npm run test:integration", "exitCode": 0, "passed": 9, "failed": 0, "runnerDurationMs": 8330.129 },
+    { "command": "fnm exec --using 24.19.0 npm run test:e2e", "exitCode": 0, "browser": "pinned Chromium", "workers": 1, "passed": 7, "failed": 0, "runnerDurationSeconds": 16.9 },
+    { "command": "fnm exec --using 24.19.0 npm run build", "exitCode": 0, "wallSeconds": 5.864 },
+    { "command": "git diff --check; git diff --cached --check (separate invocations)", "exitCodes": [0, 0] }
+  ],
+  "acceptance": {
+    "twoUserFixtureUnchanged": true,
+    "ownerPredicatesOnAllMonitorAndCheckRunOperations": true,
+    "foreignAndAbsentResourcesIndistinguishable": true,
+    "foreignMutationAuthorityRowsAndOutboundCountsUnchanged": true,
+    "csrfOriginRejections": 42,
+    "loginOriginRejections": 3,
+    "corsPreflightsDenied": 2,
+    "credentialedCorsPermissionAbsent": true,
+    "logoutAndRotationPreserveRevocation": true,
+    "csrfAloneCannotAuthenticate": true,
+    "realAuthorizedBrowserLifecyclePreserved": true,
+    "freshRepeatedAndPrevious003MigrationsVerified": true,
+    "legacyMissingAndInvalidOwnerAtomicRefusal": true,
+    "explicitSecondUserAssignmentPreservesAllRows": true,
+    "ownerSchemaDriftRejections": 4,
+    "oldSchemaMigrationsAndEvidenceUnchanged": true
+  },
+  "budget": { "loadRuns": 0, "automaticRetries": 0, "parameterSweeps": 0, "browserRetries": 0, "baselineForeignReads": 1 },
+  "cleanup": {
+    "remainingE03E04E05TestSchemas": 0,
+    "listenersOn4311Through4314": 0,
+    "ownedRootPostgres15431LeftRunning": true,
+    "publicSchemaAndVolumeUntouched": true
+  },
+  "evidencePolicy": "Only safe status, operation, count, username and boolean evidence; no passwords, hashes, session identifiers, CSRF values, traces or videos.",
+  "hostedCiRunClaimed": false,
+  "outOfScopeChanges": [],
+  "unresolved": []
+}
