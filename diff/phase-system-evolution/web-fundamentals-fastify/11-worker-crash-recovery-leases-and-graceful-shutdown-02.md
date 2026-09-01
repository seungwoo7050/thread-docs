## `test: freeze E11 child-exit observer repair`

diff --git a/evidence/phase-1/E11/baseline-attempt1.json b/evidence/phase-1/E11/baseline-attempt1.json
new file mode 100644
index 0000000..2bd6122
--- /dev/null
+++ b/evidence/phase-1/E11/baseline-attempt1.json
@@ -0,0 +1,22 @@
+{
+  "start": "591095c24b25efe59258ff07ee93e1b40adeadd4",
+  "baselineHead": "b17e00a0e96d7e15f128c20c17edcccd7c73e334",
+  "productMatchesStart": true,
+  "hashes": {
+    "scenario.json": "8ac1422579b3756d9b0b6151b20a4b2f37829cc0dfab9ab5942d5a8b2a292ce5",
+    "fixture.ts": "52ad25d97c3703205fde226fa79541f80bbeaca3d987bb9da290b9340e22f539",
+    "baseline.mjs": "2aa23e11221779c4130ea46a32ae04825b9a01c2d3c48bfe503ebe35da4a3047"
+  },
+  "budget": {
+    "loadRuns": 0,
+    "automaticRetries": 0,
+    "parameterSweeps": 0
+  },
+  "result": "FAILED",
+  "failure": "Expected values to be strictly equal:\n+ actual - expected\n\n+ 'SIGTERM'\n- 'SIGKILL'\n      ^\n",
+  "cleanup": {
+    "schemaDropped": true,
+    "portsFree": true
+  },
+  "durationMs": 727
+}
diff --git a/evidence/phase-1/E11/baseline-observer.mjs b/evidence/phase-1/E11/baseline-observer.mjs
new file mode 100644
index 0000000..eaa37d5
--- /dev/null
+++ b/evidence/phase-1/E11/baseline-observer.mjs
@@ -0,0 +1,91 @@
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
+const freeze = JSON.parse(await readFile(new URL('observer-repair-freeze.json', import.meta.url), 'utf8'));
+const changed = execFileSync('git', ['diff', '--name-only', scenario.start], { encoding: 'utf8' }).trim().split('\n').filter(Boolean);
+assert.ok(changed.every(path => path === 'test/worker.ts' || path.startsWith('evidence/phase-1/E11/')),
+  'All product, runtime, package, and other test sources must still match Thread START.');
+const untracked = execFileSync('git', ['ls-files', '--others', '--exclude-standard'], { encoding: 'utf8' }).trim().split('\n').filter(Boolean);
+assert.ok(untracked.every(path => path.startsWith('evidence/phase-1/E11/')),
+  'No untracked product, runtime, package, or other test source is allowed.');
+const workerDiff = execFileSync('git', ['diff', '--no-ext-diff', '--no-color', '--no-renames', '--full-index',
+  '--src-prefix=a/', '--dst-prefix=b/', '--unified=3', scenario.start, '--', 'test/worker.ts'], { encoding: 'utf8' });
+assert.equal(workerDiff, freeze.workerAdapter.diff, 'Only the frozen child-close observer adapter is allowed.');
+assert.equal(createHash('sha256').update(await readFile('test/worker.ts')).digest('hex'), freeze.workerAdapter.sha256);
+const began = performance.now();
+const report = { start: scenario.start, baselineHead: execFileSync('git', ['rev-parse', 'HEAD'], { encoding: 'utf8' }).trim(),
+  productMatchesStart: true, hashes: {}, budget: scenario.budget, result: 'NOT_RUN',
+  overallAttempt: freeze.overallAttempt, freshRepair: freeze.freshRepair, baselineInvocation: freeze.baselineInvocation };
+for (const file of ['scenario.json', 'fixture.ts', 'baseline.mjs', 'baseline-observer.mjs', 'baseline-attempt1.json']) {
+  report.hashes[file] = createHash('sha256').update(await readFile(new URL(file, import.meta.url))).digest('hex');
+  assert.equal(report.hashes[file], freeze.hashes[file], `${file} must match the pre-invocation freeze.`);
+}
+const config = { ...databaseConfig(), schema: scenario.checkpoints[0].schema };
+const databaseUrl = new URL(config.connectionString);
+assert.equal(databaseUrl.hostname, '127.0.0.1');
+assert.equal(databaseUrl.port, '15431');
+assert.equal(databaseUrl.pathname, '/monitor');
+const pool = databasePool(config);
+const fixture = recoveryFixture(true);
+let owned = false;
+let barrier;
+let worker;
+let workerExitObserved = false;
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
+  const exit = await worker.waitForExit();
+  workerExitObserved = true;
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
+  if (worker && !workerExitObserved) await worker.stop();
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
+  await writeFile('output/phase-1/e11/baseline-observer.json', JSON.stringify(report, null, 2) + '\n');
+  console.log(JSON.stringify(report, null, 2));
+}
diff --git a/evidence/phase-1/E11/observer-repair-freeze.json b/evidence/phase-1/E11/observer-repair-freeze.json
new file mode 100644
index 0000000..396269e
--- /dev/null
+++ b/evidence/phase-1/E11/observer-repair-freeze.json
@@ -0,0 +1,61 @@
+{
+  "thread": "E11",
+  "profile": "phase-1",
+  "specRevision": "2ada57a71cd34fa2fae9809415c362a8bbfcdf02",
+  "threadStart": "591095c24b25efe59258ff07ee93e1b40adeadd4",
+  "repairStart": "b17e00a0e96d7e15f128c20c17edcccd7c73e334",
+  "overallAttempt": 2,
+  "freshRepair": 1,
+  "baselineInvocation": 2,
+  "status": "FROZEN_BEFORE_CORRECTED_BASELINE",
+  "frozenAt": "2026-08-28T04:46:38.566081+00:00",
+  "hashes": {
+    "scenario.json": "8ac1422579b3756d9b0b6151b20a4b2f37829cc0dfab9ab5942d5a8b2a292ce5",
+    "fixture.ts": "52ad25d97c3703205fde226fa79541f80bbeaca3d987bb9da290b9340e22f539",
+    "baseline.mjs": "2aa23e11221779c4130ea46a32ae04825b9a01c2d3c48bfe503ebe35da4a3047",
+    "baseline-observer.mjs": "0f370c1c712618ad36faac8c8e023fb6b4d344af868809cfe0face385d3510d2",
+    "baseline-attempt1.json": "30f019be3e0fd568a8346711a98dfa4743e797c583aded89acdd0ac0a318a0ec"
+  },
+  "originalFailure": {
+    "artifact": "baseline-attempt1.json",
+    "source": "/Users/woopinbell/Desktop/working/workflow/web-systems-evolution/index/profiles/phase-1/verification/fundamentals-fastify/E11-artifacts/root-read-baseline-attempt1.json",
+    "sha256": "30f019be3e0fd568a8346711a98dfa4743e797c583aded89acdd0ac0a318a0ec",
+    "exitCode": 1,
+    "durationMs": 727,
+    "cause": "Original consumer sent SIGKILL, then stop() sent SIGTERM instead of only observing the existing child close promise; recorded signal was SIGTERM."
+  },
+  "workerAdapter": {
+    "path": "test/worker.ts",
+    "startSha256": "c2bf1c93b52e25c329b284fe78b70fcc6958fd1c60e2a1afa63b8500a6efeeb6",
+    "sha256": "5e32fa6954b9f17cfcd940e854935b17adcf26daaed8a7bfd6b0382198508d4a",
+    "contract": "waitForExit only awaits the existing child close promise. Its forced=false result records that this observation path creates no forced-kill watchdog. The stop implementation is unchanged.",
+    "diff": "diff --git a/test/worker.ts b/test/worker.ts\nindex 5fe93ebe0e979c785275bfad77faa1e885d25664..ce6c253577271c607831df4fd0ac4620178d565d 100644\n--- a/test/worker.ts\n+++ b/test/worker.ts\n@@ -22,6 +22,10 @@ export async function startTestWorker(config: DatabaseConfig) {\n     clearTimeout(timer);\n     return { ...result, forced };\n   }\n+  async function waitForExit() {\n+    // Observation only: no signal or watchdog is issued by this path.\n+    return { ...await exited, forced: false };\n+  }\n   try {\n     await new Promise<void>((resolve, reject) => {\n       const timer = setTimeout(() => reject(new Error('Worker readiness timed out.')), 10000);\n@@ -35,7 +39,7 @@ export async function startTestWorker(config: DatabaseConfig) {\n       child.once('exit', () => { clearTimeout(timer); reject(new Error('Worker exited before readiness.')); });\n     });\n   } catch (error) { await stop(); throw error; }\n-  return { pid: child.pid!, workerId, stop };\n+  return { pid: child.pid!, workerId, stop, waitForExit };\n }\n \n export async function waitForTerminalCheck(pool: Pool, id: string): Promise<TerminalCheckRun> {\n"
+  },
+  "runnerAdapter": {
+    "path": "evidence/phase-1/E11/baseline-observer.mjs",
+    "diff": "--- evidence/phase-1/E11/baseline.mjs\n+++ evidence/phase-1/E11/baseline-observer.mjs\n@@ -11,20 +11,36 @@\n assert.equal(process.versions.node, '24.19.0');\n assert.equal(execFileSync('git', ['branch', '--show-current'], { encoding: 'utf8' }).trim(), 'track/fundamentals-fastify');\n assert.equal((await readFile('SPEC_REVISION', 'utf8')).trim(), scenario.specRevision);\n+const freeze = JSON.parse(await readFile(new URL('observer-repair-freeze.json', import.meta.url), 'utf8'));\n const changed = execFileSync('git', ['diff', '--name-only', scenario.start], { encoding: 'utf8' }).trim().split('\\n').filter(Boolean);\n-assert.ok(changed.every(path => path.startsWith('evidence/phase-1/E11/')));\n+assert.ok(changed.every(path => path === 'test/worker.ts' || path.startsWith('evidence/phase-1/E11/')),\n+  'All product, runtime, package, and other test sources must still match Thread START.');\n+const untracked = execFileSync('git', ['ls-files', '--others', '--exclude-standard'], { encoding: 'utf8' }).trim().split('\\n').filter(Boolean);\n+assert.ok(untracked.every(path => path.startsWith('evidence/phase-1/E11/')),\n+  'No untracked product, runtime, package, or other test source is allowed.');\n+const workerDiff = execFileSync('git', ['diff', '--no-ext-diff', '--no-color', '--no-renames', '--full-index',\n+  '--src-prefix=a/', '--dst-prefix=b/', '--unified=3', scenario.start, '--', 'test/worker.ts'], { encoding: 'utf8' });\n+assert.equal(workerDiff, freeze.workerAdapter.diff, 'Only the frozen child-close observer adapter is allowed.');\n+assert.equal(createHash('sha256').update(await readFile('test/worker.ts')).digest('hex'), freeze.workerAdapter.sha256);\n const began = performance.now();\n const report = { start: scenario.start, baselineHead: execFileSync('git', ['rev-parse', 'HEAD'], { encoding: 'utf8' }).trim(),\n-  productMatchesStart: true, hashes: {}, budget: scenario.budget, result: 'NOT_RUN' };\n-for (const file of ['scenario.json', 'fixture.ts', 'baseline.mjs']) {\n+  productMatchesStart: true, hashes: {}, budget: scenario.budget, result: 'NOT_RUN',\n+  overallAttempt: freeze.overallAttempt, freshRepair: freeze.freshRepair, baselineInvocation: freeze.baselineInvocation };\n+for (const file of ['scenario.json', 'fixture.ts', 'baseline.mjs', 'baseline-observer.mjs', 'baseline-attempt1.json']) {\n   report.hashes[file] = createHash('sha256').update(await readFile(new URL(file, import.meta.url))).digest('hex');\n+  assert.equal(report.hashes[file], freeze.hashes[file], `${file} must match the pre-invocation freeze.`);\n }\n const config = { ...databaseConfig(), schema: scenario.checkpoints[0].schema };\n+const databaseUrl = new URL(config.connectionString);\n+assert.equal(databaseUrl.hostname, '127.0.0.1');\n+assert.equal(databaseUrl.port, '15431');\n+assert.equal(databaseUrl.pathname, '/monitor');\n const pool = databasePool(config);\n const fixture = recoveryFixture(true);\n let owned = false;\n let barrier;\n let worker;\n+let workerExitObserved = false;\n try {\n   await requireFreePorts();\n   await pool.query(`CREATE SCHEMA ${schemaIdentifier(config.schema)}`);\n@@ -41,7 +57,8 @@\n   assert.equal(before.worker_id, worker.workerId);\n   assert.equal(fixture.calls.size, 0);\n   assert.equal(process.kill(worker.pid, scenario.kill.signal), true);\n-  const exit = await worker.stop();\n+  const exit = await worker.waitForExit();\n+  workerExitObserved = true;\n   assert.equal(exit.signal, 'SIGKILL');\n   assert.equal(exit.forced, false);\n   await barrier.release(); barrier = undefined;\n@@ -56,7 +73,7 @@\n } catch (error) {\n   report.result = 'FAILED'; report.failure = error.message; process.exitCode = 1;\n } finally {\n-  if (worker) await worker.stop();\n+  if (worker && !workerExitObserved) await worker.stop();\n   if (barrier) await barrier.release();\n   fixture.release();\n   if (fixture.server.listening) {\n@@ -69,6 +86,6 @@\n   report.cleanup = { schemaDropped: owned, portsFree: true };\n   report.durationMs = Math.round(performance.now() - began);\n   await mkdir('output/phase-1/e11', { recursive: true });\n-  await writeFile('output/phase-1/e11/baseline.json', JSON.stringify(report, null, 2) + '\\n');\n+  await writeFile('output/phase-1/e11/baseline-observer.json', JSON.stringify(report, null, 2) + '\\n');\n   console.log(JSON.stringify(report, null, 2));\n }\n",
+    "changes": [
+      "Accept only the exact frozen test/worker.ts observer diff and E11 evidence; reject all other source changes and untracked source files.",
+      "Check frozen originals, corrected runner, preserved failure and worker helper hashes before the invocation.",
+      "Assert the configured database is the existing local PostgreSQL15431/monitor instance.",
+      "Await waitForExit after the same SIGKILL; do not call stop after exit has been observed.",
+      "Record repair and invocation identity; write corrected output separately so the original failed output is retained."
+    ]
+  },
+  "unchanged": {
+    "product": "Every tracked path outside evidence/phase-1/E11 and the exact test/worker.ts adapter must equal Thread START, including server, app, runtime and package sources.",
+    "counterexample": "Original scenario.json and fixture.ts, durable RUNNING/blocked Monitor SELECT barrier, SIGKILL, row equality/no finite lease assertions, data and thresholds are unchanged."
+  },
+  "budget": {
+    "priorBaselineInvocations": 1,
+    "authorizedCorrectedInvocations": 1,
+    "totalBaselineInvocationsAfterThisRun": 2,
+    "freshRepairsUsed": 1,
+    "maximumFreshRepairs": 2,
+    "loadRuns": 0,
+    "automaticRetries": 0,
+    "parameterSweeps": 0
+  },
+  "command": "fnm exec --using 24.19.0 node evidence/phase-1/E11/baseline-observer.mjs",
+  "afterRun": "Preserve actual outcome and timing; clean only owned schema/processes; stop for root verification without E11 product implementation."
+}
diff --git a/test/worker.ts b/test/worker.ts
index 5fe93eb..ce6c253 100644
--- a/test/worker.ts
+++ b/test/worker.ts
@@ -22,6 +22,10 @@ export async function startTestWorker(config: DatabaseConfig) {
     clearTimeout(timer);
     return { ...result, forced };
   }
+  async function waitForExit() {
+    // Observation only: no signal or watchdog is issued by this path.
+    return { ...await exited, forced: false };
+  }
   try {
     await new Promise<void>((resolve, reject) => {
       const timer = setTimeout(() => reject(new Error('Worker readiness timed out.')), 10000);
@@ -35,7 +39,7 @@ export async function startTestWorker(config: DatabaseConfig) {
       child.once('exit', () => { clearTimeout(timer); reject(new Error('Worker exited before readiness.')); });
     });
   } catch (error) { await stop(); throw error; }
-  return { pid: child.pid!, workerId, stop };
+  return { pid: child.pid!, workerId, stop, waitForExit };
 }
 
 export async function waitForTerminalCheck(pool: Pool, id: string): Promise<TerminalCheckRun> {


## `test: record E11 corrected SIGKILL baseline`

diff --git a/evidence/phase-1/E11/baseline-observer.json b/evidence/phase-1/E11/baseline-observer.json
new file mode 100644
index 0000000..7a9c4c0
--- /dev/null
+++ b/evidence/phase-1/E11/baseline-observer.json
@@ -0,0 +1,56 @@
+{
+  "start": "591095c24b25efe59258ff07ee93e1b40adeadd4",
+  "baselineHead": "c776c6545274019e78cd056a226ce460a9699262",
+  "productMatchesStart": true,
+  "hashes": {
+    "scenario.json": "8ac1422579b3756d9b0b6151b20a4b2f37829cc0dfab9ab5942d5a8b2a292ce5",
+    "fixture.ts": "52ad25d97c3703205fde226fa79541f80bbeaca3d987bb9da290b9340e22f539",
+    "baseline.mjs": "2aa23e11221779c4130ea46a32ae04825b9a01c2d3c48bfe503ebe35da4a3047",
+    "baseline-observer.mjs": "0f370c1c712618ad36faac8c8e023fb6b4d344af868809cfe0face385d3510d2",
+    "baseline-attempt1.json": "30f019be3e0fd568a8346711a98dfa4743e797c583aded89acdd0ac0a318a0ec"
+  },
+  "budget": {
+    "loadRuns": 0,
+    "automaticRetries": 0,
+    "parameterSweeps": 0
+  },
+  "result": "REPRODUCED",
+  "overallAttempt": 2,
+  "freshRepair": 1,
+  "baselineInvocation": 2,
+  "observation": {
+    "observerPid": 77219,
+    "workerPid": 77255,
+    "workerId": "8287a309-a2e5-4b3d-a102-bdf18cb36bd1",
+    "checkpoint": "before_io",
+    "exit": {
+      "code": null,
+      "signal": "SIGKILL",
+      "forced": false
+    },
+    "outboundCalls": 0,
+    "row": {
+      "id": "11000000-0000-4000-a000-000000000001",
+      "monitor_id": "11000000-0000-4000-9000-000000000001",
+      "trigger": "MANUAL",
+      "state": "RUNNING",
+      "http_status": null,
+      "latency_ms": null,
+      "failure_reason": null,
+      "started_at": "2026-08-28T04:48:11.133Z",
+      "finished_at": null,
+      "queued_at": "2026-08-28T00:00:00.000Z",
+      "scheduled_for": null,
+      "request_user_id": "11000000-0000-4000-8000-000000000001",
+      "idempotency_key": "manual-recovery-e11-1",
+      "worker_id": "8287a309-a2e5-4b3d-a102-bdf18cb36bd1"
+    },
+    "finiteLeasePresent": false
+  },
+  "decisiveFailure": "Actual SIGKILL leaves the committed RUNNING row with no persisted finite lease or expiry recovery.",
+  "cleanup": {
+    "schemaDropped": true,
+    "portsFree": true
+  },
+  "durationMs": 518
+}
diff --git a/evidence/phase-1/E11/observer-repair-result.json b/evidence/phase-1/E11/observer-repair-result.json
new file mode 100644
index 0000000..d72d372
--- /dev/null
+++ b/evidence/phase-1/E11/observer-repair-result.json
@@ -0,0 +1,115 @@
+{
+  "thread": "E11",
+  "profile": "phase-1",
+  "specRevision": "2ada57a71cd34fa2fae9809415c362a8bbfcdf02",
+  "threadStart": "591095c24b25efe59258ff07ee93e1b40adeadd4",
+  "repairStart": "b17e00a0e96d7e15f128c20c17edcccd7c73e334",
+  "frozenCommit": "c776c6545274019e78cd056a226ce460a9699262",
+  "overallAttempt": 2,
+  "freshRepair": 1,
+  "status": "CORRECTED_BASELINE_REPRODUCED_AWAITING_ROOT_VERIFICATION",
+  "actualArtifact": "baseline-observer.json",
+  "preservedOriginalFailure": "baseline-attempt1.json",
+  "correctedRunnerSha256": "0f370c1c712618ad36faac8c8e023fb6b4d344af868809cfe0face385d3510d2",
+  "invocation": {
+    "command": "fnm exec --using 24.19.0 node evidence/phase-1/E11/baseline-observer.mjs",
+    "baselineInvocation": 2,
+    "exitCode": 0,
+    "result": "REPRODUCED",
+    "durationMs": 518,
+    "toolWallSeconds": 0.789725459,
+    "toolChunkId": "7d98f6"
+  },
+  "observation": {
+    "checkpoint": "Same frozen committed RUNNING row and worker Monitor SELECT blocked by monitors ACCESS EXCLUSIVE lock, before outbound I/O.",
+    "workerPid": 77255,
+    "workerId": "8287a309-a2e5-4b3d-a102-bdf18cb36bd1",
+    "exit": {
+      "code": null,
+      "signal": "SIGKILL",
+      "forced": false
+    },
+    "outboundCalls": 0,
+    "fullRowEqualityAssertionPassed": true,
+    "persistedState": "RUNNING",
+    "finiteLeasePresent": false,
+    "productMatchesThreadStart": true,
+    "originalsUnchanged": true,
+    "cleanupSignalsAfterObservedExit": 0
+  },
+  "staticChecks": [
+    {
+      "command": "fnm exec --using 24.19.0 node --check test/worker.ts",
+      "exitCode": 0
+    },
+    {
+      "command": "fnm exec --using 24.19.0 node --check evidence/phase-1/E11/baseline-observer.mjs",
+      "exitCode": 0
+    }
+  ],
+  "independentCleanupChecks": [
+    {
+      "name": "cleanup-schema",
+      "exitCode": 0,
+      "output": "0|0\n"
+    },
+    {
+      "name": "cleanup-ports",
+      "exitCode": 1,
+      "output": ""
+    },
+    {
+      "name": "cleanup-processes",
+      "exitCode": 127,
+      "output": "zsh:1: operation not permitted: ps\n"
+    },
+    {
+      "name": "product-invariance",
+      "exitCode": 0,
+      "output": ""
+    },
+    {
+      "name": "original-failure-invariance",
+      "exitCode": 0,
+      "output": ""
+    }
+  ],
+  "authorizedProcessCleanupCheck": {
+    "command": "ps -p 77219,77255 -o pid=,comm=",
+    "exitCode": 1,
+    "output": "",
+    "meaning": "No owned observer or worker PID remains. Initial sandbox denial is retained above; this was only a read-only cleanup query, not a baseline rerun."
+  },
+  "cleanup": {
+    "schemaCount": 0,
+    "workerDatabaseSessions": 0,
+    "reservedPortListeners": 0,
+    "ownedProcessesRemaining": 0
+  },
+  "budget": {
+    "priorFailedBaseline": {
+      "invocation": 1,
+      "exitCode": 1,
+      "durationMs": 727,
+      "artifact": "baseline-attempt1.json"
+    },
+    "correctedBaseline": {
+      "invocation": 2,
+      "exitCode": 0,
+      "durationMs": 518,
+      "artifact": "baseline-observer.json"
+    },
+    "totalBaselineInvocations": 2,
+    "freshRepairsUsed": 1,
+    "maximumFreshRepairs": 2,
+    "observerOnlyStaticChecks": 2,
+    "otherTestInvocations": 0,
+    "loadRuns": 0,
+    "automaticRetries": 0,
+    "parameterSweeps": 0
+  },
+  "scope": "Only the test child-close observer and new E11 evidence changed. No original frozen files, product, runtime, package, data, lease, threshold, or other test source changed.",
+  "unresolved": [
+    "E11 product implementation and acceptance are intentionally unstarted; this bounded repair stops for root verification of the baseline."
+  ]
+}


