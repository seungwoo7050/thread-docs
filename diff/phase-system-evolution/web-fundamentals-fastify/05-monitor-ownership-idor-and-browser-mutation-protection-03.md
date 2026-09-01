## `Require session-bound CSRF evidence for browser mutations`

diff --git a/TRACK.md b/TRACK.md
index 514358e..889d976 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -335,3 +335,44 @@ Monitors is restricted by the foreign key. Migrations 001–003 stay unchanged.
 The E05 frozen two-user dataset and one unchanged-START read are in `evidence/E05/`.
 The regression compares all authoritative Monitor/CheckRun rows and fixture call
 counts after every foreign mutation, and tests both directions of access.
+
+### Browser mutation protection
+
+`GET /auth/csrf` is authenticated and returns `{data: {csrfToken}}` with
+`Cache-Control: no-store`. Its token is an HMAC-SHA256 of the fixed context
+`wse-csrf-v1`, keyed by that session's random HttpOnly identifier. It is distinct
+from both the cookie and the stored authentication hash. A token from another
+session cannot authorize a mutation, and a CSRF token alone cannot authenticate.
+Rotation changes the token; expiry and revocation still consult PostgreSQL first.
+The server bounds the supplied header and uses a constant-time comparison.
+
+Every authenticated state-changing request, including logout, requires both
+`X-CSRF-Token` and an exact `Origin` equal to `BROWSER_ORIGIN` (default
+`http://127.0.0.1:4313`). Missing, incorrect or another session's CSRF evidence,
+and missing, `null` or foreign Origin, return `403 FORBIDDEN` before parsing or
+product SQL. Anonymous requests still return `401 UNAUTHENTICATED` first. Origin
+comparison never trusts `Host`, `X-Forwarded-Host` or a substring. An explicit
+foreign Origin on an authenticated read is also refused. Login requires that
+same exact Origin before processing credentials; this is the pre-session login
+CSRF policy. Nonbrowser clients must supply the configured Origin too.
+
+The browser obtains its CSRF evidence immediately before each mutation and keeps
+it only within that request, not React state, URLs, cookies or browser storage.
+The session identifier remains HttpOnly. The additional GET is deliberately
+simple; there is no server-state cache or E06 restructuring.
+
+CORS is separately **deny all cross-origin API access**: no
+`Access-Control-Allow-Origin` or `Access-Control-Allow-Credentials` is emitted,
+and all OPTIONS preflights return 403. The browser uses the existing same-origin
+Next proxy, so it does not need CORS permission even though the proxy's backend
+uses a different port. CORS controls whether a browser can read a cross-origin
+response; it does not substitute for checking a mutation's CSRF token and Origin.
+The layered policy follows [OWASP's CSRF guidance](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html).
+
+E05 adds 403 to the established error-code/status mapping without interpreting
+server prose. API tests cover all seven mutation operations against six rejected
+CSRF/Origin combinations, login Origin rejection, no-write/no-outbound assertions,
+rotation and logout. Chromium retains the E01–E04 lifecycle tests and adds two
+independent logged-in browser contexts, missing-CSRF update/logout and an actual
+blocked cross-origin preflight. Traces/videos remain disabled and test evidence
+contains only statuses, counts and booleans; no credential values are serialized.
diff --git a/app/monitors/api.ts b/app/monitors/api.ts
index 48f47ff..88a2d19 100644
--- a/app/monitors/api.ts
+++ b/app/monitors/api.ts
@@ -3,10 +3,11 @@ import type { ApiErrorCode } from '../../server/model.ts';
 export const ERROR_MESSAGES: Record<ApiErrorCode, string> = {
   INVALID_INPUT: 'Check the monitor input and try again.',
   UNAUTHENTICATED: 'Sign in to continue.',
+  FORBIDDEN: 'The request was not permitted. Reload and try again.',
   NOT_FOUND: 'The requested monitor or resource was not found.',
   INTERNAL_ERROR: 'The monitoring service could not complete the request.',
 };
-const ERROR_STATUS = { INVALID_INPUT: 400, UNAUTHENTICATED: 401, NOT_FOUND: 404, INTERNAL_ERROR: 500 } as const;
+const ERROR_STATUS = { INVALID_INPUT: 400, UNAUTHENTICATED: 401, FORBIDDEN: 403, NOT_FOUND: 404, INTERNAL_ERROR: 500 } as const;
 
 export class RequestFailure extends Error {
   readonly code: ApiErrorCode;
@@ -30,7 +31,7 @@ export async function responseData<T>(response: Response): Promise<T> {
   if (!response.ok) {
     if (isObject(body) && isObject(body.error) && typeof body.error.message === 'string') {
       const code = body.error.code;
-      if ((code === 'INVALID_INPUT' || code === 'UNAUTHENTICATED' || code === 'NOT_FOUND' || code === 'INTERNAL_ERROR') && ERROR_STATUS[code] === response.status) {
+      if ((code === 'INVALID_INPUT' || code === 'UNAUTHENTICATED' || code === 'FORBIDDEN' || code === 'NOT_FOUND' || code === 'INTERNAL_ERROR') && ERROR_STATUS[code] === response.status) {
         throw new RequestFailure(code);
       }
     }
@@ -45,3 +46,13 @@ export async function responseData<T>(response: Response): Promise<T> {
 export function failureCode(failure: unknown): ApiErrorCode {
   return failure instanceof RequestFailure ? failure.code : 'INTERNAL_ERROR';
 }
+
+// Obtain CSRF evidence for the current cookie immediately before a mutation.
+// Keep it only in this request's scope, never in React state or browser storage.
+export async function mutationFetch(path: string, options: RequestInit): Promise<Response> {
+  const data = await responseData<{ csrfToken: string }>(await fetch('/api/auth/csrf', { credentials: 'same-origin' }));
+  if (typeof data?.csrfToken !== 'string' || !/^[A-Za-z0-9_-]{43}$/.test(data.csrfToken)) throw new RequestFailure('INTERNAL_ERROR');
+  const headers = new Headers(options.headers);
+  headers.set('x-csrf-token', data.csrfToken);
+  return fetch(path, { ...options, credentials: 'same-origin', headers });
+}
diff --git a/app/monitors/page.tsx b/app/monitors/page.tsx
index 4550383..1ae2710 100644
--- a/app/monitors/page.tsx
+++ b/app/monitors/page.tsx
@@ -4,7 +4,7 @@ import { useCallback, useEffect, useState } from 'react';
 import type { FormEvent } from 'react';
 import { useRouter } from 'next/navigation';
 import type { ApiErrorCode, CheckRun, MonitorView } from '../../server/model';
-import { ERROR_MESSAGES, failureCode, responseData } from './api';
+import { ERROR_MESSAGES, failureCode, mutationFetch, responseData } from './api';
 
 export default function MonitorsPage() {
   const router = useRouter();
@@ -45,7 +45,7 @@ export default function MonitorsPage() {
     setLoggingOut(true);
     setError(null);
     try {
-      await responseData(await fetch('/api/auth/logout', { method: 'POST', credentials: 'same-origin' }));
+      await responseData(await mutationFetch('/api/auth/logout', { method: 'POST' }));
       setAuthenticated(false);
       setMonitors([]);
       setHistory([]);
@@ -62,7 +62,7 @@ export default function MonitorsPage() {
     setCreating(true);
     setError(null);
     try {
-      const response = await fetch('/api/monitors', {
+      const response = await mutationFetch('/api/monitors', {
         method: 'POST', credentials: 'same-origin', headers: { 'content-type': 'application/json' },
         body: JSON.stringify({
           name: fields.get('name'), url: fields.get('url'),
@@ -81,7 +81,7 @@ export default function MonitorsPage() {
     setChecking(id);
     setError(null);
     try {
-      const response = await fetch(`/api/monitors/${id}/checks`, { method: 'POST', credentials: 'same-origin' });
+      const response = await mutationFetch(`/api/monitors/${id}/checks`, { method: 'POST' });
       const result = await responseData<CheckRun>(response);
       setMonitors((current) => current.map((monitor) => monitor.id === id ? { ...monitor, latestCheck: result } : monitor));
       if (historyId === id) await showHistory(id);
@@ -94,7 +94,7 @@ export default function MonitorsPage() {
     setSaving(monitor.id);
     setError(null);
     try {
-      const response = await fetch(`/api/monitors/${monitor.id}`, {
+      const response = await mutationFetch(`/api/monitors/${monitor.id}`, {
         method: 'PUT', credentials: 'same-origin', headers: { 'content-type': 'application/json' }, body: JSON.stringify(input),
       });
       const saved = await responseData<MonitorView>(response);
@@ -118,7 +118,7 @@ export default function MonitorsPage() {
     setDeleting(id);
     setError(null);
     try {
-      const response = await fetch(`/api/monitors/${id}`, { method: 'DELETE', credentials: 'same-origin' });
+      const response = await mutationFetch(`/api/monitors/${id}`, { method: 'DELETE' });
       await responseData<{ id: string }>(response);
       setMonitors((current) => current.filter((monitor) => monitor.id !== id));
       if (historyId === id) { setHistoryId(null); setHistory([]); }
diff --git a/server/auth.ts b/server/auth.ts
index b2e25bf..c3e3a4a 100644
--- a/server/auth.ts
+++ b/server/auth.ts
@@ -1,4 +1,4 @@
-import { createHash, randomBytes } from 'node:crypto';
+import { createHash, createHmac, randomBytes, timingSafeEqual } from 'node:crypto';
 import type { FastifyInstance } from 'fastify';
 import type { Pool } from 'pg';
 import { ApiError, loginInput } from './contracts.ts';
@@ -13,6 +13,7 @@ declare module 'fastify' {
 
 export const SESSION_TTL_MS = 3_600_000;
 export const SESSION_COOKIE_NAME = 'wse_session';
+export const DEFAULT_BROWSER_ORIGIN = 'http://127.0.0.1:4313';
 // This application only serves loopback HTTP. Secure is intentionally absent for
 // that transport; no public/production HTTP deployment is supported.
 const COOKIE_ATTRIBUTES = 'Path=/; HttpOnly; SameSite=Lax';
@@ -33,14 +34,38 @@ export function sessionTokenHash(token: string): string {
 }
 
 function unauthenticated() { return new ApiError('UNAUTHENTICATED', 'Authentication is required.'); }
+function forbidden() { return new ApiError('FORBIDDEN', 'Request is not permitted.'); }
+
+// The random HttpOnly session identifier is the per-session secret. This
+// domain-separated HMAC is not the identifier or its stored authentication hash.
+// It changes on rotation and cannot authenticate without the live DB session.
+export function csrfTokenForSession(sessionToken: string): string {
+  return createHmac('sha256', sessionToken).update('wse-csrf-v1').digest('base64url');
+}
+
+export function validCsrfToken(value: unknown, sessionToken: string): boolean {
+  return typeof value === 'string' && /^[A-Za-z0-9_-]{43}$/.test(value) &&
+    timingSafeEqual(Buffer.from(value), Buffer.from(csrfTokenForSession(sessionToken)));
+}
 
 export function registerAuthentication(app: FastifyInstance, pool: Pool, now: () => number) {
+  const browserOrigin = process.env.BROWSER_ORIGIN ?? DEFAULT_BROWSER_ORIGIN;
+  if (new URL(browserOrigin).origin !== browserOrigin) throw new Error('BROWSER_ORIGIN must be one exact origin.');
   app.decorateRequest('user', null);
   app.addHook('onRequest', async (request, reply) => {
     reply.header('cache-control', 'no-store');
+    // CORS policy: no cross-origin API clients. The browser uses Next's
+    // same-origin proxy. Never grant ACAO/ACAC; deny every preflight explicitly.
+    // This only controls browser response access; mutations still check CSRF.
+    if (request.method === 'OPTIONS') throw forbidden();
     const route = request.routeOptions.url;
-    if ((route === '/health' && (request.method === 'GET' || request.method === 'HEAD')) ||
-      (route === '/auth/login' && request.method === 'POST')) return;
+    if (route === '/health' && (request.method === 'GET' || request.method === 'HEAD')) return;
+    if (route === '/auth/login' && request.method === 'POST') {
+      // Login has no authenticated session yet: reject missing/null/foreign
+      // Origin before credentials or session replacement are processed.
+      if (request.headers.origin !== browserOrigin) throw forbidden();
+      return;
+    }
 
     const token = sessionTokenFromCookie(request.headers.cookie);
     const result = token ? await pool.query<Pick<UserRow, 'id' | 'username'>>(`
@@ -51,6 +76,10 @@ export function registerAuthentication(app: FastifyInstance, pool: Pool, now: ()
       throw unauthenticated();
     }
     request.user = userFromRow(result.rows[0]);
+    if (request.headers.origin !== undefined && request.headers.origin !== browserOrigin) throw forbidden();
+    if (request.method !== 'GET' && request.method !== 'HEAD') {
+      if (request.headers.origin !== browserOrigin || !validCsrfToken(request.headers['x-csrf-token'], token!)) throw forbidden();
+    }
   });
 
   app.post<{ Body: unknown }>('/auth/login', async (request, reply) => {
@@ -80,6 +109,10 @@ export function registerAuthentication(app: FastifyInstance, pool: Pool, now: ()
 
   app.get('/auth/session', async (request) => ({ data: { user: request.user } }));
 
+  app.get('/auth/csrf', async (request) => ({
+    data: { csrfToken: csrfTokenForSession(sessionTokenFromCookie(request.headers.cookie)!) },
+  }));
+
   app.post('/auth/logout', async (request, reply) => {
     const token = sessionTokenFromCookie(request.headers.cookie)!; // onRequest authenticated this exact cookie.
     await pool.query('DELETE FROM sessions WHERE token_hash = $1', [sessionTokenHash(token)]);
diff --git a/server/contracts.ts b/server/contracts.ts
index 3c0c5a9..541fd11 100644
--- a/server/contracts.ts
+++ b/server/contracts.ts
@@ -1,7 +1,7 @@
 import { fixtureUrl } from './check.ts';
 import type { ApiErrorCode, Monitor } from './model.ts';
 
-export const ERROR_STATUS = { INVALID_INPUT: 400, UNAUTHENTICATED: 401, NOT_FOUND: 404, INTERNAL_ERROR: 500 } as const;
+export const ERROR_STATUS = { INVALID_INPUT: 400, UNAUTHENTICATED: 401, FORBIDDEN: 403, NOT_FOUND: 404, INTERNAL_ERROR: 500 } as const;
 
 export class ApiError extends Error {
   readonly code: ApiErrorCode;
diff --git a/server/model.ts b/server/model.ts
index 51fa3af..569c24c 100644
--- a/server/model.ts
+++ b/server/model.ts
@@ -24,6 +24,6 @@ export type MonitorView = Monitor & { latestCheck: CheckRun | null };
 
 export type User = { id: string; username: string };
 
-export type ApiErrorCode = 'INVALID_INPUT' | 'UNAUTHENTICATED' | 'NOT_FOUND' | 'INTERNAL_ERROR';
+export type ApiErrorCode = 'INVALID_INPUT' | 'UNAUTHENTICATED' | 'FORBIDDEN' | 'NOT_FOUND' | 'INTERNAL_ERROR';
 export type ApiSuccess<T> = { data: T };
 export type ApiFailure = { error: { code: ApiErrorCode; message: string } };
diff --git a/test/auth.ts b/test/auth.ts
index cecded1..466ff75 100644
--- a/test/auth.ts
+++ b/test/auth.ts
@@ -3,7 +3,7 @@ import type { FastifyInstance, InjectOptions } from 'fastify';
 import { databasePool } from '../server/database.ts';
 import type { DatabaseConfig } from '../server/database.ts';
 import { hashPassword } from '../server/password.ts';
-import { SESSION_COOKIE_NAME } from '../server/auth.ts';
+import { DEFAULT_BROWSER_ORIGIN, SESSION_COOKIE_NAME } from '../server/auth.ts';
 import scenario from '../evidence/E04/scenario.json' with { type: 'json' };
 
 export type TestCredentials = { username: string; password: string };
@@ -33,22 +33,36 @@ export function cookieFromHeader(header: string | string[] | undefined): string
 }
 
 export async function loginForTest(app: FastifyInstance, credentials: TestCredentials): Promise<string> {
-  const response = await app.inject({ method: 'POST', url: '/auth/login', payload: credentials });
+  const response = await app.inject({ method: 'POST', url: '/auth/login', headers: { origin: DEFAULT_BROWSER_ORIGIN }, payload: credentials });
   if (response.statusCode !== 200) throw new Error(`Fixture login failed with status ${response.statusCode}.`);
   return cookieFromHeader(response.headers['set-cookie']);
 }
 
+export async function csrfForTest(app: FastifyInstance, cookie: string): Promise<string> {
+  const response = await app.inject({ url: '/auth/csrf', headers: { cookie } });
+  if (response.statusCode !== 200 || typeof response.json().data.csrfToken !== 'string') throw new Error('CSRF fixture preparation failed.');
+  return response.json().data.csrfToken;
+}
+
 export function authenticatedInject(app: FastifyInstance, cookie: string) {
-  return (input: string | InjectOptions) => {
+  return async (input: string | InjectOptions) => {
     const options = typeof input === 'string' ? { url: input } : input;
-    return app.inject({ ...options, headers: { ...options.headers, cookie } });
+    const unsafe = options.method && !['GET', 'HEAD'].includes(options.method);
+    const csrf = unsafe ? { 'x-csrf-token': await csrfForTest(app, cookie) } : {};
+    return app.inject({ ...options, headers: { origin: DEFAULT_BROWSER_ORIGIN, ...csrf, ...options.headers, cookie } });
   };
 }
 
 export function authenticatedFetch(cookie: string) {
-  return (url: string, options: RequestInit = {}) => {
+  return async (url: string, options: RequestInit = {}) => {
     const headers = new Headers(options.headers);
     headers.set('cookie', cookie);
+    headers.set('origin', DEFAULT_BROWSER_ORIGIN);
+    if (options.method && !['GET', 'HEAD'].includes(options.method)) {
+      const response = await fetch(new URL('/auth/csrf', url), { headers: { cookie } });
+      if (response.status !== 200) throw new Error('CSRF fixture preparation failed.');
+      headers.set('x-csrf-token', (await response.json()).data.csrfToken);
+    }
     return fetch(url, { ...options, headers });
   };
 }
diff --git a/test/browser/lifecycle.spec.ts b/test/browser/lifecycle.spec.ts
index 90b8a1f..ac630d5 100644
--- a/test/browser/lifecycle.spec.ts
+++ b/test/browser/lifecycle.spec.ts
@@ -5,6 +5,7 @@ import { test, submitCredentials } from './session';
 import type { CheckRun, MonitorView } from '../../server/model';
 import scenario from '../../evidence/E03/scenario.json' with { type: 'json' };
 import authScenario from '../../evidence/E04/scenario.json' with { type: 'json' };
+import ownershipScenario from '../../evidence/E05/scenario.json' with { type: 'json' };
 
 test('E03 persist A,A,B history, edit, pause, enable and delete through the real browser and PostgreSQL API', async ({ page }) => {
   await page.goto('/monitors');
@@ -141,3 +142,90 @@ test.describe('E04 real browser session lifecycle', () => {
     }, null, 2) + '\n');
   });
 });
+
+test.describe('E05 real browser ownership and CSRF', () => {
+  test.use({ autoLogin: false });
+
+  test('independent Alice/Bob sessions isolate A/B and send CSRF evidence on authorized actions', async ({ page, browser, users }) => {
+    const started = performance.now();
+    const bobContext = await browser.newContext({ baseURL: ownershipScenario.browserOrigin });
+    const bob = await bobContext.newPage();
+    const monitors: MonitorView[] = [];
+    const checks: CheckRun[] = [];
+    try {
+      for (const [index, current] of [page, bob].entries()) {
+        await current.goto('/login');
+        await submitCredentials(current, users[index]);
+        await expect(current).toHaveURL(`${ownershipScenario.browserOrigin}/monitors`);
+        const input = ownershipScenario.monitors[index];
+        await current.getByLabel('Name', { exact: true }).fill(input.name);
+        await current.getByLabel('Endpoint URL', { exact: true }).fill(input.url);
+        await current.getByLabel('Interval (seconds)', { exact: true }).fill(String(input.interval));
+        await current.getByLabel('Enabled', { exact: true }).check();
+        const createdResponse = current.waitForResponse((response) => response.url().endsWith('/api/monitors') && response.request().method() === 'POST');
+        await current.getByRole('button', { name: 'Create monitor', exact: true }).click();
+        const created = await createdResponse;
+        expect(created.status()).toBe(201);
+        expect(Boolean(await created.request().headerValue('x-csrf-token'))).toBe(true);
+        expect(await created.request().headerValue('origin')).toBe(ownershipScenario.browserOrigin);
+        monitors.push((await created.json()).data);
+        const article = current.getByRole('article', { name: input.name, exact: true });
+        const checkedResponse = current.waitForResponse((response) => response.url().endsWith(`/api/monitors/${monitors[index].id}/checks`) && response.request().method() === 'POST');
+        await article.getByRole('button', { name: 'Run check', exact: true }).click();
+        const checked = await checkedResponse;
+        expect(checked.status()).toBe(200);
+        expect(Boolean(await checked.request().headerValue('x-csrf-token'))).toBe(true);
+        checks.push((await checked.json()).data);
+      }
+      for (const [index, current] of [page, bob].entries()) {
+        const other = 1 - index;
+        await current.reload();
+        const own = current.getByRole('article', { name: ownershipScenario.monitors[index].name, exact: true });
+        await expect(own).toBeVisible();
+        await expect(current.getByRole('article', { name: ownershipScenario.monitors[other].name, exact: true })).toHaveCount(0);
+        await own.getByRole('button', { name: 'View history', exact: true }).click();
+        await expect(own.locator('tbody tr')).toHaveCount(1);
+        await expect(own.locator('tbody')).toContainText(checks[index].id);
+        for (const path of [`/api/monitors/${monitors[other].id}`, `/api/monitors/${monitors[other].id}/checks`, `/api/checks/${checks[other].id}`]) {
+          const rejected = await current.request.get(path);
+          expect(rejected.status()).toBe(404);
+          expect((await rejected.json()).error.code).toBe('NOT_FOUND');
+        }
+      }
+      const csrfFailure = await page.evaluate(async ({ id, input }) => {
+        const response = await fetch(`/api/monitors/${id}`, {
+          method: 'PUT', credentials: 'same-origin', headers: { 'content-type': 'application/json' }, body: JSON.stringify(input),
+        });
+        return { status: response.status, code: (await response.json()).error.code };
+      }, { id: monitors[0].id, input: ownershipScenario.mutationInputs.update });
+      expect(csrfFailure).toEqual({ status: 403, code: 'FORBIDDEN' });
+      const unsafeLogout = await page.evaluate(async () => {
+        const response = await fetch('/api/auth/logout', { method: 'POST', credentials: 'same-origin' });
+        return response.status;
+      });
+      expect(unsafeLogout).toBe(403);
+      await page.reload();
+      await expect(page.getByRole('article', { name: ownershipScenario.monitors[0].name, exact: true })).toBeVisible();
+      const crossOriginBlocked = await page.evaluate(async (apiOrigin) => {
+        try { await fetch(`${apiOrigin}/monitors`, { credentials: 'include', headers: { 'x-csrf-token': 'not-a-token' } }); return false; }
+        catch { return true; }
+      }, ownershipScenario.apiOrigin);
+      expect(crossOriginBlocked).toBe(true);
+      const logoutResponse = page.waitForResponse((response) => response.url().endsWith('/api/auth/logout'));
+      await page.getByRole('button', { name: 'Sign out', exact: true }).click();
+      const logout = await logoutResponse;
+      expect(logout.status()).toBe(200);
+      expect(Boolean(await logout.request().headerValue('x-csrf-token'))).toBe(true);
+      await expect(page).toHaveURL(`${ownershipScenario.browserOrigin}/login`);
+      await expect(bob.getByRole('article', { name: ownershipScenario.monitors[1].name, exact: true })).toBeVisible();
+      await mkdir('output/e05', { recursive: true });
+      await writeFile('output/e05/browser.json', JSON.stringify({
+        distinctBrowserSessions: 2, ownerCreates: 2, ownerChecks: 2, isolatedCollections: true,
+        ownHistoryReads: 2, foreignReadsRejected: 6, csrfHeadersPresent: true, allowedOriginPresent: true,
+        missingCsrfMutationStatus: 403, missingCsrfLogoutStatus: 403, noUnauthorizedChange: true,
+        browserPreflightBlocked: crossOriginBlocked, authorizedLogoutStatus: logout.status(),
+        durationMs: Math.round(performance.now() - started),
+      }, null, 2) + '\n');
+    } finally { await bobContext.close(); }
+  });
+});
diff --git a/test/browser/session.ts b/test/browser/session.ts
index 8a33d81..af22469 100644
--- a/test/browser/session.ts
+++ b/test/browser/session.ts
@@ -3,6 +3,7 @@ import type { Page } from '@playwright/test';
 import { prepareTestUsers } from '../auth.ts';
 import type { TestCredentials } from '../auth.ts';
 import { testDatabaseConfig } from '../database.ts';
+import { DEFAULT_BROWSER_ORIGIN } from '../../server/auth.ts';
 
 export const test = base.extend<{ autoLogin: boolean; authenticate: void }, { users: TestCredentials[] }>({
   autoLogin: [true, { option: true }],
@@ -10,7 +11,7 @@ export const test = base.extend<{ autoLogin: boolean; authenticate: void }, { us
   authenticate: [async ({ autoLogin, page, users }, use) => {
     if (autoLogin) {
       try {
-        const response = await page.request.post('/api/auth/login', { data: users[0] });
+        const response = await page.request.post('/api/auth/login', { headers: { origin: DEFAULT_BROWSER_ORIGIN }, data: users[0] });
         if (response.status() !== 200) throw new Error('Login rejected.');
       } catch { throw new Error('Browser authentication fixture failed; credential details are suppressed.'); }
     }
diff --git a/test/contracts.test.ts b/test/contracts.test.ts
index e9c921e..18f9eab 100644
--- a/test/contracts.test.ts
+++ b/test/contracts.test.ts
@@ -4,13 +4,14 @@ import { randomBytes } from 'node:crypto';
 import { mkdir, writeFile } from 'node:fs/promises';
 import { buildApp } from '../server/app.ts';
 import { fixtureServer } from './fixture.ts';
-import { authenticatedFetch, cookieFromHeader, loginForTest, prepareTestUsers } from './auth.ts';
+import { authenticatedFetch, authenticatedInject, cookieFromHeader, csrfForTest, loginForTest, prepareTestUsers } from './auth.ts';
 import { SESSION_COOKIE_NAME, sessionTokenFromCookie, sessionTokenHash } from '../server/auth.ts';
 import { databasePool } from '../server/database.ts';
 import { dropTestSchema, resetTestSchema, testDatabaseConfig } from './database.ts';
 import type { ApiErrorCode, ApiSuccess, CheckRun, MonitorView } from '../server/model.ts';
 import scenario from '../evidence/E02/scenario.json' with { type: 'json' };
 import authScenario from '../evidence/E04/scenario.json' with { type: 'json' };
+import ownershipScenario from '../evidence/E05/scenario.json' with { type: 'json' };
 
 const fixture = fixtureServer();
 const database = testDatabaseConfig('e03_contracts');
@@ -187,7 +188,7 @@ test('E04 fixed session sequence: anonymous, login, wrong password, rotation, lo
   };
   const get = (cookie?: string) => sessionApp.inject({ url: '/monitors', headers: cookie ? { cookie } : {} });
   const login = (credentials: { username: string; password: string }, cookie?: string) => sessionApp.inject({
-    method: 'POST', url: '/auth/login', payload: credentials, headers: cookie ? { cookie } : {},
+    method: 'POST', url: '/auth/login', payload: credentials, headers: { origin: authScenario.browserOrigin, ...(cookie ? { cookie } : {}) },
   });
   const acceptLogin = (response: Awaited<ReturnType<typeof sessionApp.inject>>, username: string) => {
     assert.equal(response.statusCode, 200);
@@ -236,7 +237,7 @@ test('E04 fixed session sequence: anonymous, login, wrong password, rotation, lo
     const oldStatus = rejection(await get(firstCookie));
     assert.equal((await get(secondCookie)).statusCode, 200);
     observations.rotation = { changed: true, oldStatus, newStatus: 200 };
-    const logout = await sessionApp.inject({ method: 'POST', url: '/auth/logout', headers: { cookie: secondCookie } });
+    const logout = await authenticatedInject(sessionApp, secondCookie)({ method: 'POST', url: '/auth/logout' });
     assert.equal(logout.statusCode, 200);
     assert.deepEqual(logout.json(), { data: { loggedOut: true } });
     assert.ok(String(logout.headers['set-cookie']).includes('; Max-Age=0'));
@@ -295,3 +296,190 @@ test('E04 fixed session sequence: anonymous, login, wrong password, rotation, lo
     await writeFile('output/e04/sessions.json', JSON.stringify({ ...observations, durationMs: Math.round(performance.now() - started) }, null, 2) + '\n');
   } finally { await sessionApp.close(); await pool.end(); await dropTestSchema(config.schema); }
 });
+
+test('E05 fixed A/B ownership matrix: collection, nested/direct reads and every mutation use the session owner', async () => {
+  const started = performance.now();
+  const config = await resetTestSchema('e05_authorization');
+  const pool = databasePool(config);
+  const ownerApp = buildApp(ownershipScenario.fixtureOrigin, config);
+  const observations: { operation: string; status: number; noWrite?: boolean; noOutbound?: boolean }[] = [];
+  const snapshot = async () => ({
+    monitors: (await pool.query('SELECT * FROM monitors ORDER BY id')).rows,
+    checks: (await pool.query('SELECT * FROM check_runs ORDER BY id')).rows,
+    calls: Object.fromEntries(fixture.calls),
+  });
+  try {
+    const users = await prepareTestUsers(config);
+    const owners: ReturnType<typeof authenticatedInject>[] = [];
+    const cookies: string[] = [];
+    const monitors: MonitorView[] = [];
+    const checks: CheckRun[] = [];
+    for (let i = 0; i < users.length; i++) {
+      cookies.push(await loginForTest(ownerApp, users[i]));
+      const owner = authenticatedInject(ownerApp, cookies[i]);
+      owners.push(owner);
+      const created = await owner({ method: 'POST', url: '/monitors', payload: ownershipScenario.monitors[i] });
+      assert.equal(created.statusCode, 201);
+      monitors.push(created.json().data);
+      const checked = await owner({ method: 'POST', url: `/monitors/${monitors[i].id}/checks` });
+      assert.equal(checked.statusCode, 200);
+      checks.push(checked.json().data);
+    }
+    assert.deepEqual(checks.map(({ state, httpStatus }) => ({ state, httpStatus })), [
+      { state: 'SUCCEEDED', httpStatus: 200 }, { state: 'FAILED', httpStatus: 503 },
+    ]);
+    const assigned = (await pool.query(`SELECT m.name, u.username FROM monitors m
+      JOIN users u ON u.id = m.owner_user_id ORDER BY m.name`)).rows;
+    assert.deepEqual(assigned, ownershipScenario.monitors.map((monitor, i) => ({ name: monitor.name, username: users[i].username })));
+
+    for (let i = 0; i < owners.length; i++) {
+      const owner = owners[i];
+      const other = 1 - i;
+      const foreign = monitors[other];
+      const missing = ownershipScenario.missingId;
+      const list = await owner('/monitors');
+      assert.equal(list.statusCode, 200);
+      assert.deepEqual(list.json().data.map((item: MonitorView) => item.id), [monitors[i].id]);
+      assert.equal(list.json().data[0].latestCheck.id, checks[i].id);
+      assert.deepEqual((await owner(`/monitors/${monitors[i].id}`)).json().data, list.json().data[0]);
+      assert.deepEqual((await owner(`/monitors/${monitors[i].id}/checks`)).json().data, [checks[i]]);
+      assert.deepEqual((await owner(`/checks/${checks[i].id}`)).json().data, checks[i]);
+      observations.push({ operation: `${i}: owner collection/detail/history/direct CheckRun`, status: 200 });
+
+      const cases = [
+        { name: 'detail', method: 'GET', path: `/monitors/${foreign.id}`, absent: `/monitors/${missing}` },
+        { name: 'history', method: 'GET', path: `/monitors/${foreign.id}/checks`, absent: `/monitors/${missing}/checks` },
+        { name: 'direct CheckRun', method: 'GET', path: `/checks/${checks[other].id}`, absent: `/checks/${missing}` },
+        ...Object.entries(ownershipScenario.mutationInputs).map(([name, payload]) => ({ name, method: 'PUT', path: `/monitors/${foreign.id}`, absent: `/monitors/${missing}`, payload })),
+        { name: 'delete', method: 'DELETE', path: `/monitors/${foreign.id}`, absent: `/monitors/${missing}` },
+        { name: 'manual check', method: 'POST', path: `/monitors/${foreign.id}/checks`, absent: `/monitors/${missing}/checks` },
+      ];
+      for (const input of cases) {
+        const before = await snapshot();
+        const options = { method: input.method as 'GET' | 'POST' | 'PUT' | 'DELETE', ...('payload' in input ? { payload: input.payload } : {}) };
+        const rejected = await owner({ ...options, url: input.path });
+        const absent = await owner({ ...options, url: input.absent });
+        assert.equal(rejected.statusCode, 404);
+        assert.equal(absent.statusCode, 404);
+        assert.equal(rejected.json().error.code, 'NOT_FOUND');
+        assert.deepEqual(rejected.json(), absent.json());
+        assert.deepEqual(await snapshot(), before, `Foreign ${input.name} must not write or contact the fixture.`);
+        observations.push({ operation: `${i}: foreign/absent ${input.name}`, status: 404, noWrite: true, noOutbound: true });
+      }
+      for (const path of [`/monitors/${foreign.id}`, `/monitors/${foreign.id}/checks`, `/checks/${checks[other].id}`]) {
+        assert.equal((await owner({ method: 'HEAD', url: path })).statusCode, 404);
+      }
+    }
+
+    const csrf = await csrfForTest(ownerApp, cookies[0]);
+    const foreignCsrf = await csrfForTest(ownerApp, cookies[1]);
+    assert.ok(csrf !== foreignCsrf);
+    assert.ok(csrf !== sessionTokenFromCookie(cookies[0]));
+    const beforeSecurity = await snapshot();
+    const sessionCount = (await pool.query('SELECT count(*)::int AS count FROM sessions')).rows[0].count;
+    const mutationCases = [
+      { name: 'create', method: 'POST', url: '/monitors', payload: ownershipScenario.monitors[0] },
+      ...Object.entries(ownershipScenario.mutationInputs).map(([name, payload]) => ({ name, method: 'PUT', url: `/monitors/${monitors[0].id}`, payload })),
+      { name: 'delete', method: 'DELETE', url: `/monitors/${monitors[0].id}` },
+      { name: 'check', method: 'POST', url: `/monitors/${monitors[0].id}/checks` },
+      { name: 'logout', method: 'POST', url: '/auth/logout' },
+    ];
+    const rejectionHeaders = [
+      { name: 'missing CSRF', headers: { origin: ownershipScenario.browserOrigin } },
+      { name: 'incorrect CSRF', headers: { origin: ownershipScenario.browserOrigin, 'x-csrf-token': randomBytes(32).toString('base64url') } },
+      { name: 'other session CSRF', headers: { origin: ownershipScenario.browserOrigin, 'x-csrf-token': foreignCsrf } },
+      { name: 'foreign Origin', headers: { origin: ownershipScenario.foreignOrigin, 'x-csrf-token': csrf } },
+      { name: 'missing Origin', headers: { 'x-csrf-token': csrf } },
+      { name: 'null Origin', headers: { origin: 'null', 'x-csrf-token': csrf } },
+    ];
+    const assertForbidden = (response: Awaited<ReturnType<typeof ownerApp.inject>>) => {
+      assert.equal(response.statusCode, 403);
+      assert.deepEqual(response.json(), { error: { code: 'FORBIDDEN', message: 'Request is not permitted.' } });
+      assert.equal(response.headers['access-control-allow-origin'], undefined);
+      assert.equal(response.headers['access-control-allow-credentials'], undefined);
+    };
+    let csrfRejections = 0;
+    for (const input of mutationCases) {
+      for (const condition of rejectionHeaders) {
+        const response = await ownerApp.inject({
+          method: input.method as 'POST' | 'PUT' | 'DELETE', url: input.url,
+          ...('payload' in input ? { payload: input.payload } : {}),
+          headers: { cookie: cookies[0], ...condition.headers },
+        });
+        assertForbidden(response);
+        assert.deepEqual(await snapshot(), beforeSecurity, `${input.name}/${condition.name} must not write or contact the fixture.`);
+        assert.equal((await pool.query('SELECT count(*)::int AS count FROM sessions')).rows[0].count, sessionCount);
+        assert.equal((await ownerApp.inject({ url: '/auth/session', headers: { cookie: cookies[0] } })).statusCode, 200);
+        csrfRejections++;
+        observations.push({ operation: `${input.name}: ${condition.name}`, status: 403, noWrite: true, noOutbound: true });
+      }
+    }
+    for (const origin of [undefined, 'null', ownershipScenario.foreignOrigin]) {
+      const response = await ownerApp.inject({ method: 'POST', url: '/auth/login', payload: users[1],
+        headers: { cookie: cookies[0], ...(origin === undefined ? {} : { origin }) } });
+      assertForbidden(response);
+      assert.equal((await pool.query('SELECT count(*)::int AS count FROM sessions')).rows[0].count, sessionCount);
+      assert.equal((await ownerApp.inject({ url: '/auth/session', headers: { cookie: cookies[0] } })).json().data.user.username, users[0].username);
+    }
+    for (const [method, path] of [...authScenario.additionalAssertions.protectedRequests, ['GET', '/auth/csrf']]) {
+      const response = await ownerApp.inject({ method: method as 'GET' | 'POST' | 'PUT' | 'DELETE', url: path.replace(ownershipScenario.missingId, monitors[0].id) });
+      assert.equal(response.statusCode, 401);
+      assert.equal(response.json().error.code, 'UNAUTHENTICATED');
+    }
+    for (const origin of [ownershipScenario.foreignOrigin, ownershipScenario.browserOrigin]) {
+      const preflight = await ownerApp.inject({ method: 'OPTIONS', url: '/monitors',
+        headers: { origin, 'access-control-request-method': 'POST', 'access-control-request-headers': 'content-type,x-csrf-token' } });
+      assertForbidden(preflight);
+      assert.equal(preflight.headers['access-control-allow-methods'], undefined);
+      assert.equal(preflight.headers['access-control-allow-headers'], undefined);
+    }
+    assertForbidden(await ownerApp.inject({ url: '/monitors', headers: { cookie: cookies[0], origin: ownershipScenario.foreignOrigin } }));
+    const allowedRead = await owners[0]('/auth/csrf');
+    assert.equal(allowedRead.statusCode, 200);
+    assert.equal(allowedRead.headers['cache-control'], 'no-store');
+    assert.equal(allowedRead.headers['access-control-allow-origin'], undefined);
+    assert.equal(allowedRead.headers['access-control-allow-credentials'], undefined);
+
+    const oldCookie = cookies[0];
+    const rotated = await ownerApp.inject({ method: 'POST', url: '/auth/login', payload: users[0],
+      headers: { origin: ownershipScenario.browserOrigin, cookie: oldCookie } });
+    assert.equal(rotated.statusCode, 200);
+    cookies[0] = cookieFromHeader(rotated.headers['set-cookie']);
+    owners[0] = authenticatedInject(ownerApp, cookies[0]);
+    const rotatedCsrf = await csrfForTest(ownerApp, cookies[0]);
+    assert.ok(rotatedCsrf !== csrf);
+    assert.equal((await ownerApp.inject({ url: '/auth/csrf', headers: { cookie: oldCookie } })).statusCode, 401);
+    assertForbidden(await ownerApp.inject({ method: 'POST', url: '/auth/logout',
+      headers: { cookie: cookies[0], origin: ownershipScenario.browserOrigin, 'x-csrf-token': csrf } }));
+    assert.deepEqual(await snapshot(), beforeSecurity);
+
+    for (const [name, payload] of Object.entries(ownershipScenario.mutationInputs)) {
+      const response = await owners[0]({ method: 'PUT', url: `/monitors/${monitors[0].id}`, payload });
+      assert.equal(response.statusCode, 200);
+      assert.equal(response.json().data.enabled, payload.enabled);
+      assert.equal(response.json().data.name, payload.name);
+      observations.push({ operation: `owner ${name}`, status: 200 });
+    }
+    assert.equal((await owners[1]({ method: 'DELETE', url: `/monitors/${monitors[1].id}` })).statusCode, 200);
+    assert.equal((await pool.query('SELECT id FROM check_runs WHERE monitor_id = $1', [monitors[1].id])).rowCount, 0);
+    assert.equal((await owners[0](`/checks/${checks[0].id}`)).statusCode, 200);
+    assert.equal((await owners[0]({ method: 'POST', url: '/auth/logout' })).statusCode, 200);
+    for (const headers of [
+      { cookie: cookies[0], 'x-csrf-token': rotatedCsrf },
+      { 'x-csrf-token': rotatedCsrf },
+      { cookie: `${SESSION_COOKIE_NAME}=${rotatedCsrf}` },
+    ]) {
+      const response = await ownerApp.inject({ url: '/monitors', headers });
+      assert.equal(response.statusCode, 401);
+      assert.equal(response.json().error.code, 'UNAUTHENTICATED');
+    }
+    await mkdir('output/e05', { recursive: true });
+    await writeFile('output/e05/ownership.json', JSON.stringify({
+      observations, isolatedCollections: true, authoritativeOwnership: true, headForeignStatus: 404,
+      ownerDeleteCascade: true, csrfRejections, loginOriginRejections: 3, deniedPreflights: 2,
+      originCheckedSeparatelyFromCsrf: true, corsPermissionAbsent: true,
+      rotationChangesCsrf: true, oldCsrfRejectedByNewSession: true, logoutRevoked: true, csrfAloneUnauthenticated: true,
+      durationMs: Math.round(performance.now() - started),
+    }, null, 2) + '\n');
+  } finally { await ownerApp.close(); await pool.end(); await dropTestSchema(config.schema); }
+});
diff --git a/test/persistence.test.ts b/test/persistence.test.ts
index 175bab1..7182577 100644
--- a/test/persistence.test.ts
+++ b/test/persistence.test.ts
@@ -10,7 +10,7 @@ import { monitorFromRow, checkRunFromRow, monitorToValues, checkRunToValues } fr
 import type { MonitorRow, CheckRunRow } from '../server/mapping.ts';
 import type { CheckRun, Monitor, MonitorView } from '../server/model.ts';
 import { fixtureServer } from './fixture.ts';
-import { authenticatedInject, loginForTest, prepareTestUsers } from './auth.ts';
+import { authenticatedFetch, authenticatedInject, loginForTest, prepareTestUsers } from './auth.ts';
 import authScenario from '../evidence/E04/scenario.json' with { type: 'json' };
 import { migrate, migrationFiles } from '../server/migrate.ts';
 import ownershipScenario from '../evidence/E05/scenario.json' with { type: 'json' };
@@ -37,7 +37,7 @@ async function failure(response: Response, status: number, code: string) {
 }
 let cookie: string;
 function request(path: string, method = 'GET', body?: unknown) {
-  return fetch(`${scenario.apiOrigin}${path}`, {
+  return authenticatedFetch(cookie)(`${scenario.apiOrigin}${path}`, {
     method, headers: { cookie, ...(body === undefined ? {} : { 'content-type': 'application/json' }) },
     ...(body === undefined ? {} : { body: JSON.stringify(body) }),
   });
diff --git a/test/unit.test.ts b/test/unit.test.ts
index cddca38..2249b80 100644
--- a/test/unit.test.ts
+++ b/test/unit.test.ts
@@ -8,7 +8,7 @@ import scenario from '../evidence/E02/scenario.json' with { type: 'json' };
 import authScenario from '../evidence/E04/scenario.json' with { type: 'json' };
 import { hashPassword, verifyPassword, SCRYPT_OPTIONS } from '../server/password.ts';
 import { loginInput } from '../server/contracts.ts';
-import { SESSION_COOKIE_NAME, SESSION_TTL_MS, sessionTokenFromCookie } from '../server/auth.ts';
+import { csrfTokenForSession, validCsrfToken, SESSION_COOKIE_NAME, SESSION_TTL_MS, sessionTokenFromCookie } from '../server/auth.ts';
 
 test('a path on the configured fixture origin is allowed', () => {
   assert.equal(fixtureUrl(`${DEFAULT_FIXTURE_ORIGIN}/ok`, DEFAULT_FIXTURE_ORIGIN).pathname, '/ok');
@@ -102,3 +102,23 @@ test('E04 browser recognizes UNAUTHENTICATED only with HTTP 401 and ignores serv
   await assert.rejects(responseData(Response.json({ error: { code: 'UNAUTHENTICATED', message: 'ignored' } }, { status: 400 })),
     (error: unknown) => failureCode(error) === 'INTERNAL_ERROR');
 });
+
+test('E05 CSRF evidence is bounded and tied to the current session without exposing the identifier', () => {
+  const session = randomBytes(32).toString('base64url');
+  const otherSession = randomBytes(32).toString('base64url');
+  const csrf = csrfTokenForSession(session);
+  assert.ok(csrf !== session);
+  assert.ok(csrf !== csrfTokenForSession(otherSession));
+  assert.equal(validCsrfToken(csrf, session), true);
+  assert.equal(validCsrfToken(csrf, otherSession), false);
+  for (const value of [undefined, '', [], {}, session, csrf.slice(1), `${csrf},${csrf}`]) {
+    assert.equal(validCsrfToken(value, session), false);
+  }
+});
+
+test('E05 browser recognizes FORBIDDEN only with HTTP 403 and ignores server prose', async () => {
+  await assert.rejects(responseData(Response.json({ error: { code: 'FORBIDDEN', message: 'ignored' } }, { status: 403 })),
+    (error: unknown) => failureCode(error) === 'FORBIDDEN');
+  await assert.rejects(responseData(Response.json({ error: { code: 'FORBIDDEN', message: 'ignored' } }, { status: 401 })),
+    (error: unknown) => failureCode(error) === 'INTERNAL_ERROR');
+});


