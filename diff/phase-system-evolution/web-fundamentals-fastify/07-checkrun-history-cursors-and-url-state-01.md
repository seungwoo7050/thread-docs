# E07 CheckRun 이력, Cursor와 URL 상태

## `고정 이력에서 페이지 제한 누락을 재현`

diff --git a/evidence/E07/baseline-reproducer.mjs b/evidence/E07/baseline-reproducer.mjs
new file mode 100644
index 0000000..f326415
--- /dev/null
+++ b/evidence/E07/baseline-reproducer.mjs
@@ -0,0 +1,48 @@
+import assert from 'node:assert/strict';
+import { createHash } from 'node:crypto';
+import { execFileSync } from 'node:child_process';
+import { readFile, writeFile } from 'node:fs/promises';
+import { buildApp } from '../../server/app.ts';
+import { databaseConfig, databasePool, schemaIdentifier } from '../../server/database.ts';
+import { migrate } from '../../server/migrate.ts';
+import { authenticatedInject, loginForTest, prepareTestUsers } from '../../test/auth.ts';
+import { insertHistoryFixture } from './fixture.ts';
+import scenario from './scenario.json' with { type: 'json' };
+
+assert.equal(process.versions.node, '24.19.0');
+assert.equal(execFileSync('git', ['rev-parse', 'HEAD'], { encoding: 'utf8' }).trim(), scenario.start);
+assert.equal(execFileSync('git', ['diff', '--name-only', 'HEAD'], { encoding: 'utf8' }).trim(), '');
+const hashes = {};
+for (const file of ['scenario.json', 'fixture.ts']) {
+  hashes[file] = createHash('sha256').update(await readFile(new URL(file, import.meta.url))).digest('hex');
+}
+const config = { ...databaseConfig(), schema: scenario.baselineSchema };
+assert.match(config.schema, /^e07_[a-z_]+$/);
+const pool = databasePool(config);
+const app = buildApp(scenario.fixtureOrigin, config);
+let ownedSchema = false;
+try {
+  await pool.query(`CREATE SCHEMA ${schemaIdentifier(config.schema)}`);
+  ownedSchema = true;
+  await migrate(config);
+  const users = await prepareTestUsers(config);
+  await insertHistoryFixture(pool);
+  const inject = authenticatedInject(app, await loginForTest(app, users[0]));
+  const response = await inject(`/monitors/${scenario.monitor.id}/checks?limit=${scenario.page.size}`);
+  assert.equal(response.statusCode, 200);
+  const data = response.json().data;
+  const rows = Array.isArray(data) ? data : data.items;
+  const evidence = {
+    start: scenario.start, hashes, codeUnmodified: true, baselineRuns: 1,
+    requestedLimit: scenario.page.size, status: response.statusCode,
+    observedRows: rows.length, nextCursorPresent: typeof data.nextCursor === 'string',
+    originalIds: rows.map((row) => row.id),
+    result: rows.length > scenario.page.size || typeof data.nextCursor !== 'string' ? 'REPRODUCED' : 'NOT_REPRODUCED',
+  };
+  await writeFile(new URL('./baseline.json', import.meta.url), JSON.stringify(evidence, null, 2) + '\n');
+  console.log(JSON.stringify(evidence));
+} finally {
+  await app.close();
+  if (ownedSchema) await pool.query(`DROP SCHEMA ${schemaIdentifier(config.schema)} CASCADE`);
+  await pool.end();
+}
diff --git a/evidence/E07/baseline.json b/evidence/E07/baseline.json
new file mode 100644
index 0000000..cc9d475
--- /dev/null
+++ b/evidence/E07/baseline.json
@@ -0,0 +1,23 @@
+{
+  "start": "f9adb6423b6ff5e732f89e52c30f7ac383bb0b9e",
+  "hashes": {
+    "scenario.json": "069013e4ee0766c7fec016a1d00c41eae7bdc37fbbb49d8b46cc004325713dd0",
+    "fixture.ts": "3a7a9a22668ad2eb25c534ad51968859b7d70263d5089d566363955ac3779d51"
+  },
+  "codeUnmodified": true,
+  "baselineRuns": 1,
+  "requestedLimit": 3,
+  "status": 200,
+  "observedRows": 7,
+  "nextCursorPresent": false,
+  "originalIds": [
+    "07000000-0000-4000-8000-000000000007",
+    "07000000-0000-4000-8000-000000000006",
+    "07000000-0000-4000-8000-000000000005",
+    "07000000-0000-4000-8000-000000000004",
+    "07000000-0000-4000-8000-000000000003",
+    "07000000-0000-4000-8000-000000000002",
+    "07000000-0000-4000-8000-000000000001"
+  ],
+  "result": "REPRODUCED"
+}
diff --git a/evidence/E07/fixture.ts b/evidence/E07/fixture.ts
new file mode 100644
index 0000000..0bfd70f
--- /dev/null
+++ b/evidence/E07/fixture.ts
@@ -0,0 +1,31 @@
+import type { Pool } from 'pg';
+import type { CheckRun, Monitor } from '../../server/model.ts';
+import { checkRunToValues, monitorToValues } from '../../server/mapping.ts';
+import scenario from './scenario.json' with { type: 'json' };
+
+export const originalChecks: CheckRun[] = scenario.runs.map((run) => ({
+  id: run.id, monitorId: scenario.monitor.id, trigger: 'MANUAL', state: run.state as CheckRun['state'],
+  httpStatus: run.state === 'SUCCEEDED' ? scenario.terminalFields.successStatus : scenario.terminalFields.failedStatus,
+  latencyMs: scenario.terminalFields.latencyMs, failureReason: run.state === 'SUCCEEDED' ? null : 'HTTP_STATUS',
+  startedAt: scenario.orderingTimestamp, finishedAt: scenario.orderingTimestamp,
+}));
+
+export const newerCheck: CheckRun = {
+  ...originalChecks[0], id: scenario.newerRun.id,
+  startedAt: scenario.newerRun.timestamp, finishedAt: scenario.newerRun.timestamp,
+};
+
+export async function insertFixedCheck(pool: Pool, check: CheckRun) {
+  await pool.query(`INSERT INTO check_runs
+    (id, monitor_id, trigger, state, http_status, latency_ms, failure_reason, started_at, finished_at)
+    VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)`, checkRunToValues(check));
+}
+
+export async function insertHistoryFixture(pool: Pool) {
+  const owner = (await pool.query<{ id: string }>('SELECT id FROM users WHERE username = $1', [scenario.ownerUsername])).rows[0];
+  if (!owner) throw new Error('The fixed history owner must already exist.');
+  await pool.query(`INSERT INTO monitors
+    (id, name, url, interval_seconds, enabled, created_at, updated_at, owner_user_id)
+    VALUES ($1, $2, $3, $4, $5, $6, $7, $8)`, [...monitorToValues(scenario.monitor as Monitor), owner.id]);
+  for (const check of originalChecks) await insertFixedCheck(pool, check);
+}
diff --git a/evidence/E07/scenario.json b/evidence/E07/scenario.json
new file mode 100644
index 0000000..7118c81
--- /dev/null
+++ b/evidence/E07/scenario.json
@@ -0,0 +1,60 @@
+{
+  "thread": "E07",
+  "attempt": 1,
+  "branch": "track/fundamentals-fastify",
+  "specRevision": "0a006589477f8ae47bad3faa5510c999cff85ee4",
+  "start": "f9adb6423b6ff5e732f89e52c30f7ac383bb0b9e",
+  "baselineSchema": "e07_baseline",
+  "integrationSchema": "e07_history",
+  "fixtureOrigin": "http://127.0.0.1:4311",
+  "browserOrigin": "http://127.0.0.1:4313",
+  "ownerUsername": "alice-e04",
+  "monitor": {
+    "id": "07000000-0000-4000-8000-000000000100",
+    "name": "E07 fixed tied history",
+    "url": "http://127.0.0.1:4311/ok",
+    "interval": 60,
+    "enabled": true,
+    "createdAt": "2026-08-28T00:00:00.000Z",
+    "updatedAt": "2026-08-28T00:00:00.000Z"
+  },
+  "orderingTimestamp": "2026-08-28T00:00:00.000Z",
+  "runs": [
+    { "id": "07000000-0000-4000-8000-000000000001", "state": "SUCCEEDED" },
+    { "id": "07000000-0000-4000-8000-000000000002", "state": "FAILED" },
+    { "id": "07000000-0000-4000-8000-000000000003", "state": "SUCCEEDED" },
+    { "id": "07000000-0000-4000-8000-000000000004", "state": "FAILED" },
+    { "id": "07000000-0000-4000-8000-000000000005", "state": "SUCCEEDED" },
+    { "id": "07000000-0000-4000-8000-000000000006", "state": "FAILED" },
+    { "id": "07000000-0000-4000-8000-000000000007", "state": "SUCCEEDED" }
+  ],
+  "newerRun": {
+    "id": "07000000-0000-4000-8000-000000000008",
+    "state": "SUCCEEDED",
+    "timestamp": "2026-08-28T00:00:01.000Z"
+  },
+  "terminalFields": { "trigger": "MANUAL", "latencyMs": 0, "successStatus": 200, "failedStatus": 503, "failureReason": "HTTP_STATUS" },
+  "page": {
+    "size": 3,
+    "defaultSize": 20,
+    "maximumSize": 100,
+    "order": "finished_at DESC, id DESC",
+    "cursorMaximumLength": 512,
+    "cursorConditions": ["monitorId", "state", "limit"],
+    "filters": ["SUCCEEDED", "FAILED"],
+    "invalidLimits": ["", "0", "-1", "1.5", "abc", "101", "1e1", "999999999999999999999"],
+    "invalidStates": ["", "RUNNING", "ABORTED", "failed"],
+    "invalidStatus": 400,
+    "invalidCode": "INVALID_INPUT"
+  },
+  "sequence": [
+    "At unchanged START request limit=3 over the exact seven rows; stop after the first decisive unbounded or missing-cursor result.",
+    "Traverse limit=3 pages over the seven equal timestamps; IDs must be 7,6,5,4,3,2,1 with no gaps or duplicates.",
+    "Read state=FAILED&limit=3; exactly IDs 6,4,2 and no further page.",
+    "Reject malformed, oversized, resource/filter/limit-mismatched cursors and nonpositive/noninteger/out-of-range limits with 400 INVALID_INPUT.",
+    "Read first page, insert only the fixed newer eighth row, then continue: all seven original IDs exactly once and no newer eighth row in that continuation.",
+    "Browser opens monitor/state=FAILED/limit=3 from URL, changes to SUCCEEDED, follows next cursor, then back/back/forward/forward and reload restores each condition and its rows."
+  ],
+  "budget": { "loadRuns": 0, "automaticRetries": 0, "parameterSweeps": 0, "baselineRuns": 1 },
+  "sensitiveArtifacts": { "trace": false, "video": false, "screenshot": false, "credentialSerialization": false }
+}


