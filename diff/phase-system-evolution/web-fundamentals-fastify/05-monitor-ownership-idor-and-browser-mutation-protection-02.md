## `Enforce Monitor ownership in PostgreSQL access`

diff --git a/TRACK.md b/TRACK.md
index e894872..514358e 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -293,3 +293,45 @@ It also checks 40 protected-route rejections across absent, unknown, expired and
 revoked credentials. The integration gate retains all E03 cases and rejects ten
 fixed auth schema mutations before listening. Safe status/flag/boolean evidence,
 the unchanged-START anonymous baseline, counts and durations are in `evidence/E04/`.
+
+## E05 Monitor ownership
+
+Migration `004_monitor_ownership.sql` adds the required UUID `owner_user_id` and
+its user foreign key. Creation uses only the authenticated user, never a body
+field. Collection, detail, update, delete, manual check and both nested/direct
+CheckRun reads carry that owner in their SQL predicates. A foreign ID and a
+nonexistent ID return the same per-route `404 NOT_FOUND` response. CheckRun
+insertion repeats the parent-owner condition; a foreign check request never
+reaches the outbound fixture. The public Monitor wire fields are unchanged.
+
+### Upgrading a schema with existing Monitors
+
+Stop the old API before migration and retain a database backup. E04 did not
+record owners, so ownership cannot safely be guessed from names or the first
+user row. Fresh empty schemas migrate without any owner designation. For a
+schema containing Monitors, absent or unknown `MIGRATION_OWNER_USERNAME` causes
+the entire migration transaction to roll back: original rows, CheckRuns and
+migration history remain intact, and the new API refuses that incomplete schema.
+
+An operator must verify that **every existing Monitor belongs to one explicitly
+selected existing account** before running, for example:
+
+```sh
+MIGRATION_OWNER_USERNAME=alice-e04 npm run db:migrate
+```
+
+The value is the nonsecret username of an already prepared E04 account. It is
+passed as a bound SQL value through a transaction-local setting; it is not a
+password or an automatic account bootstrap. To reproduce in isolation, the
+existing `prepareTestUsers(config)` fixture prepares both accounts on schema
+003; `test/persistence.test.ts` then designates Bob, the second account, and
+proves both old Monitors and CheckRuns survive and are visible only to Bob.
+If the operator cannot establish that all old rows have the same rightful owner,
+**do not designate one or restart the old API**: leave the preserved database
+offline until a reviewed per-row ownership migration is available. This version
+does not invent missing ownership or expose orphan rows. Deleting an owner with
+Monitors is restricted by the foreign key. Migrations 001–003 stay unchanged.
+
+The E05 frozen two-user dataset and one unchanged-START read are in `evidence/E05/`.
+The regression compares all authoritative Monitor/CheckRun rows and fixture call
+counts after every foreign mutation, and tests both directions of access.
diff --git a/server/app.ts b/server/app.ts
index 7e6be67..12b53c9 100644
--- a/server/app.ts
+++ b/server/app.ts
@@ -20,7 +20,7 @@ const inputErrors = [
   errorCodes.FST_ERR_BAD_URL,
 ];
 
-const monitorViewSql = `SELECT m.id, m.name, m.url, m.interval_seconds, m.enabled, m.created_at, m.updated_at,
+const monitorViewSql = `SELECT m.id, m.owner_user_id, m.name, m.url, m.interval_seconds, m.enabled, m.created_at, m.updated_at,
   c.id AS check_id, c.monitor_id, c.trigger, c.state, c.http_status, c.latency_ms, c.failure_reason, c.started_at, c.finished_at
   FROM monitors m LEFT JOIN LATERAL (
     SELECT id, monitor_id, trigger, state, http_status, latency_ms, failure_reason, started_at, finished_at
@@ -53,13 +53,13 @@ export function buildApp(fixtureOrigin = DEFAULT_FIXTURE_ORIGIN, database: Datab
   registerAuthentication(app, pool, now);
 
   app.get('/health', async () => ({ data: { status: 'ok' } }));
-  app.get('/monitors', async () => {
-    const result = await pool.query<MonitorViewRow>(`${monitorViewSql} ORDER BY m.created_at, m.id`);
+  app.get('/monitors', async (request) => {
+    const result = await pool.query<MonitorViewRow>(`${monitorViewSql} WHERE m.owner_user_id = $1 ORDER BY m.created_at, m.id`, [request.user!.id]);
     return { data: result.rows.map(monitorViewFromRow) };
   });
 
   app.get<{ Params: { id: string } }>('/monitors/:id', async (request) => {
-    const result = await pool.query<MonitorViewRow>(`${monitorViewSql} WHERE m.id = $1`, [monitorId(request.params.id)]);
+    const result = await pool.query<MonitorViewRow>(`${monitorViewSql} WHERE m.id = $1 AND m.owner_user_id = $2`, [monitorId(request.params.id), request.user!.id]);
     if (!result.rows[0]) throw new ApiError('NOT_FOUND', 'Monitor not found.');
     return { data: monitorViewFromRow(result.rows[0]) };
   });
@@ -72,8 +72,8 @@ export function buildApp(fixtureOrigin = DEFAULT_FIXTURE_ORIGIN, database: Datab
         id: randomUUID(), ...input, createdAt: now, updatedAt: now,
       };
       const result = await pool.query<MonitorRow>(`INSERT INTO monitors
-        (id, name, url, interval_seconds, enabled, created_at, updated_at)
-        VALUES ($1, $2, $3, $4, $5, $6, $7) RETURNING *`, monitorToValues(monitor));
+        (id, name, url, interval_seconds, enabled, created_at, updated_at, owner_user_id)
+        VALUES ($1, $2, $3, $4, $5, $6, $7, $8) RETURNING *`, [...monitorToValues(monitor), request.user!.id]);
       return reply.code(201).send({ data: { ...monitorFromRow(result.rows[0]), latestCheck: null } });
     },
   );
@@ -82,40 +82,45 @@ export function buildApp(fixtureOrigin = DEFAULT_FIXTURE_ORIGIN, database: Datab
     const id = monitorId(request.params.id);
     const input = monitorInput(request.body, fixtureOrigin);
     const result = await pool.query<MonitorRow>(`UPDATE monitors SET name = $2, url = $3,
-      interval_seconds = $4, enabled = $5, updated_at = $6 WHERE id = $1 RETURNING *`,
-    [id, input.name, input.url, input.interval, input.enabled, new Date()]);
+      interval_seconds = $4, enabled = $5, updated_at = $6 WHERE id = $1 AND owner_user_id = $7 RETURNING *`,
+    [id, input.name, input.url, input.interval, input.enabled, new Date(), request.user!.id]);
     if (!result.rows[0]) throw new ApiError('NOT_FOUND', 'Monitor not found.');
-    const latest = await pool.query<CheckRunRow>('SELECT * FROM check_runs WHERE monitor_id = $1 ORDER BY finished_at DESC, id DESC LIMIT 1', [id]);
+    const latest = await pool.query<CheckRunRow>(`SELECT c.* FROM check_runs c JOIN monitors m ON m.id = c.monitor_id
+      WHERE m.id = $1 AND m.owner_user_id = $2 ORDER BY c.finished_at DESC, c.id DESC LIMIT 1`, [id, request.user!.id]);
     return { data: { ...monitorFromRow(result.rows[0]), latestCheck: latest.rows[0] ? checkRunFromRow(latest.rows[0]) : null } };
   });
 
   app.delete<{ Params: { id: string } }>('/monitors/:id', async (request) => {
     // PostgreSQL's FK cascade removes every CheckRun in the same statement.
-    const result = await pool.query<{ id: string }>('DELETE FROM monitors WHERE id = $1 RETURNING id', [monitorId(request.params.id)]);
+    const result = await pool.query<{ id: string }>('DELETE FROM monitors WHERE id = $1 AND owner_user_id = $2 RETURNING id', [monitorId(request.params.id), request.user!.id]);
     if (!result.rows[0]) throw new ApiError('NOT_FOUND', 'Monitor not found.');
     return { data: { id: result.rows[0].id } };
   });
 
   app.post<{ Params: { id: string } }>('/monitors/:id/checks', async (request) => {
-    const stored = await pool.query<MonitorRow>('SELECT * FROM monitors WHERE id = $1', [monitorId(request.params.id)]);
+    const stored = await pool.query<MonitorRow>('SELECT * FROM monitors WHERE id = $1 AND owner_user_id = $2', [monitorId(request.params.id), request.user!.id]);
     if (!stored.rows[0]) throw new ApiError('NOT_FOUND', 'Monitor not found.');
     const result = await checkMonitor(monitorFromRow(stored.rows[0]), fixtureOrigin);
     const saved = await pool.query<CheckRunRow>(`INSERT INTO check_runs
       (id, monitor_id, trigger, state, http_status, latency_ms, failure_reason, started_at, finished_at)
-      VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9) RETURNING *`, checkRunToValues(result));
+      SELECT $1, $2, $3, $4, $5, $6, $7, $8, $9 FROM monitors
+      WHERE id = $2 AND owner_user_id = $10 RETURNING *`, [...checkRunToValues(result), request.user!.id]);
+    if (!saved.rows[0]) throw new ApiError('NOT_FOUND', 'Monitor not found.');
     return { data: checkRunFromRow(saved.rows[0]) };
   });
 
   app.get<{ Params: { id: string } }>('/monitors/:id/checks', async (request) => {
     const id = monitorId(request.params.id);
-    const parent = await pool.query('SELECT id FROM monitors WHERE id = $1', [id]);
+    const parent = await pool.query('SELECT id FROM monitors WHERE id = $1 AND owner_user_id = $2', [id, request.user!.id]);
     if (!parent.rows[0]) throw new ApiError('NOT_FOUND', 'Monitor not found.');
-    const history = await pool.query<CheckRunRow>('SELECT * FROM check_runs WHERE monitor_id = $1 ORDER BY finished_at DESC, id DESC', [id]);
+    const history = await pool.query<CheckRunRow>(`SELECT c.* FROM check_runs c JOIN monitors m ON m.id = c.monitor_id
+      WHERE m.id = $1 AND m.owner_user_id = $2 ORDER BY c.finished_at DESC, c.id DESC`, [id, request.user!.id]);
     return { data: history.rows.map(checkRunFromRow) };
   });
 
   app.get<{ Params: { id: string } }>('/checks/:id', async (request) => {
-    const result = await pool.query<CheckRunRow>('SELECT * FROM check_runs WHERE id = $1', [monitorId(request.params.id)]);
+    const result = await pool.query<CheckRunRow>(`SELECT c.* FROM check_runs c JOIN monitors m ON m.id = c.monitor_id
+      WHERE c.id = $1 AND m.owner_user_id = $2`, [monitorId(request.params.id), request.user!.id]);
     if (!result.rows[0]) throw new ApiError('NOT_FOUND', 'CheckRun not found.');
     return { data: checkRunFromRow(result.rows[0]) };
   });
diff --git a/server/mapping.ts b/server/mapping.ts
index a475d5f..31ea438 100644
--- a/server/mapping.ts
+++ b/server/mapping.ts
@@ -3,7 +3,7 @@ import type { CheckRun, Monitor, MonitorView, User } from './model.ts';
 // PostgreSQL rows use snake_case, integer/boolean columns, and pg Date objects.
 // API/domain models use camelCase and canonical UTC ISO strings, never raw rows.
 export type MonitorRow = {
-  id: string; name: string; url: string; interval_seconds: number; enabled: boolean;
+  id: string; owner_user_id: string; name: string; url: string; interval_seconds: number; enabled: boolean;
   created_at: Date; updated_at: Date;
 };
 export type CheckRunRow = {
diff --git a/server/migrate.ts b/server/migrate.ts
index 91c3c91..43c640a 100644
--- a/server/migrate.ts
+++ b/server/migrate.ts
@@ -5,7 +5,7 @@ import { databaseConfig, databasePool, schemaIdentifier } from './database.ts';
 import type { DatabaseConfig } from './database.ts';
 
 export async function migrationFiles() {
-  const names = ['001_monitors.sql', '002_check_runs.sql', '003_sessions.sql'];
+  const names = ['001_monitors.sql', '002_check_runs.sql', '003_sessions.sql', '004_monitor_ownership.sql'];
   return Promise.all(names.map(async (version) => {
     const sql = await readFile(new URL(`./migrations/${version}`, import.meta.url), 'utf8');
     return { version, sql, checksum: createHash('sha256').update(sql).digest('hex') };
@@ -23,7 +23,7 @@ export function checkMigrationHistory(
   }
 }
 
-export async function migrate(config: DatabaseConfig = databaseConfig()): Promise<string[]> {
+export async function migrate(config: DatabaseConfig = databaseConfig(), ownerUsername = process.env.MIGRATION_OWNER_USERNAME ?? ''): Promise<string[]> {
   const migrations = await migrationFiles();
   const pool = databasePool(config);
   let client;
@@ -33,6 +33,7 @@ export async function migrate(config: DatabaseConfig = databaseConfig()): Promis
     await client.query('BEGIN');
     await client.query(`CREATE SCHEMA IF NOT EXISTS ${schemaIdentifier(config.schema)}`);
     await client.query(`SET LOCAL search_path TO ${schemaIdentifier(config.schema)}`);
+    await client.query("SELECT set_config('wse.migration_owner_username', $1, true)", [ownerUsername]);
     await client.query(`CREATE TABLE IF NOT EXISTS schema_migrations (
       version text PRIMARY KEY, checksum text NOT NULL, applied_at timestamptz(3) NOT NULL DEFAULT current_timestamp
     )`);
diff --git a/server/migrations/004_monitor_ownership.sql b/server/migrations/004_monitor_ownership.sql
new file mode 100644
index 0000000..fb06dad
--- /dev/null
+++ b/server/migrations/004_monitor_ownership.sql
@@ -0,0 +1,22 @@
+ALTER TABLE monitors ADD COLUMN owner_user_id uuid;
+
+-- Older schemas cannot infer a Monitor's owner. The operator must first verify
+-- that ALL existing rows belong to this explicitly designated, existing account.
+-- Failure rolls back this entire migration, including the new column.
+DO $$
+DECLARE
+  designated_owner uuid;
+BEGIN
+  IF EXISTS (SELECT 1 FROM monitors) THEN
+    SELECT id INTO designated_owner FROM users
+      WHERE username = current_setting('wse.migration_owner_username', true);
+    IF designated_owner IS NULL THEN
+      RAISE EXCEPTION 'Existing Monitors require an explicit existing MIGRATION_OWNER_USERNAME. Stop the old API and follow TRACK.md before retrying.';
+    END IF;
+    UPDATE monitors SET owner_user_id = designated_owner;
+  END IF;
+END $$;
+
+ALTER TABLE monitors ALTER COLUMN owner_user_id SET NOT NULL;
+ALTER TABLE monitors ADD CONSTRAINT monitors_owner_user_id_fkey
+  FOREIGN KEY (owner_user_id) REFERENCES users(id) ON DELETE RESTRICT;
diff --git a/server/schema.ts b/server/schema.ts
index 2eba2ff..28a026a 100644
--- a/server/schema.ts
+++ b/server/schema.ts
@@ -3,7 +3,7 @@ import { checkMigrationHistory, migrationFiles } from './migrate.ts';
 
 const expectedColumns = {
   monitors: {
-    id: ['uuid', 'NO'], name: ['text', 'NO'], url: ['text', 'NO'],
+    id: ['uuid', 'NO'], owner_user_id: ['uuid', 'NO'], name: ['text', 'NO'], url: ['text', 'NO'],
     interval_seconds: ['int4', 'NO'], enabled: ['bool', 'NO'],
     created_at: ['timestamptz', 'NO'], updated_at: ['timestamptz', 'NO'],
   },
@@ -64,5 +64,8 @@ export async function verifySchema(pool: Pool): Promise<void> {
     if (!keys.rows.some((row) => row.table_name === 'sessions' && row.definition === 'FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE')) {
       throw new Error('Incompatible database schema: session user deletion rule.');
     }
+    if (!keys.rows.some((row) => row.table_name === 'monitors' && row.definition === 'FOREIGN KEY (owner_user_id) REFERENCES users(id) ON DELETE RESTRICT')) {
+      throw new Error('Incompatible database schema: Monitor owner deletion rule.');
+    }
   } finally { client.release(); }
 }
diff --git a/test/database.ts b/test/database.ts
index 1df6eeb..47e11a3 100644
--- a/test/database.ts
+++ b/test/database.ts
@@ -2,7 +2,7 @@ import { databaseConfig, databasePool, schemaIdentifier } from '../server/databa
 import { migrate } from '../server/migrate.ts';
 
 export function testDatabaseConfig(schema: string) {
-  if (!/^e0[34]_[a-z_]+$/.test(schema)) throw new Error('Test cleanup is restricted to an explicit e03_ or e04_ schema.');
+  if (!/^e0[345]_[a-z_]+$/.test(schema)) throw new Error('Test cleanup is restricted to an explicit e03_, e04_ or e05_ schema.');
   return { ...databaseConfig(), schema };
 }
 
diff --git a/test/persistence.test.ts b/test/persistence.test.ts
index 25791c1..175bab1 100644
--- a/test/persistence.test.ts
+++ b/test/persistence.test.ts
@@ -2,15 +2,18 @@ import { test } from 'node:test';
 import assert from 'node:assert/strict';
 import { execFile } from 'node:child_process';
 import { promisify } from 'node:util';
+import { randomUUID } from 'node:crypto';
 import { mkdir, writeFile } from 'node:fs/promises';
 import { buildApp } from '../server/app.ts';
-import { databasePool } from '../server/database.ts';
+import { databasePool, schemaIdentifier } from '../server/database.ts';
 import { monitorFromRow, checkRunFromRow, monitorToValues, checkRunToValues } from '../server/mapping.ts';
 import type { MonitorRow, CheckRunRow } from '../server/mapping.ts';
 import type { CheckRun, Monitor, MonitorView } from '../server/model.ts';
 import { fixtureServer } from './fixture.ts';
 import { authenticatedInject, loginForTest, prepareTestUsers } from './auth.ts';
 import authScenario from '../evidence/E04/scenario.json' with { type: 'json' };
+import { migrate, migrationFiles } from '../server/migrate.ts';
+import ownershipScenario from '../evidence/E05/scenario.json' with { type: 'json' };
 import { dropTestSchema, resetTestSchema } from './database.ts';
 import scenario from '../evidence/E03/scenario.json' with { type: 'json' };
 
@@ -50,9 +53,9 @@ test('E03 migration CLI applies the fresh chain and a repeated command changes n
   try {
     const options = { cwd: process.cwd(), env: { ...process.env, DATABASE_URL: config.connectionString, DATABASE_SCHEMA: config.schema }, timeout: 10_000 };
     const first = await execute(process.execPath, ['server/migrate.ts'], options);
-    assert.deepEqual(JSON.parse(first.stdout).applied, authScenario.additionalAssertions.expectedMigrationFiles);
+    assert.deepEqual(JSON.parse(first.stdout).applied, [...authScenario.additionalAssertions.expectedMigrationFiles, '004_monitor_ownership.sql']);
     const before = (await pool.query('SELECT version, checksum, applied_at FROM schema_migrations ORDER BY version')).rows;
-    assert.equal(before.length, authScenario.additionalAssertions.expectedMigrationFiles.length);
+    assert.equal(before.length, authScenario.additionalAssertions.expectedMigrationFiles.length + 1);
     for (const row of before) { assert.match(row.checksum, /^[0-9a-f]{64}$/); assert.ok(row.applied_at instanceof Date); }
     const second = await execute(process.execPath, ['server/migrate.ts'], options);
     assert.deepEqual(JSON.parse(second.stdout).applied, scenario.additionalAssertions.repeatMigrationExpectedApplied);
@@ -188,8 +191,10 @@ test('E03 canonical rows round-trip UUID, timezone, boolean, integer, zero and n
   const monitor: Monitor = { id: input.id, name: input.name.trim(), url: input.url, interval: input.interval, enabled: input.enabled, createdAt: input.timestamp, updatedAt: input.timestamp };
   const check: CheckRun = { id: '00000000-0000-4000-8000-000000000102', monitorId: monitor.id, trigger: 'MANUAL', state: 'FAILED', httpStatus: null, latencyMs: input.latencyMs, failureReason: 'CONNECTION_FAILURE', startedAt: input.timestamp, finishedAt: input.timestamp };
   try {
-    const storedMonitor = (await pool.query<MonitorRow>(`INSERT INTO monitors (id, name, url, interval_seconds, enabled, created_at, updated_at)
-      VALUES ($1, $2, $3, $4, $5, $6, $7) RETURNING *`, monitorToValues(monitor))).rows[0];
+    const credentials = (await prepareTestUsers(config))[0];
+    const owner = (await pool.query('SELECT id FROM users WHERE username = $1', [credentials.username])).rows[0].id;
+    const storedMonitor = (await pool.query<MonitorRow>(`INSERT INTO monitors (id, name, url, interval_seconds, enabled, created_at, updated_at, owner_user_id)
+      VALUES ($1, $2, $3, $4, $5, $6, $7, $8) RETURNING *`, [...monitorToValues(monitor), owner])).rows[0];
     const storedCheck = (await pool.query<CheckRunRow>(`INSERT INTO check_runs (id, monitor_id, trigger, state, http_status, latency_ms, failure_reason, started_at, finished_at)
       VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9) RETURNING *`, checkRunToValues(check))).rows[0];
     assert.ok(storedMonitor.created_at instanceof Date);
@@ -202,7 +207,7 @@ test('E03 canonical rows round-trip UUID, timezone, boolean, integer, zero and n
     const mappedCheck = checkRunFromRow(storedCheck);
     assert.deepEqual(mappedMonitor, { ...monitor, createdAt: input.expectedTimestamp, updatedAt: input.expectedTimestamp });
     assert.deepEqual(mappedCheck, { ...check, startedAt: input.expectedTimestamp, finishedAt: input.expectedTimestamp });
-    const inject = authenticatedInject(app, await loginForTest(app, (await prepareTestUsers(config))[0]));
+    const inject = authenticatedInject(app, await loginForTest(app, credentials));
     const response = await inject(`/monitors/${monitor.id}`);
     assert.equal(response.statusCode, 200);
     assert.deepEqual(response.json(), { data: { ...mappedMonitor, latestCheck: mappedCheck } });
@@ -210,3 +215,70 @@ test('E03 canonical rows round-trip UUID, timezone, boolean, integer, zero and n
     await record('mapping', { monitor: mappedMonitor, check: mappedCheck, sqlTimestampInput: input.timestamp, sqlTimestampRuntimeType: storedMonitor.created_at.constructor.name, durationMs: Math.round(performance.now() - started) });
   } finally { await app.close(); await pool.end(); await dropTestSchema(config.schema); }
 });
+
+test('E05 upgrades existing rows only with explicit owner designation and preserves the prior schema on refusal', async () => {
+  const started = performance.now();
+  const config = await resetTestSchema('e05_upgrade', false);
+  const pool = databasePool(config);
+  const migrations = await migrationFiles();
+  try {
+    await pool.query(`CREATE SCHEMA ${schemaIdentifier(config.schema)}`);
+    await pool.query('CREATE TABLE schema_migrations (version text PRIMARY KEY, checksum text NOT NULL, applied_at timestamptz(3) NOT NULL DEFAULT current_timestamp)');
+    for (const migration of migrations.slice(0, 3)) {
+      await pool.query(migration.sql);
+      await pool.query('INSERT INTO schema_migrations (version, checksum) VALUES ($1, $2)', [migration.version, migration.checksum]);
+    }
+    const users = await prepareTestUsers(config);
+    for (let i = 0; i < ownershipScenario.monitors.length; i++) {
+      const input = ownershipScenario.monitors[i];
+      const monitor: Monitor = { id: randomUUID(), ...input, createdAt: authScenario.session.clockStart, updatedAt: authScenario.session.clockStart };
+      await pool.query('INSERT INTO monitors VALUES ($1, $2, $3, $4, $5, $6, $7)', monitorToValues(monitor));
+      const check: CheckRun = { id: randomUUID(), monitorId: monitor.id, trigger: 'MANUAL',
+        state: i === 0 ? 'SUCCEEDED' : 'FAILED', httpStatus: i === 0 ? 200 : 503,
+        latencyMs: 0, failureReason: i === 0 ? null : 'HTTP_STATUS', startedAt: monitor.createdAt, finishedAt: monitor.createdAt };
+      await pool.query('INSERT INTO check_runs VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)', checkRunToValues(check));
+    }
+    const before = {
+      monitors: (await pool.query('SELECT * FROM monitors ORDER BY id')).rows,
+      checks: (await pool.query('SELECT * FROM check_runs ORDER BY id')).rows,
+      migrations: (await pool.query('SELECT * FROM schema_migrations ORDER BY version')).rows,
+    };
+    for (const designation of ['', authScenario.unknownUsername]) {
+      await assert.rejects(migrate(config, designation), /Existing Monitors require an explicit existing MIGRATION_OWNER_USERNAME/);
+      assert.deepEqual((await pool.query('SELECT * FROM monitors ORDER BY id')).rows, before.monitors);
+      assert.deepEqual((await pool.query('SELECT * FROM check_runs ORDER BY id')).rows, before.checks);
+      assert.deepEqual((await pool.query('SELECT * FROM schema_migrations ORDER BY version')).rows, before.migrations);
+      assert.equal((await pool.query(`SELECT column_name FROM information_schema.columns
+        WHERE table_schema = current_schema() AND table_name = 'monitors' AND column_name = 'owner_user_id'`)).rowCount, 0);
+    }
+    const oldApp = buildApp(undefined, config);
+    try { await assert.rejects(oldApp.listen({ host: '127.0.0.1', port: 4312 }), /migration history/); assert.equal(oldApp.server.listening, false); }
+    finally { await oldApp.close(); }
+
+    // Deliberately designate the second account, never the first row returned.
+    assert.deepEqual(await migrate(config, users[1].username), ['004_monitor_ownership.sql']);
+    const owner = (await pool.query('SELECT id FROM users WHERE username = $1', [users[1].username])).rows[0].id;
+    const after = (await pool.query('SELECT * FROM monitors ORDER BY id')).rows;
+    assert.deepEqual(after.map(({ owner_user_id: _owner, ...row }) => row), before.monitors);
+    assert.ok(after.every((row) => row.owner_user_id === owner));
+    assert.deepEqual((await pool.query('SELECT * FROM check_runs ORDER BY id')).rows, before.checks);
+    assert.deepEqual((await pool.query('SELECT * FROM schema_migrations ORDER BY version LIMIT 3')).rows, before.migrations);
+    await assert.rejects(pool.query('DELETE FROM users WHERE id = $1', [owner]), { code: '23503' });
+    const app = buildApp(undefined, config);
+    try {
+      for (let i = 0; i < users.length; i++) {
+        const inject = authenticatedInject(app, await loginForTest(app, users[i]));
+        const response = await inject('/monitors');
+        assert.equal(response.statusCode, 200);
+        assert.equal(response.json().data.length, i === 1 ? 2 : 0);
+      }
+    } finally { await app.close(); }
+    await mkdir('output/e05', { recursive: true });
+    await writeFile('output/e05/migration.json', JSON.stringify({
+      previousSchema: '001/002/003', refusedAbsentOwner: true, refusedUnknownOwner: true,
+      refusalAtomic: true, priorStartupRefused: true, selectedOwner: users[1].username,
+      firstUserNotAssumed: true, preservedMonitors: 2, preservedChecks: 2, preservedOldHistory: true,
+      onlyDesignatedOwnerCanRead: true, ownerDeletionRestricted: true, durationMs: Math.round(performance.now() - started),
+    }, null, 2) + '\n');
+  } finally { await pool.end(); await dropTestSchema(config.schema); }
+});
diff --git a/test/storage-schema.test.ts b/test/storage-schema.test.ts
index 0054235..7a6fe89 100644
--- a/test/storage-schema.test.ts
+++ b/test/storage-schema.test.ts
@@ -45,3 +45,26 @@ test('E04 authentication columns, nullability, types, precision, keys and user r
   await mkdir('output/e04', { recursive: true });
   await writeFile('output/e04/schema.json', JSON.stringify({ results, count: results.length, durationMs: Math.round(performance.now() - started) }, null, 2) + '\n');
 });
+
+test('E05 owner column and user foreign key are required before listening', async () => {
+  const mutations = [
+    ['missing_owner', 'ALTER TABLE monitors DROP COLUMN owner_user_id'],
+    ['nullable_owner', 'ALTER TABLE monitors ALTER COLUMN owner_user_id DROP NOT NULL'],
+    ['missing_owner_fk', 'ALTER TABLE monitors DROP CONSTRAINT monitors_owner_user_id_fkey'],
+    ['wrong_owner_type', 'ALTER TABLE monitors DROP CONSTRAINT monitors_owner_user_id_fkey; ALTER TABLE monitors ALTER COLUMN owner_user_id TYPE text'],
+  ];
+  const results = [];
+  for (const [name, sql] of mutations) {
+    const config = await resetTestSchema(`e05_${name}`);
+    const pool = databasePool(config);
+    const app = buildApp(undefined, config);
+    try {
+      await pool.query(sql);
+      await assert.rejects(app.listen({ host: '127.0.0.1', port: 4312 }), /Incompatible database schema/);
+      assert.equal(app.server.listening, false);
+      results.push({ mutation: name, startupRejected: true, listening: false });
+    } finally { await app.close(); await pool.end(); await dropTestSchema(config.schema); }
+  }
+  await mkdir('output/e05', { recursive: true });
+  await writeFile('output/e05/schema.json', JSON.stringify({ results, count: results.length }, null, 2) + '\n');
+});


