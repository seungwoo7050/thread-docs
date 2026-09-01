## `test(authz): preserve E05 security gates and observation history`

diff --git a/evidence/E05/browser-fixture-cleanup.json b/evidence/E05/browser-fixture-cleanup.json
new file mode 100644
index 0000000..f958b92
--- /dev/null
+++ b/evidence/E05/browser-fixture-cleanup.json
@@ -0,0 +1,5 @@
+{
+  "port": 4999,
+  "serverClosedAndAwaited": true,
+  "browserContextsClosedAndAwaited": true
+}
diff --git a/evidence/E05/browser-gate-outcomes.json b/evidence/E05/browser-gate-outcomes.json
new file mode 100644
index 0000000..f2dc799
--- /dev/null
+++ b/evidence/E05/browser-gate-outcomes.json
@@ -0,0 +1,67 @@
+[
+  {
+    "invocation": 1,
+    "source": "output/e05/full-verify-1-browser.json",
+    "stats": {
+      "startTime": "2026-08-28T01:33:03.354Z",
+      "duration": 22366.406,
+      "expected": 12,
+      "skipped": 0,
+      "unexpected": 1,
+      "flaky": 0
+    },
+    "observation": "Response wait remained pending; cleanup masked the original request failure. Later foreign-Origin/CORS/lifecycle assertions did not complete."
+  },
+  {
+    "invocation": 2,
+    "source": "output/e05/browser-verify-2-report.json",
+    "stats": {
+      "startTime": "2026-08-28T01:37:35.678Z",
+      "duration": 25729,
+      "expected": 12,
+      "skipped": 0,
+      "unexpected": 1,
+      "flaky": 0
+    },
+    "observation": "Safely settled observation reported net::ERR_FAILED from routed attacker document; no observable HTTP response. Earlier ownership/no-write/missing-CSRF checks completed; later CORS/lifecycle did not. No cause is claimed."
+  },
+  {
+    "invocation": 3,
+    "source": "output/e05/browser-verify-3-report.json",
+    "stats": {
+      "startTime": "2026-08-28T01:41:58.486Z",
+      "duration": 22213.913,
+      "expected": 12,
+      "skipped": 0,
+      "unexpected": 1,
+      "flaky": 0
+    },
+    "observation": "Real owned4999 fixture reached and asserted HTTP403; synchronous Playwright Origin metadata was undefined. Later CORS/lifecycle did not complete."
+  },
+  {
+    "invocation": 4,
+    "source": "output/e05/browser-verify-4-report.json",
+    "stats": {
+      "startTime": "2026-08-28T01:46:51.118Z",
+      "duration": 26778.172,
+      "expected": 12,
+      "skipped": 0,
+      "unexpected": 1,
+      "flaky": 0
+    },
+    "observation": "Real owned4999 fixture reached and asserted HTTP403; awaited Playwright Origin metadata was null. Later CORS/lifecycle did not complete."
+  },
+  {
+    "invocation": 5,
+    "source": "output/e05/browser-verify-5-report.json",
+    "stats": {
+      "startTime": "2026-08-28T01:51:08.217Z",
+      "duration": 27186.07,
+      "expected": 13,
+      "skipped": 0,
+      "unexpected": 0,
+      "flaky": 0
+    },
+    "observation": "All13 cases passed, including complete fixed two-context security and authorized lifecycle sequence. Actual document Origin4999 asserted; unavailable request Origin metadata recorded as null."
+  }
+]
diff --git a/evidence/E05/browser-isolation.json b/evidence/E05/browser-isolation.json
new file mode 100644
index 0000000..18cceb9
--- /dev/null
+++ b/evidence/E05/browser-isolation.json
@@ -0,0 +1,145 @@
+{
+  "browserContexts": 2,
+  "fixedDataset": "Alice A /ok60 and Bob B /fail120, one initial CheckRun each",
+  "ownerReadsAndLifecycle": true,
+  "foreignReadsAndWrites404": true,
+  "deniedWritesKeepRowsAndHistory": true,
+  "missingCsrf403": true,
+  "realForeignOriginWrite403": true,
+  "credentialedCrossOriginReadBlocked": true,
+  "foreignOriginFetchResult": "opaque",
+  "attackerDocumentOrigin": "http://127.0.0.1:4999",
+  "foreignRequestOriginMetadata": null,
+  "trustedBrowserDocumentOriginsVerified": true,
+  "requestOriginMetadataUnavailable": 0,
+  "requestOriginMetadataReadFailed": false,
+  "logout401": true,
+  "observedBrowserMutationEvidence": [
+    {
+      "path": "/api/session/login",
+      "method": "POST",
+      "origin": "http://127.0.0.1:4323",
+      "csrfPresent": true
+    },
+    {
+      "path": "/api/session/login",
+      "method": "POST",
+      "origin": "http://127.0.0.1:4323",
+      "csrfPresent": true
+    },
+    {
+      "path": "/api/monitors",
+      "method": "POST",
+      "origin": "http://127.0.0.1:4323",
+      "csrfPresent": true
+    },
+    {
+      "path": "/api/monitors/93a18387-2e59-4ef3-9291-3318f818e960/checks",
+      "method": "POST",
+      "origin": "http://127.0.0.1:4323",
+      "csrfPresent": true
+    },
+    {
+      "path": "/api/monitors",
+      "method": "POST",
+      "origin": "http://127.0.0.1:4323",
+      "csrfPresent": true
+    },
+    {
+      "path": "/api/monitors/6596e8ae-159f-4428-9951-efbf1f2a5ac5/checks",
+      "method": "POST",
+      "origin": "http://127.0.0.1:4323",
+      "csrfPresent": true
+    },
+    {
+      "path": "/api/monitors/93a18387-2e59-4ef3-9291-3318f818e960",
+      "method": "PUT",
+      "origin": "http://127.0.0.1:4323",
+      "csrfPresent": true
+    },
+    {
+      "path": "/api/monitors/93a18387-2e59-4ef3-9291-3318f818e960",
+      "method": "PUT",
+      "origin": "http://127.0.0.1:4323",
+      "csrfPresent": true
+    },
+    {
+      "path": "/api/monitors/93a18387-2e59-4ef3-9291-3318f818e960",
+      "method": "PUT",
+      "origin": "http://127.0.0.1:4323",
+      "csrfPresent": true
+    },
+    {
+      "path": "/api/monitors/93a18387-2e59-4ef3-9291-3318f818e960",
+      "method": "DELETE",
+      "origin": "http://127.0.0.1:4323",
+      "csrfPresent": true
+    },
+    {
+      "path": "/api/monitors/93a18387-2e59-4ef3-9291-3318f818e960/checks",
+      "method": "POST",
+      "origin": "http://127.0.0.1:4323",
+      "csrfPresent": true
+    },
+    {
+      "path": "/api/monitors/93a18387-2e59-4ef3-9291-3318f818e960/checks",
+      "method": "POST",
+      "origin": "http://127.0.0.1:4323",
+      "csrfPresent": false
+    },
+    {
+      "path": "/api/monitors/93a18387-2e59-4ef3-9291-3318f818e960",
+      "method": "PUT",
+      "origin": "http://127.0.0.1:4323",
+      "csrfPresent": true
+    },
+    {
+      "path": "/api/monitors/93a18387-2e59-4ef3-9291-3318f818e960",
+      "method": "PUT",
+      "origin": "http://127.0.0.1:4323",
+      "csrfPresent": true
+    },
+    {
+      "path": "/api/monitors/93a18387-2e59-4ef3-9291-3318f818e960",
+      "method": "PUT",
+      "origin": "http://127.0.0.1:4323",
+      "csrfPresent": true
+    },
+    {
+      "path": "/api/monitors/93a18387-2e59-4ef3-9291-3318f818e960/checks",
+      "method": "POST",
+      "origin": "http://127.0.0.1:4323",
+      "csrfPresent": true
+    },
+    {
+      "path": "/api/monitors/93a18387-2e59-4ef3-9291-3318f818e960",
+      "method": "DELETE",
+      "origin": "http://127.0.0.1:4323",
+      "csrfPresent": true
+    },
+    {
+      "path": "/api/monitors/6596e8ae-159f-4428-9951-efbf1f2a5ac5",
+      "method": "DELETE",
+      "origin": "http://127.0.0.1:4323",
+      "csrfPresent": true
+    },
+    {
+      "path": "/api/session/logout",
+      "method": "POST",
+      "origin": "http://127.0.0.1:4323",
+      "csrfPresent": true
+    },
+    {
+      "path": "/api/session/logout",
+      "method": "POST",
+      "origin": "http://127.0.0.1:4323",
+      "csrfPresent": true
+    }
+  ],
+  "attackerDocument": "owned HTTP fixture http://127.0.0.1:4999/attack; occupied ports refused",
+  "credentialsRecorded": false,
+  "traces": false,
+  "screenshots": false,
+  "attackerPort4999ClosedAndAwaited": true,
+  "browserContextsClosedAndAwaited": true
+}
diff --git a/evidence/E05/browser-mutation.json b/evidence/E05/browser-mutation.json
new file mode 100644
index 0000000..ebefdbe
--- /dev/null
+++ b/evidence/E05/browser-mutation.json
@@ -0,0 +1,303 @@
+{
+  "csrfAloneDoesNotAuthenticate" : true,
+  "authorizedWrites" : [ "create", "edit", "pause", "resume", "check", "delete", "logout" ],
+  "runtimeFilterOrderVerified" : true,
+  "deniedWrites" : [ {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "missing-csrf",
+    "mutation" : "create"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "incorrect-csrf",
+    "mutation" : "create"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "other-session-csrf",
+    "mutation" : "create"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "missing-origin",
+    "mutation" : "create"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "null-origin",
+    "mutation" : "create"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "foreign-origin",
+    "mutation" : "create"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "missing-csrf",
+    "mutation" : "edit"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "incorrect-csrf",
+    "mutation" : "edit"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "other-session-csrf",
+    "mutation" : "edit"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "missing-origin",
+    "mutation" : "edit"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "null-origin",
+    "mutation" : "edit"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "foreign-origin",
+    "mutation" : "edit"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "missing-csrf",
+    "mutation" : "pause"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "incorrect-csrf",
+    "mutation" : "pause"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "other-session-csrf",
+    "mutation" : "pause"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "missing-origin",
+    "mutation" : "pause"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "null-origin",
+    "mutation" : "pause"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "foreign-origin",
+    "mutation" : "pause"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "missing-csrf",
+    "mutation" : "resume"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "incorrect-csrf",
+    "mutation" : "resume"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "other-session-csrf",
+    "mutation" : "resume"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "missing-origin",
+    "mutation" : "resume"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "null-origin",
+    "mutation" : "resume"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "foreign-origin",
+    "mutation" : "resume"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "missing-csrf",
+    "mutation" : "check"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "incorrect-csrf",
+    "mutation" : "check"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "other-session-csrf",
+    "mutation" : "check"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "missing-origin",
+    "mutation" : "check"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "null-origin",
+    "mutation" : "check"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "foreign-origin",
+    "mutation" : "check"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "missing-csrf",
+    "mutation" : "delete"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "incorrect-csrf",
+    "mutation" : "delete"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "other-session-csrf",
+    "mutation" : "delete"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "missing-origin",
+    "mutation" : "delete"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "null-origin",
+    "mutation" : "delete"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "foreign-origin",
+    "mutation" : "delete"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "missing-csrf",
+    "mutation" : "logout"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "incorrect-csrf",
+    "mutation" : "logout"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "other-session-csrf",
+    "mutation" : "logout"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "missing-origin",
+    "mutation" : "logout"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "null-origin",
+    "mutation" : "logout"
+  }, {
+    "status" : 403,
+    "allRowsUnchanged" : true,
+    "sessionRetained" : true,
+    "outboundUnchanged" : true,
+    "invalidEvidence" : "foreign-origin",
+    "mutation" : "logout"
+  } ],
+  "anonymousCsrfLogout401" : true,
+  "defaultSessionCsrfRepository" : true,
+  "oldCookieRevoked" : true
+}
diff --git a/evidence/E05/full-verify-1-commands.json b/evidence/E05/full-verify-1-commands.json
new file mode 100644
index 0000000..fb0346a
--- /dev/null
+++ b/evidence/E05/full-verify-1-commands.json
@@ -0,0 +1,62 @@
+[
+  {
+    "command": "node scripts/database.mjs up",
+    "startedAt": "2026-08-28T01:32:19.253Z",
+    "elapsedSeconds": 0.872,
+    "exitCode": 0
+  },
+  {
+    "command": "mvn -B -ntp -f backend/pom.xml package",
+    "startedAt": "2026-08-28T01:32:20.125Z",
+    "elapsedSeconds": 21.888,
+    "exitCode": 0
+  },
+  {
+    "command": "npm run typecheck",
+    "startedAt": "2026-08-28T01:32:42.014Z",
+    "elapsedSeconds": 1.343,
+    "exitCode": 0
+  },
+  {
+    "command": "npm run build",
+    "startedAt": "2026-08-28T01:32:43.357Z",
+    "elapsedSeconds": 7.93,
+    "exitCode": 0
+  },
+  {
+    "command": "node scripts/persistence-isolation.mjs",
+    "startedAt": "2026-08-28T01:32:51.288Z",
+    "elapsedSeconds": 0.108,
+    "exitCode": 0
+  },
+  {
+    "command": "node scripts/database.mjs reset e04_restart",
+    "startedAt": "2026-08-28T01:32:51.396Z",
+    "elapsedSeconds": 0.261,
+    "exitCode": 0
+  },
+  {
+    "command": "node scripts/persistence-scenario.mjs fixed",
+    "startedAt": "2026-08-28T01:32:51.657Z",
+    "elapsedSeconds": 11.241,
+    "exitCode": 0
+  },
+  {
+    "command": "npm run test:e2e",
+    "startedAt": "2026-08-28T01:33:02.898Z",
+    "elapsedSeconds": 22.846,
+    "exitCode": 1
+  },
+  {
+    "command": "node scripts/database.mjs drop e04_restart",
+    "startedAt": "2026-08-28T01:33:25.746Z",
+    "elapsedSeconds": 0.29,
+    "exitCode": 0
+  },
+  {
+    "command": "node scripts/database.mjs drop e04_browser",
+    "startedAt": "2026-08-28T01:33:26.037Z",
+    "elapsedSeconds": 0.166,
+    "exitCode": 0
+  }
+]
diff --git a/evidence/E05/login-cors.json b/evidence/E05/login-cors.json
new file mode 100644
index 0000000..fa599cc
--- /dev/null
+++ b/evidence/E05/login-cors.json
@@ -0,0 +1,25 @@
+{
+  "validLoginRotates" : true,
+  "cors" : [ {
+    "allowOriginHeader" : false,
+    "authenticatedReadStatus" : 200,
+    "allowCredentialsHeader" : false,
+    "origin" : "http://127.0.0.1:4323",
+    "preflightStatus" : 403
+  }, {
+    "allowOriginHeader" : false,
+    "authenticatedReadStatus" : 200,
+    "allowCredentialsHeader" : false,
+    "origin" : "http://127.0.0.1:4999",
+    "preflightStatus" : 403
+  }, {
+    "allowOriginHeader" : false,
+    "authenticatedReadStatus" : 200,
+    "allowCredentialsHeader" : false,
+    "origin" : "null",
+    "preflightStatus" : 403
+  } ],
+  "invalidAnonymousLoginEvidenceStatus" : 401,
+  "anonymousSessionNotRotatedOnDenial" : true,
+  "invalidAuthenticatedLoginEvidenceStatus" : 403
+}
diff --git a/evidence/E05/verification-runs.jsonl b/evidence/E05/verification-runs.jsonl
new file mode 100644
index 0000000..209e116
--- /dev/null
+++ b/evidence/E05/verification-runs.jsonl
@@ -0,0 +1,9 @@
+{"command":"mvn -B -ntp -f backend/pom.xml -Dtest=OwnershipMigrationTest,OwnershipAuthorizationTest,PostgresPersistenceTest,UserAccountPersistenceTest,ApiErrorBoundaryTest test","startedAt":"2026-08-28T01:22:52.588Z","elapsedSeconds":17.34,"exitCode":0,"log":"output/e05/ownership-focused.log"}
+{"command":"mvn -B -ntp -f backend/pom.xml -Dtest=OwnershipAuthorizationTest,SessionAuthenticationTest test","startedAt":"2026-08-28T01:27:05.317Z","elapsedSeconds":15.559,"exitCode":0,"log":"output/e05/browser-boundary-focused.log"}
+{"command":"npm run verify","startedAt":"2026-08-28T01:32:19.036Z","elapsedSeconds":67.178,"exitCode":1,"log":"output/e05/full-verify-1.log"}
+{"command":"npm run test:e2e","startedAt":"2026-08-28T01:37:34.753Z","elapsedSeconds":26.684,"exitCode":1,"log":"output/e05/browser-verify-2.log"}
+{"command":"npm run test:e2e","startedAt":"2026-08-28T01:41:57.945Z","elapsedSeconds":22.778,"exitCode":1,"log":"output/e05/browser-verify-3.log"}
+{"command":"npm run test:e2e","startedAt":"2026-08-28T01:46:50.497Z","elapsedSeconds":27.425,"exitCode":1,"log":"output/e05/browser-verify-4.log"}
+{"command":"npm run test:e2e","startedAt":"2026-08-28T01:51:07.660Z","elapsedSeconds":27.774,"exitCode":0,"log":"output/e05/browser-verify-5.log"}
+{"command":"npm run typecheck","startedAt":"2026-08-28T01:52:03.440Z","elapsedSeconds":1.347,"exitCode":0,"log":"output/e05/final-typecheck.log"}
+{"command":"node scripts/database.mjs drop e04_browser","kind":"cleanup","startedAt":"2026-08-28T01:53:02.054Z","elapsedSeconds":0.3,"exitCode":0,"log":"output/e05/final-schema-cleanup.log"}
diff --git a/evidence/E05/verification.md b/evidence/E05/verification.md
new file mode 100644
index 0000000..df5996a
--- /dev/null
+++ b/evidence/E05/verification.md
@@ -0,0 +1,110 @@
+# E05 verification — attempt 1
+
+START: `c59551aeb8e5a351b7d875489578f31caea6f160`  
+Spec: `0a006589477f8ae47bad3faa5510c999cff85ee4`  
+Frozen fixture SHA-256: `03cd149112e7bed3613c2dab85c6f5c46d82e5037bef1f638abb0111f40e237d`
+
+The unchanged START baseline ran exactly once: Bob read Alice's Monitor A and
+received 200 with Alice's identifier. The required result was 404 / NOT_FOUND.
+The fixed two-user, two-Monitor, two-CheckRun dataset was not changed.
+
+## Invocation ledger
+
+Exact commands, start times, durations and exit codes are in
+`verification-runs.jsonl`; the default full command's nested gates are in
+`full-verify-1-commands.json`. `browser-gate-outcomes.json` preserves all five
+browser invocations, including their original reporter statistics.
+
+| Invocation | Result | Seconds |
+| --- | --- | ---: |
+| Focused ownership/migration/persistence/error tests | 12 Java tests passed | 17.340 |
+| Focused browser-security/session tests | 6 Java tests passed | 15.559 |
+| Default `npm run verify` | Java36, typecheck, production build, occupied-port guard and real restart/lifecycle passed; browser12 passed, new case failed | 67.178 |
+| Browser gate 2 | Browser12 passed, new case failed during observation | 26.684 |
+| Browser gate 3 | Browser12 passed, new case failed during Origin metadata audit | 22.778 |
+| Browser gate 4 | Browser12 passed, new case failed during Origin metadata audit | 27.425 |
+| Browser gate 5 | All13 passed, including the complete new two-context case | 27.774 |
+| Final `npm run typecheck` | Passed after the final test-only edits | 1.347 |
+| Explicit disposable `e04_browser` cleanup | Passed | 0.300 |
+
+The default full command exited 1; it is not represented as a passing invocation.
+Its successful backend/build/restart gates were not repeated for subsequent
+test-only observation changes. The final browser and static-type gates passed.
+The parent will independently verify the final candidate.
+
+One inline evidence-preparation helper failed to parse before any I/O or test
+execution. Its corrected copy operation succeeded; it did not rerun a gate or
+change production data.
+
+## Failed browser observations and bounded corrections
+
+1. The original response waiter remained pending when the routed attacker fetch
+   failed, and cleanup masked the original failure. Earlier ownership,
+   unchanged-row and missing-CSRF assertions completed; later assertions did not.
+2. A safely settled response-or-failure observer exposed `net::ERR_FAILED` from
+   the routed `http://127.0.0.1:4999` document, without an observable HTTP response.
+   No cause is claimed for that transport result.
+3. With the parent-approved exclusively owned HTTP document server on 4999,
+   the unchanged foreign POST reached an observed 403. Playwright's synchronous
+   Origin metadata was unavailable (`undefined`), stopping the added audit.
+4. The awaited Origin accessor also returned unavailable metadata (`null`) after
+   an observed 403. Later CORS and lifecycle assertions still had not completed.
+5. Per the parent's observation clarification, the browser test asserts the
+   actual attacker document's `location.origin` is exactly
+   `http://127.0.0.1:4999`. It retains the original path, POST, empty body,
+   `mode: 'no-cors'`, `credentials: 'include'`, observed 403, unchanged history
+   and no CORS-grant assertions. Unavailable request-header metadata is recorded
+   as null, not turned into an extra acceptance criterion. The separate real-HTTP
+   Java matrix still explicitly sends Origin4999 and requires 403 with no writes
+   or outbound requests. No browser flags, permissions or product code changed
+   during these observation corrections; no CDP observer was added.
+
+`fixtures.md` remained unchanged. The only added fixture transport is the owned
+4999 document server: occupied ports are refused, other processes are never
+reused or stopped, and server/context closure is awaited in `finally`.
+
+## Passing evidence
+
+- `authorization.json` and `owner-sql.txt`: every resource read and mutation is
+  owner-scoped, foreign/nonexistent404 responses match, 20 foreign/nonexistent
+  mutation denials leave all authoritative Monitor/CheckRun rows and outbound
+  counts unchanged. Generated SQL and actual proxy transaction flags were checked;
+  outbound HTTP holds no store transaction.
+- `migration.json`: fresh V4, repeated startup, previous-V3 atomic refusal,
+  unknown/unassigned owner rejection and explicit verified Bob assignment preserve
+  historical data. Missing/nullable owner columns and missing owner FK refuse
+  startup. V1/V2/V3 and previous evidence remain unchanged.
+- `browser-mutation.json`: seven mutation types times six invalid-evidence cases
+  produce42 authenticated403 denials with unchanged authority/outbound/session
+  state, followed by authorized success. Default session CSRF, actual filter
+  ordering, revoked-cookie401, CSRF-alone401 and anonymous-CSRF/logout401 pass.
+  The unchanged `SessionAuthenticationTest` separately retains the exact one-hour
+  expiry, rotation and revocation regressions.
+- `login-cors.json`: pre-authentication login evidence policy and rotation,
+  authenticated invalid-evidence403, and independent denied credentialed CORS for
+  trusted, foreign4999 and null Origins.
+- `browser-isolation.json`: two real browser sessions complete isolation, owner
+  CRUD/check/history, missing-CSRF403, actual foreign-document POST403, blocked
+  cross-origin credentialed read and both logout401 checks. The foreign request
+  Origin metadata is explicitly null. All20 same-origin mutation observations
+  exposed Origin4323;19 had CSRF and the intentionally denied one did not.
+
+## Cleanup, toolchain and budget
+
+Final type validation passed. After the standalone browser gate, the remaining
+disposable `e04_browser` schema was explicitly removed by the existing allowlisted
+cleanup command. A subsequent metadata query showed zero `e04_*`/`e05_*` schemas
+and the public schema present. The owned `wse-industry` PostgreSQL remains healthy
+on15432 for the parent's verification; `wse-industry_postgres-data` was preserved.
+The final listener inspection found4321–4325 and4999 free. The owned4999 HTTP server
+and both browser contexts were closed and awaited (`browser-fixture-cleanup.json`).
+
+Node24.19.0 / npm11.17.0 / Java21.0.7+6 / Maven3.9.11; dependencies unchanged.
+No credentials, password hashes, session cookies or CSRF values are saved in these
+artifacts. Browser traces, screenshots, videos and storage-state capture remain off.
+
+Budget: one unchanged-START baseline; two focused Java invocations; one default
+full verification; four subsequent browser-only invocations; one final static
+typecheck. Zero load runs, automatic retries, parameter sweeps or formal repair
+tasks. All observation corrections stayed within original E05 attempt1 and were
+approved by the parent. No E06+ work was performed. Stop after this report.
