# E12 URL 정규화, SSRF와 외부 자원 제한

## `test: freeze E12 outbound safety and resource cases`

diff --git a/evidence/phase-1/E12/baseline.mjs b/evidence/phase-1/E12/baseline.mjs
new file mode 100644
index 0000000..57ef1f2
--- /dev/null
+++ b/evidence/phase-1/E12/baseline.mjs
@@ -0,0 +1,54 @@
+import assert from 'node:assert/strict';
+import { createHash } from 'node:crypto';
+import { execFileSync } from 'node:child_process';
+import { mkdir, readFile, writeFile } from 'node:fs/promises';
+import { buildApp } from '../../../server/app.ts';
+import { databaseConfig, databasePool, schemaIdentifier } from '../../../server/database.ts';
+import { migrate } from '../../../server/migrate.ts';
+import { authenticatedInject, loginForTest } from '../../../test/auth.ts';
+import { insertOutboundUser, requireFreePorts } from './fixture.ts';
+import scenario from './scenario.json' with { type: 'json' };
+
+assert.equal(process.versions.node, '24.19.0');
+assert.equal(execFileSync('git', ['branch', '--show-current'], { encoding: 'utf8' }).trim(), 'track/fundamentals-fastify');
+assert.equal((await readFile('SPEC_REVISION', 'utf8')).trim(), scenario.specRevision);
+const changed = execFileSync('git', ['diff', '--name-only', scenario.start], { encoding: 'utf8' }).trim().split('\n').filter(Boolean);
+assert.ok(changed.every(path => path.startsWith('evidence/phase-1/E12/')));
+const began = performance.now();
+const report = { start: scenario.start, baselineHead: execFileSync('git', ['rev-parse', 'HEAD'], { encoding: 'utf8' }).trim(),
+  productMatchesStart: true, result: 'NOT_RUN', hashes: {}, budget: scenario.budget };
+for (const file of ['scenario.json', 'fixture.ts', 'baseline.mjs']) {
+  report.hashes[file] = createHash('sha256').update(await readFile(new URL(file, import.meta.url))).digest('hex');
+}
+const config = { ...databaseConfig(), schema: scenario.schema };
+const pool = databasePool(config);
+const app = buildApp(undefined, config);
+let owned = false;
+try {
+  await requireFreePorts();
+  await pool.query(`CREATE SCHEMA ${schemaIdentifier(config.schema)}`); owned = true;
+  await migrate(config);
+  const inject = authenticatedInject(app, await loginForTest(app, await insertOutboundUser(pool)));
+  const response = await inject({ method: scenario.baseline.method, url: scenario.baseline.path, payload: scenario.monitor });
+  const rows = (await pool.query('SELECT count(*)::int AS count FROM monitors')).rows[0].count;
+  report.observation = { method: scenario.baseline.method, path: scenario.baseline.path, monitor: scenario.monitor,
+    status: response.statusCode, body: response.json(), monitorRows: rows, workerProcesses: 0,
+    endpointRequests: 0, transport: 'Fastify injection only; no worker or outbound HTTP client is started.' };
+  assert.equal(response.statusCode, scenario.baseline.startStatus);
+  assert.equal(response.json().error.code, scenario.baseline.startCode);
+  assert.equal(rows, 0);
+  report.result = 'REPRODUCED';
+  report.decisiveFailure = 'The fixture-only allowlist rejects the fixed general HTTP destination before persistence.';
+} catch (error) {
+  report.result = 'FAILED'; report.failure = error.message; process.exitCode = 1;
+} finally {
+  await app.close();
+  if (owned) await pool.query(`DROP SCHEMA ${schemaIdentifier(config.schema)} CASCADE`);
+  await pool.end();
+  await requireFreePorts();
+  report.cleanup = { schemaDropped: owned, portsFree: true };
+  report.durationMs = Math.round(performance.now() - began);
+  await mkdir('output/phase-1/e12', { recursive: true });
+  await writeFile('output/phase-1/e12/baseline.json', JSON.stringify(report, null, 2) + '\n');
+  console.log(JSON.stringify(report, null, 2));
+}
diff --git a/evidence/phase-1/E12/fixture.ts b/evidence/phase-1/E12/fixture.ts
new file mode 100644
index 0000000..44641a2
--- /dev/null
+++ b/evidence/phase-1/E12/fixture.ts
@@ -0,0 +1,45 @@
+import { randomBytes } from 'node:crypto';
+import { createServer } from 'node:http';
+import type { Socket } from 'node:net';
+import type { Pool } from 'pg';
+import { hashPassword } from '../../../server/password.ts';
+import scenario from './scenario.json' with { type: 'json' };
+
+export { requireFreePorts } from '../E10/fixture.ts';
+
+export async function insertOutboundUser(pool: Pool) {
+  const credentials = { username: scenario.user.username, password: randomBytes(32).toString('base64url') };
+  await pool.query('INSERT INTO users (id, username, password_hash) VALUES ($1, $2, $3)',
+    [scenario.user.id, credentials.username, await hashPassword(credentials.password)]);
+  return credentials;
+}
+
+export function outboundFixture() {
+  const calls: string[] = [];
+  const sockets = new Set<Socket>();
+  let guardRequests = 0;
+  const server = createServer((request, response) => {
+    const path = request.url ?? '/';
+    calls.push(path);
+    const send = () => {
+      if (path === '/private') response.writeHead(302, { location: `${scenario.guardOrigin}/blocked` }).end();
+      else if (path.startsWith('/redirect/') || path.startsWith('/timed/')) {
+        const parts = path.split('/');
+        const hop = Number(parts[2]);
+        if (hop < scenario.resources.redirectChainHops) response.writeHead(302, { location: `/${parts[1]}/${hop + 1}` }).end();
+        else response.writeHead(200).end();
+      } else if (path === '/large') response.writeHead(200).end(Buffer.alloc(scenario.resources.largeFixtureBytes, 120));
+      else response.writeHead(200).end();
+    };
+    const delayMs = path === '/slow' ? scenario.resources.slowFixtureMs : path.startsWith('/timed/') ? scenario.timedRedirectDelayMs : 0;
+    if (delayMs) {
+      const timer = setTimeout(send, delayMs);
+      response.once('close', () => clearTimeout(timer));
+    } else send();
+  });
+  const guard = createServer((_request, response) => { guardRequests++; response.writeHead(500).end(); });
+  for (const listener of [server, guard]) listener.on('connection', socket => {
+    sockets.add(socket); socket.once('close', () => sockets.delete(socket));
+  });
+  return { server, guard, calls, sockets, guardRequests: () => guardRequests };
+}
diff --git a/evidence/phase-1/E12/scenario.json b/evidence/phase-1/E12/scenario.json
new file mode 100644
index 0000000..1bf22c2
--- /dev/null
+++ b/evidence/phase-1/E12/scenario.json
@@ -0,0 +1,49 @@
+{
+  "thread": "E12",
+  "profile": "phase-1",
+  "specRevision": "2ada57a71cd34fa2fae9809415c362a8bbfcdf02",
+  "start": "2e86db44f1f4f5dd605da18579647418deeefb01",
+  "resultSemanticsSha256": "3846e8660512ed0a13bceb783c9e6553fcfa614ab61a1012ac9c1b99a811baf9",
+  "ports": [4311, 4312, 4313, 4314],
+  "schema": "e12_baseline",
+  "user": { "id": "12000000-0000-4000-8000-000000000001", "username": "alice-e12" },
+  "monitor": { "name": "Public destination fixture", "url": "http://public.e12.test/ok", "interval": 60, "enabled": true },
+  "baseline": { "method": "POST", "path": "/monitors", "invocations": 1, "startStatus": 400, "startCode": "INVALID_INPUT", "requiredStatus": 201, "workerProcesses": 0, "endpointRequests": 0 },
+  "publicIpv4": "93.184.216.34",
+  "publicIpv6": "2606:4700:4700::1111",
+  "destinationCases": [
+    { "url": "http://public.e12.test/ok", "addresses": ["93.184.216.34"], "allowed": true },
+    { "url": "https://public.e12.test/ok", "addresses": ["2606:4700:4700::1111"], "allowed": true },
+    { "url": "http://user:fixture@public.e12.test/ok", "allowed": false },
+    { "url": "file:///fixture", "allowed": false },
+    { "url": "http://localhost/ok", "allowed": false },
+    { "url": "http://127.0.0.1/ok", "allowed": false },
+    { "url": "http://[::1]/ok", "allowed": false },
+    { "url": "http://10.0.0.1/ok", "allowed": false },
+    { "url": "http://[fc00::1]/ok", "allowed": false },
+    { "url": "http://169.254.169.254/ok", "allowed": false, "neverContact": true },
+    { "url": "http://[fe80::1]/ok", "allowed": false, "neverContact": true },
+    { "url": "http://[::ffff:127.0.0.1]/ok", "allowed": false },
+    { "url": "http://private.e12.test/ok", "addresses": ["10.0.0.1"], "allowed": false },
+    { "url": "http://mixed.e12.test/ok", "addresses": ["93.184.216.34", "10.0.0.1"], "allowed": false }
+  ],
+  "dnsRebinding": { "url": "http://public.e12.test/ok", "first": ["93.184.216.34"], "second": ["127.0.0.1"], "expectedResolverCalls": 1, "expectedConnectedAddress": "93.184.216.34", "unsafeConnectorCalls": 0 },
+  "redirectPrivate": { "from": "http://public.e12.test/ok", "location": "http://127.0.0.1:4314/blocked", "expectedInitialConnectorCalls": 1, "unsafeConnectorCalls": 0, "httpStatus": null },
+  "resources": { "connectTimeoutMs": 500, "readTimeoutMs": 500, "totalTimeoutMs": 1500, "redirectsMax": 3, "bodyBytesMax": 65536, "slowFixtureMs": 2000, "largeFixtureBytes": 65537, "redirectChainHops": 4 },
+  "fixtureOrigin": "http://127.0.0.1:4311",
+  "guardOrigin": "http://127.0.0.1:4314",
+  "bodyPolicy": "Do not consume final response body; close after validated final headers. Application body consumption is zero, within 65536 bytes; do not claim zero TCP buffering.",
+  "timedRedirectDelayMs": 400,
+  "completionCeilingMs": 1750,
+  "watchdogMs": 5000,
+  "resourceSequence": [
+    { "path": "/slow", "state": "FAILED", "httpStatus": null, "reason": "TIMEOUT", "requests": 1 },
+    { "path": "/large", "state": "SUCCEEDED", "httpStatus": 200, "reason": null, "requests": 1 },
+    { "path": "/redirect/0", "state": "ABORTED", "httpStatus": null, "reason": "REDIRECT_LIMIT", "requests": 4 },
+    { "path": "/private", "state": "ABORTED", "httpStatus": null, "reason": "UNSAFE_DESTINATION", "requests": 1, "guardRequests": 0 },
+    { "path": "/timed/0", "state": "FAILED", "httpStatus": null, "reason": "TIMEOUT", "requests": 4 }
+  ],
+  "requestConcurrency": 1,
+  "safety": "All public/private address decisions use deterministic stubs; only explicitly trusted local fixture I/O is real. No metadata/public service is contacted. Production has no fixture exception by default.",
+  "budget": { "loadRuns": 0, "parameterSweeps": 0, "automaticRetries": 0 }
+}


