## `Make PostgreSQL authoritative for Monitor lifecycle`

diff --git a/evidence/E03/mapping.json b/evidence/E03/mapping.json
new file mode 100644
index 0000000..29b72c9
--- /dev/null
+++ b/evidence/E03/mapping.json
@@ -0,0 +1,25 @@
+{
+  "monitor": {
+    "id": "00000000-0000-4000-8000-000000000101",
+    "name": "Canonical 😀",
+    "url": "http://127.0.0.1:4311/ok",
+    "interval": 1,
+    "enabled": false,
+    "createdAt": "2026-01-01T03:34:56.789Z",
+    "updatedAt": "2026-01-01T03:34:56.789Z"
+  },
+  "check": {
+    "id": "00000000-0000-4000-8000-000000000102",
+    "monitorId": "00000000-0000-4000-8000-000000000101",
+    "trigger": "MANUAL",
+    "state": "FAILED",
+    "httpStatus": null,
+    "latencyMs": 0,
+    "failureReason": "CONNECTION_FAILURE",
+    "startedAt": "2026-01-01T03:34:56.789Z",
+    "finishedAt": "2026-01-01T03:34:56.789Z"
+  },
+  "sqlTimestampInput": "2026-01-01T12:34:56.789+09:00",
+  "sqlTimestampRuntimeType": "Date",
+  "durationMs": 107
+}
diff --git a/evidence/E03/migrations.json b/evidence/E03/migrations.json
new file mode 100644
index 0000000..60e0a67
--- /dev/null
+++ b/evidence/E03/migrations.json
@@ -0,0 +1,27 @@
+{
+  "command": "node server/migrate.ts (twice)",
+  "schema": "e03_migrations",
+  "first": {
+    "applied": [
+      "001_monitors.sql",
+      "002_check_runs.sql"
+    ]
+  },
+  "second": {
+    "applied": []
+  },
+  "history": [
+    {
+      "version": "001_monitors.sql",
+      "checksum": "e695bcf8abe711f63b8eee31f9ca98bb50eb90921349156bffe183bfddacc86e",
+      "applied_at": "2026-08-27T23:49:26.847Z"
+    },
+    {
+      "version": "002_check_runs.sql",
+      "checksum": "13878c39225e292140a81312f98c7db76da3a3c49aa105218b1fc1193ee8f389",
+      "applied_at": "2026-08-27T23:49:26.847Z"
+    }
+  ],
+  "startupVerified": true,
+  "durationMs": 1055
+}
diff --git a/evidence/E03/persistence.json b/evidence/E03/persistence.json
new file mode 100644
index 0000000..b77dab4
--- /dev/null
+++ b/evidence/E03/persistence.json
@@ -0,0 +1,218 @@
+{
+  "databaseSchema": "e03_persistence",
+  "beforeRestart": [
+    {
+      "id": "05f1c0db-8e99-4859-9bc4-8f675ff80d85",
+      "name": "Persisted A",
+      "url": "http://127.0.0.1:4311/ok",
+      "interval": 60,
+      "enabled": true,
+      "createdAt": "2026-08-27T23:49:27.980Z",
+      "updatedAt": "2026-08-27T23:49:27.980Z",
+      "latestCheck": {
+        "id": "d8bdd46d-ee5b-4c9e-bd7a-fcd6ecc02a84",
+        "monitorId": "05f1c0db-8e99-4859-9bc4-8f675ff80d85",
+        "trigger": "MANUAL",
+        "state": "SUCCEEDED",
+        "httpStatus": 200,
+        "latencyMs": 1,
+        "failureReason": null,
+        "startedAt": "2026-08-27T23:49:28.037Z",
+        "finishedAt": "2026-08-27T23:49:28.038Z"
+      }
+    },
+    {
+      "id": "cbfc7491-d3a8-4bf8-9bc9-5ef98f04b5be",
+      "name": "Persisted B",
+      "url": "http://127.0.0.1:4311/fail",
+      "interval": 120,
+      "enabled": true,
+      "createdAt": "2026-08-27T23:49:28.004Z",
+      "updatedAt": "2026-08-27T23:49:28.004Z",
+      "latestCheck": {
+        "id": "91e79699-cfcf-4e89-a6e8-edf0b53b76a8",
+        "monitorId": "cbfc7491-d3a8-4bf8-9bc9-5ef98f04b5be",
+        "trigger": "MANUAL",
+        "state": "FAILED",
+        "httpStatus": 503,
+        "latencyMs": 5,
+        "failureReason": "HTTP_STATUS",
+        "startedAt": "2026-08-27T23:49:28.061Z",
+        "finishedAt": "2026-08-27T23:49:28.067Z"
+      }
+    }
+  ],
+  "afterRestart": [
+    {
+      "id": "05f1c0db-8e99-4859-9bc4-8f675ff80d85",
+      "name": "Persisted A",
+      "url": "http://127.0.0.1:4311/ok",
+      "interval": 60,
+      "enabled": true,
+      "createdAt": "2026-08-27T23:49:27.980Z",
+      "updatedAt": "2026-08-27T23:49:27.980Z",
+      "latestCheck": {
+        "id": "d8bdd46d-ee5b-4c9e-bd7a-fcd6ecc02a84",
+        "monitorId": "05f1c0db-8e99-4859-9bc4-8f675ff80d85",
+        "trigger": "MANUAL",
+        "state": "SUCCEEDED",
+        "httpStatus": 200,
+        "latencyMs": 1,
+        "failureReason": null,
+        "startedAt": "2026-08-27T23:49:28.037Z",
+        "finishedAt": "2026-08-27T23:49:28.038Z"
+      }
+    },
+    {
+      "id": "cbfc7491-d3a8-4bf8-9bc9-5ef98f04b5be",
+      "name": "Persisted B",
+      "url": "http://127.0.0.1:4311/fail",
+      "interval": 120,
+      "enabled": true,
+      "createdAt": "2026-08-27T23:49:28.004Z",
+      "updatedAt": "2026-08-27T23:49:28.004Z",
+      "latestCheck": {
+        "id": "91e79699-cfcf-4e89-a6e8-edf0b53b76a8",
+        "monitorId": "cbfc7491-d3a8-4bf8-9bc9-5ef98f04b5be",
+        "trigger": "MANUAL",
+        "state": "FAILED",
+        "httpStatus": 503,
+        "latencyMs": 5,
+        "failureReason": "HTTP_STATUS",
+        "startedAt": "2026-08-27T23:49:28.061Z",
+        "finishedAt": "2026-08-27T23:49:28.067Z"
+      }
+    }
+  ],
+  "historicalChecksAfterRestart": [
+    {
+      "id": "d8bdd46d-ee5b-4c9e-bd7a-fcd6ecc02a84",
+      "monitorId": "05f1c0db-8e99-4859-9bc4-8f675ff80d85",
+      "trigger": "MANUAL",
+      "state": "SUCCEEDED",
+      "httpStatus": 200,
+      "latencyMs": 1,
+      "failureReason": null,
+      "startedAt": "2026-08-27T23:49:28.037Z",
+      "finishedAt": "2026-08-27T23:49:28.038Z"
+    },
+    {
+      "id": "7df3e37d-c398-481f-b292-c12daaef28a8",
+      "monitorId": "05f1c0db-8e99-4859-9bc4-8f675ff80d85",
+      "trigger": "MANUAL",
+      "state": "SUCCEEDED",
+      "httpStatus": 200,
+      "latencyMs": 7,
+      "failureReason": null,
+      "startedAt": "2026-08-27T23:49:28.012Z",
+      "finishedAt": "2026-08-27T23:49:28.019Z"
+    },
+    {
+      "id": "91e79699-cfcf-4e89-a6e8-edf0b53b76a8",
+      "monitorId": "cbfc7491-d3a8-4bf8-9bc9-5ef98f04b5be",
+      "trigger": "MANUAL",
+      "state": "FAILED",
+      "httpStatus": 503,
+      "latencyMs": 5,
+      "failureReason": "HTTP_STATUS",
+      "startedAt": "2026-08-27T23:49:28.061Z",
+      "finishedAt": "2026-08-27T23:49:28.067Z"
+    }
+  ],
+  "lifecycle": [
+    {
+      "id": "05f1c0db-8e99-4859-9bc4-8f675ff80d85",
+      "name": "Updated A",
+      "url": "http://127.0.0.1:4311/ok",
+      "interval": 90,
+      "enabled": true,
+      "createdAt": "2026-08-27T23:49:27.980Z",
+      "updatedAt": "2026-08-27T23:49:28.207Z",
+      "latestCheck": {
+        "id": "d8bdd46d-ee5b-4c9e-bd7a-fcd6ecc02a84",
+        "monitorId": "05f1c0db-8e99-4859-9bc4-8f675ff80d85",
+        "trigger": "MANUAL",
+        "state": "SUCCEEDED",
+        "httpStatus": 200,
+        "latencyMs": 1,
+        "failureReason": null,
+        "startedAt": "2026-08-27T23:49:28.037Z",
+        "finishedAt": "2026-08-27T23:49:28.038Z"
+      }
+    },
+    {
+      "id": "05f1c0db-8e99-4859-9bc4-8f675ff80d85",
+      "name": "Updated A",
+      "url": "http://127.0.0.1:4311/ok",
+      "interval": 90,
+      "enabled": false,
+      "createdAt": "2026-08-27T23:49:27.980Z",
+      "updatedAt": "2026-08-27T23:49:28.217Z",
+      "latestCheck": {
+        "id": "d8bdd46d-ee5b-4c9e-bd7a-fcd6ecc02a84",
+        "monitorId": "05f1c0db-8e99-4859-9bc4-8f675ff80d85",
+        "trigger": "MANUAL",
+        "state": "SUCCEEDED",
+        "httpStatus": 200,
+        "latencyMs": 1,
+        "failureReason": null,
+        "startedAt": "2026-08-27T23:49:28.037Z",
+        "finishedAt": "2026-08-27T23:49:28.038Z"
+      }
+    },
+    {
+      "id": "05f1c0db-8e99-4859-9bc4-8f675ff80d85",
+      "name": "Updated A",
+      "url": "http://127.0.0.1:4311/ok",
+      "interval": 90,
+      "enabled": true,
+      "createdAt": "2026-08-27T23:49:27.980Z",
+      "updatedAt": "2026-08-27T23:49:28.252Z",
+      "latestCheck": {
+        "id": "d8bdd46d-ee5b-4c9e-bd7a-fcd6ecc02a84",
+        "monitorId": "05f1c0db-8e99-4859-9bc4-8f675ff80d85",
+        "trigger": "MANUAL",
+        "state": "SUCCEEDED",
+        "httpStatus": 200,
+        "latencyMs": 1,
+        "failureReason": null,
+        "startedAt": "2026-08-27T23:49:28.037Z",
+        "finishedAt": "2026-08-27T23:49:28.038Z"
+      }
+    }
+  ],
+  "deletedMonitor": "cbfc7491-d3a8-4bf8-9bc9-5ef98f04b5be",
+  "deletedCheckRun": "91e79699-cfcf-4e89-a6e8-edf0b53b76a8",
+  "afterDeletion": [
+    {
+      "id": "05f1c0db-8e99-4859-9bc4-8f675ff80d85",
+      "name": "Updated A",
+      "url": "http://127.0.0.1:4311/ok",
+      "interval": 90,
+      "enabled": true,
+      "createdAt": "2026-08-27T23:49:27.980Z",
+      "updatedAt": "2026-08-27T23:49:28.252Z",
+      "latestCheck": {
+        "id": "d8bdd46d-ee5b-4c9e-bd7a-fcd6ecc02a84",
+        "monitorId": "05f1c0db-8e99-4859-9bc4-8f675ff80d85",
+        "trigger": "MANUAL",
+        "state": "SUCCEEDED",
+        "httpStatus": 200,
+        "latencyMs": 1,
+        "failureReason": null,
+        "startedAt": "2026-08-27T23:49:28.037Z",
+        "finishedAt": "2026-08-27T23:49:28.038Z"
+      }
+    }
+  ],
+  "counts": {
+    "monitors": 1,
+    "checks": 2,
+    "orphans": 0
+  },
+  "fixtureCalls": {
+    "/ok": 2,
+    "/fail": 1
+  },
+  "durationMs": 600
+}
diff --git a/evidence/E03/schema-mismatch.json b/evidence/E03/schema-mismatch.json
new file mode 100644
index 0000000..6bba18d
--- /dev/null
+++ b/evidence/E03/schema-mismatch.json
@@ -0,0 +1,8 @@
+{
+  "schema": "e03_incompatible",
+  "removedColumn": "monitors.interval_seconds",
+  "startupRejected": true,
+  "listening": false,
+  "message": "Incompatible database schema: monitors.interval_seconds.",
+  "durationMs": 113
+}
diff --git a/evidence/E03/supplemental-baseline.json b/evidence/E03/supplemental-baseline.json
new file mode 100644
index 0000000..b6cd4f4
--- /dev/null
+++ b/evidence/E03/supplemental-baseline.json
@@ -0,0 +1,28 @@
+{
+  "extraRequiredColumn": {
+    "expectedStartupRejected": true,
+    "actualStartupRejected": false,
+    "listening": true,
+    "startupError": null,
+    "validInsertStatus": 500,
+    "validInsertBody": {
+      "error": {
+        "code": "INTERNAL_ERROR",
+        "message": "The monitoring service could not complete the request."
+      }
+    }
+  },
+  "nulName": {
+    "expectedStatus": 400,
+    "actualStatus": 500,
+    "actualBody": {
+      "error": {
+        "code": "INTERNAL_ERROR",
+        "message": "The monitoring service could not complete the request."
+      }
+    },
+    "createdCount": 0
+  },
+  "result": "REPRODUCED both independent PostgreSQL boundaries; stopped",
+  "durationMs": 379
+}
diff --git a/evidence/E03/supplemental-reproducer.mjs b/evidence/E03/supplemental-reproducer.mjs
new file mode 100644
index 0000000..0e30f55
--- /dev/null
+++ b/evidence/E03/supplemental-reproducer.mjs
@@ -0,0 +1,36 @@
+import assert from 'node:assert/strict';
+import { writeFile } from 'node:fs/promises';
+import { buildApp } from './server/app.ts';
+import { databasePool } from './server/database.ts';
+import { resetTestSchema, dropTestSchema } from './test/database.ts';
+import scenario from './evidence/E03/supplemental-scenario.json' with { type: 'json' };
+const start = performance.now();
+const evidence = {};
+const extra = scenario.extraRequiredColumn;
+let config = await resetTestSchema(extra.schema);
+let pool = databasePool(config);
+let app = buildApp(undefined, config);
+try {
+ await pool.query(extra.sql);
+ let startupError = null;
+ try { await app.listen({host:'127.0.0.1',port:4312}); }
+ catch (error) { startupError = error.message; }
+ const response = app.server.listening ? await app.inject({method:'POST',url:'/monitors',payload:extra.probeInput}) : null;
+ evidence.extraRequiredColumn = {expectedStartupRejected:true,actualStartupRejected:startupError!==null,listening:app.server.listening,startupError,validInsertStatus:response?.statusCode??null,validInsertBody:response?.json()??null};
+ assert.equal(startupError,null);
+ assert.equal(response.statusCode,500);
+} finally { await app.close(); await pool.end(); await dropTestSchema(config.schema); }
+config = await resetTestSchema(scenario.nulName.schema);
+pool = databasePool(config);
+app = buildApp(undefined, config);
+try {
+ const response = await app.inject({method:'POST',url:'/monitors',payload:scenario.nulName.input});
+ const createdCount = (await pool.query('SELECT count(*)::int AS count FROM monitors')).rows[0].count;
+ evidence.nulName = {expectedStatus:400,actualStatus:response.statusCode,actualBody:response.json(),createdCount};
+ assert.equal(response.statusCode,500);
+ assert.equal(createdCount,0);
+} finally { await app.close(); await pool.end(); await dropTestSchema(config.schema); }
+evidence.result='REPRODUCED both independent PostgreSQL boundaries; stopped';
+evidence.durationMs=Math.round(performance.now()-start);
+await writeFile('evidence/E03/supplemental-baseline.json',JSON.stringify(evidence,null,2)+'\n');
+console.log(JSON.stringify(evidence,null,2));
diff --git a/evidence/E03/supplemental-scenario.json b/evidence/E03/supplemental-scenario.json
new file mode 100644
index 0000000..9957440
--- /dev/null
+++ b/evidence/E03/supplemental-scenario.json
@@ -0,0 +1,23 @@
+{
+  "thread": "E03",
+  "attempt": 1,
+  "source": "Root read-only source audit during initial implementation, after primary gates",
+  "frozenBeforeFirstSupplementalExecution": true,
+  "originalScenarioUnchanged": true,
+  "extraRequiredColumn": {
+    "schema": "e03_extra_required",
+    "sql": "ALTER TABLE monitors ADD COLUMN e03_required_extra text NOT NULL",
+    "expected": "Application startup rejects before listening",
+    "probeInput": { "name": "Persisted A", "url": "http://127.0.0.1:4311/ok", "interval": 60, "enabled": true }
+  },
+  "nulName": {
+    "schema": "e03_nul",
+    "input": { "name": "A\u0000B", "url": "http://127.0.0.1:4311/ok", "interval": 60, "enabled": true },
+    "expectedStatus": 400,
+    "expectedCode": "INVALID_INPUT",
+    "expectedCreatedCount": 0
+  },
+  "retries": 0,
+  "loadRuns": 0,
+  "stop": "One baseline per independent case; fix only confirmed E03 boundaries and run deterministic regressions."
+}
diff --git a/package.json b/package.json
index 509ae19..461a2c4 100644
--- a/package.json
+++ b/package.json
@@ -19,7 +19,8 @@
     "test": "npm run test:unit && npm run test:functional",
     "db:up": "docker compose --project-name wse-fundamentals up -d --wait",
     "db:down": "docker compose --project-name wse-fundamentals down",
-    "db:migrate": "node server/migrate.ts"
+    "db:migrate": "node server/migrate.ts",
+    "test:integration": "node --test --test-concurrency=1 test/persistence.test.ts"
   },
   "dependencies": {
     "fastify": "5.12.1",
diff --git a/server/app.ts b/server/app.ts
index b802586..7f23a17 100644
--- a/server/app.ts
+++ b/server/app.ts
@@ -3,7 +3,12 @@ import Fastify, { errorCodes } from 'fastify';
 import type { FastifyReply, FastifyRequest } from 'fastify';
 import { checkMonitor, DEFAULT_FIXTURE_ORIGIN } from './check.ts';
 import { ApiError, ERROR_STATUS, monitorId, monitorInput } from './contracts.ts';
-import type { ApiFailure, CheckRun, Monitor } from './model.ts';
+import type { ApiFailure, Monitor } from './model.ts';
+import { databaseConfig, databasePool } from './database.ts';
+import type { DatabaseConfig } from './database.ts';
+import { verifySchema } from './schema.ts';
+import { checkRunFromRow, checkRunToValues, monitorFromRow, monitorToValues, monitorViewFromRow } from './mapping.ts';
+import type { CheckRunRow, MonitorRow, MonitorViewRow } from './mapping.ts';
 
 const inputErrors = [
   errorCodes.FST_ERR_CTP_INVALID_JSON_BODY,
@@ -14,7 +19,14 @@ const inputErrors = [
   errorCodes.FST_ERR_BAD_URL,
 ];
 
-export function buildApp(fixtureOrigin = DEFAULT_FIXTURE_ORIGIN) {
+const monitorViewSql = `SELECT m.id, m.name, m.url, m.interval_seconds, m.enabled, m.created_at, m.updated_at,
+  c.id AS check_id, c.monitor_id, c.trigger, c.state, c.http_status, c.latency_ms, c.failure_reason, c.started_at, c.finished_at
+  FROM monitors m LEFT JOIN LATERAL (
+    SELECT id, monitor_id, trigger, state, http_status, latency_ms, failure_reason, started_at, finished_at
+    FROM check_runs WHERE monitor_id = m.id ORDER BY finished_at DESC, id DESC LIMIT 1
+  ) c ON true`;
+
+export function buildApp(fixtureOrigin = DEFAULT_FIXTURE_ORIGIN, database: DatabaseConfig = databaseConfig()) {
   const handleError = (error: unknown, _request: FastifyRequest, reply: FastifyReply) => {
     const failure = error instanceof ApiError ? error
       : inputErrors.some((ErrorType) => error instanceof ErrorType)
@@ -24,16 +36,31 @@ export function buildApp(fixtureOrigin = DEFAULT_FIXTURE_ORIGIN) {
     return reply.code(ERROR_STATUS[failure.code]).type('application/json').send(body);
   };
   const app = Fastify({ logger: false, bodyLimit: 8_192, frameworkErrors: handleError });
-  const monitors = new Map<string, Monitor>();
-  const latestChecks = new Map<string, CheckRun>();
+  const pool = databasePool(database);
+  let poolClosed = false;
+  async function closePool() {
+    if (!poolClosed) { poolClosed = true; await pool.end(); }
+  }
+  app.addHook('onReady', async () => {
+    try { await verifySchema(pool); }
+    catch (error) { await closePool(); throw error; }
+  });
+  app.addHook('onClose', closePool);
 
   app.setErrorHandler(handleError);
   app.setNotFoundHandler(async () => { throw new ApiError('NOT_FOUND', 'Resource not found.'); });
 
   app.get('/health', async () => ({ data: { status: 'ok' } }));
-  app.get('/monitors', async () => ({ data: Array.from(monitors.values(), (monitor) => ({
-    ...monitor, latestCheck: latestChecks.get(monitor.id) ?? null,
-  })) }));
+  app.get('/monitors', async () => {
+    const result = await pool.query<MonitorViewRow>(`${monitorViewSql} ORDER BY m.created_at, m.id`);
+    return { data: result.rows.map(monitorViewFromRow) };
+  });
+
+  app.get<{ Params: { id: string } }>('/monitors/:id', async (request) => {
+    const result = await pool.query<MonitorViewRow>(`${monitorViewSql} WHERE m.id = $1`, [monitorId(request.params.id)]);
+    if (!result.rows[0]) throw new ApiError('NOT_FOUND', 'Monitor not found.');
+    return { data: monitorViewFromRow(result.rows[0]) };
+  });
 
   app.post<{ Body: unknown }>(
     '/monitors', async (request, reply) => {
@@ -42,17 +69,53 @@ export function buildApp(fixtureOrigin = DEFAULT_FIXTURE_ORIGIN) {
       const monitor: Monitor = {
         id: randomUUID(), ...input, createdAt: now, updatedAt: now,
       };
-      monitors.set(monitor.id, monitor);
-      return reply.code(201).send({ data: { ...monitor, latestCheck: null } });
+      const result = await pool.query<MonitorRow>(`INSERT INTO monitors
+        (id, name, url, interval_seconds, enabled, created_at, updated_at)
+        VALUES ($1, $2, $3, $4, $5, $6, $7) RETURNING *`, monitorToValues(monitor));
+      return reply.code(201).send({ data: { ...monitorFromRow(result.rows[0]), latestCheck: null } });
     },
   );
 
+  app.put<{ Params: { id: string }; Body: unknown }>('/monitors/:id', async (request) => {
+    const id = monitorId(request.params.id);
+    const input = monitorInput(request.body, fixtureOrigin);
+    const result = await pool.query<MonitorRow>(`UPDATE monitors SET name = $2, url = $3,
+      interval_seconds = $4, enabled = $5, updated_at = $6 WHERE id = $1 RETURNING *`,
+    [id, input.name, input.url, input.interval, input.enabled, new Date()]);
+    if (!result.rows[0]) throw new ApiError('NOT_FOUND', 'Monitor not found.');
+    const latest = await pool.query<CheckRunRow>('SELECT * FROM check_runs WHERE monitor_id = $1 ORDER BY finished_at DESC, id DESC LIMIT 1', [id]);
+    return { data: { ...monitorFromRow(result.rows[0]), latestCheck: latest.rows[0] ? checkRunFromRow(latest.rows[0]) : null } };
+  });
+
+  app.delete<{ Params: { id: string } }>('/monitors/:id', async (request) => {
+    // PostgreSQL's FK cascade removes every CheckRun in the same statement.
+    const result = await pool.query<{ id: string }>('DELETE FROM monitors WHERE id = $1 RETURNING id', [monitorId(request.params.id)]);
+    if (!result.rows[0]) throw new ApiError('NOT_FOUND', 'Monitor not found.');
+    return { data: { id: result.rows[0].id } };
+  });
+
   app.post<{ Params: { id: string } }>('/monitors/:id/checks', async (request) => {
-    const monitor = monitors.get(monitorId(request.params.id));
-    if (!monitor) throw new ApiError('NOT_FOUND', 'Monitor not found.');
-    const result = await checkMonitor(monitor, fixtureOrigin);
-    latestChecks.set(monitor.id, result);
-    return { data: result };
+    const stored = await pool.query<MonitorRow>('SELECT * FROM monitors WHERE id = $1', [monitorId(request.params.id)]);
+    if (!stored.rows[0]) throw new ApiError('NOT_FOUND', 'Monitor not found.');
+    const result = await checkMonitor(monitorFromRow(stored.rows[0]), fixtureOrigin);
+    const saved = await pool.query<CheckRunRow>(`INSERT INTO check_runs
+      (id, monitor_id, trigger, state, http_status, latency_ms, failure_reason, started_at, finished_at)
+      VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9) RETURNING *`, checkRunToValues(result));
+    return { data: checkRunFromRow(saved.rows[0]) };
+  });
+
+  app.get<{ Params: { id: string } }>('/monitors/:id/checks', async (request) => {
+    const id = monitorId(request.params.id);
+    const parent = await pool.query('SELECT id FROM monitors WHERE id = $1', [id]);
+    if (!parent.rows[0]) throw new ApiError('NOT_FOUND', 'Monitor not found.');
+    const history = await pool.query<CheckRunRow>('SELECT * FROM check_runs WHERE monitor_id = $1 ORDER BY finished_at DESC, id DESC', [id]);
+    return { data: history.rows.map(checkRunFromRow) };
+  });
+
+  app.get<{ Params: { id: string } }>('/checks/:id', async (request) => {
+    const result = await pool.query<CheckRunRow>('SELECT * FROM check_runs WHERE id = $1', [monitorId(request.params.id)]);
+    if (!result.rows[0]) throw new ApiError('NOT_FOUND', 'CheckRun not found.');
+    return { data: checkRunFromRow(result.rows[0]) };
   });
 
   return app;
diff --git a/server/mapping.ts b/server/mapping.ts
new file mode 100644
index 0000000..f8fddbd
--- /dev/null
+++ b/server/mapping.ts
@@ -0,0 +1,46 @@
+import type { CheckRun, Monitor, MonitorView } from './model.ts';
+
+// PostgreSQL rows use snake_case, integer/boolean columns, and pg Date objects.
+// API/domain models use camelCase and canonical UTC ISO strings, never raw rows.
+export type MonitorRow = {
+  id: string; name: string; url: string; interval_seconds: number; enabled: boolean;
+  created_at: Date; updated_at: Date;
+};
+export type CheckRunRow = {
+  id: string; monitor_id: string; trigger: CheckRun['trigger']; state: CheckRun['state'];
+  http_status: number | null; latency_ms: number; failure_reason: CheckRun['failureReason'];
+  started_at: Date; finished_at: Date;
+};
+export type MonitorViewRow = MonitorRow & Omit<CheckRunRow, 'id'> & { check_id: string | null };
+
+export function monitorFromRow(row: MonitorRow): Monitor {
+  return {
+    id: row.id, name: row.name, url: row.url, interval: row.interval_seconds, enabled: row.enabled,
+    createdAt: row.created_at.toISOString(), updatedAt: row.updated_at.toISOString(),
+  };
+}
+
+export function checkRunFromRow(row: CheckRunRow): CheckRun {
+  return {
+    id: row.id, monitorId: row.monitor_id, trigger: row.trigger, state: row.state,
+    httpStatus: row.http_status, latencyMs: row.latency_ms, failureReason: row.failure_reason,
+    startedAt: row.started_at.toISOString(), finishedAt: row.finished_at.toISOString(),
+  };
+}
+
+export function monitorViewFromRow(row: MonitorViewRow): MonitorView {
+  return {
+    ...monitorFromRow(row),
+    latestCheck: row.check_id === null ? null : checkRunFromRow({ ...row, id: row.check_id }),
+  };
+}
+
+export function monitorToValues(monitor: Monitor) {
+  return [monitor.id, monitor.name, monitor.url, monitor.interval, monitor.enabled,
+    new Date(monitor.createdAt), new Date(monitor.updatedAt)];
+}
+
+export function checkRunToValues(check: CheckRun) {
+  return [check.id, check.monitorId, check.trigger, check.state, check.httpStatus,
+    check.latencyMs, check.failureReason, new Date(check.startedAt), new Date(check.finishedAt)];
+}
diff --git a/server/schema.ts b/server/schema.ts
new file mode 100644
index 0000000..ff6fe07
--- /dev/null
+++ b/server/schema.ts
@@ -0,0 +1,49 @@
+import type { Pool } from 'pg';
+import { checkMigrationHistory, migrationFiles } from './migrate.ts';
+
+const expectedColumns = {
+  monitors: {
+    id: ['uuid', 'NO'], name: ['text', 'NO'], url: ['text', 'NO'],
+    interval_seconds: ['int4', 'NO'], enabled: ['bool', 'NO'],
+    created_at: ['timestamptz', 'NO'], updated_at: ['timestamptz', 'NO'],
+  },
+  check_runs: {
+    id: ['uuid', 'NO'], monitor_id: ['uuid', 'NO'], trigger: ['text', 'NO'], state: ['text', 'NO'],
+    http_status: ['int4', 'YES'], latency_ms: ['int4', 'NO'], failure_reason: ['text', 'YES'],
+    started_at: ['timestamptz', 'NO'], finished_at: ['timestamptz', 'NO'],
+  },
+} as const;
+
+export async function verifySchema(pool: Pool): Promise<void> {
+  const client = await pool.connect();
+  try {
+    const history = await client.query<{ version: string; checksum: string }>('SELECT version, checksum FROM schema_migrations ORDER BY version');
+    checkMigrationHistory(history.rows, await migrationFiles(), true);
+    const columns = await client.query<{
+      table_name: string; column_name: string; udt_name: string; is_nullable: string; datetime_precision: number | null;
+    }>(`SELECT table_name, column_name, udt_name, is_nullable, datetime_precision
+      FROM information_schema.columns WHERE table_schema = current_schema() AND table_name IN ('monitors', 'check_runs')`);
+    for (const [table, fields] of Object.entries(expectedColumns)) {
+      for (const [name, [type, nullable]] of Object.entries(fields)) {
+        const column = columns.rows.find((row) => row.table_name === table && row.column_name === name);
+        if (!column || column.udt_name !== type || column.is_nullable !== nullable ||
+          (type === 'timestamptz' && column.datetime_precision !== 3)) {
+          throw new Error(`Incompatible database schema: ${table}.${name}.`);
+        }
+      }
+    }
+    const keys = await client.query<{ table_name: string; definition: string }>(`
+      SELECT rel.relname AS table_name, pg_get_constraintdef(con.oid) AS definition
+      FROM pg_constraint con JOIN pg_class rel ON rel.oid = con.conrelid
+      JOIN pg_namespace ns ON ns.oid = rel.relnamespace
+      WHERE ns.nspname = current_schema() AND rel.relname IN ('monitors', 'check_runs') AND con.contype IN ('p', 'f')`);
+    for (const table of ['monitors', 'check_runs']) {
+      if (!keys.rows.some((row) => row.table_name === table && row.definition === 'PRIMARY KEY (id)')) {
+        throw new Error(`Incompatible database schema: ${table} primary key.`);
+      }
+    }
+    if (!keys.rows.some((row) => row.table_name === 'check_runs' && row.definition === 'FOREIGN KEY (monitor_id) REFERENCES monitors(id) ON DELETE CASCADE')) {
+      throw new Error('Incompatible database schema: CheckRun parent deletion rule.');
+    }
+  } finally { client.release(); }
+}
diff --git a/test/contracts.test.ts b/test/contracts.test.ts
index eb7ac5a..3790cac 100644
--- a/test/contracts.test.ts
+++ b/test/contracts.test.ts
@@ -2,21 +2,25 @@ import { after, before, test } from 'node:test';
 import assert from 'node:assert/strict';
 import { buildApp } from '../server/app.ts';
 import { fixtureServer } from './fixture.ts';
+import { dropTestSchema, resetTestSchema, testDatabaseConfig } from './database.ts';
 import type { ApiErrorCode, ApiSuccess, CheckRun, MonitorView } from '../server/model.ts';
 import scenario from '../evidence/E02/scenario.json' with { type: 'json' };
 
 const fixture = fixtureServer();
-const app = buildApp(scenario.fixtureOrigin);
+const database = testDatabaseConfig('e03_contracts');
+const app = buildApp(scenario.fixtureOrigin, database);
 const boundaries = scenario.additionalBoundaries;
 app.get(boundaries.internalErrorRoute, async () => { throw new Error(boundaries.privateInternalMessage); });
 
 before(async () => {
+  await resetTestSchema(database.schema);
   await new Promise<void>((resolve) => fixture.server.listen(4311, '127.0.0.1', resolve));
   await app.listen({ host: '127.0.0.1', port: 4312 });
 });
 
 after(async () => {
   await app.close();
+  await dropTestSchema(database.schema);
   fixture.server.closeAllConnections();
   await new Promise<void>((resolve) => fixture.server.close(() => resolve()));
 });
diff --git a/test/database.ts b/test/database.ts
new file mode 100644
index 0000000..01562f8
--- /dev/null
+++ b/test/database.ts
@@ -0,0 +1,20 @@
+import { databaseConfig, databasePool, schemaIdentifier } from '../server/database.ts';
+import { migrate } from '../server/migrate.ts';
+
+export function testDatabaseConfig(schema: string) {
+  if (!/^e03_[a-z_]+$/.test(schema)) throw new Error('Test cleanup is restricted to an explicit e03_ schema.');
+  return { ...databaseConfig(), schema };
+}
+
+export async function resetTestSchema(schema: string, applyMigrations = true) {
+  const config = testDatabaseConfig(schema);
+  const pool = databasePool(config);
+  try { await pool.query(`DROP SCHEMA IF EXISTS ${schemaIdentifier(schema)} CASCADE`); }
+  finally { await pool.end(); }
+  if (applyMigrations) await migrate(config);
+  return config;
+}
+
+export async function dropTestSchema(schema: string) {
+  await resetTestSchema(schema, false);
+}
diff --git a/test/functional.test.ts b/test/functional.test.ts
index fa93544..daa27eb 100644
--- a/test/functional.test.ts
+++ b/test/functional.test.ts
@@ -5,18 +5,22 @@ import { buildApp } from '../server/app.ts';
 import { checkMonitor, DEFAULT_FIXTURE_ORIGIN } from '../server/check.ts';
 import type { ApiSuccess, Monitor, MonitorView } from '../server/model.ts';
 import { fixtureServer } from './fixture.ts';
+import { dropTestSchema, resetTestSchema, testDatabaseConfig } from './database.ts';
 
 const fixture = fixtureServer();
-const app = buildApp();
+const database = testDatabaseConfig('e03_functional');
+const app = buildApp(DEFAULT_FIXTURE_ORIGIN, database);
 let guardCalls = 0;
 const guard = createServer((_request, response) => { guardCalls++; response.end('guard'); });
 
 before(async () => {
+  await resetTestSchema(database.schema);
   await new Promise<void>((resolve) => fixture.server.listen(4311, '127.0.0.1', resolve));
   await new Promise<void>((resolve) => guard.listen(4314, '127.0.0.1', resolve));
 });
 after(async () => {
   await app.close();
+  await dropTestSchema(database.schema);
   fixture.server.closeAllConnections();
   guard.closeAllConnections();
   await Promise.all([
@@ -33,7 +37,7 @@ async function create(path: string): Promise<MonitorView> {
   return response.json<ApiSuccess<MonitorView>>().data;
 }
 
-test('create a Monitor in memory and synchronously observe GET /ok', async () => {
+test('create a Monitor in PostgreSQL and synchronously observe GET /ok', async () => {
   const monitor = await create('/ok');
   assert.equal(monitor.interval, 60);
   assert.equal(monitor.enabled, true);
@@ -80,8 +84,9 @@ test('a non-fixture URL is rejected without contacting the controlled guard', as
   assert.equal(guardCalls, 0);
 });
 
-test('another application instance starts with empty memory', async () => {
-  const fresh = buildApp();
-  try { assert.deepEqual((await fresh.inject('/monitors')).json(), { data: [] }); }
+test('another application instance reads the same persisted Monitors and latest checks', async () => {
+  const existing = (await app.inject('/monitors')).json();
+  const fresh = buildApp(DEFAULT_FIXTURE_ORIGIN, database);
+  try { assert.deepEqual((await fresh.inject('/monitors')).json(), existing); }
   finally { await fresh.close(); }
 });
diff --git a/test/persistence.test.ts b/test/persistence.test.ts
new file mode 100644
index 0000000..7f9ed58
--- /dev/null
+++ b/test/persistence.test.ts
@@ -0,0 +1,207 @@
+import { test } from 'node:test';
+import assert from 'node:assert/strict';
+import { execFile } from 'node:child_process';
+import { promisify } from 'node:util';
+import { mkdir, writeFile } from 'node:fs/promises';
+import { buildApp } from '../server/app.ts';
+import { databasePool } from '../server/database.ts';
+import { monitorFromRow, checkRunFromRow, monitorToValues, checkRunToValues } from '../server/mapping.ts';
+import type { MonitorRow, CheckRunRow } from '../server/mapping.ts';
+import type { CheckRun, Monitor, MonitorView } from '../server/model.ts';
+import { fixtureServer } from './fixture.ts';
+import { dropTestSchema, resetTestSchema } from './database.ts';
+import scenario from '../evidence/E03/scenario.json' with { type: 'json' };
+
+const execute = promisify(execFile);
+async function record(name: string, evidence: unknown) {
+  await mkdir('output/e03', { recursive: true });
+  await writeFile(`output/e03/${name}.json`, JSON.stringify(evidence, null, 2) + '\n');
+}
+async function success<T>(response: Response, status = 200): Promise<T> {
+  const body = await response.json();
+  assert.equal(response.status, status, JSON.stringify(body));
+  assert.deepEqual(Object.keys(body), ['data']);
+  return body.data;
+}
+async function failure(response: Response, status: number, code: string) {
+  const body = await response.json();
+  assert.equal(response.status, status, JSON.stringify(body));
+  assert.deepEqual(Object.keys(body), ['error']);
+  assert.deepEqual(Object.keys(body.error).sort(), ['code', 'message']);
+  assert.equal(body.error.code, code);
+}
+function request(path: string, method = 'GET', body?: unknown) {
+  return fetch(`${scenario.apiOrigin}${path}`, {
+    method,
+    ...(body === undefined ? {} : { headers: { 'content-type': 'application/json' }, body: JSON.stringify(body) }),
+  });
+}
+function sortedById<T extends { id: string }>(values: T[]) {
+  return [...values].sort((a, b) => a.id.localeCompare(b.id));
+}
+
+test('E03 migration CLI applies the fresh chain and a repeated command changes nothing', async () => {
+  const started = performance.now();
+  const config = await resetTestSchema(scenario.additionalAssertions.migrationSchema, false);
+  const pool = databasePool(config);
+  try {
+    const options = { cwd: process.cwd(), env: { ...process.env, DATABASE_URL: config.connectionString, DATABASE_SCHEMA: config.schema }, timeout: 10_000 };
+    const first = await execute(process.execPath, ['server/migrate.ts'], options);
+    assert.deepEqual(JSON.parse(first.stdout).applied, scenario.additionalAssertions.expectedMigrationFiles);
+    const before = (await pool.query('SELECT version, checksum, applied_at FROM schema_migrations ORDER BY version')).rows;
+    assert.equal(before.length, 2);
+    for (const row of before) { assert.match(row.checksum, /^[0-9a-f]{64}$/); assert.ok(row.applied_at instanceof Date); }
+    const second = await execute(process.execPath, ['server/migrate.ts'], options);
+    assert.deepEqual(JSON.parse(second.stdout).applied, scenario.additionalAssertions.repeatMigrationExpectedApplied);
+    const after = (await pool.query('SELECT version, checksum, applied_at FROM schema_migrations ORDER BY version')).rows;
+    assert.deepEqual(after, before);
+    const app = buildApp(scenario.fixtureOrigin, config);
+    try { await app.ready(); }
+    finally { await app.close(); }
+    await record('migrations', { command: 'node server/migrate.ts (twice)', schema: config.schema, first: JSON.parse(first.stdout), second: JSON.parse(second.stdout), history: after, startupVerified: true, durationMs: Math.round(performance.now() - started) });
+  } finally { await pool.end(); await dropTestSchema(config.schema); }
+});
+
+test('E03 missing mapped interval_seconds rejects application startup before listening', async () => {
+  const started = performance.now();
+  const config = await resetTestSchema(scenario.additionalAssertions.incompatibleSchema);
+  const pool = databasePool(config);
+  const app = buildApp(scenario.fixtureOrigin, config);
+  try {
+    await pool.query('ALTER TABLE monitors DROP COLUMN interval_seconds');
+    let message = '';
+    await assert.rejects(app.listen({ host: '127.0.0.1', port: 4312 }), (error: unknown) => {
+      assert.ok(error instanceof Error);
+      message = error.message;
+      assert.match(message, /Incompatible database schema: monitors\.interval_seconds/);
+      return true;
+    });
+    assert.equal(app.server.listening, false);
+    await record('schema-mismatch', { schema: config.schema, removedColumn: scenario.additionalAssertions.incompatibleColumn, startupRejected: true, listening: app.server.listening, message, durationMs: Math.round(performance.now() - started) });
+  } finally { await app.close(); await pool.end(); await dropTestSchema(config.schema); }
+});
+
+test('E03 A,A,B survive a fresh application; all Monitor lifecycle and CheckRun reads use PostgreSQL', async () => {
+  const started = performance.now();
+  const config = await resetTestSchema('e03_persistence');
+  const pool = databasePool(config);
+  const fixture = fixtureServer();
+  let app = buildApp(scenario.fixtureOrigin, config);
+  try {
+    await new Promise<void>((resolve) => fixture.server.listen(4311, '127.0.0.1', resolve));
+    await app.listen({ host: '127.0.0.1', port: 4312 });
+    const monitors: MonitorView[] = [];
+    for (const input of scenario.monitors) monitors.push(await success<MonitorView>(await request('/monitors', 'POST', input), 201));
+    const checks: CheckRun[] = [];
+    for (const expected of scenario.checkSequence) {
+      const check = await success<CheckRun>(await request(`/monitors/${monitors[expected.monitor].id}/checks`, 'POST'));
+      assert.equal(check.state, expected.state);
+      assert.equal(check.httpStatus, expected.httpStatus);
+      assert.equal(check.failureReason, expected.failureReason);
+      checks.push(check);
+    }
+    const before = await success<MonitorView[]>(await request('/monitors'));
+    assert.equal(before.length, scenario.restart.expectedMonitorCount);
+    assert.equal(new Set(checks.map((check) => check.id)).size, scenario.restart.expectedCheckCount);
+    await app.close();
+    app = buildApp(scenario.fixtureOrigin, config);
+    await app.listen({ host: '127.0.0.1', port: 4312 });
+    const after = await success<MonitorView[]>(await request('/monitors'));
+    assert.deepEqual(after, before);
+    const histories: CheckRun[] = [];
+    for (const monitor of monitors) {
+      const history = await success<CheckRun[]>(await request(`/monitors/${monitor.id}/checks`));
+      assert.deepEqual(sortedById(history), sortedById(checks.filter((check) => check.monitorId === monitor.id)));
+      histories.push(...history);
+      const detail = await success<MonitorView>(await request(`/monitors/${monitor.id}`));
+      assert.deepEqual(detail, after.find((item) => item.id === monitor.id));
+    }
+    assert.equal(histories.length, 3);
+    for (const check of checks) assert.deepEqual(await success(await request(`/checks/${check.id}`)), check);
+    assert.deepEqual(Object.fromEntries(fixture.calls), { '/ok': 2, '/fail': 1 });
+
+    const a = monitors[0];
+    const b = monitors[1];
+    const updatedInput = { ...scenario.monitors[0], ...scenario.update };
+    let updated = await success<MonitorView>(await request(`/monitors/${a.id}`, 'PUT', updatedInput));
+    assert.equal(updated.name, scenario.update.name);
+    assert.equal(updated.interval, scenario.update.interval);
+    assert.equal(updated.createdAt, a.createdAt);
+    assert.ok(Date.parse(updated.updatedAt) >= Date.parse(a.updatedAt));
+    const lifecycle = [updated];
+    for (const enabled of scenario.enabledSequence) {
+      updated = await success<MonitorView>(await request(`/monitors/${a.id}`, 'PUT', { ...updatedInput, enabled }));
+      assert.equal(updated.enabled, enabled);
+      const authoritative = await success<MonitorView>(await request(`/monitors/${a.id}`));
+      assert.deepEqual(authoritative, updated);
+      const row = (await pool.query<MonitorRow>('SELECT * FROM monitors WHERE id = $1', [a.id])).rows[0];
+      const { latestCheck: _latest, ...domain } = updated;
+      assert.deepEqual(monitorFromRow(row), domain);
+      lifecycle.push(updated);
+    }
+    for (const override of scenario.additionalAssertions.invalidUpdateOverrides) {
+      await failure(await request(`/monitors/${a.id}`, 'PUT', { ...updatedInput, ...override }), 400, 'INVALID_INPUT');
+      assert.deepEqual(await success(await request(`/monitors/${a.id}`)), updated);
+    }
+    assert.deepEqual(await success(await request(`/monitors/${b.id}`, 'DELETE')), { id: b.id });
+    for (const path of [`/monitors/${b.id}`, `/monitors/${b.id}/checks`, `/checks/${checks[2].id}`]) {
+      await failure(await request(path), 404, 'NOT_FOUND');
+    }
+    await failure(await request(`/monitors/${b.id}/checks`, 'POST'), 404, 'NOT_FOUND');
+    const missing = scenario.additionalAssertions.missingId;
+    for (const [method, suffix] of [['GET', ''], ['PUT', ''], ['DELETE', ''], ['GET', '/checks'], ['POST', '/checks']]) {
+      await failure(await request(`/monitors/${missing}${suffix}`, method, method === 'PUT' ? updatedInput : undefined), 404, 'NOT_FOUND');
+      await failure(await request(`/monitors/${scenario.additionalAssertions.malformedId}${suffix}`, method, method === 'PUT' ? updatedInput : undefined), 400, 'INVALID_INPUT');
+    }
+    await failure(await request(`/checks/${missing}`), 404, 'NOT_FOUND');
+    await failure(await request(`/checks/${scenario.additionalAssertions.malformedId}`), 400, 'INVALID_INPUT');
+    const remaining = await success<MonitorView[]>(await request('/monitors'));
+    assert.deepEqual(remaining, [updated]);
+    const aHistory = await success<CheckRun[]>(await request(`/monitors/${a.id}/checks`));
+    assert.deepEqual(sortedById(aHistory), sortedById(checks.slice(0, 2)));
+    const counts = (await pool.query(`SELECT (SELECT count(*)::int FROM monitors) AS monitors,
+      (SELECT count(*)::int FROM check_runs) AS checks,
+      (SELECT count(*)::int FROM check_runs c LEFT JOIN monitors m ON m.id = c.monitor_id WHERE m.id IS NULL) AS orphans`)).rows[0];
+    assert.deepEqual(counts, { monitors: 1, checks: 2, orphans: 0 });
+    assert.equal((await pool.query('SELECT id FROM check_runs WHERE monitor_id = $1', [b.id])).rowCount, 0);
+    assert.deepEqual(Object.fromEntries(fixture.calls), { '/ok': 2, '/fail': 1 });
+    await record('persistence', { databaseSchema: config.schema, beforeRestart: before, afterRestart: after, historicalChecksAfterRestart: histories, lifecycle, deletedMonitor: b.id, deletedCheckRun: checks[2].id, afterDeletion: remaining, counts, fixtureCalls: Object.fromEntries(fixture.calls), durationMs: Math.round(performance.now() - started) });
+  } finally {
+    await app.close();
+    await pool.end();
+    fixture.server.closeAllConnections();
+    await new Promise<void>((resolve) => fixture.server.close(() => resolve()));
+    await dropTestSchema(config.schema);
+  }
+});
+
+test('E03 canonical rows round-trip UUID, timezone, boolean, integer, zero and null without leaking SQL names', async () => {
+  const started = performance.now();
+  const config = await resetTestSchema('e03_mapping');
+  const pool = databasePool(config);
+  const app = buildApp(scenario.fixtureOrigin, config);
+  const input = scenario.additionalAssertions.mappingFixture;
+  const monitor: Monitor = { id: input.id, name: input.name.trim(), url: input.url, interval: input.interval, enabled: input.enabled, createdAt: input.timestamp, updatedAt: input.timestamp };
+  const check: CheckRun = { id: '00000000-0000-4000-8000-000000000102', monitorId: monitor.id, trigger: 'MANUAL', state: 'FAILED', httpStatus: null, latencyMs: input.latencyMs, failureReason: 'CONNECTION_FAILURE', startedAt: input.timestamp, finishedAt: input.timestamp };
+  try {
+    const storedMonitor = (await pool.query<MonitorRow>(`INSERT INTO monitors (id, name, url, interval_seconds, enabled, created_at, updated_at)
+      VALUES ($1, $2, $3, $4, $5, $6, $7) RETURNING *`, monitorToValues(monitor))).rows[0];
+    const storedCheck = (await pool.query<CheckRunRow>(`INSERT INTO check_runs (id, monitor_id, trigger, state, http_status, latency_ms, failure_reason, started_at, finished_at)
+      VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9) RETURNING *`, checkRunToValues(check))).rows[0];
+    assert.ok(storedMonitor.created_at instanceof Date);
+    assert.ok(storedCheck.started_at instanceof Date);
+    assert.equal(storedMonitor.enabled, false);
+    assert.equal(storedMonitor.interval_seconds, 1);
+    assert.equal(storedCheck.latency_ms, 0);
+    assert.equal(storedCheck.http_status, null);
+    const mappedMonitor = monitorFromRow(storedMonitor);
+    const mappedCheck = checkRunFromRow(storedCheck);
+    assert.deepEqual(mappedMonitor, { ...monitor, createdAt: input.expectedTimestamp, updatedAt: input.expectedTimestamp });
+    assert.deepEqual(mappedCheck, { ...check, startedAt: input.expectedTimestamp, finishedAt: input.expectedTimestamp });
+    const response = await app.inject(`/monitors/${monitor.id}`);
+    assert.equal(response.statusCode, 200);
+    assert.deepEqual(response.json(), { data: { ...mappedMonitor, latestCheck: mappedCheck } });
+    assert.deepEqual((await app.inject(`/checks/${check.id}`)).json(), { data: mappedCheck });
+    await record('mapping', { monitor: mappedMonitor, check: mappedCheck, sqlTimestampInput: input.timestamp, sqlTimestampRuntimeType: storedMonitor.created_at.constructor.name, durationMs: Math.round(performance.now() - started) });
+  } finally { await app.close(); await pool.end(); await dropTestSchema(config.schema); }
+});


