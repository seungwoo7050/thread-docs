# E12 URL 정규화, SSRF와 외부 자원 제한

## `test(e12): freeze safe outbound baseline and fixtures`

diff --git a/evidence/phase-1/E12/baseline.json b/evidence/phase-1/E12/baseline.json
new file mode 100644
index 0000000..421a366
--- /dev/null
+++ b/evidence/phase-1/E12/baseline.json
@@ -0,0 +1,39 @@
+{
+  "start": "1b168ca405b0b69fd1409a68bf8d1e3f65ea23bd",
+  "fixtureSha256": "5889fee87a5ec4506c701e6d509a5ce43af542a680502b7fd48bde44fa993ba1",
+  "completed": [
+    "one authenticated public-shaped Monitor create completed; authority and local fixture counts read"
+  ],
+  "workersStarted": 0,
+  "observation": {
+    "method": "POST",
+    "path": "/api/monitors",
+    "input": {
+      "name": "Public destination fixture",
+      "url": "http://public.e12.test/ok",
+      "interval": 60,
+      "enabled": true
+    },
+    "status": 400,
+    "errorCode": "INVALID_INPUT",
+    "counts": {
+      "monitors": 0,
+      "checks": 0
+    },
+    "fixtureCalls": 0,
+    "apiPid": 82046
+  },
+  "result": "COUNTEREXAMPLE: fixture-only allowlist rejects the general HTTP destination",
+  "failure": "E12 requires acceptance of the valid public-shaped Monitor without performing outbound I/O\n\n400 !== 201\n",
+  "apiExit": {
+    "code": 143,
+    "signal": null,
+    "awaited": true,
+    "purpose": "owned baseline cleanup"
+  },
+  "cleanup": {
+    "fixtureClosed": true,
+    "disposableSchemaDropped": true
+  },
+  "elapsedSeconds": 6.834
+}
diff --git a/evidence/phase-1/E12/fixtures.md b/evidence/phase-1/E12/fixtures.md
new file mode 100644
index 0000000..4b8f09d
--- /dev/null
+++ b/evidence/phase-1/E12/fixtures.md
@@ -0,0 +1,111 @@
+# E12 fixed outbound contract
+
+Frozen before product edits and the one baseline. Profile phase-1;
+START 1b168ca405b0b69fd1409a68bf8d1e3f65ea23bd;
+Spec-Revision 2ada57a71cd34fa2fae9809415c362a8bbfcdf02.
+The main E12 common packet and pre-dispatch result-semantics supplement govern.
+
+## One unchanged baseline
+
+Own e04_restart, fixture4321 and API4322. Bootstrap the existing two runtime-only
+users, authenticate Alice through the existing session/Origin4323/CSRF boundary,
+and start no worker. Submit exactlyone Monitor create: name `Public destination
+fixture`, URL `http://public.e12.test/ok`, interval60, enabledtrue. Record its
+actual response/error category, persisted Monitor/CheckRun counts and local
+fixture request count. Stop at the first decisive fixture-only rejection.
+Do not resolve or connect to that public-shaped hostname; no external or metadata
+destination is contacted. Reuse the root-verified unchanged packaged E11 artifact.
+
+## Canonical URL and deterministic destination cases
+
+Canonicalization lowercases scheme/host, removes default port, normalizes dot
+segments and supplies `/` for an empty path, preserving query meaning. The fixed
+canonical example is `HTTP://PUBLIC.E12.TEST:80/a/../ok` →
+`http://public.e12.test/ok`. Credentials, fragments, non-HTTP(S), ambiguous numeric
+hosts and scoped IPv6 input are refused. No DNS occurs during Monitor creation.
+
+The positive resolver fixtures are public IPv4 `93.184.216.34` for
+`http://public.e12.test/ok`, and public IPv6 `2606:4700:4700::1111` for
+`https://public.e12.test/ok`. Connector stubs return controlled HTTP headers;
+they never open a socket to these addresses. Each answer set is checked in full.
+
+The fixed refusal matrix covers credential-bearing input (userinfo only, no
+secret), file://, IPv4/IPv6 loopback127.0.0.1/::1, private10.0.0.1/fc00::1,
+link-local169.254.169.254/fe80::1, and IPv4-mapped loopback::ffff:127.0.0.1.
+These are resolver/validator fixtures only. public-shaped `private.e12.test`
+resolves to10.0.0.1; `mixed.e12.test` returns93.184.216.34 and10.0.0.1.
+Every unsafe case requires zero unsafe connector calls, including mixed answers.
+
+The safe initial `http://public.e12.test/private` stub returns302 with Location
+`http://10.0.0.1/ok`: only the initial safe connector call is allowed, then
+ABORTED/null. The rebinding resolver returns93.184.216.34 on its first call and
+10.0.0.1 on a hypothetical second call. Require one resolution for that hop and
+connection to the first validated InetAddress, with no hostname resolution in
+the connector. Public IPv4/IPv6 and refused special-use address decisions are
+byte-based, not hostname substring or string-prefix checks.
+
+## Transport, deadlines and observation bounds
+
+Use JDK Socket/SSLSocket, without proxy or automatic redirects. Resolve once per
+hop, validate every actual A/AAAA result, then connect using InetSocketAddress
+with a validated InetAddress. HTTPS wraps that already-connected socket with
+the original canonical hostname, sets SNI for a DNS hostname and HTTPS endpoint
+identification, and retains the JDK trust configuration. The public-IPv6 HTTPS
+case uses a connector stub; TLS configuration/source proof is labelled separately
+from real local HTTP I/O. Do not claim a live public TLS connection was tested.
+
+Connect timeout500ms; read timeout500ms; total request deadline1500ms across DNS,
+connect, TLS, informational headers and all redirect hops. I/O executor capacity
+is one task/thread per runner with zero queued tasks. Timeout cancels the task,
+closes its active socket, and forbids a late resolver result from opening a new
+connection. A resolver that ignores interruption cannot accumulate new threads
+or queued work. Executor/authoritative-service rejection stays unknown for E11
+recovery, rather than becoming an invented endpoint result.
+
+At most3 redirects are followed. The parser stores at most65536 header bytes
+across informational/final blocks of a hop, respects read/total deadlines, and
+waits past normal informational responses for the final response. The final
+validated response ends observation immediately: body consumption0 is within
+the fixed65536-byte body maximum. No body-content or body-size health assertion
+is added. Final2xx is SUCCEEDED; other final status is FAILED/HTTP_STATUS.
+
+Real local fixtures are isolated on4325 with an explicit trusted-test exception:
+an immediate200, a200 with65537-byte body, a200 whose headers are withheld2000ms,
+headers trickled every400ms until2000ms (read timeout alone cannot bound this),
+a103 informational block followed by200, and the fixed4-redirect chain. The
+chain permits only the initial request plus3 follows; its next redirect is
+refused ABORTED/null without a fifth connection. Existing /redirect→/ok and
+/redirect-outside→4324 retain their exact requests/destinations; E12 changes
+their results to final200 and policy-ABORTED/null respectively, preserving the
+outside trap's zero-request assertion.
+
+Measure run completion separately from JVM/fixture setup: read-timeout completion
+must be below1000ms, total-deadline completion below1750ms (1500ms product bound
+plus250ms observer/scheduler margin), and other fixed cases below1750ms. Record
+actual elapsed time, connector/redirect counts, body bytes consumed and closed
+sockets. A controlled blocked resolver uses the same1500ms deadline and is
+released explicitly in finally; no retained task may connect after cancellation.
+Fixture startup/cleanup watchdog is5000ms, not an extension of request deadlines.
+
+## Semantics, regression adapters and budget
+
+Policy refusal before a final endpoint response is ABORTED/null with a fixed
+permanent reason documented/logged without URL or userinfo. Transport timeout or
+connection failure before final headers is FAILED/null with the existing reason.
+Final headers stay authoritative even when remaining I/O is closed locally.
+Service/authoritative-store uncertainty propagates without a fabricated outcome;
+E11 recovery and terminal fencing remain unchanged. Retryability is a small
+documented reason mapping only: no automatic retry, new setting or terminal reset.
+
+Production defaults reject unsafe destinations. Existing controlled Java/browser/
+restart fixtures gain only an explicit test-owned allow-fixture switch, scoped
+to the exact configured loopback HTTP origin; no request field can enable it.
+The previous fixture-only validator test is adapted to the new public URL rule
+while keeping all old unsafe cases. Redirect expectations change only as required
+by the frozen final-response semantics. Old evidence/migrations/runtime pins stay
+immutable. No E11 crash checkpoint, E20 work or future feature is run or added.
+
+Baseline1; cheapest affected author gate then one reserved independent root gate.
+Load0, retries0, parameter sweeps0. Every invocation and failure is preserved;
+an unexpected validation/build failure stops this attempt before any fix/rerun.
+No credentials, session/CSRF values, private bodies or browser captures in evidence.
diff --git a/evidence/phase-1/E12/invocations.jsonl b/evidence/phase-1/E12/invocations.jsonl
new file mode 100644
index 0000000..fd6c3fe
--- /dev/null
+++ b/evidence/phase-1/E12/invocations.jsonl
@@ -0,0 +1 @@
+{"command":"node scripts/e12-baseline.mjs","startedAt":"2026-08-28T06:55:52.514Z","elapsedSeconds":6.834,"exitCode":1}
diff --git a/scripts/e12-baseline.mjs b/scripts/e12-baseline.mjs
new file mode 100644
index 0000000..100ea21
--- /dev/null
+++ b/scripts/e12-baseline.mjs
@@ -0,0 +1,108 @@
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
+const start = '1b168ca405b0b69fd1409a68bf8d1e3f65ea23bd';
+assert.equal(execFileSync('git', ['rev-parse', 'HEAD'], { encoding: 'utf8' }).trim(), start);
+assert.equal(execFileSync('git', ['diff', '--name-only', start, '--', 'app', 'backend/src/main',
+  'package.json', 'package-lock.json'], { encoding: 'utf8' }).trim(), '');
+const directory = 'output/phase-1/e12';
+mkdirSync(directory, { recursive: true });
+const started = Date.now();
+const evidence = { start, fixtureSha256: createHash('sha256')
+  .update(readFileSync('evidence/phase-1/E12/fixtures.md')).digest('hex'), completed: [], workersStarted: 0 };
+const credentials = { E04_ALICE_PASSWORD: randomBytes(32).toString('base64url'),
+  E04_BOB_PASSWORD: randomBytes(32).toString('base64url') };
+let api;
+let cookie;
+let fixtureCalls = 0;
+let exitCode = 0;
+const fixture = createServer((request, response) => { fixtureCalls++; response.writeHead(200).end('ok\n'); });
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
+  return { Cookie: cookie, [data.headerName]: data.token, Origin: 'http://127.0.0.1:4323' };
+}
+async function post(path, body) {
+  const response = await fetch(`http://127.0.0.1:4322${path}`, {
+    method: 'POST', headers: { ...(await csrf()), 'Content-Type': 'application/json' },
+    body: JSON.stringify(body), signal: AbortSignal.timeout(10_000),
+  });
+  const issued = response.headers.get('set-cookie');
+  if (issued) cookie = issued.split(';', 1)[0];
+  return { status: response.status, body: await response.json() };
+}
+try {
+  await free(4321);
+  await free(4322);
+  database('reset');
+  bootstrapUsers('e04_restart', credentials);
+  await new Promise((resolve, reject) => { fixture.once('error', reject); fixture.listen(4321, '127.0.0.1', resolve); });
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
+    assert.ok(api.exitCode === null && api.signalCode === null, 'Owned API remains alive');
+    if (readFileSync(logPath, 'utf8').includes('Started MonitorApplication')) { ready = true; break; }
+    await delay(25);
+  }
+  assert.ok(ready, 'Owned API startup must complete');
+  assert.equal((await post('/api/session/login', { username: 'alice-e04', password: credentials.E04_ALICE_PASSWORD })).status, 200);
+  const input = { name: 'Public destination fixture', url: 'http://public.e12.test/ok', interval: 60, enabled: true };
+  const response = await post('/api/monitors', input);
+  const counts = JSON.parse(execFileSync('docker', ['compose', '--project-name', 'wse-industry', '--file', 'compose.yaml',
+    'exec', '--no-TTY', 'postgres', 'psql', '--username', 'wse_industry', '--dbname', 'monitor', '--tuples-only',
+    '--no-align', '--command', "SELECT json_build_object('monitors',(SELECT count(*) FROM e04_restart.monitors),"
+      + "'checks',(SELECT count(*) FROM e04_restart.check_runs));"], { encoding: 'utf8' }).trim());
+  evidence.observation = { method: 'POST', path: '/api/monitors', input, status: response.status,
+    errorCode: response.body.error?.code ?? null, counts, fixtureCalls, apiPid: api.pid };
+  evidence.completed.push('one authenticated public-shaped Monitor create completed; authority and local fixture counts read');
+  evidence.result = response.status === 400 ? 'COUNTEREXAMPLE: fixture-only allowlist rejects the general HTTP destination'
+    : 'NO_FIXTURE_ONLY_REJECTION_OBSERVED';
+  assert.equal(response.status, 201, 'E12 requires acceptance of the valid public-shaped Monitor without performing outbound I/O');
+} catch (error) {
+  exitCode = 1;
+  evidence.failure = error.message;
+  console.error(error.message);
+} finally {
+  if (api && api.exitCode === null && api.signalCode === null) {
+    const exited = once(api, 'exit');
+    api.kill('SIGTERM');
+    const [code, signal] = await exited;
+    evidence.apiExit = { code, signal, awaited: true, purpose: 'owned baseline cleanup' };
+  }
+  if (fixture.listening) await new Promise(resolve => fixture.close(resolve));
+  database('drop');
+  evidence.cleanup = { fixtureClosed: !fixture.listening, disposableSchemaDropped: true };
+  evidence.elapsedSeconds = (Date.now() - started) / 1000;
+  writeFileSync(`${directory}/baseline.json`, JSON.stringify(evidence, null, 2) + '\n');
+  appendFileSync(`${directory}/invocations.jsonl`, JSON.stringify({ command: 'node scripts/e12-baseline.mjs',
+    startedAt: new Date(started).toISOString(), elapsedSeconds: evidence.elapsedSeconds, exitCode }) + '\n');
+  process.exitCode = exitCode;
+}


