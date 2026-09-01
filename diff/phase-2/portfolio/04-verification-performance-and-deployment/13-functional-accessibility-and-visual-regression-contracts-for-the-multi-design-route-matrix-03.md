## `test(portfolio): selector와 presentation 회귀 계약 보강`

diff --git a/src/lib/portfolio.test.ts b/src/lib/portfolio.test.ts
index 081ed22..101fe21 100644
--- a/src/lib/portfolio.test.ts
+++ b/src/lib/portfolio.test.ts
@@ -9,6 +9,7 @@ import siteJson from "@/content/site.json";
 import { SITE_DESIGN_IDS } from "@/designs/config";
 import { describe, expect, it } from "vitest";
 
+import * as portfolio from "./portfolio";
 import { validatePortfolioAssets } from "./content-assets";
 import { loadPortfolioSource, PortfolioContentError } from "./content-loader";
 import {
@@ -65,6 +66,45 @@ function captureContentError(run: () => unknown) {
 }
 
 describe("portfolio content", () => {
+  it("preserves the public module surface and clone boundaries", () => {
+    expect(Object.keys(portfolio).sort()).toEqual(
+      [
+        "getContentLinksByPlacement",
+        "getEnabledLinks",
+        "getExternalLinkProps",
+        "getFeaturedProjects",
+        "getPortfolioContent",
+        "getPreferredContactLinks",
+        "getProjectById",
+        "getProjectCardLinks",
+        "getProjectDetailLinks",
+        "getProjectLink",
+        "getProjectLinksForPlacement",
+        "getProjectMetricValue",
+        "getResumeProjects",
+        "getTemplateHref",
+        "isProjectLive",
+        "isSitePageEnabled",
+        "resolveContentDebug",
+        "resolveHomeTemplateId",
+        "resolveTechStackItem",
+      ].sort(),
+    );
+
+    const first = getPortfolioContent();
+    const second = getPortfolioContent();
+
+    expect(first).not.toBe(second);
+    expect(first.projects).not.toBe(second.projects);
+    expect(first.projects[0]).not.toBe(second.projects[0]);
+    expect(first.projects[0].links).not.toBe(second.projects[0].links);
+    expect(first.links).not.toBe(second.links);
+    expect(first.site).toBe(second.site);
+    expect(first.profile).toBe(second.profile);
+    expect(first.presentation).toBe(second.presentation);
+    expect(first.journey).toBe(second.journey);
+  });
+
   it("loads and derives the reusable projects content model", () => {
     const source = validatePortfolioAssets(
       loadPortfolioSource(),


## `test(e2e): production server 검증 경로 추가`

diff --git a/package.json b/package.json
index cffeede..f600b81 100644
--- a/package.json
+++ b/package.json
@@ -12,12 +12,14 @@
     "prebuild": "npm run content:check",
     "build": "next build",
     "start": "next start -p 3100",
+    "start:e2e": "next start -p 3200",
     "lint": "eslint .",
     "typecheck": "tsc --noEmit",
     "content:check": "node --import tsx scripts/validate-content.ts",
     "test": "vitest run",
     "test:watch": "vitest",
-    "test:e2e": "playwright test"
+    "test:e2e": "playwright test",
+    "test:e2e:production": "npm run build && playwright test --config=playwright.production.config.ts"
   },
   "dependencies": {
     "next": "16.2.4",
diff --git a/playwright.production.config.ts b/playwright.production.config.ts
new file mode 100644
index 0000000..c5f7502
--- /dev/null
+++ b/playwright.production.config.ts
@@ -0,0 +1,32 @@
+import { defineConfig, devices } from "@playwright/test";
+
+export default defineConfig({
+  testDir: "./tests/e2e",
+  timeout: 30_000,
+  expect: {
+    timeout: 5_000,
+  },
+  reporter: [["list"]],
+  outputDir: "test-results",
+  workers: 1,
+  use: {
+    baseURL: "http://localhost:3200",
+    trace: "on-first-retry",
+  },
+  webServer: {
+    command: "npm run start:e2e",
+    url: "http://localhost:3200",
+    reuseExistingServer: false,
+    timeout: 120_000,
+  },
+  projects: [
+    {
+      name: "chromium",
+      use: { ...devices["Desktop Chrome"] },
+    },
+    {
+      name: "mobile-chrome",
+      use: { ...devices["Pixel 7"] },
+    },
+  ],
+});


## `fix(a11y): 디자인별 색상 대비 보정`

diff --git a/src/app/globals.css b/src/app/globals.css
index 264befe..9a5b736 100644
--- a/src/app/globals.css
+++ b/src/app/globals.css
@@ -9,7 +9,7 @@
   --surface: #ffffff;
   --surface-soft: #f2f7f6;
   --surface-hover: #f8fbfb;
-  --accent: #008c89;
+  --accent: #007b78;
   --accent-strong: #006b69;
   --accent-soft: rgba(0, 140, 137, 0.12);
   --signal: #4f46e5;
@@ -332,7 +332,7 @@ a {
   border: 1px solid rgba(255, 255, 255, 0.08);
   border-radius: 8px;
   box-shadow: 0 10px 30px rgba(0, 0, 0, 0.18);
-  color: var(--foreground);
+  color: #f2efe7;
   display: inline-flex;
   flex: 0 0 auto;
   font-size: 0.78rem;
diff --git a/src/designs/editorial/editorial-route.module.css b/src/designs/editorial/editorial-route.module.css
index 9e1633d..311dbe0 100644
--- a/src/designs/editorial/editorial-route.module.css
+++ b/src/designs/editorial/editorial-route.module.css
@@ -3,8 +3,9 @@
   --paper-deep: #e6ddcc;
   --ink: #171614;
   --ink-soft: #5f5a52;
-  --vermilion: #d64b32;
+  --vermilion: #c7432e;
   --vermilion-text: #a93222;
+  --vermilion-on-dark: #e15b43;
   --hairline: rgb(23 22 20 / 24%);
   min-height: 100vh;
   overflow: clip;
@@ -1059,7 +1060,7 @@
 }
 
 .darkSectionTitle > span {
-  color: var(--vermilion-text);
+  color: var(--vermilion-on-dark);
 }
 
 .darkSectionTitle h2 {
@@ -1082,7 +1083,7 @@
 
 .architectureSpread aside > p {
   margin-bottom: 20px;
-  color: var(--vermilion-text);
+  color: var(--vermilion-on-dark);
   font-size: 0.64rem;
   font-weight: 800;
   text-transform: uppercase;
@@ -1122,6 +1123,10 @@
   border-color: rgb(242 235 221 / 22%);
 }
 
+.architectureSpread .evidenceList span {
+  color: var(--vermilion-on-dark);
+}
+
 .evidenceList span {
   color: var(--vermilion-text);
   font-family: var(--font-noto-serif-kr), Georgia, serif;
@@ -1627,6 +1632,10 @@
   border-top-color: rgb(242 235 221 / 46%);
 }
 
+.nextReview .curationPanelHeader > span {
+  color: var(--vermilion-on-dark);
+}
+
 .nextReview .curationPanelHeader p,
 .nextReview article p {
   color: rgb(242 235 221 / 72%);
@@ -2353,7 +2362,7 @@
 }
 
 .gapsSpread > div > span {
-  color: var(--vermilion-text);
+  color: var(--vermilion-on-dark);
   font-family: var(--font-noto-serif-kr), Georgia, serif;
   font-size: 0.8rem;
   font-style: italic;
@@ -2375,6 +2384,10 @@
   line-height: 1.75;
 }
 
+.gapsSpread .evidenceList span {
+  color: var(--vermilion-on-dark);
+}
+
 @media (max-width: 1180px) {
   .mastheadMain {
     grid-template-columns: 0.7fr 1.3fr auto;


## `fix(a11y): skip link focus target 복원`

diff --git a/src/components/portfolio/site-shell.tsx b/src/components/portfolio/site-shell.tsx
index e9bc83a..e70c7c6 100644
--- a/src/components/portfolio/site-shell.tsx
+++ b/src/components/portfolio/site-shell.tsx
@@ -168,7 +168,7 @@ export function PageShell({
         templateSwitcher={templateSwitcher}
         ui={ui}
       />
-      <main data-home-template={homeTemplate} id="main-content">
+      <main data-home-template={homeTemplate} id="main-content" tabIndex={-1}>
         {children}
       </main>
       <SiteFooter
diff --git a/src/designs/cinematic/cinematic-route.tsx b/src/designs/cinematic/cinematic-route.tsx
index 16e25bb..1f96871 100644
--- a/src/designs/cinematic/cinematic-route.tsx
+++ b/src/designs/cinematic/cinematic-route.tsx
@@ -151,7 +151,9 @@ function Frame({
           </nav>
         </details>
       </header>
-      <main id="cinematic-content">{children}</main>
+      <main id="cinematic-content" tabIndex={-1}>
+        {children}
+      </main>
       <footer className={styles.footer}>
         <p>{content.site.footer.note}</p>
         {footerLinks.length > 0 ? (
diff --git a/src/designs/editorial/editorial-route.tsx b/src/designs/editorial/editorial-route.tsx
index 4cdb2d3..acbe1f5 100644
--- a/src/designs/editorial/editorial-route.tsx
+++ b/src/designs/editorial/editorial-route.tsx
@@ -233,7 +233,9 @@ function EditorialShell({
           </details>
         </div>
       </header>
-      <main id="editorial-main">{children}</main>
+      <main id="editorial-main" tabIndex={-1}>
+        {children}
+      </main>
       <footer className={styles.footer}>
         <div className={styles.footerLead}>
           <p>{content.site.footer.note}</p>


## `fix(a11y): Brutalist 지표의 definition semantics 수정`

diff --git a/src/designs/brutalist/brutalist-route.tsx b/src/designs/brutalist/brutalist-route.tsx
index e71b497..c6c60f6 100644
--- a/src/designs/brutalist/brutalist-route.tsx
+++ b/src/designs/brutalist/brutalist-route.tsx
@@ -272,7 +272,7 @@ function BrutalistShell({
           </span>
         </aside>
       ) : null}
-      <main className={styles.main} id="brutalist-main">
+      <main className={styles.main} id="brutalist-main" tabIndex={-1}>
         {children}
       </main>
       <footer className={styles.footer}>
@@ -363,7 +363,11 @@ function HomeView({
                         {String(index + 1).padStart(2, "0")} / {metric.label}
                       </dt>
                       <dd>{String(metric.value).padStart(2, "0")}</dd>
-                      {metric.description ? <p>{metric.description}</p> : null}
+                      {metric.description ? (
+                        <dd className={styles.metricDescription}>
+                          {metric.description}
+                        </dd>
+                      ) : null}
                     </div>
                   ))}
                 </dl>
diff --git a/src/designs/brutalist/brutalist.module.css b/src/designs/brutalist/brutalist.module.css
index 604157c..4339a91 100644
--- a/src/designs/brutalist/brutalist.module.css
+++ b/src/designs/brutalist/brutalist.module.css
@@ -373,8 +373,10 @@
   margin: 1.5rem 0 0;
 }
 
-.metricBlock p {
+.metricBlock .metricDescription {
   font-size: 0.68rem;
+  font-weight: 400;
+  letter-spacing: normal;
   line-height: 1.4;
   margin: 1rem 0 0;
   max-width: 16rem;


## `test(a11y): 디자인×route WCAG 행렬 추가`

diff --git a/package-lock.json b/package-lock.json
index 23de53f..95624f5 100644
--- a/package-lock.json
+++ b/package-lock.json
@@ -15,6 +15,7 @@
         "zod": "^4.4.3"
       },
       "devDependencies": {
+        "@axe-core/playwright": "^4.12.1",
         "@playwright/test": "^1.59.1",
         "@tailwindcss/postcss": "^4",
         "@testing-library/jest-dom": "^6.9.1",
@@ -110,6 +111,19 @@
       "dev": true,
       "license": "MIT"
     },
+    "node_modules/@axe-core/playwright": {
+      "version": "4.12.1",
+      "resolved": "https://registry.npmjs.org/@axe-core/playwright/-/playwright-4.12.1.tgz",
+      "integrity": "sha512-rMd7xriptqKpP+w5265i4Hdkv2X5kbu6uiBi/B2I7uf3hieRBM3qDCfaKPtxfiYb2mKXfF+yLODJwIx+Jv1GDw==",
+      "dev": true,
+      "license": "MPL-2.0",
+      "dependencies": {
+        "axe-core": "~4.12.1"
+      },
+      "peerDependencies": {
+        "playwright-core": ">= 1.0.0"
+      }
+    },
     "node_modules/@babel/code-frame": {
       "version": "7.29.0",
       "resolved": "https://registry.npmjs.org/@babel/code-frame/-/code-frame-7.29.0.tgz",
@@ -3711,9 +3725,9 @@
       }
     },
     "node_modules/axe-core": {
-      "version": "4.11.4",
-      "resolved": "https://registry.npmjs.org/axe-core/-/axe-core-4.11.4.tgz",
-      "integrity": "sha512-KunSNx+TVpkAw/6ULfhnx+HWRecjqZGTOyquAoWHYLRSdK1tB5Ihce1ZW+UY3fj33bYAFWPu7W/GRSmmrCGuxA==",
+      "version": "4.12.1",
+      "resolved": "https://registry.npmjs.org/axe-core/-/axe-core-4.12.1.tgz",
+      "integrity": "sha512-s7iGf5GaVMxEG0ENN9x+xTr7GFZCb1ZP/1uATUpCEK2X78nDB3RwbtFCo9pGAf9ru+VwoQ464DkaLEeRM08wJA==",
       "dev": true,
       "license": "MPL-2.0",
       "engines": {
diff --git a/package.json b/package.json
index bb92b96..ad0f4c4 100644
--- a/package.json
+++ b/package.json
@@ -31,6 +31,7 @@
     "zod": "^4.4.3"
   },
   "devDependencies": {
+    "@axe-core/playwright": "^4.12.1",
     "@playwright/test": "^1.59.1",
     "@tailwindcss/postcss": "^4",
     "@testing-library/jest-dom": "^6.9.1",
diff --git a/tests/e2e/accessibility.spec.ts b/tests/e2e/accessibility.spec.ts
new file mode 100644
index 0000000..86e126b
--- /dev/null
+++ b/tests/e2e/accessibility.spec.ts
@@ -0,0 +1,84 @@
+import AxeBuilder from "@axe-core/playwright";
+import { expect, test, type Page } from "@playwright/test";
+
+import presentationJson from "../../src/content/presentation.json";
+import {
+  designIds,
+  enabledRoutes,
+  withExplicitDesign,
+  type DesignId,
+} from "./site-matrix";
+
+const wcag22AATags = [
+  "wcag2a",
+  "wcag2aa",
+  "wcag21a",
+  "wcag21aa",
+  "wcag22aa",
+] as const;
+
+function formatViolations(
+  designId: DesignId,
+  path: string,
+  violations: Awaited<ReturnType<AxeBuilder["analyze"]>>["violations"],
+) {
+  return violations
+    .map(
+      (violation) =>
+        `${designId} ${path}: ${violation.id} (${violation.impact ?? "unknown"})\n${violation.nodes
+          .map((node) => `  ${node.target.join(" ")}\n  ${node.failureSummary}`)
+          .join("\n")}`,
+    )
+    .join("\n\n");
+}
+
+async function expectAccessibleRoute(
+  page: Page,
+  path: string,
+  designId: DesignId,
+) {
+  const response = await page.goto(withExplicitDesign(path, designId));
+
+  expect(response?.ok()).toBe(true);
+  await expect(page.locator(`[data-site-design="${designId}"]`)).toBeVisible();
+  await expect(page.getByRole("main")).toHaveCount(1);
+  await expect(page.getByRole("banner")).toHaveCount(1);
+  await expect(page.getByRole("contentinfo")).toHaveCount(1);
+
+  const results = await new AxeBuilder({ page })
+    .withTags([...wcag22AATags])
+    .analyze();
+
+  expect(
+    results.violations,
+    formatViolations(designId, path, results.violations),
+  ).toEqual([]);
+}
+
+for (const designId of designIds) {
+  test(`${designId}: every enabled route passes WCAG 2.2 AA checks`, async ({
+    page,
+  }) => {
+    test.slow();
+
+    for (const { path } of enabledRoutes) {
+      await expectAccessibleRoute(page, path, designId);
+    }
+  });
+
+  test(`${designId}: keyboard users can skip repeated navigation`, async ({
+    page,
+  }) => {
+    await page.goto(withExplicitDesign("/", designId));
+
+    const skipLink = page.getByRole("link", {
+      name: presentationJson.ui.skipLinkLabel,
+    });
+    const main = page.getByRole("main");
+
+    await page.keyboard.press("Tab");
+    await expect(skipLink).toBeFocused();
+    await page.keyboard.press("Enter");
+    await expect(main).toBeFocused();
+  });
+}
diff --git a/tests/e2e/portfolio.spec.ts b/tests/e2e/portfolio.spec.ts
index bab285c..3e0eb1e 100644
--- a/tests/e2e/portfolio.spec.ts
+++ b/tests/e2e/portfolio.spec.ts
@@ -13,15 +13,13 @@ import resumeJson from "../../src/content/resume.json";
 import siteJson from "../../src/content/site.json";
 import skillsJson from "../../src/content/skills.json";
 import techStackJson from "../../src/content/tech-stack.json";
-
-const designIds = [
-  "design",
-  "classic",
-  "editorial",
-  "brutalist",
-  "cinematic",
-] as const;
-type DesignId = (typeof designIds)[number];
+import {
+  designIds,
+  enabledRoutes,
+  firstEnabledProject,
+  withExplicitDesign,
+  type DesignId,
+} from "./site-matrix";
 
 function requireFixture<T>(value: T | undefined, message: string): T {
   if (value === undefined) {
@@ -32,7 +30,7 @@ function requireFixture<T>(value: T | undefined, message: string): T {
 }
 
 const firstProject = requireFixture(
-  projectsJson.items.find((project) => project.enabled !== false),
+  firstEnabledProject,
   "The portfolio needs at least one enabled project.",
 );
 const firstProjectTechnology = requireFixture(
@@ -55,21 +53,6 @@ const templateLabels = new Map(
   presentationJson.templates.map((template) => [template.id, template.label]),
 );
 
-const routeDefinitions = [
-  { path: "/", pageId: undefined },
-  { path: "/projects", pageId: "projects" },
-  { path: `/projects/${firstProject.id}`, pageId: "projects" },
-  { path: "/about", pageId: "about" },
-  { path: "/resume", pageId: "resume" },
-  { path: "/contact", pageId: "contact" },
-  { path: "/journey", pageId: "journey" },
-  { path: "/interview-map", pageId: "interviewMap" },
-] as const;
-
-const enabledRoutes = routeDefinitions.filter(
-  ({ pageId }) => !pageId || siteJson.pages?.[pageId] !== false,
-);
-
 const projectsNavigation = siteJson.navigation.find(
   (item) => item.href === "/projects",
 );
@@ -78,12 +61,6 @@ if (!projectsNavigation) {
   throw new Error("site.json must include a /projects navigation item.");
 }
 
-function withExplicitDesign(path: string, designId: DesignId) {
-  const url = new URL(path, "https://portfolio.test");
-  url.searchParams.set("view", designId);
-  return `${url.pathname}${url.search}${url.hash}`;
-}
-
 function expectedInternalHref(
   path: string,
   designId: DesignId,
diff --git a/tests/e2e/site-matrix.ts b/tests/e2e/site-matrix.ts
new file mode 100644
index 0000000..beca9f7
--- /dev/null
+++ b/tests/e2e/site-matrix.ts
@@ -0,0 +1,41 @@
+import projectsJson from "../../src/content/projects.json";
+import siteJson from "../../src/content/site.json";
+
+export const designIds = [
+  "design",
+  "classic",
+  "editorial",
+  "brutalist",
+  "cinematic",
+] as const;
+
+export type DesignId = (typeof designIds)[number];
+
+export const firstEnabledProject = projectsJson.items.find(
+  (project) => project.enabled !== false,
+);
+
+if (!firstEnabledProject) {
+  throw new Error("The portfolio needs at least one enabled project.");
+}
+
+const routeDefinitions = [
+  { path: "/", pageId: undefined },
+  { path: "/projects", pageId: "projects" },
+  { path: `/projects/${firstEnabledProject.id}`, pageId: "projects" },
+  { path: "/about", pageId: "about" },
+  { path: "/resume", pageId: "resume" },
+  { path: "/contact", pageId: "contact" },
+  { path: "/journey", pageId: "journey" },
+  { path: "/interview-map", pageId: "interviewMap" },
+] as const;
+
+export const enabledRoutes = routeDefinitions.filter(
+  ({ pageId }) => !pageId || siteJson.pages?.[pageId] !== false,
+);
+
+export function withExplicitDesign(path: string, designId: DesignId) {
+  const url = new URL(path, "https://portfolio.test");
+  url.searchParams.set("view", designId);
+  return `${url.pathname}${url.search}${url.hash}`;
+}


