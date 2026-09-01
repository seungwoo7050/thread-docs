## `feat: render authenticated monitors from request data`

diff --git a/app/login/login-form.tsx b/app/login/login-form.tsx
new file mode 100644
index 0000000..c6c38ed
--- /dev/null
+++ b/app/login/login-form.tsx
@@ -0,0 +1,50 @@
+'use client';
+
+import { useState } from 'react';
+import type { FormEvent } from 'react';
+import type { ApiErrorCode } from '../../server/model';
+import { ERROR_MESSAGES, failureCode, responseData } from '../monitors/api';
+
+export default function LoginForm() {
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
+      // Replace the document so another account cannot reuse a prior user's
+      // client route cache. This browser API runs only after form submission.
+      window.location.replace('/monitors');
+    } catch (failure) { setError(failureCode(failure)); }
+    finally { setPending(false); }
+  }
+
+  const message = error === 'UNAUTHENTICATED' ? 'Username or password was not accepted.'
+    : error === 'INVALID_INPUT' ? 'Check the username and password format.'
+      : error ? ERROR_MESSAGES[error] : null;
+
+  return <>
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
+  </>;
+}
diff --git a/app/login/page.tsx b/app/login/page.tsx
index a1b5041..bff51ad 100644
--- a/app/login/page.tsx
+++ b/app/login/page.tsx
@@ -1,54 +1,11 @@
-'use client';
-
-import { useState } from 'react';
-import type { FormEvent } from 'react';
-import { useRouter } from 'next/navigation';
-import type { ApiErrorCode } from '../../server/model';
-import { ERROR_MESSAGES, failureCode, responseData } from '../monitors/api';
+import LoginForm from './login-form';
 
 export default function LoginPage() {
-  const router = useRouter();
-  const [pending, setPending] = useState(false);
-  const [error, setError] = useState<ApiErrorCode | null>(null);
-
-  async function signIn(event: FormEvent<HTMLFormElement>) {
-    event.preventDefault();
-    const form = event.currentTarget;
-    const fields = new FormData(form);
-    setPending(true);
-    setError(null);
-    try {
-      const response = fetch('/api/auth/login', {
-        method: 'POST', credentials: 'same-origin', headers: { 'content-type': 'application/json' },
-        body: JSON.stringify({ username: fields.get('username'), password: fields.get('password') }),
-      });
-      // The password is transient form input, never React state or browser storage.
-      fields.delete('password');
-      (form.elements.namedItem('password') as HTMLInputElement).value = '';
-      await responseData(await response);
-      router.replace('/monitors');
-    } catch (failure) { setError(failureCode(failure)); }
-    finally { setPending(false); }
-  }
-
-  const message = error === 'UNAUTHENTICATED' ? 'Username or password was not accepted.'
-    : error === 'INVALID_INPUT' ? 'Check the username and password format.'
-      : error ? ERROR_MESSAGES[error] : null;
-
   return <main>
     <header><p className="eyebrow">HTTP ENDPOINT MONITOR · LOCAL FIXTURE</p><h1>Sign in</h1>
       <p>Sign in to create monitors and inspect their check history.</p>
     </header>
     <aside>Development only. Use an account prepared by the documented local fixture.</aside>
-    {error && <p role="alert" className="error" data-error-code={error}>{message}</p>}
-    <section className="panel" aria-label="Sign in form">
-      <form onSubmit={signIn}>
-        <label htmlFor="username">Username</label>
-        <input id="username" name="username" autoComplete="username" required minLength={3} maxLength={64} />
-        <label htmlFor="password">Password</label>
-        <input id="password" name="password" type="password" autoComplete="current-password" required maxLength={1024} />
-        <button type="submit" disabled={pending}>{pending ? 'Signing in…' : 'Sign in'}</button>
-      </form>
-    </section>
+    <LoginForm />
   </main>;
 }
diff --git a/app/monitors/initial-state.ts b/app/monitors/initial-state.ts
new file mode 100644
index 0000000..cfbdc84
--- /dev/null
+++ b/app/monitors/initial-state.ts
@@ -0,0 +1,24 @@
+import type { ApiErrorCode, CheckHistoryPage, MonitorView } from '../../server/model.ts';
+
+export type HistoryQuery = { search: string; status: 'pending' | 'success' | 'error'; data: CheckHistoryPage | null; error: ApiErrorCode | null };
+// Only public API data crosses the server/client boundary; never credentials.
+export type InitialMonitors = {
+  authenticated: boolean;
+  monitors: MonitorView[];
+  histories: Record<string, HistoryQuery>;
+  error: ApiErrorCode | null;
+};
+
+export function emptyMonitors(): InitialMonitors {
+  return { authenticated: false, monitors: [], histories: {}, error: null };
+}
+
+export function historyLocation(parameters: URLSearchParams) {
+  const id = parameters.get('monitor');
+  const history = new URLSearchParams(parameters);
+  history.delete('monitor');
+  // Next page searchParams groups repeated keys. Canonical order matches the
+  // browser snapshot without dropping malformed or repeated API parameters.
+  history.sort();
+  return { id, search: history.toString() };
+}
diff --git a/app/monitors/monitor-workspace.tsx b/app/monitors/monitor-workspace.tsx
new file mode 100644
index 0000000..e8bc9f6
--- /dev/null
+++ b/app/monitors/monitor-workspace.tsx
@@ -0,0 +1,167 @@
+'use client';
+
+import { useEffect, useState } from 'react';
+import type { FormEvent } from 'react';
+import { useSearchParams } from 'next/navigation';
+import type { MonitorView } from '../../server/model';
+import { ERROR_MESSAGES } from './api';
+import { historyLocation } from './initial-state';
+import type { InitialMonitors } from './initial-state';
+import { useMonitors } from './use-monitors';
+
+export default function MonitorWorkspace({ initial }: { initial: InitialMonitors }) {
+  const { authenticated, monitors, histories, mutations, error, mutate, ensureHistory } = useMonitors(initial);
+  const creating = mutations.create?.status === 'pending';
+  const loggingOut = mutations.logout?.status === 'pending';
+  const anyPending = Object.values(mutations).some((mutation) => mutation.status === 'pending');
+  const [editing, setEditing] = useState<string | null>(null);
+  const searchParams = useSearchParams();
+  const { id: historyId, search: historySearch } = historyLocation(new URLSearchParams(searchParams.toString()));
+
+  useEffect(() => {
+    if (authenticated && historyId) void ensureHistory(historyId, historySearch);
+  }, [authenticated, historyId, historySearch, ensureHistory]);
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
+
+  async function createMonitor(event: FormEvent<HTMLFormElement>) {
+    event.preventDefault();
+    const form = event.currentTarget;
+    const fields = new FormData(form);
+    if (await mutate({ kind: 'create', input: {
+      name: String(fields.get('name')), url: String(fields.get('url')),
+      interval: Number(fields.get('interval')), enabled: fields.get('enabled') === 'on',
+    } })) form.reset();
+  }
+
+  async function editMonitor(event: FormEvent<HTMLFormElement>, monitor: MonitorView) {
+    event.preventDefault();
+    const fields = new FormData(event.currentTarget);
+    if (await mutate({ kind: 'update', id: monitor.id, input: {
+      name: String(fields.get('name')), url: String(fields.get('url')),
+      interval: Number(fields.get('interval')), enabled: monitor.enabled,
+    } })) setEditing((current) => current === monitor.id ? null : current);
+  }
+
+  async function deleteMonitor(id: string) {
+    if (!window.confirm('Delete this monitor and all its check history?')) return;
+    if (await mutate({ kind: 'delete', id })) {
+      if (new URLSearchParams(window.location.search).get('monitor') === id) navigateHistory(null);
+      setEditing((current) => current === id ? null : current);
+    }
+  }
+
+  if (!authenticated) return error && error !== 'UNAUTHENTICATED'
+    ? <p role="alert" className="error" data-error-code={error}>{ERROR_MESSAGES[error]}</p>
+    : <p>Opening sign in…</p>;
+
+  return <>
+    <p><button onClick={() => mutate({ kind: 'logout' })} disabled={anyPending}>{loggingOut ? 'Signing out…' : 'Sign out'}</button></p>
+    {error && <p role="alert" className="error" data-error-code={error}>{ERROR_MESSAGES[error]}</p>}
+    <section aria-labelledby="create-heading" className="panel">
+      <h2 id="create-heading">Create monitor</h2>
+      <form onSubmit={createMonitor}>
+        <label htmlFor="name">Name</label>
+        <input id="name" name="name" required placeholder="Fixture monitor" disabled={creating || loggingOut} />
+        <label htmlFor="url">Endpoint URL</label>
+        <input id="url" name="url" type="url" required defaultValue="http://127.0.0.1:4311/ok" disabled={creating || loggingOut} />
+        <label htmlFor="interval">Interval (seconds)</label>
+        <input id="interval" name="interval" type="number" required min="1" max="86400" step="1" defaultValue="60" disabled={creating || loggingOut} />
+        <label className="checkbox"><input name="enabled" type="checkbox" defaultChecked disabled={creating || loggingOut} /> Enabled</label>
+        <p className="hint">Interval and enabled are saved; this version only runs manual checks.</p>
+        <button type="submit" disabled={creating || loggingOut}>{creating ? 'Creating…' : 'Create monitor'}</button>
+      </form>
+      {mutations.create?.status === 'success' && <p role="status">Monitor created.</p>}
+    </section>
+    <section aria-labelledby="saved-heading">
+      <h2 id="saved-heading">Your monitors</h2>
+      {monitors.length === 0 && <p>No monitors yet.</p>}
+      {monitors.map((monitor) => {
+        const mutation = mutations[`monitor:${monitor.id}`];
+        const pending = mutation?.status === 'pending';
+        const historyQuery = histories[monitor.id]?.search === historySearch ? histories[monitor.id] : undefined;
+        const history = historyQuery?.data?.items ?? [];
+        return <article key={monitor.id} className="panel" aria-labelledby={`monitor-${monitor.id}`}>
+        <h3 id={`monitor-${monitor.id}`}>{monitor.name}</h3>
+        <p className="endpoint">{monitor.url}</p>
+        <p>{monitor.enabled ? 'Enabled' : 'Paused'} · {monitor.interval} seconds</p>
+        <div className="actions">
+        <button onClick={() => mutate({ kind: 'check', id: monitor.id })} disabled={pending || loggingOut}>
+          {pending && mutation.kind === 'check' ? 'Checking…' : 'Run check'}
+        </button>
+        <button onClick={() => setEditing(editing === monitor.id ? null : monitor.id)} disabled={pending || loggingOut} aria-expanded={editing === monitor.id}>
+          {editing === monitor.id ? 'Cancel edit' : 'Edit monitor'}
+        </button>
+        <button onClick={() => mutate({ kind: 'update', id: monitor.id, input: { ...monitor, enabled: !monitor.enabled } })} disabled={pending || loggingOut}>
+          {monitor.enabled ? 'Pause monitor' : 'Enable monitor'}
+        </button>
+        <button onClick={() => deleteMonitor(monitor.id)} disabled={pending || loggingOut}>
+          {pending && mutation.kind === 'delete' ? 'Deleting…' : 'Delete monitor'}
+        </button>
+        <button onClick={() => navigateHistory(historyId === monitor.id ? null : monitor.id)} disabled={loggingOut} aria-expanded={historyId === monitor.id}>
+          {historyId === monitor.id ? 'Hide history' : 'View history'}
+        </button>
+        </div>
+        {mutation?.status === 'success' && <p role="status">{mutation.kind === 'check' ? 'Check completed.' : 'Monitor saved.'}</p>}
+        {editing === monitor.id && <form onSubmit={(event) => editMonitor(event, monitor)} className="edit-form">
+          <label htmlFor={`edit-name-${monitor.id}`}>Edit name</label>
+          <input id={`edit-name-${monitor.id}`} name="name" required defaultValue={monitor.name} disabled={pending || loggingOut} />
+          <label htmlFor={`edit-url-${monitor.id}`}>Edit endpoint URL</label>
+          <input id={`edit-url-${monitor.id}`} name="url" type="url" required defaultValue={monitor.url} disabled={pending || loggingOut} />
+          <label htmlFor={`edit-interval-${monitor.id}`}>Edit interval (seconds)</label>
+          <input id={`edit-interval-${monitor.id}`} name="interval" type="number" min="1" max="86400" step="1" required defaultValue={monitor.interval} disabled={pending || loggingOut} />
+          <button type="submit" disabled={pending || loggingOut}>{pending && mutation.kind === 'update' ? 'Saving…' : 'Save changes'}</button>
+        </form>}
+        {!monitor.enabled && <p className="hint">Paused. Manual checks remain available; no scheduler runs in this version.</p>}
+        {monitor.latestCheck ? <dl aria-label="Latest check result">
+          <dt>State</dt><dd>{monitor.latestCheck.state}</dd>
+          <dt>HTTP status</dt><dd>{monitor.latestCheck.httpStatus ?? 'No response'}</dd>
+          <dt>Latency</dt><dd>{monitor.latestCheck.latencyMs} ms</dd>
+          <dt>Failure reason</dt><dd>{monitor.latestCheck.failureReason ?? 'None'}</dd>
+          <dt>Finished</dt><dd><time dateTime={monitor.latestCheck.finishedAt}>{monitor.latestCheck.finishedAt}</time></dd>
+        </dl> : <p>No checks yet.</p>}
+        {historyId === monitor.id && <section aria-label={`Check history for ${monitor.name}`}>
+          <h4>Check history</h4>
+          <label htmlFor={`history-state-${monitor.id}`}>History state</label>
+          <select id={`history-state-${monitor.id}`} size={3} value={searchParams.get('state') ?? ''}
+            onChange={(event) => navigateHistory(monitor.id, { state: event.currentTarget.value })} disabled={loggingOut}>
+            <option value="">All states</option><option value="SUCCEEDED">SUCCEEDED</option><option value="FAILED">FAILED</option>
+          </select>
+          <div className="actions">
+            <button onClick={() => navigateHistory(monitor.id, { cursor: null })} disabled={loggingOut || !searchParams.has('cursor')}>First page</button>
+            <button onClick={() => navigateHistory(monitor.id, { cursor: historyQuery?.data?.nextCursor })}
+              disabled={loggingOut || historyQuery?.status !== 'success' || !historyQuery.data?.nextCursor}>Next page</button>
+          </div>
+          {!historyQuery || historyQuery.status === 'pending' ? <p role="status">Loading history…</p> : historyQuery.data === null ? <p>History could not be loaded.</p> : history.length === 0 ? <p>No historical checks.</p> :
+            <div className="history-scroll" role="region" aria-label={`Scrollable check results for ${monitor.name}`} tabIndex={0}><table>
+              <caption>Terminal check results for {monitor.name}</caption>
+              <thead><tr><th scope="col">Check ID</th><th scope="col">State</th><th scope="col">HTTP status</th><th scope="col">Latency</th><th scope="col">Failure reason</th><th scope="col">Finished</th></tr></thead>
+              <tbody>{history.map((check) => <tr key={check.id}>
+                <td>{check.id}</td><td>{check.state}</td><td>{check.httpStatus ?? 'No response'}</td>
+                <td>{check.latencyMs} ms</td><td>{check.failureReason ?? 'None'}</td>
+                <td><time dateTime={check.finishedAt}>{check.finishedAt}</time></td>
+              </tr>)}</tbody>
+            </table></div>}
+        </section>}
+      </article>; })}
+    </section>
+  </>;
+}
diff --git a/app/monitors/page.tsx b/app/monitors/page.tsx
index 932f67a..d5ae3a3 100644
--- a/app/monitors/page.tsx
+++ b/app/monitors/page.tsx
@@ -1,178 +1,28 @@
-'use client';
-
-import { Suspense, useEffect, useState } from 'react';
-import type { FormEvent } from 'react';
-import { useSearchParams } from 'next/navigation';
-import type { MonitorView } from '../../server/model';
-import { ERROR_MESSAGES } from './api';
-import { useMonitors } from './use-monitors';
-
-export default function MonitorsPage() {
-  return <Suspense fallback={<main><h1>Monitors</h1><p>Loading monitors…</p></main>}><Monitors /></Suspense>;
-}
-
-function Monitors() {
-  const { authenticated, loading, monitors, histories, mutations, error, mutate, loadHistory } = useMonitors();
-  const creating = mutations.create?.status === 'pending';
-  const loggingOut = mutations.logout?.status === 'pending';
-  const anyPending = Object.values(mutations).some((mutation) => mutation.status === 'pending');
-  const [editing, setEditing] = useState<string | null>(null);
-  const searchParams = useSearchParams();
-  const historyId = searchParams.get('monitor');
-  const historyParameters = new URLSearchParams(searchParams.toString());
-  historyParameters.delete('monitor');
-  const historySearch = historyParameters.toString();
-
-  useEffect(() => {
-    if (authenticated && historyId) void loadHistory(historyId, historySearch);
-  }, [authenticated, historyId, historySearch, loadHistory]);
-
-  function navigateHistory(id: string | null, change: { state?: string; cursor?: string | null } = {}) {
-    const next = new URLSearchParams(window.location.search);
-    if (id === null) {
-      for (const key of ['monitor', 'state', 'limit', 'cursor']) next.delete(key);
-    } else {
-      if (id !== next.get('monitor')) next.delete('cursor');
-      next.set('monitor', id);
-      if (change.state !== undefined) {
-        if (change.state) next.set('state', change.state); else next.delete('state');
-        next.delete('cursor');
-      }
-      if (change.cursor !== undefined) {
-        if (change.cursor) next.set('cursor', change.cursor); else next.delete('cursor');
-      }
-    }
-    // Next integrates native history changes with useSearchParams, including
-    // back/forward, while preserving in-flight mutations and local form drafts.
-    window.history.pushState(null, '', `/monitors${next.size ? `?${next}` : ''}`);
-  }
-
-  async function createMonitor(event: FormEvent<HTMLFormElement>) {
-    event.preventDefault();
-    const form = event.currentTarget;
-    const fields = new FormData(form);
-    if (await mutate({ kind: 'create', input: {
-      name: String(fields.get('name')), url: String(fields.get('url')),
-      interval: Number(fields.get('interval')), enabled: fields.get('enabled') === 'on',
-    } })) form.reset();
-  }
-
-  async function editMonitor(event: FormEvent<HTMLFormElement>, monitor: MonitorView) {
-    event.preventDefault();
-    const fields = new FormData(event.currentTarget);
-    if (await mutate({ kind: 'update', id: monitor.id, input: {
-      name: String(fields.get('name')), url: String(fields.get('url')),
-      interval: Number(fields.get('interval')), enabled: monitor.enabled,
-    } })) setEditing((current) => current === monitor.id ? null : current);
+import { redirect } from 'next/navigation';
+import { failureCode } from './api';
+import { emptyMonitors } from './initial-state';
+import { readInitialMonitors } from './server-data';
+import MonitorWorkspace from './monitor-workspace';
+
+export default async function MonitorsPage({ searchParams }: {
+  searchParams: Promise<Record<string, string | string[] | undefined>>;
+}) {
+  const parameters = new URLSearchParams();
+  for (const [key, value] of Object.entries(await searchParams)) {
+    for (const item of Array.isArray(value) ? value : value === undefined ? [] : [value]) parameters.append(key, item);
   }
-
-  async function deleteMonitor(id: string) {
-    if (!window.confirm('Delete this monitor and all its check history?')) return;
-    if (await mutate({ kind: 'delete', id })) {
-      if (new URLSearchParams(window.location.search).get('monitor') === id) navigateHistory(null);
-      setEditing((current) => current === id ? null : current);
-    }
+  let initial;
+  try { initial = await readInitialMonitors(parameters); }
+  catch (failure) {
+    const error = failureCode(failure);
+    if (error === 'UNAUTHENTICATED') redirect('/login');
+    initial = { ...emptyMonitors(), error };
   }
-
-  if (!authenticated) return <main>
-    <h1>Monitors</h1>
-    {loading ? <p>Loading monitors…</p> : error && error !== 'UNAUTHENTICATED'
-      ? <p role="alert" className="error" data-error-code={error}>{ERROR_MESSAGES[error]}</p>
-      : <p>Opening sign in…</p>}
-  </main>;
-
   return <main>
     <header><p className="eyebrow">HTTP ENDPOINT MONITOR · LOCAL FIXTURE</p><h1>Monitors</h1>
       <p>Create an endpoint monitor, run a check, and inspect the response.</p>
-      <button onClick={() => mutate({ kind: 'logout' })} disabled={anyPending}>{loggingOut ? 'Signing out…' : 'Sign out'}</button>
     </header>
     <aside>Development only. Checks can access the configured fixture origin. Monitors and all check history are stored in PostgreSQL.</aside>
-    {error && <p role="alert" className="error" data-error-code={error}>{ERROR_MESSAGES[error]}</p>}
-    <section aria-labelledby="create-heading" className="panel">
-      <h2 id="create-heading">Create monitor</h2>
-      <form onSubmit={createMonitor}>
-        <label htmlFor="name">Name</label>
-        <input id="name" name="name" required placeholder="Fixture monitor" disabled={creating || loggingOut} />
-        <label htmlFor="url">Endpoint URL</label>
-        <input id="url" name="url" type="url" required defaultValue="http://127.0.0.1:4311/ok" disabled={creating || loggingOut} />
-        <label htmlFor="interval">Interval (seconds)</label>
-        <input id="interval" name="interval" type="number" required min="1" max="86400" step="1" defaultValue="60" disabled={creating || loggingOut} />
-        <label className="checkbox"><input name="enabled" type="checkbox" defaultChecked disabled={creating || loggingOut} /> Enabled</label>
-        <p className="hint">Interval and enabled are saved; this version only runs manual checks.</p>
-        <button type="submit" disabled={creating || loggingOut}>{creating ? 'Creating…' : 'Create monitor'}</button>
-      </form>
-      {mutations.create?.status === 'success' && <p role="status">Monitor created.</p>}
-    </section>
-    <section aria-labelledby="saved-heading">
-      <h2 id="saved-heading">Your monitors</h2>
-      {monitors.length === 0 && <p>No monitors yet.</p>}
-      {monitors.map((monitor) => {
-        const mutation = mutations[`monitor:${monitor.id}`];
-        const pending = mutation?.status === 'pending';
-        const historyQuery = histories[monitor.id]?.search === historySearch ? histories[monitor.id] : undefined;
-        const history = historyQuery?.data?.items ?? [];
-        return <article key={monitor.id} className="panel" aria-labelledby={`monitor-${monitor.id}`}>
-        <h3 id={`monitor-${monitor.id}`}>{monitor.name}</h3>
-        <p className="endpoint">{monitor.url}</p>
-        <p>{monitor.enabled ? 'Enabled' : 'Paused'} · {monitor.interval} seconds</p>
-        <div className="actions">
-        <button onClick={() => mutate({ kind: 'check', id: monitor.id })} disabled={pending || loggingOut}>
-          {pending && mutation.kind === 'check' ? 'Checking…' : 'Run check'}
-        </button>
-        <button onClick={() => setEditing(editing === monitor.id ? null : monitor.id)} disabled={pending || loggingOut}>
-          {editing === monitor.id ? 'Cancel edit' : 'Edit monitor'}
-        </button>
-        <button onClick={() => mutate({ kind: 'update', id: monitor.id, input: { ...monitor, enabled: !monitor.enabled } })} disabled={pending || loggingOut}>
-          {monitor.enabled ? 'Pause monitor' : 'Enable monitor'}
-        </button>
-        <button onClick={() => deleteMonitor(monitor.id)} disabled={pending || loggingOut}>
-          {pending && mutation.kind === 'delete' ? 'Deleting…' : 'Delete monitor'}
-        </button>
-        <button onClick={() => navigateHistory(historyId === monitor.id ? null : monitor.id)} disabled={loggingOut}>
-          {historyId === monitor.id ? 'Hide history' : 'View history'}
-        </button>
-        </div>
-        {mutation?.status === 'success' && <p role="status">{mutation.kind === 'check' ? 'Check completed.' : 'Monitor saved.'}</p>}
-        {editing === monitor.id && <form onSubmit={(event) => editMonitor(event, monitor)} className="edit-form">
-          <label htmlFor={`edit-name-${monitor.id}`}>Edit name</label>
-          <input id={`edit-name-${monitor.id}`} name="name" required defaultValue={monitor.name} disabled={pending || loggingOut} />
-          <label htmlFor={`edit-url-${monitor.id}`}>Edit endpoint URL</label>
-          <input id={`edit-url-${monitor.id}`} name="url" type="url" required defaultValue={monitor.url} disabled={pending || loggingOut} />
-          <label htmlFor={`edit-interval-${monitor.id}`}>Edit interval (seconds)</label>
-          <input id={`edit-interval-${monitor.id}`} name="interval" type="number" min="1" max="86400" step="1" required defaultValue={monitor.interval} disabled={pending || loggingOut} />
-          <button type="submit" disabled={pending || loggingOut}>{pending && mutation.kind === 'update' ? 'Saving…' : 'Save changes'}</button>
-        </form>}
-        {!monitor.enabled && <p className="hint">Paused. Manual checks remain available; no scheduler runs in this version.</p>}
-        {monitor.latestCheck ? <dl aria-label="Latest check result">
-          <dt>State</dt><dd>{monitor.latestCheck.state}</dd>
-          <dt>HTTP status</dt><dd>{monitor.latestCheck.httpStatus ?? 'No response'}</dd>
-          <dt>Latency</dt><dd>{monitor.latestCheck.latencyMs} ms</dd>
-          <dt>Failure reason</dt><dd>{monitor.latestCheck.failureReason ?? 'None'}</dd>
-          <dt>Finished</dt><dd><time dateTime={monitor.latestCheck.finishedAt}>{monitor.latestCheck.finishedAt}</time></dd>
-        </dl> : <p>No checks yet.</p>}
-        {historyId === monitor.id && <section aria-label={`Check history for ${monitor.name}`}>
-          <h4>Check history</h4>
-          <label htmlFor={`history-state-${monitor.id}`}>History state</label>
-          <select id={`history-state-${monitor.id}`} value={searchParams.get('state') ?? ''}
-            onChange={(event) => navigateHistory(monitor.id, { state: event.currentTarget.value })} disabled={loggingOut}>
-            <option value="">All states</option><option value="SUCCEEDED">SUCCEEDED</option><option value="FAILED">FAILED</option>
-          </select>
-          <div className="actions">
-            <button onClick={() => navigateHistory(monitor.id, { cursor: null })} disabled={loggingOut || !searchParams.has('cursor')}>First page</button>
-            <button onClick={() => navigateHistory(monitor.id, { cursor: historyQuery?.data?.nextCursor })}
-              disabled={loggingOut || historyQuery?.status !== 'success' || !historyQuery.data?.nextCursor}>Next page</button>
-          </div>
-          {!historyQuery || historyQuery.status === 'pending' ? <p>Loading history…</p> : historyQuery.data === null ? <p>History could not be loaded.</p> : history.length === 0 ? <p>No historical checks.</p> :
-            <div className="history-scroll"><table>
-              <thead><tr><th>Check ID</th><th>State</th><th>HTTP status</th><th>Latency</th><th>Failure reason</th><th>Finished</th></tr></thead>
-              <tbody>{history.map((check) => <tr key={check.id}>
-                <td>{check.id}</td><td>{check.state}</td><td>{check.httpStatus ?? 'No response'}</td>
-                <td>{check.latencyMs} ms</td><td>{check.failureReason ?? 'None'}</td>
-                <td><time dateTime={check.finishedAt}>{check.finishedAt}</time></td>
-              </tr>)}</tbody>
-            </table></div>}
-        </section>}
-      </article>; })}
-    </section>
+    <MonitorWorkspace initial={initial} />
   </main>;
 }
diff --git a/app/monitors/server-data.ts b/app/monitors/server-data.ts
new file mode 100644
index 0000000..def987b
--- /dev/null
+++ b/app/monitors/server-data.ts
@@ -0,0 +1,42 @@
+import 'server-only';
+
+import { headers } from 'next/headers';
+import type { CheckHistoryPage, MonitorView } from '../../server/model';
+import { failureCode, responseData } from './api';
+import { historyLocation } from './initial-state';
+import type { InitialMonitors } from './initial-state';
+
+export async function readInitialMonitors(parameters: URLSearchParams): Promise<InitialMonitors> {
+  const incoming = await headers();
+  const forwarded = new Headers();
+  // Preserve raw Cookie ambiguity and explicit Origin. Fastify continues to
+  // enforce session validity, duplicate-cookie rejection and read authorization.
+  for (const name of ['cookie', 'origin']) {
+    const value = incoming.get(name);
+    if (value !== null) forwarded.set(name, value);
+  }
+  const origin = process.env.API_ORIGIN ?? 'http://127.0.0.1:4312';
+  async function read<T>(path: string): Promise<T> {
+    return responseData<T>(await fetch(new URL(path, origin), {
+      headers: forwarded, cache: 'no-store', redirect: 'error',
+    }));
+  }
+  // Request-time headers and explicit no-store reads exclude shared user-data
+  // caching. Only this request's public payload reaches the client island.
+  const monitors = await read<MonitorView[]>('/monitors');
+  const initial: InitialMonitors = { authenticated: true, monitors, histories: {}, error: null };
+  const { id, search } = historyLocation(parameters);
+  if (id) {
+    try {
+      const data = await read<CheckHistoryPage>(`/monitors/${encodeURIComponent(id)}/checks${search ? `?${search}` : ''}`);
+      initial.histories[id] = { search, status: 'success', data, error: null };
+    } catch (failure) {
+      const error = failureCode(failure);
+      // Session loss between reads discards the earlier collection too.
+      if (error === 'UNAUTHENTICATED') throw failure;
+      initial.error = error;
+      initial.histories[id] = { search, status: 'error', data: null, error };
+    }
+  }
+  return initial;
+}
diff --git a/app/monitors/use-monitors.ts b/app/monitors/use-monitors.ts
index 4022ad4..49469cc 100644
--- a/app/monitors/use-monitors.ts
+++ b/app/monitors/use-monitors.ts
@@ -1,9 +1,10 @@
 'use client';
 
 import { useCallback, useEffect, useRef, useState } from 'react';
-import { useRouter } from 'next/navigation';
 import type { ApiErrorCode, CheckHistoryPage, CheckRun, MonitorView } from '../../server/model.ts';
 import { failureCode, mutationFetch, responseData } from './api.ts';
+import { emptyMonitors } from './initial-state.ts';
+import type { InitialMonitors } from './initial-state.ts';
 
 type MonitorInput = Pick<MonitorView, 'name' | 'url' | 'interval' | 'enabled'>;
 type Mutation =
@@ -12,36 +13,31 @@ type Mutation =
   | { kind: 'delete' | 'check'; id: string }
   | { kind: 'logout' };
 type MutationState = { kind: Mutation['kind']; status: 'pending' | 'success' | 'error'; error: ApiErrorCode | null };
-type HistoryQuery = { search: string; status: 'pending' | 'success' | 'error'; data: CheckHistoryPage | null; error: ApiErrorCode | null };
-type ServerState = {
-  authenticated: boolean;
-  loading: boolean;
-  monitors: MonitorView[];
-  histories: Record<string, HistoryQuery>;
+type ServerState = InitialMonitors & {
   mutations: Record<string, MutationState>;
-  error: ApiErrorCode | null;
 };
 
-const emptyState = (): ServerState => ({ authenticated: false, loading: true, monitors: [], histories: {}, mutations: {}, error: null });
+const emptyState = (): ServerState => ({ ...emptyMonitors(), mutations: {} });
 
 // One owner for this route's server state. Forms stay in the component; history
 // selection/conditions come from the URL. Keep only one page per Monitor.
-export function useMonitors() {
-  const router = useRouter();
-  const [state, setState] = useState<ServerState>(emptyState);
+export function useMonitors(initial: InitialMonitors) {
+  const [state, setState] = useState<ServerState>(() => ({ ...initial, mutations: {} }));
   const lifetime = useRef(0);
   const pending = useRef(new Set<string>());
   const historyVersions = useRef(new Map<string, number>());
-  const historySearches = useRef(new Map<string, string>());
+  const historySearches = useRef(new Map(Object.entries(initial.histories).map(([id, query]) => [id, query.search])));
 
   const clearSession = useCallback(() => {
     lifetime.current++;
     pending.current.clear();
     historyVersions.current.clear();
     historySearches.current.clear();
-    setState({ ...emptyState(), loading: false });
-    router.replace('/login');
-  }, [router]);
+    setState(emptyState());
+    // Browser callbacks replace the document at the authentication boundary,
+    // discarding client route caches together with the cleared state owner.
+    window.location.replace('/login');
+  }, []);
 
   const handleFailure = useCallback((failure: unknown) => {
     const code = failureCode(failure);
@@ -50,17 +46,9 @@ export function useMonitors() {
     return code;
   }, [clearSession]);
 
-  useEffect(() => {
-    const generation = ++lifetime.current;
-    fetch('/api/monitors', { credentials: 'same-origin' }).then(responseData<MonitorView[]>).then((monitors) => {
-      if (generation === lifetime.current) setState((current) => ({ ...current, monitors, authenticated: true }));
-    }).catch((failure: unknown) => {
-      if (generation === lifetime.current) handleFailure(failure);
-    }).finally(() => {
-      if (generation === lifetime.current) setState((current) => ({ ...current, loading: false }));
-    });
-    return () => { lifetime.current++; };
-  }, [handleFailure]);
+  // The authoritative server payload is also the first client state. Mount
+  // does not replace it with loading or issue duplicate collection/history GETs.
+  useEffect(() => () => { lifetime.current++; }, []);
 
   const loadHistory = useCallback(async (id: string, search = '', clearError = true) => {
     const generation = lifetime.current;
@@ -86,6 +74,10 @@ export function useMonitors() {
     }
   }, [handleFailure]);
 
+  const ensureHistory = useCallback(async (id: string, search = '') => {
+    if (historySearches.current.get(id) !== search) await loadHistory(id, search);
+  }, [loadHistory]);
+
   async function mutate(action: Mutation): Promise<boolean> {
     const key = 'id' in action ? `monitor:${action.id}` : action.kind;
     // Admission is synchronous, before CSRF or POST. Rendered disabled buttons
@@ -148,5 +140,5 @@ export function useMonitors() {
     }
   }
 
-  return { ...state, mutate, loadHistory };
+  return { ...state, mutate, ensureHistory };
 }


