## `Reject unexpected PostgreSQL columns before API startup`

diff --git a/evidence/E03/supplemental-schema.json b/evidence/E03/supplemental-schema.json
new file mode 100644
index 0000000..408be03
--- /dev/null
+++ b/evidence/E03/supplemental-schema.json
@@ -0,0 +1,6 @@
+{
+  "startupRejected": true,
+  "listening": false,
+  "message": "Incompatible database schema: unexpected monitors.e03_required_extra.",
+  "durationMs": 76
+}
diff --git a/package.json b/package.json
index 461a2c4..dcbf7dc 100644
--- a/package.json
+++ b/package.json
@@ -20,7 +20,7 @@
     "db:up": "docker compose --project-name wse-fundamentals up -d --wait",
     "db:down": "docker compose --project-name wse-fundamentals down",
     "db:migrate": "node server/migrate.ts",
-    "test:integration": "node --test --test-concurrency=1 test/persistence.test.ts"
+    "test:integration": "node --test --test-concurrency=1 test/persistence.test.ts test/storage-*.test.ts"
   },
   "dependencies": {
     "fastify": "5.12.1",
diff --git a/server/schema.ts b/server/schema.ts
index ff6fe07..27f16b8 100644
--- a/server/schema.ts
+++ b/server/schema.ts
@@ -23,6 +23,12 @@ export async function verifySchema(pool: Pool): Promise<void> {
       table_name: string; column_name: string; udt_name: string; is_nullable: string; datetime_precision: number | null;
     }>(`SELECT table_name, column_name, udt_name, is_nullable, datetime_precision
       FROM information_schema.columns WHERE table_schema = current_schema() AND table_name IN ('monitors', 'check_runs')`);
+    for (const column of columns.rows) {
+      const fields = expectedColumns[column.table_name as keyof typeof expectedColumns];
+      if (!Object.hasOwn(fields, column.column_name)) {
+        throw new Error(`Incompatible database schema: unexpected ${column.table_name}.${column.column_name}.`);
+      }
+    }
     for (const [table, fields] of Object.entries(expectedColumns)) {
       for (const [name, [type, nullable]] of Object.entries(fields)) {
         const column = columns.rows.find((row) => row.table_name === table && row.column_name === name);
diff --git a/test/storage-schema.test.ts b/test/storage-schema.test.ts
new file mode 100644
index 0000000..4924945
--- /dev/null
+++ b/test/storage-schema.test.ts
@@ -0,0 +1,28 @@
+import { test } from 'node:test';
+import assert from 'node:assert/strict';
+import { mkdir, writeFile } from 'node:fs/promises';
+import { buildApp } from '../server/app.ts';
+import { databasePool } from '../server/database.ts';
+import { resetTestSchema, dropTestSchema } from './database.ts';
+import scenario from '../evidence/E03/supplemental-scenario.json' with { type: 'json' };
+
+test('E03 unexpected required Monitor column rejects startup before accepting valid requests', async () => {
+  const started = performance.now();
+  const input = scenario.extraRequiredColumn;
+  const config = await resetTestSchema(input.schema);
+  const pool = databasePool(config);
+  const app = buildApp(undefined, config);
+  try {
+    await pool.query(input.sql);
+    let message = '';
+    await assert.rejects(app.listen({ host: '127.0.0.1', port: 4312 }), (error: unknown) => {
+      assert.ok(error instanceof Error);
+      message = error.message;
+      assert.match(message, /Incompatible database schema: unexpected monitors\.e03_required_extra/);
+      return true;
+    });
+    assert.equal(app.server.listening, false);
+    await mkdir('output/e03', { recursive: true });
+    await writeFile('output/e03/supplemental-schema.json', JSON.stringify({ startupRejected: true, listening: false, message, durationMs: Math.round(performance.now() - started) }, null, 2) + '\n');
+  } finally { await app.close(); await pool.end(); await dropTestSchema(config.schema); }
+});


## `Classify unpersistable NUL Monitor names as invalid input`

diff --git a/evidence/E03/supplemental-contract.json b/evidence/E03/supplemental-contract.json
new file mode 100644
index 0000000..6424824
--- /dev/null
+++ b/evidence/E03/supplemental-contract.json
@@ -0,0 +1,13 @@
+{
+  "createStatus": 400,
+  "createBody": {
+    "error": {
+      "code": "INVALID_INPUT",
+      "message": "Name must contain 1–100 UTF-16 code units after trimming and no NUL character."
+    }
+  },
+  "createdCount": 0,
+  "updateStatus": 400,
+  "updateUnchanged": true,
+  "durationMs": 80
+}
diff --git a/server/contracts.ts b/server/contracts.ts
index ef23fc0..0428a43 100644
--- a/server/contracts.ts
+++ b/server/contracts.ts
@@ -18,8 +18,8 @@ export function monitorInput(value: unknown, fixtureOrigin: string): Pick<Monito
     throw new ApiError('INVALID_INPUT', 'A Monitor JSON object is required.');
   }
   const { name, url, interval, enabled } = value as Record<string, unknown>;
-  if (typeof name !== 'string' || name.trim().length < 1 || name.trim().length > 100) {
-    throw new ApiError('INVALID_INPUT', 'Name must contain 1–100 UTF-16 code units after trimming.');
+  if (typeof name !== 'string' || name.trim().length < 1 || name.trim().length > 100 || name.includes('\0')) {
+    throw new ApiError('INVALID_INPUT', 'Name must contain 1–100 UTF-16 code units after trimming and no NUL character.');
   }
   if (typeof interval !== 'number' || !Number.isInteger(interval) || interval < 1 || interval > 86_400) {
     throw new ApiError('INVALID_INPUT', 'Interval must be an integer from 1 to 86400 seconds.');
diff --git a/test/storage-contract.test.ts b/test/storage-contract.test.ts
new file mode 100644
index 0000000..0edd4f4
--- /dev/null
+++ b/test/storage-contract.test.ts
@@ -0,0 +1,31 @@
+import { test } from 'node:test';
+import assert from 'node:assert/strict';
+import { mkdir, writeFile } from 'node:fs/promises';
+import { buildApp } from '../server/app.ts';
+import { databasePool } from '../server/database.ts';
+import { resetTestSchema, dropTestSchema } from './database.ts';
+import scenario from '../evidence/E03/supplemental-scenario.json' with { type: 'json' };
+
+test('E03 NUL name is INVALID_INPUT on create/update without PostgreSQL mutation', async () => {
+  const started = performance.now();
+  const input = scenario.nulName;
+  const config = await resetTestSchema(input.schema);
+  const pool = databasePool(config);
+  const app = buildApp(undefined, config);
+  try {
+    const created = await app.inject({ method: 'POST', url: '/monitors', payload: input.input });
+    assert.equal(created.statusCode, input.expectedStatus);
+    assert.equal(created.json().error.code, input.expectedCode);
+    const createdCount = (await pool.query('SELECT count(*)::int AS count FROM monitors')).rows[0].count;
+    assert.equal(createdCount, input.expectedCreatedCount);
+    const valid = await app.inject({ method: 'POST', url: '/monitors', payload: scenario.extraRequiredColumn.probeInput });
+    assert.equal(valid.statusCode, 201);
+    const monitor = valid.json().data;
+    const updated = await app.inject({ method: 'PUT', url: `/monitors/${monitor.id}`, payload: input.input });
+    assert.equal(updated.statusCode, input.expectedStatus);
+    assert.equal(updated.json().error.code, input.expectedCode);
+    assert.deepEqual((await app.inject(`/monitors/${monitor.id}`)).json().data, monitor);
+    await mkdir('output/e03', { recursive: true });
+    await writeFile('output/e03/supplemental-contract.json', JSON.stringify({ createStatus: created.statusCode, createBody: created.json(), createdCount, updateStatus: updated.statusCode, updateUnchanged: true, durationMs: Math.round(performance.now() - started) }, null, 2) + '\n');
+  } finally { await app.close(); await pool.end(); await dropTestSchema(config.schema); }
+});


