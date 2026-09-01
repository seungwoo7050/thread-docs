## `fix: centralize browser mutation state ownership`

diff --git a/TRACK.md b/TRACK.md
index 889d976..963fedb 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -376,3 +376,35 @@ rotation and logout. Chromium retains the E01–E04 lifecycle tests and adds two
 independent logged-in browser contexts, missing-CSRF update/logout and an actual
 blocked cross-origin preflight. Traces/videos remain disabled and test evidence
 contains only statuses, counts and booleans; no credential values are serialized.
+
+## E06 browser server state
+
+`app/monitors/use-monitors.ts` owns the route's Monitor list, ID-keyed history
+queries, authentication boundary and mutation states. The existing expanded
+Monitor article/history is the detail view. Its Monitor fields come from the
+same list objects; the component owns only the selected panels and form inputs.
+There is no additional cache dependency, browser persistence, Redis or new route.
+
+Create and update apply the API's authoritative response to the list. Delete
+removes the Monitor and its history entry. A completed manual Check updates the
+latest result and invalidates/refetches any already requested history. Versioned
+history reads cannot restore a stale result after a newer request or deletion.
+Session loss, logout and unmount invalidate outstanding read/mutation responses.
+
+Mutation admission uses an immediate ref-backed set before CSRF retrieval, as
+[React refs](https://react.dev/reference/react/useRef) can retain event-handler
+coordination independently of a rendered state snapshot. Create has one key;
+update, pause and delete share a Monitor key. The state owner separately exposes
+pending, success and failure for rendering. Form drafts stay unchanged on failure;
+only acknowledged success updates authoritative UI. Logout waits for pending
+mutations, while unrelated Monitor actions can complete during a slow create.
+This prevents repeated browser submission; it does not claim E10 server-side
+request idempotency across clients, reloads or network retransmission.
+
+`evidence/E06/scenario.json` and its browser harness were hashed before the single
+unchanged-START run. Sequential create/update/delete/check refresh already worked.
+With the first create response held after the real PostgreSQL commit, a second
+submit event produced two POSTs and two rows. The same frozen harness now requires
+one POST/row, holds that response while one update and one delete return fixed
+500 errors, and verifies state before and after explicit release. No sleeps,
+automatic retries, traces, videos or credentials are used as evidence.
diff --git a/app/monitors/page.tsx b/app/monitors/page.tsx
index 1ae2710..5d73da9 100644
--- a/app/monitors/page.tsx
+++ b/app/monitors/page.tsx
@@ -1,142 +1,49 @@
 'use client';
 
-import { useCallback, useEffect, useState } from 'react';
+import { useState } from 'react';
 import type { FormEvent } from 'react';
-import { useRouter } from 'next/navigation';
-import type { ApiErrorCode, CheckRun, MonitorView } from '../../server/model';
-import { ERROR_MESSAGES, failureCode, mutationFetch, responseData } from './api';
+import type { MonitorView } from '../../server/model';
+import { ERROR_MESSAGES } from './api';
+import { useMonitors } from './use-monitors';
 
 export default function MonitorsPage() {
-  const router = useRouter();
-  const [authenticated, setAuthenticated] = useState(false);
-  const [loading, setLoading] = useState(true);
-  const [loggingOut, setLoggingOut] = useState(false);
-  const [monitors, setMonitors] = useState<MonitorView[]>([]);
-  const [error, setError] = useState<ApiErrorCode | null>(null);
-  const [creating, setCreating] = useState(false);
-  const [checking, setChecking] = useState<string | null>(null);
+  const { authenticated, loading, monitors, histories, mutations, error, mutate, loadHistory } = useMonitors();
+  const creating = mutations.create?.status === 'pending';
+  const loggingOut = mutations.logout?.status === 'pending';
+  const anyPending = Object.values(mutations).some((mutation) => mutation.status === 'pending');
   const [editing, setEditing] = useState<string | null>(null);
-  const [saving, setSaving] = useState<string | null>(null);
-  const [deleting, setDeleting] = useState<string | null>(null);
   const [historyId, setHistoryId] = useState<string | null>(null);
-  const [history, setHistory] = useState<CheckRun[]>([]);
-  const [loadingHistory, setLoadingHistory] = useState(false);
-
-  const handleFailure = useCallback((failure: unknown) => {
-    const code = failureCode(failure);
-    setError(code);
-    if (code === 'UNAUTHENTICATED') {
-      setAuthenticated(false);
-      setMonitors([]);
-      setHistory([]);
-      setHistoryId(null);
-      router.replace('/login');
-    }
-  }, [router]);
-
-  useEffect(() => {
-    fetch('/api/monitors', { credentials: 'same-origin' }).then(responseData<MonitorView[]>).then((data) => {
-      setMonitors(data);
-      setAuthenticated(true);
-    }).catch(handleFailure).finally(() => setLoading(false));
-  }, [handleFailure]);
-
-  async function signOut() {
-    setLoggingOut(true);
-    setError(null);
-    try {
-      await responseData(await mutationFetch('/api/auth/logout', { method: 'POST' }));
-      setAuthenticated(false);
-      setMonitors([]);
-      setHistory([]);
-      setHistoryId(null);
-      router.replace('/login');
-    } catch (failure) { handleFailure(failure); }
-    finally { setLoggingOut(false); }
-  }
 
   async function createMonitor(event: FormEvent<HTMLFormElement>) {
     event.preventDefault();
     const form = event.currentTarget;
     const fields = new FormData(form);
-    setCreating(true);
-    setError(null);
-    try {
-      const response = await mutationFetch('/api/monitors', {
-        method: 'POST', credentials: 'same-origin', headers: { 'content-type': 'application/json' },
-        body: JSON.stringify({
-          name: fields.get('name'), url: fields.get('url'),
-          interval: Number(fields.get('interval')), enabled: fields.get('enabled') === 'on',
-        }),
-      });
-      const monitor = await responseData<MonitorView>(response);
-      setMonitors((current) => [...current, monitor]);
-      form.reset();
-    } catch (failure) {
-      handleFailure(failure);
-    } finally { setCreating(false); }
-  }
-
-  async function runCheck(id: string) {
-    setChecking(id);
-    setError(null);
-    try {
-      const response = await mutationFetch(`/api/monitors/${id}/checks`, { method: 'POST' });
-      const result = await responseData<CheckRun>(response);
-      setMonitors((current) => current.map((monitor) => monitor.id === id ? { ...monitor, latestCheck: result } : monitor));
-      if (historyId === id) await showHistory(id);
-    } catch (failure) {
-      handleFailure(failure);
-    } finally { setChecking(null); }
-  }
-
-  async function updateMonitor(monitor: MonitorView, input: { name: string; url: string; interval: number; enabled: boolean }) {
-    setSaving(monitor.id);
-    setError(null);
-    try {
-      const response = await mutationFetch(`/api/monitors/${monitor.id}`, {
-        method: 'PUT', credentials: 'same-origin', headers: { 'content-type': 'application/json' }, body: JSON.stringify(input),
-      });
-      const saved = await responseData<MonitorView>(response);
-      setMonitors((current) => current.map((item) => item.id === saved.id ? saved : item));
-      setEditing(null);
-    } catch (failure) { handleFailure(failure); }
-    finally { setSaving(null); }
+    if (await mutate({ kind: 'create', input: {
+      name: String(fields.get('name')), url: String(fields.get('url')),
+      interval: Number(fields.get('interval')), enabled: fields.get('enabled') === 'on',
+    } })) form.reset();
   }
 
   async function editMonitor(event: FormEvent<HTMLFormElement>, monitor: MonitorView) {
     event.preventDefault();
     const fields = new FormData(event.currentTarget);
-    await updateMonitor(monitor, {
+    if (await mutate({ kind: 'update', id: monitor.id, input: {
       name: String(fields.get('name')), url: String(fields.get('url')),
       interval: Number(fields.get('interval')), enabled: monitor.enabled,
-    });
+    } })) setEditing((current) => current === monitor.id ? null : current);
   }
 
   async function deleteMonitor(id: string) {
     if (!window.confirm('Delete this monitor and all its check history?')) return;
-    setDeleting(id);
-    setError(null);
-    try {
-      const response = await mutationFetch(`/api/monitors/${id}`, { method: 'DELETE' });
-      await responseData<{ id: string }>(response);
-      setMonitors((current) => current.filter((monitor) => monitor.id !== id));
-      if (historyId === id) { setHistoryId(null); setHistory([]); }
-      if (editing === id) setEditing(null);
-    } catch (failure) { handleFailure(failure); }
-    finally { setDeleting(null); }
+    if (await mutate({ kind: 'delete', id })) {
+      setHistoryId((current) => current === id ? null : current);
+      setEditing((current) => current === id ? null : current);
+    }
   }
 
-  async function showHistory(id: string) {
+  function showHistory(id: string) {
     setHistoryId(id);
-    setHistory([]);
-    setLoadingHistory(true);
-    setError(null);
-    try {
-      const response = await fetch(`/api/monitors/${id}/checks`, { credentials: 'same-origin' });
-      setHistory(await responseData<CheckRun[]>(response));
-    } catch (failure) { handleFailure(failure); }
-    finally { setLoadingHistory(false); }
+    void loadHistory(id);
   }
 
   if (!authenticated) return <main>
@@ -149,7 +56,7 @@ export default function MonitorsPage() {
   return <main>
     <header><p className="eyebrow">HTTP ENDPOINT MONITOR · LOCAL FIXTURE</p><h1>Monitors</h1>
       <p>Create an endpoint monitor, run a check, and inspect the response.</p>
-      <button onClick={signOut} disabled={loggingOut}>{loggingOut ? 'Signing out…' : 'Sign out'}</button>
+      <button onClick={() => mutate({ kind: 'logout' })} disabled={anyPending}>{loggingOut ? 'Signing out…' : 'Sign out'}</button>
     </header>
     <aside>Development only. Checks can access the configured fixture origin. Monitors and all check history are stored in PostgreSQL.</aside>
     {error && <p role="alert" className="error" data-error-code={error}>{ERROR_MESSAGES[error]}</p>}
@@ -157,48 +64,55 @@ export default function MonitorsPage() {
       <h2 id="create-heading">Create monitor</h2>
       <form onSubmit={createMonitor}>
         <label htmlFor="name">Name</label>
-        <input id="name" name="name" required placeholder="Fixture monitor" />
+        <input id="name" name="name" required placeholder="Fixture monitor" disabled={creating || loggingOut} />
         <label htmlFor="url">Endpoint URL</label>
-        <input id="url" name="url" type="url" required defaultValue="http://127.0.0.1:4311/ok" />
+        <input id="url" name="url" type="url" required defaultValue="http://127.0.0.1:4311/ok" disabled={creating || loggingOut} />
         <label htmlFor="interval">Interval (seconds)</label>
-        <input id="interval" name="interval" type="number" required min="1" max="86400" step="1" defaultValue="60" />
-        <label className="checkbox"><input name="enabled" type="checkbox" defaultChecked /> Enabled</label>
+        <input id="interval" name="interval" type="number" required min="1" max="86400" step="1" defaultValue="60" disabled={creating || loggingOut} />
+        <label className="checkbox"><input name="enabled" type="checkbox" defaultChecked disabled={creating || loggingOut} /> Enabled</label>
         <p className="hint">Interval and enabled are saved; this version only runs manual checks.</p>
-        <button type="submit" disabled={creating}>{creating ? 'Creating…' : 'Create monitor'}</button>
+        <button type="submit" disabled={creating || loggingOut}>{creating ? 'Creating…' : 'Create monitor'}</button>
       </form>
+      {mutations.create?.status === 'success' && <p role="status">Monitor created.</p>}
     </section>
     <section aria-labelledby="saved-heading">
       <h2 id="saved-heading">Your monitors</h2>
       {monitors.length === 0 && <p>No monitors yet.</p>}
-      {monitors.map((monitor) => <article key={monitor.id} className="panel" aria-labelledby={`monitor-${monitor.id}`}>
+      {monitors.map((monitor) => {
+        const mutation = mutations[`monitor:${monitor.id}`];
+        const pending = mutation?.status === 'pending';
+        const historyQuery = histories[monitor.id];
+        const history = historyQuery?.data ?? [];
+        return <article key={monitor.id} className="panel" aria-labelledby={`monitor-${monitor.id}`}>
         <h3 id={`monitor-${monitor.id}`}>{monitor.name}</h3>
         <p className="endpoint">{monitor.url}</p>
         <p>{monitor.enabled ? 'Enabled' : 'Paused'} · {monitor.interval} seconds</p>
         <div className="actions">
-        <button onClick={() => runCheck(monitor.id)} disabled={checking !== null || saving !== null || deleting !== null}>
-          {checking === monitor.id ? 'Checking…' : 'Run check'}
+        <button onClick={() => mutate({ kind: 'check', id: monitor.id })} disabled={pending || loggingOut}>
+          {pending && mutation.kind === 'check' ? 'Checking…' : 'Run check'}
         </button>
-        <button onClick={() => setEditing(editing === monitor.id ? null : monitor.id)} disabled={saving !== null || deleting !== null}>
+        <button onClick={() => setEditing(editing === monitor.id ? null : monitor.id)} disabled={pending || loggingOut}>
           {editing === monitor.id ? 'Cancel edit' : 'Edit monitor'}
         </button>
-        <button onClick={() => updateMonitor(monitor, { ...monitor, enabled: !monitor.enabled })} disabled={saving !== null || deleting !== null}>
+        <button onClick={() => mutate({ kind: 'update', id: monitor.id, input: { ...monitor, enabled: !monitor.enabled } })} disabled={pending || loggingOut}>
           {monitor.enabled ? 'Pause monitor' : 'Enable monitor'}
         </button>
-        <button onClick={() => deleteMonitor(monitor.id)} disabled={saving !== null || deleting !== null || checking !== null}>
-          {deleting === monitor.id ? 'Deleting…' : 'Delete monitor'}
+        <button onClick={() => deleteMonitor(monitor.id)} disabled={pending || loggingOut}>
+          {pending && mutation.kind === 'delete' ? 'Deleting…' : 'Delete monitor'}
         </button>
-        <button onClick={() => historyId === monitor.id ? setHistoryId(null) : showHistory(monitor.id)} disabled={loadingHistory}>
+        <button onClick={() => historyId === monitor.id ? setHistoryId(null) : showHistory(monitor.id)} disabled={loggingOut}>
           {historyId === monitor.id ? 'Hide history' : 'View history'}
         </button>
         </div>
+        {mutation?.status === 'success' && <p role="status">{mutation.kind === 'check' ? 'Check completed.' : 'Monitor saved.'}</p>}
         {editing === monitor.id && <form onSubmit={(event) => editMonitor(event, monitor)} className="edit-form">
           <label htmlFor={`edit-name-${monitor.id}`}>Edit name</label>
-          <input id={`edit-name-${monitor.id}`} name="name" required defaultValue={monitor.name} />
+          <input id={`edit-name-${monitor.id}`} name="name" required defaultValue={monitor.name} disabled={pending || loggingOut} />
           <label htmlFor={`edit-url-${monitor.id}`}>Edit endpoint URL</label>
-          <input id={`edit-url-${monitor.id}`} name="url" type="url" required defaultValue={monitor.url} />
+          <input id={`edit-url-${monitor.id}`} name="url" type="url" required defaultValue={monitor.url} disabled={pending || loggingOut} />
           <label htmlFor={`edit-interval-${monitor.id}`}>Edit interval (seconds)</label>
-          <input id={`edit-interval-${monitor.id}`} name="interval" type="number" min="1" max="86400" step="1" required defaultValue={monitor.interval} />
-          <button type="submit" disabled={saving !== null}>{saving === monitor.id ? 'Saving…' : 'Save changes'}</button>
+          <input id={`edit-interval-${monitor.id}`} name="interval" type="number" min="1" max="86400" step="1" required defaultValue={monitor.interval} disabled={pending || loggingOut} />
+          <button type="submit" disabled={pending || loggingOut}>{pending && mutation.kind === 'update' ? 'Saving…' : 'Save changes'}</button>
         </form>}
         {!monitor.enabled && <p className="hint">Paused. Manual checks remain available; no scheduler runs in this version.</p>}
         {monitor.latestCheck ? <dl aria-label="Latest check result">
@@ -210,7 +124,7 @@ export default function MonitorsPage() {
         </dl> : <p>No checks yet.</p>}
         {historyId === monitor.id && <section aria-label={`Check history for ${monitor.name}`}>
           <h4>Check history</h4>
-          {loadingHistory ? <p>Loading history…</p> : history.length === 0 ? <p>No historical checks.</p> :
+          {historyQuery?.status === 'pending' ? <p>Loading history…</p> : historyQuery?.data === null ? <p>History could not be loaded.</p> : history.length === 0 ? <p>No historical checks.</p> :
             <div className="history-scroll"><table>
               <thead><tr><th>Check ID</th><th>State</th><th>HTTP status</th><th>Latency</th><th>Failure reason</th><th>Finished</th></tr></thead>
               <tbody>{history.map((check) => <tr key={check.id}>
@@ -220,7 +134,7 @@ export default function MonitorsPage() {
               </tr>)}</tbody>
             </table></div>}
         </section>}
-      </article>)}
+      </article>; })}
     </section>
   </main>;
 }
diff --git a/app/monitors/use-monitors.ts b/app/monitors/use-monitors.ts
new file mode 100644
index 0000000..466b25d
--- /dev/null
+++ b/app/monitors/use-monitors.ts
@@ -0,0 +1,146 @@
+'use client';
+
+import { useCallback, useEffect, useRef, useState } from 'react';
+import { useRouter } from 'next/navigation';
+import type { ApiErrorCode, CheckRun, MonitorView } from '../../server/model.ts';
+import { failureCode, mutationFetch, responseData } from './api.ts';
+
+type MonitorInput = Pick<MonitorView, 'name' | 'url' | 'interval' | 'enabled'>;
+type Mutation =
+  | { kind: 'create'; input: MonitorInput }
+  | { kind: 'update'; id: string; input: MonitorInput }
+  | { kind: 'delete' | 'check'; id: string }
+  | { kind: 'logout' };
+type MutationState = { kind: Mutation['kind']; status: 'pending' | 'success' | 'error'; error: ApiErrorCode | null };
+type HistoryQuery = { status: 'pending' | 'success' | 'error'; data: CheckRun[] | null; error: ApiErrorCode | null };
+type ServerState = {
+  authenticated: boolean;
+  loading: boolean;
+  monitors: MonitorView[];
+  histories: Record<string, HistoryQuery>;
+  mutations: Record<string, MutationState>;
+  error: ApiErrorCode | null;
+};
+
+const emptyState = (): ServerState => ({ authenticated: false, loading: true, monitors: [], histories: {}, mutations: {}, error: null });
+
+// One owner for this route's server state. Forms and the selected panels stay in
+// the component; list/detail share Monitor objects and history is keyed by ID.
+export function useMonitors() {
+  const router = useRouter();
+  const [state, setState] = useState<ServerState>(emptyState);
+  const lifetime = useRef(0);
+  const pending = useRef(new Set<string>());
+  const historyVersions = useRef(new Map<string, number>());
+
+  const clearSession = useCallback(() => {
+    lifetime.current++;
+    pending.current.clear();
+    historyVersions.current.clear();
+    setState({ ...emptyState(), loading: false });
+    router.replace('/login');
+  }, [router]);
+
+  const handleFailure = useCallback((failure: unknown) => {
+    const code = failureCode(failure);
+    if (code === 'UNAUTHENTICATED') clearSession();
+    else setState((current) => ({ ...current, error: code }));
+    return code;
+  }, [clearSession]);
+
+  useEffect(() => {
+    const generation = ++lifetime.current;
+    fetch('/api/monitors', { credentials: 'same-origin' }).then(responseData<MonitorView[]>).then((monitors) => {
+      if (generation === lifetime.current) setState((current) => ({ ...current, monitors, authenticated: true }));
+    }).catch((failure: unknown) => {
+      if (generation === lifetime.current) handleFailure(failure);
+    }).finally(() => {
+      if (generation === lifetime.current) setState((current) => ({ ...current, loading: false }));
+    });
+    return () => { lifetime.current++; };
+  }, [handleFailure]);
+
+  const loadHistory = useCallback(async (id: string, clearError = true) => {
+    const generation = lifetime.current;
+    const version = (historyVersions.current.get(id) ?? 0) + 1;
+    historyVersions.current.set(id, version);
+    setState((current) => ({ ...current, error: clearError ? null : current.error, histories: {
+      ...current.histories, [id]: { status: 'pending', data: current.histories[id]?.data ?? null, error: null },
+    } }));
+    const isCurrent = () => generation === lifetime.current && version === historyVersions.current.get(id);
+    try {
+      const data = await responseData<CheckRun[]>(await fetch(`/api/monitors/${id}/checks`, { credentials: 'same-origin' }));
+      if (isCurrent()) setState((current) => ({ ...current, histories: {
+        ...current.histories, [id]: { status: 'success', data, error: null },
+      } }));
+    } catch (failure) {
+      if (!isCurrent()) return;
+      const error = handleFailure(failure);
+      if (isCurrent()) setState((current) => ({ ...current, histories: {
+        ...current.histories, [id]: { status: 'error', data: current.histories[id]?.data ?? null, error },
+      } }));
+    }
+  }, [handleFailure]);
+
+  async function mutate(action: Mutation): Promise<boolean> {
+    const key = 'id' in action ? `monitor:${action.id}` : action.kind;
+    // Admission is synchronous, before CSRF or POST. Rendered disabled buttons
+    // alone cannot protect a repeated submit event. Conflicting Monitor writes
+    // share one key; unrelated Monitors and create can still progress separately.
+    if (pending.current.has(key) || pending.current.has('logout') || (action.kind === 'logout' && pending.current.size > 0)) return false;
+    pending.current.add(key);
+    const generation = lifetime.current;
+    setState((current) => ({ ...current, error: null, mutations: {
+      ...current.mutations, [key]: { kind: action.kind, status: 'pending', error: null },
+    } }));
+    try {
+      const path = action.kind === 'create' ? '/api/monitors' : action.kind === 'logout' ? '/api/auth/logout'
+        : `/api/monitors/${action.id}${action.kind === 'check' ? '/checks' : ''}`;
+      const options: RequestInit = { method: action.kind === 'delete' ? 'DELETE' : action.kind === 'update' ? 'PUT' : 'POST' };
+      if ('input' in action) {
+        options.headers = { 'content-type': 'application/json' };
+        options.body = JSON.stringify(action.input);
+      }
+      const data = await responseData<MonitorView | CheckRun | { id: string } | { loggedOut: true }>(await mutationFetch(path, options));
+      if (generation !== lifetime.current) return false;
+      if (action.kind === 'logout') { clearSession(); return true; }
+      const refreshHistory = action.kind === 'check' && historyVersions.current.has(action.id);
+      if (action.kind === 'delete' || (action.kind === 'check' && refreshHistory)) {
+        // A read started before this mutation must not restore deleted or stale
+        // history, even if its response arrives after the new authoritative one.
+        historyVersions.current.set(action.id, (historyVersions.current.get(action.id) ?? 0) + 1);
+      }
+      setState((current) => {
+        let monitors = current.monitors;
+        let histories = current.histories;
+        if (action.kind === 'create') monitors = [...monitors, data as MonitorView];
+        else if (action.kind === 'update') monitors = monitors.map((monitor) => monitor.id === action.id ? data as MonitorView : monitor);
+        else if (action.kind === 'delete') {
+          monitors = monitors.filter((monitor) => monitor.id !== action.id);
+          histories = { ...histories };
+          delete histories[action.id];
+        } else if (action.kind === 'check') {
+          monitors = monitors.map((monitor) => monitor.id === action.id ? { ...monitor, latestCheck: data as CheckRun } : monitor);
+        }
+        return { ...current, monitors, histories, mutations: {
+          ...current.mutations, [key]: { kind: action.kind, status: 'success', error: null },
+        } };
+      });
+      // Mutation acknowledgement and query refresh are separate states: a failed
+      // history read must never label an already committed Check as a failed POST.
+      if (refreshHistory && 'id' in action) void loadHistory(action.id, false);
+      return true;
+    } catch (failure) {
+      if (generation !== lifetime.current) return false;
+      const error = handleFailure(failure);
+      if (generation === lifetime.current) setState((current) => ({ ...current, mutations: {
+        ...current.mutations, [key]: { kind: action.kind, status: 'error', error },
+      } }));
+      return false;
+    } finally {
+      if (generation === lifetime.current) pending.current.delete(key);
+    }
+  }
+
+  return { ...state, mutate, loadHistory };
+}
diff --git a/evidence/E06/browser.json b/evidence/E06/browser.json
new file mode 100644
index 0000000..794b030
--- /dev/null
+++ b/evidence/E06/browser.json
@@ -0,0 +1,17 @@
+{
+  "scenarioSha256": "be0ba3ef17b67140d17266028b257b41bb8ec2d4138beeb051f73681ecfd899b",
+  "harnessSha256": "c0ab1374a9f46658fcac642dc77095cb6abd39a8d204afe4095484c4303ddb56",
+  "result": "PASS",
+  "sequentialCreateUpdatePauseResumeDeleteAndCheckAlreadyCorrect": true,
+  "submitEvents": 2,
+  "csrfReadsAfterSecondSubmit": 1,
+  "forwardedCreateRequests": 1,
+  "authoritativeCreatedRows": 1,
+  "pendingVisibleBeforeSecondSubmit": true,
+  "heldResponses": 1,
+  "failedUpdates": 1,
+  "failedDeletes": 1,
+  "failedMutationsPreservedAuthoritativeState": true,
+  "pendingClearedAfterRelease": true,
+  "durationMs": 1870
+}
diff --git a/evidence/E06/verification.json b/evidence/E06/verification.json
new file mode 100644
index 0000000..b424390
--- /dev/null
+++ b/evidence/E06/verification.json
@@ -0,0 +1,110 @@
+{
+  "thread": "E06",
+  "attempt": 1,
+  "specRevision": "0a006589477f8ae47bad3faa5510c999cff85ee4",
+  "start": "33adeab16aab2b56ee2314e0324ab5c46cbf47c0",
+  "scenarioSha256": "be0ba3ef17b67140d17266028b257b41bb8ec2d4138beeb051f73681ecfd899b",
+  "harnessSha256": "c0ab1374a9f46658fcac642dc77095cb6abd39a8d204afe4095484c4303ddb56",
+  "runtime": {
+    "node": "24.19.0",
+    "npm": "11.17.0",
+    "dependenciesAdded": [],
+    "pinsChanged": false
+  },
+  "baseline": {
+    "command": "fnm exec --using 24.19.0 node evidence/E06/baseline-reproducer.mjs",
+    "exitCode": 0,
+    "unchangedStart": true,
+    "durationMs": 1426,
+    "result": "REPRODUCED",
+    "csrfReads": 2,
+    "createRequests": 2,
+    "authoritativeCreatedRows": 2,
+    "sequentialLifecycleAlreadyCorrect": true,
+    "failureInjectionsSkippedAfterFirstDecisiveFailure": true
+  },
+  "preliminaryTypecheck": {
+    "command": "fnm exec --using 24.19.0 npm run typecheck",
+    "exitCode": 0,
+    "durationMs": null,
+    "note": "One preliminary pass before the final timed gates; exact duration was not recorded."
+  },
+  "commands": [
+    {
+      "command": "fnm exec --using 24.19.0 npm run typecheck",
+      "exitCode": 0,
+      "durationMs": 1591
+    },
+    {
+      "command": "fnm exec --using 24.19.0 npm run test:unit",
+      "exitCode": 0,
+      "durationMs": 1294,
+      "passed": 13,
+      "failed": 0
+    },
+    {
+      "command": "fnm exec --using 24.19.0 npm run test:functional",
+      "exitCode": 0,
+      "durationMs": 8473,
+      "passed": 15,
+      "failed": 0
+    },
+    {
+      "command": "fnm exec --using 24.19.0 npm run test:integration",
+      "exitCode": 0,
+      "durationMs": 5694,
+      "passed": 9,
+      "failed": 0
+    },
+    {
+      "command": "fnm exec --using 24.19.0 npm run test:e2e",
+      "exitCode": 0,
+      "durationMs": 16406,
+      "passed": 8,
+      "failed": 0
+    },
+    {
+      "command": "fnm exec --using 24.19.0 npm run build",
+      "exitCode": 0,
+      "durationMs": 3210
+    }
+  ],
+  "acceptance": {
+    "singleRouteServerStateOwner": "app/monitors/use-monitors.ts",
+    "componentOnlyState": [
+      "editing panel",
+      "selected history panel",
+      "form inputs"
+    ],
+    "canonicalCreateUpdateDeleteLatestAndHistory": true,
+    "pendingSuccessFailureStates": true,
+    "guardedCreateRequests": 1,
+    "guardedAuthoritativeCreatedRows": 1,
+    "failedUpdates": 1,
+    "failedDeletes": 1,
+    "failedMutationsPreservedAuthoritativeState": true,
+    "pendingClearedAfterRelease": true,
+    "earlierApiPostgresAuthOwnershipBrowserTestsPassed": true,
+    "earlierFixturesScenariosMigrationsRuntimeAndCiUnchanged": true
+  },
+  "budget": {
+    "baselineRuns": 1,
+    "finalBrowserRuns": 1,
+    "browserWorkers": 1,
+    "browserRetries": 0,
+    "loadRuns": 0,
+    "automaticRetries": 0,
+    "parameterSweeps": 0,
+    "repairTasks": 0
+  },
+  "cleanup": {
+    "listenersOn4311Through4314": 0,
+    "remainingE03E04E05E06Schemas": 0,
+    "rootPostgres15431HealthyAndLeftRunning": true,
+    "publicSchemaAndVolumeUntouched": true
+  },
+  "evidencePolicy": "Safe operation names, counts, statuses and booleans only; no credentials, session or CSRF values, traces or videos.",
+  "hostedCiRunClaimed": false,
+  "outOfScopeChanges": [],
+  "unresolved": []
+}
diff --git a/test/browser/server-state.spec.ts b/test/browser/server-state.spec.ts
new file mode 100644
index 0000000..868cfe3
--- /dev/null
+++ b/test/browser/server-state.spec.ts
@@ -0,0 +1,14 @@
+import { expect } from '@playwright/test';
+import { createHash } from 'node:crypto';
+import { mkdir, readFile, writeFile } from 'node:fs/promises';
+import { test } from './session';
+import { runServerStateScenario } from '../../evidence/E06/browser-scenario';
+
+test('E06 authoritative list/detail refresh, response barrier, duplicate submit and failed mutations', async ({ page }) => {
+  const result = await runServerStateScenario(page, 'verification');
+  expect(result.result).toBe('PASS');
+  const scenarioSha256 = createHash('sha256').update(await readFile('evidence/E06/scenario.json')).digest('hex');
+  const harnessSha256 = createHash('sha256').update(await readFile('evidence/E06/browser-scenario.ts')).digest('hex');
+  await mkdir('output/e06', { recursive: true });
+  await writeFile('output/e06/browser.json', JSON.stringify({ scenarioSha256, harnessSha256, ...result }, null, 2) + '\n');
+});
