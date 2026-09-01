# E11 Worker Crash Recovery, Lease와 Graceful Shutdown

## `test: freeze E11 crash and shutdown checkpoints`

diff --git a/evidence/phase-1/E11/baseline.mjs b/evidence/phase-1/E11/baseline.mjs
new file mode 100644
index 0000000..f34a5ed
--- /dev/null
+++ b/evidence/phase-1/E11/baseline.mjs
@@ -0,0 +1,74 @@
+import assert from 'node:assert/strict';
+import { createHash } from 'node:crypto';
+import { execFileSync } from 'node:child_process';
+import { mkdir, readFile, writeFile } from 'node:fs/promises';
+import { databaseConfig, databasePool, schemaIdentifier } from '../../../server/database.ts';
+import { migrate } from '../../../server/migrate.ts';
+import { startTestWorker } from '../../../test/worker.ts';
+import { holdBeforeIo, insertRecoveryFixture, recoveryFixture, requireFreePorts } from './fixture.ts';
+import scenario from './scenario.json' with { type: 'json' };
+
+assert.equal(process.versions.node, '24.19.0');
+assert.equal(execFileSync('git', ['branch', '--show-current'], { encoding: 'utf8' }).trim(), 'track/fundamentals-fastify');
+assert.equal((await readFile('SPEC_REVISION', 'utf8')).trim(), scenario.specRevision);
+const changed = execFileSync('git', ['diff', '--name-only', scenario.start], { encoding: 'utf8' }).trim().split('\n').filter(Boolean);
+assert.ok(changed.every(path => path.startsWith('evidence/phase-1/E11/')));
+const began = performance.now();
+const report = { start: scenario.start, baselineHead: execFileSync('git', ['rev-parse', 'HEAD'], { encoding: 'utf8' }).trim(),
+  productMatchesStart: true, hashes: {}, budget: scenario.budget, result: 'NOT_RUN' };
+for (const file of ['scenario.json', 'fixture.ts', 'baseline.mjs']) {
+  report.hashes[file] = createHash('sha256').update(await readFile(new URL(file, import.meta.url))).digest('hex');
+}
+const config = { ...databaseConfig(), schema: scenario.checkpoints[0].schema };
+const pool = databasePool(config);
+const fixture = recoveryFixture(true);
+let owned = false;
+let barrier;
+let worker;
+try {
+  await requireFreePorts();
+  await pool.query(`CREATE SCHEMA ${schemaIdentifier(config.schema)}`);
+  owned = true;
+  await migrate(config);
+  await insertRecoveryFixture(pool);
+  await new Promise((resolve, reject) => { fixture.server.once('error', reject); fixture.server.listen(4311, '127.0.0.1', resolve); });
+  barrier = await holdBeforeIo(pool);
+  const applicationName = 'e11_before_io_worker';
+  const workerUrl = new URL(config.connectionString);
+  workerUrl.searchParams.set('application_name', applicationName);
+  worker = await startTestWorker({ ...config, connectionString: workerUrl.href });
+  const before = await barrier.arrived(applicationName);
+  assert.equal(before.worker_id, worker.workerId);
+  assert.equal(fixture.calls.size, 0);
+  assert.equal(process.kill(worker.pid, scenario.kill.signal), true);
+  const exit = await worker.stop();
+  assert.equal(exit.signal, 'SIGKILL');
+  assert.equal(exit.forced, false);
+  await barrier.release(); barrier = undefined;
+  const after = (await pool.query('SELECT * FROM check_runs WHERE id = $1', [scenario.check.id])).rows[0];
+  assert.deepEqual(after, before);
+  assert.equal(after.state, 'RUNNING');
+  assert.equal(Object.hasOwn(after, 'lease_expires_at'), false);
+  report.observation = { observerPid: process.pid, workerPid: worker.pid, workerId: worker.workerId,
+    checkpoint: 'before_io', exit, outboundCalls: 0, row: after, finiteLeasePresent: false };
+  report.result = 'REPRODUCED';
+  report.decisiveFailure = 'Actual SIGKILL leaves the committed RUNNING row with no persisted finite lease or expiry recovery.';
+} catch (error) {
+  report.result = 'FAILED'; report.failure = error.message; process.exitCode = 1;
+} finally {
+  if (worker) await worker.stop();
+  if (barrier) await barrier.release();
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
+  await mkdir('output/phase-1/e11', { recursive: true });
+  await writeFile('output/phase-1/e11/baseline.json', JSON.stringify(report, null, 2) + '\n');
+  console.log(JSON.stringify(report, null, 2));
+}
diff --git a/evidence/phase-1/E11/fixture.ts b/evidence/phase-1/E11/fixture.ts
new file mode 100644
index 0000000..d6b2756
--- /dev/null
+++ b/evidence/phase-1/E11/fixture.ts
@@ -0,0 +1,48 @@
+import { randomBytes } from 'node:crypto';
+import { setTimeout as delay } from 'node:timers/promises';
+import assert from 'node:assert/strict';
+import type { Pool, PoolClient } from 'pg';
+import { hashPassword } from '../../../server/password.ts';
+import { monitorToValues } from '../../../server/mapping.ts';
+import scenario from './scenario.json' with { type: 'json' };
+
+// Reuse the frozen /ok hold/release and reserved-port guard without changing it.
+export { identityFixture as recoveryFixture, requireFreePorts } from '../E10/fixture.ts';
+
+export async function insertRecoveryFixture(pool: Pool) {
+  const credentials = { username: scenario.user.username, password: randomBytes(32).toString('base64url') };
+  await pool.query('INSERT INTO users (id, username, password_hash) VALUES ($1, $2, $3)',
+    [scenario.user.id, credentials.username, await hashPassword(credentials.password)]);
+  await pool.query(`INSERT INTO monitors (id, name, url, interval_seconds, enabled, created_at, updated_at, owner_user_id)
+    VALUES ($1, $2, $3, $4, $5, $6, $7, $8)`, [...monitorToValues(scenario.monitor), scenario.user.id]);
+  await pool.query(`INSERT INTO check_runs (id, monitor_id, trigger, state, queued_at, request_user_id, idempotency_key)
+    VALUES ($1, $2, 'MANUAL', 'QUEUED', $3, $4, $5)`,
+  [scenario.check.id, scenario.monitor.id, new Date(scenario.check.queuedAt), scenario.user.id, scenario.check.key]);
+  return credentials;
+}
+
+// This exact durable-claim barrier is shared by the single baseline and fixed test.
+export async function holdBeforeIo(pool: Pool) {
+  const client: PoolClient = await pool.connect();
+  await client.query('BEGIN');
+  await client.query('LOCK TABLE monitors IN ACCESS EXCLUSIVE MODE');
+  const pid = (await client.query<{ pid: number }>('SELECT pg_backend_pid() AS pid')).rows[0].pid;
+  return {
+    async arrived(applicationName: string) {
+      const deadline = Date.now() + scenario.watchdogMs;
+      while (Date.now() < deadline) {
+        const waiting = await pool.query(`SELECT pid FROM pg_stat_activity WHERE application_name = $1
+          AND wait_event_type = 'Lock' AND $2 = ANY(pg_blocking_pids(pid))
+          AND query = 'SELECT * FROM monitors WHERE id = $1'`, [applicationName, pid]);
+        if (waiting.rowCount === 1) {
+          const row = (await pool.query('SELECT * FROM check_runs WHERE id = $1', [scenario.check.id])).rows[0];
+          assert.equal(row.state, 'RUNNING');
+          return row;
+        }
+        await delay(scenario.observationPollMs);
+      }
+      throw new Error('Durable claim before-I/O barrier timed out.');
+    },
+    async release() { await client.query('ROLLBACK'); client.release(); },
+  };
+}
diff --git a/evidence/phase-1/E11/scenario.json b/evidence/phase-1/E11/scenario.json
new file mode 100644
index 0000000..2e1c280
--- /dev/null
+++ b/evidence/phase-1/E11/scenario.json
@@ -0,0 +1,61 @@
+{
+  "thread": "E11",
+  "profile": "phase-1",
+  "specRevision": "2ada57a71cd34fa2fae9809415c362a8bbfcdf02",
+  "start": "591095c24b25efe59258ff07ee93e1b40adeadd4",
+  "ports": [4311, 4312, 4313, 4314],
+  "watchdogMs": 10000,
+  "observationPollMs": 25,
+  "leaseMs": 5000,
+  "httpDeadlineMs": 2000,
+  "user": { "id": "11000000-0000-4000-8000-000000000001", "username": "alice-e11" },
+  "monitor": {
+    "id": "11000000-0000-4000-9000-000000000001",
+    "name": "E11 Monitor A",
+    "url": "http://127.0.0.1:4311/ok",
+    "interval": 60,
+    "enabled": true,
+    "createdAt": "2026-08-28T00:00:00.000Z",
+    "updatedAt": "2026-08-28T00:00:00.000Z"
+  },
+  "check": {
+    "id": "11000000-0000-4000-a000-000000000001",
+    "trigger": "MANUAL",
+    "state": "QUEUED",
+    "queuedAt": "2026-08-28T00:00:00.000Z",
+    "key": "manual-recovery-e11-1",
+    "newIntentKey": "manual-recovery-e11-2"
+  },
+  "checkpoints": [
+    { "name": "before_io", "schema": "e11_before_io", "barrier": "monitors ACCESS EXCLUSIVE; committed RUNNING and worker Monitor SELECT waiting on that lock", "outboundCalls": 0, "recoveredState": "ABORTED" },
+    { "name": "held_response", "schema": "e11_held_response", "barrier": "fixture received GET /ok and withholds all HTTP headers", "outboundCalls": 1, "recoveredState": "ABORTED" },
+    { "name": "before_commit", "schema": "e11_before_commit", "barrier": "hold CheckRun row lock; release HTTP200; worker terminal UPDATE waiting on that lock", "outboundCalls": 1, "recoveredState": "ABORTED" },
+    { "name": "after_commit", "schema": "e11_after_commit", "barrier": "HTTP200 terminal row observed through a separate connection after COMMIT", "outboundCalls": 1, "recoveredState": "SUCCEEDED" }
+  ],
+  "kill": { "signal": "SIGKILL", "runsPerCheckpoint": 1, "releaseStatus": 200 },
+  "recovery": {
+    "process": "separate Node process invoking the production recoverExpiredChecks operation; the real CLI worker loop invokes the same operation",
+    "clock": "persisted lease_expires_at minus 1ms, then exact equality",
+    "beforeExpiryWrites": 0,
+    "unknownFields": { "httpStatus": null, "latencyMs": null, "failureReason": null },
+    "staleCompletionWrites": 0,
+    "requeue": false,
+    "renewal": false,
+    "identity": "original owner/key returns same terminal ID; new key creates a different QUEUED ID without resetting original"
+  },
+  "shutdown": {
+    "schema": "e11_shutdown",
+    "signal": "SIGTERM",
+    "policy": "stop admission at signal; drain only current HTTP check; exit nonzero at 3000ms if drain or database close is still stuck, leaving uncertainty for lease recovery",
+    "graceMs": 3000,
+    "secondCheckId": "11000000-0000-4000-a000-000000000002",
+    "secondKey": "manual-recovery-e11-second",
+    "barrier": "first GET /ok held; second row QUEUED; observe stopping signal acknowledgement before releasing first HTTP200",
+    "expectedFirstState": "SUCCEEDED",
+    "expectedSecondState": "QUEUED",
+    "expectedOutboundCalls": 1,
+    "expectedExitCode": 0
+  },
+  "baseline": { "checkpoint": "before_io", "runs": 1, "stopAt": "killed owner leaves RUNNING with no finite persisted lease" },
+  "budget": { "loadRuns": 0, "automaticRetries": 0, "parameterSweeps": 0 }
+}


