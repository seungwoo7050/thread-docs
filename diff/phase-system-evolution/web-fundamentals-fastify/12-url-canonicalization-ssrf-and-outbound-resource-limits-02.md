## `feat: validate outbound destinations and bound check resources`

diff --git a/.github/workflows/check.yml b/.github/workflows/check.yml
index 1bc9972..0ecd7ef 100644
--- a/.github/workflows/check.yml
+++ b/.github/workflows/check.yml
@@ -37,6 +37,8 @@ jobs:
         run: npm run test:ownership
       - name: Worker crash recovery and shutdown
         run: npm run test:recovery
+      - name: Outbound safety and resource limits
+        run: npm run test:outbound
       - name: Install pinned Chromium
         run: npx playwright install --with-deps chromium
       - name: Production build and browser E2E
diff --git a/TRACK.md b/TRACK.md
index d1fbe88..415a6ac 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -6,11 +6,12 @@ Spec revision: `2ada57a71cd34fa2fae9809415c362a8bbfcdf02`.
 
 E01 established Monitor creation and synchronous manual GET checks in process memory.
 E03 stores Monitors and every terminal CheckRun in PostgreSQL, including after API restart.
-Only the configured fixture origin is eligible for outbound requests. The default
-is `http://127.0.0.1:4311`; API and fixture bind to `127.0.0.1`.
-Checks have a fixed 2 second total deadline, perform no retries, do not follow
-redirects, and close the response after observing headers without retaining a body.
-HTTP 200–299 is `SUCCEEDED`; other observed statuses are `FAILED/HTTP_STATUS`.
+E12 accepts general HTTP(S) destinations and validates every resolved IPv4/IPv6
+address and redirect hop. Production has no local-fixture exception by default.
+Connect and read deadlines are 500ms each, total time is 1500ms, and at most three
+redirects are followed. Checks perform no retries and close after final headers
+without consuming a response body. Final HTTP 200–299 is `SUCCEEDED`; other
+final statuses are `FAILED/HTTP_STATUS`.
 A transport failure records `FAILED/CONNECTION_FAILURE` or `FAILED/TIMEOUT` with
 `httpStatus: null`. E09 shows an active CheckRun, or the latest terminal result,
 on each Monitor; bounded history retains completed results. A separate worker
@@ -59,9 +60,9 @@ npm run db:up
 npm run db:migrate
 npm run fixture
 # A separate terminal:
-npm run dev:api
+NODE_ENV=test WSE_TEST_FIXTURE_ORIGIN=http://127.0.0.1:4311 npm run dev:api
 # A third terminal:
-npm run start:worker
+NODE_ENV=test WSE_TEST_FIXTURE_ORIGIN=http://127.0.0.1:4311 npm run start:worker
 # A fourth terminal:
 npm run dev:web
 ```
@@ -73,7 +74,9 @@ login command are described below. After signing in, create a Monitor for `/ok`,
 `/fail`, or `/redirect`, then choose **Run check**.
 
 `API_PORT` changes the API port (default 4312). `FIXTURE_PORT` changes the fixture
-listener (default 4311); set `FIXTURE_ORIGIN` on the API to the same trusted origin.
+listener (default 4311); set `WSE_TEST_FIXTURE_ORIGIN` on the test API and worker
+to that exact loopback origin. The override requires `NODE_ENV=test` and is
+ignored in production. Normal API/worker startup has no fixture exception.
 Do not expose this development service to a public interface.
 
 ```sh
@@ -85,6 +88,7 @@ npx playwright install chromium
 npm run test:e2e
 npm run test:execution
 npm run test:ownership
+npm run test:outbound
 ```
 
 Functional tests own fixed loopback ports 4311, 4312 and 4314; stop manual fixture
@@ -661,3 +665,44 @@ bounded repair adopted that source unchanged and recorded its provenance in
 `evidence/phase-1/E11/verification-attempt3.json`. The corrected full-suite and
 regression acceptance remain for root's final verification; no additional
 baseline or recovery run was spent by that repair.
+
+## E12 outbound destination and resource boundaries
+
+The URL parser canonicalizes HTTP(S) scheme, host, port and path, removes the
+fragment, and rejects credentials and unsafe literal destinations. At execution,
+all returned DNS addresses must pass the IPv4/IPv6 checks. Local, private,
+link-local, mapped/transition, multicast and reserved ranges are refused; IPv6
+is limited to ordinary global unicast outside special-purpose exclusions.
+The first validated numeric address is used directly, with the original Host
+and TLS servername. There is no second DNS lookup, address fallback or proxy.
+Each redirect repeats destination validation. Only an explicit internal test
+argument or the test bootstrap above permits its exact loopback fixture origin.
+The address exclusions use the [IANA IPv4](https://www.iana.org/assignments/iana-ipv4-special-registry/)
+and [IPv6 registries](https://www.iana.org/assignments/iana-ipv6-special-registry/);
+transport uses [Node HTTP](https://nodejs.org/docs/latest-v24.x/api/http.html)
+and [HTTPS](https://nodejs.org/docs/latest-v24.x/api/https.html) with no new dependency.
+
+The connect budget includes DNS/TCP/TLS; the read budget waits for response
+headers. A single total timer spans all hops. Closing after final headers consumes
+zero application body bytes, within the fixed 65536-byte ceiling; this does not
+claim zero TCP buffering. A 65537-byte HTTP200 body remains SUCCEEDED/200.
+
+| Outcome | Stored state/status | Failure disposition |
+| --- | --- | --- |
+| Final 2xx | SUCCEEDED / observed status | None |
+| Other final HTTP | FAILED / observed status | Retryable for429/5xx; otherwise permanent |
+| Transport timeout/connection failure | FAILED / null | Retryable |
+| Unsafe destination or redirect limit | ABORTED / null | Permanent |
+| Worker/store/local-service uncertainty | Pending until safe E11 recovery, then ABORTED / null | Uncertain |
+
+Migration `009_outbound_policy_result.sql` permits only the two new ABORTED
+policy reasons, `UNSAFE_DESTINATION` and `REDIRECT_LIMIT`; crash-recovery nulls
+and earlier migration bytes are unchanged. Policy errors do not log the URL.
+There is no automatic retry, terminal reopening or additional retry setting.
+
+`test:outbound` uses the frozen E12 resolver/connector stubs and controlled local
+slow/large/redirect fixtures. The private redirect in that same sequence uses
+an actual worker, proving persisted ABORTED and same-key replay with no guard
+request. No public or metadata endpoint is contacted. Existing live redirect
+and direct-refusal consumers adopt E12 semantics; all earlier frozen evidence
+is retained. E11 recovery scenarios were not rerun as E12 author smoke tests.
diff --git a/app/monitors/monitor-workspace.tsx b/app/monitors/monitor-workspace.tsx
index 6ed845e..1999c64 100644
--- a/app/monitors/monitor-workspace.tsx
+++ b/app/monitors/monitor-workspace.tsx
@@ -136,7 +136,7 @@ export default function MonitorWorkspace({ initial }: { initial: InitialMonitors
           <dt>State</dt><dd><span role="status">{monitor.latestCheck.state}</span></dd>
           <dt>HTTP status</dt><dd>{monitor.latestCheck.state === 'ABORTED' ? 'Unknown' : monitor.latestCheck.httpStatus ?? 'No response'}</dd>
           <dt>Latency</dt><dd>{monitor.latestCheck.state === 'ABORTED' ? 'Unknown' : monitor.latestCheck.latencyMs === null ? 'Pending' : `${monitor.latestCheck.latencyMs} ms`}</dd>
-          <dt>Failure reason</dt><dd>{monitor.latestCheck.state === 'ABORTED' ? 'Unknown' : monitor.latestCheck.failureReason ?? 'None'}</dd>
+          <dt>Failure reason</dt><dd>{monitor.latestCheck.failureReason ?? (monitor.latestCheck.state === 'ABORTED' ? 'Unknown' : 'None')}</dd>
           <dt>Finished</dt><dd>{monitor.latestCheck.finishedAt === null ? 'Pending' : <time dateTime={monitor.latestCheck.finishedAt}>{monitor.latestCheck.finishedAt}</time>}</dd>
         </dl> : <p>No checks yet.</p>}
         {historyId === monitor.id && <section aria-label={`Check history for ${monitor.name}`}>
@@ -157,7 +157,7 @@ export default function MonitorWorkspace({ initial }: { initial: InitialMonitors
               <thead><tr><th scope="col">Check ID</th><th scope="col">State</th><th scope="col">HTTP status</th><th scope="col">Latency</th><th scope="col">Failure reason</th><th scope="col">Finished</th></tr></thead>
               <tbody>{history.map((check) => <tr key={check.id}>
                 <td>{check.id}</td><td>{check.state}</td><td>{check.state === 'ABORTED' ? 'Unknown' : check.httpStatus ?? 'No response'}</td>
-                <td>{check.latencyMs === null ? 'Unknown' : `${check.latencyMs} ms`}</td><td>{check.state === 'ABORTED' ? 'Unknown' : check.failureReason ?? 'None'}</td>
+                <td>{check.latencyMs === null ? 'Unknown' : `${check.latencyMs} ms`}</td><td>{check.failureReason ?? (check.state === 'ABORTED' ? 'Unknown' : 'None')}</td>
                 <td><time dateTime={check.finishedAt}>{check.finishedAt}</time></td>
               </tr>)}</tbody>
             </table></div>}
diff --git a/package.json b/package.json
index c6a4417..ee56fe5 100644
--- a/package.json
+++ b/package.json
@@ -24,7 +24,8 @@
     "test:integration": "node --test --test-concurrency=1 test/persistence.test.ts test/storage-*.test.ts",
     "test:execution": "node --test test/execution.test.ts",
     "test:ownership": "node --test test/ownership.test.ts",
-    "test:recovery": "node --test test/recovery.test.ts"
+    "test:recovery": "node --test test/recovery.test.ts",
+    "test:outbound": "node --test test/outbound.test.ts"
   },
   "dependencies": {
     "fastify": "5.12.1",
diff --git a/playwright.config.ts b/playwright.config.ts
index af740dc..c2d5436 100644
--- a/playwright.config.ts
+++ b/playwright.config.ts
@@ -16,7 +16,7 @@ export default defineConfig({
   webServer: [
     { command: 'npm run fixture', url: 'http://127.0.0.1:4311/ok', reuseExistingServer: false, timeout: 30_000 },
     { command: 'node test/prepare-browser-db.ts && npm run start:api', url: 'http://127.0.0.1:4312/health', reuseExistingServer: false, timeout: 30_000,
-      env: { DATABASE_SCHEMA: 'e03_browser' } },
+      env: { DATABASE_SCHEMA: 'e03_browser', NODE_ENV: 'test', WSE_TEST_FIXTURE_ORIGIN: 'http://127.0.0.1:4311' } },
     { command: 'npm run start:web', url: 'http://127.0.0.1:4313/login', reuseExistingServer: false, timeout: 90_000,
       env: { NEXT_TELEMETRY_DISABLED: '1' } },
   ],
diff --git a/server/app.ts b/server/app.ts
index 8611bd1..e331e1d 100644
--- a/server/app.ts
+++ b/server/app.ts
@@ -1,7 +1,6 @@
 import { randomUUID } from 'node:crypto';
 import Fastify, { errorCodes } from 'fastify';
 import type { FastifyReply, FastifyRequest } from 'fastify';
-import { DEFAULT_FIXTURE_ORIGIN } from './check.ts';
 import { ApiError, ERROR_STATUS, idempotencyKey, monitorId, monitorInput } from './contracts.ts';
 import type { ApiFailure, Monitor, TerminalCheckRun } from './model.ts';
 import { databaseConfig, databasePool } from './database.ts';
@@ -29,7 +28,7 @@ const monitorViewSql = `SELECT m.id, m.owner_user_id, m.name, m.url, m.interval_
       ORDER BY (finished_at IS NULL) DESC, COALESCE(finished_at, queued_at) DESC, id DESC LIMIT 1
   ) c ON true`;
 
-export function buildApp(fixtureOrigin = DEFAULT_FIXTURE_ORIGIN, database: DatabaseConfig = databaseConfig(), now: () => number = Date.now) {
+export function buildApp(testFixtureOrigin: string | undefined = undefined, database: DatabaseConfig = databaseConfig(), now: () => number = Date.now) {
   const handleError = (error: unknown, _request: FastifyRequest, reply: FastifyReply) => {
     const failure = error instanceof ApiError ? error
       : inputErrors.some((ErrorType) => error instanceof ErrorType)
@@ -68,7 +67,7 @@ export function buildApp(fixtureOrigin = DEFAULT_FIXTURE_ORIGIN, database: Datab
 
   app.post<{ Body: unknown }>(
     '/monitors', async (request, reply) => {
-      const input = monitorInput(request.body, fixtureOrigin);
+      const input = monitorInput(request.body, testFixtureOrigin);
       const now = new Date().toISOString();
       const monitor: Monitor = {
         id: randomUUID(), ...input, createdAt: now, updatedAt: now,
@@ -82,7 +81,7 @@ export function buildApp(fixtureOrigin = DEFAULT_FIXTURE_ORIGIN, database: Datab
 
   app.put<{ Params: { id: string }; Body: unknown }>('/monitors/:id', async (request) => {
     const id = monitorId(request.params.id);
-    const input = monitorInput(request.body, fixtureOrigin);
+    const input = monitorInput(request.body, testFixtureOrigin);
     const result = await pool.query<MonitorRow>(`UPDATE monitors SET name = $2, url = $3,
       interval_seconds = $4, enabled = $5, updated_at = $6 WHERE id = $1 AND owner_user_id = $7 RETURNING *`,
     [id, input.name, input.url, input.interval, input.enabled, new Date(), request.user!.id]);
diff --git a/server/check.ts b/server/check.ts
index 330024a..e811a04 100644
--- a/server/check.ts
+++ b/server/check.ts
@@ -1,58 +1,101 @@
 import { randomUUID } from 'node:crypto';
-import { request as httpRequest } from 'node:http';
-import { request as httpsRequest } from 'node:https';
-import type { CheckRun, Monitor, ObservedCheckRun } from './model.ts';
+import type { ClientRequest } from 'node:http';
+import type { CheckRun, Monitor, TerminalCheckRun } from './model.ts';
+import { CONNECT_TIMEOUT_MS, READ_TIMEOUT_MS, TOTAL_TIMEOUT_MS, MAX_REDIRECTS,
+  OutboundPolicyError, connectDestination, validatedDestination } from './outbound.ts';
+import type { Connector, Resolver } from './outbound.ts';
+export { DEFAULT_FIXTURE_ORIGIN } from './outbound.ts';
 
-export const DEFAULT_FIXTURE_ORIGIN = 'http://127.0.0.1:4311';
-export const CHECK_TIMEOUT_MS = 2_000;
-
-export function fixtureUrl(value: string, fixtureOrigin: string): URL {
-  const url = new URL(value);
-  const allowed = new URL(fixtureOrigin);
-  if (
-    !['http:', 'https:'].includes(url.protocol) ||
-    url.origin !== allowed.origin ||
-    url.username || url.password
-  ) {
-    throw new Error('Only the configured fixture origin is allowed.');
-  }
-  return url;
-}
-
-export async function checkMonitor(monitor: Monitor, fixtureOrigin: string,
-  execution?: Pick<ObservedCheckRun, 'id' | 'trigger' | 'startedAt'>): Promise<ObservedCheckRun> {
-  // Recheck at the outbound boundary, even though creation also checks the URL.
-  const url = fixtureUrl(monitor.url, fixtureOrigin);
+export async function checkMonitor(monitor: Monitor, testFixtureOrigin?: string,
+  execution?: Pick<TerminalCheckRun, 'id' | 'trigger' | 'startedAt'>,
+  dependencies: { resolve?: Resolver; connect?: Connector } = {}): Promise<TerminalCheckRun> {
   const startedAt = execution?.startedAt ?? new Date().toISOString();
   const start = performance.now();
-
-  return new Promise((resolve) => {
+  return new Promise((resolve, reject) => {
     let settled = false;
+    let current: ClientRequest | undefined;
+    let connectTimer: ReturnType<typeof setTimeout> | undefined;
+    let readTimer: ReturnType<typeof setTimeout> | undefined;
+    const clearHop = () => { clearTimeout(connectTimer); clearTimeout(readTimer); };
     const finish = (httpStatus: number | null, failureReason: CheckRun['failureReason']) => {
       if (settled) return;
       settled = true;
-      clearTimeout(deadline);
-      resolve({
+      clearTimeout(totalTimer); clearHop(); current?.destroy();
+      const common: Pick<TerminalCheckRun, 'id' | 'monitorId' | 'trigger' | 'startedAt' | 'finishedAt'> = {
         id: execution?.id ?? randomUUID(), monitorId: monitor.id, trigger: execution?.trigger ?? 'MANUAL',
-        state: failureReason === null ? 'SUCCEEDED' : 'FAILED',
-        httpStatus, latencyMs: Math.round(performance.now() - start), failureReason,
         startedAt, finishedAt: new Date().toISOString(),
-      });
+      };
+      if (failureReason === 'UNSAFE_DESTINATION' || failureReason === 'REDIRECT_LIMIT') {
+        resolve({ ...common, state: 'ABORTED', httpStatus: null, latencyMs: null, failureReason });
+      } else {
+        resolve({ ...common, state: failureReason === null ? 'SUCCEEDED' : 'FAILED',
+          httpStatus, latencyMs: Math.round(performance.now() - start), failureReason });
+      }
     };
-    const request = (url.protocol === 'https:' ? httpsRequest : httpRequest)(url, {
-      method: 'GET', agent: false,
-      headers: { 'user-agent': 'monitor-fixture-check/0.1' },
-    }, (response) => {
-      const status = response.statusCode ?? 0;
-      // E01 observes response headers only: no body is retained and no redirect is followed.
-      response.destroy();
-      finish(status, status >= 200 && status < 300 ? null : 'HTTP_STATUS');
-    });
-    const deadline = setTimeout(() => {
-      finish(null, 'TIMEOUT');
-      request.destroy();
-    }, CHECK_TIMEOUT_MS);
-    request.on('error', () => finish(null, 'CONNECTION_FAILURE'));
-    request.end();
+    const totalTimer = setTimeout(() => finish(null, 'TIMEOUT'), TOTAL_TIMEOUT_MS);
+    const serviceFailure = () => {
+      if (settled) return;
+      settled = true;
+      clearTimeout(totalTimer); clearHop(); current?.destroy();
+      // The worker leaves RUNNING for E11 recovery; do not invent an endpoint result.
+      reject(new Error('Outbound execution stopped without an authoritative result.'));
+    };
+    async function visit(value: string, redirects: number): Promise<void> {
+      clearHop();
+      connectTimer = setTimeout(() => finish(null, 'TIMEOUT'), CONNECT_TIMEOUT_MS);
+      try {
+        const destination = await validatedDestination(value, testFixtureOrigin, dependencies.resolve);
+        if (settled) return;
+        const request = (dependencies.connect ?? connectDestination)(destination, response => {
+          if (settled || current !== request) { response.destroy(); return; }
+          clearHop();
+          const status = response.statusCode;
+          const location = response.headers.location;
+          // No response body is consumed, including large 2xx bodies. A local
+          // resource cap must not invent FAILED/200 after final headers.
+          response.destroy();
+          if (status !== undefined && [301, 302, 303, 307, 308].includes(status) && location !== undefined) {
+            current = undefined; request.destroy();
+            if (redirects >= MAX_REDIRECTS) { finish(null, 'REDIRECT_LIMIT'); return; }
+            let next: string;
+            try { next = new URL(location, destination.url).href; }
+            catch { finish(null, 'UNSAFE_DESTINATION'); return; }
+            void visit(next, redirects + 1);
+          } else if (status === undefined) finish(null, 'CONNECTION_FAILURE');
+          else finish(status, status >= 200 && status < 300 ? null : 'HTTP_STATUS');
+        });
+        current = request;
+        request.once('socket', socket => {
+          const connected = () => {
+            if (settled || current !== request) return;
+            clearTimeout(connectTimer);
+            readTimer = setTimeout(() => finish(null, 'TIMEOUT'), READ_TIMEOUT_MS);
+          };
+          if (destination.url.protocol === 'https:') socket.once('secureConnect', connected);
+          else if (socket.connecting) socket.once('connect', connected);
+          else connected();
+        });
+        request.on('error', (error: NodeJS.ErrnoException) => {
+          if (current !== request) return;
+          if (['ENOMEM', 'EMFILE', 'ENFILE'].includes(error.code ?? '')) serviceFailure();
+          else finish(null, 'CONNECTION_FAILURE');
+        });
+        request.end();
+      } catch (error) {
+        const code = error instanceof Error && 'code' in error ? error.code : undefined;
+        if (error instanceof OutboundPolicyError) finish(null, error.reason);
+        else if (typeof code === 'string' && ['ENOTFOUND', 'EAI_AGAIN', 'EAI_FAIL', 'ECONNREFUSED', 'ENETUNREACH', 'EHOSTUNREACH'].includes(code)) finish(null, 'CONNECTION_FAILURE');
+        else serviceFailure();
+      }
+    }
+    void visit(monitor.url, 0);
   });
 }
+
+export function failureDisposition(check: TerminalCheckRun) {
+  if (check.state === 'SUCCEEDED') return 'none';
+  if (check.failureReason === 'UNSAFE_DESTINATION' || check.failureReason === 'REDIRECT_LIMIT') return 'permanent';
+  if (check.state === 'ABORTED') return 'uncertain';
+  if (check.failureReason === 'HTTP_STATUS') return check.httpStatus === 429 || (check.httpStatus ?? 0) >= 500 ? 'retryable' : 'permanent';
+  return 'retryable';
+}
diff --git a/server/contracts.ts b/server/contracts.ts
index 5a2421d..d2c271f 100644
--- a/server/contracts.ts
+++ b/server/contracts.ts
@@ -1,4 +1,4 @@
-import { fixtureUrl } from './check.ts';
+import { canonicalUrl } from './outbound.ts';
 import type { ApiErrorCode, Monitor } from './model.ts';
 
 export const ERROR_STATUS = { INVALID_INPUT: 400, UNAUTHENTICATED: 401, FORBIDDEN: 403, NOT_FOUND: 404, CONFLICT: 409, INTERNAL_ERROR: 500 } as const;
@@ -13,7 +13,7 @@ export class ApiError extends Error {
   }
 }
 
-export function monitorInput(value: unknown, fixtureOrigin: string): Pick<Monitor, 'name' | 'url' | 'interval' | 'enabled'> {
+export function monitorInput(value: unknown, testFixtureOrigin?: string): Pick<Monitor, 'name' | 'url' | 'interval' | 'enabled'> {
   if (value === null || typeof value !== 'object' || Array.isArray(value)) {
     throw new ApiError('INVALID_INPUT', 'A Monitor JSON object is required.');
   }
@@ -28,11 +28,11 @@ export function monitorInput(value: unknown, fixtureOrigin: string): Pick<Monito
     throw new ApiError('INVALID_INPUT', 'Enabled must be a boolean.');
   }
   if (typeof url !== 'string') {
-    throw new ApiError('INVALID_INPUT', 'An absolute HTTP(S) fixture URL is required.');
+    throw new ApiError('INVALID_INPUT', 'An absolute HTTP(S) URL is required.');
   }
   let parsedUrl: URL;
-  try { parsedUrl = fixtureUrl(url, fixtureOrigin); }
-  catch { throw new ApiError('INVALID_INPUT', 'An absolute HTTP(S) URL on the configured fixture origin is required.'); }
+  try { parsedUrl = canonicalUrl(url, testFixtureOrigin); }
+  catch { throw new ApiError('INVALID_INPUT', 'A permitted absolute HTTP(S) URL without credentials is required.'); }
   return { name: name.trim(), url: parsedUrl.href, interval, enabled };
 }
 
diff --git a/server/main.ts b/server/main.ts
index 5d27ec8..e9b5e9a 100644
--- a/server/main.ts
+++ b/server/main.ts
@@ -1,7 +1,7 @@
 import { buildApp } from './app.ts';
-import { DEFAULT_FIXTURE_ORIGIN } from './check.ts';
+import { configuredTestFixtureOrigin } from './outbound.ts';
 
-const app = buildApp(process.env.FIXTURE_ORIGIN ?? DEFAULT_FIXTURE_ORIGIN);
+const app = buildApp(configuredTestFixtureOrigin());
 await app.listen({ host: '127.0.0.1', port: Number(process.env.API_PORT ?? 4312) });
 console.log('Monitor API listening on loopback.');
 for (const signal of ['SIGINT', 'SIGTERM'] as const) {
diff --git a/server/migrate.ts b/server/migrate.ts
index 8dc816e..28a35a5 100644
--- a/server/migrate.ts
+++ b/server/migrate.ts
@@ -5,7 +5,7 @@ import { databaseConfig, databasePool, schemaIdentifier } from './database.ts';
 import type { DatabaseConfig } from './database.ts';
 
 export async function migrationFiles() {
-  const names = ['001_monitors.sql', '002_check_runs.sql', '003_sessions.sql', '004_monitor_ownership.sql', '005_check_history_index.sql', '006_check_queue.sql', '007_check_ownership.sql', '008_check_lease.sql'];
+  const names = ['001_monitors.sql', '002_check_runs.sql', '003_sessions.sql', '004_monitor_ownership.sql', '005_check_history_index.sql', '006_check_queue.sql', '007_check_ownership.sql', '008_check_lease.sql', '009_outbound_policy_result.sql'];
   return Promise.all(names.map(async (version) => {
     const sql = await readFile(new URL(`./migrations/${version}`, import.meta.url), 'utf8');
     return { version, sql, checksum: createHash('sha256').update(sql).digest('hex') };
diff --git a/server/migrations/009_outbound_policy_result.sql b/server/migrations/009_outbound_policy_result.sql
new file mode 100644
index 0000000..74725c1
--- /dev/null
+++ b/server/migrations/009_outbound_policy_result.sql
@@ -0,0 +1,16 @@
+ALTER TABLE check_runs DROP CONSTRAINT check_runs_lifecycle_check;
+ALTER TABLE check_runs ADD CONSTRAINT check_runs_lifecycle_check CHECK (
+  (state = 'QUEUED' AND started_at IS NULL AND finished_at IS NULL
+    AND http_status IS NULL AND latency_ms IS NULL AND failure_reason IS NULL)
+  OR (state = 'RUNNING' AND started_at IS NOT NULL AND finished_at IS NULL
+    AND http_status IS NULL AND latency_ms IS NULL AND failure_reason IS NULL)
+  OR (state = 'ABORTED' AND started_at IS NOT NULL AND finished_at IS NOT NULL
+    AND http_status IS NULL AND latency_ms IS NULL
+    AND (failure_reason IS NULL OR failure_reason IN ('UNSAFE_DESTINATION', 'REDIRECT_LIMIT')))
+  OR (state IN ('SUCCEEDED', 'FAILED') AND started_at IS NOT NULL AND finished_at IS NOT NULL
+    AND latency_ms IS NOT NULL AND (
+      (state = 'SUCCEEDED' AND http_status IS NOT NULL AND http_status BETWEEN 200 AND 299 AND failure_reason IS NULL)
+      OR (state = 'FAILED' AND failure_reason = 'HTTP_STATUS' AND http_status IS NOT NULL AND (http_status < 200 OR http_status >= 300))
+      OR (state = 'FAILED' AND failure_reason IN ('TIMEOUT', 'CONNECTION_FAILURE') AND http_status IS NULL)
+    ))
+);
diff --git a/server/model.ts b/server/model.ts
index b41c3c0..a8a1199 100644
--- a/server/model.ts
+++ b/server/model.ts
@@ -8,6 +8,8 @@ export type Monitor = {
   updatedAt: string;
 };
 
+export type PolicyFailureReason = 'UNSAFE_DESTINATION' | 'REDIRECT_LIMIT';
+
 export type CheckRun = {
   id: string;
   monitorId: string;
@@ -15,7 +17,7 @@ export type CheckRun = {
   state: 'QUEUED' | 'RUNNING' | 'SUCCEEDED' | 'FAILED' | 'ABORTED';
   httpStatus: number | null;
   latencyMs: number | null;
-  failureReason: 'HTTP_STATUS' | 'TIMEOUT' | 'CONNECTION_FAILURE' | null;
+  failureReason: 'HTTP_STATUS' | 'TIMEOUT' | 'CONNECTION_FAILURE' | PolicyFailureReason | null;
   startedAt: string | null;
   finishedAt: string | null;
 };
@@ -25,7 +27,7 @@ export type ObservedCheckRun = CheckRun & {
 };
 
 export type TerminalCheckRun = ObservedCheckRun | (CheckRun & {
-  state: 'ABORTED'; httpStatus: null; latencyMs: null; failureReason: null; startedAt: string; finishedAt: string;
+  state: 'ABORTED'; httpStatus: null; latencyMs: null; failureReason: PolicyFailureReason | null; startedAt: string; finishedAt: string;
 });
 
 export type MonitorView = Monitor & { latestCheck: CheckRun | null };
diff --git a/server/outbound.ts b/server/outbound.ts
new file mode 100644
index 0000000..d4e5d4b
--- /dev/null
+++ b/server/outbound.ts
@@ -0,0 +1,103 @@
+import { lookup } from 'node:dns/promises';
+import { BlockList, isIP } from 'node:net';
+import { request as httpRequest } from 'node:http';
+import { request as httpsRequest } from 'node:https';
+import type { ClientRequest, IncomingMessage } from 'node:http';
+import type { PolicyFailureReason } from './model.ts';
+
+export const DEFAULT_FIXTURE_ORIGIN = 'http://127.0.0.1:4311';
+export const CONNECT_TIMEOUT_MS = 500;
+export const READ_TIMEOUT_MS = 500;
+export const TOTAL_TIMEOUT_MS = 1500;
+export const MAX_REDIRECTS = 3;
+
+const blockedV4 = new BlockList();
+for (const [address, prefix] of [
+  ['0.0.0.0', 8], ['10.0.0.0', 8], ['100.64.0.0', 10], ['127.0.0.0', 8],
+  ['169.254.0.0', 16], ['172.16.0.0', 12], ['192.0.0.0', 24], ['192.0.2.0', 24],
+  ['192.88.99.0', 24], ['192.168.0.0', 16], ['198.18.0.0', 15], ['198.51.100.0', 24],
+  ['203.0.113.0', 24], ['224.0.0.0', 4], ['240.0.0.0', 4],
+] as const) blockedV4.addSubnet(address, prefix, 'ipv4');
+const globalV6 = new BlockList();
+globalV6.addSubnet('2000::', 3, 'ipv6');
+const blockedV6 = new BlockList();
+for (const [address, prefix] of [['2001::', 23], ['2001:db8::', 32], ['2002::', 16], ['3fff::', 20]] as const) {
+  blockedV6.addSubnet(address, prefix, 'ipv6');
+}
+
+export class OutboundPolicyError extends Error {
+  readonly reason: PolicyFailureReason;
+  constructor(reason: PolicyFailureReason) {
+    super(reason === 'REDIRECT_LIMIT' ? 'The redirect limit was reached.' : 'The outbound destination is not permitted.');
+    this.reason = reason;
+  }
+}
+
+export function publicAddress(address: string): boolean {
+  if (address.includes('%')) return false;
+  const family = isIP(address);
+  if (family === 4) return !blockedV4.check(address, 'ipv4');
+  // Fail closed outside ordinary global unicast, including mapped IPv4,
+  // unspecified, loopback, local, multicast and transition address ranges.
+  return family === 6 && globalV6.check(address, 'ipv6') && !blockedV6.check(address, 'ipv6');
+}
+
+function hostname(url: URL) { return url.hostname.replace(/^\[|\]$/g, ''); }
+function trustedFixture(url: URL, origin?: string) {
+  if (!origin) return false;
+  const fixture = new URL(origin);
+  if (!['http:', 'https:'].includes(fixture.protocol) || !['127.0.0.1', '[::1]'].includes(fixture.hostname) || fixture.username || fixture.password) {
+    throw new OutboundPolicyError('UNSAFE_DESTINATION');
+  }
+  return url.origin === fixture.origin;
+}
+
+export function configuredTestFixtureOrigin(): string | undefined {
+  return process.env.NODE_ENV === 'test' ? process.env.WSE_TEST_FIXTURE_ORIGIN : undefined;
+}
+
+export function canonicalUrl(value: string, testFixtureOrigin?: string): URL {
+  let url: URL;
+  try { url = new URL(value); }
+  catch { throw new OutboundPolicyError('UNSAFE_DESTINATION'); }
+  if (!['http:', 'https:'].includes(url.protocol) || url.username || url.password || /^[^:]+:\/\/[^/?#]*@/.test(value)) {
+    throw new OutboundPolicyError('UNSAFE_DESTINATION');
+  }
+  url.hash = '';
+  if (url.hostname.endsWith('.')) url.hostname = url.hostname.slice(0, -1);
+  const host = hostname(url);
+  if (!trustedFixture(url, testFixtureOrigin) && (host === 'localhost' || host.endsWith('.localhost') || (isIP(host) !== 0 && !publicAddress(host)))) {
+    throw new OutboundPolicyError('UNSAFE_DESTINATION');
+  }
+  return url;
+}
+
+export type Resolver = (host: string) => Promise<readonly { address: string; family: number }[]>;
+export type Destination = { url: URL; address: string; family: 4 | 6 };
+export type Connector = (destination: Destination, response: (message: IncomingMessage) => void) => ClientRequest;
+
+export async function validatedDestination(value: string, testFixtureOrigin?: string,
+  resolve: Resolver = host => lookup(host, { all: true, verbatim: true })): Promise<Destination> {
+  const url = canonicalUrl(value, testFixtureOrigin);
+  const host = hostname(url);
+  const literalFamily = isIP(host);
+  const addresses = literalFamily ? [{ address: host, family: literalFamily }] : await resolve(host);
+  if (!addresses.length || addresses.some(item => isIP(item.address) !== item.family || ![4, 6].includes(item.family) ||
+    (!trustedFixture(url, testFixtureOrigin) && !publicAddress(item.address)))) {
+    throw new OutboundPolicyError('UNSAFE_DESTINATION');
+  }
+  return { url, address: addresses[0].address, family: addresses[0].family as 4 | 6 };
+}
+
+export const connectDestination: Connector = (destination, response) => {
+  const { url, address, family } = destination;
+  const options = {
+    protocol: url.protocol, hostname: address, family, autoSelectFamily: false,
+    port: url.port || undefined, path: url.pathname + url.search, method: 'GET', agent: false as const,
+    servername: isIP(hostname(url)) === 0 ? hostname(url) : undefined,
+    headers: { host: url.host, 'user-agent': 'monitor-check/0.1' },
+  };
+  // Connect to a validated numeric address, never resolve the user hostname a
+  // second time. Host and TLS servername retain the original authority.
+  return (url.protocol === 'https:' ? httpsRequest : httpRequest)(options, response);
+};
diff --git a/server/worker.ts b/server/worker.ts
index 4b75854..fb15d9c 100644
--- a/server/worker.ts
+++ b/server/worker.ts
@@ -2,10 +2,11 @@ import type { Pool } from 'pg';
 import { randomUUID } from 'node:crypto';
 import { pathToFileURL } from 'node:url';
 import { setTimeout as delay } from 'node:timers/promises';
-import { checkMonitor, DEFAULT_FIXTURE_ORIGIN } from './check.ts';
+import { checkMonitor } from './check.ts';
+import { configuredTestFixtureOrigin } from './outbound.ts';
 import { databaseConfig, databasePool } from './database.ts';
 import { checkRunFromRow, monitorFromRow } from './mapping.ts';
-import type { ObservedCheckRun } from './model.ts';
+import type { TerminalCheckRun } from './model.ts';
 import type { CheckRunRow, MonitorRow } from './mapping.ts';
 import { verifySchema } from './schema.ts';
 
@@ -34,7 +35,7 @@ export async function recoverExpiredChecks(pool: Pool, now?: Date) {
   return recovered.rows.map(checkRunFromRow);
 }
 
-export async function completeCheck(pool: Pool, workerId: string, observed: ObservedCheckRun) {
+export async function completeCheck(pool: Pool, workerId: string, observed: TerminalCheckRun) {
   const client = await pool.connect();
   try {
     // Never leave an autocommit terminal write running after its worker dies.
@@ -54,7 +55,7 @@ export async function completeCheck(pool: Pool, workerId: string, observed: Obse
   finally { client.release(); }
 }
 
-export async function runNextCheck(pool: Pool, fixtureOrigin = DEFAULT_FIXTURE_ORIGIN, workerId = randomUUID(), signal?: AbortSignal) {
+export async function runNextCheck(pool: Pool, testFixtureOrigin?: string, workerId = randomUUID(), signal?: AbortSignal) {
   if (signal?.aborted) return null;
   const client = await pool.connect();
   let claimed: CheckRunRow | undefined;
@@ -79,7 +80,7 @@ export async function runNextCheck(pool: Pool, fixtureOrigin = DEFAULT_FIXTURE_O
   const monitor = await pool.query<MonitorRow>('SELECT * FROM monitors WHERE id = $1', [claimed.monitor_id]);
   if (!monitor.rows[0]) return null;
   const execution = checkRunFromRow(claimed);
-  const observed = await checkMonitor(monitorFromRow(monitor.rows[0]), fixtureOrigin, {
+  const observed = await checkMonitor(monitorFromRow(monitor.rows[0]), testFixtureOrigin, {
     id: execution.id, trigger: execution.trigger, startedAt: execution.startedAt!,
   });
   return completeCheck(pool, workerId, observed);
@@ -109,7 +110,7 @@ if (process.argv[1] && import.meta.url === pathToFileURL(process.argv[1]).href)
       // always schedules. There is no separate scheduler daemon or queue store.
       if (!process.argv.includes('--no-schedule')) await scheduleDueChecks(pool);
       if (stopping.signal.aborted) break;
-      const completed = await runNextCheck(pool, process.env.FIXTURE_ORIGIN ?? DEFAULT_FIXTURE_ORIGIN, workerId, stopping.signal);
+      const completed = await runNextCheck(pool, configuredTestFixtureOrigin(), workerId, stopping.signal);
       if (stopping.signal.aborted) break;
       await recoverExpiredChecks(pool);
       if (!completed && !stopping.signal.aborted) await delay(250);
diff --git a/test/execution.test.ts b/test/execution.test.ts
index 2e3293b..eaccced 100644
--- a/test/execution.test.ts
+++ b/test/execution.test.ts
@@ -47,7 +47,7 @@ test('E09 persisted202, separate worker and browser follow one held execution',
   const config = { ...databaseConfig(), schema: scenario.schema };
   const pool = databasePool(config);
   const fixture = heldFixture();
-  const app = buildApp(undefined, config);
+  const app = buildApp(new URL(scenario.monitors[0].url).origin, config);
   let owned = false;
   let worker: Awaited<ReturnType<typeof startTestWorker>> | undefined;
   let web: ReturnType<typeof spawn> | undefined;
diff --git a/test/functional.test.ts b/test/functional.test.ts
index b96fa13..fc0486d 100644
--- a/test/functional.test.ts
+++ b/test/functional.test.ts
@@ -86,13 +86,13 @@ test('GET /fail is an observed endpoint failure with HTTP 503', async () => {
   assert.equal(fixture.calls.get('/fail'), 1);
 });
 
-test('the check does not follow even a same-origin redirect', async () => {
+test('the check follows a validated same-origin redirect to its final status', async () => {
   const previousOkCalls = fixture.calls.get('/ok');
   const monitor = await create('/redirect');
   const result = await check(monitor.id);
-  assert.equal(result.state, 'FAILED');
-  assert.equal(result.httpStatus, 302);
-  assert.equal(fixture.calls.get('/ok'), previousOkCalls);
+  assert.equal(result.state, 'SUCCEEDED');
+  assert.equal(result.httpStatus, 200);
+  assert.equal(fixture.calls.get('/ok'), (previousOkCalls ?? 0) + 1);
   assert.equal(fixture.calls.get('/redirect'), 1);
 });
 
@@ -101,7 +101,10 @@ test('a non-fixture URL is rejected without contacting the controlled guard', as
   const unsafe: Monitor = { ...monitor, url: 'http://127.0.0.1:4314/ok' };
   const response = await inject({ method: 'POST', url: '/monitors', payload: unsafe });
   assert.equal(response.statusCode, 400);
-  await assert.rejects(checkMonitor(unsafe, DEFAULT_FIXTURE_ORIGIN));
+  const refused = await checkMonitor(unsafe, DEFAULT_FIXTURE_ORIGIN);
+  assert.equal(refused.state, 'ABORTED');
+  assert.equal(refused.httpStatus, null);
+  assert.equal(refused.failureReason, 'UNSAFE_DESTINATION');
   assert.equal(guardCalls, 0);
 });
 
diff --git a/test/outbound.test.ts b/test/outbound.test.ts
new file mode 100644
index 0000000..4bcbcef
--- /dev/null
+++ b/test/outbound.test.ts
@@ -0,0 +1,244 @@
+import { test } from 'node:test';
+import assert from 'node:assert/strict';
+import { EventEmitter } from 'node:events';
+import { Socket } from 'node:net';
+import { IncomingMessage } from 'node:http';
+import type { ClientRequest } from 'node:http';
+import { mkdir, writeFile } from 'node:fs/promises';
+import { buildApp } from '../server/app.ts';
+import { checkMonitor, failureDisposition } from '../server/check.ts';
+import { canonicalUrl, configuredTestFixtureOrigin, connectDestination, publicAddress,
+  CONNECT_TIMEOUT_MS, READ_TIMEOUT_MS, TOTAL_TIMEOUT_MS, MAX_REDIRECTS } from '../server/outbound.ts';
+import type { Connector, Destination } from '../server/outbound.ts';
+import type { Monitor, TerminalCheckRun } from '../server/model.ts';
+import { databaseConfig, databasePool, schemaIdentifier } from '../server/database.ts';
+import { migrate } from '../server/migrate.ts';
+import { authenticatedInject, loginForTest } from './auth.ts';
+import { startTestWorker, waitForTerminalCheck } from './worker.ts';
+import { insertOutboundUser, outboundFixture, requireFreePorts } from '../evidence/phase-1/E12/fixture.ts';
+import scenario from '../evidence/phase-1/E12/scenario.json' with { type: 'json' };
+
+function monitor(url: string): Monitor {
+  return { ...scenario.monitor, id: scenario.user.id, url, createdAt: new Date(0).toISOString(), updatedAt: new Date(0).toISOString() };
+}
+async function record(name: string, value: unknown) {
+  await mkdir('output/phase-1/e12', { recursive: true });
+  await writeFile(`output/phase-1/e12/${name}.json`, JSON.stringify(value, null, 2) + '\n');
+}
+async function bounded<T>(promise: Promise<T>) {
+  let timer: ReturnType<typeof setTimeout> | undefined;
+  try {
+    return await Promise.race([promise, new Promise<never>((_, reject) => {
+      timer = setTimeout(() => reject(new Error('E12 fixture cleanup timed out.')), scenario.watchdogMs);
+    })]);
+  } finally { clearTimeout(timer); }
+}
+
+// These stubs never create a network connection, including for public addresses.
+function stubConnector(responses: { status: number; location?: string }[] = [{ status: 200 }], stalled = false) {
+  const calls: Destination[] = [];
+  let destroyed = 0;
+  const connect: Connector = (destination, receive) => {
+    const index = calls.push(destination) - 1;
+    const request = new EventEmitter() as ClientRequest;
+    let closed = false;
+    request.destroy = () => { if (!closed) { closed = true; destroyed++; request.emit('close'); } return request; };
+    request.end = ((...args: unknown[]) => {
+      assert.equal(args.length, 0, 'GET must have no request body.');
+      if (!stalled) queueMicrotask(() => {
+        if (closed) return;
+        const socket = new Socket();
+        request.emit('socket', socket);
+        socket.emit(destination.url.protocol === 'https:' ? 'secureConnect' : 'connect');
+        const response = new IncomingMessage(socket);
+        const next = responses[index] ?? { status: 200 };
+        response.statusCode = next.status;
+        response.headers = next.location === undefined ? {} : { location: next.location };
+        receive(response);
+      });
+      return request;
+    }) as ClientRequest['end'];
+    return request;
+  };
+  return { connect, calls, destroyed: () => destroyed };
+}
+
+test('E12 canonical URLs, resolved IPv4/IPv6, redirect guards and DNS pinning never call an unsafe connector', async () => {
+  assert.equal(canonicalUrl('HTTP://PUBLIC.E12.TEST:80/a/../ok#fragment').href, scenario.monitor.url);
+  const decisions = [];
+  for (const item of scenario.destinationCases) {
+    const stub = stubConnector();
+    let resolutions = 0;
+    const result = await checkMonitor(monitor(item.url), undefined, undefined, {
+      resolve: async () => {
+        resolutions++;
+        return ('addresses' in item ? item.addresses! : []).map(address => ({ address, family: address.includes(':') ? 6 : 4 }));
+      }, connect: stub.connect,
+    });
+    assert.equal(result.state, item.allowed ? 'SUCCEEDED' : 'ABORTED');
+    assert.equal(result.httpStatus, item.allowed ? 200 : null);
+    assert.equal(result.failureReason, item.allowed ? null : 'UNSAFE_DESTINATION');
+    assert.equal(stub.calls.length, item.allowed ? 1 : 0);
+    assert.ok(stub.calls.every(destination => publicAddress(destination.address)));
+    assert.equal(failureDisposition(result), item.allowed ? 'none' : 'permanent');
+    decisions.push({ input: item, result, resolutions, connectorCalls: stub.calls.length, unsafeConnectorCalls: 0 });
+  }
+  const redirected = stubConnector([{ status: 302, location: scenario.redirectPrivate.location }]);
+  const redirect = await checkMonitor(monitor(scenario.redirectPrivate.from), undefined, undefined, {
+    resolve: async () => [{ address: scenario.publicIpv4, family: 4 }], connect: redirected.connect,
+  });
+  assert.equal(redirect.state, 'ABORTED');
+  assert.equal(redirect.failureReason, 'UNSAFE_DESTINATION');
+  assert.equal(redirect.httpStatus, null);
+  assert.equal(redirected.calls.length, scenario.redirectPrivate.expectedInitialConnectorCalls);
+
+  let resolverCalls = 0;
+  const rebound = stubConnector();
+  const result = await checkMonitor(monitor(scenario.dnsRebinding.url), undefined, undefined, {
+    resolve: async () => (++resolverCalls === 1 ? scenario.dnsRebinding.first : scenario.dnsRebinding.second)
+      .map(address => ({ address, family: 4 })), connect: rebound.connect,
+  });
+  assert.equal(result.state, 'SUCCEEDED');
+  assert.equal(resolverCalls, scenario.dnsRebinding.expectedResolverCalls);
+  assert.deepEqual(rebound.calls.map(destination => destination.address), [scenario.dnsRebinding.expectedConnectedAddress]);
+  assert.ok(rebound.calls.every(destination => publicAddress(destination.address)));
+
+  const previousNodeEnv = process.env.NODE_ENV;
+  const previousFixture = process.env.WSE_TEST_FIXTURE_ORIGIN;
+  try {
+    Reflect.set(process.env, 'NODE_ENV', 'production');
+    Reflect.set(process.env, 'WSE_TEST_FIXTURE_ORIGIN', scenario.fixtureOrigin);
+    assert.equal(configuredTestFixtureOrigin(), undefined);
+    assert.throws(() => canonicalUrl(`${scenario.fixtureOrigin}/ok`), { reason: 'UNSAFE_DESTINATION' });
+    assert.equal(canonicalUrl(`${scenario.fixtureOrigin}/ok`, scenario.fixtureOrigin).origin, scenario.fixtureOrigin);
+  } finally {
+    if (previousNodeEnv === undefined) Reflect.deleteProperty(process.env, 'NODE_ENV'); else Reflect.set(process.env, 'NODE_ENV', previousNodeEnv);
+    if (previousFixture === undefined) Reflect.deleteProperty(process.env, 'WSE_TEST_FIXTURE_ORIGIN'); else Reflect.set(process.env, 'WSE_TEST_FIXTURE_ORIGIN', previousFixture);
+  }
+  await record('destinations', { result: 'PASS', decisions, redirect, initialRedirectConnections: redirected.calls.length,
+    rebinding: { resolverCalls, connectedAddresses: rebound.calls.map(destination => destination.address), unsafeConnectorCalls: 0 },
+    productionIgnoresFixtureOverride: true, actualNetworkConnections: 0 });
+});
+
+test('E12 connect timeout is bounded and terminal failure reasons have explicit dispositions without retries', async () => {
+  assert.deepEqual([CONNECT_TIMEOUT_MS, READ_TIMEOUT_MS, TOTAL_TIMEOUT_MS, MAX_REDIRECTS],
+    [scenario.resources.connectTimeoutMs, scenario.resources.readTimeoutMs, scenario.resources.totalTimeoutMs, scenario.resources.redirectsMax]);
+  const stalled = stubConnector([], true);
+  const began = performance.now();
+  const timedOut = await checkMonitor(monitor(scenario.monitor.url), undefined, undefined, {
+    resolve: async () => [{ address: scenario.publicIpv4, family: 4 }], connect: stalled.connect,
+  });
+  const durationMs = Math.round(performance.now() - began);
+  assert.equal(timedOut.state, 'FAILED');
+  assert.equal(timedOut.httpStatus, null);
+  assert.equal(timedOut.failureReason, 'TIMEOUT');
+  assert.equal(failureDisposition(timedOut), 'retryable');
+  assert.equal(stalled.calls.length, 1);
+  assert.equal(stalled.destroyed(), 1);
+  assert.ok(durationMs >= scenario.resources.connectTimeoutMs - 1 && durationMs <= scenario.completionCeilingMs);
+  const outcomes = [];
+  for (const [status, disposition] of [[404, 'permanent'], [429, 'retryable'], [503, 'retryable']] as const) {
+    const stub = stubConnector([{ status }]);
+    const check = await checkMonitor(monitor(scenario.monitor.url), undefined, undefined, {
+      resolve: async () => [{ address: scenario.publicIpv4, family: 4 }], connect: stub.connect,
+    });
+    assert.equal(check.state, 'FAILED');
+    assert.equal(check.httpStatus, status);
+    assert.equal(check.failureReason, 'HTTP_STATUS');
+    assert.equal(failureDisposition(check), disposition);
+    outcomes.push({ status, disposition, calls: stub.calls.length });
+  }
+  assert.equal(failureDisposition({ ...timedOut, state: 'ABORTED', httpStatus: null, latencyMs: null, failureReason: null }), 'uncertain');
+  await assert.rejects(checkMonitor(monitor(scenario.monitor.url), undefined, undefined, {
+    resolve: async () => { throw new Error('Unexpected local resolver failure.'); }, connect: stubConnector().connect,
+  }), { message: 'Outbound execution stopped without an authoritative result.' });
+  await record('connect-timeout', { result: 'PASS', timedOut, durationMs, closedRequests: stalled.destroyed(), outcomes, automaticRetries: 0 });
+});
+
+test('E12 controlled resource sequence preserves final status, closes sockets and persists policy ABORTED through a real worker', { timeout: 30000 }, async () => {
+  const config = { ...databaseConfig(), schema: 'e12_outbound' };
+  const pool = databasePool(config);
+  const production = buildApp(undefined, config);
+  const trusted = buildApp(scenario.fixtureOrigin, config);
+  const fixture = outboundFixture();
+  let owned = false;
+  let worker: Awaited<ReturnType<typeof startTestWorker>> | undefined;
+  const results = [];
+  let workerEvidence: unknown;
+  try {
+    await requireFreePorts();
+    await pool.query(`CREATE SCHEMA ${schemaIdentifier(config.schema)}`); owned = true;
+    await migrate(config);
+    const cookie = await loginForTest(production, await insertOutboundUser(pool));
+    const inject = authenticatedInject(production, cookie);
+    const created = await inject({ method: 'POST', url: scenario.baseline.path, payload: scenario.monitor });
+    assert.equal(created.statusCode, scenario.baseline.requiredStatus);
+    assert.equal(created.json().data.url, scenario.monitor.url);
+    const denied = await inject({ method: 'POST', url: '/monitors', payload: { ...scenario.monitor, url: `${scenario.fixtureOrigin}/private` } });
+    assert.equal(denied.statusCode, 400);
+    assert.equal(denied.json().error.code, 'INVALID_INPUT');
+    for (const [server, port] of [[fixture.server, 4311], [fixture.guard, 4314]] as const) {
+      await new Promise<void>((resolve, reject) => { server.once('error', reject); server.listen(port, '127.0.0.1', resolve); });
+    }
+    for (const input of scenario.resourceSequence) {
+      const beforeCalls = fixture.calls.length;
+      let consumedBodyBytes = 0;
+      const began = performance.now();
+      let check: TerminalCheckRun;
+      if (input.path === '/private') {
+        const fixtureInject = authenticatedInject(trusted, cookie);
+        const local = await fixtureInject({ method: 'POST', url: '/monitors', payload: { ...scenario.monitor, url: `${scenario.fixtureOrigin}${input.path}` } });
+        assert.equal(local.statusCode, 201);
+        const path = `/monitors/${local.json().data.id}/checks`;
+        const headers = { 'idempotency-key': 'manual-outbound-e12-policy' };
+        const queued = await fixtureInject({ method: 'POST', url: path, headers });
+        assert.equal(queued.statusCode, 202);
+        assert.equal(queued.json().data.state, 'QUEUED');
+        worker = await startTestWorker(config);
+        assert.notEqual(worker.pid, process.pid);
+        check = await waitForTerminalCheck(pool, queued.json().data.id);
+        assert.equal(check.id, queued.json().data.id);
+        const replay = await fixtureInject({ method: 'POST', url: path, headers });
+        assert.equal(replay.statusCode, 202);
+        assert.deepEqual(replay.json().data, check);
+        assert.equal((await pool.query('SELECT count(*)::int AS count FROM check_runs')).rows[0].count, 1);
+        workerEvidence = { workerPid: worker.pid, observerPid: process.pid, queued: queued.json().data,
+          terminal: check, sameIdentityReplay: true, rows: 1 };
+        const stopped = await worker.stop(); worker = undefined;
+        assert.equal(stopped.code, 0); assert.equal(stopped.forced, false);
+      } else {
+        check = await checkMonitor(monitor(`${scenario.fixtureOrigin}${input.path}`), scenario.fixtureOrigin, undefined, {
+          connect: (destination, receive) => connectDestination(destination, response => {
+            response.on('data', (chunk: Buffer) => { consumedBodyBytes += chunk.length; });
+            receive(response);
+          }),
+        });
+      }
+      const durationMs = Math.round(performance.now() - began);
+      assert.equal(check.state, input.state);
+      assert.equal(check.httpStatus, input.httpStatus);
+      assert.equal(check.failureReason, input.reason);
+      assert.equal(fixture.calls.length - beforeCalls, input.requests);
+      assert.ok(durationMs <= scenario.completionCeilingMs);
+      assert.equal(consumedBodyBytes, 0);
+      assert.ok(consumedBodyBytes <= scenario.resources.bodyBytesMax);
+      await bounded(Promise.all([...fixture.sockets].map(socket => new Promise<void>(resolve => socket.once('close', () => resolve())))));
+      assert.equal(fixture.sockets.size, 0);
+      assert.equal(fixture.guardRequests(), 0);
+      results.push({ input, check, durationMs, paths: fixture.calls.slice(beforeCalls), consumedBodyBytes,
+        bodyObservation: input.path === '/private' ? 'worker cancels redirect before any body consumption; source boundary' : 'real IncomingMessage data observer',
+        openSocketsAfter: fixture.sockets.size });
+    }
+    await record('resources', { result: 'PASS', baselineNowStatus: created.statusCode, productionLocalStatus: denied.statusCode,
+      results, workerEvidence, guardRequests: fixture.guardRequests(), automaticRetries: 0 });
+  } finally {
+    if (worker) await worker.stop();
+    await production.close(); await trusted.close();
+    for (const server of [fixture.server, fixture.guard]) {
+      if (server.listening) { server.closeAllConnections(); await new Promise<void>(resolve => server.close(() => resolve())); }
+    }
+    if (owned) await pool.query(`DROP SCHEMA ${schemaIdentifier(config.schema)} CASCADE`);
+    await pool.end();
+    await requireFreePorts();
+  }
+});
diff --git a/test/ownership.test.ts b/test/ownership.test.ts
index 8858a93..b2e4d5c 100644
--- a/test/ownership.test.ts
+++ b/test/ownership.test.ts
@@ -25,7 +25,7 @@ test('E10 parallel manual identity persists one intent, conflicts by meaning and
   const config = { ...databaseConfig(), schema: scenario.schema };
   const pool = databasePool(config);
   const fixture = identityFixture();
-  let app = buildApp(undefined, config);
+  let app = buildApp(new URL(scenario.monitors[0].url).origin, config);
   const barrier = manualBarrier(app);
   let owned = false;
   let worker: Awaited<ReturnType<typeof startTestWorker>> | undefined;
@@ -65,7 +65,7 @@ test('E10 parallel manual identity persists one intent, conflicts by meaning and
     report.conflict = { status: 409, code: 'CONFLICT', bRows: 0, bOutbound: 0 };
 
     await app.close();
-    app = buildApp(undefined, config);
+    app = buildApp(new URL(scenario.monitors[0].url).origin, config);
     await app.listen({ host: '127.0.0.1', port: 4312 });
     const update = await request(`http://127.0.0.1:4312/monitors/${scenario.monitors[0].id}`, {
       method: 'PUT', headers: { 'content-type': 'application/json' },
diff --git a/test/persistence.test.ts b/test/persistence.test.ts
index d8d9e74..d5ddfcc 100644
--- a/test/persistence.test.ts
+++ b/test/persistence.test.ts
@@ -54,9 +54,9 @@ test('E03 migration CLI applies the fresh chain and a repeated command changes n
   try {
     const options = { cwd: process.cwd(), env: { ...process.env, DATABASE_URL: config.connectionString, DATABASE_SCHEMA: config.schema }, timeout: 10_000 };
     const first = await execute(process.execPath, ['server/migrate.ts'], options);
-    assert.deepEqual(JSON.parse(first.stdout).applied, [...authScenario.additionalAssertions.expectedMigrationFiles, '004_monitor_ownership.sql', '005_check_history_index.sql', '006_check_queue.sql', '007_check_ownership.sql', '008_check_lease.sql']);
+    assert.deepEqual(JSON.parse(first.stdout).applied, [...authScenario.additionalAssertions.expectedMigrationFiles, '004_monitor_ownership.sql', '005_check_history_index.sql', '006_check_queue.sql', '007_check_ownership.sql', '008_check_lease.sql', '009_outbound_policy_result.sql']);
     const before = (await pool.query('SELECT version, checksum, applied_at FROM schema_migrations ORDER BY version')).rows;
-    assert.equal(before.length, authScenario.additionalAssertions.expectedMigrationFiles.length + 5);
+    assert.equal(before.length, authScenario.additionalAssertions.expectedMigrationFiles.length + 6);
     for (const row of before) { assert.match(row.checksum, /^[0-9a-f]{64}$/); assert.ok(row.applied_at instanceof Date); }
     const second = await execute(process.execPath, ['server/migrate.ts'], options);
     assert.deepEqual(JSON.parse(second.stdout).applied, scenario.additionalAssertions.repeatMigrationExpectedApplied);
@@ -263,7 +263,7 @@ test('E05 upgrades existing rows only with explicit owner designation and preser
     finally { await oldApp.close(); }
 
     // Deliberately designate the second account, never the first row returned.
-    assert.deepEqual(await migrate(config, users[1].username), ['004_monitor_ownership.sql', '005_check_history_index.sql', '006_check_queue.sql', '007_check_ownership.sql', '008_check_lease.sql']);
+    assert.deepEqual(await migrate(config, users[1].username), ['004_monitor_ownership.sql', '005_check_history_index.sql', '006_check_queue.sql', '007_check_ownership.sql', '008_check_lease.sql', '009_outbound_policy_result.sql']);
     const owner = (await pool.query('SELECT id FROM users WHERE username = $1', [users[1].username])).rows[0].id;
     const after = (await pool.query('SELECT * FROM monitors ORDER BY id')).rows;
     assert.deepEqual(after.map(({ owner_user_id: _owner, ...row }) => row), before.monitors);
diff --git a/test/storage-contract.test.ts b/test/storage-contract.test.ts
index 7cd8a0c..fa275f5 100644
--- a/test/storage-contract.test.ts
+++ b/test/storage-contract.test.ts
@@ -12,7 +12,7 @@ test('E03 NUL name is INVALID_INPUT on create/update without PostgreSQL mutation
   const input = scenario.nulName;
   const config = await resetTestSchema(input.schema);
   const pool = databasePool(config);
-  const app = buildApp(undefined, config);
+  const app = buildApp(new URL(scenario.extraRequiredColumn.probeInput.url).origin, config);
   try {
     const inject = authenticatedInject(app, await loginForTest(app, (await prepareTestUsers(config))[0]));
     const created = await inject({ method: 'POST', url: '/monitors', payload: input.input });
diff --git a/test/unit.test.ts b/test/unit.test.ts
index fe521d9..3f095b2 100644
--- a/test/unit.test.ts
+++ b/test/unit.test.ts
@@ -1,7 +1,7 @@
 import test from 'node:test';
 import assert from 'node:assert/strict';
 import { randomBytes } from 'node:crypto';
-import { fixtureUrl, DEFAULT_FIXTURE_ORIGIN } from '../server/check.ts';
+import { canonicalUrl as fixtureUrl, DEFAULT_FIXTURE_ORIGIN } from '../server/outbound.ts';
 import { ERROR_MESSAGES, failureCode, RequestFailure, responseData } from '../app/monitors/api.ts';
 import type { ApiErrorCode } from '../server/model.ts';
 import scenario from '../evidence/E02/scenario.json' with { type: 'json' };
diff --git a/test/worker.ts b/test/worker.ts
index 45f7aba..699bce9 100644
--- a/test/worker.ts
+++ b/test/worker.ts
@@ -6,10 +6,12 @@ import type { DatabaseConfig } from '../server/database.ts';
 import type { CheckRunRow } from '../server/mapping.ts';
 import { checkRunFromRow } from '../server/mapping.ts';
 import type { TerminalCheckRun } from '../server/model.ts';
+import { DEFAULT_FIXTURE_ORIGIN } from '../server/outbound.ts';
 
 export async function startTestWorker(config: DatabaseConfig) {
   const child = spawn(process.execPath, ['server/worker.ts', '--no-schedule'], {
-    env: { ...process.env, DATABASE_URL: config.connectionString, DATABASE_SCHEMA: config.schema },
+    env: { ...process.env, DATABASE_URL: config.connectionString, DATABASE_SCHEMA: config.schema,
+      NODE_ENV: 'test', WSE_TEST_FIXTURE_ORIGIN: DEFAULT_FIXTURE_ORIGIN },
     stdio: ['ignore', 'pipe', 'pipe'],
   });
   const exited = new Promise<{ code: number | null; signal: string | null }>((resolve) => child.once('close', (code, signal) => resolve({ code, signal })));


