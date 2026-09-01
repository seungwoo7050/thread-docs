## `이력 페이지와 URL 복원 검증 결과 기록`

diff --git a/evidence/E07/browser.json b/evidence/E07/browser.json
new file mode 100644
index 0000000..78fc568
--- /dev/null
+++ b/evidence/E07/browser.json
@@ -0,0 +1,31 @@
+{
+  "hashes": {
+    "scenario.json": "069013e4ee0766c7fec016a1d00c41eae7bdc37fbbb49d8b46cc004325713dd0",
+    "fixture.ts": "3a7a9a22668ad2eb25c534ad51968859b7d70263d5089d566363955ac3779d51"
+  },
+  "result": "PASS",
+  "failedIds": [
+    "07000000-0000-4000-8000-000000000006",
+    "07000000-0000-4000-8000-000000000004",
+    "07000000-0000-4000-8000-000000000002"
+  ],
+  "succeededFirstPageIds": [
+    "07000000-0000-4000-8000-000000000007",
+    "07000000-0000-4000-8000-000000000005",
+    "07000000-0000-4000-8000-000000000003"
+  ],
+  "succeededLastPageIds": [
+    "07000000-0000-4000-8000-000000000001"
+  ],
+  "pageSize": 3,
+  "backwardNavigations": 2,
+  "forwardNavigations": 2,
+  "reloadSamePage": true,
+  "directUrlSelection": true,
+  "firstPageControl": true,
+  "changedFilterResetsCursor": true,
+  "automaticRetries": 0,
+  "traces": false,
+  "videos": false,
+  "screenshots": false
+}
diff --git a/evidence/E07/e06-regression.json b/evidence/E07/e06-regression.json
new file mode 100644
index 0000000..1e4b5a5
--- /dev/null
+++ b/evidence/E07/e06-regression.json
@@ -0,0 +1,17 @@
+{
+  "scenarioSha256": "be0ba3ef17b67140d17266028b257b41bb8ec2d4138beeb051f73681ecfd899b",
+  "harnessSha256": "c0ab1374a9f46658fcac642dc77095cb6abd39a8d204afe4095484c4303ddb56",
+  "result": "PASS",
+  "sequentialCreateUpdatePauseResumeDeleteAndCheckAlreadyCorrect": true,
+  "submitEvents": 2,
+  "csrfReadsAfterSecondSubmit": 1,
+  "forwardedCreateRequests": 1,
+  "authoritativeCreatedRows": 1,
+  "pendingVisibleBeforeSecondSubmit": true,
+  "heldResponses": 1,
+  "failedUpdates": 1,
+  "failedDeletes": 1,
+  "failedMutationsPreservedAuthoritativeState": true,
+  "pendingClearedAfterRelease": true,
+  "durationMs": 1929
+}
diff --git a/evidence/E07/history.json b/evidence/E07/history.json
new file mode 100644
index 0000000..327516d
--- /dev/null
+++ b/evidence/E07/history.json
@@ -0,0 +1,45 @@
+{
+  "hashes": {
+    "scenario.json": "069013e4ee0766c7fec016a1d00c41eae7bdc37fbbb49d8b46cc004325713dd0",
+    "fixture.ts": "3a7a9a22668ad2eb25c534ad51968859b7d70263d5089d566363955ac3779d51"
+  },
+  "result": "PASS",
+  "originalIds": [
+    "07000000-0000-4000-8000-000000000007",
+    "07000000-0000-4000-8000-000000000006",
+    "07000000-0000-4000-8000-000000000005",
+    "07000000-0000-4000-8000-000000000004",
+    "07000000-0000-4000-8000-000000000003",
+    "07000000-0000-4000-8000-000000000002",
+    "07000000-0000-4000-8000-000000000001"
+  ],
+  "firstPageIds": [
+    "07000000-0000-4000-8000-000000000007",
+    "07000000-0000-4000-8000-000000000006",
+    "07000000-0000-4000-8000-000000000005"
+  ],
+  "failedIds": [
+    "07000000-0000-4000-8000-000000000006",
+    "07000000-0000-4000-8000-000000000004",
+    "07000000-0000-4000-8000-000000000002"
+  ],
+  "defaultLimit": 20,
+  "maximumLimit": 100,
+  "invalidRequests": 27,
+  "insertionContinuationIds": [
+    "07000000-0000-4000-8000-000000000007",
+    "07000000-0000-4000-8000-000000000006",
+    "07000000-0000-4000-8000-000000000005",
+    "07000000-0000-4000-8000-000000000004",
+    "07000000-0000-4000-8000-000000000003",
+    "07000000-0000-4000-8000-000000000002",
+    "07000000-0000-4000-8000-000000000001"
+  ],
+  "cursorConditionMismatchRejected": true,
+  "anonymousStatus": 401,
+  "foreignStatus": 404,
+  "finalRowCount": 8,
+  "index": "CREATE INDEX check_runs_monitor_finished_id_idx ON e07_history.check_runs USING btree (monitor_id, finished_at DESC, id DESC)",
+  "loadRuns": 0,
+  "automaticRetries": 0
+}
diff --git a/evidence/E07/isolation.json b/evidence/E07/isolation.json
new file mode 100644
index 0000000..47074f8
--- /dev/null
+++ b/evidence/E07/isolation.json
@@ -0,0 +1,17 @@
+{
+  "result": "PASS",
+  "node": "24.19.0",
+  "ownedSchemasAbsent": [
+    "e07_baseline",
+    "e07_history",
+    "e03_browser"
+  ],
+  "ownedPortsReleased": [
+    4311,
+    4312,
+    4313,
+    4314
+  ],
+  "postgresReachable": true,
+  "catalogReadOnly": true
+}
diff --git a/evidence/E07/legacy-browser-adapter.json b/evidence/E07/legacy-browser-adapter.json
new file mode 100644
index 0000000..8ec821a
--- /dev/null
+++ b/evidence/E07/legacy-browser-adapter.json
@@ -0,0 +1,18 @@
+{
+  "result": "PASS",
+  "scenarioSha256": "be0ba3ef17b67140d17266028b257b41bb8ec2d4138beeb051f73681ecfd899b",
+  "harnessSha256": "c0ab1374a9f46658fcac642dc77095cb6abd39a8d204afe4095484c4303ddb56",
+  "adaptedReloadCount": 3,
+  "removedParametersByReload": [
+    [
+      "monitor"
+    ],
+    [],
+    [
+      "monitor"
+    ]
+  ],
+  "realReloadRetained": true,
+  "scope": "only E06 test/browser/server-state.spec.ts; original method restored in finally",
+  "priorFailure": "The second E07 browser-gate invocation passed nine tests, then E06 timed out at frozen harness line 46 because URL reload now restored the already-open history panel."
+}
diff --git a/evidence/E07/stale-query.json b/evidence/E07/stale-query.json
new file mode 100644
index 0000000..c821119
--- /dev/null
+++ b/evidence/E07/stale-query.json
@@ -0,0 +1,14 @@
+{
+  "result": "PASS",
+  "heldResponses": 1,
+  "heldBeforeFilterChange": true,
+  "explicitRelease": true,
+  "finalFilter": "FAILED",
+  "finalIds": [
+    "07000000-0000-4000-8000-000000000006",
+    "07000000-0000-4000-8000-000000000004",
+    "07000000-0000-4000-8000-000000000002"
+  ],
+  "staleResponseIgnored": true,
+  "automaticRetries": 0
+}
diff --git a/evidence/E07/verification.json b/evidence/E07/verification.json
new file mode 100644
index 0000000..2eed4c9
--- /dev/null
+++ b/evidence/E07/verification.json
@@ -0,0 +1,69 @@
+{
+  "thread": "E07",
+  "attempt": 1,
+  "branch": "track/fundamentals-fastify",
+  "specRevision": "0a006589477f8ae47bad3faa5510c999cff85ee4",
+  "start": "f9adb6423b6ff5e732f89e52c30f7ac383bb0b9e",
+  "implementationEnd": "7851af36280f20a9c182f95061ccb136697a1a65",
+  "runtime": { "node": "24.19.0", "npm": "11.17.0", "selection": "fnm exec --using 24.19.0" },
+  "frozenHashes": {
+    "scenario.json": "069013e4ee0766c7fec016a1d00c41eae7bdc37fbbb49d8b46cc004325713dd0",
+    "fixture.ts": "3a7a9a22668ad2eb25c534ad51968859b7d70263d5089d566363955ac3779d51",
+    "unchangedAfterVerification": true
+  },
+  "baseline": {
+    "evidence": "baseline.json",
+    "result": "REPRODUCED",
+    "executedCases": 1,
+    "requestedLimit": 3,
+    "observedRows": 7,
+    "nextCursorPresent": false,
+    "preExecutionFailure": "The initial sandbox process could not connect to PostgreSQL (EPERM) before creating a schema or making the scenario request. The approved process executed the unchanged scenario once."
+  },
+  "gates": [
+    { "command": "fnm exec --using 24.19.0 npm run typecheck", "result": "PASS", "invocations": 3, "lastInvocationAfterLegacyAdapter": true },
+    { "command": "fnm exec --using 24.19.0 npm run test:unit", "result": "PASS", "passed": 16, "failed": 0, "invocations": 1, "durationMs": 2798.191167 },
+    { "command": "fnm exec --using 24.19.0 npm run test:functional", "result": "PASS", "passed": 15, "failed": 0, "invocations": 1, "durationMs": 8356.325792 },
+    { "command": "fnm exec --using 24.19.0 npm run test:integration", "result": "PASS", "passed": 10, "failed": 0, "invocations": 1, "durationMs": 10430.315208 },
+    { "command": "fnm exec --using 24.19.0 npm run test:e2e", "result": "PASS", "passed": 10, "failed": 0, "invocations": 3, "lastDurationSeconds": 17.8 },
+    { "command": "fnm exec --using 24.19.0 npm run build", "result": "PASS", "invocations": 1, "productionBuild": true, "productionBrowserRunClaimed": false },
+    { "command": "git diff --check", "result": "PASS" }
+  ],
+  "browserInvocations": [
+    { "invocation": 1, "result": "FAILED_BEFORE_TESTS", "executedTests": 0, "reason": "Missing Node 24 JSON import attribute in the new test/browser/history.spec.ts import.", "correction": "Added only the required import attribute; no fixture or assertion change." },
+    { "invocation": 2, "result": "FAILED", "passed": 9, "failed": 1, "durationSeconds": 48.4, "reason": "Frozen E06 harness line 46 expected View history after a bare-list reload, but E07 correctly retained the URL-selected open panel.", "correction": "Root-approved adapter only in test/browser/server-state.spec.ts removes monitor/state/limit/cursor before its three real list reloads, and restores the original method in finally." },
+    { "invocation": 3, "result": "PASS", "passed": 10, "failed": 0, "durationSeconds": 17.8, "normalE07UrlReloadRetained": true }
+  ],
+  "evidence": {
+    "paginationAndIndex": "history.json",
+    "urlBackForwardReload": "browser.json",
+    "lateResponseBoundary": "stale-query.json",
+    "legacyReloadAdapter": "legacy-browser-adapter.json",
+    "preservedDuplicateSubmitAndFailureGuarantees": "e06-regression.json",
+    "ownedResourceCleanup": "isolation.json"
+  },
+  "sqlInspection": {
+    "path": "server/app.ts GET /monitors/:id/checks",
+    "ownerPredicateInParentAndHistory": true,
+    "allExternalValuesBound": true,
+    "order": "c.finished_at DESC, c.id DESC",
+    "seek": "(c.finished_at, c.id) < ($4::timestamptz, $5::uuid)",
+    "queryResultBound": "limit + 1",
+    "apiResultBound": "limit",
+    "migration": "005_check_history_index.sql",
+    "index": "check_runs (monitor_id, finished_at DESC, id DESC)",
+    "explainRuns": 0
+  },
+  "scopeAudit": {
+    "priorEvidenceAndScenariosUnchanged": true,
+    "priorMigrationsUnchanged": true,
+    "runtimeDependencyContainerCiPinsUnchanged": true,
+    "mainHeadUnchanged": "0a006589477f8ae47bad3faa5510c999cff85ee4",
+    "frozenE06HarnessAndAssertionsUnchanged": true,
+    "publicSchemaAndVolumePreserved": true,
+    "newInfrastructure": [],
+    "newCredentialArtifacts": false
+  },
+  "budget": { "loadRuns": 0, "automaticRetries": 0, "parameterSweeps": 0, "baselineExecutions": 1, "browserGateInvocations": 3, "browserWorkers": 1, "browserRetries": 0, "browserMaxFailures": 1, "browserTestTimeoutMs": 30000 },
+  "unresolved": []
+}
