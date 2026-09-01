## `브라우저 로그인과 인증된 Monitor 흐름 연결`

diff --git a/.github/workflows/ci.yml b/.github/workflows/ci.yml
index 97eb08a..dc35b36 100644
--- a/.github/workflows/ci.yml
+++ b/.github/workflows/ci.yml
@@ -1,4 +1,4 @@
-name: E03 verification
+name: E04 verification
 on: [push, pull_request]
 permissions:
   contents: read
diff --git a/app/login/page.tsx b/app/login/page.tsx
new file mode 100644
index 0000000..ad869c7
--- /dev/null
+++ b/app/login/page.tsx
@@ -0,0 +1,43 @@
+'use client';
+
+import { useState, type FormEvent } from 'react';
+import { errorMessages, failureCode, login, type ApiErrorCode } from '../monitors/api';
+
+export default function Login() {
+  const [busy, setBusy] = useState(false);
+  const [error, setError] = useState<ApiErrorCode | null>(null);
+
+  async function submit(event: FormEvent<HTMLFormElement>) {
+    event.preventDefault();
+    const form = event.currentTarget;
+    const fields = new FormData(form);
+    setBusy(true);
+    setError(null);
+    try {
+      await login(String(fields.get('username') ?? ''), String(fields.get('password') ?? ''));
+      form.reset();
+      window.location.replace('/monitors');
+    } catch (error) {
+      setError(failureCode(error));
+    } finally {
+      const password = form.elements.namedItem('password');
+      if (password instanceof HTMLInputElement) password.value = '';
+      setBusy(false);
+    }
+  }
+
+  return <main>
+    <header><p className="eyebrow">LOCAL LAB · INDUSTRY / SPRING</p><h1>Sign in</h1>
+      <p>Use a fixture account prepared through the local bootstrap command.</p></header>
+    <section aria-label="Sign in">
+      <form onSubmit={submit} method="post" action="/api/session/login">
+        <label>Username<input name="username" autoComplete="username" required maxLength={100} /></label>
+        <label>Password<input name="password" type="password" autoComplete="current-password" required /></label>
+        <button disabled={busy}>{busy ? 'Signing in…' : 'Sign in'}</button>
+      </form>
+      {error && <p role="alert" className="error" data-error-code={error}>
+        {error === 'UNAUTHENTICATED' ? 'Sign-in failed. Check your username and password.' : errorMessages[error]}
+      </p>}
+    </section>
+  </main>;
+}
diff --git a/app/monitors/api.ts b/app/monitors/api.ts
index b01c3c9..6400368 100644
--- a/app/monitors/api.ts
+++ b/app/monitors/api.ts
@@ -4,14 +4,18 @@ export type CheckRun = {
   latencyMs: number; failureReason: 'HTTP_STATUS' | 'CONNECTION_FAILURE' | 'TIMEOUT' | null; finishedAt: string;
 };
 export type MonitorView = { monitor: Monitor; latestCheck: CheckRun | null };
-export type ApiErrorCode = 'INVALID_INPUT' | 'NOT_FOUND' | 'INTERNAL_ERROR';
+export type ApiErrorCode = 'INVALID_INPUT' | 'NOT_FOUND' | 'UNAUTHENTICATED' | 'FORBIDDEN' | 'INTERNAL_ERROR';
 
 export const errorMessages: Record<ApiErrorCode, string> = {
   INVALID_INPUT: 'Invalid monitor input. Check the name, URL, interval, and enabled value.',
   NOT_FOUND: 'Monitor not found. Reload the list and try again.',
+  UNAUTHENTICATED: 'Sign in to continue.',
+  FORBIDDEN: 'The request could not be verified. Reload the page and try again.',
   INTERNAL_ERROR: 'The service could not complete the request. Try again.',
 };
-const errorStatuses: Record<ApiErrorCode, number> = { INVALID_INPUT: 400, NOT_FOUND: 404, INTERNAL_ERROR: 500 };
+const errorStatuses: Record<ApiErrorCode, number> = {
+  INVALID_INPUT: 400, NOT_FOUND: 404, UNAUTHENTICATED: 401, FORBIDDEN: 403, INTERNAL_ERROR: 500,
+};
 
 class ApiFailure extends Error {
   constructor(readonly code: ApiErrorCode) { super(code); }
@@ -60,7 +64,8 @@ async function readData<T>(response: Response, valid: (value: unknown) => value
   if (!response.ok) {
     if (isObject(body) && isObject(body.error) && typeof body.error.message === 'string') {
       const code = body.error.code;
-      if ((code === 'INVALID_INPUT' || code === 'NOT_FOUND' || code === 'INTERNAL_ERROR')
+      if ((code === 'INVALID_INPUT' || code === 'NOT_FOUND' || code === 'UNAUTHENTICATED'
+        || code === 'FORBIDDEN' || code === 'INTERNAL_ERROR')
         && errorStatuses[code] === response.status) throw new ApiFailure(code);
     }
     throw new ApiFailure('INTERNAL_ERROR');
@@ -72,28 +77,45 @@ async function readData<T>(response: Response, valid: (value: unknown) => value
 }
 
 export async function loadMonitors(): Promise<MonitorView[]> {
-  return readData(await fetch('/api/monitors'), (value): value is MonitorView[] =>
+  return readData(await fetch('/api/monitors', { cache: 'no-store' }), (value): value is MonitorView[] =>
     Array.isArray(value) && value.every(isMonitorView));
 }
 
+async function mutation(path: string, method: string, input?: unknown): Promise<Response> {
+  // Only the CSRF value is read by JavaScript; the session cookie remains HttpOnly.
+  // Fetch a current token so login rotation and logout cannot leave a stale cached token.
+  const csrf = await readData(await fetch('/api/session/csrf', { cache: 'no-store' }),
+    (value): value is { headerName: string; token: string } => isObject(value)
+      && value.headerName === 'X-CSRF-TOKEN' && typeof value.token === 'string');
+  return fetch(path, {
+    method, credentials: 'same-origin', headers: { 'Content-Type': 'application/json', [csrf.headerName]: csrf.token },
+    body: input === undefined ? undefined : JSON.stringify(input),
+  });
+}
+
+export async function login(username: string, password: string): Promise<void> {
+  await readData(await mutation('/api/session/login', 'POST', { username, password }),
+    (value): value is { username: string } => isObject(value) && typeof value.username === 'string');
+}
+
+export async function logout(): Promise<void> {
+  await readData(await mutation('/api/session/logout', 'POST'), (value): value is null => value === null);
+}
+
 export async function createMonitor(input: Omit<Monitor, 'id'>): Promise<MonitorView> {
-  return readData(await fetch('/api/monitors', {
-    method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(input),
-  }), isMonitorView);
+  return readData(await mutation('/api/monitors', 'POST', input), isMonitorView);
 }
 
 export async function runCheck(id: string): Promise<CheckRun> {
-  return readData(await fetch(`/api/monitors/${id}/checks`, { method: 'POST' }), isCheckRun);
+  return readData(await mutation(`/api/monitors/${id}/checks`, 'POST'), isCheckRun);
 }
 
 export async function replaceMonitor(id: string, input: Omit<Monitor, 'id'>): Promise<MonitorView> {
-  return readData(await fetch(`/api/monitors/${id}`, {
-    method: 'PUT', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(input),
-  }), isMonitorView);
+  return readData(await mutation(`/api/monitors/${id}`, 'PUT', input), isMonitorView);
 }
 
 export async function deleteMonitor(id: string): Promise<null> {
-  return readData(await fetch(`/api/monitors/${id}`, { method: 'DELETE' }), (value): value is null => value === null);
+  return readData(await mutation(`/api/monitors/${id}`, 'DELETE'), (value): value is null => value === null);
 }
 
 export async function loadChecks(id: string): Promise<CheckRun[]> {
diff --git a/app/monitors/page.tsx b/app/monitors/page.tsx
index 445c9df..0b34d67 100644
--- a/app/monitors/page.tsx
+++ b/app/monitors/page.tsx
@@ -2,7 +2,7 @@
 
 import { useEffect, useState, type FormEvent } from 'react';
 import { createMonitor, deleteMonitor, errorMessages, failureCode, loadChecks, loadMonitors,
-  replaceMonitor, runCheck, type ApiErrorCode, type CheckRun, type Monitor, type MonitorView } from './api';
+  logout, replaceMonitor, runCheck, type ApiErrorCode, type CheckRun, type Monitor, type MonitorView } from './api';
 
 function inputFrom(form: HTMLFormElement): Omit<Monitor, 'id'> {
   const fields = new FormData(form);
@@ -17,8 +17,34 @@ export default function Monitors() {
   const [editingId, setEditingId] = useState<string | null>(null);
   const [histories, setHistories] = useState<Record<string, CheckRun[]>>({});
 
+  function reject(error: unknown) {
+    const code = failureCode(error);
+    if (code === 'UNAUTHENTICATED') {
+      setMonitors([]);
+      setHistories({});
+      setEditingId(null);
+      window.location.replace('/login');
+    } else setError(code);
+  }
+
+  async function signOut() {
+    setBusy(true);
+    setError(null);
+    try {
+      await logout();
+      setMonitors([]);
+      setHistories({});
+      setEditingId(null);
+      window.location.replace('/login');
+    } catch (error) {
+      reject(error);
+    } finally {
+      setBusy(false);
+    }
+  }
+
   useEffect(() => {
-    loadMonitors().then(setMonitors).catch(error => setError(failureCode(error)));
+    loadMonitors().then(setMonitors).catch(reject);
   }, []);
 
   async function create(event: FormEvent<HTMLFormElement>) {
@@ -31,7 +57,7 @@ export default function Monitors() {
       setMonitors(current => [...current, created]);
       form.reset();
     } catch (error) {
-      setError(failureCode(error));
+      reject(error);
     } finally {
       setBusy(false);
     }
@@ -45,7 +71,7 @@ export default function Monitors() {
       setMonitors(current => current.map(row => row.monitor.id === id ? { ...row, latestCheck } : row));
       setHistories(current => current[id] ? { ...current, [id]: [latestCheck, ...current[id]] } : current);
     } catch (error) {
-      setError(failureCode(error));
+      reject(error);
     } finally {
       setBusy(false);
     }
@@ -59,7 +85,7 @@ export default function Monitors() {
       setMonitors(current => current.map(row => row.monitor.id === id ? updated : row));
       setEditingId(null);
     } catch (error) {
-      setError(failureCode(error));
+      reject(error);
     } finally {
       setBusy(false);
     }
@@ -74,7 +100,7 @@ export default function Monitors() {
       setHistories(current => { const remaining = { ...current }; delete remaining[id]; return remaining; });
       if (editingId === id) setEditingId(null);
     } catch (error) {
-      setError(failureCode(error));
+      reject(error);
     } finally {
       setBusy(false);
     }
@@ -91,7 +117,7 @@ export default function Monitors() {
       const checks = await loadChecks(id);
       setHistories(current => ({ ...current, [id]: checks }));
     } catch (error) {
-      setError(failureCode(error));
+      reject(error);
     } finally {
       setBusy(false);
     }
@@ -99,7 +125,8 @@ export default function Monitors() {
 
   return <main>
     <header><p className="eyebrow">LOCAL LAB · INDUSTRY / SPRING</p><h1>Endpoint Monitor</h1>
-      <p>Create a monitor and check its HTTP response.</p></header>
+      <p>Create a monitor and check its HTTP response.</p>
+      <button disabled={busy} onClick={signOut}>Sign out</button></header>
     <aside>Development fixture only. Monitors and completed checks are stored in PostgreSQL.</aside>
     <section aria-labelledby="create-title">
       <h2 id="create-title">Create monitor</h2>
diff --git a/package.json b/package.json
index 9d5673e..3bf1803 100644
--- a/package.json
+++ b/package.json
@@ -14,6 +14,7 @@
     "db:destroy": "node scripts/database.mjs destroy",
     "api:dev": "mvn -B -ntp -f backend/pom.xml spring-boot:run",
     "api:package": "mvn -B -ntp -f backend/pom.xml -DskipTests package",
+    "bootstrap:users": "node scripts/bootstrap-users.mjs",
     "test:api": "mvn -B -ntp -f backend/pom.xml test",
     "typecheck": "next typegen && tsc --noEmit",
     "test:e2e": "playwright test",
diff --git a/playwright.config.ts b/playwright.config.ts
index d218fb9..9b818d9 100644
--- a/playwright.config.ts
+++ b/playwright.config.ts
@@ -1,16 +1,25 @@
 import { defineConfig, devices } from '@playwright/test';
+import { randomBytes } from 'node:crypto';
+
+// Pinned Playwright's switch disables automatic failure DOM/ARIA snapshots.
+process.env.PLAYWRIGHT_NO_COPY_PROMPT = '1';
+// Inherited by workers and the bootstrap child, never persisted to storage-state or evidence.
+process.env.E04_ALICE_PASSWORD ??= randomBytes(32).toString('base64url');
+process.env.E04_BOB_PASSWORD ??= randomBytes(32).toString('base64url');
 
 export default defineConfig({
   testDir: './tests/browser',
   fullyParallel: false,
   workers: 1,
   retries: 0,
+  preserveOutput: 'never',
   timeout: 30_000,
   reporter: [['list'], ['json', { outputFile: 'test-results/browser.json' }]],
-  use: { ...devices['Desktop Chrome'], baseURL: 'http://127.0.0.1:4323', trace: 'retain-on-failure' },
+  use: { ...devices['Desktop Chrome'], baseURL: 'http://127.0.0.1:4323',
+    trace: 'off', screenshot: 'off', video: 'off' },
   webServer: [
     { command: 'node scripts/fixture.mjs', url: 'http://127.0.0.1:4321/ok', reuseExistingServer: false },
-    { command: 'node scripts/test-api.mjs', url: 'http://127.0.0.1:4322/api/monitors', reuseExistingServer: false },
+    { command: 'node scripts/test-api.mjs', url: 'http://127.0.0.1:4322/api/session', reuseExistingServer: false },
     { command: 'npm run dev', url: 'http://127.0.0.1:4323/monitors', reuseExistingServer: false, timeout: 90_000 },
   ],
 });
diff --git a/scripts/bootstrap-users.mjs b/scripts/bootstrap-users.mjs
new file mode 100644
index 0000000..008666c
--- /dev/null
+++ b/scripts/bootstrap-users.mjs
@@ -0,0 +1,21 @@
+import assert from 'node:assert/strict';
+import { spawnSync } from 'node:child_process';
+import { fileURLToPath } from 'node:url';
+
+export function bootstrapUsers(schema, credentials) {
+  assert.ok(credentials.E04_ALICE_PASSWORD && credentials.E04_BOB_PASSWORD,
+    'Provide both E04 fixture passwords through runtime secret input');
+  const result = spawnSync('java', ['-jar', 'backend/target/monitor-api-0.0.1.jar',
+    '--spring.profiles.active=bootstrap', '--spring.main.web-application-type=none',
+    '--spring.main.banner-mode=off', '--logging.level.root=ERROR'], {
+    env: { ...process.env, ...credentials, DB_SCHEMA: schema }, encoding: 'utf8', timeout: 30_000,
+  });
+  // Do not dump child output or environment on failure; neither belongs in credential evidence.
+  assert.equal(result.status, 0, 'Fixture bootstrap process failed');
+  assert.ok(result.stdout.includes('Fixture users prepared: count=2'), 'Owned bootstrap must confirm both users');
+}
+
+if (process.argv[1] === fileURLToPath(import.meta.url)) {
+  bootstrapUsers(process.env.DB_SCHEMA ?? 'public', process.env);
+  console.log('Fixture users prepared: count=2; no credentials printed');
+}
diff --git a/scripts/database.mjs b/scripts/database.mjs
index 0f650dc..8a64c9a 100644
--- a/scripts/database.mjs
+++ b/scripts/database.mjs
@@ -2,8 +2,9 @@ import assert from 'node:assert/strict';
 import { spawnSync } from 'node:child_process';
 
 const compose = ['compose', '--project-name', 'wse-industry', '--file', 'compose.yaml'];
-const schemas = new Set(['e03_restart', 'e03_functional', 'e03_browser', 'e03_migrations',
-  'e03_mapping', 'e03_missing_column', 'e03_extra_required']);
+const schemas = new Set(['e04_restart', 'e04_functional', 'e04_browser', 'e04_migrations',
+  'e04_mapping', 'e04_missing_column', 'e04_extra_required', 'e04_session', 'e04_users',
+  'e04_missing_user_column', 'e04_extra_user_required']);
 const [action, schema] = process.argv.slice(2);
 let args;
 if (action === 'up') args = [...compose, 'up', '--detach', '--wait', '--wait-timeout', '30', 'postgres'];
diff --git a/scripts/persistence-scenario.mjs b/scripts/persistence-scenario.mjs
index aaee967..83e1f8e 100644
--- a/scripts/persistence-scenario.mjs
+++ b/scripts/persistence-scenario.mjs
@@ -4,6 +4,8 @@ import { once } from 'node:events';
 import { appendFileSync, mkdirSync, openSync, closeSync, readFileSync, writeFileSync } from 'node:fs';
 import { createServer } from 'node:net';
 import { setTimeout as delay } from 'node:timers/promises';
+import { randomBytes } from 'node:crypto';
+import { bootstrapUsers } from './bootstrap-users.mjs';
 
 // The label changes evidence filenames only. The baseline and repair use identical inputs.
 const label = process.argv[2];
@@ -14,12 +16,15 @@ const started = Date.now();
 const evidence = { label, requests: [], monitors: [], checks: [], processes: [], portChecks: [] };
 const processes = [];
 const base = 'http://127.0.0.1:4322';
+const credentials = { E04_ALICE_PASSWORD: randomBytes(32).toString('base64url'),
+  E04_BOB_PASSWORD: randomBytes(32).toString('base64url') };
+let sessionCookie;
 
 function start(command, args, logName) {
   const logPath = `${directory}/${label}-${logName}.log`;
   const log = openSync(logPath, 'w');
   const child = spawn(command, args, {
-    env: { ...process.env, DB_SCHEMA: 'e03_restart' }, stdio: ['ignore', log, log],
+    env: { ...process.env, DB_SCHEMA: 'e04_restart' }, stdio: ['ignore', log, log],
   });
   closeSync(log);
   child.evidence = { role: logName, pid: child.pid, startedAt: new Date().toISOString(), logPath };
@@ -46,7 +51,7 @@ function requireAlive(child) {
     `Owned ${child.evidence.role} process is not alive`);
 }
 
-async function ready(url, child, startupMarker) {
+async function ready(url, child, startupMarker, expectedStatus = 200) {
   const deadline = Date.now() + 30_000;
   while (Date.now() < deadline) {
     requireAlive(child);
@@ -56,7 +61,7 @@ async function ready(url, child, startupMarker) {
       try { response = await fetch(url, { signal: AbortSignal.timeout(1000) }); }
       catch { /* Bounded readiness polling, never a scenario retry. */ }
       requireAlive(child);
-      if (response?.ok) {
+      if (response?.status === expectedStatus) {
         child.evidence.readyAt = new Date().toISOString();
         child.evidence.ownedStartupMarkerObserved = true;
         return;
@@ -82,11 +87,39 @@ async function stop(child) {
   } finally { clearTimeout(force); }
 }
 
+async function csrfHeaders() {
+  requireAlive(api);
+  const response = await fetch(`${base}/api/session/csrf`, {
+    headers: sessionCookie ? { Cookie: sessionCookie } : {}, signal: AbortSignal.timeout(5000),
+  });
+  assert.equal(response.status, 200, 'CSRF setup must succeed');
+  const cookie = response.headers.get('set-cookie');
+  if (cookie) sessionCookie = cookie.split(';', 1)[0];
+  const wire = await response.json();
+  assert.ok(sessionCookie && wire.data?.headerName === 'X-CSRF-TOKEN' && typeof wire.data?.token === 'string');
+  return { Cookie: sessionCookie, [wire.data.headerName]: wire.data.token };
+}
+
+async function authenticate() {
+  sessionCookie = undefined;
+  const response = await fetch(`${base}/api/session/login`, {
+    method: 'POST', headers: { ...(await csrfHeaders()), 'Content-Type': 'application/json' },
+    body: JSON.stringify({ username: 'alice-e04', password: credentials.E04_ALICE_PASSWORD }),
+    signal: AbortSignal.timeout(5000),
+  });
+  assert.equal(response.status, 200, 'Authorized regression setup must succeed');
+  const cookie = response.headers.get('set-cookie');
+  assert.ok(cookie, 'Successful login must issue its rotated session cookie');
+  sessionCookie = cookie.split(';', 1)[0];
+  // Credentials, session headers and CSRF responses are deliberately omitted from wire evidence.
+}
+
 async function request(path, method = 'GET', body, status = 200) {
   requireAlive(api);
   const requestStarted = Date.now();
   const response = await fetch(`${base}${path}`, {
-    method, headers: body === undefined ? {} : { 'Content-Type': 'application/json' },
+    method, headers: { ...(method === 'GET' ? { Cookie: sessionCookie } : await csrfHeaders()),
+      ...(body === undefined ? {} : { 'Content-Type': 'application/json' }) },
     body: body === undefined ? undefined : JSON.stringify(body), signal: AbortSignal.timeout(10_000),
   });
   const wire = await response.json();
@@ -106,10 +139,12 @@ try {
   // Refuse existing listeners before starting any child or sending any request.
   await requireFreePort(4321);
   await requireFreePort(4322);
+  bootstrapUsers('e04_restart', credentials);
   const fixture = start(process.execPath, ['scripts/fixture.mjs'], 'fixture');
   await ready('http://127.0.0.1:4321/ok', fixture, 'Fixture http://127.0.0.1:4321');
   api = start('java', ['-jar', 'backend/target/monitor-api-0.0.1.jar'], 'api-first');
-  await ready(`${base}/api/monitors`, api, 'Started MonitorApplication');
+  await ready(`${base}/api/session`, api, 'Started MonitorApplication', 401);
+  await authenticate();
   const firstApiPid = api.pid;
   assert.deepEqual(await request('/api/monitors'), [], 'Scenario requires an isolated empty store');
   const a = (await request('/api/monitors', 'POST', {
@@ -131,7 +166,8 @@ try {
   await requireFreePort(4322);
   api = start('java', ['-jar', 'backend/target/monitor-api-0.0.1.jar'], 'api-fresh');
   assert.notEqual(api.pid, firstApiPid, 'Persistence verification must use a distinct fresh API process');
-  await ready(`${base}/api/monitors`, api, 'Started MonitorApplication');
+  await ready(`${base}/api/session`, api, 'Started MonitorApplication', 401);
+  await authenticate();
   const restored = await request('/api/monitors');
   assert.equal(restored.length, 2, 'Fresh API must retain both Monitors');
   for (const monitor of [a, b]) {
diff --git a/scripts/test-api.mjs b/scripts/test-api.mjs
index a85612d..404a207 100644
--- a/scripts/test-api.mjs
+++ b/scripts/test-api.mjs
@@ -1,10 +1,36 @@
 import { spawn, spawnSync } from 'node:child_process';
+import { once } from 'node:events';
+import { createServer } from 'node:net';
+import { bootstrapUsers } from './bootstrap-users.mjs';
 
-const reset = spawnSync(process.execPath, ['scripts/database.mjs', 'reset', 'e03_browser'], { stdio: 'inherit' });
-if (reset.status !== 0) process.exit(reset.status ?? 1);
-const api = spawn('java', ['-jar', 'backend/target/monitor-api-0.0.1.jar'], {
-  env: { ...process.env, DB_SCHEMA: 'e03_browser' }, stdio: 'inherit',
+const probe = createServer();
+await new Promise((resolve, reject) => {
+  probe.once('error', () => reject(new Error('Refusing occupied API port 4322')));
+  probe.listen({ host: '127.0.0.1', port: 4322, exclusive: true }, () => probe.close(resolve));
 });
-for (const signal of ['SIGINT', 'SIGTERM']) process.on(signal, () => api.kill(signal));
-api.once('error', error => { console.error(error.message); process.exitCode = 1; });
-api.once('exit', code => { process.exitCode = code ?? 0; });
+const reset = spawnSync(process.execPath, ['scripts/database.mjs', 'reset', 'e04_browser'], { stdio: 'inherit' });
+if (reset.status !== 0) process.exit(reset.status ?? 1);
+let api;
+let force;
+try {
+  bootstrapUsers('e04_browser', process.env);
+  const { E04_ALICE_PASSWORD, E04_BOB_PASSWORD, ...runtime } = process.env;
+  api = spawn('java', ['-jar', 'backend/target/monitor-api-0.0.1.jar'], {
+    env: { ...runtime, DB_SCHEMA: 'e04_browser' }, stdio: 'inherit',
+  });
+  for (const signal of ['SIGINT', 'SIGTERM']) process.on(signal, () => {
+    api.kill(signal);
+    force ??= setTimeout(() => api.kill('SIGKILL'), 5000);
+  });
+  const [code, signal] = await once(api, 'exit');
+  process.exitCode = signal === 'SIGTERM' || signal === 'SIGINT' ? 0 : (code ?? 1);
+} finally {
+  clearTimeout(force);
+  if (api && api.exitCode === null && api.signalCode === null) {
+    const exited = once(api, 'exit');
+    api.kill('SIGTERM');
+    await exited;
+  }
+  const cleanup = spawnSync(process.execPath, ['scripts/database.mjs', 'drop', 'e04_browser'], { stdio: 'inherit' });
+  if (cleanup.status !== 0) process.exitCode = cleanup.status ?? 1;
+}
diff --git a/scripts/verify.mjs b/scripts/verify.mjs
index 349e4cf..09067b2 100644
--- a/scripts/verify.mjs
+++ b/scripts/verify.mjs
@@ -9,7 +9,7 @@ const commands = [
   ['npm', ['run', 'typecheck']],
   ['npm', ['run', 'build']],
   ['node', ['scripts/persistence-isolation.mjs']],
-  ['node', ['scripts/database.mjs', 'reset', 'e03_restart']],
+  ['node', ['scripts/database.mjs', 'reset', 'e04_restart']],
   ['node', ['scripts/persistence-scenario.mjs', 'fixed']],
   ['npm', ['run', 'test:e2e']],
 ];
@@ -28,7 +28,7 @@ try {
     if (status !== 0) break;
   }
 } finally {
-  for (const schema of ['e03_restart', 'e03_browser']) {
+  for (const schema of ['e04_restart', 'e04_browser']) {
     const cleanup = run('node', ['scripts/database.mjs', 'drop', schema]);
     if (status === 0) status = cleanup;
   }
diff --git a/tests/browser/authenticated.ts b/tests/browser/authenticated.ts
new file mode 100644
index 0000000..f11dabe
--- /dev/null
+++ b/tests/browser/authenticated.ts
@@ -0,0 +1,45 @@
+import { test as base, expect, type APIRequestContext, type APIResponse, type Page } from '@playwright/test';
+
+export function fixturePassword(user: 'alice' | 'bob'): string {
+  const value = process.env[user === 'alice' ? 'E04_ALICE_PASSWORD' : 'E04_BOB_PASSWORD'];
+  if (!value) throw new Error('Runtime fixture credential is missing');
+  return value;
+}
+
+export async function csrfHeaders(request: APIRequestContext): Promise<Record<string, string>> {
+  const response = await safeRequest(() => request.get('/api/session/csrf'));
+  expect(response.status()).toBe(200);
+  let body;
+  try { body = await response.json(); }
+  catch { throw new Error('CSRF bootstrap returned invalid JSON'); }
+  expect(body.data?.headerName === 'X-CSRF-TOKEN' && typeof body.data?.token === 'string').toBe(true);
+  return { [body.data.headerName]: body.data.token };
+}
+
+export async function safeRequest(action: () => Promise<APIResponse>): Promise<APIResponse> {
+  try { return await action(); }
+  catch { throw new Error('Authenticated browser API request could not complete'); }
+}
+
+export async function fillSecret(page: Page, password: string): Promise<void> {
+  try {
+    await page.getByLabel('Password', { exact: true }).fill(password);
+  } catch {
+    // A Playwright fill timeout may include the input value in its call log. Never retain that error.
+    throw new Error('Password field could not be filled');
+  }
+}
+
+export const test = base.extend({
+  page: async ({ page }, use) => {
+    const headers = await csrfHeaders(page.request);
+    const response = await safeRequest(() => page.request.post('/api/session/login', {
+      headers,
+      data: { username: 'alice-e04', password: fixturePassword('alice') },
+    }));
+    expect(response.status()).toBe(200);
+    await use(page);
+  },
+});
+
+export { expect };
diff --git a/tests/browser/errors.spec.ts b/tests/browser/errors.spec.ts
index ebd576d..639fe54 100644
--- a/tests/browser/errors.spec.ts
+++ b/tests/browser/errors.spec.ts
@@ -1,4 +1,5 @@
-import { test, expect, type Page, type Route } from '@playwright/test';
+import { test, expect } from './authenticated';
+import type { Page, Route } from '@playwright/test';
 
 const inputError = 'Invalid monitor input. Check the name, URL, interval, and enabled value.';
 const internalError = 'The service could not complete the request. Try again.';
diff --git a/tests/browser/monitor.spec.ts b/tests/browser/monitor.spec.ts
index 4369a10..20a52f1 100644
--- a/tests/browser/monitor.spec.ts
+++ b/tests/browser/monitor.spec.ts
@@ -1,4 +1,4 @@
-import { test, expect } from '@playwright/test';
+import { test, expect } from './authenticated';
 
 test('create fixture monitor and display successful synchronous check', async ({ page }) => {
   await page.goto('/monitors');
diff --git a/tests/browser/session.spec.ts b/tests/browser/session.spec.ts
new file mode 100644
index 0000000..05cbd4f
--- /dev/null
+++ b/tests/browser/session.spec.ts
@@ -0,0 +1,68 @@
+import { randomBytes } from 'node:crypto';
+import { test, expect, type Page } from '@playwright/test';
+import { csrfHeaders, fillSecret, fixturePassword, safeRequest } from './authenticated';
+
+async function signIn(page: Page, user: 'alice' | 'bob'): Promise<void> {
+  await page.goto('/login');
+  await page.getByLabel('Username', { exact: true }).fill(`${user}-e04`);
+  await fillSecret(page, fixturePassword(user));
+  await page.getByRole('button', { name: 'Sign in', exact: true }).click();
+  await expect(page).toHaveURL('/monitors');
+  await expect(page.getByRole('heading', { name: 'Endpoint Monitor' })).toBeVisible();
+}
+
+test('browser login recovers from invalid credentials, rotates and revokes its HttpOnly session', async ({ page, context }) => {
+  await page.goto('/monitors');
+  await expect(page).toHaveURL('/login');
+  const anonymous = await safeRequest(() => page.request.get('/api/monitors'));
+  expect(anonymous.status()).toBe(401);
+  expect((await anonymous.json()).error.code).toBe('UNAUTHENTICATED');
+
+  await page.getByLabel('Username', { exact: true }).fill('alice-e04');
+  await fillSecret(page, randomBytes(32).toString('base64url'));
+  await page.getByRole('button', { name: 'Sign in', exact: true }).click();
+  await expect(page.getByRole('main').getByRole('alert')).toHaveAttribute('data-error-code', 'UNAUTHENTICATED');
+  await expect(page.getByRole('main').getByRole('alert')).toHaveText('Sign-in failed. Check your username and password.');
+  await expect(page.getByLabel('Password', { exact: true })).toHaveValue('');
+
+  await fillSecret(page, fixturePassword('alice'));
+  await page.getByRole('button', { name: 'Sign in', exact: true }).click();
+  await expect(page).toHaveURL('/monitors');
+  const first = (await context.cookies()).find(cookie => cookie.name === 'WSESESSION');
+  expect(!!first && first.httpOnly && first.sameSite === 'Lax' && !first.secure).toBe(true);
+  expect(await page.evaluate(() => document.cookie.includes('WSESESSION='))).toBe(false);
+  expect((await safeRequest(() => page.request.get('/api/monitors'))).status()).toBe(200);
+
+  await signIn(page, 'alice');
+  const rotated = (await context.cookies()).find(cookie => cookie.name === 'WSESESSION');
+  expect(!!rotated && !!first && rotated.value !== first.value).toBe(true);
+  const old = await fetch('http://127.0.0.1:4323/api/monitors', {
+    headers: { Cookie: `WSESESSION=${first!.value}` }, signal: AbortSignal.timeout(5000),
+  });
+  expect(old.status).toBe(401);
+
+  await page.getByRole('button', { name: 'Sign out', exact: true }).click();
+  await expect(page).toHaveURL('/login');
+  expect((await context.cookies()).some(cookie => cookie.name === 'WSESESSION')).toBe(false);
+  const revoked = await fetch('http://127.0.0.1:4323/api/monitors', {
+    headers: { Cookie: `WSESESSION=${rotated!.value}` }, signal: AbortSignal.timeout(5000),
+  });
+  expect(revoked.status).toBe(401);
+});
+
+test('Bob logs in independently and the UI recovers after out-of-band session revocation', async ({ page }) => {
+  await signIn(page, 'bob');
+  const current = await safeRequest(() => page.request.get('/api/session'));
+  expect(current.status()).toBe(200);
+  expect((await current.json()).data.username).toBe('bob-e04');
+  const headers = await csrfHeaders(page.request);
+  const revoke = await safeRequest(() => page.request.post('/api/session/logout', { headers }));
+  expect(revoke.status()).toBe(200);
+
+  await page.getByLabel('Name', { exact: true }).fill('Session recovery fixture');
+  await page.getByRole('button', { name: 'Create monitor', exact: true }).click();
+  await expect(page).toHaveURL('/login');
+  await expect(page.getByRole('heading', { name: 'Sign in', exact: true })).toBeVisible();
+  await signIn(page, 'bob');
+  await expect(page.getByRole('article', { name: 'Session recovery fixture', exact: true })).toHaveCount(0);
+});


