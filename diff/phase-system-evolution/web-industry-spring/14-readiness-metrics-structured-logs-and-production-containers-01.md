# E24 Readiness, Metrics, Structured Logs와 Production Container

## `test(e24): freeze the operations and container contract`

diff --git a/evidence/phase-1/E24/baseline-output.json b/evidence/phase-1/E24/baseline-output.json
new file mode 100644
index 0000000..95cef73
--- /dev/null
+++ b/evidence/phase-1/E24/baseline-output.json
@@ -0,0 +1,6 @@
+{
+  "command": "fnm exec --using 24.19.0 node scripts/e24-baseline.mjs",
+  "exitCode": 0,
+  "stdoutUtf8": "E24 unchanged baseline: authenticated liveness404; fixed seed2/2/4; no outage or worker/container run.\n",
+  "source": "native command completion; runtime API logs were inspected only in RAM"
+}
diff --git a/evidence/phase-1/E24/baseline.json b/evidence/phase-1/E24/baseline.json
new file mode 100644
index 0000000..0ed089c
--- /dev/null
+++ b/evidence/phase-1/E24/baseline.json
@@ -0,0 +1,43 @@
+{
+  "start": "563b325ef871fe6d1fbfef7cf39a6581f2d0a94d",
+  "fixtureSha256": "47f86919e5f251c3f8ee394727235b906fc03a5eac67e36cc3e273486a21a8fd",
+  "completed": [
+    "fixed seed and authentication; one liveness observation; no further guarantee probed"
+  ],
+  "postgresStopSequences": 0,
+  "workersStarted": 0,
+  "applicationContainersStarted": 0,
+  "seedCounts": {
+    "users": 2,
+    "monitors": 2,
+    "checks": 4
+  },
+  "observation": {
+    "method": "GET",
+    "path": "/ops/health/liveness",
+    "authenticated": true,
+    "status": 404,
+    "requiredStatus": 200,
+    "criterionMet": false,
+    "apiPid": 97310
+  },
+  "result": "COUNTEREXAMPLE: liveness interface is absent at unchanged START",
+  "apiExit": {
+    "code": 143,
+    "signal": null,
+    "elapsedMs": 42,
+    "awaited": true,
+    "purpose": "baseline cleanup"
+  },
+  "logInspection": {
+    "lines": 52,
+    "runtimeSentinelMatches": 0,
+    "rawLogsPersisted": false
+  },
+  "cleanup": {
+    "ownedSchemaDropped": true,
+    "cleanupFailures": [],
+    "apiExitAwaited": true
+  },
+  "elapsedSeconds": 7.477
+}
diff --git a/evidence/phase-1/E24/baseline.started.json b/evidence/phase-1/E24/baseline.started.json
new file mode 100644
index 0000000..a7682fe
--- /dev/null
+++ b/evidence/phase-1/E24/baseline.started.json
@@ -0,0 +1 @@
+{"command":"node scripts/e24-baseline.mjs","startedAt":"2026-08-28T08:51:05.183Z","start":"563b325ef871fe6d1fbfef7cf39a6581f2d0a94d"}
diff --git a/evidence/phase-1/E24/fixture.json b/evidence/phase-1/E24/fixture.json
new file mode 100644
index 0000000..4f5515f
--- /dev/null
+++ b/evidence/phase-1/E24/fixture.json
@@ -0,0 +1,113 @@
+{
+  "thread": "E24",
+  "profile": "phase-1",
+  "attempt": 1,
+  "start": "563b325ef871fe6d1fbfef7cf39a6581f2d0a94d",
+  "specRevision": "2ada57a71cd34fa2fae9809415c362a8bbfcdf02",
+  "roles": {
+    "api": { "port": 4322, "process": "java -jar /app/app.jar", "uid": 10001 },
+    "worker": { "port": 4324, "process": "java -jar /app/app.jar --spring.profiles.active=worker --spring.main.web-application-type=none", "uid": 10001 },
+    "frontend": { "port": 4323, "process": "node server.js", "uid": 1000 },
+    "fixture": { "port": 4321, "testOnly": true, "networkMode": "service:worker" },
+    "postgres": { "project": "wse-industry", "service": "postgres", "port": 15432, "database": "monitor" }
+  },
+  "project": "wse-industry-e24",
+  "schemas": { "baseline": "e24_baseline", "container": "e24_container", "metricIntegration": "e24_metrics" },
+  "endpoints": {
+    "liveness": "/ops/health/liveness",
+    "readiness": "/ops/health/readiness",
+    "metricNames": "/ops/metrics",
+    "metricValue": "/ops/metrics/{name}",
+    "browser": "http://127.0.0.1:4323/monitors",
+    "serverOriginAtBuildAndRuntime": "http://api:4322",
+    "trustedBrowserOrigin": "http://127.0.0.1:4323",
+    "fixtureOrigin": "http://127.0.0.1:4321",
+    "fixtureControl": ["GET /__e24/status", "POST /__e24/release"]
+  },
+  "images": {
+    "java": "eclipse-temurin:21.0.7_6-jre-jammy@sha256:c4e6542e774de504da9b4729ff8d761287c965c1d788528ca78da30024efdb23",
+    "node": "node:24.19.0-bookworm-slim@sha256:a9f5f7c91a432850b2a8a7797adf5eadb6c733ceed61167806cee7ea7fbc29df",
+    "postgres": "postgres:17.11-bookworm@sha256:051f7b7b3abdd564d5d1bd1e8c4b9c1b6e77087d1dd22020ede611c096a272e0",
+    "backendArtifact": "backend/target/monitor-api-0.0.1.jar built by Java21.0.7+6/Maven3.9.11",
+    "frontendArtifact": "Next16.3.3 standalone plus .next/static, built with Node24.19.0/npm11.17.0 and unchanged lockfile pins"
+  },
+  "seed": {
+    "alice": "24000000-0000-4000-a000-000000000001",
+    "bob": "24000000-0000-4000-a000-000000000002",
+    "monitorA": "24000000-0000-4000-8000-000000000001",
+    "monitorB": "24000000-0000-4000-8000-000000000002",
+    "names": ["E24 Alice monitor", "E24 Bob monitor"],
+    "paths": ["/hold?marker=<runtime URL sentinel>", "/ok?marker=<independent runtime URL sentinel>"],
+    "interval": 60,
+    "enabled": false,
+    "monitorTime": "2026-08-28T00:00:00.000Z",
+    "checkIds": ["24000000-0000-4000-b000-000000000001", "24000000-0000-4000-b000-000000000002", "24000000-0000-4000-b000-000000000003", "24000000-0000-4000-b000-000000000004"],
+    "checkTimes": "2026-08-28T00:00:01Z through 00:00:04Z; queued=started=finished; latency1",
+    "checks": "Alice first3: MANUAL/FAILED/503/HTTP_STATUS; Bob fourth: MANUAL/SUCCEEDED/200/null",
+    "credentials": "Existing bootstrapUsers; independent random runtime passwords; fixed UUIDs assigned before Monitors exist",
+    "missingMonitorIds": "24000000-0000-4000-9000-000000000001 through 000000000010"
+  },
+  "metrics": {
+    "names": ["http.server.requests", "check.queue.age", "check.worker.active", "check.claims", "check.recoveries"],
+    "httpMeasurements": "COUNT/TOTAL_TIME/MAX; error counts are request COUNT filtered by HTTP status>=400",
+    "httpTags": ["uri", "method", "status", "process_role"],
+    "workerAndQueueTags": ["process_role"],
+    "routeBoundary": "fixed route templates or UNMATCHED; no raw path, resource ID, query, name or URL",
+    "methodBoundary": "GET/HEAD/POST/PUT/PATCH/DELETE/OPTIONS/TRACE or OTHER",
+    "workerEmpty": "queue age0 and active0 are actual empty/idle observations; claim/recovery counters change only after committed operations"
+  },
+  "limits": {
+    "startupWatchdogMs": 90000,
+    "httpObservationTimeoutMs": 3000,
+    "authorityConnectionTimeoutMs": 1000,
+    "authorityValidationTimeoutMs": 500,
+    "postgresRestoreWatchdogMs": 20000,
+    "gracefulExitBoundMs": 5000,
+    "exitObservationWatchdogMs": 6000,
+    "springShutdownPhaseMs": 3000,
+    "fixtureHoldSafetyReleaseMs": 350,
+    "unchangedOutbound": { "connectMs": 500, "readMs": 500, "totalMs": 1500, "redirects": 3, "headerBytes": 65536, "bodyReadBytes": 0 }
+  },
+  "nativeSigterm": {
+    "api": 143,
+    "worker": 143,
+    "frontend": 143,
+    "source": "E11 real Java shutdown and pinned Next start-server.js cleanup/flush then process.exit(143); NEXT_MANUAL_SIG_HANDLE remains unset",
+    "nextSourceSha256": "cf15b6389af3269979848e6cc23ecd7373051bb459e9ec370bfe8b05ee908285",
+    "observation": "signal actual container process once; await docker wait; persist actual exit and duration before asserting; no forced kill or second signal as observation"
+  },
+  "postgresOutage": {
+    "stop": "docker compose -p wse-industry -f compose.yaml stop --timeout 5 postgres",
+    "restore": "docker compose -p wse-industry -f compose.yaml start postgres, then bounded health/pg_isready observation",
+    "installedCompose": "2.34.0-desktop.1; start has no --wait option",
+    "boundary": "all five CheckRuns terminal, worker active0, exactly one outbound completed",
+    "attempt": "one authenticated POST /api/monitors/{monitorA}/checks with valid pre-issued CSRF, exact Origin and fresh runtime Idempotency-Key",
+    "expect": "liveness200/readiness503 in both roles, unsafe request503/no accepted ID, same live processes, no new claim/outbound/result; after restore same PG container/image/volume/public authority and exact existing rows"
+  },
+  "sequence": [
+    "baseline: unchanged START artifact, owned schema and same fixed seed; authenticate Alice; one GET liveness; record first missing guarantee and stop without outage/worker/container gate",
+    "final: refuse occupied ports/project/schemas; seed once; start actual non-root production API/worker/frontend and explicit loopback test fixture; both roles live/ready",
+    "authenticated Chromium list/detail/server HTML and static asset; Alice sees only A and three failures, Bob only B and one success; no trace/HAR/screenshot/credential artifact",
+    "one real browser manual intent202; observe actual held outbound and worker active1, release before safety bound, same ID SUCCEEDED200; queue age/claim/recovery observations from actual meters",
+    "close browser activity at idle; stop owned PG once; verify liveness/readiness and one rejected unsafe intent; restore same PG and verify same authority/no extra outbound or claim",
+    "ten authenticated missing UUID GETs; compare bounded metric tags and error-count delta10; source-check arbitrary methods/unmatched path normalization",
+    "scan runtime-only URL/token/key/body sentinels and request/process correlation in RAM; save safe counts/booleans only",
+    "actual UID/PID1/artifact version inspection; one SIGTERM and awaited native143<=5s per application role; remove only owned containers/schema, retain PG public/volume"
+  ],
+  "affectedMetricTest": "One isolated e24_metrics case: a queued row proves positive age, a held production CheckWorker call proves active1/committed claim count, and an explicitly expired row proves committed recovery count. No process crash or dependency stop.",
+  "budget": {
+    "baseline": 1,
+    "authorContainerScenario": 1,
+    "rootContainerScenario": 1,
+    "postgresStopRestorePerFullGate": 1,
+    "manualAcceptedIntentPerFullGate": 1,
+    "manualRejectedOutageIntentPerFullGate": 1,
+    "missingUuidRequestsPerFullGate": 10,
+    "freshRepairsMaximum": 2,
+    "loadRuns": 0,
+    "parameterSweeps": 0,
+    "automaticRetries": 0,
+    "e11KillOrE20DatasetRuns": 0,
+    "defaultContainerGate": false
+  }
+}
diff --git a/evidence/phase-1/E24/fixtures.md b/evidence/phase-1/E24/fixtures.md
new file mode 100644
index 0000000..bfe84c5
--- /dev/null
+++ b/evidence/phase-1/E24/fixtures.md
@@ -0,0 +1,47 @@
+# E24 frozen phase-1 contract
+
+The exact dataset, roles, endpoints, images, ports, limits, sequence and budgets
+are in `fixture.json`. They are frozen before the single unchanged-START
+baseline. Only phase-1 E24 is selected; no omitted-role metric or future policy
+is introduced.
+
+The first baseline uses the already built START Java artifact and existing
+bootstrap helper. It seeds the fixed two users/two Monitors/four terminal
+results, authenticates Alice and observes one `/ops/health/liveness` request.
+It stops at the first decisive missing guarantee; no PG outage or container
+scenario is spent on that known-old product. No password, session, CSRF value,
+URL marker, idempotency key or raw response body is retained.
+
+Container transport preserves E12: the explicit test-only fixture shares the
+worker network namespace. The target remains loopback `127.0.0.1:4321`, and only
+the smoke override enables `ALLOW_TEST_FIXTURE`. Production defaults and the
+validated-IP/TLS rules are unchanged. The Node fixture retains the existing
+hold/release semantics while accepting a runtime-only query marker and serving
+a runtime-only body sentinel. The 350ms release watchdog protects the existing
+500ms read bound; a successful active-state observation releases it explicitly.
+The watchdog is not an acceptance substitute if that observation is missed.
+
+Both Java and this pinned Next server natively exit143 on SIGTERM. Next closes
+the server and awaits cleanup/trace flushing first. Each role receives one real
+signal, then an awaited exit observation. Actual code/duration is recorded before
+assertions; a forced kill does not satisfy graceful shutdown. Startup watchdogs
+are separate from the 5000ms exit criterion and existing Spring 3000ms phase.
+
+Compose2.34.0 start does not support `--wait`. Stop and restore only the existing
+`wse-industry` PostgreSQL service, preserving its container, image, volume and
+public authority. Restore uses `start postgres` plus bounded health/pg_isready
+observation. The new application project and schemas are independently owned;
+refuse collisions and never run `down -v`.
+
+Metrics use only fixed route/method/status/process-role categories. Ten distinct
+missing UUIDs exercise the same route; source inspection covers arbitrary
+unmatched paths and methods without adding a sweep. Runtime sentinel values,
+container logs and secret-bearing HTTP captures stay in RAM. Persist only safe
+counts/booleans and correlation fields generated by the service, never an
+untrusted request ID. Browser trace/HAR/screenshots/video are disabled.
+
+Author budget is one decisive baseline, minimal affected tests/builds and one
+full container scenario. Root has one independent final container gate. Every
+invocation/failure is retained; an unexpected formal failure stops work before
+repair or rerun. No E11 kill, E20 dataset, load run or parameter sweep is part
+of E24. Hosted CI remains unexecuted evidence until it actually runs.
diff --git a/evidence/phase-1/E24/invocations.jsonl b/evidence/phase-1/E24/invocations.jsonl
new file mode 100644
index 0000000..9b9ee94
--- /dev/null
+++ b/evidence/phase-1/E24/invocations.jsonl
@@ -0,0 +1 @@
+{"command":"node scripts/e24-baseline.mjs","startedAt":"2026-08-28T08:51:05.183Z","elapsedSeconds":7.477,"exitCode":0}
diff --git a/scripts/database.mjs b/scripts/database.mjs
index 8a64c9a..388094e 100644
--- a/scripts/database.mjs
+++ b/scripts/database.mjs
@@ -4,7 +4,7 @@ import { spawnSync } from 'node:child_process';
 const compose = ['compose', '--project-name', 'wse-industry', '--file', 'compose.yaml'];
 const schemas = new Set(['e04_restart', 'e04_functional', 'e04_browser', 'e04_migrations',
   'e04_mapping', 'e04_missing_column', 'e04_extra_required', 'e04_session', 'e04_users',
-  'e04_missing_user_column', 'e04_extra_user_required']);
+  'e04_missing_user_column', 'e04_extra_user_required', 'e24_baseline', 'e24_container']);
 const [action, schema] = process.argv.slice(2);
 let args;
 if (action === 'up') args = [...compose, 'up', '--detach', '--wait', '--wait-timeout', '30', 'postgres'];
diff --git a/scripts/e24-baseline.mjs b/scripts/e24-baseline.mjs
new file mode 100644
index 0000000..2adced4
--- /dev/null
+++ b/scripts/e24-baseline.mjs
@@ -0,0 +1,125 @@
+import assert from 'node:assert/strict';
+import { execFileSync, spawn } from 'node:child_process';
+import { createHash } from 'node:crypto';
+import { once } from 'node:events';
+import { appendFileSync, mkdirSync, readFileSync, writeFileSync } from 'node:fs';
+import { createServer } from 'node:net';
+import { setTimeout as delay } from 'node:timers/promises';
+import { fixture, runtimeSecrets, seed, sql, drop } from './e24-seed.mjs';
+
+const directory = 'output/phase-1/e24';
+mkdirSync(directory, { recursive: true });
+const started = Date.now();
+writeFileSync(`${directory}/baseline.started.json`, JSON.stringify({ command: 'node scripts/e24-baseline.mjs',
+  startedAt: new Date(started).toISOString(), start: fixture.start }) + '\n', { flag: 'wx' });
+const evidence = { start: fixture.start, fixtureSha256: createHash('sha256')
+  .update(readFileSync('evidence/phase-1/E24/fixture.json')).digest('hex'), completed: [], postgresStopSequences: 0,
+  workersStarted: 0, applicationContainersStarted: 0 };
+const secrets = runtimeSecrets();
+const schema = fixture.schemas.baseline;
+let api;
+let cookie;
+let apiLogs = '';
+let schemaCreated = false;
+let exitCode = 0;
+let phase = 'owned port/schema preflight';
+async function csrf() {
+  const response = await fetch('http://127.0.0.1:4322/api/session/csrf', {
+    headers: cookie ? { Cookie: cookie } : {}, signal: AbortSignal.timeout(3000) });
+  assert.equal(response.status, 200, 'CSRF bootstrap status');
+  const issued = response.headers.get('set-cookie');
+  if (issued) cookie = issued.split(';', 1)[0];
+  const { data } = await response.json();
+  return { Cookie: cookie, [data.headerName]: data.token, Origin: fixture.endpoints.trustedBrowserOrigin };
+}
+try {
+  assert.equal(execFileSync('git', ['rev-parse', 'HEAD'], { encoding: 'utf8' }).trim(), fixture.start);
+  assert.equal(execFileSync('git', ['diff', '--name-only', fixture.start, '--', 'app', 'backend/src/main',
+    'backend/pom.xml', 'package.json', 'package-lock.json', 'next.config.mjs'], { encoding: 'utf8' }).trim(), '');
+  const probe = createServer();
+  await new Promise((resolve, reject) => {
+    probe.once('error', () => reject(new Error('Refusing occupied API port4322')));
+    probe.listen({ host: '127.0.0.1', port: 4322, exclusive: true }, () => probe.close(resolve));
+  });
+  assert.equal(sql(`SELECT count(*) FROM pg_namespace WHERE nspname='${schema}';`), '0');
+  phase = 'runtime user bootstrap and fixed seed';
+  seed(schema, secrets, () => { schemaCreated = true; });
+  evidence.seedCounts = JSON.parse(sql(`SELECT json_build_object('users',(SELECT count(*) FROM ${schema}.users),
+    'monitors',(SELECT count(*) FROM ${schema}.monitors),'checks',(SELECT count(*) FROM ${schema}.check_runs));`));
+  assert.deepEqual(evidence.seedCounts, { users: 2, monitors: 2, checks: 4 });
+  const { E04_ALICE_PASSWORD, E04_BOB_PASSWORD, ...runtime } = process.env;
+  phase = 'unchanged API startup';
+  api = spawn('java', ['-jar', 'backend/target/monitor-api-0.0.1.jar'], {
+    env: { ...runtime, DB_SCHEMA: schema }, stdio: ['ignore', 'pipe', 'pipe'],
+  });
+  api.stdout.on('data', chunk => { apiLogs += chunk; });
+  api.stderr.on('data', chunk => { apiLogs += chunk; });
+  const deadline = Date.now() + fixture.limits.startupWatchdogMs;
+  while (!apiLogs.includes('Started MonitorApplication') && Date.now() < deadline) {
+    assert.ok(api.exitCode === null && api.signalCode === null, 'Owned baseline API remains alive');
+    await delay(25);
+  }
+  assert.ok(apiLogs.includes('Started MonitorApplication'), 'Owned API startup completed');
+  phase = 'authenticated baseline session';
+  const login = await fetch('http://127.0.0.1:4322/api/session/login', {
+    method: 'POST', headers: { ...(await csrf()), 'Content-Type': 'application/json' },
+    body: JSON.stringify({ username: 'alice-e04', password: secrets.E04_ALICE_PASSWORD }),
+    signal: AbortSignal.timeout(3000),
+  });
+  assert.equal(login.status, 200, 'Authenticated baseline fixture');
+  const issued = login.headers.get('set-cookie');
+  if (issued) cookie = issued.split(';', 1)[0];
+  await login.arrayBuffer();
+  phase = 'single liveness observation';
+  const response = await fetch(`http://127.0.0.1:4322${fixture.endpoints.liveness}`, {
+    headers: { Cookie: cookie }, signal: AbortSignal.timeout(3000),
+  });
+  await response.arrayBuffer();
+  evidence.observation = { method: 'GET', path: fixture.endpoints.liveness, authenticated: true,
+    status: response.status, requiredStatus: 200, criterionMet: response.status === 200, apiPid: api.pid };
+  evidence.result = response.status === 404 ? 'COUNTEREXAMPLE: liveness interface is absent at unchanged START'
+    : 'Unexpected baseline observation';
+  evidence.completed.push('fixed seed and authentication; one liveness observation; no further guarantee probed');
+  assert.equal(response.status, 404, 'Source-predicted missing liveness baseline');
+  console.log('E24 unchanged baseline: authenticated liveness404; fixed seed2/2/4; no outage or worker/container run.');
+} catch (error) {
+  exitCode = 1;
+  evidence.failure = { phase, type: error.name, code: error.code ?? null,
+    exitStatus: typeof error.status === 'number' ? error.status : null,
+    actual: typeof error.actual === 'number' ? error.actual : null,
+    expected: typeof error.expected === 'number' ? error.expected : null,
+    privateExceptionOutputRetained: false };
+  console.error('E24 baseline setup/observation failed; safe phase/status details preserved.');
+} finally {
+  const cleanupFailures = [];
+  try {
+    if (api && api.exitCode === null && api.signalCode === null) {
+      const exiting = once(api, 'exit');
+      const signalled = Date.now();
+      api.kill('SIGTERM');
+      const observed = await Promise.race([exiting,
+        delay(fixture.limits.exitObservationWatchdogMs, null, { ref: false })]);
+      evidence.apiExit = { code: observed?.[0] ?? null, signal: observed?.[1] ?? null,
+        elapsedMs: Date.now() - signalled, awaited: observed !== null, purpose: 'baseline cleanup' };
+      if (!observed || evidence.apiExit.elapsedMs > fixture.limits.gracefulExitBoundMs) exitCode = 1;
+    }
+  } catch (error) { cleanupFailures.push({ phase: 'API exit observation', type: error.name }); exitCode = 1; }
+  evidence.logInspection = { lines: apiLogs.split('\n').length,
+    runtimeSentinelMatches: Object.values(secrets).filter(value => apiLogs.includes(value)).length,
+    rawLogsPersisted: false };
+  let ownedSchemaDropped = false;
+  try {
+    if (schemaCreated) { drop(schema); ownedSchemaDropped = true; }
+  } catch (error) {
+    cleanupFailures.push({ phase: 'owned schema drop', type: error.name,
+      exitStatus: typeof error.status === 'number' ? error.status : null });
+    exitCode = 1;
+  }
+  evidence.cleanup = { ownedSchemaDropped, cleanupFailures,
+    apiExitAwaited: !api || api.exitCode !== null || api.signalCode !== null };
+  evidence.elapsedSeconds = (Date.now() - started) / 1000;
+  writeFileSync(`${directory}/baseline.json`, JSON.stringify(evidence, null, 2) + '\n');
+  appendFileSync(`${directory}/invocations.jsonl`, JSON.stringify({ command: 'node scripts/e24-baseline.mjs',
+    startedAt: new Date(started).toISOString(), elapsedSeconds: evidence.elapsedSeconds, exitCode }) + '\n');
+  process.exitCode = exitCode;
+}
diff --git a/scripts/e24-seed.mjs b/scripts/e24-seed.mjs
new file mode 100644
index 0000000..568bb7f
--- /dev/null
+++ b/scripts/e24-seed.mjs
@@ -0,0 +1,47 @@
+import assert from 'node:assert/strict';
+import { execFileSync } from 'node:child_process';
+import { randomBytes } from 'node:crypto';
+import { readFileSync } from 'node:fs';
+import { bootstrapUsers } from './bootstrap-users.mjs';
+
+export const fixture = JSON.parse(readFileSync('evidence/phase-1/E24/fixture.json', 'utf8'));
+const compose = ['compose', '-p', 'wse-industry', '-f', 'compose.yaml'];
+export function sql(statement) {
+  return execFileSync('docker', [...compose, 'exec', '-T', 'postgres', 'psql', '-U', 'wse_industry',
+    '-d', 'monitor', '-v', 'ON_ERROR_STOP=1', '-At', '-f', '-'],
+  { input: statement, encoding: 'utf8', stdio: ['pipe', 'pipe', 'pipe'] }).trim();
+}
+export function runtimeSecrets() {
+  return { E04_ALICE_PASSWORD: randomBytes(32).toString('base64url'),
+    E04_BOB_PASSWORD: randomBytes(32).toString('base64url'),
+    urlMarkerA: randomBytes(24).toString('hex'), urlMarkerB: randomBytes(24).toString('hex'),
+    bodyMarker: randomBytes(24).toString('hex'), outageKey: randomBytes(24).toString('hex'),
+    untrustedRequestId: randomBytes(24).toString('hex') };
+}
+export function seed(schema, secrets, onSchemaCreated) {
+  assert.ok(Object.values(fixture.schemas).includes(schema), 'Only the frozen owned schema may be seeded');
+  assert.equal(sql(`SELECT count(*) FROM pg_namespace WHERE nspname='${schema}';`), '0', 'Refusing an occupied E24 schema');
+  sql(`CREATE SCHEMA ${schema};`);
+  onSchemaCreated();
+  bootstrapUsers(schema, { E04_ALICE_PASSWORD: secrets.E04_ALICE_PASSWORD, E04_BOB_PASSWORD: secrets.E04_BOB_PASSWORD });
+  const f = fixture.seed;
+  const urlA = `${fixture.endpoints.fixtureOrigin}/hold?marker=${secrets.urlMarkerA}`;
+  const urlB = `${fixture.endpoints.fixtureOrigin}/ok?marker=${secrets.urlMarkerB}`;
+  // Values are frozen constants or generated hex sentinels; SQL stays on stdin, never command arguments or evidence.
+  sql(`BEGIN;
+    UPDATE ${schema}.users SET id='${f.alice}' WHERE username='alice-e04';
+    UPDATE ${schema}.users SET id='${f.bob}' WHERE username='bob-e04';
+    INSERT INTO ${schema}.monitors (id,name,url,interval_seconds,enabled,created_at,updated_at,owner_user_id) VALUES
+      ('${f.monitorA}','${f.names[0]}','${urlA}',60,false,'${f.monitorTime}','${f.monitorTime}','${f.alice}'),
+      ('${f.monitorB}','${f.names[1]}','${urlB}',60,false,'${f.monitorTime}','${f.monitorTime}','${f.bob}');
+    INSERT INTO ${schema}.check_runs (id,monitor_id,trigger_kind,state,http_status,latency_ms,failure_reason,started_at,finished_at,queued_at)
+      VALUES ${f.checkIds.map((id, i) => `('${id}','${i < 3 ? f.monitorA : f.monitorB}','MANUAL',
+        '${i < 3 ? 'FAILED' : 'SUCCEEDED'}',${i < 3 ? 503 : 200},1,${i < 3 ? "'HTTP_STATUS'" : 'null'},
+        '2026-08-28T00:00:0${i + 1}Z','2026-08-28T00:00:0${i + 1}Z','2026-08-28T00:00:0${i + 1}Z')`).join(',')};
+    COMMIT;`);
+  return { urlA, urlB };
+}
+export function drop(schema) {
+  assert.ok(Object.values(fixture.schemas).includes(schema), 'Only the frozen owned schema may be dropped');
+  sql(`DROP SCHEMA ${schema} CASCADE;`);
+}


