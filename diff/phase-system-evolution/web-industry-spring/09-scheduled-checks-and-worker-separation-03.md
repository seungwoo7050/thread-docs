## `feat(worker): expose durable browser lifecycle and verification`

diff --git a/TRACK.md b/TRACK.md
index 9fa02c8..a80303c 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -4,7 +4,7 @@ Profile: `phase-1` (E01–E12 → E20 → profile E24; phase-2 deferred, E25 not
 
 Spec revision: `2ada57a71cd34fa2fae9809415c362a8bbfcdf02`
 
-E06 is a local development product: Next.js/React renders login, logout, Monitor creation, editing, pause/activation, deletion and Check history. Spring Security authenticates browser sessions; PostgreSQL stores salted account passwords, Monitors and every completed CheckRun. A manual check is synchronous. Monitor and CheckRun access is scoped to the signed-in user. One React hook owns the page's server data and request state. There is no signup, scheduler, worker, Redis, broker, or production application container.
+E09 is a local development product: Next.js/React renders login, logout, Monitor lifecycle and Check history from request-scoped server data. Spring Security authenticates browser sessions; PostgreSQL stores salted account passwords, Monitors and Check execution intents/results. The API accepts a manual Check with202/QUEUED, and a separate worker executes it. Monitor and CheckRun access is scoped to the signed-in user. There is no signup, Redis, broker or production application container.
 
 ## Pinned toolchain
 
@@ -54,16 +54,17 @@ Start each process in a separate terminal:
 ```sh
 npm run fixture
 npm run api:dev
+npm run worker
 npm run dev
 ```
 
-Open [Monitors](http://127.0.0.1:4323/monitors), sign in with a prepared account, then create `Fixture monitor` with URL `http://127.0.0.1:4321/ok`, interval `60`, enabled checked. Click **Run check** to observe `SUCCEEDED` and `HTTP 200`. `/fail` yields `FAILED` and `HTTP 503`.
+Open [Monitors](http://127.0.0.1:4323/monitors), sign in with a prepared account, then create `Fixture monitor` with URL `http://127.0.0.1:4321/ok`, interval `60`, enabled checked. Click **Run check** to accept a QUEUED execution; the worker progresses it through RUNNING to `SUCCEEDED / HTTP 200`. `/fail` yields `FAILED / HTTP 503`. Enabled Monitors also receive scheduled intents at their interval.
 
 All defaults bind to `127.0.0.1`. Fixture port is 4321, API port 4322, UI port 4323, PostgreSQL port 15432. `FIXTURE_PORT`, `FIXTURE_ORIGIN`, and `API_PORT` can configure the server processes; `API_ORIGIN` configures the Next API proxy. The committed E01 tests use the fixed default ports; stop local servers before verification. Tests also reserve 4324 for a forbidden destination trap and 4325 for timeout/connection fixtures.
 
 The compose project is exclusively `wse-industry`, with its own bridge network and persistent volume. It uses explicitly nonsecret local test trust authentication and a loopback-only published port. An internal-only Docker network cannot provide the required published port on the verified Docker Desktop runtime. Never use this compose configuration for production or put unrelated data in it. `npm run db:down` stops the project but preserves data; `npm run db:up` restores it. `npm run db:destroy` explicitly deletes this project's disposable database volume.
 
-The default connection is database `monitor`, local test identity `wse_industry`, schema `public`. `DB_URL` (JDBC URL), `DB_USER`, `DB_PASSWORD`, and `DB_SCHEMA` support external runtime configuration. Never commit or log real credentials or put a password in a JDBC URL. Current verification resets only explicitly named `e04_*` schemas, not the developer's `public` schema.
+The default connection is database `monitor`, local test identity `wse_industry`, schema `public`. `DB_URL` (JDBC URL), `DB_USER`, `DB_PASSWORD`, and `DB_SCHEMA` support external runtime configuration. Never commit or log real credentials or put a password in a JDBC URL. Verification resets only explicitly declared disposable schemas, not the developer's `public` schema.
 
 ## Check boundary
 
@@ -71,7 +72,7 @@ The default connection is database `monitor`, local test identity `wse_industry`
 - Checks send one GET with no request body, use no proxy, and never follow redirects. `/redirect` therefore records `FAILED / 302`, with no request to `/ok`.
 - Connect timeout is 1 second and response-header read timeout is 2 seconds. No response body is materialized or retained. This is a controlled fixture implementation, not general Internet monitoring or general SSRF defense.
 - `200..299` is `SUCCEEDED`. Other observed HTTP statuses are `FAILED / HTTP_STATUS`. No HTTP response produces a null status and `TIMEOUT` or `CONNECTION_FAILURE`; no synthetic status is invented.
-- Monitors and all completed results survive API restarts. Interval and enabled are stored, without automatic scheduling; a paused Monitor can still be checked manually.
+- Monitors, queued intents and completed results survive API restarts. Enabled controls scheduling; a paused Monitor can still be checked manually.
 
 ## HTTP contract (E02)
 
@@ -79,7 +80,7 @@ The default connection is database `monitor`, local test identity `wse_industry`
 - Names are stripped of surrounding whitespace and must contain 1–100 UTF-16 code units. Interval is 1–86400 seconds inclusive; the JSON number `60.0` is the integer value 60, but the string `"60"` is invalid.
 - E03 also rejects a NUL character in a name before persistence, because PostgreSQL text cannot store it. Create and replacement use the same runtime validator, including the existing non-finite numeric rejection.
 - URL syntax must be absolute HTTP(S). The existing fixture policy then restricts it to the configured HTTP hostname and port, without credentials or fragments. This does not enable general public or HTTPS monitoring.
-- Successful list/create/check responses contain exactly `{ "data": <payload> }`. Create returns 201; list and completed synchronous checks return 200. The existing MonitorView/CheckRun payload fields are preserved, including explicit nulls.
+- Successful responses contain exactly `{ "data": <payload> }`. Create returns201; Check acceptance returns202/QUEUED; reads return200. The existing CheckRun fields remain, with null execution/result fields before they have been observed.
 - API failures contain `{ "error": { "code": "...", "message": "..." } }`: INVALID_INPUT / 400, UNAUTHENTICATED / 401, FORBIDDEN / 403, NOT_FOUND / 404, or INTERNAL_ERROR / 500. Malformed JSON and IDs use INVALID_INPUT after authentication. Unexpected exception details are not returned.
 - Endpoint HTTP 503 is still an API success containing a FAILED CheckRun with HTTP_STATUS; timeout and connection failure retain null httpStatus. API errors never fabricate a CheckRun.
 - The browser validates the envelope and the displayed payload shape, selects errors by the stable code/status pair, and does not classify or display arbitrary server prose. Network or malformed responses use the INTERNAL_ERROR UI fallback without applying the mutation.
@@ -90,9 +91,9 @@ The default connection is database `monitor`, local test identity `wse_industry`
 - A startup metadata check supplements Hibernate validation by rejecting unmapped required columns without defaults. Missing mapped columns or incompatible insert requirements prevent the web server from becoming ready.
 - `MonitorEntity.fromDomain/toDomain` and `CheckRunEntity.fromDomain/toDomain` explicitly map canonical immutable records. Entities never reach JSON. UUIDs remain UUIDs; interval is PostgreSQL integer, enabled boolean, latency bigint, and nullable HTTP status/reason retain null rather than zero/empty text.
 - Timestamp values are truncated to microseconds before the first response and stored as `timestamp(6) with time zone`. Canonical Java Instant/JSON values remain UTC even in a non-UTC PostgreSQL session.
-- Controller calls cross the concrete `MonitorStore` Spring transaction proxy. Public mutations use write transactions and reads use read-only transactions. Private helpers remain inside the caller's transaction; no self-invocation starts an assumed new transaction. The outbound GET runs between store calls, outside a transaction. No DB locks or worker structure are added.
+- Controller calls cross the concrete `MonitorStore` Spring transaction proxy. Public mutations use write transactions and reads use read-only transactions. The worker commits RUNNING before its outbound GET and records the outcome in a separate transaction; I/O holds no database transaction.
 - `GET /api/monitors/{id}` returns a MonitorView. `PUT /api/monitors/{id}` replaces the same four create fields, including enabled for pause/activation. `DELETE /api/monitors/{id}` returns `{ "data": null }`; PostgreSQL `ON DELETE CASCADE` removes all historical runs.
-- `GET /api/monitors/{id}/checks` returns the full history ordered by finishedAt descending, then ID descending. `GET /api/monitors/{id}/checks/{checkId}` returns a single historical result belonging to that Monitor. Deleted Monitor/history/run resources return 404/NOT_FOUND. Pagination is not introduced here.
+- `GET /api/monitors/{id}/checks` returns bounded terminal history ordered by finishedAt descending, then ID descending, with E07 limit/state/cursor conditions and `X-Next-Cursor`. Direct execution reads and the latest view also expose QUEUED/RUNNING. Deleted Monitor/history/run resources return404/NOT_FOUND.
 
 ## Browser sessions (E04)
 
@@ -134,7 +135,7 @@ npx playwright install chromium
 npm run verify
 ```
 
-`verify` starts the isolated PostgreSQL project, runs Maven unit, real-HTTP authentication/functional and real-PostgreSQL integration tests, and packages the API. It then checks TypeScript, compiles Next for production, runs the unchanged A,A,B process-restart/lifecycle product sequence with authentication setup, and runs Chromium against the local development UI and packaged API. It does not retry failed tests. Command outcomes and elapsed times are appended to `output/verification/runs.jsonl`; the process-restart probe saves only product wire evidence in `output/e03`. Committed evidence is in `evidence/E01` through `evidence/E05`.
+`verify` starts the isolated PostgreSQL project, runs Java tests and packages the backend, checks TypeScript and builds Next for production. It then runs the original A,A,B restart sequence with202/worker completion setup, the controlled E09 worker browser case, and the existing Chromium regression suite. It does not retry failed tests. Command outcomes and elapsed times are appended to `output/verification/runs.jsonl`. New profile evidence is under `evidence/phase-1`; legacy evidence remains unchanged.
 
 Tests create and remove isolated schemas for functional, browser, restart, mapping, migration, and incompatible-schema fixtures. The standard runner cleans up its browser/restart schemas even after a failure. The database remains available afterward; use `npm run db:down` to stop only this project. The Java tests explicitly close independent application contexts to verify restart persistence, capture actual generated SQL/transaction flags, check rollback and cascade, and assert startup rejection for both incompatible-schema cases.
 
@@ -159,6 +160,7 @@ npm run test:api
 npm run typecheck
 npm run build
 npm run api:package
+npm run test:worker
 npm run test:e2e
 ```
 
@@ -300,3 +302,38 @@ test forwards one real create POST and holds its201 response after commit; it th
 submits again, injects one update500 and one delete500 while held, verifies unchanged
 authority and pending state, releases the response, and checks one durable row.
 The existing article/history serves as detail; no routes or pagination were added.
+
+## Durable execution and scheduling (E09)
+
+The API only validates ownership and commits a PostgreSQL QUEUED CheckRun before
+returning202. `npm run worker` starts the same packaged application with the
+`worker` profile and no HTTP server. It reads one queued execution, commits
+RUNNING, performs the existing fixture-only GET outside a transaction, then
+updates that same ID with its observed terminal result. The browser polls only
+its visible active execution identities and invalidates their terminal history
+when they finish; other form controls remain usable after acceptance.
+
+V6 adds `queued_at` for durable intent ordering and `scheduled_for` for scheduled
+slot identity. Old terminal timestamps/outcomes and V1–V5 are unchanged;
+`queued_at` is backfilled from each old `finished_at`. Nonterminal results retain
+null HTTP status, latency and finish time; QUEUED also has no start time. Pending
+executions never enter terminal-history pagination or its finishedAt/id index
+ordering. Current/latest reads may use queued time until a result is finished.
+
+The worker's scheduler checks enabled Monitors every tick. Slots are anchored at
+Monitor creation plus the current interval; the first is due one interval after
+creation. A late tick creates only the current due slot, without historical
+catch-up. Repeating a slot is ignored, with a PostgreSQL unique scheduled-slot
+index as the durable guard. Disabled Monitors receive no scheduled intent.
+Distinct manual and scheduled intents may overlap in the queue; no Monitor-wide
+exclusion is imposed. The single E09 worker executes one at a time. Competing
+claims and recovery of a worker that stops while RUNNING are not implemented here.
+
+`npm run test:worker` owns an initially worker-free API and explicitly starts one
+separate JVM after the202 checkpoint. `npm run test:e2e` runs the earlier browser
+fixtures with a separate worker already available. Automatic scheduling is
+disabled only in these controlled regression processes; the fixed scheduler test
+uses the actual transaction at T0,T0,T+60. Neither harness mocks terminal results.
+The E05 SQL audit retains both user and worker statements, classifying only its
+explicit worker setup scope separately; every user/store owner predicate and
+denied mutation assertion remains intact.
diff --git a/app/monitors/api.ts b/app/monitors/api.ts
index cff2e64..0d63620 100644
--- a/app/monitors/api.ts
+++ b/app/monitors/api.ts
@@ -1,7 +1,8 @@
 export type Monitor = { id: string; name: string; url: string; interval: number; enabled: boolean };
 export type CheckRun = {
-  id: string; monitorId: string; state: 'SUCCEEDED' | 'FAILED'; httpStatus: number | null;
-  latencyMs: number; failureReason: 'HTTP_STATUS' | 'CONNECTION_FAILURE' | 'TIMEOUT' | null; finishedAt: string;
+  id: string; monitorId: string; state: 'QUEUED' | 'RUNNING' | 'SUCCEEDED' | 'FAILED'; httpStatus: number | null;
+  latencyMs: number | null; failureReason: 'HTTP_STATUS' | 'CONNECTION_FAILURE' | 'TIMEOUT' | null;
+  startedAt: string | null; finishedAt: string | null;
 };
 export type MonitorView = { monitor: Monitor; latestCheck: CheckRun | null };
 export type HistorySelection = { monitorId: string; limit: string; state: string | null; cursor: string | null };
@@ -43,8 +44,12 @@ function isMonitor(value: unknown): value is Monitor {
 }
 
 function isCheckRun(value: unknown): value is CheckRun {
-  if (!isObject(value) || typeof value.id !== 'string' || typeof value.monitorId !== 'string'
-    || typeof value.finishedAt !== 'string' || typeof value.latencyMs !== 'number'
+  if (!isObject(value) || typeof value.id !== 'string' || typeof value.monitorId !== 'string') return false;
+  if (value.state === 'QUEUED' || value.state === 'RUNNING') {
+    return value.finishedAt === null && value.httpStatus === null && value.latencyMs === null
+      && value.failureReason === null && (value.state === 'QUEUED' ? value.startedAt === null : typeof value.startedAt === 'string');
+  }
+  if (typeof value.startedAt !== 'string' || typeof value.finishedAt !== 'string' || typeof value.latencyMs !== 'number'
     || !Number.isInteger(value.latencyMs) || value.latencyMs < 0) return false;
   const status = value.httpStatus;
   if (status === null) {
@@ -117,7 +122,14 @@ export async function createMonitor(input: Omit<Monitor, 'id'>): Promise<Monitor
 }
 
 export async function runCheck(id: string): Promise<CheckRun> {
-  return readData(await mutation(`/api/monitors/${id}/checks`, 'POST'), isCheckRun);
+  const response = await mutation(`/api/monitors/${id}/checks`, 'POST');
+  const check = await readData(response, isCheckRun);
+  if (response.status !== 202 || check.state !== 'QUEUED') throw new ApiFailure('INTERNAL_ERROR');
+  return check;
+}
+
+export async function loadCheck(monitorId: string, checkId: string): Promise<CheckRun> {
+  return readData(await fetch(`/api/monitors/${monitorId}/checks/${checkId}`, { cache: 'no-store' }), isCheckRun);
 }
 
 export async function replaceMonitor(id: string, input: Omit<Monitor, 'id'>): Promise<MonitorView> {
diff --git a/app/monitors/monitor-controls.tsx b/app/monitors/monitor-controls.tsx
index c142fc6..1114346 100644
--- a/app/monitors/monitor-controls.tsx
+++ b/app/monitors/monitor-controls.tsx
@@ -59,7 +59,7 @@ export default function MonitorControls({ initial }: { initial: InitialMonitorSt
         <label>URL<input name="url" type="url" required defaultValue="http://127.0.0.1:4321/ok" /></label>
         <label>Interval (seconds)<input name="interval" type="number" min="1" max="86400" step="1" required defaultValue="60" /></label>
         <label className="checkbox"><input name="enabled" type="checkbox" defaultChecked />Enabled</label>
-        <p className="hint">Interval and enabled are stored. Checks still run manually; there is no scheduler.</p>
+        <p className="hint">Enabled monitors run at their interval in the worker. Manual and scheduled checks may overlap.</p>
         <button disabled={!state.loaded || state.pending('create') || state.pending('logout')}>Create monitor</button>
       </form>
       {state.pending('create') && <p role="status">Creating monitor…</p>}
@@ -102,7 +102,7 @@ export default function MonitorControls({ initial }: { initial: InitialMonitorSt
           {latestCheck ? <>
             <strong>{latestCheck.state}</strong>
             <span>HTTP {latestCheck.httpStatus ?? '—'}</span>
-            <span>{latestCheck.latencyMs} ms</span>
+            <span>{latestCheck.latencyMs ?? '—'} ms</span>
             {latestCheck.failureReason && <span>{latestCheck.failureReason}</span>}
           </> : <span>No checks yet.</span>}
         </div>
@@ -116,8 +116,8 @@ export default function MonitorControls({ initial }: { initial: InitialMonitorSt
             : checks.length === 0 ? <p>No historical checks.</p> : <div className="history-table">
             <table aria-label="Check history"><thead><tr><th>Finished (UTC)</th><th>Result</th><th>HTTP</th><th>Latency</th><th>Reason</th></tr></thead>
               <tbody>{checks.map(check => <tr key={check.id} data-check-id={check.id}>
-                <td><time dateTime={check.finishedAt}>{check.finishedAt}</time></td><td>{check.state}</td>
-                <td>{check.httpStatus ?? '—'}</td><td>{check.latencyMs} ms</td><td>{check.failureReason ?? '—'}</td>
+                <td>{check.finishedAt ? <time dateTime={check.finishedAt}>{check.finishedAt}</time> : '—'}</td><td>{check.state}</td>
+                <td>{check.httpStatus ?? '—'}</td><td>{check.latencyMs ?? '—'} ms</td><td>{check.failureReason ?? '—'}</td>
               </tr>)}</tbody>
             </table>
           </div>}
diff --git a/app/monitors/page.tsx b/app/monitors/page.tsx
index c655020..67b0754 100644
--- a/app/monitors/page.tsx
+++ b/app/monitors/page.tsx
@@ -10,7 +10,7 @@ export default async function Monitors({ searchParams }: {
   return <main>
     <header><p className="eyebrow">LOCAL LAB · INDUSTRY / SPRING</p><h1>Endpoint Monitor</h1>
       <p>Create a monitor and check its HTTP response.</p></header>
-    <aside>Development fixture only. Monitors and completed checks are stored in PostgreSQL.</aside>
+    <aside>Development fixture only. Monitors and Check execution records are stored in PostgreSQL.</aside>
     <MonitorControls initial={initial} />
   </main>;
 }
diff --git a/app/monitors/use-monitor-state.ts b/app/monitors/use-monitor-state.ts
index 46087cd..0dea958 100644
--- a/app/monitors/use-monitor-state.ts
+++ b/app/monitors/use-monitor-state.ts
@@ -1,7 +1,7 @@
 'use client';
 
 import { useEffect, useRef, useState } from 'react';
-import { createMonitor, deleteMonitor, failureCode, loadChecks, loadMonitors, logout,
+import { createMonitor, deleteMonitor, failureCode, loadCheck, loadChecks, loadMonitors, logout,
   replaceMonitor, runCheck, type ApiErrorCode, type HistoryPage, type HistorySelection, type InitialMonitorState,
   type Monitor, type MonitorView } from './api';
 
@@ -33,6 +33,9 @@ export function useMonitorState(selection: HistorySelection | null, initial: Ini
   const historyPending = !!monitorId && queries.loaded && !historyPage
     && (historyRead?.key !== historyKey || historyRead.phase === 'pending');
   const monitorPhase = monitorId ? operations[monitorId] : undefined;
+  const activeChecks = queries.monitors.flatMap(({ latestCheck }) => latestCheck
+    && (latestCheck.state === 'QUEUED' || latestCheck.state === 'RUNNING') ? [latestCheck] : []);
+  const activeCheckKey = JSON.stringify(activeChecks.map(check => [check.monitorId, check.id]));
 
   function reject(error: unknown) {
     const code = failureCode(error);
@@ -55,6 +58,34 @@ export function useMonitorState(selection: HistorySelection | null, initial: Ini
     return () => { active = false; };
   }, []);
 
+  useEffect(() => {
+    if (activeChecks.length === 0) return;
+    let active = true;
+    let timer: ReturnType<typeof setTimeout>;
+    async function poll() {
+      try {
+        const checks = await Promise.all(activeChecks.map(check => loadCheck(check.monitorId, check.id)));
+        if (!active) return;
+        setQueries(current => {
+          const histories = { ...current.histories };
+          const monitors = current.monitors.map(row => {
+            const observed = checks.find(check => check.monitorId === row.monitor.id && check.id === row.latestCheck?.id);
+            if (!observed) return row;
+            if (observed.state === 'SUCCEEDED' || observed.state === 'FAILED') delete histories[row.monitor.id];
+            return { ...row, latestCheck: observed };
+          });
+          return { ...current, monitors, histories };
+        });
+      } catch (error) {
+        if (active) reject(error);
+      }
+      if (active) timer = setTimeout(poll, 250);
+    }
+    timer = setTimeout(poll, 250);
+    // A newer manual execution or page/session boundary removes this observer's authority.
+    return () => { active = false; clearTimeout(timer); };
+  }, [activeCheckKey]);
+
   useEffect(() => {
     if (!queries.loaded || !monitorId || historyPage || monitorPhase === 'pending') return;
     let active = true;
diff --git a/evidence/phase-1/E09/browser-results.json b/evidence/phase-1/E09/browser-results.json
new file mode 100644
index 0000000..499dc60
--- /dev/null
+++ b/evidence/phase-1/E09/browser-results.json
@@ -0,0 +1,50 @@
+{
+  "controlledWorker": {
+    "stats": {
+      "startTime": "2026-08-28T04:38:12.186Z",
+      "duration": 16146.804,
+      "expected": 1,
+      "skipped": 0,
+      "unexpected": 0,
+      "flaky": 0
+    },
+    "tests": [
+      {
+        "file": "worker.spec.ts",
+        "title": "one persisted acceptance progresses in a separate worker while the browser stays usable",
+        "passed": true,
+        "attempts": 1
+      }
+    ]
+  },
+  "existingMonitor": {
+    "stats": {
+      "startTime": "2026-08-28T04:40:35.575Z",
+      "duration": 12969.550000000001,
+      "expected": 3,
+      "skipped": 0,
+      "unexpected": 0,
+      "flaky": 0
+    },
+    "tests": [
+      {
+        "file": "monitor.spec.ts",
+        "title": "create fixture monitor and display successful worker check",
+        "passed": true,
+        "attempts": 1
+      },
+      {
+        "file": "monitor.spec.ts",
+        "title": "display HTTP failure as a completed check result",
+        "passed": true,
+        "attempts": 1
+      },
+      {
+        "file": "monitor.spec.ts",
+        "title": "persist history, edits, pause, activation and deletion across page reloads",
+        "passed": true,
+        "attempts": 1
+      }
+    ]
+  }
+}
diff --git a/evidence/phase-1/E09/cleanup.json b/evidence/phase-1/E09/cleanup.json
new file mode 100644
index 0000000..4d5faed
--- /dev/null
+++ b/evidence/phase-1/E09/cleanup.json
@@ -0,0 +1,10 @@
+{
+  "checkedAt": "2026-08-28T04:51:47.103Z",
+  "schemas": [
+    "public"
+  ],
+  "listenersOn4321Through4325And4999": [],
+  "packagedApiOrWorkerJvms": 0,
+  "controlledWorkerExitAwaited": true,
+  "publicAndVolumeNotTargetedForRemoval": true
+}
diff --git a/evidence/phase-1/E09/invocations.jsonl b/evidence/phase-1/E09/invocations.jsonl
new file mode 100644
index 0000000..3202617
--- /dev/null
+++ b/evidence/phase-1/E09/invocations.jsonl
@@ -0,0 +1,17 @@
+{"command":"node scripts/e09-baseline.mjs","startedAt":"2026-08-28T04:18:29.108Z","elapsedSeconds":10.385,"exitCode":1}
+{"name":"backend-1","command":"mvn -B -ntp -f backend/pom.xml -Dtest=CheckRunnerTest,MonitorFunctionalTest,OwnershipAuthorizationTest,HistoryPaginationTest,HistoryIndexMigrationTest,CheckQueueTest package","startedAt":"2026-08-28T04:35:46.173Z","elapsedSeconds":15.506,"exitCode":1}
+{"name":"ownership-2","command":"mvn -B -ntp -f backend/pom.xml -Dtest=OwnershipAuthorizationTest package","startedAt":"2026-08-28T04:37:48.912Z","elapsedSeconds":11.797,"exitCode":0}
+{"name":"typecheck-1","command":"npm run typecheck","startedAt":"2026-08-28T04:38:00.712Z","elapsedSeconds":1.734,"exitCode":0}
+{"name":"build-1","command":"npm run build","startedAt":"2026-08-28T04:38:02.446Z","elapsedSeconds":9.155,"exitCode":0}
+{"name":"worker-browser-1","command":"npm run test:worker","startedAt":"2026-08-28T04:38:11.601Z","elapsedSeconds":16.747,"exitCode":0}
+{"name":"worker-browser-1-cleanup","command":"node scripts/database.mjs drop e04_browser","startedAt":"2026-08-28T04:38:28.349Z","elapsedSeconds":0.16,"exitCode":0}
+{"name":"monitor-browser-1","command":"npm run test:e2e -- tests/browser/monitor.spec.ts","startedAt":"2026-08-28T04:40:35.058Z","elapsedSeconds":13.501,"exitCode":0}
+{"name":"monitor-browser-1-cleanup","command":"node scripts/database.mjs drop e04_browser","startedAt":"2026-08-28T04:40:48.562Z","elapsedSeconds":0.242,"exitCode":0}
+{"name":"final-database-inspection","command":"docker compose --project-name wse-industry --file compose.yaml exec --no-TTY postgres psql <read-only non-system schema query>","startedAt":"2026-08-28T04:42:22.099Z","elapsedSeconds":0.170662458,"exitCode":0,"result":"public only"}
+{"name":"final-listener-inspection","command":"lsof -nP -iTCP:4321-4325 -iTCP:4999 -sTCP:LISTEN","startedAt":"2026-08-28T04:42:22.099Z","elapsedSeconds":0.000003667,"exitCode":1,"result":"no matching listeners; expected lsof exit1"}
+{"name":"final-process-inspection-sandbox","command":"pgrep -fl java.*monitor-api-0[.]0[.]1[.]jar","startedAt":"2026-08-28T04:43:31.082Z","elapsedSeconds":0.000003625,"exitCode":3,"result":"Sandbox could not access process list: sysmond service not found"}
+{"name":"final-process-inspection-approved","command":"pgrep -fl java.*monitor-api-0[.]0[.]1[.]jar","startedAt":"2026-08-28T04:43:37.999Z","elapsedSeconds":0.000002875,"exitCode":1,"result":"approved read-only inspection found no packaged API/worker JVM; expected pgrep exit1"}
+{"name":"ownership-migration-1","command":"mvn -B -ntp -f backend/pom.xml -Dtest=OwnershipMigrationTest test","startedAt":"2026-08-28T04:47:44.924Z","elapsedSeconds":8.785,"exitCode":0}
+{"name":"post-migration-database-inspection","command":"docker compose --project-name wse-industry --file compose.yaml exec --no-TTY postgres psql -U wse_industry -d monitor -tA -c \"SELECT schema_name FROM information_schema.schemata WHERE schema_name NOT LIKE 'pg_%' AND schema_name <> 'information_schema' ORDER BY schema_name\"","startedAt":"2026-08-28T04:51:41.063Z","elapsedSeconds":0.056013542,"exitCode":0,"result":"public only"}
+{"name":"post-migration-listener-inspection","command":"lsof -nP -iTCP:4321-4325 -iTCP:4999 -sTCP:LISTEN","startedAt":"2026-08-28T04:51:41.325Z","elapsedSeconds":0.000002166,"exitCode":1,"result":"no matching listeners; expected lsof exit1"}
+{"name":"post-migration-process-inspection","command":"pgrep -fl 'java.*monitor-api-0[.]0[.]1[.]jar'","startedAt":"2026-08-28T04:51:47.103Z","elapsedSeconds":0.000003208,"exitCode":1,"result":"no packaged API/worker JVM; expected pgrep exit1"}
diff --git a/evidence/phase-1/E09/owner-worker-sql.txt b/evidence/phase-1/E09/owner-worker-sql.txt
new file mode 100644
index 0000000..c49074e
--- /dev/null
+++ b/evidence/phase-1/E09/owner-worker-sql.txt
@@ -0,0 +1,15 @@
+SqlEvent[sql=insert into e05_ownership.monitors (created_at,enabled,interval_seconds,name,owner_user_id,updated_at,url,id) values (?,?,?,?,?,?,?,?), transaction=true, readOnly=false, worker=false]
+SqlEvent[sql=select me1_0.id,me1_0.created_at,me1_0.enabled,me1_0.interval_seconds,me1_0.name,me1_0.owner_user_id,me1_0.updated_at,me1_0.url from e05_ownership.monitors me1_0 where me1_0.id=? and me1_0.owner_user_id=?, transaction=true, readOnly=false, worker=false]
+SqlEvent[sql=insert into e05_ownership.check_runs (failure_reason,finished_at,http_status,latency_ms,monitor_id,queued_at,scheduled_for,started_at,state,trigger_kind,id) values (?,?,?,?,?,?,?,?,?,?,?), transaction=true, readOnly=false, worker=false]
+SqlEvent[sql=select cre1_0.id,cre1_0.failure_reason,cre1_0.finished_at,cre1_0.http_status,cre1_0.latency_ms,cre1_0.monitor_id,cre1_0.queued_at,cre1_0.scheduled_for,cre1_0.started_at,cre1_0.state,cre1_0.trigger_kind from e05_ownership.check_runs cre1_0 where cre1_0.state='QUEUED' order by cre1_0.queued_at,cre1_0.id fetch first ? rows only, transaction=true, readOnly=false, worker=true]
+SqlEvent[sql=select me1_0.id,me1_0.created_at,me1_0.enabled,me1_0.interval_seconds,me1_0.name,me1_0.owner_user_id,me1_0.updated_at,me1_0.url from e05_ownership.monitors me1_0 where me1_0.id=?, transaction=true, readOnly=false, worker=true]
+SqlEvent[sql=update e05_ownership.check_runs set failure_reason=?,finished_at=?,http_status=?,latency_ms=?,monitor_id=?,queued_at=?,scheduled_for=?,started_at=?,state=?,trigger_kind=? where id=?, transaction=true, readOnly=false, worker=true]
+SqlEvent[sql=update e05_ownership.check_runs cre1_0 set state=?,http_status=?,latency_ms=?,failure_reason=?,finished_at=? where cre1_0.id=? and cre1_0.monitor_id=? and cre1_0.state='RUNNING', transaction=true, readOnly=false, worker=true]
+SqlEvent[sql=select cre1_0.id,cre1_0.failure_reason,cre1_0.finished_at,cre1_0.http_status,cre1_0.latency_ms,cre1_0.monitor_id,cre1_0.queued_at,cre1_0.scheduled_for,cre1_0.started_at,cre1_0.state,cre1_0.trigger_kind from e05_ownership.check_runs cre1_0,e05_ownership.monitors me1_0 where cre1_0.id=? and cre1_0.monitor_id=? and cre1_0.monitor_id=me1_0.id and me1_0.owner_user_id=?, transaction=true, readOnly=true, worker=false]
+SqlEvent[sql=select me1_0.id,me1_0.created_at,me1_0.enabled,me1_0.interval_seconds,me1_0.name,me1_0.owner_user_id,me1_0.updated_at,me1_0.url from e05_ownership.monitors me1_0 where me1_0.id=? and me1_0.owner_user_id=?, transaction=true, readOnly=true, worker=false]
+SqlEvent[sql=select me1_0.id,me1_0.created_at,me1_0.enabled,me1_0.interval_seconds,me1_0.name,me1_0.owner_user_id,me1_0.updated_at,me1_0.url from e05_ownership.monitors me1_0 where me1_0.owner_user_id=? order by me1_0.created_at,me1_0.id, transaction=true, readOnly=true, worker=false]
+SqlEvent[sql=select cre1_0.id,cre1_0.failure_reason,cre1_0.finished_at,cre1_0.http_status,cre1_0.latency_ms,cre1_0.monitor_id,cre1_0.queued_at,cre1_0.scheduled_for,cre1_0.started_at,cre1_0.state,cre1_0.trigger_kind from e05_ownership.check_runs cre1_0,e05_ownership.monitors me1_0 where cre1_0.monitor_id=me1_0.id and me1_0.id=? and me1_0.owner_user_id=? order by coalesce(cre1_0.finished_at,cre1_0.queued_at) desc,cre1_0.id desc fetch first ? rows only, transaction=true, readOnly=true, worker=false]
+SqlEvent[sql=select cre1_0.id,cre1_0.failure_reason,cre1_0.finished_at,cre1_0.http_status,cre1_0.latency_ms,cre1_0.monitor_id,cre1_0.queued_at,cre1_0.scheduled_for,cre1_0.started_at,cre1_0.state,cre1_0.trigger_kind from e05_ownership.check_runs cre1_0,e05_ownership.monitors me1_0 where cre1_0.monitor_id=me1_0.id and me1_0.id=? and me1_0.owner_user_id=? and cre1_0.finished_at is not null order by cre1_0.finished_at desc,cre1_0.id desc fetch first ? rows only, transaction=true, readOnly=true, worker=false]
+SqlEvent[sql=update e05_ownership.monitors me1_0 set name=?,url=?,interval_seconds=?,enabled=?,updated_at=? where me1_0.id=? and me1_0.owner_user_id=?, transaction=true, readOnly=false, worker=false]
+SqlEvent[sql=delete from e05_ownership.monitors me1_0 where me1_0.id=? and me1_0.owner_user_id=?, transaction=true, readOnly=false, worker=false]
+SqlEvent[sql=select cre1_0.id,cre1_0.failure_reason,cre1_0.finished_at,cre1_0.http_status,cre1_0.latency_ms,cre1_0.monitor_id,cre1_0.queued_at,cre1_0.scheduled_for,cre1_0.started_at,cre1_0.state,cre1_0.trigger_kind from e05_ownership.check_runs cre1_0,e05_ownership.monitors me1_0 where cre1_0.monitor_id=me1_0.id and me1_0.id=? and me1_0.owner_user_id=? order by coalesce(cre1_0.finished_at,cre1_0.queued_at) desc,cre1_0.id desc fetch first ? rows only, transaction=true, readOnly=false, worker=false]
diff --git a/evidence/phase-1/E09/queue-migration.json b/evidence/phase-1/E09/queue-migration.json
new file mode 100644
index 0000000..cfdbee7
--- /dev/null
+++ b/evidence/phase-1/E09/queue-migration.json
@@ -0,0 +1,3 @@
+{"result":"PASS","upgradeFrom":5,"upgradeTo":6,"migrationsExecuted":1,
+ "repeatMigrations":0,"sevenTerminalRowsAndTimestampsUnchanged":true,
+ "priorMigrationChecksumsUnchanged":true,"queuedAtBackfilledFromFinishedAt":true}
diff --git a/evidence/phase-1/E09/scheduler.json b/evidence/phase-1/E09/scheduler.json
new file mode 100644
index 0000000..9c16bca
--- /dev/null
+++ b/evidence/phase-1/E09/scheduler.json
@@ -0,0 +1,2 @@
+{"result":"PASS","ticks":["2026-08-28T00:00:00Z","2026-08-28T00:00:00Z","2026-08-28T00:01:00Z"],
+ "enabledCounts":[1,1,2],"disabledCounts":[0,0,0],"states":"QUEUED","sameSlotIdentityRetained":true,"outboundCalls":0}
diff --git a/evidence/phase-1/E09/verification.md b/evidence/phase-1/E09/verification.md
new file mode 100644
index 0000000..af7330d
--- /dev/null
+++ b/evidence/phase-1/E09/verification.md
@@ -0,0 +1,136 @@
+# Phase-1 E09 author verification
+
+Thread E09, attempt1, branch `track/industry-spring`.
+START `0330d7741bbc0193af193be70d9dac47bfdb134f`.
+SPEC_REVISION `2ada57a71cd34fa2fae9809415c362a8bbfcdf02`.
+Frozen fixture SHA256:
+`35d362f1f6c80106e44e44b73ce58fe3c2fef5e7d0b6318ea1bbc6d8018ca6bb`.
+
+## Implementation
+
+The owner-authorized API transaction persists a QUEUED CheckRun and returns202
+without outbound I/O. A distinct non-web JVM using the worker profile reads the
+PostgreSQL queue, commits RUNNING, executes the existing fixture-only GET outside
+the transaction, then updates the same ID with its observed result. No external
+queue, new dependency, claim lock, lease or recovery policy was added.
+
+V6 adds queued time and scheduled-slot identity and permits null execution fields
+until they exist. Its constraints distinguish QUEUED/RUNNING from real terminal
+outcomes. A unique PostgreSQL index guards Monitor/scheduled-slot identity.
+Enabled Monitor slots are anchored at creation plus the current interval. A late
+tick selects only its current due slot; the implementation does not backfill a
+historical queue. Distinct manual/scheduled intents may coexist. The single E09
+worker executes them one at a time; competing-worker ownership is not claimed.
+
+The browser validates202/QUEUED, shows the accepted identity immediately and polls
+its active execution reads until completion. Completion invalidates related
+terminal history. QUEUED/RUNNING do not enter terminal pagination: its original
+finished_at/id descending order, filters, limit, cursor binding and response
+header remain unchanged. Current/latest reads can expose unfinished executions.
+
+## Invocations and failures
+
+All top-level verification invocations and final cleanup probes are recorded in
+`invocations.jsonl`. Node24.19.0/npm11.17.0, Java21.0.7+6 and Maven3.9.11 remain
+pinned. Java fixtures use owned PostgreSQL; browser cases use production Next and
+the existing pinned Chromium revision1234. There are no automatic test retries.
+
+| Invocation | Actual result | Command elapsed |
+| --- | --- | --- |
+| One unchanged synchronous baseline | Expected counterexample, exit1:200 instead of202 | 10.385s |
+| First targeted Java/package gate | 25 tests:24 pass, one old positive-path200 assertion fails on202; packaging stops | 15.506s |
+| Narrow OwnershipAuthorizationTest recheck/package | Three tests PASS after only the async setup adapter fix | 11.797s |
+| First typecheck | PASS | 1.734s |
+| First production build | PASS | 9.155s |
+| First controlled worker browser case | One test PASS | 16.747s |
+| Three existing Monitor browser cases | Three tests PASS | 13.501s |
+| Narrow OwnershipMigrationTest current-app adapter check | Three tests PASS; original migration/refusal/ownership assertions retained | 8.785s |
+
+The initial Java failure was in the allowed positive Check setup of the E05
+Origin/CSRF matrix. The existing helper now accepts202, explicitly runs the worker
+and reads that same execution before continuing. All original denied statuses,
+all-row/no-outbound checks and final outbound count3 remain unchanged. Only this
+class was rerun; the other22 passing tests were not repeated. The subsequently
+approved OwnershipMigrationTest adapter passed its three tests on its first
+narrow invocation. Thus28 distinct targeted Java tests have passing evidence
+across three Maven invocations (31 test executions, one failure), not one fresh
+all-pass28-test run. Typecheck did not execute in the first failed command chain;
+it ran once after the correction.
+
+The final process-list probe initially failed inside the sandbox with
+`sysmond service not found`/exit3. The same read-only probe was then explicitly
+approved and returned no matches/exit1. This was a cleanup permission failure,
+not a product-test failure or a test retry.
+
+Author budgets: baseline1; post-change controlled lifecycle1; ordinary browser
+invocations1; load runs0; automatic retries0; parameter sweeps0; full `npm run
+verify` invocations0. Root owns the independent full regression, including the
+remaining Java/session cases, full browser/SSR/accessibility suite and restart
+scenario. Those are not reported as fresh author passes.
+
+## Decisive observations
+
+The baseline's held checkpoint had API-response=false, durable CheckRun count0
+and outbound count1. Only after the test released HTTP200 did the API return
+200/SUCCEEDED with an ID. The baseline's after-release ID flag records response
+ID presence; its direct database count observation was the pre-release count0.
+No additional baseline or timeout variant was run.
+
+`worker-browser.json` records the post-change checkpoints:202/QUEUED with one
+persisted row and outbound0 before worker startup; RUNNING on the same row with
+startedAt and null terminal fields while one request was held; then same-ID
+SUCCEEDED/HTTP200 with observed latency/finish time only after release. The worker
+PID was distinct from the actual API listener PID. The browser displayed all
+three states, accepted input in the unrelated form while held, and showed the
+terminal history row after reload. One outbound request and one CheckRun remained.
+The held fixture was released and the owned worker exit awaited.
+
+`scheduler.json` records the real scheduler transaction at T0,T0,T+60. Persisted
+enabled counts were1,1,2; disabled counts0,0,0; due slots retained their IDs;
+all rows were SCHEDULED/QUEUED and no runner call occurred. The scheduler case uses
+its own e09_scheduler schema with exactly the fixed A/B dataset.
+
+`queue-migration.json` records V5→V6 and a no-op repeat, seven old terminal rows
+and all their original timestamps/outcome values unchanged, prior migration
+checksums unchanged, and queued_at backfilled from finished_at.
+
+## Regression adapters and transaction evidence
+
+Existing synchronous consumers now await the worker after202 before asserting
+their unchanged terminal outcomes. The restart script retains its original
+A,A,B data and restart checks; its new worker/await setup is left for root's full
+gate. The three original Monitor browser cases passed through the ordinary
+background-worker harness. The controlled E09 case is selected separately by
+`npm run test:worker`, and `npm run verify` includes both modes.
+
+E07/E08 browser inserts gained explicit original column lists only; every prior
+seed ID, value and timestamp remains unchanged. The old V5 migration proof is
+pinned to target5 before its additional E09 V6 assertions. No old evidence or
+migration file was modified.
+
+With root approval, OwnershipMigrationTest's current-application startup no
+longer pins Flyway to target4. Its explicit target3/target4 legs, migration
+counts, refusal-before-ready and owner assertions are unchanged. Its CheckRun
+snapshot selects the original nine columns so it compares the same old values
+without including newly appended queue metadata. That class passed once after
+this adapter; no additional production change or backend gate followed it.
+
+With root approval, the E05 test's ThreadLocal classifies only its explicit
+worker.executeNext setup and is removed in finally. `owner-worker-sql.txt`
+retains both user and worker statements and their actual transaction flags.
+Every user/store resource query retains its owner condition; every captured
+resource statement is transactional. Global trusted worker selection is recorded
+separately from user operations. The existing functional test additionally reads
+the committed RUNNING row through another connection inside the outbound runner
+spy, while asserting there is no active Spring transaction during I/O.
+
+## Cleanup and scope
+
+Disposable schemas were removed. Final read-only inspection found only public,
+no listeners on4321–4325/4999, and no packaged API/worker JVM. Public and the
+PostgreSQL volume were not targeted for removal. `cleanup.json` records this.
+No credentials, session/CSRF values, response bodies, browser captures or
+storage-state artifacts are included in the new evidence.
+
+No E10/E11/E12, phase-2, main/spec/index/tag edits, history rewrite or push.
+The author candidate is ready for root verification; no next Thread is started.
diff --git a/evidence/phase-1/E09/worker-browser.json b/evidence/phase-1/E09/worker-browser.json
new file mode 100644
index 0000000..2adba92
--- /dev/null
+++ b/evidence/phase-1/E09/worker-browser.json
@@ -0,0 +1,41 @@
+{
+  "start": "0330d7741bbc0193af193be70d9dac47bfdb134f",
+  "fixtureSha256": "35d362f1f6c80106e44e44b73ce58fe3c2fef5e7d0b6318ea1bbc6d8018ca6bb",
+  "completed": [
+    "202 and durable QUEUED precede all outbound I/O; browser shows the accepted ID",
+    "separate worker commits RUNNING on the same ID; held endpoint has no invented outcome and browser remains usable",
+    "release200 yields one same-ID terminal result, history row and reload"
+  ],
+  "acceptance": {
+    "status": 202,
+    "persistedRows": 1,
+    "state": "QUEUED",
+    "workerStopped": true,
+    "outboundRequests": 0
+  },
+  "running": {
+    "sameId": true,
+    "persistedRows": 1,
+    "state": "RUNNING",
+    "startedAtPresent": true,
+    "terminalFieldsNull": true,
+    "outboundRequests": 1,
+    "fixtureReleased": false,
+    "distinctWorkerJvm": true,
+    "browserUsable": true
+  },
+  "terminal": {
+    "sameId": true,
+    "persistedRows": 1,
+    "state": "SUCCEEDED",
+    "httpStatus": 200,
+    "observedLatencyAndFinishedAt": true,
+    "browserHistoryAndReload": true,
+    "outboundRequests": 1
+  },
+  "result": "PASS",
+  "cleanup": {
+    "fixtureReleased": true,
+    "workerExitAwaited": true
+  }
+}
diff --git a/package.json b/package.json
index af29b77..afee327 100644
--- a/package.json
+++ b/package.json
@@ -14,10 +14,12 @@
     "db:destroy": "node scripts/database.mjs destroy",
     "api:dev": "mvn -B -ntp -f backend/pom.xml spring-boot:run",
     "api:package": "mvn -B -ntp -f backend/pom.xml -DskipTests package",
+    "worker": "java -jar backend/target/monitor-api-0.0.1.jar --spring.profiles.active=worker --spring.main.web-application-type=none",
     "bootstrap:users": "node scripts/bootstrap-users.mjs",
     "test:api": "mvn -B -ntp -f backend/pom.xml test",
     "typecheck": "next typegen && tsc --noEmit",
     "test:e2e": "playwright test",
+    "test:worker": "E09_MANUAL_WORKER=1 playwright test tests/browser/worker.spec.ts",
     "verify": "node scripts/verify.mjs"
   },
   "dependencies": {
diff --git a/playwright.config.ts b/playwright.config.ts
index 54560a9..7762f18 100644
--- a/playwright.config.ts
+++ b/playwright.config.ts
@@ -9,6 +9,7 @@ process.env.E04_BOB_PASSWORD ??= randomBytes(32).toString('base64url');
 
 export default defineConfig({
   testDir: './tests/browser',
+  testIgnore: process.env.E09_MANUAL_WORKER === '1' ? [] : ['**/worker.spec.ts'],
   fullyParallel: false,
   workers: 1,
   retries: 0,
diff --git a/scripts/persistence-scenario.mjs b/scripts/persistence-scenario.mjs
index fef42f7..24a66fa 100644
--- a/scripts/persistence-scenario.mjs
+++ b/scripts/persistence-scenario.mjs
@@ -127,7 +127,7 @@ async function request(path, method = 'GET', body, status = 200) {
   evidence.requests.push({ method, path, status: response.status, wire,
     elapsedSeconds: (Date.now() - requestStarted) / 1000 });
   assert.equal(response.status, status);
-  assert.deepEqual(Object.keys(wire), [status === 200 || status === 201 ? 'data' : 'error']);
+  assert.deepEqual(Object.keys(wire), [status === 200 || status === 201 || status === 202 ? 'data' : 'error']);
   if (status === 404) assert.equal(wire.error.code, 'NOT_FOUND');
   if (status === 400) assert.equal(wire.error.code, 'INVALID_INPUT');
   return wire.data;
@@ -144,6 +144,8 @@ try {
   await ready('http://127.0.0.1:4321/ok', fixture, 'Fixture http://127.0.0.1:4321');
   api = start('java', ['-jar', 'backend/target/monitor-api-0.0.1.jar'], 'api-first');
   await ready(`${base}/api/session`, api, 'Started MonitorApplication', 401);
+  start('java', ['-jar', 'backend/target/monitor-api-0.0.1.jar', '--spring.profiles.active=worker',
+    '--spring.main.web-application-type=none', '--monitor.scheduler-enabled=false'], 'worker');
   await authenticate();
   const firstApiPid = api.pid;
   assert.deepEqual(await request('/api/monitors'), [], 'Scenario requires an isolated empty store');
@@ -157,7 +159,13 @@ try {
   for (const [monitor, state, httpStatus, failureReason] of [
     [a, 'SUCCEEDED', 200, null], [a, 'SUCCEEDED', 200, null], [b, 'FAILED', 503, 'HTTP_STATUS'],
   ]) {
-    const check = await request(`/api/monitors/${monitor.id}/checks`, 'POST');
+    let check = await request(`/api/monitors/${monitor.id}/checks`, 'POST', undefined, 202);
+    const deadline = Date.now() + 10_000;
+    while (check.state === 'QUEUED' || check.state === 'RUNNING') {
+      assert.ok(Date.now() < deadline, 'Owned worker did not complete the accepted execution');
+      await delay(25);
+      check = await request(`/api/monitors/${monitor.id}/checks/${check.id}`);
+    }
     assert.deepEqual([check.state, check.httpStatus, check.failureReason], [state, httpStatus, failureReason]);
     evidence.checks.push(check);
   }
diff --git a/scripts/test-api.mjs b/scripts/test-api.mjs
index 404a207..07a79a9 100644
--- a/scripts/test-api.mjs
+++ b/scripts/test-api.mjs
@@ -11,25 +11,44 @@ await new Promise((resolve, reject) => {
 const reset = spawnSync(process.execPath, ['scripts/database.mjs', 'reset', 'e04_browser'], { stdio: 'inherit' });
 if (reset.status !== 0) process.exit(reset.status ?? 1);
 let api;
+let worker;
 let force;
+let stopping = false;
+let workerFailed = false;
 try {
   bootstrapUsers('e04_browser', process.env);
   const { E04_ALICE_PASSWORD, E04_BOB_PASSWORD, ...runtime } = process.env;
   api = spawn('java', ['-jar', 'backend/target/monitor-api-0.0.1.jar'], {
     env: { ...runtime, DB_SCHEMA: 'e04_browser' }, stdio: 'inherit',
   });
+  if (process.env.E09_MANUAL_WORKER !== '1') {
+    worker = spawn('java', ['-jar', 'backend/target/monitor-api-0.0.1.jar',
+      '--spring.profiles.active=worker', '--spring.main.web-application-type=none', '--monitor.scheduler-enabled=false'], {
+      env: { ...runtime, DB_SCHEMA: 'e04_browser' }, stdio: 'inherit',
+    });
+    worker.once('exit', () => {
+      if (!stopping) { workerFailed = true; api.kill('SIGTERM'); }
+    });
+    worker.once('error', () => { workerFailed = true; api.kill('SIGTERM'); });
+  }
   for (const signal of ['SIGINT', 'SIGTERM']) process.on(signal, () => {
+    stopping = true;
     api.kill(signal);
-    force ??= setTimeout(() => api.kill('SIGKILL'), 5000);
+    worker?.kill(signal);
+    force ??= setTimeout(() => { api.kill('SIGKILL'); worker?.kill('SIGKILL'); }, 5000);
   });
   const [code, signal] = await once(api, 'exit');
-  process.exitCode = signal === 'SIGTERM' || signal === 'SIGINT' ? 0 : (code ?? 1);
+  process.exitCode = workerFailed ? 1 : signal === 'SIGTERM' || signal === 'SIGINT' ? 0 : (code ?? 1);
 } finally {
+  stopping = true;
   clearTimeout(force);
-  if (api && api.exitCode === null && api.signalCode === null) {
-    const exited = once(api, 'exit');
-    api.kill('SIGTERM');
+  for (const child of [worker, api]) {
+    if (!child || child.exitCode !== null || child.signalCode !== null) continue;
+    const exited = once(child, 'exit');
+    child.kill('SIGTERM');
+    const forceExit = setTimeout(() => child.kill('SIGKILL'), 5000);
     await exited;
+    clearTimeout(forceExit);
   }
   const cleanup = spawnSync(process.execPath, ['scripts/database.mjs', 'drop', 'e04_browser'], { stdio: 'inherit' });
   if (cleanup.status !== 0) process.exitCode = cleanup.status ?? 1;
diff --git a/scripts/verify.mjs b/scripts/verify.mjs
index 09067b2..77faf5c 100644
--- a/scripts/verify.mjs
+++ b/scripts/verify.mjs
@@ -11,6 +11,7 @@ const commands = [
   ['node', ['scripts/persistence-isolation.mjs']],
   ['node', ['scripts/database.mjs', 'reset', 'e04_restart']],
   ['node', ['scripts/persistence-scenario.mjs', 'fixed']],
+  ['npm', ['run', 'test:worker']],
   ['npm', ['run', 'test:e2e']],
 ];
 function run(command, args) {
diff --git a/tests/browser/history.spec.ts b/tests/browser/history.spec.ts
index 3f06e28..6b4a57f 100644
--- a/tests/browser/history.spec.ts
+++ b/tests/browser/history.spec.ts
@@ -9,6 +9,7 @@ const id = (number: number) => `00000000-0000-4000-8000-${String(number).padStar
 function seed(monitor: string, from: number, through: number) {
   expect(monitor).toMatch(/^[a-f0-9-]{36}$/);
   const sql = `INSERT INTO e04_browser.check_runs
+    (id,monitor_id,trigger_kind,state,http_status,latency_ms,failure_reason,started_at,finished_at)
     SELECT ('00000000-0000-4000-8000-' || lpad(n::text,12,'0'))::uuid,
     '${monitor}'::uuid, 'MANUAL', CASE WHEN n%2=1 OR n=8 THEN 'SUCCEEDED' ELSE 'FAILED' END,
     CASE WHEN n%2=1 OR n=8 THEN 200 ELSE 503 END, 1,
diff --git a/tests/browser/monitor.spec.ts b/tests/browser/monitor.spec.ts
index f36e63f..abf9482 100644
--- a/tests/browser/monitor.spec.ts
+++ b/tests/browser/monitor.spec.ts
@@ -1,6 +1,6 @@
 import { test, expect } from './authenticated';
 
-test('create fixture monitor and display successful synchronous check', async ({ page }) => {
+test('create fixture monitor and display successful worker check', async ({ page }) => {
   await page.goto('/monitors');
   await page.getByLabel('Name', { exact: true }).fill('Fixture monitor');
   await page.getByLabel('URL', { exact: true }).fill('http://127.0.0.1:4321/ok');
@@ -38,7 +38,10 @@ test('persist history, edits, pause, activation and deletion across page reloads
   let monitor = page.getByRole('article', { name: 'Lifecycle fixture', exact: true });
   await monitor.getByRole('button', { name: 'Run check' }).click();
   await expect(monitor.getByText('SUCCEEDED', { exact: true })).toBeVisible();
+  const previousCheck = await monitor.locator('[data-latest-check-id]').getAttribute('data-latest-check-id');
   await monitor.getByRole('button', { name: 'Run check' }).click();
+  await expect(monitor.locator('[data-latest-check-id]')).not.toHaveAttribute('data-latest-check-id', previousCheck!);
+  await expect(monitor.getByText('SUCCEEDED', { exact: true })).toBeVisible();
   await monitor.getByRole('button', { name: 'Show history' }).click();
   await expect(monitor.getByRole('table', { name: 'Check history' }).getByRole('row')).toHaveCount(3);
   await monitor.getByRole('button', { name: 'Edit', exact: true }).click();
diff --git a/tests/browser/ownership.spec.ts b/tests/browser/ownership.spec.ts
index e2c44ac..dfd5fe8 100644
--- a/tests/browser/ownership.spec.ts
+++ b/tests/browser/ownership.spec.ts
@@ -45,7 +45,7 @@ async function createAndCheck(page: Page, name: string, route: '/ok' | '/fail',
     && response.request().method() === 'POST');
   await article.getByRole('button', { name: 'Run check', exact: true }).click();
   const checked = await checkResponse;
-  expect(checked.status()).toBe(200);
+  expect(checked.status()).toBe(202);
   const check = (await checked.json()).data;
   await expect(article.getByText(route === '/ok' ? 'SUCCEEDED' : 'FAILED', { exact: true })).toBeVisible();
   await article.getByRole('button', { name: 'Show history', exact: true }).click();
diff --git a/tests/browser/rendering.spec.ts b/tests/browser/rendering.spec.ts
index 5265c03..77844ed 100644
--- a/tests/browser/rendering.spec.ts
+++ b/tests/browser/rendering.spec.ts
@@ -58,7 +58,8 @@ test('fixed authenticated SSR, hydration, keyboard and accessibility boundary',
     expect(monitor).toMatch(/^[a-f0-9-]{36}$/);
     execFileSync('docker', ['compose', '--project-name', 'wse-industry', '--file', 'compose.yaml',
       'exec', '--no-TTY', 'postgres', 'psql', '--username', 'wse_industry', '--dbname', 'monitor',
-      '--set', 'ON_ERROR_STOP=1', '--command', `INSERT INTO e04_browser.check_runs VALUES
+      '--set', 'ON_ERROR_STOP=1', '--command', `INSERT INTO e04_browser.check_runs
+        (id,monitor_id,trigger_kind,state,http_status,latency_ms,failure_reason,started_at,finished_at) VALUES
         ('${runIds[1]}','${monitor}','MANUAL','SUCCEEDED',200,1,null,'2026-08-28T00:00:00.000Z','2026-08-28T00:00:00.000Z'),
         ('${runIds[0]}','${monitor}','MANUAL','SUCCEEDED',200,1,null,'2026-08-28T00:00:01.000Z','2026-08-28T00:00:01.000Z')`],
     { stdio: 'pipe' });
diff --git a/tests/browser/server-state.spec.ts b/tests/browser/server-state.spec.ts
index cf0cb2d..6afd50a 100644
--- a/tests/browser/server-state.spec.ts
+++ b/tests/browser/server-state.spec.ts
@@ -98,7 +98,7 @@ test('fixed server-state lifecycle, held create double-submit and independent mu
 
     const a = page.getByRole('article', { name: 'Monitor A', exact: true });
     const checked = await uiResponse(page, `${aPath}/checks`, 'POST', () => a.getByRole('button', { name: 'Run check', exact: true }).click());
-    expect(checked.status()).toBe(200);
+    expect(checked.status()).toBe(202);
     const checkId = (await checked.json()).data.id as string;
     await expect(a.getByText('SUCCEEDED', { exact: true })).toBeVisible();
     await a.getByRole('button', { name: 'Show history', exact: true }).click();
diff --git a/tests/browser/worker.spec.ts b/tests/browser/worker.spec.ts
new file mode 100644
index 0000000..424e110
--- /dev/null
+++ b/tests/browser/worker.spec.ts
@@ -0,0 +1,133 @@
+import { spawn, execFileSync, type ChildProcess } from 'node:child_process';
+import { createHash } from 'node:crypto';
+import { once } from 'node:events';
+import { closeSync, mkdirSync, openSync, readFileSync, writeFileSync } from 'node:fs';
+import { test, expect, csrfHeaders, safeRequest } from './authenticated';
+
+const fixture = 'http://127.0.0.1:4321';
+const directory = 'output/phase-1/e09';
+function persisted() {
+  const sql = `SELECT coalesce(json_agg(c), '[]') FROM
+    (SELECT id,state,http_status,latency_ms,started_at,finished_at FROM e04_browser.check_runs) c`;
+  return JSON.parse(execFileSync('docker', ['compose', '--project-name', 'wse-industry', '--file', 'compose.yaml',
+    'exec', '--no-TTY', 'postgres', 'psql', '--username', 'wse_industry', '--dbname', 'monitor',
+    '--tuples-only', '--no-align', '--command', sql], { encoding: 'utf8' })) as {
+      id: string; state: string; http_status: number | null; latency_ms: number | null;
+      started_at: string | null; finished_at: string | null;
+    }[];
+}
+
+test('one persisted acceptance progresses in a separate worker while the browser stays usable', async ({ page }) => {
+  expect(process.env.E09_MANUAL_WORKER).toBe('1');
+  mkdirSync(directory, { recursive: true });
+  const completed: string[] = [];
+  const evidence: Record<string, unknown> = { start: '0330d7741bbc0193af193be70d9dac47bfdb134f',
+    fixtureSha256: createHash('sha256').update(readFileSync('evidence/phase-1/E09/fixtures.md')).digest('hex'), completed };
+  let worker: ChildProcess | undefined;
+  let workerError = false;
+  let released = false;
+  let workerExitAwaited = false;
+  try {
+    expect((await (await safeRequest(() => page.request.get('/api/monitors'))).json()).data.length).toBe(0);
+    let aId = '';
+    for (const [name, path, enabled] of [['A', '/hold', true], ['B', '/ok', false]] as const) {
+      const headers = await csrfHeaders(page.request);
+      const response = await safeRequest(() => page.request.post('/api/monitors', {
+        headers, data: { name, url: `${fixture}${path}`, interval: 60, enabled },
+      }));
+      expect(response.status()).toBe(201);
+      if (name === 'A') aId = (await response.json()).data.monitor.id;
+    }
+    const apiPid = Number(execFileSync('lsof', ['-t', '-nP', '-iTCP:4322', '-sTCP:LISTEN'], { encoding: 'utf8' }).trim());
+    expect(Number.isInteger(apiPid) && apiPid > 0).toBe(true);
+    await page.goto('/monitors');
+    const a = page.getByRole('article', { name: 'A', exact: true });
+    const result = a.locator('[data-latest-check-id]');
+    const [accepted] = await Promise.all([
+      page.waitForResponse(response => new URL(response.url()).pathname === `/api/monitors/${aId}/checks`
+        && response.request().method() === 'POST'),
+      a.getByRole('button', { name: 'Run check', exact: true }).click(),
+    ]);
+    expect(accepted.status()).toBe(202);
+    const queued = (await accepted.json()).data;
+    expect(queued.state).toBe('QUEUED');
+    for (const field of ['startedAt', 'finishedAt', 'httpStatus', 'latencyMs', 'failureReason']) expect(queued[field]).toBeNull();
+    const queuedRows = persisted();
+    expect(queuedRows.length).toBe(1);
+    expect(queuedRows[0].id).toBe(queued.id);
+    expect(queuedRows[0].state).toBe('QUEUED');
+    expect(queuedRows[0].started_at).toBeNull();
+    expect(queuedRows[0].finished_at).toBeNull();
+    expect((await (await fetch(`${fixture}/__e09/status`)).json()).holdRequests).toBe(0);
+    await expect(result).toHaveAttribute('data-latest-check-id', queued.id);
+    await expect(result.getByText('QUEUED', { exact: true })).toBeVisible();
+    evidence.acceptance = { status: 202, persistedRows: 1, state: 'QUEUED', workerStopped: true, outboundRequests: 0 };
+    completed.push('202 and durable QUEUED precede all outbound I/O; browser shows the accepted ID');
+
+    const log = openSync(`${directory}/worker-process.log`, 'w');
+    const { E04_ALICE_PASSWORD, E04_BOB_PASSWORD, ...runtime } = process.env;
+    worker = spawn('java', ['-jar', 'backend/target/monitor-api-0.0.1.jar', '--spring.profiles.active=worker',
+      '--spring.main.web-application-type=none', '--monitor.scheduler-enabled=false'], {
+      env: { ...runtime, DB_SCHEMA: 'e04_browser' }, stdio: ['ignore', log, log],
+    });
+    closeSync(log);
+    worker.once('error', () => { workerError = true; });
+    expect(worker.pid !== apiPid && !!worker.pid).toBe(true);
+    await expect.poll(async () => {
+      expect(workerError || worker!.exitCode !== null || worker!.signalCode !== null).toBe(false);
+      return (await (await fetch(`${fixture}/__e09/status`)).json()).held;
+    }, { intervals: [25], timeout: 30_000 }).toBe(1);
+    const running = persisted();
+    expect(running.length).toBe(1);
+    expect(running[0].id).toBe(queued.id);
+    expect(running[0].state).toBe('RUNNING');
+    expect(running[0].started_at).not.toBeNull();
+    for (const field of ['finished_at', 'http_status', 'latency_ms'] as const) expect(running[0][field]).toBeNull();
+    await expect(result.getByText('RUNNING', { exact: true })).toBeVisible();
+    await expect(result).toHaveAttribute('data-latest-check-id', queued.id);
+    await page.getByLabel('Name', { exact: true }).fill('Unsubmitted draft');
+    await expect(page.getByRole('button', { name: 'Create monitor', exact: true })).toBeEnabled();
+    const held = await (await fetch(`${fixture}/__e09/status`)).json();
+    expect(held).toEqual({ holdRequests: 1, held: 1, released: false });
+    evidence.running = { sameId: true, persistedRows: 1, state: 'RUNNING', startedAtPresent: true,
+      terminalFieldsNull: true, outboundRequests: 1, fixtureReleased: false, distinctWorkerJvm: true, browserUsable: true };
+    completed.push('separate worker commits RUNNING on the same ID; held endpoint has no invented outcome and browser remains usable');
+
+    expect((await fetch(`${fixture}/__e09/release`, { method: 'POST' })).status).toBe(200);
+    released = true;
+    await expect(result.getByText('SUCCEEDED', { exact: true })).toBeVisible();
+    await expect(result.getByText('HTTP 200', { exact: true })).toBeVisible();
+    const terminal = persisted();
+    expect(terminal.length).toBe(1);
+    expect(terminal[0].id).toBe(queued.id);
+    expect(terminal[0].state).toBe('SUCCEEDED');
+    expect(terminal[0].http_status).toBe(200);
+    expect(typeof terminal[0].latency_ms).toBe('number');
+    expect(terminal[0].finished_at).not.toBeNull();
+    await expect(result).toHaveAttribute('data-latest-check-id', queued.id);
+    await a.getByRole('button', { name: 'Show history', exact: true }).click();
+    await expect(a.locator(`tr[data-check-id="${queued.id}"]`)).toHaveCount(1);
+    await page.reload();
+    await expect(a.locator(`tr[data-check-id="${queued.id}"]`)).toHaveCount(1);
+    await expect(result).toHaveAttribute('data-latest-check-id', queued.id);
+    expect((await (await fetch(`${fixture}/__e09/status`)).json()).holdRequests).toBe(1);
+    evidence.terminal = { sameId: true, persistedRows: 1, state: 'SUCCEEDED', httpStatus: 200,
+      observedLatencyAndFinishedAt: true, browserHistoryAndReload: true, outboundRequests: 1 };
+    completed.push('release200 yields one same-ID terminal result, history row and reload');
+    evidence.result = 'PASS';
+  } finally {
+    if (!released) {
+      try { await fetch(`${fixture}/__e09/release`, { method: 'POST', signal: AbortSignal.timeout(1000) }); released = true; } catch {}
+    }
+    if (worker && worker.exitCode === null && worker.signalCode === null) {
+      const exited = once(worker, 'exit');
+      worker.kill('SIGTERM');
+      const force = setTimeout(() => worker!.kill('SIGKILL'), 5000);
+      await exited;
+      clearTimeout(force);
+      workerExitAwaited = true;
+    }
+    evidence.cleanup = { fixtureReleased: released, workerExitAwaited };
+    writeFileSync(`${directory}/worker-browser.json`, JSON.stringify(evidence, null, 2) + '\n');
+  }
+});
