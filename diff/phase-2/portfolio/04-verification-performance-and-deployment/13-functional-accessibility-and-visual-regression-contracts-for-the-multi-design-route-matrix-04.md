## `test(visual): 다섯 디자인 회귀 기준 추가`

diff --git a/playwright.config.ts b/playwright.config.ts
index 13229fd..45d895f 100644
--- a/playwright.config.ts
+++ b/playwright.config.ts
@@ -8,6 +8,8 @@ export default defineConfig({
   },
   reporter: [["list"]],
   outputDir: "test-results",
+  snapshotPathTemplate:
+    "{testDir}/{testFilePath}-snapshots/{arg}-{projectName}{ext}",
   // Next's development compiler can invalidate an in-flight navigation when
   // both device projects request a cold route at the same time.
   workers: 1,
diff --git a/playwright.production.config.ts b/playwright.production.config.ts
index c5f7502..9c6ad51 100644
--- a/playwright.production.config.ts
+++ b/playwright.production.config.ts
@@ -8,6 +8,8 @@ export default defineConfig({
   },
   reporter: [["list"]],
   outputDir: "test-results",
+  snapshotPathTemplate:
+    "{testDir}/{testFilePath}-snapshots/{arg}-{projectName}{ext}",
   workers: 1,
   use: {
     baseURL: "http://localhost:3200",
diff --git a/src/designs/visual-regression-contract.test.ts b/src/designs/visual-regression-contract.test.ts
new file mode 100644
index 0000000..005eff1
--- /dev/null
+++ b/src/designs/visual-regression-contract.test.ts
@@ -0,0 +1,26 @@
+import { readdirSync } from "node:fs";
+import { resolve } from "node:path";
+import { describe, expect, it } from "vitest";
+
+import { SITE_DESIGN_IDS } from "./config";
+
+const snapshotDirectory = resolve(
+  process.cwd(),
+  "tests/e2e/visual.spec.ts-snapshots",
+);
+
+const expectedSnapshotManifest = SITE_DESIGN_IDS.flatMap((designId) => [
+  `home-${designId}-chromium.png`,
+  `home-${designId}-mobile-chrome.png`,
+  `project-${designId}-chromium.png`,
+]).sort();
+
+describe("visual regression contract", () => {
+  it("keeps the exact desktop and mobile snapshot manifest", () => {
+    const actualSnapshotManifest = readdirSync(snapshotDirectory)
+      .filter((fileName) => fileName.endsWith(".png"))
+      .sort();
+
+    expect(actualSnapshotManifest).toEqual(expectedSnapshotManifest);
+  });
+});
diff --git a/tests/e2e/visual.spec.ts b/tests/e2e/visual.spec.ts
new file mode 100644
index 0000000..000ba48
--- /dev/null
+++ b/tests/e2e/visual.spec.ts
@@ -0,0 +1,72 @@
+import { expect, test, type Page } from "@playwright/test";
+
+import {
+  designIds,
+  firstEnabledProject,
+  withExplicitDesign,
+} from "./site-matrix";
+
+const projectId = firstEnabledProject?.id;
+
+if (!projectId) {
+  throw new Error("Visual snapshots need one enabled project.");
+}
+
+async function prepareStablePage(page: Page, path: string) {
+  await page.emulateMedia({ reducedMotion: "reduce" });
+  const response = await page.goto(path, { waitUntil: "networkidle" });
+
+  expect(response?.ok()).toBe(true);
+  await page.evaluate(async () => {
+    await document.fonts.ready;
+    await Promise.all(
+      Array.from(document.images, (image) => {
+        if (image.complete) {
+          return Promise.resolve();
+        }
+
+        return new Promise<void>((resolve) => {
+          image.addEventListener("load", () => resolve(), { once: true });
+          image.addEventListener("error", () => resolve(), { once: true });
+        });
+      }),
+    );
+  });
+}
+
+for (const designId of designIds) {
+  test(
+    `${designId}: home visual`,
+    { tag: "@visual" },
+    async ({ page }, testInfo) => {
+      await prepareStablePage(page, withExplicitDesign("/", designId));
+
+      await expect(page).toHaveScreenshot(`home-${designId}.png`, {
+        animations: "disabled",
+        fullPage: true,
+        maxDiffPixelRatio: 0.01,
+      });
+
+      expect(["chromium", "mobile-chrome"]).toContain(testInfo.project.name);
+    },
+  );
+
+  test(
+    `${designId}: project detail desktop visual`,
+    { tag: "@visual" },
+    async ({ page }, testInfo) => {
+      test.skip(testInfo.project.name !== "chromium", "Desktop reference only.");
+
+      await prepareStablePage(
+        page,
+        withExplicitDesign(`/projects/${projectId}`, designId),
+      );
+
+      await expect(page).toHaveScreenshot(`project-${designId}.png`, {
+        animations: "disabled",
+        fullPage: true,
+        maxDiffPixelRatio: 0.01,
+      });
+    },
+  );
+}
diff --git a/tests/e2e/visual.spec.ts-snapshots/home-brutalist-chromium.png b/tests/e2e/visual.spec.ts-snapshots/home-brutalist-chromium.png
new file mode 100644
index 0000000..b9ae1f1
Binary files /dev/null and b/tests/e2e/visual.spec.ts-snapshots/home-brutalist-chromium.png differ
diff --git a/tests/e2e/visual.spec.ts-snapshots/home-brutalist-mobile-chrome.png b/tests/e2e/visual.spec.ts-snapshots/home-brutalist-mobile-chrome.png
new file mode 100644
index 0000000..b290d78
Binary files /dev/null and b/tests/e2e/visual.spec.ts-snapshots/home-brutalist-mobile-chrome.png differ
diff --git a/tests/e2e/visual.spec.ts-snapshots/home-cinematic-chromium.png b/tests/e2e/visual.spec.ts-snapshots/home-cinematic-chromium.png
new file mode 100644
index 0000000..39b7deb
Binary files /dev/null and b/tests/e2e/visual.spec.ts-snapshots/home-cinematic-chromium.png differ
diff --git a/tests/e2e/visual.spec.ts-snapshots/home-cinematic-mobile-chrome.png b/tests/e2e/visual.spec.ts-snapshots/home-cinematic-mobile-chrome.png
new file mode 100644
index 0000000..5d83c05
Binary files /dev/null and b/tests/e2e/visual.spec.ts-snapshots/home-cinematic-mobile-chrome.png differ
diff --git a/tests/e2e/visual.spec.ts-snapshots/home-classic-chromium.png b/tests/e2e/visual.spec.ts-snapshots/home-classic-chromium.png
new file mode 100644
index 0000000..06fa8fb
Binary files /dev/null and b/tests/e2e/visual.spec.ts-snapshots/home-classic-chromium.png differ
diff --git a/tests/e2e/visual.spec.ts-snapshots/home-classic-mobile-chrome.png b/tests/e2e/visual.spec.ts-snapshots/home-classic-mobile-chrome.png
new file mode 100644
index 0000000..5c9bb90
Binary files /dev/null and b/tests/e2e/visual.spec.ts-snapshots/home-classic-mobile-chrome.png differ
diff --git a/tests/e2e/visual.spec.ts-snapshots/home-design-chromium.png b/tests/e2e/visual.spec.ts-snapshots/home-design-chromium.png
new file mode 100644
index 0000000..3159c93
Binary files /dev/null and b/tests/e2e/visual.spec.ts-snapshots/home-design-chromium.png differ
diff --git a/tests/e2e/visual.spec.ts-snapshots/home-design-mobile-chrome.png b/tests/e2e/visual.spec.ts-snapshots/home-design-mobile-chrome.png
new file mode 100644
index 0000000..bf5ccbc
Binary files /dev/null and b/tests/e2e/visual.spec.ts-snapshots/home-design-mobile-chrome.png differ
diff --git a/tests/e2e/visual.spec.ts-snapshots/home-editorial-chromium.png b/tests/e2e/visual.spec.ts-snapshots/home-editorial-chromium.png
new file mode 100644
index 0000000..25a9a31
Binary files /dev/null and b/tests/e2e/visual.spec.ts-snapshots/home-editorial-chromium.png differ
diff --git a/tests/e2e/visual.spec.ts-snapshots/home-editorial-mobile-chrome.png b/tests/e2e/visual.spec.ts-snapshots/home-editorial-mobile-chrome.png
new file mode 100644
index 0000000..fa777b3
Binary files /dev/null and b/tests/e2e/visual.spec.ts-snapshots/home-editorial-mobile-chrome.png differ
diff --git a/tests/e2e/visual.spec.ts-snapshots/project-brutalist-chromium.png b/tests/e2e/visual.spec.ts-snapshots/project-brutalist-chromium.png
new file mode 100644
index 0000000..1a960fa
Binary files /dev/null and b/tests/e2e/visual.spec.ts-snapshots/project-brutalist-chromium.png differ
diff --git a/tests/e2e/visual.spec.ts-snapshots/project-cinematic-chromium.png b/tests/e2e/visual.spec.ts-snapshots/project-cinematic-chromium.png
new file mode 100644
index 0000000..be00d6f
Binary files /dev/null and b/tests/e2e/visual.spec.ts-snapshots/project-cinematic-chromium.png differ
diff --git a/tests/e2e/visual.spec.ts-snapshots/project-classic-chromium.png b/tests/e2e/visual.spec.ts-snapshots/project-classic-chromium.png
new file mode 100644
index 0000000..b4301cd
Binary files /dev/null and b/tests/e2e/visual.spec.ts-snapshots/project-classic-chromium.png differ
diff --git a/tests/e2e/visual.spec.ts-snapshots/project-design-chromium.png b/tests/e2e/visual.spec.ts-snapshots/project-design-chromium.png
new file mode 100644
index 0000000..adea93a
Binary files /dev/null and b/tests/e2e/visual.spec.ts-snapshots/project-design-chromium.png differ
diff --git a/tests/e2e/visual.spec.ts-snapshots/project-editorial-chromium.png b/tests/e2e/visual.spec.ts-snapshots/project-editorial-chromium.png
new file mode 100644
index 0000000..0875737
Binary files /dev/null and b/tests/e2e/visual.spec.ts-snapshots/project-editorial-chromium.png differ
