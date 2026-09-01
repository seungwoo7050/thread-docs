## `feat(rendering): seed request-scoped server state for hydration`

diff --git a/app/login/login-form.tsx b/app/login/login-form.tsx
new file mode 100644
index 0000000..5ef29cb
--- /dev/null
+++ b/app/login/login-form.tsx
@@ -0,0 +1,39 @@
+'use client';
+
+import { useState, type FormEvent } from 'react';
+import { errorMessages, failureCode, login, type ApiErrorCode } from '../monitors/api';
+
+export default function LoginForm() {
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
+  return <section aria-label="Sign in">
+      <form onSubmit={submit} method="post" action="/api/session/login">
+        <label>Username<input name="username" autoComplete="username" required maxLength={100} /></label>
+        <label>Password<input name="password" type="password" autoComplete="current-password" required /></label>
+        <button disabled={busy}>{busy ? 'Signing in…' : 'Sign in'}</button>
+      </form>
+      {error && <p role="alert" className="error" data-error-code={error}>
+        {error === 'UNAUTHENTICATED' ? 'Sign-in failed. Check your username and password.' : errorMessages[error]}
+      </p>}
+    </section>;
+}
diff --git a/app/login/page.tsx b/app/login/page.tsx
index ad869c7..d1194c9 100644
--- a/app/login/page.tsx
+++ b/app/login/page.tsx
@@ -1,43 +1,9 @@
-'use client';
-
-import { useState, type FormEvent } from 'react';
-import { errorMessages, failureCode, login, type ApiErrorCode } from '../monitors/api';
+import LoginForm from './login-form';
 
 export default function Login() {
-  const [busy, setBusy] = useState(false);
-  const [error, setError] = useState<ApiErrorCode | null>(null);
-
-  async function submit(event: FormEvent<HTMLFormElement>) {
-    event.preventDefault();
-    const form = event.currentTarget;
-    const fields = new FormData(form);
-    setBusy(true);
-    setError(null);
-    try {
-      await login(String(fields.get('username') ?? ''), String(fields.get('password') ?? ''));
-      form.reset();
-      window.location.replace('/monitors');
-    } catch (error) {
-      setError(failureCode(error));
-    } finally {
-      const password = form.elements.namedItem('password');
-      if (password instanceof HTMLInputElement) password.value = '';
-      setBusy(false);
-    }
-  }
-
   return <main>
     <header><p className="eyebrow">LOCAL LAB · INDUSTRY / SPRING</p><h1>Sign in</h1>
       <p>Use a fixture account prepared through the local bootstrap command.</p></header>
-    <section aria-label="Sign in">
-      <form onSubmit={submit} method="post" action="/api/session/login">
-        <label>Username<input name="username" autoComplete="username" required maxLength={100} /></label>
-        <label>Password<input name="password" type="password" autoComplete="current-password" required /></label>
-        <button disabled={busy}>{busy ? 'Signing in…' : 'Sign in'}</button>
-      </form>
-      {error && <p role="alert" className="error" data-error-code={error}>
-        {error === 'UNAUTHENTICATED' ? 'Sign-in failed. Check your username and password.' : errorMessages[error]}
-      </p>}
-    </section>
+    <LoginForm />
   </main>;
 }
diff --git a/app/monitors/api.ts b/app/monitors/api.ts
index 59f86b3..cff2e64 100644
--- a/app/monitors/api.ts
+++ b/app/monitors/api.ts
@@ -6,6 +6,10 @@ export type CheckRun = {
 export type MonitorView = { monitor: Monitor; latestCheck: CheckRun | null };
 export type HistorySelection = { monitorId: string; limit: string; state: string | null; cursor: string | null };
 export type HistoryPage = { items: CheckRun[]; nextCursor: string | null };
+export type InitialMonitorState = {
+  monitors: MonitorView[]; history: { selection: HistorySelection; page: HistoryPage } | null;
+  loaded: boolean; error: ApiErrorCode | null;
+};
 export type ApiErrorCode = 'INVALID_INPUT' | 'NOT_FOUND' | 'UNAUTHENTICATED' | 'FORBIDDEN' | 'INTERNAL_ERROR';
 
 export const errorMessages: Record<ApiErrorCode, string> = {
@@ -79,7 +83,11 @@ async function readData<T>(response: Response, valid: (value: unknown) => value
 }
 
 export async function loadMonitors(): Promise<MonitorView[]> {
-  return readData(await fetch('/api/monitors', { cache: 'no-store' }), (value): value is MonitorView[] =>
+  return readMonitors(await fetch('/api/monitors', { cache: 'no-store' }));
+}
+
+export function readMonitors(response: Response): Promise<MonitorView[]> {
+  return readData(response, (value): value is MonitorView[] =>
     Array.isArray(value) && value.every(isMonitorView));
 }
 
@@ -120,11 +128,18 @@ export async function deleteMonitor(id: string): Promise<null> {
   return readData(await mutation(`/api/monitors/${id}`, 'DELETE'), (value): value is null => value === null);
 }
 
-export async function loadChecks(selection: HistorySelection): Promise<HistoryPage> {
+export function historyPath(selection: HistorySelection): string {
   const query = new URLSearchParams({ limit: selection.limit });
   if (selection.state !== null) query.set('state', selection.state);
   if (selection.cursor !== null) query.set('cursor', selection.cursor);
-  const response = await fetch(`/api/monitors/${selection.monitorId}/checks?${query}`, { cache: 'no-store' });
+  return `/api/monitors/${encodeURIComponent(selection.monitorId)}/checks?${query}`;
+}
+
+export async function loadChecks(selection: HistorySelection): Promise<HistoryPage> {
+  return readHistoryPage(await fetch(historyPath(selection), { cache: 'no-store' }));
+}
+
+export async function readHistoryPage(response: Response): Promise<HistoryPage> {
   const items = await readData(response, (value): value is CheckRun[] =>
     Array.isArray(value) && value.every(isCheckRun));
   return { items, nextCursor: response.headers.get('X-Next-Cursor') };
diff --git a/app/monitors/monitor-controls.tsx b/app/monitors/monitor-controls.tsx
new file mode 100644
index 0000000..c142fc6
--- /dev/null
+++ b/app/monitors/monitor-controls.tsx
@@ -0,0 +1,130 @@
+'use client';
+
+import { useState, type FormEvent } from 'react';
+import { useRouter, useSearchParams } from 'next/navigation';
+import { errorMessages, type InitialMonitorState, type Monitor } from './api';
+import { useMonitorState } from './use-monitor-state';
+
+function inputFrom(form: HTMLFormElement): Omit<Monitor, 'id'> {
+  const fields = new FormData(form);
+  return { name: String(fields.get('name') ?? ''), url: String(fields.get('url') ?? ''),
+    interval: Number(fields.get('interval')), enabled: fields.get('enabled') === 'on' };
+}
+
+export default function MonitorControls({ initial }: { initial: InitialMonitorState }) {
+  const router = useRouter();
+  const search = useSearchParams();
+  const historyId = search.get('history');
+  const state = useMonitorState(historyId ? { monitorId: historyId, limit: search.get('limit') ?? '20',
+    state: search.get('state'), cursor: search.get('cursor') } : null, initial);
+  const [editingId, setEditingId] = useState<string | null>(null);
+
+  function historyUrl(changes: Record<string, string | null>, replace = false) {
+    const query = new URLSearchParams(search.toString());
+    for (const [key, value] of Object.entries(changes)) {
+      if (value === null) query.delete(key); else query.set(key, value);
+    }
+    const url = `/monitors${query.size ? `?${query}` : ''}`;
+    if (replace) router.replace(url, { scroll: false }); else router.push(url, { scroll: false });
+  }
+
+  async function create(event: FormEvent<HTMLFormElement>) {
+    event.preventDefault();
+    const form = event.currentTarget;
+    if (await state.create(inputFrom(form))) form.reset();
+  }
+
+  async function update(id: string, input: Omit<Monitor, 'id'>) {
+    if (await state.update(id, input)) setEditingId(null);
+  }
+
+  async function remove(id: string) {
+    if (await state.remove(id)) {
+      if (historyId === id) historyUrl({ history: null, state: null, limit: null, cursor: null }, true);
+      if (editingId === id) setEditingId(null);
+    }
+  }
+
+  function history(id: string) {
+    historyUrl({ history: historyId === id ? null : id, state: null,
+      limit: historyId === id ? null : '3', cursor: null });
+  }
+
+  return <>
+    <button disabled={state.anyPending || state.loading} onClick={() => void state.signOut()}>Sign out</button>
+    <section aria-labelledby="create-title">
+      <h2 id="create-title">Create monitor</h2>
+      <form onSubmit={create}>
+        <label>Name<input name="name" required defaultValue="Fixture monitor" /></label>
+        <label>URL<input name="url" type="url" required defaultValue="http://127.0.0.1:4321/ok" /></label>
+        <label>Interval (seconds)<input name="interval" type="number" min="1" max="86400" step="1" required defaultValue="60" /></label>
+        <label className="checkbox"><input name="enabled" type="checkbox" defaultChecked />Enabled</label>
+        <p className="hint">Interval and enabled are stored. Checks still run manually; there is no scheduler.</p>
+        <button disabled={!state.loaded || state.pending('create') || state.pending('logout')}>Create monitor</button>
+      </form>
+      {state.pending('create') && <p role="status">Creating monitor…</p>}
+      {state.operations.create === 'success' && <p role="status">Monitor created.</p>}
+    </section>
+    {state.error && <p role="alert" className="error" data-error-code={state.error}>{errorMessages[state.error]}</p>}
+    <section aria-labelledby="monitors-title"><h2 id="monitors-title">Monitors</h2>
+      {state.loading && <p role="status">Loading monitors…</p>}
+      {state.loaded && state.monitors.length === 0 && <p>No monitors yet.</p>}
+      {state.monitors.map(({ monitor, latestCheck }) => {
+        const busy = state.pending(monitor.id) || state.pending('logout');
+        const visibleHistory = historyId === monitor.id;
+        const checks = visibleHistory ? state.historyPage?.items : undefined;
+        return <article key={monitor.id} aria-label={monitor.name}>
+        <h3>{monitor.name}</h3><p className="url">{monitor.url}</p>
+        <p>{monitor.interval}s interval · {monitor.enabled ? 'Enabled' : 'Paused'}</p>
+        <div className="actions">
+          <button disabled={busy} onClick={() => void state.check(monitor.id)}>Run check</button>
+          <button disabled={busy} onClick={() => setEditingId(monitor.id)}>Edit</button>
+          <button disabled={busy} onClick={() => update(monitor.id, { name: monitor.name, url: monitor.url,
+            interval: monitor.interval, enabled: !monitor.enabled })}>{monitor.enabled ? 'Pause' : 'Activate'}</button>
+          <button disabled={state.operations[monitor.id] === 'pending' || state.pending('logout')}
+            onClick={() => history(monitor.id)}>{visibleHistory ? 'Hide history' : 'Show history'}</button>
+          <button disabled={busy} onClick={() => remove(monitor.id)}>Delete</button>
+        </div>
+        {busy && <p role="status">Request pending…</p>}
+        {editingId === monitor.id && <form aria-label="Edit monitor" onSubmit={event => {
+          event.preventDefault();
+          void update(monitor.id, inputFrom(event.currentTarget));
+        }}>
+          <label>Edit name<input name="name" required defaultValue={monitor.name} /></label>
+          <label>Edit URL<input name="url" type="url" required defaultValue={monitor.url} /></label>
+          <label>Edit interval (seconds)<input name="interval" type="number" min="1" max="86400" step="1"
+            required defaultValue={monitor.interval} /></label>
+          <label className="checkbox"><input name="enabled" type="checkbox" defaultChecked={monitor.enabled} />Edit enabled</label>
+          <div className="actions"><button disabled={busy}>Save changes</button>
+            <button type="button" disabled={busy} onClick={() => setEditingId(null)}>Cancel</button></div>
+        </form>}
+        <div aria-live="polite" className="result" data-latest-check-id={latestCheck?.id}>
+          {latestCheck ? <>
+            <strong>{latestCheck.state}</strong>
+            <span>HTTP {latestCheck.httpStatus ?? '—'}</span>
+            <span>{latestCheck.latencyMs} ms</span>
+            {latestCheck.failureReason && <span>{latestCheck.failureReason}</span>}
+          </> : <span>No checks yet.</span>}
+        </div>
+        {visibleHistory && <section aria-label="Historical checks">
+          <h4>Check history</h4>
+          <label>Check result<select value={search.get('state') ?? ''}
+            onChange={event => historyUrl({ state: event.target.value || null, cursor: null })}>
+            <option value="">All</option><option value="SUCCEEDED">SUCCEEDED</option><option value="FAILED">FAILED</option>
+          </select></label>
+          {checks === undefined ? <p>{state.historyPending ? 'Loading history…' : 'History could not be loaded.'}</p>
+            : checks.length === 0 ? <p>No historical checks.</p> : <div className="history-table">
+            <table aria-label="Check history"><thead><tr><th>Finished (UTC)</th><th>Result</th><th>HTTP</th><th>Latency</th><th>Reason</th></tr></thead>
+              <tbody>{checks.map(check => <tr key={check.id} data-check-id={check.id}>
+                <td><time dateTime={check.finishedAt}>{check.finishedAt}</time></td><td>{check.state}</td>
+                <td>{check.httpStatus ?? '—'}</td><td>{check.latencyMs} ms</td><td>{check.failureReason ?? '—'}</td>
+              </tr>)}</tbody>
+            </table>
+          </div>}
+          <button disabled={busy || !state.historyPage?.nextCursor}
+            onClick={() => historyUrl({ cursor: state.historyPage?.nextCursor ?? null })}>Next page</button>
+        </section>}
+      </article>; })}
+    </section>
+  </>;
+}
diff --git a/app/monitors/page.tsx b/app/monitors/page.tsx
index 3b2240d..c655020 100644
--- a/app/monitors/page.tsx
+++ b/app/monitors/page.tsx
@@ -1,137 +1,16 @@
-'use client';
+import MonitorControls from './monitor-controls';
+import { loadInitialMonitorState } from './server-data';
 
-import { Suspense, useState, type FormEvent } from 'react';
-import { useRouter, useSearchParams } from 'next/navigation';
-import { errorMessages, type Monitor } from './api';
-import { useMonitorState } from './use-monitor-state';
-
-function inputFrom(form: HTMLFormElement): Omit<Monitor, 'id'> {
-  const fields = new FormData(form);
-  return { name: String(fields.get('name') ?? ''), url: String(fields.get('url') ?? ''),
-    interval: Number(fields.get('interval')), enabled: fields.get('enabled') === 'on' };
-}
-
-export default function Monitors() {
-  return <Suspense fallback={<main><p role="status">Loading monitors…</p></main>}><MonitorContent /></Suspense>;
-}
-
-function MonitorContent() {
-  const router = useRouter();
-  const search = useSearchParams();
-  const historyId = search.get('history');
-  const state = useMonitorState(historyId ? { monitorId: historyId, limit: search.get('limit') ?? '20',
-    state: search.get('state'), cursor: search.get('cursor') } : null);
-  const [editingId, setEditingId] = useState<string | null>(null);
-
-  function historyUrl(changes: Record<string, string | null>, replace = false) {
-    const query = new URLSearchParams(search.toString());
-    for (const [key, value] of Object.entries(changes)) {
-      if (value === null) query.delete(key); else query.set(key, value);
-    }
-    const url = `/monitors${query.size ? `?${query}` : ''}`;
-    if (replace) router.replace(url, { scroll: false }); else router.push(url, { scroll: false });
-  }
-
-  async function create(event: FormEvent<HTMLFormElement>) {
-    event.preventDefault();
-    const form = event.currentTarget;
-    if (await state.create(inputFrom(form))) form.reset();
-  }
-
-  async function update(id: string, input: Omit<Monitor, 'id'>) {
-    if (await state.update(id, input)) setEditingId(null);
-  }
-
-  async function remove(id: string) {
-    if (await state.remove(id)) {
-      if (historyId === id) historyUrl({ history: null, state: null, limit: null, cursor: null }, true);
-      if (editingId === id) setEditingId(null);
-    }
-  }
-
-  function history(id: string) {
-    historyUrl({ history: historyId === id ? null : id, state: null,
-      limit: historyId === id ? null : '3', cursor: null });
-  }
+export const dynamic = 'force-dynamic';
 
+export default async function Monitors({ searchParams }: {
+  searchParams: Promise<Record<string, string | string[] | undefined>>;
+}) {
+  const initial = await loadInitialMonitorState(await searchParams);
   return <main>
     <header><p className="eyebrow">LOCAL LAB · INDUSTRY / SPRING</p><h1>Endpoint Monitor</h1>
-      <p>Create a monitor and check its HTTP response.</p>
-      <button disabled={state.anyPending || state.loading} onClick={() => void state.signOut()}>Sign out</button></header>
+      <p>Create a monitor and check its HTTP response.</p></header>
     <aside>Development fixture only. Monitors and completed checks are stored in PostgreSQL.</aside>
-    <section aria-labelledby="create-title">
-      <h2 id="create-title">Create monitor</h2>
-      <form onSubmit={create}>
-        <label>Name<input name="name" required defaultValue="Fixture monitor" /></label>
-        <label>URL<input name="url" type="url" required defaultValue="http://127.0.0.1:4321/ok" /></label>
-        <label>Interval (seconds)<input name="interval" type="number" min="1" max="86400" step="1" required defaultValue="60" /></label>
-        <label className="checkbox"><input name="enabled" type="checkbox" defaultChecked />Enabled</label>
-        <p className="hint">Interval and enabled are stored. Checks still run manually; there is no scheduler.</p>
-        <button disabled={!state.loaded || state.pending('create') || state.pending('logout')}>Create monitor</button>
-      </form>
-      {state.pending('create') && <p role="status">Creating monitor…</p>}
-      {state.operations.create === 'success' && <p role="status">Monitor created.</p>}
-    </section>
-    {state.error && <p role="alert" className="error" data-error-code={state.error}>{errorMessages[state.error]}</p>}
-    <section aria-labelledby="monitors-title"><h2 id="monitors-title">Monitors</h2>
-      {state.loading && <p role="status">Loading monitors…</p>}
-      {state.loaded && state.monitors.length === 0 && <p>No monitors yet.</p>}
-      {state.monitors.map(({ monitor, latestCheck }) => {
-        const busy = state.pending(monitor.id) || state.pending('logout');
-        const visibleHistory = historyId === monitor.id;
-        const checks = visibleHistory ? state.historyPage?.items : undefined;
-        return <article key={monitor.id} aria-label={monitor.name}>
-        <h3>{monitor.name}</h3><p className="url">{monitor.url}</p>
-        <p>{monitor.interval}s interval · {monitor.enabled ? 'Enabled' : 'Paused'}</p>
-        <div className="actions">
-          <button disabled={busy} onClick={() => void state.check(monitor.id)}>Run check</button>
-          <button disabled={busy} onClick={() => setEditingId(monitor.id)}>Edit</button>
-          <button disabled={busy} onClick={() => update(monitor.id, { name: monitor.name, url: monitor.url,
-            interval: monitor.interval, enabled: !monitor.enabled })}>{monitor.enabled ? 'Pause' : 'Activate'}</button>
-          <button disabled={state.operations[monitor.id] === 'pending' || state.pending('logout')}
-            onClick={() => history(monitor.id)}>{visibleHistory ? 'Hide history' : 'Show history'}</button>
-          <button disabled={busy} onClick={() => remove(monitor.id)}>Delete</button>
-        </div>
-        {busy && <p role="status">Request pending…</p>}
-        {editingId === monitor.id && <form aria-label="Edit monitor" onSubmit={event => {
-          event.preventDefault();
-          void update(monitor.id, inputFrom(event.currentTarget));
-        }}>
-          <label>Edit name<input name="name" required defaultValue={monitor.name} /></label>
-          <label>Edit URL<input name="url" type="url" required defaultValue={monitor.url} /></label>
-          <label>Edit interval (seconds)<input name="interval" type="number" min="1" max="86400" step="1"
-            required defaultValue={monitor.interval} /></label>
-          <label className="checkbox"><input name="enabled" type="checkbox" defaultChecked={monitor.enabled} />Edit enabled</label>
-          <div className="actions"><button disabled={busy}>Save changes</button>
-            <button type="button" disabled={busy} onClick={() => setEditingId(null)}>Cancel</button></div>
-        </form>}
-        <div aria-live="polite" className="result">
-          {latestCheck ? <>
-            <strong>{latestCheck.state}</strong>
-            <span>HTTP {latestCheck.httpStatus ?? '—'}</span>
-            <span>{latestCheck.latencyMs} ms</span>
-            {latestCheck.failureReason && <span>{latestCheck.failureReason}</span>}
-          </> : <span>No checks yet.</span>}
-        </div>
-        {visibleHistory && <section aria-label="Historical checks">
-          <h4>Check history</h4>
-          <label>Check result<select value={search.get('state') ?? ''}
-            onChange={event => historyUrl({ state: event.target.value || null, cursor: null })}>
-            <option value="">All</option><option value="SUCCEEDED">SUCCEEDED</option><option value="FAILED">FAILED</option>
-          </select></label>
-          {checks === undefined ? <p>{state.historyPending ? 'Loading history…' : 'History could not be loaded.'}</p>
-            : checks.length === 0 ? <p>No historical checks.</p> : <div className="history-table">
-            <table aria-label="Check history"><thead><tr><th>Finished (UTC)</th><th>Result</th><th>HTTP</th><th>Latency</th><th>Reason</th></tr></thead>
-              <tbody>{checks.map(check => <tr key={check.id} data-check-id={check.id}>
-                <td><time dateTime={check.finishedAt}>{check.finishedAt}</time></td><td>{check.state}</td>
-                <td>{check.httpStatus ?? '—'}</td><td>{check.latencyMs} ms</td><td>{check.failureReason ?? '—'}</td>
-              </tr>)}</tbody>
-            </table>
-          </div>}
-          <button disabled={busy || !state.historyPage?.nextCursor}
-            onClick={() => historyUrl({ cursor: state.historyPage?.nextCursor ?? null })}>Next page</button>
-        </section>}
-      </article>; })}
-    </section>
+    <MonitorControls initial={initial} />
   </main>;
 }
diff --git a/app/monitors/server-data.ts b/app/monitors/server-data.ts
new file mode 100644
index 0000000..ad0680f
--- /dev/null
+++ b/app/monitors/server-data.ts
@@ -0,0 +1,32 @@
+import 'server-only';
+
+import { cookies } from 'next/headers';
+import { redirect } from 'next/navigation';
+import { failureCode, historyPath, readHistoryPage, readMonitors, type InitialMonitorState } from './api';
+
+export async function loadInitialMonitorState(query: Record<string, string | string[] | undefined>): Promise<InitialMonitorState> {
+  const session = (await cookies()).get('WSESESSION');
+  if (!session) redirect('/login');
+  // Credentials live only in this request's server closure. Only canonical
+  // product data and stable error codes cross the Server/Client boundary.
+  const read = (path: string) => fetch(`${process.env.API_ORIGIN ?? 'http://127.0.0.1:4322'}${path}`, {
+    cache: 'no-store', headers: { Cookie: `WSESESSION=${session.value}` },
+  });
+  const initial: InitialMonitorState = { monitors: [], history: null, loaded: false, error: null };
+  try {
+    initial.monitors = await readMonitors(await read('/api/monitors'));
+    initial.loaded = true;
+    const first = (key: string) => Array.isArray(query[key]) ? query[key][0] : query[key];
+    const monitorId = first('history');
+    if (monitorId) {
+      const selection = { monitorId, limit: first('limit') ?? '20', state: first('state') ?? null,
+        cursor: first('cursor') ?? null };
+      initial.history = { selection, page: await readHistoryPage(await read(historyPath(selection))) };
+    }
+  } catch (error) {
+    const code = failureCode(error);
+    if (code === 'UNAUTHENTICATED') redirect('/login');
+    initial.error = code;
+  }
+  return initial;
+}
diff --git a/app/monitors/use-monitor-state.ts b/app/monitors/use-monitor-state.ts
index d31dc43..46087cd 100644
--- a/app/monitors/use-monitor-state.ts
+++ b/app/monitors/use-monitor-state.ts
@@ -2,7 +2,8 @@
 
 import { useEffect, useRef, useState } from 'react';
 import { createMonitor, deleteMonitor, failureCode, loadChecks, loadMonitors, logout,
-  replaceMonitor, runCheck, type ApiErrorCode, type HistoryPage, type HistorySelection, type Monitor, type MonitorView } from './api';
+  replaceMonitor, runCheck, type ApiErrorCode, type HistoryPage, type HistorySelection, type InitialMonitorState,
+  type Monitor, type MonitorView } from './api';
 
 type Queries = { monitors: MonitorView[]; histories: Record<string, Record<string, HistoryPage>>; loaded: boolean };
 type Phase = 'pending' | 'success' | 'failure';
@@ -11,11 +12,19 @@ const emptyQueries = (): Queries => ({ monitors: [], histories: {}, loaded: fals
 
 // One page-scoped owner for remote data and request state. URL conditions select
 // history pages; no cache survives a session/page boundary.
-export function useMonitorState(selection: HistorySelection | null) {
-  const [queries, setQueries] = useState<Queries>(emptyQueries);
+export function useMonitorState(selection: HistorySelection | null, initial: InitialMonitorState) {
+  const [queries, setQueries] = useState<Queries>(() => {
+    const histories: Queries['histories'] = {};
+    if (initial.history) {
+      const { selection, page } = initial.history;
+      const key = JSON.stringify([selection.monitorId, selection.limit, selection.state, selection.cursor]);
+      histories[selection.monitorId] = { [key]: page };
+    }
+    return { monitors: initial.monitors, histories, loaded: initial.loaded };
+  });
   const [operations, setOperations] = useState<Record<string, Phase>>({});
-  const [error, setError] = useState<ApiErrorCode | null>(null);
-  const [loading, setLoading] = useState(true);
+  const [error, setError] = useState<ApiErrorCode | null>(initial.error);
+  const [loading, setLoading] = useState(!initial.loaded);
   const [historyRead, setHistoryRead] = useState<{ key: string; phase: Phase } | null>(null);
   const inFlight = useRef(new Set<string>());
   const { monitorId = null, limit = '20', state = null, cursor = null } = selection ?? {};
@@ -35,6 +44,7 @@ export function useMonitorState(selection: HistorySelection | null) {
 
   useEffect(() => {
     let active = true;
+    // Revalidate after hydration without discarding the identical SSR payload.
     loadMonitors().then(monitors => {
       if (active) setQueries(current => ({ ...current, monitors, loaded: true }));
     }).catch(error => {
diff --git a/tests/browser/history.spec.ts b/tests/browser/history.spec.ts
index 7b0250d..3f06e28 100644
--- a/tests/browser/history.spec.ts
+++ b/tests/browser/history.spec.ts
@@ -62,15 +62,17 @@ test('fixed tied history pages restore URL state and ignore the held older filte
   let heldReads = 0;
   let intercept: ((route: Route) => Promise<void>) | undefined;
   try {
-    const [first] = await Promise.all([response(), page.goto(`/monitors?history=${monitor}&limit=3`)]);
-    expect(first.status()).toBe(200);
-    const cursor2 = await first.headerValue('X-Next-Cursor');
-    expect(cursor2).not.toBeNull();
+    // E08 seeds the initial selected page through SSR. Observe the real document
+    // here; later client pages still consume the real API cursor response header.
+    const first = await page.goto(`/monitors?history=${monitor}&limit=3`);
+    expect(first?.status()).toBe(200);
     await urlState(null, null);
     await rows([7, 6, 5]);
     seed(monitor, 8, 8);
     const [second] = await Promise.all([response(), next.click()]);
     expect(second.status()).toBe(200);
+    const cursor2 = new URL(page.url()).searchParams.get('cursor');
+    expect(cursor2).not.toBeNull();
     const cursor3 = await second.headerValue('X-Next-Cursor');
     expect(cursor3).not.toBeNull();
     await urlState(null, cursor2);
@@ -88,8 +90,8 @@ test('fixed tied history pages restore URL state and ignore the held older filte
     await page.goForward();
     await urlState(null, cursor3);
     await rows([1]);
-    const [reloaded] = await Promise.all([response(), page.reload()]);
-    expect(reloaded.status()).toBe(200);
+    const reloaded = await page.reload();
+    expect(reloaded?.status()).toBe(200);
     await urlState(null, cursor3);
     await rows([1]);
     completed.push('cursor page back/forward/reload');
@@ -149,8 +151,8 @@ test('fixed tied history pages restore URL state and ignore the held older filte
     await page.goForward();
     await urlState(null, null);
     await rows([8, 7, 6]);
-    const [final] = await Promise.all([response(), page.reload()]);
-    expect(final.status()).toBe(200);
+    const final = await page.reload();
+    expect(final?.status()).toBe(200);
     await urlState(null, null);
     await rows([8, 7, 6]);
     completed.push('filter back/forward/reload');


