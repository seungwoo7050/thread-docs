## `영속 Monitor 수명주기와 전체 Check 이력을 웹에서 제공`

diff --git a/TRACK.md b/TRACK.md
index 65fe0ad..1279dd7 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -2,7 +2,7 @@
 
 Spec revision: `0a006589477f8ae47bad3faa5510c999cff85ee4`
 
-E02 is a local development product: Next.js/React renders the Monitor form and terminal Check result; Spring MVC owns an in-memory Monitor map and latest Check result. A manual check is synchronous. There are no accounts, database, scheduler, workers, cache, broker, or production containers.
+E03 is a local development product: Next.js/React renders Monitor creation, editing, pause/activation, deletion and Check history. Spring MVC uses PostgreSQL as the authoritative store for Monitors and every completed CheckRun. A manual check is synchronous. There are no accounts, scheduler, workers, cache, broker, or production application containers.
 
 ## Pinned toolchain
 
@@ -19,8 +19,12 @@ E02 is a local development product: Next.js/React renders the Monitor form and t
 | Playwright | 1.62.1 | `package.json`, lock; Chromium revision 1234 / 151.0.7922.34 |
 | Node / React / React DOM types | 24.10.1 / 19.2.18 / 19.2.5 | `package.json`, lock |
 | Maven Enforcer | 3.6.2 | `backend/pom.xml` |
+| PostgreSQL | 17.11-bookworm | `compose.yaml`, immutable image digest `sha256:051f7b7b3abdd564d5d1bd1e8c4b9c1b6e77087d1dd22020ede611c096a272e0` |
+| Hibernate ORM | 6.6.53.Final | existing Spring Boot 3.5.16 BOM |
+| Flyway core / PostgreSQL module | 11.7.2 | existing Spring Boot 3.5.16 BOM |
+| PostgreSQL JDBC | 42.7.11 | existing Spring Boot 3.5.16 BOM |
 
-Versions remain fixed for this evolution. The Spring parent pins build plugins and transitive dependency management. There is no container build yet; CI and local runtime files are the current execution contract.
+Versions remain fixed for this evolution. The Spring parent pins build plugins and transitive dependency management. There is no application container build yet; the only container is the isolated development/test database.
 
 ## Run locally
 
@@ -29,6 +33,7 @@ Run all commands from the repository root using the pinned runtimes. Maven artif
 ```sh
 fnm use 24.19.0
 npm ci
+npm run db:up
 ```
 
 Start each process in a separate terminal:
@@ -41,7 +46,11 @@ npm run dev
 
 Open [Monitors](http://127.0.0.1:4323/monitors). Create `Fixture monitor` with URL `http://127.0.0.1:4321/ok`, interval `60`, enabled checked. Click **Run check** to observe `SUCCEEDED` and `HTTP 200`. `/fail` yields `FAILED` and `HTTP 503`.
 
-All three defaults bind to `127.0.0.1`. Fixture port is 4321, API port 4322, UI port 4323. `FIXTURE_PORT`, `FIXTURE_ORIGIN`, and `API_PORT` can configure the server processes; `API_ORIGIN` configures the Next API proxy. The committed E01 tests use the fixed default ports; stop local servers before verification. Tests also reserve 4324 for a forbidden destination trap.
+All defaults bind to `127.0.0.1`. Fixture port is 4321, API port 4322, UI port 4323, PostgreSQL port 15432. `FIXTURE_PORT`, `FIXTURE_ORIGIN`, and `API_PORT` can configure the server processes; `API_ORIGIN` configures the Next API proxy. The committed E01 tests use the fixed default ports; stop local servers before verification. Tests also reserve 4324 for a forbidden destination trap and 4325 for timeout/connection fixtures.
+
+The compose project is exclusively `wse-industry`, with its own bridge network and persistent volume. It uses explicitly nonsecret local test trust authentication and a loopback-only published port. An internal-only Docker network cannot provide the required published port on the verified Docker Desktop runtime. Never use this compose configuration for production or put unrelated data in it. `npm run db:down` stops the project but preserves data; `npm run db:up` restores it. `npm run db:destroy` explicitly deletes this project's disposable database volume.
+
+The default connection is database `monitor`, local test identity `wse_industry`, schema `public`. `DB_URL` (JDBC URL), `DB_USER`, `DB_PASSWORD`, and `DB_SCHEMA` support external runtime configuration. Never commit or log real credentials or put a password in a JDBC URL. Verification resets only explicitly named `e03_*` schemas, not the developer's `public` schema.
 
 ## Check boundary
 
@@ -49,18 +58,29 @@ All three defaults bind to `127.0.0.1`. Fixture port is 4321, API port 4322, UI
 - Checks send one GET with no request body, use no proxy, and never follow redirects. `/redirect` therefore records `FAILED / 302`, with no request to `/ok`.
 - Connect timeout is 1 second and response-header read timeout is 2 seconds. No response body is materialized or retained. This is a controlled fixture implementation, not general Internet monitoring or general SSRF defense.
 - `200..299` is `SUCCEEDED`. Other observed HTTP statuses are `FAILED / HTTP_STATUS`. No HTTP response produces a null status and `TIMEOUT` or `CONNECTION_FAILURE`; no synthetic status is invented.
-- Latest results survive page reload, but all state disappears with the API process. Interval and enabled are stored, without automatic scheduling.
+- Monitors and all completed results survive API restarts. Interval and enabled are stored, without automatic scheduling; a paused Monitor can still be checked manually.
 
 ## HTTP contract (E02)
 
 - Create input must be a JSON object with string name and URL, a numeric integer interval, and boolean enabled. Required fields cannot be null or omitted; scalar strings/numbers/booleans are not coerced into other types.
 - Names are stripped of surrounding whitespace and must contain 1–100 UTF-16 code units. Interval is 1–86400 seconds inclusive; the JSON number `60.0` is the integer value 60, but the string `"60"` is invalid.
+- E03 also rejects a NUL character in a name before persistence, because PostgreSQL text cannot store it. Create and replacement use the same runtime validator, including the existing non-finite numeric rejection.
 - URL syntax must be absolute HTTP(S). The existing fixture policy then restricts it to the configured HTTP hostname and port, without credentials or fragments. This does not enable general public or HTTPS monitoring.
 - Successful list/create/check responses contain exactly `{ "data": <payload> }`. Create returns 201; list and completed synchronous checks return 200. The existing MonitorView/CheckRun payload fields are preserved, including explicit nulls.
 - API failures contain `{ "error": { "code": "...", "message": "..." } }`: INVALID_INPUT / 400, NOT_FOUND / 404, or INTERNAL_ERROR / 500. Malformed JSON and IDs use INVALID_INPUT. Unexpected exception details are not returned.
 - Endpoint HTTP 503 is still an API success containing a FAILED CheckRun with HTTP_STATUS; timeout and connection failure retain null httpStatus. API errors never fabricate a CheckRun.
 - The browser validates the envelope and the displayed payload shape, selects errors by the stable code/status pair, and does not classify or display arbitrary server prose. Network or malformed responses use the INTERNAL_ERROR UI fallback without applying the mutation.
 
+## PostgreSQL boundary (E03)
+
+- Flyway is the sole schema writer: V1 creates `monitors`, V2 creates `check_runs`. Repeated startup validates migration checksums and applies only pending migrations. Hibernate uses `ddl-auto=validate`, never create/update.
+- A startup metadata check supplements Hibernate validation by rejecting unmapped required columns without defaults. Missing mapped columns or incompatible insert requirements prevent the web server from becoming ready.
+- `MonitorEntity.fromDomain/toDomain` and `CheckRunEntity.fromDomain/toDomain` explicitly map canonical immutable records. Entities never reach JSON. UUIDs remain UUIDs; interval is PostgreSQL integer, enabled boolean, latency bigint, and nullable HTTP status/reason retain null rather than zero/empty text.
+- Timestamp values are truncated to microseconds before the first response and stored as `timestamp(6) with time zone`. Canonical Java Instant/JSON values remain UTC even in a non-UTC PostgreSQL session.
+- Controller calls cross the concrete `MonitorStore` Spring transaction proxy. Public mutations use write transactions and reads use read-only transactions. Private helpers remain inside the caller's transaction; no self-invocation starts an assumed new transaction. The outbound GET runs between store calls, outside a transaction. No DB locks or worker structure are added.
+- `GET /api/monitors/{id}` returns a MonitorView. `PUT /api/monitors/{id}` replaces the same four create fields, including enabled for pause/activation. `DELETE /api/monitors/{id}` returns `{ "data": null }`; PostgreSQL `ON DELETE CASCADE` removes all historical runs.
+- `GET /api/monitors/{id}/checks` returns the full history ordered by finishedAt descending, then ID descending. `GET /api/monitors/{id}/checks/{checkId}` returns a single historical result belonging to that Monitor. Deleted Monitor/history/run resources return 404/NOT_FOUND. Pagination is not introduced here.
+
 ## Verification
 
 ```sh
@@ -68,7 +88,9 @@ npx playwright install chromium
 npm run verify
 ```
 
-`verify` runs Maven unit and real-HTTP functional tests and packages the API, then TypeScript checking, a Next production compilation, and real Chromium browser tests against the local development UI and packaged API. It does not retry failed tests. Command outcomes and elapsed times are appended to `output/verification/runs.jsonl`. Committed evidence is in `evidence/E01` and `evidence/E02`.
+`verify` starts the isolated PostgreSQL project, runs Maven unit, real-HTTP functional and real-PostgreSQL integration tests, and packages the API. It then checks TypeScript, compiles Next for production, runs the fixed A,A,B process-restart/lifecycle scenario, and runs Chromium against the local development UI and packaged API. It does not retry failed tests. Command outcomes and elapsed times are appended to `output/verification/runs.jsonl`; the process-restart probe saves wire evidence in `output/e03`. Committed evidence is in `evidence/E01`, `evidence/E02`, and `evidence/E03`.
+
+Tests create and remove isolated schemas for functional, browser, restart, mapping, migration, and incompatible-schema fixtures. The standard runner cleans up its browser/restart schemas even after a failure. The database remains available afterward; use `npm run db:down` to stop only this project. The Java tests explicitly close independent application contexts to verify restart persistence, capture actual generated SQL/transaction flags, check rollback and cascade, and assert startup rejection for both incompatible-schema cases.
 
 The CI workflow installs the exact toolchain and runs the same gates. No hosted CI run is claimed by local verification. The browser gate starts and stops its own processes and refuses existing servers. There are no load tests, benchmarks, or parameter sweeps.
 
@@ -89,3 +111,6 @@ npm run test:e2e
 - [Next.js security releases](https://nextjs.org/blog)
 - [Playwright 1.62.1 browser manifest](https://github.com/microsoft/playwright/blob/v1.62.1/packages/playwright-core/browsers.json)
 - [Maven 3.9.11 distribution checksum](https://repo.maven.apache.org/maven2/org/apache/maven/apache-maven/3.9.11/apache-maven-3.9.11-bin.tar.gz.sha512)
+- [Spring Boot 3.5 database initialization](https://docs.spring.io/spring-boot/3.5/how-to/data-initialization.html)
+- [Spring transaction proxy boundaries](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html)
+- [PostgreSQL 17 timestamp precision](https://www.postgresql.org/docs/17/datatype-datetime.html)
diff --git a/app/monitors/api.ts b/app/monitors/api.ts
index fa41ef4..b01c3c9 100644
--- a/app/monitors/api.ts
+++ b/app/monitors/api.ts
@@ -1,5 +1,5 @@
 export type Monitor = { id: string; name: string; url: string; interval: number; enabled: boolean };
-type CheckRun = {
+export type CheckRun = {
   id: string; monitorId: string; state: 'SUCCEEDED' | 'FAILED'; httpStatus: number | null;
   latencyMs: number; failureReason: 'HTTP_STATUS' | 'CONNECTION_FAILURE' | 'TIMEOUT' | null; finishedAt: string;
 };
@@ -85,3 +85,18 @@ export async function createMonitor(input: Omit<Monitor, 'id'>): Promise<Monitor
 export async function runCheck(id: string): Promise<CheckRun> {
   return readData(await fetch(`/api/monitors/${id}/checks`, { method: 'POST' }), isCheckRun);
 }
+
+export async function replaceMonitor(id: string, input: Omit<Monitor, 'id'>): Promise<MonitorView> {
+  return readData(await fetch(`/api/monitors/${id}`, {
+    method: 'PUT', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(input),
+  }), isMonitorView);
+}
+
+export async function deleteMonitor(id: string): Promise<null> {
+  return readData(await fetch(`/api/monitors/${id}`, { method: 'DELETE' }), (value): value is null => value === null);
+}
+
+export async function loadChecks(id: string): Promise<CheckRun[]> {
+  return readData(await fetch(`/api/monitors/${id}/checks`), (value): value is CheckRun[] =>
+    Array.isArray(value) && value.every(isCheckRun));
+}
diff --git a/app/monitors/page.tsx b/app/monitors/page.tsx
index 5f22880..445c9df 100644
--- a/app/monitors/page.tsx
+++ b/app/monitors/page.tsx
@@ -1,13 +1,21 @@
 'use client';
 
 import { useEffect, useState, type FormEvent } from 'react';
-import { createMonitor, errorMessages, failureCode, loadMonitors, runCheck,
-  type ApiErrorCode, type MonitorView } from './api';
+import { createMonitor, deleteMonitor, errorMessages, failureCode, loadChecks, loadMonitors,
+  replaceMonitor, runCheck, type ApiErrorCode, type CheckRun, type Monitor, type MonitorView } from './api';
+
+function inputFrom(form: HTMLFormElement): Omit<Monitor, 'id'> {
+  const fields = new FormData(form);
+  return { name: String(fields.get('name') ?? ''), url: String(fields.get('url') ?? ''),
+    interval: Number(fields.get('interval')), enabled: fields.get('enabled') === 'on' };
+}
 
 export default function Monitors() {
   const [monitors, setMonitors] = useState<MonitorView[]>([]);
   const [error, setError] = useState<ApiErrorCode | null>(null);
   const [busy, setBusy] = useState(false);
+  const [editingId, setEditingId] = useState<string | null>(null);
+  const [histories, setHistories] = useState<Record<string, CheckRun[]>>({});
 
   useEffect(() => {
     loadMonitors().then(setMonitors).catch(error => setError(failureCode(error)));
@@ -16,14 +24,10 @@ export default function Monitors() {
   async function create(event: FormEvent<HTMLFormElement>) {
     event.preventDefault();
     const form = event.currentTarget;
-    const fields = new FormData(form);
     setBusy(true);
     setError(null);
     try {
-      const created = await createMonitor({
-        name: String(fields.get('name') ?? ''), url: String(fields.get('url') ?? ''),
-        interval: Number(fields.get('interval')), enabled: fields.get('enabled') === 'on',
-      });
+      const created = await createMonitor(inputFrom(form));
       setMonitors(current => [...current, created]);
       form.reset();
     } catch (error) {
@@ -39,6 +43,53 @@ export default function Monitors() {
     try {
       const latestCheck = await runCheck(id);
       setMonitors(current => current.map(row => row.monitor.id === id ? { ...row, latestCheck } : row));
+      setHistories(current => current[id] ? { ...current, [id]: [latestCheck, ...current[id]] } : current);
+    } catch (error) {
+      setError(failureCode(error));
+    } finally {
+      setBusy(false);
+    }
+  }
+
+  async function update(id: string, input: Omit<Monitor, 'id'>) {
+    setBusy(true);
+    setError(null);
+    try {
+      const updated = await replaceMonitor(id, input);
+      setMonitors(current => current.map(row => row.monitor.id === id ? updated : row));
+      setEditingId(null);
+    } catch (error) {
+      setError(failureCode(error));
+    } finally {
+      setBusy(false);
+    }
+  }
+
+  async function remove(id: string) {
+    setBusy(true);
+    setError(null);
+    try {
+      await deleteMonitor(id);
+      setMonitors(current => current.filter(row => row.monitor.id !== id));
+      setHistories(current => { const remaining = { ...current }; delete remaining[id]; return remaining; });
+      if (editingId === id) setEditingId(null);
+    } catch (error) {
+      setError(failureCode(error));
+    } finally {
+      setBusy(false);
+    }
+  }
+
+  async function history(id: string) {
+    if (histories[id]) {
+      setHistories(current => { const remaining = { ...current }; delete remaining[id]; return remaining; });
+      return;
+    }
+    setBusy(true);
+    setError(null);
+    try {
+      const checks = await loadChecks(id);
+      setHistories(current => ({ ...current, [id]: checks }));
     } catch (error) {
       setError(failureCode(error));
     } finally {
@@ -49,7 +100,7 @@ export default function Monitors() {
   return <main>
     <header><p className="eyebrow">LOCAL LAB · INDUSTRY / SPRING</p><h1>Endpoint Monitor</h1>
       <p>Create a monitor and check its HTTP response.</p></header>
-    <aside>Development fixture only. Data is held in memory and disappears when the API restarts.</aside>
+    <aside>Development fixture only. Monitors and completed checks are stored in PostgreSQL.</aside>
     <section aria-labelledby="create-title">
       <h2 id="create-title">Create monitor</h2>
       <form onSubmit={create}>
@@ -57,7 +108,7 @@ export default function Monitors() {
         <label>URL<input name="url" type="url" required defaultValue="http://127.0.0.1:4321/ok" /></label>
         <label>Interval (seconds)<input name="interval" type="number" min="1" max="86400" step="1" required defaultValue="60" /></label>
         <label className="checkbox"><input name="enabled" type="checkbox" defaultChecked />Enabled</label>
-        <p className="hint">Interval and enabled are stored; E01 runs checks manually.</p>
+        <p className="hint">Interval and enabled are stored. Checks still run manually; there is no scheduler.</p>
         <button disabled={busy}>Create monitor</button>
       </form>
     </section>
@@ -67,7 +118,26 @@ export default function Monitors() {
       {monitors.map(({ monitor, latestCheck }) => <article key={monitor.id} aria-label={monitor.name}>
         <h3>{monitor.name}</h3><p className="url">{monitor.url}</p>
         <p>{monitor.interval}s interval · {monitor.enabled ? 'Enabled' : 'Paused'}</p>
-        <button disabled={busy} onClick={() => check(monitor.id)}>Run check</button>
+        <div className="actions">
+          <button disabled={busy} onClick={() => check(monitor.id)}>Run check</button>
+          <button disabled={busy} onClick={() => setEditingId(monitor.id)}>Edit</button>
+          <button disabled={busy} onClick={() => update(monitor.id, { name: monitor.name, url: monitor.url,
+            interval: monitor.interval, enabled: !monitor.enabled })}>{monitor.enabled ? 'Pause' : 'Activate'}</button>
+          <button disabled={busy} onClick={() => history(monitor.id)}>{histories[monitor.id] ? 'Hide history' : 'Show history'}</button>
+          <button disabled={busy} onClick={() => remove(monitor.id)}>Delete</button>
+        </div>
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
         <div aria-live="polite" className="result">
           {latestCheck ? <>
             <strong>{latestCheck.state}</strong>
@@ -76,6 +146,17 @@ export default function Monitors() {
             {latestCheck.failureReason && <span>{latestCheck.failureReason}</span>}
           </> : <span>No checks yet.</span>}
         </div>
+        {histories[monitor.id] && <section aria-label="Historical checks">
+          <h4>Check history</h4>
+          {histories[monitor.id].length === 0 ? <p>No historical checks.</p> : <div className="history-table">
+            <table aria-label="Check history"><thead><tr><th>Finished (UTC)</th><th>Result</th><th>HTTP</th><th>Latency</th><th>Reason</th></tr></thead>
+              <tbody>{histories[monitor.id].map(check => <tr key={check.id} data-check-id={check.id}>
+                <td><time dateTime={check.finishedAt}>{check.finishedAt}</time></td><td>{check.state}</td>
+                <td>{check.httpStatus ?? '—'}</td><td>{check.latencyMs} ms</td><td>{check.failureReason ?? '—'}</td>
+              </tr>)}</tbody>
+            </table>
+          </div>}
+        </section>}
       </article>)}
     </section>
   </main>;
diff --git a/app/style.css b/app/style.css
index b77730e..d2aa5d1 100644
--- a/app/style.css
+++ b/app/style.css
@@ -22,3 +22,8 @@ button:focus-visible, input:focus-visible { outline: 3px solid #ae741b; outline-
 article { margin: 16px 0; }
 .url { overflow-wrap: anywhere; color: #52617a; }
 .result { display: flex; flex-wrap: wrap; gap: 16px; margin-top: 16px; padding-top: 16px; border-top: 1px solid #e3e8ef; }
+.actions { display: flex; flex-wrap: wrap; gap: 8px; }
+article form { margin-top: 16px; }
+.history-table { overflow-x: auto; }
+table { width: 100%; border-collapse: collapse; font-size: .85rem; }
+th, td { padding: 10px 8px; border-bottom: 1px solid #e3e8ef; text-align: left; white-space: nowrap; }
diff --git a/tests/browser/monitor.spec.ts b/tests/browser/monitor.spec.ts
index c28f8b2..4369a10 100644
--- a/tests/browser/monitor.spec.ts
+++ b/tests/browser/monitor.spec.ts
@@ -27,3 +27,38 @@ test('display HTTP failure as a completed check result', async ({ page }) => {
   await expect(monitor.getByText('HTTP 503', { exact: true })).toBeVisible();
   await expect(monitor.getByText('HTTP_STATUS', { exact: true })).toBeVisible();
 });
+
+test('persist history, edits, pause, activation and deletion across page reloads', async ({ page }) => {
+  await page.goto('/monitors');
+  await page.getByLabel('Name', { exact: true }).fill('Lifecycle fixture');
+  await page.getByLabel('URL', { exact: true }).fill('http://127.0.0.1:4321/ok');
+  await page.getByLabel('Interval (seconds)', { exact: true }).fill('60');
+  await page.getByLabel('Enabled', { exact: true }).check();
+  await page.getByRole('button', { name: 'Create monitor' }).click();
+  let monitor = page.getByRole('article', { name: 'Lifecycle fixture', exact: true });
+  await monitor.getByRole('button', { name: 'Run check' }).click();
+  await expect(monitor.getByText('SUCCEEDED', { exact: true })).toBeVisible();
+  await monitor.getByRole('button', { name: 'Run check' }).click();
+  await monitor.getByRole('button', { name: 'Show history' }).click();
+  await expect(monitor.getByRole('table', { name: 'Check history' }).getByRole('row')).toHaveCount(3);
+  await monitor.getByRole('button', { name: 'Edit', exact: true }).click();
+  await monitor.getByLabel('Edit name', { exact: true }).fill('Updated lifecycle fixture');
+  await monitor.getByLabel('Edit interval (seconds)', { exact: true }).fill('90');
+  await monitor.getByRole('button', { name: 'Save changes' }).click();
+  monitor = page.getByRole('article', { name: 'Updated lifecycle fixture', exact: true });
+  await expect(monitor.getByText('90s interval · Enabled', { exact: true })).toBeVisible();
+  await monitor.getByRole('button', { name: 'Pause', exact: true }).click();
+  await expect(monitor.getByText('90s interval · Paused', { exact: true })).toBeVisible();
+  await page.reload();
+  await expect(monitor.getByText('90s interval · Paused', { exact: true })).toBeVisible();
+  await monitor.getByRole('button', { name: 'Show history' }).click();
+  await expect(monitor.getByRole('table', { name: 'Check history' }).getByRole('row')).toHaveCount(3);
+  await monitor.getByRole('button', { name: 'Activate', exact: true }).click();
+  await expect(monitor.getByText('90s interval · Enabled', { exact: true })).toBeVisible();
+  await page.reload();
+  await expect(monitor.getByText('90s interval · Enabled', { exact: true })).toBeVisible();
+  await monitor.getByRole('button', { name: 'Delete', exact: true }).click();
+  await expect(monitor).toHaveCount(0);
+  await page.reload();
+  await expect(monitor).toHaveCount(0);
+});


