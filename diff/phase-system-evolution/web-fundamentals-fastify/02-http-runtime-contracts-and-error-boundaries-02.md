## `Render API failure categories without parsing server prose`

diff --git a/TRACK.md b/TRACK.md
index 1474226..d3da176 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -8,7 +8,7 @@ is `http://127.0.0.1:4311`; API and fixture bind to `127.0.0.1`.
 Checks have a fixed 2 second total deadline, perform no retries, do not follow
 redirects, and close the response after observing headers without retaining a body.
 HTTP 200–299 is `SUCCEEDED`; other observed statuses are `FAILED/HTTP_STATUS`.
-A transport failure records `FAILED/CONNECTION_FAILED` or `FAILED/TIMEOUT` with
+A transport failure records `FAILED/CONNECTION_FAILURE` or `FAILED/TIMEOUT` with
 `httpStatus: null`. Only the latest terminal result per Monitor is retained.
 `enabled` and `interval` are stored fields; there is no scheduled execution.
 
@@ -71,7 +71,7 @@ npm run test:e2e
 npm run build
 ```
 
-Functional tests own fixed loopback ports 4311 and 4314; stop manual fixture
+Functional tests own fixed loopback ports 4311, 4312 and 4314; stop manual fixture
 processes before running them. Port 4314 is a controlled destination that must
 never receive a check. State disappears on API restart. There is no login,
 database, worker, Redis, Kafka, or production container in E01.
@@ -79,6 +79,40 @@ The E2E command owns ports 4311–4313 and starts fresh fixture/API/Next process
 stop manual development processes first. It uses one Chromium worker, no test
 retries, and the fixed Monitor fixture in `evidence/E01/scenario.json`.
 
+## E02 HTTP contract
+
+Creation requires a JSON object containing `name`, `url`, `interval`, and `enabled`.
+The name is trimmed and must then contain 1–100 UTF-16 code units (JavaScript
+`String.length`). Interval is a numeric integer from 1 to 86400 seconds; JSON
+`60` and `60.0` have the same parsed value, but `"60"` is rejected. Enabled must
+be a boolean. URL must be absolute HTTP(S), without credentials, and use the
+configured fixture origin; the outbound check repeats that boundary check.
+Unknown object fields are ignored and are not copied to the stored Monitor.
+Monitor path IDs must have UUID syntax; a valid but absent ID is not found.
+
+All API successes use `{ "data": <payload> }`, including health, Monitor creation
+(201), list reads (200), and synchronous checks (200). All API errors use
+`{ "error": { "code": <category>, "message": <text> } }`:
+
+| Code | HTTP status |
+| --- | --- |
+| `INVALID_INPUT` | 400 |
+| `NOT_FOUND` | 404 |
+| `INTERNAL_ERROR` | 500 |
+
+Malformed JSON, missing bodies, unsupported media types, malformed request paths,
+and bodies above the unchanged 8192-byte limit use `INVALID_INPUT`. Unexpected
+exceptions use a fixed safe message without exception details. The browser maps
+code/status pairs to its own messages; it never classifies server prose. Invalid
+envelopes or transport errors show the generic service failure and do not update
+success state. An endpoint's 503 is still an API 200 containing a terminal
+`FAILED/HTTP_STATUS` CheckRun. Missing HTTP responses retain explicit `null`.
+
+Frozen E02 cases and invocation outcomes are in `evidence/E02/`. Functional tests
+also exercise controlled `/disconnect` and `/timeout` fixture paths, with the
+unchanged 2-second deadline and zero retries. No new dependency is required:
+validation uses standard JavaScript and Fastify's existing typed error classes.
+
 ## Basic CI
 
 `.github/workflows/check.yml` runs the type, unit, functional, Chromium E2E, and
diff --git a/app/monitors/api.ts b/app/monitors/api.ts
new file mode 100644
index 0000000..12d8b79
--- /dev/null
+++ b/app/monitors/api.ts
@@ -0,0 +1,46 @@
+import type { ApiErrorCode } from '../../server/model.ts';
+
+export const ERROR_MESSAGES: Record<ApiErrorCode, string> = {
+  INVALID_INPUT: 'Check the monitor input and try again.',
+  NOT_FOUND: 'The requested monitor or resource was not found.',
+  INTERNAL_ERROR: 'The monitoring service could not complete the request.',
+};
+const ERROR_STATUS = { INVALID_INPUT: 400, NOT_FOUND: 404, INTERNAL_ERROR: 500 } as const;
+
+export class RequestFailure extends Error {
+  readonly code: ApiErrorCode;
+
+  constructor(code: ApiErrorCode) {
+    super(ERROR_MESSAGES[code]);
+    this.name = 'RequestFailure';
+    this.code = code;
+  }
+}
+
+function isObject(value: unknown): value is Record<string, unknown> {
+  return value !== null && typeof value === 'object' && !Array.isArray(value);
+}
+
+export async function responseData<T>(response: Response): Promise<T> {
+  let body: unknown;
+  try { body = await response.json(); }
+  catch { throw new RequestFailure('INTERNAL_ERROR'); }
+
+  if (!response.ok) {
+    if (isObject(body) && isObject(body.error) && typeof body.error.message === 'string') {
+      const code = body.error.code;
+      if ((code === 'INVALID_INPUT' || code === 'NOT_FOUND' || code === 'INTERNAL_ERROR') && ERROR_STATUS[code] === response.status) {
+        throw new RequestFailure(code);
+      }
+    }
+    throw new RequestFailure('INTERNAL_ERROR');
+  }
+  if (!isObject(body) || !Object.hasOwn(body, 'data') || Object.hasOwn(body, 'error')) {
+    throw new RequestFailure('INTERNAL_ERROR');
+  }
+  return body.data as T;
+}
+
+export function failureCode(failure: unknown): ApiErrorCode {
+  return failure instanceof RequestFailure ? failure.code : 'INTERNAL_ERROR';
+}
diff --git a/app/monitors/page.tsx b/app/monitors/page.tsx
index a6eeb96..d62a3fa 100644
--- a/app/monitors/page.tsx
+++ b/app/monitors/page.tsx
@@ -2,19 +2,18 @@
 
 import { useEffect, useState } from 'react';
 import type { FormEvent } from 'react';
-import type { CheckRun, MonitorView } from '../../server/model';
+import type { ApiErrorCode, CheckRun, MonitorView } from '../../server/model';
+import { ERROR_MESSAGES, failureCode, responseData } from './api';
 
 export default function MonitorsPage() {
   const [monitors, setMonitors] = useState<MonitorView[]>([]);
-  const [error, setError] = useState<string | null>(null);
+  const [error, setError] = useState<ApiErrorCode | null>(null);
   const [creating, setCreating] = useState(false);
   const [checking, setChecking] = useState<string | null>(null);
 
   useEffect(() => {
-    fetch('/api/monitors').then(async (response) => {
-      if (!response.ok) throw new Error('Could not load monitors.');
-      setMonitors(await response.json());
-    }).catch((failure: Error) => setError(failure.message));
+    fetch('/api/monitors').then(responseData<MonitorView[]>).then(setMonitors)
+      .catch((failure: unknown) => setError(failureCode(failure)));
   }, []);
 
   async function createMonitor(event: FormEvent<HTMLFormElement>) {
@@ -31,12 +30,11 @@ export default function MonitorsPage() {
           interval: Number(fields.get('interval')), enabled: fields.get('enabled') === 'on',
         }),
       });
-      const body = await response.json();
-      if (!response.ok) throw new Error(body.message ?? 'Could not create monitor.');
-      setMonitors((current) => [...current, body]);
+      const monitor = await responseData<MonitorView>(response);
+      setMonitors((current) => [...current, monitor]);
       form.reset();
     } catch (failure) {
-      setError(failure instanceof Error ? failure.message : 'Could not create monitor.');
+      setError(failureCode(failure));
     } finally { setCreating(false); }
   }
 
@@ -45,12 +43,10 @@ export default function MonitorsPage() {
     setError(null);
     try {
       const response = await fetch(`/api/monitors/${id}/checks`, { method: 'POST' });
-      const body = await response.json();
-      if (!response.ok) throw new Error(body.message ?? 'Could not run check.');
-      const result: CheckRun = body;
+      const result = await responseData<CheckRun>(response);
       setMonitors((current) => current.map((monitor) => monitor.id === id ? { ...monitor, latestCheck: result } : monitor));
     } catch (failure) {
-      setError(failure instanceof Error ? failure.message : 'Could not run check.');
+      setError(failureCode(failure));
     } finally { setChecking(null); }
   }
 
@@ -59,7 +55,7 @@ export default function MonitorsPage() {
       <p>Create an endpoint monitor, run a check, and inspect the response.</p>
     </header>
     <aside>Development only. Checks can access the configured fixture origin. State is lost on API restart.</aside>
-    {error && <p role="alert" className="error">{error}</p>}
+    {error && <p role="alert" className="error" data-error-code={error}>{ERROR_MESSAGES[error]}</p>}
     <section aria-labelledby="create-heading" className="panel">
       <h2 id="create-heading">Create monitor</h2>
       <form onSubmit={createMonitor}>
@@ -68,7 +64,7 @@ export default function MonitorsPage() {
         <label htmlFor="url">Endpoint URL</label>
         <input id="url" name="url" type="url" required defaultValue="http://127.0.0.1:4311/ok" />
         <label htmlFor="interval">Interval (seconds)</label>
-        <input id="interval" name="interval" type="number" required min="1" step="1" defaultValue="60" />
+        <input id="interval" name="interval" type="number" required min="1" max="86400" step="1" defaultValue="60" />
         <label className="checkbox"><input name="enabled" type="checkbox" defaultChecked /> Enabled</label>
         <p className="hint">Interval and enabled are saved; this version only runs manual checks.</p>
         <button type="submit" disabled={creating}>{creating ? 'Creating…' : 'Create monitor'}</button>
diff --git a/evidence/E02/verification.json b/evidence/E02/verification.json
new file mode 100644
index 0000000..0526534
--- /dev/null
+++ b/evidence/E02/verification.json
@@ -0,0 +1,35 @@
+{
+  "thread": "E02",
+  "attempt": 1,
+  "runtime": { "node": "24.19.0", "npm": "11.17.0" },
+  "measurement": "/usr/bin/time -p; each command run with fnm exec --using 24.19.0",
+  "baseline": "baseline.json records both invocations, including the sandbox interruption and the single failing assertion before production changes.",
+  "runs": [
+    { "command": "npm run typecheck", "invocation": 1, "exitCode": 0, "elapsedSeconds": 2.28 },
+    { "command": "npm run test:unit", "invocation": 1, "exitCode": 0, "elapsedSeconds": 0.71, "passed": 7,
+      "observed": "Three E01 fixture URL regressions plus code/status classification, prose independence, malformed envelope fallback, null/false/zero serialization, and transport fallback." },
+    { "command": "npm run test:functional", "invocation": 1, "exitCode": 0, "elapsedSeconds": 3.12, "passed": 13,
+      "execution": "approved sandbox escalation for fixed loopback listeners",
+      "observed": "Eight E02 real-HTTP contract tests and all five E01 regressions passed. The identical whitespace-name assertion now returns 400 INVALID_INPUT; every frozen required invalid override does too. Missing UUID is 404 NOT_FOUND; unexpected exception is 500 INTERNAL_ERROR with no private details. All terminal outcomes, exact field keys, JSON types, timestamps and explicit null values are asserted." },
+    { "command": "npm run test:e2e", "invocation": 1, "exitCode": 0, "elapsedSeconds": 6.27, "passed": 4,
+      "execution": "approved sandbox escalation for fixed loopback servers and Chromium",
+      "browser": "Pinned Playwright Chromium, one worker, retries 0",
+      "observed": "Both 400 INVALID_INPUT prose variants show the same error category without a Monitor success update. Check 404 and list 500 remain failure states. The unmodified E01 browser flow still creates, checks, displays SUCCEEDED/200 and survives reload.",
+      "screenshot": "output/playwright/E01-success.png (generated local regression artifact)" },
+    { "command": "npm run build", "invocation": 1, "exitCode": 0, "elapsedSeconds": 2.77,
+      "observed": "Next production compilation, TypeScript, and static route generation passed." }
+  ],
+  "postFixFailures": 0,
+  "baselineCommandInvocations": 2,
+  "baselineAssertionsExecuted": 1,
+  "postFixCommandInvocations": 5,
+  "automaticRetries": 0,
+  "loadRuns": 0,
+  "benchmarkRuns": 0,
+  "profilerRuns": 0,
+  "parameterSweeps": 0,
+  "parameterChangesAfterObservation": 0,
+  "dependenciesAdded": [],
+  "hostedCiExecuted": false,
+  "stopCondition": "All acceptance checks passed; no further scenario runs or tuning."
+}
diff --git a/test/browser/contracts.spec.ts b/test/browser/contracts.spec.ts
new file mode 100644
index 0000000..fe10e55
--- /dev/null
+++ b/test/browser/contracts.spec.ts
@@ -0,0 +1,59 @@
+import { expect, test } from '@playwright/test';
+import scenario from '../../evidence/E02/scenario.json' with { type: 'json' };
+
+test('INVALID_INPUT displays the same category for both server prose variants and never creates success UI', async ({ page }) => {
+  let failure = scenario.browserErrors[0];
+  await page.route('**/api/monitors', async (route) => {
+    if (route.request().method() === 'GET') return route.fulfill({ json: { data: [] } });
+    return route.fulfill({ status: failure.status, json: { error: { code: failure.code, message: failure.message } } });
+  });
+  for (const inputError of scenario.browserErrors.filter(({ code }) => code === 'INVALID_INPUT')) {
+    failure = inputError;
+    await page.goto('/monitors');
+    await page.getByLabel('Name', { exact: true }).fill(scenario.monitor.name);
+    await page.getByLabel('Endpoint URL').fill(scenario.monitor.url);
+    await page.getByLabel('Interval (seconds)').fill(String(scenario.monitor.interval));
+    await page.getByLabel('Enabled', { exact: true }).check();
+    await page.getByRole('button', { name: 'Create monitor', exact: true }).click();
+    const alert = page.getByRole('main').getByRole('alert');
+    await expect(alert).toHaveText('Check the monitor input and try again.');
+    await expect(alert).toHaveAttribute('data-error-code', 'INVALID_INPUT');
+    await expect(page.getByRole('article')).toHaveCount(0);
+    await expect(page.getByText('No monitors yet.', { exact: true })).toBeVisible();
+    await expect(page.getByLabel('Name', { exact: true })).toHaveValue(scenario.monitor.name);
+    await expect(page.getByRole('button', { name: 'Create monitor', exact: true })).toBeEnabled();
+  }
+});
+
+test('NOT_FOUND from a manual check leaves the Monitor without a successful result', async ({ page }) => {
+  const failure = scenario.browserErrors.find(({ code }) => code === 'NOT_FOUND')!;
+  const monitor = {
+    ...scenario.monitor, id: scenario.browserMonitorId,
+    createdAt: scenario.browserMonitorTimestamp, updatedAt: scenario.browserMonitorTimestamp, latestCheck: null,
+  };
+  await page.route('**/api/monitors', (route) => route.fulfill({ json: { data: [monitor] } }));
+  await page.route(`**/api/monitors/${monitor.id}/checks`, (route) => route.fulfill({
+    status: failure.status, json: { error: { code: failure.code, message: failure.message } },
+  }));
+  await page.goto('/monitors');
+  const article = page.getByRole('article', { name: scenario.monitor.name, exact: true });
+  await article.getByRole('button', { name: 'Run check', exact: true }).click();
+  const alert = page.getByRole('main').getByRole('alert');
+  await expect(alert).toHaveText('The requested monitor or resource was not found.');
+  await expect(alert).toHaveAttribute('data-error-code', 'NOT_FOUND');
+  await expect(article).toContainText('No checks yet.');
+  await expect(article.getByText('SUCCEEDED', { exact: true })).toHaveCount(0);
+  await expect(article.getByRole('button', { name: 'Run check', exact: true })).toBeEnabled();
+});
+
+test('INTERNAL_ERROR during loading displays the service failure category without server details', async ({ page }) => {
+  const failure = scenario.browserErrors.find(({ code }) => code === 'INTERNAL_ERROR')!;
+  await page.route('**/api/monitors', (route) => route.fulfill({
+    status: failure.status, json: { error: { code: failure.code, message: failure.message } },
+  }));
+  await page.goto('/monitors');
+  const alert = page.getByRole('main').getByRole('alert');
+  await expect(alert).toHaveText('The monitoring service could not complete the request.');
+  await expect(alert).toHaveAttribute('data-error-code', 'INTERNAL_ERROR');
+  await expect(page.getByRole('article')).toHaveCount(0);
+});
diff --git a/test/unit.test.ts b/test/unit.test.ts
index ee7bbde..f28eb85 100644
--- a/test/unit.test.ts
+++ b/test/unit.test.ts
@@ -1,6 +1,9 @@
 import test from 'node:test';
 import assert from 'node:assert/strict';
 import { fixtureUrl, DEFAULT_FIXTURE_ORIGIN } from '../server/check.ts';
+import { ERROR_MESSAGES, failureCode, RequestFailure, responseData } from '../app/monitors/api.ts';
+import type { ApiErrorCode } from '../server/model.ts';
+import scenario from '../evidence/E02/scenario.json' with { type: 'json' };
 
 test('a path on the configured fixture origin is allowed', () => {
   assert.equal(fixtureUrl(`${DEFAULT_FIXTURE_ORIGIN}/ok`, DEFAULT_FIXTURE_ORIGIN).pathname, '/ok');
@@ -17,3 +20,36 @@ test('credentials and another protocol cannot bypass the fixture boundary', () =
     assert.throws(() => fixtureUrl(url, DEFAULT_FIXTURE_ORIGIN));
   }
 });
+
+test('browser classifies API failures by code and status, independently of server prose', async () => {
+  for (const { status, code, message } of scenario.browserErrors) {
+    await assert.rejects(responseData(Response.json({ error: { code, message } }, { status })), (error: unknown) => {
+      assert.ok(error instanceof RequestFailure);
+      assert.equal(error.code, code);
+      assert.equal(error.message, ERROR_MESSAGES[code as ApiErrorCode]);
+      assert.notEqual(error.message, message);
+      return true;
+    });
+  }
+});
+
+test('browser treats malformed envelopes and status/code mismatches as a safe service failure', async () => {
+  for (const { status, body } of scenario.malformedBrowserResponses) {
+    const response = typeof body === 'string' ? new Response(body, { status }) : Response.json(body, { status });
+    await assert.rejects(responseData(response), (error: unknown) => {
+      assert.equal(failureCode(error), 'INTERNAL_ERROR');
+      return true;
+    });
+  }
+});
+
+test('browser consumes data envelopes without losing false, zero or null wire values', async () => {
+  const data = { enabled: false, latencyMs: 0, httpStatus: null, failureReason: 'CONNECTION_FAILURE' };
+  assert.deepEqual(await responseData(Response.json({ data })), data);
+  assert.deepEqual(await responseData(Response.json({ data: [] })), []);
+});
+
+test('browser transport rejection has a stable fallback independent of exception text', () => {
+  assert.equal(failureCode(new TypeError(scenario.browserNetworkFailure)), 'INTERNAL_ERROR');
+  assert.equal(failureCode(new Error(scenario.additionalBoundaries.privateInternalMessage)), 'INTERNAL_ERROR');
+});
