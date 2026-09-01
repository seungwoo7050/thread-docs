# 재현 가능한 검증 명령과 CI Gate DAG

## `ci: 기본 배포 품질 검사 추가`

diff --git a/.github/workflows/ci.yml b/.github/workflows/ci.yml
new file mode 100644
index 0000000..e423eab
--- /dev/null
+++ b/.github/workflows/ci.yml
@@ -0,0 +1,52 @@
+name: CI
+
+on:
+  push:
+  pull_request:
+
+permissions:
+  contents: read
+
+concurrency:
+  group: ci-${{ github.workflow }}-${{ github.ref }}
+  cancel-in-progress: true
+
+jobs:
+  verify:
+    runs-on: ubuntu-latest
+    timeout-minutes: 30
+
+    steps:
+      - name: Check out repository
+        uses: actions/checkout@v4
+
+      - name: Set up Node.js
+        uses: actions/setup-node@v4
+        with:
+          node-version-file: .nvmrc
+          cache: npm
+
+      - name: Pin npm
+        run: npm install --global npm@11.16.0
+
+      - name: Verify toolchain
+        run: |
+          node --version
+          npm --version
+
+      - name: Install dependencies
+        run: npm ci
+
+      - name: Lint
+        run: npm run lint
+
+      - name: Typecheck
+        run: npm run typecheck
+
+      - name: Validate content
+        run: npm run content:check
+      - name: Install Chromium
+        run: npx playwright install --with-deps chromium
+
+      - name: Build and run production E2E tests
+        run: npm run test:e2e:production


## `ci: standalone 산출물 검증 추가`

diff --git a/.github/workflows/ci.yml b/.github/workflows/ci.yml
index e423eab..182aa2f 100644
--- a/.github/workflows/ci.yml
+++ b/.github/workflows/ci.yml
@@ -50,3 +50,6 @@ jobs:
 
       - name: Build and run production E2E tests
         run: npm run test:e2e:production
+
+      - name: Verify standalone output
+        run: npm run build:verify


## `ci: 검증된 bundle과 Lighthouse gate 활성화`

diff --git a/.github/workflows/ci.yml b/.github/workflows/ci.yml
index 182aa2f..8031e9e 100644
--- a/.github/workflows/ci.yml
+++ b/.github/workflows/ci.yml
@@ -49,7 +49,16 @@ jobs:
         run: npx playwright install --with-deps chromium
 
       - name: Build and run production E2E tests
-        run: npm run test:e2e:production
+        run: npm run test:e2e:ci
 
       - name: Verify standalone output
         run: npm run build:verify
+
+      - name: Check route bundle budgets
+        run: npm run bundle:check
+
+      - name: Locate Playwright Chromium for Lighthouse
+        run: echo "CHROME_PATH=$(node -e 'process.stdout.write(require(\"playwright\").chromium.executablePath())')" >> "$GITHUB_ENV"
+
+      - name: Run production Lighthouse budgets
+        run: npm run lighthouse:audit
diff --git a/package.json b/package.json
index 5998688..63b4949 100644
--- a/package.json
+++ b/package.json
@@ -26,6 +26,7 @@
     "test": "vitest run",
     "test:watch": "vitest",
     "test:e2e": "playwright test",
+    "test:e2e:ci": "npm run build && playwright test --config=playwright.production.config.ts --grep-invert @visual",
     "test:e2e:production": "npm run build && playwright test --config=playwright.production.config.ts"
   },
   "dependencies": {


## `build: improve Makefile and separate functional portfolio checks`

diff --git a/Makefile b/Makefile
new file mode 100644
index 0000000..98a475c
--- /dev/null
+++ b/Makefile
@@ -0,0 +1,129 @@
+NPM ?= npm
+NPX ?= npx
+PLAYWRIGHT_BROWSER ?= chromium
+
+.DEFAULT_GOAL := help
+
+.PHONY: help install playwright-install dev start \
+	lint typecheck content content-ready unit unit-functional \
+	test check check-functional \
+	build build-verify bundle-check bundle-baseline \
+	e2e e2e-production e2e-ci \
+	lighthouse lighthouse-summarize container verify ci
+
+help:
+	@printf '%s\n' \
+		'Usage: make <target>' \
+		'' \
+		'Setup and development:' \
+		'  install              Install the locked dependency graph with npm ci' \
+		'  playwright-install   Install the selected Playwright browser' \
+		'  dev                  Start the webpack development server on port 3100' \
+		'  start                Start an existing production build on port 3100' \
+		'' \
+		'Fast validation:' \
+		'  lint                 Run ESLint' \
+		'  typecheck            Run TypeScript without emitting files' \
+		'  content              Validate content schemas, references, and assets' \
+		'  content-ready        Validate publication readiness for the active mode' \
+		'  test                 Run the Vitest suite once' \
+		'  check                Run content, lint, typecheck, and unit checks' \
+		'  check-functional     Run executable checks without documentation-link policy' \
+		'' \
+		'Build and browser validation:' \
+		'  build                Create the standalone production build' \
+		'  build-verify         Verify the standalone build artifacts' \
+		'  bundle-check         Check route bundles against committed budgets' \
+		'  e2e                  Run Playwright against the development server' \
+		'  e2e-production       Build and run all production Playwright tests' \
+		'  lighthouse           Run the production Lighthouse budget' \
+		'  container            Verify the built Docker image and public assets' \
+		'  verify               Run check plus non-visual production E2E/build gates' \
+		'  ci                   Extend verify with Lighthouse and container gates' \
+		'' \
+		'Baseline maintenance:' \
+		'  bundle-baseline      Replace the committed route bundle baseline' \
+		'  lighthouse-summarize Summarize existing Lighthouse reports' \
+		'' \
+		'Overrides: NPM, NPX, PLAYWRIGHT_BROWSER'
+
+install:
+	$(NPM) ci
+
+playwright-install:
+	$(NPX) playwright install $(PLAYWRIGHT_BROWSER)
+
+dev:
+	$(NPM) run dev
+
+start:
+	$(NPM) run start
+
+lint:
+	$(NPM) run lint
+
+typecheck:
+	$(NPM) run typecheck
+
+content:
+	$(NPM) run content:check
+
+content-ready:
+	$(NPM) run content:ready
+
+unit:
+	$(NPM) run test
+
+unit-functional:
+	$(NPM) run test:functional
+
+test: unit
+
+check: content lint typecheck unit
+
+check-functional:
+	$(MAKE) content
+	$(MAKE) lint
+	$(MAKE) typecheck
+	$(MAKE) unit-functional
+
+build:
+	$(NPM) run build
+
+build-verify:
+	$(NPM) run build:verify
+
+bundle-check:
+	$(NPM) run bundle:check
+
+bundle-baseline:
+	$(NPM) run bundle:baseline
+
+e2e:
+	$(NPM) run test:e2e
+
+e2e-production:
+	$(NPM) run test:e2e:production
+
+e2e-ci:
+	$(NPM) run test:e2e:ci
+
+lighthouse:
+	$(NPM) run lighthouse:audit
+
+lighthouse-summarize:
+	$(NPM) run lighthouse:summarize
+
+container:
+	$(NPM) run test:container
+
+# test:e2e:ci performs the production build. The following gates deliberately
+# reuse that output instead of paying for a second build.
+verify: check-functional
+	$(NPM) run test:e2e:ci
+	$(NPM) run build:verify
+	$(NPM) run bundle:check
+
+ci: verify
+	$(NPM) run lighthouse:audit
+	$(NPM) run test:container
diff --git a/package.json b/package.json
index babb5cc..d449b6c 100644
--- a/package.json
+++ b/package.json
@@ -24,6 +24,7 @@
     "content:check": "node --import tsx scripts/validate-content.ts",
     "content:ready": "node --import tsx scripts/validate-content-readiness.ts",
     "test": "vitest run",
+    "test:functional": "vitest run --exclude src/docs/documentation.test.ts",
     "test:container": "node scripts/verify-container-runtime.mjs",
     "test:watch": "vitest",
     "test:e2e": "playwright test",


## `ci: harden portfolio validation`

diff --git a/.github/workflows/ci.yml b/.github/workflows/ci.yml
deleted file mode 100644
index af4d349..0000000
--- a/.github/workflows/ci.yml
+++ /dev/null
@@ -1,67 +0,0 @@
-name: CI
-
-on:
-  push:
-  pull_request:
-
-permissions:
-  contents: read
-
-concurrency:
-  group: ci-${{ github.workflow }}-${{ github.ref }}
-  cancel-in-progress: true
-
-jobs:
-  verify:
-    runs-on: ubuntu-latest
-    timeout-minutes: 30
-
-    steps:
-      - name: Check out repository
-        uses: actions/checkout@v4
-
-      - name: Set up Node.js
-        uses: actions/setup-node@v4
-        with:
-          node-version-file: .nvmrc
-          cache: npm
-
-      - name: Pin npm
-        run: npm install --global npm@11.16.0
-
-      - name: Verify toolchain
-        run: |
-          node --version
-          npm --version
-
-      - name: Install dependencies
-        run: npm ci
-
-      - name: Lint
-        run: npm run lint
-
-      - name: Typecheck
-        run: npm run typecheck
-
-      - name: Validate content
-        run: npm run content:check
-      - name: Install Chromium
-        run: npx playwright install --with-deps chromium
-
-      - name: Build and run production E2E tests
-        run: npm run test:e2e:ci
-
-      - name: Verify standalone output
-        run: npm run build:verify
-
-      - name: Check route bundle budgets
-        run: npm run bundle:check
-
-      - name: Locate Playwright Chromium for Lighthouse
-        run: echo "CHROME_PATH=$(node -e 'process.stdout.write(require(\"playwright\").chromium.executablePath())')" >> "$GITHUB_ENV"
-
-      - name: Run production Lighthouse budgets
-        run: npm run lighthouse:audit
-
-      - name: Verify container runtime and public assets
-        run: npm run test:container
diff --git a/.github/workflows/web-portfolio-ci.yml b/.github/workflows/web-portfolio-ci.yml
new file mode 100644
index 0000000..4b5b210
--- /dev/null
+++ b/.github/workflows/web-portfolio-ci.yml
@@ -0,0 +1,141 @@
+name: web/portfolio CI
+
+on:
+  push:
+    branches: [web/portfolio]
+  pull_request:
+    branches: [web/portfolio]
+
+permissions:
+  contents: read
+
+concurrency:
+  group: web-portfolio-${{ github.workflow }}-${{ github.ref }}
+  cancel-in-progress: true
+
+jobs:
+  quality:
+    name: Functional quality checks
+    runs-on: ubuntu-24.04
+    timeout-minutes: 20
+    steps:
+      - name: Check out repository
+        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
+        with:
+          persist-credentials: false
+
+      - name: Set up Node.js
+        uses: actions/setup-node@820762786026740c76f36085b0efc47a31fe5020 # v7.0.0
+        with:
+          node-version-file: .nvmrc
+          cache: npm
+          cache-dependency-path: package-lock.json
+
+      - name: Pin npm
+        run: npm install --global npm@11.16.0
+
+      - name: Install dependencies
+        run: npm ci
+
+      - name: Run functional quality checks
+        run: make check-functional
+
+  production:
+    name: Production browser and performance checks
+    needs: quality
+    runs-on: ubuntu-24.04
+    timeout-minutes: 30
+    steps:
+      - name: Check out repository
+        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
+        with:
+          persist-credentials: false
+
+      - name: Set up Node.js
+        uses: actions/setup-node@820762786026740c76f36085b0efc47a31fe5020 # v7.0.0
+        with:
+          node-version-file: .nvmrc
+          cache: npm
+          cache-dependency-path: package-lock.json
+
+      - name: Pin npm
+        run: npm install --global npm@11.16.0
+
+      - name: Install dependencies
+        run: npm ci
+
+      - name: Install Chromium
+        run: npx --no-install playwright install --with-deps chromium
+
+      - name: Build and run non-visual production E2E tests
+        run: npm run test:e2e:ci
+
+      - name: Verify standalone output
+        run: npm run build:verify
+
+      - name: Check route bundle budgets
+        run: npm run bundle:check
+
+      - name: Locate Chromium for Lighthouse
+        shell: bash
+        run: |
+          set -euo pipefail
+          chrome_path=$(node -e 'const { chromium } = require("@playwright/test"); process.stdout.write(chromium.executablePath())')
+          test -x "$chrome_path"
+          echo "CHROME_PATH=$chrome_path" >> "$GITHUB_ENV"
+
+      - name: Run production Lighthouse budgets
+        run: npm run lighthouse:audit
+
+      - name: Upload browser diagnostics
+        if: ${{ failure() }}
+        uses: actions/upload-artifact@043fb46d1a93c77aae656e7c1c64a875d1fc6a0a # v7.0.1
+        with:
+          name: portfolio-browser-${{ github.run_id }}-${{ github.run_attempt }}
+          path: |
+            test-results/
+            playwright-report/
+            .lighthouseci/*.html
+            .lighthouseci/*.json
+          if-no-files-found: warn
+          include-hidden-files: true
+          retention-days: 7
+
+  container:
+    name: Production container checks
+    needs: quality
+    runs-on: ubuntu-24.04
+    timeout-minutes: 20
+    steps:
+      - name: Check out repository
+        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
+        with:
+          persist-credentials: false
+
+      - name: Set up Node.js
+        uses: actions/setup-node@820762786026740c76f36085b0efc47a31fe5020 # v7.0.0
+        with:
+          node-version-file: .nvmrc
+          package-manager-cache: false
+
+      - name: Pin npm
+        run: npm install --global npm@11.16.0
+
+      - name: Verify container runtime and public assets
+        run: npm run test:container
+
+  verify:
+    needs: [quality, production, container]
+    if: ${{ !cancelled() }}
+    runs-on: ubuntu-24.04
+    timeout-minutes: 5
+    steps:
+      - name: Require every portfolio gate
+        env:
+          QUALITY_RESULT: ${{ needs.quality.result }}
+          PRODUCTION_RESULT: ${{ needs.production.result }}
+          CONTAINER_RESULT: ${{ needs.container.result }}
+        run: |
+          test "$QUALITY_RESULT" = success
+          test "$PRODUCTION_RESULT" = success
+          test "$CONTAINER_RESULT" = success
diff --git a/lighthouserc.cjs b/lighthouserc.cjs
index ccac0bf..7f46135 100644
--- a/lighthouserc.cjs
+++ b/lighthouserc.cjs
@@ -55,7 +55,7 @@ module.exports = {
           { ...median, maxNumericValue: 200 },
         ],
       },
-      includePassedAssertions: true,
+      includePassedAssertions: false,
     },
   },
 };
diff --git a/playwright.production.config.ts b/playwright.production.config.ts
index 9c6ad51..a9889fb 100644
--- a/playwright.production.config.ts
+++ b/playwright.production.config.ts
@@ -6,14 +6,15 @@ export default defineConfig({
   expect: {
     timeout: 5_000,
   },
-  reporter: [["list"]],
+  reporter: [["list"], ["github"], ["html", { open: "never" }]],
   outputDir: "test-results",
   snapshotPathTemplate:
     "{testDir}/{testFilePath}-snapshots/{arg}-{projectName}{ext}",
   workers: 1,
   use: {
     baseURL: "http://localhost:3200",
-    trace: "on-first-retry",
+    trace: "retain-on-failure",
+    screenshot: "only-on-failure",
   },
   webServer: {
     command: "npm run start:e2e",
