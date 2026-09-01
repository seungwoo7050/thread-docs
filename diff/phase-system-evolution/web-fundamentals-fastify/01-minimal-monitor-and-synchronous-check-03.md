## `E01 기본 검증을 CI에 연결`

diff --git a/.github/workflows/check.yml b/.github/workflows/check.yml
new file mode 100644
index 0000000..1899fe6
--- /dev/null
+++ b/.github/workflows/check.yml
@@ -0,0 +1,37 @@
+name: Fundamentals checks
+
+on:
+  push:
+    branches: [track/fundamentals-fastify]
+  pull_request:
+    branches: [track/fundamentals-fastify]
+
+permissions:
+  contents: read
+
+jobs:
+  verify:
+    runs-on: ubuntu-24.04
+    timeout-minutes: 10
+    env:
+      NEXT_TELEMETRY_DISABLED: '1'
+    steps:
+      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
+      - uses: actions/setup-node@49933ea5288caeca8642d1e84afbd3f7d6820020 # v4.4.0
+        with:
+          node-version-file: .node-version
+          cache: npm
+      - name: Pin npm
+        run: npm install --global npm@11.17.0
+      - run: npm ci
+      - run: npm run typecheck
+      - name: Unit
+        run: npm run test:unit
+      - name: Functional
+        run: npm run test:functional
+      - name: Install pinned Chromium
+        run: npx playwright install --with-deps chromium
+      - name: Browser E2E
+        run: npm run test:e2e
+      - name: Next build
+        run: npm run build
diff --git a/TRACK.md b/TRACK.md
index ede90d6..1474226 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -78,3 +78,13 @@ database, worker, Redis, Kafka, or production container in E01.
 The E2E command owns ports 4311–4313 and starts fresh fixture/API/Next processes;
 stop manual development processes first. It uses one Chromium worker, no test
 retries, and the fixed Monitor fixture in `evidence/E01/scenario.json`.
+
+## Basic CI
+
+`.github/workflows/check.yml` runs the type, unit, functional, Chromium E2E, and
+Next build checks. It selects Node from `.node-version`, installs npm 11.17.0,
+and installs dependencies from the lockfile. CI actions are pinned to the full
+commits for [checkout 4.2.2](https://github.com/actions/checkout/releases/tag/v4.2.2)
+and [setup-node 4.4.0](https://github.com/actions/setup-node/releases/tag/v4.4.0).
+Local invocation outcomes and counts are recorded in `evidence/E01/verification.json`;
+a hosted CI run is not claimed by those local results.
diff --git a/evidence/E01/verification.json b/evidence/E01/verification.json
index ac67ce6..9c91f55 100644
--- a/evidence/E01/verification.json
+++ b/evidence/E01/verification.json
@@ -13,11 +13,18 @@
       "browser": "Chromium 151.0.7922.34 (revision 1234)", "screenshot": "output/playwright/E01-success.png (local generated artifact, visually inspected)" },
     { "command": "npm run typecheck", "invocation": 3, "exitCode": 0, "elapsedSeconds": 1.74,
       "note": "Now generates Next route types before tsc so a clean checkout is supported." },
-    { "command": "npm run build", "invocation": 1, "exitCode": 0, "elapsedSeconds": 5.66 }
+    { "command": "npm run build", "invocation": 1, "exitCode": 0, "elapsedSeconds": 5.66 },
+    { "command": "Ruby YAML workflow gate checker", "invocation": 1, "exitCode": 1, "elapsedSeconds": 0.12,
+      "observed": "Host Ruby lacks Array#filter_map; YAML itself parsed. The temporary checker was changed to map.compact, not the workflow." },
+    { "command": "Ruby YAML workflow gate checker (map.compact)", "invocation": 2, "exitCode": 0, "elapsedSeconds": 0.07,
+      "observed": "Unit, functional, browser gates and Node version-file configuration verified." },
+    { "command": "npm ci --offline", "invocation": 1, "exitCode": 0, "elapsedSeconds": 4.32,
+      "observed": "Lockfile installation succeeded. npm warned that optional macOS fsevents@2.3.2 install script has no allowScripts approval; no script approval was granted." }
   ],
   "measurement": "/usr/bin/time -p; each command run with fnm exec --using 24.19.0",
   "loadRuns": 0,
   "benchmarkRuns": 0,
   "automaticRetries": 0,
-  "parameterChangesAfterObservation": 0
+  "parameterChangesAfterObservation": 0,
+  "hostedCiExecuted": false
 }
