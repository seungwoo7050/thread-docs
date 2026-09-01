## `test(load): fault scenario 설정과 report 검증`

diff --git a/tests/load/fault-scenario.test.mjs b/tests/load/fault-scenario.test.mjs
new file mode 100644
index 0000000..d3b9d03
--- /dev/null
+++ b/tests/load/fault-scenario.test.mjs
@@ -0,0 +1,182 @@
+import assert from "node:assert/strict";
+import test from "node:test";
+import {
+  createFaultScenarioConfig,
+  formatFaultReport,
+  runFaultScenario
+} from "./fault-scenario.mjs";
+
+const apiReadinessUrl = "http://127.0.0.1:14000/health/ready";
+const edgeReadinessUrl = "http://127.0.0.1:18080/api/health/ready";
+
+test("fault scenario defaults to loopback targets and a 300ms database delay", () => {
+  assert.deepEqual(createFaultScenarioConfig({}), {
+    toxiproxyApiUrl: "http://127.0.0.1:8474",
+    apiReadinessUrl,
+    edgeReadinessUrl,
+    databaseLatencyMs: 300,
+    edgeLatencyMs: 150,
+    requestTimeoutMs: 5_000,
+    recoveryTimeoutMs: 15_000,
+    pollIntervalMs: 250,
+    includeEdge: true
+  });
+
+  assert.equal(
+    createFaultScenarioConfig({ FAULT_INCLUDE_EDGE: "0" }).includeEdge,
+    false
+  );
+});
+
+test("fault scenario refuses to alter a non-loopback target", () => {
+  for (const environment of [
+    { TOXIPROXY_API_URL: "http://toxiproxy.example.com:8474" },
+    { FAULT_API_READINESS_URL: "https://api.example.com/health/ready" },
+    { FAULT_EDGE_READINESS_URL: "http://192.0.2.10/api/health/ready" }
+  ]) {
+    assert.throws(
+      () => createFaultScenarioConfig(environment),
+      /loopback/
+    );
+  }
+});
+
+test("fault scenario records database and edge failure recovery as JSON", async () => {
+  const commands = [];
+  const sleeps = [];
+  const probeResults = new Map([
+    [apiReadinessUrl, [
+      ready(200, 9),
+      ready(200, 318),
+      ready(200, 4),
+      notReady(503, 6),
+      notReady(503, 5),
+      ready(200, 8)
+    ]],
+    [edgeReadinessUrl, [
+      ready(200, 165),
+      { status: null, durationMs: 3, error: "socket reset" },
+      ready(200, 7)
+    ]]
+  ]);
+
+  const report = await runFaultScenario(createFaultScenarioConfig({}), {
+    applyToxiproxyCommand: async (command, args = []) => {
+      commands.push([command, args]);
+    },
+    probeReadiness: async (url) => {
+      const result = probeResults.get(url)?.shift();
+      assert.ok(result, `unexpected readiness probe: ${url}`);
+      return result;
+    },
+    sleep: async (durationMs) => {
+      sleeps.push(durationMs);
+    },
+    now: sequence(
+      "2026-07-23T03:00:00.000Z",
+      "2026-07-23T03:00:03.000Z"
+    )
+  });
+
+  assert.deepEqual(commands, [
+    ["reset", []],
+    ["db-latency", ["300", "0"]],
+    ["db-down", []],
+    ["db-up", []],
+    ["edge-latency", ["150", "0"]],
+    ["edge-reset", ["0"]],
+    ["edge-up", []],
+    ["reset", []]
+  ]);
+  assert.deepEqual(sleeps, [250, 250]);
+  assert.equal(report.schemaVersion, 1);
+  assert.equal(report.startedAt, "2026-07-23T03:00:00.000Z");
+  assert.equal(report.finishedAt, "2026-07-23T03:00:03.000Z");
+  assert.equal(report.passed, true);
+  assert.deepEqual(report.targets, {
+    toxiproxyApiUrl: "http://127.0.0.1:8474",
+    apiReadinessUrl,
+    edgeReadinessUrl
+  });
+  assert.deepEqual(
+    report.steps.map(({ name, passed }) => [name, passed]),
+    [
+      ["baseline", true],
+      ["database_latency", true],
+      ["database_down", true],
+      ["database_recovery", true],
+      ["edge_latency", true],
+      ["edge_reset", true],
+      ["edge_recovery", true]
+    ]
+  );
+  assert.equal(report.steps.find(({ name }) => name === "database_latency").durationMs, 318);
+  assert.equal(
+    report.steps.find(({ name }) => name === "database_down").body.checks.database,
+    "down"
+  );
+  assert.equal(report.steps.find(({ name }) => name === "edge_reset").error, "socket reset");
+
+  assert.deepEqual(JSON.parse(formatFaultReport(report)), report);
+});
+
+test("fault scenario resets every proxy when a command fails", async () => {
+  const commands = [];
+
+  await assert.rejects(
+    runFaultScenario(createFaultScenarioConfig({ FAULT_INCLUDE_EDGE: "0" }), {
+      applyToxiproxyCommand: async (command, args = []) => {
+        commands.push([command, args]);
+        if (command === "db-down") throw new Error("control unavailable");
+      },
+      probeReadiness: async () => ready(200, 5),
+      sleep: async () => {},
+      now: () => "2026-07-23T03:00:00.000Z"
+    }),
+    /control unavailable/
+  );
+
+  assert.deepEqual(commands, [
+    ["reset", []],
+    ["db-latency", ["300", "0"]],
+    ["db-down", []],
+    ["reset", []]
+  ]);
+});
+
+function ready(status, durationMs) {
+  return {
+    status,
+    durationMs,
+    body: {
+      status: "ready",
+      service: "pong-pong-api",
+      checks: {
+        lifecycle: "accepting",
+        database: "up",
+        migrations: "current"
+      }
+    }
+  };
+}
+
+function notReady(status, durationMs) {
+  return {
+    status,
+    durationMs,
+    body: {
+      status: "not_ready",
+      service: "pong-pong-api",
+      checks: {
+        lifecycle: "accepting",
+        database: "down",
+        migrations: "unknown"
+      }
+    }
+  };
+}
+
+function sequence(...values) {
+  let index = 0;
+  return () => values[Math.min(index++, values.length - 1)];
+}


## `ci(e2e): 비회원 체험 browser job 실행`

diff --git a/.github/workflows/ci.yml b/.github/workflows/ci.yml
index 73e472a..57defb5 100644
--- a/.github/workflows/ci.yml
+++ b/.github/workflows/ci.yml
@@ -138,7 +138,7 @@ jobs:
           }
           trap cleanup EXIT
 
-          for attempt in $(seq 1 60); do
+          for _ in $(seq 1 60); do
             if curl --fail --silent http://127.0.0.1:4000/health/ready >/dev/null; then
               break
             fi
@@ -146,7 +146,7 @@ jobs:
           done
           curl --fail --silent http://127.0.0.1:4000/health/ready >/dev/null
 
-          for attempt in $(seq 1 60); do
+          for _ in $(seq 1 60); do
             if curl --fail --silent http://127.0.0.1:3000 >/dev/null; then
               break
             fi
@@ -157,3 +157,71 @@ jobs:
           pnpm smoke:http
           pnpm smoke:ws
           pnpm e2e
+
+  guest-demo-browser:
+    name: Guest demo browser tests
+    runs-on: ubuntu-latest
+    timeout-minutes: 20
+    env:
+      APP_MODE: demo
+      E2E_APP_MODE: demo
+      E2E_BASE_URL: http://localhost:3000
+      NEXT_PUBLIC_APP_MODE: demo
+      NEXT_PUBLIC_API_BASE_URL: http://localhost:4000
+      SESSION_SECRET: ci-guest-demo-session-secret-32-bytes
+      LOG_LEVEL: warn
+
+    steps:
+      - name: Check out repository
+        uses: actions/checkout@v4
+
+      - name: Set up pnpm
+        uses: pnpm/action-setup@v4
+        with:
+          version: 10.32.1
+
+      - name: Set up Node.js
+        uses: actions/setup-node@v4
+        with:
+          node-version: 24.18.0
+          cache: pnpm
+
+      - name: Install dependencies
+        run: pnpm install --frozen-lockfile
+
+      - name: Install Chromium
+        run: pnpm exec playwright install --with-deps chromium
+
+      - name: Build demo processes
+        run: pnpm build
+
+      - name: Run tests/e2e/guest-demo.spec.ts
+        run: |
+          set -euo pipefail
+          node apps/api/dist/index.js >/tmp/pong-demo-api.log 2>&1 &
+          API_PID=$!
+          pnpm --filter @pong-pong/web start >/tmp/pong-demo-web.log 2>&1 &
+          WEB_PID=$!
+          cleanup() {
+            kill "$API_PID" "$WEB_PID" 2>/dev/null || true
+            wait "$API_PID" "$WEB_PID" 2>/dev/null || true
+          }
+          trap cleanup EXIT
+
+          for _ in $(seq 1 60); do
+            if curl --fail --silent http://localhost:4000/health/ready >/dev/null; then
+              break
+            fi
+            sleep 1
+          done
+          curl --fail --silent http://localhost:4000/health/ready >/dev/null
+
+          for _ in $(seq 1 60); do
+            if curl --fail --silent http://localhost:3000 >/dev/null; then
+              break
+            fi
+            sleep 1
+          done
+          curl --fail --silent http://localhost:3000 >/dev/null
+
+          pnpm e2e:guest-demo


## `test(repo): 정적 계약 검사 명령 연결`

diff --git a/Makefile b/Makefile
index 59dc34f..a0601cf 100644
--- a/Makefile
+++ b/Makefile
@@ -1,4 +1,4 @@
-.PHONY: install typecheck build test unit smoke smoke-http smoke-ws e2e dev down
+.PHONY: install typecheck build test unit contracts smoke smoke-http smoke-ws e2e dev down
 
 install:
 	pnpm install
@@ -15,6 +15,9 @@ test:
 unit:
 	pnpm unit
 
+contracts:
+	pnpm test:contracts
+
 smoke: smoke-http smoke-ws
 
 smoke-http:
diff --git a/package.json b/package.json
index 29105db..78e4575 100644
--- a/package.json
+++ b/package.json
@@ -12,6 +12,7 @@
     "typecheck": "pnpm -r typecheck",
     "test": "pnpm unit",
     "unit": "pnpm -r test",
+    "test:contracts": "node --test tests/ci-contract.test.mjs tests/docker-production.test.mjs tests/load/fault-scenario.test.mjs tests/load/load-harness.test.mjs",
     "postgres-integration": "pnpm --filter @pong-pong/db postgres-integration",
     "load:faults": "node tests/load/fault-scenario.mjs",
     "smoke:http": "node tests/smoke-api.mjs",


## `ci(repo): 정적 계약 검사 실행`

diff --git a/.github/workflows/ci.yml b/.github/workflows/ci.yml
index 57defb5..3eacb98 100644
--- a/.github/workflows/ci.yml
+++ b/.github/workflows/ci.yml
@@ -37,6 +37,9 @@ jobs:
       - name: Run unit tests
         run: pnpm unit
 
+      - name: Run static contract tests
+        run: pnpm test:contracts
+
       - name: Build
         run: pnpm build
 


## `fix(ci): 브라우저 E2E API origin 정렬`

diff --git a/.github/workflows/ci.yml b/.github/workflows/ci.yml
index 9d0a41b..f37e690 100644
--- a/.github/workflows/ci.yml
+++ b/.github/workflows/ci.yml
@@ -92,7 +92,7 @@ jobs:
           --health-retries 20
     env:
       DATABASE_URL: postgresql://pong:pong@127.0.0.1:5432/pong_pong_test
-      API_BASE_URL: http://127.0.0.1:4000
+      API_BASE_URL: http://localhost:4000
       WS_URL: ws://127.0.0.1:4000/ws
 
     steps:


## `test(ci): 브라우저 E2E cookie origin 계약 검증`

diff --git a/tests/ci-contract.test.mjs b/tests/ci-contract.test.mjs
index 495e8d8..1a91290 100644
--- a/tests/ci-contract.test.mjs
+++ b/tests/ci-contract.test.mjs
@@ -30,6 +30,10 @@ test("CI separates unit, PostgreSQL integration, process smoke, and browser E2E"
   assert.match(workflow, /pnpm --filter @pong-pong\/db seed:dev/);
 });
 
+test("CI keeps registered browser API requests on the login cookie host", () => {
+  assert.match(workflow, /^      API_BASE_URL: http:\/\/localhost:4000$/m);
+});
+
 test("CI runs the Guest browser flow against a dedicated demo process", () => {
   assert.match(workflow, /guest-demo-browser:/);
   assert.match(workflow, /APP_MODE:\s*demo/);


## `build: improve Makefile and separate functional pong checks`

diff --git a/Makefile b/Makefile
index a0601cf..e3d90b2 100644
--- a/Makefile
+++ b/Makefile
@@ -1,36 +1,93 @@
-.PHONY: install typecheck build test unit contracts smoke smoke-http smoke-ws e2e dev down
+PNPM ?= pnpm
+COMPOSE ?= docker compose
+.DEFAULT_GOAL := install
+
+.PHONY: help install install-update typecheck build test unit unit-functional contracts \
+	verify-build check postgres-integration smoke smoke-http smoke-ws e2e \
+	e2e-guest-demo compose-config dev down
+
+help:
+	@printf '%s\n' \
+		'Local verification:' \
+		'  install              install the exact lockfile dependency graph' \
+		'  install-update       install dependencies and allow lockfile updates' \
+		'  typecheck            type-check every workspace package' \
+		'  unit                 run workspace unit tests' \
+		'  unit-functional      run unit tests without documentation policy' \
+		'  contracts            run static CI and container contract tests' \
+		'  build                 build packages in dependency order' \
+		'  verify-build          verify expected production artifacts' \
+		'  check                 run the non-container CI verification sequence' \
+		'  postgres-integration  run integration tests against PostgreSQL' \
+		'' \
+		'Runtime verification:' \
+		'  smoke                 run HTTP and WebSocket smoke tests in order' \
+		'  e2e                   run the Playwright browser suite' \
+		'  e2e-guest-demo        run the guest-demo Playwright suite' \
+		'' \
+		'Compose lifecycle:' \
+		'  compose-config        validate the resolved Compose configuration' \
+		'  dev                   build and run the Compose stack in foreground' \
+		'  down                  stop the stack and remove orphan containers'
 
 install:
-	pnpm install
+	$(PNPM) install --frozen-lockfile
+
+install-update:
+	$(PNPM) install
 
 typecheck:
-	pnpm -r typecheck
+	$(PNPM) typecheck
 
 build:
-	pnpm -r build
+	$(PNPM) build
 
 test:
-	pnpm unit
+	$(PNPM) unit
 
 unit:
-	pnpm unit
+	$(PNPM) unit
+
+unit-functional:
+	$(PNPM) unit:functional
 
 contracts:
-	pnpm test:contracts
+	$(PNPM) test:contracts
+
+verify-build:
+	$(PNPM) verify:build
+
+check:
+	$(MAKE) typecheck
+	$(MAKE) unit-functional
+	$(MAKE) contracts
+	$(MAKE) build
+	$(MAKE) verify-build
+
+postgres-integration:
+	$(PNPM) postgres-integration
 
-smoke: smoke-http smoke-ws
+smoke:
+	$(MAKE) smoke-http
+	$(MAKE) smoke-ws
 
 smoke-http:
-	pnpm smoke:http
+	$(PNPM) smoke:http
 
 smoke-ws:
-	pnpm smoke:ws
+	$(PNPM) smoke:ws
+
+e2e:
+	$(PNPM) e2e
+
+e2e-guest-demo:
+	$(PNPM) e2e:guest-demo
+
+compose-config:
+	$(COMPOSE) config --quiet
 
 dev:
-	docker compose up --build
+	$(COMPOSE) up --build
 
 down:
-	docker compose down --remove-orphans
-
-e2e:
-	pnpm e2e
+	$(COMPOSE) down --remove-orphans
diff --git a/apps/api/package.json b/apps/api/package.json
index c0e2dc9..d9dc63a 100644
--- a/apps/api/package.json
+++ b/apps/api/package.json
@@ -8,7 +8,8 @@
     "typecheck": "tsc --noEmit",
     "dev": "tsx watch src/index.ts",
     "start": "node dist/index.js",
-    "test": "vitest run"
+    "test": "vitest run",
+    "test:functional": "vitest run --exclude src/documentation.test.ts"
   },
   "dependencies": {
     "@fastify/cookie": "^11.0.1",
diff --git a/package.json b/package.json
index 6f808c5..86dc7d0 100644
--- a/package.json
+++ b/package.json
@@ -12,6 +12,7 @@
     "typecheck": "pnpm -r typecheck",
     "test": "pnpm unit",
     "unit": "pnpm -r test",
+    "unit:functional": "pnpm --filter @pong-pong/shared test && pnpm --filter @pong-pong/db test && pnpm --filter @pong-pong/api test:functional && pnpm --filter @pong-pong/web test",
     "test:contracts": "node --test tests/ci-contract.test.mjs tests/docker-production.test.mjs tests/load/fault-scenario.test.mjs tests/load/load-harness.test.mjs",
     "postgres-integration": "pnpm --filter @pong-pong/db postgres-integration",
     "load:faults": "node tests/load/fault-scenario.mjs",
@@ -28,7 +29,8 @@
     "@playwright/test": "^1.52.0",
     "@types/node": "^22.15.3",
     "typescript": "^5.8.3",
-    "vitest": "^3.1.2"
+    "vitest": "^3.1.2",
+    "yaml": "^2.9.0"
   },
   "pnpm": {
     "overrides": {


