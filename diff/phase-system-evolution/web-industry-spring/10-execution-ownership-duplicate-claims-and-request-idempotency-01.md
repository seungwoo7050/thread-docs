# E10 실행 소유권, 중복 Claim과 요청 멱등성

## `test(idempotency): freeze parallel E10 request baseline`

diff --git a/evidence/phase-1/E10/baseline.json b/evidence/phase-1/E10/baseline.json
new file mode 100644
index 0000000..d849031
--- /dev/null
+++ b/evidence/phase-1/E10/baseline.json
@@ -0,0 +1,31 @@
+{
+  "start": "3cc49f3d2a35055c92d0312fca6167c89dfadec5",
+  "fixtureSha256": "8628e42090fb5a71d9f6e4570742b670a63a1e9ce6d5d11603f9c6b03b693649",
+  "completed": [
+    "two actual same-owner/key requests completed after one barrier; database and fixture counts observed"
+  ],
+  "observation": {
+    "preparedRequests": 2,
+    "clientBarrierReleased": true,
+    "workersStarted": 0,
+    "statuses": [
+      202,
+      202
+    ],
+    "states": [
+      "QUEUED",
+      "QUEUED"
+    ],
+    "sameCheckId": false,
+    "persistedCheckRuns": 2,
+    "fixtureCalls": 0
+  },
+  "result": "COUNTEREXAMPLE: retransmission created distinct durable intents",
+  "failure": "E10 requires one ID and durable intent for this same-owner/key pair",
+  "cleanup": {
+    "apiExitAwaited": true,
+    "fixtureClosed": true,
+    "disposableSchemaDropped": true
+  },
+  "elapsedSeconds": 8.036
+}
diff --git a/evidence/phase-1/E10/fixtures.md b/evidence/phase-1/E10/fixtures.md
new file mode 100644
index 0000000..938f10a
--- /dev/null
+++ b/evidence/phase-1/E10/fixtures.md
@@ -0,0 +1,108 @@
+# Phase-1 E10 fixed scenario
+
+Thread E10, attempt1; branch `track/industry-spring`.
+START `3cc49f3d2a35055c92d0312fca6167c89dfadec5`.
+SPEC_REVISION `2ada57a71cd34fa2fae9809415c362a8bbfcdf02`.
+Frozen before the single unchanged-product baseline and all product edits.
+
+## Data and manual request identity
+
+Alice owns A (`http://127.0.0.1:4321/ok`, interval60, enabledtrue) and
+B (`http://127.0.0.1:4321/fail`, interval60, enabledfalse). Runtime UUIDs are
+captured and reused; credentials, sessions and CSRF values stay in memory.
+Each scenario starts with these two Monitors and no CheckRuns. The baseline
+uses disposable e04_restart; the Java proof uses e10_ownership; the browser
+uses the existing e04_browser harness. No public schema or volume reset.
+
+The identity is authenticated owner plus header `Idempotency-Key` with value
+`manual-intent-e10-1`. Its meaning is the target Monitor ID, not mutable name,
+URL, interval, enabled flag or current execution state. Requests are POSTs to
+the existing nested Check endpoint, empty body, exact trusted Origin4323 and
+valid session-bound CSRF. All previous authentication and owner boundaries stay.
+
+## One baseline
+
+Workers are stopped. Prepare both authenticated A requests before releasing
+one client barrier, then await both actual HTTP responses. Record statuses,
+identity equality, durable CheckRun count and actual fixture request count.
+The two endpoint handlers return the existing /ok200 and /fail503 outcomes.
+Stop on a decisive duplicate ID/row observation; do not run a second baseline
+or a claim race against START. No expected result is substituted for an HTTP
+response or database count.
+
+## Post-change manual sequence
+
+1. Stop workers and hold a PostgreSQL SHARE table lock on check_runs. Prepare
+   the same two A requests at a client barrier, release them, and observe both
+   insert transactions waiting for the table lock before releasing it. Require
+  202 responses, the same ID and current QUEUED state, exactlyone durable row
+   and zero fixture calls.
+2. Reuse the same key for Alice's B:409 CONFLICT, no B intent or fixture call.
+   Change only A's name to `A edited`, interval to120 and enabled tofalse;
+   replay its same key and require the same current ID without another row.
+3. Missing key, empty key, `has space`, non-ASCII `é`, and129 ASCII characters
+   each produce400 INVALID_INPUT and no new row or fixture call. A128-character
+   printable key is valid (tested after the single-run worker proof).
+4. Replay the first key while RUNNING and after SUCCEEDED:202 with that same
+   current ID/state. A different valid key is a new intentional A execution;
+   its replay is not another intent. The original key must still identify the
+   original execution. No application cache is the source of identity.
+
+## Two actual workers and deterministic claim proof
+
+Exactlyone QUEUED row exists when this part begins. Two separate non-web JVMs
+load the production application and invoke its actual CheckWorker once; the
+scheduled ticker is not enabled in this controlled test. A test-only Java entry
+point acquires a shared PostgreSQL advisory gate after application startup.
+The coordinator owns the exclusive gate and observes both distinct processes'
+ungranted shared locks before release. The gate does not serialize the workers
+after release. No production pause, lease or retry hook is added.
+
+The real /ok handler counts each request but withholds all response headers
+until an explicit test latch release. After release of the worker gate, require
+one committed RUNNING row with an execution owner and one held outbound request.
+The losing worker waits at a second PostgreSQL gate. Only after the fixture has
+seen the winner and the database shows RUNNING does the coordinator release
+that gate. The loser attempts the production terminal update with an all-zero
+non-owner identity; require zero changed rows and the original RUNNING/null
+outcome to remain. Then release /ok with the original200 response. Require
+exactlyone winning worker, one losing worker, one outbound request, one same-ID
+SUCCEEDED/HTTP200 row, and both child exits awaited. No repeated race loops.
+
+Startup/readiness observations have bounded30-second waits and25ms polling;
+these are test synchronization bounds, not performance acceptance thresholds.
+The existing1-second connect and2-second response timeout remain unchanged.
+All held barriers/responses are released and owned children stopped in finally.
+
+## Browser intention and retransmission
+
+Reuse the ordinary production-browser/login/worker harness. In one browser case,
+create A/B with the same initial inputs. Forward the first real A Check POST to
+the server, retain its actual accepted ID, hold its response and dispatch a
+second click before acknowledgement: onlyone request may leave the browser.
+Then abort delivery of that already processed response, preserving the actual
+database outcome. The page must expose failure and release pending state.
+An explicit retry must send the same generated key, return202 with the same
+current ID, and leave one durable A row. After that acknowledgement a new click
+must use a different key and create exactlyone additional A row. Observe actual
+server responses and rows; no fabricated acceptance body. Keys are compared in
+memory and evidence records only equality/validity flags. Old E09 browser
+lifecycle assertions remain unchanged.
+
+## Necessary legacy adapters and bounds
+
+Existing manual Check test consumers gain a fresh valid key for each original
+intent, without changing their request data, expected rows or statuses. The
+E05 two-owner initial Check setup may use the same key for Alice and Bob to
+prove owner-scoped identity while retaining its original two-row dataset.
+Existing migration legs stay explicitly pinned to their original target before
+any new migration assertion; prior evidence and V1–V6 files stay immutable.
+Terminal history remains finished_at/id descending and all security matrices
+retain their original denied-write/outbound assertions. Scheduler slots retain
+their existing identity and enabled/disabled semantics.
+
+Baseline1; load runs0; automatic test retries0; parameter sweeps0. Authoring uses
+only affected checks; root owns one complete final regression. Every actual
+invocation and failure is recorded. No E11 recovery/leases, phase-2 policy,
+credentials, session/CSRF values, private bodies, browser captures or storage
+state in evidence.
diff --git a/scripts/e10-baseline.mjs b/scripts/e10-baseline.mjs
new file mode 100644
index 0000000..d452b05
--- /dev/null
+++ b/scripts/e10-baseline.mjs
@@ -0,0 +1,139 @@
+import assert from 'node:assert/strict';
+import { spawn, spawnSync, execFileSync } from 'node:child_process';
+import { createHash, randomBytes } from 'node:crypto';
+import { once } from 'node:events';
+import { appendFileSync, closeSync, mkdirSync, openSync, readFileSync, writeFileSync } from 'node:fs';
+import { createServer as httpServer } from 'node:http';
+import { createServer as portProbe } from 'node:net';
+import { setTimeout as delay } from 'node:timers/promises';
+import { bootstrapUsers } from './bootstrap-users.mjs';
+
+const start = '3cc49f3d2a35055c92d0312fca6167c89dfadec5';
+assert.equal(execFileSync('git', ['rev-parse', 'HEAD'], { encoding: 'utf8' }).trim(), start);
+assert.equal(execFileSync('git', ['diff', '--name-only', start, '--', 'app', 'backend/src/main',
+  'package.json', 'package-lock.json'], { encoding: 'utf8' }).trim(), '');
+const directory = 'output/phase-1/e10';
+mkdirSync(directory, { recursive: true });
+const started = Date.now();
+const evidence = { start, fixtureSha256: createHash('sha256')
+  .update(readFileSync('evidence/phase-1/E10/fixtures.md')).digest('hex'), completed: [] };
+const credentials = { E04_ALICE_PASSWORD: randomBytes(32).toString('base64url'),
+  E04_BOB_PASSWORD: randomBytes(32).toString('base64url') };
+let api;
+let cookie;
+let fixtureCalls = 0;
+let exitCode = 0;
+const fixture = httpServer((request, response) => {
+  if (request.url === '/ok' || request.url === '/fail') fixtureCalls++;
+  response.setHeader('Content-Type', 'text/plain');
+  if (request.url === '/ok') response.writeHead(200).end('ok\n');
+  else if (request.url === '/fail') response.writeHead(503).end('unavailable\n');
+  else response.writeHead(404).end('not found\n');
+});
+function database(action) {
+  assert.equal(spawnSync(process.execPath, ['scripts/database.mjs', action, 'e04_restart'],
+    { stdio: 'ignore' }).status, 0, `Disposable database ${action} must succeed`);
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
+  assert.equal(data.headerName, 'X-CSRF-TOKEN');
+  return { Cookie: cookie, [data.headerName]: data.token, Origin: 'http://127.0.0.1:4323' };
+}
+async function post(path, body) {
+  const response = await fetch(`http://127.0.0.1:4322${path}`, {
+    method: 'POST', headers: { ...(await csrf()), 'Content-Type': 'application/json' },
+    body: body === undefined ? undefined : JSON.stringify(body), signal: AbortSignal.timeout(10_000),
+  });
+  const issued = response.headers.get('set-cookie');
+  if (issued) cookie = issued.split(';', 1)[0];
+  return { status: response.status, data: (await response.json()).data };
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
+  const logPath = `${directory}/baseline-api.log`;
+  const log = openSync(logPath, 'w');
+  const { E04_ALICE_PASSWORD, E04_BOB_PASSWORD, ...runtime } = process.env;
+  api = spawn('java', ['-jar', 'backend/target/monitor-api-0.0.1.jar'], {
+    env: { ...runtime, DB_SCHEMA: 'e04_restart' }, stdio: ['ignore', log, log],
+  });
+  closeSync(log);
+  const deadline = Date.now() + 30_000;
+  let ready = false;
+  while (Date.now() < deadline) {
+    assert.ok(api.exitCode === null && api.signalCode === null, 'Owned API must remain alive');
+    if (readFileSync(logPath, 'utf8').includes('Started MonitorApplication')) {
+      try { ready = (await fetch('http://127.0.0.1:4322/api/session')).status === 401; } catch {}
+    }
+    if (ready) break;
+    await delay(25);
+  }
+  assert.ok(ready, 'Owned API readiness must complete');
+  assert.equal((await post('/api/session/login', { username: 'alice-e04', password: credentials.E04_ALICE_PASSWORD })).status, 200);
+  const a = await post('/api/monitors', { name: 'A', url: 'http://127.0.0.1:4321/ok', interval: 60, enabled: true });
+  const b = await post('/api/monitors', { name: 'B', url: 'http://127.0.0.1:4321/fail', interval: 60, enabled: false });
+  assert.equal(a.status, 201);
+  assert.equal(b.status, 201);
+  const headers = { ...(await csrf()), 'Idempotency-Key': 'manual-intent-e10-1' };
+  let release;
+  const barrier = new Promise(resolve => { release = resolve; });
+  const requests = [0, 1].map(async () => {
+    await barrier;
+    const response = await fetch(`http://127.0.0.1:4322/api/monitors/${a.data.monitor.id}/checks`, {
+      method: 'POST', headers, signal: AbortSignal.timeout(10_000),
+    });
+    return { status: response.status, data: (await response.json()).data };
+  });
+  release();
+  const results = await Promise.allSettled(requests);
+  assert.ok(results.every(result => result.status === 'fulfilled'), 'Both actual requests must complete');
+  const responses = results.map(result => result.value);
+  const count = Number(execFileSync('docker', ['compose', '--project-name', 'wse-industry', '--file', 'compose.yaml',
+    'exec', '--no-TTY', 'postgres', 'psql', '--username', 'wse_industry', '--dbname', 'monitor', '--tuples-only',
+    '--no-align', '--command', 'SELECT count(*) FROM e04_restart.check_runs;'], { encoding: 'utf8' }).trim());
+  evidence.observation = { preparedRequests: 2, clientBarrierReleased: true, workersStarted: 0,
+    statuses: responses.map(response => response.status), states: responses.map(response => response.data?.state),
+    sameCheckId: responses[0].data?.id === responses[1].data?.id, persistedCheckRuns: count, fixtureCalls };
+  evidence.completed.push('two actual same-owner/key requests completed after one barrier; database and fixture counts observed');
+  assert.ok(responses.every(response => response.status === 202 && response.data.state === 'QUEUED'));
+  evidence.result = evidence.observation.sameCheckId && count === 1 ? 'GUARANTEE_ALREADY_HOLDS'
+    : 'COUNTEREXAMPLE: retransmission created distinct durable intents';
+  assert.ok(evidence.observation.sameCheckId && count === 1, 'E10 requires one ID and durable intent for this same-owner/key pair');
+} catch (error) {
+  exitCode = 1;
+  evidence.failure = error.message;
+  console.error(error.message);
+} finally {
+  if (api && api.exitCode === null && api.signalCode === null) {
+    const exited = once(api, 'exit');
+    api.kill('SIGTERM');
+    const force = setTimeout(() => api.kill('SIGKILL'), 5000);
+    await exited;
+    clearTimeout(force);
+  }
+  if (fixture.listening) await new Promise(resolve => fixture.close(resolve));
+  database('drop');
+  evidence.cleanup = { apiExitAwaited: !!api, fixtureClosed: !fixture.listening, disposableSchemaDropped: true };
+  evidence.elapsedSeconds = (Date.now() - started) / 1000;
+  writeFileSync(`${directory}/baseline.json`, JSON.stringify(evidence, null, 2) + '\n');
+  appendFileSync(`${directory}/invocations.jsonl`, JSON.stringify({ command: 'node scripts/e10-baseline.mjs',
+    startedAt: new Date(started).toISOString(), elapsedSeconds: evidence.elapsedSeconds, exitCode }) + '\n');
+  process.exitCode = exitCode;
+}


