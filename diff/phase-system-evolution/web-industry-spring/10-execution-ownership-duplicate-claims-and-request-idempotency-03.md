## `feat(ui): adopt preserved E10 manual intent lifecycle`

diff --git a/app/monitors/api.ts b/app/monitors/api.ts
index 0d63620..4ad6487 100644
--- a/app/monitors/api.ts
+++ b/app/monitors/api.ts
@@ -11,17 +11,18 @@ export type InitialMonitorState = {
   monitors: MonitorView[]; history: { selection: HistorySelection; page: HistoryPage } | null;
   loaded: boolean; error: ApiErrorCode | null;
 };
-export type ApiErrorCode = 'INVALID_INPUT' | 'NOT_FOUND' | 'UNAUTHENTICATED' | 'FORBIDDEN' | 'INTERNAL_ERROR';
+export type ApiErrorCode = 'INVALID_INPUT' | 'NOT_FOUND' | 'UNAUTHENTICATED' | 'FORBIDDEN' | 'CONFLICT' | 'INTERNAL_ERROR';
 
 export const errorMessages: Record<ApiErrorCode, string> = {
   INVALID_INPUT: 'Invalid monitor input. Check the name, URL, interval, and enabled value.',
   NOT_FOUND: 'Monitor not found. Reload the list and try again.',
   UNAUTHENTICATED: 'Sign in to continue.',
   FORBIDDEN: 'The request could not be verified. Reload the page and try again.',
+  CONFLICT: 'This request key already belongs to another monitor.',
   INTERNAL_ERROR: 'The service could not complete the request. Try again.',
 };
 const errorStatuses: Record<ApiErrorCode, number> = {
-  INVALID_INPUT: 400, NOT_FOUND: 404, UNAUTHENTICATED: 401, FORBIDDEN: 403, INTERNAL_ERROR: 500,
+  INVALID_INPUT: 400, NOT_FOUND: 404, UNAUTHENTICATED: 401, FORBIDDEN: 403, CONFLICT: 409, INTERNAL_ERROR: 500,
 };
 
 class ApiFailure extends Error {
@@ -76,7 +77,7 @@ async function readData<T>(response: Response, valid: (value: unknown) => value
     if (isObject(body) && isObject(body.error) && typeof body.error.message === 'string') {
       const code = body.error.code;
       if ((code === 'INVALID_INPUT' || code === 'NOT_FOUND' || code === 'UNAUTHENTICATED'
-        || code === 'FORBIDDEN' || code === 'INTERNAL_ERROR')
+        || code === 'FORBIDDEN' || code === 'CONFLICT' || code === 'INTERNAL_ERROR')
         && errorStatuses[code] === response.status) throw new ApiFailure(code);
     }
     throw new ApiFailure('INTERNAL_ERROR');
@@ -96,14 +97,15 @@ export function readMonitors(response: Response): Promise<MonitorView[]> {
     Array.isArray(value) && value.every(isMonitorView));
 }
 
-async function mutation(path: string, method: string, input?: unknown): Promise<Response> {
+async function mutation(path: string, method: string, input?: unknown, key?: string): Promise<Response> {
   // Only the CSRF value is read by JavaScript; the session cookie remains HttpOnly.
   // Fetch a current token so login rotation and logout cannot leave a stale cached token.
   const csrf = await readData(await fetch('/api/session/csrf', { cache: 'no-store' }),
     (value): value is { headerName: string; token: string } => isObject(value)
       && value.headerName === 'X-CSRF-TOKEN' && typeof value.token === 'string');
   return fetch(path, {
-    method, credentials: 'same-origin', headers: { 'Content-Type': 'application/json', [csrf.headerName]: csrf.token },
+    method, credentials: 'same-origin', headers: { 'Content-Type': 'application/json', [csrf.headerName]: csrf.token,
+      ...(key === undefined ? {} : { 'Idempotency-Key': key }) },
     body: input === undefined ? undefined : JSON.stringify(input),
   });
 }
@@ -121,10 +123,10 @@ export async function createMonitor(input: Omit<Monitor, 'id'>): Promise<Monitor
   return readData(await mutation('/api/monitors', 'POST', input), isMonitorView);
 }
 
-export async function runCheck(id: string): Promise<CheckRun> {
-  const response = await mutation(`/api/monitors/${id}/checks`, 'POST');
+export async function runCheck(id: string, key: string): Promise<CheckRun> {
+  const response = await mutation(`/api/monitors/${id}/checks`, 'POST', undefined, key);
   const check = await readData(response, isCheckRun);
-  if (response.status !== 202 || check.state !== 'QUEUED') throw new ApiFailure('INTERNAL_ERROR');
+  if (response.status !== 202) throw new ApiFailure('INTERNAL_ERROR');
   return check;
 }
 
diff --git a/app/monitors/use-monitor-state.ts b/app/monitors/use-monitor-state.ts
index 0dea958..2bdbcf2 100644
--- a/app/monitors/use-monitor-state.ts
+++ b/app/monitors/use-monitor-state.ts
@@ -27,6 +27,7 @@ export function useMonitorState(selection: HistorySelection | null, initial: Ini
   const [loading, setLoading] = useState(!initial.loaded);
   const [historyRead, setHistoryRead] = useState<{ key: string; phase: Phase } | null>(null);
   const inFlight = useRef(new Set<string>());
+  const manualIntents = useRef(new Map<string, string>());
   const { monitorId = null, limit = '20', state = null, cursor = null } = selection ?? {};
   const historyKey = JSON.stringify([monitorId, limit, state, cursor]);
   const historyPage = monitorId ? queries.histories[monitorId]?.[historyKey] : undefined;
@@ -157,7 +158,11 @@ export function useMonitorState(selection: HistorySelection | null, initial: Ini
 
   function check(id: string) {
     return perform(id, async () => {
-      const latestCheck = await runCheck(id);
+      // Until an acknowledgement arrives, another attempt is the same intention.
+      const key = manualIntents.current.get(id) ?? crypto.randomUUID();
+      manualIntents.current.set(id, key);
+      const latestCheck = await runCheck(id, key);
+      manualIntents.current.delete(id);
       setQueries(current => {
         const histories = { ...current.histories };
         // A new run can change several filtered pages. Invalidate this Monitor's
diff --git a/scripts/persistence-scenario.mjs b/scripts/persistence-scenario.mjs
index 24a66fa..5e5fca5 100644
--- a/scripts/persistence-scenario.mjs
+++ b/scripts/persistence-scenario.mjs
@@ -119,6 +119,7 @@ async function request(path, method = 'GET', body, status = 200) {
   const requestStarted = Date.now();
   const response = await fetch(`${base}${path}`, {
     method, headers: { ...(method === 'GET' ? { Cookie: sessionCookie } : await csrfHeaders()),
+      ...(method === 'POST' && path.endsWith('/checks') ? { 'Idempotency-Key': crypto.randomUUID() } : {}),
       ...(body === undefined ? {} : { 'Content-Type': 'application/json' }) },
     body: body === undefined ? undefined : JSON.stringify(body), signal: AbortSignal.timeout(10_000),
   });
diff --git a/tests/browser/ownership.spec.ts b/tests/browser/ownership.spec.ts
index dfd5fe8..09b5898 100644
--- a/tests/browser/ownership.spec.ts
+++ b/tests/browser/ownership.spec.ts
@@ -62,6 +62,7 @@ async function browserApi(page: Page, path: string, method = 'GET', body?: Recor
       headers[csrf.headerName] = csrf.token;
     }
     if (body !== undefined) headers['Content-Type'] = 'application/json';
+    if (method === 'POST' && path.endsWith('/checks')) headers['Idempotency-Key'] = crypto.randomUUID();
     const response = await fetch(path, { method, credentials: 'same-origin', headers,
       body: body === undefined ? undefined : JSON.stringify(body) });
     // Only a product/error response leaves the browser; the CSRF response never does.
diff --git a/tests/browser/server-state.spec.ts b/tests/browser/server-state.spec.ts
index 6afd50a..3b1ada3 100644
--- a/tests/browser/server-state.spec.ts
+++ b/tests/browser/server-state.spec.ts
@@ -209,3 +209,109 @@ test('fixed server-state lifecycle, held create double-submit and independent mu
     writeFileSync(`output/e06/${baseline ? 'baseline' : 'server-state'}.json`, `${JSON.stringify(evidence, null, 2)}\n`);
   }
 });
+
+test('E10 retains one manual intent after an accepted response is lost and creates a new key after acknowledgement', async ({ page }) => {
+  const evidence: Record<string, unknown> = {
+    start: '3cc49f3d2a35055c92d0312fca6167c89dfadec5',
+    fixtureSha256: createHash('sha256').update(readFileSync('evidence/phase-1/E10/fixtures.md')).digest('hex'),
+    result: 'INCOMPLETE',
+  };
+  const existing = await safeRequest(() => page.request.get('/api/monitors'));
+  expect(existing.status()).toBe(200);
+  for (const row of (await existing.json()).data) {
+    const headers = await csrfHeaders(page.request);
+    expect((await safeRequest(() => page.request.delete(`/api/monitors/${row.monitor.id}`, { headers }))).status()).toBe(200);
+  }
+  let aId = '';
+  let bId = '';
+  for (const [name, path, enabled] of [['A', '/ok', true], ['B', '/fail', false]] as const) {
+    const headers = await csrfHeaders(page.request);
+    const created = await safeRequest(() => page.request.post('/api/monitors', { headers,
+      data: { name, url: `http://127.0.0.1:4321${path}`, interval: 60, enabled } }));
+    expect(created.status()).toBe(201);
+    const id = (await created.json()).data.monitor.id as string;
+    expect(id).toMatch(/^[0-9a-f-]{36}$/);
+    if (name === 'A') aId = id;
+    else bId = id;
+  }
+  const persisted = (monitorId: string) => JSON.parse(execFileSync('docker', ['compose', '--project-name', 'wse-industry',
+    '--file', 'compose.yaml', 'exec', '--no-TTY', 'postgres', 'psql', '--username', 'wse_industry', '--dbname', 'monitor',
+    '--tuples-only', '--no-align', '--command', `SELECT coalesce(json_agg(c), '[]') FROM
+      (SELECT id,state FROM e04_browser.check_runs WHERE monitor_id='${monitorId}' ORDER BY queued_at,id) c`],
+  { encoding: 'utf8' })) as { id: string; state: string }[];
+  const accepted = Promise.withResolvers<string>();
+  const discard = Promise.withResolvers<void>();
+  const routes: Promise<void>[] = [];
+  const keys: string[] = [];
+  const path = `/api/monitors/${aId}/checks`;
+  const intercepted = (route: Route) => {
+    if (route.request().method() !== 'POST') return route.continue();
+    const work = (async () => {
+      const key = route.request().headers()['idempotency-key'];
+      expect(typeof key === 'string' && /^[!-~]{1,128}$/.test(key)).toBe(true);
+      const ordinal = keys.push(key);
+      const response = await safeRequest(() => route.fetch());
+      expect(response.status()).toBe(202);
+      if (ordinal === 1) {
+        accepted.resolve((await response.json()).data.id as string);
+        await discard.promise;
+        await route.abort('failed');
+      } else await route.fulfill({ response });
+    })();
+    routes.push(work);
+    void work.catch(error => accepted.reject(error));
+    return work;
+  };
+  await page.route(`**${path}`, intercepted);
+  try {
+    await page.goto('/monitors');
+    const a = page.getByRole('article', { name: 'A', exact: true });
+    const run = a.getByRole('button', { name: 'Run check', exact: true });
+    await run.click();
+    const originalId = await accepted.promise;
+    await expect(run).toBeDisabled();
+    await run.dispatchEvent('click');
+    expect(keys.length).toBe(1);
+    expect(persisted(aId).map(row => row.id)).toEqual([originalId]);
+    evidence.heldAcceptance = { status: 202, clickEvents: 2, requests: keys.length, persistedRows: 1 };
+    discard.resolve();
+    await expect(page.locator('[role="alert"][data-error-code]')).toHaveAttribute('data-error-code', 'INTERNAL_ERROR');
+    await expect(run).toBeEnabled();
+    await expect.poll(() => persisted(aId)[0]?.state).toBe('SUCCEEDED');
+
+    const retried = await uiResponse(page, path, 'POST', () => run.click());
+    expect(retried.status()).toBe(202);
+    const replay = (await retried.json()).data;
+    expect(replay.id).toBe(originalId);
+    expect(replay.state).toBe('SUCCEEDED');
+    expect(keys.length).toBe(2);
+    expect(keys[1] === keys[0]).toBe(true);
+    expect(persisted(aId).map(row => row.id)).toEqual([originalId]);
+    await expect(a.locator('[data-latest-check-id]')).toHaveAttribute('data-latest-check-id', originalId);
+    await expect(a.getByText('SUCCEEDED', { exact: true })).toBeVisible();
+    await expect(page.locator('[role="alert"][data-error-code]')).toHaveCount(0);
+    evidence.retransmission = { responseWasLostAfterAcceptance: true, sameKey: true, sameId: true,
+      status: retried.status(), currentState: replay.state, requests: keys.length, persistedRows: 1, failureAndPendingCleared: true };
+
+    const next = await uiResponse(page, path, 'POST', () => run.click());
+    expect(next.status()).toBe(202);
+    const nextId = (await next.json()).data.id as string;
+    expect(nextId).not.toBe(originalId);
+    expect(keys.length).toBe(3);
+    expect(keys[2] !== keys[0]).toBe(true);
+    expect(persisted(aId).map(row => row.id)).toEqual([originalId, nextId]);
+    expect(persisted(bId)).toHaveLength(0);
+    await expect(a.locator('[data-latest-check-id]')).toHaveAttribute('data-latest-check-id', nextId);
+    await expect(a.getByText('SUCCEEDED', { exact: true })).toBeVisible();
+    evidence.nextIntent = { differentKey: true, differentId: true, status: next.status(), totalRequests: keys.length,
+      persistedRows: 2, otherMonitorRows: 0 };
+    evidence.result = 'PASS';
+  } finally {
+    discard.resolve();
+    const settled = await Promise.allSettled(routes);
+    await page.unroute(`**${path}`, intercepted);
+    evidence.heldRoutesSettled = settled.every(result => result.status === 'fulfilled');
+    mkdirSync('output/phase-1/e10', { recursive: true });
+    writeFileSync('output/phase-1/e10/browser-intent.json', JSON.stringify(evidence, null, 2) + '\n');
+  }
+});


