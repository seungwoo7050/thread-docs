# E11 Worker Crash Recovery, Lease와 Graceful Shutdown

## `test: freeze E11 crash checkpoints and stranded-run baseline`

diff --git a/evidence/phase-1/E11/baseline-invocations.jsonl b/evidence/phase-1/E11/baseline-invocations.jsonl
new file mode 100644
index 0000000..f6cf99d
--- /dev/null
+++ b/evidence/phase-1/E11/baseline-invocations.jsonl
@@ -0,0 +1 @@
+{"command":"node scripts/e11-baseline.mjs","startedAt":"2026-08-28T06:12:37.228Z","elapsedSeconds":13.661,"exitCode":1}
diff --git a/evidence/phase-1/E11/baseline.json b/evidence/phase-1/E11/baseline.json
new file mode 100644
index 0000000..02540ff
--- /dev/null
+++ b/evidence/phase-1/E11/baseline.json
@@ -0,0 +1,83 @@
+{
+  "start": "d51673b78cd4702584741e12f80c15af9f34cd4d",
+  "fixtureSha256": "38362668a1c4ae3f544cdf42f941df978a587e615cc1f40e93b86a56dd520003",
+  "completed": [
+    "actual SIGKILL exit awaited; replacement normal worker started; fixed expiry/tick observation completed"
+  ],
+  "processes": [
+    {
+      "label": "api",
+      "pid": 25917,
+      "entry": "API jar",
+      "exit": {
+        "code": 143,
+        "signal": null,
+        "awaited": true
+      }
+    },
+    {
+      "label": "first-worker",
+      "pid": 25969,
+      "entry": "normal worker profile",
+      "exit": {
+        "code": null,
+        "signal": "SIGKILL",
+        "awaited": true
+      }
+    },
+    {
+      "label": "replacement-worker",
+      "pid": 25995,
+      "entry": "normal worker profile",
+      "exit": {
+        "code": 143,
+        "signal": null,
+        "awaited": true
+      }
+    }
+  ],
+  "signals": [
+    {
+      "pid": 25969,
+      "signal": "SIGKILL",
+      "purpose": "baseline durable RUNNING while response withheld"
+    },
+    {
+      "pid": 25995,
+      "signal": "SIGTERM",
+      "purpose": "owned baseline cleanup"
+    },
+    {
+      "pid": 25917,
+      "signal": "SIGTERM",
+      "purpose": "owned baseline cleanup"
+    }
+  ],
+  "beforeKill": {
+    "id": "ba59f981-53b3-4986-a7a3-b3df926f4b57",
+    "state": "RUNNING",
+    "httpStatus": null,
+    "claimOwner": "bc566775-e95c-481a-91e2-2fffbc97208b",
+    "startedAt": "2026-08-28T06:12:45.541152+00:00",
+    "fixtureCalls": 1
+  },
+  "afterReplacement": {
+    "id": "ba59f981-53b3-4986-a7a3-b3df926f4b57",
+    "state": "RUNNING",
+    "httpStatus": null,
+    "finishedAt": null,
+    "sameId": true,
+    "sameRow": true,
+    "fixtureCalls": 1,
+    "observedAt": "2026-08-28T06:12:50.664Z",
+    "requiredNotBefore": "2026-08-28T06:12:50.541Z"
+  },
+  "result": "COUNTEREXAMPLE: dead owner leaves a durable RUNNING execution stranded",
+  "failure": "E11 requires unknown dead-owner execution to recover as ABORTED\n+ actual - expected\n\n+ 'RUNNING'\n- 'ABORTED'\n",
+  "cleanup": {
+    "ownedExitsAwaited": true,
+    "fixtureClosed": true,
+    "disposableSchemaDropped": true
+  },
+  "elapsedSeconds": 13.661
+}
diff --git a/evidence/phase-1/E11/fixtures.md b/evidence/phase-1/E11/fixtures.md
new file mode 100644
index 0000000..b608bdf
--- /dev/null
+++ b/evidence/phase-1/E11/fixtures.md
@@ -0,0 +1,95 @@
+# E11 fixed crash and shutdown contract
+
+Frozen before product edits and the one unchanged-product baseline.
+START d51673b78cd4702584741e12f80c15af9f34cd4d; profile phase-1;
+Spec-Revision 2ada57a71cd34fa2fae9809415c362a8bbfcdf02.
+
+## Data and clock
+
+Use the existing two runtime-only users and session/Origin4323/CSRF helpers.
+Alice owns A (http://127.0.0.1:4321/ok, interval60, disabled) and B
+(http://127.0.0.1:4321/fail, interval60, disabled). Start with no CheckRuns.
+Each manual request has its own fixed printable key. The baseline uses
+e04_restart; the process proof uses e11_recovery; public and volumes are untouched.
+The fixture counts actual requests, returns /fail503, and holds /ok headers
+until a coordinator release. No endpoint body is needed.
+
+Lease duration is exactly5000ms from the durable started_at. The recovery clock
+is the stored lease_expires_at: first invoke recovery at expiry minus1microsecond
+(zero changes), then at exact expiry. This explicit clock is independent of
+JVM startup time. For START, which has no lease, observe after started_at+5000ms
+and a replacement worker's readiness plus500ms (two250ms tick opportunities).
+Startup/file/database observations have a30-second watchdog and25ms polling;
+they are synchronization limits, not lease extensions or performance targets.
+The original connect1000ms/read2000ms endpoint timeouts stay unchanged.
+
+## One baseline
+
+Run the unchanged built artifact. Create A/B, accept A with key e11-baseline,
+start one normal non-web worker with scheduling disabled, and wait for the
+actual held /ok request and committed RUNNING row. Send one real SIGKILL and
+await that process's exit. Start a replacement normal worker. At the frozen
+observation clock record the same row/ID, outcome fields and request count.
+Stop at the first decisive stranded-RUNNING observation. Do not rerun START.
+
+## Four post-change SIGKILL checkpoints, one execution each
+
+Keys are e11-before-io, e11-during-io, e11-before-commit and e11-after-commit.
+Each separate non-web JVM invokes the production CheckWorker once; a test-only
+startup adapter supplies barriers, not alternate claim/completion logic.
+
+1. After startNext has returned through its transactional proxy (durable claim),
+   pause at CheckRunner.run entry before outbound I/O. Require zero requests.
+2. After the real /ok handler receives the request, withhold response headers.
+   Require one request and committed RUNNING/null outcome before SIGKILL.
+3. Let the real runner observe HTTP200. Pause inside the production finish
+   transaction after its SQL update and before commit. Record active transaction,
+   JDBC autoCommit=false, PostgreSQL backend PID/transaction and observed200.
+   An independent connection must still see RUNNING. After SIGKILL await the
+   actual JVM exit and disappearance of its database session/rollback before
+   recovery. No blocked autocommit UPDATE is released after the worker dies.
+4. Pause only after executeNext returns and a separate connection observes its
+   committed SUCCEEDED/HTTP200 row. Kill once and await exit.
+
+For the first three, the exact expiry changes one same-ID row to ABORTED with
+null HTTP status, latency and failure reason; no endpoint result is invented.
+Repeat recovery is a no-op. Attempt the actual finish operation with the recorded
+dead owner identity and a valid200 result: zero changed rows, byte-identical
+terminal result. The fourth case's terminal row and ID survive expiry unchanged,
+with no new request. Original keys return their original current terminal rows.
+A new explicit key e11-new-intent creates a new QUEUED ID, never resets an old row.
+There is no automatic retry, lease renewal or replay of an expired run.
+
+## One real SIGTERM case
+
+Use the normal worker profile and its actual scheduled loop, scheduling disabled.
+Start it before requests, then accept e11-drain on A, observe one held request,
+and accept e11-still-queued on B while A is in flight. Send SIGTERM exactlyonce.
+An observation-only ContextClosedEvent listener records that claims are stopped;
+then release A's actual HTTP200. Policy: drain the current operation for at most
+3000ms via Spring's lifecycle shutdown phase, with no new claims. If unfinished
+at the deadline, shutdown leaves an unknown RUNNING result for finite lease
+recovery to ABORTED, never fabricates endpoint failure. The fixed in-flight case
+must drain to SUCCEEDED/200 and actually exit within3000ms of SIGTERM. B stays
+QUEUED with no owner/start/outcome and zero /fail requests. Do not use another
+signal as a shutdown observation. Await owned exits; record cleanup signals
+separately from checkpoint signals.
+
+## Minimal regression and evidence scope
+
+V1–V7 and all previous evidence stay immutable. V8 preserves original rows,
+timestamps, request identities and checksums, adds finite lease metadata to
+existing RUNNING rows, and supports ABORTED without endpoint outcome fields.
+History keeps finished_at/id descending. All existing owner, CSRF, Origin,
+idempotency and scheduler conditions remain. The browser must parse/display
+ABORTED as terminal; its current polling/history invalidation may be narrowly
+extended. Any browser observation uses the existing production harness, with
+an explicitly seeded expired RUNNING row and actual normal-worker recovery;
+it is not an additional crash checkpoint or substitute for the real kills.
+
+Baseline1, each four-checkpoint scenario1 per approved affected gate, SIGTERM1
+per approved affected gate; author uses the cheapest affected gate and root owns
+one final regression. Every invocation, failure and process is recorded. Load0,
+automatic retries0, parameter sweeps0. Stop on an unexpected acceptance/build
+failure; do not fix or rerun without root's bounded repair dispatch. No credentials,
+session/CSRF values, private bodies, browser captures or storage state in evidence.
diff --git a/scripts/e11-baseline.mjs b/scripts/e11-baseline.mjs
new file mode 100644
index 0000000..2eb0422
--- /dev/null
+++ b/scripts/e11-baseline.mjs
@@ -0,0 +1,157 @@
+import assert from 'node:assert/strict';
+import { spawn, spawnSync, execFileSync } from 'node:child_process';
+import { createHash, randomBytes } from 'node:crypto';
+import { once } from 'node:events';
+import { appendFileSync, closeSync, mkdirSync, openSync, readFileSync, writeFileSync } from 'node:fs';
+import { createServer } from 'node:http';
+import { createServer as portProbe } from 'node:net';
+import { setTimeout as delay } from 'node:timers/promises';
+import { bootstrapUsers } from './bootstrap-users.mjs';
+
+const start = 'd51673b78cd4702584741e12f80c15af9f34cd4d';
+assert.equal(execFileSync('git', ['rev-parse', 'HEAD'], { encoding: 'utf8' }).trim(), start);
+assert.equal(execFileSync('git', ['diff', '--name-only', start, '--', 'app', 'backend/src/main',
+  'package.json', 'package-lock.json'], { encoding: 'utf8' }).trim(), '');
+const directory = 'output/phase-1/e11';
+mkdirSync(directory, { recursive: true });
+const started = Date.now();
+const evidence = { start, fixtureSha256: createHash('sha256')
+  .update(readFileSync('evidence/phase-1/E11/fixtures.md')).digest('hex'), completed: [], processes: [], signals: [] };
+const credentials = { E04_ALICE_PASSWORD: randomBytes(32).toString('base64url'),
+  E04_BOB_PASSWORD: randomBytes(32).toString('base64url') };
+const children = [];
+const held = new Set();
+let cookie;
+let fixtureCalls = 0;
+let exitCode = 0;
+const fixture = createServer((request, response) => {
+  if (request.url === '/ok') { fixtureCalls++; held.add(response); }
+  else if (request.url === '/fail') { fixtureCalls++; response.writeHead(503).end(); }
+  else response.writeHead(404).end();
+});
+function database(action) {
+  assert.equal(spawnSync(process.execPath, ['scripts/database.mjs', action, 'e04_restart'],
+    { stdio: 'ignore' }).status, 0, `Disposable database ${action} must succeed`);
+}
+function row() {
+  const text = execFileSync('docker', ['compose', '--project-name', 'wse-industry', '--file', 'compose.yaml',
+    'exec', '--no-TTY', 'postgres', 'psql', '--username', 'wse_industry', '--dbname', 'monitor', '--tuples-only',
+    '--no-align', '--command', 'SELECT row_to_json(c) FROM e04_restart.check_runs c;'], { encoding: 'utf8' }).trim();
+  return text ? JSON.parse(text) : null;
+}
+async function free(port) {
+  const probe = portProbe();
+  await new Promise((resolve, reject) => {
+    probe.once('error', () => reject(new Error(`Refusing occupied port ${port}`)));
+    probe.listen({ host: '127.0.0.1', port, exclusive: true }, () => probe.close(resolve));
+  });
+}
+async function csrf() {
+  const response = await fetch('http://127.0.0.1:4322/api/session/csrf', { headers: cookie ? { Cookie: cookie } : {} });
+  assert.equal(response.status, 200);
+  const issued = response.headers.get('set-cookie');
+  if (issued) cookie = issued.split(';', 1)[0];
+  const { data } = await response.json();
+  return { Cookie: cookie, [data.headerName]: data.token, Origin: 'http://127.0.0.1:4323' };
+}
+async function post(path, body, key) {
+  const response = await fetch(`http://127.0.0.1:4322${path}`, {
+    method: 'POST', headers: { ...(await csrf()), 'Content-Type': 'application/json',
+      ...(key ? { 'Idempotency-Key': key } : {}) },
+    body: body === undefined ? undefined : JSON.stringify(body), signal: AbortSignal.timeout(10_000),
+  });
+  const issued = response.headers.get('set-cookie');
+  if (issued) cookie = issued.split(';', 1)[0];
+  return { status: response.status, data: (await response.json()).data };
+}
+function launch(label, worker) {
+  const logPath = `${directory}/baseline-${label}.log`;
+  const log = openSync(logPath, 'w');
+  const { E04_ALICE_PASSWORD, E04_BOB_PASSWORD, ...runtime } = process.env;
+  const args = ['-jar', 'backend/target/monitor-api-0.0.1.jar', ...(worker ? ['--spring.profiles.active=worker',
+    '--spring.main.web-application-type=none', '--monitor.scheduler-enabled=false'] : [])];
+  const child = spawn('java', args, { env: { ...runtime, DB_SCHEMA: 'e04_restart' }, stdio: ['ignore', log, log] });
+  closeSync(log);
+  children.push(child);
+  evidence.processes.push({ label, pid: child.pid, entry: worker ? 'normal worker profile' : 'API jar' });
+  return { process: child, logPath };
+}
+async function until(observed, message) {
+  const deadline = Date.now() + 30_000;
+  while (Date.now() < deadline) {
+    if (await observed()) return;
+    await delay(25);
+  }
+  assert.fail(message);
+}
+async function ready(child) {
+  await until(() => {
+    assert.ok(child.process.exitCode === null && child.process.signalCode === null, 'Owned child remains alive');
+    return readFileSync(child.logPath, 'utf8').includes('Started MonitorApplication');
+  }, 'Owned JVM startup must complete');
+}
+async function signalAndAwait(child, signal, purpose) {
+  if (child.exitCode !== null || child.signalCode !== null) return;
+  const exited = once(child, 'exit');
+  evidence.signals.push({ pid: child.pid, signal, purpose });
+  assert.ok(child.kill(signal));
+  const [code, observedSignal] = await exited;
+  evidence.processes.find(value => value.pid === child.pid).exit = { code, signal: observedSignal, awaited: true };
+}
+try {
+  await free(4321);
+  await free(4322);
+  database('reset');
+  bootstrapUsers('e04_restart', credentials);
+  await new Promise((resolve, reject) => {
+    fixture.once('error', reject);
+    fixture.listen(4321, '127.0.0.1', resolve);
+  });
+  const api = launch('api', false);
+  await ready(api);
+  assert.equal((await post('/api/session/login', { username: 'alice-e04', password: credentials.E04_ALICE_PASSWORD })).status, 200);
+  const a = await post('/api/monitors', { name: 'A', url: 'http://127.0.0.1:4321/ok', interval: 60, enabled: false });
+  const b = await post('/api/monitors', { name: 'B', url: 'http://127.0.0.1:4321/fail', interval: 60, enabled: false });
+  assert.equal(a.status, 201);
+  assert.equal(b.status, 201);
+  const accepted = await post(`/api/monitors/${a.data.monitor.id}/checks`, undefined, 'e11-baseline');
+  assert.equal(accepted.status, 202);
+  assert.equal(accepted.data.state, 'QUEUED');
+  const first = launch('first-worker', true);
+  await until(() => fixtureCalls === 1, 'Worker must make the held real request');
+  const running = row();
+  assert.equal(running.id, accepted.data.id);
+  assert.equal(running.state, 'RUNNING');
+  assert.equal(running.http_status, null);
+  evidence.beforeKill = { id: running.id, state: running.state, httpStatus: running.http_status,
+    claimOwner: running.claim_owner, startedAt: running.started_at, fixtureCalls };
+  await signalAndAwait(first.process, 'SIGKILL', 'baseline durable RUNNING while response withheld');
+  const replacement = launch('replacement-worker', true);
+  await ready(replacement);
+  const observeAt = Math.max(Date.parse(running.started_at) + 5000, Date.now() + 500);
+  await delay(Math.max(0, observeAt - Date.now()));
+  const observed = row();
+  evidence.afterReplacement = { id: observed.id, state: observed.state, httpStatus: observed.http_status,
+    finishedAt: observed.finished_at, sameId: observed.id === running.id,
+    sameRow: JSON.stringify(observed) === JSON.stringify(running), fixtureCalls,
+    observedAt: new Date().toISOString(), requiredNotBefore: new Date(observeAt).toISOString() };
+  evidence.completed.push('actual SIGKILL exit awaited; replacement normal worker started; fixed expiry/tick observation completed');
+  evidence.result = observed.state === 'RUNNING' ? 'COUNTEREXAMPLE: dead owner leaves a durable RUNNING execution stranded'
+    : 'NO_STRANDED_RUNNING_OBSERVED';
+  assert.equal(observed.state, 'ABORTED', 'E11 requires unknown dead-owner execution to recover as ABORTED');
+} catch (error) {
+  exitCode = 1;
+  evidence.failure = error.message;
+  console.error(error.message);
+} finally {
+  for (const response of held) response.destroy();
+  for (const child of children.toReversed()) await signalAndAwait(child, 'SIGTERM', 'owned baseline cleanup');
+  if (fixture.listening) await new Promise(resolve => fixture.close(resolve));
+  database('drop');
+  evidence.cleanup = { ownedExitsAwaited: true, fixtureClosed: !fixture.listening, disposableSchemaDropped: true };
+  evidence.elapsedSeconds = (Date.now() - started) / 1000;
+  writeFileSync(`${directory}/baseline.json`, JSON.stringify(evidence, null, 2) + '\n');
+  appendFileSync(`${directory}/invocations.jsonl`, JSON.stringify({ command: 'node scripts/e11-baseline.mjs',
+    startedAt: new Date(started).toISOString(), elapsedSeconds: evidence.elapsedSeconds, exitCode }) + '\n');
+  process.exitCode = exitCode;
+}


