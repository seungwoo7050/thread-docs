## `test(perf): 유휴 route 요청과 글꼴 경계 검증`

diff --git a/tests/e2e/performance.spec.ts b/tests/e2e/performance.spec.ts
new file mode 100644
index 0000000..5f8ba28
--- /dev/null
+++ b/tests/e2e/performance.spec.ts
@@ -0,0 +1,123 @@
+import { expect, test, type Request } from "@playwright/test";
+
+import {
+  designIds,
+  firstEnabledProject,
+  withExplicitDesign,
+} from "./site-matrix";
+
+const firstProjectId = firstEnabledProject?.id;
+
+if (!firstProjectId) {
+  throw new Error("The performance check needs an enabled project.");
+}
+
+const initialRoutes = [
+  { label: "home", path: "/" },
+  {
+    label: "project detail",
+    path: `/projects/${firstProjectId}`,
+  },
+] as const;
+
+for (const designId of designIds) {
+  for (const route of initialRoutes) {
+    test(`${designId} ${route.label} does not prefetch before user interaction`, async ({
+      page,
+    }) => {
+      const prefetchedRoutes: string[] = [];
+      const loadedFonts: string[] = [];
+
+      const recordPrefetch = (request: Request) => {
+        const url = new URL(request.url());
+
+        if (url.searchParams.has("_rsc")) {
+          prefetchedRoutes.push(`${url.pathname}${url.search}`);
+        }
+        if (request.resourceType() === "font") {
+          loadedFonts.push(url.pathname);
+        }
+      };
+
+      page.on("request", recordPrefetch);
+      const response = await page.goto(
+        withExplicitDesign(route.path, designId),
+      );
+
+      expect(response?.ok()).toBe(true);
+      await expect(page.getByRole("heading", { level: 1 }).first()).toBeVisible();
+      await page.waitForTimeout(1_000);
+      page.off("request", recordPrefetch);
+
+      const monoUsers = await page.locator("body *").evaluateAll((elements) =>
+        elements
+          .map((element) => ({
+            className:
+              element instanceof HTMLElement ? element.className : undefined,
+            fontFamily: getComputedStyle(element).fontFamily,
+            tagName: element.tagName,
+          }))
+          .filter(({ fontFamily }) => /geist.?mono/i.test(fontFamily))
+          .slice(0, 20),
+      );
+
+      const { monoFontPaths, preloadedFontPaths } = await page.evaluate(() => {
+        const monoSources = new Set<string>();
+
+        for (const stylesheet of Array.from(document.styleSheets)) {
+          let rules: CSSRuleList;
+
+          try {
+            rules = stylesheet.cssRules;
+          } catch {
+            continue;
+          }
+
+          for (const rule of Array.from(rules)) {
+            if (
+              !(rule instanceof CSSFontFaceRule) ||
+              !/geist.?mono/i.test(rule.style.getPropertyValue("font-family"))
+            ) {
+              continue;
+            }
+
+            const source = rule.style.getPropertyValue("src");
+            const baseUrl = stylesheet.href ?? document.baseURI;
+            const sourcePattern = /url\(\s*["']?([^"')]+)["']?\s*\)/g;
+
+            for (const match of source.matchAll(sourcePattern)) {
+              const sourceUrl = match[1];
+
+              if (sourceUrl) {
+                monoSources.add(new URL(sourceUrl, baseUrl).pathname);
+              }
+            }
+          }
+        }
+
+        return {
+          monoFontPaths: Array.from(monoSources),
+          preloadedFontPaths: Array.from(
+            document.querySelectorAll<HTMLLinkElement>(
+              'link[rel~="preload"][as="font"]',
+            ),
+            (link) => new URL(link.href, document.baseURI).pathname,
+          ),
+        };
+      });
+
+      expect(prefetchedRoutes, designId).toEqual([]);
+      expect(monoFontPaths, "Geist Mono @font-face source").not.toEqual([]);
+      expect(
+        preloadedFontPaths.filter((path) => monoFontPaths.includes(path)),
+        `${designId}: Geist Mono preload`,
+      ).toEqual([]);
+      if (designId === "design" || designId === "editorial") {
+        expect(
+          loadedFonts.filter((path) => monoFontPaths.includes(path)),
+          `${designId}: ${JSON.stringify(monoUsers)}`,
+        ).toEqual([]);
+      }
+    });
+  }
+}


## `test(perf): 사용자 상호작용 지연 측정 추가`

diff --git a/tests/e2e/interaction-performance.spec.ts b/tests/e2e/interaction-performance.spec.ts
new file mode 100644
index 0000000..4c50094
--- /dev/null
+++ b/tests/e2e/interaction-performance.spec.ts
@@ -0,0 +1,328 @@
+import { expect, test, type Page } from "@playwright/test";
+
+import presentationJson from "../../src/content/presentation.json";
+import { designIds, withExplicitDesign, type DesignId } from "./site-matrix";
+
+const EVENT_TIMING_THRESHOLD_MS = 16;
+const INTERACTION_TARGET_MS = 200;
+const SAMPLE_COUNT = 3;
+const DESIGN_SWITCHER_LABEL_PATTERN = new RegExp(
+  `^${presentationJson.ui.designSwitcherAriaTemplate
+    .split("{label}")
+    .map((part) => part.replace(/[.*+?^${}()|[\]\\]/g, "\\$&"))
+    .join(".+")}$`,
+);
+
+type EventTimingRecord = {
+  duration: number;
+  interactionId: number;
+  name: string;
+  startTime: number;
+};
+
+type InteractionProbe = {
+  entries: EventTimingRecord[];
+  observer: PerformanceObserver;
+  sampleStartedAt: number;
+  trustedClickCount: number;
+};
+
+type InteractionSample = {
+  durationUpperBoundMs: number;
+  reportedDuration: string;
+};
+
+declare global {
+  interface Window {
+    __portfolioInteractionProbe?: InteractionProbe;
+  }
+}
+
+async function settleNextPaint(page: Page) {
+  await page.evaluate(
+    () =>
+      new Promise<void>((resolve) => {
+        requestAnimationFrame(() => {
+          requestAnimationFrame(() => {
+            setTimeout(resolve, 0);
+          });
+        });
+      }),
+  );
+}
+
+async function installInteractionProbe(page: Page) {
+  const supported = await page.evaluate((durationThreshold) => {
+    if (!PerformanceObserver.supportedEntryTypes.includes("event")) {
+      return false;
+    }
+
+    const probe: InteractionProbe = {
+      entries: [],
+      observer: undefined as unknown as PerformanceObserver,
+      sampleStartedAt: performance.now(),
+      trustedClickCount: 0,
+    };
+    const observer = new PerformanceObserver((entryList) => {
+      for (const entry of entryList.getEntries()) {
+        const eventEntry = entry as PerformanceEntry & {
+          interactionId?: number;
+        };
+
+        probe.entries.push({
+          duration: eventEntry.duration,
+          interactionId: eventEntry.interactionId ?? 0,
+          name: eventEntry.name,
+          startTime: eventEntry.startTime,
+        });
+      }
+    });
+
+    observer.observe({
+      buffered: true,
+      durationThreshold,
+      type: "event",
+    } as PerformanceObserverInit & { durationThreshold: number });
+    probe.observer = observer;
+    window.__portfolioInteractionProbe = probe;
+
+    document.addEventListener(
+      "click",
+      (event) => {
+        if (event.isTrusted) {
+          const currentProbe = window.__portfolioInteractionProbe;
+
+          if (currentProbe) {
+            currentProbe.trustedClickCount += 1;
+          }
+        }
+      },
+      { capture: true },
+    );
+
+    return true;
+  }, EVENT_TIMING_THRESHOLD_MS);
+
+  expect(
+    supported,
+    "Chromium must expose PerformanceObserver Event Timing entries.",
+  ).toBe(true);
+}
+
+async function resetInteractionProbe(page: Page) {
+  await page.evaluate(() => {
+    const probe = window.__portfolioInteractionProbe;
+
+    if (!probe) {
+      throw new Error("The interaction timing probe is not installed.");
+    }
+
+    probe.entries = [];
+    probe.sampleStartedAt = performance.now();
+    probe.trustedClickCount = 0;
+  });
+}
+
+async function readInteractionSample(page: Page): Promise<InteractionSample> {
+  await settleNextPaint(page);
+
+  const snapshot = await page.evaluate(() => {
+    const probe = window.__portfolioInteractionProbe;
+
+    if (!probe) {
+      throw new Error("The interaction timing probe is not installed.");
+    }
+
+    return {
+      entries: probe.entries.filter(
+        (entry) => entry.startTime >= probe.sampleStartedAt,
+      ),
+      trustedClickCount: probe.trustedClickCount,
+    };
+  });
+
+  expect(
+    snapshot.trustedClickCount,
+    "Each sample must contain exactly one browser-trusted click.",
+  ).toBe(1);
+
+  const interactionEntries = snapshot.entries.filter(
+    (entry) => entry.interactionId > 0,
+  );
+
+  if (interactionEntries.length === 0) {
+    return {
+      durationUpperBoundMs: EVENT_TIMING_THRESHOLD_MS,
+      reportedDuration: `<${EVENT_TIMING_THRESHOLD_MS}ms`,
+    };
+  }
+
+  const interactionIds = [
+    ...new Set(interactionEntries.map((entry) => entry.interactionId)),
+  ];
+  expect(
+    interactionIds,
+    "One trusted click must resolve to one Event Timing interaction.",
+  ).toHaveLength(1);
+
+  const interactionId = interactionIds[0];
+  const entries = interactionEntries.filter(
+    (entry) => entry.interactionId === interactionId,
+  );
+  const duration = Math.max(...entries.map((entry) => entry.duration));
+
+  return {
+    durationUpperBoundMs: duration,
+    reportedDuration: `${duration.toFixed(1)}ms`,
+  };
+}
+
+function median(values: number[]) {
+  const sorted = [...values].sort((left, right) => left - right);
+  return sorted[Math.floor(sorted.length / 2)];
+}
+
+function reportAndAssertSamples({
+  designId,
+  projectName,
+  samples,
+  scenario,
+}: {
+  designId: DesignId;
+  projectName: string;
+  samples: InteractionSample[];
+  scenario: string;
+}) {
+  expect(samples).toHaveLength(SAMPLE_COUNT);
+
+  const upperBounds = samples.map((sample) => sample.durationUpperBoundMs);
+  const medianUpperBoundMs = median(upperBounds);
+  const maxUpperBoundMs = Math.max(...upperBounds);
+  const output = [
+    `[interaction-performance] ${projectName}`,
+    designId,
+    scenario,
+    `samples=${samples.map((sample) => sample.reportedDuration).join(",")}`,
+    `medianUpperBound=${medianUpperBoundMs.toFixed(1)}ms`,
+    `maxUpperBound=${maxUpperBoundMs.toFixed(1)}ms`,
+    `target=${INTERACTION_TARGET_MS}ms`,
+  ].join(" ");
+
+  console.info(output);
+
+  expect(
+    medianUpperBoundMs,
+    `${designId} ${scenario} median interaction duration upper bound`,
+  ).toBeLessThanOrEqual(INTERACTION_TARGET_MS);
+  expect(
+    maxUpperBoundMs,
+    `${designId} ${scenario} maximum interaction duration upper bound`,
+  ).toBeLessThanOrEqual(INTERACTION_TARGET_MS);
+}
+
+async function openDesignSwitcher(page: Page) {
+  const navigation = page.getByRole("navigation", {
+    name: presentationJson.ui.designNavigationAriaLabel,
+  });
+  const switcher = page.getByLabel(DESIGN_SWITCHER_LABEL_PATTERN);
+
+  await switcher.click();
+  await expect(navigation).toBeVisible();
+  await settleNextPaint(page);
+
+  return {
+    closeButton: navigation.getByRole("button", {
+      name: presentationJson.ui.designSwitcherCloseLabel,
+    }),
+    navigation,
+    switcher,
+  };
+}
+
+for (const designId of designIds) {
+  test(`${designId}: design switcher close stays within the interaction target`, async ({
+    page,
+  }, testInfo) => {
+    const isMobile = testInfo.project.name.includes("mobile");
+    const response = await page.goto(withExplicitDesign("/", designId));
+
+    expect(response?.ok()).toBe(true);
+    await expect(page.getByRole("heading", { level: 1 }).first()).toBeVisible();
+    await installInteractionProbe(page);
+
+    const warmup = await openDesignSwitcher(page);
+    await (isMobile ? warmup.closeButton : warmup.switcher).click();
+    await expect(warmup.navigation).toBeHidden();
+    await expect(warmup.switcher).toBeFocused();
+    await settleNextPaint(page);
+
+    const samples: InteractionSample[] = [];
+
+    for (let sampleIndex = 0; sampleIndex < SAMPLE_COUNT; sampleIndex += 1) {
+      const { closeButton, navigation, switcher } =
+        await openDesignSwitcher(page);
+
+      await resetInteractionProbe(page);
+      await (isMobile ? closeButton : switcher).click();
+      await expect(navigation).toBeHidden();
+      await expect(switcher).toBeFocused();
+      samples.push(await readInteractionSample(page));
+    }
+
+    reportAndAssertSamples({
+      designId,
+      projectName: testInfo.project.name,
+      samples,
+      scenario: "design-switcher-close",
+    });
+  });
+
+  test(`${designId}: mobile menu toggle stays within the interaction target`, async ({
+    page,
+  }, testInfo) => {
+    test.skip(
+      !testInfo.project.name.includes("mobile"),
+      "The menu toggle is measured with the mobile viewport.",
+    );
+
+    const response = await page.goto(withExplicitDesign("/", designId));
+
+    expect(response?.ok()).toBe(true);
+    await expect(page.getByRole("heading", { level: 1 }).first()).toBeVisible();
+    await installInteractionProbe(page);
+
+    const navigation = page.locator(
+      `nav[aria-label="${presentationJson.ui.mobileNavigationAriaLabel}"]`,
+    );
+    const menu = page
+      .locator("details")
+      .filter({ has: navigation })
+      .locator(":scope > summary");
+
+    await menu.click();
+    await expect(navigation).toBeVisible();
+    await menu.click();
+    await expect(navigation).toBeHidden();
+    await settleNextPaint(page);
+
+    const samples: InteractionSample[] = [];
+
+    for (let sampleIndex = 0; sampleIndex < SAMPLE_COUNT; sampleIndex += 1) {
+      await resetInteractionProbe(page);
+      await menu.click();
+      await expect(navigation).toBeVisible();
+      samples.push(await readInteractionSample(page));
+
+      await menu.click();
+      await expect(navigation).toBeHidden();
+      await settleNextPaint(page);
+    }
+
+    reportAndAssertSamples({
+      designId,
+      projectName: testInfo.project.name,
+      samples,
+      scenario: "mobile-menu-open",
+    });
+  });
+}


## `fix(perf): webpack route manifest parser 보강`

diff --git a/scripts/route-budgets.d.mts b/scripts/route-budgets.d.mts
new file mode 100644
index 0000000..309bdc9
--- /dev/null
+++ b/scripts/route-budgets.d.mts
@@ -0,0 +1,12 @@
+export type ClientReferenceManifest = {
+  entryCSSFiles?: Record<
+    string,
+    Array<{ inlined: boolean; path: string }>
+  >;
+  entryJSFiles?: Record<string, string[]>;
+};
+
+export function parseClientReferenceManifest(
+  source: string,
+  filename: string,
+): ClientReferenceManifest;
diff --git a/scripts/route-budgets.mjs b/scripts/route-budgets.mjs
new file mode 100644
index 0000000..1a7bb1b
--- /dev/null
+++ b/scripts/route-budgets.mjs
@@ -0,0 +1,13 @@
+export function parseClientReferenceManifest(source, filename) {
+  const assignment =
+    /globalThis\.__RSC_MANIFEST\[[\s\S]+?\]\s*=\s*/.exec(source);
+  const serialized = assignment
+    ? source.slice(assignment.index + assignment[0].length).trim()
+    : "";
+
+  if (!assignment || !serialized.endsWith(";")) {
+    throw new Error(`Cannot parse client reference manifest: ${filename}`);
+  }
+
+  return JSON.parse(serialized.slice(0, -1));
+}


## `test(build): compiler와 manifest parser 계약 검증`

diff --git a/src/performance/build-manifest-contract.test.ts b/src/performance/build-manifest-contract.test.ts
new file mode 100644
index 0000000..03eef34
--- /dev/null
+++ b/src/performance/build-manifest-contract.test.ts
@@ -0,0 +1,36 @@
+import { createRequire } from "node:module";
+
+import { describe, expect, it } from "vitest";
+
+import { parseClientReferenceManifest } from "../../scripts/route-budgets.mjs";
+
+const require = createRequire(import.meta.url);
+const packageJson = require("../../package.json");
+
+describe("production build pipeline", () => {
+  it("uses the verified webpack compiler path", () => {
+    expect(packageJson.scripts.build).toBe("next build --webpack");
+  });
+});
+
+describe("webpack client reference manifest parser", () => {
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
+});


## `build(perf): route별 client asset 측정 추가`

diff --git a/scripts/route-budgets.d.mts b/scripts/route-budgets.d.mts
index 309bdc9..e17e41f 100644
--- a/scripts/route-budgets.d.mts
+++ b/scripts/route-budgets.d.mts
@@ -1,3 +1,10 @@
+export type RouteBundleSize = {
+  cssBytes: number;
+  jsBytes: number;
+};
+
+export type RouteBundleMeasurement = Record<string, RouteBundleSize>;
+
 export type ClientReferenceManifest = {
   entryCSSFiles?: Record<
     string,
@@ -10,3 +17,7 @@ export function parseClientReferenceManifest(
   source: string,
   filename: string,
 ): ClientReferenceManifest;
+
+export function collectRouteBundleMeasurements(
+  buildDirectory?: string,
+): Promise<RouteBundleMeasurement>;
diff --git a/scripts/route-budgets.mjs b/scripts/route-budgets.mjs
index 1a7bb1b..aee7a9c 100644
--- a/scripts/route-budgets.mjs
+++ b/scripts/route-budgets.mjs
@@ -1,3 +1,16 @@
+import { readFile, stat } from "node:fs/promises";
+import path from "node:path";
+
+const DEFAULT_BUILD_DIRECTORY = ".next";
+
+function routeFromManifestKey(key) {
+  if (key === "/page") {
+    return "/";
+  }
+
+  return key.replace(/\/page$/, "");
+}
+
 export function parseClientReferenceManifest(source, filename) {
   const assignment =
     /globalThis\.__RSC_MANIFEST\[[\s\S]+?\]\s*=\s*/.exec(source);
@@ -11,3 +24,62 @@ export function parseClientReferenceManifest(source, filename) {
 
   return JSON.parse(serialized.slice(0, -1));
 }
+
+async function assetBytes(buildDirectory, assets) {
+  const sizes = await Promise.all(
+    [...new Set(assets)].map(async (asset) => {
+      const assetPath = path.join(buildDirectory, asset.replace(/^\//, ""));
+      return (await stat(assetPath)).size;
+    }),
+  );
+
+  return sizes.reduce((total, size) => total + size, 0);
+}
+
+export async function collectRouteBundleMeasurements(
+  buildDirectory = DEFAULT_BUILD_DIRECTORY,
+) {
+  const appPaths = JSON.parse(
+    await readFile(
+      path.join(buildDirectory, "server/app-paths-manifest.json"),
+      "utf8",
+    ),
+  );
+  const buildManifest = JSON.parse(
+    await readFile(path.join(buildDirectory, "build-manifest.json"), "utf8"),
+  );
+  const sharedJavaScript = buildManifest.rootMainFiles ?? [];
+  const measurements = {};
+
+  for (const key of Object.keys(appPaths).sort()) {
+    if (!key.endsWith("/page") || key.startsWith("/_")) {
+      continue;
+    }
+
+    const route = routeFromManifestKey(key);
+    const relativeManifestPath = `server/${appPaths[key].replace(
+      /\.js$/,
+      "_client-reference-manifest.js",
+    )}`;
+    const manifestPath = path.join(buildDirectory, relativeManifestPath);
+    const manifest = parseClientReferenceManifest(
+      await readFile(manifestPath, "utf8"),
+      manifestPath,
+    );
+    const routeJavaScript = Object.values(manifest.entryJSFiles ?? {}).flat();
+    const routeCss = Object.values(manifest.entryCSSFiles ?? {})
+      .flat()
+      .filter(({ inlined }) => !inlined)
+      .map(({ path: assetPath }) => assetPath);
+
+    measurements[route] = {
+      cssBytes: await assetBytes(buildDirectory, routeCss),
+      jsBytes: await assetBytes(buildDirectory, [
+        ...sharedJavaScript,
+        ...routeJavaScript,
+      ]),
+    };
+  }
+
+  return measurements;
+}


## `build(perf): route bundle 성장 예산 평가 추가`

diff --git a/scripts/route-budgets.d.mts b/scripts/route-budgets.d.mts
index e17e41f..bc36423 100644
--- a/scripts/route-budgets.d.mts
+++ b/scripts/route-budgets.d.mts
@@ -1,3 +1,5 @@
+export const BUDGET_GROWTH_FACTOR: 1.05;
+
 export type RouteBundleSize = {
   cssBytes: number;
   jsBytes: number;
@@ -5,6 +7,21 @@ export type RouteBundleSize = {
 
 export type RouteBundleMeasurement = Record<string, RouteBundleSize>;
 
+export type RouteBudgetBaseline = {
+  schemaVersion: 1;
+  growthLimitPercent: 5;
+  routes: RouteBundleMeasurement;
+};
+
+export type RouteBudgetViolation = {
+  actualBytes?: number;
+  allowedBytes?: number;
+  asset: "baseline" | "css" | "js" | "route";
+  baselineBytes?: number;
+  message: string;
+  route: string;
+};
+
 export type ClientReferenceManifest = {
   entryCSSFiles?: Record<
     string,
@@ -21,3 +38,8 @@ export function parseClientReferenceManifest(
 export function collectRouteBundleMeasurements(
   buildDirectory?: string,
 ): Promise<RouteBundleMeasurement>;
+
+export function evaluateRouteBudgets(
+  measurements: RouteBundleMeasurement,
+  baseline: RouteBudgetBaseline,
+): RouteBudgetViolation[];
diff --git a/scripts/route-budgets.mjs b/scripts/route-budgets.mjs
index aee7a9c..573dabe 100644
--- a/scripts/route-budgets.mjs
+++ b/scripts/route-budgets.mjs
@@ -1,6 +1,8 @@
 import { readFile, stat } from "node:fs/promises";
 import path from "node:path";
 
+export const BUDGET_GROWTH_FACTOR = 1.05;
+
 const DEFAULT_BUILD_DIRECTORY = ".next";
 
 function routeFromManifestKey(key) {
@@ -83,3 +85,50 @@ export async function collectRouteBundleMeasurements(
 
   return measurements;
 }
+
+export function evaluateRouteBudgets(measurements, baseline) {
+  const violations = [];
+  const baselineRoutes = baseline.routes ?? {};
+
+  for (const [route, expected] of Object.entries(baselineRoutes)) {
+    const actual = measurements[route];
+
+    if (!actual) {
+      violations.push({
+        asset: "route",
+        message: `${route}: route output is missing`,
+        route,
+      });
+      continue;
+    }
+
+    for (const [property, asset] of [
+      ["cssBytes", "css"],
+      ["jsBytes", "js"],
+    ]) {
+      const allowedBytes = Math.floor(expected[property] * BUDGET_GROWTH_FACTOR);
+      if (actual[property] > allowedBytes) {
+        violations.push({
+          actualBytes: actual[property],
+          allowedBytes,
+          asset,
+          baselineBytes: expected[property],
+          message: `${route}: ${asset} is ${actual[property]} bytes (limit ${allowedBytes})`,
+          route,
+        });
+      }
+    }
+  }
+
+  for (const route of Object.keys(measurements)) {
+    if (!baselineRoutes[route]) {
+      violations.push({
+        asset: "baseline",
+        message: `${route}: route does not have a committed baseline`,
+        route,
+      });
+    }
+  }
+
+  return violations;
+}


