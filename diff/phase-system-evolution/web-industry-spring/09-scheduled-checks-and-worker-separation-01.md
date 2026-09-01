# E09 예약 Check와 Worker 분리

## `test(worker): freeze held synchronous E09 baseline`

diff --git a/evidence/phase-1/E09/baseline.json b/evidence/phase-1/E09/baseline.json
new file mode 100644
index 0000000..7bf298c
--- /dev/null
+++ b/evidence/phase-1/E09/baseline.json
@@ -0,0 +1,22 @@
+{
+  "start": "0330d7741bbc0193af193be70d9dac47bfdb134f",
+  "fixtureSha256": "35d362f1f6c80106e44e44b73ce58fe3c2fef5e7d0b6318ea1bbc6d8018ca6bb",
+  "completed": [
+    "one held manual request observed before release and its actual response recorded after HTTP200 release"
+  ],
+  "whileHeld": {
+    "apiResponded": false,
+    "persistedCheckRuns": 0,
+    "outboundRequests": 1,
+    "fixtureReleased": false
+  },
+  "afterRelease": {
+    "status": 200,
+    "state": "SUCCEEDED",
+    "httpStatus": 200,
+    "hasDurableId": true
+  },
+  "result": "COUNTEREXAMPLE: synchronous terminal response instead of persisted202 QUEUED",
+  "failure": "E09 requires persisted202 QUEUED before outbound execution\n\n200 !== 202\n",
+  "elapsedSeconds": 10.385
+}
diff --git a/evidence/phase-1/E09/fixtures.md b/evidence/phase-1/E09/fixtures.md
new file mode 100644
index 0000000..ae91492
--- /dev/null
+++ b/evidence/phase-1/E09/fixtures.md
@@ -0,0 +1,86 @@
+# Phase-1 E09 frozen queue, worker and scheduler case
+
+Thread E09, attempt1. START `0330d7741bbc0193af193be70d9dac47bfdb134f`.
+SPEC_REVISION `2ada57a71cd34fa2fae9809415c362a8bbfcdf02`.
+Branch `track/industry-spring`; profile `phase-1`.
+Frozen before the single unchanged-product baseline or product edits.
+
+## Data and environment
+
+Use owned PostgreSQL15432/project wse-industry and disposable e04_restart for the
+baseline, e04_browser for the browser lifecycle, and an explicitly isolated Java
+test schema for the scheduler. Preserve public and the volume. Use Node24.19.0,
+Java21.0.7+6/Maven3.9.11, the existing session/CSRF helpers, production Next and
+pinned Chromium. Runtime-only Alice/Bob credentials are never retained.
+
+Each fixed case begins with exactly two Alice-owned Monitors, no CheckRuns:
+
+| Monitor | URL | Interval | Enabled |
+| --- | --- | --- | --- |
+| A | http://127.0.0.1:4321/hold | 60 seconds | true |
+| B | http://127.0.0.1:4321/ok | 60 seconds | false |
+
+IDs are runtime UUIDs, reused consistently within the case. The existing fixture's
+old routes remain unchanged. Its new /hold withholds all headers until one explicit
+test POST to /__e09/release. GET /__e09/status exposes only hold request/connection
+counts. These control requests carry no product credentials. No fixed sleep or
+machine-dependent API latency threshold is an acceptance criterion. The existing
+two-second outbound read timeout remains unchanged; release follows the required
+state checkpoints, not a chosen duration.
+
+## One unchanged synchronous baseline
+
+Use the existing root-verified backend artifact at START, with no worker. Create
+A and B through the real authenticated API. Send exactly one manual A Check POST
+and retain its pending response. At the fixture's first held-request checkpoint,
+record whether the API responded and the number of durable A CheckRuns. Release
+the fixture with HTTP200 and record the actual API status and terminal result.
+Require the E09 boundary202/QUEUED; an observed synchronous200/terminal result is
+the decisive baseline counterexample. Stop after that one observation. Do not
+repeat the baseline or run exploratory failure/timeout variants.
+
+## Fixed post-change lifecycle and browser checkpoints
+
+1. Start the real API with no worker and create A/B. Through production Chromium,
+   submit exactly one manual A Check. Require202/QUEUED with one persisted ID,
+   null execution/result fields and zero /hold outbound requests. The browser
+   shows QUEUED and remains usable; no queue is stored outside PostgreSQL.
+2. Start one distinct JVM worker process against the same disposable schema,
+   with automatic scheduling disabled only for this controlled manual case.
+   At the first held /hold request, require the same durable ID in RUNNING,
+   startedAt present, terminal fields still null, and one outbound request.
+   Require the browser to show RUNNING for that ID. Edit the create form's Name
+   input without submitting it to prove the page remains usable while I/O is held.
+3. Release exactly once with HTTP200. Require the same ID to become
+   SUCCEEDED/httpStatus200 with observed latency/finishedAt and one durable row.
+   Require that same browser result, then its terminal history row and reload.
+   No success/failure may be fabricated before release. Settle requests and stop
+   the owned worker before dropping the disposable schema.
+
+The browser may poll authoritative current-execution reads. Terminal history
+retains the exact E07 finished_at/id descending ordering, finite cap, filters and
+cursor conditions. QUEUED/RUNNING are exposed by current/latest and direct reads;
+they are not inserted into the completed-history pagination. Existing terminal
+fixtures, IDs, timestamps, assertions and committed evidence remain unchanged.
+Only strictly necessary202/worker-await setup adapters are permitted for older
+synchronous consumers, preserving their final-state/security assertions.
+
+## Fixed scheduler clock
+
+Use A/B with creation time `2026-08-27T23:59:00.000Z` and interval60. Invoke the
+actual scheduler transaction at `2026-08-28T00:00:00.000Z`, repeat that identical
+tick once, then invoke `2026-08-28T00:01:00.000Z`. Assert persisted SCHEDULED A
+intent counts1,1,2 and B count0 at every checkpoint, with distinct due slots,
+QUEUED state and no outbound I/O. Scheduled due-slot identity must be unique in
+PostgreSQL. Distinct manual and scheduled intents may overlap; no Monitor-wide
+exclusion, competing-worker proof, lease/recovery or idempotency-key policy is added.
+
+## Budget and cleanup
+
+One baseline; load runs0, automatic retries0, parameter sweeps0. Record every test
+invocation/failure and partial completion honestly. Use cheapest affected author
+checks; root owns the independent final acceptance gate. No E10/E11/E12, phase-2,
+Kafka/Redis, broad worker framework or new dependency. Refuse occupied ports,
+settle held requests, await owned process exits and drop only disposable schemas.
+No old evidence/migration changes, main/spec/index/tag edits, credentials, private
+bodies, browser captures/storage-state artifacts or push.
diff --git a/scripts/e09-baseline.mjs b/scripts/e09-baseline.mjs
new file mode 100644
index 0000000..dce9c99
--- /dev/null
+++ b/scripts/e09-baseline.mjs
@@ -0,0 +1,138 @@
+import assert from 'node:assert/strict';
+import { spawn, spawnSync, execFileSync } from 'node:child_process';
+import { createHash, randomBytes } from 'node:crypto';
+import { once } from 'node:events';
+import { appendFileSync, closeSync, mkdirSync, openSync, readFileSync, writeFileSync } from 'node:fs';
+import { createServer } from 'node:net';
+import { setTimeout as delay } from 'node:timers/promises';
+import { bootstrapUsers } from './bootstrap-users.mjs';
+
+const start = '0330d7741bbc0193af193be70d9dac47bfdb134f';
+assert.equal(execFileSync('git', ['rev-parse', 'HEAD'], { encoding: 'utf8' }).trim(), start);
+assert.equal(execFileSync('git', ['diff', '--name-only', start, '--', 'app', 'backend/src/main',
+  'package.json', 'package-lock.json'], { encoding: 'utf8' }).trim(), '');
+const directory = 'output/phase-1/e09';
+mkdirSync(directory, { recursive: true });
+const started = Date.now();
+const evidence = { start, fixtureSha256: createHash('sha256')
+  .update(readFileSync('evidence/phase-1/E09/fixtures.md')).digest('hex'), completed: [] };
+const children = [];
+const credentials = { E04_ALICE_PASSWORD: randomBytes(32).toString('base64url'),
+  E04_BOB_PASSWORD: randomBytes(32).toString('base64url') };
+let cookie;
+let pending;
+let released = false;
+let exitCode = 0;
+
+function database(action) {
+  const result = spawnSync(process.execPath, ['scripts/database.mjs', action, 'e04_restart'], { stdio: 'ignore' });
+  assert.equal(result.status, 0, `Disposable database ${action} must succeed`);
+}
+async function free(port) {
+  const probe = createServer();
+  await new Promise((resolve, reject) => {
+    probe.once('error', () => reject(new Error(`Refusing occupied port ${port}`)));
+    probe.listen({ host: '127.0.0.1', port, exclusive: true }, () => probe.close(resolve));
+  });
+}
+function child(command, args, role) {
+  const logPath = `${directory}/baseline-${role}.log`;
+  const log = openSync(logPath, 'w');
+  const { E04_ALICE_PASSWORD, E04_BOB_PASSWORD, ...runtime } = process.env;
+  const processChild = spawn(command, args, { env: { ...runtime, DB_SCHEMA: 'e04_restart' },
+    stdio: ['ignore', log, log] });
+  closeSync(log);
+  children.push(processChild);
+  return { processChild, logPath };
+}
+async function until(check) {
+  const deadline = Date.now() + 30_000;
+  while (Date.now() < deadline) {
+    if (await check()) return;
+    await delay(25);
+  }
+  throw new Error('Owned baseline checkpoint did not complete');
+}
+async function ready(owned, marker, path, expected) {
+  await until(async () => {
+    assert.ok(owned.processChild.exitCode === null && owned.processChild.signalCode === null);
+    if (!readFileSync(owned.logPath, 'utf8').includes(marker)) return false;
+    try { return (await fetch(path)).status === expected; } catch { return false; }
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
+  const fixture = child(process.execPath, ['scripts/fixture.mjs'], 'fixture');
+  await ready(fixture, 'Fixture http://127.0.0.1:4321', 'http://127.0.0.1:4321/ok', 200);
+  const api = child('java', ['-jar', 'backend/target/monitor-api-0.0.1.jar'], 'api');
+  await ready(api, 'Started MonitorApplication', 'http://127.0.0.1:4322/api/session', 401);
+  assert.equal((await post('/api/session/login', { username: 'alice-e04', password: credentials.E04_ALICE_PASSWORD })).status, 200);
+  const a = await post('/api/monitors', { name: 'A', url: 'http://127.0.0.1:4321/hold', interval: 60, enabled: true });
+  const b = await post('/api/monitors', { name: 'B', url: 'http://127.0.0.1:4321/ok', interval: 60, enabled: false });
+  assert.equal(a.status, 201);
+  assert.equal(b.status, 201);
+  let responded = false;
+  pending = post(`/api/monitors/${a.data.monitor.id}/checks`).then(value => {
+    responded = true; return { value };
+  }, () => ({ failure: true }));
+  await until(async () => (await (await fetch('http://127.0.0.1:4321/__e09/status')).json()).held === 1);
+  const count = Number(execFileSync('docker', ['compose', '--project-name', 'wse-industry', '--file', 'compose.yaml',
+    'exec', '--no-TTY', 'postgres', 'psql', '--username', 'wse_industry', '--dbname', 'monitor', '--tuples-only',
+    '--no-align', '--command', 'SELECT count(*) FROM e04_restart.check_runs;'], { encoding: 'utf8' }).trim());
+  evidence.whileHeld = { apiResponded: responded, persistedCheckRuns: count, outboundRequests: 1, fixtureReleased: false };
+  assert.equal((await fetch('http://127.0.0.1:4321/__e09/release', { method: 'POST' })).status, 200);
+  released = true;
+  const result = await pending;
+  assert.ok(!result.failure);
+  evidence.afterRelease = { status: result.value.status, state: result.value.data.state,
+    httpStatus: result.value.data.httpStatus, hasDurableId: typeof result.value.data.id === 'string' };
+  evidence.completed.push('one held manual request observed before release and its actual response recorded after HTTP200 release');
+  assert.equal(result.value.data.state, 'SUCCEEDED');
+  assert.equal(result.value.data.httpStatus, 200);
+  evidence.result = 'COUNTEREXAMPLE: synchronous terminal response instead of persisted202 QUEUED';
+  assert.equal(result.value.status, 202, 'E09 requires persisted202 QUEUED before outbound execution');
+} catch (error) {
+  exitCode = 1;
+  evidence.failure = error.message;
+  console.error(error.message);
+} finally {
+  if (!released) {
+    try { await fetch('http://127.0.0.1:4321/__e09/release', { method: 'POST', signal: AbortSignal.timeout(1000) }); } catch {}
+  }
+  if (pending) await pending;
+  for (const owned of children.toReversed()) {
+    if (owned.exitCode !== null || owned.signalCode !== null) continue;
+    const exited = once(owned, 'exit');
+    owned.kill('SIGTERM');
+    const force = setTimeout(() => owned.kill('SIGKILL'), 5000);
+    await exited;
+    clearTimeout(force);
+  }
+  database('drop');
+  evidence.elapsedSeconds = (Date.now() - started) / 1000;
+  writeFileSync(`${directory}/baseline.json`, JSON.stringify(evidence, null, 2) + '\n');
+  appendFileSync(`${directory}/invocations.jsonl`, JSON.stringify({ command: 'node scripts/e09-baseline.mjs',
+    startedAt: new Date(started).toISOString(), elapsedSeconds: evidence.elapsedSeconds, exitCode }) + '\n');
+  process.exitCode = exitCode;
+}
diff --git a/scripts/fixture.mjs b/scripts/fixture.mjs
index d485b24..a29d828 100644
--- a/scripts/fixture.mjs
+++ b/scripts/fixture.mjs
@@ -1,14 +1,39 @@
 import { createServer } from 'node:http';
 
 const port = Number(process.env.FIXTURE_PORT ?? 4321);
+const held = new Set();
+let holdRequests = 0;
+let released = false;
 const server = createServer((request, response) => {
   response.setHeader('Content-Type', 'text/plain');
   switch (request.url) {
     case '/ok': response.writeHead(200).end('ok\n'); break;
     case '/fail': response.writeHead(503).end('unavailable\n'); break;
     case '/redirect': response.writeHead(302, { Location: '/ok' }).end(); break;
+    case '/hold':
+      holdRequests++;
+      if (released) response.writeHead(200).end('ok\n');
+      else {
+        held.add(response);
+        response.once('close', () => held.delete(response));
+      }
+      break;
+    case '/__e09/status':
+      response.setHeader('Content-Type', 'application/json');
+      response.end(JSON.stringify({ holdRequests, held: held.size, released }));
+      break;
+    case '/__e09/release':
+      if (request.method !== 'POST') { response.writeHead(405).end(); break; }
+      released = true;
+      for (const pending of held) pending.writeHead(200).end('ok\n');
+      held.clear();
+      response.writeHead(200).end('released\n');
+      break;
     default: response.writeHead(404).end('not found\n');
   }
 });
 server.listen(port, '127.0.0.1', () => console.log(`Fixture http://127.0.0.1:${port}`));
-for (const signal of ['SIGINT', 'SIGTERM']) process.on(signal, () => server.close());
+for (const signal of ['SIGINT', 'SIGTERM']) process.on(signal, () => {
+  for (const pending of held) pending.destroy();
+  server.close();
+});


