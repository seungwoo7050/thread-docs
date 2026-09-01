## `test: verify E10 identity and competing worker boundaries`

diff --git a/.github/workflows/check.yml b/.github/workflows/check.yml
index b220a58..a421129 100644
--- a/.github/workflows/check.yml
+++ b/.github/workflows/check.yml
@@ -33,6 +33,8 @@ jobs:
         run: npm run test:integration
       - name: Functional
         run: npm run test:functional
+      - name: Request identity and competing workers
+        run: npm run test:ownership
       - name: Install pinned Chromium
         run: npx playwright install --with-deps chromium
       - name: Production build and browser E2E
diff --git a/TRACK.md b/TRACK.md
index 849b708..6127dcd 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -84,6 +84,7 @@ npm run test:integration
 npx playwright install chromium
 npm run test:e2e
 npm run test:execution
+npm run test:ownership
 ```
 
 Functional tests own fixed loopback ports 4311, 4312 and 4314; stop manual fixture
@@ -572,3 +573,47 @@ artifact for the fixed held-endpoint Chromium flow and exact scheduler ticks.
 It owns only `e09_execution`/`e09_scheduler`, refuses occupied reserved ports, and
 awaits owned process/schema cleanup. Safe results and actual invocation counts
 are in `evidence/E09`; no credential-bearing browser artifacts are retained.
+
+## E10 manual intent and execution ownership
+
+Manual Check POSTs require `Idempotency-Key`: 1–128 printable ASCII characters
+without whitespace. Missing or invalid keys return `400 INVALID_INPUT`.
+Identity is the authenticated user ID plus that exact key. Reusing it for the
+same canonical Monitor ID returns **202** with the same CheckRun ID and its
+current state; a different owned Monitor returns `409 CONFLICT`. Monitor name,
+URL, interval and enabled state are not part of that request's meaning.
+
+Migration `007_check_ownership.sql` adds internal request-user/key and worker-ID
+columns. PostgreSQL uniqueness, not a process cache, arbitrates parallel requests.
+One explicit transaction holds the owned parent and uses
+[ON CONFLICT](https://www.postgresql.org/docs/17/sql-insert.html#SQL-ON-CONFLICT);
+a following READ COMMITTED statement reads the winner after it commits.
+The existing Monitor deletion cascade removes its CheckRuns and their identities;
+this Thread adds no independent tombstone or expiration policy. Old rows retain
+null identity/worker fields rather than invented historical ownership. Stop old
+API/worker processes before applying the migration. Public CheckRun fields,
+terminal history and all earlier migration files are unchanged.
+
+Each worker process has one UUID. In one transaction,
+[FOR UPDATE SKIP LOCKED](https://www.postgresql.org/docs/17/sql-select.html#SQL-FOR-UPDATE-SHARE)
+selects a queued row and stores its RUNNING state, start time and owner.
+Only after commit does it perform the outbound request. Completion updates only
+that ID with the matching recorded owner and RUNNING state. A competing worker
+cannot overwrite the result. Normal scheduling retains E09 due-slot identity.
+There is no lease or crash recovery in this Thread.
+
+The browser creates an identity in its check event handler, keeps it until the
+request is acknowledged, and reuses it when the user explicitly repeats an
+unacknowledged request. A successful acknowledgement makes the next click a new
+intention.
+Delete and session cleanup discard pending identities; none are sent as SSR
+props or put in browser storage or logs. Automatic retries remain disabled.
+
+`npm run test:ownership` uses the frozen `evidence/phase-1/E10` dataset and real
+PostgreSQL: the parallel same-key pair, changed-target conflict, API restart,
+mutable Monitor fields, key validation, and two actual CLI workers blocked at
+one database barrier. The fixture holds the winner's headers while the losing
+worker ID is refused completion. Production Chromium additionally holds a real
+202 after commit, loses that delivery, then verifies manual retransmission and
+a new acknowledged intention. The frozen files and prior evidence are unchanged;
+live regression helpers supply a fresh key for each prior intentional Check POST.
diff --git a/evidence/phase-1/E10/baseline.json b/evidence/phase-1/E10/baseline.json
new file mode 100644
index 0000000..6444cba
--- /dev/null
+++ b/evidence/phase-1/E10/baseline.json
@@ -0,0 +1,48 @@
+{
+  "start": "d42b8767e90bfc495800cefccce807f1edc3c588",
+  "baselineHead": "bf7a3d8a687daba5b9cc730587a8d85b5deef91a",
+  "productMatchesStart": true,
+  "hashes": {
+    "scenario.json": "2ee0cab740c72774372ecf06f669d2b05843944b905877598f92ee013a125c70",
+    "fixture.ts": "8038b62ea5c1188661ffe63b99f25ea3eb27826aa7560ff32306643e8f7a1705",
+    "baseline.mjs": "afd2bbad96dedd148c3c9e769679f0241e4c2732a653d938f35257d2fcbfcb9e"
+  },
+  "budget": {
+    "loadRuns": 0,
+    "automaticRetries": 0,
+    "parameterSweeps": 0,
+    "unchangedStartBaselines": 1
+  },
+  "result": "REPRODUCED",
+  "observation": {
+    "barrierArrivals": 2,
+    "statuses": [
+      202,
+      202
+    ],
+    "returnedIds": [
+      "281924f6-2325-4b5c-bdd6-dca0dadc8644",
+      "4e64ffd4-d0f1-4b15-9d33-d5480e7e415a"
+    ],
+    "rows": [
+      {
+        "id": "281924f6-2325-4b5c-bdd6-dca0dadc8644",
+        "monitor_id": "10000000-0000-4000-9000-000000000001",
+        "state": "QUEUED"
+      },
+      {
+        "id": "4e64ffd4-d0f1-4b15-9d33-d5480e7e415a",
+        "monitor_id": "10000000-0000-4000-9000-000000000001",
+        "state": "QUEUED"
+      }
+    ],
+    "distinctIds": 2,
+    "outboundCalls": 0
+  },
+  "decisiveFailure": "Same owner and Idempotency-Key produced two durable QUEUED IDs instead of one intent.",
+  "cleanup": {
+    "schemaDropped": true,
+    "portsFree": true
+  },
+  "durationMs": 708
+}
diff --git a/evidence/phase-1/E10/browser.json b/evidence/phase-1/E10/browser.json
new file mode 100644
index 0000000..8e1979f
--- /dev/null
+++ b/evidence/phase-1/E10/browser.json
@@ -0,0 +1,23 @@
+{
+  "result": "PASS",
+  "productionBuild": true,
+  "observation": {
+    "submittedActions": 4,
+    "forwardedPosts": 3,
+    "sameKeyOnRetransmission": true,
+    "newKeyAfterAcknowledgement": true,
+    "returnedIds": [
+      "43792762-5296-4aef-b189-1c9ce3946963",
+      "43792762-5296-4aef-b189-1c9ce3946963",
+      "81069487-9f29-4d7e-ae1d-b1f420422afc"
+    ],
+    "intentRows": 2,
+    "failedDeliveries": 1,
+    "actualWorkerCompletedIds": [
+      "43792762-5296-4aef-b189-1c9ce3946963",
+      "81069487-9f29-4d7e-ae1d-b1f420422afc"
+    ],
+    "automaticRetries": 0
+  },
+  "durationMs": 2189
+}
diff --git a/evidence/phase-1/E10/claim.json b/evidence/phase-1/E10/claim.json
new file mode 100644
index 0000000..e453782
--- /dev/null
+++ b/evidence/phase-1/E10/claim.json
@@ -0,0 +1,65 @@
+{
+  "result": "PASS",
+  "barrier": {
+    "workers": [
+      {
+        "pid": 57310,
+        "workerId": "63eb7d08-b252-4206-86a8-98a973265f51"
+      },
+      {
+        "pid": 57317,
+        "workerId": "379fa0ff-0bd9-487d-957d-9d6247e521d1"
+      }
+    ],
+    "blocked": [
+      {
+        "pid": 7815,
+        "application_name": "e10_worker_1"
+      },
+      {
+        "pid": 7816,
+        "application_name": "e10_worker_2"
+      }
+    ],
+    "outboundBeforeRelease": 0
+  },
+  "held": {
+    "id": "10000000-0000-4000-a000-000000000001",
+    "state": "RUNNING",
+    "owner": "63eb7d08-b252-4206-86a8-98a973265f51",
+    "outbound": 1,
+    "loserWorker": "379fa0ff-0bd9-487d-957d-9d6247e521d1",
+    "loserCompletionWrites": 0
+  },
+  "terminal": {
+    "id": "10000000-0000-4000-a000-000000000001",
+    "monitorId": "10000000-0000-4000-9000-000000000001",
+    "trigger": "MANUAL",
+    "state": "SUCCEEDED",
+    "httpStatus": 200,
+    "latencyMs": 11,
+    "failureReason": null,
+    "startedAt": "2026-08-28T04:04:45.338Z",
+    "finishedAt": "2026-08-28T04:04:45.353Z",
+    "ownerUnchanged": true,
+    "outbound": 1,
+    "rows": 1
+  },
+  "workerStops": [
+    {
+      "code": null,
+      "signal": "SIGTERM",
+      "forced": false
+    },
+    {
+      "code": null,
+      "signal": "SIGTERM",
+      "forced": false
+    }
+  ],
+  "cleanup": {
+    "schemaDropped": true,
+    "portsFree": true
+  },
+  "durationMs": 631
+}
diff --git a/evidence/phase-1/E10/cleanup.json b/evidence/phase-1/E10/cleanup.json
new file mode 100644
index 0000000..9d81395
--- /dev/null
+++ b/evidence/phase-1/E10/cleanup.json
@@ -0,0 +1,15 @@
+{
+  "portsFree": [
+    4311,
+    4312,
+    4313,
+    4314
+  ],
+  "residualTestSchemas": [],
+  "ownedProcessesAwaited": true,
+  "e10WorkerForcedKills": 0,
+  "e09WorkerAndNextForcedKills": 0,
+  "browserWebservers": "Playwright-managed teardown; reserved ports confirmed free",
+  "publicAndVolumesNotModified": true,
+  "postgresLeftRunning": true
+}
diff --git a/evidence/phase-1/E10/e09-execution.json b/evidence/phase-1/E10/e09-execution.json
new file mode 100644
index 0000000..bc2592f
--- /dev/null
+++ b/evidence/phase-1/E10/e09-execution.json
@@ -0,0 +1,61 @@
+{
+  "result": "PASS",
+  "productionBuild": true,
+  "states": [
+    "QUEUED",
+    "RUNNING",
+    "SUCCEEDED"
+  ],
+  "beforeWorker": {
+    "status": 202,
+    "id": "393ae919-35ef-4fce-b77c-e4925b341b88",
+    "state": "QUEUED",
+    "http_status": null,
+    "started_at": null,
+    "finished_at": null,
+    "outboundCalls": 0,
+    "visibleState": "QUEUED"
+  },
+  "whileHeld": {
+    "apiPid": 61868,
+    "workerPid": 61874,
+    "sameId": true,
+    "state": "RUNNING",
+    "httpStatus": null,
+    "finishedAt": null,
+    "outboundCalls": 1,
+    "browserUsable": true
+  },
+  "afterRelease": {
+    "id": "393ae919-35ef-4fce-b77c-e4925b341b88",
+    "monitorId": "09000000-0000-4000-9000-000000000001",
+    "trigger": "MANUAL",
+    "state": "SUCCEEDED",
+    "httpStatus": 200,
+    "latencyMs": 233,
+    "failureReason": null,
+    "startedAt": "2026-08-28T04:14:27.369Z",
+    "finishedAt": "2026-08-28T04:14:27.605Z",
+    "outboundCalls": 1,
+    "historyCount": 1
+  },
+  "errors": {
+    "page": 0,
+    "console": 0
+  },
+  "workerStop": {
+    "code": null,
+    "signal": "SIGTERM",
+    "forced": false
+  },
+  "webStop": {
+    "code": 143,
+    "signal": null,
+    "forced": false
+  },
+  "cleanup": {
+    "schemaDropped": true,
+    "portsFree": true
+  },
+  "durationMs": 2206
+}
diff --git a/evidence/phase-1/E10/e09-scheduler.json b/evidence/phase-1/E10/e09-scheduler.json
new file mode 100644
index 0000000..34e6345
--- /dev/null
+++ b/evidence/phase-1/E10/e09-scheduler.json
@@ -0,0 +1,32 @@
+{
+  "result": "PASS",
+  "workerStopped": true,
+  "counts": [
+    {
+      "tick": "2026-08-28T00:00:00.000Z",
+      "a": 1,
+      "b": 0,
+      "slots": [
+        "2026-08-28T00:00:00.000Z"
+      ]
+    },
+    {
+      "tick": "2026-08-28T00:00:00.000Z",
+      "a": 1,
+      "b": 0,
+      "slots": [
+        "2026-08-28T00:00:00.000Z"
+      ]
+    },
+    {
+      "tick": "2026-08-28T00:01:00.000Z",
+      "a": 2,
+      "b": 0,
+      "slots": [
+        "2026-08-28T00:00:00.000Z",
+        "2026-08-28T00:01:00.000Z"
+      ]
+    }
+  ],
+  "durationMs": 330
+}
diff --git a/evidence/phase-1/E10/identity.json b/evidence/phase-1/E10/identity.json
new file mode 100644
index 0000000..c28b0f1
--- /dev/null
+++ b/evidence/phase-1/E10/identity.json
@@ -0,0 +1,49 @@
+{
+  "result": "PASS",
+  "parallel": {
+    "arrivals": 2,
+    "statuses": [
+      202,
+      202
+    ],
+    "returnedIds": [
+      "eb9b8af4-b192-4fcc-9658-ad53de07ab30",
+      "eb9b8af4-b192-4fcc-9658-ad53de07ab30"
+    ],
+    "intentRows": 1,
+    "outbound": 0
+  },
+  "conflict": {
+    "status": 409,
+    "code": "CONFLICT",
+    "bRows": 0,
+    "bOutbound": 0
+  },
+  "restart": {
+    "retainedIdentity": true,
+    "mutableMonitorFieldsExcluded": true
+  },
+  "final": {
+    "distinctIntentIds": [
+      "eb9b8af4-b192-4fcc-9658-ad53de07ab30",
+      "36212072-5602-4080-b642-9e18f3126791"
+    ],
+    "rows": 2,
+    "invalidIdentityCases": 6,
+    "workerPid": 57259,
+    "state": "SUCCEEDED",
+    "httpStatus": 200,
+    "okOutbound": 2,
+    "failOutbound": 0
+  },
+  "workerStop": {
+    "code": null,
+    "signal": "SIGTERM",
+    "forced": false
+  },
+  "cleanup": {
+    "schemaDropped": true,
+    "portsFree": true
+  },
+  "durationMs": 1162
+}
diff --git a/evidence/phase-1/E10/verification.json b/evidence/phase-1/E10/verification.json
new file mode 100644
index 0000000..05f7c82
--- /dev/null
+++ b/evidence/phase-1/E10/verification.json
@@ -0,0 +1,134 @@
+{
+  "thread": "E10",
+  "profile": "phase-1",
+  "attempt": 1,
+  "branch": "track/fundamentals-fastify",
+  "specRevision": "2ada57a71cd34fa2fae9809415c362a8bbfcdf02",
+  "start": "d42b8767e90bfc495800cefccce807f1edc3c588",
+  "result": "PASS",
+  "runtime": { "node": "24.19.0", "npm": "11.17.0", "next": "16.3.3", "playwright": "1.62.1" },
+  "revisionTransition": {
+    "sourceRevision": "0a006589477f8ae47bad3faa5510c999cff85ee4",
+    "firstCommit": "bf7a3d8a687daba5b9cc730587a8d85b5deef91a",
+    "changes": "Root-authorized SPEC_REVISION/TRACK header adoption plus frozen E10 inputs; product still matches START.",
+    "oldCommitsTagsEvidencePreserved": true
+  },
+  "frozen": {
+    "scenario.json": "2ee0cab740c72774372ecf06f669d2b05843944b905877598f92ee013a125c70",
+    "fixture.ts": "8038b62ea5c1188661ffe63b99f25ea3eb27826aa7560ff32306643e8f7a1705",
+    "baseline.mjs": "afd2bbad96dedd148c3c9e769679f0241e4c2732a653d938f35257d2fcbfcb9e",
+    "unchangedAfterBaseline": true
+  },
+  "commands": [
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 node evidence/phase-1/E10/baseline.mjs",
+      "invocation": 1, "exitCode": 0, "wallDurationMs": 1120, "runnerDurationMs": 708,
+      "result": "REPRODUCED", "report": "baseline.json",
+      "observation": "Two requests released at one barrier returned202 with two distinct durable QUEUED IDs. Worker stopped; outbound0. Stopped at this decisive failure."
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run typecheck",
+      "invocation": 1, "exitCode": 2, "wallDurationMs": 3140,
+      "failure": "The legacy authenticated test helper URL is string|URL-object; a regex call needed a pathname/string branch. Only that helper was corrected."
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:unit",
+      "invocation": 1, "exitCode": 0, "wallDurationMs": 1940, "testDurationMs": 1616.796917,
+      "passed": 21, "failed": 0
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run typecheck",
+      "invocation": 2, "exitCode": 0, "wallDurationMs": 1820
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:ownership",
+      "invocation": 1, "exitCode": 0, "wallDurationMs": 2280, "testDurationMs": 2063.912167,
+      "passed": 2, "failed": 0, "reports": ["identity.json", "claim.json"]
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:functional",
+      "invocation": 1, "exitCode": 0, "wallDurationMs": 12120, "testDurationMs": 11816.561292,
+      "passed": 15, "failed": 0
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:integration",
+      "invocation": 1, "exitCode": 0, "wallDurationMs": 8690, "testDurationMs": 8338.496459,
+      "passed": 10, "failed": 0
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:e2e",
+      "invocation": 1, "exitCode": 1, "wallDurationMs": 13090, "browserRunnerDurationMs": 7300,
+      "build": "PASS; included in this command, no separate build duration measured",
+      "passed": 5, "failed": 1, "notRun": 6,
+      "failure": "E10's unqualified role=alert selector matched both Next's route announcer and the correct product INTERNAL_ERROR. Narrowed only the new test to the alert carrying data-error-code; product and frozen assertions unchanged."
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 node node_modules/@playwright/test/cli.js test test/browser/idempotency.spec.ts",
+      "invocation": 1, "exitCode": 0, "wallDurationMs": 21550, "browserRunnerDurationMs": 20500,
+      "passed": 1, "failed": 0, "build": "Reused unchanged production artifact from preceding command"
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 node node_modules/@playwright/test/cli.js test",
+      "invocation": 1, "exitCode": 0, "wallDurationMs": 35250, "browserRunnerDurationMs": 34400,
+      "passed": 12, "failed": 0, "build": "Same unchanged production artifact; no product change after build",
+      "report": "browser.json"
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:execution",
+      "invocation": 1, "exitCode": 0, "wallDurationMs": 3310, "testDurationMs": 2944.638833,
+      "passed": 2, "failed": 0, "reports": ["e09-execution.json", "e09-scheduler.json"]
+    },
+    {
+      "command": "fnm exec --using 24.19.0 node --input-type=module (inline final cleanup audit and safe report copy)",
+      "invocation": 1, "exitCode": 0, "toolWallDurationMs": 647.348875,
+      "report": "cleanup.json"
+    }
+  ],
+  "acceptance": {
+    "parallelSameKey": "Same owner/key ->one CheckRun ID and row; two actual202 responses; no API outbound.",
+    "meaningConflict": "Same owner/key targeting B ->409 CONFLICT, B rows/outbound0.",
+    "durability": "API restart and changes to A name/enabled retain the same identity; a different valid key creates one new intention.",
+    "invalidIdentity": "Six frozen invalid/missing cases ->400 INVALID_INPUT with no rows added; unit coverage also rejects control/non-ASCII/coerced values.",
+    "claim": "Two actual worker PIDs with different UUIDs blocked on one PostgreSQL barrier. One owner and outbound call; losing worker UUID passed to production completion guard ->0 writes; the unchanged RUNNING row later becomes SUCCEEDED200 under the same owner.",
+    "browser": "Four explicit check actions ->three POSTs:held original plus ignored duplicate; lost delivery; same-key/same-ID manual retransmission; a new key after acknowledgement. Exactlytwo intent rows, both completed by the actual worker."
+  },
+  "necessaryConsumerChanges": [
+    "Legacy authenticated test helpers add a fresh key only to an intentional manual Check POST lacking one; explicit keys are preserved. E10 missing/invalid tests use direct injection and do not use this setup helper.",
+    "The existing worker helper reads the worker UUID from its readiness line for the actual two-process ownership proof; all prior separate-worker/202 assertions remain.",
+    "Migration consumers include007 and compare prior fields unchanged, with null request/worker fields on legacy rows.",
+    "The browser's frozen submit wording uses the existing Run check button action; no form, automatic retry or response facade was added."
+  ],
+  "regressions": {
+    "e01ThroughE07": "All existing API, storage and browser tests passed, including exact E06 duplicate-submit/failure barriers and E07 cursor/URL history cases.",
+    "e08": "Production SSR, hydration, owner privacy, keyboard and axe gate passed unchanged. Previously reported native-listbox contrast incomplete is not a manual contrast verdict.",
+    "e09": "Same-ID QUEUED ->RUNNING ->SUCCEEDED200 and real separate worker held-fixture/browser flow; fixed scheduler A counts1/1/2 and disabled B0."
+  },
+  "budget": {
+    "loadRuns": 0, "automaticRetries": 0, "parameterSweeps": 0,
+    "unchangedStartBaselines": 1, "workerRaceBaselines": 0,
+    "postFixOwnershipSuiteInvocations": 1, "postFixManualPairInvocations": 1, "twoWorkerClaimInvocations": 1,
+    "typecheckInvocations": 2, "unitInvocations": 1, "apiInvocations": 1, "postgresIntegrationInvocations": 1,
+    "productionBuildInvocations": 1, "fullBrowserSuiteInvocations": 2, "targetedBrowserInvocations": 1,
+    "e10BrowserScenarioInvocations": 3, "e09ExecutionSuiteInvocations": 1,
+    "failedVerificationInvocations": 2,
+    "note": "The first E10 browser invocation stopped after the held/lost first response. The selector repair used only that test before the final complete suite. Typecheck/unit and second typecheck/ownership were independent parallel pairs. Successful-state polling is observation, not automatic work retry. Root invocations are recorded separately."
+  },
+  "nonVerificationTooling": {
+    "execSyntaxOrReferenceFailuresBeforeCommandExecution": 8,
+    "statusMessageArgumentFailure": 1,
+    "patchContextMismatch": 1,
+    "readPathMisses": 2,
+    "permissionDenials": 0,
+    "note": "These authoring/read errors did not execute a scenario or repeat a verification command. No permissions were bypassed."
+  },
+  "scope": {
+    "newDependencies": [], "earlierPinsLockfileMigrationsEvidenceUnchanged": true,
+    "requestIdentityLifetime": "Existing CheckRun/Monitor cascade; no independent tombstone or expiration policy",
+    "noLeasesRecoveryRedisKafkaOrPhase2": true,
+    "mainSpecIndexTagsUntouchedByImplementationAgent": true,
+    "credentialsTracesScreenshotsVideosStorageStateCommitted": false,
+    "pushPerformed": false
+  },
+  "cleanup": "cleanup.json",
+  "unresolved": "No E10 acceptance failure; no E11 or phase-2 work started."
+}
diff --git a/package.json b/package.json
index c099d60..ae88a3f 100644
--- a/package.json
+++ b/package.json
@@ -22,7 +22,8 @@
     "db:down": "docker compose --project-name wse-fundamentals down",
     "db:migrate": "node server/migrate.ts",
     "test:integration": "node --test --test-concurrency=1 test/persistence.test.ts test/storage-*.test.ts",
-    "test:execution": "node --test test/execution.test.ts"
+    "test:execution": "node --test test/execution.test.ts",
+    "test:ownership": "node --test test/ownership.test.ts"
   },
   "dependencies": {
     "fastify": "5.12.1",
diff --git a/test/auth.ts b/test/auth.ts
index 466ff75..395ea34 100644
--- a/test/auth.ts
+++ b/test/auth.ts
@@ -49,7 +49,11 @@ export function authenticatedInject(app: FastifyInstance, cookie: string) {
     const options = typeof input === 'string' ? { url: input } : input;
     const unsafe = options.method && !['GET', 'HEAD'].includes(options.method);
     const csrf = unsafe ? { 'x-csrf-token': await csrfForTest(app, cookie) } : {};
-    return app.inject({ ...options, headers: { origin: DEFAULT_BROWSER_ORIGIN, ...csrf, ...options.headers, cookie } });
+    const path = typeof options.url === 'string' ? options.url : options.url?.pathname ?? '';
+    const check = options.method === 'POST' && /^\/monitors\/[^/?]+\/checks$/.test(path);
+    const identity = check && !Object.keys(options.headers ?? {}).some(name => name.toLowerCase() === 'idempotency-key')
+      ? { 'idempotency-key': randomUUID() } : {};
+    return app.inject({ ...options, headers: { origin: DEFAULT_BROWSER_ORIGIN, ...csrf, ...identity, ...options.headers, cookie } });
   };
 }
 
@@ -58,6 +62,9 @@ export function authenticatedFetch(cookie: string) {
     const headers = new Headers(options.headers);
     headers.set('cookie', cookie);
     headers.set('origin', DEFAULT_BROWSER_ORIGIN);
+    if (options.method === 'POST' && /^\/monitors\/[^/]+\/checks$/.test(new URL(url).pathname) && !headers.has('idempotency-key')) {
+      headers.set('idempotency-key', randomUUID());
+    }
     if (options.method && !['GET', 'HEAD'].includes(options.method)) {
       const response = await fetch(new URL('/auth/csrf', url), { headers: { cookie } });
       if (response.status !== 200) throw new Error('CSRF fixture preparation failed.');
diff --git a/test/browser/idempotency.spec.ts b/test/browser/idempotency.spec.ts
new file mode 100644
index 0000000..7e22ebf
--- /dev/null
+++ b/test/browser/idempotency.spec.ts
@@ -0,0 +1,100 @@
+import { test, readTerminalCheck } from './session.ts';
+import { expect } from '@playwright/test';
+import { mkdir, writeFile } from 'node:fs/promises';
+import { databasePool } from '../../server/database.ts';
+import { testDatabaseConfig } from '../database.ts';
+import { DEFAULT_BROWSER_ORIGIN } from '../../server/auth.ts';
+import { insertIdentityFixture } from '../../evidence/phase-1/E10/fixture.ts';
+import scenario from '../../evidence/phase-1/E10/scenario.json' with { type: 'json' };
+
+test.use({ autoLogin: false });
+
+test('E10 lost acknowledgement resends one intent; the next acknowledged click starts another', async ({ page }, testInfo) => {
+  const began = performance.now();
+  expect(testInfo.config.workers).toBe(1);
+  expect(testInfo.parallelIndex).toBe(0);
+  const pool = databasePool(testDatabaseConfig('e03_browser'));
+  const committed = Promise.withResolvers<void>();
+  const release = Promise.withResolvers<void>();
+  const keys: string[] = [];
+  const ids: string[] = [];
+  const report: Record<string, unknown> = { result: 'FAILED', productionBuild: true };
+  const route = `**/api${scenario.manual.path}`;
+  let failedDeliveries = 0;
+  page.on('requestfailed', request => {
+    if (request.method() === 'POST' && request.url().endsWith(`/api${scenario.manual.path}`)) failedDeliveries++;
+  });
+  try {
+    const credentials = await insertIdentityFixture(pool);
+    try {
+      const login = await page.request.post('/api/auth/login', { headers: { origin: DEFAULT_BROWSER_ORIGIN }, data: credentials });
+      if (login.status() !== 200) throw new Error('Login rejected.');
+    } catch { throw new Error('E10 browser authentication failed; credential details suppressed.'); }
+    await page.route(route, async intercepted => {
+      if (intercepted.request().method() !== 'POST') { await intercepted.continue(); return; }
+      const key = intercepted.request().headers()['idempotency-key'] ?? '';
+      keys.push(key);
+      const response = await intercepted.fetch({ maxRedirects: 0, maxRetries: 0 });
+      expect(response.status()).toBe(202);
+      ids.push((await response.json()).data.id);
+      if (ids.length === 1) {
+        committed.resolve();
+        await release.promise;
+        await intercepted.abort('failed');
+      } else await intercepted.fulfill({ response });
+    });
+    await page.goto(scenario.browser.route);
+    const a = page.getByRole('article', { name: scenario.monitors[0].name, exact: true });
+    await a.getByRole('button', { name: 'Run check', exact: true }).click();
+    await committed.promise;
+    // "Submit" is the existing check button action. Dispatch it again while
+    // the actual first202 is held; do not add a form or a response facade.
+    await a.getByRole('button', { name: 'Queueing…', exact: true }).dispatchEvent('click');
+    expect(keys.length).toBe(1);
+    expect((await pool.query('SELECT count(*)::int AS count FROM check_runs WHERE monitor_id = $1', [scenario.monitors[0].id])).rows[0].count).toBe(1);
+    release.resolve();
+    await expect(page.locator('[role="alert"][data-error-code]')).toHaveAttribute('data-error-code', 'INTERNAL_ERROR');
+    await expect(a.getByText('No checks yet.', { exact: true })).toBeVisible();
+
+    const replayed = page.waitForResponse(response => response.url().endsWith(`/api${scenario.manual.path}`) && response.request().method() === 'POST');
+    await a.getByRole('button', { name: 'Run check', exact: true }).click();
+    expect((await replayed).status()).toBe(202);
+    expect(ids.length).toBe(2);
+    expect(ids[1]).toBe(ids[0]);
+    expect(keys[0].length > 0 && keys[0].length <= 128 && !/[^!-~]/.test(keys[0])).toBe(true);
+    expect(keys[1] === keys[0]).toBe(true);
+    expect((await pool.query('SELECT count(*)::int AS count FROM check_runs WHERE monitor_id = $1', [scenario.monitors[0].id])).rows[0].count).toBe(1);
+    await expect(a.getByRole('button', { name: 'Run check', exact: true })).toBeEnabled();
+
+    const fresh = page.waitForResponse(response => response.url().endsWith(`/api${scenario.manual.path}`) && response.request().method() === 'POST');
+    await a.getByRole('button', { name: 'Run check', exact: true }).click();
+    expect((await fresh).status()).toBe(202);
+    expect(keys.length).toBe(scenario.browser.expectedForwardedPosts);
+    expect(keys[2] !== keys[0]).toBe(true);
+    expect(ids[2]).not.toBe(ids[0]);
+    const terminal = await Promise.all([ids[0], ids[2]].map(id => readTerminalCheck(page, id)));
+    expect(terminal.every(check => check.state === 'SUCCEEDED' && check.httpStatus === 200)).toBe(true);
+    await expect(a.locator('dl')).toContainText(ids[2]);
+    await expect(a.locator('dl')).toContainText('SUCCEEDED');
+    await expect(a.locator('tbody tr')).toHaveCount(2);
+    const rows = (await pool.query('SELECT id FROM check_runs WHERE monitor_id = $1 ORDER BY id', [scenario.monitors[0].id])).rows;
+    expect(rows.map(row => row.id).sort()).toEqual([ids[0], ids[2]].sort());
+    expect(rows.length).toBe(scenario.browser.expectedIntentRows);
+    expect(failedDeliveries).toBe(1);
+    report.observation = { submittedActions: 4, forwardedPosts: keys.length, sameKeyOnRetransmission: true,
+      newKeyAfterAcknowledgement: true, returnedIds: ids, intentRows: rows.length, failedDeliveries,
+      actualWorkerCompletedIds: terminal.map(check => check.id), automaticRetries: 0 };
+    report.result = 'PASS';
+  } catch (error) { report.failure = error instanceof Error ? error.message : 'Browser identity verification failed.'; throw error; }
+  finally {
+    release.resolve();
+    await page.unroute(route);
+    await page.close();
+    await pool.query('DELETE FROM monitors WHERE owner_user_id = $1', [scenario.user.id]);
+    await pool.query('DELETE FROM users WHERE id = $1', [scenario.user.id]);
+    await pool.end();
+    report.durationMs = Math.round(performance.now() - began);
+    await mkdir('output/phase-1/e10', { recursive: true });
+    await writeFile('output/phase-1/e10/browser.json', JSON.stringify(report, null, 2) + '\n');
+  }
+});
diff --git a/test/ownership.test.ts b/test/ownership.test.ts
new file mode 100644
index 0000000..8858a93
--- /dev/null
+++ b/test/ownership.test.ts
@@ -0,0 +1,212 @@
+import { test } from 'node:test';
+import assert from 'node:assert/strict';
+import { setTimeout as delay } from 'node:timers/promises';
+import { mkdir, writeFile } from 'node:fs/promises';
+import type { PoolClient } from 'pg';
+import { buildApp } from '../server/app.ts';
+import { DEFAULT_BROWSER_ORIGIN } from '../server/auth.ts';
+import { databaseConfig, databasePool, schemaIdentifier } from '../server/database.ts';
+import { migrate } from '../server/migrate.ts';
+import { checkRunFromRow } from '../server/mapping.ts';
+import type { CheckRun, TerminalCheckRun } from '../server/model.ts';
+import { completeCheck } from '../server/worker.ts';
+import { authenticatedFetch, csrfForTest, loginForTest } from './auth.ts';
+import { startTestWorker, waitForTerminalCheck } from './worker.ts';
+import { identityFixture, insertIdentityFixture, manualBarrier, requireFreePorts } from '../evidence/phase-1/E10/fixture.ts';
+import scenario from '../evidence/phase-1/E10/scenario.json' with { type: 'json' };
+
+async function record(name: string, report: Record<string, unknown>) {
+  await mkdir('output/phase-1/e10', { recursive: true });
+  await writeFile(`output/phase-1/e10/${name}.json`, JSON.stringify(report, null, 2) + '\n');
+}
+
+test('E10 parallel manual identity persists one intent, conflicts by meaning and survives restart', { timeout: 30000 }, async () => {
+  const began = performance.now();
+  const config = { ...databaseConfig(), schema: scenario.schema };
+  const pool = databasePool(config);
+  const fixture = identityFixture();
+  let app = buildApp(undefined, config);
+  const barrier = manualBarrier(app);
+  let owned = false;
+  let worker: Awaited<ReturnType<typeof startTestWorker>> | undefined;
+  let pair: Promise<Response[]> | undefined;
+  let timer: ReturnType<typeof setTimeout> | undefined;
+  const report: Record<string, unknown> = { result: 'FAILED' };
+  try {
+    await requireFreePorts();
+    await pool.query(`CREATE SCHEMA ${schemaIdentifier(config.schema)}`);
+    owned = true;
+    await migrate(config);
+    const cookie = await loginForTest(app, await insertIdentityFixture(pool));
+    const request = authenticatedFetch(cookie);
+    await new Promise<void>((resolve, reject) => { fixture.server.once('error', reject); fixture.server.listen(4311, '127.0.0.1', resolve); });
+    await app.listen({ host: '127.0.0.1', port: 4312 });
+    const url = `http://127.0.0.1:4312${scenario.manual.path}`;
+    const headers = { [scenario.manual.header]: scenario.manual.key };
+    pair = Promise.all(Array.from({ length: scenario.manual.concurrency }, () => request(url,
+      { method: scenario.manual.method, headers, signal: AbortSignal.timeout(scenario.watchdogMs) })));
+    await Promise.race([barrier.arrived, new Promise((_, reject) => { timer = setTimeout(() => reject(new Error('Manual request barrier timed out.')), scenario.watchdogMs); })]);
+    clearTimeout(timer);
+    barrier.release();
+    const responses = await pair;
+    assert.deepEqual(responses.map(response => response.status), scenario.manual.expectedStatuses);
+    const checks: CheckRun[] = await Promise.all(responses.map(async response => (await response.json()).data));
+    assert.equal(barrier.arrivals(), 2);
+    assert.equal(new Set(checks.map(check => check.id)).size, scenario.manual.expectedUniqueIds);
+    assert.ok(checks.every(check => check.state === 'QUEUED' && check.monitorId === scenario.monitors[0].id));
+    assert.equal((await pool.query('SELECT count(*)::int AS count FROM check_runs')).rows[0].count, scenario.manual.expectedRows);
+    assert.equal(fixture.calls.size, 0);
+    report.parallel = { arrivals: 2, statuses: responses.map(response => response.status), returnedIds: checks.map(check => check.id), intentRows: 1, outbound: 0 };
+
+    const conflict = await request(`http://127.0.0.1:4312/monitors/${scenario.monitors[1].id}/checks`, { method: 'POST', headers });
+    assert.equal(conflict.status, 409);
+    assert.equal((await conflict.json()).error.code, 'CONFLICT');
+    assert.equal((await pool.query('SELECT count(*)::int AS count FROM check_runs WHERE monitor_id = $1', [scenario.monitors[1].id])).rows[0].count, 0);
+    report.conflict = { status: 409, code: 'CONFLICT', bRows: 0, bOutbound: 0 };
+
+    await app.close();
+    app = buildApp(undefined, config);
+    await app.listen({ host: '127.0.0.1', port: 4312 });
+    const update = await request(`http://127.0.0.1:4312/monitors/${scenario.monitors[0].id}`, {
+      method: 'PUT', headers: { 'content-type': 'application/json' },
+      body: JSON.stringify({ ...scenario.monitors[0], name: 'E10 renamed A', enabled: false }),
+    });
+    assert.equal(update.status, 200);
+    const replay = await request(url, { method: 'POST', headers });
+    assert.equal(replay.status, 202);
+    assert.equal((await replay.json()).data.id, checks[0].id);
+    report.restart = { retainedIdentity: true, mutableMonitorFieldsExcluded: true };
+
+    const fresh = await request(url, { method: 'POST', headers: { [scenario.manual.header]: scenario.manual.newKey } });
+    assert.equal(fresh.status, 202);
+    const newCheck: CheckRun = (await fresh.json()).data;
+    assert.notEqual(newCheck.id, checks[0].id);
+    assert.equal(newCheck.state, 'QUEUED');
+    const csrf = await csrfForTest(app, cookie);
+    for (const invalid of scenario.manual.invalidKeys) {
+      const key = typeof invalid === 'object' && invalid !== null ? invalid.repeat.repeat(invalid.length) : invalid;
+      // Direct injection deliberately bypasses the old-test helper's new-key
+      // setup, including header characters a browser cannot put on the wire.
+      const response = await app.inject({ method: 'POST', url: scenario.manual.path,
+        headers: { cookie, origin: DEFAULT_BROWSER_ORIGIN, 'x-csrf-token': csrf, ...(key === null ? {} : { 'idempotency-key': key }) } });
+      assert.equal(response.statusCode, 400);
+      assert.equal(response.json().error.code, 'INVALID_INPUT');
+    }
+    const rows = (await pool.query('SELECT id, monitor_id, request_user_id FROM check_runs ORDER BY id')).rows;
+    assert.equal(rows.length, 2);
+    assert.ok(rows.every(row => row.monitor_id === scenario.monitors[0].id && row.request_user_id === scenario.user.id));
+    worker = await startTestWorker(config);
+    const terminal = await Promise.all([checks[0].id, newCheck.id].map(id => waitForTerminalCheck(pool, id)));
+    assert.ok(terminal.every(check => check.state === 'SUCCEEDED' && check.httpStatus === 200));
+    assert.equal(fixture.calls.get('/ok'), 2);
+    assert.equal(fixture.calls.get('/fail') ?? 0, 0);
+    report.final = { distinctIntentIds: terminal.map(check => check.id), rows: 2, invalidIdentityCases: scenario.manual.invalidKeys.length,
+      workerPid: worker.pid, state: 'SUCCEEDED', httpStatus: 200, okOutbound: 2, failOutbound: 0 };
+    report.result = 'PASS';
+  } catch (error) { report.failure = error instanceof Error ? error.message : 'Identity verification failed.'; throw error; }
+  finally {
+    clearTimeout(timer);
+    barrier.release();
+    if (pair) await pair.catch(() => {});
+    if (worker) report.workerStop = await worker.stop();
+    await app.close();
+    fixture.release();
+    if (fixture.server.listening) { fixture.server.closeAllConnections(); await new Promise<void>(resolve => fixture.server.close(() => resolve())); }
+    if (owned) await pool.query(`DROP SCHEMA ${schemaIdentifier(config.schema)} CASCADE`);
+    await pool.end();
+    await requireFreePorts();
+    report.cleanup = { schemaDropped: owned, portsFree: true };
+    report.durationMs = Math.round(performance.now() - began);
+    await record('identity', report);
+  }
+});
+
+test('E10 two actual workers claim one queued row and only its owner can complete', { timeout: 30000 }, async () => {
+  const began = performance.now();
+  const config = { ...databaseConfig(), schema: scenario.workerSchema };
+  const pool = databasePool(config);
+  const fixture = identityFixture(true);
+  const workers: Awaited<ReturnType<typeof startTestWorker>>[] = [];
+  const report: Record<string, unknown> = { result: 'FAILED' };
+  let owned = false;
+  let barrier: PoolClient | undefined;
+  let locked = false;
+  let timer: ReturnType<typeof setTimeout> | undefined;
+  try {
+    await requireFreePorts();
+    await pool.query(`CREATE SCHEMA ${schemaIdentifier(config.schema)}`);
+    owned = true;
+    await migrate(config);
+    await insertIdentityFixture(pool);
+    await pool.query(`INSERT INTO check_runs (id, monitor_id, trigger, state, queued_at)
+      VALUES ($1, $2, 'MANUAL', 'QUEUED', $3)`, [scenario.worker.queuedId, scenario.monitors[0].id, scenario.worker.queuedAt]);
+    await new Promise<void>((resolve, reject) => { fixture.server.once('error', reject); fixture.server.listen(4311, '127.0.0.1', resolve); });
+    barrier = await pool.connect();
+    const barrierPid = (await barrier.query('SELECT pg_backend_pid() AS pid')).rows[0].pid;
+    await barrier.query('BEGIN');
+    await barrier.query('LOCK TABLE check_runs IN ACCESS EXCLUSIVE MODE');
+    locked = true;
+    const names = ['e10_worker_1', 'e10_worker_2'];
+    for (const name of names) {
+      const connection = new URL(config.connectionString);
+      connection.searchParams.set('application_name', name);
+      workers.push(await startTestWorker({ ...config, connectionString: connection.href }));
+    }
+    assert.equal(new Set(workers.map(worker => worker.pid)).size, 2);
+    assert.equal(new Set(workers.map(worker => worker.workerId)).size, 2);
+    assert.ok(workers.every(worker => worker.pid !== process.pid));
+    const deadline = Date.now() + scenario.watchdogMs;
+    let blocked: { pid: number; application_name: string }[] = [];
+    do {
+      blocked = (await pool.query(`SELECT pid, application_name FROM pg_stat_activity
+        WHERE application_name = ANY($1::text[]) AND wait_event_type = 'Lock'
+          AND $2::int = ANY(pg_blocking_pids(pid)) AND query LIKE '%FOR UPDATE SKIP LOCKED%'`, [names, barrierPid])).rows;
+      if (blocked.length === 2) break;
+      await delay(25);
+    } while (Date.now() < deadline);
+    assert.equal(blocked.length, 2, 'Both real claim transactions must reach the held database barrier.');
+    assert.equal(fixture.calls.size, 0);
+    report.barrier = { workers: workers.map(worker => ({ pid: worker.pid, workerId: worker.workerId })), blocked, outboundBeforeRelease: 0 };
+    await barrier.query('COMMIT');
+    locked = false;
+    barrier.release();
+    barrier = undefined;
+    await Promise.race([fixture.entered, new Promise((_, reject) => { timer = setTimeout(() => reject(new Error('Claim winner did not reach the fixture.')), scenario.watchdogMs); })]);
+    clearTimeout(timer);
+    const running = (await pool.query('SELECT * FROM check_runs WHERE id = $1', [scenario.worker.queuedId])).rows[0];
+    assert.equal(running.state, 'RUNNING');
+    assert.ok(workers.some(worker => worker.workerId === running.worker_id));
+    assert.equal(fixture.calls.get('/ok'), scenario.worker.expectedOutbound);
+    const loser = workers.find(worker => worker.workerId !== running.worker_id)!;
+    const forged: TerminalCheckRun = { ...checkRunFromRow(running), state: 'FAILED', httpStatus: 503,
+      failureReason: 'HTTP_STATUS', latencyMs: 0, startedAt: running.started_at.toISOString(), finishedAt: new Date().toISOString() };
+    assert.equal(await completeCheck(pool, loser.workerId, forged), null);
+    assert.deepEqual((await pool.query('SELECT * FROM check_runs WHERE id = $1', [scenario.worker.queuedId])).rows[0], running);
+    report.held = { id: running.id, state: running.state, owner: running.worker_id, outbound: 1, loserWorker: loser.workerId, loserCompletionWrites: 0 };
+    fixture.release();
+    const terminal = await waitForTerminalCheck(pool, scenario.worker.queuedId);
+    assert.equal(terminal.state, scenario.worker.expectedTerminal);
+    assert.equal(terminal.httpStatus, scenario.worker.releaseStatus);
+    assert.equal((await pool.query('SELECT count(*)::int AS count FROM check_runs')).rows[0].count, scenario.worker.expectedRows);
+    assert.equal((await pool.query('SELECT worker_id FROM check_runs WHERE id = $1', [terminal.id])).rows[0].worker_id, running.worker_id);
+    assert.equal(fixture.calls.get('/ok'), scenario.worker.expectedOutbound);
+    assert.equal(fixture.calls.get('/fail') ?? 0, 0);
+    report.terminal = { ...terminal, ownerUnchanged: true, outbound: 1, rows: 1 };
+    report.result = 'PASS';
+  } catch (error) { report.failure = error instanceof Error ? error.message : 'Claim verification failed.'; throw error; }
+  finally {
+    clearTimeout(timer);
+    if (locked) await barrier?.query('ROLLBACK');
+    barrier?.release();
+    fixture.release();
+    report.workerStops = [];
+    for (const worker of workers) (report.workerStops as unknown[]).push(await worker.stop());
+    if (fixture.server.listening) { fixture.server.closeAllConnections(); await new Promise<void>(resolve => fixture.server.close(() => resolve())); }
+    if (owned) await pool.query(`DROP SCHEMA ${schemaIdentifier(config.schema)} CASCADE`);
+    await pool.end();
+    await requireFreePorts();
+    report.cleanup = { schemaDropped: owned, portsFree: true };
+    report.durationMs = Math.round(performance.now() - began);
+    await record('claim', report);
+  }
+});
diff --git a/test/persistence.test.ts b/test/persistence.test.ts
index a395f7b..ce73c2f 100644
--- a/test/persistence.test.ts
+++ b/test/persistence.test.ts
@@ -54,9 +54,9 @@ test('E03 migration CLI applies the fresh chain and a repeated command changes n
   try {
     const options = { cwd: process.cwd(), env: { ...process.env, DATABASE_URL: config.connectionString, DATABASE_SCHEMA: config.schema }, timeout: 10_000 };
     const first = await execute(process.execPath, ['server/migrate.ts'], options);
-    assert.deepEqual(JSON.parse(first.stdout).applied, [...authScenario.additionalAssertions.expectedMigrationFiles, '004_monitor_ownership.sql', '005_check_history_index.sql', '006_check_queue.sql']);
+    assert.deepEqual(JSON.parse(first.stdout).applied, [...authScenario.additionalAssertions.expectedMigrationFiles, '004_monitor_ownership.sql', '005_check_history_index.sql', '006_check_queue.sql', '007_check_ownership.sql']);
     const before = (await pool.query('SELECT version, checksum, applied_at FROM schema_migrations ORDER BY version')).rows;
-    assert.equal(before.length, authScenario.additionalAssertions.expectedMigrationFiles.length + 3);
+    assert.equal(before.length, authScenario.additionalAssertions.expectedMigrationFiles.length + 4);
     for (const row of before) { assert.match(row.checksum, /^[0-9a-f]{64}$/); assert.ok(row.applied_at instanceof Date); }
     const second = await execute(process.execPath, ['server/migrate.ts'], options);
     assert.deepEqual(JSON.parse(second.stdout).applied, scenario.additionalAssertions.repeatMigrationExpectedApplied);
@@ -263,14 +263,15 @@ test('E05 upgrades existing rows only with explicit owner designation and preser
     finally { await oldApp.close(); }
 
     // Deliberately designate the second account, never the first row returned.
-    assert.deepEqual(await migrate(config, users[1].username), ['004_monitor_ownership.sql', '005_check_history_index.sql', '006_check_queue.sql']);
+    assert.deepEqual(await migrate(config, users[1].username), ['004_monitor_ownership.sql', '005_check_history_index.sql', '006_check_queue.sql', '007_check_ownership.sql']);
     const owner = (await pool.query('SELECT id FROM users WHERE username = $1', [users[1].username])).rows[0].id;
     const after = (await pool.query('SELECT * FROM monitors ORDER BY id')).rows;
     assert.deepEqual(after.map(({ owner_user_id: _owner, ...row }) => row), before.monitors);
     assert.ok(after.every((row) => row.owner_user_id === owner));
     const upgradedChecks = (await pool.query('SELECT * FROM check_runs ORDER BY id')).rows;
-    assert.deepEqual(upgradedChecks.map(({ queued_at: _queued, scheduled_for: _slot, ...row }) => row), before.checks);
+    assert.deepEqual(upgradedChecks.map(({ queued_at: _queued, scheduled_for: _slot, request_user_id: _requestUser, idempotency_key: _key, worker_id: _worker, ...row }) => row), before.checks);
     assert.ok(upgradedChecks.every((row) => row.queued_at.getTime() === row.started_at.getTime() && row.scheduled_for === null));
+    assert.ok(upgradedChecks.every((row) => row.request_user_id === null && row.idempotency_key === null && row.worker_id === null));
     assert.deepEqual((await pool.query('SELECT * FROM schema_migrations ORDER BY version LIMIT 3')).rows, before.migrations);
     await assert.rejects(pool.query('DELETE FROM users WHERE id = $1', [owner]), { code: '23503' });
     const app = buildApp(undefined, config);
diff --git a/test/unit.test.ts b/test/unit.test.ts
index 7a0bd7d..fe521d9 100644
--- a/test/unit.test.ts
+++ b/test/unit.test.ts
@@ -7,7 +7,7 @@ import type { ApiErrorCode } from '../server/model.ts';
 import scenario from '../evidence/E02/scenario.json' with { type: 'json' };
 import authScenario from '../evidence/E04/scenario.json' with { type: 'json' };
 import { hashPassword, verifyPassword, SCRYPT_OPTIONS } from '../server/password.ts';
-import { loginInput } from '../server/contracts.ts';
+import { idempotencyKey, loginInput } from '../server/contracts.ts';
 import { csrfTokenForSession, validCsrfToken, SESSION_COOKIE_NAME, SESSION_TTL_MS, sessionTokenFromCookie } from '../server/auth.ts';
 import { DEFAULT_HISTORY_LIMIT, MAX_HISTORY_LIMIT, MAX_HISTORY_CURSOR_LENGTH, historyCursor, historyQuery } from '../server/history.ts';
 import historyScenario from '../evidence/E07/scenario.json' with { type: 'json' };
@@ -15,6 +15,20 @@ import { emptyMonitors, historyLocation } from '../app/monitors/initial-state.ts
 import { e08RscTransport } from './e08-rsc-transport.mjs';
 import renderingScenario from '../evidence/E08/scenario.json' with { type: 'json' };
 
+test('E10 manual identity is bounded printable ASCII without whitespace or coercion', () => {
+  for (const key of ['manual-intent-e10-1', '!', '~', 'x'.repeat(128)]) assert.equal(idempotencyKey(key), key);
+  for (const key of [undefined, null, [], 1, '', ' ', 'two words', 'two\twords', 'x\n', '\r', '\0', '\x7f', '한글', 'x'.repeat(129)]) {
+    assert.throws(() => idempotencyKey(key), { code: 'INVALID_INPUT' });
+  }
+});
+
+test('E10 browser recognizes semantic conflict only with its409 category', async () => {
+  await assert.rejects(responseData(Response.json({ error: { code: 'CONFLICT', message: 'untrusted prose' } }, { status: 409 })),
+    (error: unknown) => failureCode(error) === 'CONFLICT');
+  await assert.rejects(responseData(Response.json({ error: { code: 'CONFLICT', message: 'untrusted prose' } }, { status: 400 })),
+    (error: unknown) => failureCode(error) === 'INTERNAL_ERROR');
+});
+
 test('E08 RSC caller follows only one same-route framework canonicalization', async () => {
   const route = renderingScenario.primaryRoute;
   for (const [location, count] of [
diff --git a/test/worker.ts b/test/worker.ts
index 319bb2c..5fe93eb 100644
--- a/test/worker.ts
+++ b/test/worker.ts
@@ -13,6 +13,7 @@ export async function startTestWorker(config: DatabaseConfig) {
     stdio: ['ignore', 'pipe', 'pipe'],
   });
   const exited = new Promise<{ code: number | null; signal: string | null }>((resolve) => child.once('close', (code, signal) => resolve({ code, signal })));
+  let workerId = '';
   async function stop() {
     let forced = false;
     const timer = setTimeout(() => { forced = true; child.kill('SIGKILL'); }, 10000);
@@ -24,14 +25,17 @@ export async function startTestWorker(config: DatabaseConfig) {
   try {
     await new Promise<void>((resolve, reject) => {
       const timer = setTimeout(() => reject(new Error('Worker readiness timed out.')), 10000);
+      let output = '';
       child.stdout.on('data', (chunk) => {
-        if (String(chunk).includes('Check worker ready.')) { clearTimeout(timer); resolve(); }
+        output += String(chunk);
+        const ready = /Check worker ready\. ([0-9a-f-]{36})/.exec(output);
+        if (ready) { workerId = ready[1]; clearTimeout(timer); resolve(); }
       });
       child.once('error', () => { clearTimeout(timer); reject(new Error('Worker could not start.')); });
       child.once('exit', () => { clearTimeout(timer); reject(new Error('Worker exited before readiness.')); });
     });
   } catch (error) { await stop(); throw error; }
-  return { pid: child.pid!, stop };
+  return { pid: child.pid!, workerId, stop };
 }
 
 export async function waitForTerminalCheck(pool: Pool, id: string): Promise<TerminalCheckRun> {
