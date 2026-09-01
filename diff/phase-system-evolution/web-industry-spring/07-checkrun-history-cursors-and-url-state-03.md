## `feat(history): restore filter and page state from URLs`

diff --git a/app/monitors/api.ts b/app/monitors/api.ts
index 6400368..59f86b3 100644
--- a/app/monitors/api.ts
+++ b/app/monitors/api.ts
@@ -4,6 +4,8 @@ export type CheckRun = {
   latencyMs: number; failureReason: 'HTTP_STATUS' | 'CONNECTION_FAILURE' | 'TIMEOUT' | null; finishedAt: string;
 };
 export type MonitorView = { monitor: Monitor; latestCheck: CheckRun | null };
+export type HistorySelection = { monitorId: string; limit: string; state: string | null; cursor: string | null };
+export type HistoryPage = { items: CheckRun[]; nextCursor: string | null };
 export type ApiErrorCode = 'INVALID_INPUT' | 'NOT_FOUND' | 'UNAUTHENTICATED' | 'FORBIDDEN' | 'INTERNAL_ERROR';
 
 export const errorMessages: Record<ApiErrorCode, string> = {
@@ -118,7 +120,12 @@ export async function deleteMonitor(id: string): Promise<null> {
   return readData(await mutation(`/api/monitors/${id}`, 'DELETE'), (value): value is null => value === null);
 }
 
-export async function loadChecks(id: string): Promise<CheckRun[]> {
-  return readData(await fetch(`/api/monitors/${id}/checks`), (value): value is CheckRun[] =>
+export async function loadChecks(selection: HistorySelection): Promise<HistoryPage> {
+  const query = new URLSearchParams({ limit: selection.limit });
+  if (selection.state !== null) query.set('state', selection.state);
+  if (selection.cursor !== null) query.set('cursor', selection.cursor);
+  const response = await fetch(`/api/monitors/${selection.monitorId}/checks?${query}`, { cache: 'no-store' });
+  const items = await readData(response, (value): value is CheckRun[] =>
     Array.isArray(value) && value.every(isCheckRun));
+  return { items, nextCursor: response.headers.get('X-Next-Cursor') };
 }
diff --git a/app/monitors/page.tsx b/app/monitors/page.tsx
index 092653f..3b2240d 100644
--- a/app/monitors/page.tsx
+++ b/app/monitors/page.tsx
@@ -1,6 +1,7 @@
 'use client';
 
-import { useState, type FormEvent } from 'react';
+import { Suspense, useState, type FormEvent } from 'react';
+import { useRouter, useSearchParams } from 'next/navigation';
 import { errorMessages, type Monitor } from './api';
 import { useMonitorState } from './use-monitor-state';
 
@@ -11,9 +12,25 @@ function inputFrom(form: HTMLFormElement): Omit<Monitor, 'id'> {
 }
 
 export default function Monitors() {
-  const state = useMonitorState();
+  return <Suspense fallback={<main><p role="status">Loading monitors…</p></main>}><MonitorContent /></Suspense>;
+}
+
+function MonitorContent() {
+  const router = useRouter();
+  const search = useSearchParams();
+  const historyId = search.get('history');
+  const state = useMonitorState(historyId ? { monitorId: historyId, limit: search.get('limit') ?? '20',
+    state: search.get('state'), cursor: search.get('cursor') } : null);
   const [editingId, setEditingId] = useState<string | null>(null);
-  const [visibleHistories, setVisibleHistories] = useState<Record<string, boolean>>({});
+
+  function historyUrl(changes: Record<string, string | null>, replace = false) {
+    const query = new URLSearchParams(search.toString());
+    for (const [key, value] of Object.entries(changes)) {
+      if (value === null) query.delete(key); else query.set(key, value);
+    }
+    const url = `/monitors${query.size ? `?${query}` : ''}`;
+    if (replace) router.replace(url, { scroll: false }); else router.push(url, { scroll: false });
+  }
 
   async function create(event: FormEvent<HTMLFormElement>) {
     event.preventDefault();
@@ -27,18 +44,14 @@ export default function Monitors() {
 
   async function remove(id: string) {
     if (await state.remove(id)) {
-      setVisibleHistories(current => { const remaining = { ...current }; delete remaining[id]; return remaining; });
+      if (historyId === id) historyUrl({ history: null, state: null, limit: null, cursor: null }, true);
       if (editingId === id) setEditingId(null);
     }
   }
 
-  async function history(id: string) {
-    if (visibleHistories[id]) {
-      setVisibleHistories(current => ({ ...current, [id]: false }));
-      return;
-    }
-    setVisibleHistories(current => ({ ...current, [id]: true }));
-    await state.history(id);
+  function history(id: string) {
+    historyUrl({ history: historyId === id ? null : id, state: null,
+      limit: historyId === id ? null : '3', cursor: null });
   }
 
   return <main>
@@ -65,7 +78,8 @@ export default function Monitors() {
       {state.loaded && state.monitors.length === 0 && <p>No monitors yet.</p>}
       {state.monitors.map(({ monitor, latestCheck }) => {
         const busy = state.pending(monitor.id) || state.pending('logout');
-        const checks = state.histories[monitor.id];
+        const visibleHistory = historyId === monitor.id;
+        const checks = visibleHistory ? state.historyPage?.items : undefined;
         return <article key={monitor.id} aria-label={monitor.name}>
         <h3>{monitor.name}</h3><p className="url">{monitor.url}</p>
         <p>{monitor.interval}s interval · {monitor.enabled ? 'Enabled' : 'Paused'}</p>
@@ -74,7 +88,8 @@ export default function Monitors() {
           <button disabled={busy} onClick={() => setEditingId(monitor.id)}>Edit</button>
           <button disabled={busy} onClick={() => update(monitor.id, { name: monitor.name, url: monitor.url,
             interval: monitor.interval, enabled: !monitor.enabled })}>{monitor.enabled ? 'Pause' : 'Activate'}</button>
-          <button disabled={busy} onClick={() => history(monitor.id)}>{visibleHistories[monitor.id] ? 'Hide history' : 'Show history'}</button>
+          <button disabled={state.operations[monitor.id] === 'pending' || state.pending('logout')}
+            onClick={() => history(monitor.id)}>{visibleHistory ? 'Hide history' : 'Show history'}</button>
           <button disabled={busy} onClick={() => remove(monitor.id)}>Delete</button>
         </div>
         {busy && <p role="status">Request pending…</p>}
@@ -98,9 +113,13 @@ export default function Monitors() {
             {latestCheck.failureReason && <span>{latestCheck.failureReason}</span>}
           </> : <span>No checks yet.</span>}
         </div>
-        {visibleHistories[monitor.id] && <section aria-label="Historical checks">
+        {visibleHistory && <section aria-label="Historical checks">
           <h4>Check history</h4>
-          {checks === undefined ? <p>{busy ? 'Loading history…' : 'History could not be loaded.'}</p>
+          <label>Check result<select value={search.get('state') ?? ''}
+            onChange={event => historyUrl({ state: event.target.value || null, cursor: null })}>
+            <option value="">All</option><option value="SUCCEEDED">SUCCEEDED</option><option value="FAILED">FAILED</option>
+          </select></label>
+          {checks === undefined ? <p>{state.historyPending ? 'Loading history…' : 'History could not be loaded.'}</p>
             : checks.length === 0 ? <p>No historical checks.</p> : <div className="history-table">
             <table aria-label="Check history"><thead><tr><th>Finished (UTC)</th><th>Result</th><th>HTTP</th><th>Latency</th><th>Reason</th></tr></thead>
               <tbody>{checks.map(check => <tr key={check.id} data-check-id={check.id}>
@@ -109,6 +128,8 @@ export default function Monitors() {
               </tr>)}</tbody>
             </table>
           </div>}
+          <button disabled={busy || !state.historyPage?.nextCursor}
+            onClick={() => historyUrl({ cursor: state.historyPage?.nextCursor ?? null })}>Next page</button>
         </section>}
       </article>; })}
     </section>
diff --git a/app/monitors/use-monitor-state.ts b/app/monitors/use-monitor-state.ts
index ed57e05..d31dc43 100644
--- a/app/monitors/use-monitor-state.ts
+++ b/app/monitors/use-monitor-state.ts
@@ -2,21 +2,28 @@
 
 import { useEffect, useRef, useState } from 'react';
 import { createMonitor, deleteMonitor, failureCode, loadChecks, loadMonitors, logout,
-  replaceMonitor, runCheck, type ApiErrorCode, type CheckRun, type Monitor, type MonitorView } from './api';
+  replaceMonitor, runCheck, type ApiErrorCode, type HistoryPage, type HistorySelection, type Monitor, type MonitorView } from './api';
 
-type Queries = { monitors: MonitorView[]; histories: Record<string, CheckRun[]>; loaded: boolean };
+type Queries = { monitors: MonitorView[]; histories: Record<string, Record<string, HistoryPage>>; loaded: boolean };
 type Phase = 'pending' | 'success' | 'failure';
 type Input = Omit<Monitor, 'id'>;
 const emptyQueries = (): Queries => ({ monitors: [], histories: {}, loaded: false });
 
-// One page-scoped owner for remote data and request state. Form drafts and which
-// history is visible stay in the view; no cache survives a session/page boundary.
-export function useMonitorState() {
+// One page-scoped owner for remote data and request state. URL conditions select
+// history pages; no cache survives a session/page boundary.
+export function useMonitorState(selection: HistorySelection | null) {
   const [queries, setQueries] = useState<Queries>(emptyQueries);
   const [operations, setOperations] = useState<Record<string, Phase>>({});
   const [error, setError] = useState<ApiErrorCode | null>(null);
   const [loading, setLoading] = useState(true);
+  const [historyRead, setHistoryRead] = useState<{ key: string; phase: Phase } | null>(null);
   const inFlight = useRef(new Set<string>());
+  const { monitorId = null, limit = '20', state = null, cursor = null } = selection ?? {};
+  const historyKey = JSON.stringify([monitorId, limit, state, cursor]);
+  const historyPage = monitorId ? queries.histories[monitorId]?.[historyKey] : undefined;
+  const historyPending = !!monitorId && queries.loaded && !historyPage
+    && (historyRead?.key !== historyKey || historyRead.phase === 'pending');
+  const monitorPhase = monitorId ? operations[monitorId] : undefined;
 
   function reject(error: unknown) {
     const code = failureCode(error);
@@ -38,9 +45,29 @@ export function useMonitorState() {
     return () => { active = false; };
   }, []);
 
+  useEffect(() => {
+    if (!queries.loaded || !monitorId || historyPage || monitorPhase === 'pending') return;
+    let active = true;
+    setHistoryRead({ key: historyKey, phase: 'pending' });
+    setError(null);
+    loadChecks({ monitorId, limit, state, cursor }).then(page => {
+      if (!active) return;
+      setQueries(current => ({ ...current, histories: { ...current.histories,
+        [monitorId]: { ...current.histories[monitorId], [historyKey]: page } } }));
+      setHistoryRead({ key: historyKey, phase: 'success' });
+    }).catch(error => {
+      if (!active) return;
+      setHistoryRead({ key: historyKey, phase: 'failure' });
+      reject(error);
+    });
+    // Filter/page changes may overtake a real request. Its success, failure and
+    // pending state all lose authority when these URL conditions are replaced.
+    return () => { active = false; };
+  }, [queries.loaded, monitorId, limit, state, cursor, historyKey, historyPage, monitorPhase]);
+
   async function perform(key: string, work: () => Promise<void>): Promise<boolean> {
     if ((key !== 'logout' && !queries.loaded) || inFlight.current.has(key)
-      || inFlight.current.has('logout')) return false;
+      || inFlight.current.has('logout') || (key === monitorId && historyPending)) return false;
     // The ref closes the gap before React renders disabled buttons, including
     // programmatic form submissions while an earlier response is still pending.
     inFlight.current.add(key);
@@ -90,26 +117,19 @@ export function useMonitorState() {
   function check(id: string) {
     return perform(id, async () => {
       const latestCheck = await runCheck(id);
-      setQueries(current => ({ ...current,
-        monitors: current.monitors.map(row => row.monitor.id === id ? { ...row, latestCheck } : row),
-        histories: current.histories[id] === undefined ? current.histories
-          : { ...current.histories, [id]: [latestCheck, ...current.histories[id]] },
-      }));
-    });
-  }
-
-  function history(id: string) {
-    if (queries.histories[id] !== undefined) return Promise.resolve(true);
-    // Sharing the Monitor key with its writes prevents an older history read
-    // from overwriting a completed check or restoring deleted history.
-    return perform(id, async () => {
-      const checks = await loadChecks(id);
-      setQueries(current => ({ ...current, histories: { ...current.histories, [id]: checks } }));
+      setQueries(current => {
+        const histories = { ...current.histories };
+        // A new run can change several filtered pages. Invalidate this Monitor's
+        // pages, then refetch the selected conditions instead of prepending blindly.
+        delete histories[id];
+        return { ...current, histories,
+          monitors: current.monitors.map(row => row.monitor.id === id ? { ...row, latestCheck } : row) };
+      });
     });
   }
 
   function signOut() {
-    if (inFlight.current.size !== 0) return Promise.resolve(false);
+    if (inFlight.current.size !== 0 || historyPending) return Promise.resolve(false);
     return perform('logout', async () => {
       await logout();
       setQueries(emptyQueries());
@@ -117,8 +137,8 @@ export function useMonitorState() {
     });
   }
 
-  const pending = (key: string) => operations[key] === 'pending';
-  return { ...queries, loading, error, operations, pending,
-    anyPending: Object.values(operations).includes('pending'),
-    create, update, remove, check, history, signOut };
+  const pending = (key: string) => operations[key] === 'pending' || (key === monitorId && historyPending);
+  return { ...queries, loading, error, operations, pending, historyPage, historyPending,
+    anyPending: Object.values(operations).includes('pending') || historyPending,
+    create, update, remove, check, signOut };
 }
diff --git a/evidence/E07/browser-1-partial.json b/evidence/E07/browser-1-partial.json
new file mode 100644
index 0000000..cf5cbce
--- /dev/null
+++ b/evidence/E07/browser-1-partial.json
@@ -0,0 +1,5 @@
+{
+  "fixtureSha256": "a22129aa5b5137b1b4da6907c16f80e3193b58449c91a145ec1329a9d9233bd1",
+  "completed": [],
+  "heldRoutesSettled": true
+}
diff --git a/evidence/E07/browser.json b/evidence/E07/browser.json
new file mode 100644
index 0000000..036bd5a
--- /dev/null
+++ b/evidence/E07/browser.json
@@ -0,0 +1,33 @@
+{
+  "fixtureSha256": "a22129aa5b5137b1b4da6907c16f80e3193b58449c91a145ec1329a9d9233bd1",
+  "completed": [
+    "actual browser cursor header, seven unique originals across pages with newer eighth insertion",
+    "cursor page back/forward/reload",
+    "FAILED filter clears cursor, preserves size and displays exactly three failed rows",
+    "newer All result displayed while one real SUCCEEDED response remains held",
+    "released older response cannot overwrite current rows, URL or pending/error status",
+    "filter back/forward/reload"
+  ],
+  "result": "PASS",
+  "heldRealReads": 1,
+  "originalContinuation": [
+    7,
+    6,
+    5,
+    4,
+    3,
+    2,
+    1
+  ],
+  "currentAll": [
+    8,
+    7,
+    6
+  ],
+  "staleSucceeded": [
+    8,
+    7,
+    5
+  ],
+  "heldRoutesSettled": true
+}
diff --git a/evidence/E07/cleanup.json b/evidence/E07/cleanup.json
new file mode 100644
index 0000000..f6589c3
--- /dev/null
+++ b/evidence/E07/cleanup.json
@@ -0,0 +1,33 @@
+{
+  "schemas": [
+    "public"
+  ],
+  "listeners": [
+    {
+      "port": 4321,
+      "free": true
+    },
+    {
+      "port": 4322,
+      "free": true
+    },
+    {
+      "port": 4323,
+      "free": true
+    },
+    {
+      "port": 4324,
+      "free": true
+    },
+    {
+      "port": 4325,
+      "free": true
+    },
+    {
+      "port": 4999,
+      "free": true
+    }
+  ],
+  "disposableSchemasRemoved": true,
+  "publicNotTargetedAndVolumePreserved": true
+}
diff --git a/evidence/E07/invocations.jsonl b/evidence/E07/invocations.jsonl
new file mode 100644
index 0000000..04dbf85
--- /dev/null
+++ b/evidence/E07/invocations.jsonl
@@ -0,0 +1,10 @@
+{"name":"database-up","command":"node scripts/database.mjs up","startedAt":"2026-08-28T02:52:50.113Z","elapsedSeconds":0.948,"exitCode":0}
+{"name":"baseline","command":"mvn -B -ntp -f backend/pom.xml -Dtest=HistoryPaginationTest -De07.baseline=true test","startedAt":"2026-08-28T02:52:51.063Z","elapsedSeconds":7.683,"exitCode":1}
+{"name":"java-1","command":"mvn -B -ntp -f backend/pom.xml -Dtest=HistoryPaginationTest,HistoryIndexMigrationTest,OwnershipMigrationTest package","startedAt":"2026-08-28T02:57:13.877Z","elapsedSeconds":17.617,"exitCode":0}
+{"name":"typecheck-1","command":"npm run typecheck","startedAt":"2026-08-28T03:02:20.581Z","elapsedSeconds":2.123,"exitCode":0}
+{"name":"build-1","command":"npm run build","startedAt":"2026-08-28T03:02:22.706Z","elapsedSeconds":11.937,"exitCode":0}
+{"name":"browser-1","command":"npm run test:e2e -- tests/browser/history.spec.ts","startedAt":"2026-08-28T03:02:34.644Z","elapsedSeconds":23.391,"exitCode":1}
+{"name":"browser-1-cleanup","command":"node scripts/database.mjs drop e04_browser","startedAt":"2026-08-28T03:02:58.035Z","elapsedSeconds":0.188,"exitCode":0}
+{"name":"browser-2","command":"npm run test:e2e -- tests/browser/history.spec.ts","startedAt":"2026-08-28T03:04:37.870Z","elapsedSeconds":12.652,"exitCode":0}
+{"name":"browser-2-cleanup","command":"node scripts/database.mjs drop e04_browser","startedAt":"2026-08-28T03:04:50.525Z","elapsedSeconds":0.177,"exitCode":0}
+{"name":"cleanup-inspection","command":"read-only PostgreSQL schema names and lsof listener inspection","startedAt":"2026-08-28T03:06:25.559Z","elapsedSeconds":0.481,"exitCode":0}
diff --git a/evidence/E07/verification.md b/evidence/E07/verification.md
new file mode 100644
index 0000000..c48601e
--- /dev/null
+++ b/evidence/E07/verification.md
@@ -0,0 +1,25 @@
+# E07 author verification
+
+START `d2abc8a44e38a05660e49ce664de6f38f8831edd`; attempt1.
+Frozen scenario SHA256 `a22129aa5b5137b1b4da6907c16f80e3193b58449c91a145ec1329a9d9233bd1`.
+Exact commands, timestamps, durations and exit codes are in `invocations.jsonl`.
+Raw local logs remain under ignored `output/e07/`; no credentials or browser captures were retained.
+
+| Invocation | Result | Observed scope |
+| --- | --- | --- |
+| Single unchanged-product baseline | Expected failure, 7.683s | One `limit=3` history GET returned200, seven rows7→1 and no cursor. Product diff from START was empty. |
+| Focused Java/package | PASS, 17.617s | Five tests: pagination/boundaries, V4→V5 index upgrade, three existing ownership migration cases. |
+| Typecheck | PASS, 2.123s | Current production and initial browser test source. |
+| Production build | PASS, 11.937s | Current production source; `/monitors` remains static with a Suspense-wrapped URL reader. |
+| Targeted browser1 | Observation failure, 23.391s | First history200 and cursor header plus URL check passed; the exact wrapped-label locator found no element. No full pagination/stale-response step was completed. |
+| Targeted browser2 | PASS, 12.652s wall / 2.5s case | Sole article combobox locator replaced the unavailable exact-label locator; requests, rows, barriers and expectations unchanged. All fixed steps completed. |
+
+No production/backend change followed the passing typecheck/build. The only subsequent test correction was the combobox locator. No backend or build rerun was needed for that observation-only change.
+
+`backend.json` retains the actual generated Hibernate SQL: verified-owner predicate, optional state predicate, strict `(finished_at,id)` continuation, descending tie ordering and database fetch cap. `setMaxResults(limit + 1)` supplies the finite cap (default20, maximum100). Both actual supporting index definitions were inspected. The V4→V5 test preserves the seven historical rows, Monitor and prior migration checksums; repeated migration executes zero changes.
+
+The body remains `{data: CheckRun[]}` with optional `X-Next-Cursor`. The actual same-origin browser consumed that header to navigate pages. Its original continuation stayed7→1 after newer8 was inserted; FAILED was6/4/2. Cursor and filter back/forward/reload passed. Exactly one real SUCCEEDED response8/7/5 was held while All8/7/6 completed; releasing it left current rows, URL and pending/error status intact.
+
+Existing migration fixtures explicitly target their original V4, without changing their data or assertions. Three legacy browser cases explicitly close URL history before their original closed-history reload setup; their product inputs and assertions remain intact. The full prior browser/Java/restart regression is reserved for root's independent final verification, not claimed as an author run here.
+
+Budget: baseline1, post-change focused Java1, targeted browser2 (one observation failure, one pass), typecheck1, build1; load runs0, automatic retries0, parameter sweeps0. Both browser invocations settled their held routes and dropped `e04_browser`. Final read-only cleanup found only the public schema and free ports4321–4325 and4999. Public data was never targeted; the PostgreSQL volume was preserved.
diff --git a/tests/browser/history.spec.ts b/tests/browser/history.spec.ts
new file mode 100644
index 0000000..7b0250d
--- /dev/null
+++ b/tests/browser/history.spec.ts
@@ -0,0 +1,170 @@
+import { createHash } from 'node:crypto';
+import { execFileSync } from 'node:child_process';
+import { mkdirSync, readFileSync, writeFileSync } from 'node:fs';
+import type { Route } from '@playwright/test';
+import { test, expect, csrfHeaders, safeRequest } from './authenticated';
+
+const id = (number: number) => `00000000-0000-4000-8000-${String(number).padStart(12, '0')}`;
+
+function seed(monitor: string, from: number, through: number) {
+  expect(monitor).toMatch(/^[a-f0-9-]{36}$/);
+  const sql = `INSERT INTO e04_browser.check_runs
+    SELECT ('00000000-0000-4000-8000-' || lpad(n::text,12,'0'))::uuid,
+    '${monitor}'::uuid, 'MANUAL', CASE WHEN n%2=1 OR n=8 THEN 'SUCCEEDED' ELSE 'FAILED' END,
+    CASE WHEN n%2=1 OR n=8 THEN 200 ELSE 503 END, 1,
+    CASE WHEN n%2=1 OR n=8 THEN NULL ELSE 'HTTP_STATUS' END,
+    CASE WHEN n=8 THEN '2026-08-28T00:00:01.000Z' ELSE '2026-08-28T00:00:00.000Z' END::timestamptz,
+    CASE WHEN n=8 THEN '2026-08-28T00:00:01.000Z' ELSE '2026-08-28T00:00:00.000Z' END::timestamptz
+    FROM generate_series(${from},${through}) n`;
+  execFileSync('docker', ['compose', '--project-name', 'wse-industry', '--file', 'compose.yaml',
+    'exec', '--no-TTY', 'postgres', 'psql', '--username', 'wse_industry', '--dbname', 'monitor',
+    '--set', 'ON_ERROR_STOP=1', '--command', sql], { stdio: 'pipe' });
+}
+
+test('fixed tied history pages restore URL state and ignore the held older filter response', async ({ page }) => {
+  const evidence: Record<string, unknown> = {
+    fixtureSha256: createHash('sha256').update(readFileSync('evidence/E07/fixtures.md')).digest('hex'),
+    completed: [] as string[],
+  };
+  const completed = evidence.completed as string[];
+  const existing = await safeRequest(() => page.request.get('/api/monitors'));
+  expect(existing.status()).toBe(200);
+  for (const row of (await existing.json()).data) {
+    const headers = await csrfHeaders(page.request);
+    expect((await safeRequest(() => page.request.delete(`/api/monitors/${row.monitor.id}`, { headers }))).status()).toBe(200);
+  }
+  const headers = await csrfHeaders(page.request);
+  const created = await safeRequest(() => page.request.post('/api/monitors', { headers,
+    data: { name: 'History fixture', url: 'http://127.0.0.1:4321/ok', interval: 60, enabled: true } }));
+  expect(created.status()).toBe(201);
+  const monitor = (await created.json()).data.monitor.id as string;
+  seed(monitor, 1, 7);
+  const path = `/api/monitors/${monitor}/checks`;
+  const article = page.getByRole('article', { name: 'History fixture', exact: true });
+  const filter = article.getByRole('combobox');
+  const next = article.getByRole('button', { name: 'Next page', exact: true });
+  const rows = async (numbers: number[]) => {
+    const visible = article.locator('tbody tr[data-check-id]');
+    await expect(visible).toHaveCount(numbers.length);
+    for (const [index, number] of numbers.entries()) await expect(visible.nth(index)).toHaveAttribute('data-check-id', id(number));
+  };
+  const urlState = async (state: string | null, cursor: string | null) => {
+    await expect(page).toHaveURL(url => url.pathname === '/monitors' && url.searchParams.get('history') === monitor
+      && url.searchParams.get('limit') === '3' && url.searchParams.get('state') === state
+      && url.searchParams.get('cursor') === cursor);
+    await expect(filter).toHaveValue(state ?? '');
+  };
+  const response = () => page.waitForResponse(response => new URL(response.url()).pathname === path
+    && response.request().method() === 'GET');
+  const held = Promise.withResolvers<void>();
+  const release = Promise.withResolvers<void>();
+  const routes: Promise<void>[] = [];
+  let heldReads = 0;
+  let intercept: ((route: Route) => Promise<void>) | undefined;
+  try {
+    const [first] = await Promise.all([response(), page.goto(`/monitors?history=${monitor}&limit=3`)]);
+    expect(first.status()).toBe(200);
+    const cursor2 = await first.headerValue('X-Next-Cursor');
+    expect(cursor2).not.toBeNull();
+    await urlState(null, null);
+    await rows([7, 6, 5]);
+    seed(monitor, 8, 8);
+    const [second] = await Promise.all([response(), next.click()]);
+    expect(second.status()).toBe(200);
+    const cursor3 = await second.headerValue('X-Next-Cursor');
+    expect(cursor3).not.toBeNull();
+    await urlState(null, cursor2);
+    await rows([4, 3, 2]);
+    const [third] = await Promise.all([response(), next.click()]);
+    expect(third.status()).toBe(200);
+    expect(await third.headerValue('X-Next-Cursor')).toBeNull();
+    await urlState(null, cursor3);
+    await rows([1]);
+    await expect(next).toBeDisabled();
+    completed.push('actual browser cursor header, seven unique originals across pages with newer eighth insertion');
+    await page.goBack();
+    await urlState(null, cursor2);
+    await rows([4, 3, 2]);
+    await page.goForward();
+    await urlState(null, cursor3);
+    await rows([1]);
+    const [reloaded] = await Promise.all([response(), page.reload()]);
+    expect(reloaded.status()).toBe(200);
+    await urlState(null, cursor3);
+    await rows([1]);
+    completed.push('cursor page back/forward/reload');
+
+    const [failed] = await Promise.all([response(), filter.selectOption('FAILED')]);
+    expect(failed.status()).toBe(200);
+    await urlState('FAILED', null);
+    await rows([6, 4, 2]);
+    completed.push('FAILED filter clears cursor, preserves size and displays exactly three failed rows');
+    intercept = route => {
+      const url = new URL(route.request().url());
+      if (route.request().method() !== 'GET' || url.searchParams.get('state') !== 'SUCCEEDED' || heldReads !== 0) {
+        return route.continue();
+      }
+      heldReads++;
+      const work = (async () => {
+        expect(url.searchParams.get('limit')).toBe('3');
+        expect(url.searchParams.get('cursor')).toBeNull();
+        const real = await safeRequest(() => route.fetch());
+        expect(real.status()).toBe(200);
+        expect((await real.json()).data.map((row: { id: string }) => row.id)).toEqual([id(8), id(7), id(5)]);
+        held.resolve();
+        await release.promise;
+        await route.fulfill({ response: real });
+      })();
+      routes.push(work);
+      void work.catch(error => held.reject(error));
+      return work;
+    };
+    await page.route(`**${path}?*`, intercept);
+    await filter.selectOption('SUCCEEDED');
+    await held.promise;
+    await urlState('SUCCEEDED', null);
+    await expect(article.getByText('Loading history…', { exact: true })).toBeVisible();
+    const [all] = await Promise.all([response(), filter.selectOption('')]);
+    expect(all.status()).toBe(200);
+    await urlState(null, null);
+    await rows([8, 7, 6]);
+    await expect(article.getByRole('button', { name: 'Run check', exact: true })).toBeEnabled();
+    completed.push('newer All result displayed while one real SUCCEEDED response remains held');
+
+    const stale = response();
+    release.resolve();
+    expect((await stale).status()).toBe(200);
+    await (await stale).finished();
+    await Promise.all(routes);
+    expect(heldReads).toBe(1);
+    await urlState(null, null);
+    await rows([8, 7, 6]);
+    await expect(article.getByRole('button', { name: 'Run check', exact: true })).toBeEnabled();
+    await expect(article.getByText('Loading history…', { exact: true })).toHaveCount(0);
+    await expect(page.locator('main [role="alert"][data-error-code]')).toHaveCount(0);
+    completed.push('released older response cannot overwrite current rows, URL or pending/error status');
+    await page.goBack();
+    await urlState('SUCCEEDED', null);
+    await rows([8, 7, 5]);
+    await page.goForward();
+    await urlState(null, null);
+    await rows([8, 7, 6]);
+    const [final] = await Promise.all([response(), page.reload()]);
+    expect(final.status()).toBe(200);
+    await urlState(null, null);
+    await rows([8, 7, 6]);
+    completed.push('filter back/forward/reload');
+    evidence.result = 'PASS';
+    evidence.heldRealReads = heldReads;
+    evidence.originalContinuation = [7, 6, 5, 4, 3, 2, 1];
+    evidence.currentAll = [8, 7, 6];
+    evidence.staleSucceeded = [8, 7, 5];
+  } finally {
+    release.resolve();
+    const settled = await Promise.allSettled(routes);
+    if (intercept) await page.unroute(`**${path}?*`, intercept);
+    evidence.heldRoutesSettled = settled.every(result => result.status === 'fulfilled');
+    mkdirSync('output/e07', { recursive: true });
+    writeFileSync('output/e07/history-browser.json', `${JSON.stringify(evidence, null, 2)}\n`);
+  }
+});
diff --git a/tests/browser/monitor.spec.ts b/tests/browser/monitor.spec.ts
index 20a52f1..f36e63f 100644
--- a/tests/browser/monitor.spec.ts
+++ b/tests/browser/monitor.spec.ts
@@ -49,6 +49,9 @@ test('persist history, edits, pause, activation and deletion across page reloads
   await expect(monitor.getByText('90s interval · Enabled', { exact: true })).toBeVisible();
   await monitor.getByRole('button', { name: 'Pause', exact: true }).click();
   await expect(monitor.getByText('90s interval · Paused', { exact: true })).toBeVisible();
+  // Keep this case's original closed-history reload setup now that E07 preserves the URL.
+  await monitor.getByRole('button', { name: 'Hide history', exact: true }).click();
+  await expect(page).toHaveURL('/monitors');
   await page.reload();
   await expect(monitor.getByText('90s interval · Paused', { exact: true })).toBeVisible();
   await monitor.getByRole('button', { name: 'Show history' }).click();
diff --git a/tests/browser/ownership.spec.ts b/tests/browser/ownership.spec.ts
index c832217..e2c44ac 100644
--- a/tests/browser/ownership.spec.ts
+++ b/tests/browser/ownership.spec.ts
@@ -109,6 +109,11 @@ test('two real browser sessions isolate resources and retain authorized lifecycl
     await clearOwned(bob.request);
     const a = await createAndCheck(alice, 'Monitor A', '/ok', 60);
     const b = await createAndCheck(bob, 'Monitor B', '/fail', 120);
+    // E07 makes history URL-restorable; retain this security case's original closed-history setup.
+    for (const page of [alice, bob]) {
+      await page.getByRole('button', { name: 'Hide history', exact: true }).click();
+      await expect(page).toHaveURL('/monitors');
+    }
     await alice.reload();
     await bob.reload();
     await expect(alice.getByRole('article')).toHaveCount(1);
diff --git a/tests/browser/server-state.spec.ts b/tests/browser/server-state.spec.ts
index 147a69b..cf0cb2d 100644
--- a/tests/browser/server-state.spec.ts
+++ b/tests/browser/server-state.spec.ts
@@ -48,6 +48,12 @@ test('fixed server-state lifecycle, held create double-submit and independent mu
     return { status: response.status(), body: await response.json() };
   };
   const reload = async () => {
+    // E07 persists history in the URL; explicitly restore this older case's closed-history reload setup.
+    const hide = page.getByRole('button', { name: 'Hide history', exact: true });
+    if (await hide.count()) {
+      await hide.click();
+      await expect(page).toHaveURL('/monitors');
+    }
     expect((await uiResponse(page, '/api/monitors', 'GET', () => page.reload())).status()).toBe(200);
   };
   const firstCommitted = Promise.withResolvers<void>();
