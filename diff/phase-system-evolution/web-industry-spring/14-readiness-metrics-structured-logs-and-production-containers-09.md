## `test: preserve passing phase-1 container metrics gate`

diff --git a/TRACK.md b/TRACK.md
index 45c351b..d4d3609 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -488,26 +488,32 @@ crash, E12 outbound, browser or load gate was repeated for E20.
 
 ## Phase-1 operations and production-container candidate (E24)
 
-E24 is incomplete. Attempt4 / repair3 passed its five affected Java tests and
-one production image build, then failed the author container gate at the metric
-family inventory assertion. The original failed attempts and frozen inputs are
-preserved; no baseline, load scenario or automatic retry was run. A new repair
-requires root review and explicit dispatch. The independent root gate and hosted
-CI execution have not run.
+Attempt5 / repair4 passed its six affected Java tests, one backend-only image
+rebuild and the complete author container gate. E24 acceptance still awaits the
+independent root gate; hosted CI execution has not run. Attempt4's metric-family
+inventory rejection and the three earlier Maven failures remain preserved.
+The baseline ran once in the original attempt and was not repeated. No load
+scenario or automatic retry was run.
 
 The candidate separates API and worker `/ops/health/liveness` from
 `/ops/health/readiness`, with PostgreSQL as authority. Required meter families are
 `http.server.requests`, `check.queue.age`, `check.worker.active`, `check.claims`
 and `check.recoveries`. The HTTP contract permits only `uri`, `method`, `status`
 and `process_role` dimensions; queue/worker dimensions use only `process_role`.
-The pending issue is at the exported registry boundary: pinned framework code
-can add an HTTP `.active` family and an `error` dimension after the convention
-runs. These are source-derived findings; the failed observer did not retain its
-actual fetched inventory. The frozen family/tag criteria have not been changed.
+The completed HTTP timer removes only the framework-added `error` tag; the
+specific `http.server.requests.active` family is disabled. The existing HTTP test
+now drives the actual observation handler, configured registry and metrics
+endpoint with both ordinary and exceptional observations. The author gate
+observed exactly the required role-specific families and four HTTP dimensions;
+ten missing UUIDs changed the selected404 count from2 to12 without new tag values.
+No exact unique-series count is inferred from the endpoint's available tag sets.
+The observer saves only whitelisted instrumentation names before asserting the
+inventory; unknown names remain in memory and are represented by a count.
+The frozen family/tag criteria have not been changed.
 
 API and worker structured request/process events use generated correlation IDs.
-They do not claim distributed CheckRun tracing. The failed run's cleanup scan
-matched8 observed response IDs, scanned19 runtime secret/sentinel values with
+They do not claim distributed CheckRun tracing. The passing author run's scan
+matched18 observed response IDs, scanned19 runtime secret/sentinel values with
 zero matches, and retained no raw runtime logs, bodies or credentials.
 
 `Dockerfile.api` runs the host/CI-built pinned Maven JAR. `Dockerfile.frontend`
@@ -530,20 +536,27 @@ fnm exec --using 24.19.0 node scripts/e24-container.mjs author
 ```
 
 The frozen scenario is2users/2Monitors/4seed results, one browser manual intent,
-one idle PostgreSQL stop/restore, and ten missing-UUID requests. Author attempt4
+one idle PostgreSQL stop/restore, and ten missing-UUID requests. Author attempt5
 verified authenticated owner-specific list/history, server HTML, JavaScript and
 CSS, a same-ID `SUCCEEDED/200` result, and the single outage/restore preserving
-exact authority. Its held request was explicitly released in15.899ms with no
-watchdog release. It stopped before the ten UUID requests and runtime UID/artifact
-inspection. Native143 exits were observed only during failed-gate cleanup; they
-do not substitute for a full passing container gate.
+exact authority. Its held request was explicitly released in13.711ms with no
+watchdog release; the immediate local observation/release path took6.367ms.
+API and worker ran the same packaged JAR as PID1/UID10001; Next's production
+standalone server ran as PID1/UID1000. Each application role received one SIGTERM
+and returned native143: frontend150.634ms, worker445.092ms and API310.039ms,
+all within the frozen5s bound. The backend-only rebuild reused the unchanged
+frontend image; the full build command above also supports a fresh approved setup.
 
 Cleanup closed Chromium, removed only the owned application containers/network
 and `e24_container` schema, and preserved PostgreSQL's container, image, volume
 and public data. No forced container kill, `down -v`, cache removal or shared-data
 cleanup is part of this observer. Safe evidence is under
-`evidence/phase-1/E24/container/` and `evidence/phase-1/E24/repair3/`; the latter
-contains the complete cumulative native invocation snapshot and source diagnosis.
+`evidence/phase-1/E24/container/` and `evidence/phase-1/E24/repair4/`; repair4
+contains the native Java/build/container bundles and the cumulative10-invocation
+snapshot. Prior `repair3/` evidence, including the failed inventory gate and its
+source-derived diagnosis, remains unchanged. Across all attempts there are
+five affected Maven invocations, two image builds, two author container scenarios
+and four formal failures. Root's independent runtime gate is still pending.
 
 CI has separate unit, integration, browser and container jobs. Its container job
 builds the actual images and invokes the same observer with actor `ci`; local
diff --git a/evidence/phase-1/E24/container/author-5464ab8650b03ea5.json b/evidence/phase-1/E24/container/author-5464ab8650b03ea5.json
new file mode 100644
index 0000000..e9a62fb
--- /dev/null
+++ b/evidence/phase-1/E24/container/author-5464ab8650b03ea5.json
@@ -0,0 +1,1787 @@
+{
+  "actor": "author",
+  "thread": "E24",
+  "profile": "phase-1",
+  "specRevision": "2ada57a71cd34fa2fae9809415c362a8bbfcdf02",
+  "start": "563b325ef871fe6d1fbfef7cf39a6581f2d0a94d",
+  "startedAt": "2026-08-28T11:05:08.268Z",
+  "result": "PASS",
+  "completed": [
+    "production roles ready; exact seed and idle metrics",
+    "one real browser intent; active held worker; released200; four-result history and reload",
+    "one idle PostgreSQL stop/restore;503 unsafe rejection; same authority and processes",
+    "actual bounded latency/error and worker meters; ten UUIDs keep tag sets unchanged",
+    "non-root PID1 artifacts; each native143 exit within5000ms; safe correlated structured logs"
+  ],
+  "invocations": {
+    "fullContainerScenario": 1,
+    "browser": 1,
+    "acceptedManualIntent": 1,
+    "rejectedOutageIntent": 1,
+    "postgresStop": 1,
+    "postgresRestore": 1,
+    "missingUuidRequests": 10,
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
+      "elapsedMs": 25.112583,
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
+      "elapsedMs": 18.37691699999999,
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
+      "elapsedMs": 17.634374999999977,
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
+      "elapsedMs": 19.043791,
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
+      "elapsedMs": 44.21291600000001,
+      "stdoutBytes": 6193,
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
+      "elapsedMs": 231.077334,
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
+      "elapsedMs": 18.614416000000006,
+      "stdoutBytes": 12844,
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
+      "elapsedMs": 786.5470420000001,
+      "stdoutBytes": 0,
+      "stderrBytes": 831,
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
+      "elapsedMs": 102.15845900000022,
+      "stdoutBytes": 52,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "182225b5363a"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 39.11604100000022,
+      "stdoutBytes": 8814,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "746586355e03"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 40.71445799999947,
+      "stdoutBytes": 9158,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "ac59c5d53a8a"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 36.395792000000256,
+      "stdoutBytes": 11267,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "6d28708a0dcf"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 44.734707999999955,
+      "stdoutBytes": 10610,
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
+      "elapsedMs": 45.68525000000045,
+      "stdoutBytes": 13,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "6d28708a0dcf717ab404d91f568f8b1111faca1d2fc47e026f2988da35672725"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 19.112832999999227,
+      "stdoutBytes": 10610,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "ac59c5d53a8a4163edf72421a2df7ee2c857d4d2a425f13fbb8a6b2c7c90a8d2"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 17.494457999999213,
+      "stdoutBytes": 11267,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "746586355e0310d76ede8eaa02168e7328eb580197063108a153a0b12e55f53a"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 16.778416999999536,
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
+      "elapsedMs": 261.21079199999986,
+      "stdoutBytes": 0,
+      "stderrBytes": 89,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "6d28708a0dcf717ab404d91f568f8b1111faca1d2fc47e026f2988da35672725"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 37.89466700000048,
+      "stdoutBytes": 10610,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "ac59c5d53a8a4163edf72421a2df7ee2c857d4d2a425f13fbb8a6b2c7c90a8d2"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 27.307833999999275,
+      "stdoutBytes": 11267,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "746586355e0310d76ede8eaa02168e7328eb580197063108a153a0b12e55f53a"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 21.947667000000365,
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
+      "elapsedMs": 153.81974999999875,
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
+      "elapsedMs": 22.018292000000656,
+      "stdoutBytes": 12844,
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
+      "elapsedMs": 25.167208999999275,
+      "stdoutBytes": 12844,
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
+      "elapsedMs": 27.25787499999933,
+      "stdoutBytes": 12844,
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
+      "elapsedMs": 25.05845799999952,
+      "stdoutBytes": 12844,
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
+      "elapsedMs": 27.202957999998034,
+      "stdoutBytes": 12844,
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
+      "elapsedMs": 30.372208000000683,
+      "stdoutBytes": 12844,
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
+      "elapsedMs": 29.76408299999821,
+      "stdoutBytes": 12844,
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
+      "elapsedMs": 23.341208000001643,
+      "stdoutBytes": 12844,
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
+      "elapsedMs": 25.846957999998267,
+      "stdoutBytes": 12930,
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
+      "elapsedMs": 26.161292000000685,
+      "stdoutBytes": 12842,
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
+      "elapsedMs": 23.887375000002066,
+      "stdoutBytes": 12842,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "6d28708a0dcf717ab404d91f568f8b1111faca1d2fc47e026f2988da35672725"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 18.07341699999961,
+      "stdoutBytes": 10610,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "ac59c5d53a8a4163edf72421a2df7ee2c857d4d2a425f13fbb8a6b2c7c90a8d2"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 21.393458999998984,
+      "stdoutBytes": 11267,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "746586355e0310d76ede8eaa02168e7328eb580197063108a153a0b12e55f53a"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 18.864832999999635,
+      "stdoutBytes": 9158,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "6d28708a0dcf717ab404d91f568f8b1111faca1d2fc47e026f2988da35672725"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 20.225666999998793,
+      "stdoutBytes": 10610,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "exec",
+        "6d28708a0dcf717ab404d91f568f8b1111faca1d2fc47e026f2988da35672725",
+        "cat",
+        "/proc/1/status"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 44.83062499999869,
+      "stdoutBytes": 1128,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "exec",
+        "6d28708a0dcf717ab404d91f568f8b1111faca1d2fc47e026f2988da35672725",
+        "java",
+        "-version"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 83.16204199999993,
+      "stdoutBytes": 0,
+      "stderrBytes": 190,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "exec",
+        "6d28708a0dcf717ab404d91f568f8b1111faca1d2fc47e026f2988da35672725",
+        "sha256sum",
+        "/app/app.jar"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 260.24675000000207,
+      "stdoutBytes": 79,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "ac59c5d53a8a4163edf72421a2df7ee2c857d4d2a425f13fbb8a6b2c7c90a8d2"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 20.387500000000728,
+      "stdoutBytes": 11267,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "exec",
+        "ac59c5d53a8a4163edf72421a2df7ee2c857d4d2a425f13fbb8a6b2c7c90a8d2",
+        "cat",
+        "/proc/1/status"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 64.43870900000184,
+      "stdoutBytes": 1128,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "exec",
+        "ac59c5d53a8a4163edf72421a2df7ee2c857d4d2a425f13fbb8a6b2c7c90a8d2",
+        "java",
+        "-version"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 87.0512499999968,
+      "stdoutBytes": 0,
+      "stderrBytes": 190,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "exec",
+        "ac59c5d53a8a4163edf72421a2df7ee2c857d4d2a425f13fbb8a6b2c7c90a8d2",
+        "sha256sum",
+        "/app/app.jar"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 246.38258300000234,
+      "stdoutBytes": 79,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "746586355e0310d76ede8eaa02168e7328eb580197063108a153a0b12e55f53a"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 17.982874999997875,
+      "stdoutBytes": 9158,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "exec",
+        "746586355e0310d76ede8eaa02168e7328eb580197063108a153a0b12e55f53a",
+        "cat",
+        "/proc/1/status"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 49.66837499999747,
+      "stdoutBytes": 1132,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "exec",
+        "746586355e0310d76ede8eaa02168e7328eb580197063108a153a0b12e55f53a",
+        "node",
+        "-e",
+        "console.log(JSON.stringify({node:process.versions.node,next:require(\"next/package.json\").version,react:require(\"react/package.json\").version,mode:process.env.NODE_ENV,origin:process.env.API_ORIGIN,manualSignal:!!process.env.NEXT_MANUAL_SIG_HANDLE}))"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 83.17229100000259,
+      "stdoutBytes": 120,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "746586355e0310d76ede8eaa02168e7328eb580197063108a153a0b12e55f53a"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 17.139999999999418,
+      "stdoutBytes": 9158,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "kill",
+        "--signal=SIGTERM",
+        "746586355e0310d76ede8eaa02168e7328eb580197063108a153a0b12e55f53a"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 26.766458000001876,
+      "stdoutBytes": 65,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "wait",
+        "746586355e0310d76ede8eaa02168e7328eb580197063108a153a0b12e55f53a"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 150.5955419999991,
+      "stdoutBytes": 4,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "ac59c5d53a8a4163edf72421a2df7ee2c857d4d2a425f13fbb8a6b2c7c90a8d2"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 22.34887499999968,
+      "stdoutBytes": 11267,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "kill",
+        "--signal=SIGTERM",
+        "ac59c5d53a8a4163edf72421a2df7ee2c857d4d2a425f13fbb8a6b2c7c90a8d2"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 23.439916999999696,
+      "stdoutBytes": 65,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "wait",
+        "ac59c5d53a8a4163edf72421a2df7ee2c857d4d2a425f13fbb8a6b2c7c90a8d2"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 445.06495899999936,
+      "stdoutBytes": 4,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "6d28708a0dcf717ab404d91f568f8b1111faca1d2fc47e026f2988da35672725"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 28.380291999998008,
+      "stdoutBytes": 10610,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "kill",
+        "--signal=SIGTERM",
+        "6d28708a0dcf717ab404d91f568f8b1111faca1d2fc47e026f2988da35672725"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 27.419708000001265,
+      "stdoutBytes": 65,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "wait",
+        "6d28708a0dcf717ab404d91f568f8b1111faca1d2fc47e026f2988da35672725"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 309.9940829999978,
+      "stdoutBytes": 4,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "182225b5363a217f6fdc9c2bb12f2913845fe260748f330b7e52a43a607e38fb"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 27.874416000002384,
+      "stdoutBytes": 8814,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "kill",
+        "--signal=SIGTERM",
+        "182225b5363a217f6fdc9c2bb12f2913845fe260748f330b7e52a43a607e38fb"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 24.580582999999024,
+      "stdoutBytes": 65,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "wait",
+        "182225b5363a217f6fdc9c2bb12f2913845fe260748f330b7e52a43a607e38fb"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 45.79500000000189,
+      "stdoutBytes": 4,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "logs",
+        "6d28708a0dcf717ab404d91f568f8b1111faca1d2fc47e026f2988da35672725"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 21.805583000001207,
+      "stdoutBytes": 47991,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "logs",
+        "ac59c5d53a8a4163edf72421a2df7ee2c857d4d2a425f13fbb8a6b2c7c90a8d2"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 20.40124999999898,
+      "stdoutBytes": 31600,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "logs",
+        "746586355e0310d76ede8eaa02168e7328eb580197063108a153a0b12e55f53a"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 20.60558300000048,
+      "stdoutBytes": 147,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "logs",
+        "182225b5363a217f6fdc9c2bb12f2913845fe260748f330b7e52a43a607e38fb"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 18.591915999997582,
+      "stdoutBytes": 0,
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
+      "elapsedMs": 30.938750000001164,
+      "stdoutBytes": 52,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "182225b5363a"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 17.92316700000083,
+      "stdoutBytes": 8822,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "746586355e03"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 20.181958999997732,
+      "stdoutBytes": 8771,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "ac59c5d53a8a"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 18.938791000000492,
+      "stdoutBytes": 10561,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "6d28708a0dcf"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 17.49658400000044,
+      "stdoutBytes": 10087,
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
+      "elapsedMs": 16.53216700000121,
+      "stdoutBytes": 13,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "182225b5363a217f6fdc9c2bb12f2913845fe260748f330b7e52a43a607e38fb"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 16.499750000002678,
+      "stdoutBytes": 8822,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "rm",
+        "182225b5363a217f6fdc9c2bb12f2913845fe260748f330b7e52a43a607e38fb"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 27.92279200000121,
+      "stdoutBytes": 65,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "746586355e0310d76ede8eaa02168e7328eb580197063108a153a0b12e55f53a"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 21.16437500000029,
+      "stdoutBytes": 8771,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "rm",
+        "746586355e0310d76ede8eaa02168e7328eb580197063108a153a0b12e55f53a"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 29.126541999998153,
+      "stdoutBytes": 65,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "ac59c5d53a8a4163edf72421a2df7ee2c857d4d2a425f13fbb8a6b2c7c90a8d2"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 23.223707999997714,
+      "stdoutBytes": 10561,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "rm",
+        "ac59c5d53a8a4163edf72421a2df7ee2c857d4d2a425f13fbb8a6b2c7c90a8d2"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 32.459249999999884,
+      "stdoutBytes": 65,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "6d28708a0dcf717ab404d91f568f8b1111faca1d2fc47e026f2988da35672725"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 19.73320899999817,
+      "stdoutBytes": 10087,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "rm",
+        "6d28708a0dcf717ab404d91f568f8b1111faca1d2fc47e026f2988da35672725"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 26.627416999999696,
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
+      "elapsedMs": 20.515666999999667,
+      "stdoutBytes": 1092,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "network",
+        "rm",
+        "8f7a69d3cd59a48cc561da8e173cc56bc8d685fd9d55e086c0fcb3b975833815"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 218.19920800000182,
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
+      "elapsedMs": 18.884624999998778,
+      "stdoutBytes": 12844,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    }
+  ],
+  "exits": {
+    "frontend": {
+      "purpose": "frozen production signal acceptance",
+      "signal": "SIGTERM",
+      "signalCommandExit": 0,
+      "exitCode": 143,
+      "elapsedMs": 150.63358399999925,
+      "awaited": true,
+      "observerTimedOut": false,
+      "forcedContainerKill": false
+    },
+    "worker": {
+      "purpose": "frozen production signal acceptance",
+      "signal": "SIGTERM",
+      "signalCommandExit": 0,
+      "exitCode": 143,
+      "elapsedMs": 445.09245799999917,
+      "awaited": true,
+      "observerTimedOut": false,
+      "forcedContainerKill": false
+    },
+    "api": {
+      "purpose": "frozen production signal acceptance",
+      "signal": "SIGTERM",
+      "signalCommandExit": 0,
+      "exitCode": 143,
+      "elapsedMs": 310.0386249999974,
+      "awaited": true,
+      "observerTimedOut": false,
+      "forcedContainerKill": false
+    },
+    "fixture": {
+      "purpose": "owned fixture cleanup",
+      "signal": "SIGTERM",
+      "signalCommandExit": 0,
+      "exitCode": 143,
+      "elapsedMs": 45.81941699999879,
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
+  "head": "c97c2320e5a0909036f8fa1e13c666cd58f0669e",
+  "sourceFingerprint": "5464ab8650b03ea5c281bd53d32940af8d72ce2fe1b1317c3e7df562725a4b27",
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
+      "localObservationAndReleaseMs": 6.366958000000523,
+      "dockerOrSqlBeforeRelease": false
+    },
+    "fixture": {
+      "outboundCalls": 1,
+      "holdRequests": 1,
+      "held": 0,
+      "releases": 1,
+      "watchdogReleases": 0,
+      "lastHeldMs": 13.711250000000291
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
+  "metricNameInventory": {
+    "api": {
+      "status": 200,
+      "validNameArray": true,
+      "nameCount": 2,
+      "names": [
+        "check.queue.age",
+        "http.server.requests"
+      ],
+      "unlistedNameCount": 0
+    },
+    "worker": {
+      "status": 200,
+      "validNameArray": true,
+      "nameCount": 4,
+      "names": [
+        "check.claims",
+        "check.queue.age",
+        "check.recoveries",
+        "check.worker.active"
+      ],
+      "unlistedNameCount": 0
+    }
+  },
+  "metrics": {
+    "apiQueueAge": 0,
+    "worker": {
+      "queueAge": 0,
+      "active": 0,
+      "claims": 1,
+      "recoveries": 0
+    },
+    "httpMeasurements": {
+      "COUNT": 55,
+      "TOTAL_TIME": 3.0125017969999996,
+      "MAX": 1.026114084
+    },
+    "boundedHttpTags": {
+      "method": [
+        "GET",
+        "POST"
+      ],
+      "process_role": [
+        "api"
+      ],
+      "status": [
+        "200",
+        "202",
+        "404",
+        "503"
+      ],
+      "uri": [
+        "/api/monitors",
+        "/api/monitors/{id}",
+        "/api/monitors/{id}/checks",
+        "/api/monitors/{id}/checks/{checkId}",
+        "/api/session/csrf",
+        "/api/session/login",
+        "/ops/health/**",
+        "/ops/metrics",
+        "/ops/metrics/{requiredMetricName}"
+      ]
+    },
+    "missingUuidRequests": 10,
+    "before404Count": 2,
+    "after404Count": 12,
+    "errorCountDelta": 10,
+    "tagSetsUnchanged": true,
+    "sourceNormalizationSha256": "f3a54450d30ecf7533ed5256f932c6eb161132df443b856de371cc45af20f88f",
+    "exactUniqueSeriesCountClaimed": false,
+    "positiveQueueAgeAndRecoveryProof": "separate affected OperationsIntegrationTest; no extra crash or outage scenario"
+  },
+  "runtime": {
+    "api": {
+      "id": "6d28708a0dcf717ab404d91f568f8b1111faca1d2fc47e026f2988da35672725",
+      "image": "sha256:d6f6ab129f0199ab6cb92ae12ee40fb83dba92bc40d9421a16f874b0265fbcc1",
+      "pid": 68781,
+      "startedAt": "2026-08-28T11:05:13.162688255Z",
+      "restartCount": 0,
+      "pidInsideContainer": 1,
+      "uid": 10001,
+      "command": [
+        "java",
+        "-jar",
+        "/app/app.jar"
+      ],
+      "versions": {
+        "java": "21.0.7+6",
+        "jarSha256": "2e7108c8ec7572131714be32de6c7f6da63c324a45c9d3fc76c39a280be728d8"
+      }
+    },
+    "worker": {
+      "id": "ac59c5d53a8a4163edf72421a2df7ee2c857d4d2a425f13fbb8a6b2c7c90a8d2",
+      "image": "sha256:d6f6ab129f0199ab6cb92ae12ee40fb83dba92bc40d9421a16f874b0265fbcc1",
+      "pid": 68782,
+      "startedAt": "2026-08-28T11:05:13.156771505Z",
+      "restartCount": 0,
+      "pidInsideContainer": 1,
+      "uid": 10001,
+      "command": [
+        "java",
+        "-jar",
+        "/app/app.jar",
+        "--spring.profiles.active=worker",
+        "--spring.main.web-application-type=none"
+      ],
+      "versions": {
+        "java": "21.0.7+6",
+        "jarSha256": "2e7108c8ec7572131714be32de6c7f6da63c324a45c9d3fc76c39a280be728d8"
+      }
+    },
+    "frontend": {
+      "id": "746586355e0310d76ede8eaa02168e7328eb580197063108a153a0b12e55f53a",
+      "image": "sha256:9356a3b65bd58be8e12f5a4721975f4801e1601f7dbef0a4bb7ddfd6ef93ac9e",
+      "pid": 68780,
+      "startedAt": "2026-08-28T11:05:13.162191589Z",
+      "restartCount": 0,
+      "pidInsideContainer": 1,
+      "uid": 1000,
+      "command": [
+        "node",
+        "server.js"
+      ],
+      "versions": {
+        "node": "24.19.0",
+        "next": "16.3.3",
+        "react": "19.2.8",
+        "mode": "production",
+        "origin": "http://api:4322",
+        "manualSignal": false
+      }
+    }
+  },
+  "logs": {
+    "roles": {
+      "api": {
+        "lines": 104,
+        "structuredLines": 104,
+        "observationEvents": 56
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
+    "responseCorrelations": 18,
+    "matchedResponseCorrelations": 18,
+    "generatedResponseRequestIds": [
+      "1a86653a-4074-45af-b6b1-cd9c46e249df",
+      "7bce98f8-4ff6-41bf-82c5-fc6bb3ae9910",
+      "8b0eb817-02b6-4257-9bd3-31a000c5755f",
+      "640f1963-fc28-4ff7-8429-113bc6347eb4",
+      "ad9760b3-47f8-499b-ab51-1849a35aa8d5",
+      "08763de1-7628-433f-97f1-4cf6e78b0f4b",
+      "0a094451-07b1-47d5-a301-fd779095a37e",
+      "54342522-bddb-4dc6-8816-ce448ad7f14d",
+      "d774945d-dab7-47a6-a086-e3fa04857047",
+      "b2c17ebf-ffed-4c2f-a5ef-9eead046b4f7",
+      "a4c50240-d8b4-4f17-818e-46fbf3e8d0e9",
+      "8f9953fd-8e9c-4107-9a1b-f52cb44de02a",
+      "689577b5-190a-4030-91f1-51ea015cd336",
+      "b5e0d24b-ab62-47d7-b30d-178fffc7a129",
+      "8951ae58-7344-4543-a1ad-45fcc0e89397",
+      "b0d6ce63-81e8-4b00-acc4-5db49ef06e9b",
+      "9b3950ee-b22f-444e-a42e-3a2312c2f6e7",
+      "60e9fc81-c095-4a40-bc4f-ee015b6d0bd8"
+    ],
+    "stableApiProcess": true,
+    "stableWorkerProcess": true,
+    "distinctRoleProcesses": true,
+    "apiProcessId": "ebd69ae2-d1dc-4aee-af8c-fc677ecfb8b3",
+    "workerProcessId": "5e678e3a-b8d5-4e3c-841d-9d1b0c3cb77a",
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
+  "elapsedSeconds": 21.791,
+  "hostedCiExecutionClaimed": false
+}
diff --git a/evidence/phase-1/E24/container/author-5464ab8650b03ea5.started.json b/evidence/phase-1/E24/container/author-5464ab8650b03ea5.started.json
new file mode 100644
index 0000000..ba1d933
--- /dev/null
+++ b/evidence/phase-1/E24/container/author-5464ab8650b03ea5.started.json
@@ -0,0 +1 @@
+{"actor":"author","head":"c97c2320e5a0909036f8fa1e13c666cd58f0669e","fingerprint":"5464ab8650b03ea5c281bd53d32940af8d72ce2fe1b1317c3e7df562725a4b27","startedAt":"2026-08-28T11:05:08.268Z"}
diff --git a/evidence/phase-1/E24/container/invocations.jsonl b/evidence/phase-1/E24/container/invocations.jsonl
index 773e611..5f7615e 100644
--- a/evidence/phase-1/E24/container/invocations.jsonl
+++ b/evidence/phase-1/E24/container/invocations.jsonl
@@ -1 +1,2 @@
 {"actor":"author","head":"2dbe240409db95ac0435f8a4d4270189f60b7d82","fingerprint":"ff11785dd9b6da221b6b863102b24e38cf028f3b71790d5f8c787642f5603610","startedAt":"2026-08-28T10:43:01.107Z","command":"node scripts/e24-container.mjs author"}
+{"actor":"author","head":"c97c2320e5a0909036f8fa1e13c666cd58f0669e","fingerprint":"5464ab8650b03ea5c281bd53d32940af8d72ce2fe1b1317c3e7df562725a4b27","startedAt":"2026-08-28T11:05:08.268Z","command":"node scripts/e24-container.mjs author"}
diff --git a/evidence/phase-1/E24/repair4/cumulative-invocations.jsonl b/evidence/phase-1/E24/repair4/cumulative-invocations.jsonl
new file mode 100644
index 0000000..6b7b135
--- /dev/null
+++ b/evidence/phase-1/E24/repair4/cumulative-invocations.jsonl
@@ -0,0 +1,10 @@
+{"command":"node scripts/e24-baseline.mjs","startedAt":"2026-08-28T08:51:05.183Z","elapsedSeconds":7.477,"exitCode":0}
+{"command":"mvn -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package","startedAt":"2026-08-28T09:11:45.983Z","elapsedSeconds":1.724,"exitCode":1,"signal":null}
+{"command":"mvn -U -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package","startedAt":"2026-08-28T09:22:21.048583+00:00","attempt":2,"repair":1,"actor":"author","permission":"require_escalated network/cache authorization","forcedDescriptorRefreshes":1,"elapsedSeconds":4.051,"exitCode":1,"signal":null}
+{"command":"mvn -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package","startedAt":"2026-08-28T09:31:22.232667+00:00","attempt":3,"repair":2,"actor":"author","permission":"require_escalated network/cache/PostgreSQL authorization","forcedDescriptorRefreshes":0,"elapsedSeconds":6.999,"exitCode":1,"signal":null}
+{"command":"mvn -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package","startedAt":"2026-08-28T10:35:41.429288+00:00","attempt":4,"repair":3,"actor":"author","permission":"require_escalated Java/Maven/cache/network/PostgreSQL authorization","forcedDescriptorRefreshes":0,"elapsedSeconds":8.772,"exitCode":0,"signal":null,"nativeBundle":"evidence/phase-1/E24/repair3/repair3-affected-java-native.json"}
+{"command":"DB_SCHEMA=e24_container docker compose -p wse-industry-e24 -f compose.production.yaml build api frontend","startedAt":"2026-08-28T10:39:20.133345+00:00","attempt":4,"repair":3,"actor":"author","permission":"require_escalated pinned production image build/cache/network authorization","elapsedSeconds":48.361,"exitCode":0,"signal":null}
+{"command":"fnm exec --using 24.19.0 node scripts/e24-container.mjs author","startedAt":"2026-08-28T10:43:00.859467+00:00","attempt":4,"repair":3,"actor":"author","permission":"require_escalated actual Docker/PostgreSQL/browser/network authorization","elapsedSeconds":24.842,"exitCode":1,"signal":null,"rawDiagnosticOutputRetained":false}
+{"command":"mvn -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package","startedAt":"2026-08-28T11:02:28.242864+00:00","attempt":5,"repair":4,"actor":"author","permission":"require_escalated pinned Java/Maven/cache/network/PostgreSQL authorization","forcedDescriptorRefreshes":0,"elapsedSeconds":9.621,"exitCode":0,"signal":null,"nativeBundle":"evidence/phase-1/E24/repair4/repair4-affected-java-native.json"}
+{"command":"DB_SCHEMA=e24_container docker compose -p wse-industry-e24 -f compose.production.yaml build api","startedAt":"2026-08-28T11:03:32.495550+00:00","attempt":5,"repair":4,"actor":"author","permission":"require_escalated pinned backend image build/cache/network authorization","elapsedSeconds":4.054,"exitCode":0,"signal":null,"nativeBundle":"evidence/phase-1/E24/repair4/repair4-image-build-native.json"}
+{"command":"fnm exec --using 24.19.0 node scripts/e24-container.mjs author","startedAt":"2026-08-28T11:05:08.079267+00:00","attempt":5,"repair":4,"actor":"author","permission":"require_escalated actual Docker/PostgreSQL/browser/network authorization","elapsedSeconds":21.991,"exitCode":0,"signal":null,"rawDiagnosticOutputRetained":false,"nativeBundle":"evidence/phase-1/E24/repair4/repair4-author-container-native.json"}
diff --git a/evidence/phase-1/E24/repair4/repair4-author-container-native.json b/evidence/phase-1/E24/repair4/repair4-author-container-native.json
new file mode 100644
index 0000000..7fd3759
--- /dev/null
+++ b/evidence/phase-1/E24/repair4/repair4-author-container-native.json
@@ -0,0 +1,131 @@
+{
+  "thread": "E24",
+  "profile": "phase-1",
+  "attempt": 5,
+  "repair": 4,
+  "actor": "author",
+  "result": "PASS",
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
+    "startedAt": "2026-08-28T11:05:08.079267+00:00",
+    "attempt": 5,
+    "repair": 4,
+    "actor": "author",
+    "permission": "require_escalated actual Docker/PostgreSQL/browser/network authorization",
+    "authorContainerScenarioInvocationThisRepair": 1,
+    "automaticRetries": 0,
+    "observerSha256": "aa444c00ed2054557177e2d10fa8f5aac6dfb54faeaabf144d683cf09cc776e8",
+    "authorizationSha256": "831b8dd80b373c90efa0597416b23c2dfed3e36b4ab040d7a730f31ce4acf025"
+  },
+  "nativeResult": {
+    "exitCode": 0,
+    "elapsedSeconds": 21.991,
+    "startedAt": "2026-08-28T11:05:08.079267+00:00",
+    "endedAt": "2026-08-28T11:05:30.071869+00:00",
+    "signal": null,
+    "safeObserverSummary": {
+      "actor": "author",
+      "result": "PASS",
+      "phase": "completed",
+      "evidence": "evidence/phase-1/E24/container/author-5464ab8650b03ea5.json",
+      "elapsedSeconds": 21.791,
+      "cleanupFailures": 0
+    },
+    "stdoutBytes": 172,
+    "stderrBytes": 0,
+    "rawDiagnosticOutputRetained": false
+  },
+  "cumulativeLedger": {
+    "previousBytes": 3085,
+    "previousSha256": "09b3005f21a6dbdb72ad0714eb490d8da1636f1ee180a233a74b65a6f9835217",
+    "previousPrefixPreserved": true,
+    "currentEntries": 10,
+    "currentSha256": "f038964ccef2b1008275ea85e1204f1f0aed8db997759b6897c4b5d2a9937d86"
+  },
+  "actorLedger": {
+    "previousBytes": 239,
+    "previousSha256": "344c93072897b713765ed5504655ea7f6e45c2d1a1e1e8d7603e6cf83d4941b7",
+    "previousPrefixPreserved": true,
+    "currentEntries": 2
+  },
+  "cumulativeBudget": {
+    "baselineInvocations": 1,
+    "expectedBaselineCounterexamples": 1,
+    "baselineWallSeconds": 7.477,
+    "affectedMavenInvocations": 5,
+    "affectedMavenWallSeconds": 31.167,
+    "javaTestsExecuted": 16,
+    "javaTestsPassed": 15,
+    "javaTestErrors": 1,
+    "javaAssertionFailures": 0,
+    "javaTestsSkipped": 0,
+    "unexpectedFormalFailures": 4,
+    "repairTasksUsed": 4,
+    "maximumFreshRepairs": null,
+    "historicalMaximumFreshRepairs": 2,
+    "forcedDescriptorRefreshes": 1,
+    "imageBuildInvocations": 2,
+    "imageBuildWallSeconds": 52.415,
+    "containerScenarios": 2,
+    "containerScenarioWallSeconds": 46.833,
+    "browserInvocations": 2,
+    "postgresStopRestoreSequences": 2,
+    "rootRuntimeProductGateRuns": 0,
+    "cumulativeGateWallSeconds": 137.892,
+    "loadRuns": 0,
+    "parameterSweeps": 0,
+    "automaticRetries": 0,
+    "e11KillScenarioRuns": 0,
+    "e20DatasetRuns": 0
+  },
+  "rawRuntimeLogsOrResponsesRetained": false,
+  "candidateSourceChanged": false,
+  "independentRootFinalGate": "pending",
+  "nativeFiles": [
+    {
+      "source": "output/phase-1/e24/repair4-author-container.started.json",
+      "sha256": "584a490c1f1e9798ed27b639a5b67f5157cb19acc8dfa50ed9f3b1010a8406fa",
+      "bytes": 673,
+      "raw": "{\n  \"command\": [\n    \"fnm\",\n    \"exec\",\n    \"--using\",\n    \"24.19.0\",\n    \"node\",\n    \"scripts/e24-container.mjs\",\n    \"author\"\n  ],\n  \"displayCommand\": \"fnm exec --using 24.19.0 node scripts/e24-container.mjs author\",\n  \"startedAt\": \"2026-08-28T11:05:08.079267+00:00\",\n  \"attempt\": 5,\n  \"repair\": 4,\n  \"actor\": \"author\",\n  \"permission\": \"require_escalated actual Docker/PostgreSQL/browser/network authorization\",\n  \"authorContainerScenarioInvocationThisRepair\": 1,\n  \"automaticRetries\": 0,\n  \"observerSha256\": \"aa444c00ed2054557177e2d10fa8f5aac6dfb54faeaabf144d683cf09cc776e8\",\n  \"authorizationSha256\": \"831b8dd80b373c90efa0597416b23c2dfed3e36b4ab040d7a730f31ce4acf025\"\n}\n"
+    },
+    {
+      "source": "output/phase-1/e24/repair4-author-container.result.json",
+      "sha256": "845ed9426e46061eea4efaddf4cc153f2679e93251fb983b66765e1662f94a2a",
+      "bytes": 485,
+      "raw": "{\n  \"exitCode\": 0,\n  \"elapsedSeconds\": 21.991,\n  \"startedAt\": \"2026-08-28T11:05:08.079267+00:00\",\n  \"endedAt\": \"2026-08-28T11:05:30.071869+00:00\",\n  \"signal\": null,\n  \"safeObserverSummary\": {\n    \"actor\": \"author\",\n    \"result\": \"PASS\",\n    \"phase\": \"completed\",\n    \"evidence\": \"evidence/phase-1/E24/container/author-5464ab8650b03ea5.json\",\n    \"elapsedSeconds\": 21.791,\n    \"cleanupFailures\": 0\n  },\n  \"stdoutBytes\": 172,\n  \"stderrBytes\": 0,\n  \"rawDiagnosticOutputRetained\": false\n}\n"
+    },
+    {
+      "source": "output/phase-1/e24/invocations.jsonl",
+      "sha256": "f038964ccef2b1008275ea85e1204f1f0aed8db997759b6897c4b5d2a9937d86",
+      "bytes": 3507,
+      "raw": "{\"command\":\"node scripts/e24-baseline.mjs\",\"startedAt\":\"2026-08-28T08:51:05.183Z\",\"elapsedSeconds\":7.477,\"exitCode\":0}\n{\"command\":\"mvn -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package\",\"startedAt\":\"2026-08-28T09:11:45.983Z\",\"elapsedSeconds\":1.724,\"exitCode\":1,\"signal\":null}\n{\"command\":\"mvn -U -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package\",\"startedAt\":\"2026-08-28T09:22:21.048583+00:00\",\"attempt\":2,\"repair\":1,\"actor\":\"author\",\"permission\":\"require_escalated network/cache authorization\",\"forcedDescriptorRefreshes\":1,\"elapsedSeconds\":4.051,\"exitCode\":1,\"signal\":null}\n{\"command\":\"mvn -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package\",\"startedAt\":\"2026-08-28T09:31:22.232667+00:00\",\"attempt\":3,\"repair\":2,\"actor\":\"author\",\"permission\":\"require_escalated network/cache/PostgreSQL authorization\",\"forcedDescriptorRefreshes\":0,\"elapsedSeconds\":6.999,\"exitCode\":1,\"signal\":null}\n{\"command\":\"mvn -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package\",\"startedAt\":\"2026-08-28T10:35:41.429288+00:00\",\"attempt\":4,\"repair\":3,\"actor\":\"author\",\"permission\":\"require_escalated Java/Maven/cache/network/PostgreSQL authorization\",\"forcedDescriptorRefreshes\":0,\"elapsedSeconds\":8.772,\"exitCode\":0,\"signal\":null,\"nativeBundle\":\"evidence/phase-1/E24/repair3/repair3-affected-java-native.json\"}\n{\"command\":\"DB_SCHEMA=e24_container docker compose -p wse-industry-e24 -f compose.production.yaml build api frontend\",\"startedAt\":\"2026-08-28T10:39:20.133345+00:00\",\"attempt\":4,\"repair\":3,\"actor\":\"author\",\"permission\":\"require_escalated pinned production image build/cache/network authorization\",\"elapsedSeconds\":48.361,\"exitCode\":0,\"signal\":null}\n{\"command\":\"fnm exec --using 24.19.0 node scripts/e24-container.mjs author\",\"startedAt\":\"2026-08-28T10:43:00.859467+00:00\",\"attempt\":4,\"repair\":3,\"actor\":\"author\",\"permission\":\"require_escalated actual Docker/PostgreSQL/browser/network authorization\",\"elapsedSeconds\":24.842,\"exitCode\":1,\"signal\":null,\"rawDiagnosticOutputRetained\":false}\n{\"command\":\"mvn -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package\",\"startedAt\":\"2026-08-28T11:02:28.242864+00:00\",\"attempt\":5,\"repair\":4,\"actor\":\"author\",\"permission\":\"require_escalated pinned Java/Maven/cache/network/PostgreSQL authorization\",\"forcedDescriptorRefreshes\":0,\"elapsedSeconds\":9.621,\"exitCode\":0,\"signal\":null,\"nativeBundle\":\"evidence/phase-1/E24/repair4/repair4-affected-java-native.json\"}\n{\"command\":\"DB_SCHEMA=e24_container docker compose -p wse-industry-e24 -f compose.production.yaml build api\",\"startedAt\":\"2026-08-28T11:03:32.495550+00:00\",\"attempt\":5,\"repair\":4,\"actor\":\"author\",\"permission\":\"require_escalated pinned backend image build/cache/network authorization\",\"elapsedSeconds\":4.054,\"exitCode\":0,\"signal\":null,\"nativeBundle\":\"evidence/phase-1/E24/repair4/repair4-image-build-native.json\"}\n{\"command\":\"fnm exec --using 24.19.0 node scripts/e24-container.mjs author\",\"startedAt\":\"2026-08-28T11:05:08.079267+00:00\",\"attempt\":5,\"repair\":4,\"actor\":\"author\",\"permission\":\"require_escalated actual Docker/PostgreSQL/browser/network authorization\",\"elapsedSeconds\":21.991,\"exitCode\":0,\"signal\":null,\"rawDiagnosticOutputRetained\":false,\"nativeBundle\":\"evidence/phase-1/E24/repair4/repair4-author-container-native.json\"}\n"
+    },
+    {
+      "source": "evidence/phase-1/E24/container/invocations.jsonl",
+      "sha256": "c698afd0e0312c0ad80929679c048bbd22c1578a3a77d1df6355e3672a9b7cb3",
+      "bytes": 478,
+      "raw": "{\"actor\":\"author\",\"head\":\"2dbe240409db95ac0435f8a4d4270189f60b7d82\",\"fingerprint\":\"ff11785dd9b6da221b6b863102b24e38cf028f3b71790d5f8c787642f5603610\",\"startedAt\":\"2026-08-28T10:43:01.107Z\",\"command\":\"node scripts/e24-container.mjs author\"}\n{\"actor\":\"author\",\"head\":\"c97c2320e5a0909036f8fa1e13c666cd58f0669e\",\"fingerprint\":\"5464ab8650b03ea5c281bd53d32940af8d72ce2fe1b1317c3e7df562725a4b27\",\"startedAt\":\"2026-08-28T11:05:08.268Z\",\"command\":\"node scripts/e24-container.mjs author\"}\n"
+    },
+    {
+      "source": "evidence/phase-1/E24/container/author-5464ab8650b03ea5.json",
+      "sha256": "e2af8cae161592bf031fbef5e0c0dd3eaf4453fdff83fea531f236d36ab040ec",
+      "bytes": 44432,
+      "raw": "{\n  \"actor\": \"author\",\n  \"thread\": \"E24\",\n  \"profile\": \"phase-1\",\n  \"specRevision\": \"2ada57a71cd34fa2fae9809415c362a8bbfcdf02\",\n  \"start\": \"563b325ef871fe6d1fbfef7cf39a6581f2d0a94d\",\n  \"startedAt\": \"2026-08-28T11:05:08.268Z\",\n  \"result\": \"PASS\",\n  \"completed\": [\n    \"production roles ready; exact seed and idle metrics\",\n    \"one real browser intent; active held worker; released200; four-result history and reload\",\n    \"one idle PostgreSQL stop/restore;503 unsafe rejection; same authority and processes\",\n    \"actual bounded latency/error and worker meters; ten UUIDs keep tag sets unchanged\",\n    \"non-root PID1 artifacts; each native143 exit within5000ms; safe correlated structured logs\"\n  ],\n  \"invocations\": {\n    \"fullContainerScenario\": 1,\n    \"browser\": 1,\n    \"acceptedManualIntent\": 1,\n    \"rejectedOutageIntent\": 1,\n    \"postgresStop\": 1,\n    \"postgresRestore\": 1,\n    \"missingUuidRequests\": 10,\n    \"baseline\": 0,\n    \"maven\": 0,\n    \"imageBuild\": 0,\n    \"load\": 0,\n    \"parameterSweep\": 0,\n    \"automaticRetry\": 0\n  },\n  \"nativeCommands\": [\n    {\n      \"command\": [\n        \"docker\",\n        \"ps\",\n        \"-aq\",\n        \"--filter\",\n        \"label=com.docker.compose.project=wse-industry-e24\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 25.112583,\n      \"stdoutBytes\": 0,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"network\",\n        \"ls\",\n        \"-q\",\n        \"--filter\",\n        \"label=com.docker.compose.project=wse-industry-e24\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 18.37691699999999,\n      \"stdoutBytes\": 0,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"network\",\n        \"ls\",\n        \"--format\",\n        \"{{.Name}}\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 17.634374999999977,\n      \"stdoutBytes\": 64,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"volume\",\n        \"ls\",\n        \"-q\",\n        \"--filter\",\n        \"label=com.docker.compose.project=wse-industry-e24\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 19.043791,\n      \"stdoutBytes\": 0,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"image\",\n        \"inspect\",\n        \"wse-industry-e24-api:local\",\n        \"wse-industry-e24-frontend:local\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 44.21291600000001,\n      \"stdoutBytes\": 6193,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"compose\",\n        \"-p\",\n        \"wse-industry\",\n        \"-f\",\n        \"compose.yaml\",\n        \"ps\",\n        \"--all\",\n        \"--quiet\",\n        \"postgres\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 231.077334,\n      \"stdoutBytes\": 65,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 18.614416000000006,\n      \"stdoutBytes\": 12844,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"compose\",\n        \"-p\",\n        \"wse-industry-e24\",\n        \"-f\",\n        \"compose.production.yaml\",\n        \"-f\",\n        \"compose.e24.yaml\",\n        \"up\",\n        \"--detach\",\n        \"--no-build\",\n        \"api\",\n        \"worker\",\n        \"frontend\",\n        \"fixture\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 786.5470420000001,\n      \"stdoutBytes\": 0,\n      \"stderrBytes\": 831,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"ps\",\n        \"-aq\",\n        \"--filter\",\n        \"label=com.docker.compose.project=wse-industry-e24\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 102.15845900000022,\n      \"stdoutBytes\": 52,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"182225b5363a\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 39.11604100000022,\n      \"stdoutBytes\": 8814,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"746586355e03\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 40.71445799999947,\n      \"stdoutBytes\": 9158,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"ac59c5d53a8a\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 36.395792000000256,\n      \"stdoutBytes\": 11267,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"6d28708a0dcf\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 44.734707999999955,\n      \"stdoutBytes\": 10610,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"network\",\n        \"ls\",\n        \"-q\",\n        \"--filter\",\n        \"label=com.docker.compose.project=wse-industry-e24\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 45.68525000000045,\n      \"stdoutBytes\": 13,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"6d28708a0dcf717ab404d91f568f8b1111faca1d2fc47e026f2988da35672725\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 19.112832999999227,\n      \"stdoutBytes\": 10610,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"ac59c5d53a8a4163edf72421a2df7ee2c857d4d2a425f13fbb8a6b2c7c90a8d2\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 17.494457999999213,\n      \"stdoutBytes\": 11267,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"746586355e0310d76ede8eaa02168e7328eb580197063108a153a0b12e55f53a\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 16.778416999999536,\n      \"stdoutBytes\": 9158,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"compose\",\n        \"-p\",\n        \"wse-industry\",\n        \"-f\",\n        \"compose.yaml\",\n        \"stop\",\n        \"--timeout\",\n        \"5\",\n        \"postgres\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 261.21079199999986,\n      \"stdoutBytes\": 0,\n      \"stderrBytes\": 89,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"6d28708a0dcf717ab404d91f568f8b1111faca1d2fc47e026f2988da35672725\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 37.89466700000048,\n      \"stdoutBytes\": 10610,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"ac59c5d53a8a4163edf72421a2df7ee2c857d4d2a425f13fbb8a6b2c7c90a8d2\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 27.307833999999275,\n      \"stdoutBytes\": 11267,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"746586355e0310d76ede8eaa02168e7328eb580197063108a153a0b12e55f53a\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 21.947667000000365,\n      \"stdoutBytes\": 9158,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"compose\",\n        \"-p\",\n        \"wse-industry\",\n        \"-f\",\n        \"compose.yaml\",\n        \"start\",\n        \"postgres\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 153.81974999999875,\n      \"stdoutBytes\": 0,\n      \"stderrBytes\": 89,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 22.018292000000656,\n      \"stdoutBytes\": 12844,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 25.167208999999275,\n      \"stdoutBytes\": 12844,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 27.25787499999933,\n      \"stdoutBytes\": 12844,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 25.05845799999952,\n      \"stdoutBytes\": 12844,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 27.202957999998034,\n      \"stdoutBytes\": 12844,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 30.372208000000683,\n      \"stdoutBytes\": 12844,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 29.76408299999821,\n      \"stdoutBytes\": 12844,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 23.341208000001643,\n      \"stdoutBytes\": 12844,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 25.846957999998267,\n      \"stdoutBytes\": 12930,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 26.161292000000685,\n      \"stdoutBytes\": 12842,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 23.887375000002066,\n      \"stdoutBytes\": 12842,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"6d28708a0dcf717ab404d91f568f8b1111faca1d2fc47e026f2988da35672725\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 18.07341699999961,\n      \"stdoutBytes\": 10610,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"ac59c5d53a8a4163edf72421a2df7ee2c857d4d2a425f13fbb8a6b2c7c90a8d2\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 21.393458999998984,\n      \"stdoutBytes\": 11267,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"746586355e0310d76ede8eaa02168e7328eb580197063108a153a0b12e55f53a\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 18.864832999999635,\n      \"stdoutBytes\": 9158,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"6d28708a0dcf717ab404d91f568f8b1111faca1d2fc47e026f2988da35672725\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 20.225666999998793,\n      \"stdoutBytes\": 10610,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"exec\",\n        \"6d28708a0dcf717ab404d91f568f8b1111faca1d2fc47e026f2988da35672725\",\n        \"cat\",\n        \"/proc/1/status\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 44.83062499999869,\n      \"stdoutBytes\": 1128,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"exec\",\n        \"6d28708a0dcf717ab404d91f568f8b1111faca1d2fc47e026f2988da35672725\",\n        \"java\",\n        \"-version\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 83.16204199999993,\n      \"stdoutBytes\": 0,\n      \"stderrBytes\": 190,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"exec\",\n        \"6d28708a0dcf717ab404d91f568f8b1111faca1d2fc47e026f2988da35672725\",\n        \"sha256sum\",\n        \"/app/app.jar\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 260.24675000000207,\n      \"stdoutBytes\": 79,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"ac59c5d53a8a4163edf72421a2df7ee2c857d4d2a425f13fbb8a6b2c7c90a8d2\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 20.387500000000728,\n      \"stdoutBytes\": 11267,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"exec\",\n        \"ac59c5d53a8a4163edf72421a2df7ee2c857d4d2a425f13fbb8a6b2c7c90a8d2\",\n        \"cat\",\n        \"/proc/1/status\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 64.43870900000184,\n      \"stdoutBytes\": 1128,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"exec\",\n        \"ac59c5d53a8a4163edf72421a2df7ee2c857d4d2a425f13fbb8a6b2c7c90a8d2\",\n        \"java\",\n        \"-version\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 87.0512499999968,\n      \"stdoutBytes\": 0,\n      \"stderrBytes\": 190,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"exec\",\n        \"ac59c5d53a8a4163edf72421a2df7ee2c857d4d2a425f13fbb8a6b2c7c90a8d2\",\n        \"sha256sum\",\n        \"/app/app.jar\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 246.38258300000234,\n      \"stdoutBytes\": 79,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"746586355e0310d76ede8eaa02168e7328eb580197063108a153a0b12e55f53a\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 17.982874999997875,\n      \"stdoutBytes\": 9158,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"exec\",\n        \"746586355e0310d76ede8eaa02168e7328eb580197063108a153a0b12e55f53a\",\n        \"cat\",\n        \"/proc/1/status\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 49.66837499999747,\n      \"stdoutBytes\": 1132,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"exec\",\n        \"746586355e0310d76ede8eaa02168e7328eb580197063108a153a0b12e55f53a\",\n        \"node\",\n        \"-e\",\n        \"console.log(JSON.stringify({node:process.versions.node,next:require(\\\"next/package.json\\\").version,react:require(\\\"react/package.json\\\").version,mode:process.env.NODE_ENV,origin:process.env.API_ORIGIN,manualSignal:!!process.env.NEXT_MANUAL_SIG_HANDLE}))\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 83.17229100000259,\n      \"stdoutBytes\": 120,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"746586355e0310d76ede8eaa02168e7328eb580197063108a153a0b12e55f53a\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 17.139999999999418,\n      \"stdoutBytes\": 9158,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"kill\",\n        \"--signal=SIGTERM\",\n        \"746586355e0310d76ede8eaa02168e7328eb580197063108a153a0b12e55f53a\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 26.766458000001876,\n      \"stdoutBytes\": 65,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"wait\",\n        \"746586355e0310d76ede8eaa02168e7328eb580197063108a153a0b12e55f53a\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 150.5955419999991,\n      \"stdoutBytes\": 4,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"ac59c5d53a8a4163edf72421a2df7ee2c857d4d2a425f13fbb8a6b2c7c90a8d2\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 22.34887499999968,\n      \"stdoutBytes\": 11267,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"kill\",\n        \"--signal=SIGTERM\",\n        \"ac59c5d53a8a4163edf72421a2df7ee2c857d4d2a425f13fbb8a6b2c7c90a8d2\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 23.439916999999696,\n      \"stdoutBytes\": 65,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"wait\",\n        \"ac59c5d53a8a4163edf72421a2df7ee2c857d4d2a425f13fbb8a6b2c7c90a8d2\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 445.06495899999936,\n      \"stdoutBytes\": 4,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"6d28708a0dcf717ab404d91f568f8b1111faca1d2fc47e026f2988da35672725\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 28.380291999998008,\n      \"stdoutBytes\": 10610,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"kill\",\n        \"--signal=SIGTERM\",\n        \"6d28708a0dcf717ab404d91f568f8b1111faca1d2fc47e026f2988da35672725\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 27.419708000001265,\n      \"stdoutBytes\": 65,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"wait\",\n        \"6d28708a0dcf717ab404d91f568f8b1111faca1d2fc47e026f2988da35672725\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 309.9940829999978,\n      \"stdoutBytes\": 4,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"182225b5363a217f6fdc9c2bb12f2913845fe260748f330b7e52a43a607e38fb\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 27.874416000002384,\n      \"stdoutBytes\": 8814,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"kill\",\n        \"--signal=SIGTERM\",\n        \"182225b5363a217f6fdc9c2bb12f2913845fe260748f330b7e52a43a607e38fb\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 24.580582999999024,\n      \"stdoutBytes\": 65,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"wait\",\n        \"182225b5363a217f6fdc9c2bb12f2913845fe260748f330b7e52a43a607e38fb\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 45.79500000000189,\n      \"stdoutBytes\": 4,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"logs\",\n        \"6d28708a0dcf717ab404d91f568f8b1111faca1d2fc47e026f2988da35672725\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 21.805583000001207,\n      \"stdoutBytes\": 47991,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"logs\",\n        \"ac59c5d53a8a4163edf72421a2df7ee2c857d4d2a425f13fbb8a6b2c7c90a8d2\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 20.40124999999898,\n      \"stdoutBytes\": 31600,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"logs\",\n        \"746586355e0310d76ede8eaa02168e7328eb580197063108a153a0b12e55f53a\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 20.60558300000048,\n      \"stdoutBytes\": 147,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"logs\",\n        \"182225b5363a217f6fdc9c2bb12f2913845fe260748f330b7e52a43a607e38fb\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 18.591915999997582,\n      \"stdoutBytes\": 0,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"ps\",\n        \"-aq\",\n        \"--filter\",\n        \"label=com.docker.compose.project=wse-industry-e24\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 30.938750000001164,\n      \"stdoutBytes\": 52,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"182225b5363a\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 17.92316700000083,\n      \"stdoutBytes\": 8822,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"746586355e03\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 20.181958999997732,\n      \"stdoutBytes\": 8771,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"ac59c5d53a8a\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 18.938791000000492,\n      \"stdoutBytes\": 10561,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"6d28708a0dcf\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 17.49658400000044,\n      \"stdoutBytes\": 10087,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"network\",\n        \"ls\",\n        \"-q\",\n        \"--filter\",\n        \"label=com.docker.compose.project=wse-industry-e24\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 16.53216700000121,\n      \"stdoutBytes\": 13,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"182225b5363a217f6fdc9c2bb12f2913845fe260748f330b7e52a43a607e38fb\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 16.499750000002678,\n      \"stdoutBytes\": 8822,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"rm\",\n        \"182225b5363a217f6fdc9c2bb12f2913845fe260748f330b7e52a43a607e38fb\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 27.92279200000121,\n      \"stdoutBytes\": 65,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"746586355e0310d76ede8eaa02168e7328eb580197063108a153a0b12e55f53a\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 21.16437500000029,\n      \"stdoutBytes\": 8771,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"rm\",\n        \"746586355e0310d76ede8eaa02168e7328eb580197063108a153a0b12e55f53a\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 29.126541999998153,\n      \"stdoutBytes\": 65,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"ac59c5d53a8a4163edf72421a2df7ee2c857d4d2a425f13fbb8a6b2c7c90a8d2\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 23.223707999997714,\n      \"stdoutBytes\": 10561,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"rm\",\n        \"ac59c5d53a8a4163edf72421a2df7ee2c857d4d2a425f13fbb8a6b2c7c90a8d2\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 32.459249999999884,\n      \"stdoutBytes\": 65,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"6d28708a0dcf717ab404d91f568f8b1111faca1d2fc47e026f2988da35672725\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 19.73320899999817,\n      \"stdoutBytes\": 10087,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"rm\",\n        \"6d28708a0dcf717ab404d91f568f8b1111faca1d2fc47e026f2988da35672725\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 26.627416999999696,\n      \"stdoutBytes\": 65,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"network\",\n        \"inspect\",\n        \"wse-industry-e24_default\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 20.515666999999667,\n      \"stdoutBytes\": 1092,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"network\",\n        \"rm\",\n        \"8f7a69d3cd59a48cc561da8e173cc56bc8d685fd9d55e086c0fcb3b975833815\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 218.19920800000182,\n      \"stdoutBytes\": 65,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    },\n    {\n      \"command\": [\n        \"docker\",\n        \"inspect\",\n        \"7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4\"\n      ],\n      \"exitCode\": 0,\n      \"signal\": null,\n      \"timedOut\": false,\n      \"outputLimit\": false,\n      \"spawnFailed\": false,\n      \"elapsedMs\": 18.884624999998778,\n      \"stdoutBytes\": 12844,\n      \"stderrBytes\": 0,\n      \"rawOutputRetained\": false\n    }\n  ],\n  \"exits\": {\n    \"frontend\": {\n      \"purpose\": \"frozen production signal acceptance\",\n      \"signal\": \"SIGTERM\",\n      \"signalCommandExit\": 0,\n      \"exitCode\": 143,\n      \"elapsedMs\": 150.63358399999925,\n      \"awaited\": true,\n      \"observerTimedOut\": false,\n      \"forcedContainerKill\": false\n    },\n    \"worker\": {\n      \"purpose\": \"frozen production signal acceptance\",\n      \"signal\": \"SIGTERM\",\n      \"signalCommandExit\": 0,\n      \"exitCode\": 143,\n      \"elapsedMs\": 445.09245799999917,\n      \"awaited\": true,\n      \"observerTimedOut\": false,\n      \"forcedContainerKill\": false\n    },\n    \"api\": {\n      \"purpose\": \"frozen production signal acceptance\",\n      \"signal\": \"SIGTERM\",\n      \"signalCommandExit\": 0,\n      \"exitCode\": 143,\n      \"elapsedMs\": 310.0386249999974,\n      \"awaited\": true,\n      \"observerTimedOut\": false,\n      \"forcedContainerKill\": false\n    },\n    \"fixture\": {\n      \"purpose\": \"owned fixture cleanup\",\n      \"signal\": \"SIGTERM\",\n      \"signalCommandExit\": 0,\n      \"exitCode\": 143,\n      \"elapsedMs\": 45.81941699999879,\n      \"awaited\": true,\n      \"observerTimedOut\": false,\n      \"forcedContainerKill\": false\n    }\n  },\n  \"cleanupFailures\": [],\n  \"privateArtifacts\": {\n    \"rawLogs\": false,\n    \"httpBodies\": false,\n    \"credentials\": false,\n    \"browserTrace\": false,\n    \"screenshot\": false,\n    \"video\": false,\n    \"har\": false,\n    \"storageState\": false\n  },\n  \"head\": \"c97c2320e5a0909036f8fa1e13c666cd58f0669e\",\n  \"sourceFingerprint\": \"5464ab8650b03ea5c281bd53d32940af8d72ce2fe1b1317c3e7df562725a4b27\",\n  \"frozenHashes\": {\n    \"evidence/phase-1/E24/fixture.json\": \"47f86919e5f251c3f8ee394727235b906fc03a5eac67e36cc3e273486a21a8fd\",\n    \"scripts/e24-seed.mjs\": \"16d0f9286acfe76a5dfb79819d0ee49e975098ecb710464327d41c0f54ec4025\",\n    \"scripts/e24-baseline.mjs\": \"37bdf00c299706b5e5ef2d7ff11311605a9529fa0ee7abcfa9ef78dd62ce49c4\",\n    \"evidence/phase-1/E24/fixtures.md\": \"5574f4d4d6c8e8cbbd17273ba8385e3a1ac43c2193d34d40b0dca4da11090a1c\"\n  },\n  \"preflight\": {\n    \"vacantPorts\": [\n      4322,\n      4324,\n      4323,\n      4321\n    ],\n    \"existingApplicationResources\": 0,\n    \"postgres\": {\n      \"id\": \"7f30b7e5c30b6b818d8ddd61c01bc3ee74c29ccf854300cd0b05d12d8189a8e4\",\n      \"image\": \"sha256:051f7b7b3abdd564d5d1bd1e8c4b9c1b6e77087d1dd22020ede611c096a272e0\",\n      \"configuredImage\": \"postgres:17.11-bookworm@sha256:051f7b7b3abdd564d5d1bd1e8c4b9c1b6e77087d1dd22020ede611c096a272e0\",\n      \"mounts\": [\n        {\n          \"type\": \"volume\",\n          \"name\": \"wse-industry_postgres-data\",\n          \"destination\": \"/var/lib/postgresql/data\"\n        }\n      ]\n    },\n    \"publicTableCount\": 0,\n    \"frozenInputsUnchanged\": true\n  },\n  \"seed\": {\n    \"users\": 2,\n    \"monitors\": 2,\n    \"checks\": 4,\n    \"queued\": 0,\n    \"running\": 0,\n    \"terminal\": 4\n  },\n  \"startup\": {\n    \"liveness\": {\n      \"api\": 200,\n      \"worker\": 200\n    },\n    \"readiness\": {\n      \"api\": 200,\n      \"worker\": 200\n    },\n    \"worker\": {\n      \"queueAge\": 0,\n      \"active\": 0,\n      \"claims\": 0,\n      \"recoveries\": 0\n    }\n  },\n  \"browser\": {\n    \"chromium\": \"151.0.7922.34\",\n    \"contexts\": 2,\n    \"ownerArticleCounts\": [\n      1,\n      1\n    ],\n    \"initialHistoryRows\": [\n      3,\n      1\n    ],\n    \"foreignDetailStatuses\": [\n      404,\n      404\n    ],\n    \"authenticatedServerHtml\": true,\n    \"serverHistoryBeforeHydration\": true,\n    \"javascriptStatus\": 200,\n    \"cssStatus\": 200,\n    \"pageErrors\": 0,\n    \"hydrationErrors\": 0,\n    \"closedBeforeOutage\": true\n  },\n  \"manual\": {\n    \"acceptedStatus\": 202,\n    \"acceptedState\": \"QUEUED\",\n    \"acceptedOutcomeFieldsNull\": true,\n    \"held\": {\n      \"held\": 1,\n      \"outboundCalls\": 1,\n      \"active\": 1,\n      \"localObservationAndReleaseMs\": 6.366958000000523,\n      \"dockerOrSqlBeforeRelease\": false\n    },\n    \"fixture\": {\n      \"outboundCalls\": 1,\n      \"holdRequests\": 1,\n      \"held\": 0,\n      \"releases\": 1,\n      \"watchdogReleases\": 0,\n      \"lastHeldMs\": 13.711250000000291\n    },\n    \"sameTerminalId\": true,\n    \"terminalState\": \"SUCCEEDED\",\n    \"httpStatus\": 200,\n    \"historyRows\": 4,\n    \"reloadRetainsResult\": true\n  },\n  \"outage\": {\n    \"liveness\": {\n      \"api\": 200,\n      \"worker\": 200\n    },\n    \"readiness\": {\n      \"api\": 503,\n      \"worker\": 503\n    },\n    \"rejectedStatus\": 503,\n    \"acceptedId\": false,\n    \"active\": 0,\n    \"claims\": 1,\n    \"recoveries\": 0,\n    \"outboundCalls\": 1,\n    \"sameLiveProcesses\": true\n  },\n  \"restore\": {\n    \"readiness\": {\n      \"api\": 200,\n      \"worker\": 200\n    },\n    \"samePostgresContainerImageVolumes\": true,\n    \"publicAuthorityUnchanged\": true,\n    \"exactPrivateRowsUnchanged\": true,\n    \"counts\": {\n      \"users\": 2,\n      \"monitors\": 2,\n      \"checks\": 5,\n      \"queued\": 0,\n      \"running\": 0,\n      \"terminal\": 5\n    },\n    \"rejectedIntentPersisted\": false,\n    \"sameLiveProcesses\": true\n  },\n  \"metricNameInventory\": {\n    \"api\": {\n      \"status\": 200,\n      \"validNameArray\": true,\n      \"nameCount\": 2,\n      \"names\": [\n        \"check.queue.age\",\n        \"http.server.requests\"\n      ],\n      \"unlistedNameCount\": 0\n    },\n    \"worker\": {\n      \"status\": 200,\n      \"validNameArray\": true,\n      \"nameCount\": 4,\n      \"names\": [\n        \"check.claims\",\n        \"check.queue.age\",\n        \"check.recoveries\",\n        \"check.worker.active\"\n      ],\n      \"unlistedNameCount\": 0\n    }\n  },\n  \"metrics\": {\n    \"apiQueueAge\": 0,\n    \"worker\": {\n      \"queueAge\": 0,\n      \"active\": 0,\n      \"claims\": 1,\n      \"recoveries\": 0\n    },\n    \"httpMeasurements\": {\n      \"COUNT\": 55,\n      \"TOTAL_TIME\": 3.0125017969999996,\n      \"MAX\": 1.026114084\n    },\n    \"boundedHttpTags\": {\n      \"method\": [\n        \"GET\",\n        \"POST\"\n      ],\n      \"process_role\": [\n        \"api\"\n      ],\n      \"status\": [\n        \"200\",\n        \"202\",\n        \"404\",\n        \"503\"\n      ],\n      \"uri\": [\n        \"/api/monitors\",\n        \"/api/monitors/{id}\",\n        \"/api/monitors/{id}/checks\",\n        \"/api/monitors/{id}/checks/{checkId}\",\n        \"/api/session/csrf\",\n        \"/api/session/login\",\n        \"/ops/health/**\",\n        \"/ops/metrics\",\n        \"/ops/metrics/{requiredMetricName}\"\n      ]\n    },\n    \"missingUuidRequests\": 10,\n    \"before404Count\": 2,\n    \"after404Count\": 12,\n    \"errorCountDelta\": 10,\n    \"tagSetsUnchanged\": true,\n    \"sourceNormalizationSha256\": \"f3a54450d30ecf7533ed5256f932c6eb161132df443b856de371cc45af20f88f\",\n    \"exactUniqueSeriesCountClaimed\": false,\n    \"positiveQueueAgeAndRecoveryProof\": \"separate affected OperationsIntegrationTest; no extra crash or outage scenario\"\n  },\n  \"runtime\": {\n    \"api\": {\n      \"id\": \"6d28708a0dcf717ab404d91f568f8b1111faca1d2fc47e026f2988da35672725\",\n      \"image\": \"sha256:d6f6ab129f0199ab6cb92ae12ee40fb83dba92bc40d9421a16f874b0265fbcc1\",\n      \"pid\": 68781,\n      \"startedAt\": \"2026-08-28T11:05:13.162688255Z\",\n      \"restartCount\": 0,\n      \"pidInsideContainer\": 1,\n      \"uid\": 10001,\n      \"command\": [\n        \"java\",\n        \"-jar\",\n        \"/app/app.jar\"\n      ],\n      \"versions\": {\n        \"java\": \"21.0.7+6\",\n        \"jarSha256\": \"2e7108c8ec7572131714be32de6c7f6da63c324a45c9d3fc76c39a280be728d8\"\n      }\n    },\n    \"worker\": {\n      \"id\": \"ac59c5d53a8a4163edf72421a2df7ee2c857d4d2a425f13fbb8a6b2c7c90a8d2\",\n      \"image\": \"sha256:d6f6ab129f0199ab6cb92ae12ee40fb83dba92bc40d9421a16f874b0265fbcc1\",\n      \"pid\": 68782,\n      \"startedAt\": \"2026-08-28T11:05:13.156771505Z\",\n      \"restartCount\": 0,\n      \"pidInsideContainer\": 1,\n      \"uid\": 10001,\n      \"command\": [\n        \"java\",\n        \"-jar\",\n        \"/app/app.jar\",\n        \"--spring.profiles.active=worker\",\n        \"--spring.main.web-application-type=none\"\n      ],\n      \"versions\": {\n        \"java\": \"21.0.7+6\",\n        \"jarSha256\": \"2e7108c8ec7572131714be32de6c7f6da63c324a45c9d3fc76c39a280be728d8\"\n      }\n    },\n    \"frontend\": {\n      \"id\": \"746586355e0310d76ede8eaa02168e7328eb580197063108a153a0b12e55f53a\",\n      \"image\": \"sha256:9356a3b65bd58be8e12f5a4721975f4801e1601f7dbef0a4bb7ddfd6ef93ac9e\",\n      \"pid\": 68780,\n      \"startedAt\": \"2026-08-28T11:05:13.162191589Z\",\n      \"restartCount\": 0,\n      \"pidInsideContainer\": 1,\n      \"uid\": 1000,\n      \"command\": [\n        \"node\",\n        \"server.js\"\n      ],\n      \"versions\": {\n        \"node\": \"24.19.0\",\n        \"next\": \"16.3.3\",\n        \"react\": \"19.2.8\",\n        \"mode\": \"production\",\n        \"origin\": \"http://api:4322\",\n        \"manualSignal\": false\n      }\n    }\n  },\n  \"logs\": {\n    \"roles\": {\n      \"api\": {\n        \"lines\": 104,\n        \"structuredLines\": 104,\n        \"observationEvents\": 56\n      },\n      \"worker\": {\n        \"lines\": 45,\n        \"structuredLines\": 45,\n        \"observationEvents\": 4\n      },\n      \"frontend\": {\n        \"lines\": 5,\n        \"structuredLines\": 0,\n        \"observationEvents\": 0\n      },\n      \"fixture\": {\n        \"lines\": 0,\n        \"structuredLines\": 0,\n        \"observationEvents\": 0\n      }\n    },\n    \"runtimeSentinelValuesScanned\": 19,\n    \"runtimeSentinelMatches\": 0,\n    \"unboundedInputMatches\": 0,\n    \"responseCorrelations\": 18,\n    \"matchedResponseCorrelations\": 18,\n    \"generatedResponseRequestIds\": [\n      \"1a86653a-4074-45af-b6b1-cd9c46e249df\",\n      \"7bce98f8-4ff6-41bf-82c5-fc6bb3ae9910\",\n      \"8b0eb817-02b6-4257-9bd3-31a000c5755f\",\n      \"640f1963-fc28-4ff7-8429-113bc6347eb4\",\n      \"ad9760b3-47f8-499b-ab51-1849a35aa8d5\",\n      \"08763de1-7628-433f-97f1-4cf6e78b0f4b\",\n      \"0a094451-07b1-47d5-a301-fd779095a37e\",\n      \"54342522-bddb-4dc6-8816-ce448ad7f14d\",\n      \"d774945d-dab7-47a6-a086-e3fa04857047\",\n      \"b2c17ebf-ffed-4c2f-a5ef-9eead046b4f7\",\n      \"a4c50240-d8b4-4f17-818e-46fbf3e8d0e9\",\n      \"8f9953fd-8e9c-4107-9a1b-f52cb44de02a\",\n      \"689577b5-190a-4030-91f1-51ea015cd336\",\n      \"b5e0d24b-ab62-47d7-b30d-178fffc7a129\",\n      \"8951ae58-7344-4543-a1ad-45fcc0e89397\",\n      \"b0d6ce63-81e8-4b00-acc4-5db49ef06e9b\",\n      \"9b3950ee-b22f-444e-a42e-3a2312c2f6e7\",\n      \"60e9fc81-c095-4a40-bc4f-ee015b6d0bd8\"\n    ],\n    \"stableApiProcess\": true,\n    \"stableWorkerProcess\": true,\n    \"distinctRoleProcesses\": true,\n    \"apiProcessId\": \"ebd69ae2-d1dc-4aee-af8c-fc677ecfb8b3\",\n    \"workerProcessId\": \"5e678e3a-b8d5-4e3c-841d-9d1b0c3cb77a\",\n    \"committedClaimEvents\": 1,\n    \"unavailableAuthorityEvents\": 3,\n    \"recoveryEvents\": 0,\n    \"crossProcessCheckRunTraceClaimed\": false,\n    \"rawLogsRetained\": false\n  },\n  \"cleanup\": {\n    \"browserClosed\": true,\n    \"applicationContainersRemoved\": true,\n    \"applicationNetworkRemoved\": true,\n    \"ownedSchemaDropped\": true,\n    \"postgresPreserved\": true,\n    \"postgresStopAttempted\": true,\n    \"postgresRestoreAttempted\": true,\n    \"postgresRestored\": true,\n    \"forcedContainerKill\": false,\n    \"sharedOrPublicDataRemoved\": false\n  },\n  \"elapsedSeconds\": 21.791,\n  \"hostedCiExecutionClaimed\": false\n}\n"
+    },
+    {
+      "source": "evidence/phase-1/E24/container/author-5464ab8650b03ea5.started.json",
+      "sha256": "522dcd61aa0e296934a6709f847b0af8f3328005f7911a0d0ce4c8d29da2b56c",
+      "bytes": 189,
+      "raw": "{\"actor\":\"author\",\"head\":\"c97c2320e5a0909036f8fa1e13c666cd58f0669e\",\"fingerprint\":\"5464ab8650b03ea5c281bd53d32940af8d72ce2fe1b1317c3e7df562725a4b27\",\"startedAt\":\"2026-08-28T11:05:08.268Z\"}\n"
+    }
+  ]
+}
diff --git a/evidence/phase-1/E24/repair4/repair4-image-build-native.json b/evidence/phase-1/E24/repair4/repair4-image-build-native.json
new file mode 100644
index 0000000..d9ef8cb
--- /dev/null
+++ b/evidence/phase-1/E24/repair4/repair4-image-build-native.json
@@ -0,0 +1,109 @@
+{
+  "thread": "E24",
+  "profile": "phase-1",
+  "attempt": 5,
+  "repair": 4,
+  "actor": "author",
+  "result": "PASS",
+  "invocation": {
+    "command": [
+      "docker",
+      "compose",
+      "-p",
+      "wse-industry-e24",
+      "-f",
+      "compose.production.yaml",
+      "build",
+      "api"
+    ],
+    "environment": {
+      "DB_SCHEMA": "e24_container"
+    },
+    "displayCommand": "DB_SCHEMA=e24_container docker compose -p wse-industry-e24 -f compose.production.yaml build api",
+    "startedAt": "2026-08-28T11:03:32.495550+00:00",
+    "attempt": 5,
+    "repair": 4,
+    "actor": "author",
+    "permission": "require_escalated pinned backend image build/cache/network authorization",
+    "buildInvocationThisRepair": 1,
+    "frontendRebuild": false,
+    "automaticRetries": 0,
+    "authorizationSha256": "831b8dd80b373c90efa0597416b23c2dfed3e36b4ab040d7a730f31ce4acf025",
+    "backendJarSha256": "2e7108c8ec7572131714be32de6c7f6da63c324a45c9d3fc76c39a280be728d8"
+  },
+  "nativeResult": {
+    "exitCode": 0,
+    "elapsedSeconds": 4.054,
+    "startedAt": "2026-08-28T11:03:32.495550+00:00",
+    "endedAt": "2026-08-28T11:03:36.550196+00:00",
+    "signal": null
+  },
+  "applicationScenarioStarted": false,
+  "browserStarted": false,
+  "baselineRerun": false,
+  "sourceModified": false,
+  "cumulativeLedger": {
+    "previousBytes": 2672,
+    "previousSha256": "1ae79a2d4bc2843c8f492084a10efc4c026b46bce56309869918403f0c1d60fa",
+    "previousPrefixPreserved": true,
+    "currentEntries": 9,
+    "currentSha256": "09b3005f21a6dbdb72ad0714eb490d8da1636f1ee180a233a74b65a6f9835217"
+  },
+  "cumulativeBudget": {
+    "baselineInvocations": 1,
+    "expectedBaselineCounterexamples": 1,
+    "baselineWallSeconds": 7.477,
+    "affectedMavenInvocations": 5,
+    "affectedMavenWallSeconds": 31.167,
+    "javaTestsExecuted": 16,
+    "javaTestsPassed": 15,
+    "javaTestErrors": 1,
+    "javaAssertionFailures": 0,
+    "javaTestsSkipped": 0,
+    "unexpectedFormalFailures": 4,
+    "repairTasksUsed": 4,
+    "maximumFreshRepairs": null,
+    "historicalMaximumFreshRepairs": 2,
+    "forcedDescriptorRefreshes": 1,
+    "imageBuildInvocations": 2,
+    "imageBuildWallSeconds": 52.415,
+    "containerScenarios": 1,
+    "containerScenarioWallSeconds": 24.842,
+    "browserInvocations": 1,
+    "postgresStopRestoreSequences": 1,
+    "rootRuntimeProductGateRuns": 0,
+    "cumulativeGateWallSeconds": 115.901,
+    "loadRuns": 0,
+    "parameterSweeps": 0,
+    "automaticRetries": 0,
+    "e11KillScenarioRuns": 0,
+    "e20DatasetRuns": 0
+  },
+  "cleanup": "Docker build command awaited; no application service started; images/cache retained for authorized scenario and root-owned final cleanup",
+  "nativeFiles": [
+    {
+      "source": "output/phase-1/e24/repair4-image-build.started.json",
+      "sha256": "8219bfbdd51bbe1cef1d9810f95cc4643ea1c17fffa75f97c45f3aeb44237977",
+      "bytes": 791,
+      "raw": "{\n  \"command\": [\n    \"docker\",\n    \"compose\",\n    \"-p\",\n    \"wse-industry-e24\",\n    \"-f\",\n    \"compose.production.yaml\",\n    \"build\",\n    \"api\"\n  ],\n  \"environment\": {\n    \"DB_SCHEMA\": \"e24_container\"\n  },\n  \"displayCommand\": \"DB_SCHEMA=e24_container docker compose -p wse-industry-e24 -f compose.production.yaml build api\",\n  \"startedAt\": \"2026-08-28T11:03:32.495550+00:00\",\n  \"attempt\": 5,\n  \"repair\": 4,\n  \"actor\": \"author\",\n  \"permission\": \"require_escalated pinned backend image build/cache/network authorization\",\n  \"buildInvocationThisRepair\": 1,\n  \"frontendRebuild\": false,\n  \"automaticRetries\": 0,\n  \"authorizationSha256\": \"831b8dd80b373c90efa0597416b23c2dfed3e36b4ab040d7a730f31ce4acf025\",\n  \"backendJarSha256\": \"2e7108c8ec7572131714be32de6c7f6da63c324a45c9d3fc76c39a280be728d8\"\n}\n"
+    },
+    {
+      "source": "output/phase-1/e24/repair4-image-build.log",
+      "sha256": "d3d5714fe034a209206caf0ce52346863343e981c97b22802eba6a921128d5e7",
+      "bytes": 1935,
+      "raw": "Compose can now delegate builds to bake for better performance.\n To do so, set COMPOSE_BAKE=true.\n#0 building with \"desktop-linux\" instance using docker driver\n\n#1 [api internal] load build definition from Dockerfile.api\n#1 transferring dockerfile: 442B done\n#1 DONE 0.0s\n\n#2 [api internal] load metadata for docker.io/library/eclipse-temurin:21.0.7_6-jre-jammy@sha256:c4e6542e774de504da9b4729ff8d761287c965c1d788528ca78da30024efdb23\n#2 DONE 1.4s\n\n#3 [api internal] load .dockerignore\n#3 transferring context: 226B done\n#3 DONE 0.0s\n\n#4 [api 1/3] FROM docker.io/library/eclipse-temurin:21.0.7_6-jre-jammy@sha256:c4e6542e774de504da9b4729ff8d761287c965c1d788528ca78da30024efdb23\n#4 resolve docker.io/library/eclipse-temurin:21.0.7_6-jre-jammy@sha256:c4e6542e774de504da9b4729ff8d761287c965c1d788528ca78da30024efdb23 0.0s done\n#4 DONE 0.0s\n\n#5 [api internal] load build context\n#5 transferring context: 60.10MB 0.4s done\n#5 DONE 0.4s\n\n#6 [api 2/3] WORKDIR /app\n#6 CACHED\n\n#7 [api 3/3] COPY --chown=10001:10001 backend/target/monitor-api-0.0.1.jar /app/app.jar\n#7 DONE 0.2s\n\n#8 [api] exporting to image\n#8 exporting layers\n#8 exporting layers 1.4s done\n#8 exporting manifest sha256:5c4e33a650592e9429eddb31225415aa496604737c36de3dfb76ddc0a6699c9b\n#8 exporting manifest sha256:5c4e33a650592e9429eddb31225415aa496604737c36de3dfb76ddc0a6699c9b done\n#8 exporting config sha256:5ad6db3889e3fd235c5e4f7f09d4c4a60aea6592f1c38d4ca174fa94ef479423 done\n#8 exporting attestation manifest sha256:3a08a4a985793ad02112cd2315ba276917c64de31acade4ca3470669b5accbf4 done\n#8 exporting manifest list sha256:d6f6ab129f0199ab6cb92ae12ee40fb83dba92bc40d9421a16f874b0265fbcc1 done\n#8 naming to docker.io/library/wse-industry-e24-api:local done\n#8 unpacking to docker.io/library/wse-industry-e24-api:local\n#8 unpacking to docker.io/library/wse-industry-e24-api:local 0.3s done\n#8 DONE 1.7s\n\n#9 [api] resolving provenance for metadata file\n#9 DONE 0.0s\n api  Built\n"
+    },
+    {
+      "source": "output/phase-1/e24/repair4-image-build.result.json",
+      "sha256": "db05b5d79d7371d3f6cb1bac50898dea0b3a9f25356248a72c7712aaa606250b",
+      "bytes": 165,
+      "raw": "{\n  \"exitCode\": 0,\n  \"elapsedSeconds\": 4.054,\n  \"startedAt\": \"2026-08-28T11:03:32.495550+00:00\",\n  \"endedAt\": \"2026-08-28T11:03:36.550196+00:00\",\n  \"signal\": null\n}\n"
+    },
+    {
+      "source": "output/phase-1/e24/invocations.jsonl",
+      "sha256": "09b3005f21a6dbdb72ad0714eb490d8da1636f1ee180a233a74b65a6f9835217",
+      "bytes": 3085,
+      "raw": "{\"command\":\"node scripts/e24-baseline.mjs\",\"startedAt\":\"2026-08-28T08:51:05.183Z\",\"elapsedSeconds\":7.477,\"exitCode\":0}\n{\"command\":\"mvn -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package\",\"startedAt\":\"2026-08-28T09:11:45.983Z\",\"elapsedSeconds\":1.724,\"exitCode\":1,\"signal\":null}\n{\"command\":\"mvn -U -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package\",\"startedAt\":\"2026-08-28T09:22:21.048583+00:00\",\"attempt\":2,\"repair\":1,\"actor\":\"author\",\"permission\":\"require_escalated network/cache authorization\",\"forcedDescriptorRefreshes\":1,\"elapsedSeconds\":4.051,\"exitCode\":1,\"signal\":null}\n{\"command\":\"mvn -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package\",\"startedAt\":\"2026-08-28T09:31:22.232667+00:00\",\"attempt\":3,\"repair\":2,\"actor\":\"author\",\"permission\":\"require_escalated network/cache/PostgreSQL authorization\",\"forcedDescriptorRefreshes\":0,\"elapsedSeconds\":6.999,\"exitCode\":1,\"signal\":null}\n{\"command\":\"mvn -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package\",\"startedAt\":\"2026-08-28T10:35:41.429288+00:00\",\"attempt\":4,\"repair\":3,\"actor\":\"author\",\"permission\":\"require_escalated Java/Maven/cache/network/PostgreSQL authorization\",\"forcedDescriptorRefreshes\":0,\"elapsedSeconds\":8.772,\"exitCode\":0,\"signal\":null,\"nativeBundle\":\"evidence/phase-1/E24/repair3/repair3-affected-java-native.json\"}\n{\"command\":\"DB_SCHEMA=e24_container docker compose -p wse-industry-e24 -f compose.production.yaml build api frontend\",\"startedAt\":\"2026-08-28T10:39:20.133345+00:00\",\"attempt\":4,\"repair\":3,\"actor\":\"author\",\"permission\":\"require_escalated pinned production image build/cache/network authorization\",\"elapsedSeconds\":48.361,\"exitCode\":0,\"signal\":null}\n{\"command\":\"fnm exec --using 24.19.0 node scripts/e24-container.mjs author\",\"startedAt\":\"2026-08-28T10:43:00.859467+00:00\",\"attempt\":4,\"repair\":3,\"actor\":\"author\",\"permission\":\"require_escalated actual Docker/PostgreSQL/browser/network authorization\",\"elapsedSeconds\":24.842,\"exitCode\":1,\"signal\":null,\"rawDiagnosticOutputRetained\":false}\n{\"command\":\"mvn -B -ntp -f backend/pom.xml -Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest package\",\"startedAt\":\"2026-08-28T11:02:28.242864+00:00\",\"attempt\":5,\"repair\":4,\"actor\":\"author\",\"permission\":\"require_escalated pinned Java/Maven/cache/network/PostgreSQL authorization\",\"forcedDescriptorRefreshes\":0,\"elapsedSeconds\":9.621,\"exitCode\":0,\"signal\":null,\"nativeBundle\":\"evidence/phase-1/E24/repair4/repair4-affected-java-native.json\"}\n{\"command\":\"DB_SCHEMA=e24_container docker compose -p wse-industry-e24 -f compose.production.yaml build api\",\"startedAt\":\"2026-08-28T11:03:32.495550+00:00\",\"attempt\":5,\"repair\":4,\"actor\":\"author\",\"permission\":\"require_escalated pinned backend image build/cache/network authorization\",\"elapsedSeconds\":4.054,\"exitCode\":0,\"signal\":null,\"nativeBundle\":\"evidence/phase-1/E24/repair4/repair4-image-build-native.json\"}\n"
+    }
+  ]
+}
diff --git a/evidence/phase-1/E24/repair4/repair4-preservation.json b/evidence/phase-1/E24/repair4/repair4-preservation.json
new file mode 100644
index 0000000..357b6eb
--- /dev/null
+++ b/evidence/phase-1/E24/repair4/repair4-preservation.json
@@ -0,0 +1,515 @@
+{
+  "thread": "E24",
+  "profile": "phase-1",
+  "attempt": 5,
+  "repair": 4,
+  "start": "563b325ef871fe6d1fbfef7cf39a6581f2d0a94d",
+  "headBeforeRepair": "c97c2320e5a0909036f8fa1e13c666cd58f0669e",
+  "specRevision": "2ada57a71cd34fa2fae9809415c362a8bbfcdf02",
+  "status": "AUTHOR_PASS; independent root final gate pending",
+  "authorizationSha256": "831b8dd80b373c90efa0597416b23c2dfed3e36b4ab040d7a730f31ce4acf025",
+  "source": {
+    "backend/src/main/java/dev/evolution/monitor/HttpObservations.java": "f3a54450d30ecf7533ed5256f932c6eb161132df443b856de371cc45af20f88f",
+    "backend/src/main/resources/application.properties": "08b7a7389896025c27f9bda0fb42069c4bfd3283a6ebff400a2971ac6458d871",
+    "backend/src/test/java/dev/evolution/monitor/HttpObservationsTest.java": "a917d048a3265c83f0e9080e26408df87cae6ff069f983a15b78772d7c33d6a6",
+    "scripts/e24-container.mjs": "aa444c00ed2054557177e2d10fa8f5aac6dfb54faeaabf144d683cf09cc776e8"
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
+      "elapsedSeconds": 9.621,
+      "startedAt": "2026-08-28T11:02:28.242864+00:00",
+      "endedAt": "2026-08-28T11:02:37.864711+00:00",
+      "signal": null,
+      "testCounts": {
+        "tests": 6,
+        "failures": 0,
+        "errors": 0,
+        "skipped": 0,
+        "passed": 6
+      },
+      "ownedCleanupPassed": true
+    },
+    {
+      "name": "production backend-only image build",
+      "exitCode": 0,
+      "elapsedSeconds": 4.054,
+      "startedAt": "2026-08-28T11:03:32.495550+00:00",
+      "endedAt": "2026-08-28T11:03:36.550196+00:00",
+      "signal": null,
+      "frontendRebuild": false
+    },
+    {
+      "name": "author production container scenario",
+      "exitCode": 0,
+      "elapsedSeconds": 21.991,
+      "startedAt": "2026-08-28T11:05:08.079267+00:00",
+      "endedAt": "2026-08-28T11:05:30.071869+00:00",
+      "signal": null,
+      "safeObserverSummary": {
+        "actor": "author",
+        "result": "PASS",
+        "phase": "completed",
+        "evidence": "evidence/phase-1/E24/container/author-5464ab8650b03ea5.json",
+        "elapsedSeconds": 21.791,
+        "cleanupFailures": 0
+      },
+      "stdoutBytes": 172,
+      "stderrBytes": 0,
+      "rawDiagnosticOutputRetained": false
+    }
+  ],
+  "observations": {
+    "sourceFingerprint": "5464ab8650b03ea5c281bd53d32940af8d72ce2fe1b1317c3e7df562725a4b27",
+    "seed": {
+      "users": 2,
+      "monitors": 2,
+      "checks": 4,
+      "queued": 0,
+      "running": 0,
+      "terminal": 4
+    },
+    "startup": {
+      "liveness": {
+        "api": 200,
+        "worker": 200
+      },
+      "readiness": {
+        "api": 200,
+        "worker": 200
+      },
+      "worker": {
+        "queueAge": 0,
+        "active": 0,
+        "claims": 0,
+        "recoveries": 0
+      }
+    },
+    "browser": {
+      "chromium": "151.0.7922.34",
+      "contexts": 2,
+      "ownerArticleCounts": [
+        1,
+        1
+      ],
+      "initialHistoryRows": [
+        3,
+        1
+      ],
+      "foreignDetailStatuses": [
+        404,
+        404
+      ],
+      "authenticatedServerHtml": true,
+      "serverHistoryBeforeHydration": true,
+      "javascriptStatus": 200,
+      "cssStatus": 200,
+      "pageErrors": 0,
+      "hydrationErrors": 0,
+      "closedBeforeOutage": true
+    },
+    "manual": {
+      "acceptedStatus": 202,
+      "acceptedState": "QUEUED",
+      "acceptedOutcomeFieldsNull": true,
+      "held": {
+        "held": 1,
+        "outboundCalls": 1,
+        "active": 1,
+        "localObservationAndReleaseMs": 6.366958000000523,
+        "dockerOrSqlBeforeRelease": false
+      },
+      "fixture": {
+        "outboundCalls": 1,
+        "holdRequests": 1,
+        "held": 0,
+        "releases": 1,
+        "watchdogReleases": 0,
+        "lastHeldMs": 13.711250000000291
+      },
+      "sameTerminalId": true,
+      "terminalState": "SUCCEEDED",
+      "httpStatus": 200,
+      "historyRows": 4,
+      "reloadRetainsResult": true
+    },
+    "outage": {
+      "liveness": {
+        "api": 200,
+        "worker": 200
+      },
+      "readiness": {
+        "api": 503,
+        "worker": 503
+      },
+      "rejectedStatus": 503,
+      "acceptedId": false,
+      "active": 0,
+      "claims": 1,
+      "recoveries": 0,
+      "outboundCalls": 1,
+      "sameLiveProcesses": true
+    },
+    "restore": {
+      "readiness": {
+        "api": 200,
+        "worker": 200
+      },
+      "samePostgresContainerImageVolumes": true,
+      "publicAuthorityUnchanged": true,
+      "exactPrivateRowsUnchanged": true,
+      "counts": {
+        "users": 2,
+        "monitors": 2,
+        "checks": 5,
+        "queued": 0,
+        "running": 0,
+        "terminal": 5
+      },
+      "rejectedIntentPersisted": false,
+      "sameLiveProcesses": true
+    },
+    "metricNameInventory": {
+      "api": {
+        "status": 200,
+        "validNameArray": true,
+        "nameCount": 2,
+        "names": [
+          "check.queue.age",
+          "http.server.requests"
+        ],
+        "unlistedNameCount": 0
+      },
+      "worker": {
+        "status": 200,
+        "validNameArray": true,
+        "nameCount": 4,
+        "names": [
+          "check.claims",
+          "check.queue.age",
+          "check.recoveries",
+          "check.worker.active"
+        ],
+        "unlistedNameCount": 0
+      }
+    },
+    "metrics": {
+      "apiQueueAge": 0,
+      "worker": {
+        "queueAge": 0,
+        "active": 0,
+        "claims": 1,
+        "recoveries": 0
+      },
+      "httpMeasurements": {
+        "COUNT": 55,
+        "TOTAL_TIME": 3.0125017969999996,
+        "MAX": 1.026114084
+      },
+      "boundedHttpTags": {
+        "method": [
+          "GET",
+          "POST"
+        ],
+        "process_role": [
+          "api"
+        ],
+        "status": [
+          "200",
+          "202",
+          "404",
+          "503"
+        ],
+        "uri": [
+          "/api/monitors",
+          "/api/monitors/{id}",
+          "/api/monitors/{id}/checks",
+          "/api/monitors/{id}/checks/{checkId}",
+          "/api/session/csrf",
+          "/api/session/login",
+          "/ops/health/**",
+          "/ops/metrics",
+          "/ops/metrics/{requiredMetricName}"
+        ]
+      },
+      "missingUuidRequests": 10,
+      "before404Count": 2,
+      "after404Count": 12,
+      "errorCountDelta": 10,
+      "tagSetsUnchanged": true,
+      "sourceNormalizationSha256": "f3a54450d30ecf7533ed5256f932c6eb161132df443b856de371cc45af20f88f",
+      "exactUniqueSeriesCountClaimed": false,
+      "positiveQueueAgeAndRecoveryProof": "separate affected OperationsIntegrationTest; no extra crash or outage scenario"
+    },
+    "runtime": {
+      "api": {
+        "id": "6d28708a0dcf717ab404d91f568f8b1111faca1d2fc47e026f2988da35672725",
+        "image": "sha256:d6f6ab129f0199ab6cb92ae12ee40fb83dba92bc40d9421a16f874b0265fbcc1",
+        "pid": 68781,
+        "startedAt": "2026-08-28T11:05:13.162688255Z",
+        "restartCount": 0,
+        "pidInsideContainer": 1,
+        "uid": 10001,
+        "command": [
+          "java",
+          "-jar",
+          "/app/app.jar"
+        ],
+        "versions": {
+          "java": "21.0.7+6",
+          "jarSha256": "2e7108c8ec7572131714be32de6c7f6da63c324a45c9d3fc76c39a280be728d8"
+        }
+      },
+      "worker": {
+        "id": "ac59c5d53a8a4163edf72421a2df7ee2c857d4d2a425f13fbb8a6b2c7c90a8d2",
+        "image": "sha256:d6f6ab129f0199ab6cb92ae12ee40fb83dba92bc40d9421a16f874b0265fbcc1",
+        "pid": 68782,
+        "startedAt": "2026-08-28T11:05:13.156771505Z",
+        "restartCount": 0,
+        "pidInsideContainer": 1,
+        "uid": 10001,
+        "command": [
+          "java",
+          "-jar",
+          "/app/app.jar",
+          "--spring.profiles.active=worker",
+          "--spring.main.web-application-type=none"
+        ],
+        "versions": {
+          "java": "21.0.7+6",
+          "jarSha256": "2e7108c8ec7572131714be32de6c7f6da63c324a45c9d3fc76c39a280be728d8"
+        }
+      },
+      "frontend": {
+        "id": "746586355e0310d76ede8eaa02168e7328eb580197063108a153a0b12e55f53a",
+        "image": "sha256:9356a3b65bd58be8e12f5a4721975f4801e1601f7dbef0a4bb7ddfd6ef93ac9e",
+        "pid": 68780,
+        "startedAt": "2026-08-28T11:05:13.162191589Z",
+        "restartCount": 0,
+        "pidInsideContainer": 1,
+        "uid": 1000,
+        "command": [
+          "node",
+          "server.js"
+        ],
+        "versions": {
+          "node": "24.19.0",
+          "next": "16.3.3",
+          "react": "19.2.8",
+          "mode": "production",
+          "origin": "http://api:4322",
+          "manualSignal": false
+        }
+      }
+    },
+    "exits": {
+      "frontend": {
+        "purpose": "frozen production signal acceptance",
+        "signal": "SIGTERM",
+        "signalCommandExit": 0,
+        "exitCode": 143,
+        "elapsedMs": 150.63358399999925,
+        "awaited": true,
+        "observerTimedOut": false,
+        "forcedContainerKill": false
+      },
+      "worker": {
+        "purpose": "frozen production signal acceptance",
+        "signal": "SIGTERM",
+        "signalCommandExit": 0,
+        "exitCode": 143,
+        "elapsedMs": 445.09245799999917,
+        "awaited": true,
+        "observerTimedOut": false,
+        "forcedContainerKill": false
+      },
+      "api": {
+        "purpose": "frozen production signal acceptance",
+        "signal": "SIGTERM",
+        "signalCommandExit": 0,
+        "exitCode": 143,
+        "elapsedMs": 310.0386249999974,
+        "awaited": true,
+        "observerTimedOut": false,
+        "forcedContainerKill": false
+      },
+      "fixture": {
+        "purpose": "owned fixture cleanup",
+        "signal": "SIGTERM",
+        "signalCommandExit": 0,
+        "exitCode": 143,
+        "elapsedMs": 45.81941699999879,
+        "awaited": true,
+        "observerTimedOut": false,
+        "forcedContainerKill": false
+      }
+    },
+    "cleanup": {
+      "browserClosed": true,
+      "applicationContainersRemoved": true,
+      "applicationNetworkRemoved": true,
+      "ownedSchemaDropped": true,
+      "postgresPreserved": true,
+      "postgresStopAttempted": true,
+      "postgresRestoreAttempted": true,
+      "postgresRestored": true,
+      "forcedContainerKill": false,
+      "sharedOrPublicDataRemoved": false
+    }
+  },
+  "safeLogScan": {
+    "roles": {
+      "api": {
+        "lines": 104,
+        "structuredLines": 104,
+        "observationEvents": 56
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
+    "responseCorrelations": 18,
+    "matchedResponseCorrelations": 18,
+    "stableApiProcess": true,
+    "stableWorkerProcess": true,
+    "distinctRoleProcesses": true,
+    "committedClaimEvents": 1,
+    "unavailableAuthorityEvents": 3,
+    "recoveryEvents": 0,
+    "crossProcessCheckRunTraceClaimed": false,
+    "rawLogsRetained": false
+  },
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
+  "nativeFiles": [
+    {
+      "path": "evidence/phase-1/E24/container/author-5464ab8650b03ea5.json",
+      "sha256": "e2af8cae161592bf031fbef5e0c0dd3eaf4453fdff83fea531f236d36ab040ec",
+      "bytes": 44432
+    },
+    {
+      "path": "evidence/phase-1/E24/container/author-5464ab8650b03ea5.started.json",
+      "sha256": "522dcd61aa0e296934a6709f847b0af8f3328005f7911a0d0ce4c8d29da2b56c",
+      "bytes": 189
+    },
+    {
+      "path": "evidence/phase-1/E24/container/invocations.jsonl",
+      "sha256": "c698afd0e0312c0ad80929679c048bbd22c1578a3a77d1df6355e3672a9b7cb3",
+      "bytes": 478
+    },
+    {
+      "path": "evidence/phase-1/E24/repair4/cumulative-invocations.jsonl",
+      "sha256": "f038964ccef2b1008275ea85e1204f1f0aed8db997759b6897c4b5d2a9937d86",
+      "bytes": 3507
+    },
+    {
+      "path": "evidence/phase-1/E24/repair4/repair4-affected-java-native.json",
+      "sha256": "38f3a1213358d9f6f36bf09243827d7fdc6aaf565112849564915664a9d26f8c",
+      "bytes": 156066
+    },
+    {
+      "path": "evidence/phase-1/E24/repair4/repair4-image-build-native.json",
+      "sha256": "0ce73007b053b26e495d9d31193b5dc6203018b8dd8aa8c7224253a1ae8efa33",
+      "bytes": 9858
+    },
+    {
+      "path": "evidence/phase-1/E24/repair4/repair4-author-container-native.json",
+      "sha256": "ced7b31c76c377989cc07bfda0002f26a847d17a730eabc85a3774d565710fd4",
+      "bytes": 59269
+    }
+  ],
+  "cumulativeBudget": {
+    "baselineInvocations": 1,
+    "expectedBaselineCounterexamples": 1,
+    "baselineWallSeconds": 7.477,
+    "affectedMavenInvocations": 5,
+    "affectedMavenWallSeconds": 31.167,
+    "javaTestsExecuted": 16,
+    "javaTestsPassed": 15,
+    "javaTestErrors": 1,
+    "javaAssertionFailures": 0,
+    "javaTestsSkipped": 0,
+    "unexpectedFormalFailures": 4,
+    "repairTasksUsed": 4,
+    "maximumFreshRepairs": null,
+    "historicalMaximumFreshRepairs": 2,
+    "forcedDescriptorRefreshes": 1,
+    "imageBuildInvocations": 2,
+    "imageBuildWallSeconds": 52.415,
+    "containerScenarios": 2,
+    "containerScenarioWallSeconds": 46.833,
+    "browserInvocations": 2,
+    "postgresStopRestoreSequences": 2,
+    "rootRuntimeProductGateRuns": 0,
+    "cumulativeGateWallSeconds": 137.892,
+    "loadRuns": 0,
+    "parameterSweeps": 0,
+    "automaticRetries": 0,
+    "e11KillScenarioRuns": 0,
+    "e20DatasetRuns": 0
+  },
+  "preservation": {
+    "priorCheckpoint": "c97c2320e5a0909036f8fa1e13c666cd58f0669e",
+    "priorUnexpectedFormalFailures": 4,
+    "priorMavenFailures": 3,
+    "priorContainerFailures": 1,
+    "newFormalFailures": 0,
+    "priorCumulativeLedgerBytes": 2207,
+    "priorCumulativeLedgerSha256": "7b6d4221eaf0d1cabb7480e8e802d0a41474c9675c507ce58f3a717ba0f9f58c",
+    "priorLedgerPrefixPreserved": true,
+    "priorEvidenceRewritten": false,
+    "baselineRepeated": false,
+    "additionalProductGates": 0
+  },
+  "unresolved": [
+    "Independent root affected Java and container verification remains pending.",
+    "Hosted CI execution is not claimed."
+  ],
+  "hostedCiExecutionClaimed": false
+}
diff --git a/scripts/e24-container.mjs b/scripts/e24-container.mjs
index f693479..0926161 100644
--- a/scripts/e24-container.mjs
+++ b/scripts/e24-container.mjs
@@ -638,6 +638,15 @@ try {
 
   phase = 'bounded real metrics and ten missing UUID requests';
   const names = await Promise.all([http(apiOrigin + fixture.endpoints.metricNames), http(workerOrigin + fixture.endpoints.metricNames)]);
+  // Retain only fixed instrumentation names; unexpected names remain RAM-only, represented by a count.
+  const diagnosticNames = new Set([...fixture.metrics.names, 'http.server.requests.active']);
+  evidence.metricNameInventory = Object.fromEntries(names.map((response, index) => {
+    const inventory = Array.isArray(response.json?.names) ? response.json.names : [];
+    return [['api', 'worker'][index], { status: response.status, validNameArray: Array.isArray(response.json?.names),
+      nameCount: inventory.length, names: [...new Set(inventory.filter(name => diagnosticNames.has(name)))].sort(),
+      unlistedNameCount: inventory.filter(name => !diagnosticNames.has(name)).length }];
+  }));
+  persist();
   check(same(names[0].json?.names?.slice().sort(), ['check.queue.age', 'http.server.requests'])
     && same(names[1].json?.names?.slice().sort(), ['check.claims', 'check.queue.age', 'check.recoveries', 'check.worker.active']),
   'Only the applicable API and worker metric families are exposed');


