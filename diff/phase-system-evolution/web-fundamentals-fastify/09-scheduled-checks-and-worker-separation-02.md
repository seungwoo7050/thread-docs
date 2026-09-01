## `feat: persist check intents and execute them in a worker`

diff --git a/app/monitors/monitor-workspace.tsx b/app/monitors/monitor-workspace.tsx
index e8bc9f6..0dd66e9 100644
--- a/app/monitors/monitor-workspace.tsx
+++ b/app/monitors/monitor-workspace.tsx
@@ -86,7 +86,7 @@ export default function MonitorWorkspace({ initial }: { initial: InitialMonitors
         <label htmlFor="interval">Interval (seconds)</label>
         <input id="interval" name="interval" type="number" required min="1" max="86400" step="1" defaultValue="60" disabled={creating || loggingOut} />
         <label className="checkbox"><input name="enabled" type="checkbox" defaultChecked disabled={creating || loggingOut} /> Enabled</label>
-        <p className="hint">Interval and enabled are saved; this version only runs manual checks.</p>
+        <p className="hint">Enabled monitors are scheduled at this interval. Manual checks remain available while paused.</p>
         <button type="submit" disabled={creating || loggingOut}>{creating ? 'Creating…' : 'Create monitor'}</button>
       </form>
       {mutations.create?.status === 'success' && <p role="status">Monitor created.</p>}
@@ -105,7 +105,7 @@ export default function MonitorWorkspace({ initial }: { initial: InitialMonitors
         <p>{monitor.enabled ? 'Enabled' : 'Paused'} · {monitor.interval} seconds</p>
         <div className="actions">
         <button onClick={() => mutate({ kind: 'check', id: monitor.id })} disabled={pending || loggingOut}>
-          {pending && mutation.kind === 'check' ? 'Checking…' : 'Run check'}
+          {pending && mutation.kind === 'check' ? 'Queueing…' : 'Run check'}
         </button>
         <button onClick={() => setEditing(editing === monitor.id ? null : monitor.id)} disabled={pending || loggingOut} aria-expanded={editing === monitor.id}>
           {editing === monitor.id ? 'Cancel edit' : 'Edit monitor'}
@@ -120,7 +120,7 @@ export default function MonitorWorkspace({ initial }: { initial: InitialMonitors
           {historyId === monitor.id ? 'Hide history' : 'View history'}
         </button>
         </div>
-        {mutation?.status === 'success' && <p role="status">{mutation.kind === 'check' ? 'Check completed.' : 'Monitor saved.'}</p>}
+        {mutation?.status === 'success' && <p role="status">{mutation.kind === 'check' ? 'Check queued.' : 'Monitor saved.'}</p>}
         {editing === monitor.id && <form onSubmit={(event) => editMonitor(event, monitor)} className="edit-form">
           <label htmlFor={`edit-name-${monitor.id}`}>Edit name</label>
           <input id={`edit-name-${monitor.id}`} name="name" required defaultValue={monitor.name} disabled={pending || loggingOut} />
@@ -130,13 +130,14 @@ export default function MonitorWorkspace({ initial }: { initial: InitialMonitors
           <input id={`edit-interval-${monitor.id}`} name="interval" type="number" min="1" max="86400" step="1" required defaultValue={monitor.interval} disabled={pending || loggingOut} />
           <button type="submit" disabled={pending || loggingOut}>{pending && mutation.kind === 'update' ? 'Saving…' : 'Save changes'}</button>
         </form>}
-        {!monitor.enabled && <p className="hint">Paused. Manual checks remain available; no scheduler runs in this version.</p>}
+        {!monitor.enabled && <p className="hint">Paused. New scheduled checks are disabled; manual checks remain available.</p>}
         {monitor.latestCheck ? <dl aria-label="Latest check result">
-          <dt>State</dt><dd>{monitor.latestCheck.state}</dd>
+          <dt>Check ID</dt><dd>{monitor.latestCheck.id}</dd>
+          <dt>State</dt><dd><span role="status">{monitor.latestCheck.state}</span></dd>
           <dt>HTTP status</dt><dd>{monitor.latestCheck.httpStatus ?? 'No response'}</dd>
-          <dt>Latency</dt><dd>{monitor.latestCheck.latencyMs} ms</dd>
+          <dt>Latency</dt><dd>{monitor.latestCheck.latencyMs === null ? 'Pending' : `${monitor.latestCheck.latencyMs} ms`}</dd>
           <dt>Failure reason</dt><dd>{monitor.latestCheck.failureReason ?? 'None'}</dd>
-          <dt>Finished</dt><dd><time dateTime={monitor.latestCheck.finishedAt}>{monitor.latestCheck.finishedAt}</time></dd>
+          <dt>Finished</dt><dd>{monitor.latestCheck.finishedAt === null ? 'Pending' : <time dateTime={monitor.latestCheck.finishedAt}>{monitor.latestCheck.finishedAt}</time>}</dd>
         </dl> : <p>No checks yet.</p>}
         {historyId === monitor.id && <section aria-label={`Check history for ${monitor.name}`}>
           <h4>Check history</h4>
diff --git a/app/monitors/use-monitors.ts b/app/monitors/use-monitors.ts
index 49469cc..985dedd 100644
--- a/app/monitors/use-monitors.ts
+++ b/app/monitors/use-monitors.ts
@@ -78,6 +78,41 @@ export function useMonitors(initial: InitialMonitors) {
     if (historySearches.current.get(id) !== search) await loadHistory(id, search);
   }, [loadHistory]);
 
+  const activeChecks = state.monitors.filter((monitor) => monitor.latestCheck?.state === 'QUEUED' || monitor.latestCheck?.state === 'RUNNING')
+    .map((monitor) => monitor.latestCheck!.id).join(',');
+  useEffect(() => {
+    if (!activeChecks) return;
+    const generation = lifetime.current;
+    let stopped = false;
+    let timer: ReturnType<typeof setTimeout>;
+    const current = () => !stopped && generation === lifetime.current;
+    async function poll() {
+      try {
+        const checks = await Promise.all(activeChecks.split(',').map(async (id) => {
+          return responseData<CheckRun>(await fetch(`/api/checks/${id}`, { credentials: 'same-origin' }));
+        }));
+        if (!current()) return;
+        setState((state) => ({ ...state, monitors: state.monitors.map((monitor) => {
+          const check = checks.find((item) => item.id === monitor.latestCheck?.id);
+          return check ? { ...monitor, latestCheck: check } : monitor;
+        }) }));
+        for (const check of checks) {
+          if (check.state !== 'QUEUED' && check.state !== 'RUNNING') {
+            const search = historySearches.current.get(check.monitorId);
+            if (search !== undefined) void loadHistory(check.monitorId, search, false);
+          }
+        }
+        if (checks.some((check) => check.state === 'QUEUED' || check.state === 'RUNNING')) timer = setTimeout(poll, 250);
+      } catch (failure) {
+        // A failed read stops this poll. No retry policy or fabricated endpoint
+        // result; reload can read the durable execution again.
+        if (current()) handleFailure(failure);
+      }
+    }
+    timer = setTimeout(poll, 250);
+    return () => { stopped = true; clearTimeout(timer); };
+  }, [activeChecks, handleFailure, loadHistory]);
+
   async function mutate(action: Mutation): Promise<boolean> {
     const key = 'id' in action ? `monitor:${action.id}` : action.kind;
     // Admission is synchronous, before CSRF or POST. Rendered disabled buttons
diff --git a/server/app.ts b/server/app.ts
index 42aceab..2aaa12f 100644
--- a/server/app.ts
+++ b/server/app.ts
@@ -1,13 +1,13 @@
 import { randomUUID } from 'node:crypto';
 import Fastify, { errorCodes } from 'fastify';
 import type { FastifyReply, FastifyRequest } from 'fastify';
-import { checkMonitor, DEFAULT_FIXTURE_ORIGIN } from './check.ts';
+import { DEFAULT_FIXTURE_ORIGIN } from './check.ts';
 import { ApiError, ERROR_STATUS, monitorId, monitorInput } from './contracts.ts';
-import type { ApiFailure, Monitor } from './model.ts';
+import type { ApiFailure, Monitor, TerminalCheckRun } from './model.ts';
 import { databaseConfig, databasePool } from './database.ts';
 import type { DatabaseConfig } from './database.ts';
 import { verifySchema } from './schema.ts';
-import { checkRunFromRow, checkRunToValues, monitorFromRow, monitorToValues, monitorViewFromRow } from './mapping.ts';
+import { checkRunFromRow, monitorFromRow, monitorToValues, monitorViewFromRow } from './mapping.ts';
 import type { CheckRunRow, MonitorRow, MonitorViewRow } from './mapping.ts';
 import { registerAuthentication } from './auth.ts';
 import { historyCursor, historyQuery } from './history.ts';
@@ -25,7 +25,8 @@ const monitorViewSql = `SELECT m.id, m.owner_user_id, m.name, m.url, m.interval_
   c.id AS check_id, c.monitor_id, c.trigger, c.state, c.http_status, c.latency_ms, c.failure_reason, c.started_at, c.finished_at
   FROM monitors m LEFT JOIN LATERAL (
     SELECT id, monitor_id, trigger, state, http_status, latency_ms, failure_reason, started_at, finished_at
-    FROM check_runs WHERE monitor_id = m.id ORDER BY finished_at DESC, id DESC LIMIT 1
+    FROM check_runs WHERE monitor_id = m.id
+      ORDER BY (finished_at IS NULL) DESC, COALESCE(finished_at, queued_at) DESC, id DESC LIMIT 1
   ) c ON true`;
 
 export function buildApp(fixtureOrigin = DEFAULT_FIXTURE_ORIGIN, database: DatabaseConfig = databaseConfig(), now: () => number = Date.now) {
@@ -87,7 +88,8 @@ export function buildApp(fixtureOrigin = DEFAULT_FIXTURE_ORIGIN, database: Datab
     [id, input.name, input.url, input.interval, input.enabled, new Date(), request.user!.id]);
     if (!result.rows[0]) throw new ApiError('NOT_FOUND', 'Monitor not found.');
     const latest = await pool.query<CheckRunRow>(`SELECT c.* FROM check_runs c JOIN monitors m ON m.id = c.monitor_id
-      WHERE m.id = $1 AND m.owner_user_id = $2 ORDER BY c.finished_at DESC, c.id DESC LIMIT 1`, [id, request.user!.id]);
+      WHERE m.id = $1 AND m.owner_user_id = $2
+      ORDER BY (c.finished_at IS NULL) DESC, COALESCE(c.finished_at, c.queued_at) DESC, c.id DESC LIMIT 1`, [id, request.user!.id]);
     return { data: { ...monitorFromRow(result.rows[0]), latestCheck: latest.rows[0] ? checkRunFromRow(latest.rows[0]) : null } };
   });
 
@@ -98,16 +100,13 @@ export function buildApp(fixtureOrigin = DEFAULT_FIXTURE_ORIGIN, database: Datab
     return { data: { id: result.rows[0].id } };
   });
 
-  app.post<{ Params: { id: string } }>('/monitors/:id/checks', async (request) => {
-    const stored = await pool.query<MonitorRow>('SELECT * FROM monitors WHERE id = $1 AND owner_user_id = $2', [monitorId(request.params.id), request.user!.id]);
-    if (!stored.rows[0]) throw new ApiError('NOT_FOUND', 'Monitor not found.');
-    const result = await checkMonitor(monitorFromRow(stored.rows[0]), fixtureOrigin);
+  app.post<{ Params: { id: string } }>('/monitors/:id/checks', async (request, reply) => {
     const saved = await pool.query<CheckRunRow>(`INSERT INTO check_runs
-      (id, monitor_id, trigger, state, http_status, latency_ms, failure_reason, started_at, finished_at)
-      SELECT $1, $2, $3, $4, $5, $6, $7, $8, $9 FROM monitors
-      WHERE id = $2 AND owner_user_id = $10 RETURNING *`, [...checkRunToValues(result), request.user!.id]);
+      (id, monitor_id, trigger, state)
+      SELECT $1, id, 'MANUAL', 'QUEUED' FROM monitors
+      WHERE id = $2 AND owner_user_id = $3 RETURNING *`, [randomUUID(), monitorId(request.params.id), request.user!.id]);
     if (!saved.rows[0]) throw new ApiError('NOT_FOUND', 'Monitor not found.');
-    return { data: checkRunFromRow(saved.rows[0]) };
+    return reply.code(202).send({ data: checkRunFromRow(saved.rows[0]) });
   });
 
   app.get<{ Params: { id: string } }>('/monitors/:id/checks', async (request) => {
@@ -118,12 +117,12 @@ export function buildApp(fixtureOrigin = DEFAULT_FIXTURE_ORIGIN, database: Datab
     // Bind every external value, including the optional filter, seek tuple and
     // limit. One look-ahead row determines continuation without a COUNT/OFFSET.
     const history = await pool.query<CheckRunRow>(`SELECT c.* FROM check_runs c JOIN monitors m ON m.id = c.monitor_id
-      WHERE m.id = $1 AND m.owner_user_id = $2
+      WHERE m.id = $1 AND m.owner_user_id = $2 AND c.finished_at IS NOT NULL
         AND ($3::text IS NULL OR c.state = $3)
         AND ($4::timestamptz IS NULL OR (c.finished_at, c.id) < ($4::timestamptz, $5::uuid))
       ORDER BY c.finished_at DESC, c.id DESC LIMIT $6`,
     [id, request.user!.id, query.state, query.after?.finishedAt ?? null, query.after?.id ?? null, query.limit + 1]);
-    const items = history.rows.slice(0, query.limit).map(checkRunFromRow);
+    const items = history.rows.slice(0, query.limit).map((row) => checkRunFromRow(row) as TerminalCheckRun);
     const nextCursor = history.rows.length > query.limit ? historyCursor(query, items[items.length - 1]) : null;
     return { data: { items, nextCursor, limit: query.limit, state: query.state } };
   });
diff --git a/server/check.ts b/server/check.ts
index e027c07..e920ee9 100644
--- a/server/check.ts
+++ b/server/check.ts
@@ -1,7 +1,7 @@
 import { randomUUID } from 'node:crypto';
 import { request as httpRequest } from 'node:http';
 import { request as httpsRequest } from 'node:https';
-import type { CheckRun, Monitor } from './model.ts';
+import type { CheckRun, Monitor, TerminalCheckRun } from './model.ts';
 
 export const DEFAULT_FIXTURE_ORIGIN = 'http://127.0.0.1:4311';
 export const CHECK_TIMEOUT_MS = 2_000;
@@ -19,10 +19,11 @@ export function fixtureUrl(value: string, fixtureOrigin: string): URL {
   return url;
 }
 
-export async function checkMonitor(monitor: Monitor, fixtureOrigin: string): Promise<CheckRun> {
+export async function checkMonitor(monitor: Monitor, fixtureOrigin: string,
+  execution?: Pick<TerminalCheckRun, 'id' | 'trigger' | 'startedAt'>): Promise<TerminalCheckRun> {
   // Recheck at the outbound boundary, even though creation also checks the URL.
   const url = fixtureUrl(monitor.url, fixtureOrigin);
-  const startedAt = new Date().toISOString();
+  const startedAt = execution?.startedAt ?? new Date().toISOString();
   const start = performance.now();
 
   return new Promise((resolve) => {
@@ -32,7 +33,7 @@ export async function checkMonitor(monitor: Monitor, fixtureOrigin: string): Pro
       settled = true;
       clearTimeout(deadline);
       resolve({
-        id: randomUUID(), monitorId: monitor.id, trigger: 'MANUAL',
+        id: execution?.id ?? randomUUID(), monitorId: monitor.id, trigger: execution?.trigger ?? 'MANUAL',
         state: failureReason === null ? 'SUCCEEDED' : 'FAILED',
         httpStatus, latencyMs: Math.round(performance.now() - start), failureReason,
         startedAt, finishedAt: new Date().toISOString(),
diff --git a/server/mapping.ts b/server/mapping.ts
index 31ea438..89352e3 100644
--- a/server/mapping.ts
+++ b/server/mapping.ts
@@ -8,8 +8,8 @@ export type MonitorRow = {
 };
 export type CheckRunRow = {
   id: string; monitor_id: string; trigger: CheckRun['trigger']; state: CheckRun['state'];
-  http_status: number | null; latency_ms: number; failure_reason: CheckRun['failureReason'];
-  started_at: Date; finished_at: Date;
+  http_status: number | null; latency_ms: number | null; failure_reason: CheckRun['failureReason'];
+  started_at: Date | null; finished_at: Date | null;
 };
 export type MonitorViewRow = MonitorRow & Omit<CheckRunRow, 'id'> & { check_id: string | null };
 export type UserRow = { id: string; username: string; password_hash: string };
@@ -30,7 +30,7 @@ export function checkRunFromRow(row: CheckRunRow): CheckRun {
   return {
     id: row.id, monitorId: row.monitor_id, trigger: row.trigger, state: row.state,
     httpStatus: row.http_status, latencyMs: row.latency_ms, failureReason: row.failure_reason,
-    startedAt: row.started_at.toISOString(), finishedAt: row.finished_at.toISOString(),
+    startedAt: row.started_at?.toISOString() ?? null, finishedAt: row.finished_at?.toISOString() ?? null,
   };
 }
 
@@ -48,5 +48,6 @@ export function monitorToValues(monitor: Monitor) {
 
 export function checkRunToValues(check: CheckRun) {
   return [check.id, check.monitorId, check.trigger, check.state, check.httpStatus,
-    check.latencyMs, check.failureReason, new Date(check.startedAt), new Date(check.finishedAt)];
+    check.latencyMs, check.failureReason, check.startedAt === null ? null : new Date(check.startedAt),
+    check.finishedAt === null ? null : new Date(check.finishedAt)];
 }
diff --git a/server/migrate.ts b/server/migrate.ts
index a79630f..357413c 100644
--- a/server/migrate.ts
+++ b/server/migrate.ts
@@ -5,7 +5,7 @@ import { databaseConfig, databasePool, schemaIdentifier } from './database.ts';
 import type { DatabaseConfig } from './database.ts';
 
 export async function migrationFiles() {
-  const names = ['001_monitors.sql', '002_check_runs.sql', '003_sessions.sql', '004_monitor_ownership.sql', '005_check_history_index.sql'];
+  const names = ['001_monitors.sql', '002_check_runs.sql', '003_sessions.sql', '004_monitor_ownership.sql', '005_check_history_index.sql', '006_check_queue.sql'];
   return Promise.all(names.map(async (version) => {
     const sql = await readFile(new URL(`./migrations/${version}`, import.meta.url), 'utf8');
     return { version, sql, checksum: createHash('sha256').update(sql).digest('hex') };
diff --git a/server/migrations/006_check_queue.sql b/server/migrations/006_check_queue.sql
new file mode 100644
index 0000000..c6cc8ae
--- /dev/null
+++ b/server/migrations/006_check_queue.sql
@@ -0,0 +1,33 @@
+ALTER TABLE check_runs
+  DROP CONSTRAINT check_runs_trigger_check,
+  DROP CONSTRAINT check_runs_state_check,
+  DROP CONSTRAINT check_runs_check1,
+  ALTER COLUMN latency_ms DROP NOT NULL,
+  ALTER COLUMN started_at DROP NOT NULL,
+  ALTER COLUMN finished_at DROP NOT NULL,
+  ADD COLUMN queued_at timestamptz(3),
+  ADD COLUMN scheduled_for timestamptz(3);
+
+UPDATE check_runs SET queued_at = started_at;
+ALTER TABLE check_runs
+  ALTER COLUMN queued_at SET NOT NULL,
+  ALTER COLUMN queued_at SET DEFAULT current_timestamp,
+  ADD CONSTRAINT check_runs_trigger_check CHECK (trigger IN ('MANUAL', 'SCHEDULED')),
+  ADD CONSTRAINT check_runs_state_check CHECK (state IN ('QUEUED', 'RUNNING', 'SUCCEEDED', 'FAILED')),
+  ADD CONSTRAINT check_runs_scheduled_slot_check CHECK (
+    (trigger = 'MANUAL' AND scheduled_for IS NULL)
+    OR (trigger = 'SCHEDULED' AND scheduled_for IS NOT NULL)
+  ),
+  ADD CONSTRAINT check_runs_scheduled_slot_key UNIQUE (monitor_id, scheduled_for),
+  ADD CONSTRAINT check_runs_lifecycle_check CHECK (
+    (state = 'QUEUED' AND started_at IS NULL AND finished_at IS NULL
+      AND http_status IS NULL AND latency_ms IS NULL AND failure_reason IS NULL)
+    OR (state = 'RUNNING' AND started_at IS NOT NULL AND finished_at IS NULL
+      AND http_status IS NULL AND latency_ms IS NULL AND failure_reason IS NULL)
+    OR (state IN ('SUCCEEDED', 'FAILED') AND started_at IS NOT NULL AND finished_at IS NOT NULL
+      AND latency_ms IS NOT NULL AND (
+        (state = 'SUCCEEDED' AND http_status IS NOT NULL AND http_status BETWEEN 200 AND 299 AND failure_reason IS NULL)
+        OR (state = 'FAILED' AND failure_reason = 'HTTP_STATUS' AND http_status IS NOT NULL AND (http_status < 200 OR http_status >= 300))
+        OR (state = 'FAILED' AND failure_reason IN ('TIMEOUT', 'CONNECTION_FAILURE') AND http_status IS NULL)
+      ))
+  );
diff --git a/server/model.ts b/server/model.ts
index 8f53931..82bfb4a 100644
--- a/server/model.ts
+++ b/server/model.ts
@@ -11,22 +11,26 @@ export type Monitor = {
 export type CheckRun = {
   id: string;
   monitorId: string;
-  trigger: 'MANUAL';
-  state: 'SUCCEEDED' | 'FAILED';
+  trigger: 'MANUAL' | 'SCHEDULED';
+  state: 'QUEUED' | 'RUNNING' | 'SUCCEEDED' | 'FAILED';
   httpStatus: number | null;
-  latencyMs: number;
+  latencyMs: number | null;
   failureReason: 'HTTP_STATUS' | 'TIMEOUT' | 'CONNECTION_FAILURE' | null;
-  startedAt: string;
-  finishedAt: string;
+  startedAt: string | null;
+  finishedAt: string | null;
+};
+
+export type TerminalCheckRun = CheckRun & {
+  state: 'SUCCEEDED' | 'FAILED'; latencyMs: number; startedAt: string; finishedAt: string;
 };
 
 export type MonitorView = Monitor & { latestCheck: CheckRun | null };
 
 export type CheckHistoryPage = {
-  items: CheckRun[];
+  items: TerminalCheckRun[];
   nextCursor: string | null;
   limit: number;
-  state: CheckRun['state'] | null;
+  state: TerminalCheckRun['state'] | null;
 };
 
 export type User = { id: string; username: string };
diff --git a/server/schema.ts b/server/schema.ts
index 28a026a..33936ab 100644
--- a/server/schema.ts
+++ b/server/schema.ts
@@ -9,8 +9,9 @@ const expectedColumns = {
   },
   check_runs: {
     id: ['uuid', 'NO'], monitor_id: ['uuid', 'NO'], trigger: ['text', 'NO'], state: ['text', 'NO'],
-    http_status: ['int4', 'YES'], latency_ms: ['int4', 'NO'], failure_reason: ['text', 'YES'],
-    started_at: ['timestamptz', 'NO'], finished_at: ['timestamptz', 'NO'],
+    http_status: ['int4', 'YES'], latency_ms: ['int4', 'YES'], failure_reason: ['text', 'YES'],
+    started_at: ['timestamptz', 'YES'], finished_at: ['timestamptz', 'YES'],
+    queued_at: ['timestamptz', 'NO'], scheduled_for: ['timestamptz', 'YES'],
   },
   users: {
     id: ['uuid', 'NO'], username: ['text', 'NO'], password_hash: ['text', 'NO'],
@@ -58,6 +59,9 @@ export async function verifySchema(pool: Pool): Promise<void> {
     if (!keys.rows.some((row) => row.table_name === 'check_runs' && row.definition === 'FOREIGN KEY (monitor_id) REFERENCES monitors(id) ON DELETE CASCADE')) {
       throw new Error('Incompatible database schema: CheckRun parent deletion rule.');
     }
+    if (!keys.rows.some((row) => row.table_name === 'check_runs' && row.definition === 'UNIQUE (monitor_id, scheduled_for)')) {
+      throw new Error('Incompatible database schema: scheduled due-slot identity.');
+    }
     if (!keys.rows.some((row) => row.table_name === 'users' && row.definition === 'UNIQUE (username)')) {
       throw new Error('Incompatible database schema: unique username.');
     }
diff --git a/server/worker.ts b/server/worker.ts
new file mode 100644
index 0000000..c9ec922
--- /dev/null
+++ b/server/worker.ts
@@ -0,0 +1,57 @@
+import type { Pool } from 'pg';
+import { pathToFileURL } from 'node:url';
+import { setTimeout as delay } from 'node:timers/promises';
+import { checkMonitor, DEFAULT_FIXTURE_ORIGIN } from './check.ts';
+import { databaseConfig, databasePool } from './database.ts';
+import { checkRunFromRow, monitorFromRow } from './mapping.ts';
+import type { CheckRunRow, MonitorRow } from './mapping.ts';
+import { verifySchema } from './schema.ts';
+
+export async function scheduleDueChecks(pool: Pool, now = new Date()) {
+  // The current interval slot is anchored to creation. Repeated ticks address
+  // the same durable slot; manual intents have no slot and may overlap it.
+  const result = await pool.query<CheckRunRow>(`INSERT INTO check_runs
+    (id, monitor_id, trigger, state, queued_at, scheduled_for)
+    SELECT gen_random_uuid(), id, 'SCHEDULED', 'QUEUED', $1,
+      date_bin(interval_seconds * interval '1 second', $1::timestamptz, created_at)
+    FROM monitors WHERE enabled AND created_at <= $1
+    ON CONFLICT (monitor_id, scheduled_for) DO NOTHING RETURNING *`, [now]);
+  return result.rows.map(checkRunFromRow);
+}
+
+export async function runNextCheck(pool: Pool, fixtureOrigin = DEFAULT_FIXTURE_ORIGIN) {
+  // E09 operates one worker. Selection and update are deliberately separate;
+  // competing-worker ownership and interrupted RUNNING recovery are later work.
+  const selected = await pool.query<CheckRunRow>("SELECT * FROM check_runs WHERE state = 'QUEUED' ORDER BY queued_at, id LIMIT 1");
+  if (!selected.rows[0]) return null;
+  const running = await pool.query<CheckRunRow>("UPDATE check_runs SET state = 'RUNNING', started_at = $2 WHERE id = $1 RETURNING *",
+    [selected.rows[0].id, new Date()]);
+  if (!running.rows[0]) return null;
+  const monitor = await pool.query<MonitorRow>('SELECT * FROM monitors WHERE id = $1', [running.rows[0].monitor_id]);
+  if (!monitor.rows[0]) return null;
+  const execution = checkRunFromRow(running.rows[0]);
+  const observed = await checkMonitor(monitorFromRow(monitor.rows[0]), fixtureOrigin, {
+    id: execution.id, trigger: execution.trigger, startedAt: execution.startedAt!,
+  });
+  const saved = await pool.query<CheckRunRow>(`UPDATE check_runs SET state = $2, http_status = $3,
+    latency_ms = $4, failure_reason = $5, finished_at = $6 WHERE id = $1 RETURNING *`,
+  [observed.id, observed.state, observed.httpStatus, observed.latencyMs, observed.failureReason, new Date(observed.finishedAt)]);
+  return saved.rows[0] ? checkRunFromRow(saved.rows[0]) : null;
+}
+
+if (process.argv[1] && import.meta.url === pathToFileURL(process.argv[1]).href) {
+  const pool = databasePool(databaseConfig());
+  try {
+    await verifySchema(pool);
+    console.log('Check worker ready.');
+    for (;;) {
+      // Regression fixtures control scheduler time explicitly; normal operation
+      // always schedules. There is no separate scheduler daemon or queue store.
+      if (!process.argv.includes('--no-schedule')) await scheduleDueChecks(pool);
+      if (!await runNextCheck(pool, process.env.FIXTURE_ORIGIN ?? DEFAULT_FIXTURE_ORIGIN)) await delay(250);
+    }
+  } catch {
+    console.error('Check worker stopped after an execution or database failure.');
+    process.exitCode = 1;
+  } finally { await pool.end(); }
+}


