## `build(perf): Lighthouse 결과 요약기 추가`

diff --git a/package.json b/package.json
index 730df6f..5998688 100644
--- a/package.json
+++ b/package.json
@@ -15,6 +15,7 @@
     "bundle:baseline": "node scripts/route-budgets.mjs --write-baseline",
     "bundle:check": "node scripts/route-budgets.mjs",
     "lighthouse:audit": "lhci autorun",
+    "lighthouse:summarize": "node scripts/summarize-lighthouse.mjs",
     "start": "next start -p 3100",
     "start:e2e": "next start -p 3200",
     "start:performance": "next start -p 3300",
diff --git a/scripts/summarize-lighthouse.mjs b/scripts/summarize-lighthouse.mjs
new file mode 100644
index 0000000..9870aae
--- /dev/null
+++ b/scripts/summarize-lighthouse.mjs
@@ -0,0 +1,104 @@
+import { mkdir, readdir, readFile, writeFile } from "node:fs/promises";
+import os from "node:os";
+import path from "node:path";
+
+const INPUT_DIRECTORY = ".lighthouseci";
+const OUTPUT_PATH = "performance/lighthouse-baseline.json";
+
+function median(values) {
+  const sorted = [...values].sort((left, right) => left - right);
+  const middle = Math.floor(sorted.length / 2);
+
+  return sorted.length % 2 === 0
+    ? (sorted[middle - 1] + sorted[middle]) / 2
+    : sorted[middle];
+}
+
+function resultFromReport(report) {
+  return {
+    accessibilityScore: report.categories.accessibility.score * 100,
+    cls: report.audits["cumulative-layout-shift"].numericValue,
+    lcpMs: report.audits["largest-contentful-paint"].numericValue,
+    performanceScore: report.categories.performance.score * 100,
+    tbtMs: report.audits["total-blocking-time"].numericValue,
+  };
+}
+
+function aggregate(results) {
+  return {
+    accessibilityScore: median(
+      results.map(({ accessibilityScore }) => accessibilityScore),
+    ),
+    cls: median(results.map(({ cls }) => cls)),
+    lcpMs: median(results.map(({ lcpMs }) => lcpMs)),
+    performanceScore: median(
+      results.map(({ performanceScore }) => performanceScore),
+    ),
+    tbtMs: median(results.map(({ tbtMs }) => tbtMs)),
+  };
+}
+
+async function main() {
+  const filenames = (await readdir(INPUT_DIRECTORY)).filter(
+    (filename) => filename.startsWith("lhr-") && filename.endsWith(".json"),
+  );
+  if (filenames.length === 0) {
+    throw new Error("No Lighthouse JSON reports were found in .lighthouseci.");
+  }
+
+  const reports = await Promise.all(
+    filenames.map(async (filename) =>
+      JSON.parse(
+        await readFile(path.join(INPUT_DIRECTORY, filename), "utf8"),
+      ),
+    ),
+  );
+  const grouped = new Map();
+
+  for (const report of reports) {
+    const url = report.finalUrl;
+    const results = grouped.get(url) ?? [];
+    results.push(resultFromReport(report));
+    grouped.set(url, results);
+  }
+
+  const routes = Object.fromEntries(
+    [...grouped.entries()]
+      .sort(([left], [right]) => left.localeCompare(right))
+      .map(([url, runs]) => [url, { median: aggregate(runs), runs }]),
+  );
+  const firstReport = reports[0];
+  const summary = {
+    schemaVersion: 1,
+    measuredAt: new Date().toISOString(),
+    measurement: {
+      kind: "Lighthouse desktop lab run against the local production server",
+      runCountPerUrl: 3,
+      aggregation: "median",
+      interactionProxy: "total-blocking-time",
+    },
+    targets: {
+      accessibilityScore: 95,
+      cls: 0.1,
+      lcpMs: 2_500,
+      performanceScore: 90,
+      tbtMs: 200,
+    },
+    environment: {
+      arch: os.arch(),
+      chromeUserAgent: firstReport.environment.hostUserAgent,
+      cpu: os.cpus()[0]?.model ?? "unknown",
+      logicalCpuCount: os.cpus().length,
+      memoryBytes: os.totalmem(),
+      node: process.version,
+      platform: os.platform(),
+    },
+    routes,
+  };
+
+  await mkdir(path.dirname(OUTPUT_PATH), { recursive: true });
+  await writeFile(OUTPUT_PATH, `${JSON.stringify(summary, null, 2)}\n`, "utf8");
+  console.log(`Wrote ${OUTPUT_PATH} from ${reports.length} Lighthouse runs.`);
+}
+
+await main();


## `test(perf): 배포 성능 gate 규칙 검증`

diff --git a/src/performance/build-manifest-contract.test.ts b/src/performance/build-manifest-contract.test.ts
deleted file mode 100644
index 03eef34..0000000
--- a/src/performance/build-manifest-contract.test.ts
+++ /dev/null
@@ -1,36 +0,0 @@
-import { createRequire } from "node:module";
-
-import { describe, expect, it } from "vitest";
-
-import { parseClientReferenceManifest } from "../../scripts/route-budgets.mjs";
-
-const require = createRequire(import.meta.url);
-const packageJson = require("../../package.json");
-
-describe("production build pipeline", () => {
-  it("uses the verified webpack compiler path", () => {
-    expect(packageJson.scripts.build).toBe("next build --webpack");
-  });
-});
-
-describe("webpack client reference manifest parser", () => {
-  it("parses the compact webpack client reference manifest", () => {
-    const source =
-      'globalThis.__RSC_MANIFEST=(globalThis.__RSC_MANIFEST||{});globalThis.__RSC_MANIFEST["/about/page"]={"entryJSFiles":{"route":["static/chunks/page.js"]}};';
-
-    expect(parseClientReferenceManifest(source, "about.js")).toEqual({
-      entryJSFiles: {
-        route: ["static/chunks/page.js"],
-      },
-    });
-  });
-
-  it("parses a dynamic route key containing square brackets", () => {
-    const source =
-      'globalThis.__RSC_MANIFEST=(globalThis.__RSC_MANIFEST||{});globalThis.__RSC_MANIFEST["/projects/[projectId]/page"]={"entryJSFiles":{}};';
-
-    expect(parseClientReferenceManifest(source, "project.js")).toEqual({
-      entryJSFiles: {},
-    });
-  });
-});
diff --git a/src/performance/performance-gates.test.ts b/src/performance/performance-gates.test.ts
new file mode 100644
index 0000000..5c1650b
--- /dev/null
+++ b/src/performance/performance-gates.test.ts
@@ -0,0 +1,137 @@
+import { createRequire } from "node:module";
+
+import { describe, expect, it } from "vitest";
+
+import {
+  BUDGET_GROWTH_FACTOR,
+  evaluateRouteBudgets,
+  parseClientReferenceManifest,
+  type RouteBudgetBaseline,
+  type RouteBundleMeasurement,
+} from "../../scripts/route-budgets.mjs";
+
+const require = createRequire(import.meta.url);
+const lighthouseConfig = require("../../lighthouserc.cjs");
+const packageJson = require("../../package.json");
+
+describe("production build pipeline", () => {
+  it("uses the verified webpack compiler path", () => {
+    expect(packageJson.scripts.build).toBe("next build --webpack");
+  });
+});
+
+describe("production performance gates", () => {
+  it("runs three production measurements for every visual design", () => {
+    const collect = lighthouseConfig.ci.collect;
+    const urls = collect.url as string[];
+
+    expect(collect.startServerCommand).toBe("npm run start:performance");
+    expect(collect.numberOfRuns).toBe(3);
+    expect(collect.settings.preset).toBe("desktop");
+    expect(urls).toHaveLength(10);
+
+    for (const designId of [
+      "design",
+      "classic",
+      "editorial",
+      "brutalist",
+      "cinematic",
+    ]) {
+      expect(urls).toContain(`http://localhost:3300/?view=${designId}`);
+      expect(urls).toContain(
+        `http://localhost:3300/projects/example-project?view=${designId}`,
+      );
+    }
+  });
+
+  it("enforces the agreed Lighthouse and lab responsiveness targets", () => {
+    const assertions = lighthouseConfig.ci.assert.assertions;
+
+    expect(assertions["categories:performance"]).toEqual([
+      "error",
+      expect.objectContaining({ minScore: 0.9 }),
+    ]);
+    expect(assertions["categories:accessibility"]).toEqual([
+      "error",
+      expect.objectContaining({ minScore: 0.95 }),
+    ]);
+    expect(assertions["largest-contentful-paint"]).toEqual([
+      "error",
+      expect.objectContaining({ maxNumericValue: 2_500 }),
+    ]);
+    expect(assertions["cumulative-layout-shift"]).toEqual([
+      "error",
+      expect.objectContaining({ maxNumericValue: 0.1 }),
+    ]);
+    expect(assertions["total-blocking-time"]).toEqual([
+      "error",
+      expect.objectContaining({ maxNumericValue: 200 }),
+    ]);
+  });
+});
+
+describe("route bundle budgets", () => {
+  it("parses the compact webpack client reference manifest", () => {
+    const source =
+      'globalThis.__RSC_MANIFEST=(globalThis.__RSC_MANIFEST||{});globalThis.__RSC_MANIFEST["/about/page"]={"entryJSFiles":{"route":["static/chunks/page.js"]}};';
+
+    expect(parseClientReferenceManifest(source, "about.js")).toEqual({
+      entryJSFiles: {
+        route: ["static/chunks/page.js"],
+      },
+    });
+  });
+
+  it("parses a dynamic route key containing square brackets", () => {
+    const source =
+      'globalThis.__RSC_MANIFEST=(globalThis.__RSC_MANIFEST||{});globalThis.__RSC_MANIFEST["/projects/[projectId]/page"]={"entryJSFiles":{}};';
+
+    expect(parseClientReferenceManifest(source, "project.js")).toEqual({
+      entryJSFiles: {},
+    });
+  });
+
+  const baseline: RouteBudgetBaseline = {
+    schemaVersion: 1,
+    growthLimitPercent: 5,
+    routes: {
+      "/": { cssBytes: 100, jsBytes: 1_000 },
+      "/projects/[projectId]": { cssBytes: 200, jsBytes: 2_000 },
+    },
+  };
+
+  it("allows at most five percent growth per route and asset type", () => {
+    const measurements: RouteBundleMeasurement = {
+      "/": { cssBytes: 105, jsBytes: 1_050 },
+      "/projects/[projectId]": { cssBytes: 210, jsBytes: 2_100 },
+    };
+
+    expect(BUDGET_GROWTH_FACTOR).toBe(1.05);
+    expect(evaluateRouteBudgets(measurements, baseline)).toEqual([]);
+  });
+
+  it("reports a route and asset when the measured output exceeds its budget", () => {
+    const measurements: RouteBundleMeasurement = {
+      "/": { cssBytes: 106, jsBytes: 1_050 },
+      "/projects/[projectId]": { cssBytes: 210, jsBytes: 2_101 },
+    };
+
+    expect(evaluateRouteBudgets(measurements, baseline)).toEqual([
+      expect.objectContaining({ asset: "css", route: "/" }),
+      expect.objectContaining({
+        asset: "js",
+        route: "/projects/[projectId]",
+      }),
+    ]);
+  });
+
+  it("fails closed when a baseline route is missing from the build", () => {
+    expect(evaluateRouteBudgets({}, baseline)).toEqual([
+      expect.objectContaining({ asset: "route", route: "/" }),
+      expect.objectContaining({
+        asset: "route",
+        route: "/projects/[projectId]",
+      }),
+    ]);
+  });
+});


