# Template·Production 콘텐츠 준비도 정책

## `feat(content): 콘텐츠 mode와 readiness 오류 모델 추가`

diff --git a/src/lib/content-readiness.ts b/src/lib/content-readiness.ts
new file mode 100644
index 0000000..75fab85
--- /dev/null
+++ b/src/lib/content-readiness.ts
@@ -0,0 +1,46 @@
+export type PortfolioContentMode = "template" | "production";
+
+export type PortfolioReadinessEnvironment = {
+  PORTFOLIO_CONTENT_MODE?: string;
+  SITE_URL?: string;
+};
+
+export type PortfolioReadinessIssue = {
+  file: string;
+  path: string;
+  message: string;
+};
+
+export type PortfolioReadinessResult =
+  | { mode: "template"; siteUrl: undefined }
+  | { mode: "production"; siteUrl: URL };
+
+export class PortfolioReadinessError extends Error {
+  readonly issues: PortfolioReadinessIssue[];
+
+  constructor(issues: PortfolioReadinessIssue[]) {
+    const details = issues
+      .map(({ file, path, message }) => `- ${file}:${path} ${message}`)
+      .join("\n");
+
+    super(`Portfolio production readiness failed:\n${details}`);
+    this.name = "PortfolioReadinessError";
+    this.issues = issues;
+  }
+}
+
+export function resolvePortfolioContentMode(
+  value: string | undefined,
+): PortfolioContentMode {
+  if (value === undefined || value === "" || value === "template") {
+    return "template";
+  }
+
+  if (value === "production") {
+    return "production";
+  }
+
+  throw new Error(
+    `PORTFOLIO_CONTENT_MODE must be "template" or "production"; received ${JSON.stringify(value)}.`,
+  );
+}


## `feat(content): public origin과 자산 경계 검증 추가`

diff --git a/src/lib/content-readiness.ts b/src/lib/content-readiness.ts
index 88d5a92..f7f667b 100644
--- a/src/lib/content-readiness.ts
+++ b/src/lib/content-readiness.ts
@@ -126,3 +126,91 @@ export function collectPlaceholderIssues(
     );
   }
 }
+
+export function addProductionAssetIssue(
+  issues: PortfolioReadinessIssue[],
+  file: string,
+  path: string,
+  assetPath: string,
+) {
+  if (!assetPath.startsWith("/content/")) {
+    issues.push({
+      file,
+      path,
+      message: `Use a production asset under public/content instead of "${assetPath}".`,
+    });
+  }
+}
+
+export function isReservedHostname(hostname: string) {
+  return (
+    ["example.com", "example.net", "example.org"].some(
+      (domain) => hostname === domain || hostname.endsWith(`.${domain}`),
+    ) ||
+    [".example", ".invalid", ".test"].some((suffix) =>
+      hostname.endsWith(suffix),
+    )
+  );
+}
+
+export function parsePublicSiteUrl(
+  value: string | undefined,
+  issues: PortfolioReadinessIssue[],
+) {
+  if (!value) {
+    issues.push({
+      file: "environment",
+      path: "SITE_URL",
+      message: "SITE_URL is required in production content mode.",
+    });
+    return undefined;
+  }
+
+  let siteUrl: URL;
+  try {
+    siteUrl = new URL(value);
+  } catch {
+    issues.push({
+      file: "environment",
+      path: "SITE_URL",
+      message: "SITE_URL must be an absolute http(s) URL.",
+    });
+    return undefined;
+  }
+
+  const hostname = siteUrl.hostname.toLowerCase();
+  const isLocal =
+    hostname === "localhost" ||
+    hostname === "127.0.0.1" ||
+    hostname === "::1" ||
+    hostname.endsWith(".localhost");
+
+  if (
+    !["http:", "https:"].includes(siteUrl.protocol) ||
+    isLocal ||
+    isReservedHostname(hostname) ||
+    siteUrl.username !== "" ||
+    siteUrl.password !== ""
+  ) {
+    issues.push({
+      file: "environment",
+      path: "SITE_URL",
+      message:
+        "SITE_URL must use a real public http(s) origin, not a local or placeholder address.",
+    });
+    return undefined;
+  }
+
+  return siteUrl;
+}
+
+export function resolveProductionSiteUrl(value: string | undefined) {
+  const issues: PortfolioReadinessIssue[] = [];
+  const siteUrl = parsePublicSiteUrl(value, issues);
+
+  if (!siteUrl || issues.length > 0) {
+    throw new PortfolioReadinessError(issues);
+  }
+
+  return siteUrl;
+}


## `feat(content): 공개 URL과 연락 링크 검증 추가`

diff --git a/src/lib/content-readiness.ts b/src/lib/content-readiness.ts
index f7f667b..03a070f 100644
--- a/src/lib/content-readiness.ts
+++ b/src/lib/content-readiness.ts
@@ -214,3 +214,33 @@ export function resolveProductionSiteUrl(value: string | undefined) {
 
   return siteUrl;
 }
+
+export function isUsablePublicUrl(href: string) {
+  if (findPlaceholderMarker(href)) {
+    return false;
+  }
+
+  try {
+    const url = new URL(href);
+    const hostname = url.hostname.toLowerCase();
+
+    return (
+      (url.protocol === "http:" || url.protocol === "https:") &&
+      !isReservedHostname(hostname)
+    );
+  } catch {
+    return false;
+  }
+}
+
+export function isUsableContactHref(href: string) {
+  if (findPlaceholderMarker(href)) {
+    return false;
+  }
+
+  return (
+    href.startsWith("mailto:") ||
+    href.startsWith("tel:") ||
+    isUsablePublicUrl(href)
+  );
+}


## `feat(content): template placeholder 탐색 경계 추가`

diff --git a/src/lib/content-readiness.ts b/src/lib/content-readiness.ts
index 75fab85..88d5a92 100644
--- a/src/lib/content-readiness.ts
+++ b/src/lib/content-readiness.ts
@@ -1,3 +1,5 @@
+import type { PortfolioSource } from "./content-loader";
+
 export type PortfolioContentMode = "template" | "production";
 
 export type PortfolioReadinessEnvironment = {
@@ -15,6 +17,40 @@ export type PortfolioReadinessResult =
   | { mode: "template"; siteUrl: undefined }
   | { mode: "production"; siteUrl: URL };
 
+export const contentFiles = [
+  ["site", "src/content/site.json"],
+  ["profile", "src/content/profile.json"],
+  ["projects", "src/content/projects.json"],
+  ["presentation", "src/content/presentation.json"],
+  ["skills", "src/content/skills.json"],
+  ["techStack", "src/content/tech-stack.json"],
+  ["experience", "src/content/experience.json"],
+  ["journey", "src/content/journey.json"],
+  ["journeyNarrative", "src/content/journey-narrative.json"],
+  ["interviewMap", "src/content/interview-map.json"],
+  ["curation", "src/content/curation.json"],
+  ["links", "src/content/links.json"],
+  ["contact", "src/content/contact.json"],
+  ["resume", "src/content/resume.json"],
+] as const satisfies ReadonlyArray<
+  readonly [keyof PortfolioSource, `src/content/${string}.json`]
+>;
+
+export const placeholderMarkers = [
+  { label: "Your Name", pattern: /\byour name\b/i },
+  { label: "your-handle", pattern: /\byour-handle\b/i },
+  { label: "Your City", pattern: /\byour city\b/i },
+  {
+    label: "Your Program or Practice",
+    pattern: /\byour program(?: or practice)?\b/i,
+  },
+  { label: "hello@example.com", pattern: /\bhello@example\.com\b/i },
+  { label: "Example Project", pattern: /\bexample[- ]project\b/i },
+  { label: "placeholder", pattern: /\bplaceholder\b/i },
+  { label: "Replace this/the", pattern: /\breplace (?:this|the)\b/i },
+  { label: "starter", pattern: /\bstarter\b/i },
+] as const;
+
 export class PortfolioReadinessError extends Error {
   readonly issues: PortfolioReadinessIssue[];
 
@@ -44,3 +80,49 @@ export function resolvePortfolioContentMode(
     `PORTFOLIO_CONTENT_MODE must be "template" or "production"; received ${JSON.stringify(value)}.`,
   );
 }
+
+export function appendPath(path: string, key: string | number) {
+  if (typeof key === "number") {
+    return `${path}[${key}]`;
+  }
+
+  return /^[a-zA-Z_$][\w$]*$/.test(key)
+    ? `${path}.${key}`
+    : `${path}[${JSON.stringify(key)}]`;
+}
+
+export function findPlaceholderMarker(value: string) {
+  return placeholderMarkers.find(({ pattern }) => pattern.test(value));
+}
+
+export function collectPlaceholderIssues(
+  value: unknown,
+  file: string,
+  path: string,
+  issues: PortfolioReadinessIssue[],
+) {
+  if (typeof value === "string") {
+    const marker = findPlaceholderMarker(value);
+    if (marker) {
+      issues.push({
+        file,
+        path,
+        message: `Replace the template marker "${marker.label}" with production content.`,
+      });
+    }
+    return;
+  }
+
+  if (Array.isArray(value)) {
+    value.forEach((item, index) =>
+      collectPlaceholderIssues(item, file, appendPath(path, index), issues),
+    );
+    return;
+  }
+
+  if (value && typeof value === "object") {
+    Object.entries(value).forEach(([key, item]) =>
+      collectPlaceholderIssues(item, file, appendPath(path, key), issues),
+    );
+  }
+}


## `feat(content): production readiness 기본 검사 추가`

diff --git a/src/lib/content-readiness.ts b/src/lib/content-readiness.ts
index 03a070f..f32e37e 100644
--- a/src/lib/content-readiness.ts
+++ b/src/lib/content-readiness.ts
@@ -17,6 +17,11 @@ export type PortfolioReadinessResult =
   | { mode: "template"; siteUrl: undefined }
   | { mode: "production"; siteUrl: URL };
 
+type ProductionReadinessResult = Extract<
+  PortfolioReadinessResult,
+  { mode: "production" }
+>;
+
 export const contentFiles = [
   ["site", "src/content/site.json"],
   ["profile", "src/content/profile.json"],
@@ -244,3 +249,21 @@ export function isUsableContactHref(href: string) {
     isUsablePublicUrl(href)
   );
 }
+
+export function validateProductionReadiness(
+  content: PortfolioSource,
+  environment: Pick<PortfolioReadinessEnvironment, "SITE_URL">,
+): ProductionReadinessResult {
+  const issues: PortfolioReadinessIssue[] = [];
+  const siteUrl = parsePublicSiteUrl(environment.SITE_URL, issues);
+
+  for (const [key, file] of contentFiles) {
+    collectPlaceholderIssues(content[key], file, "$", issues);
+  }
+
+  if (!siteUrl || issues.length > 0) {
+    throw new PortfolioReadinessError(issues);
+  }
+
+  return { mode: "production", siteUrl };
+}


## `feat(content): 필수 자산과 프로젝트 readiness 추가`

diff --git a/src/lib/content-readiness.ts b/src/lib/content-readiness.ts
index f32e37e..778978c 100644
--- a/src/lib/content-readiness.ts
+++ b/src/lib/content-readiness.ts
@@ -261,6 +261,95 @@ export function validateProductionReadiness(
     collectPlaceholderIssues(content[key], file, "$", issues);
   }
 
+  if (!content.site.socialImage) {
+    issues.push({
+      file: "src/content/site.json",
+      path: "$.socialImage",
+      message: "Add a social image under public/content for production.",
+    });
+  } else {
+    addProductionAssetIssue(
+      issues,
+      "src/content/site.json",
+      "$.socialImage",
+      content.site.socialImage,
+    );
+  }
+
+  if (!content.profile.photo) {
+    issues.push({
+      file: "src/content/profile.json",
+      path: "$.photo",
+      message: "Add a profile image under public/content for production.",
+    });
+  } else {
+    addProductionAssetIssue(
+      issues,
+      "src/content/profile.json",
+      "$.photo.src",
+      content.profile.photo.src,
+    );
+  }
+
+  if (!content.resume.downloadUrl) {
+    issues.push({
+      file: "src/content/resume.json",
+      path: "$.downloadUrl",
+      message: "Add a downloadable resume under public/content for production.",
+    });
+  } else {
+    addProductionAssetIssue(
+      issues,
+      "src/content/resume.json",
+      "$.downloadUrl",
+      content.resume.downloadUrl,
+    );
+  }
+
+  const enabledProjects = content.projects.items.filter(
+    (project) => project.enabled !== false,
+  );
+
+  if (enabledProjects.length === 0) {
+    issues.push({
+      file: "src/content/projects.json",
+      path: "$.items",
+      message: "Enable at least one real project for production.",
+    });
+  }
+
+  content.projects.items.forEach((project, projectIndex) => {
+    if (project.enabled === false) {
+      return;
+    }
+
+    addProductionAssetIssue(
+      issues,
+      "src/content/projects.json",
+      `$.items[${projectIndex}].screenshot.src`,
+      project.screenshot.src,
+    );
+    project.screenshots.forEach((screenshot, screenshotIndex) =>
+      addProductionAssetIssue(
+        issues,
+        "src/content/projects.json",
+        `$.items[${projectIndex}].screenshots[${screenshotIndex}].src`,
+        screenshot.src,
+      ),
+    );
+
+    if (
+      !project.links.some(
+        (link) => link.enabled !== false && isUsablePublicUrl(link.href),
+      )
+    ) {
+      issues.push({
+        file: "src/content/projects.json",
+        path: `$.items[${projectIndex}].links`,
+        message: "Add at least one enabled public project URL for production.",
+      });
+    }
+  });
   if (!siteUrl || issues.length > 0) {
     throw new PortfolioReadinessError(issues);
   }


## `feat(content): 연락 수단과 build readiness 연결`

diff --git a/src/lib/content-readiness.ts b/src/lib/content-readiness.ts
index 778978c..5f92c5c 100644
--- a/src/lib/content-readiness.ts
+++ b/src/lib/content-readiness.ts
@@ -22,7 +22,7 @@ type ProductionReadinessResult = Extract<
   { mode: "production" }
 >;
 
-export const contentFiles = [
+const contentFiles = [
   ["site", "src/content/site.json"],
   ["profile", "src/content/profile.json"],
   ["projects", "src/content/projects.json"],
@@ -41,7 +41,7 @@ export const contentFiles = [
   readonly [keyof PortfolioSource, `src/content/${string}.json`]
 >;
 
-export const placeholderMarkers = [
+const placeholderMarkers = [
   { label: "Your Name", pattern: /\byour name\b/i },
   { label: "your-handle", pattern: /\byour-handle\b/i },
   { label: "Your City", pattern: /\byour city\b/i },
@@ -86,7 +86,7 @@ export function resolvePortfolioContentMode(
   );
 }
 
-export function appendPath(path: string, key: string | number) {
+function appendPath(path: string, key: string | number) {
   if (typeof key === "number") {
     return `${path}[${key}]`;
   }
@@ -96,11 +96,11 @@ export function appendPath(path: string, key: string | number) {
     : `${path}[${JSON.stringify(key)}]`;
 }
 
-export function findPlaceholderMarker(value: string) {
+function findPlaceholderMarker(value: string) {
   return placeholderMarkers.find(({ pattern }) => pattern.test(value));
 }
 
-export function collectPlaceholderIssues(
+function collectPlaceholderIssues(
   value: unknown,
   file: string,
   path: string,
@@ -132,7 +132,7 @@ export function collectPlaceholderIssues(
   }
 }
 
-export function addProductionAssetIssue(
+function addProductionAssetIssue(
   issues: PortfolioReadinessIssue[],
   file: string,
   path: string,
@@ -147,7 +147,7 @@ export function addProductionAssetIssue(
   }
 }
 
-export function isReservedHostname(hostname: string) {
+function isReservedHostname(hostname: string) {
   return (
     ["example.com", "example.net", "example.org"].some(
       (domain) => hostname === domain || hostname.endsWith(`.${domain}`),
@@ -158,7 +158,7 @@ export function isReservedHostname(hostname: string) {
   );
 }
 
-export function parsePublicSiteUrl(
+function parsePublicSiteUrl(
   value: string | undefined,
   issues: PortfolioReadinessIssue[],
 ) {
@@ -220,7 +220,7 @@ export function resolveProductionSiteUrl(value: string | undefined) {
   return siteUrl;
 }
 
-export function isUsablePublicUrl(href: string) {
+function isUsablePublicUrl(href: string) {
   if (findPlaceholderMarker(href)) {
     return false;
   }
@@ -238,7 +238,7 @@ export function isUsablePublicUrl(href: string) {
   }
 }
 
-export function isUsableContactHref(href: string) {
+function isUsableContactHref(href: string) {
   if (findPlaceholderMarker(href)) {
     return false;
   }
@@ -350,9 +350,41 @@ export function validateProductionReadiness(
       });
     }
   });
+
+  const hasContactMethod = content.links.some(
+    (link) =>
+      link.enabled !== false &&
+      link.placements?.includes("contact") &&
+      ["email", "github", "website"].includes(link.type) &&
+      isUsableContactHref(link.href),
+  );
+
+  if (!hasContactMethod) {
+    issues.push({
+      file: "src/content/links.json",
+      path: "$",
+      message: "Enable at least one non-placeholder contact method for production.",
+    });
+  }
+
   if (!siteUrl || issues.length > 0) {
     throw new PortfolioReadinessError(issues);
   }
 
   return { mode: "production", siteUrl };
 }
+
+export function validateBuildReadiness(
+  content: PortfolioSource,
+  environment: PortfolioReadinessEnvironment,
+): PortfolioReadinessResult {
+  const mode = resolvePortfolioContentMode(
+    environment.PORTFOLIO_CONTENT_MODE,
+  );
+
+  if (mode === "template") {
+    return { mode, siteUrl: undefined };
+  }
+
+  return validateProductionReadiness(content, environment);
+}


## `build(content): readiness 검사를 prebuild에 연결`

diff --git a/package.json b/package.json
index f600b81..7478f7c 100644
--- a/package.json
+++ b/package.json
@@ -9,13 +9,14 @@
   },
   "scripts": {
     "dev": "next dev --webpack -p 3100",
-    "prebuild": "npm run content:check",
+    "prebuild": "npm run content:check && npm run content:ready",
     "build": "next build",
     "start": "next start -p 3100",
     "start:e2e": "next start -p 3200",
     "lint": "eslint .",
     "typecheck": "tsc --noEmit",
     "content:check": "node --import tsx scripts/validate-content.ts",
+    "content:ready": "node --import tsx scripts/validate-content-readiness.ts",
     "test": "vitest run",
     "test:watch": "vitest",
     "test:e2e": "playwright test",
diff --git a/scripts/validate-content-readiness.ts b/scripts/validate-content-readiness.ts
new file mode 100644
index 0000000..1461313
--- /dev/null
+++ b/scripts/validate-content-readiness.ts
@@ -0,0 +1,27 @@
+import {
+  PortfolioReadinessError,
+  validateBuildReadiness,
+} from "../src/lib/content-readiness";
+import { loadPortfolioSource } from "../src/lib/content-loader";
+
+try {
+  const result = validateBuildReadiness(loadPortfolioSource(), {
+    PORTFOLIO_CONTENT_MODE: process.env.PORTFOLIO_CONTENT_MODE,
+    SITE_URL: process.env.SITE_URL,
+  });
+
+  if (result.mode === "template") {
+    console.log(
+      "Content mode: template. Production readiness is skipped and indexing remains disabled.",
+    );
+  } else {
+    console.log(`Production readiness valid for ${result.siteUrl.origin}.`);
+  }
+} catch (error) {
+  if (error instanceof PortfolioReadinessError) {
+    console.error(error.message);
+    process.exitCode = 1;
+  } else {
+    throw error;
+  }
+}


## `test(content): readiness와 indexing 계약 검증`

diff --git a/src/lib/content-readiness.test.ts b/src/lib/content-readiness.test.ts
new file mode 100644
index 0000000..d84b9c1
--- /dev/null
+++ b/src/lib/content-readiness.test.ts
@@ -0,0 +1,153 @@
+import { loadPortfolioSource } from "@/lib/content-loader";
+import { describe, expect, it } from "vitest";
+
+import {
+  PortfolioReadinessError,
+  resolvePortfolioContentMode,
+  validateBuildReadiness,
+  validateProductionReadiness,
+} from "./content-readiness";
+
+function captureReadinessError(run: () => unknown) {
+  let caught: unknown;
+
+  try {
+    run();
+  } catch (error) {
+    caught = error;
+  }
+
+  expect(caught).toBeInstanceOf(PortfolioReadinessError);
+  return caught as PortfolioReadinessError;
+}
+
+function replaceTemplateMarkers(value: unknown): unknown {
+  if (typeof value === "string") {
+    return value
+      .replaceAll("Your Name", "Portfolio Owner")
+      .replaceAll("your-handle", "portfolio-owner")
+      .replaceAll("Your City", "Seoul")
+      .replaceAll("Your Program or Practice", "Software Engineering Program")
+      .replaceAll("hello@example.com", "owner@portfolio.dev")
+      .replaceAll("Example Project", "Realtime Collaboration Project")
+      .replaceAll("example-project", "realtime-collaboration")
+      .replaceAll("placeholder", "preview")
+      .replaceAll("Placeholder", "Preview")
+      .replaceAll("starter", "published")
+      .replaceAll("Replace this", "This section contains")
+      .replaceAll("Replace the", "Update the")
+      .replace("/template/", "/content/");
+  }
+
+  if (Array.isArray(value)) {
+    return value.map(replaceTemplateMarkers);
+  }
+
+  if (value && typeof value === "object") {
+    return Object.fromEntries(
+      Object.entries(value).map(([key, item]) => [
+        key,
+        replaceTemplateMarkers(item),
+      ]),
+    );
+  }
+
+  return value;
+}
+
+function createProductionReadyContent() {
+  const content = replaceTemplateMarkers(
+    structuredClone(loadPortfolioSource()),
+  ) as ReturnType<typeof loadPortfolioSource>;
+
+  content.site.socialImage = "/content/social-card.png";
+  content.profile.photo = {
+    src: "/content/profile/portrait.png",
+    alt: "Portfolio Owner portrait",
+  };
+  content.resume.downloadUrl = "/content/resume/portfolio-owner.pdf";
+  content.projects.items[0].links[0] = {
+    ...content.projects.items[0].links[0],
+    enabled: true,
+    href: "https://github.com/portfolio-owner/realtime-collaboration",
+    label: "Source",
+  };
+
+  return content;
+}
+
+describe("portfolio production readiness", () => {
+  it("defaults to template mode and rejects unsupported values", () => {
+    expect(resolvePortfolioContentMode(undefined)).toBe("template");
+    expect(resolvePortfolioContentMode("template")).toBe("template");
+    expect(resolvePortfolioContentMode("production")).toBe("production");
+
+    expect(() => resolvePortfolioContentMode("preview")).toThrow(
+      /PORTFOLIO_CONTENT_MODE.*template.*production/,
+    );
+  });
+
+  it("allows the checked-in placeholders in template mode", () => {
+    const result = validateBuildReadiness(loadPortfolioSource(), {
+      PORTFOLIO_CONTENT_MODE: "template",
+    });
+
+    expect(result).toEqual({ mode: "template", siteUrl: undefined });
+  });
+
+  it("reports every production input category without masking later issues", () => {
+    const error = captureReadinessError(() =>
+      validateBuildReadiness(loadPortfolioSource(), {
+        PORTFOLIO_CONTENT_MODE: "production",
+      }),
+    );
+
+    expect(error.message).toContain("Portfolio production readiness failed");
+    expect(error.issues).toEqual(
+      expect.arrayContaining([
+        expect.objectContaining({ path: "SITE_URL" }),
+        expect.objectContaining({ path: "$.socialImage" }),
+        expect.objectContaining({ path: "$.name" }),
+        expect.objectContaining({ path: "$.photo.src" }),
+        expect.objectContaining({ path: "$[0].href" }),
+        expect.objectContaining({ path: "$.downloadUrl" }),
+        expect.objectContaining({
+          path: "$.items[0].screenshot.src",
+        }),
+        expect.objectContaining({ path: "$.items[0].links" }),
+      ]),
+    );
+  });
+
+  it("accepts complete content and a valid public site URL", () => {
+    const content = createProductionReadyContent();
+
+    expect(
+      validateProductionReadiness(content, {
+        SITE_URL: "https://portfolio.example.dev",
+      }),
+    ).toEqual({
+      mode: "production",
+      siteUrl: new URL("https://portfolio.example.dev"),
+    });
+  });
+
+  it("rejects local and malformed production site URLs", () => {
+    for (const siteUrl of [
+      "not-a-url",
+      "ftp://portfolio.example.dev",
+      "http://localhost:3100",
+      "https://example.com",
+    ]) {
+      const error = captureReadinessError(() =>
+        validateProductionReadiness(createProductionReadyContent(), {
+          SITE_URL: siteUrl,
+        }),
+      );
+
+      expect(error.issues).toEqual(
+        expect.arrayContaining([expect.objectContaining({ path: "SITE_URL" })]),
+      );
+    }
+  });
+});
diff --git a/src/lib/site-metadata.test.ts b/src/lib/site-metadata.test.ts
new file mode 100644
index 0000000..971e956
--- /dev/null
+++ b/src/lib/site-metadata.test.ts
@@ -0,0 +1,43 @@
+import siteJson from "@/content/site.json";
+import { describe, expect, it } from "vitest";
+
+import { createPortfolioMetadata, createRobots } from "./site-metadata";
+
+describe("site indexing metadata", () => {
+  it("keeps template sites out of search results", () => {
+    const metadata = createPortfolioMetadata({
+      mode: "template",
+      metadataBase: new URL("http://localhost:3100"),
+      site: siteJson,
+    });
+
+    expect(metadata.robots).toEqual({ follow: false, index: false });
+    expect(createRobots({ mode: "template" })).toEqual({
+      rules: { disallow: "/", userAgent: "*" },
+    });
+  });
+
+  it("indexes a production site from its configured canonical origin", () => {
+    const metadataBase = new URL("https://portfolio.example.dev");
+    const metadata = createPortfolioMetadata({
+      mode: "production",
+      metadataBase,
+      site: { ...siteJson, socialImage: "/content/social-card.png" },
+    });
+
+    expect(metadata.metadataBase).toEqual(metadataBase);
+    expect(metadata.alternates).toEqual({ canonical: "./" });
+    expect(metadata.robots).toEqual({ follow: true, index: true });
+    expect(metadata.openGraph).toEqual(
+      expect.objectContaining({
+        images: [{ url: "https://portfolio.example.dev/content/social-card.png" }],
+      }),
+    );
+    expect(
+      createRobots({ mode: "production", siteUrl: metadataBase }),
+    ).toEqual({
+      host: "https://portfolio.example.dev",
+      rules: { allow: "/", userAgent: "*" },
+    });
+  });
+});
