## `Connect browser sign-in to the session lifecycle`

diff --git a/TRACK.md b/TRACK.md
index 1adcdf6..e894872 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -60,9 +60,11 @@ npm run dev:api
 npm run dev:web
 ```
 
-Open `http://127.0.0.1:4313/monitors`. Next proxies same-origin `/api` requests to
+Open `http://127.0.0.1:4313/login`. Next proxies same-origin `/api` requests to
 the Fastify loopback API; `API_ORIGIN` configures that trusted proxy target.
-Create a Monitor for `/ok`, `/fail`, or `/redirect`, then choose **Run check**.
+E04 requires a prepared account; the reproducible two-user fixture and browser
+login command are described below. After signing in, create a Monitor for `/ok`,
+`/fail`, or `/redirect`, then choose **Run check**.
 
 `API_PORT` changes the API port (default 4312). `FIXTURE_PORT` changes the fixture
 listener (default 4311); set `FIXTURE_ORIGIN` on the API to the same trusted origin.
@@ -80,8 +82,8 @@ npm run build
 
 Functional tests own fixed loopback ports 4311, 4312 and 4314; stop manual fixture
 processes before running them. Port 4314 is a controlled destination that must
-never receive a check. E03 uses PostgreSQL on loopback port 15431. There is no login,
-worker, Redis, Kafka, or production application container.
+never receive a check. E03 uses PostgreSQL on loopback port 15431. E04 adds login;
+there is no worker, Redis, Kafka, or production application container.
 The E2E command owns ports 4311–4313 and starts fresh fixture/API/Next processes;
 stop manual development processes first. It uses one Chromium worker, no test
 retries, and the fixed Monitor fixture in `evidence/E01/scenario.json`.
@@ -105,6 +107,7 @@ All API successes use `{ "data": <payload> }`, including health, Monitor creatio
 | Code | HTTP status |
 | --- | --- |
 | `INVALID_INPUT` | 400 |
+| `UNAUTHENTICATED` (E04) | 401 |
 | `NOT_FOUND` | 404 |
 | `INTERNAL_ERROR` | 500 |
 
@@ -149,13 +152,14 @@ different disposable database/schema; do not commit a DSN containing credentials
 `npm run db:down` stops only this project and retains its named data volume.
 There is no application container.
 
-Run `npm run db:migrate` before starting the API. It applies `001_monitors.sql`
-and then `002_check_runs.sql`, on one explicit `pg` client within a DDL
+Run `npm run db:migrate` before starting the API. It applies `001_monitors.sql`,
+`002_check_runs.sql`, and E04's `003_sessions.sql`, on one explicit `pg` client within a DDL
 transaction, and records exact file checksums in `schema_migrations`. Run this
 command serially; a repeated command prints `{"applied":[]}`. Applied files must
 not be edited. Startup never creates tables: it requires the complete matching
 migration history, the exact mapped columns and their types/nullability/timestamp precision,
-primary keys and the CheckRun parent cascade. A missing or incompatible mapped
+primary keys, the CheckRun parent cascade, and E04's unique username/session user
+relation. A missing or incompatible mapped
 column rejects `app.listen` before the API opens a port.
 
 `server/mapping.ts` is the explicit SQL-row/domain boundary. `interval_seconds`
@@ -200,7 +204,7 @@ npm run build
 npm run db:down
 ```
 
-Functional/integration tests recreate only their explicit `e03_*` schemas and
+Functional/integration tests recreate only their explicit `e03_*`/`e04_*` schemas and
 remove them afterwards; they never truncate the application's `public` schema.
 The browser API command prepares `e03_browser`, and Playwright teardown removes
 it. Every test uses one worker/serial execution and zero automatic retries.
@@ -212,3 +216,80 @@ The frozen packet, one memory-loss baseline, real PostgreSQL results, command
 outcomes and durations are in `evidence/E03/`. The historical E01/E02 evidence is
 unchanged; only E01's now-obsolete empty-new-instance assertion was replaced by
 the E03 persistence assertion.
+
+## E04 session authentication
+
+Every API route except `GET /health` (and its HEAD counterpart) and
+`POST /auth/login` requires a live session cookie. Authentication runs in
+Fastify's `onRequest` hook before body validation and Monitor SQL. Valid protected
+routes reject absent, unknown, expired or revoked sessions with HTTP 401 and
+`{ "error": { "code": "UNAUTHENTICATED", "message": "Authentication is required." } }`.
+Authorized E02/E03 input, envelope and persistence contracts are unchanged.
+
+| Operation | Route | Success data |
+| --- | --- | --- |
+| Sign in | `POST /auth/login` | `{user: {id, username}}` |
+| Current user | `GET /auth/session` | `{user: {id, username}}` |
+| Sign out | `POST /auth/logout` | `{loggedOut: true}` |
+
+Login takes a JSON object with `username` and `password`. Usernames are exactly
+3–64 lowercase ASCII letters, digits, underscores or hyphens, beginning with a
+letter. Password input is a string of 1–1024 UTF-16 code units; it is not trimmed
+or coerced. Wrong passwords and unknown accounts have the same status and body;
+unknown accounts still perform the password KDF. Malformed inputs use
+`INVALID_INPUT`. Passwords, hashes and session identifiers are never response data.
+
+Node's [asynchronous scrypt](https://nodejs.org/docs/latest-v24.x/api/crypto.html#cryptoscryptpassword-salt-keylen-options-callback)
+uses a fresh 16-byte salt, 64-byte derived key, and fixed N=131072, r=8, p=1,
+maxmem=256 MiB. This is one of the published [OWASP scrypt settings](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html#scrypt).
+`users.password_hash` stores the algorithm, parameters, salt and one-way result;
+verification uses `timingSafeEqual`. No dependency or version pin changed.
+
+Each successful login generates a fresh 32-byte opaque identifier. PostgreSQL
+stores only its SHA-256 digest, its user relation, creation time and absolute
+expiration time. The TTL is fixed at **one hour**, with no rolling extension.
+Every authenticated request checks `expires_at > now`; equality is expired.
+Login deletes the presented old session and inserts the replacement on one
+checked-out SQL connection in one transaction, then sets the cookie after commit.
+Logout deletes the session row and clears the cookie. These decisions persist
+across application instances; there is no in-memory session authority.
+
+The host-only `wse_session` cookie has `Path=/`, `HttpOnly`, explicit
+`SameSite=Lax`, and `Max-Age=3600`; clearing uses the same scope and `Max-Age=0`.
+**Secure is intentionally absent only for the current loopback HTTP transport.**
+This configuration is not supported on a public HTTP interface. The browser uses
+the existing same-origin Next proxy with `credentials: 'same-origin'`. HttpOnly
+keeps the identifier out of [frontend JavaScript cookie access](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Set-Cookie#httponly).
+API responses use `Cache-Control: no-store`. The login form clears its transient
+password input after sending; it never keeps passwords or tokens in React state,
+localStorage or sessionStorage. A 401 clears displayed Monitor/history state and
+opens `/login`; successful logout does the same. Ownership and CSRF enforcement
+remain E05 work; the present cookie attributes are credential transport policy.
+
+`test/auth.ts` is the reproducible account fixture. Each invocation prepares
+**alice-e04** and **bob-e04** with independent 32-byte random passwords, inserts
+only salted hashes, and returns credentials only to its calling test process.
+It never writes or prints plaintext credentials. There is no signup or account
+management route. To reproduce actual browser login and the authorized UI:
+
+```sh
+npm run db:up
+npm run test:e2e
+```
+
+The runner creates a disposable `e03_browser` schema and uses those two in-memory
+credentials to sign in through the real UI. Earlier browser scenarios acquire
+authenticated cookies in setup; their Monitor inputs and expectations are unchanged.
+The E04 scenario additionally covers anonymous redirects, wrong-password feedback,
+reload, logout, a replayed revoked cookie and Bob's independent login. Traces,
+videos and automatic screenshots are disabled because they can retain credentials;
+existing explicit Monitor screenshots contain no credentials. One worker, zero
+retries and a first-failure stop keep the run bounded.
+
+Unit tests cover hashing, login input, cookie parsing and browser 401 categories.
+The functional gate runs the frozen session sequence against real PostgreSQL and
+an injected clock at expiry minus 1 ms and exactly expiry, without sleeping.
+It also checks 40 protected-route rejections across absent, unknown, expired and
+revoked credentials. The integration gate retains all E03 cases and rejects ten
+fixed auth schema mutations before listening. Safe status/flag/boolean evidence,
+the unchanged-START anonymous baseline, counts and durations are in `evidence/E04/`.
diff --git a/app/login/page.tsx b/app/login/page.tsx
new file mode 100644
index 0000000..a1b5041
--- /dev/null
+++ b/app/login/page.tsx
@@ -0,0 +1,54 @@
+'use client';
+
+import { useState } from 'react';
+import type { FormEvent } from 'react';
+import { useRouter } from 'next/navigation';
+import type { ApiErrorCode } from '../../server/model';
+import { ERROR_MESSAGES, failureCode, responseData } from '../monitors/api';
+
+export default function LoginPage() {
+  const router = useRouter();
+  const [pending, setPending] = useState(false);
+  const [error, setError] = useState<ApiErrorCode | null>(null);
+
+  async function signIn(event: FormEvent<HTMLFormElement>) {
+    event.preventDefault();
+    const form = event.currentTarget;
+    const fields = new FormData(form);
+    setPending(true);
+    setError(null);
+    try {
+      const response = fetch('/api/auth/login', {
+        method: 'POST', credentials: 'same-origin', headers: { 'content-type': 'application/json' },
+        body: JSON.stringify({ username: fields.get('username'), password: fields.get('password') }),
+      });
+      // The password is transient form input, never React state or browser storage.
+      fields.delete('password');
+      (form.elements.namedItem('password') as HTMLInputElement).value = '';
+      await responseData(await response);
+      router.replace('/monitors');
+    } catch (failure) { setError(failureCode(failure)); }
+    finally { setPending(false); }
+  }
+
+  const message = error === 'UNAUTHENTICATED' ? 'Username or password was not accepted.'
+    : error === 'INVALID_INPUT' ? 'Check the username and password format.'
+      : error ? ERROR_MESSAGES[error] : null;
+
+  return <main>
+    <header><p className="eyebrow">HTTP ENDPOINT MONITOR · LOCAL FIXTURE</p><h1>Sign in</h1>
+      <p>Sign in to create monitors and inspect their check history.</p>
+    </header>
+    <aside>Development only. Use an account prepared by the documented local fixture.</aside>
+    {error && <p role="alert" className="error" data-error-code={error}>{message}</p>}
+    <section className="panel" aria-label="Sign in form">
+      <form onSubmit={signIn}>
+        <label htmlFor="username">Username</label>
+        <input id="username" name="username" autoComplete="username" required minLength={3} maxLength={64} />
+        <label htmlFor="password">Password</label>
+        <input id="password" name="password" type="password" autoComplete="current-password" required maxLength={1024} />
+        <button type="submit" disabled={pending}>{pending ? 'Signing in…' : 'Sign in'}</button>
+      </form>
+    </section>
+  </main>;
+}
diff --git a/app/monitors/page.tsx b/app/monitors/page.tsx
index c41fc4d..4550383 100644
--- a/app/monitors/page.tsx
+++ b/app/monitors/page.tsx
@@ -1,11 +1,16 @@
 'use client';
 
-import { useEffect, useState } from 'react';
+import { useCallback, useEffect, useState } from 'react';
 import type { FormEvent } from 'react';
+import { useRouter } from 'next/navigation';
 import type { ApiErrorCode, CheckRun, MonitorView } from '../../server/model';
 import { ERROR_MESSAGES, failureCode, responseData } from './api';
 
 export default function MonitorsPage() {
+  const router = useRouter();
+  const [authenticated, setAuthenticated] = useState(false);
+  const [loading, setLoading] = useState(true);
+  const [loggingOut, setLoggingOut] = useState(false);
   const [monitors, setMonitors] = useState<MonitorView[]>([]);
   const [error, setError] = useState<ApiErrorCode | null>(null);
   const [creating, setCreating] = useState(false);
@@ -17,10 +22,38 @@ export default function MonitorsPage() {
   const [history, setHistory] = useState<CheckRun[]>([]);
   const [loadingHistory, setLoadingHistory] = useState(false);
 
+  const handleFailure = useCallback((failure: unknown) => {
+    const code = failureCode(failure);
+    setError(code);
+    if (code === 'UNAUTHENTICATED') {
+      setAuthenticated(false);
+      setMonitors([]);
+      setHistory([]);
+      setHistoryId(null);
+      router.replace('/login');
+    }
+  }, [router]);
+
   useEffect(() => {
-    fetch('/api/monitors').then(responseData<MonitorView[]>).then(setMonitors)
-      .catch((failure: unknown) => setError(failureCode(failure)));
-  }, []);
+    fetch('/api/monitors', { credentials: 'same-origin' }).then(responseData<MonitorView[]>).then((data) => {
+      setMonitors(data);
+      setAuthenticated(true);
+    }).catch(handleFailure).finally(() => setLoading(false));
+  }, [handleFailure]);
+
+  async function signOut() {
+    setLoggingOut(true);
+    setError(null);
+    try {
+      await responseData(await fetch('/api/auth/logout', { method: 'POST', credentials: 'same-origin' }));
+      setAuthenticated(false);
+      setMonitors([]);
+      setHistory([]);
+      setHistoryId(null);
+      router.replace('/login');
+    } catch (failure) { handleFailure(failure); }
+    finally { setLoggingOut(false); }
+  }
 
   async function createMonitor(event: FormEvent<HTMLFormElement>) {
     event.preventDefault();
@@ -30,7 +63,7 @@ export default function MonitorsPage() {
     setError(null);
     try {
       const response = await fetch('/api/monitors', {
-        method: 'POST', headers: { 'content-type': 'application/json' },
+        method: 'POST', credentials: 'same-origin', headers: { 'content-type': 'application/json' },
         body: JSON.stringify({
           name: fields.get('name'), url: fields.get('url'),
           interval: Number(fields.get('interval')), enabled: fields.get('enabled') === 'on',
@@ -40,7 +73,7 @@ export default function MonitorsPage() {
       setMonitors((current) => [...current, monitor]);
       form.reset();
     } catch (failure) {
-      setError(failureCode(failure));
+      handleFailure(failure);
     } finally { setCreating(false); }
   }
 
@@ -48,12 +81,12 @@ export default function MonitorsPage() {
     setChecking(id);
     setError(null);
     try {
-      const response = await fetch(`/api/monitors/${id}/checks`, { method: 'POST' });
+      const response = await fetch(`/api/monitors/${id}/checks`, { method: 'POST', credentials: 'same-origin' });
       const result = await responseData<CheckRun>(response);
       setMonitors((current) => current.map((monitor) => monitor.id === id ? { ...monitor, latestCheck: result } : monitor));
       if (historyId === id) await showHistory(id);
     } catch (failure) {
-      setError(failureCode(failure));
+      handleFailure(failure);
     } finally { setChecking(null); }
   }
 
@@ -62,12 +95,12 @@ export default function MonitorsPage() {
     setError(null);
     try {
       const response = await fetch(`/api/monitors/${monitor.id}`, {
-        method: 'PUT', headers: { 'content-type': 'application/json' }, body: JSON.stringify(input),
+        method: 'PUT', credentials: 'same-origin', headers: { 'content-type': 'application/json' }, body: JSON.stringify(input),
       });
       const saved = await responseData<MonitorView>(response);
       setMonitors((current) => current.map((item) => item.id === saved.id ? saved : item));
       setEditing(null);
-    } catch (failure) { setError(failureCode(failure)); }
+    } catch (failure) { handleFailure(failure); }
     finally { setSaving(null); }
   }
 
@@ -85,12 +118,12 @@ export default function MonitorsPage() {
     setDeleting(id);
     setError(null);
     try {
-      const response = await fetch(`/api/monitors/${id}`, { method: 'DELETE' });
+      const response = await fetch(`/api/monitors/${id}`, { method: 'DELETE', credentials: 'same-origin' });
       await responseData<{ id: string }>(response);
       setMonitors((current) => current.filter((monitor) => monitor.id !== id));
       if (historyId === id) { setHistoryId(null); setHistory([]); }
       if (editing === id) setEditing(null);
-    } catch (failure) { setError(failureCode(failure)); }
+    } catch (failure) { handleFailure(failure); }
     finally { setDeleting(null); }
   }
 
@@ -100,15 +133,23 @@ export default function MonitorsPage() {
     setLoadingHistory(true);
     setError(null);
     try {
-      const response = await fetch(`/api/monitors/${id}/checks`);
+      const response = await fetch(`/api/monitors/${id}/checks`, { credentials: 'same-origin' });
       setHistory(await responseData<CheckRun[]>(response));
-    } catch (failure) { setError(failureCode(failure)); }
+    } catch (failure) { handleFailure(failure); }
     finally { setLoadingHistory(false); }
   }
 
+  if (!authenticated) return <main>
+    <h1>Monitors</h1>
+    {loading ? <p>Loading monitors…</p> : error && error !== 'UNAUTHENTICATED'
+      ? <p role="alert" className="error" data-error-code={error}>{ERROR_MESSAGES[error]}</p>
+      : <p>Opening sign in…</p>}
+  </main>;
+
   return <main>
     <header><p className="eyebrow">HTTP ENDPOINT MONITOR · LOCAL FIXTURE</p><h1>Monitors</h1>
       <p>Create an endpoint monitor, run a check, and inspect the response.</p>
+      <button onClick={signOut} disabled={loggingOut}>{loggingOut ? 'Signing out…' : 'Sign out'}</button>
     </header>
     <aside>Development only. Checks can access the configured fixture origin. Monitors and all check history are stored in PostgreSQL.</aside>
     {error && <p role="alert" className="error" data-error-code={error}>{ERROR_MESSAGES[error]}</p>}
diff --git a/playwright.config.ts b/playwright.config.ts
index 1f657f2..f9d061c 100644
--- a/playwright.config.ts
+++ b/playwright.config.ts
@@ -7,9 +7,11 @@ export default defineConfig({
   fullyParallel: false,
   workers: 1,
   retries: 0,
+  maxFailures: 1,
   timeout: 30_000,
   reporter: 'list',
-  use: { baseURL: 'http://127.0.0.1:4313', trace: 'retain-on-failure' },
+  // Traces include request bodies and cookies; never retain login credentials.
+  use: { baseURL: 'http://127.0.0.1:4313', trace: 'off', screenshot: 'off', video: 'off' },
   projects: [{ name: 'chromium', use: { ...devices['Desktop Chrome'] } }],
   webServer: [
     { command: 'npm run fixture', url: 'http://127.0.0.1:4311/ok', reuseExistingServer: false, timeout: 30_000 },
diff --git a/test/browser/contracts.spec.ts b/test/browser/contracts.spec.ts
index fe10e55..6a2cb54 100644
--- a/test/browser/contracts.spec.ts
+++ b/test/browser/contracts.spec.ts
@@ -1,4 +1,5 @@
-import { expect, test } from '@playwright/test';
+import { expect } from '@playwright/test';
+import { test } from './session';
 import scenario from '../../evidence/E02/scenario.json' with { type: 'json' };
 
 test('INVALID_INPUT displays the same category for both server prose variants and never creates success UI', async ({ page }) => {
diff --git a/test/browser/lifecycle.spec.ts b/test/browser/lifecycle.spec.ts
index f8de350..90b8a1f 100644
--- a/test/browser/lifecycle.spec.ts
+++ b/test/browser/lifecycle.spec.ts
@@ -1,8 +1,12 @@
-import { expect, test } from '@playwright/test';
+import { expect } from '@playwright/test';
+import { randomBytes } from 'node:crypto';
+import { mkdir, writeFile } from 'node:fs/promises';
+import { test, submitCredentials } from './session';
 import type { CheckRun, MonitorView } from '../../server/model';
 import scenario from '../../evidence/E03/scenario.json' with { type: 'json' };
+import authScenario from '../../evidence/E04/scenario.json' with { type: 'json' };
 
-test('E03 persist A,A,B history, edit, pause, enable and delete through the real browser and PostgreSQL API', async ({ page, request }) => {
+test('E03 persist A,A,B history, edit, pause, enable and delete through the real browser and PostgreSQL API', async ({ page }) => {
   await page.goto('/monitors');
   const created: MonitorView[] = [];
   for (const input of scenario.monitors) {
@@ -63,9 +67,77 @@ test('E03 persist A,A,B history, edit, pause, enable and delete through the real
   await a.getByRole('button', { name: 'View history', exact: true }).click();
   await expect(a.locator('tbody tr')).toHaveCount(2);
   for (const check of checks.slice(0, 2)) await expect(a.locator('tbody')).toContainText(check.id);
-  const removed = await request.get(`/api/checks/${checks[2].id}`);
+  const removed = await page.request.get(`/api/checks/${checks[2].id}`);
   expect(removed.status()).toBe(404);
   expect((await removed.json()).error.code).toBe('NOT_FOUND');
   await expect(page.getByRole('main').getByRole('alert')).toHaveCount(0);
   await page.screenshot({ path: 'output/playwright/E03-lifecycle.png', fullPage: true });
 });
+
+test.describe('E04 real browser session lifecycle', () => {
+  test.use({ autoLogin: false });
+
+  test('anonymous redirect, wrong login, Alice login/reload/logout, revoked cookie, and Bob login', async ({ page, context, users }) => {
+    const started = performance.now();
+    const cookieName = authScenario.session.cookie.name;
+    await page.goto('/monitors');
+    await expect(page).toHaveURL(`${authScenario.browserOrigin}/login`);
+    await expect(page.getByRole('heading', { name: 'Sign in', exact: true })).toBeVisible();
+
+    const wrongResponse = page.waitForResponse((response) => response.url().endsWith('/api/auth/login') && response.request().method() === 'POST');
+    await submitCredentials(page, { username: users[0].username, password: randomBytes(32).toString('base64url') });
+    const wrongStatus = (await wrongResponse).status();
+    expect(wrongStatus).toBe(401);
+    const alert = page.getByRole('main').getByRole('alert');
+    await expect(alert).toHaveAttribute('data-error-code', 'UNAUTHENTICATED');
+    await expect(alert).toHaveText('Username or password was not accepted.');
+    expect(await page.getByLabel('Password', { exact: true }).evaluate((input: HTMLInputElement) => input.value.length === 0)).toBe(true);
+
+    const aliceResponse = page.waitForResponse((response) => response.url().endsWith('/api/auth/login') && response.request().method() === 'POST');
+    await submitCredentials(page, users[0]);
+    const aliceStatus = (await aliceResponse).status();
+    expect(aliceStatus).toBe(200);
+    await expect(page).toHaveURL(`${authScenario.browserOrigin}/monitors`);
+    await expect(page.getByRole('button', { name: 'Sign out', exact: true })).toBeVisible();
+    const cookies = (await context.cookies()).filter((cookie) => cookie.name === cookieName);
+    expect(cookies.length).toBe(1);
+    const cookie = cookies[0];
+    const cookieFlags = { httpOnly: cookie.httpOnly, sameSite: cookie.sameSite, secure: cookie.secure, path: cookie.path, hostOnly: cookie.domain === '127.0.0.1' };
+    expect(cookieFlags).toEqual({ httpOnly: true, sameSite: 'Lax', secure: false, path: '/', hostOnly: true });
+    expect(await page.evaluate((name) => !document.cookie.includes(`${name}=`), cookieName)).toBe(true);
+    expect(await page.evaluate(() => [...Object.keys(localStorage), ...Object.keys(sessionStorage)].filter((key) => /password|credential|session|token|auth/i.test(key)).length)).toBe(0);
+    await page.reload();
+    await expect(page.getByRole('button', { name: 'Sign out', exact: true })).toBeVisible();
+
+    const logoutResponse = page.waitForResponse((response) => response.url().endsWith('/api/auth/logout') && response.request().method() === 'POST');
+    await page.getByRole('button', { name: 'Sign out', exact: true }).click();
+    const logoutStatus = (await logoutResponse).status();
+    expect(logoutStatus).toBe(200);
+    await expect(page).toHaveURL(`${authScenario.browserOrigin}/login`);
+    expect((await context.cookies()).filter((item) => item.name === cookieName).length).toBe(0);
+    const revoked = await page.request.get('/api/monitors', { headers: { cookie: `${cookieName}=${cookie.value}` } });
+    expect(revoked.status()).toBe(401);
+    expect((await revoked.json()).error.code).toBe('UNAUTHENTICATED');
+    await page.goto('/monitors');
+    await expect(page).toHaveURL(`${authScenario.browserOrigin}/login`);
+
+    const bobResponse = page.waitForResponse((response) => response.url().endsWith('/api/auth/login') && response.request().method() === 'POST');
+    await submitCredentials(page, users[1]);
+    const response = await bobResponse;
+    expect(response.status()).toBe(200);
+    expect((await response.json()).data.user.username).toBe(users[1].username);
+    await expect(page).toHaveURL(`${authScenario.browserOrigin}/monitors`);
+    await expect(page.getByRole('button', { name: 'Sign out', exact: true })).toBeVisible();
+    const bobSession = await page.request.get('/api/auth/session');
+    expect(bobSession.status()).toBe(200);
+    expect((await bobSession.json()).data.user.username).toBe(users[1].username);
+    await expect(page.getByRole('main').getByRole('alert')).toHaveCount(0);
+    await mkdir('output/e04', { recursive: true });
+    await writeFile('output/e04/browser.json', JSON.stringify({
+      wrongLoginStatus: wrongStatus, aliceLoginStatus: aliceStatus, reloadAuthenticated: true,
+      cookieFlags, cookieInvisibleToJavaScript: true, credentialStorageEntries: 0,
+      logoutStatus, logoutClearedCookie: true, revokedStatus: revoked.status(), bobLoginStatus: response.status(),
+      durationMs: Math.round(performance.now() - started),
+    }, null, 2) + '\n');
+  });
+});
diff --git a/test/browser/monitor.spec.ts b/test/browser/monitor.spec.ts
index 98dfe49..4e5973f 100644
--- a/test/browser/monitor.spec.ts
+++ b/test/browser/monitor.spec.ts
@@ -1,4 +1,5 @@
-import { expect, test } from '@playwright/test';
+import { expect } from '@playwright/test';
+import { test } from './session';
 
 test('create Fixture monitor, run one manual check, and display the terminal result', async ({ page }) => {
   await page.goto('/monitors');
diff --git a/test/browser/session.ts b/test/browser/session.ts
new file mode 100644
index 0000000..8a33d81
--- /dev/null
+++ b/test/browser/session.ts
@@ -0,0 +1,29 @@
+import { test as base } from '@playwright/test';
+import type { Page } from '@playwright/test';
+import { prepareTestUsers } from '../auth.ts';
+import type { TestCredentials } from '../auth.ts';
+import { testDatabaseConfig } from '../database.ts';
+
+export const test = base.extend<{ autoLogin: boolean; authenticate: void }, { users: TestCredentials[] }>({
+  autoLogin: [true, { option: true }],
+  users: [async ({}, use) => { await use(await prepareTestUsers(testDatabaseConfig('e03_browser'))); }, { scope: 'worker' }],
+  authenticate: [async ({ autoLogin, page, users }, use) => {
+    if (autoLogin) {
+      try {
+        const response = await page.request.post('/api/auth/login', { data: users[0] });
+        if (response.status() !== 200) throw new Error('Login rejected.');
+      } catch { throw new Error('Browser authentication fixture failed; credential details are suppressed.'); }
+    }
+    await use();
+  }, { auto: true }],
+});
+
+export async function submitCredentials(page: Page, credentials: TestCredentials) {
+  // Disable trace recording in the config and suppress action-error payloads so
+  // browser verification cannot publish a typed password on a failing run.
+  try {
+    await page.getByLabel('Username', { exact: true }).fill(credentials.username);
+    await page.getByLabel('Password', { exact: true }).fill(credentials.password);
+    await page.getByRole('button', { name: 'Sign in', exact: true }).click();
+  } catch { throw new Error('Browser sign-in action failed; credential details are suppressed.'); }
+}


