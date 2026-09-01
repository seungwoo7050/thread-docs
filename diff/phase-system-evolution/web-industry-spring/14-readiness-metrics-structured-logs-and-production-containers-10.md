## `docs: preserve independent E24 acceptance evidence`

diff --git a/TRACK.md b/TRACK.md
index d4d3609..7db58a3 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -4,7 +4,7 @@ Profile: `phase-1` (E01–E12 → E20 → profile E24; phase-2 deferred, E25 not
 
 Spec revision: `2ada57a71cd34fa2fae9809415c362a8bbfcdf02`
 
-The phase-1 product: Next.js/React renders login, logout, Monitor lifecycle and Check history from request-scoped server data. Spring Security authenticates browser sessions; PostgreSQL stores salted account passwords, Monitors and Check execution intents/results. Manual requests have durable owner-scoped identities, and competing workers claim one execution owner before bounded HTTP/HTTPS I/O. Monitor and CheckRun access is scoped to the signed-in user. There is no signup, Redis or broker. The E24 production-container candidate is present but not yet accepted; see the E24 status below.
+The phase-1 product: Next.js/React renders login, logout, Monitor lifecycle and Check history from request-scoped server data. Spring Security authenticates browser sessions; PostgreSQL stores salted account passwords, Monitors and Check execution intents/results. Manual requests have durable owner-scoped identities, and competing workers claim one execution owner before bounded HTTP/HTTPS I/O. Monitor and CheckRun access is scoped to the signed-in user. There is no signup, Redis or broker. The E24 production-container source passed independent root acceptance; see the E24 status below.
 
 ## Pinned toolchain
 
@@ -27,7 +27,7 @@ The phase-1 product: Next.js/React renders login, logout, Monitor lifecycle and
 | PostgreSQL JDBC | 42.7.11 | existing Spring Boot 3.5.16 BOM |
 | Spring Security core / config / web / crypto | 6.5.11 | existing Spring Boot 3.5.16 BOM; starter-security 3.5.16 |
 
-Versions remain fixed for this evolution. The Spring parent pins build plugins and transitive dependency management. The E24 candidate adds pinned Java and Node production image builds while retaining the isolated PostgreSQL project. Its complete operational acceptance remains pending.
+Versions remain fixed for this evolution. The Spring parent pins build plugins and transitive dependency management. E24 adds pinned Java and Node production image builds while retaining the isolated PostgreSQL project. Its phase-1 source acceptance is recorded below.
 
 ## Run locally
 
@@ -486,11 +486,15 @@ independent final EXPLAIN and related history regression are pending. No E11
 crash, E12 outbound, browser or load gate was repeated for E20.
 
 
-## Phase-1 operations and production-container candidate (E24)
+## Phase-1 operations and production containers (E24)
 
 Attempt5 / repair4 passed its six affected Java tests, one backend-only image
-rebuild and the complete author container gate. E24 acceptance still awaits the
-independent root gate; hosted CI execution has not run. Attempt4's metric-family
+rebuild and the complete author container gate. Independent root source acceptance
+passed at `aa5cf9b932baacbe888915d438c4982daf7d1483`: Compose configuration,
+TypeScript, six affected Java tests and the complete root container gate passed.
+The later checkpoint adds only verification evidence and this documentation;
+all115 reviewed source hashes remain unchanged. Hosted CI is `NOT_EXECUTED`.
+Attempt4's metric-family
 inventory rejection and the three earlier Maven failures remain preserved.
 The baseline ran once in the original attempt and was not repeated. No load
 scenario or automatic retry was run.
@@ -554,9 +558,12 @@ cleanup is part of this observer. Safe evidence is under
 `evidence/phase-1/E24/container/` and `evidence/phase-1/E24/repair4/`; repair4
 contains the native Java/build/container bundles and the cumulative10-invocation
 snapshot. Prior `repair3/` evidence, including the failed inventory gate and its
-source-derived diagnosis, remains unchanged. Across all attempts there are
+source-derived diagnosis, remains unchanged. The preserved author accounting has
 five affected Maven invocations, two image builds, two author container scenarios
-and four formal failures. Root's independent runtime gate is still pending.
+and four formal failures across four repairs. Root's separate passing final
+report, native logs, Java results and container copy are preserved byte-for-byte
+under `evidence/phase-1/E24/root-final/`, with a path/hash mapping manifest.
+The native container ledger retains both author entries and the root entry.
 
 CI has separate unit, integration, browser and container jobs. Its container job
 builds the actual images and invokes the same observer with actor `ci`; local
diff --git a/evidence/phase-1/E24/container/invocations.jsonl b/evidence/phase-1/E24/container/invocations.jsonl
index 5f7615e..5cc1c87 100644
--- a/evidence/phase-1/E24/container/invocations.jsonl
+++ b/evidence/phase-1/E24/container/invocations.jsonl
@@ -1,2 +1,3 @@
 {"actor":"author","head":"2dbe240409db95ac0435f8a4d4270189f60b7d82","fingerprint":"ff11785dd9b6da221b6b863102b24e38cf028f3b71790d5f8c787642f5603610","startedAt":"2026-08-28T10:43:01.107Z","command":"node scripts/e24-container.mjs author"}
 {"actor":"author","head":"c97c2320e5a0909036f8fa1e13c666cd58f0669e","fingerprint":"5464ab8650b03ea5c281bd53d32940af8d72ce2fe1b1317c3e7df562725a4b27","startedAt":"2026-08-28T11:05:08.268Z","command":"node scripts/e24-container.mjs author"}
+{"actor":"root","head":"aa5cf9b932baacbe888915d438c4982daf7d1483","fingerprint":"5464ab8650b03ea5c281bd53d32940af8d72ce2fe1b1317c3e7df562725a4b27","startedAt":"2026-08-28T11:12:25.184Z","command":"node scripts/e24-container.mjs root"}
diff --git a/evidence/phase-1/E24/container/root-5464ab8650b03ea5.json b/evidence/phase-1/E24/container/root-5464ab8650b03ea5.json
new file mode 100644
index 0000000..0245b0d
--- /dev/null
+++ b/evidence/phase-1/E24/container/root-5464ab8650b03ea5.json
@@ -0,0 +1,1755 @@
+{
+  "actor": "root",
+  "thread": "E24",
+  "profile": "phase-1",
+  "specRevision": "2ada57a71cd34fa2fae9809415c362a8bbfcdf02",
+  "start": "563b325ef871fe6d1fbfef7cf39a6581f2d0a94d",
+  "startedAt": "2026-08-28T11:12:25.184Z",
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
+      "elapsedMs": 26.514375,
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
+      "elapsedMs": 21.128041999999994,
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
+      "elapsedMs": 17.610917,
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
+      "elapsedMs": 17.710875000000016,
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
+      "elapsedMs": 71.667416,
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
+      "elapsedMs": 177.01641700000002,
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
+      "elapsedMs": 19.68866700000001,
+      "stdoutBytes": 12843,
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
+      "elapsedMs": 646.3197909999999,
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
+      "elapsedMs": 117.70800000000054,
+      "stdoutBytes": 52,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "2564eed743c9"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 48.34620799999993,
+      "stdoutBytes": 8814,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "c3770de5dc87"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 101.88887500000055,
+      "stdoutBytes": 9157,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "8ac045d9b3b8"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 93.28087499999947,
+      "stdoutBytes": 11267,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "6edc0ebcc8dc"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 49.57112499999948,
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
+      "elapsedMs": 66.45499999999993,
+      "stdoutBytes": 13,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "6edc0ebcc8dc9ffcfff9b28c0b31f19422b57526644e02bd3b537cc39d707e3b"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 40.3660830000008,
+      "stdoutBytes": 10610,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "8ac045d9b3b8a2a2bb35e8c3da247409cdf040fcee44e59d0538f8e999f6dda4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 34.53016600000046,
+      "stdoutBytes": 11267,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "c3770de5dc87e527cb24c2735687024a2a4d0c4c445d522ca8f9fade66e50c93"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 39.303249999997206,
+      "stdoutBytes": 9157,
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
+      "elapsedMs": 409.3273330000011,
+      "stdoutBytes": 0,
+      "stderrBytes": 89,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "6edc0ebcc8dc9ffcfff9b28c0b31f19422b57526644e02bd3b537cc39d707e3b"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 44.92533299999923,
+      "stdoutBytes": 10610,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "8ac045d9b3b8a2a2bb35e8c3da247409cdf040fcee44e59d0538f8e999f6dda4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 37.57154199999786,
+      "stdoutBytes": 11267,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "c3770de5dc87e527cb24c2735687024a2a4d0c4c445d522ca8f9fade66e50c93"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 37.20024999999805,
+      "stdoutBytes": 9157,
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
+      "elapsedMs": 333.46295800000007,
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
+      "elapsedMs": 44.386915999999474,
+      "stdoutBytes": 12845,
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
+      "elapsedMs": 52.418416000000434,
+      "stdoutBytes": 12845,
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
+      "elapsedMs": 56.17216700000063,
+      "stdoutBytes": 12845,
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
+      "elapsedMs": 40.9163750000007,
+      "stdoutBytes": 12845,
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
+      "elapsedMs": 62.617333999998664,
+      "stdoutBytes": 12845,
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
+      "elapsedMs": 34.91716599999927,
+      "stdoutBytes": 12845,
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
+      "elapsedMs": 33.00691699999879,
+      "stdoutBytes": 12845,
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
+      "elapsedMs": 28.99066700000185,
+      "stdoutBytes": 12845,
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
+      "elapsedMs": 21.5525000000016,
+      "stdoutBytes": 12845,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "6edc0ebcc8dc9ffcfff9b28c0b31f19422b57526644e02bd3b537cc39d707e3b"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 18.66308299999946,
+      "stdoutBytes": 10610,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "8ac045d9b3b8a2a2bb35e8c3da247409cdf040fcee44e59d0538f8e999f6dda4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 18.095916000002035,
+      "stdoutBytes": 11267,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "c3770de5dc87e527cb24c2735687024a2a4d0c4c445d522ca8f9fade66e50c93"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 17.180457999998907,
+      "stdoutBytes": 9157,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "6edc0ebcc8dc9ffcfff9b28c0b31f19422b57526644e02bd3b537cc39d707e3b"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 20.435457999999926,
+      "stdoutBytes": 10610,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "exec",
+        "6edc0ebcc8dc9ffcfff9b28c0b31f19422b57526644e02bd3b537cc39d707e3b",
+        "cat",
+        "/proc/1/status"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 53.40879099999802,
+      "stdoutBytes": 1128,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "exec",
+        "6edc0ebcc8dc9ffcfff9b28c0b31f19422b57526644e02bd3b537cc39d707e3b",
+        "java",
+        "-version"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 117.48845799999981,
+      "stdoutBytes": 0,
+      "stderrBytes": 190,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "exec",
+        "6edc0ebcc8dc9ffcfff9b28c0b31f19422b57526644e02bd3b537cc39d707e3b",
+        "sha256sum",
+        "/app/app.jar"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 261.94087500000023,
+      "stdoutBytes": 79,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "8ac045d9b3b8a2a2bb35e8c3da247409cdf040fcee44e59d0538f8e999f6dda4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 20.13145799999984,
+      "stdoutBytes": 11267,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "exec",
+        "8ac045d9b3b8a2a2bb35e8c3da247409cdf040fcee44e59d0538f8e999f6dda4",
+        "cat",
+        "/proc/1/status"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 84.7937090000014,
+      "stdoutBytes": 1128,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "exec",
+        "8ac045d9b3b8a2a2bb35e8c3da247409cdf040fcee44e59d0538f8e999f6dda4",
+        "java",
+        "-version"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 87.88441600000078,
+      "stdoutBytes": 0,
+      "stderrBytes": 190,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "exec",
+        "8ac045d9b3b8a2a2bb35e8c3da247409cdf040fcee44e59d0538f8e999f6dda4",
+        "sha256sum",
+        "/app/app.jar"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 243.38837500000227,
+      "stdoutBytes": 79,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "c3770de5dc87e527cb24c2735687024a2a4d0c4c445d522ca8f9fade66e50c93"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 20.42583399999785,
+      "stdoutBytes": 9157,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "exec",
+        "c3770de5dc87e527cb24c2735687024a2a4d0c4c445d522ca8f9fade66e50c93",
+        "cat",
+        "/proc/1/status"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 58.44133299999885,
+      "stdoutBytes": 1132,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "exec",
+        "c3770de5dc87e527cb24c2735687024a2a4d0c4c445d522ca8f9fade66e50c93",
+        "node",
+        "-e",
+        "console.log(JSON.stringify({node:process.versions.node,next:require(\"next/package.json\").version,react:require(\"react/package.json\").version,mode:process.env.NODE_ENV,origin:process.env.API_ORIGIN,manualSignal:!!process.env.NEXT_MANUAL_SIG_HANDLE}))"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 83.07404099999985,
+      "stdoutBytes": 120,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "c3770de5dc87e527cb24c2735687024a2a4d0c4c445d522ca8f9fade66e50c93"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 18.752292000001034,
+      "stdoutBytes": 9157,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "kill",
+        "--signal=SIGTERM",
+        "c3770de5dc87e527cb24c2735687024a2a4d0c4c445d522ca8f9fade66e50c93"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 26.545874999999796,
+      "stdoutBytes": 65,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "wait",
+        "c3770de5dc87e527cb24c2735687024a2a4d0c4c445d522ca8f9fade66e50c93"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 171.80375000000276,
+      "stdoutBytes": 4,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "8ac045d9b3b8a2a2bb35e8c3da247409cdf040fcee44e59d0538f8e999f6dda4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 22.56012499999997,
+      "stdoutBytes": 11267,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "kill",
+        "--signal=SIGTERM",
+        "8ac045d9b3b8a2a2bb35e8c3da247409cdf040fcee44e59d0538f8e999f6dda4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 26.61975000000166,
+      "stdoutBytes": 65,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "wait",
+        "8ac045d9b3b8a2a2bb35e8c3da247409cdf040fcee44e59d0538f8e999f6dda4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 447.42766699999993,
+      "stdoutBytes": 4,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "6edc0ebcc8dc9ffcfff9b28c0b31f19422b57526644e02bd3b537cc39d707e3b"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 26.450292000001355,
+      "stdoutBytes": 10610,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "kill",
+        "--signal=SIGTERM",
+        "6edc0ebcc8dc9ffcfff9b28c0b31f19422b57526644e02bd3b537cc39d707e3b"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 23.895999999997002,
+      "stdoutBytes": 65,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "wait",
+        "6edc0ebcc8dc9ffcfff9b28c0b31f19422b57526644e02bd3b537cc39d707e3b"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 336.93645800000013,
+      "stdoutBytes": 4,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "2564eed743c90bb5cef7246b5658e437ef38e6d6718a1e936daeaae6ec426667"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 27.75749999999971,
+      "stdoutBytes": 8814,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "kill",
+        "--signal=SIGTERM",
+        "2564eed743c90bb5cef7246b5658e437ef38e6d6718a1e936daeaae6ec426667"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 28.53387500000099,
+      "stdoutBytes": 65,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "wait",
+        "2564eed743c90bb5cef7246b5658e437ef38e6d6718a1e936daeaae6ec426667"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 54.952666999997746,
+      "stdoutBytes": 4,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "logs",
+        "6edc0ebcc8dc9ffcfff9b28c0b31f19422b57526644e02bd3b537cc39d707e3b"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 29.957542000000103,
+      "stdoutBytes": 48006,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "logs",
+        "8ac045d9b3b8a2a2bb35e8c3da247409cdf040fcee44e59d0538f8e999f6dda4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 20.380583000001934,
+      "stdoutBytes": 27979,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "logs",
+        "c3770de5dc87e527cb24c2735687024a2a4d0c4c445d522ca8f9fade66e50c93"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 21.188249999999243,
+      "stdoutBytes": 147,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "logs",
+        "2564eed743c90bb5cef7246b5658e437ef38e6d6718a1e936daeaae6ec426667"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 20.601833000000624,
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
+      "elapsedMs": 27.693124999997963,
+      "stdoutBytes": 52,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "2564eed743c9"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 18.656999999999243,
+      "stdoutBytes": 8822,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "c3770de5dc87"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 18.99554200000057,
+      "stdoutBytes": 8769,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "8ac045d9b3b8"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 19.78425000000061,
+      "stdoutBytes": 10561,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "6edc0ebcc8dc"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 19.592666999997164,
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
+      "elapsedMs": 16.455792000000656,
+      "stdoutBytes": 13,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "2564eed743c90bb5cef7246b5658e437ef38e6d6718a1e936daeaae6ec426667"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 16.55645799999911,
+      "stdoutBytes": 8822,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "rm",
+        "2564eed743c90bb5cef7246b5658e437ef38e6d6718a1e936daeaae6ec426667"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 34.504000000000815,
+      "stdoutBytes": 65,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "c3770de5dc87e527cb24c2735687024a2a4d0c4c445d522ca8f9fade66e50c93"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 25.951624999997875,
+      "stdoutBytes": 8769,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "rm",
+        "c3770de5dc87e527cb24c2735687024a2a4d0c4c445d522ca8f9fade66e50c93"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 37.452792000000045,
+      "stdoutBytes": 65,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "8ac045d9b3b8a2a2bb35e8c3da247409cdf040fcee44e59d0538f8e999f6dda4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 32.005999999997584,
+      "stdoutBytes": 10561,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "rm",
+        "8ac045d9b3b8a2a2bb35e8c3da247409cdf040fcee44e59d0538f8e999f6dda4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 56.38225000000239,
+      "stdoutBytes": 65,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "6edc0ebcc8dc9ffcfff9b28c0b31f19422b57526644e02bd3b537cc39d707e3b"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 41.92250000000058,
+      "stdoutBytes": 10087,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "rm",
+        "6edc0ebcc8dc9ffcfff9b28c0b31f19422b57526644e02bd3b537cc39d707e3b"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 78.61891599999944,
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
+      "elapsedMs": 40.234042000000045,
+      "stdoutBytes": 1093,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "network",
+        "rm",
+        "a5ce5854e4b8444a6af2b62619a3e78123903c2f5b4494b636c57541cc4f3fb5"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 265.0152080000007,
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
+      "elapsedMs": 52.35337499999878,
+      "stdoutBytes": 12845,
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
+      "elapsedMs": 171.8413330000003,
+      "awaited": true,
+      "observerTimedOut": false,
+      "forcedContainerKill": false
+    },
+    "worker": {
+      "purpose": "frozen production signal acceptance",
+      "signal": "SIGTERM",
+      "signalCommandExit": 0,
+      "exitCode": 143,
+      "elapsedMs": 447.5100829999974,
+      "awaited": true,
+      "observerTimedOut": false,
+      "forcedContainerKill": false
+    },
+    "api": {
+      "purpose": "frozen production signal acceptance",
+      "signal": "SIGTERM",
+      "signalCommandExit": 0,
+      "exitCode": 143,
+      "elapsedMs": 336.9692920000016,
+      "awaited": true,
+      "observerTimedOut": false,
+      "forcedContainerKill": false
+    },
+    "fixture": {
+      "purpose": "owned fixture cleanup",
+      "signal": "SIGTERM",
+      "signalCommandExit": 0,
+      "exitCode": 143,
+      "elapsedMs": 54.98158400000102,
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
+  "head": "aa5cf9b932baacbe888915d438c4982daf7d1483",
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
+      "localObservationAndReleaseMs": 11.655416999998124,
+      "dockerOrSqlBeforeRelease": false
+    },
+    "fixture": {
+      "outboundCalls": 1,
+      "holdRequests": 1,
+      "held": 0,
+      "releases": 1,
+      "watchdogReleases": 0,
+      "lastHeldMs": 25.01008399999955
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
+      "TOTAL_TIME": 3.3804640029999997,
+      "MAX": 1.021714042
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
+      "id": "6edc0ebcc8dc9ffcfff9b28c0b31f19422b57526644e02bd3b537cc39d707e3b",
+      "image": "sha256:d6f6ab129f0199ab6cb92ae12ee40fb83dba92bc40d9421a16f874b0265fbcc1",
+      "pid": 85959,
+      "startedAt": "2026-08-28T11:12:31.152108291Z",
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
+      "id": "8ac045d9b3b8a2a2bb35e8c3da247409cdf040fcee44e59d0538f8e999f6dda4",
+      "image": "sha256:d6f6ab129f0199ab6cb92ae12ee40fb83dba92bc40d9421a16f874b0265fbcc1",
+      "pid": 85960,
+      "startedAt": "2026-08-28T11:12:31.156308375Z",
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
+      "id": "c3770de5dc87e527cb24c2735687024a2a4d0c4c445d522ca8f9fade66e50c93",
+      "image": "sha256:9356a3b65bd58be8e12f5a4721975f4801e1601f7dbef0a4bb7ddfd6ef93ac9e",
+      "pid": 85961,
+      "startedAt": "2026-08-28T11:12:31.154399416Z",
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
+        "lines": 47,
+        "structuredLines": 47,
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
+      "49ef3919-d659-4c4b-9ecf-617be9513423",
+      "8b6bfb74-f6ef-4d02-8eef-ccebd3c14008",
+      "d20469fa-f525-4fdc-9b87-4aa9c79cc287",
+      "c27ce740-4b0a-4881-a29e-d0674ff9e627",
+      "0232a72a-30cb-42df-97a1-4c295ff642f8",
+      "76ae6e6f-153c-4b9f-8eb8-044b5208e6db",
+      "c13834a1-69a1-4732-ad8c-6b46aa8a5be3",
+      "e46fe443-f83f-4646-99bb-441cbe1c893b",
+      "08b0951e-6780-468d-bb7f-46757f9635b1",
+      "d23790f6-bdd0-4f03-8cd4-e347b9dfb280",
+      "db742755-1a62-4a8e-9a8d-e789cab8f661",
+      "880039c6-3d01-44f8-9b2c-30bc5eae7e97",
+      "fa7ad054-5f11-4259-80bc-375cdeca0436",
+      "b8bed647-fd6a-42e7-84bb-554200b14a88",
+      "171a537c-46f5-4b2d-8786-b94adfecdb38",
+      "7c1cc562-33ae-46bd-a70d-7d64a7275624",
+      "75affb21-7d38-4583-847f-af6b6c19997d",
+      "d6a944fd-b7cf-4704-9117-76baa40f410e"
+    ],
+    "stableApiProcess": true,
+    "stableWorkerProcess": true,
+    "distinctRoleProcesses": true,
+    "apiProcessId": "2d377846-c2d7-4584-b60c-9954394485fe",
+    "workerProcessId": "e11835ca-086b-47d1-b3a4-c8716f58fb8d",
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
+  "elapsedSeconds": 26.942,
+  "hostedCiExecutionClaimed": false
+}
diff --git a/evidence/phase-1/E24/container/root-5464ab8650b03ea5.started.json b/evidence/phase-1/E24/container/root-5464ab8650b03ea5.started.json
new file mode 100644
index 0000000..ca8944f
--- /dev/null
+++ b/evidence/phase-1/E24/container/root-5464ab8650b03ea5.started.json
@@ -0,0 +1 @@
+{"actor":"root","head":"aa5cf9b932baacbe888915d438c4982daf7d1483","fingerprint":"5464ab8650b03ea5c281bd53d32940af8d72ce2fe1b1317c3e7df562725a4b27","startedAt":"2026-08-28T11:12:25.184Z"}
diff --git a/evidence/phase-1/E24/root-final/manifest.json b/evidence/phase-1/E24/root-final/manifest.json
new file mode 100644
index 0000000..b5155a9
--- /dev/null
+++ b/evidence/phase-1/E24/root-final/manifest.json
@@ -0,0 +1,167 @@
+{
+  "thread": "E24",
+  "profile": "phase-1",
+  "attempt": 5,
+  "repair": 4,
+  "specRevision": "2ada57a71cd34fa2fae9809415c362a8bbfcdf02",
+  "acceptedSourceCandidate": "aa5cf9b932baacbe888915d438c4982daf7d1483",
+  "scope": "Verification evidence and TRACK documentation only; no source or product gate changes",
+  "originalRoot": "/Users/woopinbell/Desktop/working/workflow/web-systems-evolution",
+  "copies": [
+    {
+      "original": "index/profiles/phase-1/verification/industry-spring/E24-artifacts/root-final-attempt5.json",
+      "copy": "evidence/phase-1/E24/root-final/root-final-attempt5.json",
+      "sha256": "e179cf72c9fd19b6d4f6434da02e91808ce414e582273a6a63fced1a433f85a9",
+      "bytes": 5884
+    },
+    {
+      "original": "index/profiles/phase-1/verification/industry-spring/E24-artifacts/root-reviewed-final-source-sha256.json",
+      "copy": "evidence/phase-1/E24/root-final/root-reviewed-final-source-sha256.json",
+      "sha256": "9d8043f7260c3f7ebfea918f2ad1f32bf7c44985b670756b870575a6e9d3d8be",
+      "bytes": 13446
+    },
+    {
+      "original": "index/profiles/phase-1/verification/industry-spring/E24-artifacts/root-final-attempt5-compose-config.log",
+      "copy": "evidence/phase-1/E24/root-final/root-final-attempt5-compose-config.log",
+      "sha256": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
+      "bytes": 0
+    },
+    {
+      "original": "index/profiles/phase-1/verification/industry-spring/E24-artifacts/root-final-attempt5-typecheck.log",
+      "copy": "evidence/phase-1/E24/root-final/root-final-attempt5-typecheck.log",
+      "sha256": "2e2cf148badf4fe2a73749b44b300bdcf32682947e228fd9c498c31cf63989e0",
+      "bytes": 134
+    },
+    {
+      "original": "index/profiles/phase-1/verification/industry-spring/E24-artifacts/root-final-attempt5-affected-java.log",
+      "copy": "evidence/phase-1/E24/root-final/root-final-attempt5-affected-java.log.native.json",
+      "sha256": "a64fca20a45528dd33e8f12123d5ae4644b2dab3e267c5fc778f402eac14e4d1",
+      "bytes": 16237,
+      "encoding": "json-utf8-raw",
+      "physicalSha256": "43ddc3ff48f67655ca230746aa6ff9eb414ef58b61da11612fcc3cd290247ab0",
+      "physicalBytes": 18238
+    },
+    {
+      "original": "index/profiles/phase-1/verification/industry-spring/E24-artifacts/root-final-attempt5-full-container.log",
+      "copy": "evidence/phase-1/E24/root-final/root-final-attempt5-full-container.log",
+      "sha256": "4eafc7062153fcd5aa22220b956813cf635687d6c7c484b8e4c1a7d3fe26b274",
+      "bytes": 168
+    },
+    {
+      "original": "index/profiles/phase-1/verification/industry-spring/E24-artifacts/root-final-attempt5-TEST-dev.evolution.monitor.HttpObservationsTest.xml",
+      "copy": "evidence/phase-1/E24/root-final/root-final-attempt5-TEST-dev.evolution.monitor.HttpObservationsTest.xml",
+      "sha256": "5cd6a0f2530cacac738dd7a23f9b51c57de2d6ee2e903518fd165f35d3b839f0",
+      "bytes": 36702
+    },
+    {
+      "original": "index/profiles/phase-1/verification/industry-spring/E24-artifacts/root-final-attempt5-dev.evolution.monitor.HttpObservationsTest.txt",
+      "copy": "evidence/phase-1/E24/root-final/root-final-attempt5-dev.evolution.monitor.HttpObservationsTest.txt",
+      "sha256": "7b3f87a42ab895826d11e4f3d5753d6cb3e53c42421a1629d0bf2f4213de784e",
+      "bytes": 334
+    },
+    {
+      "original": "index/profiles/phase-1/verification/industry-spring/E24-artifacts/root-final-attempt5-TEST-dev.evolution.monitor.OperationsIntegrationTest.xml",
+      "copy": "evidence/phase-1/E24/root-final/root-final-attempt5-TEST-dev.evolution.monitor.OperationsIntegrationTest.xml",
+      "sha256": "594fa44d28cffa1dfaf40f19e78232b393aabb591009f57a0531545d785e8000",
+      "bytes": 47622
+    },
+    {
+      "original": "index/profiles/phase-1/verification/industry-spring/E24-artifacts/root-final-attempt5-dev.evolution.monitor.OperationsIntegrationTest.txt",
+      "copy": "evidence/phase-1/E24/root-final/root-final-attempt5-dev.evolution.monitor.OperationsIntegrationTest.txt",
+      "sha256": "ce15555041499d56c8dd09ee6164df2510e382eeb54e377040a15c7a898766e0",
+      "bytes": 344
+    },
+    {
+      "original": "index/profiles/phase-1/verification/industry-spring/E24-artifacts/root-final-attempt5-TEST-dev.evolution.monitor.ApiErrorBoundaryTest.xml",
+      "copy": "evidence/phase-1/E24/root-final/root-final-attempt5-TEST-dev.evolution.monitor.ApiErrorBoundaryTest.xml",
+      "sha256": "495fb0620622e9ffafdfeeffeadbea0071e5ff4b7432ffb0af78659a2c34fb3a",
+      "bytes": 37632
+    },
+    {
+      "original": "index/profiles/phase-1/verification/industry-spring/E24-artifacts/root-final-attempt5-dev.evolution.monitor.ApiErrorBoundaryTest.txt",
+      "copy": "evidence/phase-1/E24/root-final/root-final-attempt5-dev.evolution.monitor.ApiErrorBoundaryTest.txt",
+      "sha256": "374ee2091e9a3a9322fd659c36ac00ff687fbfaf49efda7b9aa703de631b6ce5",
+      "bytes": 334
+    },
+    {
+      "original": "index/profiles/phase-1/verification/industry-spring/E24-artifacts/root-final-attempt5-e24-metrics.json",
+      "copy": "evidence/phase-1/E24/root-final/root-final-attempt5-e24-metrics.json",
+      "sha256": "2156f5970e15f9fc2213bfb19d8033e9004f36cb0e8403ded5012f91a2f727d8",
+      "bytes": 187
+    },
+    {
+      "original": "index/profiles/phase-1/verification/industry-spring/E24-artifacts/root-final-attempt5-container-native.json",
+      "copy": "evidence/phase-1/E24/root-final/root-final-attempt5-container-native.json",
+      "sha256": "bc412d5d0a607297e795352a716663f2701ee1db35f70ad1148d760d21ec67f4",
+      "bytes": 43613
+    }
+  ],
+  "sourceVerification": {
+    "manifestCopy": "evidence/phase-1/E24/root-final/root-reviewed-final-source-sha256.json",
+    "sha256": "9d8043f7260c3f7ebfea918f2ad1f32bf7c44985b670756b870575a6e9d3d8be",
+    "files": 115,
+    "allHashesUnchanged": true
+  },
+  "branchNativeFiles": [
+    {
+      "path": "evidence/phase-1/E24/container/root-5464ab8650b03ea5.json",
+      "sha256": "bc412d5d0a607297e795352a716663f2701ee1db35f70ad1148d760d21ec67f4",
+      "bytes": 43613
+    },
+    {
+      "path": "evidence/phase-1/E24/container/root-5464ab8650b03ea5.started.json",
+      "sha256": "6bcda2bc509646e1f5ba4835880061942940db0154c2f7d5281a7633983a323b",
+      "bytes": 187
+    }
+  ],
+  "containerLedger": {
+    "path": "evidence/phase-1/E24/container/invocations.jsonl",
+    "sha256": "494bc4c75175a647b86bbc751a8d64fd7c477f55207805be5fa744ed7f568c0c",
+    "priorCandidate": "aa5cf9b932baacbe888915d438c4982daf7d1483",
+    "priorBytes": 478,
+    "priorSha256": "c698afd0e0312c0ad80929679c048bbd22c1578a3a77d1df6355e3672a9b7cb3",
+    "priorPrefixPreserved": true,
+    "newEntries": [
+      {
+        "actor": "root",
+        "head": "aa5cf9b932baacbe888915d438c4982daf7d1483",
+        "fingerprint": "5464ab8650b03ea5c281bd53d32940af8d72ce2fe1b1317c3e7df562725a4b27",
+        "startedAt": "2026-08-28T11:12:25.184Z",
+        "command": "node scripts/e24-container.mjs root"
+      }
+    ],
+    "totalEntries": 3
+  },
+  "rootVerification": {
+    "status": "PASS",
+    "commands": 4,
+    "javaTests": 6,
+    "elapsedSeconds": 38.977,
+    "sourceCandidate": "aa5cf9b932baacbe888915d438c4982daf7d1483",
+    "imageBuilds": 0,
+    "baselineRuns": 0,
+    "automaticRetries": 0
+  },
+  "priorAuthorAccounting": {
+    "path": "evidence/phase-1/E24/repair4/repair4-preservation.json",
+    "sha256": "20229ce990069add2121c4c6c9fdb1b029e4db73238fc334c82697e2cde5b0a3",
+    "formalFailures": 4,
+    "repairs": 4,
+    "baselineRuns": 1,
+    "unchanged": true
+  },
+  "thisEvidenceTask": {
+    "productGates": 0,
+    "builds": 0,
+    "runtimeInvocations": 0,
+    "sourceChanges": 0,
+    "historicalNativeBytesRewritten": false
+  },
+  "hostedCi": "NOT_EXECUTED",
+  "provenance": "Copied reports retain their original paths. One native Maven log is encoded as a JSON UTF-8 raw string whose decoded bytes match the original exactly; the other13 files are direct byte copies. The mapping records physical branch-local paths without changing original content or historical references.",
+  "packaging": {
+    "exactByteCopies": 13,
+    "jsonUtf8RawCopies": 1,
+    "decodedOriginalBytesUnchanged": true
+  }
+}
diff --git a/evidence/phase-1/E24/root-final/root-final-attempt5-TEST-dev.evolution.monitor.ApiErrorBoundaryTest.xml b/evidence/phase-1/E24/root-final/root-final-attempt5-TEST-dev.evolution.monitor.ApiErrorBoundaryTest.xml
new file mode 100644
index 0000000..d3e6d9a
--- /dev/null
+++ b/evidence/phase-1/E24/root-final/root-final-attempt5-TEST-dev.evolution.monitor.ApiErrorBoundaryTest.xml
@@ -0,0 +1,82 @@
+<?xml version="1.0" encoding="UTF-8"?>
+<testsuite xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="https://maven.apache.org/surefire/maven-surefire-plugin/xsd/surefire-test-report.xsd" version="3.0.2" name="dev.evolution.monitor.ApiErrorBoundaryTest" time="1.264" tests="2" errors="0" skipped="0" failures="0" flakes="0">
+  <properties>
+    <property name="java.specification.version" value="21"/>
+    <property name="sun.jnu.encoding" value="UTF-8"/>
+    <property name="java.class.path" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/test-classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-web/3.5.16/spring-boot-starter-web-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter/3.5.16/spring-boot-starter-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot/3.5.16/spring-boot-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-autoconfigure/3.5.16/spring-boot-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-logging/3.5.16/spring-boot-starter-logging-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-classic/1.5.34/logback-classic-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-core/1.5.34/logback-core-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-to-slf4j/2.24.3/log4j-to-slf4j-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-api/2.24.3/log4j-api-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/jul-to-slf4j/2.0.18/jul-to-slf4j-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/annotation/jakarta.annotation-api/2.1.1/jakarta.annotation-api-2.1.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/yaml/snakeyaml/2.4/snakeyaml-2.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-json/3.5.16/spring-boot-starter-json-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-databind/2.21.4/jackson-databind-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-annotations/2.21/jackson-annotations-2.21.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-core/2.21.4/jackson-core-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jdk8/2.21.4/jackson-datatype-jdk8-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jsr310/2.21.4/jackson-datatype-jsr310-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/module/jackson-module-parameter-names/2.21.4/jackson-module-parameter-names-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-tomcat/3.5.16/spring-boot-starter-tomcat-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-core/10.1.55/tomcat-embed-core-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-el/10.1.55/tomcat-embed-el-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-websocket/10.1.55/tomcat-embed-websocket-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-web/6.2.19/spring-web-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-beans/6.2.19/spring-beans-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-webmvc/6.2.19/spring-webmvc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-context/6.2.19/spring-context-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-expression/6.2.19/spring-expression-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-data-jpa/3.5.16/spring-boot-starter-data-jpa-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-jdbc/3.5.16/spring-boot-starter-jdbc-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/zaxxer/HikariCP/6.3.3/HikariCP-6.3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jdbc/6.2.19/spring-jdbc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/orm/hibernate-core/6.6.53.Final/hibernate-core-6.6.53.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/persistence/jakarta.persistence-api/3.1.0/jakarta.persistence-api-3.1.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/transaction/jakarta.transaction-api/2.0.1/jakarta.transaction-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/jboss/logging/jboss-logging/3.6.3.Final/jboss-logging-3.6.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/common/hibernate-commons-annotations/7.0.3.Final/hibernate-commons-annotations-7.0.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/smallrye/jandex/3.2.0/jandex-3.2.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/classmate/1.7.3/classmate-1.7.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy/1.17.8/byte-buddy-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-runtime/4.0.9/jaxb-runtime-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-core/4.0.9/jaxb-core-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/eclipse/angus/angus-activation/2.0.3/angus-activation-2.0.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/txw2/4.0.9/txw2-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/sun/istack/istack-commons-runtime/4.1.2/istack-commons-runtime-4.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/inject/jakarta.inject-api/2.0.1/jakarta.inject-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/antlr/antlr4-runtime/4.13.2/antlr4-runtime-4.13.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-jpa/3.5.13/spring-data-jpa-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-commons/3.5.13/spring-data-commons-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-orm/6.2.19/spring-orm-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-tx/6.2.19/spring-tx-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/slf4j-api/2.0.18/slf4j-api-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aspects/6.2.19/spring-aspects-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/aspectj/aspectjweaver/1.9.25.1/aspectjweaver-1.9.25.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-security/3.5.16/spring-boot-starter-security-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aop/6.2.19/spring-aop-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-config/6.5.11/spring-security-config-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-core/6.5.11/spring-security-core-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-crypto/6.5.11/spring-security-crypto-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-web/6.5.11/spring-security-web-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-actuator/3.5.16/spring-boot-starter-actuator-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-actuator-autoconfigure/3.5.16/spring-boot-actuator-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-actuator/3.5.16/spring-boot-actuator-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-observation/1.15.12/micrometer-observation-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-commons/1.15.12/micrometer-commons-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-jakarta9/1.15.12/micrometer-jakarta9-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-core/1.15.12/micrometer-core-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hdrhistogram/HdrHistogram/2.2.2/HdrHistogram-2.2.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/latencyutils/LatencyUtils/2.0.3/LatencyUtils-2.0.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-database-postgresql/11.7.2/flyway-database-postgresql-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-core/11.7.2/flyway-core-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/dataformat/jackson-dataformat-toml/2.21.4/jackson-dataformat-toml-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/postgresql/postgresql/42.7.11/postgresql-42.7.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-test/3.5.16/spring-boot-starter-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test/3.5.16/spring-boot-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test-autoconfigure/3.5.16/spring-boot-test-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/jayway/jsonpath/json-path/2.9.0/json-path-2.9.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/xml/bind/jakarta.xml.bind-api/4.0.5/jakarta.xml.bind-api-4.0.5.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/activation/jakarta.activation-api/2.1.4/jakarta.activation-api-2.1.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/json-smart/2.5.2/json-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/accessors-smart/2.5.2/accessors-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/ow2/asm/asm/9.7.1/asm-9.7.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/assertj/assertj-core/3.27.7/assertj-core-3.27.7.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/awaitility/awaitility/4.2.2/awaitility-4.2.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hamcrest/hamcrest/3.0/hamcrest-3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter/5.12.2/junit-jupiter-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-api/5.12.2/junit-jupiter-api-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/opentest4j/opentest4j/1.3.0/opentest4j-1.3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-commons/1.12.2/junit-platform-commons-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apiguardian/apiguardian-api/1.1.2/apiguardian-api-1.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-params/5.12.2/junit-jupiter-params-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-engine/5.12.2/junit-jupiter-engine-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-engine/1.12.2/junit-platform-engine-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-core/5.17.0/mockito-core-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy-agent/1.17.8/byte-buddy-agent-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/objenesis/objenesis/3.3/objenesis-3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-junit-jupiter/5.17.0/mockito-junit-jupiter-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/skyscreamer/jsonassert/1.5.3/jsonassert-1.5.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/vaadin/external/google/android-json/0.0.20131108.vaadin1/android-json-0.0.20131108.vaadin1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-core/6.2.19/spring-core-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jcl/6.2.19/spring-jcl-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-test/6.2.19/spring-test-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/xmlunit/xmlunit-core/2.10.4/xmlunit-core-2.10.4.jar:"/>
+    <property name="java.vm.vendor" value="Eclipse Adoptium"/>
+    <property name="sun.arch.data.model" value="64"/>
+    <property name="java.vendor.url" value="https://adoptium.net/"/>
+    <property name="user.timezone" value="Asia/Seoul"/>
+    <property name="os.name" value="Mac OS X"/>
+    <property name="java.vm.specification.version" value="21"/>
+    <property name="sun.java.launcher" value="SUN_STANDARD"/>
+    <property name="user.country" value="KR"/>
+    <property name="sun.boot.library.path" value="/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem/lib"/>
+    <property name="sun.java.command" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/surefire/surefirebooter-20260828201217920_3.jar /private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/surefire 2026-08-28T20-12-17_843-jvmRun1 surefire-20260828201217920_1tmp surefire_0-20260828201217920_2tmp"/>
+    <property name="http.nonProxyHosts" value="local|*.local|169.254/16|*.169.254/16"/>
+    <property name="jdk.debug" value="release"/>
+    <property name="test" value="HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest"/>
+    <property name="surefire.test.class.path" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/test-classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-web/3.5.16/spring-boot-starter-web-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter/3.5.16/spring-boot-starter-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot/3.5.16/spring-boot-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-autoconfigure/3.5.16/spring-boot-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-logging/3.5.16/spring-boot-starter-logging-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-classic/1.5.34/logback-classic-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-core/1.5.34/logback-core-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-to-slf4j/2.24.3/log4j-to-slf4j-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-api/2.24.3/log4j-api-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/jul-to-slf4j/2.0.18/jul-to-slf4j-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/annotation/jakarta.annotation-api/2.1.1/jakarta.annotation-api-2.1.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/yaml/snakeyaml/2.4/snakeyaml-2.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-json/3.5.16/spring-boot-starter-json-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-databind/2.21.4/jackson-databind-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-annotations/2.21/jackson-annotations-2.21.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-core/2.21.4/jackson-core-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jdk8/2.21.4/jackson-datatype-jdk8-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jsr310/2.21.4/jackson-datatype-jsr310-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/module/jackson-module-parameter-names/2.21.4/jackson-module-parameter-names-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-tomcat/3.5.16/spring-boot-starter-tomcat-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-core/10.1.55/tomcat-embed-core-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-el/10.1.55/tomcat-embed-el-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-websocket/10.1.55/tomcat-embed-websocket-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-web/6.2.19/spring-web-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-beans/6.2.19/spring-beans-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-webmvc/6.2.19/spring-webmvc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-context/6.2.19/spring-context-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-expression/6.2.19/spring-expression-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-data-jpa/3.5.16/spring-boot-starter-data-jpa-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-jdbc/3.5.16/spring-boot-starter-jdbc-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/zaxxer/HikariCP/6.3.3/HikariCP-6.3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jdbc/6.2.19/spring-jdbc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/orm/hibernate-core/6.6.53.Final/hibernate-core-6.6.53.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/persistence/jakarta.persistence-api/3.1.0/jakarta.persistence-api-3.1.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/transaction/jakarta.transaction-api/2.0.1/jakarta.transaction-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/jboss/logging/jboss-logging/3.6.3.Final/jboss-logging-3.6.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/common/hibernate-commons-annotations/7.0.3.Final/hibernate-commons-annotations-7.0.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/smallrye/jandex/3.2.0/jandex-3.2.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/classmate/1.7.3/classmate-1.7.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy/1.17.8/byte-buddy-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-runtime/4.0.9/jaxb-runtime-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-core/4.0.9/jaxb-core-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/eclipse/angus/angus-activation/2.0.3/angus-activation-2.0.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/txw2/4.0.9/txw2-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/sun/istack/istack-commons-runtime/4.1.2/istack-commons-runtime-4.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/inject/jakarta.inject-api/2.0.1/jakarta.inject-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/antlr/antlr4-runtime/4.13.2/antlr4-runtime-4.13.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-jpa/3.5.13/spring-data-jpa-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-commons/3.5.13/spring-data-commons-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-orm/6.2.19/spring-orm-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-tx/6.2.19/spring-tx-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/slf4j-api/2.0.18/slf4j-api-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aspects/6.2.19/spring-aspects-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/aspectj/aspectjweaver/1.9.25.1/aspectjweaver-1.9.25.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-security/3.5.16/spring-boot-starter-security-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aop/6.2.19/spring-aop-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-config/6.5.11/spring-security-config-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-core/6.5.11/spring-security-core-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-crypto/6.5.11/spring-security-crypto-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-web/6.5.11/spring-security-web-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-actuator/3.5.16/spring-boot-starter-actuator-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-actuator-autoconfigure/3.5.16/spring-boot-actuator-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-actuator/3.5.16/spring-boot-actuator-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-observation/1.15.12/micrometer-observation-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-commons/1.15.12/micrometer-commons-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-jakarta9/1.15.12/micrometer-jakarta9-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-core/1.15.12/micrometer-core-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hdrhistogram/HdrHistogram/2.2.2/HdrHistogram-2.2.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/latencyutils/LatencyUtils/2.0.3/LatencyUtils-2.0.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-database-postgresql/11.7.2/flyway-database-postgresql-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-core/11.7.2/flyway-core-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/dataformat/jackson-dataformat-toml/2.21.4/jackson-dataformat-toml-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/postgresql/postgresql/42.7.11/postgresql-42.7.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-test/3.5.16/spring-boot-starter-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test/3.5.16/spring-boot-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test-autoconfigure/3.5.16/spring-boot-test-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/jayway/jsonpath/json-path/2.9.0/json-path-2.9.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/xml/bind/jakarta.xml.bind-api/4.0.5/jakarta.xml.bind-api-4.0.5.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/activation/jakarta.activation-api/2.1.4/jakarta.activation-api-2.1.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/json-smart/2.5.2/json-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/accessors-smart/2.5.2/accessors-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/ow2/asm/asm/9.7.1/asm-9.7.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/assertj/assertj-core/3.27.7/assertj-core-3.27.7.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/awaitility/awaitility/4.2.2/awaitility-4.2.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hamcrest/hamcrest/3.0/hamcrest-3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter/5.12.2/junit-jupiter-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-api/5.12.2/junit-jupiter-api-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/opentest4j/opentest4j/1.3.0/opentest4j-1.3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-commons/1.12.2/junit-platform-commons-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apiguardian/apiguardian-api/1.1.2/apiguardian-api-1.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-params/5.12.2/junit-jupiter-params-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-engine/5.12.2/junit-jupiter-engine-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-engine/1.12.2/junit-platform-engine-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-core/5.17.0/mockito-core-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy-agent/1.17.8/byte-buddy-agent-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/objenesis/objenesis/3.3/objenesis-3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-junit-jupiter/5.17.0/mockito-junit-jupiter-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/skyscreamer/jsonassert/1.5.3/jsonassert-1.5.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/vaadin/external/google/android-json/0.0.20131108.vaadin1/android-json-0.0.20131108.vaadin1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-core/6.2.19/spring-core-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jcl/6.2.19/spring-jcl-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-test/6.2.19/spring-test-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/xmlunit/xmlunit-core/2.10.4/xmlunit-core-2.10.4.jar:"/>
+    <property name="sun.cpu.endian" value="little"/>
+    <property name="user.home" value="/Users/woopinbell"/>
+    <property name="user.language" value="ko"/>
+    <property name="java.specification.vendor" value="Oracle Corporation"/>
+    <property name="java.version.date" value="2025-04-15"/>
+    <property name="java.home" value="/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"/>
+    <property name="file.separator" value="/"/>
+    <property name="basedir" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend"/>
+    <property name="java.vm.compressedOopsMode" value="Zero based"/>
+    <property name="line.separator" value="&#10;"/>
+    <property name="java.vm.specification.vendor" value="Oracle Corporation"/>
+    <property name="java.specification.name" value="Java Platform API Specification"/>
+    <property name="apple.awt.application.name" value="ForkedBooter"/>
+    <property name="surefire.real.class.path" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/surefire/surefirebooter-20260828201217920_3.jar"/>
+    <property name="sun.management.compiler" value="HotSpot 64-Bit Tiered Compilers"/>
+    <property name="ftp.nonProxyHosts" value="local|*.local|169.254/16|*.169.254/16"/>
+    <property name="java.runtime.version" value="21.0.7+6-LTS"/>
+    <property name="user.name" value="woopinbell"/>
+    <property name="stdout.encoding" value="UTF-8"/>
+    <property name="path.separator" value=":"/>
+    <property name="os.version" value="26.6.2"/>
+    <property name="java.runtime.name" value="OpenJDK Runtime Environment"/>
+    <property name="file.encoding" value="UTF-8"/>
+    <property name="java.vm.name" value="OpenJDK 64-Bit Server VM"/>
+    <property name="java.vendor.version" value="Temurin-21.0.7+6"/>
+    <property name="localRepository" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository"/>
+    <property name="java.vendor.url.bug" value="https://github.com/adoptium/adoptium-support/issues"/>
+    <property name="java.io.tmpdir" value="/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/"/>
+    <property name="java.version" value="21.0.7"/>
+    <property name="user.dir" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend"/>
+    <property name="os.arch" value="aarch64"/>
+    <property name="java.vm.specification.name" value="Java Virtual Machine Specification"/>
+    <property name="native.encoding" value="UTF-8"/>
+    <property name="java.library.path" value="/Users/woopinbell/Library/Java/Extensions:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java:."/>
+    <property name="java.vm.info" value="mixed mode, sharing"/>
+    <property name="stderr.encoding" value="UTF-8"/>
+    <property name="java.vendor" value="Eclipse Adoptium"/>
+    <property name="java.vm.version" value="21.0.7+6-LTS"/>
+    <property name="sun.io.unicode.encoding" value="UnicodeBig"/>
+    <property name="maven.repo.local" value=".m2/repository"/>
+    <property name="socksNonProxyHosts" value="local|*.local|169.254/16|*.169.254/16"/>
+    <property name="java.class.version" value="65.0"/>
+  </properties>
+  <testcase name="unexpectedFailureIsAnInternalErrorWithoutPrivateExceptionDetails" classname="dev.evolution.monitor.ApiErrorBoundaryTest" time="1.231">
+    <system-out><![CDATA[20:12:19.461 [main] INFO org.springframework.mock.web.MockServletContext -- Initializing Spring TestDispatcherServlet ''
+20:12:19.462 [main] INFO org.springframework.test.web.servlet.TestDispatcherServlet -- Initializing Servlet ''
+20:12:19.465 [main] INFO org.springframework.test.web.servlet.TestDispatcherServlet -- Completed initialization in 2 ms
+]]></system-out>
+    <system-err><![CDATA[Mockito is currently self-attaching to enable the inline-mock-maker. This will no longer work in future releases of the JDK. Please add Mockito as an agent to your build as described in Mockito's documentation: https://javadoc.io/doc/org.mockito/mockito-core/latest/org.mockito/org/mockito/Mockito.html#0.3
+WARNING: A Java agent has been loaded dynamically (/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy-agent/1.17.8/byte-buddy-agent-1.17.8.jar)
+WARNING: If a serviceability tool is in use, please run with -XX:+EnableDynamicAgentLoading to hide this warning
+WARNING: If a serviceability tool is not in use, please run with -Djdk.instrument.traceUsage for more information
+WARNING: Dynamic loading of agents will be disallowed by default in a future release
+]]></system-err>
+  </testcase>
+  <testcase name="unavailableAuthorityRejectsTheRequestWithoutPrivateConnectionDetails" classname="dev.evolution.monitor.ApiErrorBoundaryTest" time="0.02">
+    <system-out><![CDATA[20:12:19.656 [main] INFO org.springframework.mock.web.MockServletContext -- Initializing Spring TestDispatcherServlet ''
+20:12:19.656 [main] INFO org.springframework.test.web.servlet.TestDispatcherServlet -- Initializing Servlet ''
+20:12:19.656 [main] INFO org.springframework.test.web.servlet.TestDispatcherServlet -- Completed initialization in 0 ms
+]]></system-out>
+  </testcase>
+</testsuite>
\ No newline at end of file
diff --git a/evidence/phase-1/E24/root-final/root-final-attempt5-TEST-dev.evolution.monitor.HttpObservationsTest.xml b/evidence/phase-1/E24/root-final/root-final-attempt5-TEST-dev.evolution.monitor.HttpObservationsTest.xml
new file mode 100644
index 0000000..890564b
--- /dev/null
+++ b/evidence/phase-1/E24/root-final/root-final-attempt5-TEST-dev.evolution.monitor.HttpObservationsTest.xml
@@ -0,0 +1,77 @@
+<?xml version="1.0" encoding="UTF-8"?>
+<testsuite xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="https://maven.apache.org/surefire/maven-surefire-plugin/xsd/surefire-test-report.xsd" version="3.0.2" name="dev.evolution.monitor.HttpObservationsTest" time="0.048" tests="3" errors="0" skipped="0" failures="0" flakes="0">
+  <properties>
+    <property name="java.specification.version" value="21"/>
+    <property name="sun.jnu.encoding" value="UTF-8"/>
+    <property name="java.class.path" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/test-classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-web/3.5.16/spring-boot-starter-web-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter/3.5.16/spring-boot-starter-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot/3.5.16/spring-boot-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-autoconfigure/3.5.16/spring-boot-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-logging/3.5.16/spring-boot-starter-logging-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-classic/1.5.34/logback-classic-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-core/1.5.34/logback-core-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-to-slf4j/2.24.3/log4j-to-slf4j-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-api/2.24.3/log4j-api-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/jul-to-slf4j/2.0.18/jul-to-slf4j-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/annotation/jakarta.annotation-api/2.1.1/jakarta.annotation-api-2.1.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/yaml/snakeyaml/2.4/snakeyaml-2.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-json/3.5.16/spring-boot-starter-json-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-databind/2.21.4/jackson-databind-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-annotations/2.21/jackson-annotations-2.21.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-core/2.21.4/jackson-core-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jdk8/2.21.4/jackson-datatype-jdk8-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jsr310/2.21.4/jackson-datatype-jsr310-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/module/jackson-module-parameter-names/2.21.4/jackson-module-parameter-names-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-tomcat/3.5.16/spring-boot-starter-tomcat-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-core/10.1.55/tomcat-embed-core-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-el/10.1.55/tomcat-embed-el-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-websocket/10.1.55/tomcat-embed-websocket-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-web/6.2.19/spring-web-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-beans/6.2.19/spring-beans-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-webmvc/6.2.19/spring-webmvc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-context/6.2.19/spring-context-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-expression/6.2.19/spring-expression-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-data-jpa/3.5.16/spring-boot-starter-data-jpa-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-jdbc/3.5.16/spring-boot-starter-jdbc-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/zaxxer/HikariCP/6.3.3/HikariCP-6.3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jdbc/6.2.19/spring-jdbc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/orm/hibernate-core/6.6.53.Final/hibernate-core-6.6.53.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/persistence/jakarta.persistence-api/3.1.0/jakarta.persistence-api-3.1.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/transaction/jakarta.transaction-api/2.0.1/jakarta.transaction-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/jboss/logging/jboss-logging/3.6.3.Final/jboss-logging-3.6.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/common/hibernate-commons-annotations/7.0.3.Final/hibernate-commons-annotations-7.0.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/smallrye/jandex/3.2.0/jandex-3.2.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/classmate/1.7.3/classmate-1.7.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy/1.17.8/byte-buddy-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-runtime/4.0.9/jaxb-runtime-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-core/4.0.9/jaxb-core-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/eclipse/angus/angus-activation/2.0.3/angus-activation-2.0.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/txw2/4.0.9/txw2-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/sun/istack/istack-commons-runtime/4.1.2/istack-commons-runtime-4.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/inject/jakarta.inject-api/2.0.1/jakarta.inject-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/antlr/antlr4-runtime/4.13.2/antlr4-runtime-4.13.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-jpa/3.5.13/spring-data-jpa-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-commons/3.5.13/spring-data-commons-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-orm/6.2.19/spring-orm-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-tx/6.2.19/spring-tx-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/slf4j-api/2.0.18/slf4j-api-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aspects/6.2.19/spring-aspects-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/aspectj/aspectjweaver/1.9.25.1/aspectjweaver-1.9.25.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-security/3.5.16/spring-boot-starter-security-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aop/6.2.19/spring-aop-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-config/6.5.11/spring-security-config-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-core/6.5.11/spring-security-core-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-crypto/6.5.11/spring-security-crypto-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-web/6.5.11/spring-security-web-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-actuator/3.5.16/spring-boot-starter-actuator-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-actuator-autoconfigure/3.5.16/spring-boot-actuator-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-actuator/3.5.16/spring-boot-actuator-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-observation/1.15.12/micrometer-observation-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-commons/1.15.12/micrometer-commons-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-jakarta9/1.15.12/micrometer-jakarta9-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-core/1.15.12/micrometer-core-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hdrhistogram/HdrHistogram/2.2.2/HdrHistogram-2.2.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/latencyutils/LatencyUtils/2.0.3/LatencyUtils-2.0.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-database-postgresql/11.7.2/flyway-database-postgresql-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-core/11.7.2/flyway-core-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/dataformat/jackson-dataformat-toml/2.21.4/jackson-dataformat-toml-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/postgresql/postgresql/42.7.11/postgresql-42.7.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-test/3.5.16/spring-boot-starter-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test/3.5.16/spring-boot-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test-autoconfigure/3.5.16/spring-boot-test-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/jayway/jsonpath/json-path/2.9.0/json-path-2.9.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/xml/bind/jakarta.xml.bind-api/4.0.5/jakarta.xml.bind-api-4.0.5.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/activation/jakarta.activation-api/2.1.4/jakarta.activation-api-2.1.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/json-smart/2.5.2/json-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/accessors-smart/2.5.2/accessors-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/ow2/asm/asm/9.7.1/asm-9.7.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/assertj/assertj-core/3.27.7/assertj-core-3.27.7.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/awaitility/awaitility/4.2.2/awaitility-4.2.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hamcrest/hamcrest/3.0/hamcrest-3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter/5.12.2/junit-jupiter-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-api/5.12.2/junit-jupiter-api-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/opentest4j/opentest4j/1.3.0/opentest4j-1.3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-commons/1.12.2/junit-platform-commons-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apiguardian/apiguardian-api/1.1.2/apiguardian-api-1.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-params/5.12.2/junit-jupiter-params-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-engine/5.12.2/junit-jupiter-engine-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-engine/1.12.2/junit-platform-engine-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-core/5.17.0/mockito-core-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy-agent/1.17.8/byte-buddy-agent-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/objenesis/objenesis/3.3/objenesis-3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-junit-jupiter/5.17.0/mockito-junit-jupiter-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/skyscreamer/jsonassert/1.5.3/jsonassert-1.5.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/vaadin/external/google/android-json/0.0.20131108.vaadin1/android-json-0.0.20131108.vaadin1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-core/6.2.19/spring-core-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jcl/6.2.19/spring-jcl-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-test/6.2.19/spring-test-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/xmlunit/xmlunit-core/2.10.4/xmlunit-core-2.10.4.jar:"/>
+    <property name="java.vm.vendor" value="Eclipse Adoptium"/>
+    <property name="sun.arch.data.model" value="64"/>
+    <property name="java.vendor.url" value="https://adoptium.net/"/>
+    <property name="user.timezone" value="Asia/Seoul"/>
+    <property name="org.jboss.logging.provider" value="slf4j"/>
+    <property name="os.name" value="Mac OS X"/>
+    <property name="java.vm.specification.version" value="21"/>
+    <property name="APPLICATION_NAME" value="monitor-api"/>
+    <property name="sun.java.launcher" value="SUN_STANDARD"/>
+    <property name="user.country" value="KR"/>
+    <property name="sun.boot.library.path" value="/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem/lib"/>
+    <property name="sun.java.command" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/surefire/surefirebooter-20260828201217920_3.jar /private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/surefire 2026-08-28T20-12-17_843-jvmRun1 surefire-20260828201217920_1tmp surefire_0-20260828201217920_2tmp"/>
+    <property name="http.nonProxyHosts" value="local|*.local|169.254/16|*.169.254/16"/>
+    <property name="jdk.debug" value="release"/>
+    <property name="test" value="HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest"/>
+    <property name="surefire.test.class.path" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/test-classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-web/3.5.16/spring-boot-starter-web-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter/3.5.16/spring-boot-starter-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot/3.5.16/spring-boot-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-autoconfigure/3.5.16/spring-boot-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-logging/3.5.16/spring-boot-starter-logging-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-classic/1.5.34/logback-classic-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-core/1.5.34/logback-core-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-to-slf4j/2.24.3/log4j-to-slf4j-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-api/2.24.3/log4j-api-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/jul-to-slf4j/2.0.18/jul-to-slf4j-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/annotation/jakarta.annotation-api/2.1.1/jakarta.annotation-api-2.1.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/yaml/snakeyaml/2.4/snakeyaml-2.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-json/3.5.16/spring-boot-starter-json-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-databind/2.21.4/jackson-databind-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-annotations/2.21/jackson-annotations-2.21.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-core/2.21.4/jackson-core-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jdk8/2.21.4/jackson-datatype-jdk8-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jsr310/2.21.4/jackson-datatype-jsr310-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/module/jackson-module-parameter-names/2.21.4/jackson-module-parameter-names-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-tomcat/3.5.16/spring-boot-starter-tomcat-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-core/10.1.55/tomcat-embed-core-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-el/10.1.55/tomcat-embed-el-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-websocket/10.1.55/tomcat-embed-websocket-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-web/6.2.19/spring-web-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-beans/6.2.19/spring-beans-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-webmvc/6.2.19/spring-webmvc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-context/6.2.19/spring-context-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-expression/6.2.19/spring-expression-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-data-jpa/3.5.16/spring-boot-starter-data-jpa-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-jdbc/3.5.16/spring-boot-starter-jdbc-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/zaxxer/HikariCP/6.3.3/HikariCP-6.3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jdbc/6.2.19/spring-jdbc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/orm/hibernate-core/6.6.53.Final/hibernate-core-6.6.53.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/persistence/jakarta.persistence-api/3.1.0/jakarta.persistence-api-3.1.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/transaction/jakarta.transaction-api/2.0.1/jakarta.transaction-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/jboss/logging/jboss-logging/3.6.3.Final/jboss-logging-3.6.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/common/hibernate-commons-annotations/7.0.3.Final/hibernate-commons-annotations-7.0.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/smallrye/jandex/3.2.0/jandex-3.2.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/classmate/1.7.3/classmate-1.7.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy/1.17.8/byte-buddy-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-runtime/4.0.9/jaxb-runtime-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-core/4.0.9/jaxb-core-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/eclipse/angus/angus-activation/2.0.3/angus-activation-2.0.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/txw2/4.0.9/txw2-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/sun/istack/istack-commons-runtime/4.1.2/istack-commons-runtime-4.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/inject/jakarta.inject-api/2.0.1/jakarta.inject-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/antlr/antlr4-runtime/4.13.2/antlr4-runtime-4.13.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-jpa/3.5.13/spring-data-jpa-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-commons/3.5.13/spring-data-commons-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-orm/6.2.19/spring-orm-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-tx/6.2.19/spring-tx-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/slf4j-api/2.0.18/slf4j-api-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aspects/6.2.19/spring-aspects-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/aspectj/aspectjweaver/1.9.25.1/aspectjweaver-1.9.25.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-security/3.5.16/spring-boot-starter-security-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aop/6.2.19/spring-aop-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-config/6.5.11/spring-security-config-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-core/6.5.11/spring-security-core-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-crypto/6.5.11/spring-security-crypto-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-web/6.5.11/spring-security-web-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-actuator/3.5.16/spring-boot-starter-actuator-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-actuator-autoconfigure/3.5.16/spring-boot-actuator-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-actuator/3.5.16/spring-boot-actuator-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-observation/1.15.12/micrometer-observation-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-commons/1.15.12/micrometer-commons-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-jakarta9/1.15.12/micrometer-jakarta9-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-core/1.15.12/micrometer-core-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hdrhistogram/HdrHistogram/2.2.2/HdrHistogram-2.2.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/latencyutils/LatencyUtils/2.0.3/LatencyUtils-2.0.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-database-postgresql/11.7.2/flyway-database-postgresql-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-core/11.7.2/flyway-core-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/dataformat/jackson-dataformat-toml/2.21.4/jackson-dataformat-toml-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/postgresql/postgresql/42.7.11/postgresql-42.7.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-test/3.5.16/spring-boot-starter-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test/3.5.16/spring-boot-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test-autoconfigure/3.5.16/spring-boot-test-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/jayway/jsonpath/json-path/2.9.0/json-path-2.9.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/xml/bind/jakarta.xml.bind-api/4.0.5/jakarta.xml.bind-api-4.0.5.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/activation/jakarta.activation-api/2.1.4/jakarta.activation-api-2.1.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/json-smart/2.5.2/json-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/accessors-smart/2.5.2/accessors-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/ow2/asm/asm/9.7.1/asm-9.7.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/assertj/assertj-core/3.27.7/assertj-core-3.27.7.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/awaitility/awaitility/4.2.2/awaitility-4.2.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hamcrest/hamcrest/3.0/hamcrest-3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter/5.12.2/junit-jupiter-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-api/5.12.2/junit-jupiter-api-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/opentest4j/opentest4j/1.3.0/opentest4j-1.3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-commons/1.12.2/junit-platform-commons-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apiguardian/apiguardian-api/1.1.2/apiguardian-api-1.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-params/5.12.2/junit-jupiter-params-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-engine/5.12.2/junit-jupiter-engine-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-engine/1.12.2/junit-platform-engine-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-core/5.17.0/mockito-core-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy-agent/1.17.8/byte-buddy-agent-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/objenesis/objenesis/3.3/objenesis-3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-junit-jupiter/5.17.0/mockito-junit-jupiter-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/skyscreamer/jsonassert/1.5.3/jsonassert-1.5.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/vaadin/external/google/android-json/0.0.20131108.vaadin1/android-json-0.0.20131108.vaadin1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-core/6.2.19/spring-core-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jcl/6.2.19/spring-jcl-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-test/6.2.19/spring-test-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/xmlunit/xmlunit-core/2.10.4/xmlunit-core-2.10.4.jar:"/>
+    <property name="sun.cpu.endian" value="little"/>
+    <property name="user.home" value="/Users/woopinbell"/>
+    <property name="user.language" value="ko"/>
+    <property name="java.specification.vendor" value="Oracle Corporation"/>
+    <property name="java.version.date" value="2025-04-15"/>
+    <property name="java.home" value="/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"/>
+    <property name="file.separator" value="/"/>
+    <property name="basedir" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend"/>
+    <property name="java.vm.compressedOopsMode" value="Zero based"/>
+    <property name="line.separator" value="&#10;"/>
+    <property name="java.vm.specification.vendor" value="Oracle Corporation"/>
+    <property name="java.specification.name" value="Java Platform API Specification"/>
+    <property name="FILE_LOG_CHARSET" value="UTF-8"/>
+    <property name="java.awt.headless" value="true"/>
+    <property name="apple.awt.application.name" value="ForkedBooter"/>
+    <property name="surefire.real.class.path" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/surefire/surefirebooter-20260828201217920_3.jar"/>
+    <property name="polyglot.engine.WarnInterpreterOnly" value="false"/>
+    <property name="sun.management.compiler" value="HotSpot 64-Bit Tiered Compilers"/>
+    <property name="ftp.nonProxyHosts" value="local|*.local|169.254/16|*.169.254/16"/>
+    <property name="java.runtime.version" value="21.0.7+6-LTS"/>
+    <property name="user.name" value="woopinbell"/>
+    <property name="stdout.encoding" value="UTF-8"/>
+    <property name="path.separator" value=":"/>
+    <property name="os.version" value="26.6.2"/>
+    <property name="java.runtime.name" value="OpenJDK Runtime Environment"/>
+    <property name="file.encoding" value="UTF-8"/>
+    <property name="java.vm.name" value="OpenJDK 64-Bit Server VM"/>
+    <property name="java.vendor.version" value="Temurin-21.0.7+6"/>
+    <property name="localRepository" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository"/>
+    <property name="java.vendor.url.bug" value="https://github.com/adoptium/adoptium-support/issues"/>
+    <property name="java.io.tmpdir" value="/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/"/>
+    <property name="com.zaxxer.hikari.pool_number" value="1"/>
+    <property name="java.version" value="21.0.7"/>
+    <property name="user.dir" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend"/>
+    <property name="os.arch" value="aarch64"/>
+    <property name="java.vm.specification.name" value="Java Virtual Machine Specification"/>
+    <property name="PID" value="21068"/>
+    <property name="CONSOLE_LOG_CHARSET" value="UTF-8"/>
+    <property name="native.encoding" value="UTF-8"/>
+    <property name="java.library.path" value="/Users/woopinbell/Library/Java/Extensions:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java:."/>
+    <property name="java.vm.info" value="mixed mode, sharing"/>
+    <property name="stderr.encoding" value="UTF-8"/>
+    <property name="java.vendor" value="Eclipse Adoptium"/>
+    <property name="java.vm.version" value="21.0.7+6-LTS"/>
+    <property name="sun.io.unicode.encoding" value="UnicodeBig"/>
+    <property name="maven.repo.local" value=".m2/repository"/>
+    <property name="CONSOLE_LOG_STRUCTURED_FORMAT" value="ecs"/>
+    <property name="socksNonProxyHosts" value="local|*.local|169.254/16|*.169.254/16"/>
+    <property name="java.class.version" value="65.0"/>
+    <property name="LOGGED_APPLICATION_NAME" value="[monitor-api] "/>
+  </properties>
+  <testcase name="responseCorrelationIsGeneratedAndRecordedWithoutEchoingInput" classname="dev.evolution.monitor.HttpObservationsTest" time="0.032"/>
+  <testcase name="arbitraryMethodAndUnmatchedPathCannotBecomeMetricLabels" classname="dev.evolution.monitor.HttpObservationsTest" time="0.002"/>
+  <testcase name="completedObservationExportsOnlyBoundedHttpTagsAndNoActiveFamily" classname="dev.evolution.monitor.HttpObservationsTest" time="0.013"/>
+</testsuite>
\ No newline at end of file
diff --git a/evidence/phase-1/E24/root-final/root-final-attempt5-TEST-dev.evolution.monitor.OperationsIntegrationTest.xml b/evidence/phase-1/E24/root-final/root-final-attempt5-TEST-dev.evolution.monitor.OperationsIntegrationTest.xml
new file mode 100644
index 0000000..59ed5ff
--- /dev/null
+++ b/evidence/phase-1/E24/root-final/root-final-attempt5-TEST-dev.evolution.monitor.OperationsIntegrationTest.xml
@@ -0,0 +1,110 @@
+<?xml version="1.0" encoding="UTF-8"?>
+<testsuite xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="https://maven.apache.org/surefire/maven-surefire-plugin/xsd/surefire-test-report.xsd" version="3.0.2" name="dev.evolution.monitor.OperationsIntegrationTest" time="5.327" tests="1" errors="0" skipped="0" failures="0" flakes="0">
+  <properties>
+    <property name="java.specification.version" value="21"/>
+    <property name="sun.jnu.encoding" value="UTF-8"/>
+    <property name="java.class.path" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/test-classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-web/3.5.16/spring-boot-starter-web-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter/3.5.16/spring-boot-starter-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot/3.5.16/spring-boot-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-autoconfigure/3.5.16/spring-boot-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-logging/3.5.16/spring-boot-starter-logging-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-classic/1.5.34/logback-classic-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-core/1.5.34/logback-core-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-to-slf4j/2.24.3/log4j-to-slf4j-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-api/2.24.3/log4j-api-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/jul-to-slf4j/2.0.18/jul-to-slf4j-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/annotation/jakarta.annotation-api/2.1.1/jakarta.annotation-api-2.1.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/yaml/snakeyaml/2.4/snakeyaml-2.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-json/3.5.16/spring-boot-starter-json-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-databind/2.21.4/jackson-databind-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-annotations/2.21/jackson-annotations-2.21.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-core/2.21.4/jackson-core-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jdk8/2.21.4/jackson-datatype-jdk8-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jsr310/2.21.4/jackson-datatype-jsr310-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/module/jackson-module-parameter-names/2.21.4/jackson-module-parameter-names-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-tomcat/3.5.16/spring-boot-starter-tomcat-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-core/10.1.55/tomcat-embed-core-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-el/10.1.55/tomcat-embed-el-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-websocket/10.1.55/tomcat-embed-websocket-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-web/6.2.19/spring-web-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-beans/6.2.19/spring-beans-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-webmvc/6.2.19/spring-webmvc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-context/6.2.19/spring-context-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-expression/6.2.19/spring-expression-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-data-jpa/3.5.16/spring-boot-starter-data-jpa-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-jdbc/3.5.16/spring-boot-starter-jdbc-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/zaxxer/HikariCP/6.3.3/HikariCP-6.3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jdbc/6.2.19/spring-jdbc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/orm/hibernate-core/6.6.53.Final/hibernate-core-6.6.53.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/persistence/jakarta.persistence-api/3.1.0/jakarta.persistence-api-3.1.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/transaction/jakarta.transaction-api/2.0.1/jakarta.transaction-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/jboss/logging/jboss-logging/3.6.3.Final/jboss-logging-3.6.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/common/hibernate-commons-annotations/7.0.3.Final/hibernate-commons-annotations-7.0.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/smallrye/jandex/3.2.0/jandex-3.2.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/classmate/1.7.3/classmate-1.7.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy/1.17.8/byte-buddy-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-runtime/4.0.9/jaxb-runtime-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-core/4.0.9/jaxb-core-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/eclipse/angus/angus-activation/2.0.3/angus-activation-2.0.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/txw2/4.0.9/txw2-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/sun/istack/istack-commons-runtime/4.1.2/istack-commons-runtime-4.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/inject/jakarta.inject-api/2.0.1/jakarta.inject-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/antlr/antlr4-runtime/4.13.2/antlr4-runtime-4.13.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-jpa/3.5.13/spring-data-jpa-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-commons/3.5.13/spring-data-commons-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-orm/6.2.19/spring-orm-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-tx/6.2.19/spring-tx-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/slf4j-api/2.0.18/slf4j-api-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aspects/6.2.19/spring-aspects-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/aspectj/aspectjweaver/1.9.25.1/aspectjweaver-1.9.25.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-security/3.5.16/spring-boot-starter-security-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aop/6.2.19/spring-aop-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-config/6.5.11/spring-security-config-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-core/6.5.11/spring-security-core-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-crypto/6.5.11/spring-security-crypto-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-web/6.5.11/spring-security-web-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-actuator/3.5.16/spring-boot-starter-actuator-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-actuator-autoconfigure/3.5.16/spring-boot-actuator-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-actuator/3.5.16/spring-boot-actuator-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-observation/1.15.12/micrometer-observation-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-commons/1.15.12/micrometer-commons-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-jakarta9/1.15.12/micrometer-jakarta9-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-core/1.15.12/micrometer-core-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hdrhistogram/HdrHistogram/2.2.2/HdrHistogram-2.2.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/latencyutils/LatencyUtils/2.0.3/LatencyUtils-2.0.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-database-postgresql/11.7.2/flyway-database-postgresql-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-core/11.7.2/flyway-core-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/dataformat/jackson-dataformat-toml/2.21.4/jackson-dataformat-toml-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/postgresql/postgresql/42.7.11/postgresql-42.7.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-test/3.5.16/spring-boot-starter-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test/3.5.16/spring-boot-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test-autoconfigure/3.5.16/spring-boot-test-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/jayway/jsonpath/json-path/2.9.0/json-path-2.9.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/xml/bind/jakarta.xml.bind-api/4.0.5/jakarta.xml.bind-api-4.0.5.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/activation/jakarta.activation-api/2.1.4/jakarta.activation-api-2.1.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/json-smart/2.5.2/json-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/accessors-smart/2.5.2/accessors-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/ow2/asm/asm/9.7.1/asm-9.7.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/assertj/assertj-core/3.27.7/assertj-core-3.27.7.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/awaitility/awaitility/4.2.2/awaitility-4.2.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hamcrest/hamcrest/3.0/hamcrest-3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter/5.12.2/junit-jupiter-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-api/5.12.2/junit-jupiter-api-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/opentest4j/opentest4j/1.3.0/opentest4j-1.3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-commons/1.12.2/junit-platform-commons-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apiguardian/apiguardian-api/1.1.2/apiguardian-api-1.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-params/5.12.2/junit-jupiter-params-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-engine/5.12.2/junit-jupiter-engine-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-engine/1.12.2/junit-platform-engine-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-core/5.17.0/mockito-core-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy-agent/1.17.8/byte-buddy-agent-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/objenesis/objenesis/3.3/objenesis-3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-junit-jupiter/5.17.0/mockito-junit-jupiter-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/skyscreamer/jsonassert/1.5.3/jsonassert-1.5.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/vaadin/external/google/android-json/0.0.20131108.vaadin1/android-json-0.0.20131108.vaadin1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-core/6.2.19/spring-core-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jcl/6.2.19/spring-jcl-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-test/6.2.19/spring-test-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/xmlunit/xmlunit-core/2.10.4/xmlunit-core-2.10.4.jar:"/>
+    <property name="java.vm.vendor" value="Eclipse Adoptium"/>
+    <property name="sun.arch.data.model" value="64"/>
+    <property name="java.vendor.url" value="https://adoptium.net/"/>
+    <property name="user.timezone" value="Asia/Seoul"/>
+    <property name="org.jboss.logging.provider" value="slf4j"/>
+    <property name="os.name" value="Mac OS X"/>
+    <property name="java.vm.specification.version" value="21"/>
+    <property name="APPLICATION_NAME" value="monitor-api"/>
+    <property name="sun.java.launcher" value="SUN_STANDARD"/>
+    <property name="user.country" value="KR"/>
+    <property name="sun.boot.library.path" value="/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem/lib"/>
+    <property name="sun.java.command" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/surefire/surefirebooter-20260828201217920_3.jar /private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/surefire 2026-08-28T20-12-17_843-jvmRun1 surefire-20260828201217920_1tmp surefire_0-20260828201217920_2tmp"/>
+    <property name="http.nonProxyHosts" value="local|*.local|169.254/16|*.169.254/16"/>
+    <property name="jdk.debug" value="release"/>
+    <property name="test" value="HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest"/>
+    <property name="surefire.test.class.path" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/test-classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/classes:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-web/3.5.16/spring-boot-starter-web-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter/3.5.16/spring-boot-starter-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot/3.5.16/spring-boot-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-autoconfigure/3.5.16/spring-boot-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-logging/3.5.16/spring-boot-starter-logging-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-classic/1.5.34/logback-classic-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/ch/qos/logback/logback-core/1.5.34/logback-core-1.5.34.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-to-slf4j/2.24.3/log4j-to-slf4j-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/logging/log4j/log4j-api/2.24.3/log4j-api-2.24.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/jul-to-slf4j/2.0.18/jul-to-slf4j-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/annotation/jakarta.annotation-api/2.1.1/jakarta.annotation-api-2.1.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/yaml/snakeyaml/2.4/snakeyaml-2.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-json/3.5.16/spring-boot-starter-json-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-databind/2.21.4/jackson-databind-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-annotations/2.21/jackson-annotations-2.21.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/core/jackson-core/2.21.4/jackson-core-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jdk8/2.21.4/jackson-datatype-jdk8-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jsr310/2.21.4/jackson-datatype-jsr310-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/module/jackson-module-parameter-names/2.21.4/jackson-module-parameter-names-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-tomcat/3.5.16/spring-boot-starter-tomcat-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-core/10.1.55/tomcat-embed-core-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-el/10.1.55/tomcat-embed-el-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apache/tomcat/embed/tomcat-embed-websocket/10.1.55/tomcat-embed-websocket-10.1.55.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-web/6.2.19/spring-web-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-beans/6.2.19/spring-beans-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-webmvc/6.2.19/spring-webmvc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-context/6.2.19/spring-context-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-expression/6.2.19/spring-expression-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-data-jpa/3.5.16/spring-boot-starter-data-jpa-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-jdbc/3.5.16/spring-boot-starter-jdbc-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/zaxxer/HikariCP/6.3.3/HikariCP-6.3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jdbc/6.2.19/spring-jdbc-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/orm/hibernate-core/6.6.53.Final/hibernate-core-6.6.53.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/persistence/jakarta.persistence-api/3.1.0/jakarta.persistence-api-3.1.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/transaction/jakarta.transaction-api/2.0.1/jakarta.transaction-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/jboss/logging/jboss-logging/3.6.3.Final/jboss-logging-3.6.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hibernate/common/hibernate-commons-annotations/7.0.3.Final/hibernate-commons-annotations-7.0.3.Final.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/smallrye/jandex/3.2.0/jandex-3.2.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/classmate/1.7.3/classmate-1.7.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy/1.17.8/byte-buddy-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-runtime/4.0.9/jaxb-runtime-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/jaxb-core/4.0.9/jaxb-core-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/eclipse/angus/angus-activation/2.0.3/angus-activation-2.0.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/glassfish/jaxb/txw2/4.0.9/txw2-4.0.9.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/sun/istack/istack-commons-runtime/4.1.2/istack-commons-runtime-4.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/inject/jakarta.inject-api/2.0.1/jakarta.inject-api-2.0.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/antlr/antlr4-runtime/4.13.2/antlr4-runtime-4.13.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-jpa/3.5.13/spring-data-jpa-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/data/spring-data-commons/3.5.13/spring-data-commons-3.5.13.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-orm/6.2.19/spring-orm-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-tx/6.2.19/spring-tx-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/slf4j/slf4j-api/2.0.18/slf4j-api-2.0.18.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aspects/6.2.19/spring-aspects-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/aspectj/aspectjweaver/1.9.25.1/aspectjweaver-1.9.25.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-security/3.5.16/spring-boot-starter-security-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-aop/6.2.19/spring-aop-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-config/6.5.11/spring-security-config-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-core/6.5.11/spring-security-core-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-crypto/6.5.11/spring-security-crypto-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/security/spring-security-web/6.5.11/spring-security-web-6.5.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-actuator/3.5.16/spring-boot-starter-actuator-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-actuator-autoconfigure/3.5.16/spring-boot-actuator-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-actuator/3.5.16/spring-boot-actuator-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-observation/1.15.12/micrometer-observation-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-commons/1.15.12/micrometer-commons-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-jakarta9/1.15.12/micrometer-jakarta9-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/io/micrometer/micrometer-core/1.15.12/micrometer-core-1.15.12.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hdrhistogram/HdrHistogram/2.2.2/HdrHistogram-2.2.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/latencyutils/LatencyUtils/2.0.3/LatencyUtils-2.0.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-database-postgresql/11.7.2/flyway-database-postgresql-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/flywaydb/flyway-core/11.7.2/flyway-core-11.7.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/fasterxml/jackson/dataformat/jackson-dataformat-toml/2.21.4/jackson-dataformat-toml-2.21.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/postgresql/postgresql/42.7.11/postgresql-42.7.11.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-starter-test/3.5.16/spring-boot-starter-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test/3.5.16/spring-boot-test-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/boot/spring-boot-test-autoconfigure/3.5.16/spring-boot-test-autoconfigure-3.5.16.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/jayway/jsonpath/json-path/2.9.0/json-path-2.9.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/xml/bind/jakarta.xml.bind-api/4.0.5/jakarta.xml.bind-api-4.0.5.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/jakarta/activation/jakarta.activation-api/2.1.4/jakarta.activation-api-2.1.4.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/json-smart/2.5.2/json-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/minidev/accessors-smart/2.5.2/accessors-smart-2.5.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/ow2/asm/asm/9.7.1/asm-9.7.1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/assertj/assertj-core/3.27.7/assertj-core-3.27.7.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/awaitility/awaitility/4.2.2/awaitility-4.2.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/hamcrest/hamcrest/3.0/hamcrest-3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter/5.12.2/junit-jupiter-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-api/5.12.2/junit-jupiter-api-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/opentest4j/opentest4j/1.3.0/opentest4j-1.3.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-commons/1.12.2/junit-platform-commons-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/apiguardian/apiguardian-api/1.1.2/apiguardian-api-1.1.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-params/5.12.2/junit-jupiter-params-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/jupiter/junit-jupiter-engine/5.12.2/junit-jupiter-engine-5.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/junit/platform/junit-platform-engine/1.12.2/junit-platform-engine-1.12.2.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-core/5.17.0/mockito-core-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy-agent/1.17.8/byte-buddy-agent-1.17.8.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/objenesis/objenesis/3.3/objenesis-3.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/mockito/mockito-junit-jupiter/5.17.0/mockito-junit-jupiter-5.17.0.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/skyscreamer/jsonassert/1.5.3/jsonassert-1.5.3.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/com/vaadin/external/google/android-json/0.0.20131108.vaadin1/android-json-0.0.20131108.vaadin1.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-core/6.2.19/spring-core-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-jcl/6.2.19/spring-jcl-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/springframework/spring-test/6.2.19/spring-test-6.2.19.jar:/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/org/xmlunit/xmlunit-core/2.10.4/xmlunit-core-2.10.4.jar:"/>
+    <property name="sun.cpu.endian" value="little"/>
+    <property name="user.home" value="/Users/woopinbell"/>
+    <property name="user.language" value="ko"/>
+    <property name="java.specification.vendor" value="Oracle Corporation"/>
+    <property name="java.version.date" value="2025-04-15"/>
+    <property name="java.home" value="/Users/woopinbell/.sdkman/candidates/java/21.0.7-tem"/>
+    <property name="file.separator" value="/"/>
+    <property name="basedir" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend"/>
+    <property name="java.vm.compressedOopsMode" value="Zero based"/>
+    <property name="line.separator" value="&#10;"/>
+    <property name="java.vm.specification.vendor" value="Oracle Corporation"/>
+    <property name="java.specification.name" value="Java Platform API Specification"/>
+    <property name="FILE_LOG_CHARSET" value="UTF-8"/>
+    <property name="java.awt.headless" value="true"/>
+    <property name="apple.awt.application.name" value="ForkedBooter"/>
+    <property name="surefire.real.class.path" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend/target/surefire/surefirebooter-20260828201217920_3.jar"/>
+    <property name="polyglot.engine.WarnInterpreterOnly" value="false"/>
+    <property name="sun.management.compiler" value="HotSpot 64-Bit Tiered Compilers"/>
+    <property name="ftp.nonProxyHosts" value="local|*.local|169.254/16|*.169.254/16"/>
+    <property name="java.runtime.version" value="21.0.7+6-LTS"/>
+    <property name="user.name" value="woopinbell"/>
+    <property name="stdout.encoding" value="UTF-8"/>
+    <property name="path.separator" value=":"/>
+    <property name="os.version" value="26.6.2"/>
+    <property name="java.runtime.name" value="OpenJDK Runtime Environment"/>
+    <property name="file.encoding" value="UTF-8"/>
+    <property name="java.vm.name" value="OpenJDK 64-Bit Server VM"/>
+    <property name="java.vendor.version" value="Temurin-21.0.7+6"/>
+    <property name="localRepository" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository"/>
+    <property name="java.vendor.url.bug" value="https://github.com/adoptium/adoptium-support/issues"/>
+    <property name="java.io.tmpdir" value="/var/folders/92/jftxv3md5_z3jr5ybm1c3yx40000gn/T/"/>
+    <property name="com.zaxxer.hikari.pool_number" value="1"/>
+    <property name="java.version" value="21.0.7"/>
+    <property name="user.dir" value="/private/tmp/web-systems-evolution-0a006589-industry-spring/backend"/>
+    <property name="os.arch" value="aarch64"/>
+    <property name="java.vm.specification.name" value="Java Virtual Machine Specification"/>
+    <property name="PID" value="21068"/>
+    <property name="CONSOLE_LOG_CHARSET" value="UTF-8"/>
+    <property name="native.encoding" value="UTF-8"/>
+    <property name="java.library.path" value="/Users/woopinbell/Library/Java/Extensions:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java:."/>
+    <property name="java.vm.info" value="mixed mode, sharing"/>
+    <property name="stderr.encoding" value="UTF-8"/>
+    <property name="java.vendor" value="Eclipse Adoptium"/>
+    <property name="java.vm.version" value="21.0.7+6-LTS"/>
+    <property name="sun.io.unicode.encoding" value="UnicodeBig"/>
+    <property name="maven.repo.local" value=".m2/repository"/>
+    <property name="CONSOLE_LOG_STRUCTURED_FORMAT" value="ecs"/>
+    <property name="socksNonProxyHosts" value="local|*.local|169.254/16|*.169.254/16"/>
+    <property name="java.class.version" value="65.0"/>
+    <property name="LOGGED_APPLICATION_NAME" value="[monitor-api] "/>
+  </properties>
+  <testcase name="actualQueueClaimActivityAndRecoveryAreMeasuredAfterCommit" classname="dev.evolution.monitor.OperationsIntegrationTest" time="0.952">
+    <system-out><![CDATA[20:12:19.724 [main] INFO org.springframework.test.context.support.AnnotationConfigContextLoaderUtils -- Could not detect default configuration classes for test class [dev.evolution.monitor.OperationsIntegrationTest]: OperationsIntegrationTest does not declare any static, non-private, non-final, nested classes annotated with @Configuration.
+20:12:19.877 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper -- Found @SpringBootConfiguration dev.evolution.monitor.MonitorApplication for test class dev.evolution.monitor.OperationsIntegrationTest
+{"@timestamp":"2026-08-28T11:12:20.395796Z","log":{"level":"INFO","logger":"dev.evolution.monitor.OperationsIntegrationTest"},"process":{"pid":21068,"thread":{"name":"main"}},"service":{"name":"monitor-api","node":{}},"message":"Starting OperationsIntegrationTest using Java 21.0.7 with PID 21068 (started by woopinbell in /private/tmp/web-systems-evolution-0a006589-industry-spring/backend)","ecs":{"version":"8.11"}}
+{"@timestamp":"2026-08-28T11:12:20.402129Z","log":{"level":"INFO","logger":"dev.evolution.monitor.OperationsIntegrationTest"},"process":{"pid":21068,"thread":{"name":"main"}},"service":{"name":"monitor-api","node":{}},"message":"No active profile set, falling back to 1 default profile: \"default\"","ecs":{"version":"8.11"}}
+{"@timestamp":"2026-08-28T11:12:21.170407Z","log":{"level":"INFO","logger":"org.springframework.data.repository.config.RepositoryConfigurationDelegate"},"process":{"pid":21068,"thread":{"name":"main"}},"service":{"name":"monitor-api","node":{}},"message":"Bootstrapping Spring Data JPA repositories in DEFAULT mode.","ecs":{"version":"8.11"}}
+{"@timestamp":"2026-08-28T11:12:21.193530Z","log":{"level":"INFO","logger":"org.springframework.data.repository.config.RepositoryConfigurationDelegate"},"process":{"pid":21068,"thread":{"name":"main"}},"service":{"name":"monitor-api","node":{}},"message":"Finished Spring Data repository scanning in 16 ms. Found 0 JPA repository interfaces.","ecs":{"version":"8.11"}}
+{"@timestamp":"2026-08-28T11:12:21.573756Z","log":{"level":"INFO","logger":"com.zaxxer.hikari.HikariDataSource"},"process":{"pid":21068,"thread":{"name":"main"}},"service":{"name":"monitor-api","node":{}},"message":"HikariPool-1 - Starting...","ecs":{"version":"8.11"}}
+{"@timestamp":"2026-08-28T11:12:21.595567Z","log":{"level":"INFO","logger":"com.zaxxer.hikari.pool.HikariPool"},"process":{"pid":21068,"thread":{"name":"main"}},"service":{"name":"monitor-api","node":{}},"message":"HikariPool-1 - Added connection org.postgresql.jdbc.PgConnection@34d5eac","ecs":{"version":"8.11"}}
+{"@timestamp":"2026-08-28T11:12:21.597080Z","log":{"level":"INFO","logger":"com.zaxxer.hikari.HikariDataSource"},"process":{"pid":21068,"thread":{"name":"main"}},"service":{"name":"monitor-api","node":{}},"message":"HikariPool-1 - Start completed.","ecs":{"version":"8.11"}}
+{"@timestamp":"2026-08-28T11:12:21.622982Z","log":{"level":"INFO","logger":"org.flywaydb.core.FlywayExecutor"},"process":{"pid":21068,"thread":{"name":"main"}},"service":{"name":"monitor-api","node":{}},"message":"Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)","ecs":{"version":"8.11"}}
+{"@timestamp":"2026-08-28T11:12:21.675174Z","log":{"level":"INFO","logger":"org.flywaydb.core.internal.schemahistory.JdbcTableSchemaHistory"},"process":{"pid":21068,"thread":{"name":"main"}},"service":{"name":"monitor-api","node":{}},"message":"Schema history table \"e24_metrics\".\"flyway_schema_history\" does not exist yet","ecs":{"version":"8.11"}}
+{"@timestamp":"2026-08-28T11:12:21.683789Z","log":{"level":"INFO","logger":"org.flywaydb.core.internal.command.DbValidate"},"process":{"pid":21068,"thread":{"name":"main"}},"service":{"name":"monitor-api","node":{}},"message":"Successfully validated 8 migrations (execution time 00:00.021s)","ecs":{"version":"8.11"}}
+{"@timestamp":"2026-08-28T11:12:21.720222Z","log":{"level":"INFO","logger":"org.flywaydb.core.internal.schemahistory.JdbcTableSchemaHistory"},"process":{"pid":21068,"thread":{"name":"main"}},"service":{"name":"monitor-api","node":{}},"message":"Creating Schema History table \"e24_metrics\".\"flyway_schema_history\" ...","ecs":{"version":"8.11"}}
+{"@timestamp":"2026-08-28T11:12:21.808410Z","log":{"level":"INFO","logger":"org.flywaydb.core.internal.command.DbMigrate"},"process":{"pid":21068,"thread":{"name":"main"}},"service":{"name":"monitor-api","node":{}},"message":"Current version of schema \"e24_metrics\": << Empty Schema >>","ecs":{"version":"8.11"}}
+{"@timestamp":"2026-08-28T11:12:21.836677Z","log":{"level":"INFO","logger":"org.flywaydb.core.internal.command.DbMigrate"},"process":{"pid":21068,"thread":{"name":"main"}},"service":{"name":"monitor-api","node":{}},"message":"Migrating schema \"e24_metrics\" to version \"1 - create monitors\"","ecs":{"version":"8.11"}}
+{"@timestamp":"2026-08-28T11:12:21.904472Z","log":{"level":"INFO","logger":"org.flywaydb.core.internal.command.DbMigrate"},"process":{"pid":21068,"thread":{"name":"main"}},"service":{"name":"monitor-api","node":{}},"message":"Migrating schema \"e24_metrics\" to version \"2 - create check runs\"","ecs":{"version":"8.11"}}
+{"@timestamp":"2026-08-28T11:12:21.939941Z","log":{"level":"INFO","logger":"org.flywaydb.core.internal.command.DbMigrate"},"process":{"pid":21068,"thread":{"name":"main"}},"service":{"name":"monitor-api","node":{}},"message":"Migrating schema \"e24_metrics\" to version \"3 - create users\"","ecs":{"version":"8.11"}}
+{"@timestamp":"2026-08-28T11:12:21.976115Z","log":{"level":"INFO","logger":"org.flywaydb.core.internal.command.DbMigrate"},"process":{"pid":21068,"thread":{"name":"main"}},"service":{"name":"monitor-api","node":{}},"message":"Migrating schema \"e24_metrics\" to version \"4 - require monitor ownership\"","ecs":{"version":"8.11"}}
+{"@timestamp":"2026-08-28T11:12:22.011696Z","log":{"level":"INFO","logger":"org.flywaydb.core.internal.command.DbMigrate"},"process":{"pid":21068,"thread":{"name":"main"}},"service":{"name":"monitor-api","node":{}},"message":"Migrating schema \"e24_metrics\" to version \"5 - index check history\"","ecs":{"version":"8.11"}}
+{"@timestamp":"2026-08-28T11:12:22.061390Z","log":{"level":"INFO","logger":"org.flywaydb.core.internal.command.DbMigrate"},"process":{"pid":21068,"thread":{"name":"main"}},"service":{"name":"monitor-api","node":{}},"message":"Migrating schema \"e24_metrics\" to version \"6 - queue check execution\"","ecs":{"version":"8.11"}}
+{"@timestamp":"2026-08-28T11:12:22.104936Z","log":{"level":"INFO","logger":"org.flywaydb.core.internal.command.DbMigrate"},"process":{"pid":21068,"thread":{"name":"main"}},"service":{"name":"monitor-api","node":{}},"message":"Migrating schema \"e24_metrics\" to version \"7 - execution ownership and manual identity\"","ecs":{"version":"8.11"}}
+{"@timestamp":"2026-08-28T11:12:22.136957Z","log":{"level":"INFO","logger":"org.flywaydb.core.internal.command.DbMigrate"},"process":{"pid":21068,"thread":{"name":"main"}},"service":{"name":"monitor-api","node":{}},"message":"Migrating schema \"e24_metrics\" to version \"8 - recover expired executions\"","ecs":{"version":"8.11"}}
+{"@timestamp":"2026-08-28T11:12:22.173961Z","log":{"level":"INFO","logger":"org.flywaydb.core.internal.command.DbMigrate"},"process":{"pid":21068,"thread":{"name":"main"}},"service":{"name":"monitor-api","node":{}},"message":"Successfully applied 8 migrations to schema \"e24_metrics\", now at version v8 (execution time 00:00.123s)","ecs":{"version":"8.11"}}
+{"@timestamp":"2026-08-28T11:12:22.269671Z","log":{"level":"INFO","logger":"org.hibernate.jpa.internal.util.LogHelper"},"process":{"pid":21068,"thread":{"name":"main"}},"service":{"name":"monitor-api","node":{}},"message":"HHH000204: Processing PersistenceUnitInfo [name: default]","ecs":{"version":"8.11"}}
+{"@timestamp":"2026-08-28T11:12:22.314255Z","log":{"level":"INFO","logger":"org.hibernate.Version"},"process":{"pid":21068,"thread":{"name":"main"}},"service":{"name":"monitor-api","node":{}},"message":"HHH000412: Hibernate ORM core version 6.6.53.Final","ecs":{"version":"8.11"}}
+{"@timestamp":"2026-08-28T11:12:22.338138Z","log":{"level":"INFO","logger":"org.hibernate.cache.internal.RegionFactoryInitiator"},"process":{"pid":21068,"thread":{"name":"main"}},"service":{"name":"monitor-api","node":{}},"message":"HHH000026: Second-level cache disabled","ecs":{"version":"8.11"}}
+{"@timestamp":"2026-08-28T11:12:22.522044Z","log":{"level":"INFO","logger":"org.springframework.orm.jpa.persistenceunit.SpringPersistenceUnitInfo"},"process":{"pid":21068,"thread":{"name":"main"}},"service":{"name":"monitor-api","node":{}},"message":"No LoadTimeWeaver setup: ignoring JPA class transformer","ecs":{"version":"8.11"}}
+{"@timestamp":"2026-08-28T11:12:22.600438Z","log":{"level":"INFO","logger":"org.hibernate.orm.connections.pooling"},"process":{"pid":21068,"thread":{"name":"main"}},"service":{"name":"monitor-api","node":{}},"message":"HHH10001005: Database info:\n\tDatabase JDBC URL [Connecting through datasource 'HikariDataSource (HikariPool-1)']\n\tDatabase driver: undefined/unknown\n\tDatabase version: 17.11\n\tAutocommit mode: undefined/unknown\n\tIsolation level: undefined/unknown\n\tMinimum pool size: undefined/unknown\n\tMaximum pool size: undefined/unknown","ecs":{"version":"8.11"}}
+{"@timestamp":"2026-08-28T11:12:23.407226Z","log":{"level":"INFO","logger":"org.hibernate.engine.transaction.jta.platform.internal.JtaPlatformInitiator"},"process":{"pid":21068,"thread":{"name":"main"}},"service":{"name":"monitor-api","node":{}},"message":"HHH000489: No JTA platform available (set 'hibernate.transaction.jta.platform' to enable JTA platform integration)","ecs":{"version":"8.11"}}
+{"@timestamp":"2026-08-28T11:12:23.468284Z","log":{"level":"INFO","logger":"org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean"},"process":{"pid":21068,"thread":{"name":"main"}},"service":{"name":"monitor-api","node":{}},"message":"Initialized JPA EntityManagerFactory for persistence unit 'default'","ecs":{"version":"8.11"}}
+{"@timestamp":"2026-08-28T11:12:24.037899Z","log":{"level":"INFO","logger":"dev.evolution.monitor.OperationsIntegrationTest"},"process":{"pid":21068,"thread":{"name":"main"}},"service":{"name":"monitor-api","node":{}},"message":"Started OperationsIntegrationTest in 4.094 seconds (process running for 6.061)","ecs":{"version":"8.11"}}
+{"@timestamp":"2026-08-28T11:12:24.869239Z","log":{"level":"INFO","logger":"dev.evolution.monitor.ProcessObservations"},"process":{"pid":21068,"thread":{"name":"pool-3-thread-1"}},"service":{"name":"monitor-api","node":{}},"message":"Worker observation","event":"check_claimed","process_role":"worker","process_id":"c3aeb497-a0fa-48c7-ae7f-f27ad341a967","ecs":{"version":"8.11"}}
+{"@timestamp":"2026-08-28T11:12:24.977202Z","log":{"level":"INFO","logger":"dev.evolution.monitor.ProcessObservations"},"process":{"pid":21068,"thread":{"name":"main"}},"service":{"name":"monitor-api","node":{}},"message":"Worker observation","event":"checks_recovered","process_role":"worker","process_id":"c3aeb497-a0fa-48c7-ae7f-f27ad341a967","ecs":{"version":"8.11"}}
+]]></system-out>
+  </testcase>
+</testsuite>
\ No newline at end of file
diff --git a/evidence/phase-1/E24/root-final/root-final-attempt5-affected-java.log.native.json b/evidence/phase-1/E24/root-final/root-final-attempt5-affected-java.log.native.json
new file mode 100644
index 0000000..4c168ef
--- /dev/null
+++ b/evidence/phase-1/E24/root-final/root-final-attempt5-affected-java.log.native.json
@@ -0,0 +1,8 @@
+{
+  "encoding": "json-utf8-raw",
+  "source": "index/profiles/phase-1/verification/industry-spring/E24-artifacts/root-final-attempt5-affected-java.log",
+  "sourceRoot": "/Users/woopinbell/Desktop/working/workflow/web-systems-evolution",
+  "bytes": 16237,
+  "sha256": "a64fca20a45528dd33e8f12123d5ae4644b2dab3e267c5fc778f402eac14e4d1",
+  "raw": "[INFO] Scanning for projects...\n[INFO] \n[INFO] ---------------------< dev.evolution:monitor-api >----------------------\n[INFO] Building monitor-api 0.0.1\n[INFO]   from pom.xml\n[INFO] --------------------------------[ jar ]---------------------------------\n[INFO] \n[INFO] --- enforcer:3.6.2:enforce (pinned-runtimes) @ monitor-api ---\n[INFO] Rule 0: org.apache.maven.enforcer.rules.version.RequireJavaVersion passed\n[INFO] Rule 1: org.apache.maven.enforcer.rules.version.RequireMavenVersion passed\n[INFO] \n[INFO] --- resources:3.3.1:resources (default-resources) @ monitor-api ---\n[INFO] Copying 2 resources from src/main/resources to target/classes\n[INFO] Copying 8 resources from src/main/resources to target/classes\n[INFO] \n[INFO] --- compiler:3.14.1:compile (default-compile) @ monitor-api ---\n[INFO] Nothing to compile - all classes are up to date.\n[INFO] \n[INFO] --- resources:3.3.1:testResources (default-testResources) @ monitor-api ---\n[INFO] skip non existing resourceDirectory /private/tmp/web-systems-evolution-0a006589-industry-spring/backend/src/test/resources\n[INFO] \n[INFO] --- compiler:3.14.1:testCompile (default-testCompile) @ monitor-api ---\n[INFO] Nothing to compile - all classes are up to date.\n[INFO] \n[INFO] --- surefire:3.5.6:test (default-test) @ monitor-api ---\n[INFO] Using auto detected provider org.apache.maven.surefire.junitplatform.JUnitPlatformProvider\n[INFO] \n[INFO] -------------------------------------------------------\n[INFO]  T E S T S\n[INFO] -------------------------------------------------------\n[INFO] Running dev.evolution.monitor.ApiErrorBoundaryTest\nMockito is currently self-attaching to enable the inline-mock-maker. This will no longer work in future releases of the JDK. Please add Mockito as an agent to your build as described in Mockito's documentation: https://javadoc.io/doc/org.mockito/mockito-core/latest/org.mockito/org/mockito/Mockito.html#0.3\nOpenJDK 64-Bit Server VM warning: Sharing is only supported for boot loader classes because bootstrap classpath has been appended\nWARNING: A Java agent has been loaded dynamically (/private/tmp/web-systems-evolution-0a006589-industry-spring/.m2/repository/net/bytebuddy/byte-buddy-agent/1.17.8/byte-buddy-agent-1.17.8.jar)\nWARNING: If a serviceability tool is in use, please run with -XX:+EnableDynamicAgentLoading to hide this warning\nWARNING: If a serviceability tool is not in use, please run with -Djdk.instrument.traceUsage for more information\nWARNING: Dynamic loading of agents will be disallowed by default in a future release\n20:12:19.461 [main] INFO org.springframework.mock.web.MockServletContext -- Initializing Spring TestDispatcherServlet ''\n20:12:19.462 [main] INFO org.springframework.test.web.servlet.TestDispatcherServlet -- Initializing Servlet ''\n20:12:19.465 [main] INFO org.springframework.test.web.servlet.TestDispatcherServlet -- Completed initialization in 2 ms\n20:12:19.656 [main] INFO org.springframework.mock.web.MockServletContext -- Initializing Spring TestDispatcherServlet ''\n20:12:19.656 [main] INFO org.springframework.test.web.servlet.TestDispatcherServlet -- Initializing Servlet ''\n20:12:19.656 [main] INFO org.springframework.test.web.servlet.TestDispatcherServlet -- Completed initialization in 0 ms\n[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 1.264 s -- in dev.evolution.monitor.ApiErrorBoundaryTest\n[INFO] Running dev.evolution.monitor.OperationsIntegrationTest\n20:12:19.724 [main] INFO org.springframework.test.context.support.AnnotationConfigContextLoaderUtils -- Could not detect default configuration classes for test class [dev.evolution.monitor.OperationsIntegrationTest]: OperationsIntegrationTest does not declare any static, non-private, non-final, nested classes annotated with @Configuration.\n20:12:19.877 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper -- Found @SpringBootConfiguration dev.evolution.monitor.MonitorApplication for test class dev.evolution.monitor.OperationsIntegrationTest\n{\"@timestamp\":\"2026-08-28T11:12:20.395796Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"dev.evolution.monitor.OperationsIntegrationTest\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"main\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"Starting OperationsIntegrationTest using Java 21.0.7 with PID 21068 (started by woopinbell in /private/tmp/web-systems-evolution-0a006589-industry-spring/backend)\",\"ecs\":{\"version\":\"8.11\"}}\n{\"@timestamp\":\"2026-08-28T11:12:20.402129Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"dev.evolution.monitor.OperationsIntegrationTest\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"main\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"No active profile set, falling back to 1 default profile: \\\"default\\\"\",\"ecs\":{\"version\":\"8.11\"}}\n{\"@timestamp\":\"2026-08-28T11:12:21.170407Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"org.springframework.data.repository.config.RepositoryConfigurationDelegate\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"main\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"Bootstrapping Spring Data JPA repositories in DEFAULT mode.\",\"ecs\":{\"version\":\"8.11\"}}\n{\"@timestamp\":\"2026-08-28T11:12:21.193530Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"org.springframework.data.repository.config.RepositoryConfigurationDelegate\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"main\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"Finished Spring Data repository scanning in 16 ms. Found 0 JPA repository interfaces.\",\"ecs\":{\"version\":\"8.11\"}}\n{\"@timestamp\":\"2026-08-28T11:12:21.573756Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"com.zaxxer.hikari.HikariDataSource\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"main\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"HikariPool-1 - Starting...\",\"ecs\":{\"version\":\"8.11\"}}\n{\"@timestamp\":\"2026-08-28T11:12:21.595567Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"com.zaxxer.hikari.pool.HikariPool\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"main\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"HikariPool-1 - Added connection org.postgresql.jdbc.PgConnection@34d5eac\",\"ecs\":{\"version\":\"8.11\"}}\n{\"@timestamp\":\"2026-08-28T11:12:21.597080Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"com.zaxxer.hikari.HikariDataSource\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"main\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"HikariPool-1 - Start completed.\",\"ecs\":{\"version\":\"8.11\"}}\n{\"@timestamp\":\"2026-08-28T11:12:21.622982Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"org.flywaydb.core.FlywayExecutor\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"main\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"Database: jdbc:postgresql://127.0.0.1:15432/monitor (PostgreSQL 17.11)\",\"ecs\":{\"version\":\"8.11\"}}\n{\"@timestamp\":\"2026-08-28T11:12:21.675174Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"org.flywaydb.core.internal.schemahistory.JdbcTableSchemaHistory\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"main\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"Schema history table \\\"e24_metrics\\\".\\\"flyway_schema_history\\\" does not exist yet\",\"ecs\":{\"version\":\"8.11\"}}\n{\"@timestamp\":\"2026-08-28T11:12:21.683789Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"org.flywaydb.core.internal.command.DbValidate\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"main\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"Successfully validated 8 migrations (execution time 00:00.021s)\",\"ecs\":{\"version\":\"8.11\"}}\n{\"@timestamp\":\"2026-08-28T11:12:21.720222Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"org.flywaydb.core.internal.schemahistory.JdbcTableSchemaHistory\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"main\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"Creating Schema History table \\\"e24_metrics\\\".\\\"flyway_schema_history\\\" ...\",\"ecs\":{\"version\":\"8.11\"}}\n{\"@timestamp\":\"2026-08-28T11:12:21.808410Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"org.flywaydb.core.internal.command.DbMigrate\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"main\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"Current version of schema \\\"e24_metrics\\\": << Empty Schema >>\",\"ecs\":{\"version\":\"8.11\"}}\n{\"@timestamp\":\"2026-08-28T11:12:21.836677Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"org.flywaydb.core.internal.command.DbMigrate\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"main\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"Migrating schema \\\"e24_metrics\\\" to version \\\"1 - create monitors\\\"\",\"ecs\":{\"version\":\"8.11\"}}\n{\"@timestamp\":\"2026-08-28T11:12:21.904472Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"org.flywaydb.core.internal.command.DbMigrate\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"main\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"Migrating schema \\\"e24_metrics\\\" to version \\\"2 - create check runs\\\"\",\"ecs\":{\"version\":\"8.11\"}}\n{\"@timestamp\":\"2026-08-28T11:12:21.939941Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"org.flywaydb.core.internal.command.DbMigrate\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"main\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"Migrating schema \\\"e24_metrics\\\" to version \\\"3 - create users\\\"\",\"ecs\":{\"version\":\"8.11\"}}\n{\"@timestamp\":\"2026-08-28T11:12:21.976115Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"org.flywaydb.core.internal.command.DbMigrate\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"main\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"Migrating schema \\\"e24_metrics\\\" to version \\\"4 - require monitor ownership\\\"\",\"ecs\":{\"version\":\"8.11\"}}\n{\"@timestamp\":\"2026-08-28T11:12:22.011696Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"org.flywaydb.core.internal.command.DbMigrate\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"main\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"Migrating schema \\\"e24_metrics\\\" to version \\\"5 - index check history\\\"\",\"ecs\":{\"version\":\"8.11\"}}\n{\"@timestamp\":\"2026-08-28T11:12:22.061390Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"org.flywaydb.core.internal.command.DbMigrate\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"main\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"Migrating schema \\\"e24_metrics\\\" to version \\\"6 - queue check execution\\\"\",\"ecs\":{\"version\":\"8.11\"}}\n{\"@timestamp\":\"2026-08-28T11:12:22.104936Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"org.flywaydb.core.internal.command.DbMigrate\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"main\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"Migrating schema \\\"e24_metrics\\\" to version \\\"7 - execution ownership and manual identity\\\"\",\"ecs\":{\"version\":\"8.11\"}}\n{\"@timestamp\":\"2026-08-28T11:12:22.136957Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"org.flywaydb.core.internal.command.DbMigrate\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"main\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"Migrating schema \\\"e24_metrics\\\" to version \\\"8 - recover expired executions\\\"\",\"ecs\":{\"version\":\"8.11\"}}\n{\"@timestamp\":\"2026-08-28T11:12:22.173961Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"org.flywaydb.core.internal.command.DbMigrate\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"main\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"Successfully applied 8 migrations to schema \\\"e24_metrics\\\", now at version v8 (execution time 00:00.123s)\",\"ecs\":{\"version\":\"8.11\"}}\n{\"@timestamp\":\"2026-08-28T11:12:22.269671Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"org.hibernate.jpa.internal.util.LogHelper\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"main\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"HHH000204: Processing PersistenceUnitInfo [name: default]\",\"ecs\":{\"version\":\"8.11\"}}\n{\"@timestamp\":\"2026-08-28T11:12:22.314255Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"org.hibernate.Version\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"main\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"HHH000412: Hibernate ORM core version 6.6.53.Final\",\"ecs\":{\"version\":\"8.11\"}}\n{\"@timestamp\":\"2026-08-28T11:12:22.338138Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"org.hibernate.cache.internal.RegionFactoryInitiator\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"main\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"HHH000026: Second-level cache disabled\",\"ecs\":{\"version\":\"8.11\"}}\n{\"@timestamp\":\"2026-08-28T11:12:22.522044Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"org.springframework.orm.jpa.persistenceunit.SpringPersistenceUnitInfo\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"main\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"No LoadTimeWeaver setup: ignoring JPA class transformer\",\"ecs\":{\"version\":\"8.11\"}}\n{\"@timestamp\":\"2026-08-28T11:12:22.600438Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"org.hibernate.orm.connections.pooling\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"main\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"HHH10001005: Database info:\\n\\tDatabase JDBC URL [Connecting through datasource 'HikariDataSource (HikariPool-1)']\\n\\tDatabase driver: undefined/unknown\\n\\tDatabase version: 17.11\\n\\tAutocommit mode: undefined/unknown\\n\\tIsolation level: undefined/unknown\\n\\tMinimum pool size: undefined/unknown\\n\\tMaximum pool size: undefined/unknown\",\"ecs\":{\"version\":\"8.11\"}}\n{\"@timestamp\":\"2026-08-28T11:12:23.407226Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"org.hibernate.engine.transaction.jta.platform.internal.JtaPlatformInitiator\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"main\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"HHH000489: No JTA platform available (set 'hibernate.transaction.jta.platform' to enable JTA platform integration)\",\"ecs\":{\"version\":\"8.11\"}}\n{\"@timestamp\":\"2026-08-28T11:12:23.468284Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"main\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"Initialized JPA EntityManagerFactory for persistence unit 'default'\",\"ecs\":{\"version\":\"8.11\"}}\n{\"@timestamp\":\"2026-08-28T11:12:24.037899Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"dev.evolution.monitor.OperationsIntegrationTest\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"main\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"Started OperationsIntegrationTest in 4.094 seconds (process running for 6.061)\",\"ecs\":{\"version\":\"8.11\"}}\n{\"@timestamp\":\"2026-08-28T11:12:24.869239Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"dev.evolution.monitor.ProcessObservations\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"pool-3-thread-1\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"Worker observation\",\"event\":\"check_claimed\",\"process_role\":\"worker\",\"process_id\":\"c3aeb497-a0fa-48c7-ae7f-f27ad341a967\",\"ecs\":{\"version\":\"8.11\"}}\n{\"@timestamp\":\"2026-08-28T11:12:24.977202Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"dev.evolution.monitor.ProcessObservations\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"main\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"Worker observation\",\"event\":\"checks_recovered\",\"process_role\":\"worker\",\"process_id\":\"c3aeb497-a0fa-48c7-ae7f-f27ad341a967\",\"ecs\":{\"version\":\"8.11\"}}\n{\"@timestamp\":\"2026-08-28T11:12:25.011451Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"main\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"Closing JPA EntityManagerFactory for persistence unit 'default'\",\"ecs\":{\"version\":\"8.11\"}}\n{\"@timestamp\":\"2026-08-28T11:12:25.013162Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"com.zaxxer.hikari.HikariDataSource\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"main\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"HikariPool-1 - Shutdown initiated...\",\"ecs\":{\"version\":\"8.11\"}}\n{\"@timestamp\":\"2026-08-28T11:12:25.015218Z\",\"log\":{\"level\":\"INFO\",\"logger\":\"com.zaxxer.hikari.HikariDataSource\"},\"process\":{\"pid\":21068,\"thread\":{\"name\":\"main\"}},\"service\":{\"name\":\"monitor-api\",\"node\":{}},\"message\":\"HikariPool-1 - Shutdown completed.\",\"ecs\":{\"version\":\"8.11\"}}\n[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 5.327 s -- in dev.evolution.monitor.OperationsIntegrationTest\n[INFO] Running dev.evolution.monitor.HttpObservationsTest\n[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.048 s -- in dev.evolution.monitor.HttpObservationsTest\n[INFO] \n[INFO] Results:\n[INFO] \n[INFO] Tests run: 6, Failures: 0, Errors: 0, Skipped: 0\n[INFO] \n[INFO] ------------------------------------------------------------------------\n[INFO] BUILD SUCCESS\n[INFO] ------------------------------------------------------------------------\n[INFO] Total time:  8.200 s\n[INFO] Finished at: 2026-08-28T20:12:25+09:00\n[INFO] ------------------------------------------------------------------------\n"
+}
diff --git a/evidence/phase-1/E24/root-final/root-final-attempt5-compose-config.log b/evidence/phase-1/E24/root-final/root-final-attempt5-compose-config.log
new file mode 100644
index 0000000..e69de29
diff --git a/evidence/phase-1/E24/root-final/root-final-attempt5-container-native.json b/evidence/phase-1/E24/root-final/root-final-attempt5-container-native.json
new file mode 100644
index 0000000..0245b0d
--- /dev/null
+++ b/evidence/phase-1/E24/root-final/root-final-attempt5-container-native.json
@@ -0,0 +1,1755 @@
+{
+  "actor": "root",
+  "thread": "E24",
+  "profile": "phase-1",
+  "specRevision": "2ada57a71cd34fa2fae9809415c362a8bbfcdf02",
+  "start": "563b325ef871fe6d1fbfef7cf39a6581f2d0a94d",
+  "startedAt": "2026-08-28T11:12:25.184Z",
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
+      "elapsedMs": 26.514375,
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
+      "elapsedMs": 21.128041999999994,
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
+      "elapsedMs": 17.610917,
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
+      "elapsedMs": 17.710875000000016,
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
+      "elapsedMs": 71.667416,
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
+      "elapsedMs": 177.01641700000002,
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
+      "elapsedMs": 19.68866700000001,
+      "stdoutBytes": 12843,
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
+      "elapsedMs": 646.3197909999999,
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
+      "elapsedMs": 117.70800000000054,
+      "stdoutBytes": 52,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "2564eed743c9"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 48.34620799999993,
+      "stdoutBytes": 8814,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "c3770de5dc87"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 101.88887500000055,
+      "stdoutBytes": 9157,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "8ac045d9b3b8"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 93.28087499999947,
+      "stdoutBytes": 11267,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "6edc0ebcc8dc"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 49.57112499999948,
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
+      "elapsedMs": 66.45499999999993,
+      "stdoutBytes": 13,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "6edc0ebcc8dc9ffcfff9b28c0b31f19422b57526644e02bd3b537cc39d707e3b"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 40.3660830000008,
+      "stdoutBytes": 10610,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "8ac045d9b3b8a2a2bb35e8c3da247409cdf040fcee44e59d0538f8e999f6dda4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 34.53016600000046,
+      "stdoutBytes": 11267,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "c3770de5dc87e527cb24c2735687024a2a4d0c4c445d522ca8f9fade66e50c93"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 39.303249999997206,
+      "stdoutBytes": 9157,
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
+      "elapsedMs": 409.3273330000011,
+      "stdoutBytes": 0,
+      "stderrBytes": 89,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "6edc0ebcc8dc9ffcfff9b28c0b31f19422b57526644e02bd3b537cc39d707e3b"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 44.92533299999923,
+      "stdoutBytes": 10610,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "8ac045d9b3b8a2a2bb35e8c3da247409cdf040fcee44e59d0538f8e999f6dda4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 37.57154199999786,
+      "stdoutBytes": 11267,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "c3770de5dc87e527cb24c2735687024a2a4d0c4c445d522ca8f9fade66e50c93"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 37.20024999999805,
+      "stdoutBytes": 9157,
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
+      "elapsedMs": 333.46295800000007,
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
+      "elapsedMs": 44.386915999999474,
+      "stdoutBytes": 12845,
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
+      "elapsedMs": 52.418416000000434,
+      "stdoutBytes": 12845,
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
+      "elapsedMs": 56.17216700000063,
+      "stdoutBytes": 12845,
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
+      "elapsedMs": 40.9163750000007,
+      "stdoutBytes": 12845,
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
+      "elapsedMs": 62.617333999998664,
+      "stdoutBytes": 12845,
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
+      "elapsedMs": 34.91716599999927,
+      "stdoutBytes": 12845,
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
+      "elapsedMs": 33.00691699999879,
+      "stdoutBytes": 12845,
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
+      "elapsedMs": 28.99066700000185,
+      "stdoutBytes": 12845,
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
+      "elapsedMs": 21.5525000000016,
+      "stdoutBytes": 12845,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "6edc0ebcc8dc9ffcfff9b28c0b31f19422b57526644e02bd3b537cc39d707e3b"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 18.66308299999946,
+      "stdoutBytes": 10610,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "8ac045d9b3b8a2a2bb35e8c3da247409cdf040fcee44e59d0538f8e999f6dda4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 18.095916000002035,
+      "stdoutBytes": 11267,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "c3770de5dc87e527cb24c2735687024a2a4d0c4c445d522ca8f9fade66e50c93"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 17.180457999998907,
+      "stdoutBytes": 9157,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "6edc0ebcc8dc9ffcfff9b28c0b31f19422b57526644e02bd3b537cc39d707e3b"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 20.435457999999926,
+      "stdoutBytes": 10610,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "exec",
+        "6edc0ebcc8dc9ffcfff9b28c0b31f19422b57526644e02bd3b537cc39d707e3b",
+        "cat",
+        "/proc/1/status"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 53.40879099999802,
+      "stdoutBytes": 1128,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "exec",
+        "6edc0ebcc8dc9ffcfff9b28c0b31f19422b57526644e02bd3b537cc39d707e3b",
+        "java",
+        "-version"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 117.48845799999981,
+      "stdoutBytes": 0,
+      "stderrBytes": 190,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "exec",
+        "6edc0ebcc8dc9ffcfff9b28c0b31f19422b57526644e02bd3b537cc39d707e3b",
+        "sha256sum",
+        "/app/app.jar"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 261.94087500000023,
+      "stdoutBytes": 79,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "8ac045d9b3b8a2a2bb35e8c3da247409cdf040fcee44e59d0538f8e999f6dda4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 20.13145799999984,
+      "stdoutBytes": 11267,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "exec",
+        "8ac045d9b3b8a2a2bb35e8c3da247409cdf040fcee44e59d0538f8e999f6dda4",
+        "cat",
+        "/proc/1/status"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 84.7937090000014,
+      "stdoutBytes": 1128,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "exec",
+        "8ac045d9b3b8a2a2bb35e8c3da247409cdf040fcee44e59d0538f8e999f6dda4",
+        "java",
+        "-version"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 87.88441600000078,
+      "stdoutBytes": 0,
+      "stderrBytes": 190,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "exec",
+        "8ac045d9b3b8a2a2bb35e8c3da247409cdf040fcee44e59d0538f8e999f6dda4",
+        "sha256sum",
+        "/app/app.jar"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 243.38837500000227,
+      "stdoutBytes": 79,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "c3770de5dc87e527cb24c2735687024a2a4d0c4c445d522ca8f9fade66e50c93"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 20.42583399999785,
+      "stdoutBytes": 9157,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "exec",
+        "c3770de5dc87e527cb24c2735687024a2a4d0c4c445d522ca8f9fade66e50c93",
+        "cat",
+        "/proc/1/status"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 58.44133299999885,
+      "stdoutBytes": 1132,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "exec",
+        "c3770de5dc87e527cb24c2735687024a2a4d0c4c445d522ca8f9fade66e50c93",
+        "node",
+        "-e",
+        "console.log(JSON.stringify({node:process.versions.node,next:require(\"next/package.json\").version,react:require(\"react/package.json\").version,mode:process.env.NODE_ENV,origin:process.env.API_ORIGIN,manualSignal:!!process.env.NEXT_MANUAL_SIG_HANDLE}))"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 83.07404099999985,
+      "stdoutBytes": 120,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "c3770de5dc87e527cb24c2735687024a2a4d0c4c445d522ca8f9fade66e50c93"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 18.752292000001034,
+      "stdoutBytes": 9157,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "kill",
+        "--signal=SIGTERM",
+        "c3770de5dc87e527cb24c2735687024a2a4d0c4c445d522ca8f9fade66e50c93"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 26.545874999999796,
+      "stdoutBytes": 65,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "wait",
+        "c3770de5dc87e527cb24c2735687024a2a4d0c4c445d522ca8f9fade66e50c93"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 171.80375000000276,
+      "stdoutBytes": 4,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "8ac045d9b3b8a2a2bb35e8c3da247409cdf040fcee44e59d0538f8e999f6dda4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 22.56012499999997,
+      "stdoutBytes": 11267,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "kill",
+        "--signal=SIGTERM",
+        "8ac045d9b3b8a2a2bb35e8c3da247409cdf040fcee44e59d0538f8e999f6dda4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 26.61975000000166,
+      "stdoutBytes": 65,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "wait",
+        "8ac045d9b3b8a2a2bb35e8c3da247409cdf040fcee44e59d0538f8e999f6dda4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 447.42766699999993,
+      "stdoutBytes": 4,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "6edc0ebcc8dc9ffcfff9b28c0b31f19422b57526644e02bd3b537cc39d707e3b"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 26.450292000001355,
+      "stdoutBytes": 10610,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "kill",
+        "--signal=SIGTERM",
+        "6edc0ebcc8dc9ffcfff9b28c0b31f19422b57526644e02bd3b537cc39d707e3b"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 23.895999999997002,
+      "stdoutBytes": 65,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "wait",
+        "6edc0ebcc8dc9ffcfff9b28c0b31f19422b57526644e02bd3b537cc39d707e3b"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 336.93645800000013,
+      "stdoutBytes": 4,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "2564eed743c90bb5cef7246b5658e437ef38e6d6718a1e936daeaae6ec426667"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 27.75749999999971,
+      "stdoutBytes": 8814,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "kill",
+        "--signal=SIGTERM",
+        "2564eed743c90bb5cef7246b5658e437ef38e6d6718a1e936daeaae6ec426667"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 28.53387500000099,
+      "stdoutBytes": 65,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "wait",
+        "2564eed743c90bb5cef7246b5658e437ef38e6d6718a1e936daeaae6ec426667"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 54.952666999997746,
+      "stdoutBytes": 4,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "logs",
+        "6edc0ebcc8dc9ffcfff9b28c0b31f19422b57526644e02bd3b537cc39d707e3b"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 29.957542000000103,
+      "stdoutBytes": 48006,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "logs",
+        "8ac045d9b3b8a2a2bb35e8c3da247409cdf040fcee44e59d0538f8e999f6dda4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 20.380583000001934,
+      "stdoutBytes": 27979,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "logs",
+        "c3770de5dc87e527cb24c2735687024a2a4d0c4c445d522ca8f9fade66e50c93"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 21.188249999999243,
+      "stdoutBytes": 147,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "logs",
+        "2564eed743c90bb5cef7246b5658e437ef38e6d6718a1e936daeaae6ec426667"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 20.601833000000624,
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
+      "elapsedMs": 27.693124999997963,
+      "stdoutBytes": 52,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "2564eed743c9"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 18.656999999999243,
+      "stdoutBytes": 8822,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "c3770de5dc87"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 18.99554200000057,
+      "stdoutBytes": 8769,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "8ac045d9b3b8"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 19.78425000000061,
+      "stdoutBytes": 10561,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "6edc0ebcc8dc"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 19.592666999997164,
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
+      "elapsedMs": 16.455792000000656,
+      "stdoutBytes": 13,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "2564eed743c90bb5cef7246b5658e437ef38e6d6718a1e936daeaae6ec426667"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 16.55645799999911,
+      "stdoutBytes": 8822,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "rm",
+        "2564eed743c90bb5cef7246b5658e437ef38e6d6718a1e936daeaae6ec426667"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 34.504000000000815,
+      "stdoutBytes": 65,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "c3770de5dc87e527cb24c2735687024a2a4d0c4c445d522ca8f9fade66e50c93"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 25.951624999997875,
+      "stdoutBytes": 8769,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "rm",
+        "c3770de5dc87e527cb24c2735687024a2a4d0c4c445d522ca8f9fade66e50c93"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 37.452792000000045,
+      "stdoutBytes": 65,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "8ac045d9b3b8a2a2bb35e8c3da247409cdf040fcee44e59d0538f8e999f6dda4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 32.005999999997584,
+      "stdoutBytes": 10561,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "rm",
+        "8ac045d9b3b8a2a2bb35e8c3da247409cdf040fcee44e59d0538f8e999f6dda4"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 56.38225000000239,
+      "stdoutBytes": 65,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "inspect",
+        "6edc0ebcc8dc9ffcfff9b28c0b31f19422b57526644e02bd3b537cc39d707e3b"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 41.92250000000058,
+      "stdoutBytes": 10087,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "rm",
+        "6edc0ebcc8dc9ffcfff9b28c0b31f19422b57526644e02bd3b537cc39d707e3b"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 78.61891599999944,
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
+      "elapsedMs": 40.234042000000045,
+      "stdoutBytes": 1093,
+      "stderrBytes": 0,
+      "rawOutputRetained": false
+    },
+    {
+      "command": [
+        "docker",
+        "network",
+        "rm",
+        "a5ce5854e4b8444a6af2b62619a3e78123903c2f5b4494b636c57541cc4f3fb5"
+      ],
+      "exitCode": 0,
+      "signal": null,
+      "timedOut": false,
+      "outputLimit": false,
+      "spawnFailed": false,
+      "elapsedMs": 265.0152080000007,
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
+      "elapsedMs": 52.35337499999878,
+      "stdoutBytes": 12845,
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
+      "elapsedMs": 171.8413330000003,
+      "awaited": true,
+      "observerTimedOut": false,
+      "forcedContainerKill": false
+    },
+    "worker": {
+      "purpose": "frozen production signal acceptance",
+      "signal": "SIGTERM",
+      "signalCommandExit": 0,
+      "exitCode": 143,
+      "elapsedMs": 447.5100829999974,
+      "awaited": true,
+      "observerTimedOut": false,
+      "forcedContainerKill": false
+    },
+    "api": {
+      "purpose": "frozen production signal acceptance",
+      "signal": "SIGTERM",
+      "signalCommandExit": 0,
+      "exitCode": 143,
+      "elapsedMs": 336.9692920000016,
+      "awaited": true,
+      "observerTimedOut": false,
+      "forcedContainerKill": false
+    },
+    "fixture": {
+      "purpose": "owned fixture cleanup",
+      "signal": "SIGTERM",
+      "signalCommandExit": 0,
+      "exitCode": 143,
+      "elapsedMs": 54.98158400000102,
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
+  "head": "aa5cf9b932baacbe888915d438c4982daf7d1483",
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
+      "localObservationAndReleaseMs": 11.655416999998124,
+      "dockerOrSqlBeforeRelease": false
+    },
+    "fixture": {
+      "outboundCalls": 1,
+      "holdRequests": 1,
+      "held": 0,
+      "releases": 1,
+      "watchdogReleases": 0,
+      "lastHeldMs": 25.01008399999955
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
+      "TOTAL_TIME": 3.3804640029999997,
+      "MAX": 1.021714042
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
+      "id": "6edc0ebcc8dc9ffcfff9b28c0b31f19422b57526644e02bd3b537cc39d707e3b",
+      "image": "sha256:d6f6ab129f0199ab6cb92ae12ee40fb83dba92bc40d9421a16f874b0265fbcc1",
+      "pid": 85959,
+      "startedAt": "2026-08-28T11:12:31.152108291Z",
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
+      "id": "8ac045d9b3b8a2a2bb35e8c3da247409cdf040fcee44e59d0538f8e999f6dda4",
+      "image": "sha256:d6f6ab129f0199ab6cb92ae12ee40fb83dba92bc40d9421a16f874b0265fbcc1",
+      "pid": 85960,
+      "startedAt": "2026-08-28T11:12:31.156308375Z",
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
+      "id": "c3770de5dc87e527cb24c2735687024a2a4d0c4c445d522ca8f9fade66e50c93",
+      "image": "sha256:9356a3b65bd58be8e12f5a4721975f4801e1601f7dbef0a4bb7ddfd6ef93ac9e",
+      "pid": 85961,
+      "startedAt": "2026-08-28T11:12:31.154399416Z",
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
+        "lines": 47,
+        "structuredLines": 47,
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
+      "49ef3919-d659-4c4b-9ecf-617be9513423",
+      "8b6bfb74-f6ef-4d02-8eef-ccebd3c14008",
+      "d20469fa-f525-4fdc-9b87-4aa9c79cc287",
+      "c27ce740-4b0a-4881-a29e-d0674ff9e627",
+      "0232a72a-30cb-42df-97a1-4c295ff642f8",
+      "76ae6e6f-153c-4b9f-8eb8-044b5208e6db",
+      "c13834a1-69a1-4732-ad8c-6b46aa8a5be3",
+      "e46fe443-f83f-4646-99bb-441cbe1c893b",
+      "08b0951e-6780-468d-bb7f-46757f9635b1",
+      "d23790f6-bdd0-4f03-8cd4-e347b9dfb280",
+      "db742755-1a62-4a8e-9a8d-e789cab8f661",
+      "880039c6-3d01-44f8-9b2c-30bc5eae7e97",
+      "fa7ad054-5f11-4259-80bc-375cdeca0436",
+      "b8bed647-fd6a-42e7-84bb-554200b14a88",
+      "171a537c-46f5-4b2d-8786-b94adfecdb38",
+      "7c1cc562-33ae-46bd-a70d-7d64a7275624",
+      "75affb21-7d38-4583-847f-af6b6c19997d",
+      "d6a944fd-b7cf-4704-9117-76baa40f410e"
+    ],
+    "stableApiProcess": true,
+    "stableWorkerProcess": true,
+    "distinctRoleProcesses": true,
+    "apiProcessId": "2d377846-c2d7-4584-b60c-9954394485fe",
+    "workerProcessId": "e11835ca-086b-47d1-b3a4-c8716f58fb8d",
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
+  "elapsedSeconds": 26.942,
+  "hostedCiExecutionClaimed": false
+}
diff --git a/evidence/phase-1/E24/root-final/root-final-attempt5-dev.evolution.monitor.ApiErrorBoundaryTest.txt b/evidence/phase-1/E24/root-final/root-final-attempt5-dev.evolution.monitor.ApiErrorBoundaryTest.txt
new file mode 100644
index 0000000..294b2da
--- /dev/null
+++ b/evidence/phase-1/E24/root-final/root-final-attempt5-dev.evolution.monitor.ApiErrorBoundaryTest.txt
@@ -0,0 +1,4 @@
+-------------------------------------------------------------------------------
+Test set: dev.evolution.monitor.ApiErrorBoundaryTest
+-------------------------------------------------------------------------------
+Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 1.264 s -- in dev.evolution.monitor.ApiErrorBoundaryTest
diff --git a/evidence/phase-1/E24/root-final/root-final-attempt5-dev.evolution.monitor.HttpObservationsTest.txt b/evidence/phase-1/E24/root-final/root-final-attempt5-dev.evolution.monitor.HttpObservationsTest.txt
new file mode 100644
index 0000000..9e8b425
--- /dev/null
+++ b/evidence/phase-1/E24/root-final/root-final-attempt5-dev.evolution.monitor.HttpObservationsTest.txt
@@ -0,0 +1,4 @@
+-------------------------------------------------------------------------------
+Test set: dev.evolution.monitor.HttpObservationsTest
+-------------------------------------------------------------------------------
+Tests run: 3, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.048 s -- in dev.evolution.monitor.HttpObservationsTest
diff --git a/evidence/phase-1/E24/root-final/root-final-attempt5-dev.evolution.monitor.OperationsIntegrationTest.txt b/evidence/phase-1/E24/root-final/root-final-attempt5-dev.evolution.monitor.OperationsIntegrationTest.txt
new file mode 100644
index 0000000..81cb293
--- /dev/null
+++ b/evidence/phase-1/E24/root-final/root-final-attempt5-dev.evolution.monitor.OperationsIntegrationTest.txt
@@ -0,0 +1,4 @@
+-------------------------------------------------------------------------------
+Test set: dev.evolution.monitor.OperationsIntegrationTest
+-------------------------------------------------------------------------------
+Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 5.327 s -- in dev.evolution.monitor.OperationsIntegrationTest
diff --git a/evidence/phase-1/E24/root-final/root-final-attempt5-e24-metrics.json b/evidence/phase-1/E24/root-final/root-final-attempt5-e24-metrics.json
new file mode 100644
index 0000000..7e7c424
--- /dev/null
+++ b/evidence/phase-1/E24/root-final/root-final-attempt5-e24-metrics.json
@@ -0,0 +1,2 @@
+{"result":"PASS","positiveQueueAge":true,"heldWorkerActive":1,"committedWorkerClaims":1,
+ "committedRecoveries":1,"idleActive":0,"emptyQueueAge":0,"processCrashes":0,"dependencyStops":0}
diff --git a/evidence/phase-1/E24/root-final/root-final-attempt5-full-container.log b/evidence/phase-1/E24/root-final/root-final-attempt5-full-container.log
new file mode 100644
index 0000000..b9ddfa0
--- /dev/null
+++ b/evidence/phase-1/E24/root-final/root-final-attempt5-full-container.log
@@ -0,0 +1 @@
+{"actor":"root","result":"PASS","phase":"completed","evidence":"evidence/phase-1/E24/container/root-5464ab8650b03ea5.json","elapsedSeconds":26.942,"cleanupFailures":0}
diff --git a/evidence/phase-1/E24/root-final/root-final-attempt5-typecheck.log b/evidence/phase-1/E24/root-final/root-final-attempt5-typecheck.log
new file mode 100644
index 0000000..9a53a55
--- /dev/null
+++ b/evidence/phase-1/E24/root-final/root-final-attempt5-typecheck.log
@@ -0,0 +1,6 @@
+
+> industry-spring-monitor@0.0.1 typecheck
+> next typegen && tsc --noEmit
+
+Generating route types...
+✓ Types generated successfully
diff --git a/evidence/phase-1/E24/root-final/root-final-attempt5.json b/evidence/phase-1/E24/root-final/root-final-attempt5.json
new file mode 100644
index 0000000..dc2b202
--- /dev/null
+++ b/evidence/phase-1/E24/root-final/root-final-attempt5.json
@@ -0,0 +1,146 @@
+{
+  "status": "PASS",
+  "track": "industry-spring",
+  "thread": "E24",
+  "profile": "phase-1",
+  "attempt": 5,
+  "spec_revision": "2ada57a71cd34fa2fae9809415c362a8bbfcdf02",
+  "start": "563b325ef871fe6d1fbfef7cf39a6581f2d0a94d",
+  "candidate_end": "aa5cf9b932baacbe888915d438c4982daf7d1483",
+  "started_at": "2026-08-28T11:12:13.276782+00:00",
+  "author_evidence": {
+    "branch_path": "evidence/phase-1/E24/container/author-5464ab8650b03ea5.json",
+    "sha256": "e2af8cae161592bf031fbef5e0c0dd3eaf4453fdff83fea531f236d36ab040ec"
+  },
+  "reviewed_source_sha256": "9d8043f7260c3f7ebfea918f2ad1f32bf7c44985b670756b870575a6e9d3d8be",
+  "commands": [
+    {
+      "label": "compose-config",
+      "command": [
+        "docker",
+        "compose",
+        "-p",
+        "wse-industry-e24",
+        "-f",
+        "compose.production.yaml",
+        "config",
+        "--quiet"
+      ],
+      "started_at": "2026-08-28T11:12:13.278767+00:00",
+      "exit_code": 0,
+      "elapsed_seconds": 0.241,
+      "log": "index/profiles/phase-1/verification/industry-spring/E24-artifacts/root-final-attempt5-compose-config.log",
+      "log_sha256": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
+    },
+    {
+      "label": "typecheck",
+      "command": [
+        "npm",
+        "run",
+        "typecheck"
+      ],
+      "started_at": "2026-08-28T11:12:13.520646+00:00",
+      "exit_code": 0,
+      "elapsed_seconds": 2.075,
+      "log": "index/profiles/phase-1/verification/industry-spring/E24-artifacts/root-final-attempt5-typecheck.log",
+      "log_sha256": "2e2cf148badf4fe2a73749b44b300bdcf32682947e228fd9c498c31cf63989e0"
+    },
+    {
+      "label": "affected-java",
+      "command": [
+        "mvn",
+        "-B",
+        "-ntp",
+        "-f",
+        "backend/pom.xml",
+        "-Dtest=HttpObservationsTest,OperationsIntegrationTest,ApiErrorBoundaryTest",
+        "test"
+      ],
+      "started_at": "2026-08-28T11:12:15.596041+00:00",
+      "exit_code": 0,
+      "elapsed_seconds": 9.54,
+      "log": "index/profiles/phase-1/verification/industry-spring/E24-artifacts/root-final-attempt5-affected-java.log",
+      "log_sha256": "a64fca20a45528dd33e8f12123d5ae4644b2dab3e267c5fc778f402eac14e4d1"
+    },
+    {
+      "label": "full-container",
+      "command": [
+        "node",
+        "scripts/e24-container.mjs",
+        "root"
+      ],
+      "started_at": "2026-08-28T11:12:25.140468+00:00",
+      "exit_code": 0,
+      "elapsed_seconds": 26.999,
+      "log": "index/profiles/phase-1/verification/industry-spring/E24-artifacts/root-final-attempt5-full-container.log",
+      "log_sha256": "4eafc7062153fcd5aa22220b956813cf635687d6c7c484b8e4c1a7d3fe26b274"
+    }
+  ],
+  "baseline_invocations": 0,
+  "image_build_invocations": 0,
+  "automatic_retries": 0,
+  "load_runs": 0,
+  "parameter_sweeps": 0,
+  "e11_crash_reruns": 0,
+  "e20_dataset_reruns": 0,
+  "java_suites": [
+    {
+      "name": "dev.evolution.monitor.HttpObservationsTest",
+      "tests": 3,
+      "failures": 0,
+      "errors": 0,
+      "skipped": 0
+    },
+    {
+      "name": "dev.evolution.monitor.OperationsIntegrationTest",
+      "tests": 1,
+      "failures": 0,
+      "errors": 0,
+      "skipped": 0
+    },
+    {
+      "name": "dev.evolution.monitor.ApiErrorBoundaryTest",
+      "tests": 2,
+      "failures": 0,
+      "errors": 0,
+      "skipped": 0
+    }
+  ],
+  "native_java_files": {
+    "index/profiles/phase-1/verification/industry-spring/E24-artifacts/root-final-attempt5-TEST-dev.evolution.monitor.HttpObservationsTest.xml": "5cd6a0f2530cacac738dd7a23f9b51c57de2d6ee2e903518fd165f35d3b839f0",
+    "index/profiles/phase-1/verification/industry-spring/E24-artifacts/root-final-attempt5-dev.evolution.monitor.HttpObservationsTest.txt": "7b3f87a42ab895826d11e4f3d5753d6cb3e53c42421a1629d0bf2f4213de784e",
+    "index/profiles/phase-1/verification/industry-spring/E24-artifacts/root-final-attempt5-TEST-dev.evolution.monitor.OperationsIntegrationTest.xml": "594fa44d28cffa1dfaf40f19e78232b393aabb591009f57a0531545d785e8000",
+    "index/profiles/phase-1/verification/industry-spring/E24-artifacts/root-final-attempt5-dev.evolution.monitor.OperationsIntegrationTest.txt": "ce15555041499d56c8dd09ee6164df2510e382eeb54e377040a15c7a898766e0",
+    "index/profiles/phase-1/verification/industry-spring/E24-artifacts/root-final-attempt5-TEST-dev.evolution.monitor.ApiErrorBoundaryTest.xml": "495fb0620622e9ffafdfeeffeadbea0071e5ff4b7432ffb0af78659a2c34fb3a",
+    "index/profiles/phase-1/verification/industry-spring/E24-artifacts/root-final-attempt5-dev.evolution.monitor.ApiErrorBoundaryTest.txt": "374ee2091e9a3a9322fd659c36ac00ff687fbfaf49efda7b9aa703de631b6ce5",
+    "index/profiles/phase-1/verification/industry-spring/E24-artifacts/root-final-attempt5-e24-metrics.json": "2156f5970e15f9fc2213bfb19d8033e9004f36cb0e8403ded5012f91a2f727d8"
+  },
+  "new_container_invocations": [
+    {
+      "actor": "root",
+      "head": "aa5cf9b932baacbe888915d438c4982daf7d1483",
+      "fingerprint": "5464ab8650b03ea5c281bd53d32940af8d72ce2fe1b1317c3e7df562725a4b27",
+      "startedAt": "2026-08-28T11:12:25.184Z",
+      "command": "node scripts/e24-container.mjs root"
+    }
+  ],
+  "container_native": {
+    "branch_path": "evidence/phase-1/E24/container/root-5464ab8650b03ea5.json",
+    "path": "index/profiles/phase-1/verification/industry-spring/E24-artifacts/root-final-attempt5-container-native.json",
+    "sha256": "bc412d5d0a607297e795352a716663f2701ee1db35f70ad1148d760d21ec67f4"
+  },
+  "acceptance_audit": {
+    "java_native_pass": true,
+    "container_native_pass": true,
+    "same_built_jar": true
+  },
+  "exit_code": 0,
+  "elapsed_seconds": 38.977,
+  "finished_at": "2026-08-28T11:12:52.254969+00:00",
+  "tracked_worktree_unchanged": false,
+  "changed_tracked_paths": [
+    "evidence/phase-1/E24/container/invocations.jsonl"
+  ],
+  "only_authorized_native_ledger_append": true,
+  "source_hashes_unchanged": true
+}
diff --git a/evidence/phase-1/E24/root-final/root-reviewed-final-source-sha256.json b/evidence/phase-1/E24/root-final/root-reviewed-final-source-sha256.json
new file mode 100644
index 0000000..d54ab8e
--- /dev/null
+++ b/evidence/phase-1/E24/root-final/root-reviewed-final-source-sha256.json
@@ -0,0 +1,117 @@
+{
+  ".dockerignore": "037d68e018325d47a656397e8a5f25c75eb3e819e8a8ef30dfec3f39252c5903",
+  ".github/workflows/ci.yml": "6be6d23aa460b59114e4a54d36436166d59dd38bd6f2ea5a1d8b37083256adef",
+  ".gitignore": "be9abe74dbcfbca726f28f9134327104044c8e3ab40ea084d88735ce403f5e27",
+  ".java-version": "bf148a67f20980c385728f0941b266f5cfde5738a68294af2fb1f3895709746d",
+  ".maven-version": "856e0cfb25b3f7aa5e4d1c7d24bd71e932eeb3e4842eeeb8eba16dbb93622b3f",
+  ".mvn/maven.config": "bab846e0333c1c14ef867196f428e64cb6d8819b593a129d6a421c0cc15d51bd",
+  ".node-version": "7e8a2fa94951112b894a3dbe3d05efef5e9263741fa49125f0a70f40fedab4cc",
+  ".npmrc": "68b03c60d573ccde7f999f91586350d7f0c096b5110ff825776cee3e97144dc3",
+  "AGENTS.md": "63f2c50380ed6303237cce215ce27af1d620d094c215e28d1b1538a3c070e3bb",
+  "CLAUDE.md": "336cc4fbf19beaada7ccf9986414fa91851a8d7a07dfb3ccbe800a69eed0ab49",
+  "Dockerfile.api": "772bf2675423c425cf634e86ec4a85cffbe4bdaaecea8b0331af576df1271572",
+  "Dockerfile.frontend": "42da9feb4474ab612ee0aa8621bcd91fd071d37d5e148a1641c84be671770b41",
+  "SPEC_REVISION": "71fcd29ba0d651cbe239f07a9397b6b4202ef930d3c9c99627f7b124ec4074cc",
+  "app/layout.tsx": "a2495bd5350faa1b1a8d6c844fce662e0184ca1a25048599cf1a66050a5abe46",
+  "app/login/login-form.tsx": "16bd8292476966fd48a233f7a1db665bba2bcc82c4d2cd427f872863e3c0b352",
+  "app/login/page.tsx": "003e21f8756ce27100fc55245afa108c0a90e274b5712c7d9f3d2740a743e0d6",
+  "app/monitors/api.ts": "78796b2ea67b4d67188dbc4e401774984cb08140ab711450d288e173771d24e7",
+  "app/monitors/monitor-controls.tsx": "10da53e514e54a433a58b11ce571f5b093b3ccac9fa753785882a25f9eab2304",
+  "app/monitors/page.tsx": "6d928f31eefb403048b42b5e2b868434a75a04f16285ffd351ce3d2014ebacd2",
+  "app/monitors/server-data.ts": "5b66ef9b14f952fde10e72c94015b7117eee7c6545b2be873b8e171182ed795f",
+  "app/monitors/use-monitor-state.ts": "5396e2c39b86ef9801424ff42bf1abc9ef4f628fcd97b9259984bc0465cad24c",
+  "app/page.tsx": "7a79152a2fc206ac90a5524d2e1ba97117176a4948218e06fd6cbcaae777a3ba",
+  "app/style.css": "27c69839255be19e4ed938ff64e7c5b1551ff351637bacd972bf4ecec6952913",
+  "backend/pom.xml": "9c010b8a0bf11de522b5c0fc66dfffa7ba55ae2212b56ff8206e63c710c1a901",
+  "backend/src/main/java/dev/evolution/monitor/ApiData.java": "f1b93b929e3438fd2b27eefe9b3b14f141969fe8a65ce7d25e72d033817b6a95",
+  "backend/src/main/java/dev/evolution/monitor/ApiErrors.java": "81c0f40ecee1eeca6c3d55df7c7a6b022e5db2eaddbe4aa696335834d4ad1f0e",
+  "backend/src/main/java/dev/evolution/monitor/AuthenticationConfig.java": "00865cb087e10016fc8dba499b673ed1e2e89fc064028a078428a5f55285c0b0",
+  "backend/src/main/java/dev/evolution/monitor/AuthorityHealthIndicator.java": "8b65df49a8409afe7bbdfcf5de57f82e968ef338272d5226dbf7dbe142d0eede",
+  "backend/src/main/java/dev/evolution/monitor/BootstrapUsers.java": "0de49802402637484657a761803e9114a6212a4d043f1b54e9e5bcdbdd7abcea",
+  "backend/src/main/java/dev/evolution/monitor/BrowserOriginFilter.java": "7585da02f821d64b96153bcbd795a1131951a60ca902d9d4528495b0505eaf68",
+  "backend/src/main/java/dev/evolution/monitor/CheckQueue.java": "6d68d05bdfcb919160fc6a39fb456befd1308c0e594c72487fbaab73b0a6ffde",
+  "backend/src/main/java/dev/evolution/monitor/CheckRunEntity.java": "d960343ecbb9d95a0e45c5427792f6d596d3f709e49a7dea4b4a557ad7ac0fae",
+  "backend/src/main/java/dev/evolution/monitor/CheckRunner.java": "3c19ab3c35bfdbcf0587018b6ea0d033fb6352c948aa575ed1acebe2ce40db08",
+  "backend/src/main/java/dev/evolution/monitor/CheckWorker.java": "d955d7d0f55ad9671cc46185f02aea4bf6cb36c3c482427f425d2cd2aeca16fc",
+  "backend/src/main/java/dev/evolution/monitor/HistoryQuery.java": "0714da965db609c9d117614aa7249c879aa723735abf9f75350475d75ff50d76",
+  "backend/src/main/java/dev/evolution/monitor/HttpObservations.java": "f3a54450d30ecf7533ed5256f932c6eb161132df443b856de371cc45af20f88f",
+  "backend/src/main/java/dev/evolution/monitor/Monitor.java": "1a2ad9dd43c6db3fcda5aae8717328b329d83913cd095dddc75c04daeeec7159",
+  "backend/src/main/java/dev/evolution/monitor/MonitorApplication.java": "041c5259a6a5c237c568843842ce8aaaae2cd29fb2070e230ecb5435c3ff3dac",
+  "backend/src/main/java/dev/evolution/monitor/MonitorController.java": "c5f387343b3e2ece2e0b3eef5d85e24cf639b5eeafa4f3c8cdc8923e21e2b119",
+  "backend/src/main/java/dev/evolution/monitor/MonitorEntity.java": "c5cdbd85ab051d29b4945c1815d48d161339c74ed3959f0cd90615ad08512cd4",
+  "backend/src/main/java/dev/evolution/monitor/MonitorStore.java": "5dc1845f37fad4fdafe21b0aebb465c3bd51c19640f9dc4440e4d3fade135122",
+  "backend/src/main/java/dev/evolution/monitor/OutboundUrl.java": "b9d3d71a9f806f7d0774c90234df5116c3f65464cc1bec59368dd7cf9a0edf81",
+  "backend/src/main/java/dev/evolution/monitor/ProcessObservations.java": "8f31dcf1a2cc3f1943a8671a0b6bf98c441520e2aa94d0dd47552d451eca56a5",
+  "backend/src/main/java/dev/evolution/monitor/SchemaCompatibility.java": "24dbe175a93ff68b8342cade3a31bc3a8924ca23e14bf55783f997ab11e5b965",
+  "backend/src/main/java/dev/evolution/monitor/SessionController.java": "ae504fb0fd351ed76accc9def93db0b81f47013f0566dfc0c632dbc15810e536",
+  "backend/src/main/java/dev/evolution/monitor/SessionExpiryFilter.java": "185017adf9169c2fb2e155ae496f3072f6aad6b3d096a6635eeab81d3f1dfd3c",
+  "backend/src/main/java/dev/evolution/monitor/UserAccounts.java": "aff27825123c8a5485bb9ce1f6f700be883c18b343018284615ccc6c27aa3f1e",
+  "backend/src/main/java/dev/evolution/monitor/UserEntity.java": "05e221647f6bf2381fce8f8db513fe10544133a2f8cb332787817bb7cc9227b4",
+  "backend/src/main/java/dev/evolution/monitor/WorkerManagement.java": "50662ea34e7678ee625eb1ce361711727a3a5bc1cb34ff9ac268597997fa1eb5",
+  "backend/src/main/java/dev/evolution/monitor/WorkerProcess.java": "fc626881cde513da0eb992f8778c6ee1b17d2bfc77a1a7c3b0052ee4b73e7df6",
+  "backend/src/main/resources/application-worker.properties": "c43808ca3ace52e98d8a1e510680b096da1bf591ccaf65a387dada1d740b2290",
+  "backend/src/main/resources/application.properties": "08b7a7389896025c27f9bda0fb42069c4bfd3283a6ebff400a2971ac6458d871",
+  "backend/src/main/resources/db/migration/V1__create_monitors.sql": "6ba851e332222856fba7c07e1c9832259659b5c753f0e70923b16ab97e31f6a1",
+  "backend/src/main/resources/db/migration/V2__create_check_runs.sql": "6dd6dde4245d3238ece1b254d73c1065ac7c930f38496d605d7664f1a9bd5848",
+  "backend/src/main/resources/db/migration/V3__create_users.sql": "16f82378495af891da9a1617885b6c51a7a5c4a3871e2105e66e2a282c62752b",
+  "backend/src/main/resources/db/migration/V4__require_monitor_ownership.sql": "d4b9c875336983a8a8219ec140075fae8ece9837b49187aac14af80556f35855",
+  "backend/src/main/resources/db/migration/V5__index_check_history.sql": "1a2115ff8ddae50bc1f75b5ef13e1e06c9ebfd2f0cb988cc577e7c4de9c14dcc",
+  "backend/src/main/resources/db/migration/V6__queue_check_execution.sql": "b85f0c009272aa23b34143276704d37e0861e10dff825b1a353ba74f27e96761",
+  "backend/src/main/resources/db/migration/V7__execution_ownership_and_manual_identity.sql": "56e172e958f2cf9b1e336cdf488131843035a5a0bc8aa2abf3823f48c9d78712",
+  "backend/src/main/resources/db/migration/V8__recover_expired_executions.sql": "52881acc201da4096c737bd7b25c8a03a97cee51f0464415e79633fb07a09240",
+  "backend/src/test/java/dev/evolution/monitor/ApiErrorBoundaryTest.java": "35c193a4b02a088a030213d31381540f5b6e96a63ecfc145ff9c7f8f9469216b",
+  "backend/src/test/java/dev/evolution/monitor/CheckQueueTest.java": "9f6bfb045dda2cb2c33efb9205398d4264afbc4bce0abe8fdeeb9cc1314fdbcb",
+  "backend/src/test/java/dev/evolution/monitor/CheckRunnerTest.java": "85140678d0981514f0b5e6d78b1714294814a7603f2bc548470a0047bae349b9",
+  "backend/src/test/java/dev/evolution/monitor/E10WorkerProcess.java": "2a829b14dcfb40e2d1016f025922f81493f55e6a3f14ec65cde870a8c3c62e30",
+  "backend/src/test/java/dev/evolution/monitor/E11WorkerProcess.java": "fbbca9762e2cf8cea6a0a5a5ad843b855013b9349f6414694e91f18646d9274d",
+  "backend/src/test/java/dev/evolution/monitor/ExecutionOwnershipTest.java": "ef277eec07ec2c5a45d2bdf853bbadf522fecb7e85fd352bcf4107a81a2c132a",
+  "backend/src/test/java/dev/evolution/monitor/HistoryIndexMigrationTest.java": "2dc9e3b94e5fcec5709db586392a7580b091c1fda16a873370eadd7840fcd532",
+  "backend/src/test/java/dev/evolution/monitor/HistoryPaginationTest.java": "b57dd4a55b6ce65886d93d631f64a7cbc3d95d47a878dd871e36507191f471e8",
+  "backend/src/test/java/dev/evolution/monitor/HistoryQueryPlanTest.java": "fb5660ea1860b868e6dd9eed0a8d80bf2d4f9eac817bfca3b4e4c08574b58c23",
+  "backend/src/test/java/dev/evolution/monitor/HttpObservationsTest.java": "a917d048a3265c83f0e9080e26408df87cae6ff069f983a15b78772d7c33d6a6",
+  "backend/src/test/java/dev/evolution/monitor/MonitorFunctionalTest.java": "bacff4a53750f1290d3d8f990f94de6718fd11f5306edf91ea8d2f6db6ed22f3",
+  "backend/src/test/java/dev/evolution/monitor/OperationsIntegrationTest.java": "264c758e4e2c8ef3ab3b688573ae30c2198fee0ed1d05b4f3583ca0cd16cc487",
+  "backend/src/test/java/dev/evolution/monitor/OwnershipAuthorizationTest.java": "0ce73cb2342372b723b15aa1e62f8609502f468b78fc91269389621749afda0a",
+  "backend/src/test/java/dev/evolution/monitor/OwnershipMigrationTest.java": "dccf198064881691b04f512ab3b2d29a2558d665ac6947e8aa556b2ace416dc9",
+  "backend/src/test/java/dev/evolution/monitor/PostgresPersistenceTest.java": "4ee275984bc1824904fe8cabd69e8573771fa01d7f4df2b1ea044ada6cd61519",
+  "backend/src/test/java/dev/evolution/monitor/SessionAuthenticationTest.java": "069615fcd7106ad4c644bdfcbde27a1921335b927e5e33bbb505a24880bd2237",
+  "backend/src/test/java/dev/evolution/monitor/SessionClient.java": "8d081b46f939152ca0b854eec697fbe27d1a2cbc6d0f38fd68c599b8ef8701bf",
+  "backend/src/test/java/dev/evolution/monitor/TestDatabase.java": "247c98f468b8f5a1322f41b4287209f7ecdb316db2b1bfc9f47d16301bb5ad8c",
+  "backend/src/test/java/dev/evolution/monitor/UserAccountPersistenceTest.java": "f37398f0d10b2e8390005a029959a41ef15b164a042c399042c784ceb6a922ce",
+  "backend/src/test/java/dev/evolution/monitor/WorkerRecoveryTest.java": "b5300f5767d14ba787f2b1f3d63ccd2125561a628c68421958f68b0fb8f96cd1",
+  "compose.e24.yaml": "28a9daf0021b0127335044ef31fdb30bf7451cbdf1fcc8f35586d7bc33b09111",
+  "compose.production.yaml": "1f995bb361c0e90dcc562cfe82ff05067137d900b392a270e361efb9ae26de89",
+  "compose.yaml": "8797ce443b21886bda33b74649c0778c8fbc4b9f04a35ca7c1ff568fa9df46dd",
+  "evidence/phase-1/E24/fixture.json": "47f86919e5f251c3f8ee394727235b906fc03a5eac67e36cc3e273486a21a8fd",
+  "evidence/phase-1/E24/fixtures.md": "5574f4d4d6c8e8cbbd17273ba8385e3a1ac43c2193d34d40b0dca4da11090a1c",
+  "next.config.mjs": "bbf15404bb72ee9b1abe5e7a22e99ab9398c2dba6d8e6e2025593639fcb3d7d7",
+  "package-lock.json": "b8565d7775b725f2b12e9d2a2736b171eeac2eda4c4e12ca112eb250322926fd",
+  "package.json": "ef05c9197eda68c152eed320d91570a327b39266af2d0ab189a440d450ef5948",
+  "playwright.config.ts": "b7cf8bf296372a6f72dc2289b9372e0de8281f90df2c1bfe8589d2be010b0234",
+  "scripts/bootstrap-users.mjs": "3aa00ccb62340905b90ae4fb0b333a0cbfc3422d27d51ef13c24baa6091f264a",
+  "scripts/database.mjs": "5cce532eb3c414de41d1b6e059cc857d90110f25314e1516f518db4d6c10b51e",
+  "scripts/e04-baseline.mjs": "4e515dd0565bfae4a24c418292a02ec1161d68023f223aa8688c61544d058ce3",
+  "scripts/e09-baseline.mjs": "3cbea07b6fb24673eaa36df86a926116c6e454d6af0c7ca11876bec731e56986",
+  "scripts/e10-baseline.mjs": "c0b28e17b497378a7eee2ba598b45e0d7e5890ebd0e88b0387df36d694c2690e",
+  "scripts/e11-baseline.mjs": "4c888213e588f707b8ec9b893afa35afdbb591b50180d538a50598ebaedf8854",
+  "scripts/e12-baseline.mjs": "53f6da8f313709964f546e6051d219a93e2d05b23209b75b275b85e7fbef05b9",
+  "scripts/e24-baseline.mjs": "37bdf00c299706b5e5ef2d7ff11311605a9529fa0ee7abcfa9ef78dd62ce49c4",
+  "scripts/e24-container.mjs": "aa444c00ed2054557177e2d10fa8f5aac6dfb54faeaabf144d683cf09cc776e8",
+  "scripts/e24-fixture.mjs": "20bfd59220394c824a1910183e8857b707164e7d4387957846c8bc155cc08509",
+  "scripts/e24-seed.mjs": "16d0f9286acfe76a5dfb79819d0ee49e975098ecb710464327d41c0f54ec4025",
+  "scripts/fixture.mjs": "e8b17f79bc2705aba5e21a12a964f276e89f9b54aa2e964b10a983d3ffa74ef9",
+  "scripts/persistence-isolation.mjs": "15b9a15b8c250676db78137a0da9c3f937fa022af024994d7477a837209d9127",
+  "scripts/persistence-scenario.mjs": "8cbeba58ea1c13dd41415a75196a8020e9c7a75bf2f59b5d78cd7813e83abd39",
+  "scripts/test-api.mjs": "e87e0ab0604fb0417baf84152dcf319dad29c9433c32219d0828f7baeccef9c1",
+  "scripts/verify.mjs": "ac7ae87edd09c43faaf3e12bf7688391c82219aedec3127b9e1c06c865eb7789",
+  "tests/browser/authenticated.ts": "22f3f8ce1e5df9a3afc4327af452d5c9cff0e3de650960fd67cab3652a73e4c2",
+  "tests/browser/errors.spec.ts": "513176c9d86731f47c44d03f084864371e1021cbf93bf824d6b6350d85f5f2d5",
+  "tests/browser/history.spec.ts": "cb2856bd76099dfeb936355824acf1664573e473bd6947cc9ebbee66bffe2b81",
+  "tests/browser/monitor.spec.ts": "43c641a54bb5959fc99841eb58accfbf4a6ccba3f1bfd26e5afaca91a689c57d",
+  "tests/browser/ownership.spec.ts": "3dd51622fc3c9738d7463fe3d7edf493577e9ca14d9d2d1fd024ad655ef57719",
+  "tests/browser/rendering.spec.ts": "b9cdc02ead1e3c424a0a8016c276c07e8cc30165e5fee8f90f76ce3b5ca837e0",
+  "tests/browser/server-state.spec.ts": "d22a15dcccf1b8f5ec7ba29bf6b4880d4bd51f77c65306b9872f31012f15748c",
+  "tests/browser/session.spec.ts": "65fef5630a2d93700959e6aabb7a792d9345bd1e7e6198aac1126ec988701aa8",
+  "tests/browser/worker.spec.ts": "b0843f9f87d31fe30af281f09e3d0fb504327d5193c8c53eae6ff93657d78d5e",
+  "tsconfig.json": "12bd44eb4271af9528a30352c2d61c727eafef12a865096aff5282bcdcd52608"
+}


