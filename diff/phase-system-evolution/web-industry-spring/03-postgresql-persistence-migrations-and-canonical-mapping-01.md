# E03 PostgreSQL 영속성, Migration과 Canonical Mapping

## `API 재시작으로 사라지는 Monitor와 CheckRun 고정 반례 기록`

diff --git a/evidence/E03/baseline-persistence.json b/evidence/E03/baseline-persistence.json
new file mode 100644
index 0000000..0a887c1
--- /dev/null
+++ b/evidence/E03/baseline-persistence.json
@@ -0,0 +1,177 @@
+{
+  "label": "baseline",
+  "requests": [
+    {
+      "method": "GET",
+      "path": "/api/monitors",
+      "status": 200,
+      "wire": {
+        "data": []
+      },
+      "elapsedSeconds": 0.003
+    },
+    {
+      "method": "POST",
+      "path": "/api/monitors",
+      "status": 201,
+      "wire": {
+        "data": {
+          "monitor": {
+            "id": "9d0ae6cf-bd27-418d-b648-5a1023e381b4",
+            "name": "Persisted A",
+            "url": "http://127.0.0.1:4321/ok",
+            "interval": 60,
+            "enabled": true,
+            "createdAt": "2026-08-28T00:00:00.765895Z",
+            "updatedAt": "2026-08-28T00:00:00.765895Z"
+          },
+          "latestCheck": null
+        }
+      },
+      "elapsedSeconds": 0.027
+    },
+    {
+      "method": "POST",
+      "path": "/api/monitors",
+      "status": 201,
+      "wire": {
+        "data": {
+          "monitor": {
+            "id": "1437ccbc-5e45-4941-aff5-ee3e78c92368",
+            "name": "Persisted B",
+            "url": "http://127.0.0.1:4321/fail",
+            "interval": 120,
+            "enabled": true,
+            "createdAt": "2026-08-28T00:00:00.774661Z",
+            "updatedAt": "2026-08-28T00:00:00.774661Z"
+          },
+          "latestCheck": null
+        }
+      },
+      "elapsedSeconds": 0.003
+    },
+    {
+      "method": "POST",
+      "path": "/api/monitors/9d0ae6cf-bd27-418d-b648-5a1023e381b4/checks",
+      "status": 200,
+      "wire": {
+        "data": {
+          "id": "0f777fc2-6c14-47f3-a41c-9240938a7cd4",
+          "monitorId": "9d0ae6cf-bd27-418d-b648-5a1023e381b4",
+          "trigger": "MANUAL",
+          "state": "SUCCEEDED",
+          "httpStatus": 200,
+          "latencyMs": 13,
+          "failureReason": null,
+          "startedAt": "2026-08-28T00:00:00.779678Z",
+          "finishedAt": "2026-08-28T00:00:00.792939Z"
+        }
+      },
+      "elapsedSeconds": 0.02
+    },
+    {
+      "method": "POST",
+      "path": "/api/monitors/9d0ae6cf-bd27-418d-b648-5a1023e381b4/checks",
+      "status": 200,
+      "wire": {
+        "data": {
+          "id": "57667525-d7e3-49c6-b2dd-8ece64313a41",
+          "monitorId": "9d0ae6cf-bd27-418d-b648-5a1023e381b4",
+          "trigger": "MANUAL",
+          "state": "SUCCEEDED",
+          "httpStatus": 200,
+          "latencyMs": 0,
+          "failureReason": null,
+          "startedAt": "2026-08-28T00:00:00.798091Z",
+          "finishedAt": "2026-08-28T00:00:00.799060Z"
+        }
+      },
+      "elapsedSeconds": 0.005
+    },
+    {
+      "method": "POST",
+      "path": "/api/monitors/1437ccbc-5e45-4941-aff5-ee3e78c92368/checks",
+      "status": 200,
+      "wire": {
+        "data": {
+          "id": "5364cfc5-ab28-48e4-93e3-af3e34ed03a8",
+          "monitorId": "1437ccbc-5e45-4941-aff5-ee3e78c92368",
+          "trigger": "MANUAL",
+          "state": "FAILED",
+          "httpStatus": 503,
+          "latencyMs": 0,
+          "failureReason": "HTTP_STATUS",
+          "startedAt": "2026-08-28T00:00:00.802802Z",
+          "finishedAt": "2026-08-28T00:00:00.803800Z"
+        }
+      },
+      "elapsedSeconds": 0.004
+    },
+    {
+      "method": "GET",
+      "path": "/api/monitors",
+      "status": 200,
+      "wire": {
+        "data": []
+      },
+      "elapsedSeconds": 0.003
+    }
+  ],
+  "monitors": [
+    {
+      "id": "9d0ae6cf-bd27-418d-b648-5a1023e381b4",
+      "name": "Persisted A",
+      "url": "http://127.0.0.1:4321/ok",
+      "interval": 60,
+      "enabled": true,
+      "createdAt": "2026-08-28T00:00:00.765895Z",
+      "updatedAt": "2026-08-28T00:00:00.765895Z"
+    },
+    {
+      "id": "1437ccbc-5e45-4941-aff5-ee3e78c92368",
+      "name": "Persisted B",
+      "url": "http://127.0.0.1:4321/fail",
+      "interval": 120,
+      "enabled": true,
+      "createdAt": "2026-08-28T00:00:00.774661Z",
+      "updatedAt": "2026-08-28T00:00:00.774661Z"
+    }
+  ],
+  "checks": [
+    {
+      "id": "0f777fc2-6c14-47f3-a41c-9240938a7cd4",
+      "monitorId": "9d0ae6cf-bd27-418d-b648-5a1023e381b4",
+      "trigger": "MANUAL",
+      "state": "SUCCEEDED",
+      "httpStatus": 200,
+      "latencyMs": 13,
+      "failureReason": null,
+      "startedAt": "2026-08-28T00:00:00.779678Z",
+      "finishedAt": "2026-08-28T00:00:00.792939Z"
+    },
+    {
+      "id": "57667525-d7e3-49c6-b2dd-8ece64313a41",
+      "monitorId": "9d0ae6cf-bd27-418d-b648-5a1023e381b4",
+      "trigger": "MANUAL",
+      "state": "SUCCEEDED",
+      "httpStatus": 200,
+      "latencyMs": 0,
+      "failureReason": null,
+      "startedAt": "2026-08-28T00:00:00.798091Z",
+      "finishedAt": "2026-08-28T00:00:00.799060Z"
+    },
+    {
+      "id": "5364cfc5-ab28-48e4-93e3-af3e34ed03a8",
+      "monitorId": "1437ccbc-5e45-4941-aff5-ee3e78c92368",
+      "trigger": "MANUAL",
+      "state": "FAILED",
+      "httpStatus": 503,
+      "latencyMs": 0,
+      "failureReason": "HTTP_STATUS",
+      "startedAt": "2026-08-28T00:00:00.802802Z",
+      "finishedAt": "2026-08-28T00:00:00.803800Z"
+    }
+  ],
+  "result": "FAIL: Fresh API must retain both Monitors\n\n0 !== 2\n",
+  "elapsedSeconds": 3.016
+}
diff --git a/evidence/E03/fixtures.md b/evidence/E03/fixtures.md
new file mode 100644
index 0000000..fc395b2
--- /dev/null
+++ b/evidence/E03/fixtures.md
@@ -0,0 +1,66 @@
+# E03 frozen verification fixtures
+
+Frozen before the first E03 scenario execution on branch `track/industry-spring`,
+START `3aef68e3b873efea811a5f29f4009450ce74500e`,
+spec revision `0a006589477f8ae47bad3faa5510c999cff85ee4`.
+
+## Unchanged counterexample and repaired verification
+
+Run the same `scripts/persistence-scenario.mjs` request sequence once against the
+unchanged packaged API, then once against the repaired package. The argument only
+labels the evidence; it does not change the requests or assertions.
+
+- A = `{ "name": "Persisted A", "url": "http://127.0.0.1:4321/ok", "interval": 60, "enabled": true }`.
+- B = `{ "name": "Persisted B", "url": "http://127.0.0.1:4321/fail", "interval": 120, "enabled": true }`.
+- Create A, B; synchronously check A, A, B. Expect SUCCEEDED/200/null twice and
+  FAILED/503/HTTP_STATUS once. Capture both complete Monitor values and all three
+  complete CheckRun values, including generated IDs, timestamps, and nulls.
+- Stop the API and start a fresh independent process with identical configuration.
+  Expect the same two Monitor IDs and all three exact historical CheckRun values.
+  Stop the baseline immediately on the decisive missing-data assertion; do not
+  repeat it, alter the implementation, or weaken the expected result.
+- After successful persistence verification, update A to `Updated A`, interval 90,
+  then pause and reactivate it using full replacements. Authoritative GETs must
+  reflect each mutation and retain createdAt and previous history.
+- Delete B. Its Monitor, history collection, and individual CheckRun return
+  404/NOT_FOUND; the database has no remaining child records for B.
+
+## Supplementary cases (also frozen before execution)
+
+- Real PostgreSQL only; compose project `wse-industry`, loopback port 15432. Local
+  disposable test authentication is explicitly nonsecret trust authentication,
+  with no production application containers or externally exposed database.
+- Isolated schemas: `e03_restart`, `e03_functional`, `e03_browser`,
+  `e03_migrations`, `e03_mapping`, `e03_missing_column`, `e03_extra_required`.
+  Test setup recreates only these schemas; teardown removes the schemas it owns.
+- Fresh migration chain V1 (Monitor) then V2 (CheckRun), followed by an unchanged
+  second migration invocation: one successful row per migration, no duplicate
+  DDL/data effects. Also upgrade an independently created V1 schema to V2.
+- Canonical mapping: Monitor UUID `00000000-0000-4000-8000-000000000031`, name
+  `Mapping fixture`, URL `/ok` on the configured fixture, interval 1, enabled false;
+  timestamp `2026-08-28T00:00:00.123456789Z` is canonically stored at microsecond
+  precision `2026-08-28T00:00:00.123456Z`. A SUCCEEDED/200 CheckRun has latency 0
+  and null failureReason. A FAILED/TIMEOUT CheckRun has null httpStatus and latency
+  2000. Preserve integer/boolean/zero/null meanings through database and JSON.
+- Transaction boundary: a public Store mutation called through the Spring bean
+  proxy joins a transaction; an outer transaction that rolls back cannot leave a
+  Monitor. Generated SQL must use the actual Monitor/CheckRun tables, and the
+  synchronous outbound HTTP request must execute outside a database transaction.
+- Incompatible migrated schema 1: remove mapped `monitors.name`; application
+  startup fails, rather than serving an unsafe API.
+- Incompatible migrated schema 2 (root supplement): add
+  `monitors.unmapped_required text NOT NULL` with no default; application startup
+  fails before normal inserts can reach an unsafe API.
+- Root input supplement: exact JSON name `"A\u0000B"`, URL
+  `http://127.0.0.1:4321/ok`, interval 60, enabled true must return
+  400/INVALID_INPUT with no Monitor created. The same invalid replacement must not
+  change an existing Monitor.
+- Browser: create `Lifecycle fixture` at `/ok`, interval 60/enabled; run twice;
+  read two history rows; edit to `Updated lifecycle fixture`, interval 90; pause,
+  reload, reactivate, reload, delete, reload. Persisted state remains authoritative.
+- Existing E01/E02 fixed Java and browser fixtures, including raw interval `1e309`,
+  numeric `60.0`, string `"60"`, timeout/null, and fixture-only destinations remain
+  unchanged. Use the standard verification runner with no retries.
+
+No load, benchmark, sweep, automatic scenario retry, or profiling. Safety timeouts
+only bound process startup and cleanup; they are not performance assertions.
diff --git a/evidence/E03/runs.jsonl b/evidence/E03/runs.jsonl
new file mode 100644
index 0000000..f9f0268
--- /dev/null
+++ b/evidence/E03/runs.jsonl
@@ -0,0 +1,2 @@
+{"command":"/usr/bin/time -p mvn -B -ntp -f backend/pom.xml -DskipTests package","phase":"unchanged baseline package","elapsedSeconds":2.12,"exitCode":0}
+{"command":"fnm exec --using 24.19.0 node scripts/persistence-scenario.mjs baseline","startedAt":"2026-08-27T23:59:59.179Z","elapsedSeconds":3.016,"exitCode":1,"result":"Expected decisive failure: fresh API returned 0 Monitors instead of 2; scenario stopped and owned processes cleaned up"}
diff --git a/evidence/E03/verification.md b/evidence/E03/verification.md
new file mode 100644
index 0000000..f06123c
--- /dev/null
+++ b/evidence/E03/verification.md
@@ -0,0 +1,24 @@
+# E03 actual verification
+
+Branch `track/industry-spring`, spec `0a006589477f8ae47bad3faa5510c999cff85ee4`.
+Attempt 1 starts at `3aef68e3b873efea811a5f29f4009450ce74500e`.
+
+## Unchanged baseline
+
+The frozen fixture was written before execution. No existing production or test
+source was changed before the baseline.
+
+| Invocation | Actual result | Seconds |
+| --- | --- | ---: |
+| `mvn -B -ntp -f backend/pom.xml -DskipTests package` under `/usr/bin/time -p` | exit 0, unchanged package built | 2.12 |
+| `fnm exec --using 24.19.0 node scripts/persistence-scenario.mjs baseline` | exit 1, expected decisive persistence failure | 3.016 |
+
+A and B were created and A,A,B produced SUCCEEDED/200/null twice and
+FAILED/503/HTTP_STATUS once. The independently restarted API returned an empty
+Monitor list (0 instead of 2). The scenario stopped at that assertion and cleaned
+up both API processes and its fixture. It was not repeated. Full captured wire
+values, including all generated IDs/timestamps, are in `baseline-persistence.json`.
+
+The baseline uses the existing pinned Java 21.0.7 and Maven 3.9.11, with Node
+24.19.0 selected via fnm. No permission, build, or fixture failure occurred in
+these two invocations. No load, retries, sweep, or profiling was performed.
diff --git a/scripts/persistence-scenario.mjs b/scripts/persistence-scenario.mjs
new file mode 100644
index 0000000..97ca700
--- /dev/null
+++ b/scripts/persistence-scenario.mjs
@@ -0,0 +1,136 @@
+import assert from 'node:assert/strict';
+import { spawn } from 'node:child_process';
+import { once } from 'node:events';
+import { appendFileSync, mkdirSync, openSync, closeSync, writeFileSync } from 'node:fs';
+import { setTimeout as delay } from 'node:timers/promises';
+
+// The label changes evidence filenames only. The baseline and repair use identical inputs.
+const label = process.argv[2];
+assert.ok(['baseline', 'fixed'].includes(label), 'Use baseline or fixed as the evidence label');
+const directory = 'output/e03';
+mkdirSync(directory, { recursive: true });
+const started = Date.now();
+const evidence = { label, requests: [], monitors: [], checks: [] };
+const processes = [];
+const base = 'http://127.0.0.1:4322';
+
+function start(command, args, logName) {
+  const log = openSync(`${directory}/${label}-${logName}.log`, 'w');
+  const child = spawn(command, args, {
+    env: { ...process.env, DB_SCHEMA: 'e03_restart' }, stdio: ['ignore', log, log],
+  });
+  closeSync(log);
+  processes.push(child);
+  return child;
+}
+
+async function ready(url, child) {
+  const deadline = Date.now() + 30_000;
+  while (Date.now() < deadline) {
+    if (child.exitCode !== null) throw new Error(`Owned process exited ${child.exitCode} before ready`);
+    try {
+      if ((await fetch(url, { signal: AbortSignal.timeout(1000) })).ok) return;
+    } catch { /* Bounded readiness polling, never a scenario retry. */ }
+    await delay(100);
+  }
+  throw new Error(`Owned process did not become ready within 30 seconds: ${url}`);
+}
+
+async function stop(child) {
+  if (!child || child.exitCode !== null || child.signalCode !== null) return;
+  const exited = once(child, 'exit');
+  child.kill('SIGTERM');
+  const force = setTimeout(() => child.kill('SIGKILL'), 5000);
+  try { await exited; } finally { clearTimeout(force); }
+}
+
+async function request(path, method = 'GET', body, status = 200) {
+  const requestStarted = Date.now();
+  const response = await fetch(`${base}${path}`, {
+    method, headers: body === undefined ? {} : { 'Content-Type': 'application/json' },
+    body: body === undefined ? undefined : JSON.stringify(body), signal: AbortSignal.timeout(10_000),
+  });
+  const wire = await response.json();
+  evidence.requests.push({ method, path, status: response.status, wire,
+    elapsedSeconds: (Date.now() - requestStarted) / 1000 });
+  assert.equal(response.status, status);
+  assert.deepEqual(Object.keys(wire), [status === 200 || status === 201 ? 'data' : 'error']);
+  if (status === 404) assert.equal(wire.error.code, 'NOT_FOUND');
+  if (status === 400) assert.equal(wire.error.code, 'INVALID_INPUT');
+  return wire.data;
+}
+
+let api;
+let exitCode = 0;
+try {
+  const fixture = start(process.execPath, ['scripts/fixture.mjs'], 'fixture');
+  await ready('http://127.0.0.1:4321/ok', fixture);
+  api = start('java', ['-jar', 'backend/target/monitor-api-0.0.1.jar'], 'api-first');
+  await ready(`${base}/api/monitors`, api);
+  assert.deepEqual(await request('/api/monitors'), [], 'Scenario requires an isolated empty store');
+  const a = (await request('/api/monitors', 'POST', {
+    name: 'Persisted A', url: 'http://127.0.0.1:4321/ok', interval: 60, enabled: true,
+  }, 201)).monitor;
+  const b = (await request('/api/monitors', 'POST', {
+    name: 'Persisted B', url: 'http://127.0.0.1:4321/fail', interval: 120, enabled: true,
+  }, 201)).monitor;
+  evidence.monitors = [a, b];
+  for (const [monitor, state, httpStatus, failureReason] of [
+    [a, 'SUCCEEDED', 200, null], [a, 'SUCCEEDED', 200, null], [b, 'FAILED', 503, 'HTTP_STATUS'],
+  ]) {
+    const check = await request(`/api/monitors/${monitor.id}/checks`, 'POST');
+    assert.deepEqual([check.state, check.httpStatus, check.failureReason], [state, httpStatus, failureReason]);
+    evidence.checks.push(check);
+  }
+  await stop(api);
+  api = start('java', ['-jar', 'backend/target/monitor-api-0.0.1.jar'], 'api-fresh');
+  await ready(`${base}/api/monitors`, api);
+  const restored = await request('/api/monitors');
+  assert.equal(restored.length, 2, 'Fresh API must retain both Monitors');
+  for (const monitor of [a, b]) {
+    assert.deepEqual(restored.find(row => row.monitor.id === monitor.id)?.monitor, monitor);
+    const expected = evidence.checks.filter(check => check.monitorId === monitor.id);
+    const history = await request(`/api/monitors/${monitor.id}/checks`);
+    assert.equal(history.length, expected.length);
+    for (const check of expected) {
+      assert.deepEqual(history.find(row => row.id === check.id), check);
+      assert.deepEqual(await request(`/api/monitors/${monitor.id}/checks/${check.id}`), check);
+    }
+    assert.deepEqual((await request(`/api/monitors/${monitor.id}`)).latestCheck, expected.at(-1));
+  }
+  const nul = { name: 'A\u0000B', url: a.url, interval: 60, enabled: true };
+  await request('/api/monitors', 'POST', nul, 400);
+  await request(`/api/monitors/${a.id}`, 'PUT', nul, 400);
+  assert.deepEqual((await request(`/api/monitors/${a.id}`)).monitor, a);
+  assert.equal((await request('/api/monitors')).length, 2);
+  for (const enabled of [true, false, true]) {
+    const update = { name: 'Updated A', url: a.url, interval: 90, enabled };
+    const changed = await request(`/api/monitors/${a.id}`, 'PUT', update);
+    const loaded = await request(`/api/monitors/${a.id}`);
+    assert.deepEqual(loaded, changed);
+    assert.deepEqual({ name: loaded.monitor.name, url: loaded.monitor.url,
+      interval: loaded.monitor.interval, enabled: loaded.monitor.enabled }, update);
+    assert.equal(loaded.monitor.createdAt, a.createdAt);
+  }
+  assert.equal(await request(`/api/monitors/${b.id}`, 'DELETE'), null);
+  await request(`/api/monitors/${b.id}`, 'GET', undefined, 404);
+  await request(`/api/monitors/${b.id}/checks`, 'GET', undefined, 404);
+  await request(`/api/monitors/${b.id}/checks/${evidence.checks[2].id}`, 'GET', undefined, 404);
+  assert.equal((await request('/api/monitors')).length, 1);
+  assert.equal((await request(`/api/monitors/${a.id}/checks`)).length, 2);
+  evidence.result = 'PASS: all historical records survive a new API process; lifecycle and NUL boundary hold';
+  console.log(evidence.result);
+} catch (error) {
+  exitCode = 1;
+  evidence.result = `FAIL: ${error.message}`;
+  console.error(evidence.result);
+} finally {
+  for (const child of processes.toReversed()) await stop(child);
+  evidence.elapsedSeconds = (Date.now() - started) / 1000;
+  writeFileSync(`${directory}/${label}-persistence.json`, `${JSON.stringify(evidence, null, 2)}\n`);
+  appendFileSync(`${directory}/runs.jsonl`, `${JSON.stringify({
+    command: `node scripts/persistence-scenario.mjs ${label}`, startedAt: new Date(started).toISOString(),
+    elapsedSeconds: evidence.elapsedSeconds, exitCode, result: evidence.result,
+  })}\n`);
+  process.exitCode = exitCode;
+}


