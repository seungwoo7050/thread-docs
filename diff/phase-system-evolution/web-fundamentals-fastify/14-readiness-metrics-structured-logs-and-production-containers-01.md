# E24 Readiness, Metrics, Structured Logs와 Production Container

## `test: freeze E24 operations and container boundaries`

diff --git a/evidence/phase-1/E24/baseline.mjs b/evidence/phase-1/E24/baseline.mjs
new file mode 100644
index 0000000..78a8cac
--- /dev/null
+++ b/evidence/phase-1/E24/baseline.mjs
@@ -0,0 +1,56 @@
+import assert from 'node:assert/strict';
+import { createHash } from 'node:crypto';
+import { execFileSync } from 'node:child_process';
+import { mkdir, readFile, writeFile } from 'node:fs/promises';
+import { buildApp } from '../../../server/app.ts';
+import { databaseConfig, databasePool, schemaIdentifier } from '../../../server/database.ts';
+import { migrate } from '../../../server/migrate.ts';
+import { requireFreePorts } from '../E10/fixture.ts';
+import { insertOperationsFixture } from './fixture.ts';
+import scenario from './scenario.json' with { type: 'json' };
+
+const git = (...args) => execFileSync('git', args, { encoding: 'utf8' }).trim();
+assert.equal(process.versions.node, scenario.runtime.node);
+assert.equal(git('branch', '--show-current'), 'track/fundamentals-fastify');
+assert.equal((await readFile('SPEC_REVISION', 'utf8')).trim(), scenario.specRevision);
+assert.ok(git('diff', '--name-only', scenario.start).split('\n').filter(Boolean)
+  .every(path => path.startsWith('evidence/phase-1/E24/')), 'Baseline product must match START.');
+const hashes = {};
+for (const file of ['scenario.json', 'fixture.ts', 'baseline.mjs']) {
+  hashes[file] = createHash('sha256').update(await readFile(new URL(file, import.meta.url))).digest('hex');
+}
+const output = 'output/phase-1/e24';
+await mkdir(output, { recursive: true });
+await writeFile(`${output}/baseline.started.json`, JSON.stringify({ hashes, head: git('rev-parse', 'HEAD') }) + '\n', { flag: 'wx' });
+const config = { ...databaseConfig(), schema: scenario.ownership.baselineSchema };
+const pool = databasePool(config);
+const app = buildApp(undefined, config);
+const began = performance.now();
+const report = { head: git('rev-parse', 'HEAD'), start: scenario.start, productMatchesStart: true, hashes, result: 'NOT_RUN' };
+let owned = false;
+try {
+  await requireFreePorts();
+  await pool.query(`CREATE SCHEMA ${schemaIdentifier(config.schema)}`); owned = true;
+  await migrate(config);
+  await insertOperationsFixture(pool);
+  const live = await app.inject(scenario.paths.legacyLive);
+  const ready = await app.inject(scenario.baseline.path);
+  report.observation = { legacyLiveStatus: live.statusCode, readinessStatus: ready.statusCode,
+    readinessCode: ready.json().error?.code, users: 2, monitors: 2, terminalChecks: 4,
+    containerBuilds: 0, postgresStops: 0, outboundRequests: 0 };
+  assert.equal(live.statusCode, 200);
+  assert.equal(ready.statusCode, scenario.baseline.unchangedStatus);
+  assert.equal(ready.json().error.code, scenario.baseline.unchangedCode);
+  report.result = 'REPRODUCED';
+  report.decisiveFailure = 'The existing liveness endpoint works, but the frozen readiness path has no unauthenticated operational interface.';
+} catch (error) { report.result = 'FAILED'; report.failure = error.message; process.exitCode = 1; }
+finally {
+  await app.close();
+  if (owned) await pool.query(`DROP SCHEMA ${schemaIdentifier(config.schema)} CASCADE`);
+  await pool.end();
+  await requireFreePorts();
+  report.cleanup = { schemaDropped: owned, portsFree: true };
+  report.durationMs = Math.round(performance.now() - began);
+  await writeFile(`${output}/baseline.json`, JSON.stringify(report, null, 2) + '\n', { flag: 'wx' });
+  console.log(JSON.stringify(report));
+}
diff --git a/evidence/phase-1/E24/fixture.ts b/evidence/phase-1/E24/fixture.ts
new file mode 100644
index 0000000..0628607
--- /dev/null
+++ b/evidence/phase-1/E24/fixture.ts
@@ -0,0 +1,40 @@
+import { randomBytes } from 'node:crypto';
+import type { Pool } from 'pg';
+import { hashPassword } from '../../../server/password.ts';
+import scenario from './scenario.json' with { type: 'json' };
+
+export async function insertOperationsFixture(pool: Pool) {
+  const secret = () => randomBytes(24).toString('hex');
+  const body = secret();
+  const manualKey = secret();
+  const unavailableKey = secret();
+  const names = [secret(), secret()];
+  const urls = [secret(), secret()];
+  const users = [];
+  const monitors = [];
+  for (let i = 0; i < 2; i++) {
+    const credentials = { username: scenario.dataset.users[i], password: secret() };
+    await pool.query('INSERT INTO users (id, username, password_hash) VALUES ($1, $2, $3)',
+      [scenario.dataset.userIds[i], credentials.username, await hashPassword(credentials.password)]);
+    users.push(credentials);
+    const monitor = { id: scenario.dataset.monitorIds[i], name: `${i === 0 ? 'Alice' : 'Bob'} ${names[i]}`,
+      url: `${scenario.fixture.origin}${scenario.fixture.paths[i]}?e24=${urls[i]}`, interval: 60, enabled: false };
+    await pool.query(`INSERT INTO monitors
+      (id, name, url, interval_seconds, enabled, created_at, updated_at, owner_user_id)
+      VALUES ($1, $2, $3, $4, $5, $6, $6, $7)`,
+    [monitor.id, monitor.name, monitor.url, monitor.interval, monitor.enabled,
+      scenario.dataset.monitorFields.timestamp, scenario.dataset.userIds[i]]);
+    monitors.push(monitor);
+  }
+  for (let n = 1; n <= 4; n++) {
+    const failed = n <= 3;
+    const at = new Date(Date.parse(scenario.dataset.monitorFields.timestamp) + n);
+    await pool.query(`INSERT INTO check_runs
+      (id, monitor_id, trigger, state, http_status, latency_ms, failure_reason, queued_at, started_at, finished_at)
+      VALUES ($1, $2, 'MANUAL', $3, $4, 0, $5, $6, $6, $6)`,
+    [`24000000-0000-4000-a000-${String(n).padStart(12, '0')}`, scenario.dataset.monitorIds[failed ? 0 : 1],
+      failed ? 'FAILED' : 'SUCCEEDED', failed ? 503 : 200, failed ? 'HTTP_STATUS' : null, at]);
+  }
+  return { users, monitors, body, manualKey, unavailableKey,
+    sentinels: [body, manualKey, unavailableKey, ...names, ...urls, ...users.map(user => user.password)] };
+}
diff --git a/evidence/phase-1/E24/scenario.json b/evidence/phase-1/E24/scenario.json
new file mode 100644
index 0000000..6521d16
--- /dev/null
+++ b/evidence/phase-1/E24/scenario.json
@@ -0,0 +1,71 @@
+{
+  "thread": "E24",
+  "profile": "phase-1",
+  "specRevision": "2ada57a71cd34fa2fae9809415c362a8bbfcdf02",
+  "start": "9d0a1974e1f279146917b840e69bbee19dcfc0c4",
+  "roles": ["api", "worker", "frontend"],
+  "paths": { "live": "/health/live", "ready": "/health/ready", "metrics": "/metrics", "legacyLive": "/health" },
+  "ports": { "fixture": 4311, "api": 4312, "frontend": 4313, "worker": 4314, "postgres": 15431 },
+  "ownership": {
+    "runtimeProject": "wse-fundamentals-e24",
+    "postgresProject": "wse-fundamentals",
+    "postgresService": "postgres",
+    "postgresNetwork": "wse-fundamentals_default",
+    "baselineSchema": "e24_baseline",
+    "containerSchema": "e24_container",
+    "rule": "Refuse existing runtime containers, fixture/schema or occupied ports. Attach API/worker to the existing owned PostgreSQL network. Stop/start only that exact PostgreSQL service at idle; preserve all volumes and public data. Drop only the created E24 schema and remove only E24 runtime containers/network."
+  },
+  "runtime": {
+    "node": "24.19.0",
+    "npm": "11.17.0",
+    "baseImage": "node:24.19.0-bookworm-slim@sha256:a9f5f7c91a432850b2a8a7797adf5eadb6c733ceed61167806cee7ea7fbc29df",
+    "postgresImage": "postgres:17.6-bookworm@sha256:f3bd19c606e442c3d7bdfa8002e03fe260a1023351e0ea4598032022b68dd6e3",
+    "uid": 1000,
+    "commands": { "api": ["node", "server/main.ts"], "worker": ["node", "server/worker.ts"], "frontend": ["node", "server.js"] },
+    "frontend": "Next standalone production output, copied .next/static, API_ORIGIN=http://api:4312 during both build and runtime; no dev server.",
+    "signal": "SIGTERM to the real container PID1; await each container exit, require code0 and no forced kill.",
+    "shutdownMs": { "api": 3000, "worker": 3000, "frontend": 30000 },
+    "observerExitDeadlineMs": 35000
+  },
+  "fixture": {
+    "origin": "http://127.0.0.1:4311",
+    "transport": "Explicit test-only sidecar shares the worker network namespace. API/worker smoke override NODE_ENV=test and WSE_TEST_FIXTURE_ORIGIN to this exact loopback origin. Production images/default compose remain NODE_ENV=production with no local trust.",
+    "paths": ["/fail", "/ok"],
+    "statuses": [503, 200],
+    "headerDelayMs": 250,
+    "body": "Runtime-only random sentinel; existing controlled fixture handlers supply the fixed statuses, with test-only query stripping/body replacement and request counts.",
+    "control": "/e24/control",
+    "counts": "/e24/counts"
+  },
+  "dataset": {
+    "users": ["alice-e24", "bob-e24"],
+    "userIds": ["24000000-0000-4000-9000-000000000001", "24000000-0000-4000-9000-000000000002"],
+    "monitorIds": ["24000000-0000-4000-8000-000000000001", "24000000-0000-4000-8000-000000000002"],
+    "names": "Alice/Bob followed by one runtime-only name sentinel each.",
+    "urls": "The exact loopback fixture /fail and /ok URLs, each with a runtime-only e24 query sentinel.",
+    "monitorFields": { "interval": 60, "enabled": false, "timestamp": "2026-08-28T00:00:00.000Z" },
+    "runs": "UUID24000000-0000-4000-a000- followed by12-digit ordinals1..4. Alice owns1..3 FAILED/503/HTTP_STATUS; Bob owns4 SUCCEEDED/200/null. MANUAL, latency0, queued/start/finish=base+ordinal milliseconds; other execution fields null.",
+    "manualIntents": 1,
+    "missingIds": "24000000-0000-4000-b000- followed by12-digit ordinals1..10.",
+    "secrets": "Passwords, issued session/CSRF tokens, URL/name/body sentinels and both idempotency keys remain in runtime memory/database only; artifacts contain counts/booleans, never their values."
+  },
+  "metrics": {
+    "http": ["http_request_duration_seconds_sum", "http_request_duration_seconds_count", "http_errors_total"],
+    "queue": "check_queue_age_seconds",
+    "worker": ["worker_active", "worker_claims_total", "worker_recovery_runs_total", "worker_recovered_checks_total"],
+    "dependency": "postgres_ready",
+    "labels": ["role", "route", "method", "status"],
+    "rule": "HTTP method is a fixed standard-method set plus OTHER, route is the registered template or unmatched, status is bounded HTTP status, role is api/worker. No user/resource/input/process-ID label. Queue age is read from authoritative QUEUED timestamps; omit it when PostgreSQL cannot answer, never report fabricated zero.",
+    "recovery": "Successful production recovery-operation calls and actually recovered row count are separate counters. This healthy smoke expects scans>0 and recovered0; it does not repeat E11 crashes."
+  },
+  "sequence": [
+    "Seed the fixed four terminal results; start API/frontend. Accept the one real manual intent, wait100ms and read a positive API queue-age sample before worker startup; no additional job or hook.",
+    "Start worker and its explicit test fixture, observe committed claim and active1 during the250ms header delay, then same-ID FAILED503 completion and active0. Both roles live200/ready200. Inspect production browser login/list/detail, static asset/server route and owner isolation.",
+    "Take metric labels before and after ten authenticated missing-resource requests. Each adds only the same template/method/status series, with exact404 count increase10. Check runtime sentinel exclusion and API-request/worker-process correlation for the one check.",
+    "At idle with zero QUEUED/RUNNING rows, stop only the owned PostgreSQL service once. API/worker live200 and ready503, preauthenticated one manual POST with a distinct key is rejected (non2xx), no new fixture request. Restore PostgreSQL once; ready200, original owners/results/session authority preserved and total CheckRuns5.",
+    "Inspect actual non-root PID1 commands/UIDs for API/worker/frontend, send SIGTERM and await bounded code0 exits. Remove only owned E24 runtime resources and schema; preserve the restored PostgreSQL service/volumes/public."
+  ],
+  "baseline": { "method": "GET", "path": "/health/ready", "unchangedStatus": 401, "unchangedCode": "UNAUTHENTICATED", "stop": "First decisive absent unauthenticated readiness interface; no container build or dependency stop in baseline." },
+  "budget": { "baselineInvocations": 1, "authorFullRuns": 1, "rootFullRuns": 1, "postgresStopsPerFullRun": 1, "loadRuns": 0, "automaticRetries": 0, "parameterSweeps": 0, "repairsMaximum": 2, "e11CrashReruns": 0, "e20DatasetReruns": 0 },
+  "ci": "Separate unit/integration/browser/container jobs. Default container smoke omits the capped PostgreSQL stop; full mode requires explicit invocation. No hosted execution claim."
+}


