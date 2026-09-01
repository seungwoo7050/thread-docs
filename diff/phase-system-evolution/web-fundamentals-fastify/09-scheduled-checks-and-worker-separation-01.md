# E09 예약 Check와 Worker 분리

## `test: freeze E09 worker separation counterexample`

diff --git a/evidence/E09/baseline.json b/evidence/E09/baseline.json
new file mode 100644
index 0000000..43129fe
--- /dev/null
+++ b/evidence/E09/baseline.json
@@ -0,0 +1,39 @@
+{
+  "start": "19138e0618cb22b0250c00ea3f52de6095d95af8",
+  "hashes": {
+    "scenario.json": "d2412d309e282a7d99a7a73ee64c204b4fddfef8b7211e42b573e501de4f8240",
+    "fixture.ts": "4ecfea011cda7656e7e59eb7236f378f3ad251193e23d01135f642139327c078",
+    "baseline.mjs": "a95b8b69b99a838899c1b71b2d0beec9fc57502de33b100b151e3ccedfdfc946"
+  },
+  "budget": {
+    "loadRuns": 0,
+    "automaticRetries": 0,
+    "parameterSweeps": 0
+  },
+  "result": "REPRODUCED",
+  "beforeRelease": {
+    "responseReturned": false,
+    "persisted": [],
+    "outboundCalls": 1
+  },
+  "afterRelease": {
+    "status": 200,
+    "data": {
+      "id": "f2e54adb-0c7d-48dd-8dc1-c3aa29574844",
+      "monitorId": "09000000-0000-4000-9000-000000000001",
+      "trigger": "MANUAL",
+      "state": "SUCCEEDED",
+      "httpStatus": 200,
+      "latencyMs": 4,
+      "failureReason": null,
+      "startedAt": "2026-08-28T02:29:38.541Z",
+      "finishedAt": "2026-08-28T02:29:38.545Z"
+    }
+  },
+  "decisiveFailure": "The API performed outbound I/O without a persisted intent and returned a terminal200 only after release, rather than persisted202 QUEUED before worker execution.",
+  "cleanup": {
+    "schemaDropped": true,
+    "portsFree": true
+  },
+  "durationMs": 757
+}
diff --git a/evidence/E09/baseline.mjs b/evidence/E09/baseline.mjs
new file mode 100644
index 0000000..386b7ed
--- /dev/null
+++ b/evidence/E09/baseline.mjs
@@ -0,0 +1,79 @@
+import assert from 'node:assert/strict';
+import { createHash } from 'node:crypto';
+import { execFileSync } from 'node:child_process';
+import { createServer } from 'node:net';
+import { mkdir, readFile, writeFile } from 'node:fs/promises';
+import { buildApp } from '../../server/app.ts';
+import { databaseConfig, databasePool, schemaIdentifier } from '../../server/database.ts';
+import { migrate } from '../../server/migrate.ts';
+import { authenticatedFetch, loginForTest } from '../../test/auth.ts';
+import { heldFixture, insertExecutionFixture } from './fixture.ts';
+import scenario from './scenario.json' with { type: 'json' };
+
+assert.equal(process.versions.node, '24.19.0');
+assert.equal(execFileSync('git', ['branch', '--show-current'], { encoding: 'utf8' }).trim(), 'track/fundamentals-fastify');
+assert.equal(execFileSync('git', ['rev-parse', 'HEAD'], { encoding: 'utf8' }).trim(), scenario.start);
+assert.equal(execFileSync('git', ['diff', '--name-only', 'HEAD'], { encoding: 'utf8' }).trim(), '');
+assert.equal((await readFile('SPEC_REVISION', 'utf8')).trim(), scenario.specRevision);
+const began = performance.now();
+const report = { start: scenario.start, hashes: {}, budget: scenario.budget, result: 'NOT_RUN' };
+for (const file of ['scenario.json', 'fixture.ts', 'baseline.mjs']) {
+  report.hashes[file] = createHash('sha256').update(await readFile(new URL(file, import.meta.url))).digest('hex');
+}
+const config = { ...databaseConfig(), schema: scenario.schema };
+const pool = databasePool(config);
+const fixture = heldFixture();
+const app = buildApp(undefined, config);
+let ownsSchema = false;
+async function requireFreePorts() {
+  for (const port of scenario.ports) {
+    const server = createServer();
+    await new Promise((resolve, reject) => { server.once('error', reject); server.listen(port, '127.0.0.1', resolve); });
+    await new Promise((resolve, reject) => server.close(error => error ? reject(error) : resolve()));
+  }
+}
+try {
+  await requireFreePorts();
+  await pool.query(`CREATE SCHEMA ${schemaIdentifier(config.schema)}`);
+  ownsSchema = true;
+  await migrate(config);
+  const cookie = await loginForTest(app, await insertExecutionFixture(pool));
+  await new Promise((resolve, reject) => { fixture.server.once('error', reject); fixture.server.listen(4311, '127.0.0.1', resolve); });
+  await app.listen({ host: '127.0.0.1', port: 4312 });
+  let responseReturned = false;
+  const request = authenticatedFetch(cookie)(`http://127.0.0.1:4312${scenario.manual.path}`, { method: scenario.manual.method })
+    .then(response => { responseReturned = true; return response; });
+  await fixture.entered;
+  const persisted = (await pool.query('SELECT id, state FROM check_runs')).rows;
+  report.beforeRelease = { responseReturned, persisted, outboundCalls: fixture.calls.get('/hold') ?? 0 };
+  fixture.release();
+  const response = await request;
+  report.afterRelease = { status: response.status, data: (await response.json()).data };
+  assert.equal(report.beforeRelease.responseReturned, false);
+  assert.equal(report.beforeRelease.persisted.length, 0);
+  assert.equal(report.beforeRelease.outboundCalls, 1);
+  assert.equal(report.afterRelease.status, 200);
+  assert.equal(report.afterRelease.data.state, 'SUCCEEDED');
+  assert.equal(report.afterRelease.data.httpStatus, 200);
+  report.result = 'REPRODUCED';
+  report.decisiveFailure = 'The API performed outbound I/O without a persisted intent and returned a terminal200 only after release, rather than persisted202 QUEUED before worker execution.';
+} catch (error) {
+  report.result = 'FAILED';
+  report.failure = error.message;
+  process.exitCode = 1;
+} finally {
+  fixture.release();
+  await app.close();
+  if (fixture.server.listening) {
+    fixture.server.closeAllConnections();
+    await new Promise(resolve => fixture.server.close(resolve));
+  }
+  if (ownsSchema) await pool.query(`DROP SCHEMA ${schemaIdentifier(config.schema)} CASCADE`);
+  await pool.end();
+  await requireFreePorts();
+  report.cleanup = { schemaDropped: ownsSchema, portsFree: true };
+  report.durationMs = Math.round(performance.now() - began);
+  await mkdir('output/e09', { recursive: true });
+  await writeFile('output/e09/baseline.json', JSON.stringify(report, null, 2) + '\n');
+  console.log(JSON.stringify(report, null, 2));
+}
diff --git a/evidence/E09/fixture.ts b/evidence/E09/fixture.ts
new file mode 100644
index 0000000..7e5d7ef
--- /dev/null
+++ b/evidence/E09/fixture.ts
@@ -0,0 +1,43 @@
+import { randomBytes } from 'node:crypto';
+import { createServer } from 'node:http';
+import type { ServerResponse } from 'node:http';
+import type { Pool } from 'pg';
+import { hashPassword } from '../../server/password.ts';
+import { monitorToValues } from '../../server/mapping.ts';
+import scenario from './scenario.json' with { type: 'json' };
+
+export async function insertExecutionFixture(pool: Pool) {
+  const credentials = { username: scenario.user.username, password: randomBytes(32).toString('base64url') };
+  await pool.query('INSERT INTO users (id, username, password_hash) VALUES ($1, $2, $3)',
+    [scenario.user.id, credentials.username, await hashPassword(credentials.password)]);
+  for (const monitor of scenario.monitors) {
+    await pool.query(`INSERT INTO monitors (id, name, url, interval_seconds, enabled, created_at, updated_at, owner_user_id)
+      VALUES ($1, $2, $3, $4, $5, $6, $7, $8)`, [...monitorToValues(monitor), scenario.user.id]);
+  }
+  return credentials;
+}
+
+export function heldFixture() {
+  const entered = Promise.withResolvers<void>();
+  const calls = new Map<string, number>();
+  const held = new Set<ServerResponse>();
+  let released = false;
+  const server = createServer((request, response) => {
+    const path = request.url ?? '/';
+    calls.set(path, (calls.get(path) ?? 0) + 1);
+    if (path === '/hold' && !released) {
+      held.add(response);
+      response.once('close', () => held.delete(response));
+      entered.resolve();
+      return;
+    }
+    response.writeHead(path === '/hold' || path === '/ok' ? 200 : 404);
+    response.end('fixture\n');
+  });
+  function release() {
+    released = true;
+    for (const response of held) { response.writeHead(scenario.manual.releaseStatus); response.end('released\n'); }
+    held.clear();
+  }
+  return { server, calls, entered: entered.promise, release };
+}
diff --git a/evidence/E09/scenario.json b/evidence/E09/scenario.json
new file mode 100644
index 0000000..565f984
--- /dev/null
+++ b/evidence/E09/scenario.json
@@ -0,0 +1,52 @@
+{
+  "thread": "E09",
+  "specRevision": "0a006589477f8ae47bad3faa5510c999cff85ee4",
+  "start": "19138e0618cb22b0250c00ea3f52de6095d95af8",
+  "schema": "e09_execution",
+  "ports": [4311, 4312, 4313, 4314],
+  "user": { "id": "09000000-0000-4000-8000-000000000001", "username": "alice-e09" },
+  "monitors": [
+    {
+      "id": "09000000-0000-4000-9000-000000000001",
+      "name": "E09 Monitor A",
+      "url": "http://127.0.0.1:4311/hold",
+      "interval": 60,
+      "enabled": true,
+      "createdAt": "2026-08-28T00:00:00.000Z",
+      "updatedAt": "2026-08-28T00:00:00.000Z"
+    },
+    {
+      "id": "09000000-0000-4000-9000-000000000002",
+      "name": "E09 Monitor B",
+      "url": "http://127.0.0.1:4311/ok",
+      "interval": 60,
+      "enabled": false,
+      "createdAt": "2026-08-28T00:00:00.000Z",
+      "updatedAt": "2026-08-28T00:00:00.000Z"
+    }
+  ],
+  "initialChecks": [],
+  "manual": {
+    "method": "POST",
+    "path": "/monitors/09000000-0000-4000-9000-000000000001/checks",
+    "concurrency": 1,
+    "expectedStatus": 202,
+    "beforeWorker": { "state": "QUEUED", "outboundCalls": 0, "httpStatus": null, "startedAt": null, "finishedAt": null },
+    "atHeldFixture": { "state": "RUNNING", "outboundCalls": 1, "httpStatus": null, "finishedAt": null },
+    "releaseStatus": 200,
+    "afterRelease": { "state": "SUCCEEDED", "httpStatus": 200, "outboundCalls": 1 },
+    "sameCheckRunIdThroughout": true,
+    "browserPrimaryRoute": "/monitors?monitor=09000000-0000-4000-9000-000000000001&limit=20",
+    "browserUsableWhileHeld": "Edit Monitor B locally without saving while A is RUNNING",
+    "fixtureDelay": "Withhold all headers until explicit release; no sleeps or latency threshold"
+  },
+  "scheduler": {
+    "ticks": ["2026-08-28T00:00:00.000Z", "2026-08-28T00:00:00.000Z", "2026-08-28T00:01:00.000Z"],
+    "expectedScheduledA": [1, 1, 2],
+    "expectedScheduledB": [0, 0, 0],
+    "workerStoppedDuringTicks": true,
+    "overlap": "Distinct manual and scheduled intents may overlap; a scheduled due slot is unique per Monitor."
+  },
+  "watchdogMs": 10000,
+  "budget": { "loadRuns": 0, "automaticRetries": 0, "parameterSweeps": 0 }
+}


