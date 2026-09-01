## `CheckRun 이력을 검증된 커서로 제한 조회`

diff --git a/TRACK.md b/TRACK.md
index 963fedb..7670f38 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -408,3 +408,39 @@ submit event produced two POSTs and two rows. The same frozen harness now requir
 one POST/row, holds that response while one update and one delete return fixed
 500 errors, and verifies state before and after explicit release. No sleeps,
 automatic retries, traces, videos or credentials are used as evidence.
+
+## E07 bounded CheckRun history
+
+`GET /monitors/:id/checks` now returns
+`{data: {items: CheckRun[], nextCursor: string | null, limit: number, state: "SUCCEEDED" | "FAILED" | null}}`.
+This replaces E03's unbounded history array; individual CheckRun fields and all
+other response contracts are unchanged. Omitted `limit` means **20**; explicit
+limits must be decimal integers from **1 through 100**. Optional `state` is
+exactly `SUCCEEDED` or `FAILED`. Empty, repeated, malformed or unsupported
+parameters return `400 INVALID_INPUT`, as do invalid cursors.
+
+Rows retain `finished_at DESC, id DESC` ordering. A query reads at most `limit + 1`
+rows and returns at most `limit`; the extra row determines whether `nextCursor`
+is non-null. The next request repeats the same resource, state and limit and
+passes that cursor. A cursor is canonical base64url JSON, bounded to 512
+characters, with version 1, Monitor ID, state, limit, finished timestamp and row
+ID. Its exact fields, types, millisecond UTC timestamp and UUID are validated.
+Changed resource/filter/size conditions are rejected. Cursors carry no credential
+and grant no access: parent and history SQL still constrain the authenticated
+owner, including on continuation requests.
+
+The seek condition `(c.finished_at, c.id) < ($4::timestamptz, $5::uuid)` avoids
+OFFSET movement when newer rows arrive. It guarantees the unchanged older rows
+continue once each; it is not a snapshot across deletions or changing row data.
+Every external SQL value is a bound parameter. Migration
+`005_check_history_index.sql` adds only `(monitor_id, finished_at DESC, id DESC)`;
+the state predicate remains a filter over that Monitor history. Earlier
+migrations and pins are unchanged. No EXPLAIN study or planner tuning is claimed.
+
+`evidence/E07/scenario.json` and `fixture.ts` were hashed before one request at
+the unchanged E06 START. Its seven tied terminal rows returned as seven despite
+`limit=3`, with no cursor. Unit and actual PostgreSQL tests use those same rows,
+validate the default/maximum and bad cursor boundaries, traverse all seven IDs,
+read the three FAILED rows, and continue after inserting the fixed eighth newer
+row. Existing E03/E05 assertions still compare the exact historical rows after
+extracting `items` from the new envelope.
diff --git a/app/monitors/page.tsx b/app/monitors/page.tsx
index 5d73da9..d3daa15 100644
--- a/app/monitors/page.tsx
+++ b/app/monitors/page.tsx
@@ -82,7 +82,7 @@ export default function MonitorsPage() {
         const mutation = mutations[`monitor:${monitor.id}`];
         const pending = mutation?.status === 'pending';
         const historyQuery = histories[monitor.id];
-        const history = historyQuery?.data ?? [];
+        const history = historyQuery?.data?.items ?? [];
         return <article key={monitor.id} className="panel" aria-labelledby={`monitor-${monitor.id}`}>
         <h3 id={`monitor-${monitor.id}`}>{monitor.name}</h3>
         <p className="endpoint">{monitor.url}</p>
diff --git a/app/monitors/use-monitors.ts b/app/monitors/use-monitors.ts
index 466b25d..8415182 100644
--- a/app/monitors/use-monitors.ts
+++ b/app/monitors/use-monitors.ts
@@ -2,7 +2,7 @@
 
 import { useCallback, useEffect, useRef, useState } from 'react';
 import { useRouter } from 'next/navigation';
-import type { ApiErrorCode, CheckRun, MonitorView } from '../../server/model.ts';
+import type { ApiErrorCode, CheckHistoryPage, CheckRun, MonitorView } from '../../server/model.ts';
 import { failureCode, mutationFetch, responseData } from './api.ts';
 
 type MonitorInput = Pick<MonitorView, 'name' | 'url' | 'interval' | 'enabled'>;
@@ -12,7 +12,7 @@ type Mutation =
   | { kind: 'delete' | 'check'; id: string }
   | { kind: 'logout' };
 type MutationState = { kind: Mutation['kind']; status: 'pending' | 'success' | 'error'; error: ApiErrorCode | null };
-type HistoryQuery = { status: 'pending' | 'success' | 'error'; data: CheckRun[] | null; error: ApiErrorCode | null };
+type HistoryQuery = { status: 'pending' | 'success' | 'error'; data: CheckHistoryPage | null; error: ApiErrorCode | null };
 type ServerState = {
   authenticated: boolean;
   loading: boolean;
@@ -69,7 +69,7 @@ export function useMonitors() {
     } }));
     const isCurrent = () => generation === lifetime.current && version === historyVersions.current.get(id);
     try {
-      const data = await responseData<CheckRun[]>(await fetch(`/api/monitors/${id}/checks`, { credentials: 'same-origin' }));
+      const data = await responseData<CheckHistoryPage>(await fetch(`/api/monitors/${id}/checks`, { credentials: 'same-origin' }));
       if (isCurrent()) setState((current) => ({ ...current, histories: {
         ...current.histories, [id]: { status: 'success', data, error: null },
       } }));
diff --git a/server/app.ts b/server/app.ts
index 12b53c9..42aceab 100644
--- a/server/app.ts
+++ b/server/app.ts
@@ -10,6 +10,7 @@ import { verifySchema } from './schema.ts';
 import { checkRunFromRow, checkRunToValues, monitorFromRow, monitorToValues, monitorViewFromRow } from './mapping.ts';
 import type { CheckRunRow, MonitorRow, MonitorViewRow } from './mapping.ts';
 import { registerAuthentication } from './auth.ts';
+import { historyCursor, historyQuery } from './history.ts';
 
 const inputErrors = [
   errorCodes.FST_ERR_CTP_INVALID_JSON_BODY,
@@ -111,11 +112,20 @@ export function buildApp(fixtureOrigin = DEFAULT_FIXTURE_ORIGIN, database: Datab
 
   app.get<{ Params: { id: string } }>('/monitors/:id/checks', async (request) => {
     const id = monitorId(request.params.id);
+    const query = historyQuery(id, request.query);
     const parent = await pool.query('SELECT id FROM monitors WHERE id = $1 AND owner_user_id = $2', [id, request.user!.id]);
     if (!parent.rows[0]) throw new ApiError('NOT_FOUND', 'Monitor not found.');
+    // Bind every external value, including the optional filter, seek tuple and
+    // limit. One look-ahead row determines continuation without a COUNT/OFFSET.
     const history = await pool.query<CheckRunRow>(`SELECT c.* FROM check_runs c JOIN monitors m ON m.id = c.monitor_id
-      WHERE m.id = $1 AND m.owner_user_id = $2 ORDER BY c.finished_at DESC, c.id DESC`, [id, request.user!.id]);
-    return { data: history.rows.map(checkRunFromRow) };
+      WHERE m.id = $1 AND m.owner_user_id = $2
+        AND ($3::text IS NULL OR c.state = $3)
+        AND ($4::timestamptz IS NULL OR (c.finished_at, c.id) < ($4::timestamptz, $5::uuid))
+      ORDER BY c.finished_at DESC, c.id DESC LIMIT $6`,
+    [id, request.user!.id, query.state, query.after?.finishedAt ?? null, query.after?.id ?? null, query.limit + 1]);
+    const items = history.rows.slice(0, query.limit).map(checkRunFromRow);
+    const nextCursor = history.rows.length > query.limit ? historyCursor(query, items[items.length - 1]) : null;
+    return { data: { items, nextCursor, limit: query.limit, state: query.state } };
   });
 
   app.get<{ Params: { id: string } }>('/checks/:id', async (request) => {
diff --git a/server/history.ts b/server/history.ts
new file mode 100644
index 0000000..94e6ba2
--- /dev/null
+++ b/server/history.ts
@@ -0,0 +1,57 @@
+import { ApiError, monitorId } from './contracts.ts';
+import type { CheckRun } from './model.ts';
+
+export const DEFAULT_HISTORY_LIMIT = 20;
+export const MAX_HISTORY_LIMIT = 100;
+export const MAX_HISTORY_CURSOR_LENGTH = 512;
+
+export type HistoryQuery = {
+  monitorId: string;
+  limit: number;
+  state: CheckRun['state'] | null;
+  after: { finishedAt: string; id: string } | null;
+};
+
+function invalidHistory(): never {
+  throw new ApiError('INVALID_INPUT', 'History requires a limit from 1 to 100, an optional SUCCEEDED or FAILED state, and a matching cursor.');
+}
+
+// The version fixes finished_at DESC, id DESC ordering. Cursor values are
+// position/condition data, never authorization; every read still checks owner.
+export function historyQuery(id: string, value: unknown): HistoryQuery {
+  if (value === null || typeof value !== 'object' || Array.isArray(value)) invalidHistory();
+  const query = value as Record<string, unknown>;
+  if (Object.keys(query).some((key) => !['limit', 'state', 'cursor'].includes(key))) invalidHistory();
+  let limit = DEFAULT_HISTORY_LIMIT;
+  if (query.limit !== undefined) {
+    if (typeof query.limit !== 'string' || !/^[0-9]{1,3}$/.test(query.limit)) invalidHistory();
+    limit = Number(query.limit);
+    if (limit < 1 || limit > MAX_HISTORY_LIMIT) invalidHistory();
+  }
+  const state = query.state ?? null;
+  if (state !== null && state !== 'SUCCEEDED' && state !== 'FAILED') invalidHistory();
+  const conditions: HistoryQuery = { monitorId: id, limit, state, after: null };
+  if (query.cursor === undefined) return conditions;
+  const token = query.cursor;
+  if (typeof token !== 'string' || token.length > MAX_HISTORY_CURSOR_LENGTH || !/^[A-Za-z0-9_-]+$/.test(token)) invalidHistory();
+  const bytes = Buffer.from(token, 'base64url');
+  if (bytes.toString('base64url') !== token) invalidHistory();
+  let decoded: unknown;
+  try { decoded = JSON.parse(bytes.toString('utf8')); }
+  catch { invalidHistory(); }
+  if (decoded === null || typeof decoded !== 'object' || Array.isArray(decoded)) invalidHistory();
+  const cursor = decoded as Record<string, unknown>;
+  if (Object.keys(cursor).length !== 6 || cursor.version !== 1 || cursor.monitorId !== id ||
+    cursor.limit !== limit || cursor.state !== state || typeof cursor.finishedAt !== 'string' ||
+    !/^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}\.\d{3}Z$/.test(cursor.finishedAt) ||
+    !Number.isFinite(Date.parse(cursor.finishedAt)) || new Date(cursor.finishedAt).toISOString() !== cursor.finishedAt ||
+    typeof cursor.id !== 'string' || monitorId(cursor.id) !== cursor.id) invalidHistory();
+  return { ...conditions, after: { finishedAt: cursor.finishedAt, id: cursor.id } };
+}
+
+export function historyCursor(query: HistoryQuery, last: Pick<CheckRun, 'id' | 'finishedAt'>): string {
+  return Buffer.from(JSON.stringify({
+    version: 1, monitorId: query.monitorId, state: query.state, limit: query.limit,
+    finishedAt: last.finishedAt, id: last.id,
+  })).toString('base64url');
+}
diff --git a/server/migrate.ts b/server/migrate.ts
index 43c640a..a79630f 100644
--- a/server/migrate.ts
+++ b/server/migrate.ts
@@ -5,7 +5,7 @@ import { databaseConfig, databasePool, schemaIdentifier } from './database.ts';
 import type { DatabaseConfig } from './database.ts';
 
 export async function migrationFiles() {
-  const names = ['001_monitors.sql', '002_check_runs.sql', '003_sessions.sql', '004_monitor_ownership.sql'];
+  const names = ['001_monitors.sql', '002_check_runs.sql', '003_sessions.sql', '004_monitor_ownership.sql', '005_check_history_index.sql'];
   return Promise.all(names.map(async (version) => {
     const sql = await readFile(new URL(`./migrations/${version}`, import.meta.url), 'utf8');
     return { version, sql, checksum: createHash('sha256').update(sql).digest('hex') };
diff --git a/server/migrations/005_check_history_index.sql b/server/migrations/005_check_history_index.sql
new file mode 100644
index 0000000..c090ec8
--- /dev/null
+++ b/server/migrations/005_check_history_index.sql
@@ -0,0 +1,2 @@
+CREATE INDEX check_runs_monitor_finished_id_idx
+  ON check_runs (monitor_id, finished_at DESC, id DESC);
diff --git a/server/model.ts b/server/model.ts
index 569c24c..8f53931 100644
--- a/server/model.ts
+++ b/server/model.ts
@@ -22,6 +22,13 @@ export type CheckRun = {
 
 export type MonitorView = Monitor & { latestCheck: CheckRun | null };
 
+export type CheckHistoryPage = {
+  items: CheckRun[];
+  nextCursor: string | null;
+  limit: number;
+  state: CheckRun['state'] | null;
+};
+
 export type User = { id: string; username: string };
 
 export type ApiErrorCode = 'INVALID_INPUT' | 'UNAUTHENTICATED' | 'FORBIDDEN' | 'NOT_FOUND' | 'INTERNAL_ERROR';
diff --git a/test/contracts.test.ts b/test/contracts.test.ts
index 18f9eab..f6db5e6 100644
--- a/test/contracts.test.ts
+++ b/test/contracts.test.ts
@@ -342,7 +342,7 @@ test('E05 fixed A/B ownership matrix: collection, nested/direct reads and every
       assert.deepEqual(list.json().data.map((item: MonitorView) => item.id), [monitors[i].id]);
       assert.equal(list.json().data[0].latestCheck.id, checks[i].id);
       assert.deepEqual((await owner(`/monitors/${monitors[i].id}`)).json().data, list.json().data[0]);
-      assert.deepEqual((await owner(`/monitors/${monitors[i].id}/checks`)).json().data, [checks[i]]);
+      assert.deepEqual((await owner(`/monitors/${monitors[i].id}/checks`)).json().data.items, [checks[i]]);
       assert.deepEqual((await owner(`/checks/${checks[i].id}`)).json().data, checks[i]);
       observations.push({ operation: `${i}: owner collection/detail/history/direct CheckRun`, status: 200 });
 
diff --git a/test/database.ts b/test/database.ts
index 47e11a3..66a1f73 100644
--- a/test/database.ts
+++ b/test/database.ts
@@ -2,7 +2,7 @@ import { databaseConfig, databasePool, schemaIdentifier } from '../server/databa
 import { migrate } from '../server/migrate.ts';
 
 export function testDatabaseConfig(schema: string) {
-  if (!/^e0[345]_[a-z_]+$/.test(schema)) throw new Error('Test cleanup is restricted to an explicit e03_, e04_ or e05_ schema.');
+  if (!/^e0[3457]_[a-z_]+$/.test(schema)) throw new Error('Test cleanup is restricted to an explicit e03_, e04_, e05_ or e07_ schema.');
   return { ...databaseConfig(), schema };
 }
 
diff --git a/test/persistence.test.ts b/test/persistence.test.ts
index 7182577..b576e16 100644
--- a/test/persistence.test.ts
+++ b/test/persistence.test.ts
@@ -8,7 +8,7 @@ import { buildApp } from '../server/app.ts';
 import { databasePool, schemaIdentifier } from '../server/database.ts';
 import { monitorFromRow, checkRunFromRow, monitorToValues, checkRunToValues } from '../server/mapping.ts';
 import type { MonitorRow, CheckRunRow } from '../server/mapping.ts';
-import type { CheckRun, Monitor, MonitorView } from '../server/model.ts';
+import type { CheckHistoryPage, CheckRun, Monitor, MonitorView } from '../server/model.ts';
 import { fixtureServer } from './fixture.ts';
 import { authenticatedFetch, authenticatedInject, loginForTest, prepareTestUsers } from './auth.ts';
 import authScenario from '../evidence/E04/scenario.json' with { type: 'json' };
@@ -53,9 +53,9 @@ test('E03 migration CLI applies the fresh chain and a repeated command changes n
   try {
     const options = { cwd: process.cwd(), env: { ...process.env, DATABASE_URL: config.connectionString, DATABASE_SCHEMA: config.schema }, timeout: 10_000 };
     const first = await execute(process.execPath, ['server/migrate.ts'], options);
-    assert.deepEqual(JSON.parse(first.stdout).applied, [...authScenario.additionalAssertions.expectedMigrationFiles, '004_monitor_ownership.sql']);
+    assert.deepEqual(JSON.parse(first.stdout).applied, [...authScenario.additionalAssertions.expectedMigrationFiles, '004_monitor_ownership.sql', '005_check_history_index.sql']);
     const before = (await pool.query('SELECT version, checksum, applied_at FROM schema_migrations ORDER BY version')).rows;
-    assert.equal(before.length, authScenario.additionalAssertions.expectedMigrationFiles.length + 1);
+    assert.equal(before.length, authScenario.additionalAssertions.expectedMigrationFiles.length + 2);
     for (const row of before) { assert.match(row.checksum, /^[0-9a-f]{64}$/); assert.ok(row.applied_at instanceof Date); }
     const second = await execute(process.execPath, ['server/migrate.ts'], options);
     assert.deepEqual(JSON.parse(second.stdout).applied, scenario.additionalAssertions.repeatMigrationExpectedApplied);
@@ -117,7 +117,7 @@ test('E03 A,A,B survive a fresh application; all Monitor lifecycle and CheckRun
     assert.deepEqual(after, before);
     const histories: CheckRun[] = [];
     for (const monitor of monitors) {
-      const history = await success<CheckRun[]>(await request(`/monitors/${monitor.id}/checks`));
+      const history = (await success<CheckHistoryPage>(await request(`/monitors/${monitor.id}/checks`))).items;
       assert.deepEqual(sortedById(history), sortedById(checks.filter((check) => check.monitorId === monitor.id)));
       histories.push(...history);
       const detail = await success<MonitorView>(await request(`/monitors/${monitor.id}`));
@@ -164,7 +164,7 @@ test('E03 A,A,B survive a fresh application; all Monitor lifecycle and CheckRun
     await failure(await request(`/checks/${scenario.additionalAssertions.malformedId}`), 400, 'INVALID_INPUT');
     const remaining = await success<MonitorView[]>(await request('/monitors'));
     assert.deepEqual(remaining, [updated]);
-    const aHistory = await success<CheckRun[]>(await request(`/monitors/${a.id}/checks`));
+    const aHistory = (await success<CheckHistoryPage>(await request(`/monitors/${a.id}/checks`))).items;
     assert.deepEqual(sortedById(aHistory), sortedById(checks.slice(0, 2)));
     const counts = (await pool.query(`SELECT (SELECT count(*)::int FROM monitors) AS monitors,
       (SELECT count(*)::int FROM check_runs) AS checks,
@@ -256,7 +256,7 @@ test('E05 upgrades existing rows only with explicit owner designation and preser
     finally { await oldApp.close(); }
 
     // Deliberately designate the second account, never the first row returned.
-    assert.deepEqual(await migrate(config, users[1].username), ['004_monitor_ownership.sql']);
+    assert.deepEqual(await migrate(config, users[1].username), ['004_monitor_ownership.sql', '005_check_history_index.sql']);
     const owner = (await pool.query('SELECT id FROM users WHERE username = $1', [users[1].username])).rows[0].id;
     const after = (await pool.query('SELECT * FROM monitors ORDER BY id')).rows;
     assert.deepEqual(after.map(({ owner_user_id: _owner, ...row }) => row), before.monitors);
diff --git a/test/storage-history.test.ts b/test/storage-history.test.ts
new file mode 100644
index 0000000..28cb14c
--- /dev/null
+++ b/test/storage-history.test.ts
@@ -0,0 +1,133 @@
+import { test } from 'node:test';
+import assert from 'node:assert/strict';
+import { createHash } from 'node:crypto';
+import { mkdir, readFile, writeFile } from 'node:fs/promises';
+import { buildApp } from '../server/app.ts';
+import { databasePool } from '../server/database.ts';
+import type { CheckHistoryPage } from '../server/model.ts';
+import { dropTestSchema, resetTestSchema } from './database.ts';
+import { authenticatedInject, loginForTest, prepareTestUsers } from './auth.ts';
+import { insertFixedCheck, insertHistoryFixture, newerCheck, originalChecks } from '../evidence/E07/fixture.ts';
+import scenario from '../evidence/E07/scenario.json' with { type: 'json' };
+
+test('E07 bounded owner history has stable ties, cursor validation, filters and insertion-safe continuation', async () => {
+  const config = await resetTestSchema(scenario.integrationSchema);
+  const pool = databasePool(config);
+  const app = buildApp(scenario.fixtureOrigin, config);
+  try {
+    const users = await prepareTestUsers(config);
+    await insertHistoryFixture(pool);
+    const alice = authenticatedInject(app, await loginForTest(app, users[0]));
+    const bob = authenticatedInject(app, await loginForTest(app, users[1]));
+    const path = `/monitors/${scenario.monitor.id}/checks`;
+    const expected = [...originalChecks].reverse();
+    const ids = (page: CheckHistoryPage) => page.items.map((row) => row.id);
+    async function page(query = ''): Promise<CheckHistoryPage> {
+      const response = await alice(`${path}${query ? `?${query}` : ''}`);
+      assert.equal(response.statusCode, 200);
+      const body = response.json();
+      assert.deepEqual(Object.keys(body), ['data']);
+      assert.deepEqual(Object.keys(body.data).sort(), ['items', 'limit', 'nextCursor', 'state']);
+      assert.ok(body.data.items.length <= body.data.limit);
+      return body.data;
+    }
+    async function next(cursor: string, state?: string) {
+      return page(new URLSearchParams({ limit: '3', ...(state ? { state } : {}), cursor }).toString());
+    }
+    async function continueFrom(first: CheckHistoryPage) {
+      const rows = [...first.items];
+      let current = first;
+      for (let remaining = 2; current.nextCursor !== null; remaining--) {
+        assert.ok(remaining > 0, 'Seven fixed rows must finish within three size-3 pages.');
+        current = await next(current.nextCursor);
+        rows.push(...current.items);
+      }
+      return rows;
+    }
+
+    const first = await page('limit=3');
+    assert.equal(first.limit, scenario.page.size);
+    assert.equal(first.state, null);
+    assert.deepEqual(first.items, expected.slice(0, 3));
+    assert.equal(typeof first.nextCursor, 'string');
+    const traversal = await continueFrom(first);
+    assert.deepEqual(traversal, expected);
+    assert.equal(new Set(traversal.map((row) => row.id)).size, 7);
+
+    const failed = await page('limit=3&state=FAILED');
+    assert.deepEqual(failed.items, expected.filter((row) => row.state === 'FAILED'));
+    assert.equal(failed.items.length, 3);
+    assert.equal(failed.state, 'FAILED');
+    assert.equal(failed.nextCursor, null, 'An exactly full final page has no continuation.');
+    const succeeded = await page('limit=3&state=SUCCEEDED');
+    assert.equal(typeof succeeded.nextCursor, 'string');
+    const succeededLast = await next(succeeded.nextCursor!, 'SUCCEEDED');
+    assert.deepEqual([...succeeded.items, ...succeededLast.items], expected.filter((row) => row.state === 'SUCCEEDED'));
+    assert.equal(succeededLast.nextCursor, null);
+
+    const defaultPage = await page();
+    assert.equal(defaultPage.limit, scenario.page.defaultSize);
+    assert.deepEqual(defaultPage.items, expected);
+    assert.equal(defaultPage.nextCursor, null);
+    const maximumPage = await page('limit=100');
+    assert.equal(maximumPage.limit, scenario.page.maximumSize);
+    assert.deepEqual(maximumPage.items, expected);
+    const minimumPage = await page('limit=1');
+    assert.deepEqual(minimumPage.items, expected.slice(0, 1));
+    assert.equal(typeof minimumPage.nextCursor, 'string');
+
+    const decoded = JSON.parse(Buffer.from(first.nextCursor!, 'base64url').toString('utf8'));
+    const encode = (value: unknown) => Buffer.from(JSON.stringify(value)).toString('base64url');
+    const endCursor = encode({ ...decoded, id: scenario.runs[0].id });
+    assert.deepEqual(await next(endCursor), { items: [], nextCursor: null, limit: 3, state: null });
+    const invalidQueries = [
+      ...scenario.page.invalidLimits.map((limit) => new URLSearchParams({ limit }).toString()),
+      ...scenario.page.invalidStates.map((state) => new URLSearchParams({ state }).toString()),
+      'limit=3&limit=3', 'state=FAILED&state=SUCCEEDED', 'sort=id',
+      ...['!', '', 'e30', `${first.nextCursor}=`, 'a'.repeat(513), encode(null), encode({ ...decoded, version: 2 }),
+        encode({ ...decoded, id: 'bad-id' }), encode({ ...decoded, finishedAt: '2026-02-30T00:00:00.000Z' })]
+        .map((cursor) => new URLSearchParams({ limit: '3', cursor }).toString()),
+      new URLSearchParams({ limit: '3', state: 'FAILED', cursor: first.nextCursor! }).toString(),
+      new URLSearchParams({ limit: '1', cursor: first.nextCursor! }).toString(),
+    ];
+    for (const query of invalidQueries) {
+      const response = await alice(`${path}?${query}`);
+      assert.equal(response.statusCode, scenario.page.invalidStatus);
+      assert.equal(response.json().error.code, scenario.page.invalidCode);
+    }
+    const mismatched = await alice(`/monitors/${scenario.runs[0].id}/checks?${new URLSearchParams({ limit: '3', cursor: first.nextCursor! })}`);
+    assert.equal(mismatched.statusCode, 400);
+    assert.equal(mismatched.json().error.code, 'INVALID_INPUT');
+    assert.equal((await app.inject(`${path}?limit=3`)).statusCode, 401);
+    const foreign = await bob(`${path}?limit=3&cursor=${first.nextCursor}`);
+    const absent = await bob(`/monitors/${scenario.runs[0].id}/checks?limit=3`);
+    assert.equal(foreign.statusCode, 404);
+    assert.equal(absent.statusCode, 404);
+    assert.deepEqual(foreign.json(), absent.json());
+
+    // Use the same first page and insert only the predeclared newer eighth row.
+    await insertFixedCheck(pool, newerCheck);
+    const afterInsert = await continueFrom(first);
+    assert.deepEqual(afterInsert, expected);
+    assert.equal(new Set(afterInsert.map((row) => row.id)).size, 7);
+    assert.ok(afterInsert.every((row) => row.id !== newerCheck.id));
+    assert.deepEqual((await page('limit=3')).items, [newerCheck, ...expected.slice(0, 2)]);
+    assert.deepEqual((await page('limit=3&state=FAILED')).items, failed.items);
+    assert.equal((await pool.query('SELECT count(*)::int AS count FROM check_runs')).rows[0].count, 8);
+    const index = (await pool.query<{ indexdef: string }>(`SELECT indexdef FROM pg_indexes
+      WHERE schemaname = current_schema() AND tablename = 'check_runs' AND indexname = $1`, ['check_runs_monitor_finished_id_idx'])).rows;
+    assert.equal(index.length, 1);
+    assert.match(index[0].indexdef, /USING btree \(monitor_id, finished_at DESC, id DESC\)$/);
+
+    const hashes: Record<string, string> = {};
+    for (const file of ['scenario.json', 'fixture.ts']) hashes[file] = createHash('sha256').update(await readFile(`evidence/E07/${file}`)).digest('hex');
+    await mkdir('output/e07', { recursive: true });
+    await writeFile('output/e07/history.json', JSON.stringify({
+      hashes, result: 'PASS', originalIds: traversal.map((row) => row.id), firstPageIds: ids(first),
+      failedIds: ids(failed), defaultLimit: defaultPage.limit, maximumLimit: maximumPage.limit,
+      invalidRequests: invalidQueries.length + 1, insertionContinuationIds: afterInsert.map((row) => row.id),
+      cursorConditionMismatchRejected: true, anonymousStatus: 401, foreignStatus: 404,
+      finalRowCount: 8, index: index[0].indexdef, loadRuns: 0, automaticRetries: 0,
+    }, null, 2) + '\n');
+  } finally { await app.close(); await pool.end(); await dropTestSchema(config.schema); }
+});
diff --git a/test/unit.test.ts b/test/unit.test.ts
index 2249b80..2defd10 100644
--- a/test/unit.test.ts
+++ b/test/unit.test.ts
@@ -9,6 +9,8 @@ import authScenario from '../evidence/E04/scenario.json' with { type: 'json' };
 import { hashPassword, verifyPassword, SCRYPT_OPTIONS } from '../server/password.ts';
 import { loginInput } from '../server/contracts.ts';
 import { csrfTokenForSession, validCsrfToken, SESSION_COOKIE_NAME, SESSION_TTL_MS, sessionTokenFromCookie } from '../server/auth.ts';
+import { DEFAULT_HISTORY_LIMIT, MAX_HISTORY_LIMIT, MAX_HISTORY_CURSOR_LENGTH, historyCursor, historyQuery } from '../server/history.ts';
+import historyScenario from '../evidence/E07/scenario.json' with { type: 'json' };
 
 test('a path on the configured fixture origin is allowed', () => {
   assert.equal(fixtureUrl(`${DEFAULT_FIXTURE_ORIGIN}/ok`, DEFAULT_FIXTURE_ORIGIN).pathname, '/ok');
@@ -122,3 +124,53 @@ test('E05 browser recognizes FORBIDDEN only with HTTP 403 and ignores server pro
   await assert.rejects(responseData(Response.json({ error: { code: 'FORBIDDEN', message: 'ignored' } }, { status: 401 })),
     (error: unknown) => failureCode(error) === 'INTERNAL_ERROR');
 });
+
+test('E07 history input has a finite default, maximum and terminal-state filter', () => {
+  const id = historyScenario.monitor.id;
+  assert.equal(DEFAULT_HISTORY_LIMIT, historyScenario.page.defaultSize);
+  assert.equal(MAX_HISTORY_LIMIT, historyScenario.page.maximumSize);
+  assert.equal(MAX_HISTORY_CURSOR_LENGTH, historyScenario.page.cursorMaximumLength);
+  assert.deepEqual(historyQuery(id, {}), { monitorId: id, limit: 20, state: null, after: null });
+  for (const limit of ['1', '3', '20', '100']) {
+    for (const state of historyScenario.page.filters) {
+      assert.deepEqual(historyQuery(id, { limit, state }), { monitorId: id, limit: Number(limit), state, after: null });
+    }
+  }
+  for (const limit of historyScenario.page.invalidLimits) {
+    assert.throws(() => historyQuery(id, { limit }), { code: 'INVALID_INPUT' });
+  }
+  for (const state of historyScenario.page.invalidStates) {
+    assert.throws(() => historyQuery(id, { state }), { code: 'INVALID_INPUT' });
+  }
+  for (const value of [null, [], { limit: 3 }, { limit: ['3', '3'] }, { state: ['FAILED', 'SUCCEEDED'] }, { sort: 'id' }]) {
+    assert.throws(() => historyQuery(id, value), { code: 'INVALID_INPUT' });
+  }
+});
+
+test('E07 cursor preserves the timestamp/id tuple and rejects changed conditions', () => {
+  const id = historyScenario.monitor.id;
+  const conditions = historyQuery(id, { limit: '3', state: 'SUCCEEDED' });
+  const after = { id: historyScenario.runs[4].id, finishedAt: historyScenario.orderingTimestamp };
+  const cursor = historyCursor(conditions, after);
+  assert.ok(cursor.length <= MAX_HISTORY_CURSOR_LENGTH);
+  assert.deepEqual(historyQuery(id, { limit: '3', state: 'SUCCEEDED', cursor }), { ...conditions, after });
+  for (const input of [{ limit: '3', cursor }, { limit: '3', state: 'FAILED', cursor }, { limit: '1', state: 'SUCCEEDED', cursor }]) {
+    assert.throws(() => historyQuery(id, input), { code: 'INVALID_INPUT' });
+  }
+  assert.throws(() => historyQuery(historyScenario.runs[0].id, { limit: '3', state: 'SUCCEEDED', cursor }), { code: 'INVALID_INPUT' });
+});
+
+test('E07 cursor rejects malformed, oversized, noncanonical and invalid typed boundaries', () => {
+  const id = historyScenario.monitor.id;
+  const cursor = historyCursor(historyQuery(id, { limit: '3' }), { id: historyScenario.runs[0].id, finishedAt: historyScenario.orderingTimestamp });
+  const decoded = JSON.parse(Buffer.from(cursor, 'base64url').toString('utf8'));
+  const malformed = ['', '!', 'e30', `${cursor}=`, 'a'.repeat(513), [cursor, cursor]];
+  for (const data of [null, [], {}, { ...decoded, version: 2 }, { ...decoded, id: 'bad-id' },
+    { ...decoded, finishedAt: '2026-13-28T00:00:00.000Z' }, { ...decoded, finishedAt: '2026-02-30T00:00:00.000Z' },
+    { ...decoded, extra: true }, { ...decoded, limit: '3' }]) {
+    malformed.push(Buffer.from(JSON.stringify(data)).toString('base64url'));
+  }
+  for (const invalid of malformed) {
+    assert.throws(() => historyQuery(id, { limit: '3', cursor: invalid }), { code: 'INVALID_INPUT' });
+  }
+});


