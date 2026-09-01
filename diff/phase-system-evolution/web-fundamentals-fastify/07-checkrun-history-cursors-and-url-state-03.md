## `이력 필터와 페이지를 URL에서 복원`

diff --git a/TRACK.md b/TRACK.md
index 7670f38..388e036 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -419,8 +419,8 @@ limits must be decimal integers from **1 through 100**. Optional `state` is
 exactly `SUCCEEDED` or `FAILED`. Empty, repeated, malformed or unsupported
 parameters return `400 INVALID_INPUT`, as do invalid cursors.
 
-Rows retain `finished_at DESC, id DESC` ordering. A query reads at most `limit + 1`
-rows and returns at most `limit`; the extra row determines whether `nextCursor`
+Rows retain `finished_at DESC, id DESC` ordering. The query returns at most `limit + 1`
+rows to the API, which returns at most `limit`; the extra row determines whether `nextCursor`
 is non-null. The next request repeats the same resource, state and limit and
 passes that cursor. A cursor is canonical base64url JSON, bounded to 512
 characters, with version 1, Monitor ID, state, limit, finished timestamp and row
@@ -444,3 +444,25 @@ validate the default/maximum and bad cursor boundaries, traverse all seven IDs,
 read the three FAILED rows, and continue after inserting the fixed eighth newer
 row. Existing E03/E05 assertions still compare the exact historical rows after
 extracting `items` from the new envelope.
+
+The existing `/monitors` route stores history selection in `monitor`, filtering
+in `state`, page size in `limit`, and continuation in `cursor`. For example,
+`/monitors?monitor=<id>&state=FAILED&limit=3` opens the first failed-history page.
+Omitted filter/cursor means all states/the first page. Filter changes remove the
+old cursor; Next page and First page create browser history entries. Direct URL,
+reload and browser back/forward reconstruct the same requested conditions.
+Native `pushState` integrates with the pinned Next `useSearchParams` hook. Its
+required Suspense boundary is local to this URL-consuming page; no routing
+framework or server-rendering redesign is introduced.
+
+The E06 state owner retains at most one requested page per Monitor, identified
+by its query string and version. A different filter/page clears the prior data
+while loading. A late older response cannot restore it, and manual Check
+invalidation refetches the currently requested conditions. Session loss and
+deletion still invalidate pending reads. Browser tests cover URL back/forward,
+reload, exact tied-row IDs and an explicitly held earlier response released
+after changing the filter. The immutable E06 harness assumes its three reloads
+open the bare list before explicitly opening history. Its current test consumer
+therefore removes only E07's four history query parameters before those real
+reloads and restores that wrapper in `finally`. The frozen harness, fixtures,
+mutation/barrier/failure assertions and E07's normal URL reload are unchanged.
diff --git a/app/monitors/page.tsx b/app/monitors/page.tsx
index d3daa15..932f67a 100644
--- a/app/monitors/page.tsx
+++ b/app/monitors/page.tsx
@@ -1,18 +1,51 @@
 'use client';
 
-import { useState } from 'react';
+import { Suspense, useEffect, useState } from 'react';
 import type { FormEvent } from 'react';
+import { useSearchParams } from 'next/navigation';
 import type { MonitorView } from '../../server/model';
 import { ERROR_MESSAGES } from './api';
 import { useMonitors } from './use-monitors';
 
 export default function MonitorsPage() {
+  return <Suspense fallback={<main><h1>Monitors</h1><p>Loading monitors…</p></main>}><Monitors /></Suspense>;
+}
+
+function Monitors() {
   const { authenticated, loading, monitors, histories, mutations, error, mutate, loadHistory } = useMonitors();
   const creating = mutations.create?.status === 'pending';
   const loggingOut = mutations.logout?.status === 'pending';
   const anyPending = Object.values(mutations).some((mutation) => mutation.status === 'pending');
   const [editing, setEditing] = useState<string | null>(null);
-  const [historyId, setHistoryId] = useState<string | null>(null);
+  const searchParams = useSearchParams();
+  const historyId = searchParams.get('monitor');
+  const historyParameters = new URLSearchParams(searchParams.toString());
+  historyParameters.delete('monitor');
+  const historySearch = historyParameters.toString();
+
+  useEffect(() => {
+    if (authenticated && historyId) void loadHistory(historyId, historySearch);
+  }, [authenticated, historyId, historySearch, loadHistory]);
+
+  function navigateHistory(id: string | null, change: { state?: string; cursor?: string | null } = {}) {
+    const next = new URLSearchParams(window.location.search);
+    if (id === null) {
+      for (const key of ['monitor', 'state', 'limit', 'cursor']) next.delete(key);
+    } else {
+      if (id !== next.get('monitor')) next.delete('cursor');
+      next.set('monitor', id);
+      if (change.state !== undefined) {
+        if (change.state) next.set('state', change.state); else next.delete('state');
+        next.delete('cursor');
+      }
+      if (change.cursor !== undefined) {
+        if (change.cursor) next.set('cursor', change.cursor); else next.delete('cursor');
+      }
+    }
+    // Next integrates native history changes with useSearchParams, including
+    // back/forward, while preserving in-flight mutations and local form drafts.
+    window.history.pushState(null, '', `/monitors${next.size ? `?${next}` : ''}`);
+  }
 
   async function createMonitor(event: FormEvent<HTMLFormElement>) {
     event.preventDefault();
@@ -36,16 +69,11 @@ export default function MonitorsPage() {
   async function deleteMonitor(id: string) {
     if (!window.confirm('Delete this monitor and all its check history?')) return;
     if (await mutate({ kind: 'delete', id })) {
-      setHistoryId((current) => current === id ? null : current);
+      if (new URLSearchParams(window.location.search).get('monitor') === id) navigateHistory(null);
       setEditing((current) => current === id ? null : current);
     }
   }
 
-  function showHistory(id: string) {
-    setHistoryId(id);
-    void loadHistory(id);
-  }
-
   if (!authenticated) return <main>
     <h1>Monitors</h1>
     {loading ? <p>Loading monitors…</p> : error && error !== 'UNAUTHENTICATED'
@@ -81,7 +109,7 @@ export default function MonitorsPage() {
       {monitors.map((monitor) => {
         const mutation = mutations[`monitor:${monitor.id}`];
         const pending = mutation?.status === 'pending';
-        const historyQuery = histories[monitor.id];
+        const historyQuery = histories[monitor.id]?.search === historySearch ? histories[monitor.id] : undefined;
         const history = historyQuery?.data?.items ?? [];
         return <article key={monitor.id} className="panel" aria-labelledby={`monitor-${monitor.id}`}>
         <h3 id={`monitor-${monitor.id}`}>{monitor.name}</h3>
@@ -100,7 +128,7 @@ export default function MonitorsPage() {
         <button onClick={() => deleteMonitor(monitor.id)} disabled={pending || loggingOut}>
           {pending && mutation.kind === 'delete' ? 'Deleting…' : 'Delete monitor'}
         </button>
-        <button onClick={() => historyId === monitor.id ? setHistoryId(null) : showHistory(monitor.id)} disabled={loggingOut}>
+        <button onClick={() => navigateHistory(historyId === monitor.id ? null : monitor.id)} disabled={loggingOut}>
           {historyId === monitor.id ? 'Hide history' : 'View history'}
         </button>
         </div>
@@ -124,7 +152,17 @@ export default function MonitorsPage() {
         </dl> : <p>No checks yet.</p>}
         {historyId === monitor.id && <section aria-label={`Check history for ${monitor.name}`}>
           <h4>Check history</h4>
-          {historyQuery?.status === 'pending' ? <p>Loading history…</p> : historyQuery?.data === null ? <p>History could not be loaded.</p> : history.length === 0 ? <p>No historical checks.</p> :
+          <label htmlFor={`history-state-${monitor.id}`}>History state</label>
+          <select id={`history-state-${monitor.id}`} value={searchParams.get('state') ?? ''}
+            onChange={(event) => navigateHistory(monitor.id, { state: event.currentTarget.value })} disabled={loggingOut}>
+            <option value="">All states</option><option value="SUCCEEDED">SUCCEEDED</option><option value="FAILED">FAILED</option>
+          </select>
+          <div className="actions">
+            <button onClick={() => navigateHistory(monitor.id, { cursor: null })} disabled={loggingOut || !searchParams.has('cursor')}>First page</button>
+            <button onClick={() => navigateHistory(monitor.id, { cursor: historyQuery?.data?.nextCursor })}
+              disabled={loggingOut || historyQuery?.status !== 'success' || !historyQuery.data?.nextCursor}>Next page</button>
+          </div>
+          {!historyQuery || historyQuery.status === 'pending' ? <p>Loading history…</p> : historyQuery.data === null ? <p>History could not be loaded.</p> : history.length === 0 ? <p>No historical checks.</p> :
             <div className="history-scroll"><table>
               <thead><tr><th>Check ID</th><th>State</th><th>HTTP status</th><th>Latency</th><th>Failure reason</th><th>Finished</th></tr></thead>
               <tbody>{history.map((check) => <tr key={check.id}>
diff --git a/app/monitors/use-monitors.ts b/app/monitors/use-monitors.ts
index 8415182..4022ad4 100644
--- a/app/monitors/use-monitors.ts
+++ b/app/monitors/use-monitors.ts
@@ -12,7 +12,7 @@ type Mutation =
   | { kind: 'delete' | 'check'; id: string }
   | { kind: 'logout' };
 type MutationState = { kind: Mutation['kind']; status: 'pending' | 'success' | 'error'; error: ApiErrorCode | null };
-type HistoryQuery = { status: 'pending' | 'success' | 'error'; data: CheckHistoryPage | null; error: ApiErrorCode | null };
+type HistoryQuery = { search: string; status: 'pending' | 'success' | 'error'; data: CheckHistoryPage | null; error: ApiErrorCode | null };
 type ServerState = {
   authenticated: boolean;
   loading: boolean;
@@ -24,19 +24,21 @@ type ServerState = {
 
 const emptyState = (): ServerState => ({ authenticated: false, loading: true, monitors: [], histories: {}, mutations: {}, error: null });
 
-// One owner for this route's server state. Forms and the selected panels stay in
-// the component; list/detail share Monitor objects and history is keyed by ID.
+// One owner for this route's server state. Forms stay in the component; history
+// selection/conditions come from the URL. Keep only one page per Monitor.
 export function useMonitors() {
   const router = useRouter();
   const [state, setState] = useState<ServerState>(emptyState);
   const lifetime = useRef(0);
   const pending = useRef(new Set<string>());
   const historyVersions = useRef(new Map<string, number>());
+  const historySearches = useRef(new Map<string, string>());
 
   const clearSession = useCallback(() => {
     lifetime.current++;
     pending.current.clear();
     historyVersions.current.clear();
+    historySearches.current.clear();
     setState({ ...emptyState(), loading: false });
     router.replace('/login');
   }, [router]);
@@ -60,24 +62,26 @@ export function useMonitors() {
     return () => { lifetime.current++; };
   }, [handleFailure]);
 
-  const loadHistory = useCallback(async (id: string, clearError = true) => {
+  const loadHistory = useCallback(async (id: string, search = '', clearError = true) => {
     const generation = lifetime.current;
     const version = (historyVersions.current.get(id) ?? 0) + 1;
     historyVersions.current.set(id, version);
+    historySearches.current.set(id, search);
     setState((current) => ({ ...current, error: clearError ? null : current.error, histories: {
-      ...current.histories, [id]: { status: 'pending', data: current.histories[id]?.data ?? null, error: null },
+      ...current.histories, [id]: { search, status: 'pending',
+        data: current.histories[id]?.search === search ? current.histories[id].data : null, error: null },
     } }));
     const isCurrent = () => generation === lifetime.current && version === historyVersions.current.get(id);
     try {
-      const data = await responseData<CheckHistoryPage>(await fetch(`/api/monitors/${id}/checks`, { credentials: 'same-origin' }));
+      const data = await responseData<CheckHistoryPage>(await fetch(`/api/monitors/${encodeURIComponent(id)}/checks${search ? `?${search}` : ''}`, { credentials: 'same-origin' }));
       if (isCurrent()) setState((current) => ({ ...current, histories: {
-        ...current.histories, [id]: { status: 'success', data, error: null },
+        ...current.histories, [id]: { search, status: 'success', data, error: null },
       } }));
     } catch (failure) {
       if (!isCurrent()) return;
       const error = handleFailure(failure);
       if (isCurrent()) setState((current) => ({ ...current, histories: {
-        ...current.histories, [id]: { status: 'error', data: current.histories[id]?.data ?? null, error },
+        ...current.histories, [id]: { search, status: 'error', data: current.histories[id]?.data ?? null, error },
       } }));
     }
   }, [handleFailure]);
@@ -104,12 +108,14 @@ export function useMonitors() {
       const data = await responseData<MonitorView | CheckRun | { id: string } | { loggedOut: true }>(await mutationFetch(path, options));
       if (generation !== lifetime.current) return false;
       if (action.kind === 'logout') { clearSession(); return true; }
-      const refreshHistory = action.kind === 'check' && historyVersions.current.has(action.id);
+      const historySearch = action.kind === 'check' ? historySearches.current.get(action.id) : undefined;
+      const refreshHistory = historySearch !== undefined;
       if (action.kind === 'delete' || (action.kind === 'check' && refreshHistory)) {
         // A read started before this mutation must not restore deleted or stale
         // history, even if its response arrives after the new authoritative one.
         historyVersions.current.set(action.id, (historyVersions.current.get(action.id) ?? 0) + 1);
       }
+      if (action.kind === 'delete') historySearches.current.delete(action.id);
       setState((current) => {
         let monitors = current.monitors;
         let histories = current.histories;
@@ -128,7 +134,7 @@ export function useMonitors() {
       });
       // Mutation acknowledgement and query refresh are separate states: a failed
       // history read must never label an already committed Check as a failed POST.
-      if (refreshHistory && 'id' in action) void loadHistory(action.id, false);
+      if (refreshHistory && 'id' in action) void loadHistory(action.id, historySearch, false);
       return true;
     } catch (failure) {
       if (generation !== lifetime.current) return false;
diff --git a/test/browser/history.spec.ts b/test/browser/history.spec.ts
new file mode 100644
index 0000000..77777fb
--- /dev/null
+++ b/test/browser/history.spec.ts
@@ -0,0 +1,117 @@
+import { expect } from '@playwright/test';
+import { createHash } from 'node:crypto';
+import { mkdir, readFile, writeFile } from 'node:fs/promises';
+import { test } from './session';
+import { databasePool } from '../../server/database.ts';
+import { testDatabaseConfig } from '../database.ts';
+import { insertHistoryFixture, originalChecks } from '../../evidence/E07/fixture.ts';
+import scenario from '../../evidence/E07/scenario.json' with { type: 'json' };
+
+test('E07 URL restores Monitor, filter and cursor through back/forward and reload', async ({ page }) => {
+  const pool = databasePool(testDatabaseConfig('e03_browser'));
+  try {
+    await insertHistoryFixture(pool);
+    const failedIds = [...originalChecks].reverse().filter((row) => row.state === 'FAILED').map((row) => row.id);
+    const succeededIds = [...originalChecks].reverse().filter((row) => row.state === 'SUCCEEDED').map((row) => row.id);
+    const history = page.getByRole('region', { name: `Check history for ${scenario.monitor.name}` });
+    async function expectRows(ids: string[], state: string, url?: string) {
+      if (url) await expect(page).toHaveURL(url);
+      await expect(history.getByLabel('History state', { exact: true })).toHaveValue(state);
+      await expect(history.locator('tbody tr td:first-child')).toHaveText(ids);
+      expect(new URL(page.url()).searchParams.get('monitor')).toBe(scenario.monitor.id);
+      expect(new URL(page.url()).searchParams.get('limit')).toBe('3');
+    }
+
+    await page.goto(`/monitors?${new URLSearchParams({ monitor: scenario.monitor.id, state: 'FAILED', limit: '3' })}`);
+    await expectRows(failedIds, 'FAILED');
+    const failedUrl = page.url();
+    await expect(history.getByRole('button', { name: 'Next page', exact: true })).toBeDisabled();
+    await history.getByLabel('History state', { exact: true }).selectOption('SUCCEEDED');
+    await expectRows(succeededIds.slice(0, 3), 'SUCCEEDED');
+    const firstUrl = page.url();
+    expect(new URL(firstUrl).searchParams.has('cursor')).toBe(false);
+    await history.getByRole('button', { name: 'Next page', exact: true }).click();
+    await expectRows(succeededIds.slice(3), 'SUCCEEDED');
+    const nextUrl = page.url();
+    expect(new URL(nextUrl).searchParams.get('cursor')!.length).toBeLessThanOrEqual(scenario.page.cursorMaximumLength);
+    await expect(history.getByRole('button', { name: 'Next page', exact: true })).toBeDisabled();
+
+    await page.goBack();
+    await expectRows(succeededIds.slice(0, 3), 'SUCCEEDED', firstUrl);
+    await page.goBack();
+    await expectRows(failedIds, 'FAILED', failedUrl);
+    await page.goForward();
+    await expectRows(succeededIds.slice(0, 3), 'SUCCEEDED', firstUrl);
+    await page.goForward();
+    await expectRows(succeededIds.slice(3), 'SUCCEEDED', nextUrl);
+    await page.reload();
+    await expectRows(succeededIds.slice(3), 'SUCCEEDED', nextUrl);
+    await history.getByRole('button', { name: 'First page', exact: true }).click();
+    await expectRows(succeededIds.slice(0, 3), 'SUCCEEDED', firstUrl);
+    await history.getByRole('button', { name: 'Next page', exact: true }).click();
+    await expectRows(succeededIds.slice(3), 'SUCCEEDED', nextUrl);
+    await history.getByLabel('History state', { exact: true }).selectOption('FAILED');
+    await expectRows(failedIds, 'FAILED', failedUrl);
+    expect(new URL(page.url()).searchParams.has('cursor')).toBe(false);
+
+    const hashes: Record<string, string> = {};
+    for (const file of ['scenario.json', 'fixture.ts']) hashes[file] = createHash('sha256').update(await readFile(`evidence/E07/${file}`)).digest('hex');
+    await mkdir('output/e07', { recursive: true });
+    await writeFile('output/e07/browser.json', JSON.stringify({
+      hashes, result: 'PASS', failedIds, succeededFirstPageIds: succeededIds.slice(0, 3),
+      succeededLastPageIds: succeededIds.slice(3), pageSize: 3,
+      backwardNavigations: 2, forwardNavigations: 2, reloadSamePage: true,
+      directUrlSelection: true, firstPageControl: true, changedFilterResetsCursor: true,
+      automaticRetries: 0, traces: false, videos: false, screenshots: false,
+    }, null, 2) + '\n');
+  } finally {
+    await pool.query('DELETE FROM monitors WHERE id = $1', [scenario.monitor.id]);
+    await pool.end();
+  }
+});
+
+test('E07 delayed history response cannot replace a newer URL filter', async ({ page }) => {
+  const pool = databasePool(testDatabaseConfig('e03_browser'));
+  let signalHeld!: () => void;
+  const held = new Promise<void>((resolve) => { signalHeld = resolve; });
+  let release!: () => void;
+  const released = new Promise<void>((resolve) => { release = resolve; });
+  let heldOnce = false;
+  try {
+    await insertHistoryFixture(pool);
+    await page.route(`**/api/monitors/${scenario.monitor.id}/checks?*`, async (route) => {
+      const url = new URL(route.request().url());
+      if (!heldOnce && !url.searchParams.has('state')) {
+        heldOnce = true;
+        const response = await route.fetch();
+        signalHeld();
+        await released;
+        await route.fulfill({ response });
+      } else await route.continue();
+    });
+    await page.goto(`/monitors?${new URLSearchParams({ monitor: scenario.monitor.id, limit: '3' })}`);
+    await held;
+    const history = page.getByRole('region', { name: `Check history for ${scenario.monitor.name}` });
+    await expect(history.getByText('Loading history…', { exact: true })).toBeVisible();
+    await history.getByLabel('History state', { exact: true }).selectOption('FAILED');
+    const failedIds = [...originalChecks].reverse().filter((row) => row.state === 'FAILED').map((row) => row.id);
+    await expect(history.locator('tbody tr td:first-child')).toHaveText(failedIds);
+    const oldResponse = page.waitForResponse((response) => response.url().includes(`/api/monitors/${scenario.monitor.id}/checks?`) && !new URL(response.url()).searchParams.has('state'));
+    release();
+    await (await oldResponse).finished();
+    // Let the released fetch continuation and its render opportunity complete.
+    await page.evaluate(() => new Promise<void>((resolve) => { requestAnimationFrame(() => resolve()); }));
+    await expect(history.getByLabel('History state', { exact: true })).toHaveValue('FAILED');
+    await expect(history.locator('tbody tr td:first-child')).toHaveText(failedIds);
+    await mkdir('output/e07', { recursive: true });
+    await writeFile('output/e07/stale-query.json', JSON.stringify({
+      result: 'PASS', heldResponses: 1, heldBeforeFilterChange: true, explicitRelease: true,
+      finalFilter: 'FAILED', finalIds: failedIds, staleResponseIgnored: true, automaticRetries: 0,
+    }, null, 2) + '\n');
+  } finally {
+    release();
+    await page.unrouteAll({ behavior: 'wait' });
+    await pool.query('DELETE FROM monitors WHERE id = $1', [scenario.monitor.id]);
+    await pool.end();
+  }
+});
diff --git a/test/browser/server-state.spec.ts b/test/browser/server-state.spec.ts
index 868cfe3..efd6869 100644
--- a/test/browser/server-state.spec.ts
+++ b/test/browser/server-state.spec.ts
@@ -5,10 +5,39 @@ import { test } from './session';
 import { runServerStateScenario } from '../../evidence/E06/browser-scenario';
 
 test('E06 authoritative list/detail refresh, response barrier, duplicate submit and failed mutations', async ({ page }) => {
-  const result = await runServerStateScenario(page, 'verification');
-  expect(result.result).toBe('PASS');
-  const scenarioSha256 = createHash('sha256').update(await readFile('evidence/E06/scenario.json')).digest('hex');
-  const harnessSha256 = createHash('sha256').update(await readFile('evidence/E06/browser-scenario.ts')).digest('hex');
-  await mkdir('output/e06', { recursive: true });
-  await writeFile('output/e06/browser.json', JSON.stringify({ scenarioSha256, harnessSha256, ...result }, null, 2) + '\n');
+  const originalReload = page.reload;
+  const adaptedReloads: string[][] = [];
+  // The frozen E06 harness reloads the bare list and then opens history. E07
+  // persists that panel in the URL, so preserve the old list-route input here
+  // without changing any frozen harness, fixture, barrier or product assertion.
+  // The E07 tests use ordinary query-preserving reload/back/forward unchanged.
+  page.reload = async (options) => {
+    const removed = await page.evaluate(() => {
+      const url = new URL(window.location.href);
+      if (url.pathname !== '/monitors') return null;
+      const parameters = ['monitor', 'state', 'limit', 'cursor'];
+      const removed = parameters.filter((key) => url.searchParams.has(key));
+      for (const key of parameters) url.searchParams.delete(key);
+      window.history.replaceState(null, '', `${url.pathname}${url.search}${url.hash}`);
+      return removed;
+    });
+    if (removed) adaptedReloads.push(removed);
+    return originalReload.call(page, options);
+  };
+  try {
+    const result = await runServerStateScenario(page, 'verification');
+    expect(result.result).toBe('PASS');
+    expect(adaptedReloads).toHaveLength(3);
+    const scenarioSha256 = createHash('sha256').update(await readFile('evidence/E06/scenario.json')).digest('hex');
+    const harnessSha256 = createHash('sha256').update(await readFile('evidence/E06/browser-scenario.ts')).digest('hex');
+    await mkdir('output/e06', { recursive: true });
+    await writeFile('output/e06/browser.json', JSON.stringify({ scenarioSha256, harnessSha256, ...result }, null, 2) + '\n');
+    await mkdir('output/e07', { recursive: true });
+    await writeFile('output/e07/legacy-browser-adapter.json', JSON.stringify({
+      result: 'PASS', scenarioSha256, harnessSha256, adaptedReloadCount: adaptedReloads.length,
+      removedParametersByReload: adaptedReloads, realReloadRetained: true,
+      scope: 'only E06 test/browser/server-state.spec.ts; original method restored in finally',
+      priorFailure: 'The second E07 browser-gate invocation passed nine tests, then E06 timed out at frozen harness line 46 because URL reload now restored the already-open history panel.',
+    }, null, 2) + '\n');
+  } finally { page.reload = originalReload; }
 });


