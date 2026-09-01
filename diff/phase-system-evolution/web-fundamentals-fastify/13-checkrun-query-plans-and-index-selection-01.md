# E20 CheckRun Query Plan과 Index 선택

## `test: freeze E20 history query and skewed dataset`

diff --git a/evidence/phase-1/E20/history.sql b/evidence/phase-1/E20/history.sql
new file mode 100644
index 0000000..0a9044e
--- /dev/null
+++ b/evidence/phase-1/E20/history.sql
@@ -0,0 +1,5 @@
+SELECT c.* FROM check_runs c JOIN monitors m ON m.id = c.monitor_id
+      WHERE m.id = $1 AND m.owner_user_id = $2 AND c.finished_at IS NOT NULL
+        AND ($3::text IS NULL OR c.state = $3)
+        AND ($4::timestamptz IS NULL OR (c.finished_at, c.id) < ($4::timestamptz, $5::uuid))
+      ORDER BY c.finished_at DESC, c.id DESC LIMIT $6
diff --git a/evidence/phase-1/E20/observe.mjs b/evidence/phase-1/E20/observe.mjs
new file mode 100644
index 0000000..c27164a
--- /dev/null
+++ b/evidence/phase-1/E20/observe.mjs
@@ -0,0 +1,104 @@
+import assert from 'node:assert/strict';
+import { createHash } from 'node:crypto';
+import { execFileSync } from 'node:child_process';
+import { mkdir, readFile, writeFile } from 'node:fs/promises';
+import { databaseConfig, databasePool, schemaIdentifier } from '../../../server/database.ts';
+import { migrate } from '../../../server/migrate.ts';
+import { historyQuery } from '../../../server/history.ts';
+import { requireFreePorts } from '../E10/fixture.ts';
+import scenario from './scenario.json' with { type: 'json' };
+
+const mode = process.argv[2];
+assert.ok(['baseline', 'confirm', 'verify'].includes(mode), 'Choose baseline, confirm or verify explicitly.');
+assert.equal(process.versions.node, '24.19.0');
+const git = (...args) => execFileSync('git', args, { encoding: 'utf8' }).trim();
+assert.equal(git('branch', '--show-current'), 'track/fundamentals-fastify');
+assert.equal((await readFile('SPEC_REVISION', 'utf8')).trim(), scenario.specRevision);
+if (mode === 'baseline') assert.ok(git('diff', '--name-only', scenario.start).split('\n').filter(Boolean)
+  .every(path => path.startsWith('evidence/phase-1/E20/')), 'Baseline product must match START.');
+const frozen = ['scenario.json', 'history.sql', 'seed.sql', 'observe.mjs'];
+const hashes = {};
+for (const file of frozen) hashes[file] = createHash('sha256').update(await readFile(new URL(file, import.meta.url))).digest('hex');
+const sql = (await readFile(new URL('history.sql', import.meta.url), 'utf8')).trimEnd();
+const source = await readFile('server/app.ts', 'utf8');
+assert.ok(source.includes('const history = await pool.query<CheckRunRow>(`' + sql + '`,\n    ' +
+  '[id, request.user!.id, query.state, query.after?.finishedAt ?? null, query.after?.id ?? null, query.limit + 1]);'),
+'The frozen query must be the actual production history statement, including its binding order.');
+const request = historyQuery(scenario.dataset.monitorA, { state: 'FAILED', limit: '20' });
+assert.deepEqual([request.monitorId, scenario.dataset.alice, request.state,
+  request.after?.finishedAt ?? null, request.after?.id ?? null, request.limit + 1], scenario.query.binds);
+
+const output = 'output/phase-1/e20';
+await mkdir(output, { recursive: true });
+await writeFile(`${output}/${mode}.started.json`, JSON.stringify({ head: git('rev-parse', 'HEAD'), hashes }) + '\n', { flag: 'wx' });
+const baseline = mode === 'baseline' ? null : JSON.parse(await readFile(mode === 'confirm'
+  ? `${output}/baseline.json` : new URL('baseline.json', import.meta.url), 'utf8'));
+if (baseline) { assert.equal(baseline.result, 'CAPTURED'); assert.deepEqual(baseline.hashes, hashes); }
+const config = { ...databaseConfig(), schema: mode === 'verify' ? scenario.verificationSchema : scenario.schema };
+const pool = databasePool(config);
+const began = performance.now();
+const report = { mode, head: git('rev-parse', 'HEAD'), hashes, schema: config.schema, result: 'NOT_RUN',
+  sql, binds: scenario.query.binds, continuationBinds: scenario.query.continuationBinds,
+  datasetCreations: mode === 'confirm' ? 0 : 1, analyzeCommands: mode === 'confirm' ? 0 : 1 };
+let owned = mode === 'confirm' && baseline.schema === config.schema && baseline.cleanup.schemaRetained;
+function nodes(plan) { return [plan, ...(plan.Plans ?? []).flatMap(nodes)]; }
+const id = ordinal => '20000000-0000-4000-a000-' + String(ordinal).padStart(12, '0');
+try {
+  await requireFreePorts();
+  if (mode !== 'confirm') { await pool.query(`CREATE SCHEMA ${schemaIdentifier(config.schema)}`); owned = true; }
+  assert.ok(owned, 'Only the schema created by this frozen fixture may be used.');
+  report.appliedMigrations = await migrate(config);
+  if (mode !== 'confirm') await pool.query(await readFile(new URL('seed.sql', import.meta.url), 'utf8'));
+  report.serverVersion = (await pool.query('SHOW server_version')).rows[0].server_version;
+  report.distribution = (await pool.query(`SELECT m.id, m.owner_user_id, count(c.id)::int AS runs,
+    count(c.id) FILTER (WHERE c.state = 'FAILED')::int AS failed
+    FROM monitors m JOIN check_runs c ON c.monitor_id = m.id GROUP BY m.id ORDER BY m.id`)).rows;
+  assert.deepEqual(report.distribution.map(row => row.runs), scenario.dataset.distribution);
+  assert.deepEqual(report.distribution.map(row => row.failed), scenario.dataset.distribution.map(n => n / 100));
+  assert.deepEqual(report.distribution.map(row => row.owner_user_id), [scenario.dataset.alice, ...Array(9).fill(scenario.dataset.bob)]);
+  report.datasetDigest = (await pool.query("SELECT md5(string_agg(row_to_json(c)::text, E'\\n' ORDER BY id)) AS digest FROM check_runs c")).rows[0].digest;
+  if (baseline) assert.equal(report.datasetDigest, baseline.datasetDigest);
+  report.migrations = (await pool.query('SELECT version, checksum FROM schema_migrations ORDER BY version')).rows;
+  report.indexes = (await pool.query(`SELECT indexname, indexdef, pg_relation_size(indexname::regclass)::int AS bytes
+    FROM pg_indexes WHERE schemaname = current_schema() AND tablename = 'check_runs' ORDER BY indexname`)).rows;
+  report.plan = (await pool.query(`${scenario.query.explain} ${sql}`, scenario.query.binds)).rows[0]['QUERY PLAN'];
+  const all = nodes(report.plan[0].Plan);
+  const scans = all.filter(node => node['Relation Name'] === 'check_runs');
+  report.summary = scans.map(node => ({ type: node['Node Type'], index: node['Index Name'], condition: node['Index Cond'],
+    filter: node.Filter ?? null, returned: node['Actual Rows'] * node['Actual Loops'],
+    filtered: (node['Rows Removed by Filter'] ?? 0) * node['Actual Loops'],
+    sharedHitBlocks: node['Shared Hit Blocks'], sharedReadBlocks: node['Shared Read Blocks'] }));
+  report.firstRows = (await pool.query(sql, scenario.query.binds)).rows;
+  report.continuationRows = (await pool.query(sql, scenario.query.continuationBinds)).rows;
+  report.foreignRows = (await pool.query(sql, scenario.query.binds.map((value, i) => i === 1 ? scenario.dataset.bob : value))).rows;
+  assert.deepEqual(report.firstRows.map(row => row.id), Array.from({ length: 21 }, (_, n) => id(90000 - n * 100)));
+  assert.deepEqual(report.continuationRows.map(row => row.id), Array.from({ length: 21 }, (_, n) => id(88000 - n * 100)));
+  assert.deepEqual(report.foreignRows, []);
+  if (baseline) {
+    const serialized = JSON.parse(JSON.stringify(report));
+    assert.deepEqual(serialized.firstRows, baseline.firstRows);
+    assert.deepEqual(serialized.continuationRows, baseline.continuationRows);
+    assert.ok(!all.some(node => node['Node Type'].includes('Sort')));
+    assert.equal(scans.length, 1);
+    assert.match(scans[0]['Node Type'], /Index.*Scan/);
+    assert.equal(report.summary[0].filtered, 0);
+    assert.ok(report.summary[0].returned <= 21);
+    report.result = 'PASS';
+  } else {
+    report.result = 'CAPTURED';
+    report.existingSufficient = scans.length === 1 && /Index.*Scan/.test(scans[0]['Node Type']) &&
+      !all.some(node => node['Node Type'].includes('Sort')) && report.summary[0].filtered === 0 && report.summary[0].returned <= 21;
+  }
+} catch (error) {
+  report.result = 'FAILED'; report.failure = error.message; process.exitCode = 1;
+} finally {
+  const retain = mode === 'baseline' && report.result === 'CAPTURED';
+  if (owned && !retain) await pool.query(`DROP SCHEMA ${schemaIdentifier(config.schema)} CASCADE`);
+  await pool.end();
+  await requireFreePorts();
+  report.cleanup = { schemaRetained: owned && retain, schemaDropped: owned && !retain, portsFree: true };
+  report.durationMs = Math.round(performance.now() - began);
+  await writeFile(`${output}/${mode}.json`, JSON.stringify(report, null, 2) + '\n', { flag: 'wx' });
+  console.log(JSON.stringify({ mode, result: report.result, durationMs: report.durationMs,
+    summary: report.summary, existingSufficient: report.existingSufficient, failure: report.failure, cleanup: report.cleanup }));
+}
diff --git a/evidence/phase-1/E20/scenario.json b/evidence/phase-1/E20/scenario.json
new file mode 100644
index 0000000..fce7886
--- /dev/null
+++ b/evidence/phase-1/E20/scenario.json
@@ -0,0 +1,58 @@
+{
+  "thread": "E20",
+  "profile": "phase-1",
+  "specRevision": "2ada57a71cd34fa2fae9809415c362a8bbfcdf02",
+  "start": "22e5606117d3a6fbf5653b4348d1b8deca7fc2df",
+  "schema": "e20_history",
+  "verificationSchema": "e20_verify",
+  "ports": [4311, 4312, 4313, 4314],
+  "dataset": {
+    "monitors": 10,
+    "checkRuns": 99000,
+    "monitorA": "20000000-0000-4000-8000-000000000001",
+    "alice": "20000000-0000-4000-9000-000000000001",
+    "bob": "20000000-0000-4000-9000-000000000002",
+    "distribution": [90000, 1000, 1000, 1000, 1000, 1000, 1000, 1000, 1000, 1000],
+    "ownership": "Alice owns A; Bob owns the other nine Monitors.",
+    "ordinal": "Global ordinals 1..99000, contiguous by Monitor A..J.",
+    "id": "20000000-0000-4000-a000- plus the ordinal as 12 zero-padded decimal digits.",
+    "state": "Every 100th global ordinal is FAILED/503/HTTP_STATUS; all others SUCCEEDED/200/null.",
+    "failed": 990,
+    "baseTimestamp": "2026-08-28T00:00:00.000Z",
+    "time": "queued_at, started_at and finished_at are baseTimestamp + ordinal milliseconds; latency_ms=0.",
+    "otherFields": "MANUAL; scheduled_for, request_user_id, idempotency_key, worker_id and lease_expires_at are null. Monitors disabled; no worker or outbound HTTP.",
+    "authentication": "Non-login fixture users only; no credential, session or CSRF material. The existing E07 API regression exercises actual authentication separately."
+  },
+  "query": {
+    "source": "server/app.ts GET /monitors/:id/checks",
+    "file": "history.sql",
+    "publicRequest": "/monitors/20000000-0000-4000-8000-000000000001/checks?state=FAILED&limit=20",
+    "binds": ["20000000-0000-4000-8000-000000000001", "20000000-0000-4000-9000-000000000001", "FAILED", null, null, 21],
+    "continuationBinds": ["20000000-0000-4000-8000-000000000001", "20000000-0000-4000-9000-000000000001", "FAILED", "2026-08-28T00:01:28.100Z", "20000000-0000-4000-a000-000000088100", 21],
+    "firstOrdinals": "90000,89900,...,88000 (21 database rows; public first 20 end at 88100)",
+    "continuationOrdinals": "88000,87900,...,86000 (21 database rows)",
+    "explain": "EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)",
+    "sequence": "Validate source SQL/binds; seed and ANALYZE once; count/digest; EXPLAIN first page once; execute first/continuation/foreign-owner SELECTs once each. Confirmation repeats only count/digest and these same reads after the selected migration.",
+    "cache": "No cache flush or cold-cache claim. Count/digest warms table pages before each plan; use plan rows/filters/buffers, not absolute latency."
+  },
+  "selection": {
+    "existingIndex": "check_runs_monitor_finished_id_idx",
+    "candidate": "CREATE INDEX check_runs_failed_history_idx ON check_runs (monitor_id, finished_at DESC, id DESC) WHERE state = 'FAILED' AND finished_at IS NOT NULL",
+    "rule": "Keep candidate count 0 when the existing ordered plan returns at most 21 CheckRuns without filtering unrelated states. Otherwise examine only the declared partial index (candidate count 1).",
+    "finalCriteria": "Ordered CheckRun index scan, no Sort or CheckRun sequential scan, at most 21 candidate rows, zero CheckRun rows removed by filter; identical first/continuation rows and empty foreign-owner result. Buffer values are recorded, not tuned."
+  },
+  "budget": {
+    "datasets": 1,
+    "maxCandidateIndexes": 3,
+    "plannedCandidates": 1,
+    "baselineInvocations": 1,
+    "authorConfirmations": 1,
+    "rootVerification": "One verify mode on an independently owned schema with this identical fixture; no baseline replay or index drop/recreate.",
+    "concurrency": 1,
+    "loadRuns": 0,
+    "automaticRetries": 0,
+    "parameterSweeps": 0,
+    "absoluteLatencySearch": false,
+    "defaultRegressionRegistration": false
+  }
+}
diff --git a/evidence/phase-1/E20/seed.sql b/evidence/phase-1/E20/seed.sql
new file mode 100644
index 0000000..10ea4b0
--- /dev/null
+++ b/evidence/phase-1/E20/seed.sql
@@ -0,0 +1,25 @@
+INSERT INTO users (id, username, password_hash) VALUES
+  ('20000000-0000-4000-9000-000000000001', 'alice-e20', '!disabled-e20-fixture'),
+  ('20000000-0000-4000-9000-000000000002', 'bob-e20', '!disabled-e20-fixture');
+
+INSERT INTO monitors (id, name, url, interval_seconds, enabled, created_at, updated_at, owner_user_id)
+SELECT ('20000000-0000-4000-8000-' || lpad(n::text, 12, '0'))::uuid,
+  'Monitor ' || chr(64 + n), 'http://public.e20.test/ok', 60, false,
+  '2026-08-28T00:00:00.000Z'::timestamptz, '2026-08-28T00:00:00.000Z'::timestamptz,
+  CASE WHEN n = 1 THEN '20000000-0000-4000-9000-000000000001'::uuid
+    ELSE '20000000-0000-4000-9000-000000000002'::uuid END
+FROM generate_series(1, 10) AS n;
+
+INSERT INTO check_runs
+  (id, monitor_id, trigger, state, http_status, latency_ms, failure_reason, queued_at, started_at, finished_at)
+SELECT ('20000000-0000-4000-a000-' || lpad(n::text, 12, '0'))::uuid,
+  ('20000000-0000-4000-8000-' || lpad((CASE WHEN n <= 90000 THEN 1 ELSE 2 + (n - 90001) / 1000 END)::text, 12, '0'))::uuid,
+  'MANUAL', CASE WHEN n % 100 = 0 THEN 'FAILED' ELSE 'SUCCEEDED' END,
+  CASE WHEN n % 100 = 0 THEN 503 ELSE 200 END, 0,
+  CASE WHEN n % 100 = 0 THEN 'HTTP_STATUS' ELSE NULL END,
+  '2026-08-28T00:00:00.000Z'::timestamptz + n * interval '1 millisecond',
+  '2026-08-28T00:00:00.000Z'::timestamptz + n * interval '1 millisecond',
+  '2026-08-28T00:00:00.000Z'::timestamptz + n * interval '1 millisecond'
+FROM generate_series(1, 99000) AS n;
+
+ANALYZE monitors, check_runs;


## `perf: index sparse failed history without changing pagination`

diff --git a/TRACK.md b/TRACK.md
index 415a6ac..f6b8b0f 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -706,3 +706,53 @@ an actual worker, proving persisted ABORTED and same-key replay with no guard
 request. No public or metadata endpoint is contacted. Existing live redirect
 and direct-refusal consumers adopt E12 semantics; all earlier frozen evidence
 is retained. E11 recovery scenarios were not rerun as E12 author smoke tests.
+
+## E20 FAILED history query plan
+
+The frozen dataset has 99,000 terminal CheckRuns: Monitor A has 90,000 and each
+of the other nine has 1,000; every hundredth row is FAILED/503. The observer
+checks the exact SQL and binding order in `server/app.ts`. Public `limit=20`
+still fetches 21 rows for the existing look-ahead cursor. Owner checks, optional
+state filtering and `(finished_at, id)` descending seek ordering are unchanged.
+
+| CheckRun scan | E07 index baseline | E20 partial index |
+| --- | ---: | ---: |
+| Returned rows | 21 | 21 |
+| Rows removed by filter | 1,980 | 0 |
+| Shared buffer hits / reads | 60 / 0 | 21 / 2 |
+| Sort | None | None |
+
+Migration `010_failed_history_index.sql` adds only
+`(monitor_id, finished_at DESC, id DESC) WHERE state = 'FAILED' AND finished_at IS NOT NULL`.
+The existing E07 index remains for All and other-state queries. This is one
+candidate, not a latency search: both plans use the same schema, fixed seed and
+one ANALYZE; first/continuation records and the dataset digest are identical.
+The actual parameterized, unnamed query selects the partial index; this does
+not claim that a generic prepared plan can prove every partial-index predicate.
+[PostgreSQL partial indexes](https://www.postgresql.org/docs/17/indexes-partial.html)
+and [index ordering](https://www.postgresql.org/docs/17/indexes-ordering.html)
+describe these requirements.
+
+The new index occupies 81,920 bytes for 990 matching rows in this fixture.
+It adds index maintenance for FAILED completion/deletion and predicate checks
+on writes; it is not free. SUCCEEDED, ABORTED and active rows get no entries in
+this index. The broader E07 index still costs storage and maintenance, but is
+not redundant because it covers the other history queries. Migration 010 uses
+the existing transactional migration mechanism; schedule its index build with
+the normal migration maintenance window. No write-throughput or cold-cache
+claim is made; [EXPLAIN buffers](https://www.postgresql.org/docs/17/using-explain.html)
+and actual rows, not elapsed-time minima, are the comparison.
+
+`evidence/phase-1/E20` contains the immutable seed/query/observer, raw baseline
+and confirmation, and invocation/candidate ledger. The focused regression is
+`node --test --test-concurrency=1 test/persistence.test.ts test/storage-history.test.ts`;
+the existing E07 test retains ties, no gaps/duplicates, insertion continuation,
+filters, cursor validation and owner isolation. Only expected migration names
+and count in the persistence consumer changed.
+
+The capped plan scenario is deliberately absent from npm/CI regression sweeps.
+The authorized independent final check is
+`fnm exec --using 24.19.0 node evidence/phase-1/E20/observe.mjs verify`.
+It refuses an existing `e20_verify` schema or an existing invocation marker,
+applies the final migration chain, reuses the same fixed dataset, and removes
+only its own schema. Do not replay the archived baseline or search new indexes.
diff --git a/server/migrate.ts b/server/migrate.ts
index 28a35a5..3e4767d 100644
--- a/server/migrate.ts
+++ b/server/migrate.ts
@@ -5,7 +5,7 @@ import { databaseConfig, databasePool, schemaIdentifier } from './database.ts';
 import type { DatabaseConfig } from './database.ts';
 
 export async function migrationFiles() {
-  const names = ['001_monitors.sql', '002_check_runs.sql', '003_sessions.sql', '004_monitor_ownership.sql', '005_check_history_index.sql', '006_check_queue.sql', '007_check_ownership.sql', '008_check_lease.sql', '009_outbound_policy_result.sql'];
+  const names = ['001_monitors.sql', '002_check_runs.sql', '003_sessions.sql', '004_monitor_ownership.sql', '005_check_history_index.sql', '006_check_queue.sql', '007_check_ownership.sql', '008_check_lease.sql', '009_outbound_policy_result.sql', '010_failed_history_index.sql'];
   return Promise.all(names.map(async (version) => {
     const sql = await readFile(new URL(`./migrations/${version}`, import.meta.url), 'utf8');
     return { version, sql, checksum: createHash('sha256').update(sql).digest('hex') };
diff --git a/server/migrations/010_failed_history_index.sql b/server/migrations/010_failed_history_index.sql
new file mode 100644
index 0000000..edadc77
--- /dev/null
+++ b/server/migrations/010_failed_history_index.sql
@@ -0,0 +1,4 @@
+-- FAILED history is sparse; keep the E07 index for unfiltered/other-state reads.
+CREATE INDEX check_runs_failed_history_idx
+  ON check_runs (monitor_id, finished_at DESC, id DESC)
+  WHERE state = 'FAILED' AND finished_at IS NOT NULL;
diff --git a/test/persistence.test.ts b/test/persistence.test.ts
index d5ddfcc..daad639 100644
--- a/test/persistence.test.ts
+++ b/test/persistence.test.ts
@@ -54,9 +54,9 @@ test('E03 migration CLI applies the fresh chain and a repeated command changes n
   try {
     const options = { cwd: process.cwd(), env: { ...process.env, DATABASE_URL: config.connectionString, DATABASE_SCHEMA: config.schema }, timeout: 10_000 };
     const first = await execute(process.execPath, ['server/migrate.ts'], options);
-    assert.deepEqual(JSON.parse(first.stdout).applied, [...authScenario.additionalAssertions.expectedMigrationFiles, '004_monitor_ownership.sql', '005_check_history_index.sql', '006_check_queue.sql', '007_check_ownership.sql', '008_check_lease.sql', '009_outbound_policy_result.sql']);
+    assert.deepEqual(JSON.parse(first.stdout).applied, [...authScenario.additionalAssertions.expectedMigrationFiles, '004_monitor_ownership.sql', '005_check_history_index.sql', '006_check_queue.sql', '007_check_ownership.sql', '008_check_lease.sql', '009_outbound_policy_result.sql', '010_failed_history_index.sql']);
     const before = (await pool.query('SELECT version, checksum, applied_at FROM schema_migrations ORDER BY version')).rows;
-    assert.equal(before.length, authScenario.additionalAssertions.expectedMigrationFiles.length + 6);
+    assert.equal(before.length, authScenario.additionalAssertions.expectedMigrationFiles.length + 7);
     for (const row of before) { assert.match(row.checksum, /^[0-9a-f]{64}$/); assert.ok(row.applied_at instanceof Date); }
     const second = await execute(process.execPath, ['server/migrate.ts'], options);
     assert.deepEqual(JSON.parse(second.stdout).applied, scenario.additionalAssertions.repeatMigrationExpectedApplied);
@@ -263,7 +263,7 @@ test('E05 upgrades existing rows only with explicit owner designation and preser
     finally { await oldApp.close(); }
 
     // Deliberately designate the second account, never the first row returned.
-    assert.deepEqual(await migrate(config, users[1].username), ['004_monitor_ownership.sql', '005_check_history_index.sql', '006_check_queue.sql', '007_check_ownership.sql', '008_check_lease.sql', '009_outbound_policy_result.sql']);
+    assert.deepEqual(await migrate(config, users[1].username), ['004_monitor_ownership.sql', '005_check_history_index.sql', '006_check_queue.sql', '007_check_ownership.sql', '008_check_lease.sql', '009_outbound_policy_result.sql', '010_failed_history_index.sql']);
     const owner = (await pool.query('SELECT id FROM users WHERE username = $1', [users[1].username])).rows[0].id;
     const after = (await pool.query('SELECT * FROM monitors ORDER BY id')).rows;
     assert.deepEqual(after.map(({ owner_user_id: _owner, ...row }) => row), before.monitors);


