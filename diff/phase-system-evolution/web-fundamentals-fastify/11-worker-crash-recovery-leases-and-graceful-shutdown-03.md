## `feat: recover abandoned checks under finite leases`

diff --git a/app/monitors/monitor-workspace.tsx b/app/monitors/monitor-workspace.tsx
index 0dd66e9..6ed845e 100644
--- a/app/monitors/monitor-workspace.tsx
+++ b/app/monitors/monitor-workspace.tsx
@@ -134,9 +134,9 @@ export default function MonitorWorkspace({ initial }: { initial: InitialMonitors
         {monitor.latestCheck ? <dl aria-label="Latest check result">
           <dt>Check ID</dt><dd>{monitor.latestCheck.id}</dd>
           <dt>State</dt><dd><span role="status">{monitor.latestCheck.state}</span></dd>
-          <dt>HTTP status</dt><dd>{monitor.latestCheck.httpStatus ?? 'No response'}</dd>
-          <dt>Latency</dt><dd>{monitor.latestCheck.latencyMs === null ? 'Pending' : `${monitor.latestCheck.latencyMs} ms`}</dd>
-          <dt>Failure reason</dt><dd>{monitor.latestCheck.failureReason ?? 'None'}</dd>
+          <dt>HTTP status</dt><dd>{monitor.latestCheck.state === 'ABORTED' ? 'Unknown' : monitor.latestCheck.httpStatus ?? 'No response'}</dd>
+          <dt>Latency</dt><dd>{monitor.latestCheck.state === 'ABORTED' ? 'Unknown' : monitor.latestCheck.latencyMs === null ? 'Pending' : `${monitor.latestCheck.latencyMs} ms`}</dd>
+          <dt>Failure reason</dt><dd>{monitor.latestCheck.state === 'ABORTED' ? 'Unknown' : monitor.latestCheck.failureReason ?? 'None'}</dd>
           <dt>Finished</dt><dd>{monitor.latestCheck.finishedAt === null ? 'Pending' : <time dateTime={monitor.latestCheck.finishedAt}>{monitor.latestCheck.finishedAt}</time>}</dd>
         </dl> : <p>No checks yet.</p>}
         {historyId === monitor.id && <section aria-label={`Check history for ${monitor.name}`}>
@@ -156,8 +156,8 @@ export default function MonitorWorkspace({ initial }: { initial: InitialMonitors
               <caption>Terminal check results for {monitor.name}</caption>
               <thead><tr><th scope="col">Check ID</th><th scope="col">State</th><th scope="col">HTTP status</th><th scope="col">Latency</th><th scope="col">Failure reason</th><th scope="col">Finished</th></tr></thead>
               <tbody>{history.map((check) => <tr key={check.id}>
-                <td>{check.id}</td><td>{check.state}</td><td>{check.httpStatus ?? 'No response'}</td>
-                <td>{check.latencyMs} ms</td><td>{check.failureReason ?? 'None'}</td>
+                <td>{check.id}</td><td>{check.state}</td><td>{check.state === 'ABORTED' ? 'Unknown' : check.httpStatus ?? 'No response'}</td>
+                <td>{check.latencyMs === null ? 'Unknown' : `${check.latencyMs} ms`}</td><td>{check.state === 'ABORTED' ? 'Unknown' : check.failureReason ?? 'None'}</td>
                 <td><time dateTime={check.finishedAt}>{check.finishedAt}</time></td>
               </tr>)}</tbody>
             </table></div>}
diff --git a/server/check.ts b/server/check.ts
index e920ee9..330024a 100644
--- a/server/check.ts
+++ b/server/check.ts
@@ -1,7 +1,7 @@
 import { randomUUID } from 'node:crypto';
 import { request as httpRequest } from 'node:http';
 import { request as httpsRequest } from 'node:https';
-import type { CheckRun, Monitor, TerminalCheckRun } from './model.ts';
+import type { CheckRun, Monitor, ObservedCheckRun } from './model.ts';
 
 export const DEFAULT_FIXTURE_ORIGIN = 'http://127.0.0.1:4311';
 export const CHECK_TIMEOUT_MS = 2_000;
@@ -20,7 +20,7 @@ export function fixtureUrl(value: string, fixtureOrigin: string): URL {
 }
 
 export async function checkMonitor(monitor: Monitor, fixtureOrigin: string,
-  execution?: Pick<TerminalCheckRun, 'id' | 'trigger' | 'startedAt'>): Promise<TerminalCheckRun> {
+  execution?: Pick<ObservedCheckRun, 'id' | 'trigger' | 'startedAt'>): Promise<ObservedCheckRun> {
   // Recheck at the outbound boundary, even though creation also checks the URL.
   const url = fixtureUrl(monitor.url, fixtureOrigin);
   const startedAt = execution?.startedAt ?? new Date().toISOString();
diff --git a/server/migrate.ts b/server/migrate.ts
index 567f5bf..8dc816e 100644
--- a/server/migrate.ts
+++ b/server/migrate.ts
@@ -5,7 +5,7 @@ import { databaseConfig, databasePool, schemaIdentifier } from './database.ts';
 import type { DatabaseConfig } from './database.ts';
 
 export async function migrationFiles() {
-  const names = ['001_monitors.sql', '002_check_runs.sql', '003_sessions.sql', '004_monitor_ownership.sql', '005_check_history_index.sql', '006_check_queue.sql', '007_check_ownership.sql'];
+  const names = ['001_monitors.sql', '002_check_runs.sql', '003_sessions.sql', '004_monitor_ownership.sql', '005_check_history_index.sql', '006_check_queue.sql', '007_check_ownership.sql', '008_check_lease.sql'];
   return Promise.all(names.map(async (version) => {
     const sql = await readFile(new URL(`./migrations/${version}`, import.meta.url), 'utf8');
     return { version, sql, checksum: createHash('sha256').update(sql).digest('hex') };
diff --git a/server/migrations/008_check_lease.sql b/server/migrations/008_check_lease.sql
new file mode 100644
index 0000000..49f35c6
--- /dev/null
+++ b/server/migrations/008_check_lease.sql
@@ -0,0 +1,30 @@
+ALTER TABLE check_runs
+  ADD COLUMN lease_expires_at timestamptz(3),
+  DROP CONSTRAINT check_runs_state_check,
+  DROP CONSTRAINT check_runs_lifecycle_check;
+
+-- A pre-upgrade RUNNING row also gets a finite expiry, without reopening it.
+UPDATE check_runs SET lease_expires_at = started_at + interval '5 seconds'
+  WHERE state = 'RUNNING';
+
+ALTER TABLE check_runs
+  ADD CONSTRAINT check_runs_state_check CHECK (state IN ('QUEUED', 'RUNNING', 'SUCCEEDED', 'FAILED', 'ABORTED')),
+  ADD CONSTRAINT check_runs_lease_check CHECK (
+    (state = 'QUEUED' AND lease_expires_at IS NULL)
+    OR (state = 'RUNNING' AND lease_expires_at IS NOT NULL AND lease_expires_at > started_at)
+    OR state IN ('SUCCEEDED', 'FAILED', 'ABORTED')
+  ),
+  ADD CONSTRAINT check_runs_lifecycle_check CHECK (
+    (state = 'QUEUED' AND started_at IS NULL AND finished_at IS NULL
+      AND http_status IS NULL AND latency_ms IS NULL AND failure_reason IS NULL)
+    OR (state = 'RUNNING' AND started_at IS NOT NULL AND finished_at IS NULL
+      AND http_status IS NULL AND latency_ms IS NULL AND failure_reason IS NULL)
+    OR (state = 'ABORTED' AND started_at IS NOT NULL AND finished_at IS NOT NULL
+      AND http_status IS NULL AND latency_ms IS NULL AND failure_reason IS NULL)
+    OR (state IN ('SUCCEEDED', 'FAILED') AND started_at IS NOT NULL AND finished_at IS NOT NULL
+      AND latency_ms IS NOT NULL AND (
+        (state = 'SUCCEEDED' AND http_status IS NOT NULL AND http_status BETWEEN 200 AND 299 AND failure_reason IS NULL)
+        OR (state = 'FAILED' AND failure_reason = 'HTTP_STATUS' AND http_status IS NOT NULL AND (http_status < 200 OR http_status >= 300))
+        OR (state = 'FAILED' AND failure_reason IN ('TIMEOUT', 'CONNECTION_FAILURE') AND http_status IS NULL)
+      ))
+  );
diff --git a/server/model.ts b/server/model.ts
index 9d0b208..b41c3c0 100644
--- a/server/model.ts
+++ b/server/model.ts
@@ -12,7 +12,7 @@ export type CheckRun = {
   id: string;
   monitorId: string;
   trigger: 'MANUAL' | 'SCHEDULED';
-  state: 'QUEUED' | 'RUNNING' | 'SUCCEEDED' | 'FAILED';
+  state: 'QUEUED' | 'RUNNING' | 'SUCCEEDED' | 'FAILED' | 'ABORTED';
   httpStatus: number | null;
   latencyMs: number | null;
   failureReason: 'HTTP_STATUS' | 'TIMEOUT' | 'CONNECTION_FAILURE' | null;
@@ -20,17 +20,21 @@ export type CheckRun = {
   finishedAt: string | null;
 };
 
-export type TerminalCheckRun = CheckRun & {
+export type ObservedCheckRun = CheckRun & {
   state: 'SUCCEEDED' | 'FAILED'; latencyMs: number; startedAt: string; finishedAt: string;
 };
 
+export type TerminalCheckRun = ObservedCheckRun | (CheckRun & {
+  state: 'ABORTED'; httpStatus: null; latencyMs: null; failureReason: null; startedAt: string; finishedAt: string;
+});
+
 export type MonitorView = Monitor & { latestCheck: CheckRun | null };
 
 export type CheckHistoryPage = {
   items: TerminalCheckRun[];
   nextCursor: string | null;
   limit: number;
-  state: TerminalCheckRun['state'] | null;
+  state: ObservedCheckRun['state'] | null;
 };
 
 export type User = { id: string; username: string };
diff --git a/server/schema.ts b/server/schema.ts
index e7f94a4..358e866 100644
--- a/server/schema.ts
+++ b/server/schema.ts
@@ -13,6 +13,7 @@ const expectedColumns = {
     started_at: ['timestamptz', 'YES'], finished_at: ['timestamptz', 'YES'],
     queued_at: ['timestamptz', 'NO'], scheduled_for: ['timestamptz', 'YES'],
     request_user_id: ['uuid', 'YES'], idempotency_key: ['text', 'YES'], worker_id: ['uuid', 'YES'],
+    lease_expires_at: ['timestamptz', 'YES'],
   },
   users: {
     id: ['uuid', 'NO'], username: ['text', 'NO'], password_hash: ['text', 'NO'],
diff --git a/server/worker.ts b/server/worker.ts
index 6bf6277..4b75854 100644
--- a/server/worker.ts
+++ b/server/worker.ts
@@ -5,10 +5,13 @@ import { setTimeout as delay } from 'node:timers/promises';
 import { checkMonitor, DEFAULT_FIXTURE_ORIGIN } from './check.ts';
 import { databaseConfig, databasePool } from './database.ts';
 import { checkRunFromRow, monitorFromRow } from './mapping.ts';
-import type { TerminalCheckRun } from './model.ts';
+import type { ObservedCheckRun } from './model.ts';
 import type { CheckRunRow, MonitorRow } from './mapping.ts';
 import { verifySchema } from './schema.ts';
 
+export const CHECK_LEASE_MS = 5000;
+export const SHUTDOWN_GRACE_MS = 3000;
+
 export async function scheduleDueChecks(pool: Pool, now = new Date()) {
   // The current interval slot is anchored to creation. Repeated ticks address
   // the same durable slot; manual intents have no slot and may overlap it.
@@ -21,25 +24,53 @@ export async function scheduleDueChecks(pool: Pool, now = new Date()) {
   return result.rows.map(checkRunFromRow);
 }
 
-export async function completeCheck(pool: Pool, workerId: string, observed: TerminalCheckRun) {
-  const saved = await pool.query<CheckRunRow>(`UPDATE check_runs SET state = $2, http_status = $3,
-    latency_ms = $4, failure_reason = $5, finished_at = $6
-    WHERE id = $1 AND state = 'RUNNING' AND worker_id = $7 RETURNING *`,
-  [observed.id, observed.state, observed.httpStatus, observed.latencyMs, observed.failureReason, new Date(observed.finishedAt), workerId]);
-  return saved.rows[0] ? checkRunFromRow(saved.rows[0]) : null;
+export async function recoverExpiredChecks(pool: Pool, now?: Date) {
+  // Production uses the database clock. The explicit clock is for fixed expiry tests.
+  const recovered = await pool.query<CheckRunRow>(`UPDATE check_runs SET state = 'ABORTED',
+    http_status = NULL, latency_ms = NULL, failure_reason = NULL,
+    finished_at = COALESCE($1::timestamptz, clock_timestamp())
+    WHERE state = 'RUNNING' AND lease_expires_at <= COALESCE($1::timestamptz, clock_timestamp())
+    RETURNING *`, [now ?? null]);
+  return recovered.rows.map(checkRunFromRow);
+}
+
+export async function completeCheck(pool: Pool, workerId: string, observed: ObservedCheckRun) {
+  const client = await pool.connect();
+  try {
+    // Never leave an autocommit terminal write running after its worker dies.
+    await client.query('BEGIN');
+    const saved = await client.query<CheckRunRow>(`UPDATE check_runs SET state = $2, http_status = $3,
+      latency_ms = $4, failure_reason = $5, finished_at = $6
+      WHERE id = $1 AND state = 'RUNNING' AND worker_id = $7
+        AND lease_expires_at > clock_timestamp() RETURNING *`,
+    [observed.id, observed.state, observed.httpStatus, observed.latencyMs, observed.failureReason, new Date(observed.finishedAt), workerId]);
+    // The UPDATE may have waited on a row lock after evaluating its predicate.
+    const lease = saved.rows[0] && await client.query<{ valid: boolean }>(
+      'SELECT lease_expires_at > clock_timestamp() AS valid FROM check_runs WHERE id = $1', [observed.id]);
+    if (!lease?.rows[0]?.valid) { await client.query('ROLLBACK'); return null; }
+    await client.query('COMMIT');
+    return checkRunFromRow(saved.rows[0]);
+  } catch (error) { await client.query('ROLLBACK'); throw error; }
+  finally { client.release(); }
 }
 
-export async function runNextCheck(pool: Pool, fixtureOrigin = DEFAULT_FIXTURE_ORIGIN, workerId = randomUUID()) {
+export async function runNextCheck(pool: Pool, fixtureOrigin = DEFAULT_FIXTURE_ORIGIN, workerId = randomUUID(), signal?: AbortSignal) {
+  if (signal?.aborted) return null;
   const client = await pool.connect();
   let claimed: CheckRunRow | undefined;
   try {
     await client.query('BEGIN');
-    const running = await client.query<CheckRunRow>(`UPDATE check_runs
-      SET state = 'RUNNING', started_at = clock_timestamp(), worker_id = $1
+    if (signal?.aborted) { await client.query('ROLLBACK'); return null; }
+    const running = await client.query<CheckRunRow>(`WITH claim_clock AS (SELECT clock_timestamp() AS at)
+      UPDATE check_runs
+      SET state = 'RUNNING', started_at = claim_clock.at, worker_id = $1,
+        lease_expires_at = claim_clock.at + $2 * interval '1 millisecond'
+      FROM claim_clock
       WHERE id = (SELECT id FROM check_runs WHERE state = 'QUEUED'
         ORDER BY queued_at, id LIMIT 1 FOR UPDATE SKIP LOCKED)
-        AND state = 'QUEUED' RETURNING *`, [workerId]);
+        AND state = 'QUEUED' RETURNING check_runs.*`, [workerId, CHECK_LEASE_MS]);
     claimed = running.rows[0];
+    if (signal?.aborted) { await client.query('ROLLBACK'); return null; }
     await client.query('COMMIT');
   } catch (error) { await client.query('ROLLBACK'); throw error; }
   finally { client.release(); }
@@ -57,17 +88,38 @@ export async function runNextCheck(pool: Pool, fixtureOrigin = DEFAULT_FIXTURE_O
 if (process.argv[1] && import.meta.url === pathToFileURL(process.argv[1]).href) {
   const pool = databasePool(databaseConfig());
   const workerId = randomUUID();
+  const stopping = new AbortController();
+  let shutdownDeadline: ReturnType<typeof setTimeout> | undefined;
+  function stopClaims() {
+    if (stopping.signal.aborted) return;
+    stopping.abort();
+    console.log('Check worker stopping.');
+    shutdownDeadline = setTimeout(() => {
+      // A stuck operation is uncertain, not an endpoint failure. Its lease expires.
+      console.error('Check worker shutdown deadline reached.');
+      process.exit(1);
+    }, SHUTDOWN_GRACE_MS);
+  }
+  process.on('SIGTERM', stopClaims);
   try {
     await verifySchema(pool);
     console.log(`Check worker ready. ${workerId}`);
-    for (;;) {
+    while (!stopping.signal.aborted) {
       // Regression fixtures control scheduler time explicitly; normal operation
       // always schedules. There is no separate scheduler daemon or queue store.
       if (!process.argv.includes('--no-schedule')) await scheduleDueChecks(pool);
-      if (!await runNextCheck(pool, process.env.FIXTURE_ORIGIN ?? DEFAULT_FIXTURE_ORIGIN, workerId)) await delay(250);
+      if (stopping.signal.aborted) break;
+      const completed = await runNextCheck(pool, process.env.FIXTURE_ORIGIN ?? DEFAULT_FIXTURE_ORIGIN, workerId, stopping.signal);
+      if (stopping.signal.aborted) break;
+      await recoverExpiredChecks(pool);
+      if (!completed && !stopping.signal.aborted) await delay(250);
     }
   } catch {
     console.error('Check worker stopped after an execution or database failure.');
     process.exitCode = 1;
-  } finally { await pool.end(); }
+  } finally {
+    await pool.end();
+    clearTimeout(shutdownDeadline);
+    process.off('SIGTERM', stopClaims);
+  }
 }
diff --git a/test/contracts.test.ts b/test/contracts.test.ts
index ba43ef7..97e2c90 100644
--- a/test/contracts.test.ts
+++ b/test/contracts.test.ts
@@ -173,7 +173,7 @@ test('E02 terminal CheckRun wire values distinguish observed HTTP failure from n
     assert.equal(result.state, expected.state);
     assert.equal(result.httpStatus, expected.httpStatus);
     assert.equal(result.failureReason, expected.failureReason);
-    assert.ok(Number.isInteger(result.latencyMs) && result.latencyMs >= 0);
+    assert.ok(result.latencyMs !== null && Number.isInteger(result.latencyMs) && result.latencyMs >= 0);
     assertTimestamp(result.startedAt);
     assertTimestamp(result.finishedAt);
     assert.ok(Date.parse(result.finishedAt) >= Date.parse(result.startedAt));
diff --git a/test/persistence.test.ts b/test/persistence.test.ts
index ce73c2f..d8d9e74 100644
--- a/test/persistence.test.ts
+++ b/test/persistence.test.ts
@@ -54,9 +54,9 @@ test('E03 migration CLI applies the fresh chain and a repeated command changes n
   try {
     const options = { cwd: process.cwd(), env: { ...process.env, DATABASE_URL: config.connectionString, DATABASE_SCHEMA: config.schema }, timeout: 10_000 };
     const first = await execute(process.execPath, ['server/migrate.ts'], options);
-    assert.deepEqual(JSON.parse(first.stdout).applied, [...authScenario.additionalAssertions.expectedMigrationFiles, '004_monitor_ownership.sql', '005_check_history_index.sql', '006_check_queue.sql', '007_check_ownership.sql']);
+    assert.deepEqual(JSON.parse(first.stdout).applied, [...authScenario.additionalAssertions.expectedMigrationFiles, '004_monitor_ownership.sql', '005_check_history_index.sql', '006_check_queue.sql', '007_check_ownership.sql', '008_check_lease.sql']);
     const before = (await pool.query('SELECT version, checksum, applied_at FROM schema_migrations ORDER BY version')).rows;
-    assert.equal(before.length, authScenario.additionalAssertions.expectedMigrationFiles.length + 4);
+    assert.equal(before.length, authScenario.additionalAssertions.expectedMigrationFiles.length + 5);
     for (const row of before) { assert.match(row.checksum, /^[0-9a-f]{64}$/); assert.ok(row.applied_at instanceof Date); }
     const second = await execute(process.execPath, ['server/migrate.ts'], options);
     assert.deepEqual(JSON.parse(second.stdout).applied, scenario.additionalAssertions.repeatMigrationExpectedApplied);
@@ -263,15 +263,15 @@ test('E05 upgrades existing rows only with explicit owner designation and preser
     finally { await oldApp.close(); }
 
     // Deliberately designate the second account, never the first row returned.
-    assert.deepEqual(await migrate(config, users[1].username), ['004_monitor_ownership.sql', '005_check_history_index.sql', '006_check_queue.sql', '007_check_ownership.sql']);
+    assert.deepEqual(await migrate(config, users[1].username), ['004_monitor_ownership.sql', '005_check_history_index.sql', '006_check_queue.sql', '007_check_ownership.sql', '008_check_lease.sql']);
     const owner = (await pool.query('SELECT id FROM users WHERE username = $1', [users[1].username])).rows[0].id;
     const after = (await pool.query('SELECT * FROM monitors ORDER BY id')).rows;
     assert.deepEqual(after.map(({ owner_user_id: _owner, ...row }) => row), before.monitors);
     assert.ok(after.every((row) => row.owner_user_id === owner));
     const upgradedChecks = (await pool.query('SELECT * FROM check_runs ORDER BY id')).rows;
-    assert.deepEqual(upgradedChecks.map(({ queued_at: _queued, scheduled_for: _slot, request_user_id: _requestUser, idempotency_key: _key, worker_id: _worker, ...row }) => row), before.checks);
+    assert.deepEqual(upgradedChecks.map(({ queued_at: _queued, scheduled_for: _slot, request_user_id: _requestUser, idempotency_key: _key, worker_id: _worker, lease_expires_at: _lease, ...row }) => row), before.checks);
     assert.ok(upgradedChecks.every((row) => row.queued_at.getTime() === row.started_at.getTime() && row.scheduled_for === null));
-    assert.ok(upgradedChecks.every((row) => row.request_user_id === null && row.idempotency_key === null && row.worker_id === null));
+    assert.ok(upgradedChecks.every((row) => row.request_user_id === null && row.idempotency_key === null && row.worker_id === null && row.lease_expires_at === null));
     assert.deepEqual((await pool.query('SELECT * FROM schema_migrations ORDER BY version LIMIT 3')).rows, before.migrations);
     await assert.rejects(pool.query('DELETE FROM users WHERE id = $1', [owner]), { code: '23503' });
     const app = buildApp(undefined, config);


