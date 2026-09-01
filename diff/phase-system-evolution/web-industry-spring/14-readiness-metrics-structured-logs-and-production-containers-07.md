## `test: preserve E24 metric inventory rejection`

diff --git a/TRACK.md b/TRACK.md
index a6b68a3..45c351b 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -4,7 +4,7 @@ Profile: `phase-1` (E01–E12 → E20 → profile E24; phase-2 deferred, E25 not
 
 Spec revision: `2ada57a71cd34fa2fae9809415c362a8bbfcdf02`
 
-E20 is a local development product: Next.js/React renders login, logout, Monitor lifecycle and Check history from request-scoped server data. Spring Security authenticates browser sessions; PostgreSQL stores salted account passwords, Monitors and Check execution intents/results. Manual requests have durable owner-scoped identities, and competing workers claim one execution owner before bounded HTTP/HTTPS I/O. Monitor and CheckRun access is scoped to the signed-in user. There is no signup, Redis, broker or production application container.
+The phase-1 product: Next.js/React renders login, logout, Monitor lifecycle and Check history from request-scoped server data. Spring Security authenticates browser sessions; PostgreSQL stores salted account passwords, Monitors and Check execution intents/results. Manual requests have durable owner-scoped identities, and competing workers claim one execution owner before bounded HTTP/HTTPS I/O. Monitor and CheckRun access is scoped to the signed-in user. There is no signup, Redis or broker. The E24 production-container candidate is present but not yet accepted; see the E24 status below.
 
 ## Pinned toolchain
 
@@ -27,7 +27,7 @@ E20 is a local development product: Next.js/React renders login, logout, Monitor
 | PostgreSQL JDBC | 42.7.11 | existing Spring Boot 3.5.16 BOM |
 | Spring Security core / config / web / crypto | 6.5.11 | existing Spring Boot 3.5.16 BOM; starter-security 3.5.16 |
 
-Versions remain fixed for this evolution. The Spring parent pins build plugins and transitive dependency management. There is no application container build yet; the only container is the isolated development/test database.
+Versions remain fixed for this evolution. The Spring parent pins build plugins and transitive dependency management. The E24 candidate adds pinned Java and Node production image builds while retaining the isolated PostgreSQL project. Its complete operational acceptance remains pending.
 
 ## Run locally
 
@@ -484,3 +484,69 @@ read and restored afterward. Evidence, the sole author invocation and the frozen
 SQL/data are under `evidence/phase-1/E20`. At this author submission, root's one
 independent final EXPLAIN and related history regression are pending. No E11
 crash, E12 outbound, browser or load gate was repeated for E20.
+
+
+## Phase-1 operations and production-container candidate (E24)
+
+E24 is incomplete. Attempt4 / repair3 passed its five affected Java tests and
+one production image build, then failed the author container gate at the metric
+family inventory assertion. The original failed attempts and frozen inputs are
+preserved; no baseline, load scenario or automatic retry was run. A new repair
+requires root review and explicit dispatch. The independent root gate and hosted
+CI execution have not run.
+
+The candidate separates API and worker `/ops/health/liveness` from
+`/ops/health/readiness`, with PostgreSQL as authority. Required meter families are
+`http.server.requests`, `check.queue.age`, `check.worker.active`, `check.claims`
+and `check.recoveries`. The HTTP contract permits only `uri`, `method`, `status`
+and `process_role` dimensions; queue/worker dimensions use only `process_role`.
+The pending issue is at the exported registry boundary: pinned framework code
+can add an HTTP `.active` family and an `error` dimension after the convention
+runs. These are source-derived findings; the failed observer did not retain its
+actual fetched inventory. The frozen family/tag criteria have not been changed.
+
+API and worker structured request/process events use generated correlation IDs.
+They do not claim distributed CheckRun tracing. The failed run's cleanup scan
+matched8 observed response IDs, scanned19 runtime secret/sentinel values with
+zero matches, and retained no raw runtime logs, bodies or credentials.
+
+`Dockerfile.api` runs the host/CI-built pinned Maven JAR. `Dockerfile.frontend`
+builds Next's standalone server and copies its static assets. Production Compose
+uses the existing `wse-industry_database` network and an explicitly supplied
+`DB_SCHEMA`. The separate `compose.e24.yaml` override enables only the controlled
+loopback fixture, sharing the worker network namespace. Normal production
+outbound restrictions stay unchanged. API4322, worker management4324,
+frontend4323 and test fixture4321 are published only on loopback.
+
+The explicit observer accepts `author`, `root` and `ci`. It refuses occupied
+ports, schemas and project resources, fingerprints its source, and uses an
+exclusive started marker to refuse repeating the same actor/candidate. Build
+and scenario commands are separate; run them only within an approved gate:
+
+```sh
+mvn -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package
+DB_SCHEMA=e24_container docker compose -p wse-industry-e24 -f compose.production.yaml build api frontend
+fnm exec --using 24.19.0 node scripts/e24-container.mjs author
+```
+
+The frozen scenario is2users/2Monitors/4seed results, one browser manual intent,
+one idle PostgreSQL stop/restore, and ten missing-UUID requests. Author attempt4
+verified authenticated owner-specific list/history, server HTML, JavaScript and
+CSS, a same-ID `SUCCEEDED/200` result, and the single outage/restore preserving
+exact authority. Its held request was explicitly released in15.899ms with no
+watchdog release. It stopped before the ten UUID requests and runtime UID/artifact
+inspection. Native143 exits were observed only during failed-gate cleanup; they
+do not substitute for a full passing container gate.
+
+Cleanup closed Chromium, removed only the owned application containers/network
+and `e24_container` schema, and preserved PostgreSQL's container, image, volume
+and public data. No forced container kill, `down -v`, cache removal or shared-data
+cleanup is part of this observer. Safe evidence is under
+`evidence/phase-1/E24/container/` and `evidence/phase-1/E24/repair3/`; the latter
+contains the complete cumulative native invocation snapshot and source diagnosis.
+
+CI has separate unit, integration, browser and container jobs. Its container job
+builds the actual images and invokes the same observer with actor `ci`; local
+results do not establish that hosted CI ran. The container gate remains outside
+the default verification runner. Phase2, E25 and omitted-role metrics remain out
+of scope.
diff --git a/evidence/phase-1/E24/container/author-ff11785dd9b6da22.json b/evidence/phase-1/E24/container/author-ff11785dd9b6da22.json
new file mode 100644
index 0000000..8498054
--- /dev/null
+++ b/evidence/phase-1/E24/container/author-ff11785dd9b6da22.json
@@ -0,0 +1,1519 @@
+{
+  "actor": "author",
+  "thread": "E24",
+  "profile": "phase-1",
+  "specRevision": "2ada57a71cd34fa2fae9809415c362a8bbfcdf02",
+  "start": "563b325ef871fe6d1fbfef7cf39a6581f2d0a94d",
+  "startedAt": "2026-08-28T10:43:01.107Z",
+  "result": "FAIL",
+  "completed": [
+    "production roles ready; exact seed and idle metrics",
+    "one real browser intent; active held worker; released200; four-result history and reload",
+    "one idle PostgreSQL stop/restore;503 unsafe rejection; same authority and processes"
+  ],
+  "invocations": {
+    "fullContainerScenario": 1,
+    "browser": 1,
+    "acceptedManualIntent": 1,
+    "rejectedOutageIntent": 1,
+    "postgresStop": 1,
+    "postgresRestore": 1,
+    "missingUuidRequests": 0,
+    "baseline": 0,
+    "maven": 0,
+    "imageBuild": 0,
+    "load": 0,
+    "parameterSweep": 0,
+    "automaticRetry": 0
+  },
+  "nativeCommands": [
+    {
+      "command": [
+        "docker",
+        "ps",
+        "-aq",
+        "--filter",
+        "label=com.docker.compose.project=wse-industry-e24"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 28.09079200000002,
+      "stdoutBytes": 0,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "network",
+        "ls",
+        "-q",
+        "--filter",
+        "label=com.docker.compose.project=wse-industry-e24"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 20.044374999999974,
+      "stdoutBytes": 0,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "network",
+        "ls",
+        "--format",
+        "{{.Name}}"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 18.52533299999999,
+      "stdoutBytes": 64,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "volume",
+        "ls",
+        "-q",
+        "--filter",
+        "label=com.docker.compose.project=wse-industry-e24"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 19.437250000000006,
+      "stdoutBytes": 0,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "image",
+        "inspect",
+        "wse-industry-e24-api:local",
+        "wse-industry-e24-frontend:local"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 59.99212499999999,
+      "stdoutBytes": 6194,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "compose",
+        "-p",
+        "wse-industry",
+        "-f",
+        "compose.yaml",
+        "ps",
+        "--all",
+        "--quiet",
+        "postgres"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 222.97787500000004,
+      "stdoutBytes": 65,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 21.673416999999972,
+      "stdoutBytes": 12834,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "compose",
+        "-p",
+        "wse-industry-e24",
+        "-f",
+        "compose.production.yaml",
+        "-f",
+        "compose.e24.yaml",
+        "up",
+        "--detach",
+        "--no-build",
+        "api",
+        "worker",
+        "frontend",
+        "fixture"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 1974.3337090000005,
+      "stdoutBytes": 0,
+      "stderrBytes": 866,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "ps",
+        "-aq",
+        "--filter",
+        "label=com.docker.compose.project=wse-industry-e24"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 69.95479199999954,
+      "stdoutBytes": 52,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "9768c82852be"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 35.562833999999384,
+      "stdoutBytes": 8813,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "c3f7d5172190"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 30.420415999999932,
+      "stdoutBytes": 9158,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "c96c7bd75d25"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 47.92158300000028,
+      "stdoutBytes": 10610,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "da0f1d5e20d4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 56.327000000000226,
+      "stdoutBytes": 11267,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "network",
+        "ls",
+        "-q",
+        "--filter",
+        "label=com.docker.compose.project=wse-industry-e24"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 43.73887500000001,
+      "stdoutBytes": 13,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "c96c7bd75d25dc727a8f0a42d726fe1f39eccb82d2ec2fa7f0db75f46f660811"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 18.92475000000195,
+      "stdoutBytes": 10610,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "da0f1d5e20d4cf99007d87652b5101ddc40f04d833d857758fd38dc575b5ef51"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 19.372874999997293,
+      "stdoutBytes": 11267,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "c3f7d51721900e97f5a1f3a32f67c29ab3c0697a604ba1216aa3c7fe28856409"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 17.239332999997714,
+      "stdoutBytes": 9158,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "compose",
+        "-p",
+        "wse-industry",
+        "-f",
+        "compose.yaml",
+        "stop",
+        "--timeout",
+        "5",
+        "postgres"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 279.76329199999964,
+      "stdoutBytes": 0,
+      "stderrBytes": 89,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "c96c7bd75d25dc727a8f0a42d726fe1f39eccb82d2ec2fa7f0db75f46f660811"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 27.825417000000016,
+      "stdoutBytes": 10610,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "da0f1d5e20d4cf99007d87652b5101ddc40f04d833d857758fd38dc575b5ef51"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 27.377167000002373,
+      "stdoutBytes": 11267,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "c3f7d51721900e97f5a1f3a32f67c29ab3c0697a604ba1216aa3c7fe28856409"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 21.49866699999984,
+      "stdoutBytes": 9158,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "compose",
+        "-p",
+        "wse-industry",
+        "-f",
+        "compose.yaml",
+        "start",
+        "postgres"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 184.3059170000015,
+      "stdoutBytes": 0,
+      "stderrBytes": 89,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 26.94158299999981,
+      "stdoutBytes": 12794,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 24.543750000000728,
+      "stdoutBytes": 12794,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 23.880541999998968,
+      "stdoutBytes": 12794,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 27.70274999999674,
+      "stdoutBytes": 12794,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 25.868333999998868,
+      "stdoutBytes": 12794,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 27.53491600000052,
+      "stdoutBytes": 12794,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 22.172790999997233,
+      "stdoutBytes": 12794,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 24.48833300000115,
+      "stdoutBytes": 12794,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 24.884792000000743,
+      "stdoutBytes": 12880,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 22.968709000000672,
+      "stdoutBytes": 12793,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 19.232959000000847,
+      "stdoutBytes": 12793,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "c96c7bd75d25dc727a8f0a42d726fe1f39eccb82d2ec2fa7f0db75f46f660811"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 17.49212500000067,
+      "stdoutBytes": 10610,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "da0f1d5e20d4cf99007d87652b5101ddc40f04d833d857758fd38dc575b5ef51"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 18.207374999998137,
+      "stdoutBytes": 11267,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "c3f7d51721900e97f5a1f3a32f67c29ab3c0697a604ba1216aa3c7fe28856409"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 19.794041999997717,
+      "stdoutBytes": 9158,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "ps",
+        "-aq",
+        "--filter",
+        "label=com.docker.compose.project=wse-industry-e24"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 30.52979200000118,
+      "stdoutBytes": 52,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "9768c82852be"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 17.85216600000058,
+      "stdoutBytes": 8813,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "c3f7d5172190"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 20.865499999999884,
+      "stdoutBytes": 9158,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "c96c7bd75d25"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 20.33716600000116,
+      "stdoutBytes": 10610,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "da0f1d5e20d4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 22.575000000000728,
+      "stdoutBytes": 11267,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "network",
+        "ls",
+        "-q",
+        "--filter",
+        "label=com.docker.compose.project=wse-industry-e24"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 17.69395799999984,
+      "stdoutBytes": 13,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "c3f7d51721900e97f5a1f3a32f67c29ab3c0697a604ba1216aa3c7fe28856409"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 19.02666699999827,
+      "stdoutBytes": 9158,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "c3f7d51721900e97f5a1f3a32f67c29ab3c0697a604ba1216aa3c7fe28856409"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 19.365249999998923,
+      "stdoutBytes": 9158,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "kill",
+        "--signal=SIGTERM",
+        "c3f7d51721900e97f5a1f3a32f67c29ab3c0697a604ba1216aa3c7fe28856409"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 29.46216600000116,
+      "stdoutBytes": 65,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "wait",
+        "c3f7d51721900e97f5a1f3a32f67c29ab3c0697a604ba1216aa3c7fe28856409"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 189.61920800000007,
+      "stdoutBytes": 4,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "da0f1d5e20d4cf99007d87652b5101ddc40f04d833d857758fd38dc575b5ef51"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 31.175375000002532,
+      "stdoutBytes": 11267,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "da0f1d5e20d4cf99007d87652b5101ddc40f04d833d857758fd38dc575b5ef51"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 25.64191700000083,
+      "stdoutBytes": 11267,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "kill",
+        "--signal=SIGTERM",
+        "da0f1d5e20d4cf99007d87652b5101ddc40f04d833d857758fd38dc575b5ef51"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 33.654999999998836,
+      "stdoutBytes": 65,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "wait",
+        "da0f1d5e20d4cf99007d87652b5101ddc40f04d833d857758fd38dc575b5ef51"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 464.3012499999968,
+      "stdoutBytes": 4,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "c96c7bd75d25dc727a8f0a42d726fe1f39eccb82d2ec2fa7f0db75f46f660811"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 27.747040999998717,
+      "stdoutBytes": 10610,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "c96c7bd75d25dc727a8f0a42d726fe1f39eccb82d2ec2fa7f0db75f46f660811"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 19.72345800000039,
+      "stdoutBytes": 10610,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "kill",
+        "--signal=SIGTERM",
+        "c96c7bd75d25dc727a8f0a42d726fe1f39eccb82d2ec2fa7f0db75f46f660811"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 27.51941699999952,
+      "stdoutBytes": 65,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "wait",
+        "c96c7bd75d25dc727a8f0a42d726fe1f39eccb82d2ec2fa7f0db75f46f660811"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 331.8032920000005,
+      "stdoutBytes": 4,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "9768c82852be80cf777281d3e0f940bd31eb848fb73b373cc2feb94c1556217f"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 20.93199999999706,
+      "stdoutBytes": 8813,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "9768c82852be80cf777281d3e0f940bd31eb848fb73b373cc2feb94c1556217f"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 21.50866699999824,
+      "stdoutBytes": 8813,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "kill",
+        "--signal=SIGTERM",
+        "9768c82852be80cf777281d3e0f940bd31eb848fb73b373cc2feb94c1556217f"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 25.994833999997354,
+      "stdoutBytes": 65,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "wait",
+        "9768c82852be80cf777281d3e0f940bd31eb848fb73b373cc2feb94c1556217f"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 46.74712499999805,
+      "stdoutBytes": 4,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "logs",
+        "c96c7bd75d25dc727a8f0a42d726fe1f39eccb82d2ec2fa7f0db75f46f660811"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 24.649250000002212,
+      "stdoutBytes": 39893,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "logs",
+        "da0f1d5e20d4cf99007d87652b5101ddc40f04d833d857758fd38dc575b5ef51"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 20.57854099999895,
+      "stdoutBytes": 31600,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "logs",
+        "c3f7d51721900e97f5a1f3a32f67c29ab3c0697a604ba1216aa3c7fe28856409"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 18.47070800000074,
+      "stdoutBytes": 147,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "logs",
+        "9768c82852be80cf777281d3e0f940bd31eb848fb73b373cc2feb94c1556217f"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 19.00695800000176,
+      "stdoutBytes": 0,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "9768c82852be80cf777281d3e0f940bd31eb848fb73b373cc2feb94c1556217f"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 19.017875000001368,
+      "stdoutBytes": 8820,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "rm",
+        "9768c82852be80cf777281d3e0f940bd31eb848fb73b373cc2feb94c1556217f"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 29.598666000001685,
+      "stdoutBytes": 65,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "c3f7d51721900e97f5a1f3a32f67c29ab3c0697a604ba1216aa3c7fe28856409"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 26.636332999998558,
+      "stdoutBytes": 8771,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "rm",
+        "c3f7d51721900e97f5a1f3a32f67c29ab3c0697a604ba1216aa3c7fe28856409"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 30.52508300000045,
+      "stdoutBytes": 65,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "da0f1d5e20d4cf99007d87652b5101ddc40f04d833d857758fd38dc575b5ef51"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 20.772624999997788,
+      "stdoutBytes": 10561,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "rm",
+        "da0f1d5e20d4cf99007d87652b5101ddc40f04d833d857758fd38dc575b5ef51"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 27.30870800000048,
+      "stdoutBytes": 65,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "c96c7bd75d25dc727a8f0a42d726fe1f39eccb82d2ec2fa7f0db75f46f660811"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 25.26495900000009,
+      "stdoutBytes": 10087,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "rm",
+        "c96c7bd75d25dc727a8f0a42d726fe1f39eccb82d2ec2fa7f0db75f46f660811"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 28.459374999998545,
+      "stdoutBytes": 65,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "network",
+        "inspect",
+        "wse-industry-e24_default"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 19.369708999998693,
+      "stdoutBytes": 1093,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "network",
+        "rm",
+        "2b37a7e8f45b48f8f0c76323de79c34eb579fafb91301e16cefdb28125cb318d"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 229.7559160000019,
+      "stdoutBytes": 65,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 20.241750000001048,
+      "stdoutBytes": 12794,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    }
+  ],
+  "exits": {
+    "frontend": {
+      "purpose": "cleanup after incomplete/failed gate",
+      "signal": "SIGTERM",
+      "signalCommandExit": 0,
+      "exitCode": 143,
+      "elapsedMs": 189.66187499999796,
+      "awaited": true,
+      "observerTimedOut": false,
+      "forcedContainerKill": false
+    },
+    "worker": {
+      "purpose": "cleanup after incomplete/failed gate",
+      "signal": "SIGTERM",
+      "signalCommandExit": 0,
+      "exitCode": 143,
+      "elapsedMs": 464.3378339999981,
+      "awaited": true,
+      "observerTimedOut": false,
+      "forcedContainerKill": false
+    },
+    "api": {
+      "purpose": "cleanup after incomplete/failed gate",
+      "signal": "SIGTERM",
+      "signalCommandExit": 0,
+      "exitCode": 143,
+      "elapsedMs": 331.82558399999834,
+      "awaited": true,
+      "observerTimedOut": false,
+      "forcedContainerKill": false
+    },
+    "fixture": {
+      "purpose": "cleanup after incomplete/failed gate",
+      "signal": "SIGTERM",
+      "signalCommandExit": 0,
+      "exitCode": 143,
+      "elapsedMs": 46.77612499999668,
+      "awaited": true,
+      "observerTimedOut": false,
+      "forcedContainerKill": false
+    }
+  },
+  "cleanupFailures": [],
+  "privateArtifacts": {
+    "rawLogs": false,
+    "httpBodies": false,
+    "credentials": false,
+    "browserTrace": false,
+    "screenshot": false,
+    "video": false,
+    "har": false,
+    "storageState": false
+  },
+  "head": "2dbe240409db95ac0435f8a4d4270189f60b7d82",
+  "sourceFingerprint": "ff11785dd9b6da221b6b863102b24e38cf028f3b71790d5f8c787642f5603610",
+  "frozenHashes": {
+    "evidence/phase-1/E24/fixture.json": "47f86919e5f251c3f8ee394727235b906fc03a5eac67e36cc3e273486a21a8fd",
+    "scripts/e24-seed.mjs": "16d0f9286acfe76a5dfb79819d0ee49e975098ecb710464327d41c0f54ec4025",
+    "scripts/e24-baseline.mjs": "37bdf00c299706b5e5ef2d7ff11311605a9529fa0ee7abcfa9ef78dd62ce49c4",
+    "evidence/phase-1/E24/fixtures.md": "5574f4d4d6c8e8cbbd17273ba8385e3a1ac43c2193d34d40b0dca4da11090a1c"
+  },
+  "preflight": {
+    "vacantPorts": [
+      4322,
+      4324,
+      4323,
+      4321
+    ],
+    "existingApplicationResources": 0,
+    "postgres": {
+      "id": "7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4",
+      "image": "sha256:051f7b7b3abdd564d5d1bd1e8c4b9c1b6e77087d1dd22020ede611c096a272e0",
+      "configuredImage": "postgres:17.11-bookworm@sha256:051f7b7b3abdd564d5d1bd1e8c4b9c1b6e77087d1dd22020ede611c096a272e0",
+      "mounts": [
+        {
+          "type": "volume",
+          "name": "wse-industry_postgres-data",
+          "destination": "/var/lib/postgresql/data"
+        }
+      ]
+    },
+    "publicTableCount": 0,
+    "frozenInputsUnchanged": true
+  },
+  "seed": {
+    "users": 2,
+    "monitors": 2,
+    "checks": 4,
+    "queued": 0,
+    "running": 0,
+    "terminal": 4
+  },
+  "startup": {
+    "liveness": {
+      "api": 200,
+      "worker": 200
+    },
+    "readiness": {
+      "api": 200,
+      "worker": 200
+    },
+    "worker": {
+      "queueAge": 0,
+      "active": 0,
+      "claims": 0,
+      "recoveries": 0
+    }
+  },
+  "browser": {
+    "chromium": "151.0.7922.34",
+    "contexts": 2,
+    "ownerArticleCounts": [
+      1,
+      1
+    ],
+    "initialHistoryRows": [
+      3,
+      1
+    ],
+    "foreignDetailStatuses": [
+      404,
+      404
+    ],
+    "authenticatedServerHtml": true,
+    "serverHistoryBeforeHydration": true,
+    "javascriptStatus": 200,
+    "cssStatus": 200,
+    "pageErrors": 0,
+    "hydrationErrors": 0,
+    "closedBeforeOutage": true
+  },
+  "manual": {
+    "acceptedStatus": 202,
+    "acceptedState": "QUEUED",
+    "acceptedOutcomeFieldsNull": true,
+    "held": {
+      "held": 1,
+      "outboundCalls": 1,
+      "active": 1,
+      "localObservationAndReleaseMs": 5.355459000002156,
+      "dockerOrSqlBeforeRelease": false
+    },
+    "fixture": {
+      "outboundCalls": 1,
+      "holdRequests": 1,
+      "held": 0,
+      "releases": 1,
+      "watchdogReleases": 0,
+      "lastHeldMs": 15.899124999999913
+    },
+    "sameTerminalId": true,
+    "terminalState": "SUCCEEDED",
+    "httpStatus": 200,
+    "historyRows": 4,
+    "reloadRetainsResult": true
+  },
+  "outage": {
+    "liveness": {
+      "api": 200,
+      "worker": 200
+    },
+    "readiness": {
+      "api": 503,
+      "worker": 503
+    },
+    "rejectedStatus": 503,
+    "acceptedId": false,
+    "active": 0,
+    "claims": 1,
+    "recoveries": 0,
+    "outboundCalls": 1,
+    "sameLiveProcesses": true
+  },
+  "restore": {
+    "readiness": {
+      "api": 200,
+      "worker": 200
+    },
+    "samePostgresContainerImageVolumes": true,
+    "publicAuthorityUnchanged": true,
+    "exactPrivateRowsUnchanged": true,
+    "counts": {
+      "users": 2,
+      "monitors": 2,
+      "checks": 5,
+      "queued": 0,
+      "running": 0,
+      "terminal": 5
+    },
+    "rejectedIntentPersisted": false,
+    "sameLiveProcesses": true
+  },
+  "failure": {
+    "phase": "bounded real metrics and ten missing UUID requests",
+    "reason": "Only the applicable API and worker metric families are exposed",
+    "type": "Error",
+    "privateExceptionOutputRetained": false
+  },
+  "logs": {
+    "roles": {
+      "api": {
+        "lines": 89,
+        "structuredLines": 89,
+        "observationEvents": 41
+      },
+      "worker": {
+        "lines": 45,
+        "structuredLines": 45,
+        "observationEvents": 4
+      },
+      "frontend": {
+        "lines": 5,
+        "structuredLines": 0,
+        "observationEvents": 0
+      },
+      "fixture": {
+        "lines": 0,
+        "structuredLines": 0,
+        "observationEvents": 0
+      }
+    },
+    "runtimeSentinelValuesScanned": 19,
+    "runtimeSentinelMatches": 0,
+    "unboundedInputMatches": 0,
+    "responseCorrelations": 8,
+    "matchedResponseCorrelations": 8,
+    "generatedResponseRequestIds": [
+      "dcacefb8-eec4-4f86-9ed7-b453bc6f5c1a",
+      "e6fcf5f9-d9ff-4f61-9294-f3d32f7ba0b2",
+      "d7dde90a-6c7c-4a3b-a504-80bf40bc567e",
+      "d81a6a6e-3c19-4657-9371-a6ad76b73936",
+      "9061cf72-6a02-45d2-898c-e879a8e2d8e7",
+      "d209cc21-39a5-4b12-a233-0fb9121cafcc",
+      "96038228-8e3d-4440-b344-dabf2f5b35be",
+      "3ef84129-490d-4a18-956c-da511c7a3060"
+    ],
+    "stableApiProcess": true,
+    "stableWorkerProcess": true,
+    "distinctRoleProcesses": true,
+    "apiProcessId": "da50404d-7547-436d-b2fd-d509cd22554d",
+    "workerProcessId": "625401ef-cf28-47b5-86a4-7ac51732b873",
+    "committedClaimEvents": 1,
+    "unavailableAuthorityEvents": 3,
+    "recoveryEvents": 0,
+    "crossProcessCheckRunTraceClaimed": false,
+    "rawLogsRetained": false
+  },
+  "cleanup": {
+    "browserClosed": true,
+    "applicationContainersRemoved": true,
+    "applicationNetworkRemoved": true,
+    "ownedSchemaDropped": true,
+    "postgresPreserved": true,
+    "postgresStopAttempted": true,
+    "postgresRestoreAttempted": true,
+    "postgresRestored": true,
+    "forcedContainerKill": false,
+    "sharedOrPublicDataRemoved": false
+  },
+  "elapsedSeconds": 24.582,
+  "hostedCiExecutionClaimed": false
+}
diff --git a/evidence/phase-1/E24/container/author-ff11785dd9b6da22.started.json b/evidence/phase-1/E24/container/author-ff11785dd9b6da22.started.json
new file mode 100644
index 0000000..b16aec3
--- /dev/null
+++ b/evidence/phase-1/E24/container/author-ff11785dd9b6da22.started.json
@@ -0,0 +1 @@
+{"actor":"author","head":"2dbe240409db95ac0435f8a4d4270189f60b7d82","fingerprint":"ff11785dd9b6da221b6b863102b24e38cf028f3b71790d5f8c787642f5603610","startedAt":"2026-08-28T10:43:01.107Z"}
diff --git a/evidence/phase-1/E24/container/invocations.jsonl b/evidence/phase-1/E24/container/invocations.jsonl
new file mode 100644
index 0000000..773e611
--- /dev/null
+++ b/evidence/phase-1/E24/container/invocations.jsonl
@@ -0,0 +1 @@
+{"actor":"author","head":"2dbe240409db95ac0435f8a4d4270189f60b7d82","fingerprint":"ff11785dd9b6da221b6b863102b24e38cf028f3b71790d5f8c787642f5603610","startedAt":"2026-08-28T10:43:01.107Z","command":"node scripts/e24-container.mjs author"}
diff --git a/evidence/phase-1/E24/repair3/cumulative-invocations.jsonl b/evidence/phase-1/E24/repair3/cumulative-invocations.jsonl
new file mode 100644
index 0000000..9402a76
--- /dev/null
+++ b/evidence/phase-1/E24/repair3/cumulative-invocations.jsonl
@@ -0,0 +1,7 @@
+{"command":"node scripts/e24-baseline.mjs","startedAt":"2026-08-28T08:51:05.183Z","elapsedSeconds":7.477,"exitCode":0}
+{"command":"mvn -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package","startedAt":"2026-08-28T09:11:45.983Z","elapsedSeconds":1.724,"exitCode":1,"signal":null}
+{"command":"mvn -U -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package","startedAt":"2026-08-28T09:22:21.048583+00:00","attempt":2,"repair":1,"actor":"author","permission":"require_escalated network/cache authorization","forcedDescriptorRefreshes":1,"elapsedSeconds":4.051,"exitCode":1,"signal":null}
+{"command":"mvn -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package","startedAt":"2026-08-28T09:31:22.232667+00:00","attempt":3,"repair":2,"actor":"author","permission":"require_escalated network/cache/PostgreSQL authorization","forcedDescriptorRefreshes":0,"elapsedSeconds":6.999,"exitCode":1,"signal":null}
+{"command":"mvn -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package","startedAt":"2026-08-28T10:35:41.429288+00:00","attempt":4,"repair":3,"actor":"author","permission":"require_escalated Java/Maven/cache/network/PostgreSQL authorization","forcedDescriptorRefreshes":0,"elapsedSeconds":8.772,"exitCode":0,"signal":null,"nativeBundle":"evidence/phase-1/E24/repair3/repair3-affected-java-native.json"}
+{"command":"DB_SCHEMA=e24_container docker compose -p wse-industry-e24 -f compose.production.yaml build api frontend","startedAt":"2026-08-28T10:39:20.133345+00:00","attempt":4,"repair":3,"actor":"author","permission":"require_escalated pinned production image build/cache/network authorization","elapsedSeconds":48.361,"exitCode":0,"signal":null}
+{"command":"fnm exec --using 24.19.0 node scripts/e24-container.mjs author","startedAt":"2026-08-28T10:43:00.859467+00:00","attempt":4,"repair":3,"actor":"author","permission":"require_escalated actual Docker/PostgreSQL/browser/network authorization","elapsedSeconds":24.842,"exitCode":1,"signal":null,"rawDiagnosticOutputRetained":false}
diff --git a/evidence/phase-1/E24/repair3/repair3-author-container-native.json b/evidence/phase-1/E24/repair3/repair3-author-container-native.json
new file mode 100644
index 0000000..dc8cf31
--- /dev/null
+++ b/evidence/phase-1/E24/repair3/repair3-author-container-native.json
@@ -0,0 +1,104 @@
+{
+  "thread": "E24",
+  "profile": "phase-1",
+  "attempt": 4,
+  "repair": 3,
+  "actor": "author",
+  "invocation": {
+    "command": [
+      "fnm",
+      "exec",
+      "--using",
+      "24.19.0",
+      "node",
+      "scripts/e24-container.mjs",
+      "author"
+    ],
+    "displayCommand": "fnm exec --using 24.19.0 node scripts/e24-container.mjs author",
+    "startedAt": "2026-08-28T10:43:00.859467+00:00",
+    "attempt": 4,
+    "repair": 3,
+    "actor": "author",
+    "permission": "require_escalated actual Docker/PostgreSQL/browser/network authorization",
+    "authorContainerScenarioInvocation": 1,
+    "automaticRetries": 0,
+    "observerSha256": "58872e40e522808f033b34052fca67bd031bafb4f6fe26ced81b225978a1bfb8",
+    "authorizationSha256": "a24c7f7d01b50fb5d722ccf96c3734b5cd4784f9d3d265904e65ef7a5df7a811"
+  },
+  "nativeResult": {
+    "exitCode": 1,
+    "elapsedSeconds": 24.842,
+    "startedAt": "2026-08-28T10:43:00.859467+00:00",
+    "endedAt": "2026-08-28T10:43:25.701544+00:00",
+    "safeObserverSummary": {
+      "actor": "author",
+      "result": "FAIL",
+      "phase": "bounded real metrics and ten missing UUID requests",
+      "evidence": "evidence/phase-1/E24/container/author-ff11785dd9b6da22.json",
+      "elapsedSeconds": 24.582,
+      "cleanupFailures": 0
+    },
+    "stdoutBytes": 213,
+    "stderrBytes": 0,
+    "rawDiagnosticOutputRetained": false
+  },
+  "files": [
+    {
+      "source": "output/phase-1/e24/repair3-author-container.started.json",
+      "sha256": "b967c5c0bbadd9eae4e7d75056a0e4a3eb94cc9b121c42c5e064a5024b4b6dca",
+      "bytes": 663,
+      "raw": "{\n  \"command\": [\n    \"fnm\",\n    \"exec\",\n    \"--using\",\n    \"24.19.0\",\n    \"node\",\n    \"scripts/e24-container.mjs\",\n    \"author\"\n  ],\n  \"displayCommand\": \"fnm exec --using 24.19.0 node scripts/e24-container.mjs author\",\n  \"startedAt\": \"2026-08-28T10:43:00.859467+00:00\",\n  \"attempt\": 4,\n  \"repair\": 3,\n  \"actor\": \"author\",\n  \"permission\": \"require_escalated actual Docker/PostgreSQL/browser/network authorization\",\n  \"authorContainerScenarioInvocation\": 1,\n  \"automaticRetries\": 0,\n  \"observerSha256\": \"58872e40e522808f033b34052fca67bd031bafb4f6fe26ced81b225978a1bfb8\",\n  \"authorizationSha256\": \"a24c7f7d01b50fb5d722ccf96c3734b5cd4784f9d3d265904e65ef7a5df7a811\"\n}\n"
+    },
+    {
+      "source": "output/phase-1/e24/repair3-author-container.result.json",
+      "sha256": "8f549602be6baabbd43811583f78a716a4340c2ddfab5a23d4f92334fd7ae19e",
+      "bytes": 508,
+      "raw": "{\n  \"exitCode\": 1,\n  \"elapsedSeconds\": 24.842,\n  \"startedAt\": \"2026-08-28T10:43:00.859467+00:00\",\n  \"endedAt\": \"2026-08-28T10:43:25.701544+00:00\",\n  \"safeObserverSummary\": {\n    \"actor\": \"author\",\n    \"result\": \"FAIL\",\n    \"phase\": \"bounded real metrics and ten missing UUID requests\",\n    \"evidence\": \"evidence/phase-1/E24/container/author-ff11785dd9b6da22.json\",\n    \"elapsedSeconds\": 24.582,\n    \"cleanupFailures\": 0\n  },\n  \"stdoutBytes\": 213,\n  \"stderrBytes\": 0,\n  \"rawDiagnosticOutputRetained\": false\n}\n"
+    },
+    {
+      "source": "output/phase-1/e24/invocations.jsonl",
+      "sha256": "7b6d4221eaf0d1cabb7480e8e802d0a41474c9675c507ce58f3a717ba0f9f58c",
+      "bytes": 2207,
+      "raw": "{\"command\":\"node scripts/e24-baseline.mjs\",\"startedAt\":\"2026-08-28T08:51:05.183Z\",\"elapsedSeconds\":7.477,\"exitCode\":0}\n{\"command\":\"mvn -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package\",\"startedAt\":\"2026-08-28T09:11:45.983Z\",\"elapsedSeconds\":1.724,\"exitCode\":1,\"signal\":null}\n{\"command\":\"mvn -U -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package\",\"startedAt\":\"2026-08-28T09:22:21.048583+00:00\",\"attempt\":2,\"repair\":1,\"actor\":\"author\",\"permission\":\"require_escalated network/cache authorization\",\"forcedDescriptorRefreshes\":1,\"elapsedSeconds\":4.051,\"exitCode\":1,\"signal\":null}\n{\"command\":\"mvn -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package\",\"startedAt\":\"2026-08-28T09:31:22.232667+00:00\",\"attempt\":3,\"repair\":2,\"actor\":\"author\",\"permission\":\"require_escalated network/cache/PostgreSQL authorization\",\"forcedDescriptorRefreshes\":0,\"elapsedSeconds\":6.999,\"exitCode\":1,\"signal\":null}\n{\"command\":\"mvn -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package\",\"startedAt\":\"2026-08-28T10:35:41.429288+00:00\",\"attempt\":4,\"repair\":3,\"actor\":\"author\",\"permission\":\"require_escalated Java/Maven/cache/network/PostgreSQL authorization\",\"forcedDescriptorRefreshes\":0,\"elapsedSeconds\":8.772,\"exitCode\":0,\"signal\":null,\"nativeBundle\":\"evidence/phase-1/E24/repair3/repair3-affected-java-native.json\"}\n{\"command\":\"DB_SCHEMA=e24_container docker compose -p wse-industry-e24 -f compose.production.yaml build api frontend\",\"startedAt\":\"2026-08-28T10:39:20.133345+00:00\",\"attempt\":4,\"repair\":3,\"actor\":\"author\",\"permission\":\"require_escalated pinned production image build/cache/network authorization\",\"elapsedSeconds\":48.361,\"exitCode\":0,\"signal\":null}\n{\"command\":\"fnm exec --using 24.19.0 node scripts/e24-container.mjs author\",\"startedAt\":\"2026-08-28T10:43:00.859467+00:00\",\"attempt\":4,\"repair\":3,\"actor\":\"author\",\"permission\":\"require_escalated actual Docker/PostgreSQL/browser/network authorization\",\"elapsedSeconds\":24.842,\"exitCode\":1,\"signal\":null,\"rawDiagnosticOutputRetained\":false}\n"
+    },
+    {
+      "source": "evidence/phase-1/E24/container/author-ff11785dd9b6da22.json",
+      "sha256": "927dc651b22ea41ad8c8d5f4858671275ca6c1beda15866aaaf080b27938753f",
+      "bytes": 37141,
+      "raw": "{\n  \"actor\": \"author\",\n  \"thread\": \"E24\",\n  \"profile\": \"phase-1\",\n  \"specRevision\": \"2ada57a71cd34fa2fae9809415c362a8bbfcdf02\",\n  \"start\": \"563b325ef871fe6d1fbfef7cf39a6581f2d0a94d\",\n  \"startedAt\": \"2026-08-28T10:43:01.107Z\",\n  \"result\": \"FAIL\",\n  \"completed\": [\n    \"production roles ready; exact seed and idle metrics\",\n    \"one real browser intent; active held worker; released200; four-result history and reload\",\n    \"one idle PostgreSQL stop/restore;503 unsafe rejection; same authority and processes\"\n  ],\n  \"invocations\": {\n    \"fullContainerScenario\": 1,\n    \"browser\": 1,\n    \"acceptedManualIntent\": 1,\n    \"rejectedOutageIntent\": 1,\n    \"postgresStop\": 1,\n    \"postgresRestore\": 1,\n    \"missingUuidRequests\": 0,\n    \"baseline\": 0,\n    \"maven\": 0,\n    \"imageBuild\": 0,\n    \"load\": 0,\n    \"parameterSweep\": 0,\n    \"automaticRetry\": 0\n  },\n  \"nativeCommands\": [\n    {\n      \"command\": [\n        \"docker\",\n        \"ps\",\n        \"-aq\",\n        \"--filter\",\n        \"label=com.docker.compose.project=wse-industry-e24\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 28.09079200000002,\n      \"stdoutBytes\": 0,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"network\",\n        \"ls\",\n        \"-q\",\n        \"--filter\",\n        \"label=com.docker.compose.project=wse-industry-e24\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 20.044374999999974,\n      \"stdoutBytes\": 0,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"network\",\n        \"ls\",\n        \"--format\",\n        \"{{.Name}}\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 18.52533299999999,\n      \"stdoutBytes\": 64,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"volume\",\n        \"ls\",\n        \"-q\",\n        \"--filter\",\n        \"label=com.docker.compose.project=wse-industry-e24\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 19.437250000000006,\n      \"stdoutBytes\": 0,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"image\",\n        \"inspect\",\n        \"wse-industry-e24-api:local\",\n        \"wse-industry-e24-frontend:local\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 59.99212499999999,\n      \"stdoutBytes\": 6194,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"compose\",\n        \"-p\",\n        \"wse-industry\",\n        \"-f\",\n        \"compose.yaml\",\n        \"ps\",\n        \"--all\",\n        \"--quiet\",\n        \"postgres\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 222.97787500000004,\n      \"stdoutBytes\": 65,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 21.673416999999972,\n      \"stdoutBytes\": 12834,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"compose\",\n        \"-p\",\n        \"wse-industry-e24\",\n        \"-f\",\n        \"compose.production.yaml\",\n        \"-f\",\n        \"compose.e24.yaml\",\n        \"up\",\n        \"--detach\",\n        \"--no-build\",\n        \"api\",\n        \"worker\",\n        \"frontend\",\n        \"fixture\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 1974.3337090000005,\n      \"stdoutBytes\": 0,\n      \"stderrBytes\": 866,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"ps\",\n        \"-aq\",\n        \"--filter\",\n        \"label=com.docker.compose.project=wse-industry-e24\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 69.95479199999954,\n      \"stdoutBytes\": 52,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"9768c82852be\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 35.562833999999384,\n      \"stdoutBytes\": 8813,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"c3f7d5172190\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 30.420415999999932,\n      \"stdoutBytes\": 9158,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"c96c7bd75d25\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 47.92158300000028,\n      \"stdoutBytes\": 10610,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"da0f1d5e20d4\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 56.327000000000226,\n      \"stdoutBytes\": 11267,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"network\",\n        \"ls\",\n        \"-q\",\n        \"--filter\",\n        \"label=com.docker.compose.project=wse-industry-e24\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 43.73887500000001,\n      \"stdoutBytes\": 13,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"c96c7bd75d25dc727a8f0a42d726fe1f39eccb82d2ec2fa7f0db75f46f660811\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 18.92475000000195,\n      \"stdoutBytes\": 10610,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"da0f1d5e20d4cf99007d87652b5101ddc40f04d833d857758fd38dc575b5ef51\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 19.372874999997293,\n      \"stdoutBytes\": 11267,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"c3f7d51721900e97f5a1f3a32f67c29ab3c0697a604ba1216aa3c7fe28856409\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 17.239332999997714,\n      \"stdoutBytes\": 9158,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"compose\",\n        \"-p\",\n        \"wse-industry\",\n        \"-f\",\n        \"compose.yaml\",\n        \"stop\",\n        \"--timeout\",\n        \"5\",\n        \"postgres\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 279.76329199999964,\n      \"stdoutBytes\": 0,\n      \"stderrBytes\": 89,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"c96c7bd75d25dc727a8f0a42d726fe1f39eccb82d2ec2fa7f0db75f46f660811\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 27.825417000000016,\n      \"stdoutBytes\": 10610,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"da0f1d5e20d4cf99007d87652b5101ddc40f04d833d857758fd38dc575b5ef51\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 27.377167000002373,\n      \"stdoutBytes\": 11267,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"c3f7d51721900e97f5a1f3a32f67c29ab3c0697a604ba1216aa3c7fe28856409\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 21.49866699999984,\n      \"stdoutBytes\": 9158,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"compose\",\n        \"-p\",\n        \"wse-industry\",\n        \"-f\",\n        \"compose.yaml\",\n        \"start\",\n        \"postgres\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 184.3059170000015,\n      \"stdoutBytes\": 0,\n      \"stderrBytes\": 89,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 26.94158299999981,\n      \"stdoutBytes\": 12794,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 24.543750000000728,\n      \"stdoutBytes\": 12794,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 23.880541999998968,\n      \"stdoutBytes\": 12794,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 27.70274999999674,\n      \"stdoutBytes\": 12794,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 25.868333999998868,\n      \"stdoutBytes\": 12794,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 27.53491600000052,\n      \"stdoutBytes\": 12794,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 22.172790999997233,\n      \"stdoutBytes\": 12794,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 24.48833300000115,\n      \"stdoutBytes\": 12794,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 24.884792000000743,\n      \"stdoutBytes\": 12880,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 22.968709000000672,\n      \"stdoutBytes\": 12793,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 19.232959000000847,\n      \"stdoutBytes\": 12793,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"c96c7bd75d25dc727a8f0a42d726fe1f39eccb82d2ec2fa7f0db75f46f660811\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 17.49212500000067,\n      \"stdoutBytes\": 10610,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"da0f1d5e20d4cf99007d87652b5101ddc40f04d833d857758fd38dc575b5ef51\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 18.207374999998137,\n      \"stdoutBytes\": 11267,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"c3f7d51721900e97f5a1f3a32f67c29ab3c0697a604ba1216aa3c7fe28856409\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 19.794041999997717,\n      \"stdoutBytes\": 9158,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"ps\",\n        \"-aq\",\n        \"--filter\",\n        \"label=com.docker.compose.project=wse-industry-e24\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 30.52979200000118,\n      \"stdoutBytes\": 52,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"9768c82852be\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 17.85216600000058,\n      \"stdoutBytes\": 8813,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"c3f7d5172190\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 20.865499999999884,\n      \"stdoutBytes\": 9158,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"c96c7bd75d25\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 20.33716600000116,\n      \"stdoutBytes\": 10610,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"da0f1d5e20d4\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 22.575000000000728,\n      \"stdoutBytes\": 11267,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"network\",\n        \"ls\",\n        \"-q\",\n        \"--filter\",\n        \"label=com.docker.compose.project=wse-industry-e24\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 17.69395799999984,\n      \"stdoutBytes\": 13,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"c3f7d51721900e97f5a1f3a32f67c29ab3c0697a604ba1216aa3c7fe28856409\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 19.02666699999827,\n      \"stdoutBytes\": 9158,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"c3f7d51721900e97f5a1f3a32f67c29ab3c0697a604ba1216aa3c7fe28856409\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 19.365249999998923,\n      \"stdoutBytes\": 9158,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"kill\",\n        \"--signal=SIGTERM\",\n        \"c3f7d51721900e97f5a1f3a32f67c29ab3c0697a604ba1216aa3c7fe28856409\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 29.46216600000116,\n      \"stdoutBytes\": 65,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"wait\",\n        \"c3f7d51721900e97f5a1f3a32f67c29ab3c0697a604ba1216aa3c7fe28856409\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 189.61920800000007,\n      \"stdoutBytes\": 4,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"da0f1d5e20d4cf99007d87652b5101ddc40f04d833d857758fd38dc575b5ef51\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 31.175375000002532,\n      \"stdoutBytes\": 11267,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"da0f1d5e20d4cf99007d87652b5101ddc40f04d833d857758fd38dc575b5ef51\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 25.64191700000083,\n      \"stdoutBytes\": 11267,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"kill\",\n        \"--signal=SIGTERM\",\n        \"da0f1d5e20d4cf99007d87652b5101ddc40f04d833d857758fd38dc575b5ef51\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 33.654999999998836,\n      \"stdoutBytes\": 65,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"wait\",\n        \"da0f1d5e20d4cf99007d87652b5101ddc40f04d833d857758fd38dc575b5ef51\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 464.3012499999968,\n      \"stdoutBytes\": 4,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"c96c7bd75d25dc727a8f0a42d726fe1f39eccb82d2ec2fa7f0db75f46f660811\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 27.747040999998717,\n      \"stdoutBytes\": 10610,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"c96c7bd75d25dc727a8f0a42d726fe1f39eccb82d2ec2fa7f0db75f46f660811\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 19.72345800000039,\n      \"stdoutBytes\": 10610,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"kill\",\n        \"--signal=SIGTERM\",\n        \"c96c7bd75d25dc727a8f0a42d726fe1f39eccb82d2ec2fa7f0db75f46f660811\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 27.51941699999952,\n      \"stdoutBytes\": 65,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"wait\",\n        \"c96c7bd75d25dc727a8f0a42d726fe1f39eccb82d2ec2fa7f0db75f46f660811\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 331.8032920000005,\n      \"stdoutBytes\": 4,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"9768c82852be80cf777281d3e0f940bd31eb848fb73b373cc2feb94c1556217f\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 20.93199999999706,\n      \"stdoutBytes\": 8813,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"9768c82852be80cf777281d3e0f940bd31eb848fb73b373cc2feb94c1556217f\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 21.50866699999824,\n      \"stdoutBytes\": 8813,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"kill\",\n        \"--signal=SIGTERM\",\n        \"9768c82852be80cf777281d3e0f940bd31eb848fb73b373cc2feb94c1556217f\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 25.994833999997354,\n      \"stdoutBytes\": 65,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"wait\",\n        \"9768c82852be80cf777281d3e0f940bd31eb848fb73b373cc2feb94c1556217f\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 46.74712499999805,\n      \"stdoutBytes\": 4,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"logs\",\n        \"c96c7bd75d25dc727a8f0a42d726fe1f39eccb82d2ec2fa7f0db75f46f660811\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 24.649250000002212,\n      \"stdoutBytes\": 39893,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"logs\",\n        \"da0f1d5e20d4cf99007d87652b5101ddc40f04d833d857758fd38dc575b5ef51\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 20.57854099999895,\n      \"stdoutBytes\": 31600,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"logs\",\n        \"c3f7d51721900e97f5a1f3a32f67c29ab3c0697a604ba1216aa3c7fe28856409\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 18.47070800000074,\n      \"stdoutBytes\": 147,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"logs\",\n        \"9768c82852be80cf777281d3e0f940bd31eb848fb73b373cc2feb94c1556217f\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 19.00695800000176,\n      \"stdoutBytes\": 0,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"9768c82852be80cf777281d3e0f940bd31eb848fb73b373cc2feb94c1556217f\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 19.017875000001368,\n      \"stdoutBytes\": 8820,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"rm\",\n        \"9768c82852be80cf777281d3e0f940bd31eb848fb73b373cc2feb94c1556217f\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 29.598666000001685,\n      \"stdoutBytes\": 65,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"c3f7d51721900e97f5a1f3a32f67c29ab3c0697a604ba1216aa3c7fe28856409\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 26.636332999998558,\n      \"stdoutBytes\": 8771,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"rm\",\n        \"c3f7d51721900e97f5a1f3a32f67c29ab3c0697a604ba1216aa3c7fe28856409\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 30.52508300000045,\n      \"stdoutBytes\": 65,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"da0f1d5e20d4cf99007d87652b5101ddc40f04d833d857758fd38dc575b5ef51\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 20.772624999997788,\n      \"stdoutBytes\": 10561,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"rm\",\n        \"da0f1d5e20d4cf99007d87652b5101ddc40f04d833d857758fd38dc575b5ef51\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 27.30870800000048,\n      \"stdoutBytes\": 65,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"c96c7bd75d25dc727a8f0a42d726fe1f39eccb82d2ec2fa7f0db75f46f660811\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 25.26495900000009,\n      \"stdoutBytes\": 10087,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"rm\",\n        \"c96c7bd75d25dc727a8f0a42d726fe1f39eccb82d2ec2fa7f0db75f46f660811\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 28.459374999998545,\n      \"stdoutBytes\": 65,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"network\",\n        \"inspect\",\n        \"wse-industry-e24_default\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 19.369708999998693,\n      \"stdoutBytes\": 1093,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"network\",\n        \"rm\",\n        \"2b37a7e8f45b48f8f0c76323de79c34eb579fafb91301e16cefdb28125cb318d\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 229.7559160000019,\n      \"stdoutBytes\": 65,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 20.241750000001048,\n      \"stdoutBytes\": 12794,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    }\n  ],\n  \"exits\": {\n    \"frontend\": {\n      \"purpose\": \"cleanup after incomplete/failed gate\",\n      \"signal\": \"SIGTERM\",\n      \"signalCommandExit\": 0,\n      \"exitCode\": 143,\n      \"elapsedMs\": 189.66187499999796,\n      \"awaited\": true,\n      \"observerTimedOut\": false,\n      \"forcedContainerKill\": false\n    },\n    \"worker\": {\n      \"purpose\": \"cleanup after incomplete/failed gate\",\n      \"signal\": \"SIGTERM\",\n      \"signalCommandExit\": 0,\n      \"exitCode\": 143,\n      \"elapsedMs\": 464.3378339999981,\n      \"awaited\": true,\n      \"observerTimedOut\": false,\n      \"forcedContainerKill\": false\n    },\n    \"api\": {\n      \"purpose\": \"cleanup after incomplete/failed gate\",\n      \"signal\": \"SIGTERM\",\n      \"signalCommandExit\": 0,\n      \"exitCode\": 143,\n      \"elapsedMs\": 331.82558399999834,\n      \"awaited\": true,\n      \"observerTimedOut\": false,\n      \"forcedContainerKill\": false\n    },\n    \"fixture\": {\n      \"purpose\": \"cleanup after incomplete/failed gate\",\n      \"signal\": \"SIGTERM\",\n      \"signalCommandExit\": 0,\n      \"exitCode\": 143,\n      \"elapsedMs\": 46.77612499999668,\n      \"awaited\": true,\n      \"observerTimedOut\": false,\n      \"forcedContainerKill\": false\n    }\n  },\n  \"cleanupFailures\": [],\n  \"privateArtifacts\": {\n    \"rawLogs\": false,\n    \"httpBodies\": false,\n    \"credentials\": false,\n    \"browserTrace\": false,\n    \"screenshot\": false,\n    \"video\": false,\n    \"har\": false,\n    \"storageState\": false\n  },\n  \"head\": \"2dbe240409db95ac0435f8a4d4270189f60b7d82\",\n  \"sourceFingerprint\": \"ff11785dd9b6da221b6b863102b24e38cf028f3b71790d5f8c787642f5603610\",\n  \"frozenHashes\": {\n    \"evidence/phase-1/E24/fixture.json\": \"47f86919e5f251c3f8ee394727235b906fc03a5eac67e36cc3e273486a21a8fd\",\n    \"scripts/e24-seed.mjs\": \"16d0f9286acfe76a5dfb79819d0ee49e975098ecb710464327d41c0f54ec4025\",\n    \"scripts/e24-baseline.mjs\": \"37bdf00c299706b5e5ef2d7ff11311605a9529fa0ee7abcfa9ef78dd62ce49c4\",\n    \"evidence/phase-1/E24/fixtures.md\": \"5574f4d4d6c8e8cbbd17273ba8385e3a1ac43c2193d34d40b0dca4da11090a1c\"\n  },\n  \"preflight\": {\n    \"vacantPorts\": [\n      4322,\n      4324,\n      4323,\n      4321\n    ],\n    \"existingApplicationResources\": 0,\n    \"postgres\": {\n      \"id\": \"7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4\",\n      \"image\": \"sha256:051f7b7b3abdd564d5d1bd1e8c4b9c1b6e77087d1dd22020ede611c096a272e0\",\n      \"configuredImage\": \"postgres:17.11-bookworm@sha256:051f7b7b3abdd564d5d1bd1e8c4b9c1b6e77087d1dd22020ede611c096a272e0\",\n      \"mounts\": [\n        {\n          \"type\": \"volume\",\n          \"name\": \"wse-industry_postgres-data\",\n          \"destination\": \"/var/lib/postgresql/data\"\n        }\n      ]\n    },\n    \"publicTableCount\": 0,\n    \"frozenInputsUnchanged\": true\n  },\n  \"seed\": {\n    \"users\": 2,\n    \"monitors\": 2,\n    \"checks\": 4,\n    \"queued\": 0,\n    \"running\": 0,\n    \"terminal\": 4\n  },\n  \"startup\": {\n    \"liveness\": {\n      \"api\": 200,\n      \"worker\": 200\n    },\n    \"readiness\": {\n      \"api\": 200,\n      \"worker\": 200\n    },\n    \"worker\": {\n      \"queueAge\": 0,\n      \"active\": 0,\n      \"claims\": 0,\n      \"recoveries\": 0\n    }\n  },\n  \"browser\": {\n    \"chromium\": \"151.0.7922.34\",\n    \"contexts\": 2,\n    \"ownerArticleCounts\": [\n      1,\n      1\n    ],\n    \"initialHistoryRows\": [\n      3,\n      1\n    ],\n    \"foreignDetailStatuses\": [\n      404,\n      404\n    ],\n    \"authenticatedServerHtml\": true,\n    \"serverHistoryBeforeHydration\": true,\n    \"javascriptStatus\": 200,\n    \"cssStatus\": 200,\n    \"pageErrors\": 0,\n    \"hydrationErrors\": 0,\n    \"closedBeforeOutage\": true\n  },\n  \"manual\": {\n    \"acceptedStatus\": 202,\n    \"acceptedState\": \"QUEUED\",\n    \"acceptedOutcomeFieldsNull\": true,\n    \"held\": {\n      \"held\": 1,\n      \"outboundCalls\": 1,\n      \"active\": 1,\n      \"localObservationAndReleaseMs\": 5.355459000002156,\n      \"dockerOrSqlBeforeRelease\": false\n    },\n    \"fixture\": {\n      \"outboundCalls\": 1,\n      \"holdRequests\": 1,\n      \"held\": 0,\n      \"releases\": 1,\n      \"watchdogReleases\": 0,\n      \"lastHeldMs\": 15.899124999999913\n    },\n    \"sameTerminalId\": true,\n    \"terminalState\": \"SUCCEEDED\",\n    \"httpStatus\": 200,\n    \"historyRows\": 4,\n    \"reloadRetainsResult\": true\n  },\n  \"outage\": {\n    \"liveness\": {\n      \"api\": 200,\n      \"worker\": 200\n    },\n    \"readiness\": {\n      \"api\": 503,\n      \"worker\": 503\n    },\n    \"rejectedStatus\": 503,\n    \"acceptedId\": false,\n    \"active\": 0,\n    \"claims\": 1,\n    \"recoveries\": 0,\n    \"outboundCalls\": 1,\n    \"sameLiveProcesses\": true\n  },\n  \"restore\": {\n    \"readiness\": {\n      \"api\": 200,\n      \"worker\": 200\n    },\n    \"samePostgresContainerImageVolumes\": true,\n    \"publicAuthorityUnchanged\": true,\n    \"exactPrivateRowsUnchanged\": true,\n    \"counts\": {\n      \"users\": 2,\n      \"monitors\": 2,\n      \"checks\": 5,\n      \"queued\": 0,\n      \"running\": 0,\n      \"terminal\": 5\n    },\n    \"rejectedIntentPersisted\": false,\n    \"sameLiveProcesses\": true\n  },\n  \"failure\": {\n    \"phase\": \"bounded real metrics and ten missing UUID requests\",\n    \"reason\": \"Only the applicable API and worker metric families are exposed\",\n    \"type\": \"Error\",\n    \"privateExceptionOutputRetained\": false\n  },\n  \"logs\": {\n    \"roles\": {\n      \"api\": {\n        \"lines\": 89,\n        \"structuredLines\": 89,\n        \"observationEvents\": 41\n      },\n      \"worker\": {\n        \"lines\": 45,\n        \"structuredLines\": 45,\n        \"observationEvents\": 4\n      },\n      \"frontend\": {\n        \"lines\": 5,\n        \"structuredLines\": 0,\n        \"observationEvents\": 0\n      },\n      \"fixture\": {\n        \"lines\": 0,\n        \"structuredLines\": 0,\n        \"observationEvents\": 0\n      }\n    },\n    \"runtimeSentinelValuesScanned\": 19,\n    \"runtimeSentinelMatches\": 0,\n    \"unboundedInputMatches\": 0,\n    \"responseCorrelations\": 8,\n    \"matchedResponseCorrelations\": 8,\n    \"generatedResponseRequestIds\": [\n      \"dcacefb8-eec4-4f86-9ed7-b453bc6f5c1a\",\n      \"e6fcf5f9-d9ff-4f61-9294-f3d32f7ba0b2\",\n      \"d7dde90a-6c7c-4a3b-a504-80bf40bc567e\",\n      \"d81a6a6e-3c19-4657-9371-a6ad76b73936\",\n      \"9061cf72-6a02-45d2-898c-e879a8e2d8e7\",\n      \"d209cc21-39a5-4b12-a233-0fb9121cafcc\",\n      \"96038228-8e3d-4440-b344-dabf2f5b35be\",\n      \"3ef84129-490d-4a18-956c-da511c7a3060\"\n    ],\n    \"stableApiProcess\": true,\n    \"stableWorkerProcess\": true,\n    \"distinctRoleProcesses\": true,\n    \"apiProcessId\": \"da50404d-7547-436d-b2fd-d509cd22554d\",\n    \"workerProcessId\": \"625401ef-cf28-47b5-86a4-7ac51732b873\",\n    \"committedClaimEvents\": 1,\n    \"unavailableAuthorityEvents\": 3,\n    \"recoveryEvents\": 0,\n    \"crossProcessCheckRunTraceClaimed\": false,\n    \"rawLogsRetained\": false\n  },\n  \"cleanup\": {\n    \"browserClosed\": true,\n    \"applicationContainersRemoved\": true,\n    \"applicationNetworkRemoved\": true,\n    \"ownedSchemaDropped\": true,\n    \"postgresPreserved\": true,\n    \"postgresStopAttempted\": true,\n    \"postgresRestoreAttempted\": true,\n    \"postgresRestored\": true,\n    \"forcedContainerKill\": false,\n    \"sharedOrPublicDataRemoved\": false\n  },\n  \"elapsedSeconds\": 24.582,\n  \"hostedCiExecutionClaimed\": false\n}\n"
+    },
+    {
+      "source": "evidence/phase-1/E24/container/author-ff11785dd9b6da22.started.json",
+      "sha256": "b3e2fa4ab6c2c243a6ce1aa5035eb24a08abd028bba9d9027298b6a1ab835149",
+      "bytes": 189,
+      "raw": "{\"actor\":\"author\",\"head\":\"2dbe240409db95ac0435f8a4d4270189f60b7d82\",\"fingerprint\":\"ff11785dd9b6da221b6b863102b24e38cf028f3b71790d5f8c787642f5603610\",\"startedAt\":\"2026-08-28T10:43:01.107Z\"}\n"
+    },
+    {
+      "source": "evidence/phase-1/E24/container/invocations.jsonl",
+      "sha256": "344c93072897b713765ed5504655ea7f6e45c2d1a1e1e8d7603e6cf83d4941b7",
+      "bytes": 239,
+      "raw": "{\"actor\":\"author\",\"head\":\"2dbe240409db95ac0435f8a4d4270189f60b7d82\",\"fingerprint\":\"ff11785dd9b6da221b6b863102b24e38cf028f3b71790d5f8c787642f5603610\",\"startedAt\":\"2026-08-28T10:43:01.107Z\",\"command\":\"node scripts/e24-container.mjs author\"}\n"
+    }
+  ],
+  "cumulativeBudget": {
+    "baselineInvocations": 1,
+    "expectedBaselineCounterexamples": 1,
+    "affectedMavenInvocations": 4,
+    "unexpectedFormalFailures": 4,
+    "repairTasksUsed": 3,
+    "maximumFreshRepairs": null,
+    "historicalMaximumFreshRepairs": 2,
+    "imageBuildInvocations": 1,
+    "containerScenarioInvocations": 1,
+    "browserInvocations": 1,
+    "postgresStopRestoreSequences": 1,
+    "rootRuntimeProductGateRuns": 0,
+    "loadRuns": 0,
+    "parameterSweeps": 0,
+    "automaticRetries": 0,
+    "cumulativeGateWallSeconds": 102.226
+  },
+  "rawRuntimeLogsOrResponsesRetained": false,
+  "candidateSourceChanged": false,
+  "independentRootFinalGate": "pending"
+}
diff --git a/evidence/phase-1/E24/repair3/repair3-image-build-native.json b/evidence/phase-1/E24/repair3/repair3-image-build-native.json
new file mode 100644
index 0000000..1dae60a
--- /dev/null
+++ b/evidence/phase-1/E24/repair3/repair3-image-build-native.json
@@ -0,0 +1,72 @@
+{
+  "thread": "E24",
+  "profile": "phase-1",
+  "attempt": 4,
+  "repair": 3,
+  "actor": "author",
+  "invocation": {
+    "command": [
+      "docker",
+      "compose",
+      "-p",
+      "wse-industry-e24",
+      "-f",
+      "compose.production.yaml",
+      "build",
+      "api",
+      "frontend"
+    ],
+    "environment": {
+      "DB_SCHEMA": "e24_container"
+    },
+    "displayCommand": "DB_SCHEMA=e24_container docker compose -p wse-industry-e24 -f compose.production.yaml build api frontend",
+    "startedAt": "2026-08-28T10:39:20.133345+00:00",
+    "attempt": 4,
+    "repair": 3,
+    "actor": "author",
+    "permission": "require_escalated pinned production image build/cache/network authorization",
+    "buildInvocation": 1,
+    "automaticRetries": 0,
+    "sourceReviewSha256": "15665ca61bf227754a22327c72550f9568c1ae9ef47071b93de24d3126cff9aa",
+    "authorizationSha256": "a24c7f7d01b50fb5d722ccf96c3734b5cd4784f9d3d265904e65ef7a5df7a811"
+  },
+  "nativeResult": {
+    "exitCode": 0,
+    "elapsedSeconds": 48.361,
+    "startedAt": "2026-08-28T10:39:20.133345+00:00",
+    "endedAt": "2026-08-28T10:40:08.494728+00:00"
+  },
+  "files": [
+    {
+      "source": "output/phase-1/e24/repair3-image-build.started.json",
+      "sha256": "8b3502e804117cac90c2bc099cea0f5841e0cdd0c6f751a3d5cf692eaff131df",
+      "bytes": 783,
+      "raw": "{\n  \"command\": [\n    \"docker\",\n    \"compose\",\n    \"-p\",\n    \"wse-industry-e24\",\n    \"-f\",\n    \"compose.production.yaml\",\n    \"build\",\n    \"api\",\n    \"frontend\"\n  ],\n  \"environment\": {\n    \"DB_SCHEMA\": \"e24_container\"\n  },\n  \"displayCommand\": \"DB_SCHEMA=e24_container docker compose -p wse-industry-e24 -f compose.production.yaml build api frontend\",\n  \"startedAt\": \"2026-08-28T10:39:20.133345+00:00\",\n  \"attempt\": 4,\n  \"repair\": 3,\n  \"actor\": \"author\",\n  \"permission\": \"require_escalated pinned production image build/cache/network authorization\",\n  \"buildInvocation\": 1,\n  \"automaticRetries\": 0,\n  \"sourceReviewSha256\": \"15665ca61bf227754a22327c72550f9568c1ae9ef47071b93de24d3126cff9aa\",\n  \"authorizationSha256\": \"a24c7f7d01b50fb5d722ccf96c3734b5cd4784f9d3d265904e65ef7a5df7a811\"\n}\n"
+    },
+    {
+      "source": "output/phase-1/e24/repair3-image-build.log",
+      "sha256": "a8c25f7344476f6ed2c257098e5dc170149c4059366980346677649cbe1e54e3",
+      "bytes": 10064,
+      "raw": "Compose can now delegate builds to bake for better performance.\n To do so, set COMPOSE_BAKE=true.\n#0 building with \"desktop-linux\" instance using docker driver\n\n#1 [frontend internal] load build definition from Dockerfile.frontend\n#1 transferring dockerfile: 893B done\n#1 DONE 0.0s\n\n#2 [api internal] load build definition from Dockerfile.api\n#2 transferring dockerfile: 442B 0.0s done\n#2 DONE 0.0s\n\n#3 [api internal] load metadata for docker.io/library/eclipse-temurin:21.0.7_6-jre-jammy@sha256:c4e6542e774de504da9b4729ff8d761287c965c1d788528ca78da30024efdb23\n#3 ...\n\n#4 [frontend internal] load metadata for docker.io/library/node:24.19.0-bookworm-slim@sha256:a9f5f7c91a432850b2a8a7797adf5eadb6c733ceed61167806cee7ea7fbc29df\n#4 DONE 1.5s\n\n#5 [frontend internal] load .dockerignore\n#5 transferring context: 226B done\n#5 DONE 0.0s\n\n#6 [frontend build 1/8] FROM docker.io/library/node:24.19.0-bookworm-slim@sha256:a9f5f7c91a432850b2a8a7797adf5eadb6c733ceed61167806cee7ea7fbc29df\n#6 resolve docker.io/library/node:24.19.0-bookworm-slim@sha256:a9f5f7c91a432850b2a8a7797adf5eadb6c733ceed61167806cee7ea7fbc29df 0.0s done\n#6 DONE 0.0s\n\n#7 [frontend build 2/8] WORKDIR /app\n#7 CACHED\n\n#8 [frontend internal] load build context\n#8 transferring context: 71.36kB done\n#8 DONE 0.0s\n\n#9 [frontend build 3/8] RUN npm install --global npm@11.17.0\n#9 ...\n\n#3 [api internal] load metadata for docker.io/library/eclipse-temurin:21.0.7_6-jre-jammy@sha256:c4e6542e774de504da9b4729ff8d761287c965c1d788528ca78da30024efdb23\n#3 DONE 2.3s\n\n#10 [api internal] load .dockerignore\n#10 transferring context: 226B done\n#10 DONE 0.0s\n\n#11 [api 1/3] FROM docker.io/library/eclipse-temurin:21.0.7_6-jre-jammy@sha256:c4e6542e774de504da9b4729ff8d761287c965c1d788528ca78da30024efdb23\n#11 resolve docker.io/library/eclipse-temurin:21.0.7_6-jre-jammy@sha256:c4e6542e774de504da9b4729ff8d761287c965c1d788528ca78da30024efdb23 0.0s done\n#11 sha256:eb95edc6ae4d7057d35402569d6d45731db8c4222205b7295e44611d65689c07 0B / 2.28kB 0.2s\n#11 ...\n\n#12 [api internal] load build context\n#12 transferring context: 60.10MB 0.4s done\n#12 DONE 0.5s\n\n#11 [api 1/3] FROM docker.io/library/eclipse-temurin:21.0.7_6-jre-jammy@sha256:c4e6542e774de504da9b4729ff8d761287c965c1d788528ca78da30024efdb23\n#11 sha256:28867d80ab50452e2e2ad51c7e95164f101ca79abc237541e761cfeb13974810 0B / 156B 0.2s\n#11 sha256:760ec7c762f37c54f4eee8ef1d6244558fd3ec9f99cdf11a5c043e8f0cc2ea2a 0B / 52.07MB 0.2s\n#11 sha256:a1091b0892f1291aa904c1d32ba888d87e6d2c24ce1bd808445387085c3b43ec 0B / 16.06MB 0.2s\n#11 sha256:28867d80ab50452e2e2ad51c7e95164f101ca79abc237541e761cfeb13974810 156B / 156B 0.7s done\n#11 sha256:a1091b0892f1291aa904c1d32ba888d87e6d2c24ce1bd808445387085c3b43ec 3.15MB / 16.06MB 0.8s\n#11 sha256:eb95edc6ae4d7057d35402569d6d45731db8c4222205b7295e44611d65689c07 2.28kB / 2.28kB 0.8s done\n#11 sha256:ef6d179edc98e93dc6073cb3ddec6f1a6ed1d68d04cd7836a82abfd397922a05 0B / 27.36MB 0.2s\n#11 sha256:a1091b0892f1291aa904c1d32ba888d87e6d2c24ce1bd808445387085c3b43ec 16.06MB / 16.06MB 1.0s done\n#11 sha256:ef6d179edc98e93dc6073cb3ddec6f1a6ed1d68d04cd7836a82abfd397922a05 2.10MB / 27.36MB 0.5s\n#11 sha256:ef6d179edc98e93dc6073cb3ddec6f1a6ed1d68d04cd7836a82abfd397922a05 8.39MB / 27.36MB 0.6s\n#11 sha256:ef6d179edc98e93dc6073cb3ddec6f1a6ed1d68d04cd7836a82abfd397922a05 16.78MB / 27.36MB 0.8s\n#11 sha256:ef6d179edc98e93dc6073cb3ddec6f1a6ed1d68d04cd7836a82abfd397922a05 27.36MB / 27.36MB 1.0s done\n#11 extracting sha256:ef6d179edc98e93dc6073cb3ddec6f1a6ed1d68d04cd7836a82abfd397922a05\n#11 extracting sha256:ef6d179edc98e93dc6073cb3ddec6f1a6ed1d68d04cd7836a82abfd397922a05 0.6s done\n#11 sha256:760ec7c762f37c54f4eee8ef1d6244558fd3ec9f99cdf11a5c043e8f0cc2ea2a 3.15MB / 52.07MB 2.4s\n#11 extracting sha256:a1091b0892f1291aa904c1d32ba888d87e6d2c24ce1bd808445387085c3b43ec\n#11 sha256:760ec7c762f37c54f4eee8ef1d6244558fd3ec9f99cdf11a5c043e8f0cc2ea2a 6.29MB / 52.07MB 2.7s\n#11 extracting sha256:a1091b0892f1291aa904c1d32ba888d87e6d2c24ce1bd808445387085c3b43ec 0.4s done\n#11 sha256:760ec7c762f37c54f4eee8ef1d6244558fd3ec9f99cdf11a5c043e8f0cc2ea2a 10.49MB / 52.07MB 2.9s\n#11 sha256:760ec7c762f37c54f4eee8ef1d6244558fd3ec9f99cdf11a5c043e8f0cc2ea2a 13.63MB / 52.07MB 3.0s\n#11 sha256:760ec7c762f37c54f4eee8ef1d6244558fd3ec9f99cdf11a5c043e8f0cc2ea2a 17.83MB / 52.07MB 3.2s\n#11 sha256:760ec7c762f37c54f4eee8ef1d6244558fd3ec9f99cdf11a5c043e8f0cc2ea2a 22.02MB / 52.07MB 3.5s\n#11 sha256:760ec7c762f37c54f4eee8ef1d6244558fd3ec9f99cdf11a5c043e8f0cc2ea2a 26.21MB / 52.07MB 3.6s\n#11 sha256:760ec7c762f37c54f4eee8ef1d6244558fd3ec9f99cdf11a5c043e8f0cc2ea2a 29.36MB / 52.07MB 3.8s\n#11 sha256:760ec7c762f37c54f4eee8ef1d6244558fd3ec9f99cdf11a5c043e8f0cc2ea2a 32.51MB / 52.07MB 3.9s\n#11 sha256:760ec7c762f37c54f4eee8ef1d6244558fd3ec9f99cdf11a5c043e8f0cc2ea2a 36.70MB / 52.07MB 4.1s\n#11 sha256:760ec7c762f37c54f4eee8ef1d6244558fd3ec9f99cdf11a5c043e8f0cc2ea2a 40.89MB / 52.07MB 4.4s\n#11 sha256:760ec7c762f37c54f4eee8ef1d6244558fd3ec9f99cdf11a5c043e8f0cc2ea2a 45.09MB / 52.07MB 4.5s\n#11 sha256:760ec7c762f37c54f4eee8ef1d6244558fd3ec9f99cdf11a5c043e8f0cc2ea2a 49.28MB / 52.07MB 4.7s\n#11 sha256:760ec7c762f37c54f4eee8ef1d6244558fd3ec9f99cdf11a5c043e8f0cc2ea2a 52.07MB / 52.07MB 4.8s\n#11 sha256:760ec7c762f37c54f4eee8ef1d6244558fd3ec9f99cdf11a5c043e8f0cc2ea2a 52.07MB / 52.07MB 4.8s done\n#11 extracting sha256:760ec7c762f37c54f4eee8ef1d6244558fd3ec9f99cdf11a5c043e8f0cc2ea2a\n#11 extracting sha256:760ec7c762f37c54f4eee8ef1d6244558fd3ec9f99cdf11a5c043e8f0cc2ea2a 1.6s done\n#11 DONE 6.7s\n\n#9 [frontend build 3/8] RUN npm install --global npm@11.17.0\n#9 6.334 \n#9 6.334 changed 10 packages in 6s\n#9 6.334 \n#9 6.334 15 packages are looking for funding\n#9 6.334   run `npm fund` for details\n#9 DONE 7.6s\n\n#11 [api 1/3] FROM docker.io/library/eclipse-temurin:21.0.7_6-jre-jammy@sha256:c4e6542e774de504da9b4729ff8d761287c965c1d788528ca78da30024efdb23\n#11 extracting sha256:28867d80ab50452e2e2ad51c7e95164f101ca79abc237541e761cfeb13974810 0.1s done\n#11 DONE 6.8s\n\n#13 [frontend build 4/8] COPY package.json package-lock.json ./\n#13 ...\n\n#11 [api 1/3] FROM docker.io/library/eclipse-temurin:21.0.7_6-jre-jammy@sha256:c4e6542e774de504da9b4729ff8d761287c965c1d788528ca78da30024efdb23\n#11 extracting sha256:eb95edc6ae4d7057d35402569d6d45731db8c4222205b7295e44611d65689c07 0.1s done\n#11 DONE 6.8s\n\n#14 [api 2/3] WORKDIR /app\n#14 DONE 0.1s\n\n#13 [frontend build 4/8] COPY package.json package-lock.json ./\n#13 DONE 0.2s\n\n#15 [frontend build 5/8] RUN npm ci --no-audit --no-fund\n#15 ...\n\n#16 [api 3/3] COPY --chown=10001:10001 backend/target/monitor-api-0.0.1.jar /app/app.jar\n#16 DONE 0.4s\n\n#17 [api] exporting to image\n#17 exporting layers\n#17 exporting layers 2.2s done\n#17 exporting manifest sha256:f4fddd79f7463cd49f4479df93a32990f2176fcc176accfcd03aec1f8a37d532 0.0s done\n#17 exporting config sha256:ebb73527c9648a279007dbef0c1cf5cbabb291fb8f5d7920569be735137b59d0 0.0s done\n#17 exporting attestation manifest sha256:d50176aa5d4c7bc359b42360db6f3aeccc71e8e782ce60f7a6ff90f134b972a8 0.0s done\n#17 exporting manifest list sha256:c6191345230603ccff67148f69c73df84f5ceea63a052ff1cf7d394987799328 done\n#17 naming to docker.io/library/wse-industry-e24-api:local done\n#17 unpacking to docker.io/library/wse-industry-e24-api:local\n#17 unpacking to docker.io/library/wse-industry-e24-api:local 0.3s done\n#17 DONE 2.6s\n\n#18 [api] resolving provenance for metadata file\n#18 DONE 0.0s\n\n#15 [frontend build 5/8] RUN npm ci --no-audit --no-fund\n#15 9.321 \n#15 9.321 added 32 packages in 9s\n#15 9.323 npm notice\n#15 9.323 npm notice New major version of npm available! 11.17.0 -> 12.0.2\n#15 9.323 npm notice Changelog: https://github.com/npm/cli/releases/tag/v12.0.2\n#15 9.323 npm notice To update run: npm install -g npm@12.0.2\n#15 9.323 npm notice\n#15 DONE 11.0s\n\n#19 [frontend build 6/8] COPY next.config.mjs tsconfig.json ./\n#19 DONE 0.1s\n\n#20 [frontend build 7/8] COPY app ./app\n#20 DONE 0.0s\n\n#21 [frontend build 8/8] RUN npm run build\n#21 0.200 \n#21 0.200 > industry-spring-monitor@0.0.1 build\n#21 0.200 > next build --webpack\n#21 0.200 \n#21 0.414 \u25b2 Next.js 16.3.3 (webpack)\n#21 0.425 \u2713 Running next.config.mjs took 10ms\n#21 0.446 \n#21 0.465   Creating an optimized production build ...\n#21 5.158 \u2713 Compiled successfully in 3.1s\n#21 5.160   Running TypeScript ...\n#21 6.494   Finished TypeScript in 1334ms ...\n#21 6.496   Collecting page data using 6 workers ...\n#21 7.209   Generating static pages using 6 workers (0/4) ...\n#21 7.646   Generating static pages using 6 workers (1/4) \r\n#21 7.646   Generating static pages using 6 workers (2/4) \r\n#21 7.646   Generating static pages using 6 workers (3/4) \r\n#21 7.646 \u2713 Generating static pages using 6 workers (4/4) in 438ms\n#21 7.748   Finalizing page optimization ...\n#21 7.748   Collecting build traces ...\n#21 23.85 \n#21 23.86 Route (app)\n#21 23.86 \u250c \u25cb /\n#21 23.86 \u251c \u25cb /_not-found\n#21 23.86 \u251c \u25cb /login\n#21 23.86 \u2514 \u0192 /monitors\n#21 23.86 \n#21 23.86 \n#21 23.86 \u25cb  (Static)   prerendered as static content\n#21 23.86 \u0192  (Dynamic)  server-rendered on demand\n#21 23.86 \n#21 DONE 24.1s\n\n#22 [frontend stage-1 3/4] COPY --from=build --chown=1000:1000 /app/.next/standalone ./\n#22 DONE 0.9s\n\n#23 [frontend stage-1 4/4] COPY --from=build --chown=1000:1000 /app/.next/static ./.next/static\n#23 DONE 0.0s\n\n#24 [frontend] exporting to image\n#24 exporting layers\n#24 exporting layers 1.5s done\n#24 exporting manifest sha256:9f9e721835d2b45159c539af383754e70a5e00316307c5ad09da7a16131922b4 done\n#24 exporting config sha256:1cf0fb64b8d1378490da81a9705d7cdfea1b644e7e09f079c74ad7363302894e done\n#24 exporting attestation manifest sha256:2b1e1bec135d927ba872e016825032cf83d4d7de18f768e17fbdbdaa41662f66 0.0s done\n#24 exporting manifest list sha256:9356a3b65bd58be8e12f5a4721975f4801e1601f7dbef0a4bb7ddfd6ef93ac9e done\n#24 naming to docker.io/library/wse-industry-e24-frontend:local done\n#24 unpacking to docker.io/library/wse-industry-e24-frontend:local\n#24 unpacking to docker.io/library/wse-industry-e24-frontend:local 0.4s done\n#24 DONE 1.9s\n\n#25 [frontend] resolving provenance for metadata file\n#25 DONE 0.0s\n api  Built\n frontend  Built\n"
+    },
+    {
+      "source": "output/phase-1/e24/repair3-image-build.result.json",
+      "sha256": "96178d3205f7eaebe85cdf48feafd7e45c39eddc71999f6516b58ea3b3b7037e",
+      "bytes": 148,
+      "raw": "{\n  \"exitCode\": 0,\n  \"elapsedSeconds\": 48.361,\n  \"startedAt\": \"2026-08-28T10:39:20.133345+00:00\",\n  \"endedAt\": \"2026-08-28T10:40:08.494728+00:00\"\n}\n"
+    },
+    {
+      "source": "output/phase-1/e24/invocations.jsonl",
+      "sha256": "e63939ec9f58d36e5eab8b7967dce4182dd54432d4faaafe175d70715f2eb959",
+      "bytes": 1868,
+      "raw": "{\"command\":\"node scripts/e24-baseline.mjs\",\"startedAt\":\"2026-08-28T08:51:05.183Z\",\"elapsedSeconds\":7.477,\"exitCode\":0}\n{\"command\":\"mvn -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package\",\"startedAt\":\"2026-08-28T09:11:45.983Z\",\"elapsedSeconds\":1.724,\"exitCode\":1,\"signal\":null}\n{\"command\":\"mvn -U -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package\",\"startedAt\":\"2026-08-28T09:22:21.048583+00:00\",\"attempt\":2,\"repair\":1,\"actor\":\"author\",\"permission\":\"require_escalated network/cache authorization\",\"forcedDescriptorRefreshes\":1,\"elapsedSeconds\":4.051,\"exitCode\":1,\"signal\":null}\n{\"command\":\"mvn -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package\",\"startedAt\":\"2026-08-28T09:31:22.232667+00:00\",\"attempt\":3,\"repair\":2,\"actor\":\"author\",\"permission\":\"require_escalated network/cache/PostgreSQL authorization\",\"forcedDescriptorRefreshes\":0,\"elapsedSeconds\":6.999,\"exitCode\":1,\"signal\":null}\n{\"command\":\"mvn -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package\",\"startedAt\":\"2026-08-28T10:35:41.429288+00:00\",\"attempt\":4,\"repair\":3,\"actor\":\"author\",\"permission\":\"require_escalated Java/Maven/cache/network/PostgreSQL authorization\",\"forcedDescriptorRefreshes\":0,\"elapsedSeconds\":8.772,\"exitCode\":0,\"signal\":null,\"nativeBundle\":\"evidence/phase-1/E24/repair3/repair3-affected-java-native.json\"}\n{\"command\":\"DB_SCHEMA=e24_container docker compose -p wse-industry-e24 -f compose.production.yaml build api frontend\",\"startedAt\":\"2026-08-28T10:39:20.133345+00:00\",\"attempt\":4,\"repair\":3,\"actor\":\"author\",\"permission\":\"require_escalated pinned production image build/cache/network authorization\",\"elapsedSeconds\":48.361,\"exitCode\":0,\"signal\":null}\n"
+    }
+  ],
+  "applicationScenarioStarted": false,
+  "browserStarted": false,
+  "baselineRerun": false,
+  "sourceModified": false,
+  "cumulativeBuildInvocations": 1,
+  "cumulativeUnexpectedFormalFailures": 3,
+  "cleanup": "Docker build command awaited; application services were not started; build cache/images retained for authorized container gate and root-owned final cleanup"
+}
diff --git a/evidence/phase-1/E24/repair3/repair3-preservation.json b/evidence/phase-1/E24/repair3/repair3-preservation.json
new file mode 100644
index 0000000..ac94d57
--- /dev/null
+++ b/evidence/phase-1/E24/repair3/repair3-preservation.json
@@ -0,0 +1,238 @@
+{
+  "thread": "E24",
+  "profile": "phase-1",
+  "attempt": 4,
+  "repair": 3,
+  "start": "563b325ef871fe6d1fbfef7cf39a6581f2d0a94d",
+  "headBeforeRepair": "2dbe240409db95ac0435f8a4d4270189f60b7d82",
+  "specRevision": "2ada57a71cd34fa2fae9809415c362a8bbfcdf02",
+  "status": "FAILED_METRIC_INVENTORY; preserved for fresh root-directed repair",
+  "source": {
+    "scripts/e24-container.mjs": "58872e40e522808f033b34052fca67bd031bafb4f6fe26ced81b225978a1bfb8",
+    "backend/src/test/java/dev/evolution/monitor/OperationsIntegrationTest.java": "264c758e4e2c8ef3ab3b688573ae30c2198fee0ed1d05b4f3583ca0cd16cc487"
+  },
+  "frozenInputs": [
+    {
+      "path": "evidence/phase-1/E24/fixture.json",
+      "sha256": "47f86919e5f251c3f8ee394727235b906fc03a5eac67e36cc3e273486a21a8fd",
+      "unchanged": true
+    },
+    {
+      "path": "scripts/e24-seed.mjs",
+      "sha256": "16d0f9286acfe76a5dfb79819d0ee49e975098ecb710464327d41c0f54ec4025",
+      "unchanged": true
+    },
+    {
+      "path": "scripts/e24-baseline.mjs",
+      "sha256": "37bdf00c299706b5e5ef2d7ff11311605a9529fa0ee7abcfa9ef78dd62ce49c4",
+      "unchanged": true
+    },
+    {
+      "path": "evidence/phase-1/E24/fixtures.md",
+      "sha256": "5574f4d4d6c8e8cbbd17273ba8385e3a1ac43c2193d34d40b0dca4da11090a1c",
+      "unchanged": true
+    }
+  ],
+  "completedGates": [
+    {
+      "name": "affected Java package",
+      "exitCode": 0,
+      "elapsedSeconds": 8.772,
+      "tests": 5,
+      "passed": 5,
+      "errors": 0,
+      "failures": 0,
+      "skipped": 0
+    },
+    {
+      "name": "production image build",
+      "exitCode": 0,
+      "elapsedSeconds": 48.361
+    },
+    {
+      "name": "author production container scenario",
+      "exitCode": 1,
+      "elapsedSeconds": 24.842,
+      "failure": {
+        "phase": "bounded real metrics and ten missing UUID requests",
+        "reason": "Only the applicable API and worker metric families are exposed",
+        "type": "Error",
+        "privateExceptionOutputRetained": false
+      }
+    }
+  ],
+  "partialObservations": {
+    "browserOwnerHistoryStaticAndServerRoute": true,
+    "manualAcceptedIntents": 1,
+    "realOutboundRequests": 1,
+    "heldWorkerActive": 1,
+    "explicitFixtureReleaseMs": 15.899124999999913,
+    "watchdogReleases": 0,
+    "sameIdTerminal": "SUCCEEDED/200",
+    "postgresStopRestoreSequences": 1,
+    "readinessDuringOutage": {
+      "api": 503,
+      "worker": 503
+    },
+    "livenessDuringOutage": {
+      "api": 200,
+      "worker": 200
+    },
+    "unsafeSubmissionStatus": 503,
+    "exactAuthorityRowsUnchanged": true,
+    "missingUuidRequests": 0,
+    "runtimeUidArtifactInspectionReached": false,
+    "formalSignalAcceptancePhaseReached": false,
+    "cleanupOnlyNative143Exits": {
+      "frontend": {
+        "purpose": "cleanup after incomplete/failed gate",
+        "signal": "SIGTERM",
+        "signalCommandExit": 0,
+        "exitCode": 143,
+        "elapsedMs": 189.66187499999796,
+        "awaited": true,
+        "observerTimedOut": false,
+        "forcedContainerKill": false
+      },
+      "worker": {
+        "purpose": "cleanup after incomplete/failed gate",
+        "signal": "SIGTERM",
+        "signalCommandExit": 0,
+        "exitCode": 143,
+        "elapsedMs": 464.3378339999981,
+        "awaited": true,
+        "observerTimedOut": false,
+        "forcedContainerKill": false
+      },
+      "api": {
+        "purpose": "cleanup after incomplete/failed gate",
+        "signal": "SIGTERM",
+        "signalCommandExit": 0,
+        "exitCode": 143,
+        "elapsedMs": 331.82558399999834,
+        "awaited": true,
+        "observerTimedOut": false,
+        "forcedContainerKill": false
+      },
+      "fixture": {
+        "purpose": "cleanup after incomplete/failed gate",
+        "signal": "SIGTERM",
+        "signalCommandExit": 0,
+        "exitCode": 143,
+        "elapsedMs": 46.77612499999668,
+        "awaited": true,
+        "observerTimedOut": false,
+        "forcedContainerKill": false
+      }
+    },
+    "logScanDuringCleanup": {
+      "runtimeSentinelValuesScanned": 19,
+      "runtimeSentinelMatches": 0,
+      "unboundedInputMatches": 0,
+      "matchedResponseCorrelations": 8,
+      "responseCorrelations": 8
+    }
+  },
+  "cleanup": {
+    "browserClosed": true,
+    "applicationContainersRemoved": true,
+    "applicationNetworkRemoved": true,
+    "ownedSchemaDropped": true,
+    "postgresPreserved": true,
+    "postgresStopAttempted": true,
+    "postgresRestoreAttempted": true,
+    "postgresRestored": true,
+    "forcedContainerKill": false,
+    "sharedOrPublicDataRemoved": false
+  },
+  "nativeFiles": [
+    {
+      "path": "evidence/phase-1/E24/container/author-ff11785dd9b6da22.json",
+      "sha256": "927dc651b22ea41ad8c8d5f4858671275ca6c1beda15866aaaf080b27938753f",
+      "bytes": 37141
+    },
+    {
+      "path": "evidence/phase-1/E24/container/author-ff11785dd9b6da22.started.json",
+      "sha256": "b3e2fa4ab6c2c243a6ce1aa5035eb24a08abd028bba9d9027298b6a1ab835149",
+      "bytes": 189
+    },
+    {
+      "path": "evidence/phase-1/E24/container/invocations.jsonl",
+      "sha256": "344c93072897b713765ed5504655ea7f6e45c2d1a1e1e8d7603e6cf83d4941b7",
+      "bytes": 239
+    },
+    {
+      "path": "evidence/phase-1/E24/repair3/cumulative-invocations.jsonl",
+      "sha256": "7b6d4221eaf0d1cabb7480e8e802d0a41474c9675c507ce58f3a717ba0f9f58c",
+      "bytes": 2207
+    },
+    {
+      "path": "evidence/phase-1/E24/repair3/repair3-affected-java-native.json",
+      "sha256": "01ee29d19fc3625f0dffec9535136d6368ab6e9a9c2b72139ff5e0fe19e4abc2",
+      "bytes": 151262
+    },
+    {
+      "path": "evidence/phase-1/E24/repair3/repair3-author-container-native.json",
+      "sha256": "db51914f1603593bbe145073fe7a0793baf138ab0d785750ef8cd83d915a8904",
+      "bytes": 48682
+    },
+    {
+      "path": "evidence/phase-1/E24/repair3/repair3-image-build-native.json",
+      "sha256": "faf87887fdcd887cb6f342d9b4adfc395693d4979eb8a3835bc569d39b091cfa",
+      "bytes": 15670
+    },
+    {
+      "path": "evidence/phase-1/E24/repair3/repair3-java-ledger-append.json",
+      "sha256": "95635a43828ea0e7539ab5c72406ad3ad7475863f1aed6685e6e6def6020527a",
+      "bytes": 922
+    },
+    {
+      "path": "evidence/phase-1/E24/repair3/repair3-source-diagnosis.json",
+      "sha256": "645afff8254aa285652e294a5a2921a9ef67b8968fec5bbe064201cc3453bef0",
+      "bytes": 3452
+    }
+  ],
+  "cumulativeBudget": {
+    "baselineInvocations": 1,
+    "expectedBaselineCounterexamples": 1,
+    "baselineWallSeconds": 7.477,
+    "affectedMavenInvocations": 4,
+    "affectedMavenWallSeconds": 21.546,
+    "javaTestsExecuted": 10,
+    "javaTestsPassed": 9,
+    "javaTestErrors": 1,
+    "javaAssertionFailures": 0,
+    "javaTestsSkipped": 0,
+    "unexpectedFormalFailures": 4,
+    "repairTasksUsed": 3,
+    "maximumFreshRepairs": null,
+    "historicalMaximumFreshRepairs": 2,
+    "forcedDescriptorRefreshes": 1,
+    "imageBuildInvocations": 1,
+    "imageBuildWallSeconds": 48.361,
+    "containerScenarios": 1,
+    "containerScenarioWallSeconds": 24.842,
+    "browserInvocations": 1,
+    "postgresStopRestoreSequences": 1,
+    "rootRuntimeProductGateRuns": 0,
+    "cumulativeGateWallSeconds": 102.226,
+    "loadRuns": 0,
+    "parameterSweeps": 0,
+    "automaticRetries": 0,
+    "e11KillScenarioRuns": 0,
+    "e20DatasetRuns": 0
+  },
+  "afterFailure": {
+    "formalGatesRun": 0,
+    "productOrObserverChanges": false,
+    "rawRuntimeSecretsLogsOrBodiesPersisted": false,
+    "onlyReadOnlyDependencyDiagnosisAndPreservation": true
+  },
+  "unresolved": [
+    "Full actual metric inventory/tag/ten-UUID acceptance remains unverified.",
+    "The exact exported inventory was not preserved, so only a source-derived diagnosis is available.",
+    "Actual UID/artifact inspection and the normal signal-acceptance phase were not reached; cleanup exits are not substituted for full gate success.",
+    "Independent root final gate and overall phase-1 E24 acceptance remain pending."
+  ],
+  "hostedCiExecutionClaimed": false
+}
diff --git a/evidence/phase-1/E24/repair3/repair3-source-diagnosis.json b/evidence/phase-1/E24/repair3/repair3-source-diagnosis.json
new file mode 100644
index 0000000..07a1be8
--- /dev/null
+++ b/evidence/phase-1/E24/repair3/repair3-source-diagnosis.json
@@ -0,0 +1,48 @@
+{
+  "thread": "E24",
+  "profile": "phase-1",
+  "attempt": 4,
+  "repair": 3,
+  "basis": "source-derived expectations from pinned cached dependencies, not recovered runtime metric names",
+  "runtimeInventoryRetained": false,
+  "freshRuntimeProbePerformed": false,
+  "evidenceGap": "The observer checks the inventory before assigning it to evidence; the fetched names existed only in the now-exited process RAM.",
+  "classes": [
+    {
+      "jar": ".m2/repository/io/micrometer/micrometer-core/1.15.12/micrometer-core-1.15.12.jar",
+      "jarSha256": "30f6136353e39bc067b26745a6c31b0b6e419d8ff59c05b5364d7c1d4b78083e",
+      "class": "io/micrometer/core/instrument/observation/DefaultMeterObservationHandler.class",
+      "classSha256": "738bfd0b69149123435b46c8776f463c0d7997034372ef2f2d41da44ad93d63f",
+      "method": "read-only javap -c -p of the pinned cached class; no product invocation"
+    },
+    {
+      "jar": ".m2/repository/org/springframework/boot/spring-boot-actuator-autoconfigure/3.5.16/spring-boot-actuator-autoconfigure-3.5.16.jar",
+      "jarSha256": "50a5630e503c24f4421b2b3dde8b4f58ff5aedf263899aa80213ba5e02787fc3",
+      "class": "org/springframework/boot/actuate/autoconfigure/observation/ObservationAutoConfiguration$MeterObservationHandlerConfiguration$OnlyMetricsMeterObservationHandlerConfiguration.class",
+      "classSha256": "47645a28040fde5d7c981d6b414c1463fe4b8f28fdc60b186f600980ef7730ad",
+      "method": "read-only javap -c -p of the pinned cached class; no product invocation"
+    },
+    {
+      "jar": ".m2/repository/org/springframework/boot/spring-boot-actuator-autoconfigure/3.5.16/spring-boot-actuator-autoconfigure-3.5.16.jar",
+      "jarSha256": "50a5630e503c24f4421b2b3dde8b4f58ff5aedf263899aa80213ba5e02787fc3",
+      "class": "org/springframework/boot/actuate/autoconfigure/metrics/PropertiesMeterFilter.class",
+      "classSha256": "c7d858c5ba71fbecc523915154c888244c73f3ce538611ac723aa7616294dfb3",
+      "method": "read-only javap -c -p of the pinned cached class; no product invocation"
+    }
+  ],
+  "findings": [
+    "Boot3.5.16 OnlyMetricsMeterObservationHandlerConfiguration selects the default handler when management.observations.long-task-timer.enabled is true; the shipped metadata defaults it to true.",
+    "Micrometer1.15.12 DefaultMeterObservationHandler.onStart registers observation name plus .active as a LongTaskTimer by default.",
+    "Boot3.5.16 PropertiesMeterFilter.doLookup walks dotted name prefixes, so the current enable.http.server.requests=true also permits its .active child family.",
+    "DefaultMeterObservationHandler.onStop appends an error tag after reading the convention low-cardinality keys, so convention-only assertions do not cover the exported tag set."
+  ],
+  "rootApprovedNextRepairProposal": [
+    "Add the narrow management.metrics.enable.http.server.requests.active=false override.",
+    "Use an exact-http.server.requests MeterFilter to remove only the framework-added error dimension; preserve the frozen uri/method/status/process_role contract.",
+    "Extend existing HttpObservationsTest through real ObservationRegistry, DefaultMeterObservationHandler, configured MeterRegistry and MetricsEndpoint; no config-only substitute assertions.",
+    "Record bounded safe metric names before the observer inventory assertion."
+  ],
+  "proposalImplementedInThisCandidate": false,
+  "frozenCriteriaChanged": false,
+  "productOrObserverChangedAfterFailure": false
+}


