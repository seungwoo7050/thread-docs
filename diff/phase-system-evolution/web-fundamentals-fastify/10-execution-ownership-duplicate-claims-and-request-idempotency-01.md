# E10 실행 소유권, 중복 Claim과 요청 멱등성

## `test: freeze E10 intent and ownership counterexamples`

diff --git a/SPEC_REVISION b/SPEC_REVISION
index a7af57d..42d5d11 100644
--- a/SPEC_REVISION
+++ b/SPEC_REVISION
@@ -1 +1 @@
-0a006589477f8ae47bad3faa5510c999cff85ee4
+2ada57a71cd34fa2fae9809415c362a8bbfcdf02
diff --git a/TRACK.md b/TRACK.md
index 8f3ae3f..849b708 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -1,6 +1,8 @@
 # Fundamentals / Fastify
 
-Spec revision: `0a006589477f8ae47bad3faa5510c999cff85ee4`.
+Profile: `phase-1` (E01–E12 → E20 → profile E24; phase-2 deferred, E25 not selected).
+
+Spec revision: `2ada57a71cd34fa2fae9809415c362a8bbfcdf02`.
 
 E01 established Monitor creation and synchronous manual GET checks in process memory.
 E03 stores Monitors and every terminal CheckRun in PostgreSQL, including after API restart.
diff --git a/evidence/phase-1/E10/baseline.mjs b/evidence/phase-1/E10/baseline.mjs
new file mode 100644
index 0000000..0921e1d
--- /dev/null
+++ b/evidence/phase-1/E10/baseline.mjs
@@ -0,0 +1,79 @@
+import assert from 'node:assert/strict';
+import { createHash } from 'node:crypto';
+import { execFileSync } from 'node:child_process';
+import { mkdir, readFile, writeFile } from 'node:fs/promises';
+import { buildApp } from '../../../server/app.ts';
+import { databaseConfig, databasePool, schemaIdentifier } from '../../../server/database.ts';
+import { migrate } from '../../../server/migrate.ts';
+import { authenticatedFetch, loginForTest } from '../../../test/auth.ts';
+import { identityFixture, insertIdentityFixture, manualBarrier, requireFreePorts } from './fixture.ts';
+import scenario from './scenario.json' with { type: 'json' };
+
+assert.equal(process.versions.node, '24.19.0');
+assert.equal(execFileSync('git', ['branch', '--show-current'], { encoding: 'utf8' }).trim(), 'track/fundamentals-fastify');
+assert.equal((await readFile('SPEC_REVISION', 'utf8')).trim(), scenario.specRevision);
+const changed = execFileSync('git', ['diff', '--name-only', scenario.start], { encoding: 'utf8' }).trim().split('\n').filter(Boolean);
+assert.ok(changed.every(path => path === 'SPEC_REVISION' || path === 'TRACK.md' || path.startsWith('evidence/phase-1/E10/')));
+const began = performance.now();
+const report = { start: scenario.start, baselineHead: execFileSync('git', ['rev-parse', 'HEAD'], { encoding: 'utf8' }).trim(),
+  productMatchesStart: true, hashes: {}, budget: scenario.budget, result: 'NOT_RUN' };
+for (const file of ['scenario.json', 'fixture.ts', 'baseline.mjs']) {
+  report.hashes[file] = createHash('sha256').update(await readFile(new URL(file, import.meta.url))).digest('hex');
+}
+const config = { ...databaseConfig(), schema: scenario.schema };
+const pool = databasePool(config);
+const fixture = identityFixture();
+const app = buildApp(undefined, config);
+const barrier = manualBarrier(app);
+let owned = false;
+let requests;
+let timer;
+try {
+  await requireFreePorts();
+  await pool.query(`CREATE SCHEMA ${schemaIdentifier(config.schema)}`);
+  owned = true;
+  await migrate(config);
+  const cookie = await loginForTest(app, await insertIdentityFixture(pool));
+  await new Promise((resolve, reject) => { fixture.server.once('error', reject); fixture.server.listen(4311, '127.0.0.1', resolve); });
+  await app.listen({ host: '127.0.0.1', port: 4312 });
+  requests = Promise.all(Array.from({ length: scenario.manual.concurrency }, () =>
+    authenticatedFetch(cookie)(`http://127.0.0.1:4312${scenario.manual.path}`,
+      { method: scenario.manual.method, headers: { [scenario.manual.header]: scenario.manual.key }, signal: AbortSignal.timeout(scenario.watchdogMs) })));
+  await Promise.race([barrier.arrived, new Promise((_, reject) => { timer = setTimeout(() => reject(new Error('Manual request barrier timed out.')), scenario.watchdogMs); })]);
+  clearTimeout(timer);
+  barrier.release();
+  const responses = await requests;
+  const statuses = responses.map(response => response.status);
+  const data = await Promise.all(responses.map(async response => (await response.json()).data));
+  const rows = (await pool.query('SELECT id, monitor_id, state FROM check_runs ORDER BY id')).rows;
+  report.observation = { barrierArrivals: barrier.arrivals(), statuses, returnedIds: data.map(check => check.id), rows,
+    distinctIds: new Set(data.map(check => check.id)).size, outboundCalls: [...fixture.calls.values()].reduce((sum, count) => sum + count, 0) };
+  assert.deepEqual(statuses, scenario.manual.expectedStatuses);
+  assert.equal(report.observation.barrierArrivals, 2);
+  assert.equal(report.observation.outboundCalls, 0);
+  assert.equal(report.observation.distinctIds, 2);
+  assert.equal(rows.length, 2);
+  assert.ok(rows.every(row => row.monitor_id === scenario.monitors[0].id && row.state === 'QUEUED'));
+  report.result = 'REPRODUCED';
+  report.decisiveFailure = 'Same owner and Idempotency-Key produced two durable QUEUED IDs instead of one intent.';
+} catch (error) {
+  report.result = 'FAILED'; report.failure = error.message; process.exitCode = 1;
+} finally {
+  clearTimeout(timer);
+  barrier.release();
+  if (requests) await requests.catch(() => {});
+  await app.close();
+  fixture.release();
+  if (fixture.server.listening) {
+    fixture.server.closeAllConnections();
+    await new Promise(resolve => fixture.server.close(resolve));
+  }
+  if (owned) await pool.query(`DROP SCHEMA ${schemaIdentifier(config.schema)} CASCADE`);
+  await pool.end();
+  await requireFreePorts();
+  report.cleanup = { schemaDropped: owned, portsFree: true };
+  report.durationMs = Math.round(performance.now() - began);
+  await mkdir('output/phase-1/e10', { recursive: true });
+  await writeFile('output/phase-1/e10/baseline.json', JSON.stringify(report, null, 2) + '\n');
+  console.log(JSON.stringify(report, null, 2));
+}
diff --git a/evidence/phase-1/E10/fixture.ts b/evidence/phase-1/E10/fixture.ts
new file mode 100644
index 0000000..5da8c13
--- /dev/null
+++ b/evidence/phase-1/E10/fixture.ts
@@ -0,0 +1,68 @@
+import { randomBytes } from 'node:crypto';
+import { createServer } from 'node:http';
+import { createServer as portServer } from 'node:net';
+import type { ServerResponse } from 'node:http';
+import type { FastifyInstance } from 'fastify';
+import type { Pool } from 'pg';
+import { hashPassword } from '../../../server/password.ts';
+import { monitorToValues } from '../../../server/mapping.ts';
+import scenario from './scenario.json' with { type: 'json' };
+
+export async function insertIdentityFixture(pool: Pool) {
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
+export function manualBarrier(app: FastifyInstance) {
+  const arrived = Promise.withResolvers<void>();
+  const released = Promise.withResolvers<void>();
+  let arrivals = 0;
+  app.addHook('preHandler', async request => {
+    if (request.method === scenario.manual.method && request.url === scenario.manual.path &&
+      request.headers['idempotency-key'] === scenario.manual.key && arrivals < scenario.manual.concurrency) {
+      arrivals++;
+      if (arrivals === scenario.manual.concurrency) arrived.resolve();
+      await released.promise;
+    }
+  });
+  return { arrived: arrived.promise, release: released.resolve, arrivals: () => arrivals };
+}
+
+export function identityFixture(holdOk = false) {
+  const entered = Promise.withResolvers<void>();
+  const calls = new Map<string, number>();
+  const held = new Set<ServerResponse>();
+  let released = !holdOk;
+  const server = createServer((request, response) => {
+    const path = request.url ?? '/';
+    calls.set(path, (calls.get(path) ?? 0) + 1);
+    if (path === '/ok' && !released) {
+      held.add(response);
+      response.once('close', () => held.delete(response));
+      entered.resolve();
+      return;
+    }
+    response.writeHead(path === '/ok' ? 200 : path === '/fail' ? 503 : 404);
+    response.end('fixture\n');
+  });
+  function release() {
+    released = true;
+    for (const response of held) { response.writeHead(scenario.worker.releaseStatus); response.end('released\n'); }
+    held.clear();
+  }
+  return { server, calls, entered: entered.promise, release };
+}
+
+export async function requireFreePorts() {
+  for (const port of scenario.ports) {
+    const server = portServer();
+    await new Promise<void>((resolve, reject) => { server.once('error', reject); server.listen(port, '127.0.0.1', resolve); });
+    await new Promise<void>((resolve, reject) => server.close(error => error ? reject(error) : resolve()));
+  }
+}
diff --git a/evidence/phase-1/E10/scenario.json b/evidence/phase-1/E10/scenario.json
new file mode 100644
index 0000000..3ed95dc
--- /dev/null
+++ b/evidence/phase-1/E10/scenario.json
@@ -0,0 +1,64 @@
+{
+  "thread": "E10",
+  "profile": "phase-1",
+  "specRevision": "2ada57a71cd34fa2fae9809415c362a8bbfcdf02",
+  "start": "d42b8767e90bfc495800cefccce807f1edc3c588",
+  "schema": "e10_intent",
+  "workerSchema": "e10_claim",
+  "ports": [4311, 4312, 4313, 4314],
+  "user": { "id": "10000000-0000-4000-8000-000000000001", "username": "alice-e10" },
+  "monitors": [
+    {
+      "id": "10000000-0000-4000-9000-000000000001", "name": "E10 Monitor A",
+      "url": "http://127.0.0.1:4311/ok", "interval": 60, "enabled": true,
+      "createdAt": "2026-08-28T00:00:00.000Z", "updatedAt": "2026-08-28T00:00:00.000Z"
+    },
+    {
+      "id": "10000000-0000-4000-9000-000000000002", "name": "E10 Monitor B",
+      "url": "http://127.0.0.1:4311/fail", "interval": 60, "enabled": false,
+      "createdAt": "2026-08-28T00:00:00.000Z", "updatedAt": "2026-08-28T00:00:00.000Z"
+    }
+  ],
+  "manual": {
+    "method": "POST", "path": "/monitors/10000000-0000-4000-9000-000000000001/checks",
+    "header": "Idempotency-Key", "key": "manual-intent-e10-1", "newKey": "manual-intent-e10-2",
+    "concurrency": 2, "barrier": "Both actual HTTP requests enter the test pre-handler before one explicit release.",
+    "expectedStatuses": [202, 202], "expectedUniqueIds": 1, "expectedRows": 1,
+    "workerStoppedUntilPairCompletes": true, "beforeWorkerOutbound": 0,
+    "sequenceAfterPair": [
+      "Same owner/key targeting B:409 CONFLICT, no B row or outbound.",
+      "Restart API; change only A name to E10 renamed A and enabled to false; same owner/key targeting A returns the original ID.",
+      "Different key targeting A:202 with a distinct ID; exactlytwo A rows.",
+      "Missing, empty, internal-space, tab, DEL and129-character keys:400 INVALID_INPUT; counts remain unchanged.",
+      "Start one real worker; both distinct intents finish SUCCEEDED200; exactlytwo /ok and zero /fail calls."
+    ],
+    "invalidKeys": [null, "", "two words", "two\twords", "\u007f", { "repeat": "x", "length": 129 }],
+    "semanticMeaning": "Canonical target Monitor UUID only; mutable Monitor fields are not part of identity."
+  },
+  "worker": {
+    "queuedId": "10000000-0000-4000-a000-000000000001",
+    "queuedAt": "2026-08-28T00:00:00.000Z",
+    "count": 2,
+    "barrier": "Hold ACCESS EXCLUSIVE on the owned check_runs table; wait until both actual CLI workers are ready and their claim queries are blocked by that connection; release once.",
+    "fixture": "Withhold /ok response headers until the persisted RUNNING owner and one outbound call are observed.",
+    "loserCompletion": "Call the production completion guard with the losing worker ID and a fixed FAILED503 outcome; expect0 writes and the RUNNING row unchanged.",
+    "expectedOwners": 1, "expectedOutbound": 1,
+    "releaseStatus": 200, "expectedTerminal": "SUCCEEDED", "expectedRows": 1,
+    "scheduler": "Both workers use the existing --no-schedule regression control; E09 fixed due-slot ticks remain unchanged."
+  },
+  "browser": {
+    "route": "/monitors?monitor=10000000-0000-4000-9000-000000000001&limit=20",
+    "dataset": "Same Alice/A/B fixture in the existing exclusively owned browser schema; real worker and production Next.",
+    "sequence": [
+      "Submit A once, hold the actual202 response after commit, dispatch a second submit event while held:one POST and one row.",
+      "Abort only the first delivery after the real commit; show the existing service-failure message.",
+      "Explicitly submit A again:second POST with the same browser-generated identity, same CheckRun ID and stillone row.",
+      "After successful acknowledgement, explicitly submit A again:third POST with a new identity and exactlytwo rows; same-ID real worker completion for both."
+    ],
+    "expectedForwardedPosts": 3, "expectedIntentRows": 2, "automaticRetries": 0,
+    "credentialsAndKeysRetained": false
+  },
+  "watchdogMs": 10000,
+  "budget": { "loadRuns": 0, "automaticRetries": 0, "parameterSweeps": 0, "unchangedStartBaselines": 1 },
+  "baseline": "Only the parallel manual pair; stop once its status/identity/count is decisive. Product source must match START, allowing only authorized revision metadata and new E10 evidence."
+}


