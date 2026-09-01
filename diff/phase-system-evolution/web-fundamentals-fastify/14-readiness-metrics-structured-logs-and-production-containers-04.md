## `docs(e24): preserve operations evidence and repair history`

diff --git a/TRACK.md b/TRACK.md
index f6b8b0f..0faf12b 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -16,6 +16,9 @@ A transport failure records `FAILED/CONNECTION_FAILURE` or `FAILED/TIMEOUT` with
 `httpStatus: null`. E09 shows an active CheckRun, or the latest terminal result,
 on each Monitor; bounded history retains completed results. A separate worker
 now creates interval-based scheduled intents and executes persisted checks.
+Phase-1 E24 adds API/worker liveness, PostgreSQL readiness, bounded metrics,
+structured logs, and production API/worker/Next containers. The E24 section below
+describes the current container and CI commands.
 
 ## Fixed versions
 
@@ -94,7 +97,8 @@ npm run test:outbound
 Functional tests own fixed loopback ports 4311, 4312 and 4314; stop manual fixture
 processes before running them. Port 4314 is a controlled destination that must
 never receive a check. E03 uses PostgreSQL on loopback port 15431. E04 adds login
-and E09 adds one check worker; there is no Redis, Kafka, or production application container.
+and E09 adds one check worker. There is no Redis or Kafka; E24 adds the production
+application containers described below.
 The E2E command owns ports 4311–4313 and starts fresh fixture/API/Next processes;
 stop manual development processes first. It uses one Chromium worker, no test
 retries, and the fixed Monitor fixture in `evidence/E01/scenario.json`.
@@ -137,13 +141,16 @@ validation uses standard JavaScript and Fastify's existing typed error classes.
 
 ## Basic CI
 
-`.github/workflows/check.yml` runs the type, unit, functional, Chromium E2E, and
-Next build checks, plus PostgreSQL migration and persistence tests. It selects Node from `.node-version`, installs npm 11.17.0,
+`.github/workflows/check.yml` now separates unit, integration, browser, and
+container jobs. It retains type/unit, PostgreSQL/functional/worker/outbound,
+and production-build browser checks, then adds production container smoke.
+It selects Node from `.node-version`, installs npm 11.17.0,
 and installs dependencies from the lockfile. CI actions are pinned to the full
 commits for [checkout 4.2.2](https://github.com/actions/checkout/releases/tag/v4.2.2)
 and [setup-node 4.4.0](https://github.com/actions/setup-node/releases/tag/v4.4.0).
-Local invocation outcomes and counts are recorded in `evidence/E01/verification.json`;
-a hosted CI run is not claimed by those local results.
+Historical E01 outcomes remain in `evidence/E01/verification.json`; current E24
+local outcomes and reuse limits are in `evidence/phase-1/E24/verification.json`.
+These local results do not claim a hosted CI execution.
 
 ## E03 PostgreSQL lifecycle and mapping
 
@@ -161,7 +168,7 @@ The default connection is `postgresql://monitor@127.0.0.1:15431/monitor` and the
 application schema is `public`. `DATABASE_URL` and `DATABASE_SCHEMA` can select a
 different disposable database/schema; do not commit a DSN containing credentials.
 `npm run db:down` stops only this project and retains its named data volume.
-There is no application container.
+E24's application containers attach to this owned PostgreSQL network separately.
 
 Run `npm run db:migrate` before starting the API. It applies `001_monitors.sql`,
 `002_check_runs.sql`, and E04's `003_sessions.sql`, on one explicit `pg` client within a DDL
@@ -756,3 +763,73 @@ The authorized independent final check is
 It refuses an existing `e20_verify` schema or an existing invocation marker,
 applies the final migration chain, reuses the same fixed dataset, and removes
 only its own schema. Do not replay the archived baseline or search new indexes.
+
+## Phase-1 E24 operations and production containers
+
+The API on port 4312 and worker on port 4314 expose unauthenticated operational
+GET/HEAD routes. Keep these loopback bindings and the existing local PostgreSQL
+trust policy; this setup does not configure a public deployment.
+
+| Route | Meaning |
+| --- | --- |
+| `/health/live` | Process can answer HTTP; stays 200 during PostgreSQL absence |
+| `/health/ready` | PostgreSQL query succeeds: 200; otherwise 503 |
+| `/metrics` | Bounded scalar metrics; available during the database outage |
+| API `/health` | Existing liveness alias, retained |
+
+Authority remains in PostgreSQL. An outage rejects the frozen manual submission
+with the existing safe HTTP 500 error, accepts no durable intent, creates no
+outbound result, and leaves API/worker processes alive. Queue age is omitted
+when it cannot be read, instead of reporting a fabricated zero.
+
+HTTP metrics are `http_request_duration_seconds_sum`,
+`http_request_duration_seconds_count`, and `http_errors_total`. Labels use only
+fixed role, route template, bounded method, and HTTP status. Both roles publish
+`postgres_ready` and authoritative `check_queue_age_seconds`; the worker also
+publishes `worker_active`, `worker_claims_total`, `worker_recovery_runs_total`,
+and `worker_recovered_checks_total`. Recovery scans and recovered rows are
+separate observations; this healthy scenario does not repeat E11 crash tests.
+
+API/worker JSON logs carry process ID, PID, role, and event. Request IDs connect
+HTTP completion to admission, and the CheckRun ID connects admission, claim,
+and completion. URLs, bodies, cookies, CSRF tokens, and idempotency keys are not
+log fields or metric labels. The evidence scans all role logs and metric text
+for runtime sentinels, retaining only safe counts and validated metric text.
+
+With Node 24.19.0/npm 11.17.0 active and ports 4311–4314 free:
+
+```sh
+npm run db:up
+npm run db:migrate
+docker compose -f compose.production.yaml build api frontend
+docker compose -f compose.production.yaml up -d --no-build
+```
+
+The pinned Node image runs API and worker TypeScript with Node's native stripping,
+and Next's standalone `server.js` with copied `.next/static` assets. All three
+run as UID 1000, with Node as PID 1. `API_ORIGIN` is set during both Next build
+and runtime for the real server proxy. Stop only application containers with
+`docker compose -f compose.production.yaml down`; do not remove PostgreSQL volumes.
+
+Next 16.3.3 waits for HTTP closure, server cleanup, and trace flushing, then
+normally exits 143 on SIGTERM. The frontend image verifies the exact shipped
+`start-server.js` hash before adapting only that completed SIGTERM branch to
+exit 0. Other exit paths, cleanup awaits, dependency pins, and `node server.js`
+remain unchanged. A changed upstream hash stops the build and requires review.
+This image policy does not modify host `node_modules` or `next start` behavior.
+
+`npm run test:container` is the CI smoke without the capped PostgreSQL stop.
+It uses an explicit test overlay for the existing loopback HTTP fixture;
+API/worker images default to `NODE_ENV=production`, while this overlay selects
+`NODE_ENV=test` for the fixed fixture exception. Frontend stays production.
+The full E24 gate uses `node test/container-smoke.mjs full <authorized-actor>`
+only under its recorded budget; do not replay baseline, repair, E11, or E20
+scenarios. Invocation markers refuse reused actor labels.
+
+The last repair's full gate passed: readiness survived one PostgreSQL stop and
+restore, owner-isolated browser/static/server-route checks passed, metric series
+stayed 35→35 across ten distinct missing IDs, and all three SIGTERM exits were
+0 without forced kills. Its actual frontend exit took 151ms, within 30000ms.
+The two failed predecessors and their raw limitations remain in
+`evidence/phase-1/E24/results/`; see the adjacent README and verification ledger.
+Independent root acceptance is pending. Hosted CI was not executed locally.
diff --git a/evidence/phase-1/E24/README.md b/evidence/phase-1/E24/README.md
new file mode 100644
index 0000000..09d20dc
--- /dev/null
+++ b/evidence/phase-1/E24/README.md
@@ -0,0 +1,96 @@
+# Phase-1 E24 evidence
+
+Spec revision: `2ada57a71cd34fa2fae9809415c362a8bbfcdf02`.
+Start: `9d0a1974e1f279146917b840e69bbee19dcfc0c4`.
+This is attempt 3, the second and last bounded repair. Author acceptance passed;
+independent root acceptance and the profile progress tag remain pending.
+
+## Frozen inputs
+
+| File | SHA-256 |
+| --- | --- |
+| `scenario.json` | `07b5b3e2b4bd1cfc456c415134adce61f05dfdd729fe803968b32a4d813c5847` |
+| `fixture.ts` | `10d311f2e2af33063d09add435ed282b20341e2b87eeffa10b6287dce61ca784` |
+| `baseline.mjs` | `0a24805b65454cddc543d27708aec68092b34e28b67ca6ba605e3bc6602c79d6` |
+
+The sole unchanged-product baseline used actual Fastify `app.inject`, not an
+external HTTP request: readiness returned 401 while existing liveness returned
+200. It took 645ms and performed no container build, PostgreSQL stop, or outbound
+request. Product bytes matched START; the baseline was not repeated.
+
+## Actual full-run history
+
+| Actor | Result | Native duration | Decisive observation |
+| --- | --- | ---: | --- |
+| `author` | FAILED | 6930ms | Installed Compose rejected `start --wait`; PostgreSQL restoration was not completed in that run |
+| `repair1` | FAILED | 9183ms | Restore command worked; frontend failed the unchanged clean-exit assertion |
+| `repair2` | PASS | 9725ms | Readiness/outage, worker, browser, metrics, safe logs, signals, and cleanup passed |
+
+Repair1's frontend exit value was not retained: its assertion ran before the
+observation was recorded. Do not report a historical exit of 143. The pinned
+Next source contains that exit value after graceful cleanup; this is a source
+diagnosis, not a recovered runtime measurement. The final log/sentinel stage
+was not reached in either failed run.
+
+The repair2 observer records actual per-role exit values and durations before
+the unchanged assertion. All three were code 0, with no forced kill: API 286ms,
+worker 190ms, frontend 151ms. The frozen limits remain 3000/3000/30000ms.
+UID 1000, actual Node PID 1, configured commands, image IDs, and the production
+frontend build ID are retained. Next changes its process title; the raw observed
+`/proc/1/cmdline` is kept separately from Docker's configured `node server.js`.
+
+The repair2 run retained 35 metric series before and after ten distinct missing
+IDs, with the exact 404 increase of ten. It scanned 15 runtime sentinels with
+zero leaks, correlated the same CheckRun across API/worker events, and observed
+50 API plus 18 worker JSON lines. Raw runtime logs, credentials, screenshots,
+and browser traces were not retained. After the one PostgreSQL stop, both roles
+stayed live 200/ready 503; the unsafe submission returned 500 and created no new
+authority or outbound request. Restoration returned readiness 200 and preserved
+all five CheckRuns. Owned containers/network/schema were removed and ports freed.
+
+## Narrow repair and reused verification
+
+Repair2 first verified the 17 adopted files, exact repair1 observer WIP, all three
+frozen inputs, 19 preserved outputs, and 42 original build-context files. It
+committed the unchanged two-line repair1 correction separately. Root then
+reviewed and authorized the exact frontend/observer source hashes before any
+repair2 build or scenario.
+
+Only the frontend image's copied Next `start-server.js` is adapted. Its full
+shipped SHA-256 is checked before changing the sole completed SIGTERM exit from
+143 to 0. HTTP close, server close, trace flush, other exits, PID 1, UID, command,
+and pinned dependencies remain unchanged. This is a documented image-specific
+downstream change, not a relaxation of the observer or frozen scenario.
+
+The one repair2 frontend build exited 0 in 2520ms. `npm ci`, `npm run build`, and
+the standalone/static copies were cached; only the new guarded adaptation RUN
+executed (BuildKit 0.2s). The complete actual build output and before/after image
+IDs are retained. The backend image was unchanged. Typecheck, 22 unit tests,
+15 functional tests, Compose configuration, and backend build evidence are
+reused by exact source/dependency hashes; none was rerun during this repair.
+
+The original functional gate's first tool output was truncated at 6500 tokens
+(26023 original tokens). Its stored chunks are exactly what was received; the
+missing routine logs cannot be reconstructed. The final 15-pass/0-fail summary
+and duration were received in full. No hosted CI execution is claimed.
+
+## Artifacts and budget
+
+`results/` contains byte-identical copies of the native output files, including
+failed runs, cleanup attempts, complete repair2 console output, and reuse/source
+audits. `verification.json` maps every copy to its original output path and hash,
+records the cumulative budget, and distinguishes author PASS from pending root
+acceptance. Existing output files and all earlier evidence were left unchanged.
+
+There was one baseline and three full author/repair scenarios, with two failures
+and no automatic retries. Each full scenario spent one PostgreSQL stop. The
+first failed restore required one separate successful cleanup restore; its
+permission-denied preflight is preserved and is not another executed restore.
+Both repair slots are used. There were no load runs, sweeps, exploratory signal
+runs, E11 crash reruns, or E20 dataset reruns.
+
+CI has separate unit, integration, browser, and container jobs. Its container
+smoke excludes the capped dependency stop. The only remaining full invocation
+is root's independently authorized `fnm exec --using 24.19.0 node
+test/container-smoke.mjs full root`; it creates its own exclusive actor marker.
+Do not delete markers or rerun archived actors.
diff --git a/evidence/phase-1/E24/results/author-pre-container-invocations.json b/evidence/phase-1/E24/results/author-pre-container-invocations.json
new file mode 100644
index 0000000..dcb7b26
--- /dev/null
+++ b/evidence/phase-1/E24/results/author-pre-container-invocations.json
@@ -0,0 +1,135 @@
+{
+  "thread": "E24",
+  "profile": "phase-1",
+  "attempt": 1,
+  "observerChanges": "Source-only parent review before any gate: syntax brace, configured Cmd versus actual PID1 process title, post-scan metrics persistence, independent once-only cleanup. No failed acceptance invocation or repair.",
+  "runs": [
+    {
+      "gate": "compose-config",
+      "command": "/usr/bin/time -p docker compose --project-name wse-fundamentals-e24 -f compose.production.yaml -f evidence/phase-1/E24/compose.test.yaml config --quiet",
+      "exit": 0,
+      "realSeconds": 0.15,
+      "consoleComplete": true,
+      "chunks": [
+        {
+          "chunk_id": "653066",
+          "wall_time_seconds": 0.036685417,
+          "exit_code": 0,
+          "original_token_count": 8,
+          "output": "real 0.15\nuser 0.04\nsys 0.04\n"
+        }
+      ]
+    },
+    {
+      "gate": "typecheck",
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run typecheck",
+      "exit": 0,
+      "realSeconds": 2.18,
+      "consoleComplete": true,
+      "chunks": [
+        {
+          "chunk_id": "0f6322",
+          "wall_time_seconds": 1.002458167,
+          "session_id": 25630,
+          "original_token_count": 42,
+          "output": "\n> monitor-fundamentals-fastify@0.1.0 typecheck\n> NEXT_TELEMETRY_DISABLED=1 next typegen && tsc --noEmit\n\nGenerating route types...\n✓ Types generated successfully\n"
+        },
+        {
+          "chunk_id": "e4dedc",
+          "wall_time_seconds": 0.000003,
+          "exit_code": 0,
+          "original_token_count": 8,
+          "output": "real 2.18\nuser 3.39\nsys 0.28\n"
+        }
+      ]
+    },
+    {
+      "gate": "unit",
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:unit",
+      "exit": 0,
+      "realSeconds": 1.28,
+      "pass": 22,
+      "fail": 0,
+      "consoleComplete": true,
+      "chunks": [
+        {
+          "chunk_id": "846dbb",
+          "wall_time_seconds": 1.001664208,
+          "session_id": 94915,
+          "original_token_count": 316,
+          "output": "\n> monitor-fundamentals-fastify@0.1.0 test:unit\n> node --test test/unit.test.ts\n\n✔ E24 operational series normalize inputs and omit unavailable queue age (1.992541ms)\n✔ E10 manual identity is bounded printable ASCII without whitespace or coercion (0.514542ms)\n✔ E10 browser recognizes semantic conflict only with its409 category (2.168958ms)\n✔ E08 RSC caller follows only one same-route framework canonicalization (0.453125ms)\n✔ E08 server and browser history keys match without dropping invalid query parameters (0.361792ms)\n✔ E08 unauthenticated initial payloads never share mutable state between requests (0.100875ms)\n✔ a path on the configured fixture origin is allowed (0.159916ms)\n✔ a different fixture port or host is refused before networking (0.418334ms)\n✔ credentials and another protocol cannot bypass the fixture boundary (0.106875ms)\n✔ browser classifies API failures by code and status, independently of server prose (0.629333ms)\n✔ browser treats malformed envelopes and status/code mismatches as a safe service failure (0.522083ms)\n✔ browser consumes data envelopes without losing false, zero or null wire values (0.183541ms)\n✔ browser transport rejection has a stable fallback independent of exception text (0.076125ms)\n"
+        },
+        {
+          "chunk_id": "4d0b64",
+          "wall_time_seconds": 0.000003042,
+          "exit_code": 0,
+          "original_token_count": 259,
+          "output": "✔ E04 salted scrypt hashes verify passwords without storing their plaintext (1004.782042ms)\n✔ E04 login input is bounded at runtime and never trims or coerces a password (0.395834ms)\n✔ E04 a single bounded cookie identifier is required, including duplicate and malformed rejection (0.163333ms)\n✔ E04 browser recognizes UNAUTHENTICATED only with HTTP 401 and ignores server prose (0.219333ms)\n✔ E05 CSRF evidence is bounded and tied to the current session without exposing the identifier (1.202542ms)\n✔ E05 browser recognizes FORBIDDEN only with HTTP 403 and ignores server prose (0.25125ms)\n✔ E07 history input has a finite default, maximum and terminal-state filter (0.491584ms)\n✔ E07 cursor preserves the timestamp/id tuple and rejects changed conditions (0.297625ms)\n✔ E07 cursor rejects malformed, oversized, noncanonical and invalid typed boundaries (0.329792ms)\nℹ tests 22\nℹ suites 0\nℹ pass 22\nℹ fail 0\nℹ cancelled 0\nℹ skipped 0\nℹ todo 0\nℹ duration_ms 1129.518584\nreal 1.28\nuser 1.26\nsys 0.06\n"
+        }
+      ]
+    },
+    {
+      "gate": "functional",
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:functional",
+      "exit": 0,
+      "realSeconds": 10.55,
+      "pass": 15,
+      "fail": 0,
+      "consoleComplete": false,
+      "limitation": "The first polling result was truncated by exec tool max_output_tokens=6500 (original_token_count=26023). Stored chunks are the exact received text; missing routine structured logs cannot be reconstructed. The final pass/fail summary and duration were received in full. No rerun.",
+      "chunks": [
+        {
+          "chunk_id": "a3a3ea",
+          "wall_time_seconds": 1.001727292,
+          "session_id": 51362,
+          "original_token_count": 35,
+          "output": "\n> monitor-fundamentals-fastify@0.1.0 test:functional\n> node --test --test-concurrency=1 test/functional.test.ts test/contracts.test.ts\n\n"
+        },
+        {
+          "chunk_id": "b35e19",
+          "wall_time_seconds": 5.002602417,
+          "session_id": 51362,
+          "original_token_count": 26023,
+          "output": "Warning: truncated output (original token count: 26023)\nTotal output lines: 394\n\n{\"time\":\"2026-08-28T07:02:30.359Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"7bea5757-69a5-4186-b0a6-3c071bfdc719\",\"route\":\"/auth/login\",\"method\":\"POST\",\"status\":200,\"durationMs\":267.597042}\n{\"time\":\"2026-08-28T07:02:30.487Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"826b7fba-fd35-4039-bdf5-1e6807b663c2\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":1.8842500000000655}\n{\"time\":\"2026-08-28T07:02:30.495Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"c236c46b-1cab-4279-ac97-fdaae3f594f7\",\"route\":\"/monitors\",\"method\":\"POST\",\"status\":400,\"durationMs\":1.823082999999997}\n✔ E02 fixed counterexample: blank name is INVALID_INPUT (1056.352417ms)\n{\"time\":\"2026-08-28T07:02:30.501Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"9272b123-9e7f-49c2-b880-b2d00165c0cc\",\"route\":\"/monitors\",\"method\":\"GET\",\"status\":200,\"durationMs\":2.4243330000001606}\n{\"time\":\"2026-08-28T07:02:30.504Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"f8198d32-b8cc-411e-bdfb-9bd42ebc5d0f\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":0.9747079999999642}\n{\"time\":\"2026-08-28T07:02:30.507Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"7b0a8261-6ef8-496e-8d01-1cafffb3f992\",\"route\":\"/monitors\",\"method\":\"POST\",\"status\":400,\"durationMs\":1.0557079999998678}\n{\"time\":\"2026-08-28T07:02:30.510Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"5b891e87-3de8-4b3d-ac1f-4280bae2c46d\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":1.040791999999783}\n{\"time\":\"2026-08-28T07:02:30.514Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"9d95d062-3996-41ad-a739-fc1ca50b7bef\",\"route\":\"/monitors\",\"method\":\"POST\",\"status\":400,\"durationMs\":0.9438749999999345}\n{\"time\":\"2026-08-28T07:02:30.517Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"c1b2f7b8-fc1f-42f9-9b7c-5049904186ad\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":1.2349999999999}\n{\"time\":\"2026-08-28T07:02:30.520Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"2ca4ea1b-88c4-4a8f-ae97-6fe7d81eb7e2\",\"route\":\"/monitors\",\"method\":\"POST\",\"status\":400,\"durationMs\":1.1718339999999898}\n{\"time\":\"2026-08-28T07:02:30.523Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"757d6e52-7313-4db7-9bd1-d135d3debb24\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":1.1460409999999683}\n{\"time\":\"2026-08-28T07:02:30.526Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"c2bb0f6a-b2ec-4242-9e62-db1e4d8e713b\",\"route\":\"/monitors\",\"method\":\"POST\",\"status\":400,\"durationMs\":1.1806669999998576}\n{\"time\":\"2026-08-28T07:02:30.530Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"08e914ac-9af3-4de2-891b-ed65e47a6ed9\",\"route\":\"/monitors\",\"method\":\"GET\",\"status\":200,\"durationMs\":1.8108330000000024}\n✔ E02 fixed malformed values are INVALID_INPUT and do not create Monitors (33.9525ms)\n{\"time\":\"2026-08-28T07:02:30.533Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"54bd8fc6-962f-40e1-aa4e-0bb18358fc59\",\"route\":\"/monitors\",\"method\":\"GET\",\"status\":200,\"durationMs\":2.0332499999999527}\n{\"time\":\"2026-08-28T07:02:30.535Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"7d459461-cff2-4fd4-98f6-d70f1c33fe49\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":0.8804999999999836}\n{\"time\":\"2026-08-28T07:02:30.538Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"9b89c18b-1e78-49dd-b9d0-ee9ab55f3df1\",\"route\":\"/monitors\",\"method\":\"POST\",\"status\":400,\"durationMs\":0.9540420000000722}\n{\"time\":\"2026-08-28T07:02:30.540Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"f6d49132-40d3-40c8-973e-d3823cd73fbd\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":0.9513750000000982}\n{\"time\":\"2026-08-28T07:02:30.543Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"24f0c655-4e9a-4ed0-87f5-db4e0b4dfecc\",\"route\":\"/monitors\",\"method\":\"POST\",\"status\":400,\"durationMs\":1.2634579999999005}\n{\"time\":\"2026-08-28T07:02:30.546Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"5dfbbf52-0999-4081-9a7b-1b4536730b2b\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":1.0264999999999418}\n{\"time\":\"2026-08-28T07:02:30.549Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"daf31568-57ef-4acf-b5d7-6f7c965a8b63\",\"route\":\"/monitors\",\"method\":\"POST\",\"status\":400,\"durationMs\":0.9695839999999407}\n{\"time\":\"2026-08-28T07:02:30.551Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"478a5ab2-c344-4355-992a-dbddea409b5f\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":1.0111669999998867}\n{\"time\":\"2026-08-28T07:02:30.554Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"b6cb20b7-9ef0-4154-bfa6-1858353b0179\",\"route\":\"/monitors\",\"method\":\"POST\",\"status\":400,\"durationMs\":1.13087500000006}\n{\"time\":\"2026-08-28T07:02:30.557Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"41925f34-3dfe-4a8f-a338-5bba7ffa9946\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":1.0045419999999012}\n{\"time\":\"2026-08-28T07:02:30.560Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"e975f0f4-3f71-4dd8-b2c0-0f95b88eae95\",\"route\":\"/monitors\",\"method\":\"POST\",\"status\":400,\"durationMs\":1.234375}\n{\"time\":\"2026-08-28T07:02:30.562Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"3bdf8f1e-10ac-45f2-afc7-6557e74665ca\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":1.125}\n{\"time\":\"2026-08-28T07:02:30.565Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"54ec9e5a-0913-4e42-a6fc-17cab93d73ca\",\"route\":\"/monitors\",\"method\":\"POST\",\"status\":400,\"durationMs\":1.172458000000006}\n{\"time\":\"2026-08-28T07:02:30.568Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"cbc0736f-baea-4855-95ba-5f909a5affac\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":1.3824170000000322}\n{\"time\":\"2026-08-28T07:02:30.571Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"19b1292b-528a-44f5-8b8e-f886dc763f41\",\"route\":\"/monitors\",\"method\":\"POST\",\"status\":400,\"durationMs\":1.246708000000126}\n{\"time\":\"2026-08-28T07:02:30.573Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"30882036-9242-4715-9d42-7a3faf83c5f7\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":1.180749999999989}\n{\"time\":\"2026-08-28T07:02:30.576Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"f9e30f2e-bde0-4926-9491-cd55c1d35019\",\"route\":\"/monitors\",\"method\":\"POST\",\"status\":400,\"durationMs\":1.203958000000057}\n{\"time\":\"2026-08-28T07:02:30.579Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"a480d06d-a688-418e-add8-5a50e996c6cc\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":1.5051659999999174}\n{\"time\":\"2026-08-28T07:02:30.582Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"eb89d778-01e8-4907-9deb-2ae0f91e0d29\",\"route\":\"/monitors\",\"method\":\"POST\",\"status\":400,\"durationMs\":1.9338339999999334}\n{\"time\":\"2026-08-28T07:02:30.585Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"0e368d7e-6044-4bdc-8ed8-a3d07eb5b229\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":1.6351669999999103}\n{\"time\":\"2026-08-28T07:02:30.589Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"c7747674-a16f-400c-8ec7-2292e0793855\",\"route\":\"/monitors\",\"method\":\"POST\",\"status\":400,\"durationMs\":1.058999999999969}\n{\"time\":\"2026-08-28T07:02:30.592Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"f4599f40-ac98-4162-a53d-940e5b507108\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":0.9653329999998732}\n{\"time\":\"2026-08-28T07:02:30.595Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"4b9aaf3c-6061-40a1-8aed-2920f409691a\",\"route\":\"/monitors\",\"method\":\"POST\",\"status\":400,\"durationMs\":0.9547079999999823}\n{\"time\":\"2026-08-28T07:02:30.598Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"553b6b3b-2f95-4d78-9941-f2c6e843f229\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":1.765374999999949}\n{\"time\":\"2026-08-28T07:02:30.602Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"3231da04-d150-4968-aff3-29658b400f9b\",\"route\":\"/monitors\",\"method\":\"POST\",\"status\":400,\"durationMs\":2.086749999999938}\n{\"time\":\"2026-08-28T07:02:30.605Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"cc425c84-2dfc-43f1-a228-4fd9d2a84411\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":1.2710419999998521}\n{\"time\":\"2026-08-28T07:02:30.608Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"e2a9bd35-1616-49ce-bfbb-e54dd4ba39e4\",\"route\":\"/monitors\",\"method\":\"POST\",\"status\":400,\"durationMs\":1.6853750000000218}\n{\"time\":\"2026-08-28T07:02:30.611Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"6dcca31d-ae50-4ac4-9229-5c7ae0fe6b9c\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":1.4821660000000065}\n{\"time\":\"2026-08-28T07:02:30.615Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"6cba3aca-2895-4597-b282-2b9360358a1b\",\"route\":\"/monitors\",\"method\":\"POST\",\"status\":400,\"durationMs\":1.8109170000000177}\n{\"time\":\"2026-08-28T07:02:30.618Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"b59cdf4f-d911-458f-822d-c355f23dd4dd\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":1.6724159999998847}\n{\"time\":\"2026-08-28T07:02:30.621Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"1c552563-64ee-477f-9ad0-80ef980bab79\",\"route\":\"/monitors\",\"method\":\"POST\",\"status\":400,\"durationMs\":1.6884999999999764}\n{\"time\":\"2026-08-28T07:02:30.624Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"36d1b30d-fa53-48ae-b7d1-f594fbb3cf73\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":0.9951249999999163}\n{\"time\":\"2026-08-28T07:02:30.627Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"fdbd1538-60ac-46a4-92a9-07fe70508289\",\"route\":\"/monitors\",\"method\":\"POST\",\"status\":400,\"durationMs\":1.6121250000001055}\n{\"time\":\"2026-08-28T07:02:30.631Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"6f731f7e-549e-4dfc-97dc-ad87ad67394a\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":1.6274579999999332}\n{\"time\":\"2026-08-28T07:02:30.634Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"83034ee6-b166-4053-a7c4-d1d3a3c4e7a2\",\"route\":\"/monitors\",\"method\":\"POST\",\"status\":400,\"durationMs\":1.6331250000000637}\n{\"time\":\"2026-08-28T07:02:30.636Z\",\"role\":\"api\",\"processId\":\"e78bbdb…19523 tokens truncated…2:36.559Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"855f9bf5-6c7e-46a1-8dc8-374e72bee0e8\",\"route\":\"/auth/session\",\"method\":\"GET\",\"status\":200,\"durationMs\":0.5382909999998446}\n{\"time\":\"2026-08-28T07:02:36.560Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"76ac6512-fd4e-4105-a78f-4f7d903681db\",\"route\":\"/auth/logout\",\"method\":\"POST\",\"status\":403,\"durationMs\":0.5573340000000826}\n{\"time\":\"2026-08-28T07:02:36.562Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"4b71f978-a101-43c5-99f9-b4f8f7075590\",\"route\":\"/auth/session\",\"method\":\"GET\",\"status\":200,\"durationMs\":0.4593329999997877}\n{\"time\":\"2026-08-28T07:02:36.562Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"3df8746c-83e7-42e3-9153-c37f988db2a8\",\"route\":\"/auth/login\",\"method\":\"POST\",\"status\":403,\"durationMs\":0.06316599999991013}\n{\"time\":\"2026-08-28T07:02:36.563Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"ca946c2b-35f1-4105-a556-0c73ef3fa82a\",\"route\":\"/auth/session\",\"method\":\"GET\",\"status\":200,\"durationMs\":0.5101660000000265}\n{\"time\":\"2026-08-28T07:02:36.563Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"f6044508-a4e3-483e-8f2c-b9cb42711696\",\"route\":\"/auth/login\",\"method\":\"POST\",\"status\":403,\"durationMs\":0.05474999999933061}\n{\"time\":\"2026-08-28T07:02:36.564Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"3a9ccaa2-d2bb-4804-9cf5-dedb72dfeb30\",\"route\":\"/auth/session\",\"method\":\"GET\",\"status\":200,\"durationMs\":0.4718750000001819}\n{\"time\":\"2026-08-28T07:02:36.564Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"90b4a716-d93c-4e82-85b4-738bf3802fac\",\"route\":\"/auth/login\",\"method\":\"POST\",\"status\":403,\"durationMs\":0.04804100000001199}\n{\"time\":\"2026-08-28T07:02:36.565Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"f9f68bd9-745d-4f2b-9cea-f816d6934b49\",\"route\":\"/auth/session\",\"method\":\"GET\",\"status\":200,\"durationMs\":0.4842079999998532}\n{\"time\":\"2026-08-28T07:02:36.565Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"80754f0e-c48f-432c-b42a-f0a46008bd8a\",\"route\":\"/monitors\",\"method\":\"GET\",\"status\":401,\"durationMs\":0.0657920000003287}\n{\"time\":\"2026-08-28T07:02:36.565Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"041828f8-a1a4-4d60-8248-265fbc08825f\",\"route\":\"/monitors\",\"method\":\"POST\",\"status\":401,\"durationMs\":0.040957999999591266}\n{\"time\":\"2026-08-28T07:02:36.565Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"3091684f-2f23-4e4c-bfe6-f493dd5fba7c\",\"route\":\"/monitors/:id\",\"method\":\"GET\",\"status\":401,\"durationMs\":0.036291999999775726}\n{\"time\":\"2026-08-28T07:02:36.566Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"da9adbd8-c648-4f8c-af52-d5040e37f232\",\"route\":\"/monitors/:id\",\"method\":\"PUT\",\"status\":401,\"durationMs\":0.03883399999995163}\n{\"time\":\"2026-08-28T07:02:36.566Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"868a1ffb-59c7-4ae1-b069-2e3164e05e12\",\"route\":\"/monitors/:id\",\"method\":\"DELETE\",\"status\":401,\"durationMs\":0.05912499999976717}\n{\"time\":\"2026-08-28T07:02:36.566Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"83adbd3b-322c-4185-8296-85fdeb60e162\",\"route\":\"/monitors/:id/checks\",\"method\":\"GET\",\"status\":401,\"durationMs\":0.04362500000024738}\n{\"time\":\"2026-08-28T07:02:36.566Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"b159adb4-88f3-4bf4-a1f9-42838498a15e\",\"route\":\"/monitors/:id/checks\",\"method\":\"POST\",\"status\":401,\"durationMs\":0.03837500000008731}\n{\"time\":\"2026-08-28T07:02:36.566Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"40e2e683-1d1e-47ff-9a17-4327b64e66c9\",\"route\":\"/checks/:id\",\"method\":\"GET\",\"status\":401,\"durationMs\":0.03329200000007404}\n{\"time\":\"2026-08-28T07:02:36.566Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"ba03649e-84c4-4b12-9aa4-5d68d1754221\",\"route\":\"/auth/session\",\"method\":\"GET\",\"status\":401,\"durationMs\":0.03150000000005093}\n{\"time\":\"2026-08-28T07:02:36.566Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"f0d6cb9e-8427-41d8-a7f2-e1f8c8484b3a\",\"route\":\"/auth/logout\",\"method\":\"POST\",\"status\":401,\"durationMs\":0.05383400000027905}\n{\"time\":\"2026-08-28T07:02:36.566Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"4da9a03d-1136-4b93-af43-d2015cd3f3af\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":401,\"durationMs\":0.04312500000014552}\n{\"time\":\"2026-08-28T07:02:36.566Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"c1add9f0-17aa-4786-9986-d89b972f491a\",\"route\":\"unmatched\",\"method\":\"OPTIONS\",\"status\":403,\"durationMs\":0.0715409999993426}\n{\"time\":\"2026-08-28T07:02:36.567Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"e54fc701-2e2a-440a-9819-13cf0c2bd15d\",\"route\":\"unmatched\",\"method\":\"OPTIONS\",\"status\":403,\"durationMs\":0.035499999999956344}\n{\"time\":\"2026-08-28T07:02:36.567Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"8bd04b35-ed9f-4558-8f7e-0225202ab2e5\",\"route\":\"/monitors\",\"method\":\"GET\",\"status\":403,\"durationMs\":0.6104169999998703}\n{\"time\":\"2026-08-28T07:02:36.568Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"f16f1655-98bf-4717-bfba-3bf5c0a4259b\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":0.5378330000003189}\n{\"time\":\"2026-08-28T07:02:36.835Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"db049ce5-d916-49ac-ade2-1e450c343700\",\"route\":\"/auth/login\",\"method\":\"POST\",\"status\":200,\"durationMs\":267.0056249999998}\n{\"time\":\"2026-08-28T07:02:36.837Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"f2ac1282-9877-467c-b8e6-a7f2824661f6\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":1.7579999999998108}\n{\"time\":\"2026-08-28T07:02:36.838Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"f7ad19cf-eee2-40fa-9421-13a367b0736a\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":401,\"durationMs\":1.193333000000166}\n{\"time\":\"2026-08-28T07:02:36.839Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"28982e8f-8742-4e64-a35b-442e720de347\",\"route\":\"/auth/logout\",\"method\":\"POST\",\"status\":403,\"durationMs\":1.3097079999997732}\n{\"time\":\"2026-08-28T07:02:36.842Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"86ffb3ce-e9ee-455b-a140-a37d363509a8\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":0.8547920000000886}\n{\"time\":\"2026-08-28T07:02:36.844Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"25e0c246-4626-418e-a9e3-5dba97cd2048\",\"route\":\"/monitors/:id\",\"method\":\"PUT\",\"status\":200,\"durationMs\":2.248708999999508}\n{\"time\":\"2026-08-28T07:02:36.845Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"f3164ec3-863c-4dd2-a591-c2476cdf7290\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":0.6572089999999662}\n{\"time\":\"2026-08-28T07:02:36.847Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"615c036f-2699-496a-a466-b12a95f027ed\",\"route\":\"/monitors/:id\",\"method\":\"PUT\",\"status\":200,\"durationMs\":1.7821670000002996}\n{\"time\":\"2026-08-28T07:02:36.848Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"7bb0a025-181a-4bfd-b424-e2f5bdada50b\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":0.5337079999999332}\n{\"time\":\"2026-08-28T07:02:36.849Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"83a7f3ce-93e5-4ceb-b4b3-745c5245154b\",\"route\":\"/monitors/:id\",\"method\":\"PUT\",\"status\":200,\"durationMs\":1.6818750000002183}\n{\"time\":\"2026-08-28T07:02:36.850Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"f4183d8b-066e-48df-877b-c09599c3db9c\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":0.6092920000000959}\n{\"time\":\"2026-08-28T07:02:36.851Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"ffc10811-c346-4ee1-81f8-635baba2b73c\",\"route\":\"/monitors/:id\",\"method\":\"DELETE\",\"status\":200,\"durationMs\":1.1947909999998956}\n{\"time\":\"2026-08-28T07:02:36.853Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"6746a85b-76c5-4e98-b201-11e71363f017\",\"route\":\"/checks/:id\",\"method\":\"GET\",\"status\":200,\"durationMs\":0.9312919999993028}\n{\"time\":\"2026-08-28T07:02:36.853Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"07ac7062-114d-4b6b-9cc9-b7eb90fe76fc\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":0.5084170000000086}\n{\"time\":\"2026-08-28T07:02:36.854Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"966d5440-c3e4-4cb8-807a-585096db1bc2\",\"route\":\"/auth/logout\",\"method\":\"POST\",\"status\":200,\"durationMs\":1.0160839999998643}\n{\"time\":\"2026-08-28T07:02:36.855Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"f6cfd396-41eb-412c-8b2e-c0caad53a449\",\"route\":\"/monitors\",\"method\":\"GET\",\"status\":401,\"durationMs\":0.558165999999801}\n{\"time\":\"2026-08-28T07:02:36.855Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"18c6c5db-3c53-409f-9900-8a7d63b35aa5\",\"route\":\"/monitors\",\"method\":\"GET\",\"status\":401,\"durationMs\":0.07237499999973807}\n{\"time\":\"2026-08-28T07:02:36.856Z\",\"role\":\"api\",\"processId\":\"e78bbdb5-9e48-4b95-a3c3-b4be149d0db9\",\"pid\":87310,\"event\":\"http_request\",\"requestId\":\"29f6677e-a4c2-44d6-b4b4-882f56512e3b\",\"route\":\"/monitors\",\"method\":\"GET\",\"status\":401,\"durationMs\":0.6234580000000278}\n✔ E05 fixed A/B ownership matrix: collection, nested/direct reads and every mutation use the session owner (2483.873708ms)\n{\"time\":\"2026-08-28T07:02:38.299Z\",\"role\":\"api\",\"processId\":\"9f68bcec-9055-4d02-93cc-a1d9078e3838\",\"pid\":87362,\"event\":\"http_request\",\"requestId\":\"44df449f-cd14-420a-bb1c-caa5c9a5b99b\",\"route\":\"/auth/login\",\"method\":\"POST\",\"status\":200,\"durationMs\":266.98962500000005}\n{\"time\":\"2026-08-28T07:02:38.504Z\",\"role\":\"api\",\"processId\":\"9f68bcec-9055-4d02-93cc-a1d9078e3838\",\"pid\":87362,\"event\":\"http_request\",\"requestId\":\"7295a403-ac82-4495-be71-dc1525300969\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":1.5464580000000296}\n{\"time\":\"2026-08-28T07:02:38.510Z\",\"role\":\"api\",\"processId\":\"9f68bcec-9055-4d02-93cc-a1d9078e3838\",\"pid\":87362,\"event\":\"http_request\",\"requestId\":\"82a21779-b613-4ac2-b785-1561c8bf1b2e\",\"route\":\"/monitors\",\"method\":\"POST\",\"status\":201,\"durationMs\":5.361457999999857}\n{\"time\":\"2026-08-28T07:02:38.512Z\",\"role\":\"api\",\"processId\":\"9f68bcec-9055-4d02-93cc-a1d9078e3838\",\"pid\":87362,\"event\":\"http_request\",\"requestId\":\"c50c31af-e7f4-4770-97f4-2e10d9965ef8\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":1.457540999999992}\n{\"time\":\"2026-08-28T07:02:38.518Z\",\"role\":\"api\",\"processId\":\"9f68bcec-9055-4d02-93cc-a1d9078e3838\",\"pid\":87362,\"event\":\"check_accepted\",\"requestId\":\"1bb5f64b-d679-4cb1-aa34-e784ff658d99\",\"checkId\":\"dfd36aaa-eb01-42c5-ae93-449c7931a71b\"}\n{\"time\":\"2026-08-28T07:02:38.518Z\",\"role\":\"api\",\"processId\":\"9f68bcec-9055-4d02-93cc-a1d9078e3838\",\"pid\":87362,\"event\":\"http_request\",\"requestId\":\"1bb5f64b-d679-4cb1-aa34-e784ff658d99\",\"route\":\"/monitors/:id/checks\",\"method\":\"POST\",\"status\":202,\"durationMs\":5.147999999999911}\n"
+        },
+        {
+          "chunk_id": "703848",
+          "wall_time_seconds": 0.000003042,
+          "exit_code": 0,
+          "original_token_count": 1470,
+          "output": "{\"time\":\"2026-08-28T07:02:38.804Z\",\"role\":\"api\",\"processId\":\"9f68bcec-9055-4d02-93cc-a1d9078e3838\",\"pid\":87362,\"event\":\"http_request\",\"requestId\":\"b3267fe2-0acb-45aa-a14d-752ae543685d\",\"route\":\"/checks/:id\",\"method\":\"GET\",\"status\":200,\"durationMs\":3.8402920000000904}\n{\"time\":\"2026-08-28T07:02:38.810Z\",\"role\":\"api\",\"processId\":\"9f68bcec-9055-4d02-93cc-a1d9078e3838\",\"pid\":87362,\"event\":\"http_request\",\"requestId\":\"f564ba00-9b77-4815-9c28-8255adf40c95\",\"route\":\"/monitors\",\"method\":\"GET\",\"status\":200,\"durationMs\":5.738624999999956}\n✔ create a Monitor in PostgreSQL and observe the queued worker GET /ok (1349.086208ms)\n{\"time\":\"2026-08-28T07:02:38.814Z\",\"role\":\"api\",\"processId\":\"9f68bcec-9055-4d02-93cc-a1d9078e3838\",\"pid\":87362,\"event\":\"http_request\",\"requestId\":\"524e5388-6bae-4e65-b487-431f5be27054\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":2.053042000000005}\n{\"time\":\"2026-08-28T07:02:38.819Z\",\"role\":\"api\",\"processId\":\"9f68bcec-9055-4d02-93cc-a1d9078e3838\",\"pid\":87362,\"event\":\"http_request\",\"requestId\":\"04ba6035-72d6-4827-aee2-e647f04ac769\",\"route\":\"/monitors\",\"method\":\"POST\",\"status\":201,\"durationMs\":4.467499999999973}\n{\"time\":\"2026-08-28T07:02:38.822Z\",\"role\":\"api\",\"processId\":\"9f68bcec-9055-4d02-93cc-a1d9078e3838\",\"pid\":87362,\"event\":\"http_request\",\"requestId\":\"3b584fff-01d9-4d90-b8c5-497a6e7af947\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":1.866125000000011}\n{\"time\":\"2026-08-28T07:02:38.828Z\",\"role\":\"api\",\"processId\":\"9f68bcec-9055-4d02-93cc-a1d9078e3838\",\"pid\":87362,\"event\":\"check_accepted\",\"requestId\":\"bc014a27-15d5-440f-b651-999e443f18cb\",\"checkId\":\"7e36d962-4770-46e1-911e-6a9a894048c7\"}\n{\"time\":\"2026-08-28T07:02:38.828Z\",\"role\":\"api\",\"processId\":\"9f68bcec-9055-4d02-93cc-a1d9078e3838\",\"pid\":87362,\"event\":\"http_request\",\"requestId\":\"bc014a27-15d5-440f-b651-999e443f18cb\",\"route\":\"/monitors/:id/checks\",\"method\":\"POST\",\"status\":202,\"durationMs\":6.26195800000005}\n{\"time\":\"2026-08-28T07:02:39.098Z\",\"role\":\"api\",\"processId\":\"9f68bcec-9055-4d02-93cc-a1d9078e3838\",\"pid\":87362,\"event\":\"http_request\",\"requestId\":\"a48e6871-194f-4752-b976-a402fb75f30a\",\"route\":\"/checks/:id\",\"method\":\"GET\",\"status\":200,\"durationMs\":5.217791999999918}\n✔ GET /fail is an observed endpoint failure with HTTP 503 (286.512958ms)\n{\"time\":\"2026-08-28T07:02:39.103Z\",\"role\":\"api\",\"processId\":\"9f68bcec-9055-4d02-93cc-a1d9078e3838\",\"pid\":87362,\"event\":\"http_request\",\"requestId\":\"6e8cfd13-f87d-4634-97a8-2d42994beb16\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":3.7899999999999636}\n{\"time\":\"2026-08-28T07:02:39.111Z\",\"role\":\"api\",\"processId\":\"9f68bcec-9055-4d02-93cc-a1d9078e3838\",\"pid\":87362,\"event\":\"http_request\",\"requestId\":\"2f63a5ee-a107-4f6b-a273-c377bf1df467\",\"route\":\"/monitors\",\"method\":\"POST\",\"status\":201,\"durationMs\":6.856291999999939}\n{\"time\":\"2026-08-28T07:02:39.114Z\",\"role\":\"api\",\"processId\":\"9f68bcec-9055-4d02-93cc-a1d9078e3838\",\"pid\":87362,\"event\":\"http_request\",\"requestId\":\"49037f18-15ec-4dd0-b35b-b9be8603ed65\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":2.5886250000000928}\n{\"time\":\"2026-08-28T07:02:39.124Z\",\"role\":\"api\",\"processId\":\"9f68bcec-9055-4d02-93cc-a1d9078e3838\",\"pid\":87362,\"event\":\"check_accepted\",\"requestId\":\"742d32fd-f885-425c-9fd1-ab651bf4cc2c\",\"checkId\":\"6ad47ac4-f45c-4d81-afa1-2f178dcc9224\"}\n{\"time\":\"2026-08-28T07:02:39.125Z\",\"role\":\"api\",\"processId\":\"9f68bcec-9055-4d02-93cc-a1d9078e3838\",\"pid\":87362,\"event\":\"http_request\",\"requestId\":\"742d32fd-f885-425c-9fd1-ab651bf4cc2c\",\"route\":\"/monitors/:id/checks\",\"method\":\"POST\",\"status\":202,\"durationMs\":10.215832999999975}\n{\"time\":\"2026-08-28T07:02:39.357Z\",\"role\":\"api\",\"processId\":\"9f68bcec-9055-4d02-93cc-a1d9078e3838\",\"pid\":87362,\"event\":\"http_request\",\"requestId\":\"ec165e09-3114-4044-8286-2bb43446a10e\",\"route\":\"/checks/:id\",\"method\":\"GET\",\"status\":200,\"durationMs\":2.851750000000038}\n✔ the check follows a validated same-origin redirect to its final status (258.401875ms)\n{\"time\":\"2026-08-28T07:02:39.359Z\",\"role\":\"api\",\"processId\":\"9f68bcec-9055-4d02-93cc-a1d9078e3838\",\"pid\":87362,\"event\":\"http_request\",\"requestId\":\"8b222226-90db-4aab-bcbc-c60dc5388d52\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":1.9089160000000902}\n{\"time\":\"2026-08-28T07:02:39.362Z\",\"role\":\"api\",\"processId\":\"9f68bcec-9055-4d02-93cc-a1d9078e3838\",\"pid\":87362,\"event\":\"http_request\",\"requestId\":\"0fe9955d-3ce4-4719-afe9-18d3954cd96c\",\"route\":\"/monitors\",\"method\":\"POST\",\"status\":201,\"durationMs\":2.8579169999998157}\n{\"time\":\"2026-08-28T07:02:39.363Z\",\"role\":\"api\",\"processId\":\"9f68bcec-9055-4d02-93cc-a1d9078e3838\",\"pid\":87362,\"event\":\"http_request\",\"requestId\":\"56411dab-1376-4afe-b346-a0099f36c9f9\",\"route\":\"/auth/csrf\",\"method\":\"GET\",\"status\":200,\"durationMs\":0.948415999999952}\n{\"time\":\"2026-08-28T07:02:39.365Z\",\"role\":\"api\",\"processId\":\"9f68bcec-9055-4d02-93cc-a1d9078e3838\",\"pid\":87362,\"event\":\"http_request\",\"requestId\":\"0af19f69-48f0-4b98-9439-886cab5adf64\",\"route\":\"/monitors\",\"method\":\"POST\",\"status\":400,\"durationMs\":1.441000000000031}\n✔ a non-fixture URL is rejected without contacting the controlled guard (8.57125ms)\n{\"time\":\"2026-08-28T07:02:39.369Z\",\"role\":\"api\",\"processId\":\"9f68bcec-9055-4d02-93cc-a1d9078e3838\",\"pid\":87362,\"event\":\"http_request\",\"requestId\":\"629ab1d4-f0d1-4c4d-86c7-73ecd73c1fbe\",\"route\":\"/monitors\",\"method\":\"GET\",\"status\":200,\"durationMs\":3.3731669999997393}\n{\"time\":\"2026-08-28T07:02:39.388Z\",\"role\":\"api\",\"processId\":\"9f68bcec-9055-4d02-93cc-a1d9078e3838\",\"pid\":87362,\"event\":\"http_request\",\"requestId\":\"d7228279-a7a7-4147-94f8-e8ad52da8593\",\"route\":\"/monitors\",\"method\":\"GET\",\"status\":200,\"durationMs\":2.6570410000003903}\n✔ another application instance reads the same persisted Monitors and latest checks (23.720959ms)\nℹ tests 15\nℹ suites 0\nℹ pass 15\nℹ fail 0\nℹ cancelled 0\nℹ skipped 0\nℹ todo 0\nℹ duration_ms 10396.864958\nreal 10.55\nuser 6.64\nsys 0.29\n"
+        }
+      ]
+    },
+    {
+      "gate": "production-build",
+      "command": "/usr/bin/time -p docker compose --project-name wse-fundamentals-e24 -f compose.production.yaml build api frontend",
+      "exit": 0,
+      "realSeconds": 34.44,
+      "consoleComplete": true,
+      "chunks": [
+        {
+          "chunk_id": "49e99d",
+          "wall_time_seconds": 1.001431792,
+          "session_id": 10575,
+          "original_token_count": 139,
+          "output": "Compose can now delegate builds to bake for better performance.\n To do so, set COMPOSE_BAKE=true.\n#0 building with \"desktop-linux\" instance using docker driver\n\n#1 [api internal] load build definition from Dockerfile\n#1 transferring dockerfile: 1.37kB 0.0s done\n#1 DONE 0.0s\n\n#2 [frontend internal] load build definition from Dockerfile\n#2 transferring dockerfile: 1.37kB 0.0s done\n#2 DONE 0.0s\n\n#3 [frontend internal] load metadata for docker.io/library/node:24.19.0-bookworm-slim@sha256:a9f5f7c91a432850b2a8a7797adf5eadb6c733ceed61167806cee7ea7fbc29df\n"
+        },
+        {
+          "chunk_id": "1d72d1",
+          "wall_time_seconds": 5.001735167,
+          "session_id": 10575,
+          "original_token_count": 550,
+          "output": "#3 DONE 1.3s\n\n#4 [api internal] load .dockerignore\n#4 transferring context: 189B done\n#4 DONE 0.0s\n\n#5 [frontend internal] load .dockerignore\n#5 transferring context: 189B done\n#5 DONE 0.0s\n\n#6 [api internal] load build context\n#6 transferring context: 138.95kB 0.0s done\n#6 DONE 0.0s\n\n#7 [frontend internal] load build context\n#7 transferring context: 174.64kB done\n#7 DONE 0.0s\n\n#8 [api dependencies 1/4] FROM docker.io/library/node:24.19.0-bookworm-slim@sha256:a9f5f7c91a432850b2a8a7797adf5eadb6c733ceed61167806cee7ea7fbc29df\n#8 resolve docker.io/library/node:24.19.0-bookworm-slim@sha256:a9f5f7c91a432850b2a8a7797adf5eadb6c733ceed61167806cee7ea7fbc29df 0.0s done\n#8 DONE 0.1s\n\n#9 [api dependencies 2/4] WORKDIR /app\n#9 DONE 0.0s\n\n#10 [api dependencies 3/4] COPY package.json package-lock.json ./\n#10 DONE 0.0s\n\n#11 [api dependencies 4/4] RUN npm ci --no-audit --no-fund --fetch-retries=0\n#11 14.18 \n#11 14.18 added 94 packages in 14s\n#11 14.18 npm notice\n#11 14.18 npm notice New major version of npm available! 11.17.0 -> 12.0.2\n#11 14.18 npm notice Changelog: https://github.com/npm/cli/releases/tag/v12.0.2\n#11 14.18 npm notice To update run: npm install -g npm@12.0.2\n#11 14.18 npm notice\n#11 DONE 15.2s\n\n#12 [frontend dependencies 4/4] RUN npm ci --no-audit --no-fund --fetch-retries=0\n#12 CACHED\n\n#13 [frontend dependencies 3/4] COPY package.json package-lock.json ./\n#13 CACHED\n\n#14 [frontend web-build 1/4] COPY next.config.ts tsconfig.json ./\n#14 DONE 0.1s\n\n#15 [frontend web-build 2/4] COPY app ./app\n#15 DONE 0.0s\n\n#16 [frontend web-build 3/4] COPY server ./server\n#16 DONE 0.0s\n\n#17 [api production-dependencies 1/1] RUN npm prune --omit=dev --no-audit --no-fund\n#17 0.836 \n#17 0.836 removed 8 packages in 641ms\n#17 DONE 0.9s\n\n#18 [frontend web-build 4/4] RUN npm run build\n#18 0.156 \n#18 0.156 > monitor-fundamentals-fastify@0.1.0 build\n#18 0.156 > NEXT_TELEMETRY_DISABLED=1 next build\n#18 0.156 \n#18 0.430 ▲ Next.js 16.3.3 (Turbopack)\n#18 0.454 ✓ Running next.config.ts took 24ms\n#18 0.466 \n#18 0.485   Creating an optimized production build ...\n#18 ...\n\n#19 [api backend 3/5] COPY --from=production-dependencies --chown=node:node /app/node_modules ./node_modules\n#19 DONE 2.8s\n"
+        },
+        {
+          "chunk_id": "c9dc8f",
+          "wall_time_seconds": 0.000001792,
+          "exit_code": 0,
+          "original_token_count": 766,
+          "output": "\n#20 [api backend 4/5] COPY --chown=node:node package.json ./\n#20 DONE 0.2s\n\n#18 [frontend web-build 4/4] RUN npm run build\n#18 4.694 ✓ Compiled successfully in 4.1s\n#18 4.700   Running TypeScript ...\n#18 ...\n\n#21 [api backend 5/5] COPY --chown=node:node server ./server\n#21 DONE 0.2s\n\n#22 [api] exporting to image\n#22 exporting layers\n#22 ...\n\n#18 [frontend web-build 4/4] RUN npm run build\n#18 7.827   Finished TypeScript in 3.1s ...\n#18 7.829   Collecting page data using 6 workers ...\n#18 8.271   Generating static pages using 6 workers (0/5) ...\n#18 8.523   Generating static pages using 6 workers (1/5) \r\n#18 8.563   Generating static pages using 6 workers (2/5) \r\n#18 8.564   Generating static pages using 6 workers (3/5) \r\n#18 8.565 ✓ Generating static pages using 6 workers (5/5) in 293ms\n#18 8.568   Finalizing page optimization ...\n#18 8.720 \n#18 8.722 Route (app)\n#18 8.722 ┌ ○ /\n#18 8.722 ├ ○ /_not-found\n#18 8.722 ├ ○ /login\n#18 8.722 └ ƒ /monitors\n#18 8.722 \n#18 8.722 \n#18 8.722 ○  (Static)   prerendered as static content\n#18 8.722 ƒ  (Dynamic)  server-rendered on demand\n#18 8.722 \n#18 DONE 8.9s\n\n#23 [frontend frontend 3/4] COPY --from=web-build --chown=node:node /app/.next/standalone ./\n#23 DONE 0.2s\n\n#22 [api] exporting to image\n#22 ...\n\n#24 [frontend frontend 4/4] COPY --from=web-build --chown=node:node /app/.next/static ./.next/static\n#24 DONE 0.0s\n\n#25 [frontend] exporting to image\n#25 exporting layers 0.9s done\n#25 exporting manifest sha256:c77f361d5fec69321393e006714fdc4ed2f2ecad9c67f11fe4a0b585efac3fc7 done\n#25 exporting config sha256:a93e98a1702f149f9de7ffb3cc82dd59b14a52bc4370c073251482ef1db7709c done\n#25 exporting attestation manifest sha256:eb60fe3e579f12c7d84cf84c975610dc934b4f79c690272e59cae13683229ad0 done\n#25 exporting manifest list sha256:7cd37b250c2e45bcc99271b688a74336dcbee88d10a29251edc8cbb61f68de16 done\n#25 naming to docker.io/library/wse-fundamentals-e24-frontend:local done\n#25 unpacking to docker.io/library/wse-fundamentals-e24-frontend:local\n#25 unpacking to docker.io/library/wse-fundamentals-e24-frontend:local 0.3s done\n#25 DONE 1.2s\n\n#22 [api] exporting to image\n#22 ...\n\n#26 [frontend] resolving provenance for metadata file\n#26 DONE 0.0s\n\n#22 [api] exporting to image\n#22 exporting layers 10.2s done\n#22 exporting manifest sha256:8757bae8da63684472d42e21f0fc083347768fe7aa5d2a2fed810a395323032c done\n#22 exporting config sha256:0bc5bb08e0c9b39bc9c716eeacf88f179faf2e67607b668fabff1d3b590aff4e done\n#22 exporting attestation manifest sha256:96118b91a3ebb70fa1b23be5ddb17a6fd03cebce6d6ca7cd1153a41e2679cfe3 done\n#22 exporting manifest list sha256:2f6918aed823d07ac1e05980b7747bf49e0912a027326ef3c5713c98fc5bfbc8 done\n#22 naming to docker.io/library/wse-fundamentals-e24-backend:local done\n#22 unpacking to docker.io/library/wse-fundamentals-e24-backend:local\n#22 unpacking to docker.io/library/wse-fundamentals-e24-backend:local 1.9s done\n#22 DONE 12.2s\n\n#27 [api] resolving provenance for metadata file\n#27 DONE 0.0s\n api  Built\n frontend  Built\nreal 34.44\nuser 0.27\nsys 0.27\n"
+        }
+      ]
+    }
+  ]
+}
diff --git a/evidence/phase-1/E24/results/baseline.json b/evidence/phase-1/E24/results/baseline.json
new file mode 100644
index 0000000..1175ae4
--- /dev/null
+++ b/evidence/phase-1/E24/results/baseline.json
@@ -0,0 +1,28 @@
+{
+  "head": "d9234ede912b936c7896fefebc10e8273bb87583",
+  "start": "9d0a1974e1f279146917b840e69bbee19dcfc0c4",
+  "productMatchesStart": true,
+  "hashes": {
+    "scenario.json": "07b5b3e2b4bd1cfc456c415134adce61f05dfdd729fe803968b32a4d813c5847",
+    "fixture.ts": "10d311f2e2af33063d09add435ed282b20341e2b87eeffa10b6287dce61ca784",
+    "baseline.mjs": "0a24805b65454cddc543d27708aec68092b34e28b67ca6ba605e3bc6602c79d6"
+  },
+  "result": "REPRODUCED",
+  "observation": {
+    "legacyLiveStatus": 200,
+    "readinessStatus": 401,
+    "readinessCode": "UNAUTHENTICATED",
+    "users": 2,
+    "monitors": 2,
+    "terminalChecks": 4,
+    "containerBuilds": 0,
+    "postgresStops": 0,
+    "outboundRequests": 0
+  },
+  "decisiveFailure": "The existing liveness endpoint works, but the frozen readiness path has no unauthenticated operational interface.",
+  "cleanup": {
+    "schemaDropped": true,
+    "portsFree": true
+  },
+  "durationMs": 645
+}
diff --git a/evidence/phase-1/E24/results/baseline.started.json b/evidence/phase-1/E24/results/baseline.started.json
new file mode 100644
index 0000000..a76ebb6
--- /dev/null
+++ b/evidence/phase-1/E24/results/baseline.started.json
@@ -0,0 +1 @@
+{"hashes":{"scenario.json":"07b5b3e2b4bd1cfc456c415134adce61f05dfdd729fe803968b32a4d813c5847","fixture.ts":"10d311f2e2af33063d09add435ed282b20341e2b87eeffa10b6287dce61ca784","baseline.mjs":"0a24805b65454cddc543d27708aec68092b34e28b67ca6ba605e3bc6602c79d6"},"head":"d9234ede912b936c7896fefebc10e8273bb87583"}
diff --git a/evidence/phase-1/E24/results/full-author.console.txt b/evidence/phase-1/E24/results/full-author.console.txt
new file mode 100644
index 0000000..fee17e3
--- /dev/null
+++ b/evidence/phase-1/E24/results/full-author.console.txt
@@ -0,0 +1,4 @@
+{"result":"FAILED","mode":"full","actor":"author","durationMs":6930,"failure":{"stage":"owned PostgreSQL stop and restore","assertion":"Docker compose failed; raw output withheld from artifacts.","kind":"Error"},"cleanup":{"schemaDropped":false,"runtimeRemoved":true,"postgresRestored":false,"portsFree":true,"errors":[]}}
+real 7.48
+user 2.07
+sys 0.74
diff --git a/evidence/phase-1/E24/results/full-author.json b/evidence/phase-1/E24/results/full-author.json
new file mode 100644
index 0000000..ab6add0
--- /dev/null
+++ b/evidence/phase-1/E24/results/full-author.json
@@ -0,0 +1,261 @@
+{
+  "mode": "full",
+  "actor": "author",
+  "result": "FAILED",
+  "commands": [
+    {
+      "args": [
+        "ps",
+        "-aq",
+        "--filter",
+        "label=com.docker.compose.project=wse-fundamentals-e24"
+      ],
+      "exitCode": 0,
+      "durationMs": 23
+    },
+    {
+      "args": [
+        "network",
+        "ls",
+        "-q",
+        "--filter",
+        "label=com.docker.compose.project=wse-fundamentals-e24"
+      ],
+      "exitCode": 0,
+      "durationMs": 20
+    },
+    {
+      "args": [
+        "compose",
+        "--project-name",
+        "wse-fundamentals-e24",
+        "-f",
+        "compose.production.yaml",
+        "-f",
+        "evidence/phase-1/E24/compose.test.yaml",
+        "up",
+        "-d",
+        "--no-build",
+        "api",
+        "frontend",
+        "fixture"
+      ],
+      "exitCode": 0,
+      "durationMs": 786
+    },
+    {
+      "args": [
+        "compose",
+        "--project-name",
+        "wse-fundamentals-e24",
+        "-f",
+        "compose.production.yaml",
+        "-f",
+        "evidence/phase-1/E24/compose.test.yaml",
+        "up",
+        "-d",
+        "--no-build",
+        "worker"
+      ],
+      "exitCode": 0,
+      "durationMs": 902
+    },
+    {
+      "args": [
+        "compose",
+        "--project-name",
+        "wse-fundamentals-e24",
+        "-f",
+        "compose.production.yaml",
+        "-f",
+        "evidence/phase-1/E24/compose.test.yaml",
+        "ps",
+        "-q",
+        "api"
+      ],
+      "exitCode": 0,
+      "durationMs": 73
+    },
+    {
+      "args": [
+        "inspect",
+        "--format",
+        "{\"pid\":{{.State.Pid}},\"running\":{{.State.Running}},\"restarts\":{{.RestartCount}},\"image\":{{json .Image}},\"command\":{{json .Config.Cmd}}}",
+        "fb7689f1ec1e74e4123586e76ec596c5fabe6ca87229fafcf172670e36c4fa09"
+      ],
+      "exitCode": 0,
+      "durationMs": 17
+    },
+    {
+      "args": [
+        "compose",
+        "--project-name",
+        "wse-fundamentals-e24",
+        "-f",
+        "compose.production.yaml",
+        "-f",
+        "evidence/phase-1/E24/compose.test.yaml",
+        "ps",
+        "-q",
+        "worker"
+      ],
+      "exitCode": 0,
+      "durationMs": 67
+    },
+    {
+      "args": [
+        "inspect",
+        "--format",
+        "{\"pid\":{{.State.Pid}},\"running\":{{.State.Running}},\"restarts\":{{.RestartCount}},\"image\":{{json .Image}},\"command\":{{json .Config.Cmd}}}",
+        "be408feae6b6684bbced3387486108098691cf9e3c12d95440da1f06957896e2"
+      ],
+      "exitCode": 0,
+      "durationMs": 16
+    },
+    {
+      "args": [
+        "compose",
+        "--project-name",
+        "wse-fundamentals-e24",
+        "-f",
+        "compose.production.yaml",
+        "-f",
+        "evidence/phase-1/E24/compose.test.yaml",
+        "ps",
+        "-q",
+        "frontend"
+      ],
+      "exitCode": 0,
+      "durationMs": 70
+    },
+    {
+      "args": [
+        "inspect",
+        "--format",
+        "{\"pid\":{{.State.Pid}},\"running\":{{.State.Running}},\"restarts\":{{.RestartCount}},\"image\":{{json .Image}},\"command\":{{json .Config.Cmd}}}",
+        "b27da6be7bad9931686349071aa16006ac293119588e1d6b88d4b755e1a7eab6"
+      ],
+      "exitCode": 0,
+      "durationMs": 16
+    },
+    {
+      "args": [
+        "compose",
+        "--project-name",
+        "wse-fundamentals",
+        "-f",
+        "compose.yaml",
+        "stop",
+        "-t",
+        "5",
+        "postgres"
+      ],
+      "exitCode": 0,
+      "durationMs": 264
+    },
+    {
+      "args": [
+        "compose",
+        "--project-name",
+        "wse-fundamentals",
+        "-f",
+        "compose.yaml",
+        "start",
+        "--wait",
+        "--wait-timeout",
+        "30",
+        "postgres"
+      ],
+      "exitCode": 1,
+      "durationMs": 60,
+      "stdoutBytes": 0,
+      "stderrBytes": 21
+    },
+    {
+      "args": [
+        "compose",
+        "--project-name",
+        "wse-fundamentals-e24",
+        "-f",
+        "compose.production.yaml",
+        "-f",
+        "evidence/phase-1/E24/compose.test.yaml",
+        "down"
+      ],
+      "exitCode": 0,
+      "durationMs": 921
+    }
+  ],
+  "hashes": {
+    "scenario.json": "07b5b3e2b4bd1cfc456c415134adce61f05dfdd729fe803968b32a4d813c5847",
+    "fixture.ts": "10d311f2e2af33063d09add435ed282b20341e2b87eeffa10b6287dce61ca784",
+    "baseline.mjs": "0a24805b65454cddc543d27708aec68092b34e28b67ca6ba605e3bc6602c79d6",
+    "compose.test.yaml": "e84ee2f893bb851ef630a6e9e34dde1daaadc9d8f4ea8b641a3f299b55c80fbe",
+    "fixture-server.mjs": "5ba0eeb56ec9e2e664aed5e32c0742366f6122051805a777966c894cc769cbb1",
+    "observer": "637833438f7f16d064f0efd18c5dc3cc40568814b0aca55475fa6b5984017390"
+  },
+  "postgresStops": 1,
+  "postgresRestores": 1,
+  "conditionPolls": 61,
+  "healthUp": [
+    {
+      "live": 200,
+      "ready": 200
+    },
+    {
+      "live": 200,
+      "ready": 200
+    }
+  ],
+  "worker": {
+    "accepted202": true,
+    "queuedUnknownStatus": true,
+    "positiveQueueAge": true,
+    "activeObserved": true,
+    "sameIdTerminal503": true,
+    "claims": 1,
+    "recovered": 0,
+    "recoveryScans": 2
+  },
+  "browser": {
+    "version": "151.0.7922.34",
+    "login": true,
+    "list": true,
+    "detailRows": 4,
+    "ownerIsolation": true,
+    "staticStatus": 200,
+    "serverRouteStatus": 200,
+    "consoleErrors": 0,
+    "screenshots": 0,
+    "traces": 0
+  },
+  "cardinality": {
+    "before": 35,
+    "after": 35,
+    "distinctMissingIds": 10,
+    "errorCountDelta": 10
+  },
+  "healthDown": [
+    {
+      "live": 200,
+      "ready": 503
+    },
+    {
+      "live": 200,
+      "ready": 503
+    }
+  ],
+  "failure": {
+    "stage": "owned PostgreSQL stop and restore",
+    "assertion": "Docker compose failed; raw output withheld from artifacts.",
+    "kind": "Error"
+  },
+  "cleanup": {
+    "schemaDropped": false,
+    "runtimeRemoved": true,
+    "postgresRestored": false,
+    "portsFree": true,
+    "errors": []
+  },
+  "durationMs": 6930
+}
diff --git a/evidence/phase-1/E24/results/full-author.started.json b/evidence/phase-1/E24/results/full-author.started.json
new file mode 100644
index 0000000..416243f
--- /dev/null
+++ b/evidence/phase-1/E24/results/full-author.started.json
@@ -0,0 +1 @@
+{"mode":"full","actor":"author","at":"2026-08-28T07:05:58.032Z"}
diff --git a/evidence/phase-1/E24/results/full-repair1.console.json b/evidence/phase-1/E24/results/full-repair1.console.json
new file mode 100644
index 0000000..1e2b754
--- /dev/null
+++ b/evidence/phase-1/E24/results/full-repair1.console.json
@@ -0,0 +1,50 @@
+{
+  "command": "/usr/bin/time -p fnm exec --using 24.19.0 node test/container-smoke.mjs full repair1",
+  "chunks": [
+    {
+      "chunk_id": "ab4452",
+      "wall_time_seconds": 1.002003417,
+      "session_id": 78872,
+      "original_token_count": 0,
+      "output": ""
+    },
+    {
+      "chunk_id": "954705",
+      "wall_time_seconds": 3.786827625,
+      "exit_code": 1,
+      "original_token_count": 92,
+      "output": "{\"result\":\"FAILED\",\"mode\":\"full\",\"actor\":\"repair1\",\"durationMs\":9183,\"failure\":{\"stage\":\"actual runtime UID and signal boundary\",\"assertion\":\"Actual role SIGTERM exit must be clean and within its frozen bound.\",\"kind\":\"Error\"},\"cleanup\":{\"schemaDropped\":true,\"runtimeRemoved\":true,\"postgresRestored\":true,\"portsFree\":true,\"errors\":[]}}\nreal 9.67\nuser 2.36\nsys 0.89\n"
+    }
+  ],
+  "consoleComplete": true,
+  "automaticRetries": 0,
+  "additionalScenarioRuns": 0,
+  "preflightValidations": [
+    {
+      "command": "fnm exec --using 24.19.0 node --check test/container-smoke.mjs",
+      "result": {
+        "status": "fulfilled",
+        "value": {
+          "chunk_id": "254f1e",
+          "wall_time_seconds": 0.010003042,
+          "exit_code": 0,
+          "original_token_count": 0,
+          "output": ""
+        }
+      }
+    },
+    {
+      "command": "git diff --check",
+      "result": {
+        "status": "fulfilled",
+        "value": {
+          "chunk_id": "934752",
+          "wall_time_seconds": 0.000003125,
+          "exit_code": 0,
+          "original_token_count": 0,
+          "output": ""
+        }
+      }
+    }
+  ]
+}
diff --git a/evidence/phase-1/E24/results/full-repair1.console.txt b/evidence/phase-1/E24/results/full-repair1.console.txt
new file mode 100644
index 0000000..6b1f1d2
--- /dev/null
+++ b/evidence/phase-1/E24/results/full-repair1.console.txt
@@ -0,0 +1,4 @@
+{"result":"FAILED","mode":"full","actor":"repair1","durationMs":9183,"failure":{"stage":"actual runtime UID and signal boundary","assertion":"Actual role SIGTERM exit must be clean and within its frozen bound.","kind":"Error"},"cleanup":{"schemaDropped":true,"runtimeRemoved":true,"postgresRestored":true,"portsFree":true,"errors":[]}}
+real 9.67
+user 2.36
+sys 0.89
diff --git a/evidence/phase-1/E24/results/full-repair1.json b/evidence/phase-1/E24/results/full-repair1.json
new file mode 100644
index 0000000..fcdebf8
--- /dev/null
+++ b/evidence/phase-1/E24/results/full-repair1.json
@@ -0,0 +1,511 @@
+{
+  "mode": "full",
+  "actor": "repair1",
+  "result": "FAILED",
+  "commands": [
+    {
+      "args": [
+        "ps",
+        "-aq",
+        "--filter",
+        "label=com.docker.compose.project=wse-fundamentals-e24"
+      ],
+      "exitCode": 0,
+      "durationMs": 21
+    },
+    {
+      "args": [
+        "network",
+        "ls",
+        "-q",
+        "--filter",
+        "label=com.docker.compose.project=wse-fundamentals-e24"
+      ],
+      "exitCode": 0,
+      "durationMs": 18
+    },
+    {
+      "args": [
+        "compose",
+        "--project-name",
+        "wse-fundamentals-e24",
+        "-f",
+        "compose.production.yaml",
+        "-f",
+        "evidence/phase-1/E24/compose.test.yaml",
+        "up",
+        "-d",
+        "--no-build",
+        "api",
+        "frontend",
+        "fixture"
+      ],
+      "exitCode": 0,
+      "durationMs": 609
+    },
+    {
+      "args": [
+        "compose",
+        "--project-name",
+        "wse-fundamentals-e24",
+        "-f",
+        "compose.production.yaml",
+        "-f",
+        "evidence/phase-1/E24/compose.test.yaml",
+        "up",
+        "-d",
+        "--no-build",
+        "worker"
+      ],
+      "exitCode": 0,
+      "durationMs": 845
+    },
+    {
+      "args": [
+        "compose",
+        "--project-name",
+        "wse-fundamentals-e24",
+        "-f",
+        "compose.production.yaml",
+        "-f",
+        "evidence/phase-1/E24/compose.test.yaml",
+        "ps",
+        "-q",
+        "api"
+      ],
+      "exitCode": 0,
+      "durationMs": 79
+    },
+    {
+      "args": [
+        "inspect",
+        "--format",
+        "{\"pid\":{{.State.Pid}},\"running\":{{.State.Running}},\"restarts\":{{.RestartCount}},\"image\":{{json .Image}},\"command\":{{json .Config.Cmd}}}",
+        "47ad14dca0c61a1c41de2bf7e1c6d496e3b8b6684a2bf2ebe6d0002b0106e7cf"
+      ],
+      "exitCode": 0,
+      "durationMs": 19
+    },
+    {
+      "args": [
+        "compose",
+        "--project-name",
+        "wse-fundamentals-e24",
+        "-f",
+        "compose.production.yaml",
+        "-f",
+        "evidence/phase-1/E24/compose.test.yaml",
+        "ps",
+        "-q",
+        "worker"
+      ],
+      "exitCode": 0,
+      "durationMs": 69
+    },
+    {
+      "args": [
+        "inspect",
+        "--format",
+        "{\"pid\":{{.State.Pid}},\"running\":{{.State.Running}},\"restarts\":{{.RestartCount}},\"image\":{{json .Image}},\"command\":{{json .Config.Cmd}}}",
+        "485f2d0ea7a8e73df491065936a769497a75a7d9bbc02c0fe4bf97dee4c1405e"
+      ],
+      "exitCode": 0,
+      "durationMs": 16
+    },
+    {
+      "args": [
+        "compose",
+        "--project-name",
+        "wse-fundamentals-e24",
+        "-f",
+        "compose.production.yaml",
+        "-f",
+        "evidence/phase-1/E24/compose.test.yaml",
+        "ps",
+        "-q",
+        "frontend"
+      ],
+      "exitCode": 0,
+      "durationMs": 73
+    },
+    {
+      "args": [
+        "inspect",
+        "--format",
+        "{\"pid\":{{.State.Pid}},\"running\":{{.State.Running}},\"restarts\":{{.RestartCount}},\"image\":{{json .Image}},\"command\":{{json .Config.Cmd}}}",
+        "82e34bb0bb9f282a7c31f5f2176ec02a21062c1f2b7101f9cfcf7b5498501439"
+      ],
+      "exitCode": 0,
+      "durationMs": 16
+    },
+    {
+      "args": [
+        "compose",
+        "--project-name",
+        "wse-fundamentals",
+        "-f",
+        "compose.yaml",
+        "stop",
+        "-t",
+        "5",
+        "postgres"
+      ],
+      "exitCode": 0,
+      "durationMs": 257
+    },
+    {
+      "args": [
+        "compose",
+        "--project-name",
+        "wse-fundamentals",
+        "-f",
+        "compose.yaml",
+        "up",
+        "--no-recreate",
+        "--no-build",
+        "--no-deps",
+        "--pull",
+        "never",
+        "--wait",
+        "--wait-timeout",
+        "30",
+        "postgres"
+      ],
+      "exitCode": 0,
+      "durationMs": 1799
+    },
+    {
+      "args": [
+        "compose",
+        "--project-name",
+        "wse-fundamentals-e24",
+        "-f",
+        "compose.production.yaml",
+        "-f",
+        "evidence/phase-1/E24/compose.test.yaml",
+        "ps",
+        "-q",
+        "api"
+      ],
+      "exitCode": 0,
+      "durationMs": 98
+    },
+    {
+      "args": [
+        "inspect",
+        "--format",
+        "{\"pid\":{{.State.Pid}},\"running\":{{.State.Running}},\"restarts\":{{.RestartCount}},\"image\":{{json .Image}},\"command\":{{json .Config.Cmd}}}",
+        "47ad14dca0c61a1c41de2bf7e1c6d496e3b8b6684a2bf2ebe6d0002b0106e7cf"
+      ],
+      "exitCode": 0,
+      "durationMs": 18
+    },
+    {
+      "args": [
+        "compose",
+        "--project-name",
+        "wse-fundamentals-e24",
+        "-f",
+        "compose.production.yaml",
+        "-f",
+        "evidence/phase-1/E24/compose.test.yaml",
+        "ps",
+        "-q",
+        "worker"
+      ],
+      "exitCode": 0,
+      "durationMs": 71
+    },
+    {
+      "args": [
+        "inspect",
+        "--format",
+        "{\"pid\":{{.State.Pid}},\"running\":{{.State.Running}},\"restarts\":{{.RestartCount}},\"image\":{{json .Image}},\"command\":{{json .Config.Cmd}}}",
+        "485f2d0ea7a8e73df491065936a769497a75a7d9bbc02c0fe4bf97dee4c1405e"
+      ],
+      "exitCode": 0,
+      "durationMs": 17
+    },
+    {
+      "args": [
+        "compose",
+        "--project-name",
+        "wse-fundamentals-e24",
+        "-f",
+        "compose.production.yaml",
+        "-f",
+        "evidence/phase-1/E24/compose.test.yaml",
+        "ps",
+        "-q",
+        "frontend"
+      ],
+      "exitCode": 0,
+      "durationMs": 73
+    },
+    {
+      "args": [
+        "inspect",
+        "--format",
+        "{\"pid\":{{.State.Pid}},\"running\":{{.State.Running}},\"restarts\":{{.RestartCount}},\"image\":{{json .Image}},\"command\":{{json .Config.Cmd}}}",
+        "82e34bb0bb9f282a7c31f5f2176ec02a21062c1f2b7101f9cfcf7b5498501439"
+      ],
+      "exitCode": 0,
+      "durationMs": 17
+    },
+    {
+      "args": [
+        "exec",
+        "47ad14dca0c61a1c41de2bf7e1c6d496e3b8b6684a2bf2ebe6d0002b0106e7cf",
+        "node",
+        "-e",
+        "const fs=require(\"node:fs\");const s=fs.readFileSync(\"/proc/1/status\",\"utf8\");console.log(JSON.stringify({uid:Number(/Uid:\\s+(\\d+)/.exec(s)[1]),executable:fs.readlinkSync(\"/proc/1/exe\"),actualCommandLine:fs.readFileSync(\"/proc/1/cmdline\",\"utf8\").split(\"\\0\").filter(Boolean),node:process.versions.node,nodeEnv:process.env.NODE_ENV,buildId:process.env.NODE_ENV===\"production\"&&fs.existsSync(\".next/BUILD_ID\")?fs.readFileSync(\".next/BUILD_ID\",\"utf8\").trim():null}));"
+      ],
+      "exitCode": 0,
+      "durationMs": 63
+    },
+    {
+      "args": [
+        "kill",
+        "--signal=SIGTERM",
+        "47ad14dca0c61a1c41de2bf7e1c6d496e3b8b6684a2bf2ebe6d0002b0106e7cf"
+      ],
+      "exitCode": 0,
+      "durationMs": 21
+    },
+    {
+      "args": [
+        "wait",
+        "47ad14dca0c61a1c41de2bf7e1c6d496e3b8b6684a2bf2ebe6d0002b0106e7cf"
+      ],
+      "exitCode": 0,
+      "durationMs": 273
+    },
+    {
+      "args": [
+        "exec",
+        "485f2d0ea7a8e73df491065936a769497a75a7d9bbc02c0fe4bf97dee4c1405e",
+        "node",
+        "-e",
+        "const fs=require(\"node:fs\");const s=fs.readFileSync(\"/proc/1/status\",\"utf8\");console.log(JSON.stringify({uid:Number(/Uid:\\s+(\\d+)/.exec(s)[1]),executable:fs.readlinkSync(\"/proc/1/exe\"),actualCommandLine:fs.readFileSync(\"/proc/1/cmdline\",\"utf8\").split(\"\\0\").filter(Boolean),node:process.versions.node,nodeEnv:process.env.NODE_ENV,buildId:process.env.NODE_ENV===\"production\"&&fs.existsSync(\".next/BUILD_ID\")?fs.readFileSync(\".next/BUILD_ID\",\"utf8\").trim():null}));"
+      ],
+      "exitCode": 0,
+      "durationMs": 91
+    },
+    {
+      "args": [
+        "kill",
+        "--signal=SIGTERM",
+        "485f2d0ea7a8e73df491065936a769497a75a7d9bbc02c0fe4bf97dee4c1405e"
+      ],
+      "exitCode": 0,
+      "durationMs": 23
+    },
+    {
+      "args": [
+        "wait",
+        "485f2d0ea7a8e73df491065936a769497a75a7d9bbc02c0fe4bf97dee4c1405e"
+      ],
+      "exitCode": 0,
+      "durationMs": 61
+    },
+    {
+      "args": [
+        "exec",
+        "82e34bb0bb9f282a7c31f5f2176ec02a21062c1f2b7101f9cfcf7b5498501439",
+        "node",
+        "-e",
+        "const fs=require(\"node:fs\");const s=fs.readFileSync(\"/proc/1/status\",\"utf8\");console.log(JSON.stringify({uid:Number(/Uid:\\s+(\\d+)/.exec(s)[1]),executable:fs.readlinkSync(\"/proc/1/exe\"),actualCommandLine:fs.readFileSync(\"/proc/1/cmdline\",\"utf8\").split(\"\\0\").filter(Boolean),node:process.versions.node,nodeEnv:process.env.NODE_ENV,buildId:process.env.NODE_ENV===\"production\"&&fs.existsSync(\".next/BUILD_ID\")?fs.readFileSync(\".next/BUILD_ID\",\"utf8\").trim():null}));"
+      ],
+      "exitCode": 0,
+      "durationMs": 96
+    },
+    {
+      "args": [
+        "kill",
+        "--signal=SIGTERM",
+        "82e34bb0bb9f282a7c31f5f2176ec02a21062c1f2b7101f9cfcf7b5498501439"
+      ],
+      "exitCode": 0,
+      "durationMs": 27
+    },
+    {
+      "args": [
+        "wait",
+        "82e34bb0bb9f282a7c31f5f2176ec02a21062c1f2b7101f9cfcf7b5498501439"
+      ],
+      "exitCode": 0,
+      "durationMs": 98
+    },
+    {
+      "args": [
+        "compose",
+        "--project-name",
+        "wse-fundamentals-e24",
+        "-f",
+        "compose.production.yaml",
+        "-f",
+        "evidence/phase-1/E24/compose.test.yaml",
+        "down"
+      ],
+      "exitCode": 0,
+      "durationMs": 656
+    }
+  ],
+  "hashes": {
+    "scenario.json": "07b5b3e2b4bd1cfc456c415134adce61f05dfdd729fe803968b32a4d813c5847",
+    "fixture.ts": "10d311f2e2af33063d09add435ed282b20341e2b87eeffa10b6287dce61ca784",
+    "baseline.mjs": "0a24805b65454cddc543d27708aec68092b34e28b67ca6ba605e3bc6602c79d6",
+    "compose.test.yaml": "e84ee2f893bb851ef630a6e9e34dde1daaadc9d8f4ea8b641a3f299b55c80fbe",
+    "fixture-server.mjs": "5ba0eeb56ec9e2e664aed5e32c0742366f6122051805a777966c894cc769cbb1",
+    "observer": "74b9369fd18a78f1a4d02faadbf2f35c0d4f9b116ffc6af70130eb7c53452cb4"
+  },
+  "postgresStops": 1,
+  "postgresRestores": 1,
+  "conditionPolls": 58,
+  "healthUp": [
+    {
+      "live": 200,
+      "ready": 200
+    },
+    {
+      "live": 200,
+      "ready": 200
+    }
+  ],
+  "worker": {
+    "accepted202": true,
+    "queuedUnknownStatus": true,
+    "positiveQueueAge": true,
+    "activeObserved": true,
+    "sameIdTerminal503": true,
+    "claims": 1,
+    "recovered": 0,
+    "recoveryScans": 2
+  },
+  "browser": {
+    "version": "151.0.7922.34",
+    "login": true,
+    "list": true,
+    "detailRows": 4,
+    "ownerIsolation": true,
+    "staticStatus": 200,
+    "serverRouteStatus": 200,
+    "consoleErrors": 0,
+    "screenshots": 0,
+    "traces": 0
+  },
+  "cardinality": {
+    "before": 35,
+    "after": 35,
+    "distinctMissingIds": 10,
+    "errorCountDelta": 10
+  },
+  "healthDown": [
+    {
+      "live": 200,
+      "ready": 503
+    },
+    {
+      "live": 200,
+      "ready": 503
+    }
+  ],
+  "healthRestored": [
+    {
+      "live": 200,
+      "ready": 200
+    },
+    {
+      "live": 200,
+      "ready": 200
+    }
+  ],
+  "outage": {
+    "rejectedStatus": 500,
+    "authoritativeRows": 5,
+    "preserved": true,
+    "noNewOutbound": true
+  },
+  "runtime": {
+    "api": {
+      "uid": 1000,
+      "executable": "/usr/local/bin/node",
+      "actualCommandLine": [
+        "node",
+        "server/main.ts"
+      ],
+      "node": "24.19.0",
+      "nodeEnv": "test",
+      "buildId": null,
+      "configuredCommand": [
+        "node",
+        "server/main.ts"
+      ],
+      "image": "sha256:2f6918aed823d07ac1e05980b7747bf49e0912a027326ef3c5713c98fc5bfbc8",
+      "hostPid": 67959,
+      "restarts": 0
+    },
+    "worker": {
+      "uid": 1000,
+      "executable": "/usr/local/bin/node",
+      "actualCommandLine": [
+        "node",
+        "server/worker.ts"
+      ],
+      "node": "24.19.0",
+      "nodeEnv": "test",
+      "buildId": null,
+      "configuredCommand": [
+        "node",
+        "server/worker.ts"
+      ],
+      "image": "sha256:2f6918aed823d07ac1e05980b7747bf49e0912a027326ef3c5713c98fc5bfbc8",
+      "hostPid": 68230,
+      "restarts": 0
+    },
+    "frontend": {
+      "uid": 1000,
+      "executable": "/usr/local/bin/node",
+      "actualCommandLine": [
+        "next-server (v"
+      ],
+      "node": "24.19.0",
+      "nodeEnv": "production",
+      "buildId": "1WFptx7upO35yWieeZ3XK",
+      "configuredCommand": [
+        "node",
+        "server.js"
+      ],
+      "image": "sha256:7cd37b250c2e45bcc99271b688a74336dcbee88d10a29251edc8cbb61f68de16",
+      "hostPid": 67957,
+      "restarts": 0
+    }
+  },
+  "signals": {
+    "api": {
+      "signal": "SIGTERM",
+      "exitCode": 0,
+      "elapsedMs": 294,
+      "forced": false
+    },
+    "worker": {
+      "signal": "SIGTERM",
+      "exitCode": 0,
+      "elapsedMs": 84,
+      "forced": false
+    }
+  },
+  "failure": {
+    "stage": "actual runtime UID and signal boundary",
+    "assertion": "Actual role SIGTERM exit must be clean and within its frozen bound.",
+    "kind": "Error"
+  },
+  "cleanup": {
+    "schemaDropped": true,
+    "runtimeRemoved": true,
+    "postgresRestored": true,
+    "portsFree": true,
+    "errors": []
+  },
+  "durationMs": 9183
+}
diff --git a/evidence/phase-1/E24/results/full-repair1.started.json b/evidence/phase-1/E24/results/full-repair1.started.json
new file mode 100644
index 0000000..ce52f29
--- /dev/null
+++ b/evidence/phase-1/E24/results/full-repair1.started.json
@@ -0,0 +1 @@
+{"mode":"full","actor":"repair1","at":"2026-08-28T07:19:29.633Z"}
diff --git a/evidence/phase-1/E24/results/full-repair2.console.json b/evidence/phase-1/E24/results/full-repair2.console.json
new file mode 100644
index 0000000..8425e22
--- /dev/null
+++ b/evidence/phase-1/E24/results/full-repair2.console.json
@@ -0,0 +1,22 @@
+{
+  "at": "2026-08-28T07:31:45.071926+00:00",
+  "command": [
+    "fnm",
+    "exec",
+    "--using",
+    "24.19.0",
+    "node",
+    "test/container-smoke.mjs",
+    "full",
+    "repair2"
+  ],
+  "scenarioInvocations": 1,
+  "automaticRetries": 0,
+  "consoleComplete": true,
+  "exitCode": 0,
+  "durationMs": 10177,
+  "console": "output/phase-1/e24/full-repair2.console.txt",
+  "consoleSha256": "9736a65e2d1ccd1110f8fa1053e39ba63a5d2e2e81325594d5e4029b1b56791d",
+  "nativeSha256": "3831ddb53ec3bc54ba4f0f7c0ba02cb3d683ff82f77a302df950ae1e1eac887e",
+  "nativeResult": "PASS"
+}
diff --git a/evidence/phase-1/E24/results/full-repair2.console.started.json b/evidence/phase-1/E24/results/full-repair2.console.started.json
new file mode 100644
index 0000000..3fbf52a
--- /dev/null
+++ b/evidence/phase-1/E24/results/full-repair2.console.started.json
@@ -0,0 +1,16 @@
+{
+  "at": "2026-08-28T07:31:45.071926+00:00",
+  "command": [
+    "fnm",
+    "exec",
+    "--using",
+    "24.19.0",
+    "node",
+    "test/container-smoke.mjs",
+    "full",
+    "repair2"
+  ],
+  "scenarioInvocations": 1,
+  "automaticRetries": 0,
+  "consoleComplete": true
+}
diff --git a/evidence/phase-1/E24/results/full-repair2.console.txt b/evidence/phase-1/E24/results/full-repair2.console.txt
new file mode 100644
index 0000000..a2dc8d4
--- /dev/null
+++ b/evidence/phase-1/E24/results/full-repair2.console.txt
@@ -0,0 +1 @@
+{"result":"PASS","mode":"full","actor":"repair2","durationMs":9725,"cleanup":{"schemaDropped":true,"runtimeRemoved":true,"postgresRestored":true,"portsFree":true,"errors":[]}}
diff --git a/evidence/phase-1/E24/results/full-repair2.json b/evidence/phase-1/E24/results/full-repair2.json
new file mode 100644
index 0000000..bf95887
--- /dev/null
+++ b/evidence/phase-1/E24/results/full-repair2.json
@@ -0,0 +1,549 @@
+{
+  "mode": "full",
+  "actor": "repair2",
+  "result": "PASS",
+  "commands": [
+    {
+      "args": [
+        "ps",
+        "-aq",
+        "--filter",
+        "label=com.docker.compose.project=wse-fundamentals-e24"
+      ],
+      "exitCode": 0,
+      "durationMs": 23
+    },
+    {
+      "args": [
+        "network",
+        "ls",
+        "-q",
+        "--filter",
+        "label=com.docker.compose.project=wse-fundamentals-e24"
+      ],
+      "exitCode": 0,
+      "durationMs": 20
+    },
+    {
+      "args": [
+        "compose",
+        "--project-name",
+        "wse-fundamentals-e24",
+        "-f",
+        "compose.production.yaml",
+        "-f",
+        "evidence/phase-1/E24/compose.test.yaml",
+        "up",
+        "-d",
+        "--no-build",
+        "api",
+        "frontend",
+        "fixture"
+      ],
+      "exitCode": 0,
+      "durationMs": 831
+    },
+    {
+      "args": [
+        "compose",
+        "--project-name",
+        "wse-fundamentals-e24",
+        "-f",
+        "compose.production.yaml",
+        "-f",
+        "evidence/phase-1/E24/compose.test.yaml",
+        "up",
+        "-d",
+        "--no-build",
+        "worker"
+      ],
+      "exitCode": 0,
+      "durationMs": 928
+    },
+    {
+      "args": [
+        "compose",
+        "--project-name",
+        "wse-fundamentals-e24",
+        "-f",
+        "compose.production.yaml",
+        "-f",
+        "evidence/phase-1/E24/compose.test.yaml",
+        "ps",
+        "-q",
+        "api"
+      ],
+      "exitCode": 0,
+      "durationMs": 76
+    },
+    {
+      "args": [
+        "inspect",
+        "--format",
+        "{\"pid\":{{.State.Pid}},\"running\":{{.State.Running}},\"restarts\":{{.RestartCount}},\"image\":{{json .Image}},\"command\":{{json .Config.Cmd}}}",
+        "59adb79851ee369b565eb20c3aef6d0b1fa498c9b542705cc8f127e21d4bf68b"
+      ],
+      "exitCode": 0,
+      "durationMs": 19
+    },
+    {
+      "args": [
+        "compose",
+        "--project-name",
+        "wse-fundamentals-e24",
+        "-f",
+        "compose.production.yaml",
+        "-f",
+        "evidence/phase-1/E24/compose.test.yaml",
+        "ps",
+        "-q",
+        "worker"
+      ],
+      "exitCode": 0,
+      "durationMs": 86
+    },
+    {
+      "args": [
+        "inspect",
+        "--format",
+        "{\"pid\":{{.State.Pid}},\"running\":{{.State.Running}},\"restarts\":{{.RestartCount}},\"image\":{{json .Image}},\"command\":{{json .Config.Cmd}}}",
+        "3a89ac4f5d510cef0f50112ad1c290de6ecf4e4bc5e8777b7954f10fdb229c4c"
+      ],
+      "exitCode": 0,
+      "durationMs": 20
+    },
+    {
+      "args": [
+        "compose",
+        "--project-name",
+        "wse-fundamentals-e24",
+        "-f",
+        "compose.production.yaml",
+        "-f",
+        "evidence/phase-1/E24/compose.test.yaml",
+        "ps",
+        "-q",
+        "frontend"
+      ],
+      "exitCode": 0,
+      "durationMs": 83
+    },
+    {
+      "args": [
+        "inspect",
+        "--format",
+        "{\"pid\":{{.State.Pid}},\"running\":{{.State.Running}},\"restarts\":{{.RestartCount}},\"image\":{{json .Image}},\"command\":{{json .Config.Cmd}}}",
+        "b38fc72bfee47750ba04c0e736323a056c43c94ddfa7d42d8df16a7d6bfc2bbd"
+      ],
+      "exitCode": 0,
+      "durationMs": 19
+    },
+    {
+      "args": [
+        "compose",
+        "--project-name",
+        "wse-fundamentals",
+        "-f",
+        "compose.yaml",
+        "stop",
+        "-t",
+        "5",
+        "postgres"
+      ],
+      "exitCode": 0,
+      "durationMs": 237
+    },
+    {
+      "args": [
+        "compose",
+        "--project-name",
+        "wse-fundamentals",
+        "-f",
+        "compose.yaml",
+        "up",
+        "--no-recreate",
+        "--no-build",
+        "--no-deps",
+        "--pull",
+        "never",
+        "--wait",
+        "--wait-timeout",
+        "30",
+        "postgres"
+      ],
+      "exitCode": 0,
+      "durationMs": 1744
+    },
+    {
+      "args": [
+        "compose",
+        "--project-name",
+        "wse-fundamentals-e24",
+        "-f",
+        "compose.production.yaml",
+        "-f",
+        "evidence/phase-1/E24/compose.test.yaml",
+        "ps",
+        "-q",
+        "api"
+      ],
+      "exitCode": 0,
+      "durationMs": 84
+    },
+    {
+      "args": [
+        "inspect",
+        "--format",
+        "{\"pid\":{{.State.Pid}},\"running\":{{.State.Running}},\"restarts\":{{.RestartCount}},\"image\":{{json .Image}},\"command\":{{json .Config.Cmd}}}",
+        "59adb79851ee369b565eb20c3aef6d0b1fa498c9b542705cc8f127e21d4bf68b"
+      ],
+      "exitCode": 0,
+      "durationMs": 19
+    },
+    {
+      "args": [
+        "compose",
+        "--project-name",
+        "wse-fundamentals-e24",
+        "-f",
+        "compose.production.yaml",
+        "-f",
+        "evidence/phase-1/E24/compose.test.yaml",
+        "ps",
+        "-q",
+        "worker"
+      ],
+      "exitCode": 0,
+      "durationMs": 82
+    },
+    {
+      "args": [
+        "inspect",
+        "--format",
+        "{\"pid\":{{.State.Pid}},\"running\":{{.State.Running}},\"restarts\":{{.RestartCount}},\"image\":{{json .Image}},\"command\":{{json .Config.Cmd}}}",
+        "3a89ac4f5d510cef0f50112ad1c290de6ecf4e4bc5e8777b7954f10fdb229c4c"
+      ],
+      "exitCode": 0,
+      "durationMs": 17
+    },
+    {
+      "args": [
+        "compose",
+        "--project-name",
+        "wse-fundamentals-e24",
+        "-f",
+        "compose.production.yaml",
+        "-f",
+        "evidence/phase-1/E24/compose.test.yaml",
+        "ps",
+        "-q",
+        "frontend"
+      ],
+      "exitCode": 0,
+      "durationMs": 85
+    },
+    {
+      "args": [
+        "inspect",
+        "--format",
+        "{\"pid\":{{.State.Pid}},\"running\":{{.State.Running}},\"restarts\":{{.RestartCount}},\"image\":{{json .Image}},\"command\":{{json .Config.Cmd}}}",
+        "b38fc72bfee47750ba04c0e736323a056c43c94ddfa7d42d8df16a7d6bfc2bbd"
+      ],
+      "exitCode": 0,
+      "durationMs": 19
+    },
+    {
+      "args": [
+        "exec",
+        "59adb79851ee369b565eb20c3aef6d0b1fa498c9b542705cc8f127e21d4bf68b",
+        "node",
+        "-e",
+        "const fs=require(\"node:fs\");const s=fs.readFileSync(\"/proc/1/status\",\"utf8\");console.log(JSON.stringify({uid:Number(/Uid:\\s+(\\d+)/.exec(s)[1]),executable:fs.readlinkSync(\"/proc/1/exe\"),actualCommandLine:fs.readFileSync(\"/proc/1/cmdline\",\"utf8\").split(\"\\0\").filter(Boolean),node:process.versions.node,nodeEnv:process.env.NODE_ENV,buildId:process.env.NODE_ENV===\"production\"&&fs.existsSync(\".next/BUILD_ID\")?fs.readFileSync(\".next/BUILD_ID\",\"utf8\").trim():null}));"
+      ],
+      "exitCode": 0,
+      "durationMs": 80
+    },
+    {
+      "args": [
+        "kill",
+        "--signal=SIGTERM",
+        "59adb79851ee369b565eb20c3aef6d0b1fa498c9b542705cc8f127e21d4bf68b"
+      ],
+      "exitCode": 0,
+      "durationMs": 25
+    },
+    {
+      "args": [
+        "wait",
+        "59adb79851ee369b565eb20c3aef6d0b1fa498c9b542705cc8f127e21d4bf68b"
+      ],
+      "exitCode": 0,
+      "durationMs": 261
+    },
+    {
+      "args": [
+        "exec",
+        "3a89ac4f5d510cef0f50112ad1c290de6ecf4e4bc5e8777b7954f10fdb229c4c",
+        "node",
+        "-e",
+        "const fs=require(\"node:fs\");const s=fs.readFileSync(\"/proc/1/status\",\"utf8\");console.log(JSON.stringify({uid:Number(/Uid:\\s+(\\d+)/.exec(s)[1]),executable:fs.readlinkSync(\"/proc/1/exe\"),actualCommandLine:fs.readFileSync(\"/proc/1/cmdline\",\"utf8\").split(\"\\0\").filter(Boolean),node:process.versions.node,nodeEnv:process.env.NODE_ENV,buildId:process.env.NODE_ENV===\"production\"&&fs.existsSync(\".next/BUILD_ID\")?fs.readFileSync(\".next/BUILD_ID\",\"utf8\").trim():null}));"
+      ],
+      "exitCode": 0,
+      "durationMs": 95
+    },
+    {
+      "args": [
+        "kill",
+        "--signal=SIGTERM",
+        "3a89ac4f5d510cef0f50112ad1c290de6ecf4e4bc5e8777b7954f10fdb229c4c"
+      ],
+      "exitCode": 0,
+      "durationMs": 25
+    },
+    {
+      "args": [
+        "wait",
+        "3a89ac4f5d510cef0f50112ad1c290de6ecf4e4bc5e8777b7954f10fdb229c4c"
+      ],
+      "exitCode": 0,
+      "durationMs": 165
+    },
+    {
+      "args": [
+        "exec",
+        "b38fc72bfee47750ba04c0e736323a056c43c94ddfa7d42d8df16a7d6bfc2bbd",
+        "node",
+        "-e",
+        "const fs=require(\"node:fs\");const s=fs.readFileSync(\"/proc/1/status\",\"utf8\");console.log(JSON.stringify({uid:Number(/Uid:\\s+(\\d+)/.exec(s)[1]),executable:fs.readlinkSync(\"/proc/1/exe\"),actualCommandLine:fs.readFileSync(\"/proc/1/cmdline\",\"utf8\").split(\"\\0\").filter(Boolean),node:process.versions.node,nodeEnv:process.env.NODE_ENV,buildId:process.env.NODE_ENV===\"production\"&&fs.existsSync(\".next/BUILD_ID\")?fs.readFileSync(\".next/BUILD_ID\",\"utf8\").trim():null}));"
+      ],
+      "exitCode": 0,
+      "durationMs": 95
+    },
+    {
+      "args": [
+        "kill",
+        "--signal=SIGTERM",
+        "b38fc72bfee47750ba04c0e736323a056c43c94ddfa7d42d8df16a7d6bfc2bbd"
+      ],
+      "exitCode": 0,
+      "durationMs": 28
+    },
+    {
+      "args": [
+        "wait",
+        "b38fc72bfee47750ba04c0e736323a056c43c94ddfa7d42d8df16a7d6bfc2bbd"
+      ],
+      "exitCode": 0,
+      "durationMs": 123
+    },
+    {
+      "args": [
+        "logs",
+        "59adb79851ee369b565eb20c3aef6d0b1fa498c9b542705cc8f127e21d4bf68b"
+      ],
+      "exitCode": 0,
+      "durationMs": 34
+    },
+    {
+      "args": [
+        "logs",
+        "3a89ac4f5d510cef0f50112ad1c290de6ecf4e4bc5e8777b7954f10fdb229c4c"
+      ],
+      "exitCode": 0,
+      "durationMs": 30
+    },
+    {
+      "args": [
+        "logs",
+        "b38fc72bfee47750ba04c0e736323a056c43c94ddfa7d42d8df16a7d6bfc2bbd"
+      ],
+      "exitCode": 0,
+      "durationMs": 31
+    },
+    {
+      "args": [
+        "compose",
+        "--project-name",
+        "wse-fundamentals-e24",
+        "-f",
+        "compose.production.yaml",
+        "-f",
+        "evidence/phase-1/E24/compose.test.yaml",
+        "down"
+      ],
+      "exitCode": 0,
+      "durationMs": 657
+    }
+  ],
+  "hashes": {
+    "scenario.json": "07b5b3e2b4bd1cfc456c415134adce61f05dfdd729fe803968b32a4d813c5847",
+    "fixture.ts": "10d311f2e2af33063d09add435ed282b20341e2b87eeffa10b6287dce61ca784",
+    "baseline.mjs": "0a24805b65454cddc543d27708aec68092b34e28b67ca6ba605e3bc6602c79d6",
+    "compose.test.yaml": "e84ee2f893bb851ef630a6e9e34dde1daaadc9d8f4ea8b641a3f299b55c80fbe",
+    "fixture-server.mjs": "5ba0eeb56ec9e2e664aed5e32c0742366f6122051805a777966c894cc769cbb1",
+    "observer": "71566e13a522a1b83f786eaca976d2d1475d60e5a26a76d4f64a9c03859cffe5"
+  },
+  "postgresStops": 1,
+  "postgresRestores": 1,
+  "conditionPolls": 65,
+  "healthUp": [
+    {
+      "live": 200,
+      "ready": 200
+    },
+    {
+      "live": 200,
+      "ready": 200
+    }
+  ],
+  "worker": {
+    "accepted202": true,
+    "queuedUnknownStatus": true,
+    "positiveQueueAge": true,
+    "activeObserved": true,
+    "sameIdTerminal503": true,
+    "claims": 1,
+    "recovered": 0,
+    "recoveryScans": 2
+  },
+  "browser": {
+    "version": "151.0.7922.34",
+    "login": true,
+    "list": true,
+    "detailRows": 4,
+    "ownerIsolation": true,
+    "staticStatus": 200,
+    "serverRouteStatus": 200,
+    "consoleErrors": 0,
+    "screenshots": 0,
+    "traces": 0
+  },
+  "cardinality": {
+    "before": 35,
+    "after": 35,
+    "distinctMissingIds": 10,
+    "errorCountDelta": 10
+  },
+  "healthDown": [
+    {
+      "live": 200,
+      "ready": 503
+    },
+    {
+      "live": 200,
+      "ready": 503
+    }
+  ],
+  "healthRestored": [
+    {
+      "live": 200,
+      "ready": 200
+    },
+    {
+      "live": 200,
+      "ready": 200
+    }
+  ],
+  "outage": {
+    "rejectedStatus": 500,
+    "authoritativeRows": 5,
+    "preserved": true,
+    "noNewOutbound": true
+  },
+  "runtime": {
+    "api": {
+      "uid": 1000,
+      "executable": "/usr/local/bin/node",
+      "actualCommandLine": [
+        "node",
+        "server/main.ts"
+      ],
+      "node": "24.19.0",
+      "nodeEnv": "test",
+      "buildId": null,
+      "configuredCommand": [
+        "node",
+        "server/main.ts"
+      ],
+      "image": "sha256:2f6918aed823d07ac1e05980b7747bf49e0912a027326ef3c5713c98fc5bfbc8",
+      "hostPid": 95570,
+      "restarts": 0
+    },
+    "worker": {
+      "uid": 1000,
+      "executable": "/usr/local/bin/node",
+      "actualCommandLine": [
+        "node",
+        "server/worker.ts"
+      ],
+      "node": "24.19.0",
+      "nodeEnv": "test",
+      "buildId": null,
+      "configuredCommand": [
+        "node",
+        "server/worker.ts"
+      ],
+      "image": "sha256:2f6918aed823d07ac1e05980b7747bf49e0912a027326ef3c5713c98fc5bfbc8",
+      "hostPid": 95862,
+      "restarts": 0
+    },
+    "frontend": {
+      "uid": 1000,
+      "executable": "/usr/local/bin/node",
+      "actualCommandLine": [
+        "next-server (v"
+      ],
+      "node": "24.19.0",
+      "nodeEnv": "production",
+      "buildId": "1WFptx7upO35yWieeZ3XK",
+      "configuredCommand": [
+        "node",
+        "server.js"
+      ],
+      "image": "sha256:4294a766c1255bb773973ee00af8670c1eaf3315991844d938ebb5ad5396da67",
+      "hostPid": 95539,
+      "restarts": 0
+    }
+  },
+  "signals": {
+    "api": {
+      "signal": "SIGTERM",
+      "exitCode": 0,
+      "elapsedMs": 286,
+      "forced": false
+    },
+    "worker": {
+      "signal": "SIGTERM",
+      "exitCode": 0,
+      "elapsedMs": 190,
+      "forced": false
+    },
+    "frontend": {
+      "signal": "SIGTERM",
+      "exitCode": 0,
+      "elapsedMs": 151,
+      "forced": false
+    }
+  },
+  "logging": {
+    "apiJsonLines": 50,
+    "workerJsonLines": 18,
+    "correlatedCheck": true,
+    "signalHandlersObserved": true,
+    "sentinelLeaks": 0,
+    "scannedRuntimeSentinels": 15,
+    "rawLogsRetained": false
+  },
+  "metrics": [
+    "http_request_duration_seconds_sum{role=\"api\",route=\"/health/ready\",method=\"GET\",status=\"200\"} 0.008042916000000105\nhttp_request_duration_seconds_count{role=\"api\",route=\"/health/ready\",method=\"GET\",status=\"200\"} 4\nhttp_errors_total{role=\"api\",route=\"/health/ready\",method=\"GET\",status=\"200\"} 0\nhttp_request_duration_seconds_sum{role=\"api\",route=\"/auth/login\",method=\"POST\",status=\"200\"} 0.9058377499999999\nhttp_request_duration_seconds_count{role=\"api\",route=\"/auth/login\",method=\"POST\",status=\"200\"} 3\nhttp_errors_total{role=\"api\",route=\"/auth/login\",method=\"POST\",status=\"200\"} 0\nhttp_request_duration_seconds_sum{role=\"api\",route=\"/auth/csrf\",method=\"GET\",status=\"200\"} 0.0027249159999998937\nhttp_request_duration_seconds_count{role=\"api\",route=\"/auth/csrf\",method=\"GET\",status=\"200\"} 1\nhttp_errors_total{role=\"api\",route=\"/auth/csrf\",method=\"GET\",status=\"200\"} 0\nhttp_request_duration_seconds_sum{role=\"api\",route=\"/monitors/:id/checks\",method=\"POST\",status=\"202\"} 0.008813750000000026\nhttp_request_duration_seconds_count{role=\"api\",route=\"/monitors/:id/checks\",method=\"POST\",status=\"202\"} 1\nhttp_errors_total{role=\"api\",route=\"/monitors/:id/checks\",method=\"POST\",status=\"202\"} 0\nhttp_request_duration_seconds_sum{role=\"api\",route=\"/metrics\",method=\"GET\",status=\"200\"} 0.008131708999999546\nhttp_request_duration_seconds_count{role=\"api\",route=\"/metrics\",method=\"GET\",status=\"200\"} 4\nhttp_errors_total{role=\"api\",route=\"/metrics\",method=\"GET\",status=\"200\"} 0\nhttp_request_duration_seconds_sum{role=\"api\",route=\"/checks/:id\",method=\"GET\",status=\"200\"} 0.01552316800000017\nhttp_request_duration_seconds_count{role=\"api\",route=\"/checks/:id\",method=\"GET\",status=\"200\"} 9\nhttp_errors_total{role=\"api\",route=\"/checks/:id\",method=\"GET\",status=\"200\"} 0\nhttp_request_duration_seconds_sum{role=\"api\",route=\"/health/live\",method=\"GET\",status=\"200\"} 0.0007430430000003981\nhttp_request_duration_seconds_count{role=\"api\",route=\"/health/live\",method=\"GET\",status=\"200\"} 3\nhttp_errors_total{role=\"api\",route=\"/health/live\",method=\"GET\",status=\"200\"} 0\nhttp_request_duration_seconds_sum{role=\"api\",route=\"/monitors\",method=\"GET\",status=\"200\"} 0.006110251000000517\nhttp_request_duration_seconds_count{role=\"api\",route=\"/monitors\",method=\"GET\",status=\"200\"} 3\nhttp_errors_total{role=\"api\",route=\"/monitors\",method=\"GET\",status=\"200\"} 0\nhttp_request_duration_seconds_sum{role=\"api\",route=\"/monitors/:id/checks\",method=\"GET\",status=\"404\"} 0.0164859990000009\nhttp_request_duration_seconds_count{role=\"api\",route=\"/monitors/:id/checks\",method=\"GET\",status=\"404\"} 12\nhttp_errors_total{role=\"api\",route=\"/monitors/:id/checks\",method=\"GET\",status=\"404\"} 12\nhttp_request_duration_seconds_sum{role=\"api\",route=\"/monitors/:id/checks\",method=\"GET\",status=\"200\"} 0.007162374999999429\nhttp_request_duration_seconds_count{role=\"api\",route=\"/monitors/:id/checks\",method=\"GET\",status=\"200\"} 2\nhttp_errors_total{role=\"api\",route=\"/monitors/:id/checks\",method=\"GET\",status=\"200\"} 0\nhttp_request_duration_seconds_sum{role=\"api\",route=\"/auth/session\",method=\"GET\",status=\"200\"} 0.0009572920000000523\nhttp_request_duration_seconds_count{role=\"api\",route=\"/auth/session\",method=\"GET\",status=\"200\"} 1\nhttp_errors_total{role=\"api\",route=\"/auth/session\",method=\"GET\",status=\"200\"} 0\nhttp_request_duration_seconds_sum{role=\"api\",route=\"/health/ready\",method=\"GET\",status=\"503\"} 0.004780708000000231\nhttp_request_duration_seconds_count{role=\"api\",route=\"/health/ready\",method=\"GET\",status=\"503\"} 1\nhttp_errors_total{role=\"api\",route=\"/health/ready\",method=\"GET\",status=\"503\"} 1\nhttp_request_duration_seconds_sum{role=\"api\",route=\"/monitors/:id/checks\",method=\"POST\",status=\"500\"} 0.00415795799999978\nhttp_request_duration_seconds_count{role=\"api\",route=\"/monitors/:id/checks\",method=\"POST\",status=\"500\"} 1\nhttp_errors_total{role=\"api\",route=\"/monitors/:id/checks\",method=\"POST\",status=\"500\"} 1\npostgres_ready{role=\"api\"} 1\ncheck_queue_age_seconds{role=\"api\"} 0\n",
+    "http_request_duration_seconds_sum{role=\"worker\",route=\"/metrics\",method=\"GET\",status=\"200\"} 0.009836332999999798\nhttp_request_duration_seconds_count{role=\"worker\",route=\"/metrics\",method=\"GET\",status=\"200\"} 3\nhttp_errors_total{role=\"worker\",route=\"/metrics\",method=\"GET\",status=\"200\"} 0\nhttp_request_duration_seconds_sum{role=\"worker\",route=\"/health/live\",method=\"GET\",status=\"200\"} 0.0009635410000000207\nhttp_request_duration_seconds_count{role=\"worker\",route=\"/health/live\",method=\"GET\",status=\"200\"} 3\nhttp_errors_total{role=\"worker\",route=\"/health/live\",method=\"GET\",status=\"200\"} 0\nhttp_request_duration_seconds_sum{role=\"worker\",route=\"/health/ready\",method=\"GET\",status=\"200\"} 0.00195316699999961\nhttp_request_duration_seconds_count{role=\"worker\",route=\"/health/ready\",method=\"GET\",status=\"200\"} 3\nhttp_errors_total{role=\"worker\",route=\"/health/ready\",method=\"GET\",status=\"200\"} 0\nhttp_request_duration_seconds_sum{role=\"worker\",route=\"/health/ready\",method=\"GET\",status=\"503\"} 0.005211874999999964\nhttp_request_duration_seconds_count{role=\"worker\",route=\"/health/ready\",method=\"GET\",status=\"503\"} 1\nhttp_errors_total{role=\"worker\",route=\"/health/ready\",method=\"GET\",status=\"503\"} 1\npostgres_ready{role=\"worker\"} 1\ncheck_queue_age_seconds{role=\"worker\"} 0\nworker_active{role=\"worker\"} 0\nworker_claims_total{role=\"worker\"} 1\nworker_recovery_runs_total{role=\"worker\"} 15\nworker_recovered_checks_total{role=\"worker\"} 0\n"
+  ],
+  "cleanup": {
+    "schemaDropped": true,
+    "runtimeRemoved": true,
+    "postgresRestored": true,
+    "portsFree": true,
+    "errors": []
+  },
+  "durationMs": 9725
+}
diff --git a/evidence/phase-1/E24/results/full-repair2.started.json b/evidence/phase-1/E24/results/full-repair2.started.json
new file mode 100644
index 0000000..2bce192
--- /dev/null
+++ b/evidence/phase-1/E24/results/full-repair2.started.json
@@ -0,0 +1 @@
+{"mode":"full","actor":"repair2","at":"2026-08-28T07:31:45.502Z"}
diff --git a/evidence/phase-1/E24/results/repair1-cleanup-inspection.json b/evidence/phase-1/E24/results/repair1-cleanup-inspection.json
new file mode 100644
index 0000000..c609898
--- /dev/null
+++ b/evidence/phase-1/E24/results/repair1-cleanup-inspection.json
@@ -0,0 +1,94 @@
+{
+  "purpose": "Read-only failed-fixture cleanup inspection; no runtime values leave PostgreSQL",
+  "inspection": {
+    "otherConnections": 0,
+    "publicRelations": 0,
+    "publicCatalogSha256": "4f53cda18c2baa0c0354bb5f9a3ecbe5ed12ab4d8e11ba873c2f11161202b945",
+    "usersAreFrozenPair": true,
+    "monitorsAreFrozenPair": true,
+    "activeChecks": 0,
+    "checkStates": {
+      "FAILED": 4,
+      "SUCCEEDED": 1
+    },
+    "tables": [
+      {
+        "table_name": "check_runs",
+        "rows": 5,
+        "rows_sha256": "fc46112d4f6633f991e7dd7f57da0567223cff3acb44d1fd2e08be3e48c415c6"
+      },
+      {
+        "table_name": "monitors",
+        "rows": 2,
+        "rows_sha256": "944f438648c7c2276ff595c726b88129c37f9b012fba8fc67b099e0432d9cda8"
+      },
+      {
+        "table_name": "schema_migrations",
+        "rows": 10,
+        "rows_sha256": "0e92539a36041d641b0abf57629011c44b31e45b2cb151104df622f4dadf112d"
+      },
+      {
+        "table_name": "sessions",
+        "rows": 3,
+        "rows_sha256": "85cbf2b3413f918315b28f769ca89ebc1aaaa11ff76c002472457daffa11c41f"
+      },
+      {
+        "table_name": "users",
+        "rows": 2,
+        "rows_sha256": "56876a04c1c6c998dd434d9a18e74f004d4d8231f4d83f2f913caef928fa1c9e"
+      }
+    ]
+  },
+  "approved": {
+    "args": [
+      "docker",
+      "exec",
+      "d37ee72dd39c",
+      "psql",
+      "-X",
+      "-qAt",
+      "-U",
+      "monitor",
+      "-d",
+      "monitor",
+      "-v",
+      "ON_ERROR_STOP=1",
+      "-c",
+      "\nSELECT json_build_object(\n  'otherConnections', (SELECT count(*) FROM pg_stat_activity WHERE datname=current_database() AND pid<>pg_backend_pid()),\n  'publicRelations', (SELECT count(*) FROM pg_class c JOIN pg_namespace n ON n.oid=c.relnamespace WHERE n.nspname='public'),\n  'publicCatalogSha256', (SELECT encode(sha256(convert_to(COALESCE(jsonb_agg(jsonb_build_object('name',c.relname,'kind',c.relkind,'owner',c.relowner) ORDER BY c.relname)::text,'[]'),'UTF8')),'hex') FROM pg_class c JOIN pg_namespace n ON n.oid=c.relnamespace WHERE n.nspname='public'),\n  'usersAreFrozenPair', (SELECT count(*)=2 AND bool_and(id IN ('24000000-0000-4000-9000-000000000001','24000000-0000-4000-9000-000000000002')) FROM e24_container.users),\n  'monitorsAreFrozenPair', (SELECT count(*)=2 AND bool_and((id='24000000-0000-4000-8000-000000000001' AND owner_user_id='24000000-0000-4000-9000-000000000001') OR (id='24000000-0000-4000-8000-000000000002' AND owner_user_id='24000000-0000-4000-9000-000000000002')) FROM e24_container.monitors),\n  'activeChecks', (SELECT count(*) FROM e24_container.check_runs WHERE state IN ('QUEUED','RUNNING')),\n  'checkStates', (SELECT json_object_agg(state,n) FROM (SELECT state,count(*) n FROM e24_container.check_runs GROUP BY state) s),\n  'tables', (\n    SELECT json_agg(row_to_json(s) ORDER BY s.table_name) FROM (\n      SELECT 'users' AS table_name,count(*) AS rows,encode(sha256(convert_to(COALESCE(jsonb_agg(to_jsonb(t) ORDER BY to_jsonb(t)::text)::text,'[]'),'UTF8')),'hex') AS rows_sha256 FROM e24_container.users t\nUNION ALL\nSELECT 'sessions' AS table_name,count(*) AS rows,encode(sha256(convert_to(COALESCE(jsonb_agg(to_jsonb(t) ORDER BY to_jsonb(t)::text)::text,'[]'),'UTF8')),'hex') AS rows_sha256 FROM e24_container.sessions t\nUNION ALL\nSELECT 'monitors' AS table_name,count(*) AS rows,encode(sha256(convert_to(COALESCE(jsonb_agg(to_jsonb(t) ORDER BY to_jsonb(t)::text)::text,'[]'),'UTF8')),'hex') AS rows_sha256 FROM e24_container.monitors t\nUNION ALL\nSELECT 'check_runs' AS table_name,count(*) AS rows,encode(sha256(convert_to(COALESCE(jsonb_agg(to_jsonb(t) ORDER BY to_jsonb(t)::text)::text,'[]'),'UTF8')),'hex') AS rows_sha256 FROM e24_container.check_runs t\nUNION ALL\nSELECT 'schema_migrations' AS table_name,count(*) AS rows,encode(sha256(convert_to(COALESCE(jsonb_agg(to_jsonb(t) ORDER BY to_jsonb(t)::text)::text,'[]'),'UTF8')),'hex') AS rows_sha256 FROM e24_container.schema_migrations t\n    ) s\n  )\n);"
+    ],
+    "sql": "\nSELECT json_build_object(\n  'otherConnections', (SELECT count(*) FROM pg_stat_activity WHERE datname=current_database() AND pid<>pg_backend_pid()),\n  'publicRelations', (SELECT count(*) FROM pg_class c JOIN pg_namespace n ON n.oid=c.relnamespace WHERE n.nspname='public'),\n  'publicCatalogSha256', (SELECT encode(sha256(convert_to(COALESCE(jsonb_agg(jsonb_build_object('name',c.relname,'kind',c.relkind,'owner',c.relowner) ORDER BY c.relname)::text,'[]'),'UTF8')),'hex') FROM pg_class c JOIN pg_namespace n ON n.oid=c.relnamespace WHERE n.nspname='public'),\n  'usersAreFrozenPair', (SELECT count(*)=2 AND bool_and(id IN ('24000000-0000-4000-9000-000000000001','24000000-0000-4000-9000-000000000002')) FROM e24_container.users),\n  'monitorsAreFrozenPair', (SELECT count(*)=2 AND bool_and((id='24000000-0000-4000-8000-000000000001' AND owner_user_id='24000000-0000-4000-9000-000000000001') OR (id='24000000-0000-4000-8000-000000000002' AND owner_user_id='24000000-0000-4000-9000-000000000002')) FROM e24_container.monitors),\n  'activeChecks', (SELECT count(*) FROM e24_container.check_runs WHERE state IN ('QUEUED','RUNNING')),\n  'checkStates', (SELECT json_object_agg(state,n) FROM (SELECT state,count(*) n FROM e24_container.check_runs GROUP BY state) s),\n  'tables', (\n    SELECT json_agg(row_to_json(s) ORDER BY s.table_name) FROM (\n      SELECT 'users' AS table_name,count(*) AS rows,encode(sha256(convert_to(COALESCE(jsonb_agg(to_jsonb(t) ORDER BY to_jsonb(t)::text)::text,'[]'),'UTF8')),'hex') AS rows_sha256 FROM e24_container.users t\nUNION ALL\nSELECT 'sessions' AS table_name,count(*) AS rows,encode(sha256(convert_to(COALESCE(jsonb_agg(to_jsonb(t) ORDER BY to_jsonb(t)::text)::text,'[]'),'UTF8')),'hex') AS rows_sha256 FROM e24_container.sessions t\nUNION ALL\nSELECT 'monitors' AS table_name,count(*) AS rows,encode(sha256(convert_to(COALESCE(jsonb_agg(to_jsonb(t) ORDER BY to_jsonb(t)::text)::text,'[]'),'UTF8')),'hex') AS rows_sha256 FROM e24_container.monitors t\nUNION ALL\nSELECT 'check_runs' AS table_name,count(*) AS rows,encode(sha256(convert_to(COALESCE(jsonb_agg(to_jsonb(t) ORDER BY to_jsonb(t)::text)::text,'[]'),'UTF8')),'hex') AS rows_sha256 FROM e24_container.check_runs t\nUNION ALL\nSELECT 'schema_migrations' AS table_name,count(*) AS rows,encode(sha256(convert_to(COALESCE(jsonb_agg(to_jsonb(t) ORDER BY to_jsonb(t)::text)::text,'[]'),'UTF8')),'hex') AS rows_sha256 FROM e24_container.schema_migrations t\n    ) s\n  )\n);",
+    "result": {
+      "chunk_id": "d6d6d6",
+      "wall_time_seconds": 0.000004791,
+      "exit_code": 0,
+      "original_token_count": 220,
+      "output": "{\"otherConnections\" : 0, \"publicRelations\" : 0, \"publicCatalogSha256\" : \"4f53cda18c2baa0c0354bb5f9a3ecbe5ed12ab4d8e11ba873c2f11161202b945\", \"usersAreFrozenPair\" : true, \"monitorsAreFrozenPair\" : true, \"activeChecks\" : 0, \"checkStates\" : { \"FAILED\" : 4, \"SUCCEEDED\" : 1 }, \"tables\" : [{\"table_name\":\"check_runs\",\"rows\":5,\"rows_sha256\":\"fc46112d4f6633f991e7dd7f57da0567223cff3acb44d1fd2e08be3e48c415c6\"}, {\"table_name\":\"monitors\",\"rows\":2,\"rows_sha256\":\"944f438648c7c2276ff595c726b88129c37f9b012fba8fc67b099e0432d9cda8\"}, {\"table_name\":\"schema_migrations\",\"rows\":10,\"rows_sha256\":\"0e92539a36041d641b0abf57629011c44b31e45b2cb151104df622f4dadf112d\"}, {\"table_name\":\"sessions\",\"rows\":3,\"rows_sha256\":\"85cbf2b3413f918315b28f769ca89ebc1aaaa11ff76c002472457daffa11c41f\"}, {\"table_name\":\"users\",\"rows\":2,\"rows_sha256\":\"56876a04c1c6c998dd434d9a18e74f004d4d8231f4d83f2f913caef928fa1c9e\"}]}\n"
+    }
+  },
+  "permissionDeniedPreflight": {
+    "args": [
+      "docker",
+      "exec",
+      "d37ee72dd39c",
+      "psql",
+      "-X",
+      "-qAt",
+      "-U",
+      "monitor",
+      "-d",
+      "monitor",
+      "-v",
+      "ON_ERROR_STOP=1",
+      "-c",
+      "\nSELECT json_build_object(\n  'otherConnections', (SELECT count(*) FROM pg_stat_activity WHERE datname=current_database() AND pid<>pg_backend_pid()),\n  'publicRelations', (SELECT count(*) FROM pg_class c JOIN pg_namespace n ON n.oid=c.relnamespace WHERE n.nspname='public'),\n  'publicCatalogSha256', (SELECT encode(sha256(convert_to(COALESCE(jsonb_agg(jsonb_build_object('name',c.relname,'kind',c.relkind,'owner',c.relowner) ORDER BY c.relname)::text,'[]'),'UTF8')),'hex') FROM pg_class c JOIN pg_namespace n ON n.oid=c.relnamespace WHERE n.nspname='public'),\n  'usersAreFrozenPair', (SELECT count(*)=2 AND bool_and(id IN ('24000000-0000-4000-9000-000000000001','24000000-0000-4000-9000-000000000002')) FROM e24_container.users),\n  'monitorsAreFrozenPair', (SELECT count(*)=2 AND bool_and((id='24000000-0000-4000-8000-000000000001' AND owner_user_id='24000000-0000-4000-9000-000000000001') OR (id='24000000-0000-4000-8000-000000000002' AND owner_user_id='24000000-0000-4000-9000-000000000002')) FROM e24_container.monitors),\n  'activeChecks', (SELECT count(*) FROM e24_container.check_runs WHERE state IN ('QUEUED','RUNNING')),\n  'checkStates', (SELECT json_object_agg(state,n) FROM (SELECT state,count(*) n FROM e24_container.check_runs GROUP BY state) s),\n  'tables', (\n    SELECT json_agg(row_to_json(s) ORDER BY s.table_name) FROM (\n      SELECT 'users' AS table_name,count(*) AS rows,encode(sha256(convert_to(COALESCE(jsonb_agg(to_jsonb(t) ORDER BY to_jsonb(t)::text)::text,'[]'),'UTF8')),'hex') AS rows_sha256 FROM e24_container.users t\nUNION ALL\nSELECT 'sessions' AS table_name,count(*) AS rows,encode(sha256(convert_to(COALESCE(jsonb_agg(to_jsonb(t) ORDER BY to_jsonb(t)::text)::text,'[]'),'UTF8')),'hex') AS rows_sha256 FROM e24_container.sessions t\nUNION ALL\nSELECT 'monitors' AS table_name,count(*) AS rows,encode(sha256(convert_to(COALESCE(jsonb_agg(to_jsonb(t) ORDER BY to_jsonb(t)::text)::text,'[]'),'UTF8')),'hex') AS rows_sha256 FROM e24_container.monitors t\nUNION ALL\nSELECT 'check_runs' AS table_name,count(*) AS rows,encode(sha256(convert_to(COALESCE(jsonb_agg(to_jsonb(t) ORDER BY to_jsonb(t)::text)::text,'[]'),'UTF8')),'hex') AS rows_sha256 FROM e24_container.check_runs t\nUNION ALL\nSELECT 'schema_migrations' AS table_name,count(*) AS rows,encode(sha256(convert_to(COALESCE(jsonb_agg(to_jsonb(t) ORDER BY to_jsonb(t)::text)::text,'[]'),'UTF8')),'hex') AS rows_sha256 FROM e24_container.schema_migrations t\n    ) s\n  )\n);"
+    ],
+    "sql": "\nSELECT json_build_object(\n  'otherConnections', (SELECT count(*) FROM pg_stat_activity WHERE datname=current_database() AND pid<>pg_backend_pid()),\n  'publicRelations', (SELECT count(*) FROM pg_class c JOIN pg_namespace n ON n.oid=c.relnamespace WHERE n.nspname='public'),\n  'publicCatalogSha256', (SELECT encode(sha256(convert_to(COALESCE(jsonb_agg(jsonb_build_object('name',c.relname,'kind',c.relkind,'owner',c.relowner) ORDER BY c.relname)::text,'[]'),'UTF8')),'hex') FROM pg_class c JOIN pg_namespace n ON n.oid=c.relnamespace WHERE n.nspname='public'),\n  'usersAreFrozenPair', (SELECT count(*)=2 AND bool_and(id IN ('24000000-0000-4000-9000-000000000001','24000000-0000-4000-9000-000000000002')) FROM e24_container.users),\n  'monitorsAreFrozenPair', (SELECT count(*)=2 AND bool_and((id='24000000-0000-4000-8000-000000000001' AND owner_user_id='24000000-0000-4000-9000-000000000001') OR (id='24000000-0000-4000-8000-000000000002' AND owner_user_id='24000000-0000-4000-9000-000000000002')) FROM e24_container.monitors),\n  'activeChecks', (SELECT count(*) FROM e24_container.check_runs WHERE state IN ('QUEUED','RUNNING')),\n  'checkStates', (SELECT json_object_agg(state,n) FROM (SELECT state,count(*) n FROM e24_container.check_runs GROUP BY state) s),\n  'tables', (\n    SELECT json_agg(row_to_json(s) ORDER BY s.table_name) FROM (\n      SELECT 'users' AS table_name,count(*) AS rows,encode(sha256(convert_to(COALESCE(jsonb_agg(to_jsonb(t) ORDER BY to_jsonb(t)::text)::text,'[]'),'UTF8')),'hex') AS rows_sha256 FROM e24_container.users t\nUNION ALL\nSELECT 'sessions' AS table_name,count(*) AS rows,encode(sha256(convert_to(COALESCE(jsonb_agg(to_jsonb(t) ORDER BY to_jsonb(t)::text)::text,'[]'),'UTF8')),'hex') AS rows_sha256 FROM e24_container.sessions t\nUNION ALL\nSELECT 'monitors' AS table_name,count(*) AS rows,encode(sha256(convert_to(COALESCE(jsonb_agg(to_jsonb(t) ORDER BY to_jsonb(t)::text)::text,'[]'),'UTF8')),'hex') AS rows_sha256 FROM e24_container.monitors t\nUNION ALL\nSELECT 'check_runs' AS table_name,count(*) AS rows,encode(sha256(convert_to(COALESCE(jsonb_agg(to_jsonb(t) ORDER BY to_jsonb(t)::text)::text,'[]'),'UTF8')),'hex') AS rows_sha256 FROM e24_container.check_runs t\nUNION ALL\nSELECT 'schema_migrations' AS table_name,count(*) AS rows,encode(sha256(convert_to(COALESCE(jsonb_agg(to_jsonb(t) ORDER BY to_jsonb(t)::text)::text,'[]'),'UTF8')),'hex') AS rows_sha256 FROM e24_container.schema_migrations t\n    ) s\n  )\n);",
+    "result": {
+      "chunk_id": "1c1a76",
+      "wall_time_seconds": 0,
+      "exit_code": 1,
+      "original_token_count": 78,
+      "output": "permission denied while trying to connect to the Docker daemon socket at unix:///Users/woopinbell/.docker/run/docker.sock: Get \"http://%2FUsers%2Fwoopinbell%2F.docker%2Frun%2Fdocker.sock/v1.48/containers/d37ee72dd39c/json\": dial unix /Users/woopinbell/.docker/run/docker.sock: connect: operation not permitted\n"
+    }
+  }
+}
diff --git a/evidence/phase-1/E24/results/repair1-cleanup-restore-approved.json b/evidence/phase-1/E24/results/repair1-cleanup-restore-approved.json
new file mode 100644
index 0000000..9b6c199
--- /dev/null
+++ b/evidence/phase-1/E24/results/repair1-cleanup-restore-approved.json
@@ -0,0 +1,142 @@
+{
+  "purpose": "Failure cleanup only",
+  "precedingPermissionDeniedPreflight": "repair1-cleanup-restore.json",
+  "restoreInvocations": 1,
+  "commands": [
+    {
+      "args": [
+        "docker",
+        "ps",
+        "-aq",
+        "--filter",
+        "label=com.docker.compose.project=wse-fundamentals",
+        "--filter",
+        "label=com.docker.compose.service=postgres"
+      ],
+      "exitCode": 0,
+      "durationMs": 35,
+      "stdout": "d37ee72dd39c\n",
+      "stderr": ""
+    },
+    {
+      "args": [
+        "docker",
+        "inspect",
+        "--format",
+        "{\"id\":{{json .Id}},\"state\":{{json .State.Status}},\"health\":{{json .State.Health.Status}},\"image\":{{json .Image}},\"project\":{{json (index .Config.Labels \"com.docker.compose.project\")}},\"service\":{{json (index .Config.Labels \"com.docker.compose.service\")}},\"mounts\":{{json .Mounts}}}",
+        "d37ee72dd39c"
+      ],
+      "exitCode": 0,
+      "durationMs": 21,
+      "stdout": "{\"id\":\"d37ee72dd39c2912ef3e7883148ebd295186b7f6d3aed3a7a3908c3b02f47fc9\",\"state\":\"exited\",\"health\":\"unhealthy\",\"image\":\"sha256:f3bd19c606e442c3d7bdfa8002e03fe260a1023351e0ea4598032022b68dd6e3\",\"project\":\"wse-fundamentals\",\"service\":\"postgres\",\"mounts\":[{\"Destination\":\"/var/lib/postgresql/data\",\"Driver\":\"local\",\"Mode\":\"z\",\"Name\":\"wse-fundamentals_postgres-data\",\"Propagation\":\"\",\"RW\":true,\"Source\":\"/var/lib/docker/volumes/wse-fundamentals_postgres-data/_data\",\"Type\":\"volume\"}]}\n",
+      "stderr": ""
+    },
+    {
+      "args": [
+        "docker",
+        "ps",
+        "-aq",
+        "--filter",
+        "label=com.docker.compose.project=wse-fundamentals-e24"
+      ],
+      "exitCode": 0,
+      "durationMs": 20,
+      "stdout": "",
+      "stderr": ""
+    },
+    {
+      "args": [
+        "docker",
+        "network",
+        "ls",
+        "-q",
+        "--filter",
+        "label=com.docker.compose.project=wse-fundamentals-e24"
+      ],
+      "exitCode": 0,
+      "durationMs": 16,
+      "stdout": "",
+      "stderr": ""
+    },
+    {
+      "args": [
+        "docker",
+        "compose",
+        "--project-name",
+        "wse-fundamentals",
+        "-f",
+        "compose.yaml",
+        "up",
+        "--no-recreate",
+        "--no-build",
+        "--no-deps",
+        "--pull",
+        "never",
+        "--wait",
+        "--wait-timeout",
+        "30",
+        "postgres"
+      ],
+      "exitCode": 0,
+      "durationMs": 1872,
+      "stdout": "",
+      "stderr": " Container wse-fundamentals-postgres-1  Created\n Container wse-fundamentals-postgres-1  Starting\n Container wse-fundamentals-postgres-1  Started\n Container wse-fundamentals-postgres-1  Waiting\n Container wse-fundamentals-postgres-1  Healthy\n"
+    },
+    {
+      "args": [
+        "docker",
+        "inspect",
+        "--format",
+        "{\"id\":{{json .Id}},\"state\":{{json .State.Status}},\"health\":{{json .State.Health.Status}},\"image\":{{json .Image}},\"project\":{{json (index .Config.Labels \"com.docker.compose.project\")}},\"service\":{{json (index .Config.Labels \"com.docker.compose.service\")}},\"mounts\":{{json .Mounts}}}",
+        "d37ee72dd39c"
+      ],
+      "exitCode": 0,
+      "durationMs": 22,
+      "stdout": "{\"id\":\"d37ee72dd39c2912ef3e7883148ebd295186b7f6d3aed3a7a3908c3b02f47fc9\",\"state\":\"running\",\"health\":\"healthy\",\"image\":\"sha256:f3bd19c606e442c3d7bdfa8002e03fe260a1023351e0ea4598032022b68dd6e3\",\"project\":\"wse-fundamentals\",\"service\":\"postgres\",\"mounts\":[{\"Destination\":\"/var/lib/postgresql/data\",\"Driver\":\"local\",\"Mode\":\"z\",\"Name\":\"wse-fundamentals_postgres-data\",\"Propagation\":\"\",\"RW\":true,\"Source\":\"/var/lib/docker/volumes/wse-fundamentals_postgres-data/_data\",\"Type\":\"volume\"}]}\n",
+      "stderr": ""
+    }
+  ],
+  "result": "PASS",
+  "rawRuntimeValuesRetained": false,
+  "before": {
+    "id": "d37ee72dd39c2912ef3e7883148ebd295186b7f6d3aed3a7a3908c3b02f47fc9",
+    "state": "exited",
+    "health": "unhealthy",
+    "image": "sha256:f3bd19c606e442c3d7bdfa8002e03fe260a1023351e0ea4598032022b68dd6e3",
+    "project": "wse-fundamentals",
+    "service": "postgres",
+    "mounts": [
+      {
+        "Destination": "/var/lib/postgresql/data",
+        "Driver": "local",
+        "Mode": "z",
+        "Name": "wse-fundamentals_postgres-data",
+        "Propagation": "",
+        "RW": true,
+        "Source": "/var/lib/docker/volumes/wse-fundamentals_postgres-data/_data",
+        "Type": "volume"
+      }
+    ]
+  },
+  "after": {
+    "id": "d37ee72dd39c2912ef3e7883148ebd295186b7f6d3aed3a7a3908c3b02f47fc9",
+    "state": "running",
+    "health": "healthy",
+    "image": "sha256:f3bd19c606e442c3d7bdfa8002e03fe260a1023351e0ea4598032022b68dd6e3",
+    "project": "wse-fundamentals",
+    "service": "postgres",
+    "mounts": [
+      {
+        "Destination": "/var/lib/postgresql/data",
+        "Driver": "local",
+        "Mode": "z",
+        "Name": "wse-fundamentals_postgres-data",
+        "Propagation": "",
+        "RW": true,
+        "Source": "/var/lib/docker/volumes/wse-fundamentals_postgres-data/_data",
+        "Type": "volume"
+      }
+    ]
+  },
+  "sameContainerImageAndMounts": true
+}
diff --git a/evidence/phase-1/E24/results/repair1-cleanup-restore-approved.started.json b/evidence/phase-1/E24/results/repair1-cleanup-restore-approved.started.json
new file mode 100644
index 0000000..91d2287
--- /dev/null
+++ b/evidence/phase-1/E24/results/repair1-cleanup-restore-approved.started.json
@@ -0,0 +1 @@
+{"at": "2026-08-28T07:14:44.433447+00:00", "purpose": "Failure cleanup only; one restore of the already existing owned PostgreSQL. Prior sandbox-denied preflight performed zero restore calls."}
\ No newline at end of file
diff --git a/evidence/phase-1/E24/results/repair1-cleanup-restore.json b/evidence/phase-1/E24/results/repair1-cleanup-restore.json
new file mode 100644
index 0000000..5348307
--- /dev/null
+++ b/evidence/phase-1/E24/results/repair1-cleanup-restore.json
@@ -0,0 +1,27 @@
+{
+  "purpose": "Failure cleanup only",
+  "restoreInvocations": 0,
+  "commands": [
+    {
+      "args": [
+        "docker",
+        "ps",
+        "-aq",
+        "--filter",
+        "label=com.docker.compose.project=wse-fundamentals",
+        "--filter",
+        "label=com.docker.compose.service=postgres"
+      ],
+      "exitCode": 1,
+      "durationMs": 118,
+      "stdout": "",
+      "stderr": "permission denied while trying to connect to the Docker daemon socket at unix:///Users/woopinbell/.docker/run/docker.sock: Get \"http://%2FUsers%2Fwoopinbell%2F.docker%2Frun%2Fdocker.sock/v1.48/containers/json?all=1&filters=%7B%22label%22%3A%7B%22com.docker.compose.project%3Dwse-fundamentals%22%3Atrue%2C%22com.docker.compose.service%3Dpostgres%22%3Atrue%7D%7D\": dial unix /Users/woopinbell/.docker/run/docker.sock: connect: operation not permitted\n"
+    }
+  ],
+  "result": "FAILED",
+  "rawRuntimeValuesRetained": false,
+  "failure": {
+    "kind": "RuntimeError",
+    "message": "Native command failed; no retry permitted"
+  }
+}
diff --git a/evidence/phase-1/E24/results/repair1-cleanup-restore.started.json b/evidence/phase-1/E24/results/repair1-cleanup-restore.started.json
new file mode 100644
index 0000000..8706ddb
--- /dev/null
+++ b/evidence/phase-1/E24/results/repair1-cleanup-restore.started.json
@@ -0,0 +1 @@
+{"at": "2026-08-28T07:14:07.349883+00:00", "purpose": "Failure cleanup only; one restore of the already existing owned PostgreSQL, not a scenario run."}
\ No newline at end of file
diff --git a/evidence/phase-1/E24/results/repair1-image-reuse.json b/evidence/phase-1/E24/results/repair1-image-reuse.json
new file mode 100644
index 0000000..b7af36a
--- /dev/null
+++ b/evidence/phase-1/E24/results/repair1-image-reuse.json
@@ -0,0 +1,29 @@
+{
+  "expectedImages": {
+    "backend": "sha256:2f6918aed823d07ac1e05980b7747bf49e0912a027326ef3c5713c98fc5bfbc8",
+    "frontend": "sha256:7cd37b250c2e45bcc99271b688a74336dcbee88d10a29251edc8cbb61f68de16"
+  },
+  "actualImagesMatchOriginalBuild": true,
+  "readOnlyImageInspection": {
+    "status": "fulfilled",
+    "value": {
+      "chunk_id": "1d01ac",
+      "wall_time_seconds": 0.000001166,
+      "exit_code": 0,
+      "original_token_count": 89,
+      "output": "\"sha256:2f6918aed823d07ac1e05980b7747bf49e0912a027326ef3c5713c98fc5bfbc8\" [\"wse-fundamentals-e24-backend:local\"] \"2026-08-28T07:03:30.407880835Z\" \"node\" [\"node\",\"server/main.ts\"]\n\"sha256:7cd37b250c2e45bcc99271b688a74336dcbee88d10a29251edc8cbb61f68de16\" [\"wse-fundamentals-e24-frontend:local\"] \"2026-08-28T07:03:34.591821087Z\" \"node\" [\"node\",\"server.js\"]\n"
+    }
+  },
+  "originalBuildEvidenceExcerpt": {
+    "status": "fulfilled",
+    "value": {
+      "chunk_id": "1472f2",
+      "wall_time_seconds": 0.000001167,
+      "exit_code": 0,
+      "original_token_count": 277,
+      "output": "{\"chunk_id\": \"49e99d\", \"wall_time_seconds\": 1.001431792, \"session_id\": 10575, \"original_token_count\": 139}\n{\"chunk_id\": \"1d72d1\", \"wall_time_seconds\": 5.001735167, \"session_id\": 10575, \"original_token_count\": 550}\n#4 transferring context: 189B done\n#5 transferring context: 189B done\n#6 [api internal] load build context\n#6 transferring context: 138.95kB 0.0s done\n#7 [frontend internal] load build context\n#7 transferring context: 174.64kB done\n{\"chunk_id\": \"c9dc8f\", \"wall_time_seconds\": 1.792e-06, \"exit_code\": 0, \"original_token_count\": 766}\n#25 exporting config sha256:a93e98a1702f149f9de7ffb3cc82dd59b14a52bc4370c073251482ef1db7709c done\n#25 exporting manifest list sha256:7cd37b250c2e45bcc99271b688a74336dcbee88d10a29251edc8cbb61f68de16 done\n#25 naming to docker.io/library/wse-fundamentals-e24-frontend:local done\n#22 exporting config sha256:0bc5bb08e0c9b39bc9c716eeacf88f179faf2e67607b668fabff1d3b590aff4e done\n#22 exporting manifest list sha256:2f6918aed823d07ac1e05980b7747bf49e0912a027326ef3c5713c98fc5bfbc8 done\n#22 naming to docker.io/library/wse-fundamentals-e24-backend:local done\nreal 34.44\n"
+    }
+  },
+  "contextAudit": "repair1-reuse-audit.json",
+  "newBuilds": 0
+}
diff --git a/evidence/phase-1/E24/results/repair1-reuse-audit.json b/evidence/phase-1/E24/results/repair1-reuse-audit.json
new file mode 100644
index 0000000..6bc865b
--- /dev/null
+++ b/evidence/phase-1/E24/results/repair1-reuse-audit.json
@@ -0,0 +1,225 @@
+{
+  "at": "2026-08-28T07:18:21.501804+00:00",
+  "adoptionCommit": "f5156939760e4db9221999217c5f2b23ddc008d3",
+  "observerOnlyTwoAuthorizedLinesChanged": true,
+  "observerSha256": "74b9369fd18a78f1a4d02faadbf2f35c0d4f9b116ffc6af70130eb7c53452cb4",
+  "unchangedBuildContext": [
+    {
+      "path": ".dockerignore",
+      "sha256": "03528998ff52cea302a211e3e48cd9d13b35656dc50c20bc7eae8e4063811a1d"
+    },
+    {
+      "path": "Dockerfile",
+      "sha256": "3555e4151dae61b8f24b9a53f25bb637a9148bc84fe4c8788081ba1acb9f8378"
+    },
+    {
+      "path": "app/layout.tsx",
+      "sha256": "4a1d88207aa0f9e4ce3a7369207b696e42c742589a0d54eb465f4d918f9312ef"
+    },
+    {
+      "path": "app/login/login-form.tsx",
+      "sha256": "7f1e3d135b999ad3891ba0902c8c7b33352adf77c2fce2c0a96021880e38af62"
+    },
+    {
+      "path": "app/login/page.tsx",
+      "sha256": "9d1ceedf370739bef4a30d33a6724f420575ac20f5dca418a2aa33df867c930c"
+    },
+    {
+      "path": "app/monitors/api.ts",
+      "sha256": "c595e0dbb7994facb03aa7c10e552dba8f1e3cae8cb2105ca5a73bb73cd773a5"
+    },
+    {
+      "path": "app/monitors/initial-state.ts",
+      "sha256": "91d966b3dae3c5a2e6c361724a2b43bbd3dcb06aea876a77d7b425ea0f8bb562"
+    },
+    {
+      "path": "app/monitors/monitor-workspace.tsx",
+      "sha256": "57809e5341c650b78cfb4244583eaf2bcc1dcb7d5d13f45c84904eda12ab7992"
+    },
+    {
+      "path": "app/monitors/page.tsx",
+      "sha256": "830d0583a53e23363e31debcff63c321f562ceb85170706d954152b45910675f"
+    },
+    {
+      "path": "app/monitors/server-data.ts",
+      "sha256": "dc2df71ed90a5caa0ab4d842989d8680a98c764a242d8032b6dd9d56ba6cdf79"
+    },
+    {
+      "path": "app/monitors/use-monitors.ts",
+      "sha256": "6da04e2505fb261eca81b677c7f27471f399893140c7aa10e08b97827e7042f5"
+    },
+    {
+      "path": "app/page.tsx",
+      "sha256": "84426886dcce50ded872e609644821c751cab148630048f5e2d4a323920a0163"
+    },
+    {
+      "path": "app/style.css",
+      "sha256": "864bb4a66750bb988c218668d28ccfea47b5a880aea4926e27c5ee5a18642da0"
+    },
+    {
+      "path": "next.config.ts",
+      "sha256": "3e13d0b861e3058e2f69f40dbf4f24e7828611797488b820988104706f95beac"
+    },
+    {
+      "path": "package-lock.json",
+      "sha256": "516260fa3554fb45b470f61d2fa01d7745f6d499033abd197076c21fd265ac66"
+    },
+    {
+      "path": "package.json",
+      "sha256": "682a14e2d77822fecd27086454c52523e613a2ff221bc31a06e954dac6ab1450"
+    },
+    {
+      "path": "server/app.ts",
+      "sha256": "42ca7842fdd94ac1fe291bbba40f13b0bc898ba9d007fa3edd90ef44f7a363c5"
+    },
+    {
+      "path": "server/auth.ts",
+      "sha256": "f1b7a7880ebae7219eff152ff17064da5507808ee9b0a99cc35e8acc64701288"
+    },
+    {
+      "path": "server/check.ts",
+      "sha256": "d997b5060e7fc89d9fcc157057c42990c863a248541b6d6a3950e0772cc8c4ca"
+    },
+    {
+      "path": "server/contracts.ts",
+      "sha256": "795237e965e35f5e3c363183ae97cd8af51874aab88bebfdb0a38bc8f2a20e17"
+    },
+    {
+      "path": "server/database.ts",
+      "sha256": "1c31f6f034501fb3f2c55aa4e6befde1874dfdf7ca13b728565bfda1f80aa768"
+    },
+    {
+      "path": "server/history.ts",
+      "sha256": "c19bbe0310835c0c7c5379b7e52acde780c228178538345158f0676112d9b44a"
+    },
+    {
+      "path": "server/main.ts",
+      "sha256": "3459b95b964eef79398e73661f9f96395a4124dc1c741f6f434cf47bd0b51847"
+    },
+    {
+      "path": "server/mapping.ts",
+      "sha256": "27912615f39c4739f93214e079a162a5efca6412bcae23a74d52d129313ccb6a"
+    },
+    {
+      "path": "server/migrate.ts",
+      "sha256": "bfc052895b6c46383e0b360775574dd619ecf92f7e64ade83af274bac6e78ce2"
+    },
+    {
+      "path": "server/migrations/001_monitors.sql",
+      "sha256": "e695bcf8abe711f63b8eee31f9ca98bb50eb90921349156bffe183bfddacc86e"
+    },
+    {
+      "path": "server/migrations/002_check_runs.sql",
+      "sha256": "13878c39225e292140a81312f98c7db76da3a3c49aa105218b1fc1193ee8f389"
+    },
+    {
+      "path": "server/migrations/003_sessions.sql",
+      "sha256": "097df3920042f49231f1dff9589f621acce0cc6c72fa680ad8e7fad0c8d84db9"
+    },
+    {
+      "path": "server/migrations/004_monitor_ownership.sql",
+      "sha256": "02cfe4dc77a1218330da13a71044b0d5557d6ab134c2832017ff7a25fd14fcb9"
+    },
+    {
+      "path": "server/migrations/005_check_history_index.sql",
+      "sha256": "1cc86167592c56f70baf8362969db688db73ddf802bc36fd33adc8abd3c6931e"
+    },
+    {
+      "path": "server/migrations/006_check_queue.sql",
+      "sha256": "492868fcc0e95ccec1c5200f980ae55816aea5d2c64a28e65b9c96f9095376af"
+    },
+    {
+      "path": "server/migrations/007_check_ownership.sql",
+      "sha256": "9c248b9d430088c30ca417d3b49fd7f9e9e85457393f42fe55a913de62458b31"
+    },
+    {
+      "path": "server/migrations/008_check_lease.sql",
+      "sha256": "4c9346ec915c848688864c6209093dadad8459e0860996310a4bb21b3267dfd2"
+    },
+    {
+      "path": "server/migrations/009_outbound_policy_result.sql",
+      "sha256": "e066779fee21645e0894c10a80ab2ef52835f75cd60524dfaf948bdaeb0394c9"
+    },
+    {
+      "path": "server/migrations/010_failed_history_index.sql",
+      "sha256": "f9ae03555556ee1eb817c21423ce70dab3d1ceebeec57297191c6868ce01f583"
+    },
+    {
+      "path": "server/model.ts",
+      "sha256": "76a3bcf699ce2485559387422c2f42479861e8a6d99cb0ff1491819126fcfffa"
+    },
+    {
+      "path": "server/operations.ts",
+      "sha256": "2e68545cfca280ab30d7bb711ca8a18c9ec96735335d6de9597c7287699226f3"
+    },
+    {
+      "path": "server/outbound.ts",
+      "sha256": "2d1d4ef54512c1b2c74d926ea0ad13921613cb82da1fc9702714f52b17cf46fb"
+    },
+    {
+      "path": "server/password.ts",
+      "sha256": "6e5dbfe1c59ed43f081d45d8098fa82350c2763c1c4a51dc4bfa2c869c260365"
+    },
+    {
+      "path": "server/schema.ts",
+      "sha256": "f1fd635c6879c06214b4a911771b97a092744812bcc118782f62431e773dfa47"
+    },
+    {
+      "path": "server/worker.ts",
+      "sha256": "3706059240099ef29c7a2091afe31e11f7d47ea31c063bf5d8a776b8a3c33448"
+    },
+    {
+      "path": "tsconfig.json",
+      "sha256": "3155624b053efbffb8db1fe0daa189542e85a09fa75ed0f6f5b64b3771be3a0c"
+    }
+  ],
+  "buildContextContentSha256": "5632948611767cf027a6dd44402d120879a4512cad3b994e8d6381191c079aa6",
+  "exactPreservedSourceProof": true,
+  "frozenInputsUnchanged": true,
+  "originalOutputsUnchanged": true,
+  "reusedGates": [
+    {
+      "gate": "compose-config",
+      "command": "/usr/bin/time -p docker compose --project-name wse-fundamentals-e24 -f compose.production.yaml -f evidence/phase-1/E24/compose.test.yaml config --quiet",
+      "exit": 0,
+      "realSeconds": 0.15,
+      "consoleComplete": true
+    },
+    {
+      "gate": "typecheck",
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run typecheck",
+      "exit": 0,
+      "realSeconds": 2.18,
+      "consoleComplete": true
+    },
+    {
+      "gate": "unit",
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:unit",
+      "exit": 0,
+      "realSeconds": 1.28,
+      "pass": 22,
+      "fail": 0,
+      "consoleComplete": true
+    },
+    {
+      "gate": "functional",
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:functional",
+      "exit": 0,
+      "realSeconds": 10.55,
+      "pass": 15,
+      "fail": 0,
+      "consoleComplete": false,
+      "limitation": "The first polling result was truncated by exec tool max_output_tokens=6500 (original_token_count=26023). Stored chunks are the exact received text; missing routine structured logs cannot be reconstructed. The final pass/fail summary and duration were received in full. No rerun."
+    },
+    {
+      "gate": "production-build",
+      "command": "/usr/bin/time -p docker compose --project-name wse-fundamentals-e24 -f compose.production.yaml build api frontend",
+      "exit": 0,
+      "realSeconds": 34.44,
+      "consoleComplete": true
+    }
+  ],
+  "rerunGates": [],
+  "newBuilds": 0,
+  "dependencyChanges": 0
+}
diff --git a/evidence/phase-1/E24/results/repair1-schema-cleanup.json b/evidence/phase-1/E24/results/repair1-schema-cleanup.json
new file mode 100644
index 0000000..a0f1b24
--- /dev/null
+++ b/evidence/phase-1/E24/results/repair1-schema-cleanup.json
@@ -0,0 +1,21 @@
+{
+  "purpose": "Explicit failed E24 fixture cleanup, separate from all scenario invocations",
+  "observed": {
+    "schemaDropped": true,
+    "namespaces": [
+      "public"
+    ],
+    "publicRelations": 0,
+    "publicCatalogSha256": "4f53cda18c2baa0c0354bb5f9a3ecbe5ed12ab4d8e11ba873c2f11161202b945"
+  },
+  "publicPreserved": true,
+  "rawRuntimeValuesRetained": false,
+  "sql": "BEGIN;\nSET LOCAL statement_timeout='5s';\nSET LOCAL lock_timeout='1s';\nDO $safe$\nBEGIN\n  IF (SELECT count(*) FROM pg_stat_activity WHERE datname=current_database() AND pid<>pg_backend_pid()) <> 0\n     OR (SELECT count(*) FROM pg_class c JOIN pg_namespace n ON n.oid=c.relnamespace WHERE n.nspname='public') <> 0\n     OR (SELECT count(*) FROM e24_container.users) <> 2\n     OR (SELECT count(*) FROM e24_container.monitors) <> 2\n     OR (SELECT count(*) FROM e24_container.check_runs) <> 5\n     OR EXISTS (SELECT 1 FROM e24_container.check_runs WHERE state IN ('QUEUED','RUNNING'))\n  THEN RAISE EXCEPTION 'Unsafe owned fixture cleanup boundary'; END IF;\nEND\n$safe$;\nDROP SCHEMA e24_container CASCADE;\nSELECT json_build_object(\n  'schemaDropped', NOT EXISTS(SELECT 1 FROM pg_namespace WHERE nspname='e24_container'),\n  'namespaces', (SELECT json_agg(nspname ORDER BY nspname) FROM pg_namespace WHERE nspname NOT LIKE 'pg_%' AND nspname <> 'information_schema'),\n  'publicRelations', (SELECT count(*) FROM pg_class c JOIN pg_namespace n ON n.oid=c.relnamespace WHERE n.nspname='public'),\n  'publicCatalogSha256', (SELECT encode(sha256(convert_to(COALESCE(jsonb_agg(jsonb_build_object('name',c.relname,'kind',c.relkind,'owner',c.relowner) ORDER BY c.relname)::text,'[]'),'UTF8')),'hex') FROM pg_class c JOIN pg_namespace n ON n.oid=c.relnamespace WHERE n.nspname='public')\n);\nCOMMIT;",
+  "result": {
+    "chunk_id": "998d70",
+    "wall_time_seconds": 0.000006208,
+    "exit_code": 0,
+    "original_token_count": 114,
+    "output": "NOTICE:  drop cascades to 5 other objects\nDETAIL:  drop cascades to table e24_container.schema_migrations\ndrop cascades to table e24_container.monitors\ndrop cascades to table e24_container.check_runs\ndrop cascades to table e24_container.users\ndrop cascades to table e24_container.sessions\n{\"schemaDropped\" : true, \"namespaces\" : [\"public\"], \"publicRelations\" : 0, \"publicCatalogSha256\" : \"4f53cda18c2baa0c0354bb5f9a3ecbe5ed12ab4d8e11ba873c2f11161202b945\"}\n"
+  }
+}
diff --git a/evidence/phase-1/E24/results/repair1-schema-cleanup.started.json b/evidence/phase-1/E24/results/repair1-schema-cleanup.started.json
new file mode 100644
index 0000000..7dfeb59
--- /dev/null
+++ b/evidence/phase-1/E24/results/repair1-schema-cleanup.started.json
@@ -0,0 +1 @@
+{"at":"2026-08-28T07:16:20.586Z","purpose":"Drop only the preserved failed e24_container schema after explicit cleanup restore and safe counts/digests; public has zero relations and zero other connections."}
diff --git a/evidence/phase-1/E24/results/repair2-adoption-audit.json b/evidence/phase-1/E24/results/repair2-adoption-audit.json
new file mode 100644
index 0000000..a8c381d
--- /dev/null
+++ b/evidence/phase-1/E24/results/repair2-adoption-audit.json
@@ -0,0 +1,313 @@
+{
+  "at": "2026-08-28T07:27:37.993336+00:00",
+  "branch": "track/fundamentals-fastify",
+  "headBeforeAdoption": "f5156939760e4db9221999217c5f2b23ddc008d3",
+  "adoptedFiles": [
+    {
+      "path": ".dockerignore",
+      "adoptedHeadSha256": "03528998ff52cea302a211e3e48cd9d13b35656dc50c20bc7eae8e4063811a1d",
+      "headAndSnapshotMatch": true,
+      "workingBytesMatch": true
+    },
+    {
+      "path": ".github/workflows/check.yml",
+      "adoptedHeadSha256": "ef1ac4312e02837fbba15fe129817e647b4ff4f0e57dbe79903f13be488610e5",
+      "headAndSnapshotMatch": true,
+      "workingBytesMatch": true
+    },
+    {
+      "path": "Dockerfile",
+      "adoptedHeadSha256": "3555e4151dae61b8f24b9a53f25bb637a9148bc84fe4c8788081ba1acb9f8378",
+      "headAndSnapshotMatch": true,
+      "workingBytesMatch": true
+    },
+    {
+      "path": "compose.production.yaml",
+      "adoptedHeadSha256": "9998b7fb98317185690b2e21ef6559a87ff6d53bf31168b9b22b3e7b5efd9d52",
+      "headAndSnapshotMatch": true,
+      "workingBytesMatch": true
+    },
+    {
+      "path": "evidence/phase-1/E24/compose.test.yaml",
+      "adoptedHeadSha256": "e84ee2f893bb851ef630a6e9e34dde1daaadc9d8f4ea8b641a3f299b55c80fbe",
+      "headAndSnapshotMatch": true,
+      "workingBytesMatch": true
+    },
+    {
+      "path": "evidence/phase-1/E24/fixture-server.mjs",
+      "adoptedHeadSha256": "5ba0eeb56ec9e2e664aed5e32c0742366f6122051805a777966c894cc769cbb1",
+      "headAndSnapshotMatch": true,
+      "workingBytesMatch": true
+    },
+    {
+      "path": "next.config.ts",
+      "adoptedHeadSha256": "3e13d0b861e3058e2f69f40dbf4f24e7828611797488b820988104706f95beac",
+      "headAndSnapshotMatch": true,
+      "workingBytesMatch": true
+    },
+    {
+      "path": "package.json",
+      "adoptedHeadSha256": "682a14e2d77822fecd27086454c52523e613a2ff221bc31a06e954dac6ab1450",
+      "headAndSnapshotMatch": true,
+      "workingBytesMatch": true
+    },
+    {
+      "path": "server/app.ts",
+      "adoptedHeadSha256": "42ca7842fdd94ac1fe291bbba40f13b0bc898ba9d007fa3edd90ef44f7a363c5",
+      "headAndSnapshotMatch": true,
+      "workingBytesMatch": true
+    },
+    {
+      "path": "server/auth.ts",
+      "adoptedHeadSha256": "f1b7a7880ebae7219eff152ff17064da5507808ee9b0a99cc35e8acc64701288",
+      "headAndSnapshotMatch": true,
+      "workingBytesMatch": true
+    },
+    {
+      "path": "server/database.ts",
+      "adoptedHeadSha256": "1c31f6f034501fb3f2c55aa4e6befde1874dfdf7ca13b728565bfda1f80aa768",
+      "headAndSnapshotMatch": true,
+      "workingBytesMatch": true
+    },
+    {
+      "path": "server/main.ts",
+      "adoptedHeadSha256": "3459b95b964eef79398e73661f9f96395a4124dc1c741f6f434cf47bd0b51847",
+      "headAndSnapshotMatch": true,
+      "workingBytesMatch": true
+    },
+    {
+      "path": "server/operations.ts",
+      "adoptedHeadSha256": "2e68545cfca280ab30d7bb711ca8a18c9ec96735335d6de9597c7287699226f3",
+      "headAndSnapshotMatch": true,
+      "workingBytesMatch": true
+    },
+    {
+      "path": "server/worker.ts",
+      "adoptedHeadSha256": "3706059240099ef29c7a2091afe31e11f7d47ea31c063bf5d8a776b8a3c33448",
+      "headAndSnapshotMatch": true,
+      "workingBytesMatch": true
+    },
+    {
+      "path": "test/container-smoke.mjs",
+      "adoptedHeadSha256": "637833438f7f16d064f0efd18c5dc3cc40568814b0aca55475fa6b5984017390",
+      "headAndSnapshotMatch": true,
+      "workingBytesMatch": false
+    },
+    {
+      "path": "test/unit.test.ts",
+      "adoptedHeadSha256": "6ea94aa491f58505592bdfcb7206d8817c3b01a74264a8d6f53df97c54848059",
+      "headAndSnapshotMatch": true,
+      "workingBytesMatch": true
+    },
+    {
+      "path": "test/worker.ts",
+      "adoptedHeadSha256": "6af3906d1baace042612e77612746ccfad2fd09ef02a80b3cb22129e1a7818d8",
+      "headAndSnapshotMatch": true,
+      "workingBytesMatch": true
+    }
+  ],
+  "repair1ObserverSha256": "74b9369fd18a78f1a4d02faadbf2f35c0d4f9b116ffc6af70130eb7c53452cb4",
+  "repair1ObserverByteIdentical": true,
+  "frozenInputs": {
+    "evidence/phase-1/E24/scenario.json": "07b5b3e2b4bd1cfc456c415134adce61f05dfdd729fe803968b32a4d813c5847",
+    "evidence/phase-1/E24/baseline.mjs": "0a24805b65454cddc543d27708aec68092b34e28b67ca6ba605e3bc6602c79d6",
+    "evidence/phase-1/E24/fixture.ts": "10d311f2e2af33063d09add435ed282b20341e2b87eeffa10b6287dce61ca784"
+  },
+  "preservedOutputs": {
+    "output/phase-1/e24/baseline.started.json": "7ff24402449c9abdde5a6eba834884558431d6a8377dee16d1c7cf581f201657",
+    "output/phase-1/e24/full-author.console.txt": "1c9d91904186621fee63b6e29b2d8a249446d783aac69bd7fd720c9e5533fb22",
+    "output/phase-1/e24/author-pre-container-invocations.json": "7f5595349f9a0cf7a4bc27003b3dcc1e4dc72abc071dfbaf5c0ad0d931d1148a",
+    "output/phase-1/e24/baseline.json": "4da653a0f6e05c1fcb26bdf85b9ec4015fbf3ef54e71e8ce9419af3895e0c684",
+    "output/phase-1/e24/full-author.started.json": "b75153248eba165b2b6e5e0015ee51bd8d6250aa3554d1563752ace0c1039fb4",
+    "output/phase-1/e24/full-author.json": "34eba201f813d2486ea87b4fdf9694ae5614ee974ef147a7f535480dc18efcc0",
+    "output/phase-1/e24/repair1-schema-cleanup.started.json": "9221d4238379f522dd28406e5990d2c3022501cca7feb486b4d29c47dbbe8582",
+    "output/phase-1/e24/repair1-cleanup-restore.json": "94d5bfa99a4eaa69efb5bd602dfbcf60bd7b47eec0ea365fd5e029e5716ea558",
+    "output/phase-1/e24/repair1-cleanup-inspection.json": "a36fd9470c61ebd2511b50b70c36ce76bcb53659f7124bfd7383c94fccc2e450",
+    "output/phase-1/e24/full-repair1.console.txt": "9ab2858de5e3c48ef06c4278ce7ccc089de4c753f232b1344e9e68920045b6ff",
+    "output/phase-1/e24/full-repair1.started.json": "8e768a69ac9d14d49da4a45173e0a81754df7eb0b363c8321729dde7d7c6c81b",
+    "output/phase-1/e24/repair1-cleanup-restore-approved.json": "70d717a759af5bc9246674b3779627dbc41e5aeccee922ed46c4bdf509158f37",
+    "output/phase-1/e24/repair1-cleanup-restore-approved.started.json": "0626181cde88f72050d42278ea3befd2d834fbb0805689fc92160b170c4fd04c",
+    "output/phase-1/e24/full-repair1.json": "6b6c39a9dd24ca9934e7453f5298b3da8bd7cb7f1ecd9f83bbaa6e0f0d779295",
+    "output/phase-1/e24/repair1-image-reuse.json": "774a869149a39123a2771125253b8c8acee0d3690ec1989b39752870fb69245c",
+    "output/phase-1/e24/full-repair1.console.json": "be66b29b962a7d9a10a04f1627e750dd9625fd37474e98afdc68891ef87c65e3",
+    "output/phase-1/e24/repair1-schema-cleanup.json": "fcfd4106cc01702033e63e21b1a25d4ad0fcdc5bf00c4ecacb1dcdcae9c557db",
+    "output/phase-1/e24/repair1-cleanup-restore.started.json": "b6217d6a1a40c6840d78b256295cf3412286eec5519dfc639813013cdde821e9",
+    "output/phase-1/e24/repair1-reuse-audit.json": "aa742253f9070580ce187e91c50e06b8f85623106d4b98166138eed61287bde2"
+  },
+  "unchangedBuildContext": [
+    {
+      "path": ".dockerignore",
+      "sha256": "03528998ff52cea302a211e3e48cd9d13b35656dc50c20bc7eae8e4063811a1d"
+    },
+    {
+      "path": "Dockerfile",
+      "sha256": "3555e4151dae61b8f24b9a53f25bb637a9148bc84fe4c8788081ba1acb9f8378"
+    },
+    {
+      "path": "app/layout.tsx",
+      "sha256": "4a1d88207aa0f9e4ce3a7369207b696e42c742589a0d54eb465f4d918f9312ef"
+    },
+    {
+      "path": "app/login/login-form.tsx",
+      "sha256": "7f1e3d135b999ad3891ba0902c8c7b33352adf77c2fce2c0a96021880e38af62"
+    },
+    {
+      "path": "app/login/page.tsx",
+      "sha256": "9d1ceedf370739bef4a30d33a6724f420575ac20f5dca418a2aa33df867c930c"
+    },
+    {
+      "path": "app/monitors/api.ts",
+      "sha256": "c595e0dbb7994facb03aa7c10e552dba8f1e3cae8cb2105ca5a73bb73cd773a5"
+    },
+    {
+      "path": "app/monitors/initial-state.ts",
+      "sha256": "91d966b3dae3c5a2e6c361724a2b43bbd3dcb06aea876a77d7b425ea0f8bb562"
+    },
+    {
+      "path": "app/monitors/monitor-workspace.tsx",
+      "sha256": "57809e5341c650b78cfb4244583eaf2bcc1dcb7d5d13f45c84904eda12ab7992"
+    },
+    {
+      "path": "app/monitors/page.tsx",
+      "sha256": "830d0583a53e23363e31debcff63c321f562ceb85170706d954152b45910675f"
+    },
+    {
+      "path": "app/monitors/server-data.ts",
+      "sha256": "dc2df71ed90a5caa0ab4d842989d8680a98c764a242d8032b6dd9d56ba6cdf79"
+    },
+    {
+      "path": "app/monitors/use-monitors.ts",
+      "sha256": "6da04e2505fb261eca81b677c7f27471f399893140c7aa10e08b97827e7042f5"
+    },
+    {
+      "path": "app/page.tsx",
+      "sha256": "84426886dcce50ded872e609644821c751cab148630048f5e2d4a323920a0163"
+    },
+    {
+      "path": "app/style.css",
+      "sha256": "864bb4a66750bb988c218668d28ccfea47b5a880aea4926e27c5ee5a18642da0"
+    },
+    {
+      "path": "next.config.ts",
+      "sha256": "3e13d0b861e3058e2f69f40dbf4f24e7828611797488b820988104706f95beac"
+    },
+    {
+      "path": "package-lock.json",
+      "sha256": "516260fa3554fb45b470f61d2fa01d7745f6d499033abd197076c21fd265ac66"
+    },
+    {
+      "path": "package.json",
+      "sha256": "682a14e2d77822fecd27086454c52523e613a2ff221bc31a06e954dac6ab1450"
+    },
+    {
+      "path": "server/app.ts",
+      "sha256": "42ca7842fdd94ac1fe291bbba40f13b0bc898ba9d007fa3edd90ef44f7a363c5"
+    },
+    {
+      "path": "server/auth.ts",
+      "sha256": "f1b7a7880ebae7219eff152ff17064da5507808ee9b0a99cc35e8acc64701288"
+    },
+    {
+      "path": "server/check.ts",
+      "sha256": "d997b5060e7fc89d9fcc157057c42990c863a248541b6d6a3950e0772cc8c4ca"
+    },
+    {
+      "path": "server/contracts.ts",
+      "sha256": "795237e965e35f5e3c363183ae97cd8af51874aab88bebfdb0a38bc8f2a20e17"
+    },
+    {
+      "path": "server/database.ts",
+      "sha256": "1c31f6f034501fb3f2c55aa4e6befde1874dfdf7ca13b728565bfda1f80aa768"
+    },
+    {
+      "path": "server/history.ts",
+      "sha256": "c19bbe0310835c0c7c5379b7e52acde780c228178538345158f0676112d9b44a"
+    },
+    {
+      "path": "server/main.ts",
+      "sha256": "3459b95b964eef79398e73661f9f96395a4124dc1c741f6f434cf47bd0b51847"
+    },
+    {
+      "path": "server/mapping.ts",
+      "sha256": "27912615f39c4739f93214e079a162a5efca6412bcae23a74d52d129313ccb6a"
+    },
+    {
+      "path": "server/migrate.ts",
+      "sha256": "bfc052895b6c46383e0b360775574dd619ecf92f7e64ade83af274bac6e78ce2"
+    },
+    {
+      "path": "server/migrations/001_monitors.sql",
+      "sha256": "e695bcf8abe711f63b8eee31f9ca98bb50eb90921349156bffe183bfddacc86e"
+    },
+    {
+      "path": "server/migrations/002_check_runs.sql",
+      "sha256": "13878c39225e292140a81312f98c7db76da3a3c49aa105218b1fc1193ee8f389"
+    },
+    {
+      "path": "server/migrations/003_sessions.sql",
+      "sha256": "097df3920042f49231f1dff9589f621acce0cc6c72fa680ad8e7fad0c8d84db9"
+    },
+    {
+      "path": "server/migrations/004_monitor_ownership.sql",
+      "sha256": "02cfe4dc77a1218330da13a71044b0d5557d6ab134c2832017ff7a25fd14fcb9"
+    },
+    {
+      "path": "server/migrations/005_check_history_index.sql",
+      "sha256": "1cc86167592c56f70baf8362969db688db73ddf802bc36fd33adc8abd3c6931e"
+    },
+    {
+      "path": "server/migrations/006_check_queue.sql",
+      "sha256": "492868fcc0e95ccec1c5200f980ae55816aea5d2c64a28e65b9c96f9095376af"
+    },
+    {
+      "path": "server/migrations/007_check_ownership.sql",
+      "sha256": "9c248b9d430088c30ca417d3b49fd7f9e9e85457393f42fe55a913de62458b31"
+    },
+    {
+      "path": "server/migrations/008_check_lease.sql",
+      "sha256": "4c9346ec915c848688864c6209093dadad8459e0860996310a4bb21b3267dfd2"
+    },
+    {
+      "path": "server/migrations/009_outbound_policy_result.sql",
+      "sha256": "e066779fee21645e0894c10a80ab2ef52835f75cd60524dfaf948bdaeb0394c9"
+    },
+    {
+      "path": "server/migrations/010_failed_history_index.sql",
+      "sha256": "f9ae03555556ee1eb817c21423ce70dab3d1ceebeec57297191c6868ce01f583"
+    },
+    {
+      "path": "server/model.ts",
+      "sha256": "76a3bcf699ce2485559387422c2f42479861e8a6d99cb0ff1491819126fcfffa"
+    },
+    {
+      "path": "server/operations.ts",
+      "sha256": "2e68545cfca280ab30d7bb711ca8a18c9ec96735335d6de9597c7287699226f3"
+    },
+    {
+      "path": "server/outbound.ts",
+      "sha256": "2d1d4ef54512c1b2c74d926ea0ad13921613cb82da1fc9702714f52b17cf46fb"
+    },
+    {
+      "path": "server/password.ts",
+      "sha256": "6e5dbfe1c59ed43f081d45d8098fa82350c2763c1c4a51dc4bfa2c869c260365"
+    },
+    {
+      "path": "server/schema.ts",
+      "sha256": "f1fd635c6879c06214b4a911771b97a092744812bcc118782f62431e773dfa47"
+    },
+    {
+      "path": "server/worker.ts",
+      "sha256": "3706059240099ef29c7a2091afe31e11f7d47ea31c063bf5d8a776b8a3c33448"
+    },
+    {
+      "path": "tsconfig.json",
+      "sha256": "3155624b053efbffb8db1fe0daa189542e85a09fa75ed0f6f5b64b3771be3a0c"
+    }
+  ],
+  "installedNextSourceSha256": "cf15b6389af3269979848e6cc23ecd7373051bb459e9ec370bfe8b05ee908285",
+  "sourceOnly": true,
+  "newBuilds": 0,
+  "newScenarioRuns": 0,
+  "historicalFrontendExitCode": null,
+  "historicalLimitation": "Repair1 recorded the failed assertion but not the frontend exit code; 143 is the pinned source behavior, not an observed historical exit."
+}
diff --git a/evidence/phase-1/E24/results/repair2-frontend-build.console.txt b/evidence/phase-1/E24/results/repair2-frontend-build.console.txt
new file mode 100644
index 0000000..9d2db79
--- /dev/null
+++ b/evidence/phase-1/E24/results/repair2-frontend-build.console.txt
@@ -0,0 +1,66 @@
+Compose can now delegate builds to bake for better performance.
+ To do so, set COMPOSE_BAKE=true.
+#0 building with "desktop-linux" instance using docker driver
+
+#1 [frontend internal] load build definition from Dockerfile
+#1 transferring dockerfile: 2.12kB done
+#1 DONE 0.0s
+
+#2 [frontend internal] load metadata for docker.io/library/node:24.19.0-bookworm-slim@sha256:a9f5f7c91a432850b2a8a7797adf5eadb6c733ceed61167806cee7ea7fbc29df
+#2 DONE 1.6s
+
+#3 [frontend internal] load .dockerignore
+#3 transferring context: 189B done
+#3 DONE 0.0s
+
+#4 [frontend dependencies 1/4] FROM docker.io/library/node:24.19.0-bookworm-slim@sha256:a9f5f7c91a432850b2a8a7797adf5eadb6c733ceed61167806cee7ea7fbc29df
+#4 resolve docker.io/library/node:24.19.0-bookworm-slim@sha256:a9f5f7c91a432850b2a8a7797adf5eadb6c733ceed61167806cee7ea7fbc29df 0.0s done
+#4 DONE 0.0s
+
+#5 [frontend internal] load build context
+#5 transferring context: 38.04kB done
+#5 DONE 0.0s
+
+#6 [frontend web-build 1/4] COPY next.config.ts tsconfig.json ./
+#6 CACHED
+
+#7 [frontend web-build 2/4] COPY app ./app
+#7 CACHED
+
+#8 [frontend web-build 3/4] COPY server ./server
+#8 CACHED
+
+#9 [frontend web-build 4/4] RUN npm run build
+#9 CACHED
+
+#10 [frontend dependencies 2/4] WORKDIR /app
+#10 CACHED
+
+#11 [frontend dependencies 3/4] COPY package.json package-lock.json ./
+#11 CACHED
+
+#12 [frontend dependencies 4/4] RUN npm ci --no-audit --no-fund --fetch-retries=0
+#12 CACHED
+
+#13 [frontend frontend 3/5] COPY --from=web-build --chown=node:node /app/.next/standalone ./
+#13 CACHED
+
+#14 [frontend frontend 4/5] COPY --from=web-build --chown=node:node /app/.next/static ./.next/static
+#14 CACHED
+
+#15 [frontend frontend 5/5] RUN node -e 'const fs = require("node:fs");   const { createHash } = require("node:crypto");   const file = "node_modules/next/dist/server/lib/start-server.js";   const source = fs.readFileSync(file, "utf8");   if (createHash("sha256").update(source).digest("hex") !== "cf15b6389af3269979848e6cc23ecd7373051bb459e9ec370bfe8b05ee908285")     throw new Error("Review the pinned Next SIGTERM cleanup before updating this image.");   fs.writeFileSync(file, source.replace("process.exit(143);", "process.exit(0); // Completed SIGTERM cleanup is successful in this image."));'
+#15 DONE 0.2s
+
+#16 [frontend] exporting to image
+#16 exporting layers 0.0s done
+#16 exporting manifest sha256:a02e2d688d4be21c0ececa94856eac5b844b7b34cbdff48e5dc4e48f1e2d4882 done
+#16 exporting config sha256:e57d14a617874567f1941c93a8c683e3c713f91cd6f21eeaf1b8d3d2b14674ea done
+#16 exporting attestation manifest sha256:8dff44f25a74321686718a08c18cf0ea592257453d2fadfbaa773aa2247e72df done
+#16 exporting manifest list sha256:4294a766c1255bb773973ee00af8670c1eaf3315991844d938ebb5ad5396da67 done
+#16 naming to docker.io/library/wse-fundamentals-e24-frontend:local done
+#16 unpacking to docker.io/library/wse-fundamentals-e24-frontend:local 0.0s done
+#16 DONE 0.1s
+
+#17 [frontend] resolving provenance for metadata file
+#17 DONE 0.0s
+ frontend  Built
diff --git a/evidence/phase-1/E24/results/repair2-frontend-build.json b/evidence/phase-1/E24/results/repair2-frontend-build.json
new file mode 100644
index 0000000..e1121e3
--- /dev/null
+++ b/evidence/phase-1/E24/results/repair2-frontend-build.json
@@ -0,0 +1,128 @@
+{
+  "at": "2026-08-28T07:31:04.691711+00:00",
+  "actor": "repair2",
+  "authorizedFrontendBuilds": 1,
+  "buildInvocations": 1,
+  "commands": [
+    {
+      "command": [
+        "docker",
+        "image",
+        "inspect",
+        "--format",
+        "{{json .Id}}",
+        "wse-fundamentals-e24-backend:local",
+        "wse-fundamentals-e24-frontend:local"
+      ],
+      "exitCode": 0,
+      "durationMs": 57,
+      "console": "output/phase-1/e24/repair2-images-before.console.txt",
+      "consoleSha256": "c9070987f87cf33f0b3bdfbf79c674f617551b1df9b9f471974246de50ea775c",
+      "consoleComplete": true
+    },
+    {
+      "command": [
+        "docker",
+        "compose",
+        "--project-name",
+        "wse-fundamentals-e24",
+        "-f",
+        "compose.production.yaml",
+        "build",
+        "frontend"
+      ],
+      "exitCode": 0,
+      "durationMs": 2520,
+      "console": "output/phase-1/e24/repair2-frontend-build.console.txt",
+      "consoleSha256": "a3e2af2a6801b0d731217ece396806eb9a6b3935710a9806cc64005ecd71fa27",
+      "consoleComplete": true
+    },
+    {
+      "command": [
+        "docker",
+        "image",
+        "inspect",
+        "--format",
+        "{{json .Id}}",
+        "wse-fundamentals-e24-backend:local",
+        "wse-fundamentals-e24-frontend:local"
+      ],
+      "exitCode": 0,
+      "durationMs": 68,
+      "console": "output/phase-1/e24/repair2-images-after.console.txt",
+      "consoleSha256": "42187f3f151de995e8d226ae68a308da56401345a416ffd81ffcd8b71b272e24",
+      "consoleComplete": true
+    }
+  ],
+  "dockerfileSha256": "f863bb2504d305db0378ea0d926c3b9eff6ba98e7af627746415b988be4391ef",
+  "observerSha256": "71566e13a522a1b83f786eaca976d2d1475d60e5a26a76d4f64a9c03859cffe5",
+  "result": "PASS",
+  "imagesBefore": {
+    "wse-fundamentals-e24-backend:local": "sha256:2f6918aed823d07ac1e05980b7747bf49e0912a027326ef3c5713c98fc5bfbc8",
+    "wse-fundamentals-e24-frontend:local": "sha256:7cd37b250c2e45bcc99271b688a74336dcbee88d10a29251edc8cbb61f68de16"
+  },
+  "buildProgress": [
+    "#0 building with \"desktop-linux\" instance using docker driver",
+    "#1 [frontend internal] load build definition from Dockerfile",
+    "#1 transferring dockerfile: 2.12kB done",
+    "#1 DONE 0.0s",
+    "#2 [frontend internal] load metadata for docker.io/library/node:24.19.0-bookworm-slim@sha256:a9f5f7c91a432850b2a8a7797adf5eadb6c733ceed61167806cee7ea7fbc29df",
+    "#2 DONE 1.6s",
+    "#3 [frontend internal] load .dockerignore",
+    "#3 transferring context: 189B done",
+    "#3 DONE 0.0s",
+    "#4 [frontend dependencies 1/4] FROM docker.io/library/node:24.19.0-bookworm-slim@sha256:a9f5f7c91a432850b2a8a7797adf5eadb6c733ceed61167806cee7ea7fbc29df",
+    "#4 resolve docker.io/library/node:24.19.0-bookworm-slim@sha256:a9f5f7c91a432850b2a8a7797adf5eadb6c733ceed61167806cee7ea7fbc29df 0.0s done",
+    "#4 DONE 0.0s",
+    "#5 [frontend internal] load build context",
+    "#5 transferring context: 38.04kB done",
+    "#5 DONE 0.0s",
+    "#6 [frontend web-build 1/4] COPY next.config.ts tsconfig.json ./",
+    "#6 CACHED",
+    "#7 [frontend web-build 2/4] COPY app ./app",
+    "#7 CACHED",
+    "#8 [frontend web-build 3/4] COPY server ./server",
+    "#8 CACHED",
+    "#9 [frontend web-build 4/4] RUN npm run build",
+    "#9 CACHED",
+    "#10 [frontend dependencies 2/4] WORKDIR /app",
+    "#10 CACHED",
+    "#11 [frontend dependencies 3/4] COPY package.json package-lock.json ./",
+    "#11 CACHED",
+    "#12 [frontend dependencies 4/4] RUN npm ci --no-audit --no-fund --fetch-retries=0",
+    "#12 CACHED",
+    "#13 [frontend frontend 3/5] COPY --from=web-build --chown=node:node /app/.next/standalone ./",
+    "#13 CACHED",
+    "#14 [frontend frontend 4/5] COPY --from=web-build --chown=node:node /app/.next/static ./.next/static",
+    "#14 CACHED",
+    "#15 [frontend frontend 5/5] RUN node -e 'const fs = require(\"node:fs\");   const { createHash } = require(\"node:crypto\");   const file = \"node_modules/next/dist/server/lib/start-server.js\";   const source = fs.readFileSync(file, \"utf8\");   if (createHash(\"sha256\").update(source).digest(\"hex\") !== \"cf15b6389af3269979848e6cc23ecd7373051bb459e9ec370bfe8b05ee908285\")     throw new Error(\"Review the pinned Next SIGTERM cleanup before updating this image.\");   fs.writeFileSync(file, source.replace(\"process.exit(143);\", \"process.exit(0); // Completed SIGTERM cleanup is successful in this image.\"));'",
+    "#15 DONE 0.2s",
+    "#16 [frontend] exporting to image",
+    "#16 exporting layers 0.0s done",
+    "#16 exporting manifest sha256:a02e2d688d4be21c0ececa94856eac5b844b7b34cbdff48e5dc4e48f1e2d4882 done",
+    "#16 exporting config sha256:e57d14a617874567f1941c93a8c683e3c713f91cd6f21eeaf1b8d3d2b14674ea done",
+    "#16 exporting attestation manifest sha256:8dff44f25a74321686718a08c18cf0ea592257453d2fadfbaa773aa2247e72df done",
+    "#16 exporting manifest list sha256:4294a766c1255bb773973ee00af8670c1eaf3315991844d938ebb5ad5396da67 done",
+    "#16 naming to docker.io/library/wse-fundamentals-e24-frontend:local done",
+    "#16 unpacking to docker.io/library/wse-fundamentals-e24-frontend:local 0.0s done",
+    "#16 DONE 0.1s",
+    "#17 [frontend] resolving provenance for metadata file",
+    "#17 DONE 0.0s"
+  ],
+  "cachedStepIds": [
+    "6",
+    "7",
+    "8",
+    "9",
+    "10",
+    "11",
+    "12",
+    "13",
+    "14"
+  ],
+  "imagesAfter": {
+    "wse-fundamentals-e24-backend:local": "sha256:2f6918aed823d07ac1e05980b7747bf49e0912a027326ef3c5713c98fc5bfbc8",
+    "wse-fundamentals-e24-frontend:local": "sha256:4294a766c1255bb773973ee00af8670c1eaf3315991844d938ebb5ad5396da67"
+  },
+  "backendImageUnchanged": true
+}
diff --git a/evidence/phase-1/E24/results/repair2-frontend-build.started.json b/evidence/phase-1/E24/results/repair2-frontend-build.started.json
new file mode 100644
index 0000000..8ef89f1
--- /dev/null
+++ b/evidence/phase-1/E24/results/repair2-frontend-build.started.json
@@ -0,0 +1,10 @@
+{
+  "at": "2026-08-28T07:31:04.691711+00:00",
+  "actor": "repair2",
+  "authorizedFrontendBuilds": 1,
+  "buildInvocations": 0,
+  "commands": [],
+  "dockerfileSha256": "f863bb2504d305db0378ea0d926c3b9eff6ba98e7af627746415b988be4391ef",
+  "observerSha256": "71566e13a522a1b83f786eaca976d2d1475d60e5a26a76d4f64a9c03859cffe5",
+  "result": "NOT_RUN"
+}
diff --git a/evidence/phase-1/E24/results/repair2-images-after.console.txt b/evidence/phase-1/E24/results/repair2-images-after.console.txt
new file mode 100644
index 0000000..10c46de
--- /dev/null
+++ b/evidence/phase-1/E24/results/repair2-images-after.console.txt
@@ -0,0 +1,2 @@
+"sha256:2f6918aed823d07ac1e05980b7747bf49e0912a027326ef3c5713c98fc5bfbc8"
+"sha256:4294a766c1255bb773973ee00af8670c1eaf3315991844d938ebb5ad5396da67"
diff --git a/evidence/phase-1/E24/results/repair2-images-before.console.txt b/evidence/phase-1/E24/results/repair2-images-before.console.txt
new file mode 100644
index 0000000..8004fb8
--- /dev/null
+++ b/evidence/phase-1/E24/results/repair2-images-before.console.txt
@@ -0,0 +1,2 @@
+"sha256:2f6918aed823d07ac1e05980b7747bf49e0912a027326ef3c5713c98fc5bfbc8"
+"sha256:7cd37b250c2e45bcc99271b688a74336dcbee88d10a29251edc8cbb61f68de16"
diff --git a/evidence/phase-1/E24/results/repair2-source-audit.json b/evidence/phase-1/E24/results/repair2-source-audit.json
new file mode 100644
index 0000000..7faef43
--- /dev/null
+++ b/evidence/phase-1/E24/results/repair2-source-audit.json
@@ -0,0 +1,59 @@
+{
+  "at": "2026-08-28T07:29:29.538445+00:00",
+  "head": "c629006315fd762f372d73f88d008f3753b055ec",
+  "sourceOnly": true,
+  "newBuilds": 0,
+  "newScenarioRuns": 0,
+  "checks": [
+    {
+      "command": [
+        "fnm",
+        "exec",
+        "--using",
+        "24.19.0",
+        "node",
+        "--check",
+        "test/container-smoke.mjs"
+      ],
+      "stdin": null,
+      "exitCode": 0,
+      "durationMs": 204,
+      "stdout": "",
+      "stderr": ""
+    },
+    {
+      "command": [
+        "fnm",
+        "exec",
+        "--using",
+        "24.19.0",
+        "node",
+        "--check"
+      ],
+      "stdin": "Dockerfile RUN JavaScript only",
+      "exitCode": 0,
+      "durationMs": 46,
+      "stdout": "",
+      "stderr": ""
+    }
+  ],
+  "backendAndBuildStagesUnchanged": true,
+  "observerOnlyAuthorizedChanges": true,
+  "dockerfileSha256": "f863bb2504d305db0378ea0d926c3b9eff6ba98e7af627746415b988be4391ef",
+  "observerSha256": "71566e13a522a1b83f786eaca976d2d1475d60e5a26a76d4f64a9c03859cffe5",
+  "pinnedNextSourceSha256": "cf15b6389af3269979848e6cc23ecd7373051bb459e9ec370bfe8b05ee908285",
+  "expectedAdaptedSourceSha256": "aef480dd960db9c1d697bbff6a995b31fb52d6588baa967061759ce46200a449",
+  "artifactSourceDiff": [
+    "--- ",
+    "+++ ",
+    "@@ -372,7 +372,7 @@",
+    "                                 process.exit(130);",
+    "                                 break;",
+    "                             case 'SIGTERM':",
+    "-                                process.exit(143);",
+    "+                                process.exit(0); // Completed SIGTERM cleanup is successful in this image.",
+    "                                 break;",
+    "                             default:",
+    "                                 // Make sure all handled signals have explicit exit codes."
+  ]
+}
diff --git a/evidence/phase-1/E24/results/repair2-source-audit.started.json b/evidence/phase-1/E24/results/repair2-source-audit.started.json
new file mode 100644
index 0000000..d738d48
--- /dev/null
+++ b/evidence/phase-1/E24/results/repair2-source-audit.started.json
@@ -0,0 +1 @@
+{"at": "2026-08-28T07:29:29.243031+00:00", "sourceOnly": true}
diff --git a/evidence/phase-1/E24/results/repair2-source-review.patch b/evidence/phase-1/E24/results/repair2-source-review.patch
new file mode 100644
index 0000000..edde92b
--- /dev/null
+++ b/evidence/phase-1/E24/results/repair2-source-review.patch
@@ -0,0 +1,42 @@
+diff --git a/Dockerfile b/Dockerfile
+index bca44e6..7571426 100644
+--- a/Dockerfile
++++ b/Dockerfile
+@@ -29,5 +29,14 @@ WORKDIR /app
+ ENV NODE_ENV=production NEXT_TELEMETRY_DISABLED=1 HOSTNAME=0.0.0.0 PORT=4313 API_ORIGIN=http://api:4312
+ COPY --from=web-build --chown=node:node /app/.next/standalone ./
+ COPY --from=web-build --chown=node:node /app/.next/static ./.next/static
++# Next 16.3.3 finishes HTTP/server/trace cleanup before its sole SIGTERM exit.
++# This image reports that completed cleanup as success; review changed upstream bytes.
++RUN node -e 'const fs = require("node:fs"); \
++  const { createHash } = require("node:crypto"); \
++  const file = "node_modules/next/dist/server/lib/start-server.js"; \
++  const source = fs.readFileSync(file, "utf8"); \
++  if (createHash("sha256").update(source).digest("hex") !== "cf15b6389af3269979848e6cc23ecd7373051bb459e9ec370bfe8b05ee908285") \
++    throw new Error("Review the pinned Next SIGTERM cleanup before updating this image."); \
++  fs.writeFileSync(file, source.replace("process.exit(143);", "process.exit(0); // Completed SIGTERM cleanup is successful in this image."));'
+ USER node
+ CMD ["node", "server.js"]
+diff --git a/test/container-smoke.mjs b/test/container-smoke.mjs
+index e9e651a..65a4b40 100644
+--- a/test/container-smoke.mjs
++++ b/test/container-smoke.mjs
+@@ -15,7 +15,7 @@ const mode = process.argv[2] ?? 'smoke';
+ const actor = process.argv[3] ?? 'ci';
+ class SmokeFailure extends Error {}
+ function check(value, message) { if (!value) throw new SmokeFailure(message); }
+-check(['smoke', 'full'].includes(mode) && ['ci', 'author', 'repair1', 'root'].includes(actor), 'Explicit smoke/full and ci/author/repair1/root arguments required.');
++check(['smoke', 'full'].includes(mode) && ['ci', 'author', 'repair1', 'repair2', 'root'].includes(actor), 'Explicit smoke/full and ci/author/repair1/repair2/root arguments required.');
+ check(process.versions.node === scenario.runtime.node, 'Pinned host Node is required.');
+ const output = 'output/phase-1/e24';
+ await mkdir(output, { recursive: true });
+@@ -280,8 +280,8 @@ try {
+     await docker(['kill', '--signal=SIGTERM', id]);
+     const exitCode = Number(await docker(['wait', id]));
+     const elapsedMs = Math.round(performance.now() - stoppedAt);
+-    check(exitCode === 0 && elapsedMs <= scenario.runtime.shutdownMs[role], 'Actual role SIGTERM exit must be clean and within its frozen bound.');
+     report.signals[role] = { signal: 'SIGTERM', exitCode, elapsedMs, forced: false };
++    check(exitCode === 0 && elapsedMs <= scenario.runtime.shutdownMs[role], 'Actual role SIGTERM exit must be clean and within its frozen bound.');
+   }
+   stage = 'safe structured logs';
+   const logs = {};
diff --git a/evidence/phase-1/E24/verification.json b/evidence/phase-1/E24/verification.json
new file mode 100644
index 0000000..21fce3b
--- /dev/null
+++ b/evidence/phase-1/E24/verification.json
@@ -0,0 +1,908 @@
+{
+  "thread": "E24",
+  "profile": "phase-1",
+  "attempt": 3,
+  "specRevision": "2ada57a71cd34fa2fae9809415c362a8bbfcdf02",
+  "start": "9d0a1974e1f279146917b840e69bbee19dcfc0c4",
+  "failedHead": "f5156939760e4db9221999217c5f2b23ddc008d3",
+  "repair1AdoptionCommit": "c629006315fd762f372d73f88d008f3753b055ec",
+  "repair2SourceCommit": "35d8fc6e8c4985ca6e134b20f49d6db2b3665290",
+  "authorResult": "PASS",
+  "rootAcceptance": "PENDING",
+  "hostedCi": "NOT_EXECUTED",
+  "at": "2026-08-28T07:38:48.889344+00:00",
+  "frozenInputs": {
+    "evidence/phase-1/E24/scenario.json": "07b5b3e2b4bd1cfc456c415134adce61f05dfdd729fe803968b32a4d813c5847",
+    "evidence/phase-1/E24/baseline.mjs": "0a24805b65454cddc543d27708aec68092b34e28b67ca6ba605e3bc6602c79d6",
+    "evidence/phase-1/E24/fixture.ts": "10d311f2e2af33063d09add435ed282b20341e2b87eeffa10b6287dce61ca784"
+  },
+  "source": {
+    "dockerfileSha256": "f863bb2504d305db0378ea0d926c3b9eff6ba98e7af627746415b988be4391ef",
+    "observerSha256": "71566e13a522a1b83f786eaca976d2d1475d60e5a26a76d4f64a9c03859cffe5",
+    "installedNextSourceSha256": "cf15b6389af3269979848e6cc23ecd7373051bb459e9ec370bfe8b05ee908285",
+    "expectedImageNextSourceSha256": "aef480dd960db9c1d697bbff6a995b31fb52d6588baa967061759ce46200a449",
+    "hostNodeModulesUnchanged": true,
+    "unchangedSourceReference": "f5156939760e4db9221999217c5f2b23ddc008d3",
+    "unchangedFiles": [
+      {
+        "path": ".dockerignore",
+        "sha256": "03528998ff52cea302a211e3e48cd9d13b35656dc50c20bc7eae8e4063811a1d"
+      },
+      {
+        "path": ".github/workflows/check.yml",
+        "sha256": "ef1ac4312e02837fbba15fe129817e647b4ff4f0e57dbe79903f13be488610e5"
+      },
+      {
+        "path": ".node-version",
+        "sha256": "7e8a2fa94951112b894a3dbe3d05efef5e9263741fa49125f0a70f40fedab4cc"
+      },
+      {
+        "path": "app/layout.tsx",
+        "sha256": "4a1d88207aa0f9e4ce3a7369207b696e42c742589a0d54eb465f4d918f9312ef"
+      },
+      {
+        "path": "app/login/login-form.tsx",
+        "sha256": "7f1e3d135b999ad3891ba0902c8c7b33352adf77c2fce2c0a96021880e38af62"
+      },
+      {
+        "path": "app/login/page.tsx",
+        "sha256": "9d1ceedf370739bef4a30d33a6724f420575ac20f5dca418a2aa33df867c930c"
+      },
+      {
+        "path": "app/monitors/api.ts",
+        "sha256": "c595e0dbb7994facb03aa7c10e552dba8f1e3cae8cb2105ca5a73bb73cd773a5"
+      },
+      {
+        "path": "app/monitors/initial-state.ts",
+        "sha256": "91d966b3dae3c5a2e6c361724a2b43bbd3dcb06aea876a77d7b425ea0f8bb562"
+      },
+      {
+        "path": "app/monitors/monitor-workspace.tsx",
+        "sha256": "57809e5341c650b78cfb4244583eaf2bcc1dcb7d5d13f45c84904eda12ab7992"
+      },
+      {
+        "path": "app/monitors/page.tsx",
+        "sha256": "830d0583a53e23363e31debcff63c321f562ceb85170706d954152b45910675f"
+      },
+      {
+        "path": "app/monitors/server-data.ts",
+        "sha256": "dc2df71ed90a5caa0ab4d842989d8680a98c764a242d8032b6dd9d56ba6cdf79"
+      },
+      {
+        "path": "app/monitors/use-monitors.ts",
+        "sha256": "6da04e2505fb261eca81b677c7f27471f399893140c7aa10e08b97827e7042f5"
+      },
+      {
+        "path": "app/page.tsx",
+        "sha256": "84426886dcce50ded872e609644821c751cab148630048f5e2d4a323920a0163"
+      },
+      {
+        "path": "app/style.css",
+        "sha256": "864bb4a66750bb988c218668d28ccfea47b5a880aea4926e27c5ee5a18642da0"
+      },
+      {
+        "path": "compose.production.yaml",
+        "sha256": "9998b7fb98317185690b2e21ef6559a87ff6d53bf31168b9b22b3e7b5efd9d52"
+      },
+      {
+        "path": "compose.yaml",
+        "sha256": "461c9f187292e94d46272a6fddbb5d574e5f557cbfc49fff84584810d2e7bab8"
+      },
+      {
+        "path": "next.config.ts",
+        "sha256": "3e13d0b861e3058e2f69f40dbf4f24e7828611797488b820988104706f95beac"
+      },
+      {
+        "path": "package-lock.json",
+        "sha256": "516260fa3554fb45b470f61d2fa01d7745f6d499033abd197076c21fd265ac66"
+      },
+      {
+        "path": "package.json",
+        "sha256": "682a14e2d77822fecd27086454c52523e613a2ff221bc31a06e954dac6ab1450"
+      },
+      {
+        "path": "server/app.ts",
+        "sha256": "42ca7842fdd94ac1fe291bbba40f13b0bc898ba9d007fa3edd90ef44f7a363c5"
+      },
+      {
+        "path": "server/auth.ts",
+        "sha256": "f1b7a7880ebae7219eff152ff17064da5507808ee9b0a99cc35e8acc64701288"
+      },
+      {
+        "path": "server/check.ts",
+        "sha256": "d997b5060e7fc89d9fcc157057c42990c863a248541b6d6a3950e0772cc8c4ca"
+      },
+      {
+        "path": "server/contracts.ts",
+        "sha256": "795237e965e35f5e3c363183ae97cd8af51874aab88bebfdb0a38bc8f2a20e17"
+      },
+      {
+        "path": "server/database.ts",
+        "sha256": "1c31f6f034501fb3f2c55aa4e6befde1874dfdf7ca13b728565bfda1f80aa768"
+      },
+      {
+        "path": "server/history.ts",
+        "sha256": "c19bbe0310835c0c7c5379b7e52acde780c228178538345158f0676112d9b44a"
+      },
+      {
+        "path": "server/main.ts",
+        "sha256": "3459b95b964eef79398e73661f9f96395a4124dc1c741f6f434cf47bd0b51847"
+      },
+      {
+        "path": "server/mapping.ts",
+        "sha256": "27912615f39c4739f93214e079a162a5efca6412bcae23a74d52d129313ccb6a"
+      },
+      {
+        "path": "server/migrate.ts",
+        "sha256": "bfc052895b6c46383e0b360775574dd619ecf92f7e64ade83af274bac6e78ce2"
+      },
+      {
+        "path": "server/migrations/001_monitors.sql",
+        "sha256": "e695bcf8abe711f63b8eee31f9ca98bb50eb90921349156bffe183bfddacc86e"
+      },
+      {
+        "path": "server/migrations/002_check_runs.sql",
+        "sha256": "13878c39225e292140a81312f98c7db76da3a3c49aa105218b1fc1193ee8f389"
+      },
+      {
+        "path": "server/migrations/003_sessions.sql",
+        "sha256": "097df3920042f49231f1dff9589f621acce0cc6c72fa680ad8e7fad0c8d84db9"
+      },
+      {
+        "path": "server/migrations/004_monitor_ownership.sql",
+        "sha256": "02cfe4dc77a1218330da13a71044b0d5557d6ab134c2832017ff7a25fd14fcb9"
+      },
+      {
+        "path": "server/migrations/005_check_history_index.sql",
+        "sha256": "1cc86167592c56f70baf8362969db688db73ddf802bc36fd33adc8abd3c6931e"
+      },
+      {
+        "path": "server/migrations/006_check_queue.sql",
+        "sha256": "492868fcc0e95ccec1c5200f980ae55816aea5d2c64a28e65b9c96f9095376af"
+      },
+      {
+        "path": "server/migrations/007_check_ownership.sql",
+        "sha256": "9c248b9d430088c30ca417d3b49fd7f9e9e85457393f42fe55a913de62458b31"
+      },
+      {
+        "path": "server/migrations/008_check_lease.sql",
+        "sha256": "4c9346ec915c848688864c6209093dadad8459e0860996310a4bb21b3267dfd2"
+      },
+      {
+        "path": "server/migrations/009_outbound_policy_result.sql",
+        "sha256": "e066779fee21645e0894c10a80ab2ef52835f75cd60524dfaf948bdaeb0394c9"
+      },
+      {
+        "path": "server/migrations/010_failed_history_index.sql",
+        "sha256": "f9ae03555556ee1eb817c21423ce70dab3d1ceebeec57297191c6868ce01f583"
+      },
+      {
+        "path": "server/model.ts",
+        "sha256": "76a3bcf699ce2485559387422c2f42479861e8a6d99cb0ff1491819126fcfffa"
+      },
+      {
+        "path": "server/operations.ts",
+        "sha256": "2e68545cfca280ab30d7bb711ca8a18c9ec96735335d6de9597c7287699226f3"
+      },
+      {
+        "path": "server/outbound.ts",
+        "sha256": "2d1d4ef54512c1b2c74d926ea0ad13921613cb82da1fc9702714f52b17cf46fb"
+      },
+      {
+        "path": "server/password.ts",
+        "sha256": "6e5dbfe1c59ed43f081d45d8098fa82350c2763c1c4a51dc4bfa2c869c260365"
+      },
+      {
+        "path": "server/schema.ts",
+        "sha256": "f1fd635c6879c06214b4a911771b97a092744812bcc118782f62431e773dfa47"
+      },
+      {
+        "path": "server/worker.ts",
+        "sha256": "3706059240099ef29c7a2091afe31e11f7d47ea31c063bf5d8a776b8a3c33448"
+      },
+      {
+        "path": "test/auth.ts",
+        "sha256": "b220789d401e2a99eaef2f3f2802eeeeea33fcebaa6f6b60b687fead92ff090e"
+      },
+      {
+        "path": "test/browser-teardown.ts",
+        "sha256": "3f3b61fcc73759fbc8089ba6d7de58f7b3fb7237734d1ca3df89da0c87882774"
+      },
+      {
+        "path": "test/browser/contracts.spec.ts",
+        "sha256": "1cda4b9d6b69f108ab1d4dd3c874f7a5e825e9f4396e7dfa87c6a62db99a1b20"
+      },
+      {
+        "path": "test/browser/history.spec.ts",
+        "sha256": "30e84e574174c6a000d0978bf3e931f8e0835e006f7dc8bbb529efb2fcaf754b"
+      },
+      {
+        "path": "test/browser/idempotency.spec.ts",
+        "sha256": "5918afbcd04535689c3bfdf36f572d065966a26e5d89dad410fc874104267c76"
+      },
+      {
+        "path": "test/browser/lifecycle.spec.ts",
+        "sha256": "c49d0d95e4d91c1c008d131f50aba0a5ce8a41bc7fbb88b9b3c0d21ad1a18124"
+      },
+      {
+        "path": "test/browser/monitor.spec.ts",
+        "sha256": "78f67360f96f4eface911baeac2ea3644273eab70f18e5998c37c9b2789e401b"
+      },
+      {
+        "path": "test/browser/rendering.spec.ts",
+        "sha256": "415f53c21395cc3c5c4360299a0ee4b5fbaa65794dc95e2915d6bf0cb0c008d8"
+      },
+      {
+        "path": "test/browser/server-state-scenario.ts",
+        "sha256": "733ea4d0ead410d21ee129ac13e590321e2efe999c200a78617460d3a44f6f31"
+      },
+      {
+        "path": "test/browser/server-state.spec.ts",
+        "sha256": "74c54cbffdb384bb40e5773816da41ed82acf85e338752420dd5dbb802f3b2c6"
+      },
+      {
+        "path": "test/browser/session.ts",
+        "sha256": "4396d08779a95e85fb21b7f6c5808d923aa0e9668a091e451f000f05140a23dd"
+      },
+      {
+        "path": "test/contracts.test.ts",
+        "sha256": "1a9bb99877c9f73bbcc47ad026c4d3e6391b28a77e262ddc7208c28e43d1f747"
+      },
+      {
+        "path": "test/database.ts",
+        "sha256": "f8fd16ba72d596e19f6c8e7eb6de56ecd6fd64a72f92a722e562572b507e0b10"
+      },
+      {
+        "path": "test/e08-rsc-preload.mjs",
+        "sha256": "9d94fdbe323fc24d7c35fad0dff055986d9c3bc0a6dd80f9578c7b6364218871"
+      },
+      {
+        "path": "test/e08-rsc-transport.mjs",
+        "sha256": "2e5f1cccb74df15f9e28faf837bfeeb596fa4d978c5a878ec2067fa2517dd668"
+      },
+      {
+        "path": "test/execution.test.ts",
+        "sha256": "864a08f7fbf4ab3917d0d1680a861b57c850279729606b5dbf059df2b9c28b17"
+      },
+      {
+        "path": "test/fixture.ts",
+        "sha256": "3a38f5fa58f670cb5cf55bc03b461222ddf2cbf967f38d1da0076b205b67c367"
+      },
+      {
+        "path": "test/functional.test.ts",
+        "sha256": "8098b8ec7d0a717ef27833a48dc0a38aefa00a79783e1b4e21899f1d0cbfaf98"
+      },
+      {
+        "path": "test/outbound.test.ts",
+        "sha256": "8c0316c5b136538bf9d30006253c693ed53f6395c9ed02ecdf36db964e776703"
+      },
+      {
+        "path": "test/ownership.test.ts",
+        "sha256": "713cb94f73a6e278488910984193c36b52834ade0865ee1c2edfa28285e7a7e5"
+      },
+      {
+        "path": "test/persistence.test.ts",
+        "sha256": "cc414d2dae32e6e0814d6449291ea85c00f7a63c64259211438e3ccea8b27071"
+      },
+      {
+        "path": "test/prepare-browser-db.ts",
+        "sha256": "ccdcdd1761442f71eb97a017e622fe72e86392e3a3adac941a0c22db4d972173"
+      },
+      {
+        "path": "test/recovery.test.ts",
+        "sha256": "1167721f65b8750e821e9e9149ea28c08b5f140d367ef35e6915921468640920"
+      },
+      {
+        "path": "test/storage-contract.test.ts",
+        "sha256": "e3f94105304e7148d88a47c162a08fe474cfed3e1e7ce1b407bf41abe6de80e0"
+      },
+      {
+        "path": "test/storage-history.test.ts",
+        "sha256": "309841cd8d5a4bd5dd751e909c014058ef349d0eaf8216023070718badc6fd8c"
+      },
+      {
+        "path": "test/storage-schema.test.ts",
+        "sha256": "bc7dcda0e05921a3676c64eae9ba5d4282d7b8c4a5cd2a7a3651c954bc5c409e"
+      },
+      {
+        "path": "test/unit.test.ts",
+        "sha256": "6ea94aa491f58505592bdfcb7206d8817c3b01a74264a8d6f53df97c54848059"
+      },
+      {
+        "path": "test/worker.ts",
+        "sha256": "6af3906d1baace042612e77612746ccfad2fd09ef02a80b3cb22129e1a7818d8"
+      },
+      {
+        "path": "tsconfig.json",
+        "sha256": "3155624b053efbffb8db1fe0daa189542e85a09fa75ed0f6f5b64b3771be3a0c"
+      }
+    ]
+  },
+  "baseline": {
+    "invocations": 1,
+    "result": "REPRODUCED",
+    "method": "Fastify app.inject; not external HTTP",
+    "productMatchesStart": true,
+    "durationMs": 645,
+    "livenessStatus": 200,
+    "readinessStatus": 401,
+    "containerBuilds": 0,
+    "postgresStops": 0,
+    "outboundRequests": 0,
+    "artifact": "results/baseline.json"
+  },
+  "fullRuns": [
+    {
+      "actor": "author",
+      "result": "FAILED",
+      "exitCode": 1,
+      "durationMs": 6930,
+      "artifact": "results/full-author.json",
+      "failure": "Installed Compose rejected start --wait; scenario restoration did not complete."
+    },
+    {
+      "actor": "repair1",
+      "result": "FAILED",
+      "exitCode": 1,
+      "durationMs": 9183,
+      "artifact": "results/full-repair1.json",
+      "failure": "Frontend failed unchanged exit0/bounded-duration assertion; actual frontend exit value was not retained.",
+      "historicalFrontendExitCode": null
+    },
+    {
+      "actor": "repair2",
+      "result": "PASS",
+      "exitCode": 0,
+      "durationMs": 9725,
+      "wrapperDurationMs": 10177,
+      "artifact": "results/full-repair2.json",
+      "commands": 31,
+      "failedCommands": 0
+    }
+  ],
+  "repair2Build": {
+    "command": [
+      "docker",
+      "compose",
+      "--project-name",
+      "wse-fundamentals-e24",
+      "-f",
+      "compose.production.yaml",
+      "build",
+      "frontend"
+    ],
+    "exitCode": 0,
+    "durationMs": 2520,
+    "frontendImage": "sha256:4294a766c1255bb773973ee00af8670c1eaf3315991844d938ebb5ad5396da67",
+    "backendImage": "sha256:2f6918aed823d07ac1e05980b7747bf49e0912a027326ef3c5713c98fc5bfbc8",
+    "backendImageUnchanged": true,
+    "executedStep": "Frontend RUN guarded completed-SIGTERM adaptation; BuildKit reported0.2s",
+    "cachedSteps": [
+      "npm ci",
+      "npm run build",
+      "standalone copy",
+      "static copy"
+    ],
+    "cachedStepIds": [
+      "6",
+      "7",
+      "8",
+      "9",
+      "10",
+      "11",
+      "12",
+      "13",
+      "14"
+    ],
+    "consoleComplete": true,
+    "artifact": "results/repair2-frontend-build.json"
+  },
+  "newSourceOnlyChecks": [
+    {
+      "command": [
+        "fnm",
+        "exec",
+        "--using",
+        "24.19.0",
+        "node",
+        "--check",
+        "test/container-smoke.mjs"
+      ],
+      "stdin": null,
+      "exitCode": 0,
+      "durationMs": 204,
+      "stdout": "",
+      "stderr": ""
+    },
+    {
+      "command": [
+        "fnm",
+        "exec",
+        "--using",
+        "24.19.0",
+        "node",
+        "--check"
+      ],
+      "stdin": "Dockerfile RUN JavaScript only",
+      "exitCode": 0,
+      "durationMs": 46,
+      "stdout": "",
+      "stderr": ""
+    }
+  ],
+  "reusedGates": [
+    {
+      "gate": "compose-config",
+      "command": "/usr/bin/time -p docker compose --project-name wse-fundamentals-e24 -f compose.production.yaml -f evidence/phase-1/E24/compose.test.yaml config --quiet",
+      "exit": 0,
+      "realSeconds": 0.15,
+      "consoleComplete": true
+    },
+    {
+      "gate": "typecheck",
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run typecheck",
+      "exit": 0,
+      "realSeconds": 2.18,
+      "consoleComplete": true
+    },
+    {
+      "gate": "unit",
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:unit",
+      "exit": 0,
+      "realSeconds": 1.28,
+      "pass": 22,
+      "fail": 0,
+      "consoleComplete": true
+    },
+    {
+      "gate": "functional",
+      "command": "/usr/bin/time -p fnm exec --using 24.19.0 npm run test:functional",
+      "exit": 0,
+      "realSeconds": 10.55,
+      "pass": 15,
+      "fail": 0,
+      "consoleComplete": false,
+      "limitation": "The first polling result was truncated by exec tool max_output_tokens=6500 (original_token_count=26023). Stored chunks are the exact received text; missing routine structured logs cannot be reconstructed. The final pass/fail summary and duration were received in full. No rerun."
+    },
+    {
+      "gate": "production-build",
+      "command": "/usr/bin/time -p docker compose --project-name wse-fundamentals-e24 -f compose.production.yaml build api frontend",
+      "exit": 0,
+      "realSeconds": 34.44,
+      "consoleComplete": true
+    }
+  ],
+  "reuseScope": "Compose YAML, dependency pins, app/server/test sources except the two explicit observer changes are byte-identical to the adopted candidate. Old production-build proof covers unchanged backend; frontend image build and full container gate were newly run.",
+  "observed": {
+    "healthUp": [
+      {
+        "live": 200,
+        "ready": 200
+      },
+      {
+        "live": 200,
+        "ready": 200
+      }
+    ],
+    "healthDown": [
+      {
+        "live": 200,
+        "ready": 503
+      },
+      {
+        "live": 200,
+        "ready": 503
+      }
+    ],
+    "healthRestored": [
+      {
+        "live": 200,
+        "ready": 200
+      },
+      {
+        "live": 200,
+        "ready": 200
+      }
+    ],
+    "worker": {
+      "accepted202": true,
+      "queuedUnknownStatus": true,
+      "positiveQueueAge": true,
+      "activeObserved": true,
+      "sameIdTerminal503": true,
+      "claims": 1,
+      "recovered": 0,
+      "recoveryScans": 2
+    },
+    "browser": {
+      "version": "151.0.7922.34",
+      "login": true,
+      "list": true,
+      "detailRows": 4,
+      "ownerIsolation": true,
+      "staticStatus": 200,
+      "serverRouteStatus": 200,
+      "consoleErrors": 0,
+      "screenshots": 0,
+      "traces": 0
+    },
+    "cardinality": {
+      "before": 35,
+      "after": 35,
+      "distinctMissingIds": 10,
+      "errorCountDelta": 10
+    },
+    "outage": {
+      "rejectedStatus": 500,
+      "authoritativeRows": 5,
+      "preserved": true,
+      "noNewOutbound": true
+    },
+    "runtime": {
+      "api": {
+        "uid": 1000,
+        "executable": "/usr/local/bin/node",
+        "actualCommandLine": [
+          "node",
+          "server/main.ts"
+        ],
+        "node": "24.19.0",
+        "nodeEnv": "test",
+        "buildId": null,
+        "configuredCommand": [
+          "node",
+          "server/main.ts"
+        ],
+        "image": "sha256:2f6918aed823d07ac1e05980b7747bf49e0912a027326ef3c5713c98fc5bfbc8",
+        "hostPid": 95570,
+        "restarts": 0
+      },
+      "worker": {
+        "uid": 1000,
+        "executable": "/usr/local/bin/node",
+        "actualCommandLine": [
+          "node",
+          "server/worker.ts"
+        ],
+        "node": "24.19.0",
+        "nodeEnv": "test",
+        "buildId": null,
+        "configuredCommand": [
+          "node",
+          "server/worker.ts"
+        ],
+        "image": "sha256:2f6918aed823d07ac1e05980b7747bf49e0912a027326ef3c5713c98fc5bfbc8",
+        "hostPid": 95862,
+        "restarts": 0
+      },
+      "frontend": {
+        "uid": 1000,
+        "executable": "/usr/local/bin/node",
+        "actualCommandLine": [
+          "next-server (v"
+        ],
+        "node": "24.19.0",
+        "nodeEnv": "production",
+        "buildId": "1WFptx7upO35yWieeZ3XK",
+        "configuredCommand": [
+          "node",
+          "server.js"
+        ],
+        "image": "sha256:4294a766c1255bb773973ee00af8670c1eaf3315991844d938ebb5ad5396da67",
+        "hostPid": 95539,
+        "restarts": 0
+      }
+    },
+    "signals": {
+      "api": {
+        "signal": "SIGTERM",
+        "exitCode": 0,
+        "elapsedMs": 286,
+        "forced": false
+      },
+      "worker": {
+        "signal": "SIGTERM",
+        "exitCode": 0,
+        "elapsedMs": 190,
+        "forced": false
+      },
+      "frontend": {
+        "signal": "SIGTERM",
+        "exitCode": 0,
+        "elapsedMs": 151,
+        "forced": false
+      }
+    },
+    "logging": {
+      "apiJsonLines": 50,
+      "workerJsonLines": 18,
+      "correlatedCheck": true,
+      "signalHandlersObserved": true,
+      "sentinelLeaks": 0,
+      "scannedRuntimeSentinels": 15,
+      "rawLogsRetained": false
+    },
+    "cleanup": {
+      "schemaDropped": true,
+      "runtimeRemoved": true,
+      "postgresRestored": true,
+      "portsFree": true,
+      "errors": []
+    }
+  },
+  "budget": {
+    "baselineInvocationsCumulative": 1,
+    "originalAuthorFullRuns": 1,
+    "repair1FullRuns": 1,
+    "repair2FullRuns": 1,
+    "authorAndRepairFullRunsCumulative": 3,
+    "authorAndRepairFullFailuresCumulative": 2,
+    "rootFullRunsSoFar": 0,
+    "rootFullRunReserved": 1,
+    "repairTasksUsed": 2,
+    "repairTasksMaximum": 2,
+    "originalBuildInvocations": 1,
+    "repair1BuildInvocations": 0,
+    "repair2FrontendBuildInvocations": 1,
+    "scenarioPostgresStopsCumulative": 3,
+    "scenarioPostgresRestoreAttemptsCumulative": 3,
+    "scenarioPostgresRestoresSuccessfulCumulative": 2,
+    "cleanupPostgresRestoreAttempts": 1,
+    "cleanupPostgresRestoresSuccessful": 1,
+    "totalExecutedPostgresRestoreAttempts": 4,
+    "totalPostgresRestoresSuccessful": 3,
+    "permissionDeniedCleanupPreflightsWithoutRestore": 1,
+    "automaticRetries": 0,
+    "loadRuns": 0,
+    "parameterSweeps": 0,
+    "exploratorySignalRuns": 0,
+    "e11CrashReruns": 0,
+    "e20DatasetReruns": 0
+  },
+  "limitations": [
+    "Independent root acceptance remains pending; no progress tag was created by this agent.",
+    "Repair1 did not retain the actual failed frontend exit value;143 is the pinned source behavior, not historical runtime evidence.",
+    "The final log/sentinel scan was not reached in either failed predecessor.",
+    "The original functional gate first output chunk was truncated at max_output_tokens6500, original_token_count26023. Stored chunks are exact received text; missing routine logs cannot be reconstructed. The15-pass/0-fail summary and duration were received in full.",
+    "Repair2 reused cached npm installation and Next compilation; it did not execute them again.",
+    "API/worker container fixture overlay uses NODE_ENV=test for the exact loopback exception; production image/default compose use production. Frontend stays production.",
+    "No hosted CI run is claimed."
+  ],
+  "artifacts": [
+    {
+      "original": "output/phase-1/e24/author-pre-container-invocations.json",
+      "copy": "evidence/phase-1/E24/results/author-pre-container-invocations.json",
+      "sha256": "7f5595349f9a0cf7a4bc27003b3dcc1e4dc72abc071dfbaf5c0ad0d931d1148a",
+      "bytes": 48756,
+      "byteIdentical": true
+    },
+    {
+      "original": "output/phase-1/e24/baseline.json",
+      "copy": "evidence/phase-1/E24/results/baseline.json",
+      "sha256": "4da653a0f6e05c1fcb26bdf85b9ec4015fbf3ef54e71e8ce9419af3895e0c684",
+      "bytes": 935,
+      "byteIdentical": true
+    },
+    {
+      "original": "output/phase-1/e24/baseline.started.json",
+      "copy": "evidence/phase-1/E24/results/baseline.started.json",
+      "sha256": "7ff24402449c9abdde5a6eba834884558431d6a8377dee16d1c7cf581f201657",
+      "bytes": 308,
+      "byteIdentical": true
+    },
+    {
+      "original": "output/phase-1/e24/full-author.console.txt",
+      "copy": "evidence/phase-1/E24/results/full-author.console.txt",
+      "sha256": "1c9d91904186621fee63b6e29b2d8a249446d783aac69bd7fd720c9e5533fb22",
+      "bytes": 352,
+      "byteIdentical": true
+    },
+    {
+      "original": "output/phase-1/e24/full-author.json",
+      "copy": "evidence/phase-1/E24/results/full-author.json",
+      "sha256": "34eba201f813d2486ea87b4fdf9694ae5614ee974ef147a7f535480dc18efcc0",
+      "bytes": 5900,
+      "byteIdentical": true
+    },
+    {
+      "original": "output/phase-1/e24/full-author.started.json",
+      "copy": "evidence/phase-1/E24/results/full-author.started.json",
+      "sha256": "b75153248eba165b2b6e5e0015ee51bd8d6250aa3554d1563752ace0c1039fb4",
+      "bytes": 65,
+      "byteIdentical": true
+    },
+    {
+      "original": "output/phase-1/e24/full-repair1.console.json",
+      "copy": "evidence/phase-1/E24/results/full-repair1.console.json",
+      "sha256": "be66b29b962a7d9a10a04f1627e750dd9625fd37474e98afdc68891ef87c65e3",
+      "bytes": 1604,
+      "byteIdentical": true
+    },
+    {
+      "original": "output/phase-1/e24/full-repair1.console.txt",
+      "copy": "evidence/phase-1/E24/results/full-repair1.console.txt",
+      "sha256": "9ab2858de5e3c48ef06c4278ce7ccc089de4c753f232b1344e9e68920045b6ff",
+      "bytes": 365,
+      "byteIdentical": true
+    },
+    {
+      "original": "output/phase-1/e24/full-repair1.json",
+      "copy": "evidence/phase-1/E24/results/full-repair1.json",
+      "sha256": "6b6c39a9dd24ca9934e7453f5298b3da8bd7cb7f1ecd9f83bbaa6e0f0d779295",
+      "bytes": 13080,
+      "byteIdentical": true
+    },
+    {
+      "original": "output/phase-1/e24/full-repair1.started.json",
+      "copy": "evidence/phase-1/E24/results/full-repair1.started.json",
+      "sha256": "8e768a69ac9d14d49da4a45173e0a81754df7eb0b363c8321729dde7d7c6c81b",
+      "bytes": 66,
+      "byteIdentical": true
+    },
+    {
+      "original": "output/phase-1/e24/full-repair2.console.json",
+      "copy": "evidence/phase-1/E24/results/full-repair2.console.json",
+      "sha256": "f3df16bdb04fc9d60eb5958eb8f8eadaf18a18147da305e34b86ed6d549fb5d7",
+      "bytes": 569,
+      "byteIdentical": true
+    },
+    {
+      "original": "output/phase-1/e24/full-repair2.console.started.json",
+      "copy": "evidence/phase-1/E24/results/full-repair2.console.started.json",
+      "sha256": "7e0cf3c7b2c48382362ef3f66252e2eea88584da0112b43bb2f57f22c7a8c341",
+      "bytes": 270,
+      "byteIdentical": true
+    },
+    {
+      "original": "output/phase-1/e24/full-repair2.console.txt",
+      "copy": "evidence/phase-1/E24/results/full-repair2.console.txt",
+      "sha256": "9736a65e2d1ccd1110f8fa1053e39ba63a5d2e2e81325594d5e4029b1b56791d",
+      "bytes": 176,
+      "byteIdentical": true
+    },
+    {
+      "original": "output/phase-1/e24/full-repair2.json",
+      "copy": "evidence/phase-1/E24/results/full-repair2.json",
+      "sha256": "3831ddb53ec3bc54ba4f0f7c0ba02cb3d683ff82f77a302df950ae1e1eac887e",
+      "bytes": 19618,
+      "byteIdentical": true
+    },
+    {
+      "original": "output/phase-1/e24/full-repair2.started.json",
+      "copy": "evidence/phase-1/E24/results/full-repair2.started.json",
+      "sha256": "b715e4422a8223b51db61360c9ac2eb8fc8e2fa046fab7d8fae49ae8e05d0a79",
+      "bytes": 66,
+      "byteIdentical": true
+    },
+    {
+      "original": "output/phase-1/e24/repair1-cleanup-inspection.json",
+      "copy": "evidence/phase-1/E24/results/repair1-cleanup-inspection.json",
+      "sha256": "a36fd9470c61ebd2511b50b70c36ce76bcb53659f7124bfd7383c94fccc2e450",
+      "bytes": 13247,
+      "byteIdentical": true
+    },
+    {
+      "original": "output/phase-1/e24/repair1-cleanup-restore-approved.json",
+      "copy": "evidence/phase-1/E24/results/repair1-cleanup-restore-approved.json",
+      "sha256": "70d717a759af5bc9246674b3779627dbc41e5aeccee922ed46c4bdf509158f37",
+      "bytes": 5183,
+      "byteIdentical": true
+    },
+    {
+      "original": "output/phase-1/e24/repair1-cleanup-restore-approved.started.json",
+      "copy": "evidence/phase-1/E24/results/repair1-cleanup-restore-approved.started.json",
+      "sha256": "0626181cde88f72050d42278ea3befd2d834fbb0805689fc92160b170c4fd04c",
+      "bytes": 193,
+      "byteIdentical": true
+    },
+    {
+      "original": "output/phase-1/e24/repair1-cleanup-restore.json",
+      "copy": "evidence/phase-1/E24/results/repair1-cleanup-restore.json",
+      "sha256": "94d5bfa99a4eaa69efb5bd602dfbcf60bd7b47eec0ea365fd5e029e5716ea558",
+      "bytes": 1028,
+      "byteIdentical": true
+    },
+    {
+      "original": "output/phase-1/e24/repair1-cleanup-restore.started.json",
+      "copy": "evidence/phase-1/E24/results/repair1-cleanup-restore.started.json",
+      "sha256": "b6217d6a1a40c6840d78b256295cf3412286eec5519dfc639813013cdde821e9",
+      "bytes": 152,
+      "byteIdentical": true
+    },
+    {
+      "original": "output/phase-1/e24/repair1-image-reuse.json",
+      "copy": "evidence/phase-1/E24/results/repair1-image-reuse.json",
+      "sha256": "774a869149a39123a2771125253b8c8acee0d3690ec1989b39752870fb69245c",
+      "bytes": 2310,
+      "byteIdentical": true
+    },
+    {
+      "original": "output/phase-1/e24/repair1-reuse-audit.json",
+      "copy": "evidence/phase-1/E24/results/repair1-reuse-audit.json",
+      "sha256": "aa742253f9070580ce187e91c50e06b8f85623106d4b98166138eed61287bde2",
+      "bytes": 7832,
+      "byteIdentical": true
+    },
+    {
+      "original": "output/phase-1/e24/repair1-schema-cleanup.json",
+      "copy": "evidence/phase-1/E24/results/repair1-schema-cleanup.json",
+      "sha256": "fcfd4106cc01702033e63e21b1a25d4ad0fcdc5bf00c4ecacb1dcdcae9c557db",
+      "bytes": 2408,
+      "byteIdentical": true
+    },
+    {
+      "original": "output/phase-1/e24/repair1-schema-cleanup.started.json",
+      "copy": "evidence/phase-1/E24/results/repair1-schema-cleanup.started.json",
+      "sha256": "9221d4238379f522dd28406e5990d2c3022501cca7feb486b4d29c47dbbe8582",
+      "bytes": 208,
+      "byteIdentical": true
+    },
+    {
+      "original": "output/phase-1/e24/repair2-adoption-audit.json",
+      "copy": "evidence/phase-1/E24/results/repair2-adoption-audit.json",
+      "sha256": "4dd788a7383fe4c907d5f103601171c58988682deabb376ad790b173abd24e2c",
+      "bytes": 12791,
+      "byteIdentical": true
+    },
+    {
+      "original": "output/phase-1/e24/repair2-frontend-build.console.txt",
+      "copy": "evidence/phase-1/E24/results/repair2-frontend-build.console.txt",
+      "sha256": "a3e2af2a6801b0d731217ece396806eb9a6b3935710a9806cc64005ecd71fa27",
+      "bytes": 2985,
+      "byteIdentical": true
+    },
+    {
+      "original": "output/phase-1/e24/repair2-frontend-build.json",
+      "copy": "evidence/phase-1/E24/results/repair2-frontend-build.json",
+      "sha256": "d980cb12dcff549ed2ed6b479e603e86c886bdea6f0247110b27813f47c384b6",
+      "bytes": 5630,
+      "byteIdentical": true
+    },
+    {
+      "original": "output/phase-1/e24/repair2-frontend-build.started.json",
+      "copy": "evidence/phase-1/E24/results/repair2-frontend-build.started.json",
+      "sha256": "bede3e9a2fce98e34a49e91fdcaa01b0ecdc5e9e294b40f2f0d7f4289b06f750",
+      "bytes": 346,
+      "byteIdentical": true
+    },
+    {
+      "original": "output/phase-1/e24/repair2-images-after.console.txt",
+      "copy": "evidence/phase-1/E24/results/repair2-images-after.console.txt",
+      "sha256": "42187f3f151de995e8d226ae68a308da56401345a416ffd81ffcd8b71b272e24",
+      "bytes": 148,
+      "byteIdentical": true
+    },
+    {
+      "original": "output/phase-1/e24/repair2-images-before.console.txt",
+      "copy": "evidence/phase-1/E24/results/repair2-images-before.console.txt",
+      "sha256": "c9070987f87cf33f0b3bdfbf79c674f617551b1df9b9f471974246de50ea775c",
+      "bytes": 148,
+      "byteIdentical": true
+    },
+    {
+      "original": "output/phase-1/e24/repair2-source-audit.json",
+      "copy": "evidence/phase-1/E24/results/repair2-source-audit.json",
+      "sha256": "19b99ecc365a80e0204fbdfa05a4ea9abc7c2ef7ccd865ece48a05ad726dcb19",
+      "bytes": 1808,
+      "byteIdentical": true
+    },
+    {
+      "original": "output/phase-1/e24/repair2-source-audit.started.json",
+      "copy": "evidence/phase-1/E24/results/repair2-source-audit.started.json",
+      "sha256": "d1808acf0d7c53b26cdc172b9c7e32ac32d650abf41452a66b80f3e60cc38c35",
+      "bytes": 63,
+      "byteIdentical": true
+    },
+    {
+      "original": "output/phase-1/e24/repair2-source-review.patch",
+      "copy": "evidence/phase-1/E24/results/repair2-source-review.patch",
+      "sha256": "fbf7e6187ce32516910cc9b0d8d72016ae9a174962a60b60f8cd934daba0f04c",
+      "bytes": 2720,
+      "byteIdentical": true
+    }
+  ],
+  "previousOutputsUnchanged": true,
+  "previousEvidenceUntouched": true,
+  "runtimeCredentialsRetained": false,
+  "rawContainerRoleLogsRetained": false
+}


