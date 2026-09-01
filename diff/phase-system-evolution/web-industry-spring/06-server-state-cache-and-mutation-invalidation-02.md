## `fix(state): own monitor data and pending mutations together`

diff --git a/TRACK.md b/TRACK.md
index 314610d..c02d207 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -2,7 +2,7 @@
 
 Spec revision: `0a006589477f8ae47bad3faa5510c999cff85ee4`
 
-E05 is a local development product: Next.js/React renders login, logout, Monitor creation, editing, pause/activation, deletion and Check history. Spring Security authenticates browser sessions; PostgreSQL stores salted account passwords, Monitors and every completed CheckRun. A manual check is synchronous. Monitor and CheckRun access is scoped to the signed-in user. There is no signup, scheduler, worker, cache, broker, or production application container.
+E06 is a local development product: Next.js/React renders login, logout, Monitor creation, editing, pause/activation, deletion and Check history. Spring Security authenticates browser sessions; PostgreSQL stores salted account passwords, Monitors and every completed CheckRun. A manual check is synchronous. Monitor and CheckRun access is scoped to the signed-in user. One React hook owns the page's server data and request state. There is no signup, scheduler, worker, Redis, broker, or production application container.
 
 ## Pinned toolchain
 
@@ -263,3 +263,38 @@ References: [Spring Security CSRF](https://docs.spring.io/spring-security/refere
 [Spring Security CORS](https://docs.spring.io/spring-security/reference/servlet/integrations/cors.html),
 and the installed Next 16.3.3 `rewrites` guide. Runtime behavior is verified against
 the pinned framework, not inferred only from the documentation.
+
+## Frontend server state (E06)
+
+`useMonitorState` owns the Monitor collection, canonical article/detail values,
+cached histories, request phases and API failures. The page owns only transient
+editing and history visibility; form drafts remain in their existing forms.
+There is one hook instance on the Monitor page and no module-global or persistent
+cache that could survive a page/session boundary. No dependency was added.
+
+| Successful operation | Related server state |
+| --- | --- |
+| Create | Append the validated canonical response to the collection/detail source |
+| Edit / pause / resume | Replace the same canonical Monitor view; historical checks are unchanged |
+| Manual check | Update latest and any cached history together, including hidden history |
+| Delete | Remove the Monitor view and its history cache |
+| Logout / unauthenticated response | Clear remote data and leave the page |
+
+Writes apply only after a validated success response. A500 leaves the prior
+authoritative data intact and displays the existing error category. Pending,
+success and failure are separate phases. Creation has its own pending key;
+requests for each Monitor share that Monitor's key. A synchronous in-flight guard
+prevents a second submit before React renders disabled controls. An independent
+Monitor failure cannot clear a held create's pending state.
+
+Initial loading must finish before mutations can start. A history read shares its
+Monitor's request key, so it cannot overwrite a newer check or restore deleted
+history. Hiding history changes visibility without discarding the cached data;
+successful checks keep that data current. Reloading gets authoritative data again.
+
+The existing API client, session CSRF bootstrap, browser Origin behavior, backend,
+migrations, dependencies and runtime configuration are unchanged. The fixed E06
+test forwards one real create POST and holds its201 response after commit; it then
+submits again, injects one update500 and one delete500 while held, verifies unchanged
+authority and pending state, releases the response, and checks one durable row.
+The existing article/history serves as detail; no routes or pagination were added.
diff --git a/app/monitors/page.tsx b/app/monitors/page.tsx
index 0b34d67..092653f 100644
--- a/app/monitors/page.tsx
+++ b/app/monitors/page.tsx
@@ -1,8 +1,8 @@
 'use client';
 
-import { useEffect, useState, type FormEvent } from 'react';
-import { createMonitor, deleteMonitor, errorMessages, failureCode, loadChecks, loadMonitors,
-  logout, replaceMonitor, runCheck, type ApiErrorCode, type CheckRun, type Monitor, type MonitorView } from './api';
+import { useState, type FormEvent } from 'react';
+import { errorMessages, type Monitor } from './api';
+import { useMonitorState } from './use-monitor-state';
 
 function inputFrom(form: HTMLFormElement): Omit<Monitor, 'id'> {
   const fields = new FormData(form);
@@ -11,122 +11,40 @@ function inputFrom(form: HTMLFormElement): Omit<Monitor, 'id'> {
 }
 
 export default function Monitors() {
-  const [monitors, setMonitors] = useState<MonitorView[]>([]);
-  const [error, setError] = useState<ApiErrorCode | null>(null);
-  const [busy, setBusy] = useState(false);
+  const state = useMonitorState();
   const [editingId, setEditingId] = useState<string | null>(null);
-  const [histories, setHistories] = useState<Record<string, CheckRun[]>>({});
-
-  function reject(error: unknown) {
-    const code = failureCode(error);
-    if (code === 'UNAUTHENTICATED') {
-      setMonitors([]);
-      setHistories({});
-      setEditingId(null);
-      window.location.replace('/login');
-    } else setError(code);
-  }
-
-  async function signOut() {
-    setBusy(true);
-    setError(null);
-    try {
-      await logout();
-      setMonitors([]);
-      setHistories({});
-      setEditingId(null);
-      window.location.replace('/login');
-    } catch (error) {
-      reject(error);
-    } finally {
-      setBusy(false);
-    }
-  }
-
-  useEffect(() => {
-    loadMonitors().then(setMonitors).catch(reject);
-  }, []);
+  const [visibleHistories, setVisibleHistories] = useState<Record<string, boolean>>({});
 
   async function create(event: FormEvent<HTMLFormElement>) {
     event.preventDefault();
     const form = event.currentTarget;
-    setBusy(true);
-    setError(null);
-    try {
-      const created = await createMonitor(inputFrom(form));
-      setMonitors(current => [...current, created]);
-      form.reset();
-    } catch (error) {
-      reject(error);
-    } finally {
-      setBusy(false);
-    }
-  }
-
-  async function check(id: string) {
-    setBusy(true);
-    setError(null);
-    try {
-      const latestCheck = await runCheck(id);
-      setMonitors(current => current.map(row => row.monitor.id === id ? { ...row, latestCheck } : row));
-      setHistories(current => current[id] ? { ...current, [id]: [latestCheck, ...current[id]] } : current);
-    } catch (error) {
-      reject(error);
-    } finally {
-      setBusy(false);
-    }
+    if (await state.create(inputFrom(form))) form.reset();
   }
 
   async function update(id: string, input: Omit<Monitor, 'id'>) {
-    setBusy(true);
-    setError(null);
-    try {
-      const updated = await replaceMonitor(id, input);
-      setMonitors(current => current.map(row => row.monitor.id === id ? updated : row));
-      setEditingId(null);
-    } catch (error) {
-      reject(error);
-    } finally {
-      setBusy(false);
-    }
+    if (await state.update(id, input)) setEditingId(null);
   }
 
   async function remove(id: string) {
-    setBusy(true);
-    setError(null);
-    try {
-      await deleteMonitor(id);
-      setMonitors(current => current.filter(row => row.monitor.id !== id));
-      setHistories(current => { const remaining = { ...current }; delete remaining[id]; return remaining; });
+    if (await state.remove(id)) {
+      setVisibleHistories(current => { const remaining = { ...current }; delete remaining[id]; return remaining; });
       if (editingId === id) setEditingId(null);
-    } catch (error) {
-      reject(error);
-    } finally {
-      setBusy(false);
     }
   }
 
   async function history(id: string) {
-    if (histories[id]) {
-      setHistories(current => { const remaining = { ...current }; delete remaining[id]; return remaining; });
+    if (visibleHistories[id]) {
+      setVisibleHistories(current => ({ ...current, [id]: false }));
       return;
     }
-    setBusy(true);
-    setError(null);
-    try {
-      const checks = await loadChecks(id);
-      setHistories(current => ({ ...current, [id]: checks }));
-    } catch (error) {
-      reject(error);
-    } finally {
-      setBusy(false);
-    }
+    setVisibleHistories(current => ({ ...current, [id]: true }));
+    await state.history(id);
   }
 
   return <main>
     <header><p className="eyebrow">LOCAL LAB · INDUSTRY / SPRING</p><h1>Endpoint Monitor</h1>
       <p>Create a monitor and check its HTTP response.</p>
-      <button disabled={busy} onClick={signOut}>Sign out</button></header>
+      <button disabled={state.anyPending || state.loading} onClick={() => void state.signOut()}>Sign out</button></header>
     <aside>Development fixture only. Monitors and completed checks are stored in PostgreSQL.</aside>
     <section aria-labelledby="create-title">
       <h2 id="create-title">Create monitor</h2>
@@ -136,23 +54,30 @@ export default function Monitors() {
         <label>Interval (seconds)<input name="interval" type="number" min="1" max="86400" step="1" required defaultValue="60" /></label>
         <label className="checkbox"><input name="enabled" type="checkbox" defaultChecked />Enabled</label>
         <p className="hint">Interval and enabled are stored. Checks still run manually; there is no scheduler.</p>
-        <button disabled={busy}>Create monitor</button>
+        <button disabled={!state.loaded || state.pending('create') || state.pending('logout')}>Create monitor</button>
       </form>
+      {state.pending('create') && <p role="status">Creating monitor…</p>}
+      {state.operations.create === 'success' && <p role="status">Monitor created.</p>}
     </section>
-    {error && <p role="alert" className="error" data-error-code={error}>{errorMessages[error]}</p>}
+    {state.error && <p role="alert" className="error" data-error-code={state.error}>{errorMessages[state.error]}</p>}
     <section aria-labelledby="monitors-title"><h2 id="monitors-title">Monitors</h2>
-      {monitors.length === 0 && <p>No monitors yet.</p>}
-      {monitors.map(({ monitor, latestCheck }) => <article key={monitor.id} aria-label={monitor.name}>
+      {state.loading && <p role="status">Loading monitors…</p>}
+      {state.loaded && state.monitors.length === 0 && <p>No monitors yet.</p>}
+      {state.monitors.map(({ monitor, latestCheck }) => {
+        const busy = state.pending(monitor.id) || state.pending('logout');
+        const checks = state.histories[monitor.id];
+        return <article key={monitor.id} aria-label={monitor.name}>
         <h3>{monitor.name}</h3><p className="url">{monitor.url}</p>
         <p>{monitor.interval}s interval · {monitor.enabled ? 'Enabled' : 'Paused'}</p>
         <div className="actions">
-          <button disabled={busy} onClick={() => check(monitor.id)}>Run check</button>
+          <button disabled={busy} onClick={() => void state.check(monitor.id)}>Run check</button>
           <button disabled={busy} onClick={() => setEditingId(monitor.id)}>Edit</button>
           <button disabled={busy} onClick={() => update(monitor.id, { name: monitor.name, url: monitor.url,
             interval: monitor.interval, enabled: !monitor.enabled })}>{monitor.enabled ? 'Pause' : 'Activate'}</button>
-          <button disabled={busy} onClick={() => history(monitor.id)}>{histories[monitor.id] ? 'Hide history' : 'Show history'}</button>
+          <button disabled={busy} onClick={() => history(monitor.id)}>{visibleHistories[monitor.id] ? 'Hide history' : 'Show history'}</button>
           <button disabled={busy} onClick={() => remove(monitor.id)}>Delete</button>
         </div>
+        {busy && <p role="status">Request pending…</p>}
         {editingId === monitor.id && <form aria-label="Edit monitor" onSubmit={event => {
           event.preventDefault();
           void update(monitor.id, inputFrom(event.currentTarget));
@@ -173,18 +98,19 @@ export default function Monitors() {
             {latestCheck.failureReason && <span>{latestCheck.failureReason}</span>}
           </> : <span>No checks yet.</span>}
         </div>
-        {histories[monitor.id] && <section aria-label="Historical checks">
+        {visibleHistories[monitor.id] && <section aria-label="Historical checks">
           <h4>Check history</h4>
-          {histories[monitor.id].length === 0 ? <p>No historical checks.</p> : <div className="history-table">
+          {checks === undefined ? <p>{busy ? 'Loading history…' : 'History could not be loaded.'}</p>
+            : checks.length === 0 ? <p>No historical checks.</p> : <div className="history-table">
             <table aria-label="Check history"><thead><tr><th>Finished (UTC)</th><th>Result</th><th>HTTP</th><th>Latency</th><th>Reason</th></tr></thead>
-              <tbody>{histories[monitor.id].map(check => <tr key={check.id} data-check-id={check.id}>
+              <tbody>{checks.map(check => <tr key={check.id} data-check-id={check.id}>
                 <td><time dateTime={check.finishedAt}>{check.finishedAt}</time></td><td>{check.state}</td>
                 <td>{check.httpStatus ?? '—'}</td><td>{check.latencyMs} ms</td><td>{check.failureReason ?? '—'}</td>
               </tr>)}</tbody>
             </table>
           </div>}
         </section>}
-      </article>)}
+      </article>; })}
     </section>
   </main>;
 }
diff --git a/app/monitors/use-monitor-state.ts b/app/monitors/use-monitor-state.ts
new file mode 100644
index 0000000..ed57e05
--- /dev/null
+++ b/app/monitors/use-monitor-state.ts
@@ -0,0 +1,124 @@
+'use client';
+
+import { useEffect, useRef, useState } from 'react';
+import { createMonitor, deleteMonitor, failureCode, loadChecks, loadMonitors, logout,
+  replaceMonitor, runCheck, type ApiErrorCode, type CheckRun, type Monitor, type MonitorView } from './api';
+
+type Queries = { monitors: MonitorView[]; histories: Record<string, CheckRun[]>; loaded: boolean };
+type Phase = 'pending' | 'success' | 'failure';
+type Input = Omit<Monitor, 'id'>;
+const emptyQueries = (): Queries => ({ monitors: [], histories: {}, loaded: false });
+
+// One page-scoped owner for remote data and request state. Form drafts and which
+// history is visible stay in the view; no cache survives a session/page boundary.
+export function useMonitorState() {
+  const [queries, setQueries] = useState<Queries>(emptyQueries);
+  const [operations, setOperations] = useState<Record<string, Phase>>({});
+  const [error, setError] = useState<ApiErrorCode | null>(null);
+  const [loading, setLoading] = useState(true);
+  const inFlight = useRef(new Set<string>());
+
+  function reject(error: unknown) {
+    const code = failureCode(error);
+    if (code === 'UNAUTHENTICATED') {
+      setQueries(emptyQueries());
+      window.location.replace('/login');
+    } else setError(code);
+  }
+
+  useEffect(() => {
+    let active = true;
+    loadMonitors().then(monitors => {
+      if (active) setQueries(current => ({ ...current, monitors, loaded: true }));
+    }).catch(error => {
+      if (active) reject(error);
+    }).finally(() => {
+      if (active) setLoading(false);
+    });
+    return () => { active = false; };
+  }, []);
+
+  async function perform(key: string, work: () => Promise<void>): Promise<boolean> {
+    if ((key !== 'logout' && !queries.loaded) || inFlight.current.has(key)
+      || inFlight.current.has('logout')) return false;
+    // The ref closes the gap before React renders disabled buttons, including
+    // programmatic form submissions while an earlier response is still pending.
+    inFlight.current.add(key);
+    setOperations(current => ({ ...current, [key]: 'pending' }));
+    setError(null);
+    try {
+      await work();
+      setOperations(current => ({ ...current, [key]: 'success' }));
+      return true;
+    } catch (error) {
+      setOperations(current => ({ ...current, [key]: 'failure' }));
+      reject(error);
+      return false;
+    } finally {
+      inFlight.current.delete(key);
+    }
+  }
+
+  function create(input: Input) {
+    return perform('create', async () => {
+      const created = await createMonitor(input);
+      setQueries(current => ({ ...current, monitors: [...current.monitors, created] }));
+    });
+  }
+
+  function update(id: string, input: Input) {
+    return perform(id, async () => {
+      const updated = await replaceMonitor(id, input);
+      // List and article/detail read the same canonical view. Editing does not
+      // change historical CheckRuns, so their cache remains valid.
+      setQueries(current => ({ ...current,
+        monitors: current.monitors.map(row => row.monitor.id === id ? updated : row) }));
+    });
+  }
+
+  function remove(id: string) {
+    return perform(id, async () => {
+      await deleteMonitor(id);
+      setQueries(current => {
+        const histories = { ...current.histories };
+        delete histories[id];
+        return { ...current, monitors: current.monitors.filter(row => row.monitor.id !== id), histories };
+      });
+    });
+  }
+
+  function check(id: string) {
+    return perform(id, async () => {
+      const latestCheck = await runCheck(id);
+      setQueries(current => ({ ...current,
+        monitors: current.monitors.map(row => row.monitor.id === id ? { ...row, latestCheck } : row),
+        histories: current.histories[id] === undefined ? current.histories
+          : { ...current.histories, [id]: [latestCheck, ...current.histories[id]] },
+      }));
+    });
+  }
+
+  function history(id: string) {
+    if (queries.histories[id] !== undefined) return Promise.resolve(true);
+    // Sharing the Monitor key with its writes prevents an older history read
+    // from overwriting a completed check or restoring deleted history.
+    return perform(id, async () => {
+      const checks = await loadChecks(id);
+      setQueries(current => ({ ...current, histories: { ...current.histories, [id]: checks } }));
+    });
+  }
+
+  function signOut() {
+    if (inFlight.current.size !== 0) return Promise.resolve(false);
+    return perform('logout', async () => {
+      await logout();
+      setQueries(emptyQueries());
+      window.location.replace('/login');
+    });
+  }
+
+  const pending = (key: string) => operations[key] === 'pending';
+  return { ...queries, loading, error, operations, pending,
+    anyPending: Object.values(operations).includes('pending'),
+    create, update, remove, check, history, signOut };
+}
diff --git a/tests/browser/server-state.spec.ts b/tests/browser/server-state.spec.ts
index ea69fbd..147a69b 100644
--- a/tests/browser/server-state.spec.ts
+++ b/tests/browser/server-state.spec.ts
@@ -159,14 +159,14 @@ test('fixed server-state lifecycle, held create double-submit and independent mu
     await a.getByLabel('Edit name', { exact: true }).fill('Rejected A');
     await a.getByLabel('Edit interval (seconds)', { exact: true }).fill('90');
     expect((await uiResponse(page, aPath, 'PUT', () => a.getByRole('button', { name: 'Save changes', exact: true }).click())).status()).toBe(500);
-    await expect(page.getByRole('alert')).toHaveAttribute('data-error-code', 'INTERNAL_ERROR');
+    await expect(page.locator('[role="alert"][data-error-code]')).toHaveAttribute('data-error-code', 'INTERNAL_ERROR');
     await expect(a.getByText('60s interval · Enabled', { exact: true })).toBeVisible();
     expect(await authoritative(aPath)).toEqual(beforeA);
     expect(await authoritative(`${aPath}/checks`)).toEqual(beforeHistory);
     await expect(page.getByRole('button', { name: 'Create monitor', exact: true })).toBeDisabled();
     await expect(page.getByRole('status').filter({ hasText: 'Creating monitor…' })).toBeVisible();
     expect((await uiResponse(page, aPath, 'DELETE', () => a.getByRole('button', { name: 'Delete', exact: true }).click())).status()).toBe(500);
-    await expect(page.getByRole('alert')).toHaveAttribute('data-error-code', 'INTERNAL_ERROR');
+    await expect(page.locator('[role="alert"][data-error-code]')).toHaveAttribute('data-error-code', 'INTERNAL_ERROR');
     await expect(a).toBeVisible();
     expect(await authoritative(aPath)).toEqual(beforeA);
     expect(await authoritative(`${aPath}/checks`)).toEqual(beforeHistory);


