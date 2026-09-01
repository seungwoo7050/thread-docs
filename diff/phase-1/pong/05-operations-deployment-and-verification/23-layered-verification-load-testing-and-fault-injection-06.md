## `ci: harden pong validation`

diff --git a/.github/workflows/ci.yml b/.github/workflows/ci.yml
deleted file mode 100644
index f37e690..0000000
--- a/.github/workflows/ci.yml
+++ /dev/null
@@ -1,230 +0,0 @@
-name: CI
-
-on:
-  push:
-  pull_request:
-
-permissions:
-  contents: read
-
-jobs:
-  verify:
-    name: Typecheck, unit tests, and build
-    runs-on: ubuntu-latest
-    timeout-minutes: 15
-
-    steps:
-      - name: Check out repository
-        uses: actions/checkout@v4
-
-      - name: Set up pnpm
-        uses: pnpm/action-setup@v4
-        with:
-          version: 10.32.1
-
-      - name: Set up Node.js
-        uses: actions/setup-node@v4
-        with:
-          node-version: 24.18.1
-          cache: pnpm
-
-      - name: Install dependencies
-        run: pnpm install --frozen-lockfile
-
-      - name: Typecheck
-        run: pnpm typecheck
-
-      - name: Run unit tests
-        run: pnpm unit
-
-      - name: Run static contract tests
-        run: pnpm test:contracts
-
-      - name: Build
-        run: pnpm build
-
-      - name: Verify production artifacts
-        run: pnpm verify:build
-
-  postgres-integration:
-    name: PostgreSQL integration tests
-    runs-on: ubuntu-latest
-    timeout-minutes: 15
-
-    steps:
-      - name: Check out repository
-        uses: actions/checkout@v4
-
-      - name: Set up pnpm
-        uses: pnpm/action-setup@v4
-        with:
-          version: 10.32.1
-
-      - name: Set up Node.js
-        uses: actions/setup-node@v4
-        with:
-          node-version: 24.18.1
-          cache: pnpm
-
-      - name: Install dependencies
-        run: pnpm install --frozen-lockfile
-
-      - name: Run PostgreSQL integration tests
-        run: pnpm postgres-integration
-
-  process-and-browser:
-    name: HTTP, WebSocket, and browser tests
-    runs-on: ubuntu-latest
-    timeout-minutes: 20
-    services:
-      postgres:
-        image: postgres:16-alpine
-        env:
-          POSTGRES_DB: pong_pong_test
-          POSTGRES_USER: pong
-          POSTGRES_PASSWORD: pong
-        ports:
-          - 5432:5432
-        options: >-
-          --health-cmd "pg_isready -U pong -d pong_pong_test"
-          --health-interval 5s
-          --health-timeout 3s
-          --health-retries 20
-    env:
-      DATABASE_URL: postgresql://pong:pong@127.0.0.1:5432/pong_pong_test
-      API_BASE_URL: http://localhost:4000
-      WS_URL: ws://127.0.0.1:4000/ws
-
-    steps:
-      - name: Check out repository
-        uses: actions/checkout@v4
-
-      - name: Set up pnpm
-        uses: pnpm/action-setup@v4
-        with:
-          version: 10.32.1
-
-      - name: Set up Node.js
-        uses: actions/setup-node@v4
-        with:
-          node-version: 24.18.1
-          cache: pnpm
-
-      - name: Install dependencies
-        run: pnpm install --frozen-lockfile
-
-      - name: Install Chromium
-        run: pnpm exec playwright install --with-deps chromium
-
-      - name: Build production processes
-        run: pnpm build
-
-      - name: Prepare the process-test database
-        run: |
-          pnpm --filter @pong-pong/db migrate
-          pnpm --filter @pong-pong/db seed:dev
-
-      - name: Run process smoke and browser tests
-        env:
-          APP_MODE: development
-          SESSION_SECRET: ci-process-session-secret-32-bytes
-          LOG_LEVEL: warn
-        run: |
-          set -euo pipefail
-          node apps/api/dist/index.js >/tmp/pong-api.log 2>&1 &
-          API_PID=$!
-          pnpm --filter @pong-pong/web start >/tmp/pong-web.log 2>&1 &
-          WEB_PID=$!
-          cleanup() {
-            kill "$API_PID" "$WEB_PID" 2>/dev/null || true
-            wait "$API_PID" "$WEB_PID" 2>/dev/null || true
-          }
-          trap cleanup EXIT
-
-          for _ in $(seq 1 60); do
-            if curl --fail --silent http://127.0.0.1:4000/health/ready >/dev/null; then
-              break
-            fi
-            sleep 1
-          done
-          curl --fail --silent http://127.0.0.1:4000/health/ready >/dev/null
-
-          for _ in $(seq 1 60); do
-            if curl --fail --silent http://127.0.0.1:3000 >/dev/null; then
-              break
-            fi
-            sleep 1
-          done
-          curl --fail --silent http://127.0.0.1:3000 >/dev/null
-
-          pnpm smoke:http
-          pnpm smoke:ws
-          pnpm e2e
-
-  guest-demo-browser:
-    name: Guest demo browser tests
-    runs-on: ubuntu-latest
-    timeout-minutes: 20
-    env:
-      APP_MODE: demo
-      E2E_APP_MODE: demo
-      E2E_BASE_URL: http://localhost:3000
-      NEXT_PUBLIC_APP_MODE: demo
-      NEXT_PUBLIC_API_BASE_URL: http://localhost:4000
-      SESSION_SECRET: ci-guest-demo-session-secret-32-bytes
-      LOG_LEVEL: warn
-
-    steps:
-      - name: Check out repository
-        uses: actions/checkout@v4
-
-      - name: Set up pnpm
-        uses: pnpm/action-setup@v4
-        with:
-          version: 10.32.1
-
-      - name: Set up Node.js
-        uses: actions/setup-node@v4
-        with:
-          node-version: 24.18.1
-          cache: pnpm
-
-      - name: Install dependencies
-        run: pnpm install --frozen-lockfile
-
-      - name: Install Chromium
-        run: pnpm exec playwright install --with-deps chromium
-
-      - name: Build demo processes
-        run: pnpm build
-
-      - name: Run tests/e2e/guest-demo.spec.ts
-        run: |
-          set -euo pipefail
-          node apps/api/dist/index.js >/tmp/pong-demo-api.log 2>&1 &
-          API_PID=$!
-          pnpm --filter @pong-pong/web start >/tmp/pong-demo-web.log 2>&1 &
-          WEB_PID=$!
-          cleanup() {
-            kill "$API_PID" "$WEB_PID" 2>/dev/null || true
-            wait "$API_PID" "$WEB_PID" 2>/dev/null || true
-          }
-          trap cleanup EXIT
-
-          for _ in $(seq 1 60); do
-            if curl --fail --silent http://localhost:4000/health/ready >/dev/null; then
-              break
-            fi
-            sleep 1
-          done
-          curl --fail --silent http://localhost:4000/health/ready >/dev/null
-
-          for _ in $(seq 1 60); do
-            if curl --fail --silent http://localhost:3000 >/dev/null; then
-              break
-            fi
-            sleep 1
-          done
-          curl --fail --silent http://localhost:3000 >/dev/null
-
-          pnpm e2e:guest-demo
diff --git a/.github/workflows/web-ft-transcendence-ci.yml b/.github/workflows/web-ft-transcendence-ci.yml
new file mode 100644
index 0000000..4254233
--- /dev/null
+++ b/.github/workflows/web-ft-transcendence-ci.yml
@@ -0,0 +1,304 @@
+name: web/ft_transcendence CI
+
+on:
+  push:
+    branches: [web/ft_transcendence]
+  pull_request:
+    branches: [web/ft_transcendence]
+
+permissions:
+  contents: read
+
+concurrency:
+  group: web-ft-transcendence-${{ github.workflow }}-${{ github.ref }}
+  cancel-in-progress: true
+
+jobs:
+  verify:
+    name: Typecheck, unit tests, and build
+    runs-on: ubuntu-24.04
+    timeout-minutes: 15
+    steps:
+      - name: Check out repository
+        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
+        with:
+          persist-credentials: false
+
+      - name: Set up pnpm
+        uses: pnpm/action-setup@0977fd99725f1db4007ccb2928dbb4e90d06cc86 # v6.0.10
+        with:
+          version: 10.32.1
+
+      - name: Set up Node.js
+        uses: actions/setup-node@820762786026740c76f36085b0efc47a31fe5020 # v7.0.0
+        with:
+          node-version-file: .node-version
+          cache: pnpm
+          cache-dependency-path: pnpm-lock.yaml
+
+      - name: Install dependencies
+        run: make install
+
+      - name: Run functional verification
+        run: make check
+
+  postgres-integration:
+    name: PostgreSQL integration tests
+    runs-on: ubuntu-24.04
+    timeout-minutes: 15
+    steps:
+      - name: Check out repository
+        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
+        with:
+          persist-credentials: false
+
+      - name: Set up pnpm
+        uses: pnpm/action-setup@0977fd99725f1db4007ccb2928dbb4e90d06cc86 # v6.0.10
+        with:
+          version: 10.32.1
+
+      - name: Set up Node.js
+        uses: actions/setup-node@820762786026740c76f36085b0efc47a31fe5020 # v7.0.0
+        with:
+          node-version-file: .node-version
+          cache: pnpm
+          cache-dependency-path: pnpm-lock.yaml
+
+      - name: Install dependencies
+        run: make install
+
+      - name: Run PostgreSQL integration tests
+        run: make postgres-integration
+
+  process-and-browser:
+    name: HTTP, WebSocket, and browser tests
+    runs-on: ubuntu-24.04
+    timeout-minutes: 20
+    services:
+      postgres:
+        image: postgres:16-alpine
+        env:
+          POSTGRES_DB: pong_pong_test
+          POSTGRES_USER: pong
+          POSTGRES_PASSWORD: pong
+        ports:
+          - 5432:5432
+        options: >-
+          --health-cmd "pg_isready -U pong -d pong_pong_test"
+          --health-interval 5s
+          --health-timeout 3s
+          --health-retries 20
+    env:
+      APP_MODE: development
+      API_BASE_URL: http://localhost:4000
+      DATABASE_URL: postgresql://pong:pong@127.0.0.1:5432/pong_pong_test
+      LOG_LEVEL: warn
+      SESSION_SECRET: ci-process-session-secret-32-bytes
+      WS_URL: ws://127.0.0.1:4000/ws
+    steps:
+      - name: Check out repository
+        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
+        with:
+          persist-credentials: false
+
+      - name: Set up pnpm
+        uses: pnpm/action-setup@0977fd99725f1db4007ccb2928dbb4e90d06cc86 # v6.0.10
+        with:
+          version: 10.32.1
+
+      - name: Set up Node.js
+        uses: actions/setup-node@820762786026740c76f36085b0efc47a31fe5020 # v7.0.0
+        with:
+          node-version-file: .node-version
+          cache: pnpm
+          cache-dependency-path: pnpm-lock.yaml
+
+      - name: Install dependencies
+        run: make install
+
+      - name: Install Chromium headless shell
+        run: pnpm exec playwright install --with-deps --only-shell chromium
+
+      - name: Build production processes
+        run: make build
+
+      - name: Prepare the process-test database
+        run: |
+          pnpm --filter @pong-pong/db migrate
+          pnpm --filter @pong-pong/db seed:dev
+
+      - name: Start API
+        id: api_server
+        run: node apps/api/dist/index.js
+        background: true
+
+      - name: Start web application
+        id: web_server
+        run: pnpm --filter @pong-pong/web start
+        background: true
+
+      - name: Wait for API readiness
+        run: |
+          for _ in $(seq 1 60); do
+            curl --fail --silent http://127.0.0.1:4000/health/ready >/dev/null && exit 0
+            sleep 1
+          done
+          exit 1
+
+      - name: Wait for web readiness
+        run: |
+          for _ in $(seq 1 60); do
+            curl --fail --silent http://127.0.0.1:3000 >/dev/null && exit 0
+            sleep 1
+          done
+          exit 1
+
+      - name: Run HTTP and WebSocket smoke tests
+        run: make smoke
+
+      - name: Run registered-user browser tests
+        run: make e2e
+
+      - name: Stop web application
+        cancel: web_server
+
+      - name: Stop API
+        cancel: api_server
+
+      - name: Upload process and browser diagnostics
+        if: ${{ failure() }}
+        uses: actions/upload-artifact@043fb46d1a93c77aae656e7c1c64a875d1fc6a0a # v7.0.1
+        with:
+          name: pong-process-browser-${{ github.run_id }}-${{ github.run_attempt }}
+          path: |
+            test-results/
+            playwright-report/
+          if-no-files-found: warn
+          retention-days: 7
+
+  guest-demo-browser:
+    name: Guest demo browser tests
+    runs-on: ubuntu-24.04
+    timeout-minutes: 20
+    env:
+      APP_MODE: demo
+      E2E_APP_MODE: demo
+      E2E_BASE_URL: http://localhost:3000
+      LOG_LEVEL: warn
+      NEXT_PUBLIC_API_BASE_URL: http://localhost:4000
+      NEXT_PUBLIC_APP_MODE: demo
+      SESSION_SECRET: ci-guest-demo-session-secret-32-bytes
+    steps:
+      - name: Check out repository
+        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
+        with:
+          persist-credentials: false
+
+      - name: Set up pnpm
+        uses: pnpm/action-setup@0977fd99725f1db4007ccb2928dbb4e90d06cc86 # v6.0.10
+        with:
+          version: 10.32.1
+
+      - name: Set up Node.js
+        uses: actions/setup-node@820762786026740c76f36085b0efc47a31fe5020 # v7.0.0
+        with:
+          node-version-file: .node-version
+          cache: pnpm
+          cache-dependency-path: pnpm-lock.yaml
+
+      - name: Install dependencies
+        run: make install
+
+      - name: Install Chromium headless shell
+        run: pnpm exec playwright install --with-deps --only-shell chromium
+
+      - name: Build demo processes
+        run: make build
+
+      - name: Start demo API
+        id: demo_api
+        run: node apps/api/dist/index.js
+        background: true
+
+      - name: Start demo web application
+        id: demo_web
+        run: pnpm --filter @pong-pong/web start
+        background: true
+
+      - name: Wait for demo API readiness
+        run: |
+          for _ in $(seq 1 60); do
+            curl --fail --silent http://127.0.0.1:4000/health/ready >/dev/null && exit 0
+            sleep 1
+          done
+          exit 1
+
+      - name: Wait for demo web readiness
+        run: |
+          for _ in $(seq 1 60); do
+            curl --fail --silent http://127.0.0.1:3000 >/dev/null && exit 0
+            sleep 1
+          done
+          exit 1
+
+      - name: Run Guest demo browser tests
+        run: make e2e-guest-demo
+
+      - name: Stop demo web application
+        cancel: demo_web
+
+      - name: Stop demo API
+        cancel: demo_api
+
+      - name: Upload Guest demo diagnostics
+        if: ${{ failure() }}
+        uses: actions/upload-artifact@043fb46d1a93c77aae656e7c1c64a875d1fc6a0a # v7.0.1
+        with:
+          name: pong-guest-demo-${{ github.run_id }}-${{ github.run_attempt }}
+          path: |
+            test-results/
+            playwright-report/
+          if-no-files-found: warn
+          retention-days: 7
+
+  production-compose:
+    name: Production Compose smoke test
+    runs-on: ubuntu-24.04
+    timeout-minutes: 30
+    env:
+      APP_MODE: production
+      COMPOSE_PROJECT_NAME: pong-pong-ci-${{ github.run_id }}-${{ github.run_attempt }}
+      POSTGRES_PASSWORD: ci-compose-postgres-password
+      PUBLIC_ORIGIN: http://localhost:8080
+      PUBLIC_WS_URL: ws://localhost:8080/ws
+      SESSION_SECRET: ci-compose-session-secret-32-bytes
+    steps:
+      - name: Check out repository
+        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
+        with:
+          persist-credentials: false
+
+      - name: Verify Compose configuration
+        run: make compose-config
+
+      - name: Build and start production Compose stack
+        run: docker compose up --build --wait --wait-timeout 600
+
+      - name: Verify public production boundaries
+        shell: bash
+        run: |
+          set -euo pipefail
+          curl --fail --silent --show-error http://127.0.0.1:8080/ >/dev/null
+          curl --fail --silent --show-error http://127.0.0.1:8080/api/health/ready >/dev/null
+          metrics_status=$(curl --silent --output /dev/null --write-out '%{http_code}' http://127.0.0.1:8080/api/metrics)
+          test "$metrics_status" = 404
+
+      - name: Print Compose diagnostics
+        if: ${{ failure() }}
+        run: |
+          docker compose ps --all
+          docker compose logs --no-color
+
+      - name: Remove Compose resources
+        if: ${{ always() }}
+        run: docker compose down --volumes --remove-orphans
diff --git a/playwright.config.ts b/playwright.config.ts
index 9a8c73c..23b56ed 100644
--- a/playwright.config.ts
+++ b/playwright.config.ts
@@ -6,6 +6,9 @@ export default defineConfig({
   expect: {
     timeout: 10_000
   },
+  reporter: process.env.CI
+    ? [["github"], ["html", { open: "never" }]]
+    : [["list"]],
   use: {
     baseURL: process.env.E2E_BASE_URL ?? "http://localhost:3000",
     trace: "retain-on-failure",
diff --git a/pnpm-lock.yaml b/pnpm-lock.yaml
index 0489a0c..20c3152 100644
--- a/pnpm-lock.yaml
+++ b/pnpm-lock.yaml
@@ -26,6 +26,9 @@ importers:
       vitest:
         specifier: ^3.1.2
         version: 3.2.4(@types/node@22.19.17)(jiti@1.21.7)(tsx@4.21.0)(yaml@2.9.0)
+      yaml:
+        specifier: ^2.9.0
+        version: 2.9.0
 
   apps/api:
     dependencies:
diff --git a/tests/ci-contract.test.mjs b/tests/ci-contract.test.mjs
index 1a91290..57503a9 100644
--- a/tests/ci-contract.test.mjs
+++ b/tests/ci-contract.test.mjs
@@ -1,44 +1,128 @@
+import assert from "node:assert/strict";
 import { readFileSync } from "node:fs";
 import { resolve } from "node:path";
 import test from "node:test";
-import assert from "node:assert/strict";
+import { parse } from "yaml";
 
-const workflow = readFileSync(resolve(import.meta.dirname, "../.github/workflows/ci.yml"), "utf8");
-const nodeVersion = readFileSync(resolve(import.meta.dirname, "../.node-version"), "utf8").trim();
+const root = resolve(import.meta.dirname, "..");
+const workflowPath = resolve(
+  root,
+  ".github/workflows/web-ft-transcendence-ci.yml",
+);
+const workflow = parse(readFileSync(workflowPath, "utf8"));
+const makefile = readFileSync(resolve(root, "Makefile"), "utf8");
+const nodeVersion = readFileSync(resolve(root, ".node-version"), "utf8").trim();
+const packageJson = JSON.parse(readFileSync(resolve(root, "package.json"), "utf8"));
+const pnpmVersion = packageJson.packageManager.split("@").at(-1);
+const jobs = workflow.jobs;
+const steps = Object.values(jobs).flatMap((job) => job.steps ?? []);
+const runCommands = steps.flatMap((step) =>
+  typeof step.run === "string" ? [step.run] : [],
+);
 
-test("CI pins the repository toolchain in every job", () => {
-  const nodeVersions = [...workflow.matchAll(/node-version:\s*([^\s]+)/g)]
-    .map((match) => match[1]);
-  assert.ok(nodeVersions.length > 0);
-  assert.deepEqual([...new Set(nodeVersions)], [nodeVersion]);
-  assert.match(workflow, /version: 10\.32\.1/);
-  assert.match(workflow, /pnpm install --frozen-lockfile/);
+test("CI targets only the independent project branch", () => {
+  assert.equal(workflow.name, "web/ft_transcendence CI");
+  assert.deepEqual(Object.keys(workflow.on).sort(), ["pull_request", "push"]);
+  assert.deepEqual(workflow.on.push.branches, ["web/ft_transcendence"]);
+  assert.deepEqual(workflow.on.pull_request.branches, ["web/ft_transcendence"]);
+  assert.equal(workflow.on.workflow_dispatch, undefined);
+  assert.equal(workflow.on.pull_request_target, undefined);
+  assert.equal(workflow.on.push["paths-ignore"], undefined);
+  assert.equal(workflow.on.pull_request["paths-ignore"], undefined);
 });
 
-test("CI separates unit, PostgreSQL integration, process smoke, and browser E2E", () => {
-  for (const command of [
-    "pnpm unit",
-    "pnpm postgres-integration",
-    "pnpm smoke:http",
-    "pnpm smoke:ws",
-    "pnpm e2e"
-  ]) {
-    assert.match(workflow, new RegExp(command.replace(":", "\\:")));
+test("CI uses read-only permissions and immutable reviewed actions", () => {
+  assert.deepEqual(workflow.permissions, { contents: "read" });
+  assert.equal(workflow.concurrency["cancel-in-progress"], true);
+  assert.match(workflow.concurrency.group, /github\.workflow/);
+  assert.match(workflow.concurrency.group, /github\.ref/);
+
+  const expectedActions = new Set([
+    "actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1",
+    "actions/setup-node@820762786026740c76f36085b0efc47a31fe5020",
+    "actions/upload-artifact@043fb46d1a93c77aae656e7c1c64a875d1fc6a0a",
+    "pnpm/action-setup@0977fd99725f1db4007ccb2928dbb4e90d06cc86",
+  ]);
+  const actions = steps.flatMap((step) =>
+    typeof step.uses === "string" ? [step.uses] : [],
+  );
+  assert.ok(actions.length > 0);
+  assert.ok(actions.every((action) => expectedActions.has(action)));
+  assert.deepEqual(new Set(actions), expectedActions);
+
+  for (const step of steps.filter((candidate) =>
+    candidate.uses?.startsWith("actions/checkout@"),
+  )) {
+    assert.equal(step.with?.["persist-credentials"], false);
   }
-  assert.match(workflow, /services:\s*\n\s+postgres:/);
-  assert.match(workflow, /pnpm --filter @pong-pong\/db migrate/);
-  assert.match(workflow, /pnpm --filter @pong-pong\/db seed:dev/);
 });
 
-test("CI keeps registered browser API requests on the login cookie host", () => {
-  assert.match(workflow, /^      API_BASE_URL: http:\/\/localhost:4000$/m);
+test("CI pins the repository Node and pnpm toolchains", () => {
+  const nodeSteps = steps.filter((step) =>
+    step.uses?.startsWith("actions/setup-node@"),
+  );
+  const pnpmSteps = steps.filter((step) =>
+    step.uses?.startsWith("pnpm/action-setup@"),
+  );
+  assert.equal(nodeSteps.length, 4);
+  assert.equal(pnpmSteps.length, 4);
+  for (const step of nodeSteps) {
+    assert.equal(step.with?.["node-version-file"], ".node-version");
+    assert.equal(step.with?.["cache-dependency-path"], "pnpm-lock.yaml");
+  }
+  for (const step of pnpmSteps) {
+    assert.equal(String(step.with?.version), pnpmVersion);
+  }
+  assert.equal(nodeVersion, packageJson.engines.node);
+  assert.equal(runCommands.filter((command) => command === "make install").length, 4);
+});
+
+test("CI separates functional, database, process, browser, and Compose gates", () => {
+  assert.deepEqual(Object.keys(jobs).sort(), [
+    "guest-demo-browser",
+    "postgres-integration",
+    "process-and-browser",
+    "production-compose",
+    "verify",
+  ]);
+  assert.equal(jobs["process-and-browser"].services.postgres.image, "postgres:16-alpine");
+  assert.match(runCommands.join("\n"), /make check/);
+  assert.match(runCommands.join("\n"), /make postgres-integration/);
+  assert.match(runCommands.join("\n"), /make smoke/);
+  assert.match(runCommands.join("\n"), /make e2e\n?/);
+  assert.match(runCommands.join("\n"), /make e2e-guest-demo/);
+  assert.match(makefile, /check:\n\t\$\(MAKE\) typecheck\n\t\$\(MAKE\) unit-functional/);
+  assert.doesNotMatch(runCommands.join("\n"), /documentation|README|devlog/i);
+});
+
+test("CI uses native background process lifecycle and failure reports", () => {
+  const processSteps = jobs["process-and-browser"].steps;
+  const guestSteps = jobs["guest-demo-browser"].steps;
+  for (const [jobSteps, backgroundIds] of [
+    [processSteps, ["api_server", "web_server"]],
+    [guestSteps, ["demo_api", "demo_web"]],
+  ]) {
+    for (const id of backgroundIds) {
+      assert.equal(jobSteps.find((step) => step.id === id)?.background, true);
+      assert.ok(jobSteps.some((step) => step.cancel === id));
+    }
+    assert.ok(
+      jobSteps.some((step) =>
+        step.uses?.startsWith("actions/upload-artifact@"),
+      ),
+    );
+  }
+  assert.equal(jobs["process-and-browser"].env.API_BASE_URL, "http://localhost:4000");
+  assert.equal(jobs["guest-demo-browser"].env.APP_MODE, "demo");
 });
 
-test("CI runs the Guest browser flow against a dedicated demo process", () => {
-  assert.match(workflow, /guest-demo-browser:/);
-  assert.match(workflow, /APP_MODE:\s*demo/);
-  assert.match(workflow, /NEXT_PUBLIC_APP_MODE:\s*demo/);
-  assert.match(workflow, /E2E_APP_MODE:\s*demo/);
-  assert.match(workflow, /pnpm e2e:guest-demo/);
-  assert.match(workflow, /tests\/e2e\/guest-demo\.spec\.ts/);
+test("CI starts and removes the production Compose stack", () => {
+  const composeCommands = jobs["production-compose"].steps
+    .flatMap((step) => (typeof step.run === "string" ? [step.run] : []))
+    .join("\n");
+  assert.match(composeCommands, /docker compose up --build --wait --wait-timeout 600/);
+  assert.match(composeCommands, /\/api\/health\/ready/);
+  assert.match(composeCommands, /\/api\/metrics/);
+  assert.match(composeCommands, /docker compose down --volumes --remove-orphans/);
+  assert.equal(jobs["production-compose"]["timeout-minutes"], 30);
 });
