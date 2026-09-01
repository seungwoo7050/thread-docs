## `Expose durable Monitor lifecycle in the browser`

diff --git a/.github/workflows/check.yml b/.github/workflows/check.yml
index 1899fe6..3db77ca 100644
--- a/.github/workflows/check.yml
+++ b/.github/workflows/check.yml
@@ -27,6 +27,10 @@ jobs:
       - run: npm run typecheck
       - name: Unit
         run: npm run test:unit
+      - name: Start isolated PostgreSQL
+        run: npm run db:up
+      - name: PostgreSQL migrations and persistence
+        run: npm run test:integration
       - name: Functional
         run: npm run test:functional
       - name: Install pinned Chromium
@@ -35,3 +39,6 @@ jobs:
         run: npm run test:e2e
       - name: Next build
         run: npm run build
+      - name: Stop isolated PostgreSQL
+        if: always()
+        run: npm run db:down
diff --git a/TRACK.md b/TRACK.md
index d3da176..1adcdf6 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -2,14 +2,15 @@
 
 Spec revision: `0a006589477f8ae47bad3faa5510c999cff85ee4`.
 
-E01 establishes Monitor creation and synchronous manual GET checks in process memory.
+E01 established Monitor creation and synchronous manual GET checks in process memory.
+E03 stores Monitors and every terminal CheckRun in PostgreSQL, including after API restart.
 Only the configured fixture origin is eligible for outbound requests. The default
 is `http://127.0.0.1:4311`; API and fixture bind to `127.0.0.1`.
 Checks have a fixed 2 second total deadline, perform no retries, do not follow
 redirects, and close the response after observing headers without retaining a body.
 HTTP 200–299 is `SUCCEEDED`; other observed statuses are `FAILED/HTTP_STATUS`.
 A transport failure records `FAILED/CONNECTION_FAILURE` or `FAILED/TIMEOUT` with
-`httpStatus: null`. Only the latest terminal result per Monitor is retained.
+`httpStatus: null`. The list shows the latest terminal result; E03 history reads retain all results.
 `enabled` and `interval` are stored fields; there is no scheduled execution.
 
 ## Fixed versions
@@ -27,6 +28,9 @@ A transport failure records `FAILED/CONNECTION_FAILURE` or `FAILED/TIMEOUT` with
 | @types/node | 24.13.3 |
 | @types/react | 19.2.18 |
 | @types/react-dom | 19.2.5 |
+| PostgreSQL Docker image | 17.6-bookworm |
+| pg | 8.16.3 |
+| @types/pg | 8.15.5 |
 
 Direct dependencies are exact in `package.json`; transitives are pinned in
 `package-lock.json`. Node supplies TypeScript stripping and the unit/functional
@@ -47,6 +51,8 @@ With the exact Node/npm versions active:
 
 ```sh
 npm ci
+npm run db:up
+npm run db:migrate
 npm run fixture
 # A separate terminal:
 npm run dev:api
@@ -66,6 +72,7 @@ Do not expose this development service to a public interface.
 npm run typecheck
 npm run test:unit
 npm run test:functional
+npm run test:integration
 npx playwright install chromium
 npm run test:e2e
 npm run build
@@ -73,8 +80,8 @@ npm run build
 
 Functional tests own fixed loopback ports 4311, 4312 and 4314; stop manual fixture
 processes before running them. Port 4314 is a controlled destination that must
-never receive a check. State disappears on API restart. There is no login,
-database, worker, Redis, Kafka, or production container in E01.
+never receive a check. E03 uses PostgreSQL on loopback port 15431. There is no login,
+worker, Redis, Kafka, or production application container.
 The E2E command owns ports 4311–4313 and starts fresh fixture/API/Next processes;
 stop manual development processes first. It uses one Chromium worker, no test
 retries, and the fixed Monitor fixture in `evidence/E01/scenario.json`.
@@ -83,7 +90,8 @@ retries, and the fixed Monitor fixture in `evidence/E01/scenario.json`.
 
 Creation requires a JSON object containing `name`, `url`, `interval`, and `enabled`.
 The name is trimmed and must then contain 1–100 UTF-16 code units (JavaScript
-`String.length`). Interval is a numeric integer from 1 to 86400 seconds; JSON
+`String.length`). E03 rejects NUL characters because [PostgreSQL text cannot store them](https://www.postgresql.org/docs/17/datatype-character.html).
+Interval is a numeric integer from 1 to 86400 seconds; JSON
 `60` and `60.0` have the same parsed value, but `"60"` is rejected. Enabled must
 be a boolean. URL must be absolute HTTP(S), without credentials, and use the
 configured fixture origin; the outbound check repeats that boundary check.
@@ -116,9 +124,91 @@ validation uses standard JavaScript and Fastify's existing typed error classes.
 ## Basic CI
 
 `.github/workflows/check.yml` runs the type, unit, functional, Chromium E2E, and
-Next build checks. It selects Node from `.node-version`, installs npm 11.17.0,
+Next build checks, plus PostgreSQL migration and persistence tests. It selects Node from `.node-version`, installs npm 11.17.0,
 and installs dependencies from the lockfile. CI actions are pinned to the full
 commits for [checkout 4.2.2](https://github.com/actions/checkout/releases/tag/v4.2.2)
 and [setup-node 4.4.0](https://github.com/actions/setup-node/releases/tag/v4.4.0).
 Local invocation outcomes and counts are recorded in `evidence/E01/verification.json`;
 a hosted CI run is not claimed by those local results.
+
+## E03 PostgreSQL lifecycle and mapping
+
+The new image is the official [PostgreSQL 17.6 release](https://www.postgresql.org/docs/17/release-17-6.html)
+and [Postgres Docker image](https://hub.docker.com/_/postgres), fixed in `compose.yaml`
+as `postgres:17.6-bookworm@sha256:f3bd19c606e442c3d7bdfa8002e03fe260a1023351e0ea4598032022b68dd6e3`.
+The new packages are exact `pg@8.16.3` and `@types/pg@8.15.5`; earlier runtime and
+package pins are unchanged. Node's standard library reads/checksums migrations,
+so no separate migration framework is required.
+
+`npm run db:up` starts only the `wse-fundamentals` Compose project. Its database
+port is bound to `127.0.0.1:15431`, with an explicit **local test trust-auth policy**
+and no password. This is not a production database or credential configuration.
+The default connection is `postgresql://monitor@127.0.0.1:15431/monitor` and the
+application schema is `public`. `DATABASE_URL` and `DATABASE_SCHEMA` can select a
+different disposable database/schema; do not commit a DSN containing credentials.
+`npm run db:down` stops only this project and retains its named data volume.
+There is no application container.
+
+Run `npm run db:migrate` before starting the API. It applies `001_monitors.sql`
+and then `002_check_runs.sql`, on one explicit `pg` client within a DDL
+transaction, and records exact file checksums in `schema_migrations`. Run this
+command serially; a repeated command prints `{"applied":[]}`. Applied files must
+not be edited. Startup never creates tables: it requires the complete matching
+migration history, the exact mapped columns and their types/nullability/timestamp precision,
+primary keys and the CheckRun parent cascade. A missing or incompatible mapped
+column rejects `app.listen` before the API opens a port.
+
+`server/mapping.ts` is the explicit SQL-row/domain boundary. `interval_seconds`
+and `latency_ms` are SQL integers, flags are booleans, IDs are UUID strings,
+and `timestamptz(3)` values become UTC ISO strings. SQL names never leak through
+API responses. Null HTTP status and failure reason are not coerced or omitted.
+The [pg type documentation](https://node-postgres.com/features/types) describes
+its Date/UUID parsing; millisecond timestamp precision matches JavaScript's Date.
+The [pg transaction documentation](https://node-postgres.com/features/transactions)
+requires all statements in a transaction to use the same checked-out client.
+
+E03 routes retain E02's envelope and failure categories:
+
+| Operation | Route | Success |
+| --- | --- | --- |
+| Read one Monitor/latest result | `GET /monitors/:id` | 200, Monitor view |
+| Replace editable fields | `PUT /monitors/:id` | 200, updated Monitor view |
+| Delete Monitor and all its CheckRuns | `DELETE /monitors/:id` | 200, `{id}` |
+| Read all Monitor history | `GET /monitors/:id/checks` | 200, CheckRun array |
+| Read a CheckRun | `GET /checks/:id` | 200, CheckRun |
+
+PUT requires the same complete `name`, `url`, `interval`, `enabled` input as
+creation. Pause/enable changes `enabled` using this contract. Manual checks
+remain available while paused; no scheduler is present. Every read and mutation
+uses PostgreSQL. Deletion uses the database foreign-key cascade; deleted Monitor
+and CheckRun resources return `NOT_FOUND`. History is ordered by finished time
+then ID descending and has no pagination yet. The UI exposes these operations
+on `/monitors`, including a confirmation before deleting a Monitor's history.
+
+For independent verification, activate Node/npm from the version pins, then:
+
+```sh
+npm ci
+npm run db:up
+npm run typecheck
+npm run test:unit
+npm run test:functional
+npm run test:integration
+npx playwright install chromium
+npm run test:e2e
+npm run build
+npm run db:down
+```
+
+Functional/integration tests recreate only their explicit `e03_*` schemas and
+remove them afterwards; they never truncate the application's `public` schema.
+The browser API command prepares `e03_browser`, and Playwright teardown removes
+it. Every test uses one worker/serial execution and zero automatic retries.
+The browser tests operate real fixture, Fastify and Next processes and a real
+PostgreSQL schema. CI starts/stops the same Compose project and keeps all earlier
+gates. Local tests do not imply a hosted CI run.
+
+The frozen packet, one memory-loss baseline, real PostgreSQL results, command
+outcomes and durations are in `evidence/E03/`. The historical E01/E02 evidence is
+unchanged; only E01's now-obsolete empty-new-instance assertion was replaced by
+the E03 persistence assertion.
diff --git a/app/monitors/page.tsx b/app/monitors/page.tsx
index d62a3fa..c41fc4d 100644
--- a/app/monitors/page.tsx
+++ b/app/monitors/page.tsx
@@ -10,6 +10,12 @@ export default function MonitorsPage() {
   const [error, setError] = useState<ApiErrorCode | null>(null);
   const [creating, setCreating] = useState(false);
   const [checking, setChecking] = useState<string | null>(null);
+  const [editing, setEditing] = useState<string | null>(null);
+  const [saving, setSaving] = useState<string | null>(null);
+  const [deleting, setDeleting] = useState<string | null>(null);
+  const [historyId, setHistoryId] = useState<string | null>(null);
+  const [history, setHistory] = useState<CheckRun[]>([]);
+  const [loadingHistory, setLoadingHistory] = useState(false);
 
   useEffect(() => {
     fetch('/api/monitors').then(responseData<MonitorView[]>).then(setMonitors)
@@ -45,16 +51,66 @@ export default function MonitorsPage() {
       const response = await fetch(`/api/monitors/${id}/checks`, { method: 'POST' });
       const result = await responseData<CheckRun>(response);
       setMonitors((current) => current.map((monitor) => monitor.id === id ? { ...monitor, latestCheck: result } : monitor));
+      if (historyId === id) await showHistory(id);
     } catch (failure) {
       setError(failureCode(failure));
     } finally { setChecking(null); }
   }
 
+  async function updateMonitor(monitor: MonitorView, input: { name: string; url: string; interval: number; enabled: boolean }) {
+    setSaving(monitor.id);
+    setError(null);
+    try {
+      const response = await fetch(`/api/monitors/${monitor.id}`, {
+        method: 'PUT', headers: { 'content-type': 'application/json' }, body: JSON.stringify(input),
+      });
+      const saved = await responseData<MonitorView>(response);
+      setMonitors((current) => current.map((item) => item.id === saved.id ? saved : item));
+      setEditing(null);
+    } catch (failure) { setError(failureCode(failure)); }
+    finally { setSaving(null); }
+  }
+
+  async function editMonitor(event: FormEvent<HTMLFormElement>, monitor: MonitorView) {
+    event.preventDefault();
+    const fields = new FormData(event.currentTarget);
+    await updateMonitor(monitor, {
+      name: String(fields.get('name')), url: String(fields.get('url')),
+      interval: Number(fields.get('interval')), enabled: monitor.enabled,
+    });
+  }
+
+  async function deleteMonitor(id: string) {
+    if (!window.confirm('Delete this monitor and all its check history?')) return;
+    setDeleting(id);
+    setError(null);
+    try {
+      const response = await fetch(`/api/monitors/${id}`, { method: 'DELETE' });
+      await responseData<{ id: string }>(response);
+      setMonitors((current) => current.filter((monitor) => monitor.id !== id));
+      if (historyId === id) { setHistoryId(null); setHistory([]); }
+      if (editing === id) setEditing(null);
+    } catch (failure) { setError(failureCode(failure)); }
+    finally { setDeleting(null); }
+  }
+
+  async function showHistory(id: string) {
+    setHistoryId(id);
+    setHistory([]);
+    setLoadingHistory(true);
+    setError(null);
+    try {
+      const response = await fetch(`/api/monitors/${id}/checks`);
+      setHistory(await responseData<CheckRun[]>(response));
+    } catch (failure) { setError(failureCode(failure)); }
+    finally { setLoadingHistory(false); }
+  }
+
   return <main>
     <header><p className="eyebrow">HTTP ENDPOINT MONITOR · LOCAL FIXTURE</p><h1>Monitors</h1>
       <p>Create an endpoint monitor, run a check, and inspect the response.</p>
     </header>
-    <aside>Development only. Checks can access the configured fixture origin. State is lost on API restart.</aside>
+    <aside>Development only. Checks can access the configured fixture origin. Monitors and all check history are stored in PostgreSQL.</aside>
     {error && <p role="alert" className="error" data-error-code={error}>{ERROR_MESSAGES[error]}</p>}
     <section aria-labelledby="create-heading" className="panel">
       <h2 id="create-heading">Create monitor</h2>
@@ -77,9 +133,33 @@ export default function MonitorsPage() {
         <h3 id={`monitor-${monitor.id}`}>{monitor.name}</h3>
         <p className="endpoint">{monitor.url}</p>
         <p>{monitor.enabled ? 'Enabled' : 'Paused'} · {monitor.interval} seconds</p>
-        <button onClick={() => runCheck(monitor.id)} disabled={checking !== null}>
+        <div className="actions">
+        <button onClick={() => runCheck(monitor.id)} disabled={checking !== null || saving !== null || deleting !== null}>
           {checking === monitor.id ? 'Checking…' : 'Run check'}
         </button>
+        <button onClick={() => setEditing(editing === monitor.id ? null : monitor.id)} disabled={saving !== null || deleting !== null}>
+          {editing === monitor.id ? 'Cancel edit' : 'Edit monitor'}
+        </button>
+        <button onClick={() => updateMonitor(monitor, { ...monitor, enabled: !monitor.enabled })} disabled={saving !== null || deleting !== null}>
+          {monitor.enabled ? 'Pause monitor' : 'Enable monitor'}
+        </button>
+        <button onClick={() => deleteMonitor(monitor.id)} disabled={saving !== null || deleting !== null || checking !== null}>
+          {deleting === monitor.id ? 'Deleting…' : 'Delete monitor'}
+        </button>
+        <button onClick={() => historyId === monitor.id ? setHistoryId(null) : showHistory(monitor.id)} disabled={loadingHistory}>
+          {historyId === monitor.id ? 'Hide history' : 'View history'}
+        </button>
+        </div>
+        {editing === monitor.id && <form onSubmit={(event) => editMonitor(event, monitor)} className="edit-form">
+          <label htmlFor={`edit-name-${monitor.id}`}>Edit name</label>
+          <input id={`edit-name-${monitor.id}`} name="name" required defaultValue={monitor.name} />
+          <label htmlFor={`edit-url-${monitor.id}`}>Edit endpoint URL</label>
+          <input id={`edit-url-${monitor.id}`} name="url" type="url" required defaultValue={monitor.url} />
+          <label htmlFor={`edit-interval-${monitor.id}`}>Edit interval (seconds)</label>
+          <input id={`edit-interval-${monitor.id}`} name="interval" type="number" min="1" max="86400" step="1" required defaultValue={monitor.interval} />
+          <button type="submit" disabled={saving !== null}>{saving === monitor.id ? 'Saving…' : 'Save changes'}</button>
+        </form>}
+        {!monitor.enabled && <p className="hint">Paused. Manual checks remain available; no scheduler runs in this version.</p>}
         {monitor.latestCheck ? <dl aria-label="Latest check result">
           <dt>State</dt><dd>{monitor.latestCheck.state}</dd>
           <dt>HTTP status</dt><dd>{monitor.latestCheck.httpStatus ?? 'No response'}</dd>
@@ -87,6 +167,18 @@ export default function MonitorsPage() {
           <dt>Failure reason</dt><dd>{monitor.latestCheck.failureReason ?? 'None'}</dd>
           <dt>Finished</dt><dd><time dateTime={monitor.latestCheck.finishedAt}>{monitor.latestCheck.finishedAt}</time></dd>
         </dl> : <p>No checks yet.</p>}
+        {historyId === monitor.id && <section aria-label={`Check history for ${monitor.name}`}>
+          <h4>Check history</h4>
+          {loadingHistory ? <p>Loading history…</p> : history.length === 0 ? <p>No historical checks.</p> :
+            <div className="history-scroll"><table>
+              <thead><tr><th>Check ID</th><th>State</th><th>HTTP status</th><th>Latency</th><th>Failure reason</th><th>Finished</th></tr></thead>
+              <tbody>{history.map((check) => <tr key={check.id}>
+                <td>{check.id}</td><td>{check.state}</td><td>{check.httpStatus ?? 'No response'}</td>
+                <td>{check.latencyMs} ms</td><td>{check.failureReason ?? 'None'}</td>
+                <td><time dateTime={check.finishedAt}>{check.finishedAt}</time></td>
+              </tr>)}</tbody>
+            </table></div>}
+        </section>}
       </article>)}
     </section>
   </main>;
diff --git a/app/style.css b/app/style.css
index 01a2dfb..4d9b48e 100644
--- a/app/style.css
+++ b/app/style.css
@@ -22,3 +22,10 @@ button:disabled { opacity: .6; cursor: wait; }
 dl { display: grid; grid-template-columns: 110px minmax(0, 1fr); gap: 10px; margin-bottom: 0; }
 dt { color: #51626f; }
 dd { margin: 0; overflow-wrap: anywhere; }
+
+.actions { display: flex; flex-wrap: wrap; gap: 8px; }
+.edit-form { margin-top: 20px; }
+.history-scroll { overflow-x: auto; }
+table { width: 100%; border-collapse: collapse; font-size: .85rem; }
+th, td { padding: 10px; text-align: left; border-bottom: 1px solid #cbd6dd; vertical-align: top; }
+td:first-child { font-family: monospace; min-width: 180px; overflow-wrap: anywhere; }
diff --git a/evidence/E03/browser-lifecycle.png b/evidence/E03/browser-lifecycle.png
new file mode 100644
index 0000000..434516f
Binary files /dev/null and b/evidence/E03/browser-lifecycle.png differ
diff --git a/evidence/E03/verification.json b/evidence/E03/verification.json
new file mode 100644
index 0000000..5a99574
--- /dev/null
+++ b/evidence/E03/verification.json
@@ -0,0 +1,404 @@
+{
+  "thread": "E03",
+  "attempt": 1,
+  "specRevision": "0a006589477f8ae47bad3faa5510c999cff85ee4",
+  "start": "2d26b6b08e7580eef35a61093f41d7cc60f9de15",
+  "scenarioSha256": "d2b15355837aee384a01e74ca9eb7c3a250713a4df383d4fd4d22125ad414ce6",
+  "versions": {
+    "node": "24.19.0",
+    "npm": "11.17.0",
+    "postgresImage": "postgres:17.6-bookworm@sha256:f3bd19c606e442c3d7bdfa8002e03fe260a1023351e0ea4598032022b68dd6e3",
+    "pg": "8.16.3",
+    "typesPg": "8.15.5",
+    "existingPinsChanged": false
+  },
+  "baseline": {
+    "command": "fnm exec --using 24.19.0 node --input-type=module < evidence/E03/baseline-reproducer.mjs",
+    "executionNote": "The recorded script was supplied verbatim as stdin before production changes at START; the named file preserves that stdin for replay at START, not at E03 HEAD.",
+    "runs": 1,
+    "durationMs": 68,
+    "exitCode": 0,
+    "result": "REPRODUCED",
+    "expectedMonitorsAfterNewInstance": 2,
+    "actualMonitorsAfterNewInstance": 0,
+    "fixtureCalls": {
+      "/ok": 2,
+      "/fail": 1
+    }
+  },
+  "commands": [
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run typecheck",
+      "phase": "database/API/tests, before UI changes",
+      "exitCode": 0,
+      "wallSeconds": 1.8
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:unit",
+      "exitCode": 0,
+      "tests": 7,
+      "passed": 7,
+      "failed": 0,
+      "runnerDurationMs": 144.843833,
+      "wallSeconds": 0.32
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:integration",
+      "exitCode": 0,
+      "tests": 4,
+      "passed": 4,
+      "failed": 0,
+      "runnerDurationMs": 2594.586666,
+      "wallSeconds": 3.6,
+      "cases": [
+        {
+          "name": "migration CLI fresh chain and repeat",
+          "durationMs": 1081.708333
+        },
+        {
+          "name": "missing column rejects startup",
+          "durationMs": 124.439083
+        },
+        {
+          "name": "A,A,B fresh app and lifecycle",
+          "durationMs": 615.159333
+        },
+        {
+          "name": "canonical SQL/domain mapping",
+          "durationMs": 123.990958
+        }
+      ]
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:functional",
+      "exitCode": 0,
+      "tests": 13,
+      "passed": 13,
+      "failed": 0,
+      "runnerDurationMs": 3496.287125,
+      "wallSeconds": 4.29
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run typecheck",
+      "phase": "completed UI and browser tests",
+      "exitCode": 0,
+      "wallSeconds": 3.81
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:e2e",
+      "exitCode": 0,
+      "browser": "Chromium revision1234 / Playwright1.62.1",
+      "tests": 5,
+      "passed": 5,
+      "failed": 0,
+      "workers": 1,
+      "retries": 0,
+      "runnerSeconds": 10.7,
+      "wallSeconds": 11.23,
+      "cases": [
+        {
+          "name": "INVALID_INPUT UI category",
+          "displayedDurationSeconds": 2.0
+        },
+        {
+          "name": "NOT_FOUND UI category",
+          "displayedDurationMs": 588
+        },
+        {
+          "name": "INTERNAL_ERROR UI category",
+          "displayedDurationMs": 416
+        },
+        {
+          "name": "E03 real lifecycle and A,A,B history",
+          "displayedDurationSeconds": 3.1
+        },
+        {
+          "name": "E01 create/check/reload",
+          "displayedDurationMs": 929
+        }
+      ]
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run build",
+      "exitCode": 0,
+      "wallSeconds": 4.67,
+      "routes": [
+        "/",
+        "/_not-found",
+        "/monitors"
+      ]
+    },
+    {
+      "command": "git diff --check",
+      "exitCode": 0,
+      "durationNote": "read-only integrity check; elapsed time not separately instrumented"
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run typecheck",
+      "phase": "after confirmed supplemental boundary fixes",
+      "exitCode": 0,
+      "wallSeconds": 1.55
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:unit",
+      "phase": "after confirmed supplemental boundary fixes",
+      "exitCode": 0,
+      "tests": 7,
+      "passed": 7,
+      "failed": 0,
+      "runnerDurationMs": 122.405291,
+      "wallSeconds": 0.36
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:integration",
+      "phase": "after confirmed supplemental boundary fixes",
+      "exitCode": 0,
+      "tests": 6,
+      "passed": 6,
+      "failed": 0,
+      "runnerDurationMs": 1205.510875,
+      "wallSeconds": 1.39,
+      "cases": [
+        {
+          "name": "migration CLI fresh chain and repeat",
+          "durationMs": 317.075875
+        },
+        {
+          "name": "missing column rejects startup",
+          "durationMs": 38.011542
+        },
+        {
+          "name": "A,A,B fresh app and lifecycle",
+          "durationMs": 215.954
+        },
+        {
+          "name": "canonical SQL/domain mapping",
+          "durationMs": 45.86775
+        },
+        {
+          "name": "NUL create/update input",
+          "durationMs": 86.462875
+        },
+        {
+          "name": "unexpected required column rejects startup",
+          "durationMs": 83.842791
+        }
+      ]
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:functional",
+      "phase": "after confirmed supplemental boundary fixes",
+      "exitCode": 0,
+      "tests": 13,
+      "passed": 13,
+      "failed": 0,
+      "runnerDurationMs": 2658.798875,
+      "wallSeconds": 2.81
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:e2e",
+      "phase": "after confirmed supplemental boundary fixes",
+      "exitCode": 0,
+      "browser": "Chromium revision1234 / Playwright1.62.1",
+      "tests": 5,
+      "passed": 5,
+      "failed": 0,
+      "workers": 1,
+      "retries": 0,
+      "runnerSeconds": 7.0,
+      "wallSeconds": 7.66,
+      "cases": [
+        {
+          "name": "INVALID_INPUT UI category",
+          "displayedDurationMs": 968
+        },
+        {
+          "name": "NOT_FOUND UI category",
+          "displayedDurationMs": 400
+        },
+        {
+          "name": "INTERNAL_ERROR UI category",
+          "displayedDurationMs": 286
+        },
+        {
+          "name": "E03 real lifecycle and A,A,B history",
+          "displayedDurationSeconds": 1.4
+        },
+        {
+          "name": "E01 create/check/reload",
+          "displayedDurationMs": 665
+        }
+      ]
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run build",
+      "phase": "after confirmed supplemental boundary fixes",
+      "exitCode": 0,
+      "wallSeconds": 2.67,
+      "routes": [
+        "/",
+        "/_not-found",
+        "/monitors"
+      ]
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run db:down",
+      "phase": "final own-project cleanup after verification",
+      "exitCode": 0,
+      "wallSeconds": 0.57,
+      "result": "wse-fundamentals container/network removed; named volume retained"
+    },
+    {
+      "command": "git diff --exit-code 2d26b6b08e7580eef35a61093f41d7cc60f9de15 -- evidence/E01 evidence/E02 SPEC_REVISION .node-version server/check.ts",
+      "exitCode": 0,
+      "result": "earlier frozen evidence, runtime pin, spec revision and outbound implementation unchanged",
+      "durationNote": "read-only integrity check; not separately instrumented"
+    }
+  ],
+  "setup": [
+    {
+      "command": "fnm exec --using 24.19.0 npm view pg@8.16.3 version && fnm exec --using 24.19.0 npm view @types/pg@8.15.5 version",
+      "exitCode": 0,
+      "output": [
+        "8.16.3",
+        "8.15.5"
+      ],
+      "durationNote": "not instrumented"
+    },
+    {
+      "command": "fnm exec --using 24.19.0 npm install --save-exact pg@8.16.3 && fnm exec --using 24.19.0 npm install --save-dev --save-exact @types/pg@8.15.5",
+      "exitCode": 0,
+      "npmReportedDurationMs": [
+        707,
+        287
+      ],
+      "addedPackages": [
+        13,
+        1
+      ]
+    },
+    {
+      "command": "docker manifest inspect postgres:17.6-bookworm",
+      "exitCode": 0,
+      "durationNote": "not instrumented"
+    },
+    {
+      "command": "docker compose --project-name wse-fundamentals up -d --wait",
+      "exitCode": 0,
+      "result": "dedicated PostgreSQL container healthy",
+      "durationNote": "not instrumented; image pulled once"
+    },
+    {
+      "command": "docker image inspect postgres:17.6-bookworm --format '{{json .RepoDigests}}'",
+      "exitCode": 0,
+      "result": "resolved official manifest digest pinned in compose.yaml",
+      "durationNote": "not instrumented"
+    }
+  ],
+  "environmentFailures": [
+    {
+      "command": "baseline stdin invocation before escalation",
+      "exitCode": 1,
+      "error": "listen EPERM 127.0.0.1:4311",
+      "effect": "failed at first listener before any dataset request; no baseline observation; same command was approved and then run once",
+      "durationNote": "process failed before scenario duration could be recorded"
+    },
+    {
+      "command": "initial file inspection included .github/workflows/ci.yml",
+      "exitCode": 1,
+      "error": "file does not exist",
+      "effect": "correct existing .github/workflows/check.yml was subsequently read; no code/test effect",
+      "durationNote": "read-only inspection not instrumented"
+    },
+    {
+      "command": "git add/commit PostgreSQL migration boundary before escalation",
+      "exitCode": 128,
+      "error": "index.lock Operation not permitted in assigned worktree Git metadata",
+      "effect": "Exact stage/commit command rerun with explicit escalation succeeded; no scope/branch changes.",
+      "durationNote": "permission denial before commit; not separately instrumented"
+    },
+    {
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run db:down before escalation",
+      "exitCode": 1,
+      "wallSeconds": 0.45,
+      "error": "Docker daemon socket permission denied",
+      "effect": "No project change before denial; exact command rerun with explicit escalation succeeded."
+    }
+  ],
+  "warnings": [
+    "npm reported pending optional fsevents install script; no allowScripts approval or execution was added.",
+    "Playwright reported NO_COLOR/FORCE_COLOR warnings; no assertion failure."
+  ],
+  "evidence": {
+    "postgres": [
+      "migrations.json",
+      "schema-mismatch.json",
+      "persistence.json",
+      "mapping.json"
+    ],
+    "browserScreenshot": "browser-lifecycle.png",
+    "browserScreenshotSha256": "d156c97ae4ab6754ce5d6c038d0904988948f77120131fe38198e80f3216242f",
+    "screenshotVisuallyInspected": true
+  },
+  "ci": {
+    "workflow": ".github/workflows/check.yml",
+    "retains": [
+      "typecheck",
+      "unit",
+      "functional",
+      "browser",
+      "Next build"
+    ],
+    "adds": [
+      "dedicated PostgreSQL start/stop",
+      "migration/persistence integration"
+    ],
+    "hostedCiRunClaimed": false
+  },
+  "regressions": "E01/E02 frozen scenario/evidence files unchanged; only obsolete empty-memory assertion adapted to durable state.",
+  "retries": 0,
+  "loadRuns": 0,
+  "benchmarkRuns": 0,
+  "parameterSweeps": 0,
+  "unresolved": [],
+  "supplemental": {
+    "scenario": "supplemental-scenario.json",
+    "scenarioSha256": "2f70d2918b2cf439623972f57cf86f25f8ab9a823ace0debdfce00a937589446",
+    "baselineCandidateSha": "8fcea6d7db7d0bfab13b6d2c985bfcb4f5a17e70",
+    "baselineCommand": "/usr/bin/time -p fnm exec --using 24.19.0 node --input-type=module < evidence/E03/supplemental-reproducer.mjs",
+    "baselineRuns": 1,
+    "baselineExitCode": 0,
+    "baselineDurationMs": 379,
+    "baselineWallSeconds": 0.85,
+    "baselineResult": "Both confirmed: extra required column allowed startup then valid insert500; NUL name was500/no creation.",
+    "fixes": [
+      "Strict expected column set at startup",
+      "NUL name INVALID_INPUT400 on create/update"
+    ],
+    "postfixEvidence": [
+      "supplemental-schema.json",
+      "supplemental-contract.json"
+    ],
+    "originalScenarioUnchanged": true
+  },
+  "invocationCounts": {
+    "primaryBaselineCompleted": 1,
+    "supplementalBaselineCompleted": 1,
+    "typecheck": 3,
+    "unit": 2,
+    "integration": 2,
+    "functional": 2,
+    "browser": 2,
+    "build": 2,
+    "automaticTestRetries": 0,
+    "note": "Second correctness-gate passes followed the two newly confirmed supplemental fixes; no performance exploration."
+  },
+  "finalCounts": {
+    "unit": 7,
+    "integration": 6,
+    "functional": 13,
+    "browser": 5,
+    "failed": 0
+  },
+  "finalDatabaseState": "Only wse-fundamentals stopped. For independent verification run npm run db:up, then the gates in TRACK.md. The named PostgreSQL volume is retained."
+}
diff --git a/playwright.config.ts b/playwright.config.ts
index 21028f2..1f657f2 100644
--- a/playwright.config.ts
+++ b/playwright.config.ts
@@ -3,6 +3,7 @@ import { defineConfig, devices } from '@playwright/test';
 export default defineConfig({
   testDir: './test/browser',
   outputDir: './output/playwright',
+  globalTeardown: './test/browser-teardown.ts',
   fullyParallel: false,
   workers: 1,
   retries: 0,
@@ -12,7 +13,8 @@ export default defineConfig({
   projects: [{ name: 'chromium', use: { ...devices['Desktop Chrome'] } }],
   webServer: [
     { command: 'npm run fixture', url: 'http://127.0.0.1:4311/ok', reuseExistingServer: false, timeout: 30_000 },
-    { command: 'npm run start:api', url: 'http://127.0.0.1:4312/health', reuseExistingServer: false, timeout: 30_000 },
+    { command: 'node test/prepare-browser-db.ts && npm run start:api', url: 'http://127.0.0.1:4312/health', reuseExistingServer: false, timeout: 30_000,
+      env: { DATABASE_SCHEMA: 'e03_browser' } },
     { command: 'npm run dev:web', url: 'http://127.0.0.1:4313/monitors', reuseExistingServer: false, timeout: 90_000,
       env: { NEXT_TELEMETRY_DISABLED: '1' } },
   ],
diff --git a/test/browser-teardown.ts b/test/browser-teardown.ts
new file mode 100644
index 0000000..cf9a436
--- /dev/null
+++ b/test/browser-teardown.ts
@@ -0,0 +1,5 @@
+import { dropTestSchema } from './database.ts';
+
+export default async function teardown() {
+  await dropTestSchema('e03_browser');
+}
diff --git a/test/browser/lifecycle.spec.ts b/test/browser/lifecycle.spec.ts
new file mode 100644
index 0000000..f8de350
--- /dev/null
+++ b/test/browser/lifecycle.spec.ts
@@ -0,0 +1,71 @@
+import { expect, test } from '@playwright/test';
+import type { CheckRun, MonitorView } from '../../server/model';
+import scenario from '../../evidence/E03/scenario.json' with { type: 'json' };
+
+test('E03 persist A,A,B history, edit, pause, enable and delete through the real browser and PostgreSQL API', async ({ page, request }) => {
+  await page.goto('/monitors');
+  const created: MonitorView[] = [];
+  for (const input of scenario.monitors) {
+    await page.getByLabel('Name', { exact: true }).fill(input.name);
+    await page.getByLabel('Endpoint URL', { exact: true }).fill(input.url);
+    await page.getByLabel('Interval (seconds)', { exact: true }).fill(String(input.interval));
+    await page.getByLabel('Enabled', { exact: true }).check();
+    const response = page.waitForResponse((response) => response.url().endsWith('/api/monitors') && response.request().method() === 'POST');
+    await page.getByRole('button', { name: 'Create monitor', exact: true }).click();
+    created.push((await (await response).json()).data);
+    await expect(page.getByRole('article', { name: input.name, exact: true })).toBeVisible();
+  }
+  const checks: CheckRun[] = [];
+  for (const expected of scenario.checkSequence) {
+    const monitor = page.getByRole('article', { name: scenario.monitors[expected.monitor].name, exact: true });
+    const response = page.waitForResponse((response) => response.url().endsWith(`/api/monitors/${created[expected.monitor].id}/checks`) && response.request().method() === 'POST');
+    await monitor.getByRole('button', { name: 'Run check', exact: true }).click();
+    const result: CheckRun = (await (await response).json()).data;
+    checks.push(result);
+    expect(result.state).toBe(expected.state);
+    expect(result.httpStatus).toBe(expected.httpStatus);
+    await expect(monitor.locator('dl')).toContainText(expected.state);
+  }
+  await page.reload();
+  let a = page.getByRole('article', { name: scenario.monitors[0].name, exact: true });
+  const b = page.getByRole('article', { name: scenario.monitors[1].name, exact: true });
+  await a.getByRole('button', { name: 'View history', exact: true }).click();
+  const aHistory = a.getByRole('region', { name: `Check history for ${scenario.monitors[0].name}` });
+  await expect(aHistory.locator('tbody tr')).toHaveCount(2);
+  for (const check of checks.slice(0, 2)) {
+    await expect(aHistory).toContainText(check.id);
+    await expect(aHistory).toContainText(check.finishedAt);
+  }
+  await b.getByRole('button', { name: 'View history', exact: true }).click();
+  const bHistory = b.getByRole('region', { name: `Check history for ${scenario.monitors[1].name}` });
+  await expect(bHistory.locator('tbody tr')).toHaveCount(1);
+  await expect(bHistory).toContainText(checks[2].id);
+  await expect(bHistory).toContainText('FAILED');
+  await expect(bHistory).toContainText('503');
+  await a.getByRole('button', { name: 'Edit monitor', exact: true }).click();
+  await a.getByLabel('Edit name', { exact: true }).fill(scenario.update.name);
+  await a.getByLabel('Edit interval (seconds)', { exact: true }).fill(String(scenario.update.interval));
+  await a.getByRole('button', { name: 'Save changes', exact: true }).click();
+  a = page.getByRole('article', { name: scenario.update.name, exact: true });
+  await expect(a).toContainText('Enabled · 90 seconds');
+  await a.getByRole('button', { name: 'Pause monitor', exact: true }).click();
+  await expect(a).toContainText('Paused · 90 seconds');
+  await page.reload();
+  await expect(a).toContainText('Paused · 90 seconds');
+  await a.getByRole('button', { name: 'Enable monitor', exact: true }).click();
+  await expect(a).toContainText('Enabled · 90 seconds');
+  page.once('dialog', (dialog) => dialog.accept());
+  await b.getByRole('button', { name: 'Delete monitor', exact: true }).click();
+  await expect(b).toHaveCount(0);
+  await page.reload();
+  await expect(a).toContainText('Enabled · 90 seconds');
+  await expect(b).toHaveCount(0);
+  await a.getByRole('button', { name: 'View history', exact: true }).click();
+  await expect(a.locator('tbody tr')).toHaveCount(2);
+  for (const check of checks.slice(0, 2)) await expect(a.locator('tbody')).toContainText(check.id);
+  const removed = await request.get(`/api/checks/${checks[2].id}`);
+  expect(removed.status()).toBe(404);
+  expect((await removed.json()).error.code).toBe('NOT_FOUND');
+  await expect(page.getByRole('main').getByRole('alert')).toHaveCount(0);
+  await page.screenshot({ path: 'output/playwright/E03-lifecycle.png', fullPage: true });
+});
diff --git a/test/prepare-browser-db.ts b/test/prepare-browser-db.ts
new file mode 100644
index 0000000..e152a91
--- /dev/null
+++ b/test/prepare-browser-db.ts
@@ -0,0 +1,4 @@
+import { resetTestSchema } from './database.ts';
+
+await resetTestSchema('e03_browser');
+console.log('Prepared isolated e03_browser PostgreSQL schema.');
