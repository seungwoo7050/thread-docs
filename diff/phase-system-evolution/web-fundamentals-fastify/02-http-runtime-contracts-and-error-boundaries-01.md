# E02 HTTP 런타임 계약과 오류 경계

## `Define explicit Monitor HTTP runtime contracts`

diff --git a/evidence/E02/baseline.json b/evidence/E02/baseline.json
new file mode 100644
index 0000000..c2c32e7
--- /dev/null
+++ b/evidence/E02/baseline.json
@@ -0,0 +1,25 @@
+{
+  "start": "aee74fd6cedf8daed9dd4db0a070048aaf515d80",
+  "productionFilesChangedBeforeReproduction": false,
+  "command": "fnm exec --using 24.19.0 node --test --test-name-pattern='E02 fixed counterexample: blank name' test/contracts.test.ts",
+  "measurement": "/usr/bin/time -p",
+  "runs": [
+    {
+      "invocation": 1, "exitCode": 130, "elapsedSeconds": 35.85,
+      "assertionsExecuted": 0,
+      "observed": "Sandboxed local-listener invocation made no progress; manually interrupted. Diagnostic ps was also denied by the sandbox."
+    },
+    {
+      "invocation": 2, "exitCode": 1, "elapsedSeconds": 0.69,
+      "execution": "approved sandbox escalation for fixed loopback fixture/API listeners",
+      "assertionsExecuted": 1,
+      "expected": { "status": 400, "code": "INVALID_INPUT" },
+      "observed": { "status": 201, "body": { "id": "a17bfaba-e8a1-4ee9-b5a7-aebdd50584dc", "name": "   ", "url": "http://127.0.0.1:4311/ok", "interval": 60, "enabled": true, "createdAt": "2026-08-27T23:26:27.239Z", "updatedAt": "2026-08-27T23:26:27.239Z", "latestCheck": null } },
+      "outcome": "201 !== 400; stopped reproduction after this single clear counterexample."
+    }
+  ],
+  "extraBoundaryNote": "Malformed path /monitors/%ZZ/checks was recorded before its first execution after local Fastify docs showed a separate framework error boundary. No observed test result changed this or any original fixture.",
+  "automaticRetries": 0,
+  "loadRuns": 0,
+  "parameterChangesAfterObservation": 0
+}
diff --git a/evidence/E02/scenario.json b/evidence/E02/scenario.json
new file mode 100644
index 0000000..c891167
--- /dev/null
+++ b/evidence/E02/scenario.json
@@ -0,0 +1,70 @@
+{
+  "thread": "E02",
+  "attempt": 1,
+  "specRevision": "0a006589477f8ae47bad3faa5510c999cff85ee4",
+  "start": "aee74fd6cedf8daed9dd4db0a070048aaf515d80",
+  "packetFixturesFrozenBeforeBaseline": true,
+  "fixtureOrigin": "http://127.0.0.1:4311",
+  "apiOrigin": "http://127.0.0.1:4312",
+  "browserOrigin": "http://127.0.0.1:4313",
+  "monitor": { "name": "Fixture monitor", "url": "http://127.0.0.1:4311/ok", "interval": 60, "enabled": true },
+  "fixedCounterexample": { "name": "   ", "expectedStatus": 400, "expectedCode": "INVALID_INPUT" },
+  "requiredInvalidOverrides": [
+    { "interval": "60" }, { "interval": 0 }, { "interval": 86401 }, { "url": "not a URL" }
+  ],
+  "unknownMonitorId": "00000000-0000-4000-8000-000000000000",
+  "additionalBoundaries": {
+    "invalidOverrides": [
+      { "name": "" }, { "name": 42 },
+      { "interval": 1.5 }, { "interval": null }, { "interval": true },
+      { "enabled": "true" }, { "enabled": null },
+      { "url": "/ok" }, { "url": "ftp://127.0.0.1:4311/ok" }, { "url": null }
+    ],
+    "invalidNameRepetitions": [{ "unit": "a", "count": 101 }, { "unit": "😀", "count": 51 }],
+    "missingFields": ["name", "url", "interval", "enabled"],
+    "invalidBodies": [null, [], "monitor"],
+    "validOverrides": [
+      { "name": "  Fixture monitor  " }, { "name": "x" },
+      { "interval": 1 }, { "interval": 86400 }, { "enabled": false }
+    ],
+    "validNameRepetitions": [{ "unit": "a", "count": 100 }, { "unit": "😀", "count": 50 }],
+    "malformedMonitorId": "not-a-uuid",
+    "malformedRequestPath": "/monitors/%ZZ/checks",
+    "missingRoute": "/missing",
+    "malformedJson": "{\"name\":",
+    "emptyJson": "",
+    "unsupportedContentType": "application/octet-stream",
+    "unsupportedBody": "{}",
+    "oversizedBody": { "unit": "x", "count": 8193 },
+    "internalErrorRoute": "/test/internal-error",
+    "privateInternalMessage": "Private test detail"
+  },
+  "terminalResults": [
+    { "path": "/ok", "apiStatus": 200, "state": "SUCCEEDED", "httpStatus": 200, "failureReason": null },
+    { "path": "/fail", "apiStatus": 200, "state": "FAILED", "httpStatus": 503, "failureReason": "HTTP_STATUS" },
+    { "path": "/disconnect", "apiStatus": 200, "state": "FAILED", "httpStatus": null, "failureReason": "CONNECTION_FAILURE" },
+    { "path": "/timeout", "apiStatus": 200, "state": "FAILED", "httpStatus": null, "failureReason": "TIMEOUT" }
+  ],
+  "browserErrors": [
+    { "status": 400, "code": "INVALID_INPUT", "message": "Arbitrary server prose A" },
+    { "status": 400, "code": "INVALID_INPUT", "message": "Different server prose B" },
+    { "status": 404, "code": "NOT_FOUND", "message": "Private test detail" },
+    { "status": 500, "code": "INTERNAL_ERROR", "message": "Private test detail" }
+  ],
+  "malformedBrowserResponses": [
+    { "status": 502, "body": "<html>Unavailable</html>" },
+    { "status": 400, "body": { "error": { "code": "UNKNOWN", "message": "Private test detail" } } },
+    { "status": 500, "body": { "error": { "code": "INVALID_INPUT", "message": "Private test detail" } } },
+    { "status": 200, "body": {} }
+  ],
+  "browserMonitorId": "00000000-0000-4000-8000-000000000001",
+  "browserMonitorTimestamp": "2026-01-01T00:00:00.000Z",
+  "browserNetworkFailure": "failed",
+  "checkDeadlineMs": 2000,
+  "bodyLimitBytes": 8192,
+  "retries": 0,
+  "loadRuns": 0,
+  "benchmarkRuns": 0,
+  "parameterSweeps": 0,
+  "baselineStop": "Run only the whitespace-name assertion; stop after the first clear failure."
+}
diff --git a/package.json b/package.json
index 6066d8b..e2a9214 100644
--- a/package.json
+++ b/package.json
@@ -14,7 +14,7 @@
     "fixture": "node test/fixture.ts",
     "typecheck": "NEXT_TELEMETRY_DISABLED=1 next typegen && tsc --noEmit",
     "test:unit": "node --test test/unit.test.ts",
-    "test:functional": "node --test test/functional.test.ts",
+    "test:functional": "node --test --test-concurrency=1 test/functional.test.ts test/contracts.test.ts",
     "test:e2e": "playwright test",
     "test": "npm run test:unit && npm run test:functional"
   },
diff --git a/server/app.ts b/server/app.ts
index d38efc4..b802586 100644
--- a/server/app.ts
+++ b/server/app.ts
@@ -1,46 +1,58 @@
 import { randomUUID } from 'node:crypto';
-import Fastify from 'fastify';
-import { checkMonitor, DEFAULT_FIXTURE_ORIGIN, fixtureUrl } from './check.ts';
-import type { CheckRun, Monitor } from './model.ts';
+import Fastify, { errorCodes } from 'fastify';
+import type { FastifyReply, FastifyRequest } from 'fastify';
+import { checkMonitor, DEFAULT_FIXTURE_ORIGIN } from './check.ts';
+import { ApiError, ERROR_STATUS, monitorId, monitorInput } from './contracts.ts';
+import type { ApiFailure, CheckRun, Monitor } from './model.ts';
+
+const inputErrors = [
+  errorCodes.FST_ERR_CTP_INVALID_JSON_BODY,
+  errorCodes.FST_ERR_CTP_EMPTY_JSON_BODY,
+  errorCodes.FST_ERR_CTP_INVALID_MEDIA_TYPE,
+  errorCodes.FST_ERR_CTP_INVALID_CONTENT_LENGTH,
+  errorCodes.FST_ERR_CTP_BODY_TOO_LARGE,
+  errorCodes.FST_ERR_BAD_URL,
+];
 
 export function buildApp(fixtureOrigin = DEFAULT_FIXTURE_ORIGIN) {
-  const app = Fastify({ logger: false, bodyLimit: 8_192 });
+  const handleError = (error: unknown, _request: FastifyRequest, reply: FastifyReply) => {
+    const failure = error instanceof ApiError ? error
+      : inputErrors.some((ErrorType) => error instanceof ErrorType)
+        ? new ApiError('INVALID_INPUT', 'Request URL and JSON body must be valid; the body limit is 8192 bytes.')
+        : new ApiError('INTERNAL_ERROR', 'The monitoring service could not complete the request.');
+    const body: ApiFailure = { error: { code: failure.code, message: failure.message } };
+    return reply.code(ERROR_STATUS[failure.code]).type('application/json').send(body);
+  };
+  const app = Fastify({ logger: false, bodyLimit: 8_192, frameworkErrors: handleError });
   const monitors = new Map<string, Monitor>();
   const latestChecks = new Map<string, CheckRun>();
 
-  app.get('/health', async () => ({ status: 'ok' }));
-  app.get('/monitors', async () => Array.from(monitors.values(), (monitor) => ({
+  app.setErrorHandler(handleError);
+  app.setNotFoundHandler(async () => { throw new ApiError('NOT_FOUND', 'Resource not found.'); });
+
+  app.get('/health', async () => ({ data: { status: 'ok' } }));
+  app.get('/monitors', async () => ({ data: Array.from(monitors.values(), (monitor) => ({
     ...monitor, latestCheck: latestChecks.get(monitor.id) ?? null,
-  })));
+  })) }));
 
-  app.post<{ Body: { name: string; url: string; interval: number; enabled: boolean } }>(
+  app.post<{ Body: unknown }>(
     '/monitors', async (request, reply) => {
-      const body = request.body;
-      if (!body || typeof body.name !== 'string' || typeof body.url !== 'string') {
-        return reply.code(400).send({ message: 'Name and URL are required.' });
-      }
-      let url: URL;
-      try {
-        url = fixtureUrl(body.url, fixtureOrigin);
-      } catch {
-        return reply.code(400).send({ message: 'Only the configured fixture origin is allowed.' });
-      }
+      const input = monitorInput(request.body, fixtureOrigin);
       const now = new Date().toISOString();
       const monitor: Monitor = {
-        id: randomUUID(), name: body.name, url: url.href,
-        interval: body.interval, enabled: body.enabled, createdAt: now, updatedAt: now,
+        id: randomUUID(), ...input, createdAt: now, updatedAt: now,
       };
       monitors.set(monitor.id, monitor);
-      return reply.code(201).send({ ...monitor, latestCheck: null });
+      return reply.code(201).send({ data: { ...monitor, latestCheck: null } });
     },
   );
 
-  app.post<{ Params: { id: string } }>('/monitors/:id/checks', async (request, reply) => {
-    const monitor = monitors.get(request.params.id);
-    if (!monitor) return reply.code(404).send({ message: 'Monitor not found.' });
+  app.post<{ Params: { id: string } }>('/monitors/:id/checks', async (request) => {
+    const monitor = monitors.get(monitorId(request.params.id));
+    if (!monitor) throw new ApiError('NOT_FOUND', 'Monitor not found.');
     const result = await checkMonitor(monitor, fixtureOrigin);
     latestChecks.set(monitor.id, result);
-    return result;
+    return { data: result };
   });
 
   return app;
diff --git a/server/check.ts b/server/check.ts
index 6282fdb..e027c07 100644
--- a/server/check.ts
+++ b/server/check.ts
@@ -51,7 +51,7 @@ export async function checkMonitor(monitor: Monitor, fixtureOrigin: string): Pro
       finish(null, 'TIMEOUT');
       request.destroy();
     }, CHECK_TIMEOUT_MS);
-    request.on('error', () => finish(null, 'CONNECTION_FAILED'));
+    request.on('error', () => finish(null, 'CONNECTION_FAILURE'));
     request.end();
   });
 }
diff --git a/server/contracts.ts b/server/contracts.ts
new file mode 100644
index 0000000..ef23fc0
--- /dev/null
+++ b/server/contracts.ts
@@ -0,0 +1,44 @@
+import { fixtureUrl } from './check.ts';
+import type { ApiErrorCode, Monitor } from './model.ts';
+
+export const ERROR_STATUS = { INVALID_INPUT: 400, NOT_FOUND: 404, INTERNAL_ERROR: 500 } as const;
+
+export class ApiError extends Error {
+  readonly code: ApiErrorCode;
+
+  constructor(code: ApiErrorCode, message: string) {
+    super(message);
+    this.name = 'ApiError';
+    this.code = code;
+  }
+}
+
+export function monitorInput(value: unknown, fixtureOrigin: string): Pick<Monitor, 'name' | 'url' | 'interval' | 'enabled'> {
+  if (value === null || typeof value !== 'object' || Array.isArray(value)) {
+    throw new ApiError('INVALID_INPUT', 'A Monitor JSON object is required.');
+  }
+  const { name, url, interval, enabled } = value as Record<string, unknown>;
+  if (typeof name !== 'string' || name.trim().length < 1 || name.trim().length > 100) {
+    throw new ApiError('INVALID_INPUT', 'Name must contain 1–100 UTF-16 code units after trimming.');
+  }
+  if (typeof interval !== 'number' || !Number.isInteger(interval) || interval < 1 || interval > 86_400) {
+    throw new ApiError('INVALID_INPUT', 'Interval must be an integer from 1 to 86400 seconds.');
+  }
+  if (typeof enabled !== 'boolean') {
+    throw new ApiError('INVALID_INPUT', 'Enabled must be a boolean.');
+  }
+  if (typeof url !== 'string') {
+    throw new ApiError('INVALID_INPUT', 'An absolute HTTP(S) fixture URL is required.');
+  }
+  let parsedUrl: URL;
+  try { parsedUrl = fixtureUrl(url, fixtureOrigin); }
+  catch { throw new ApiError('INVALID_INPUT', 'An absolute HTTP(S) URL on the configured fixture origin is required.'); }
+  return { name: name.trim(), url: parsedUrl.href, interval, enabled };
+}
+
+export function monitorId(value: string): string {
+  if (!/^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i.test(value)) {
+    throw new ApiError('INVALID_INPUT', 'Monitor ID must be a UUID.');
+  }
+  return value.toLowerCase();
+}
diff --git a/server/model.ts b/server/model.ts
index bfc24ab..03f1253 100644
--- a/server/model.ts
+++ b/server/model.ts
@@ -15,9 +15,13 @@ export type CheckRun = {
   state: 'SUCCEEDED' | 'FAILED';
   httpStatus: number | null;
   latencyMs: number;
-  failureReason: 'HTTP_STATUS' | 'TIMEOUT' | 'CONNECTION_FAILED' | null;
+  failureReason: 'HTTP_STATUS' | 'TIMEOUT' | 'CONNECTION_FAILURE' | null;
   startedAt: string;
   finishedAt: string;
 };
 
 export type MonitorView = Monitor & { latestCheck: CheckRun | null };
+
+export type ApiErrorCode = 'INVALID_INPUT' | 'NOT_FOUND' | 'INTERNAL_ERROR';
+export type ApiSuccess<T> = { data: T };
+export type ApiFailure = { error: { code: ApiErrorCode; message: string } };
diff --git a/test/contracts.test.ts b/test/contracts.test.ts
new file mode 100644
index 0000000..eb7ac5a
--- /dev/null
+++ b/test/contracts.test.ts
@@ -0,0 +1,162 @@
+import { after, before, test } from 'node:test';
+import assert from 'node:assert/strict';
+import { buildApp } from '../server/app.ts';
+import { fixtureServer } from './fixture.ts';
+import type { ApiErrorCode, ApiSuccess, CheckRun, MonitorView } from '../server/model.ts';
+import scenario from '../evidence/E02/scenario.json' with { type: 'json' };
+
+const fixture = fixtureServer();
+const app = buildApp(scenario.fixtureOrigin);
+const boundaries = scenario.additionalBoundaries;
+app.get(boundaries.internalErrorRoute, async () => { throw new Error(boundaries.privateInternalMessage); });
+
+before(async () => {
+  await new Promise<void>((resolve) => fixture.server.listen(4311, '127.0.0.1', resolve));
+  await app.listen({ host: '127.0.0.1', port: 4312 });
+});
+
+after(async () => {
+  await app.close();
+  fixture.server.closeAllConnections();
+  await new Promise<void>((resolve) => fixture.server.close(() => resolve()));
+});
+
+test('E02 fixed counterexample: blank name is INVALID_INPUT', async () => {
+  const response = await fetch(`${scenario.apiOrigin}/monitors`, {
+    method: 'POST', headers: { 'content-type': 'application/json' },
+    body: JSON.stringify({ ...scenario.monitor, name: scenario.fixedCounterexample.name }),
+  });
+  const body = await response.json();
+  assert.equal(response.status, scenario.fixedCounterexample.expectedStatus, JSON.stringify(body));
+  assert.deepEqual(Object.keys(body), ['error']);
+  assert.equal(body.error.code, scenario.fixedCounterexample.expectedCode);
+  assert.equal(typeof body.error.message, 'string');
+});
+
+function postMonitor(body: unknown) {
+  return fetch(`${scenario.apiOrigin}/monitors`, {
+    method: 'POST', headers: { 'content-type': 'application/json' }, body: JSON.stringify(body),
+  });
+}
+
+async function assertFailure(response: Response, status: number, code: ApiErrorCode) {
+  const body = await response.json();
+  assert.equal(response.status, status, JSON.stringify(body));
+  assert.match(response.headers.get('content-type') ?? '', /^application\/json/);
+  assert.deepEqual(Object.keys(body), ['error']);
+  assert.deepEqual(Object.keys(body.error).sort(), ['code', 'message']);
+  assert.equal(body.error.code, code);
+  assert.equal(typeof body.error.message, 'string');
+  assert.ok(body.error.message.length > 0);
+  assert.ok(!JSON.stringify(body).includes(boundaries.privateInternalMessage));
+}
+
+async function success<T>(response: Response, status: number): Promise<T> {
+  const body: ApiSuccess<T> = await response.json();
+  assert.equal(response.status, status, JSON.stringify(body));
+  assert.match(response.headers.get('content-type') ?? '', /^application\/json/);
+  assert.deepEqual(Object.keys(body), ['data']);
+  return body.data;
+}
+
+function assertTimestamp(value: string) {
+  assert.equal(new Date(value).toISOString(), value);
+}
+
+function assertMonitor(monitor: MonitorView) {
+  assert.deepEqual(Object.keys(monitor).sort(), ['createdAt', 'enabled', 'id', 'interval', 'latestCheck', 'name', 'updatedAt', 'url']);
+  assert.match(monitor.id, /^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/);
+  assert.equal(typeof monitor.name, 'string');
+  assert.equal(typeof monitor.url, 'string');
+  assert.equal(typeof monitor.interval, 'number');
+  assert.equal(typeof monitor.enabled, 'boolean');
+  assertTimestamp(monitor.createdAt);
+  assertTimestamp(monitor.updatedAt);
+}
+
+test('E02 fixed malformed values are INVALID_INPUT and do not create Monitors', async () => {
+  const before = await success<MonitorView[]>(await fetch(`${scenario.apiOrigin}/monitors`), 200);
+  for (const override of scenario.requiredInvalidOverrides) {
+    await assertFailure(await postMonitor({ ...scenario.monitor, ...override }), 400, 'INVALID_INPUT');
+  }
+  const after = await success<MonitorView[]>(await fetch(`${scenario.apiOrigin}/monitors`), 200);
+  assert.deepEqual(after, before);
+});
+
+test('E02 validates required field types, trimmed name length and parsed numeric integers', async () => {
+  const before = await success<MonitorView[]>(await fetch(`${scenario.apiOrigin}/monitors`), 200);
+  const invalid = [
+    ...boundaries.invalidOverrides.map((override) => ({ ...scenario.monitor, ...override })),
+    ...boundaries.invalidNameRepetitions.map(({ unit, count }) => ({ ...scenario.monitor, name: unit.repeat(count) })),
+    ...boundaries.missingFields.map((field) => Object.fromEntries(Object.entries(scenario.monitor).filter(([key]) => key !== field))),
+    ...boundaries.invalidBodies,
+  ];
+  for (const body of invalid) await assertFailure(await postMonitor(body), 400, 'INVALID_INPUT');
+  const after = await success<MonitorView[]>(await fetch(`${scenario.apiOrigin}/monitors`), 200);
+  assert.deepEqual(after, before);
+});
+
+test('E02 accepts and serializes trimmed name and interval boundary values without coercion', async () => {
+  const valid = [
+    ...boundaries.validOverrides.map((override) => ({ ...scenario.monitor, ...override })),
+    ...boundaries.validNameRepetitions.map(({ unit, count }) => ({ ...scenario.monitor, name: unit.repeat(count) })),
+  ];
+  for (const input of valid) {
+    const monitor = await success<MonitorView>(await postMonitor(input), 201);
+    assertMonitor(monitor);
+    assert.equal(monitor.name, input.name.trim());
+    assert.equal(monitor.url, input.url);
+    assert.equal(monitor.interval, input.interval);
+    assert.equal(monitor.enabled, input.enabled);
+    assert.equal(monitor.latestCheck, null);
+    assert.equal(monitor.createdAt, monitor.updatedAt);
+  }
+});
+
+test('E02 missing Monitor and route are NOT_FOUND; malformed ID and path are INVALID_INPUT', async () => {
+  const calls = [...fixture.calls];
+  await assertFailure(await fetch(`${scenario.apiOrigin}/monitors/${scenario.unknownMonitorId}/checks`, { method: 'POST' }), 404, 'NOT_FOUND');
+  await assertFailure(await fetch(`${scenario.apiOrigin}${boundaries.missingRoute}`), 404, 'NOT_FOUND');
+  await assertFailure(await fetch(`${scenario.apiOrigin}/monitors/${boundaries.malformedMonitorId}/checks`, { method: 'POST' }), 400, 'INVALID_INPUT');
+  await assertFailure(await fetch(`${scenario.apiOrigin}${boundaries.malformedRequestPath}`, { method: 'POST' }), 400, 'INVALID_INPUT');
+  assert.deepEqual([...fixture.calls], calls);
+});
+
+test('E02 parser, media type and body size failures use the same INVALID_INPUT envelope', async () => {
+  for (const body of [boundaries.malformedJson, boundaries.emptyJson, boundaries.oversizedBody.unit.repeat(boundaries.oversizedBody.count)]) {
+    await assertFailure(await fetch(`${scenario.apiOrigin}/monitors`, {
+      method: 'POST', headers: { 'content-type': 'application/json' }, body,
+    }), 400, 'INVALID_INPUT');
+  }
+  await assertFailure(await fetch(`${scenario.apiOrigin}/monitors`, {
+    method: 'POST', headers: { 'content-type': boundaries.unsupportedContentType }, body: boundaries.unsupportedBody,
+  }), 400, 'INVALID_INPUT');
+  await assertFailure(await fetch(`${scenario.apiOrigin}/monitors`, { method: 'POST' }), 400, 'INVALID_INPUT');
+});
+
+test('E02 unexpected API failures are INTERNAL_ERROR without private details', async () => {
+  await assertFailure(await fetch(`${scenario.apiOrigin}${boundaries.internalErrorRoute}`), 500, 'INTERNAL_ERROR');
+  assert.deepEqual(await success(await fetch(`${scenario.apiOrigin}/health`), 200), { status: 'ok' });
+});
+
+test('E02 terminal CheckRun wire values distinguish observed HTTP failure from no response', async () => {
+  for (const expected of scenario.terminalResults) {
+    const monitor = await success<MonitorView>(await postMonitor({ ...scenario.monitor, url: `${scenario.fixtureOrigin}${expected.path}` }), 201);
+    assertMonitor(monitor);
+    const result = await success<CheckRun>(await fetch(`${scenario.apiOrigin}/monitors/${monitor.id}/checks`, { method: 'POST' }), expected.apiStatus);
+    assert.deepEqual(Object.keys(result).sort(), ['failureReason', 'finishedAt', 'httpStatus', 'id', 'latencyMs', 'monitorId', 'startedAt', 'state', 'trigger']);
+    assert.match(result.id, /^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/);
+    assert.equal(result.monitorId, monitor.id);
+    assert.equal(result.trigger, 'MANUAL');
+    assert.equal(result.state, expected.state);
+    assert.equal(result.httpStatus, expected.httpStatus);
+    assert.equal(result.failureReason, expected.failureReason);
+    assert.ok(Number.isInteger(result.latencyMs) && result.latencyMs >= 0);
+    assertTimestamp(result.startedAt);
+    assertTimestamp(result.finishedAt);
+    assert.ok(Date.parse(result.finishedAt) >= Date.parse(result.startedAt));
+    assert.equal(fixture.calls.get(expected.path), 1);
+    const list = await success<MonitorView[]>(await fetch(`${scenario.apiOrigin}/monitors`), 200);
+    assert.deepEqual(list.find((item) => item.id === monitor.id)?.latestCheck, result);
+  }
+});
diff --git a/test/fixture.ts b/test/fixture.ts
index 5a886c8..6450545 100644
--- a/test/fixture.ts
+++ b/test/fixture.ts
@@ -13,6 +13,8 @@ export function fixtureServer() {
       response.writeHead(302, { location: '/ok' });
       return void response.end('redirect\n');
     }
+    if (path === '/disconnect') return void request.socket.destroy();
+    if (path === '/timeout') return;
     response.statusCode = 404;
     response.end('not found\n');
   });
diff --git a/test/functional.test.ts b/test/functional.test.ts
index 23224cf..fa93544 100644
--- a/test/functional.test.ts
+++ b/test/functional.test.ts
@@ -3,7 +3,7 @@ import assert from 'node:assert/strict';
 import { createServer } from 'node:http';
 import { buildApp } from '../server/app.ts';
 import { checkMonitor, DEFAULT_FIXTURE_ORIGIN } from '../server/check.ts';
-import type { Monitor, MonitorView } from '../server/model.ts';
+import type { ApiSuccess, Monitor, MonitorView } from '../server/model.ts';
 import { fixtureServer } from './fixture.ts';
 
 const fixture = fixtureServer();
@@ -30,7 +30,7 @@ async function create(path: string): Promise<MonitorView> {
     name: 'Fixture monitor', url: `${DEFAULT_FIXTURE_ORIGIN}${path}`, interval: 60, enabled: true,
   } });
   assert.equal(response.statusCode, 201);
-  return response.json<MonitorView>();
+  return response.json<ApiSuccess<MonitorView>>().data;
 }
 
 test('create a Monitor in memory and synchronously observe GET /ok', async () => {
@@ -40,7 +40,7 @@ test('create a Monitor in memory and synchronously observe GET /ok', async () =>
   assert.equal(monitor.latestCheck, null);
   const response = await app.inject({ method: 'POST', url: `/monitors/${monitor.id}/checks` });
   assert.equal(response.statusCode, 200);
-  const result = response.json();
+  const result = response.json().data;
   assert.equal(result.state, 'SUCCEEDED');
   assert.equal(result.httpStatus, 200);
   assert.equal(result.trigger, 'MANUAL');
@@ -48,13 +48,13 @@ test('create a Monitor in memory and synchronously observe GET /ok', async () =>
   assert.ok(result.latencyMs >= 0);
   assert.ok(Date.parse(result.finishedAt) >= Date.parse(result.startedAt));
   assert.equal(fixture.calls.get('/ok'), 1);
-  const list = (await app.inject('/monitors')).json<MonitorView[]>();
+  const list = (await app.inject('/monitors')).json<ApiSuccess<MonitorView[]>>().data;
   assert.equal(list.find((item) => item.id === monitor.id)?.latestCheck?.id, result.id);
 });
 
 test('GET /fail is an observed endpoint failure with HTTP 503', async () => {
   const monitor = await create('/fail');
-  const result = (await app.inject({ method: 'POST', url: `/monitors/${monitor.id}/checks` })).json();
+  const result = (await app.inject({ method: 'POST', url: `/monitors/${monitor.id}/checks` })).json().data;
   assert.equal(result.state, 'FAILED');
   assert.equal(result.httpStatus, 503);
   assert.equal(result.failureReason, 'HTTP_STATUS');
@@ -64,7 +64,7 @@ test('GET /fail is an observed endpoint failure with HTTP 503', async () => {
 test('the check does not follow even a same-origin redirect', async () => {
   const previousOkCalls = fixture.calls.get('/ok');
   const monitor = await create('/redirect');
-  const result = (await app.inject({ method: 'POST', url: `/monitors/${monitor.id}/checks` })).json();
+  const result = (await app.inject({ method: 'POST', url: `/monitors/${monitor.id}/checks` })).json().data;
   assert.equal(result.state, 'FAILED');
   assert.equal(result.httpStatus, 302);
   assert.equal(fixture.calls.get('/ok'), previousOkCalls);
@@ -82,6 +82,6 @@ test('a non-fixture URL is rejected without contacting the controlled guard', as
 
 test('another application instance starts with empty memory', async () => {
   const fresh = buildApp();
-  try { assert.deepEqual((await fresh.inject('/monitors')).json(), []); }
+  try { assert.deepEqual((await fresh.inject('/monitors')).json(), { data: [] }); }
   finally { await fresh.close(); }
 });


