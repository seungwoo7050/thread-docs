## `feat: persist manual identities and atomic check ownership`

diff --git a/app/monitors/api.ts b/app/monitors/api.ts
index 88a2d19..778358f 100644
--- a/app/monitors/api.ts
+++ b/app/monitors/api.ts
@@ -5,9 +5,10 @@ export const ERROR_MESSAGES: Record<ApiErrorCode, string> = {
   UNAUTHENTICATED: 'Sign in to continue.',
   FORBIDDEN: 'The request was not permitted. Reload and try again.',
   NOT_FOUND: 'The requested monitor or resource was not found.',
+  CONFLICT: 'This check request was already used for another monitor.',
   INTERNAL_ERROR: 'The monitoring service could not complete the request.',
 };
-const ERROR_STATUS = { INVALID_INPUT: 400, UNAUTHENTICATED: 401, FORBIDDEN: 403, NOT_FOUND: 404, INTERNAL_ERROR: 500 } as const;
+const ERROR_STATUS = { INVALID_INPUT: 400, UNAUTHENTICATED: 401, FORBIDDEN: 403, NOT_FOUND: 404, CONFLICT: 409, INTERNAL_ERROR: 500 } as const;
 
 export class RequestFailure extends Error {
   readonly code: ApiErrorCode;
@@ -31,7 +32,7 @@ export async function responseData<T>(response: Response): Promise<T> {
   if (!response.ok) {
     if (isObject(body) && isObject(body.error) && typeof body.error.message === 'string') {
       const code = body.error.code;
-      if ((code === 'INVALID_INPUT' || code === 'UNAUTHENTICATED' || code === 'FORBIDDEN' || code === 'NOT_FOUND' || code === 'INTERNAL_ERROR') && ERROR_STATUS[code] === response.status) {
+      if ((code === 'INVALID_INPUT' || code === 'UNAUTHENTICATED' || code === 'FORBIDDEN' || code === 'NOT_FOUND' || code === 'CONFLICT' || code === 'INTERNAL_ERROR') && ERROR_STATUS[code] === response.status) {
         throw new RequestFailure(code);
       }
     }
diff --git a/app/monitors/use-monitors.ts b/app/monitors/use-monitors.ts
index 985dedd..0272e85 100644
--- a/app/monitors/use-monitors.ts
+++ b/app/monitors/use-monitors.ts
@@ -25,12 +25,14 @@ export function useMonitors(initial: InitialMonitors) {
   const [state, setState] = useState<ServerState>(() => ({ ...initial, mutations: {} }));
   const lifetime = useRef(0);
   const pending = useRef(new Set<string>());
+  const checkIntents = useRef(new Map<string, string>());
   const historyVersions = useRef(new Map<string, number>());
   const historySearches = useRef(new Map(Object.entries(initial.histories).map(([id, query]) => [id, query.search])));
 
   const clearSession = useCallback(() => {
     lifetime.current++;
     pending.current.clear();
+    checkIntents.current.clear();
     historyVersions.current.clear();
     historySearches.current.clear();
     setState(emptyState());
@@ -128,6 +130,13 @@ export function useMonitors(initial: InitialMonitors) {
       const path = action.kind === 'create' ? '/api/monitors' : action.kind === 'logout' ? '/api/auth/logout'
         : `/api/monitors/${action.id}${action.kind === 'check' ? '/checks' : ''}`;
       const options: RequestInit = { method: action.kind === 'delete' ? 'DELETE' : action.kind === 'update' ? 'PUT' : 'POST' };
+      if (action.kind === 'check') {
+        // An unacknowledged intent survives a manual retransmission. Only a
+        // successful acknowledgement makes the next click a new intention.
+        const identity = checkIntents.current.get(action.id) ?? crypto.randomUUID();
+        checkIntents.current.set(action.id, identity);
+        options.headers = { 'Idempotency-Key': identity };
+      }
       if ('input' in action) {
         options.headers = { 'content-type': 'application/json' };
         options.body = JSON.stringify(action.input);
@@ -135,6 +144,7 @@ export function useMonitors(initial: InitialMonitors) {
       const data = await responseData<MonitorView | CheckRun | { id: string } | { loggedOut: true }>(await mutationFetch(path, options));
       if (generation !== lifetime.current) return false;
       if (action.kind === 'logout') { clearSession(); return true; }
+      if (action.kind === 'check' || action.kind === 'delete') checkIntents.current.delete(action.id);
       const historySearch = action.kind === 'check' ? historySearches.current.get(action.id) : undefined;
       const refreshHistory = historySearch !== undefined;
       if (action.kind === 'delete' || (action.kind === 'check' && refreshHistory)) {
diff --git a/server/app.ts b/server/app.ts
index 2aaa12f..8611bd1 100644
--- a/server/app.ts
+++ b/server/app.ts
@@ -2,7 +2,7 @@ import { randomUUID } from 'node:crypto';
 import Fastify, { errorCodes } from 'fastify';
 import type { FastifyReply, FastifyRequest } from 'fastify';
 import { DEFAULT_FIXTURE_ORIGIN } from './check.ts';
-import { ApiError, ERROR_STATUS, monitorId, monitorInput } from './contracts.ts';
+import { ApiError, ERROR_STATUS, idempotencyKey, monitorId, monitorInput } from './contracts.ts';
 import type { ApiFailure, Monitor, TerminalCheckRun } from './model.ts';
 import { databaseConfig, databasePool } from './database.ts';
 import type { DatabaseConfig } from './database.ts';
@@ -101,12 +101,27 @@ export function buildApp(fixtureOrigin = DEFAULT_FIXTURE_ORIGIN, database: Datab
   });
 
   app.post<{ Params: { id: string } }>('/monitors/:id/checks', async (request, reply) => {
-    const saved = await pool.query<CheckRunRow>(`INSERT INTO check_runs
-      (id, monitor_id, trigger, state)
-      SELECT $1, id, 'MANUAL', 'QUEUED' FROM monitors
-      WHERE id = $2 AND owner_user_id = $3 RETURNING *`, [randomUUID(), monitorId(request.params.id), request.user!.id]);
-    if (!saved.rows[0]) throw new ApiError('NOT_FOUND', 'Monitor not found.');
-    return reply.code(202).send({ data: checkRunFromRow(saved.rows[0]) });
+    const id = monitorId(request.params.id);
+    const key = idempotencyKey(request.headers['idempotency-key']);
+    const userId = request.user!.id;
+    const client = await pool.connect();
+    try {
+      await client.query('BEGIN');
+      const parent = await client.query('SELECT id FROM monitors WHERE id = $1 AND owner_user_id = $2 FOR KEY SHARE', [id, userId]);
+      if (!parent.rows[0]) throw new ApiError('NOT_FOUND', 'Monitor not found.');
+      const inserted = await client.query<CheckRunRow>(`INSERT INTO check_runs
+        (id, monitor_id, trigger, state, request_user_id, idempotency_key)
+        VALUES ($1, $2, 'MANUAL', 'QUEUED', $3, $4)
+        ON CONFLICT (request_user_id, idempotency_key) DO NOTHING RETURNING *`, [randomUUID(), id, userId, key]);
+      // A separate READ COMMITTED statement sees the winner after a concurrent
+      // insert commits. Meaning is the target UUID, never mutable Monitor data.
+      const saved = inserted.rows[0] ?? (await client.query<CheckRunRow>(
+        'SELECT * FROM check_runs WHERE request_user_id = $1 AND idempotency_key = $2', [userId, key])).rows[0];
+      if (saved.monitor_id !== id) throw new ApiError('CONFLICT', 'Idempotency-Key was already used for another Monitor.');
+      await client.query('COMMIT');
+      return reply.code(202).send({ data: checkRunFromRow(saved) });
+    } catch (error) { await client.query('ROLLBACK'); throw error; }
+    finally { client.release(); }
   });
 
   app.get<{ Params: { id: string } }>('/monitors/:id/checks', async (request) => {
diff --git a/server/contracts.ts b/server/contracts.ts
index 541fd11..5a2421d 100644
--- a/server/contracts.ts
+++ b/server/contracts.ts
@@ -1,7 +1,7 @@
 import { fixtureUrl } from './check.ts';
 import type { ApiErrorCode, Monitor } from './model.ts';
 
-export const ERROR_STATUS = { INVALID_INPUT: 400, UNAUTHENTICATED: 401, FORBIDDEN: 403, NOT_FOUND: 404, INTERNAL_ERROR: 500 } as const;
+export const ERROR_STATUS = { INVALID_INPUT: 400, UNAUTHENTICATED: 401, FORBIDDEN: 403, NOT_FOUND: 404, CONFLICT: 409, INTERNAL_ERROR: 500 } as const;
 
 export class ApiError extends Error {
   readonly code: ApiErrorCode;
@@ -43,6 +43,13 @@ export function monitorId(value: string): string {
   return value.toLowerCase();
 }
 
+export function idempotencyKey(value: unknown): string {
+  if (typeof value !== 'string' || value.length < 1 || value.length > 128 || /[^!-~]/.test(value)) {
+    throw new ApiError('INVALID_INPUT', 'Idempotency-Key must contain 1–128 printable ASCII characters without whitespace.');
+  }
+  return value;
+}
+
 export function loginInput(value: unknown): { username: string; password: string } {
   if (value === null || typeof value !== 'object' || Array.isArray(value)) {
     throw new ApiError('INVALID_INPUT', 'A login JSON object is required.');
diff --git a/server/migrate.ts b/server/migrate.ts
index 357413c..567f5bf 100644
--- a/server/migrate.ts
+++ b/server/migrate.ts
@@ -5,7 +5,7 @@ import { databaseConfig, databasePool, schemaIdentifier } from './database.ts';
 import type { DatabaseConfig } from './database.ts';
 
 export async function migrationFiles() {
-  const names = ['001_monitors.sql', '002_check_runs.sql', '003_sessions.sql', '004_monitor_ownership.sql', '005_check_history_index.sql', '006_check_queue.sql'];
+  const names = ['001_monitors.sql', '002_check_runs.sql', '003_sessions.sql', '004_monitor_ownership.sql', '005_check_history_index.sql', '006_check_queue.sql', '007_check_ownership.sql'];
   return Promise.all(names.map(async (version) => {
     const sql = await readFile(new URL(`./migrations/${version}`, import.meta.url), 'utf8');
     return { version, sql, checksum: createHash('sha256').update(sql).digest('hex') };
diff --git a/server/migrations/007_check_ownership.sql b/server/migrations/007_check_ownership.sql
new file mode 100644
index 0000000..0824b6c
--- /dev/null
+++ b/server/migrations/007_check_ownership.sql
@@ -0,0 +1,10 @@
+ALTER TABLE check_runs
+  ADD COLUMN request_user_id uuid REFERENCES users(id) ON DELETE RESTRICT,
+  ADD COLUMN idempotency_key text,
+  ADD COLUMN worker_id uuid,
+  ADD CONSTRAINT check_runs_manual_identity_check CHECK (
+    (request_user_id IS NULL AND idempotency_key IS NULL)
+    OR (trigger = 'MANUAL' AND request_user_id IS NOT NULL AND idempotency_key IS NOT NULL
+      AND idempotency_key COLLATE "C" ~ '^[!-~]{1,128}$')
+  ),
+  ADD CONSTRAINT check_runs_manual_identity_key UNIQUE (request_user_id, idempotency_key);
diff --git a/server/model.ts b/server/model.ts
index 82bfb4a..9d0b208 100644
--- a/server/model.ts
+++ b/server/model.ts
@@ -35,6 +35,6 @@ export type CheckHistoryPage = {
 
 export type User = { id: string; username: string };
 
-export type ApiErrorCode = 'INVALID_INPUT' | 'UNAUTHENTICATED' | 'FORBIDDEN' | 'NOT_FOUND' | 'INTERNAL_ERROR';
+export type ApiErrorCode = 'INVALID_INPUT' | 'UNAUTHENTICATED' | 'FORBIDDEN' | 'NOT_FOUND' | 'CONFLICT' | 'INTERNAL_ERROR';
 export type ApiSuccess<T> = { data: T };
 export type ApiFailure = { error: { code: ApiErrorCode; message: string } };
diff --git a/server/schema.ts b/server/schema.ts
index 33936ab..e7f94a4 100644
--- a/server/schema.ts
+++ b/server/schema.ts
@@ -12,6 +12,7 @@ const expectedColumns = {
     http_status: ['int4', 'YES'], latency_ms: ['int4', 'YES'], failure_reason: ['text', 'YES'],
     started_at: ['timestamptz', 'YES'], finished_at: ['timestamptz', 'YES'],
     queued_at: ['timestamptz', 'NO'], scheduled_for: ['timestamptz', 'YES'],
+    request_user_id: ['uuid', 'YES'], idempotency_key: ['text', 'YES'], worker_id: ['uuid', 'YES'],
   },
   users: {
     id: ['uuid', 'NO'], username: ['text', 'NO'], password_hash: ['text', 'NO'],
@@ -62,6 +63,10 @@ export async function verifySchema(pool: Pool): Promise<void> {
     if (!keys.rows.some((row) => row.table_name === 'check_runs' && row.definition === 'UNIQUE (monitor_id, scheduled_for)')) {
       throw new Error('Incompatible database schema: scheduled due-slot identity.');
     }
+    if (!keys.rows.some((row) => row.table_name === 'check_runs' && row.definition === 'UNIQUE (request_user_id, idempotency_key)') ||
+      !keys.rows.some((row) => row.table_name === 'check_runs' && row.definition === 'FOREIGN KEY (request_user_id) REFERENCES users(id) ON DELETE RESTRICT')) {
+      throw new Error('Incompatible database schema: manual request identity.');
+    }
     if (!keys.rows.some((row) => row.table_name === 'users' && row.definition === 'UNIQUE (username)')) {
       throw new Error('Incompatible database schema: unique username.');
     }
diff --git a/server/worker.ts b/server/worker.ts
index c9ec922..6bf6277 100644
--- a/server/worker.ts
+++ b/server/worker.ts
@@ -1,9 +1,11 @@
 import type { Pool } from 'pg';
+import { randomUUID } from 'node:crypto';
 import { pathToFileURL } from 'node:url';
 import { setTimeout as delay } from 'node:timers/promises';
 import { checkMonitor, DEFAULT_FIXTURE_ORIGIN } from './check.ts';
 import { databaseConfig, databasePool } from './database.ts';
 import { checkRunFromRow, monitorFromRow } from './mapping.ts';
+import type { TerminalCheckRun } from './model.ts';
 import type { CheckRunRow, MonitorRow } from './mapping.ts';
 import { verifySchema } from './schema.ts';
 
@@ -19,36 +21,50 @@ export async function scheduleDueChecks(pool: Pool, now = new Date()) {
   return result.rows.map(checkRunFromRow);
 }
 
-export async function runNextCheck(pool: Pool, fixtureOrigin = DEFAULT_FIXTURE_ORIGIN) {
-  // E09 operates one worker. Selection and update are deliberately separate;
-  // competing-worker ownership and interrupted RUNNING recovery are later work.
-  const selected = await pool.query<CheckRunRow>("SELECT * FROM check_runs WHERE state = 'QUEUED' ORDER BY queued_at, id LIMIT 1");
-  if (!selected.rows[0]) return null;
-  const running = await pool.query<CheckRunRow>("UPDATE check_runs SET state = 'RUNNING', started_at = $2 WHERE id = $1 RETURNING *",
-    [selected.rows[0].id, new Date()]);
-  if (!running.rows[0]) return null;
-  const monitor = await pool.query<MonitorRow>('SELECT * FROM monitors WHERE id = $1', [running.rows[0].monitor_id]);
+export async function completeCheck(pool: Pool, workerId: string, observed: TerminalCheckRun) {
+  const saved = await pool.query<CheckRunRow>(`UPDATE check_runs SET state = $2, http_status = $3,
+    latency_ms = $4, failure_reason = $5, finished_at = $6
+    WHERE id = $1 AND state = 'RUNNING' AND worker_id = $7 RETURNING *`,
+  [observed.id, observed.state, observed.httpStatus, observed.latencyMs, observed.failureReason, new Date(observed.finishedAt), workerId]);
+  return saved.rows[0] ? checkRunFromRow(saved.rows[0]) : null;
+}
+
+export async function runNextCheck(pool: Pool, fixtureOrigin = DEFAULT_FIXTURE_ORIGIN, workerId = randomUUID()) {
+  const client = await pool.connect();
+  let claimed: CheckRunRow | undefined;
+  try {
+    await client.query('BEGIN');
+    const running = await client.query<CheckRunRow>(`UPDATE check_runs
+      SET state = 'RUNNING', started_at = clock_timestamp(), worker_id = $1
+      WHERE id = (SELECT id FROM check_runs WHERE state = 'QUEUED'
+        ORDER BY queued_at, id LIMIT 1 FOR UPDATE SKIP LOCKED)
+        AND state = 'QUEUED' RETURNING *`, [workerId]);
+    claimed = running.rows[0];
+    await client.query('COMMIT');
+  } catch (error) { await client.query('ROLLBACK'); throw error; }
+  finally { client.release(); }
+  if (!claimed) return null;
+  // Commit ownership before outbound I/O; a competing worker skips this row.
+  const monitor = await pool.query<MonitorRow>('SELECT * FROM monitors WHERE id = $1', [claimed.monitor_id]);
   if (!monitor.rows[0]) return null;
-  const execution = checkRunFromRow(running.rows[0]);
+  const execution = checkRunFromRow(claimed);
   const observed = await checkMonitor(monitorFromRow(monitor.rows[0]), fixtureOrigin, {
     id: execution.id, trigger: execution.trigger, startedAt: execution.startedAt!,
   });
-  const saved = await pool.query<CheckRunRow>(`UPDATE check_runs SET state = $2, http_status = $3,
-    latency_ms = $4, failure_reason = $5, finished_at = $6 WHERE id = $1 RETURNING *`,
-  [observed.id, observed.state, observed.httpStatus, observed.latencyMs, observed.failureReason, new Date(observed.finishedAt)]);
-  return saved.rows[0] ? checkRunFromRow(saved.rows[0]) : null;
+  return completeCheck(pool, workerId, observed);
 }
 
 if (process.argv[1] && import.meta.url === pathToFileURL(process.argv[1]).href) {
   const pool = databasePool(databaseConfig());
+  const workerId = randomUUID();
   try {
     await verifySchema(pool);
-    console.log('Check worker ready.');
+    console.log(`Check worker ready. ${workerId}`);
     for (;;) {
       // Regression fixtures control scheduler time explicitly; normal operation
       // always schedules. There is no separate scheduler daemon or queue store.
       if (!process.argv.includes('--no-schedule')) await scheduleDueChecks(pool);
-      if (!await runNextCheck(pool, process.env.FIXTURE_ORIGIN ?? DEFAULT_FIXTURE_ORIGIN)) await delay(250);
+      if (!await runNextCheck(pool, process.env.FIXTURE_ORIGIN ?? DEFAULT_FIXTURE_ORIGIN, workerId)) await delay(250);
     }
   } catch {
     console.error('Check worker stopped after an execution or database failure.');


