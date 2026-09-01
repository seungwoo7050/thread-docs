## `feat(seo): 여정과 근거 route metadata 연결`

diff --git a/src/app/interview-map/page.tsx b/src/app/interview-map/page.tsx
index 4b855cf..895feee 100644
--- a/src/app/interview-map/page.tsx
+++ b/src/app/interview-map/page.tsx
@@ -1,3 +1,4 @@
+import type { Metadata } from "next";
 import Link from "next/link";
 import { notFound } from "next/navigation";
 import { ArrowRightIcon } from "@/components/icons";
@@ -15,6 +16,19 @@ import {
   type RouteSearchParams,
 } from "@/lib/portfolio";
 import { resolvePortfolioPageContext } from "@/lib/portfolio/page-context";
+import { createRouteMetadata } from "@/lib/site-metadata";
+
+export function generateMetadata(): Metadata {
+  const content = getPortfolioContent();
+  if (!isSitePageEnabled("interviewMap", content)) notFound();
+
+  return createRouteMetadata({
+    description: content.interviewMap.intro,
+    path: "/interview-map",
+    site: content.site,
+    title: content.presentation.pages.interviewMap.hero.title,
+  });
+}
 
 export default async function InterviewMapPage({
   searchParams,
diff --git a/src/app/journey/page.tsx b/src/app/journey/page.tsx
index edece0c..0f9753a 100644
--- a/src/app/journey/page.tsx
+++ b/src/app/journey/page.tsx
@@ -1,3 +1,4 @@
+import type { Metadata } from "next";
 import Link from "next/link";
 import { notFound } from "next/navigation";
 import { ArrowRightIcon } from "@/components/icons";
@@ -17,6 +18,19 @@ import {
   type RouteSearchParams,
 } from "@/lib/portfolio";
 import { resolvePortfolioPageContext } from "@/lib/portfolio/page-context";
+import { createRouteMetadata } from "@/lib/site-metadata";
+
+export function generateMetadata(): Metadata {
+  const content = getPortfolioContent();
+  if (!isSitePageEnabled("journey", content)) notFound();
+
+  return createRouteMetadata({
+    description: content.journeyNarrative.intro,
+    path: "/journey",
+    site: content.site,
+    title: content.presentation.pages.journey.hero.title,
+  });
+}
 
 export default async function JourneyPage({
   searchParams,


## `feat(seo): 프로젝트 상세에 JSON-LD 연결`

diff --git a/src/app/projects/[projectId]/page.tsx b/src/app/projects/[projectId]/page.tsx
index 2ed7fbc..35048e6 100644
--- a/src/app/projects/[projectId]/page.tsx
+++ b/src/app/projects/[projectId]/page.tsx
@@ -2,7 +2,12 @@ import type { Metadata } from "next";
 import { notFound } from "next/navigation";
 import { ProjectDetailView } from "@/components/portfolio/project-detail-view";
 import { PageShell } from "@/components/portfolio/site-shell";
+import { StructuredData } from "@/components/portfolio/structured-data";
 import { hasDedicatedRouteRenderer, renderDesignRoute } from "@/designs/registry";
+import {
+  resolvePortfolioContentMode,
+  resolveProductionSiteUrl,
+} from "@/lib/content-readiness";
 import {
   getPortfolioContent,
   isSitePageEnabled,
@@ -10,7 +15,10 @@ import {
   type RouteSearchParams,
 } from "@/lib/portfolio";
 import { resolvePortfolioPageContext } from "@/lib/portfolio/page-context";
-import { createRouteMetadata } from "@/lib/site-metadata";
+import {
+  createProjectStructuredData,
+  createRouteMetadata,
+} from "@/lib/site-metadata";
 
 type ProjectDetailPageProps = {
   params: Promise<{ projectId: string }>;
@@ -62,24 +70,44 @@ export default async function ProjectDetailPage({
     notFound();
   }
 
+  const mode = resolvePortfolioContentMode(
+    process.env.PORTFOLIO_CONTENT_MODE,
+  );
+  const structuredData =
+    mode === "production"
+      ? createProjectStructuredData({
+          content,
+          project,
+          siteUrl: resolveProductionSiteUrl(process.env.SITE_URL),
+        })
+      : undefined;
+
   if (hasDedicatedRouteRenderer(activeTemplate)) {
-    return renderDesignRoute(activeTemplate, {
-      content,
-      contentDebug,
-      currentPath: `/projects/${project.id}`,
-      project,
-      route: "project-detail",
-    });
+    return (
+      <>
+        {structuredData ? <StructuredData data={structuredData} /> : null}
+        {renderDesignRoute(activeTemplate, {
+          content,
+          contentDebug,
+          currentPath: `/projects/${project.id}`,
+          project,
+          route: "project-detail",
+        })}
+      </>
+    );
   }
 
   return (
-    <PageShell {...shellProps}>
-      <ProjectDetailView
-        contentDebug={contentDebug}
-        homeTemplate={activeTemplate}
-        pageCopy={content.presentation.pages.projectDetail}
-        project={project}
-      />
-    </PageShell>
+    <>
+      {structuredData ? <StructuredData data={structuredData} /> : null}
+      <PageShell {...shellProps}>
+        <ProjectDetailView
+          contentDebug={contentDebug}
+          homeTemplate={activeTemplate}
+          pageCopy={content.presentation.pages.projectDetail}
+          project={project}
+        />
+      </PageShell>
+    </>
   );
 }


## `feat(seo): 공개 route sitemap 생성`

diff --git a/src/app/sitemap.ts b/src/app/sitemap.ts
new file mode 100644
index 0000000..f989088
--- /dev/null
+++ b/src/app/sitemap.ts
@@ -0,0 +1,20 @@
+import type { MetadataRoute } from "next";
+
+import {
+  resolvePortfolioContentMode,
+  resolveProductionSiteUrl,
+} from "@/lib/content-readiness";
+import { getPortfolioContent } from "@/lib/portfolio";
+import { createSitemap } from "@/lib/site-metadata";
+
+export default function sitemap(): MetadataRoute.Sitemap {
+  const mode = resolvePortfolioContentMode(
+    process.env.PORTFOLIO_CONTENT_MODE,
+  );
+  const siteUrl =
+    mode === "production"
+      ? resolveProductionSiteUrl(process.env.SITE_URL)
+      : undefined;
+
+  return createSitemap({ content: getPortfolioContent(), mode, siteUrl });
+}
diff --git a/src/lib/site-metadata.test.ts b/src/lib/site-metadata.test.ts
index b9d8fb0..8818c3f 100644
--- a/src/lib/site-metadata.test.ts
+++ b/src/lib/site-metadata.test.ts
@@ -38,6 +38,7 @@ describe("site indexing metadata", () => {
     ).toEqual({
       host: "https://portfolio.example.dev",
       rules: { allow: "/", userAgent: "*" },
+      sitemap: "https://portfolio.example.dev/sitemap.xml",
     });
   });
 });
diff --git a/src/lib/site-metadata.ts b/src/lib/site-metadata.ts
index 6386988..93afa04 100644
--- a/src/lib/site-metadata.ts
+++ b/src/lib/site-metadata.ts
@@ -2,6 +2,7 @@ import type { Metadata, MetadataRoute } from "next";
 
 import type { PortfolioSource } from "./content-loader";
 import type { PortfolioContentMode } from "./content-readiness";
+import type { PortfolioContent } from "./portfolio";
 
 type SiteContent = PortfolioSource["site"];
 
@@ -13,6 +14,10 @@ type RouteMetadataInput = {
   type?: "article" | "website";
 };
 
+function absoluteSiteUrl(path: string, siteUrl: URL) {
+  return new URL(path, siteUrl.origin).toString();
+}
+
 function routeTitle(path: string, title: string, site: SiteContent) {
   return path === "/" ? site.title : `${title} | ${site.brand}`;
 }
@@ -103,5 +108,39 @@ export function createRobots({
   return {
     host: siteUrl.origin,
     rules: { allow: "/", userAgent: "*" },
+    sitemap: absoluteSiteUrl("/sitemap.xml", siteUrl),
   };
 }
+
+export function createSitemap({
+  content,
+  mode,
+  siteUrl,
+}: {
+  content: PortfolioContent;
+  mode: PortfolioContentMode;
+  siteUrl?: URL;
+}): MetadataRoute.Sitemap {
+  if (mode === "template") {
+    return [];
+  }
+
+  if (!siteUrl) {
+    throw new Error("A production site URL is required to create sitemap.xml.");
+  }
+
+  const routes = ["/"];
+  const pages = content.site.pages;
+
+  if (pages?.projects !== false) {
+    routes.push("/projects");
+    routes.push(...content.projects.map(({ id }) => `/projects/${id}`));
+  }
+  if (pages?.about !== false) routes.push("/about");
+  if (pages?.resume !== false) routes.push("/resume");
+  if (pages?.contact !== false) routes.push("/contact");
+  if (pages?.journey !== false) routes.push("/journey");
+  if (pages?.interviewMap !== false) routes.push("/interview-map");
+
+  return routes.map((path) => ({ url: absoluteSiteUrl(path, siteUrl) }));
+}


## `test(e2e): 콘텐츠 mode별 metadata와 robots 검증`

diff --git a/tests/e2e/portfolio.spec.ts b/tests/e2e/portfolio.spec.ts
index 27eca3f..bab285c 100644
--- a/tests/e2e/portfolio.spec.ts
+++ b/tests/e2e/portfolio.spec.ts
@@ -326,6 +326,28 @@ test("design switching preserves the current route and content debug query", asy
   expect(hydrationErrors).toEqual([]);
 });
 
+test("content mode controls page metadata and robots.txt", async ({
+  page,
+  request,
+}) => {
+  const isProductionContent =
+    process.env.PORTFOLIO_CONTENT_MODE === "production";
+  const response = await page.goto("/");
+  expect(response?.ok()).toBe(true);
+  await expect(page.locator('meta[name="robots"]')).toHaveAttribute(
+    "content",
+    isProductionContent ? /index.*follow/i : /noindex.*nofollow/i,
+  );
+
+  const robotsResponse = await request.get("/robots.txt");
+  expect(robotsResponse.ok()).toBe(true);
+  expect(await robotsResponse.text()).toMatch(
+    isProductionContent
+      ? /User-Agent:\s*\*[\s\S]*Allow:\s*\//i
+      : /User-Agent:\s*\*[\s\S]*Disallow:\s*\//i,
+  );
+});
+
 test("an invalid design query falls back to editorial", async ({ page }) => {
   const response = await page.goto("/?view=not-a-design");
 


## `test(seo): route metadata export 검증`

diff --git a/src/app/route-metadata.test.ts b/src/app/route-metadata.test.ts
new file mode 100644
index 0000000..d5c0f08
--- /dev/null
+++ b/src/app/route-metadata.test.ts
@@ -0,0 +1,57 @@
+import { describe, expect, it } from "vitest";
+
+import { getPortfolioContent } from "@/lib/portfolio";
+
+import { generateMetadata as getAboutMetadata } from "./about/page";
+import { generateMetadata as getContactMetadata } from "./contact/page";
+import { generateMetadata as getInterviewMapMetadata } from "./interview-map/page";
+import { generateMetadata as getJourneyMetadata } from "./journey/page";
+import { generateMetadata as getHomeMetadata } from "./page";
+import { generateMetadata as getProjectMetadata } from "./projects/[projectId]/page";
+import { generateMetadata as getProjectsMetadata } from "./projects/page";
+import { generateMetadata as getResumeMetadata } from "./resume/page";
+
+const content = getPortfolioContent();
+const project = content.projects[0];
+
+if (!project) {
+  throw new Error("Route metadata tests require at least one enabled project.");
+}
+
+describe("route metadata exports", () => {
+  it.each([
+    ["/", content.site.title, getHomeMetadata],
+    [
+      "/projects",
+      content.presentation.pages.projects.design.hero.title,
+      getProjectsMetadata,
+    ],
+    ["/about", content.presentation.pages.about.hero.title, getAboutMetadata],
+    ["/resume", content.presentation.pages.resume.hero.title, getResumeMetadata],
+    ["/contact", content.contact.title, getContactMetadata],
+    ["/journey", content.presentation.pages.journey.hero.title, getJourneyMetadata],
+    [
+      "/interview-map",
+      content.presentation.pages.interviewMap.hero.title,
+      getInterviewMapMetadata,
+    ],
+  ])("provides content metadata for %s", async (path, title, getMetadata) => {
+    const metadata = await getMetadata();
+
+    expect(metadata.alternates).toEqual({ canonical: path });
+    expect(String(metadata.title)).toContain(title);
+    expect(metadata.description).toBeTruthy();
+  });
+
+  it("uses project content for project detail metadata", async () => {
+    const metadata = await getProjectMetadata({
+      params: Promise.resolve({ projectId: project.id }),
+    });
+
+    expect(metadata.alternates).toEqual({
+      canonical: `/projects/${project.id}`,
+    });
+    expect(metadata.title).toBe(`${project.title} | ${content.site.brand}`);
+    expect(metadata.description).toBe(project.summary);
+  });
+});


## `test(seo): route metadata와 sitemap 계약 검증`

diff --git a/src/lib/site-metadata.test.ts b/src/lib/site-metadata.test.ts
index 8818c3f..c1fa4fe 100644
--- a/src/lib/site-metadata.test.ts
+++ b/src/lib/site-metadata.test.ts
@@ -1,7 +1,13 @@
 import siteJson from "@/content/site.json";
 import { describe, expect, it } from "vitest";
 
-import { createPortfolioMetadata, createRobots } from "./site-metadata";
+import { getPortfolioContent } from "./portfolio";
+import {
+  createPortfolioMetadata,
+  createRobots,
+  createRouteMetadata,
+  createSitemap,
+} from "./site-metadata";
 
 describe("site indexing metadata", () => {
   it("keeps template sites out of search results", () => {
@@ -42,3 +48,68 @@ describe("site indexing metadata", () => {
     });
   });
 });
+
+describe("route metadata", () => {
+  it("uses route content while keeping a query-free canonical path", () => {
+    const metadata = createRouteMetadata({
+      description: "A chronological view of the work.",
+      path: "/journey",
+      site: siteJson,
+      title: "Journey",
+    });
+
+    expect(metadata).toEqual(
+      expect.objectContaining({
+        alternates: { canonical: "/journey" },
+        description: "A chronological view of the work.",
+        title: "Journey | Your Name",
+      }),
+    );
+    expect(metadata.openGraph).toEqual(
+      expect.objectContaining({
+        description: "A chronological view of the work.",
+        title: "Journey | Your Name",
+        url: "/journey",
+      }),
+    );
+  });
+});
+
+describe("sitemap", () => {
+  it("does not publish template routes", () => {
+    expect(
+      createSitemap({
+        content: getPortfolioContent(),
+        mode: "template",
+      }),
+    ).toEqual([]);
+  });
+
+  it("lists enabled pages and project details from the production site URL", () => {
+    const content = getPortfolioContent();
+    if (!content.site.pages) {
+      throw new Error("Sitemap test requires explicit page availability.");
+    }
+    const sitemap = createSitemap({
+      content: {
+        ...content,
+        site: {
+          ...content.site,
+          pages: { ...content.site.pages, interviewMap: false },
+        },
+      },
+      mode: "production",
+      siteUrl: new URL("https://portfolio.example.dev"),
+    });
+
+    expect(sitemap.map(({ url }) => url)).toEqual([
+      "https://portfolio.example.dev/",
+      "https://portfolio.example.dev/projects",
+      `https://portfolio.example.dev/projects/${content.projects[0]?.id}`,
+      "https://portfolio.example.dev/about",
+      "https://portfolio.example.dev/resume",
+      "https://portfolio.example.dev/contact",
+      "https://portfolio.example.dev/journey",
+    ]);
+  });
+});


## `test(seo): JSON-LD 계약과 직렬화 검증`

diff --git a/src/lib/site-metadata.test.ts b/src/lib/site-metadata.test.ts
index c1fa4fe..bcde0d6 100644
--- a/src/lib/site-metadata.test.ts
+++ b/src/lib/site-metadata.test.ts
@@ -4,9 +4,12 @@ import { describe, expect, it } from "vitest";
 import { getPortfolioContent } from "./portfolio";
 import {
   createPortfolioMetadata,
+  createProjectStructuredData,
   createRobots,
   createRouteMetadata,
+  createSiteStructuredData,
   createSitemap,
+  serializeStructuredData,
 } from "./site-metadata";
 
 describe("site indexing metadata", () => {
@@ -113,3 +116,58 @@ describe("sitemap", () => {
     ]);
   });
 });
+
+describe("structured data", () => {
+  it("builds Person and WebSite records only from validated content", () => {
+    const content = getPortfolioContent();
+    const structuredData = createSiteStructuredData({
+      content,
+      siteUrl: new URL("https://portfolio.example.dev"),
+    });
+
+    expect(structuredData["@graph"]).toEqual([
+      expect.objectContaining({
+        "@id": "https://portfolio.example.dev/#person",
+        "@type": "Person",
+        description: content.profile.summary,
+        name: content.profile.name,
+      }),
+      expect.objectContaining({
+        "@id": "https://portfolio.example.dev/#website",
+        "@type": "WebSite",
+        description: content.site.description,
+        name: content.site.brand,
+      }),
+    ]);
+  });
+
+  it("builds a project CreativeWork record without adding unsupported claims", () => {
+    const content = getPortfolioContent();
+    const project = content.projects[0];
+    if (!project) throw new Error("Structured data test requires a project.");
+
+    const structuredData = createProjectStructuredData({
+      content,
+      project,
+      siteUrl: new URL("https://portfolio.example.dev"),
+    });
+
+    expect(structuredData).toEqual(
+      expect.objectContaining({
+        "@id": `https://portfolio.example.dev/projects/${project.id}#creative-work`,
+        "@type": "CreativeWork",
+        description: project.summary,
+        name: project.title,
+        url: `https://portfolio.example.dev/projects/${project.id}`,
+      }),
+    );
+    expect(structuredData).not.toHaveProperty("award");
+    expect(structuredData).not.toHaveProperty("aggregateRating");
+  });
+
+  it("escapes markup-significant characters before embedding JSON-LD", () => {
+    expect(serializeStructuredData({ value: "</script>" })).toContain(
+      "\\u003c/script\\u003e",
+    );
+  });
+});
