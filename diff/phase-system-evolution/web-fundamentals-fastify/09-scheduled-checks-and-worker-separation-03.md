## `test: verify E09 worker lifecycle and async regressions`

diff --git a/.github/workflows/check.yml b/.github/workflows/check.yml
index b9e79ee..b220a58 100644
--- a/.github/workflows/check.yml
+++ b/.github/workflows/check.yml
@@ -37,6 +37,8 @@ jobs:
         run: npx playwright install --with-deps chromium
       - name: Production build and browser E2E
         run: npm run test:e2e
+      - name: Worker lifecycle and scheduler
+        run: npm run test:execution
       - name: Stop isolated PostgreSQL
         if: always()
         run: npm run db:down
diff --git a/TRACK.md b/TRACK.md
index 6d3b353..8f3ae3f 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -10,8 +10,9 @@ Checks have a fixed 2 second total deadline, perform no retries, do not follow
 redirects, and close the response after observing headers without retaining a body.
 HTTP 200–299 is `SUCCEEDED`; other observed statuses are `FAILED/HTTP_STATUS`.
 A transport failure records `FAILED/CONNECTION_FAILURE` or `FAILED/TIMEOUT` with
-`httpStatus: null`. The list shows the latest terminal result; E03 history reads retain all results.
-`enabled` and `interval` are stored fields; there is no scheduled execution.
+`httpStatus: null`. E09 shows an active CheckRun, or the latest terminal result,
+on each Monitor; bounded history retains completed results. A separate worker
+now creates interval-based scheduled intents and executes persisted checks.
 
 ## Fixed versions
 
@@ -58,6 +59,8 @@ npm run fixture
 # A separate terminal:
 npm run dev:api
 # A third terminal:
+npm run start:worker
+# A fourth terminal:
 npm run dev:web
 ```
 
@@ -78,13 +81,13 @@ npm run test:functional
 npm run test:integration
 npx playwright install chromium
 npm run test:e2e
-npm run build
+npm run test:execution
 ```
 
 Functional tests own fixed loopback ports 4311, 4312 and 4314; stop manual fixture
 processes before running them. Port 4314 is a controlled destination that must
-never receive a check. E03 uses PostgreSQL on loopback port 15431. E04 adds login;
-there is no worker, Redis, Kafka, or production application container.
+never receive a check. E03 uses PostgreSQL on loopback port 15431. E04 adds login
+and E09 adds one check worker; there is no Redis, Kafka, or production application container.
 The E2E command owns ports 4311–4313 and starts fresh fixture/API/Next processes;
 stop manual development processes first. It uses one Chromium worker, no test
 retries, and the fixed Monitor fixture in `evidence/E01/scenario.json`.
@@ -530,3 +533,40 @@ cases are unchanged.
 E04's Bob-login consumer reads the actual response body before releasing that
 single response to the browser; document replacement otherwise evicts Chromium's
 response body. The same login status, username and resulting session are checked.
+
+## E09 queued checks and one worker
+
+Manual `POST /monitors/:id/checks` now returns **202** with a persisted `QUEUED`
+CheckRun. Existing authentication, CSRF and owner predicates still run before
+insertion. No endpoint I/O runs in the API. `npm run start:worker` starts one
+separate process; it persists `RUNNING` before requesting headers and updates
+that same ID with the observed terminal result. Pending result fields are null.
+Migration `006_check_queue.sql` preserves existing rows and adds internal queue
+time and scheduled-slot identity; earlier migrations are unchanged.
+
+The worker's small scheduler tick uses the Monitor interval, anchored to its
+creation time. Each tick addresses the current slot; it does not replay missed
+intervals. PostgreSQL [date_bin](https://www.postgresql.org/docs/17/functions-datetime.html#FUNCTIONS-DATETIME-BIN)
+computes that slot, and `(monitor_id, scheduled_for)` is unique. Repeated ticks
+therefore insert it once. Disabled Monitors generate no scheduled intents.
+Distinct manual and scheduled intents may overlap; this one worker processes
+them serially. There is no manual-request idempotency, competing-worker claim
+guarantee, lease or crash recovery in E09; an interrupted RUNNING row can remain.
+
+The list/detail shows an active execution before terminal results. Browser reads
+poll only while its displayed execution is QUEUED/RUNNING, stop on completion or
+read failure, and respect the existing session/deletion guards. Reload discovers
+new scheduled work. History remains terminal-only with E07's unchanged bounded
+cursor/filter ordering. No new client state library or realtime transport is used.
+
+`--no-schedule` is a deterministic regression control on the same worker process;
+normal operation schedules by default. Prior live tests assert real202 QUEUED,
+then read the same ID after actual worker completion. The E06 historical harness
+is retained unchanged; its live copy changes only that async check block and
+imports, retaining all duplicate-submit, held-response, failure and reload checks.
+
+After `npm run test:e2e` builds production Next, `npm run test:execution` uses that
+artifact for the fixed held-endpoint Chromium flow and exact scheduler ticks.
+It owns only `e09_execution`/`e09_scheduler`, refuses occupied reserved ports, and
+awaits owned process/schema cleanup. Safe results and actual invocation counts
+are in `evidence/E09`; no credential-bearing browser artifacts are retained.
diff --git a/evidence/E09/cleanup.json b/evidence/E09/cleanup.json
new file mode 100644
index 0000000..751a972
--- /dev/null
+++ b/evidence/E09/cleanup.json
@@ -0,0 +1,11 @@
+{
+  "portsFree": [
+    4311,
+    4312,
+    4313,
+    4314
+  ],
+  "residualTestSchemas": [],
+  "postgresReachable": true,
+  "publicAndVolumesNotModified": true
+}
diff --git a/evidence/E09/execution.json b/evidence/E09/execution.json
new file mode 100644
index 0000000..c0a6917
--- /dev/null
+++ b/evidence/E09/execution.json
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
+    "id": "a08de850-dbfa-4609-99d6-46f3bcbeb625",
+    "state": "QUEUED",
+    "http_status": null,
+    "started_at": null,
+    "finished_at": null,
+    "outboundCalls": 0,
+    "visibleState": "QUEUED"
+  },
+  "whileHeld": {
+    "apiPid": 17176,
+    "workerPid": 17205,
+    "sameId": true,
+    "state": "RUNNING",
+    "httpStatus": null,
+    "finishedAt": null,
+    "outboundCalls": 1,
+    "browserUsable": true
+  },
+  "afterRelease": {
+    "id": "a08de850-dbfa-4609-99d6-46f3bcbeb625",
+    "monitorId": "09000000-0000-4000-9000-000000000001",
+    "trigger": "MANUAL",
+    "state": "SUCCEEDED",
+    "httpStatus": 200,
+    "latencyMs": 127,
+    "failureReason": null,
+    "startedAt": "2026-08-28T02:46:53.775Z",
+    "finishedAt": "2026-08-28T02:46:53.908Z",
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
+  "durationMs": 2609
+}
diff --git a/evidence/E09/scheduler.json b/evidence/E09/scheduler.json
new file mode 100644
index 0000000..9ba67ac
--- /dev/null
+++ b/evidence/E09/scheduler.json
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
+  "durationMs": 390
+}
diff --git a/evidence/E09/verification.json b/evidence/E09/verification.json
new file mode 100644
index 0000000..fd3ed75
--- /dev/null
+++ b/evidence/E09/verification.json
@@ -0,0 +1,111 @@
+{
+  "thread": "E09",
+  "attempt": 1,
+  "branch": "track/fundamentals-fastify",
+  "specRevision": "0a006589477f8ae47bad3faa5510c999cff85ee4",
+  "start": "19138e0618cb22b0250c00ea3f52de6095d95af8",
+  "result": "PASS",
+  "runtime": { "node": "24.19.0", "npm": "11.17.0", "next": "16.3.3", "playwright": "1.62.1" },
+  "frozen": {
+    "scenario.json": "d2412d309e282a7d99a7a73ee64c204b4fddfef8b7211e42b573e501de4f8240",
+    "fixture.ts": "4ecfea011cda7656e7e59eb7236f378f3ad251193e23d01135f642139327c078",
+    "baseline.mjs": "a95b8b69b99a838899c1b71b2d0beec9fc57502de33b100b151e3ccedfdfc946",
+    "unchangedAfterBaseline": true
+  },
+  "commands": [
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 node evidence/E09/baseline.mjs",
+      "invocation": 1, "exitCode": 0, "wallDurationMs": 1200, "runnerDurationMs": 757,
+      "result": "REPRODUCED",
+      "observation": "One held outbound request with no persisted intent and no API response; release produced synchronous200 SUCCEEDED. No repeat."
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 node --test --test-name-pattern=\"E09 fixed scheduler\" test/execution.test.ts",
+      "invocation": 1, "exitCode": 0, "wallDurationMs": 820, "testDurationMs": 604.945792,
+      "passed": 1, "failed": 0
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run typecheck",
+      "invocation": 1, "exitCode": 0, "wallDurationMs": 3090
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run build",
+      "invocation": 1, "exitCode": 0, "wallDurationMs": 4940
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:execution",
+      "invocation": 1, "exitCode": 0, "wallDurationMs": 3910, "testDurationMs": 3453.950375,
+      "passed": 2, "failed": 0,
+      "reports": ["scheduler.json", "execution.json"],
+      "observation": "Actual production Chromium: persisted202 QUEUED/outbound0 before worker, distinct worker PID, same ID RUNNING while held and B draft usable, then released200 SUCCEEDED and terminal history. Scheduler ticks1/1/2; disabled0."
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:functional",
+      "invocation": 1, "exitCode": 0, "wallDurationMs": 19170, "testDurationMs": 18222.564,
+      "passed": 15, "failed": 0
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:integration",
+      "invocation": 1, "exitCode": 0, "wallDurationMs": 9720, "testDurationMs": 8608.340083,
+      "passed": 10, "failed": 0
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:unit",
+      "invocation": 1, "exitCode": 0, "wallDurationMs": 1800, "testDurationMs": 1579.7615,
+      "passed": 19, "failed": 0
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:e2e",
+      "invocation": 1, "exitCode": 0, "wallDurationMs": 23280, "testDurationMs": 19000,
+      "build": "PASS", "passed": 11, "failed": 0
+    }
+  ],
+  "necessaryConsumerChanges": [
+    "Live API/browser tests assert real202 QUEUED and then read the same ID after real separate-worker completion. Terminal result, identity, error, ownership, CSRF and fixture-outbound assertions remain; no fabricated200 response or direct bypass of the worker.",
+    "One worker per test schema uses --no-schedule to preserve frozen manual counts. Production worker scheduling remains on by default; the three fixed scheduler ticks are tested separately.",
+    "E06 historical harness is unchanged. Its live copy changes only imports and the check-response block to await202 and same-ID terminal completion; all original barrier, duplicate-submit, failure and reload assertions remain.",
+    "Migration consumers include006 and compare all prior CheckRun columns unchanged, separately checking the added queue timestamp backfill and null scheduled slot."
+  ],
+  "regressions": {
+    "e06": {
+      "scenarioSha256": "be0ba3ef17b67140d17266028b257b41bb8ec2d4138beeb051f73681ecfd899b",
+      "historicalHarnessSha256": "c0ab1374a9f46658fcac642dc77095cb6abd39a8d204afe4095484c4303ddb56",
+      "liveHarnessSha256": "733ea4d0ead410d21ee129ac13e590321e2efe999c200a78617460d3a44f6f31",
+      "submitEvents": 2, "forwardedCreateRequests": 1, "authoritativeCreatedRows": 1,
+      "heldResponses": 1, "failedUpdates": 1, "failedDeletes": 1,
+      "failedMutationsPreservedAuthoritativeState": true
+    },
+    "e07": "Fixed terminal rows/cursors, URL back/forward/reload and stale-response assertions pass unchanged.",
+    "e08": {
+      "identicalInitialMainText": true, "identicalHeadingStructure": true, "initialDataReads": 0,
+      "hydrationErrors": 0, "pageErrors": 0, "consoleErrors": 0,
+      "privacy": "Alice/Bob/anonymous fixed HTML/RSC checks pass",
+      "axeViolations": { "login": 0, "list": 0, "detail": 0 },
+      "incomplete": "Existing native history-listbox color-contrast incomplete remains on detail; no manual contrast verdict claimed."
+    }
+  },
+  "budget": {
+    "loadRuns": 0, "automaticRetries": 0, "parameterSweeps": 0,
+    "unchangedStartBaselines": 1, "schedulerScenarioInvocations": 2,
+    "heldWorkerBrowserScenarioInvocations": 1, "fullProductionBrowserSuiteInvocations": 1,
+    "productionBuildInvocations": 2, "typecheckInvocations": 1, "unitInvocations": 1,
+    "apiInvocations": 1, "postgresIntegrationInvocations": 1,
+    "failedVerificationInvocations": 0, "diagnosticScenarioReplays": 0,
+    "note": "Scheduler ran once as a cheap authoring target and once in the complete E09 gate. Successful-state polling is observation, not a retry of failed work."
+  },
+  "scope": {
+    "existingPinsAndLockfileUnchanged": true, "earlierMigrationsUnchanged": true,
+    "earlierEvidenceUnchanged": true, "specRevisionUnchanged": true,
+    "mainIndexSpecTagsUntouchedByAgent": true,
+    "newDependencies": [], "queueStore": "PostgreSQL only",
+    "noIdempotencyCompetingWorkerProofLeaseOrRecovery": true,
+    "credentialsTracesScreenshotsVideosStorageStateRetained": false
+  },
+  "cleanup": {
+    "portsFree": [4311, 4312, 4313, 4314], "residualTestSchemas": [],
+    "ownedWorkerAndNextExitAwaited": true, "e09ForcedKills": 0,
+    "publicAndVolumesNotModified": true, "postgresLeftRunning": true,
+    "report": "cleanup.json"
+  },
+  "unresolved": "No E09 acceptance failure. E08's reported native-listbox contrast incomplete remains."
+}
diff --git a/package.json b/package.json
index a57aea8..c099d60 100644
--- a/package.json
+++ b/package.json
@@ -8,6 +8,7 @@
   "scripts": {
     "dev:api": "node --watch server/main.ts",
     "start:api": "node server/main.ts",
+    "start:worker": "node server/worker.ts",
     "dev:web": "NEXT_TELEMETRY_DISABLED=1 next dev --hostname 127.0.0.1 --port 4313",
     "build": "NEXT_TELEMETRY_DISABLED=1 next build",
     "start:web": "NEXT_TELEMETRY_DISABLED=1 next start --hostname 127.0.0.1 --port 4313",
@@ -20,7 +21,8 @@
     "db:up": "docker compose --project-name wse-fundamentals up -d --wait",
     "db:down": "docker compose --project-name wse-fundamentals down",
     "db:migrate": "node server/migrate.ts",
-    "test:integration": "node --test --test-concurrency=1 test/persistence.test.ts test/storage-*.test.ts"
+    "test:integration": "node --test --test-concurrency=1 test/persistence.test.ts test/storage-*.test.ts",
+    "test:execution": "node --test test/execution.test.ts"
   },
   "dependencies": {
     "fastify": "5.12.1",
diff --git a/test/browser/lifecycle.spec.ts b/test/browser/lifecycle.spec.ts
index a43cf82..71531d4 100644
--- a/test/browser/lifecycle.spec.ts
+++ b/test/browser/lifecycle.spec.ts
@@ -1,8 +1,8 @@
 import { expect } from '@playwright/test';
 import { randomBytes } from 'node:crypto';
 import { mkdir, writeFile } from 'node:fs/promises';
-import { test, submitCredentials } from './session';
-import type { CheckRun, MonitorView } from '../../server/model';
+import { test, submitCredentials, readTerminalCheck } from './session';
+import type { TerminalCheckRun, MonitorView } from '../../server/model';
 import scenario from '../../evidence/E03/scenario.json' with { type: 'json' };
 import authScenario from '../../evidence/E04/scenario.json' with { type: 'json' };
 import ownershipScenario from '../../evidence/E05/scenario.json' with { type: 'json' };
@@ -20,12 +20,16 @@ test('E03 persist A,A,B history, edit, pause, enable and delete through the real
     created.push((await (await response).json()).data);
     await expect(page.getByRole('article', { name: input.name, exact: true })).toBeVisible();
   }
-  const checks: CheckRun[] = [];
+  const checks: TerminalCheckRun[] = [];
   for (const expected of scenario.checkSequence) {
     const monitor = page.getByRole('article', { name: scenario.monitors[expected.monitor].name, exact: true });
     const response = page.waitForResponse((response) => response.url().endsWith(`/api/monitors/${created[expected.monitor].id}/checks`) && response.request().method() === 'POST');
     await monitor.getByRole('button', { name: 'Run check', exact: true }).click();
-    const result: CheckRun = (await (await response).json()).data;
+    const accepted = await response;
+    expect(accepted.status()).toBe(202);
+    const queued = (await accepted.json()).data;
+    expect(queued.state).toBe('QUEUED');
+    const result = await readTerminalCheck(page, queued.id);
     checks.push(result);
     expect(result.state).toBe(expected.state);
     expect(result.httpStatus).toBe(expected.httpStatus);
@@ -158,7 +162,7 @@ test.describe('E05 real browser ownership and CSRF', () => {
     const bobContext = await browser.newContext({ baseURL: ownershipScenario.browserOrigin });
     const bob = await bobContext.newPage();
     const monitors: MonitorView[] = [];
-    const checks: CheckRun[] = [];
+    const checks: TerminalCheckRun[] = [];
     try {
       for (const [index, current] of [page, bob].entries()) {
         await current.goto('/login');
@@ -180,9 +184,11 @@ test.describe('E05 real browser ownership and CSRF', () => {
         const checkedResponse = current.waitForResponse((response) => response.url().endsWith(`/api/monitors/${monitors[index].id}/checks`) && response.request().method() === 'POST');
         await article.getByRole('button', { name: 'Run check', exact: true }).click();
         const checked = await checkedResponse;
-        expect(checked.status()).toBe(200);
+        expect(checked.status()).toBe(202);
         expect(Boolean(await checked.request().headerValue('x-csrf-token'))).toBe(true);
-        checks.push((await checked.json()).data);
+        const queued = (await checked.json()).data;
+        expect(queued.state).toBe('QUEUED');
+        checks.push(await readTerminalCheck(current, queued.id));
       }
       for (const [index, current] of [page, bob].entries()) {
         const other = 1 - index;
diff --git a/test/browser/server-state-scenario.ts b/test/browser/server-state-scenario.ts
new file mode 100644
index 0000000..c58da01
--- /dev/null
+++ b/test/browser/server-state-scenario.ts
@@ -0,0 +1,189 @@
+import { expect } from '@playwright/test';
+import type { Page } from '@playwright/test';
+import type { MonitorView } from '../../server/model.ts';
+import scenario from '../../evidence/E06/scenario.json' with { type: 'json' };
+import { readTerminalCheck } from './session.ts';
+
+type Counters = { csrfReads: number; createRequests: number };
+
+async function createMonitor(page: Page, input: typeof scenario.monitors[number]): Promise<MonitorView> {
+  await page.getByLabel('Name', { exact: true }).fill(input.name);
+  await page.getByLabel('Endpoint URL', { exact: true }).fill(input.url);
+  await page.getByLabel('Interval (seconds)', { exact: true }).fill(String(input.interval));
+  await page.getByLabel('Enabled', { exact: true }).check();
+  const response = page.waitForResponse((item) => item.url().endsWith('/api/monitors') && item.request().method() === 'POST');
+  await page.getByRole('button', { name: 'Create monitor', exact: true }).click();
+  const received = await response;
+  expect(received.status()).toBe(201);
+  const monitor: MonitorView = (await received.json()).data;
+  await expect(page.getByRole('article', { name: input.name, exact: true })).toBeVisible();
+  return monitor;
+}
+
+export async function runServerStateScenario(page: Page, mode: 'baseline' | 'verification') {
+  const started = performance.now();
+  await page.goto('/monitors');
+  const a = await createMonitor(page, scenario.monitors[0]);
+  const b = await createMonitor(page, scenario.monitors[1]);
+  let bArticle = page.getByRole('article', { name: b.name, exact: true });
+  await bArticle.getByRole('button', { name: 'View history', exact: true }).click();
+  await expect(bArticle.getByText('No historical checks.', { exact: true })).toBeVisible();
+  await bArticle.getByRole('button', { name: 'Edit monitor', exact: true }).click();
+  await bArticle.getByLabel('Edit name', { exact: true }).fill(scenario.update.name);
+  await bArticle.getByLabel('Edit interval (seconds)', { exact: true }).fill(String(scenario.update.interval));
+  await bArticle.getByRole('button', { name: 'Save changes', exact: true }).click();
+  bArticle = page.getByRole('article', { name: scenario.update.name, exact: true });
+  await expect(bArticle).toContainText('Enabled · 90 seconds');
+  await expect(bArticle.getByRole('region', { name: `Check history for ${scenario.update.name}` })).toBeVisible();
+  await bArticle.getByRole('button', { name: 'Hide history', exact: true }).click();
+  await bArticle.getByRole('button', { name: 'View history', exact: true }).click();
+  await expect(bArticle.getByText('No historical checks.', { exact: true })).toBeVisible();
+  await page.reload();
+  await expect(bArticle).toContainText('Enabled · 90 seconds');
+  await bArticle.getByRole('button', { name: 'Pause monitor', exact: true }).click();
+  await expect(bArticle).toContainText('Paused · 90 seconds');
+  await bArticle.getByRole('button', { name: 'Enable monitor', exact: true }).click();
+  await expect(bArticle).toContainText('Enabled · 90 seconds');
+  await bArticle.getByRole('button', { name: 'View history', exact: true }).click();
+  await expect(bArticle.getByText('No historical checks.', { exact: true })).toBeVisible();
+  page.once('dialog', (dialog) => dialog.accept());
+  await bArticle.getByRole('button', { name: 'Delete monitor', exact: true }).click();
+  await expect(bArticle).toHaveCount(0);
+  await page.reload();
+  await expect(bArticle).toHaveCount(0);
+  expect((await page.request.get(`/api/monitors/${b.id}`)).status()).toBe(404);
+  expect((await page.request.get(`/api/monitors/${b.id}/checks`)).status()).toBe(404);
+  const aArticle = page.getByRole('article', { name: a.name, exact: true });
+  await aArticle.getByRole('button', { name: 'View history', exact: true }).click();
+  await expect(aArticle.getByText('No historical checks.', { exact: true })).toBeVisible();
+  const checked = page.waitForResponse((item) => item.url().endsWith(`/api/monitors/${a.id}/checks`) && item.request().method() === 'POST');
+  await aArticle.getByRole('button', { name: 'Run check', exact: true }).click();
+  const accepted = await checked;
+  expect(accepted.status()).toBe(202);
+  const queued = (await accepted.json()).data;
+  expect(queued.state).toBe('QUEUED');
+  const check = await readTerminalCheck(page, queued.id);
+  expect(check.state).toBe('SUCCEEDED');
+  expect(check.httpStatus).toBe(200);
+  await expect(aArticle.locator('dl')).toContainText('SUCCEEDED');
+  await expect(aArticle.locator('tbody tr')).toHaveCount(1);
+  await expect(aArticle.locator('tbody')).toContainText(check.id);
+  await aArticle.getByRole('button', { name: 'Hide history', exact: true }).click();
+  await aArticle.getByRole('button', { name: 'View history', exact: true }).click();
+  await expect(aArticle.locator('tbody')).toContainText(check.id);
+
+  // Count only operation names, never credentials, headers, or request bodies.
+  await page.evaluate(() => {
+    const state = window as typeof window & { e06Counters: Counters };
+    state.e06Counters = { csrfReads: 0, createRequests: 0 };
+    const originalFetch = window.fetch;
+    window.fetch = (...args) => {
+      const path = String(args[0]);
+      if (path === '/api/auth/csrf') state.e06Counters.csrfReads++;
+      if (path === '/api/monitors' && args[1]?.method === 'POST') state.e06Counters.createRequests++;
+      return originalFetch(...args);
+    };
+  });
+  const committed = Promise.withResolvers<void>();
+  const duplicateCommitted = Promise.withResolvers<void>();
+  const release = Promise.withResolvers<void>();
+  const firstDelivered = Promise.withResolvers<void>();
+  let forwardedCreates = 0;
+  let failedUpdates = 0;
+  let failedDeletes = 0;
+  await page.route('**/api/monitors', async (route) => {
+    if (route.request().method() !== 'POST') return route.continue();
+    const index = ++forwardedCreates;
+    const response = await route.fetch();
+    expect(response.status()).toBe(201);
+    if (index === 1) {
+      committed.resolve();
+      await release.promise;
+      await route.fulfill({ response });
+      firstDelivered.resolve();
+    } else {
+      await route.fulfill({ response });
+      duplicateCommitted.resolve();
+    }
+  });
+  try {
+    await page.getByLabel('Name', { exact: true }).fill(scenario.monitors[1].name);
+    await page.getByLabel('Endpoint URL', { exact: true }).fill(scenario.monitors[1].url);
+    await page.getByLabel('Interval (seconds)', { exact: true }).fill(String(scenario.monitors[1].interval));
+    await page.getByRole('button', { name: 'Create monitor', exact: true }).click();
+    await committed.promise;
+    await expect(page.getByRole('button', { name: 'Creating…', exact: true })).toBeDisabled();
+    await expect(page.getByRole('article', { name: scenario.monitors[1].name, exact: true })).toHaveCount(0);
+    const afterSubmit = await page.getByLabel('Name', { exact: true }).evaluate((input: HTMLInputElement) => {
+      input.form!.requestSubmit();
+      return (window as typeof window & { e06Counters: Counters }).e06Counters;
+    });
+    if (mode === 'baseline' && afterSubmit.csrfReads > scenario.barrier.requiredCsrfReads) {
+      await duplicateCommitted.promise;
+      const authoritative: MonitorView[] = (await (await page.request.get('/api/monitors')).json()).data;
+      return {
+        result: 'REPRODUCED', firstDecisiveFailure: 'second submit starts another mutation while the first response is held',
+        sequentialCreateUpdatePauseResumeDeleteAndCheckAlreadyCorrect: true,
+        submitEvents: scenario.barrier.submitEvents, csrfReadsAfterSecondSubmit: afterSubmit.csrfReads,
+        forwardedCreateRequests: forwardedCreates,
+        authoritativeCreatedRows: authoritative.filter((item) => item.name === scenario.monitors[1].name).length,
+        pendingVisibleBeforeSecondSubmit: true, heldResponses: 1,
+        failedUpdates, failedDeletes, durationMs: Math.round(performance.now() - started),
+      };
+    }
+    expect(afterSubmit.csrfReads).toBe(scenario.barrier.requiredCsrfReads);
+    expect(afterSubmit.createRequests).toBe(scenario.barrier.requiredCreateRequests);
+    await expect(page.getByRole('button', { name: 'Creating…', exact: true })).toBeDisabled();
+    const beforeFailures = (await (await page.request.get(`/api/monitors/${a.id}`)).json()).data;
+    const beforeHistory = (await (await page.request.get(`/api/monitors/${a.id}/checks`)).json()).data;
+    await page.route(`**/api/monitors/${a.id}`, async (route) => {
+      const method = route.request().method();
+      if (method === 'PUT') { failedUpdates++; return route.fulfill({ status: scenario.failure.status, json: scenario.failure.body }); }
+      if (method === 'DELETE') { failedDeletes++; return route.fulfill({ status: scenario.failure.status, json: scenario.failure.body }); }
+      return route.continue();
+    });
+    await aArticle.getByRole('button', { name: 'Edit monitor', exact: true }).click();
+    await aArticle.getByLabel('Edit name', { exact: true }).fill(scenario.rejectedUpdate.name);
+    await aArticle.getByLabel('Edit interval (seconds)', { exact: true }).fill(String(scenario.rejectedUpdate.interval));
+    await aArticle.getByRole('button', { name: 'Save changes', exact: true }).click();
+    const alert = page.getByRole('main').getByRole('alert');
+    await expect(alert).toHaveAttribute('data-error-code', 'INTERNAL_ERROR');
+    await expect(aArticle).toContainText('Enabled · 60 seconds');
+    await expect(aArticle.getByLabel('Edit name', { exact: true })).toHaveValue(scenario.rejectedUpdate.name);
+    await expect(aArticle.getByRole('button', { name: 'Save changes', exact: true })).toBeEnabled();
+    await aArticle.getByRole('button', { name: 'Cancel edit', exact: true }).click();
+    page.once('dialog', (dialog) => dialog.accept());
+    await aArticle.getByRole('button', { name: 'Delete monitor', exact: true }).click();
+    await expect(alert).toHaveAttribute('data-error-code', 'INTERNAL_ERROR');
+    await expect(aArticle.getByRole('button', { name: 'Delete monitor', exact: true })).toBeEnabled();
+    await expect(aArticle.locator('tbody tr')).toHaveCount(1);
+    await expect(aArticle.locator('tbody')).toContainText(check.id);
+    expect((await (await page.request.get(`/api/monitors/${a.id}`)).json()).data).toEqual(beforeFailures);
+    expect((await (await page.request.get(`/api/monitors/${a.id}/checks`)).json()).data).toEqual(beforeHistory);
+    expect(failedUpdates).toBe(1);
+    expect(failedDeletes).toBe(1);
+    release.resolve();
+    await firstDelivered.promise;
+    await expect(page.getByRole('button', { name: 'Create monitor', exact: true })).toBeEnabled();
+    await expect(page.getByRole('article', { name: scenario.monitors[1].name, exact: true })).toHaveCount(1);
+    expect(forwardedCreates).toBe(scenario.barrier.requiredCreateRequests);
+    await page.reload();
+    await expect(page.getByRole('article', { name: scenario.monitors[1].name, exact: true })).toHaveCount(1);
+    await expect(aArticle).toContainText('Enabled · 60 seconds');
+    const authoritative: MonitorView[] = (await (await page.request.get('/api/monitors')).json()).data;
+    expect(authoritative.filter((item) => item.name === scenario.monitors[1].name)).toHaveLength(scenario.barrier.requiredCreatedRows);
+    return {
+      result: mode === 'baseline' ? 'NOT_REPRODUCED' : 'PASS',
+      sequentialCreateUpdatePauseResumeDeleteAndCheckAlreadyCorrect: true,
+      submitEvents: scenario.barrier.submitEvents, csrfReadsAfterSecondSubmit: afterSubmit.csrfReads,
+      forwardedCreateRequests: forwardedCreates, authoritativeCreatedRows: scenario.barrier.requiredCreatedRows,
+      pendingVisibleBeforeSecondSubmit: true, heldResponses: 1, failedUpdates, failedDeletes,
+      failedMutationsPreservedAuthoritativeState: true, pendingClearedAfterRelease: true,
+      durationMs: Math.round(performance.now() - started),
+    };
+  } finally {
+    release.resolve();
+    await firstDelivered.promise;
+    await page.unrouteAll({ behavior: 'wait' });
+  }
+}
diff --git a/test/browser/server-state.spec.ts b/test/browser/server-state.spec.ts
index efd6869..c26fe07 100644
--- a/test/browser/server-state.spec.ts
+++ b/test/browser/server-state.spec.ts
@@ -2,7 +2,7 @@ import { expect } from '@playwright/test';
 import { createHash } from 'node:crypto';
 import { mkdir, readFile, writeFile } from 'node:fs/promises';
 import { test } from './session';
-import { runServerStateScenario } from '../../evidence/E06/browser-scenario';
+import { runServerStateScenario } from './server-state-scenario';
 
 test('E06 authoritative list/detail refresh, response barrier, duplicate submit and failed mutations', async ({ page }) => {
   const originalReload = page.reload;
@@ -30,8 +30,10 @@ test('E06 authoritative list/detail refresh, response barrier, duplicate submit
     expect(adaptedReloads).toHaveLength(3);
     const scenarioSha256 = createHash('sha256').update(await readFile('evidence/E06/scenario.json')).digest('hex');
     const harnessSha256 = createHash('sha256').update(await readFile('evidence/E06/browser-scenario.ts')).digest('hex');
+    const liveHarnessSha256 = createHash('sha256').update(await readFile('test/browser/server-state-scenario.ts')).digest('hex');
     await mkdir('output/e06', { recursive: true });
-    await writeFile('output/e06/browser.json', JSON.stringify({ scenarioSha256, harnessSha256, ...result }, null, 2) + '\n');
+    await writeFile('output/e06/browser.json', JSON.stringify({ scenarioSha256, harnessSha256, liveHarnessSha256,
+      liveHarness: 'test/browser/server-state-scenario.ts', asyncCheckConsumer: 'real202 QUEUED then same-ID worker completion', ...result }, null, 2) + '\n');
     await mkdir('output/e07', { recursive: true });
     await writeFile('output/e07/legacy-browser-adapter.json', JSON.stringify({
       result: 'PASS', scenarioSha256, harnessSha256, adaptedReloadCount: adaptedReloads.length,
diff --git a/test/browser/session.ts b/test/browser/session.ts
index af22469..e3bbf4f 100644
--- a/test/browser/session.ts
+++ b/test/browser/session.ts
@@ -1,12 +1,22 @@
-import { test as base } from '@playwright/test';
+import { test as base, expect } from '@playwright/test';
 import type { Page } from '@playwright/test';
 import { prepareTestUsers } from '../auth.ts';
 import type { TestCredentials } from '../auth.ts';
 import { testDatabaseConfig } from '../database.ts';
 import { DEFAULT_BROWSER_ORIGIN } from '../../server/auth.ts';
+import { startTestWorker } from '../worker.ts';
+import type { CheckRun, TerminalCheckRun } from '../../server/model.ts';
 
-export const test = base.extend<{ autoLogin: boolean; authenticate: void }, { users: TestCredentials[] }>({
+export const test = base.extend<{ autoLogin: boolean; authenticate: void }, { users: TestCredentials[]; checkWorker: void }>({
   autoLogin: [true, { option: true }],
+  checkWorker: [async ({}, use) => {
+    const worker = await startTestWorker(testDatabaseConfig('e03_browser'));
+    try { await use(); }
+    finally {
+      const stopped = await worker.stop();
+      if (stopped.forced || stopped.code === 1) throw new Error('The browser check worker did not stop cleanly.');
+    }
+  }, { scope: 'worker', auto: true }],
   users: [async ({}, use) => { await use(await prepareTestUsers(testDatabaseConfig('e03_browser'))); }, { scope: 'worker' }],
   authenticate: [async ({ autoLogin, page, users }, use) => {
     if (autoLogin) {
@@ -28,3 +38,15 @@ export async function submitCredentials(page: Page, credentials: TestCredentials
     await page.getByRole('button', { name: 'Sign in', exact: true }).click();
   } catch { throw new Error('Browser sign-in action failed; credential details are suppressed.'); }
 }
+
+export async function readTerminalCheck(page: Page, id: string): Promise<TerminalCheckRun> {
+  let check: CheckRun | undefined;
+  await expect.poll(async () => {
+    const response = await page.request.get(`/api/checks/${id}`);
+    expect(response.status()).toBe(200);
+    check = (await response.json()).data;
+    expect(check!.id).toBe(id);
+    return check!.state;
+  }, { timeout: 10000 }).toMatch(/^(SUCCEEDED|FAILED)$/);
+  return check as TerminalCheckRun;
+}
diff --git a/test/contracts.test.ts b/test/contracts.test.ts
index f6db5e6..ba43ef7 100644
--- a/test/contracts.test.ts
+++ b/test/contracts.test.ts
@@ -8,7 +8,8 @@ import { authenticatedFetch, authenticatedInject, cookieFromHeader, csrfForTest,
 import { SESSION_COOKIE_NAME, sessionTokenFromCookie, sessionTokenHash } from '../server/auth.ts';
 import { databasePool } from '../server/database.ts';
 import { dropTestSchema, resetTestSchema, testDatabaseConfig } from './database.ts';
-import type { ApiErrorCode, ApiSuccess, CheckRun, MonitorView } from '../server/model.ts';
+import type { ApiErrorCode, ApiSuccess, CheckRun, MonitorView, TerminalCheckRun } from '../server/model.ts';
+import { startTestWorker, waitForTerminalCheck } from './worker.ts';
 import scenario from '../evidence/E02/scenario.json' with { type: 'json' };
 import authScenario from '../evidence/E04/scenario.json' with { type: 'json' };
 import ownershipScenario from '../evidence/E05/scenario.json' with { type: 'json' };
@@ -16,6 +17,8 @@ import ownershipScenario from '../evidence/E05/scenario.json' with { type: 'json
 const fixture = fixtureServer();
 const database = testDatabaseConfig('e03_contracts');
 const app = buildApp(scenario.fixtureOrigin, database);
+const contractPool = databasePool(database);
+let worker: Awaited<ReturnType<typeof startTestWorker>>;
 const boundaries = scenario.additionalBoundaries;
 let request: ReturnType<typeof authenticatedFetch>;
 app.get(boundaries.internalErrorRoute, async () => { throw new Error(boundaries.privateInternalMessage); });
@@ -25,10 +28,13 @@ before(async () => {
   request = authenticatedFetch(await loginForTest(app, (await prepareTestUsers(database))[0]));
   await new Promise<void>((resolve) => fixture.server.listen(4311, '127.0.0.1', resolve));
   await app.listen({ host: '127.0.0.1', port: 4312 });
+  worker = await startTestWorker(database);
 });
 
 after(async () => {
+  await worker?.stop();
   await app.close();
+  await contractPool.end();
   await dropTestSchema(database.schema);
   fixture.server.closeAllConnections();
   await new Promise<void>((resolve) => fixture.server.close(() => resolve()));
@@ -156,7 +162,10 @@ test('E02 terminal CheckRun wire values distinguish observed HTTP failure from n
   for (const expected of scenario.terminalResults) {
     const monitor = await success<MonitorView>(await postMonitor({ ...scenario.monitor, url: `${scenario.fixtureOrigin}${expected.path}` }), 201);
     assertMonitor(monitor);
-    const result = await success<CheckRun>(await request(`${scenario.apiOrigin}/monitors/${monitor.id}/checks`, { method: 'POST' }), expected.apiStatus);
+    const queued = await success<CheckRun>(await request(`${scenario.apiOrigin}/monitors/${monitor.id}/checks`, { method: 'POST' }), 202);
+    assert.equal(queued.state, 'QUEUED');
+    await waitForTerminalCheck(contractPool, queued.id);
+    const result = await success<TerminalCheckRun>(await request(`${scenario.apiOrigin}/checks/${queued.id}`), expected.apiStatus);
     assert.deepEqual(Object.keys(result).sort(), ['failureReason', 'finishedAt', 'httpStatus', 'id', 'latencyMs', 'monitorId', 'startedAt', 'state', 'trigger']);
     assert.match(result.id, /^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/);
     assert.equal(result.monitorId, monitor.id);
@@ -302,6 +311,7 @@ test('E05 fixed A/B ownership matrix: collection, nested/direct reads and every
   const config = await resetTestSchema('e05_authorization');
   const pool = databasePool(config);
   const ownerApp = buildApp(ownershipScenario.fixtureOrigin, config);
+  let ownerWorker: Awaited<ReturnType<typeof startTestWorker>> | undefined;
   const observations: { operation: string; status: number; noWrite?: boolean; noOutbound?: boolean }[] = [];
   const snapshot = async () => ({
     monitors: (await pool.query('SELECT * FROM monitors ORDER BY id')).rows,
@@ -309,6 +319,7 @@ test('E05 fixed A/B ownership matrix: collection, nested/direct reads and every
     calls: Object.fromEntries(fixture.calls),
   });
   try {
+    ownerWorker = await startTestWorker(config);
     const users = await prepareTestUsers(config);
     const owners: ReturnType<typeof authenticatedInject>[] = [];
     const cookies: string[] = [];
@@ -322,8 +333,13 @@ test('E05 fixed A/B ownership matrix: collection, nested/direct reads and every
       assert.equal(created.statusCode, 201);
       monitors.push(created.json().data);
       const checked = await owner({ method: 'POST', url: `/monitors/${monitors[i].id}/checks` });
-      assert.equal(checked.statusCode, 200);
-      checks.push(checked.json().data);
+      assert.equal(checked.statusCode, 202);
+      const queued = checked.json().data;
+      assert.equal(queued.state, 'QUEUED');
+      await waitForTerminalCheck(pool, queued.id);
+      const terminal = await owner(`/checks/${queued.id}`);
+      assert.equal(terminal.statusCode, 200);
+      checks.push(terminal.json().data);
     }
     assert.deepEqual(checks.map(({ state, httpStatus }) => ({ state, httpStatus })), [
       { state: 'SUCCEEDED', httpStatus: 200 }, { state: 'FAILED', httpStatus: 503 },
@@ -481,5 +497,5 @@ test('E05 fixed A/B ownership matrix: collection, nested/direct reads and every
       rotationChangesCsrf: true, oldCsrfRejectedByNewSession: true, logoutRevoked: true, csrfAloneUnauthenticated: true,
       durationMs: Math.round(performance.now() - started),
     }, null, 2) + '\n');
-  } finally { await ownerApp.close(); await pool.end(); await dropTestSchema(config.schema); }
+  } finally { await ownerWorker?.stop(); await ownerApp.close(); await pool.end(); await dropTestSchema(config.schema); }
 });
diff --git a/test/execution.test.ts b/test/execution.test.ts
new file mode 100644
index 0000000..2e3293b
--- /dev/null
+++ b/test/execution.test.ts
@@ -0,0 +1,174 @@
+import { test } from 'node:test';
+import assert from 'node:assert/strict';
+import { access, mkdir, writeFile } from 'node:fs/promises';
+import { createServer } from 'node:net';
+import { spawn } from 'node:child_process';
+import { chromium, expect } from '@playwright/test';
+import { buildApp } from '../server/app.ts';
+import { databaseConfig, databasePool, schemaIdentifier } from '../server/database.ts';
+import { migrate } from '../server/migrate.ts';
+import { scheduleDueChecks } from '../server/worker.ts';
+import { heldFixture, insertExecutionFixture } from '../evidence/E09/fixture.ts';
+import { startTestWorker, waitForTerminalCheck } from './worker.ts';
+import scenario from '../evidence/E09/scenario.json' with { type: 'json' };
+
+test('E09 fixed scheduler ticks persist one intent per enabled interval slot', async () => {
+  const began = performance.now();
+  const config = { ...databaseConfig(), schema: 'e09_scheduler' };
+  const pool = databasePool(config);
+  let owned = false;
+  try {
+    await pool.query(`CREATE SCHEMA ${schemaIdentifier(config.schema)}`);
+    owned = true;
+    await migrate(config);
+    await insertExecutionFixture(pool);
+    const counts = [];
+    for (const [index, tick] of scenario.scheduler.ticks.entries()) {
+      await scheduleDueChecks(pool, new Date(tick));
+      const rows = (await pool.query('SELECT monitor_id, state, trigger, scheduled_for FROM check_runs ORDER BY scheduled_for')).rows;
+      const a = rows.filter((row) => row.monitor_id === scenario.monitors[0].id);
+      const b = rows.filter((row) => row.monitor_id === scenario.monitors[1].id);
+      assert.equal(a.length, scenario.scheduler.expectedScheduledA[index]);
+      assert.equal(b.length, scenario.scheduler.expectedScheduledB[index]);
+      assert.ok(a.every((row) => row.state === 'QUEUED' && row.trigger === 'SCHEDULED'));
+      counts.push({ tick, a: a.length, b: b.length, slots: a.map((row) => row.scheduled_for.toISOString()) });
+    }
+    await mkdir('output/e09', { recursive: true });
+    await writeFile('output/e09/scheduler.json', JSON.stringify({ result: 'PASS', workerStopped: true, counts, durationMs: Math.round(performance.now() - began) }, null, 2) + '\n');
+  } finally {
+    if (owned) await pool.query(`DROP SCHEMA ${schemaIdentifier(config.schema)} CASCADE`);
+    await pool.end();
+  }
+});
+
+test('E09 persisted202, separate worker and browser follow one held execution', async () => {
+  const began = performance.now();
+  await access('.next/BUILD_ID');
+  const config = { ...databaseConfig(), schema: scenario.schema };
+  const pool = databasePool(config);
+  const fixture = heldFixture();
+  const app = buildApp(undefined, config);
+  let owned = false;
+  let worker: Awaited<ReturnType<typeof startTestWorker>> | undefined;
+  let web: ReturnType<typeof spawn> | undefined;
+  let webExited: Promise<{ code: number | null; signal: string | null }> | undefined;
+  let browser: Awaited<ReturnType<typeof chromium.launch>> | undefined;
+  const report: Record<string, unknown> = { result: 'FAILED', productionBuild: true, states: [] };
+  async function requireFreePorts() {
+    for (const port of scenario.ports) {
+      const server = createServer();
+      await new Promise<void>((resolve, reject) => { server.once('error', reject); server.listen(port, '127.0.0.1', resolve); });
+      await new Promise<void>((resolve, reject) => server.close(error => error ? reject(error) : resolve()));
+    }
+  }
+  try {
+    await requireFreePorts();
+    await pool.query(`CREATE SCHEMA ${schemaIdentifier(config.schema)}`);
+    owned = true;
+    await migrate(config);
+    const credentials = await insertExecutionFixture(pool);
+    await new Promise<void>((resolve, reject) => { fixture.server.once('error', reject); fixture.server.listen(4311, '127.0.0.1', resolve); });
+    await app.listen({ host: '127.0.0.1', port: 4312 });
+    web = spawn(process.execPath, ['node_modules/next/dist/bin/next', 'start', '--hostname', '127.0.0.1', '--port', '4313'],
+      { env: { ...process.env, NEXT_TELEMETRY_DISABLED: '1' }, stdio: ['ignore', 'pipe', 'pipe'] });
+    webExited = new Promise(resolve => web!.once('close', (code, signal) => resolve({ code, signal })));
+    await new Promise<void>((resolve, reject) => {
+      const timer = setTimeout(() => reject(new Error('Production Next readiness timed out.')), scenario.watchdogMs);
+      web!.stdout!.on('data', chunk => { if (String(chunk).includes('Ready in')) { clearTimeout(timer); resolve(); } });
+      web!.once('error', () => { clearTimeout(timer); reject(new Error('Production Next could not start.')); });
+      web!.once('exit', () => { clearTimeout(timer); reject(new Error('Production Next exited before readiness.')); });
+    });
+    browser = await chromium.launch();
+    const context = await browser.newContext({ baseURL: 'http://127.0.0.1:4313' });
+    try {
+      const login = await context.request.post('/api/auth/login', { headers: { origin: 'http://127.0.0.1:4313' }, data: credentials });
+      assert.equal(login.status(), 200);
+    } catch { throw new Error('E09 browser login failed; credential details suppressed.'); }
+    const page = await context.newPage();
+    const errors = { page: 0, console: 0 };
+    page.on('pageerror', () => errors.page++);
+    page.on('console', message => { if (message.type() === 'error') errors.console++; });
+    await page.goto(scenario.manual.browserPrimaryRoute);
+    const a = page.getByRole('article', { name: scenario.monitors[0].name, exact: true });
+    const b = page.getByRole('article', { name: scenario.monitors[1].name, exact: true });
+    const accepted = page.waitForResponse(response => response.url().endsWith(`/api${scenario.manual.path}`) && response.request().method() === 'POST');
+    await a.getByRole('button', { name: 'Run check', exact: true }).click();
+    const response = await accepted;
+    assert.equal(response.status(), scenario.manual.expectedStatus);
+    const queued = (await response.json()).data;
+    assert.equal(queued.state, 'QUEUED');
+    assert.equal(queued.httpStatus, null);
+    assert.equal(queued.startedAt, null);
+    assert.equal(queued.finishedAt, null);
+    const queuedRow = (await pool.query('SELECT state, http_status, started_at, finished_at FROM check_runs WHERE id = $1', [queued.id])).rows[0];
+    assert.deepEqual(queuedRow, { state: 'QUEUED', http_status: null, started_at: null, finished_at: null });
+    assert.equal(fixture.calls.get('/hold') ?? 0, 0);
+    await expect(a.locator('dl')).toContainText('QUEUED');
+    report.beforeWorker = { status: response.status(), id: queued.id, ...queuedRow, outboundCalls: 0, visibleState: 'QUEUED' };
+    worker = await startTestWorker(config);
+    assert.notEqual(worker.pid, process.pid);
+    let heldTimer: ReturnType<typeof setTimeout> | undefined;
+    try {
+      await Promise.race([fixture.entered, new Promise((_, reject) => { heldTimer = setTimeout(() => reject(new Error('Worker did not reach the held fixture.')), scenario.watchdogMs); })]);
+    } finally { clearTimeout(heldTimer); }
+    const running = (await context.request.get(`/api/checks/${queued.id}`));
+    assert.equal(running.status(), 200);
+    const runningData = (await running.json()).data;
+    assert.equal(runningData.id, queued.id);
+    assert.equal(runningData.state, 'RUNNING');
+    assert.equal(runningData.httpStatus, null);
+    assert.equal(runningData.finishedAt, null);
+    assert.equal(typeof runningData.startedAt, 'string');
+    await expect(a.locator('dl')).toContainText('RUNNING');
+    await b.getByRole('button', { name: 'Edit monitor', exact: true }).click();
+    await b.getByLabel('Edit name', { exact: true }).fill('Local draft while A runs');
+    await expect(b.getByLabel('Edit name', { exact: true })).toHaveValue('Local draft while A runs');
+    assert.equal((await pool.query('SELECT state FROM check_runs WHERE id = $1', [queued.id])).rows[0].state, 'RUNNING');
+    assert.equal(fixture.calls.get('/hold'), 1);
+    report.whileHeld = { apiPid: process.pid, workerPid: worker.pid, sameId: runningData.id === queued.id, state: runningData.state,
+      httpStatus: runningData.httpStatus, finishedAt: runningData.finishedAt, outboundCalls: 1, browserUsable: true };
+    fixture.release();
+    const terminal = await waitForTerminalCheck(pool, queued.id);
+    assert.equal(terminal.state, scenario.manual.afterRelease.state);
+    assert.equal(terminal.httpStatus, scenario.manual.afterRelease.httpStatus);
+    assert.equal(terminal.id, queued.id);
+    await expect(a.locator('dl')).toContainText('SUCCEEDED');
+    await expect(a.locator('tbody tr td:first-child')).toHaveText([queued.id]);
+    const direct = await context.request.get(`/api/checks/${queued.id}`);
+    assert.deepEqual((await direct.json()).data, terminal);
+    assert.equal(fixture.calls.get('/hold'), 1);
+    assert.equal(fixture.calls.get('/ok') ?? 0, 0);
+    assert.equal((await pool.query('SELECT count(*)::int AS count FROM check_runs')).rows[0].count, 1);
+    assert.deepEqual(errors, { page: 0, console: 0 });
+    report.states = ['QUEUED', 'RUNNING', terminal.state];
+    report.afterRelease = { ...terminal, outboundCalls: 1, historyCount: 1 };
+    report.errors = errors;
+    report.result = 'PASS';
+  } catch (error) {
+    report.failure = error instanceof Error ? error.message : 'E09 execution failed.';
+    throw error;
+  } finally {
+    fixture.release();
+    if (browser) await browser.close();
+    if (worker) report.workerStop = await worker.stop();
+    if (web && webExited) {
+      let forced = false;
+      const timer = setTimeout(() => { forced = true; web!.kill('SIGKILL'); }, scenario.watchdogMs);
+      web.kill('SIGTERM');
+      report.webStop = { ...await webExited, forced };
+      clearTimeout(timer);
+    }
+    await app.close();
+    if (fixture.server.listening) {
+      fixture.server.closeAllConnections();
+      await new Promise<void>(resolve => fixture.server.close(() => resolve()));
+    }
+    if (owned) await pool.query(`DROP SCHEMA ${schemaIdentifier(config.schema)} CASCADE`);
+    await pool.end();
+    await requireFreePorts();
+    report.cleanup = { schemaDropped: owned, portsFree: true };
+    report.durationMs = Math.round(performance.now() - began);
+    await mkdir('output/e09', { recursive: true });
+    await writeFile('output/e09/execution.json', JSON.stringify(report, null, 2) + '\n');
+  }
+});
diff --git a/test/functional.test.ts b/test/functional.test.ts
index bedc737..b96fa13 100644
--- a/test/functional.test.ts
+++ b/test/functional.test.ts
@@ -7,10 +7,14 @@ import type { ApiSuccess, Monitor, MonitorView } from '../server/model.ts';
 import { fixtureServer } from './fixture.ts';
 import { authenticatedInject, loginForTest, prepareTestUsers } from './auth.ts';
 import { dropTestSchema, resetTestSchema, testDatabaseConfig } from './database.ts';
+import { databasePool } from '../server/database.ts';
+import { startTestWorker, waitForTerminalCheck } from './worker.ts';
 
 const fixture = fixtureServer();
 const database = testDatabaseConfig('e03_functional');
 const app = buildApp(DEFAULT_FIXTURE_ORIGIN, database);
+const pool = databasePool(database);
+let worker: Awaited<ReturnType<typeof startTestWorker>>;
 let cookie: string;
 let inject: ReturnType<typeof authenticatedInject>;
 let guardCalls = 0;
@@ -22,9 +26,12 @@ before(async () => {
   inject = authenticatedInject(app, cookie);
   await new Promise<void>((resolve) => fixture.server.listen(4311, '127.0.0.1', resolve));
   await new Promise<void>((resolve) => guard.listen(4314, '127.0.0.1', resolve));
+  worker = await startTestWorker(database);
 });
 after(async () => {
+  await worker?.stop();
   await app.close();
+  await pool.end();
   await dropTestSchema(database.schema);
   fixture.server.closeAllConnections();
   guard.closeAllConnections();
@@ -42,14 +49,23 @@ async function create(path: string): Promise<MonitorView> {
   return response.json<ApiSuccess<MonitorView>>().data;
 }
 
-test('create a Monitor in PostgreSQL and synchronously observe GET /ok', async () => {
+async function check(id: string) {
+  const response = await inject({ method: 'POST', url: `/monitors/${id}/checks` });
+  assert.equal(response.statusCode, 202);
+  const queued = response.json().data;
+  assert.equal(queued.state, 'QUEUED');
+  await waitForTerminalCheck(pool, queued.id);
+  const terminal = await inject(`/checks/${queued.id}`);
+  assert.equal(terminal.statusCode, 200);
+  return terminal.json().data;
+}
+
+test('create a Monitor in PostgreSQL and observe the queued worker GET /ok', async () => {
   const monitor = await create('/ok');
   assert.equal(monitor.interval, 60);
   assert.equal(monitor.enabled, true);
   assert.equal(monitor.latestCheck, null);
-  const response = await inject({ method: 'POST', url: `/monitors/${monitor.id}/checks` });
-  assert.equal(response.statusCode, 200);
-  const result = response.json().data;
+  const result = await check(monitor.id);
   assert.equal(result.state, 'SUCCEEDED');
   assert.equal(result.httpStatus, 200);
   assert.equal(result.trigger, 'MANUAL');
@@ -63,7 +79,7 @@ test('create a Monitor in PostgreSQL and synchronously observe GET /ok', async (
 
 test('GET /fail is an observed endpoint failure with HTTP 503', async () => {
   const monitor = await create('/fail');
-  const result = (await inject({ method: 'POST', url: `/monitors/${monitor.id}/checks` })).json().data;
+  const result = await check(monitor.id);
   assert.equal(result.state, 'FAILED');
   assert.equal(result.httpStatus, 503);
   assert.equal(result.failureReason, 'HTTP_STATUS');
@@ -73,7 +89,7 @@ test('GET /fail is an observed endpoint failure with HTTP 503', async () => {
 test('the check does not follow even a same-origin redirect', async () => {
   const previousOkCalls = fixture.calls.get('/ok');
   const monitor = await create('/redirect');
-  const result = (await inject({ method: 'POST', url: `/monitors/${monitor.id}/checks` })).json().data;
+  const result = await check(monitor.id);
   assert.equal(result.state, 'FAILED');
   assert.equal(result.httpStatus, 302);
   assert.equal(fixture.calls.get('/ok'), previousOkCalls);
diff --git a/test/persistence.test.ts b/test/persistence.test.ts
index b576e16..a395f7b 100644
--- a/test/persistence.test.ts
+++ b/test/persistence.test.ts
@@ -8,7 +8,8 @@ import { buildApp } from '../server/app.ts';
 import { databasePool, schemaIdentifier } from '../server/database.ts';
 import { monitorFromRow, checkRunFromRow, monitorToValues, checkRunToValues } from '../server/mapping.ts';
 import type { MonitorRow, CheckRunRow } from '../server/mapping.ts';
-import type { CheckHistoryPage, CheckRun, Monitor, MonitorView } from '../server/model.ts';
+import type { CheckHistoryPage, CheckRun, Monitor, MonitorView, TerminalCheckRun } from '../server/model.ts';
+import { startTestWorker, waitForTerminalCheck } from './worker.ts';
 import { fixtureServer } from './fixture.ts';
 import { authenticatedFetch, authenticatedInject, loginForTest, prepareTestUsers } from './auth.ts';
 import authScenario from '../evidence/E04/scenario.json' with { type: 'json' };
@@ -53,9 +54,9 @@ test('E03 migration CLI applies the fresh chain and a repeated command changes n
   try {
     const options = { cwd: process.cwd(), env: { ...process.env, DATABASE_URL: config.connectionString, DATABASE_SCHEMA: config.schema }, timeout: 10_000 };
     const first = await execute(process.execPath, ['server/migrate.ts'], options);
-    assert.deepEqual(JSON.parse(first.stdout).applied, [...authScenario.additionalAssertions.expectedMigrationFiles, '004_monitor_ownership.sql', '005_check_history_index.sql']);
+    assert.deepEqual(JSON.parse(first.stdout).applied, [...authScenario.additionalAssertions.expectedMigrationFiles, '004_monitor_ownership.sql', '005_check_history_index.sql', '006_check_queue.sql']);
     const before = (await pool.query('SELECT version, checksum, applied_at FROM schema_migrations ORDER BY version')).rows;
-    assert.equal(before.length, authScenario.additionalAssertions.expectedMigrationFiles.length + 2);
+    assert.equal(before.length, authScenario.additionalAssertions.expectedMigrationFiles.length + 3);
     for (const row of before) { assert.match(row.checksum, /^[0-9a-f]{64}$/); assert.ok(row.applied_at instanceof Date); }
     const second = await execute(process.execPath, ['server/migrate.ts'], options);
     assert.deepEqual(JSON.parse(second.stdout).applied, scenario.additionalAssertions.repeatMigrationExpectedApplied);
@@ -93,15 +94,20 @@ test('E03 A,A,B survive a fresh application; all Monitor lifecycle and CheckRun
   const pool = databasePool(config);
   const fixture = fixtureServer();
   let app = buildApp(scenario.fixtureOrigin, config);
+  let worker: Awaited<ReturnType<typeof startTestWorker>> | undefined;
   try {
     cookie = await loginForTest(app, (await prepareTestUsers(config))[0]);
     await new Promise<void>((resolve) => fixture.server.listen(4311, '127.0.0.1', resolve));
     await app.listen({ host: '127.0.0.1', port: 4312 });
+    worker = await startTestWorker(config);
     const monitors: MonitorView[] = [];
     for (const input of scenario.monitors) monitors.push(await success<MonitorView>(await request('/monitors', 'POST', input), 201));
     const checks: CheckRun[] = [];
     for (const expected of scenario.checkSequence) {
-      const check = await success<CheckRun>(await request(`/monitors/${monitors[expected.monitor].id}/checks`, 'POST'));
+      const queued = await success<CheckRun>(await request(`/monitors/${monitors[expected.monitor].id}/checks`, 'POST'), 202);
+      assert.equal(queued.state, 'QUEUED');
+      await waitForTerminalCheck(pool, queued.id);
+      const check = await success<TerminalCheckRun>(await request(`/checks/${queued.id}`));
       assert.equal(check.state, expected.state);
       assert.equal(check.httpStatus, expected.httpStatus);
       assert.equal(check.failureReason, expected.failureReason);
@@ -174,6 +180,7 @@ test('E03 A,A,B survive a fresh application; all Monitor lifecycle and CheckRun
     assert.deepEqual(Object.fromEntries(fixture.calls), { '/ok': 2, '/fail': 1 });
     await record('persistence', { databaseSchema: config.schema, beforeRestart: before, afterRestart: after, historicalChecksAfterRestart: histories, lifecycle, deletedMonitor: b.id, deletedCheckRun: checks[2].id, afterDeletion: remaining, counts, fixtureCalls: Object.fromEntries(fixture.calls), durationMs: Math.round(performance.now() - started) });
   } finally {
+    await worker?.stop();
     await app.close();
     await pool.end();
     fixture.server.closeAllConnections();
@@ -256,12 +263,14 @@ test('E05 upgrades existing rows only with explicit owner designation and preser
     finally { await oldApp.close(); }
 
     // Deliberately designate the second account, never the first row returned.
-    assert.deepEqual(await migrate(config, users[1].username), ['004_monitor_ownership.sql', '005_check_history_index.sql']);
+    assert.deepEqual(await migrate(config, users[1].username), ['004_monitor_ownership.sql', '005_check_history_index.sql', '006_check_queue.sql']);
     const owner = (await pool.query('SELECT id FROM users WHERE username = $1', [users[1].username])).rows[0].id;
     const after = (await pool.query('SELECT * FROM monitors ORDER BY id')).rows;
     assert.deepEqual(after.map(({ owner_user_id: _owner, ...row }) => row), before.monitors);
     assert.ok(after.every((row) => row.owner_user_id === owner));
-    assert.deepEqual((await pool.query('SELECT * FROM check_runs ORDER BY id')).rows, before.checks);
+    const upgradedChecks = (await pool.query('SELECT * FROM check_runs ORDER BY id')).rows;
+    assert.deepEqual(upgradedChecks.map(({ queued_at: _queued, scheduled_for: _slot, ...row }) => row), before.checks);
+    assert.ok(upgradedChecks.every((row) => row.queued_at.getTime() === row.started_at.getTime() && row.scheduled_for === null));
     assert.deepEqual((await pool.query('SELECT * FROM schema_migrations ORDER BY version LIMIT 3')).rows, before.migrations);
     await assert.rejects(pool.query('DELETE FROM users WHERE id = $1', [owner]), { code: '23503' });
     const app = buildApp(undefined, config);
diff --git a/test/worker.ts b/test/worker.ts
new file mode 100644
index 0000000..319bb2c
--- /dev/null
+++ b/test/worker.ts
@@ -0,0 +1,47 @@
+import { spawn } from 'node:child_process';
+import { setTimeout as delay } from 'node:timers/promises';
+import assert from 'node:assert/strict';
+import type { Pool } from 'pg';
+import type { DatabaseConfig } from '../server/database.ts';
+import type { CheckRunRow } from '../server/mapping.ts';
+import { checkRunFromRow } from '../server/mapping.ts';
+import type { TerminalCheckRun } from '../server/model.ts';
+
+export async function startTestWorker(config: DatabaseConfig) {
+  const child = spawn(process.execPath, ['server/worker.ts', '--no-schedule'], {
+    env: { ...process.env, DATABASE_URL: config.connectionString, DATABASE_SCHEMA: config.schema },
+    stdio: ['ignore', 'pipe', 'pipe'],
+  });
+  const exited = new Promise<{ code: number | null; signal: string | null }>((resolve) => child.once('close', (code, signal) => resolve({ code, signal })));
+  async function stop() {
+    let forced = false;
+    const timer = setTimeout(() => { forced = true; child.kill('SIGKILL'); }, 10000);
+    child.kill('SIGTERM');
+    const result = await exited;
+    clearTimeout(timer);
+    return { ...result, forced };
+  }
+  try {
+    await new Promise<void>((resolve, reject) => {
+      const timer = setTimeout(() => reject(new Error('Worker readiness timed out.')), 10000);
+      child.stdout.on('data', (chunk) => {
+        if (String(chunk).includes('Check worker ready.')) { clearTimeout(timer); resolve(); }
+      });
+      child.once('error', () => { clearTimeout(timer); reject(new Error('Worker could not start.')); });
+      child.once('exit', () => { clearTimeout(timer); reject(new Error('Worker exited before readiness.')); });
+    });
+  } catch (error) { await stop(); throw error; }
+  return { pid: child.pid!, stop };
+}
+
+export async function waitForTerminalCheck(pool: Pool, id: string): Promise<TerminalCheckRun> {
+  const deadline = Date.now() + 10000;
+  while (Date.now() < deadline) {
+    const result = await pool.query<CheckRunRow>('SELECT * FROM check_runs WHERE id = $1', [id]);
+    assert.ok(result.rows[0], 'The accepted execution must remain persisted.');
+    const check = checkRunFromRow(result.rows[0]);
+    if (check.state === 'SUCCEEDED' || check.state === 'FAILED') return check as TerminalCheckRun;
+    await delay(25);
+  }
+  throw new Error('Worker did not persist a terminal result before the test deadline.');
+}
