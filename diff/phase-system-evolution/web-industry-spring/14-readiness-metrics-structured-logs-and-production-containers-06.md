## `test: add frozen phase-1 production operations observer`

diff --git a/scripts/e24-container.mjs b/scripts/e24-container.mjs
new file mode 100644
index 0000000..f693479
--- /dev/null
+++ b/scripts/e24-container.mjs
@@ -0,0 +1,753 @@
+import { execFileSync, spawn } from 'node:child_process';
+import { createHash } from 'node:crypto';
+import { appendFileSync, mkdirSync, readFileSync, writeFileSync } from 'node:fs';
+import { createServer } from 'node:net';
+import { setTimeout as delay } from 'node:timers/promises';
+import { drop, fixture, runtimeSecrets, seed, sql } from './e24-seed.mjs';
+
+// This is one explicit E24 scenario, not part of the default verification runner.
+// Neither child output nor browser/HTTP exception diagnostics are written to disk.
+const actor = process.argv[2] ?? 'author';
+const directory = 'evidence/phase-1/E24/container';
+const schema = fixture.schemas.container;
+const apiOrigin = `http://127.0.0.1:${fixture.roles.api.port}`;
+const workerOrigin = `http://127.0.0.1:${fixture.roles.worker.port}`;
+const browserOrigin = fixture.endpoints.trustedBrowserOrigin;
+const fixtureOrigin = fixture.endpoints.fixtureOrigin;
+const applicationCompose = ['compose', '-p', fixture.project, '-f', 'compose.production.yaml', '-f', 'compose.e24.yaml'];
+const postgresCompose = ['compose', '-p', fixture.roles.postgres.project, '-f', 'compose.yaml'];
+const frozenHashes = {
+  'evidence/phase-1/E24/fixture.json': '47f86919e5f251c3f8ee394727235b906fc03a5eac67e36cc3e273486a21a8fd',
+  'scripts/e24-seed.mjs': '16d0f9286acfe76a5dfb79819d0ee49e975098ecb710464327d41c0f54ec4025',
+  'scripts/e24-baseline.mjs': '37bdf00c299706b5e5ef2d7ff11311605a9529fa0ee7abcfa9ef78dd62ce49c4',
+  'evidence/phase-1/E24/fixtures.md': '5574f4d4d6c8e8cbbd17273ba8385e3a1ac43c2193d34d40b0dca4da11090a1c',
+};
+const uuid = /^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/;
+const secrets = runtimeSecrets();
+const privateValues = new Set(Object.values(secrets));
+const correlations = [];
+const owned = new Map();
+const signalled = new Set();
+const nativeCommands = [];
+const cleanupFailures = [];
+const started = Date.now();
+let phase = 'invocation';
+let resultPath;
+let browser;
+let browserClosed = false;
+let schemaCreated = false;
+let composeAttempted = false;
+let postgresStopAttempted = false;
+let postgresRestoreAttempted = false;
+let postgresRestored = false;
+let postgresBefore;
+let publicBefore;
+let networkOwned = false;
+let browserPageErrors = 0;
+let browserHydrationErrors = 0;
+let evidence = {
+  actor, thread: 'E24', profile: 'phase-1', specRevision: fixture.specRevision, start: fixture.start,
+  startedAt: new Date(started).toISOString(), result: 'INCOMPLETE', completed: [],
+  invocations: { fullContainerScenario: 1, browser: 0, acceptedManualIntent: 0,
+    rejectedOutageIntent: 0, postgresStop: 0, postgresRestore: 0, missingUuidRequests: 0,
+    baseline: 0, maven: 0, imageBuild: 0, load: 0, parameterSweep: 0, automaticRetry: 0 },
+  nativeCommands, exits: {}, cleanupFailures,
+  privateArtifacts: { rawLogs: false, httpBodies: false, credentials: false,
+    browserTrace: false, screenshot: false, video: false, har: false, storageState: false },
+};
+class GateFailure extends Error {
+  constructor(reason) { super('E24 observation did not satisfy the frozen contract'); this.reason = reason; }
+}
+function check(condition, reason) { if (!condition) throw new GateFailure(reason); }
+function same(left, right) { return JSON.stringify(left) === JSON.stringify(right); }
+function digest(bytes) { return createHash('sha256').update(bytes).digest('hex'); }
+function persist() { if (resultPath) writeFileSync(resultPath, JSON.stringify(evidence, null, 2) + '\n'); }
+function safeFailure(error) {
+  return { phase, reason: error instanceof GateFailure ? error.reason : 'runtime operation failed',
+    type: ['Error', 'AssertionError', 'TimeoutError', 'TypeError', 'SyntaxError'].includes(error?.name)
+      ? error.name : 'Error', privateExceptionOutputRetained: false };
+}
+// Do not let a caller's Playwright debug setting print fixture credentials.
+delete process.env.DEBUG;
+delete process.env.PWDEBUG;
+process.env.PLAYWRIGHT_NO_COPY_PROMPT = '1';
+const { E04_ALICE_PASSWORD, E04_BOB_PASSWORD, E24_BODY_SENTINEL, ...inherited } = process.env;
+const composeEnvironment = { ...inherited, DB_SCHEMA: schema, DB_USER: 'wse_industry', DB_PASSWORD: '',
+  E24_BODY_SENTINEL: secrets.bodyMarker };
+
+async function docker(args, { timeout = fixture.limits.startupWatchdogMs, allowFailure = false } = {}) {
+  const at = performance.now();
+  let stdout = '';
+  let stderr = '';
+  let timedOut = false;
+  let outputLimit = false;
+  let spawnFailed = false;
+  const child = spawn('docker', args, { env: composeEnvironment, stdio: ['ignore', 'pipe', 'pipe'] });
+  const collect = (chunk, stream) => {
+    if (stream === 'out') stdout += chunk; else stderr += chunk;
+    if (stdout.length + stderr.length > 16 * 1024 * 1024 && !outputLimit) {
+      outputLimit = true;
+      child.kill('SIGTERM'); // Stop only this observation client, never a container.
+    }
+  };
+  child.stdout.on('data', chunk => collect(chunk, 'out'));
+  child.stderr.on('data', chunk => collect(chunk, 'err'));
+  child.once('error', () => { spawnFailed = true; });
+  const timer = setTimeout(() => { timedOut = true; child.kill('SIGTERM'); }, timeout);
+  const native = await new Promise(resolve => child.once('close', (exitCode, signal) => {
+    clearTimeout(timer);
+    resolve({ command: ['docker', ...args], exitCode, signal, timedOut, outputLimit, spawnFailed,
+      elapsedMs: performance.now() - at, stdoutBytes: Buffer.byteLength(stdout),
+      stderrBytes: Buffer.byteLength(stderr), rawOutputRetained: false });
+  }));
+  nativeCommands.push(native);
+  if (!allowFailure) check(native.exitCode === 0 && !timedOut && !outputLimit && !spawnFailed,
+    'Docker command must complete successfully');
+  return { stdout: stdout.trim(), stderr: stderr.trim(), native };
+}
+async function inspect(id) {
+  const value = await docker(['inspect', id]);
+  return JSON.parse(value.stdout)[0]; // May contain runtime env; retained only in this closure.
+}
+async function applicationIds() {
+  const value = await docker(['ps', '-aq', '--filter', `label=com.docker.compose.project=${fixture.project}`]);
+  return value.stdout.split('\n').filter(Boolean);
+}
+async function discoverOwned() {
+  for (const id of await applicationIds()) {
+    const value = await inspect(id);
+    const role = value.Config.Labels['com.docker.compose.service'];
+    check(['api', 'worker', 'frontend', 'fixture'].includes(role), 'Only the four E24 services may be owned');
+    check(!owned.has(role) || owned.get(role) === value.Id, 'Exactly one container per frozen role');
+    owned.set(role, value.Id);
+  }
+  const networks = await docker(['network', 'ls', '-q', '--filter', `label=com.docker.compose.project=${fixture.project}`]);
+  const ids = networks.stdout.split('\n').filter(Boolean);
+  check(ids.length <= 1, 'Only the owned default application network may be created');
+  networkOwned = ids.length === 1;
+}
+function processIdentity(value) {
+  return { id: value.Id, image: value.Image, pid: value.State.Pid,
+    startedAt: value.State.StartedAt, restartCount: value.RestartCount };
+}
+function postgresIdentity(value) {
+  return { id: value.Id, image: value.Image, configuredImage: value.Config.Image,
+    mounts: value.Mounts.map(mount => ({ type: mount.Type, name: mount.Name ?? null, destination: mount.Destination })) };
+}
+function counts() {
+  return JSON.parse(sql(`SELECT json_build_object('users',(SELECT count(*) FROM ${schema}.users),
+    'monitors',(SELECT count(*) FROM ${schema}.monitors),'checks',(SELECT count(*) FROM ${schema}.check_runs),
+    'queued',(SELECT count(*) FROM ${schema}.check_runs WHERE state='QUEUED'),
+    'running',(SELECT count(*) FROM ${schema}.check_runs WHERE state='RUNNING'),
+    'terminal',(SELECT count(*) FROM ${schema}.check_runs WHERE state IN ('SUCCEEDED','FAILED','ABORTED')));`));
+}
+function authoritySnapshot() {
+  // Password hashes, URL markers and keys are compared in RAM, never included in evidence.
+  return sql(`SELECT jsonb_build_object(
+    'users',(SELECT jsonb_agg(to_jsonb(r) ORDER BY id) FROM ${schema}.users r),
+    'monitors',(SELECT jsonb_agg(to_jsonb(r) ORDER BY id) FROM ${schema}.monitors r),
+    'checks',(SELECT jsonb_agg(to_jsonb(r) ORDER BY id) FROM ${schema}.check_runs r),
+    'migrations',(SELECT jsonb_agg(to_jsonb(r) ORDER BY installed_rank) FROM ${schema}.flyway_schema_history r));`);
+}
+function publicSnapshot() {
+  const tables = JSON.parse(sql("SELECT coalesce(json_agg(tablename ORDER BY tablename),'[]') FROM pg_tables WHERE schemaname='public';"));
+  const hash = createHash('sha256');
+  for (const table of tables) {
+    const name = '"' + table.replaceAll('"', '""') + '"';
+    hash.update(table).update(sql(`SELECT coalesce(jsonb_agg(to_jsonb(r) ORDER BY to_jsonb(r)::text),'[]'::jsonb) FROM public.${name} r;`));
+  }
+  return { tableCount: tables.length, digest: hash.digest('hex') };
+}
+async function vacantPort(port) {
+  const probe = createServer();
+  await new Promise((resolve, reject) => {
+    probe.once('error', () => reject(new GateFailure('Refusing an occupied frozen application port')));
+    probe.listen({ host: '127.0.0.1', port, exclusive: true }, () => probe.close(resolve));
+  });
+}
+async function http(url, options = {}) {
+  try {
+    const response = await fetch(url, { ...options, signal: AbortSignal.timeout(fixture.limits.httpObservationTimeoutMs) });
+    const body = await response.text();
+    let json = null;
+    try { json = JSON.parse(body); } catch { /* HTML/static/empty responses are inspected by their caller. */ }
+    return { status: response.status, headers: response.headers, body, json };
+  } catch { throw new GateFailure('HTTP observation must complete within the frozen timeout'); }
+}
+async function api(path, cookie, route, options = {}) {
+  const response = await http(apiOrigin + path, {
+    ...options, headers: { Cookie: cookie, ...options.headers },
+  });
+  const requestId = response.headers.get('x-request-id');
+  check(uuid.test(requestId ?? '') && requestId !== secrets.untrustedRequestId,
+    'API must issue its own valid request correlation');
+  correlations.push({ requestId, method: options.method ?? 'GET', route, status: response.status });
+  return response;
+}
+async function waitUntil(observe, timeout, reason, interval = 25) {
+  const deadline = performance.now() + timeout;
+  do {
+    const value = await observe();
+    if (value) return value;
+    await delay(interval);
+  } while (performance.now() < deadline);
+  throw new GateFailure(reason);
+}
+async function health(role, readiness) {
+  const origin = role === 'api' ? apiOrigin : workerOrigin;
+  return http(origin + fixture.endpoints[readiness ? 'readiness' : 'liveness']);
+}
+async function waitReady(timeout) {
+  return waitUntil(async () => {
+    const replies = await Promise.allSettled([health('api', true), health('worker', true)]);
+    return replies.every(reply => reply.status === 'fulfilled' && reply.value.status === 200
+      && reply.value.json?.status === 'UP');
+  }, timeout, 'API and worker authority readiness must become UP');
+}
+async function metric(role, name, tags = []) {
+  const query = new URLSearchParams();
+  for (const tag of tags) query.append('tag', tag);
+  const origin = role === 'api' ? apiOrigin : workerOrigin;
+  const response = await http(`${origin}/ops/metrics/${name}${query.size ? '?' + query : ''}`);
+  check(response.status === 200 && response.json?.name === name, 'Required actual metric must be exposed');
+  return response.json;
+}
+function measurement(value, statistic) {
+  const result = value.measurements?.find(item => item.statistic === statistic)?.value;
+  check(typeof result === 'number' && Number.isFinite(result), 'Metric measurement must be finite');
+  return result;
+}
+function tags(value) {
+  return Object.fromEntries((value.availableTags ?? []).map(tag => [tag.tag, [...tag.values].sort()])
+    .sort(([left], [right]) => left.localeCompare(right)));
+}
+function workerTags(value) {
+  check(same(tags(value), { process_role: ['worker'] }), 'Worker meter uses only its bounded role label');
+}
+async function workerMetrics() {
+  const values = await Promise.all(['check.queue.age', 'check.worker.active', 'check.claims', 'check.recoveries']
+    .map(name => metric('worker', name)));
+  values.forEach(workerTags);
+  return { queueAge: measurement(values[0], 'VALUE'), active: measurement(values[1], 'VALUE'),
+    claims: measurement(values[2], 'COUNT'), recoveries: measurement(values[3], 'COUNT') };
+}
+async function fixtureStatus() {
+  const response = await http(fixtureOrigin + '/__e24/status');
+  check(response.status === 200 && typeof response.json?.outboundCalls === 'number', 'Controlled fixture status is available');
+  return response.json;
+}
+async function heldObservation() {
+  // Critical path: no browser action, SQL, process inspection or Docker CLI before release.
+  const held = await waitUntil(async () => {
+    const state = await fixtureStatus();
+    check(state.watchdogReleases === 0, 'The 350ms safety watchdog is not an accepted release');
+    return state.held === 1 ? state : null;
+  }, fixture.limits.httpObservationTimeoutMs, 'Observe the single real held outbound request', 10);
+  let active;
+  let released;
+  const at = performance.now();
+  try { active = await metric('worker', 'check.worker.active'); }
+  finally { released = await http(fixtureOrigin + '/__e24/release', { method: 'POST' }); }
+  const elapsedMs = performance.now() - at;
+  check(released.status === 200, 'Explicit held request release succeeds');
+  workerTags(active);
+  check(measurement(active, 'VALUE') === 1, 'Actual worker active gauge is one while outbound is held');
+  return { held: held.held, outboundCalls: held.outboundCalls, active: 1,
+    localObservationAndReleaseMs: elapsedMs, dockerOrSqlBeforeRelease: false };
+}
+async function restorePostgres() {
+  check(postgresStopAttempted && !postgresRestoreAttempted, 'Only one stop/restore sequence is allowed');
+  postgresRestoreAttempted = true;
+  evidence.invocations.postgresRestore++;
+  await docker([...postgresCompose, 'start', 'postgres'], { timeout: fixture.limits.postgresRestoreWatchdogMs });
+  await waitUntil(async () => {
+    const value = await inspect(postgresBefore.id);
+    return value.State.Running && value.State.Health?.Status === 'healthy';
+  }, fixture.limits.postgresRestoreWatchdogMs, 'The same PostgreSQL container must restore healthy', 100);
+  postgresRestored = true;
+}
+async function nativeExit(role, purpose) {
+  const id = owned.get(role);
+  check(id && !signalled.has(role), 'Each owned container receives SIGTERM at most once');
+  const before = await inspect(id);
+  check(before.State.Running, 'The actual container must still be running at its signal boundary');
+  signalled.add(role);
+  const at = performance.now();
+  const waiting = docker(['wait', id], { timeout: fixture.limits.exitObservationWatchdogMs, allowFailure: true });
+  const signal = await docker(['kill', '--signal=SIGTERM', id], {
+    timeout: fixture.limits.exitObservationWatchdogMs, allowFailure: true,
+  });
+  const waited = await waiting;
+  const elapsedMs = performance.now() - at;
+  const observedCode = /^\d+$/.test(waited.stdout) ? Number(waited.stdout) : null;
+  evidence.exits[role] = { purpose, signal: 'SIGTERM', signalCommandExit: signal.native.exitCode,
+    exitCode: observedCode, elapsedMs, awaited: waited.native.exitCode === 0 && !waited.native.timedOut,
+    observerTimedOut: waited.native.timedOut, forcedContainerKill: false };
+  persist(); // Retain the actual native result BEFORE evaluating the frozen criterion.
+  check(signal.native.exitCode === 0 && !signal.native.timedOut && evidence.exits[role].awaited,
+    'SIGTERM and the native exit observation must complete');
+  check(observedCode === (fixture.nativeSigterm[role] ?? 143), 'Actual container must exit with native SIGTERM143');
+  check(elapsedMs <= fixture.limits.gracefulExitBoundMs, 'Actual container exit must meet the frozen 5000ms bound');
+}
+async function runtimeInspection(role, jarHash) {
+  const value = await inspect(owned.get(role));
+  check(value.State.Running && value.RestartCount === 0, 'Production container has a live unrestarted process');
+  const status = (await docker(['exec', value.Id, 'cat', '/proc/1/status'])).stdout;
+  const uid = Number(status.match(/^Uid:\s+(\d+)/m)?.[1]);
+  check(/^Pid:\s+1$/m.test(status) && uid === fixture.roles[role].uid && uid !== 0,
+    'The actual PID1 runs under the frozen non-root UID');
+  const command = [value.Path, ...value.Args];
+  let versions;
+  if (role === 'frontend') {
+    check(same(command, ['node', 'server.js']), 'Frontend starts the standalone server directly as PID1');
+    const version = await docker(['exec', value.Id, 'node', '-e',
+      'console.log(JSON.stringify({node:process.versions.node,next:require("next/package.json").version,react:require("react/package.json").version,mode:process.env.NODE_ENV,origin:process.env.API_ORIGIN,manualSignal:!!process.env.NEXT_MANUAL_SIG_HANDLE}))']);
+    const actual = JSON.parse(version.stdout);
+    check(same(actual, { node: '24.19.0', next: '16.3.3', react: '19.2.8', mode: 'production',
+      origin: fixture.endpoints.serverOriginAtBuildAndRuntime, manualSignal: false }), 'Runtime frontend artifacts and signal handling retain their exact pins');
+    versions = actual;
+  } else {
+    const expected = ['java', '-jar', '/app/app.jar', ...(role === 'worker'
+      ? ['--spring.profiles.active=worker', '--spring.main.web-application-type=none'] : [])];
+    check(same(command, expected), 'Java runs the production jar directly at the container process boundary');
+    const version = await docker(['exec', value.Id, 'java', '-version']);
+    check((version.stdout + version.stderr).includes('21.0.7') && (version.stdout + version.stderr).includes('21.0.7+6'),
+      'Actual Java runtime retains its exact version');
+    const artifact = await docker(['exec', value.Id, 'sha256sum', '/app/app.jar']);
+    check(artifact.stdout.split(/\s+/)[0] === jarHash, 'Runtime jar is the affected Maven gate artifact');
+    versions = { java: '21.0.7+6', jarSha256: jarHash };
+  }
+  return { ...processIdentity(value), pidInsideContainer: 1, uid, command, versions };
+}
+async function inspectLogs(requireComplete = true) {
+  const records = {};
+  let sentinelMatches = 0;
+  let unboundedInputMatches = 0;
+  const inputs = [...fixture.seed.names, fixture.seed.alice, fixture.seed.bob,
+    fixture.seed.monitorA, fixture.seed.monitorB, ...fixture.seed.checkIds];
+  for (const role of ['api', 'worker', 'frontend', 'fixture']) {
+    if (!owned.has(role)) continue;
+    const output = await docker(['logs', owned.get(role)]);
+    const raw = output.stdout + '\n' + output.stderr;
+    sentinelMatches += [...privateValues].filter(value => typeof value === 'string' && value && raw.includes(value)).length;
+    unboundedInputMatches += inputs.filter(value => raw.includes(value)).length;
+    const lines = raw.split('\n').filter(line => line.trim());
+    const json = lines.flatMap(line => { try { return [JSON.parse(line)]; } catch { return []; } });
+    records[role] = { lines: lines.length, jsonLines: json.length, events: json.filter(row => row.process_role === role) };
+  }
+  const apiEvents = records.api?.events ?? [];
+  const workerEvents = records.worker?.events ?? [];
+  const apiIds = [...new Set(apiEvents.map(row => row.process_id))];
+  const workerIds = [...new Set(workerEvents.map(row => row.process_id))];
+  const matched = correlations.filter(expected => apiEvents.some(row => row.event === 'http_request'
+    && row.request_id === expected.requestId && row.method === expected.method && row.route === expected.route
+    && row.status === expected.status && typeof row.duration_ms === 'number' && row.duration_ms >= 0)).length;
+  evidence.logs = { roles: Object.fromEntries(Object.entries(records).map(([role, record]) => [role,
+    { lines: record.lines, structuredLines: record.jsonLines, observationEvents: record.events.length }])),
+    runtimeSentinelValuesScanned: privateValues.size, runtimeSentinelMatches: sentinelMatches,
+    unboundedInputMatches, responseCorrelations: correlations.length, matchedResponseCorrelations: matched,
+    generatedResponseRequestIds: correlations.map(row => row.requestId),
+    stableApiProcess: apiIds.length === 1 && uuid.test(apiIds[0]),
+    stableWorkerProcess: workerIds.length === 1 && uuid.test(workerIds[0]),
+    distinctRoleProcesses: apiIds[0] !== workerIds[0],
+    apiProcessId: apiIds.length === 1 && uuid.test(apiIds[0]) ? apiIds[0] : null,
+    workerProcessId: workerIds.length === 1 && uuid.test(workerIds[0]) ? workerIds[0] : null,
+    committedClaimEvents: workerEvents.filter(row => row.event === 'check_claimed').length,
+    unavailableAuthorityEvents: workerEvents.filter(row => row.event === 'worker_store_unavailable').length,
+    recoveryEvents: workerEvents.filter(row => row.event === 'checks_recovered').length,
+    crossProcessCheckRunTraceClaimed: false, rawLogsRetained: false };
+  check(sentinelMatches === 0 && unboundedInputMatches === 0, 'No runtime credential or unbounded input may appear in logs');
+  if (!requireComplete) return;
+  check(correlations.length > 0 && matched === correlations.length, 'Generated response IDs correlate to actual structured API lines');
+  check(new Set(correlations.map(row => row.requestId)).size === correlations.length,
+    'Each observed API response uses a fresh generated request ID');
+  check(evidence.logs.stableApiProcess && evidence.logs.stableWorkerProcess && evidence.logs.distinctRoleProcesses,
+    'Request and worker events have stable distinct process correlation');
+  check(evidence.logs.committedClaimEvents === 1 && evidence.logs.unavailableAuthorityEvents > 0
+    && evidence.logs.recoveryEvents === 0, 'Worker logs reflect the single actual claim and authority outage');
+}
+async function cleanupStep(name, action) {
+  try { await action(); return true; }
+  catch (error) { cleanupFailures.push({ action: name, ...safeFailure(error) }); return false; }
+}
+
+try {
+  check(['author', 'root', 'ci'].includes(actor) && process.argv.length <= 3, 'Actor must be author, root or ci');
+  const head = execFileSync('git', ['rev-parse', 'HEAD'], { encoding: 'utf8' }).trim();
+  const sourcePaths = execFileSync('git', ['ls-files', '-z', '--', 'app', 'backend/src/main', 'backend/pom.xml',
+    'backend/src/test/java/dev/evolution/monitor/OperationsIntegrationTest.java', 'scripts/e24-fixture.mjs',
+    'scripts/e24-seed.mjs', 'scripts/e24-container.mjs', 'Dockerfile.api', 'Dockerfile.frontend',
+    'compose.production.yaml', 'compose.e24.yaml', 'compose.yaml', 'package.json', 'package-lock.json',
+    'next.config.mjs', '.github/workflows/ci.yml'], { encoding: 'utf8' }).split('\0').filter(Boolean);
+  sourcePaths.push('scripts/e24-container.mjs'); // Also fingerprint this file before its first commit.
+  const source = createHash('sha256');
+  for (const path of [...new Set(sourcePaths)].sort()) source.update(path).update('\0').update(readFileSync(path));
+  const fingerprint = source.digest('hex');
+  mkdirSync(directory, { recursive: true });
+  const stem = `${directory}/${actor}-${fingerprint.slice(0, 16)}`;
+  appendFileSync(`${directory}/invocations.jsonl`, JSON.stringify({ actor, head, fingerprint,
+    startedAt: evidence.startedAt, command: `node scripts/e24-container.mjs ${actor}` }) + '\n');
+  writeFileSync(stem + '.started.json', JSON.stringify({ actor, head, fingerprint, startedAt: evidence.startedAt }) + '\n', { flag: 'wx' });
+  resultPath = stem + '.json';
+  evidence = { ...evidence, head, sourceFingerprint: fingerprint, frozenHashes };
+  persist();
+  phase = 'frozen inputs and collision refusal';
+  for (const [path, expected] of Object.entries(frozenHashes)) check(digest(readFileSync(path)) === expected, 'Frozen E24 input hash must match');
+  check(readFileSync('SPEC_REVISION', 'utf8').trim() === fixture.specRevision, 'The pinned specification revision must match');
+  check(process.versions.node === '24.19.0', 'Use the pinned Node24.19.0 runtime');
+  check(!process.env.DB_URL || process.env.DB_URL === 'jdbc:postgresql://127.0.0.1:15432/monitor', 'Refusing a different bootstrap database');
+  check(!process.env.DB_USER || process.env.DB_USER === 'wse_industry', 'Refusing a different bootstrap database user');
+  check(!process.env.DB_PASSWORD, 'This frozen disposable authority uses explicit local test trust');
+  const ports = ['api', 'worker', 'frontend', 'fixture'].map(role => fixture.roles[role].port);
+  const vacant = await Promise.allSettled(ports.map(vacantPort));
+  check(vacant.every(result => result.status === 'fulfilled'), 'All frozen application ports must be vacant');
+  check((await applicationIds()).length === 0, 'Refusing an existing E24 application container');
+  check((await docker(['network', 'ls', '-q', '--filter', `label=com.docker.compose.project=${fixture.project}`])).stdout === '',
+    'Refusing an existing E24 application network');
+  check(!(await docker(['network', 'ls', '--format', '{{.Name}}'])).stdout.split('\n').includes(`${fixture.project}_default`),
+    'Refusing an existing default network even without an ownership label');
+  check((await docker(['volume', 'ls', '-q', '--filter', `label=com.docker.compose.project=${fixture.project}`])).stdout === '',
+    'Refusing an existing E24 application volume');
+  await docker(['image', 'inspect', 'wse-industry-e24-api:local', 'wse-industry-e24-frontend:local']);
+  const postgresId = (await docker([...postgresCompose, 'ps', '--all', '--quiet', 'postgres'])).stdout;
+  check(/^[0-9a-f]{12,64}$/.test(postgresId), 'Exactly one existing owned PostgreSQL container is required');
+  const postgres = await inspect(postgresId);
+  check(postgres.Config.Labels['com.docker.compose.project'] === fixture.roles.postgres.project
+    && postgres.Config.Labels['com.docker.compose.service'] === 'postgres'
+    && postgres.State.Running && postgres.State.Health?.Status === 'healthy'
+    && postgres.Config.Image === fixture.images.postgres, 'The frozen PostgreSQL authority is healthy and correctly owned');
+  postgresBefore = postgresIdentity(postgres);
+  publicBefore = publicSnapshot();
+  check(sql(`SELECT count(*) FROM pg_namespace WHERE nspname='${schema}';`) === '0', 'Refusing an existing container schema');
+  evidence.preflight = { vacantPorts: ports, existingApplicationResources: 0, postgres: postgresBefore,
+    publicTableCount: publicBefore.tableCount, frozenInputsUnchanged: true };
+
+  phase = 'fixed seed and production startup';
+  const urls = seed(schema, secrets, () => { schemaCreated = true; });
+  privateValues.add(urls.urlA); privateValues.add(urls.urlB);
+  const seeded = counts();
+  check(same(seeded, { users: 2, monitors: 2, checks: 4, queued: 0, running: 0, terminal: 4 }), 'The frozen seed is exactly 2users/2Monitors/4results');
+  evidence.seed = seeded;
+  composeAttempted = true;
+  await docker([...applicationCompose, 'up', '--detach', '--no-build', 'api', 'worker', 'frontend', 'fixture']);
+  await discoverOwned();
+  check(owned.size === 4, 'All four owned production/fixture containers were created');
+  await waitReady(fixture.limits.startupWatchdogMs);
+  const live = await Promise.all(['api', 'worker'].map(role => health(role, false)));
+  check(live.every(response => response.status === 200 && response.json?.status === 'UP'), 'Both processes are live while authority is ready');
+  await waitUntil(async () => {
+    try { return (await http(browserOrigin + '/login')).status === 200; } catch { return false; }
+  }, fixture.limits.startupWatchdogMs, 'The standalone frontend serves its real route');
+  const initialFixture = await fixtureStatus();
+  check(initialFixture.outboundCalls === 0 && initialFixture.held === 0, 'Disabled frozen Monitors cause no startup outbound');
+  const initialWorker = await workerMetrics();
+  check(same(initialWorker, { queueAge: 0, active: 0, claims: 0, recoveries: 0 }), 'Idle worker measurements reflect the empty queue');
+  evidence.startup = { liveness: { api: 200, worker: 200 }, readiness: { api: 200, worker: 200 }, worker: initialWorker };
+  evidence.completed.push('production roles ready; exact seed and idle metrics');
+
+  phase = 'authenticated production browser and server output';
+  const { chromium, expect } = await import('@playwright/test');
+  evidence.invocations.browser++;
+  browser = await chromium.launch({ headless: true, env: inherited });
+  const browserVersion = browser.version();
+  check(browserVersion === '151.0.7922.34', 'Use the pinned Chromium artifact');
+  const contexts = [];
+  async function login(username, password) {
+    const context = await browser.newContext({ baseURL: browserOrigin, acceptDownloads: false });
+    contexts.push(context);
+    context.setDefaultTimeout(fixture.limits.httpObservationTimeoutMs);
+    const csrf = await context.request.get('/api/session/csrf', { timeout: fixture.limits.httpObservationTimeoutMs });
+    check(csrf.status() === 200, 'Browser session CSRF bootstrap succeeds');
+    const data = (await csrf.json()).data;
+    check(data?.headerName === 'X-CSRF-TOKEN' && typeof data.token === 'string', 'CSRF token uses the existing contract');
+    privateValues.add(data.token);
+    for (const cookie of await context.cookies()) privateValues.add(cookie.value);
+    const response = await context.request.post('/api/session/login', {
+      headers: { [data.headerName]: data.token, Origin: browserOrigin }, data: { username, password },
+      timeout: fixture.limits.httpObservationTimeoutMs,
+    });
+    check(response.status() === 200, 'The runtime-prepared browser user authenticates');
+    const cookies = await context.cookies();
+    for (const cookie of cookies) privateValues.add(cookie.value);
+    const session = cookies.find(cookie => cookie.name === 'WSESESSION');
+    check(session?.httpOnly && session.sameSite === 'Lax', 'The actual browser session retains its protected cookie');
+    const page = await context.newPage();
+    page.on('pageerror', () => { browserPageErrors++; });
+    page.on('console', message => {
+      if (message.type() === 'error' && /hydrat|server rendered html/i.test(message.text())) browserHydrationErrors++;
+    });
+    return { context, page, cookie: `WSESESSION=${session.value}` };
+  }
+  const alice = await login('alice-e04', secrets.E04_ALICE_PASSWORD);
+  const bob = await login('bob-e04', secrets.E04_BOB_PASSWORD);
+  for (const [session, name, hidden, monitorId, historyCount] of [
+    [alice, fixture.seed.names[0], fixture.seed.names[1], fixture.seed.monitorA, 3],
+    [bob, fixture.seed.names[1], fixture.seed.names[0], fixture.seed.monitorB, 1],
+  ]) {
+    const navigation = await session.page.goto('/monitors');
+    check(navigation?.status() === 200, 'Authenticated production list route succeeds');
+    const html = await navigation.text();
+    check(html.includes(name) && !html.includes(hidden), 'Server-rendered HTML contains only the current owner data');
+    await expect(session.page.getByRole('article')).toHaveCount(1);
+    const article = session.page.getByRole('article', { name, exact: true });
+    await expect(article).toBeVisible();
+    await expect(session.page.getByRole('article', { name: hidden, exact: true })).toHaveCount(0);
+    await article.getByRole('button', { name: 'Show history', exact: true }).click();
+    await expect(article.locator('tr[data-check-id]')).toHaveCount(historyCount);
+    const expectedState = historyCount === 3 ? 'FAILED' : 'SUCCEEDED';
+    await expect(article.locator('tbody').getByText(expectedState, { exact: true })).toHaveCount(historyCount);
+    const own = await api(`/api/monitors/${monitorId}`, session.cookie, '/api/monitors/{id}');
+    check(own.status === 200 && own.json?.data?.monitor?.id === monitorId, 'Existing owner detail API is served by the container');
+  }
+  const foreignA = await api(`/api/monitors/${fixture.seed.monitorB}`, alice.cookie, '/api/monitors/{id}');
+  const foreignB = await api(`/api/monitors/${fixture.seed.monitorA}`, bob.cookie, '/api/monitors/{id}');
+  check(foreignA.status === 404 && foreignB.status === 404
+    && foreignA.json?.error?.code === 'NOT_FOUND' && foreignB.json?.error?.code === 'NOT_FOUND', 'Both owner boundaries conceal foreign detail');
+  const serverHtml = await http(browserOrigin + `/monitors?history=${fixture.seed.monitorA}&limit=20`, { headers: { Cookie: alice.cookie } });
+  check(serverHtml.status === 200 && serverHtml.body.includes(fixture.seed.names[0])
+    && !serverHtml.body.includes(fixture.seed.names[1]) && fixture.seed.checkIds.slice(0, 3).every(id => serverHtml.body.includes(id)),
+  'Server route includes the authenticated fixed history before client code runs');
+  const staticPaths = [...new Set([...serverHtml.body.matchAll(/(?:src|href)="([^" ]*\/_next\/static\/[^" ]+)"/g)]
+    .map(match => match[1].replaceAll('&amp;', '&')))];
+  const script = staticPaths.find(path => new URL(path, browserOrigin).pathname.endsWith('.js'));
+  const stylesheet = staticPaths.find(path => new URL(path, browserOrigin).pathname.endsWith('.css'));
+  check(script && stylesheet, 'Standalone HTML references actual JavaScript and CSS static assets');
+  const assets = await Promise.all([script, stylesheet].map(path => http(new URL(path, browserOrigin))));
+  check(assets.every(asset => asset.status === 200 && asset.body.length > 0)
+    && /javascript/.test(assets[0].headers.get('content-type') ?? '')
+    && /^text\/css(?:;|$)/.test(assets[1].headers.get('content-type') ?? ''),
+  'Both copied static asset types are served by the frontend');
+  evidence.browser = { chromium: browserVersion, contexts: 2, ownerArticleCounts: [1, 1], initialHistoryRows: [3, 1],
+    foreignDetailStatuses: [foreignA.status, foreignB.status], authenticatedServerHtml: true,
+    serverHistoryBeforeHydration: true, javascriptStatus: assets[0].status, cssStatus: assets[1].status };
+
+  phase = 'one real manual intent and immediate held observation';
+  const article = alice.page.getByRole('article', { name: fixture.seed.names[0], exact: true });
+  const checkPath = `/api/monitors/${fixture.seed.monitorA}/checks`;
+  evidence.invocations.acceptedManualIntent++;
+  const observed = await Promise.allSettled([
+    alice.page.waitForResponse(response => new URL(response.url()).pathname === checkPath && response.request().method() === 'POST'),
+    heldObservation(),
+    article.getByRole('button', { name: 'Run check', exact: true }).click(),
+  ]);
+  for (const value of observed) if (value.status === 'rejected') throw value.reason;
+  const accepted = observed[0].value;
+  check(accepted.status() === 202, 'One real browser manual intent is accepted as202');
+  const queued = (await accepted.json()).data;
+  check(uuid.test(queued?.id ?? '') && queued.state === 'QUEUED'
+    && ['startedAt', 'finishedAt', 'httpStatus', 'latencyMs', 'failureReason'].every(field => queued[field] === null),
+  'The accepted intent has its durable identity and no invented outcome');
+  privateValues.add(queued.id);
+  const acceptedHeaders = await accepted.request().allHeaders();
+  check(typeof acceptedHeaders['idempotency-key'] === 'string' && typeof acceptedHeaders['x-csrf-token'] === 'string',
+    'Browser manual intent uses the existing CSRF and idempotency boundary');
+  privateValues.add(acceptedHeaders['idempotency-key']); privateValues.add(acceptedHeaders['x-csrf-token']);
+  const acceptedRequestId = (await accepted.allHeaders())['x-request-id'];
+  check(uuid.test(acceptedRequestId ?? ''), 'The accepted response has generated request correlation');
+  correlations.push({ requestId: acceptedRequestId, method: 'POST', route: '/api/monitors/{id}/checks', status: 202 });
+  const released = await fixtureStatus();
+  check(released.outboundCalls === 1 && released.holdRequests === 1 && released.held === 0
+    && released.releases === 1 && released.watchdogReleases === 0 && released.lastHeldMs > 0
+    && released.lastHeldMs < fixture.limits.fixtureHoldSafetyReleaseMs, 'The one outbound request is explicitly released before350ms');
+  const terminal = await waitUntil(async () => {
+    const response = await api(`${checkPath}/${queued.id}`, alice.cookie, '/api/monitors/{id}/checks/{checkId}');
+    check(response.status === 200 && response.json?.data?.id === queued.id, 'Direct check read retains the accepted identity');
+    return ['SUCCEEDED', 'FAILED', 'ABORTED'].includes(response.json.data.state) ? response.json.data : null;
+  }, fixture.limits.httpObservationTimeoutMs, 'The one actual worker check reaches terminal state');
+  check(terminal.state === 'SUCCEEDED' && terminal.httpStatus === 200 && typeof terminal.latencyMs === 'number'
+    && terminal.startedAt && terminal.finishedAt && terminal.failureReason === null, 'Terminal outcome reflects the actual released HTTP200');
+  const latest = article.locator('[data-latest-check-id]');
+  await expect(latest).toHaveAttribute('data-latest-check-id', queued.id);
+  await expect(latest.getByText('SUCCEEDED', { exact: true })).toBeVisible();
+  await expect(latest.getByText('HTTP 200', { exact: true })).toBeVisible();
+  // Show history intentionally uses limit3; use the existing URL contract for all four results.
+  await alice.page.goto(`/monitors?history=${fixture.seed.monitorA}&limit=20`);
+  await expect(article.locator('tr[data-check-id]')).toHaveCount(4);
+  await expect(article.locator(`tr[data-check-id="${queued.id}"]`)).toHaveCount(1);
+  await alice.page.reload();
+  await expect(article.locator('tr[data-check-id]')).toHaveCount(4);
+  await expect(article.locator(`tr[data-check-id="${queued.id}"]`)).toHaveCount(1);
+  await expect(alice.page.getByRole('main').getByRole('alert')).toHaveCount(0);
+  evidence.manual = { acceptedStatus: 202, acceptedState: queued.state, acceptedOutcomeFieldsNull: true,
+    held: observed[1].value, fixture: released, sameTerminalId: terminal.id === queued.id,
+    terminalState: terminal.state, httpStatus: terminal.httpStatus, historyRows: 4, reloadRetainsResult: true };
+  const csrfResponse = await api('/api/session/csrf', alice.cookie, '/api/session/csrf');
+  const csrf = csrfResponse.json?.data;
+  check(csrfResponse.status === 200 && csrf?.headerName === 'X-CSRF-TOKEN' && typeof csrf.token === 'string',
+    'Outage mutation receives its valid CSRF token before authority stops');
+  privateValues.add(csrf.token);
+  for (const context of contexts) await context.close();
+  await browser.close();
+  browserClosed = true;
+  check(browserPageErrors === 0 && browserHydrationErrors === 0, 'The actual production browser has no page or hydration error');
+  evidence.browser.pageErrors = browserPageErrors;
+  evidence.browser.hydrationErrors = browserHydrationErrors;
+  evidence.browser.closedBeforeOutage = true;
+  evidence.completed.push('one real browser intent; active held worker; released200; four-result history and reload');
+
+  phase = 'fixed idle boundary and PostgreSQL outage';
+  const beforeCounts = counts();
+  check(same(beforeCounts, { users: 2, monitors: 2, checks: 5, queued: 0, running: 0, terminal: 5 }), 'All five existing results are terminal at the fixed outage boundary');
+  const beforeRows = authoritySnapshot();
+  const idleWorker = await workerMetrics();
+  check(same(idleWorker, { queueAge: 0, active: 0, claims: 1, recoveries: 0 }), 'Worker is idle after its one committed claim');
+  const beforeProcesses = {};
+  for (const role of ['api', 'worker', 'frontend']) beforeProcesses[role] = processIdentity(await inspect(owned.get(role)));
+  check((await fixtureStatus()).outboundCalls === 1, 'The fixed outage boundary has one completed outbound');
+  postgresStopAttempted = true;
+  evidence.invocations.postgresStop++;
+  await docker([...postgresCompose, 'stop', '--timeout', '5', 'postgres']);
+  const unavailable = await Promise.all([health('api', false), health('worker', false), health('api', true), health('worker', true)]);
+  check(same(unavailable.map(response => response.status), [200, 200, 503, 503]), 'PostgreSQL loss separates process liveness from authority readiness');
+  evidence.invocations.rejectedOutageIntent++;
+  const rejected = await api(checkPath, alice.cookie, '/api/monitors/{id}/checks', {
+    method: 'POST', headers: { [csrf.headerName]: csrf.token, Origin: browserOrigin,
+      'Idempotency-Key': secrets.outageKey },
+  });
+  check(rejected.status === 503 && rejected.json?.error?.code === 'INTERNAL_ERROR'
+    && !Object.hasOwn(rejected.json, 'data') && !Object.hasOwn(rejected.json, 'id'), 'Unavailable authority rejects the sole unsafe submission without acceptance');
+  const downActive = measurement(await metric('worker', 'check.worker.active'), 'VALUE');
+  const downClaims = measurement(await metric('worker', 'check.claims'), 'COUNT');
+  const downRecoveries = measurement(await metric('worker', 'check.recoveries'), 'COUNT');
+  check(downActive === 0 && downClaims === 1 && downRecoveries === 0 && (await fixtureStatus()).outboundCalls === 1,
+    'Authority absence creates no worker claim, recovery or outbound outcome');
+  for (const role of ['api', 'worker', 'frontend']) {
+    const value = await inspect(owned.get(role));
+    check(value.State.Running && same(processIdentity(value), beforeProcesses[role]), 'Application process remains alive and unchanged through the outage');
+  }
+  evidence.outage = { liveness: { api: unavailable[0].status, worker: unavailable[1].status },
+    readiness: { api: unavailable[2].status, worker: unavailable[3].status },
+    rejectedStatus: rejected.status, acceptedId: false, active: downActive, claims: downClaims,
+    recoveries: downRecoveries, outboundCalls: 1, sameLiveProcesses: true };
+  phase = 'same PostgreSQL authority restore';
+  await restorePostgres();
+  await waitReady(fixture.limits.postgresRestoreWatchdogMs);
+  const restoredPg = postgresIdentity(await inspect(postgresBefore.id));
+  const restoredPublic = publicSnapshot();
+  check(same(restoredPg, postgresBefore), 'Restore preserves the same PostgreSQL container, image and volumes');
+  check(same(restoredPublic, publicBefore) && authoritySnapshot() === beforeRows, 'Restore preserves public authority and every pre-existing private row');
+  check(same(counts(), beforeCounts) && same(await workerMetrics(), idleWorker)
+    && (await fixtureStatus()).outboundCalls === 1, 'No intent or result was created during the outage');
+  for (const role of ['api', 'worker', 'frontend']) {
+    check(same(processIdentity(await inspect(owned.get(role))), beforeProcesses[role]), 'Restored authority uses the same application processes');
+  }
+  evidence.restore = { readiness: { api: 200, worker: 200 }, samePostgresContainerImageVolumes: true,
+    publicAuthorityUnchanged: true, exactPrivateRowsUnchanged: true, counts: beforeCounts,
+    rejectedIntentPersisted: false, sameLiveProcesses: true };
+  evidence.completed.push('one idle PostgreSQL stop/restore;503 unsafe rejection; same authority and processes');
+
+  phase = 'bounded real metrics and ten missing UUID requests';
+  const names = await Promise.all([http(apiOrigin + fixture.endpoints.metricNames), http(workerOrigin + fixture.endpoints.metricNames)]);
+  check(same(names[0].json?.names?.slice().sort(), ['check.queue.age', 'http.server.requests'])
+    && same(names[1].json?.names?.slice().sort(), ['check.claims', 'check.queue.age', 'check.recoveries', 'check.worker.active']),
+  'Only the applicable API and worker metric families are exposed');
+  const queue = await metric('api', 'check.queue.age');
+  check(measurement(queue, 'VALUE') === 0 && same(tags(queue), { process_role: ['api'] }), 'API queue age reflects the actual empty queue');
+  const httpSource = readFileSync('backend/src/main/java/dev/evolution/monitor/HttpObservations.java', 'utf8');
+  const defined = name => [...httpSource.match(new RegExp(`${name} = Set\\.of\\(([\\s\\S]*?)\\);`))[1].matchAll(/"([^"]+)"/g)].map(match => match[1]);
+  const routes = new Set([...defined('ROUTES'), 'UNMATCHED']);
+  const methods = new Set([...defined('METHODS'), 'OTHER']);
+  check(httpSource.includes('METHODS.contains(value) ? value : "OTHER"')
+    && httpSource.includes('ROUTES.contains(value) ? value : "UNMATCHED"')
+    && /getHighCardinalityKeyValues[\s\S]*?return KeyValues\.empty\(\)/.test(httpSource),
+  'Source normalizes arbitrary methods/unmatched routes and omits high-cardinality HTTP values');
+  await metric('api', 'http.server.requests'); // Establish the metric-value route before comparing tag sets.
+  const beforeHttp = await metric('api', 'http.server.requests');
+  const selection = ['uri:/api/monitors/{id}', 'method:GET', 'status:404', 'process_role:api'];
+  const beforeErrors = measurement(await metric('api', 'http.server.requests', selection), 'COUNT');
+  for (let index = 1; index <= 10; index++) {
+    const id = `24000000-0000-4000-9000-${String(index).padStart(12, '0')}`;
+    evidence.invocations.missingUuidRequests++;
+    const missing = await api(`/api/monitors/${id}`, alice.cookie, '/api/monitors/{id}', {
+      headers: { 'X-Request-Id': secrets.untrustedRequestId },
+    });
+    check(missing.status === 404 && missing.json?.error?.code === 'NOT_FOUND', 'Each frozen missing UUID has the existing404 contract');
+  }
+  const afterErrors = measurement(await metric('api', 'http.server.requests', selection), 'COUNT');
+  const afterHttp = await metric('api', 'http.server.requests');
+  const bounded = tags(afterHttp);
+  check(same(Object.keys(bounded).sort(), [...fixture.metrics.httpTags].sort()), 'HTTP metric has exactly the source-defined tag dimensions');
+  check(bounded.uri.every(value => routes.has(value)) && bounded.method.every(value => methods.has(value))
+    && bounded.status.every(value => /^(?:[1-5]\d\d|UNKNOWN)$/.test(value))
+    && same(bounded.process_role, ['api']), 'Every actual tag value belongs to the bounded source categories');
+  check(same(tags(beforeHttp), bounded) && afterErrors - beforeErrors === 10, 'Ten distinct UUIDs add ten404 errors without new tag values');
+  const httpMeasurements = Object.fromEntries(['COUNT', 'TOTAL_TIME', 'MAX'].map(name => [name, measurement(afterHttp, name)]));
+  check(httpMeasurements.COUNT > 0 && httpMeasurements.TOTAL_TIME > 0 && httpMeasurements.MAX > 0, 'Actual HTTP count, total latency and maximum latency are observed');
+  evidence.metrics = { apiQueueAge: 0, worker: await workerMetrics(), httpMeasurements, boundedHttpTags: bounded,
+    missingUuidRequests: 10, before404Count: beforeErrors, after404Count: afterErrors, errorCountDelta: afterErrors - beforeErrors,
+    tagSetsUnchanged: true, sourceNormalizationSha256: digest(httpSource), exactUniqueSeriesCountClaimed: false,
+    positiveQueueAgeAndRecoveryProof: 'separate affected OperationsIntegrationTest; no extra crash or outage scenario' };
+  evidence.completed.push('actual bounded latency/error and worker meters; ten UUIDs keep tag sets unchanged');
+
+  phase = 'actual non-root production runtime inspection';
+  const jarHash = digest(readFileSync('backend/target/monitor-api-0.0.1.jar'));
+  evidence.runtime = {};
+  for (const role of ['api', 'worker', 'frontend']) evidence.runtime[role] = await runtimeInspection(role, jarHash);
+  check(evidence.runtime.api.image === evidence.runtime.worker.image, 'API and worker execute the same production backend image');
+  phase = 'native production container SIGTERM and awaited exits';
+  for (const role of ['frontend', 'worker', 'api']) await nativeExit(role, 'frozen production signal acceptance');
+  await nativeExit('fixture', 'owned fixture cleanup');
+  phase = 'RAM-only structured-log and sentinel inspection';
+  await inspectLogs();
+  evidence.completed.push('non-root PID1 artifacts; each native143 exit within5000ms; safe correlated structured logs');
+  evidence.result = 'PASS';
+} catch (error) {
+  evidence.result = 'FAIL';
+  evidence.failure = safeFailure(error);
+} finally {
+  phase = 'owned resource cleanup';
+  if (browser && !browserClosed) browserClosed = await cleanupStep('await owned browser close', () => browser.close());
+  if (postgresStopAttempted && !postgresRestoreAttempted) await cleanupStep('restore the stopped owned PostgreSQL service once', restorePostgres);
+  let cleanupDiscovered = !composeAttempted;
+  if (composeAttempted) cleanupDiscovered = await cleanupStep('discover only the collision-checked application resources', discoverOwned);
+  for (const role of ['frontend', 'worker', 'api', 'fixture']) {
+    if (!owned.has(role) || signalled.has(role)) continue;
+    await cleanupStep(`await ${role} cleanup SIGTERM`, async () => {
+      const value = await inspect(owned.get(role));
+      if (value.State.Running) await nativeExit(role, 'cleanup after incomplete/failed gate');
+      else evidence.exits[role] = { purpose: 'already exited before cleanup', exitCode: value.State.ExitCode,
+        awaited: true, signalSent: false, acceptance: false, forcedContainerKill: false };
+    });
+  }
+  if (owned.size > 0 && !evidence.logs) await cleanupStep('scan failed gate logs in RAM', () => inspectLogs(false));
+  let allRemoved = cleanupDiscovered;
+  for (const role of ['fixture', 'frontend', 'worker', 'api']) {
+    if (!owned.has(role)) continue;
+    const removed = await cleanupStep(`remove exited owned ${role} container`, async () => {
+      const value = await inspect(owned.get(role));
+      check(!value.State.Running, 'Never force-remove a running container');
+      await docker(['rm', owned.get(role)]);
+    });
+    allRemoved = removed && allRemoved;
+  }
+  let networkRemoved = !networkOwned;
+  if (networkOwned && allRemoved) networkRemoved = await cleanupStep('remove only the owned application default network', async () => {
+    const value = JSON.parse((await docker(['network', 'inspect', `${fixture.project}_default`])).stdout)[0];
+    check(value.Labels['com.docker.compose.project'] === fixture.project && Object.keys(value.Containers ?? {}).length === 0,
+      'Only the empty owned application network may be removed');
+    await docker(['network', 'rm', value.Id]);
+  });
+  let schemaDropped = false;
+  if (schemaCreated && allRemoved) schemaDropped = await cleanupStep('drop only the owned E24 schema', async () => {
+    drop(schema);
+    check(sql(`SELECT count(*) FROM pg_namespace WHERE nspname='${schema}';`) === '0', 'Owned schema removal is observed');
+  });
+  let postgresPreserved = false;
+  if (postgresBefore) postgresPreserved = await cleanupStep('verify the preserved PostgreSQL authority', async () => {
+    const value = await inspect(postgresBefore.id);
+    check(value.State.Running && same(postgresIdentity(value), postgresBefore), 'The same PostgreSQL container, image and volumes remain');
+    check(same(publicSnapshot(), publicBefore), 'Public authority remains unchanged after owned cleanup');
+  });
+  evidence.cleanup = { browserClosed: !browser || browserClosed, applicationContainersRemoved: allRemoved,
+    applicationNetworkRemoved: networkRemoved, ownedSchemaDropped: !schemaCreated || schemaDropped,
+    postgresPreserved: !postgresBefore || postgresPreserved, postgresStopAttempted, postgresRestoreAttempted,
+    postgresRestored, forcedContainerKill: false, sharedOrPublicDataRemoved: false };
+  if (cleanupFailures.length > 0) evidence.result = 'FAIL';
+  evidence.elapsedSeconds = (Date.now() - started) / 1000;
+  evidence.hostedCiExecutionClaimed = false;
+  persist();
+  console.log(JSON.stringify({ actor: ['author', 'root', 'ci'].includes(actor) ? actor : 'invalid',
+    result: evidence.result, phase: evidence.failure?.phase ?? 'completed', evidence: resultPath ?? null,
+    elapsedSeconds: evidence.elapsedSeconds, cleanupFailures: cleanupFailures.length }));
+  process.exitCode = evidence.result === 'PASS' ? 0 : 1;
+}


