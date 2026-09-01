## `build(perf): bundle budget CLI 연결`

diff --git a/package.json b/package.json
index 3a8f502..e8db7ff 100644
--- a/package.json
+++ b/package.json
@@ -12,6 +12,8 @@
     "prebuild": "npm run content:check && npm run content:ready",
     "build": "next build --webpack",
     "build:verify": "node scripts/verify-build-output.mjs",
+    "bundle:baseline": "node scripts/route-budgets.mjs --write-baseline",
+    "bundle:check": "node scripts/route-budgets.mjs",
     "start": "next start -p 3100",
     "start:e2e": "next start -p 3200",
     "lint": "eslint .",
diff --git a/scripts/route-budgets.mjs b/scripts/route-budgets.mjs
index 573dabe..2fe7f9c 100644
--- a/scripts/route-budgets.mjs
+++ b/scripts/route-budgets.mjs
@@ -1,9 +1,12 @@
-import { readFile, stat } from "node:fs/promises";
+import { mkdir, readFile, stat, writeFile } from "node:fs/promises";
 import path from "node:path";
+import process from "node:process";
+import { fileURLToPath } from "node:url";
 
 export const BUDGET_GROWTH_FACTOR = 1.05;
 
 const DEFAULT_BUILD_DIRECTORY = ".next";
+const DEFAULT_BASELINE_PATH = "performance/route-budgets.json";
 
 function routeFromManifestKey(key) {
   if (key === "/page") {
@@ -132,3 +135,61 @@ export function evaluateRouteBudgets(measurements, baseline) {
 
   return violations;
 }
+
+function printMeasurements(measurements) {
+  for (const [route, sizes] of Object.entries(measurements)) {
+    console.log(
+      `${route}: js=${sizes.jsBytes} bytes, css=${sizes.cssBytes} bytes`,
+    );
+  }
+}
+
+async function main() {
+  const writeBaseline = process.argv.includes("--write-baseline");
+  const measurements = await collectRouteBundleMeasurements();
+
+  printMeasurements(measurements);
+
+  if (writeBaseline) {
+    const baseline = {
+      schemaVersion: 1,
+      growthLimitPercent: 5,
+      source: "Next.js production client assets (uncompressed bytes)",
+      routes: measurements,
+    };
+    await mkdir(path.dirname(DEFAULT_BASELINE_PATH), { recursive: true });
+    await writeFile(
+      DEFAULT_BASELINE_PATH,
+      `${JSON.stringify(baseline, null, 2)}\n`,
+      "utf8",
+    );
+    console.log(`Wrote ${DEFAULT_BASELINE_PATH}`);
+    return;
+  }
+
+  const baseline = JSON.parse(
+    await readFile(DEFAULT_BASELINE_PATH, "utf8"),
+  );
+  if (baseline.growthLimitPercent !== 5) {
+    throw new Error("The committed route budget must use a five percent limit.");
+  }
+
+  const violations = evaluateRouteBudgets(measurements, baseline);
+  if (violations.length > 0) {
+    for (const violation of violations) {
+      console.error(violation.message);
+    }
+    process.exitCode = 1;
+    return;
+  }
+
+  console.log("All route JS/CSS bundles are within the five percent budget.");
+}
+
+const isMain = process.argv[1]
+  ? path.resolve(process.argv[1]) === fileURLToPath(import.meta.url)
+  : false;
+
+if (isMain) {
+  await main();
+}


## `chore(perf): route bundle 기준값 기록`

diff --git a/performance/route-budgets.json b/performance/route-budgets.json
new file mode 100644
index 0000000..c8a851e
--- /dev/null
+++ b/performance/route-budgets.json
@@ -0,0 +1,39 @@
+{
+  "schemaVersion": 1,
+  "growthLimitPercent": 5,
+  "source": "Next.js production client assets (uncompressed bytes)",
+  "routes": {
+    "/about": {
+      "cssBytes": 169861,
+      "jsBytes": 425976
+    },
+    "/contact": {
+      "cssBytes": 169861,
+      "jsBytes": 425976
+    },
+    "/interview-map": {
+      "cssBytes": 169861,
+      "jsBytes": 425976
+    },
+    "/journey": {
+      "cssBytes": 169861,
+      "jsBytes": 425976
+    },
+    "/": {
+      "cssBytes": 169861,
+      "jsBytes": 425976
+    },
+    "/projects/[projectId]": {
+      "cssBytes": 169861,
+      "jsBytes": 425976
+    },
+    "/projects": {
+      "cssBytes": 169861,
+      "jsBytes": 425976
+    },
+    "/resume": {
+      "cssBytes": 169861,
+      "jsBytes": 425976
+    }
+  }
+}


