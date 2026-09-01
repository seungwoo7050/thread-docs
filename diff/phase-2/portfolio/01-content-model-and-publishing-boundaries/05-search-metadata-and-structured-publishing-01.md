# 검색 메타데이터와 구조화 게시

## `feat(seo): 콘텐츠 mode별 metadata 정책 추가`

diff --git a/src/lib/site-metadata.ts b/src/lib/site-metadata.ts
new file mode 100644
index 0000000..4b69d34
--- /dev/null
+++ b/src/lib/site-metadata.ts
@@ -0,0 +1,41 @@
+import type { Metadata } from "next";
+
+import type { PortfolioSource } from "./content-loader";
+import type { PortfolioContentMode } from "./content-readiness";
+
+type SiteContent = PortfolioSource["site"];
+
+export function createPortfolioMetadata({
+  metadataBase,
+  mode,
+  site,
+}: {
+  metadataBase: URL;
+  mode: PortfolioContentMode;
+  site: SiteContent;
+}): Metadata {
+  const socialImage = site.socialImage
+    ? new URL(site.socialImage, metadataBase).toString()
+    : undefined;
+  const shouldIndex = mode === "production";
+
+  return {
+    alternates: { canonical: "./" },
+    description: site.description,
+    metadataBase,
+    openGraph: {
+      description: site.description,
+      images: socialImage ? [{ url: socialImage }] : undefined,
+      title: site.title,
+      type: "website",
+    },
+    robots: { follow: shouldIndex, index: shouldIndex },
+    title: site.title,
+    twitter: {
+      card: "summary_large_image",
+      description: site.description,
+      images: socialImage ? [socialImage] : undefined,
+      title: site.title,
+    },
+  };
+}


## `feat(seo): 콘텐츠 mode별 robots 정책 추가`

diff --git a/src/app/robots.ts b/src/app/robots.ts
new file mode 100644
index 0000000..4279753
--- /dev/null
+++ b/src/app/robots.ts
@@ -0,0 +1,19 @@
+import type { MetadataRoute } from "next";
+
+import {
+  resolvePortfolioContentMode,
+  resolveProductionSiteUrl,
+} from "@/lib/content-readiness";
+import { createRobots } from "@/lib/site-metadata";
+
+export default function robots(): MetadataRoute.Robots {
+  const mode = resolvePortfolioContentMode(
+    process.env.PORTFOLIO_CONTENT_MODE,
+  );
+  const siteUrl =
+    mode === "production"
+      ? resolveProductionSiteUrl(process.env.SITE_URL)
+      : undefined;
+
+  return createRobots({ mode, siteUrl });
+}
diff --git a/src/lib/site-metadata.ts b/src/lib/site-metadata.ts
index 4b69d34..751b958 100644
--- a/src/lib/site-metadata.ts
+++ b/src/lib/site-metadata.ts
@@ -1,4 +1,4 @@
-import type { Metadata } from "next";
+import type { Metadata, MetadataRoute } from "next";
 
 import type { PortfolioSource } from "./content-loader";
 import type { PortfolioContentMode } from "./content-readiness";
@@ -39,3 +39,24 @@ export function createPortfolioMetadata({
     },
   };
 }
+
+export function createRobots({
+  mode,
+  siteUrl,
+}: {
+  mode: PortfolioContentMode;
+  siteUrl?: URL;
+}): MetadataRoute.Robots {
+  if (mode === "template") {
+    return { rules: { disallow: "/", userAgent: "*" } };
+  }
+
+  if (!siteUrl) {
+    throw new Error("A production site URL is required to create robots.txt.");
+  }
+
+  return {
+    host: siteUrl.origin,
+    rules: { allow: "/", userAgent: "*" },
+  };
+}


## `feat(seo): layout metadata를 콘텐츠 mode에 연결`

diff --git a/src/app/layout.tsx b/src/app/layout.tsx
index 1a1be5b..426d57a 100644
--- a/src/app/layout.tsx
+++ b/src/app/layout.tsx
@@ -1,7 +1,12 @@
 import type { Metadata } from "next";
 import { Geist, Geist_Mono, Noto_Serif_KR } from "next/font/google";
 import { headers } from "next/headers";
+import {
+  resolvePortfolioContentMode,
+  resolveProductionSiteUrl,
+} from "@/lib/content-readiness";
 import { getPortfolioContent } from "@/lib/portfolio";
+import { createPortfolioMetadata } from "@/lib/site-metadata";
 import "./globals.css";
 
 const geistSans = Geist({
@@ -24,34 +29,28 @@ const notoSerif = Noto_Serif_KR({
 const { site } = getPortfolioContent();
 
 export async function generateMetadata(): Promise<Metadata> {
-  const requestHeaders = await headers();
-  const host = requestHeaders.get("x-forwarded-host") ?? requestHeaders.get("host") ?? "localhost:3100";
-  const protocol =
-    requestHeaders.get("x-forwarded-proto") ??
-    (host.startsWith("localhost") || host.startsWith("127.0.0.1") ? "http" : "https");
-  const metadataBase = new URL(`${protocol}://${host}`);
-  const socialImage = site.socialImage
-    ? new URL(site.socialImage, metadataBase).toString()
-    : undefined;
+  const mode = resolvePortfolioContentMode(
+    process.env.PORTFOLIO_CONTENT_MODE,
+  );
+  let metadataBase: URL;
+
+  if (mode === "production") {
+    metadataBase = resolveProductionSiteUrl(process.env.SITE_URL);
+  } else {
+    const requestHeaders = await headers();
+    const host =
+      requestHeaders.get("x-forwarded-host") ??
+      requestHeaders.get("host") ??
+      "localhost:3100";
+    const protocol =
+      requestHeaders.get("x-forwarded-proto") ??
+      (host.startsWith("localhost") || host.startsWith("127.0.0.1")
+        ? "http"
+        : "https");
+    metadataBase = new URL(`${protocol}://${host}`);
+  }
 
-  return {
-    alternates: { canonical: "./" },
-    description: site.description,
-    metadataBase,
-    openGraph: {
-      description: site.description,
-      images: socialImage ? [{ url: socialImage }] : undefined,
-      title: site.title,
-      type: "website",
-    },
-    title: site.title,
-    twitter: {
-      card: "summary_large_image",
-      description: site.description,
-      images: socialImage ? [socialImage] : undefined,
-      title: site.title,
-    },
-  };
+  return createPortfolioMetadata({ metadataBase, mode, site });
 }
 
 export default function RootLayout({


## `feat(seo): JSON-LD 안전 직렬화 경계 추가`

diff --git a/src/components/portfolio/structured-data.tsx b/src/components/portfolio/structured-data.tsx
new file mode 100644
index 0000000..95934fe
--- /dev/null
+++ b/src/components/portfolio/structured-data.tsx
@@ -0,0 +1,10 @@
+import { serializeStructuredData } from "@/lib/site-metadata";
+
+export function StructuredData({ data }: { data: Record<string, unknown> }) {
+  return (
+    <script
+      dangerouslySetInnerHTML={{ __html: serializeStructuredData(data) }}
+      type="application/ld+json"
+    />
+  );
+}
diff --git a/src/lib/site-metadata.ts b/src/lib/site-metadata.ts
index 93afa04..bc5e63f 100644
--- a/src/lib/site-metadata.ts
+++ b/src/lib/site-metadata.ts
@@ -14,6 +14,8 @@ type RouteMetadataInput = {
   type?: "article" | "website";
 };
 
+type StructuredData = Record<string, unknown>;
+
 function absoluteSiteUrl(path: string, siteUrl: URL) {
   return new URL(path, siteUrl.origin).toString();
 }
@@ -144,3 +146,10 @@ export function createSitemap({
 
   return routes.map((path) => ({ url: absoluteSiteUrl(path, siteUrl) }));
 }
+
+export function serializeStructuredData(data: StructuredData) {
+  return JSON.stringify(data)
+    .replaceAll("<", "\\u003c")
+    .replaceAll(">", "\\u003e")
+    .replaceAll("&", "\\u0026");
+}


## `feat(seo): 사이트 소유자 JSON-LD 모델 추가`

diff --git a/src/lib/site-metadata.ts b/src/lib/site-metadata.ts
index bc5e63f..e069d25 100644
--- a/src/lib/site-metadata.ts
+++ b/src/lib/site-metadata.ts
@@ -147,6 +147,49 @@ export function createSitemap({
   return routes.map((path) => ({ url: absoluteSiteUrl(path, siteUrl) }));
 }
 
+export function createSiteStructuredData({
+  content,
+  siteUrl,
+}: {
+  content: PortfolioContent;
+  siteUrl: URL;
+}): StructuredData {
+  const personId = absoluteSiteUrl("/#person", siteUrl);
+  const websiteId = absoluteSiteUrl("/#website", siteUrl);
+
+  const person: StructuredData = {
+    "@id": personId,
+    "@type": "Person",
+    description: content.profile.summary,
+    jobTitle: content.profile.role,
+    name: content.profile.name,
+    url: absoluteSiteUrl("/", siteUrl),
+  };
+
+  if (content.profile.koreanName) {
+    person.alternateName = content.profile.koreanName;
+  }
+  if (content.profile.photo) {
+    person.image = absoluteSiteUrl(content.profile.photo.src, siteUrl);
+  }
+
+  return {
+    "@context": "https://schema.org",
+    "@graph": [
+      person,
+      {
+        "@id": websiteId,
+        "@type": "WebSite",
+        author: { "@id": personId },
+        description: content.site.description,
+        inLanguage: content.site.language,
+        name: content.site.brand,
+        url: absoluteSiteUrl("/", siteUrl),
+      },
+    ],
+  };
+}
+
 export function serializeStructuredData(data: StructuredData) {
   return JSON.stringify(data)
     .replaceAll("<", "\\u003c")


## `feat(seo): production layout에 사이트 JSON-LD 연결`

diff --git a/src/app/layout.tsx b/src/app/layout.tsx
index 473e2a5..54241f5 100644
--- a/src/app/layout.tsx
+++ b/src/app/layout.tsx
@@ -1,12 +1,16 @@
 import type { Metadata } from "next";
 import localFont from "next/font/local";
 import { headers } from "next/headers";
+import { StructuredData } from "@/components/portfolio/structured-data";
 import {
   resolvePortfolioContentMode,
   resolveProductionSiteUrl,
 } from "@/lib/content-readiness";
 import { getPortfolioContent } from "@/lib/portfolio";
-import { createPortfolioMetadata } from "@/lib/site-metadata";
+import {
+  createPortfolioMetadata,
+  createSiteStructuredData,
+} from "@/lib/site-metadata";
 import "./globals.css";
 
 const geistSans = localFont({
@@ -63,13 +67,27 @@ export default function RootLayout({
 }: Readonly<{
   children: React.ReactNode;
 }>) {
+  const mode = resolvePortfolioContentMode(
+    process.env.PORTFOLIO_CONTENT_MODE,
+  );
+  const siteStructuredData =
+    mode === "production"
+      ? createSiteStructuredData({
+          content: getPortfolioContent(),
+          siteUrl: resolveProductionSiteUrl(process.env.SITE_URL),
+        })
+      : undefined;
+
   return (
     <html
       lang={site.language}
       data-scroll-behavior="smooth"
       className={`${geistSans.variable} ${geistMono.variable} ${koreanSerif.variable} h-full antialiased`}
     >
-      <body className="min-h-full flex flex-col">{children}</body>
+      <body className="min-h-full flex flex-col">
+        {siteStructuredData ? <StructuredData data={siteStructuredData} /> : null}
+        {children}
+      </body>
     </html>
   );
 }


## `feat(seo): 프로젝트 CreativeWork JSON-LD 모델 추가`

diff --git a/src/lib/site-metadata.ts b/src/lib/site-metadata.ts
index e069d25..e188423 100644
--- a/src/lib/site-metadata.ts
+++ b/src/lib/site-metadata.ts
@@ -2,7 +2,7 @@ import type { Metadata, MetadataRoute } from "next";
 
 import type { PortfolioSource } from "./content-loader";
 import type { PortfolioContentMode } from "./content-readiness";
-import type { PortfolioContent } from "./portfolio";
+import type { PortfolioContent, PortfolioProject } from "./portfolio";
 
 type SiteContent = PortfolioSource["site"];
 
@@ -190,6 +190,31 @@ export function createSiteStructuredData({
   };
 }
 
+export function createProjectStructuredData({
+  content,
+  project,
+  siteUrl,
+}: {
+  content: PortfolioContent;
+  project: PortfolioProject;
+  siteUrl: URL;
+}): StructuredData {
+  const projectPath = `/projects/${project.id}`;
+
+  return {
+    "@context": "https://schema.org",
+    "@id": `${absoluteSiteUrl(projectPath, siteUrl)}#creative-work`,
+    "@type": "CreativeWork",
+    author: { "@id": absoluteSiteUrl("/#person", siteUrl) },
+    description: project.summary,
+    image: absoluteSiteUrl(project.screenshot.src, siteUrl),
+    inLanguage: content.site.language,
+    keywords: project.tags,
+    name: project.title,
+    url: absoluteSiteUrl(projectPath, siteUrl),
+  };
+}
+
 export function serializeStructuredData(data: StructuredData) {
   return JSON.stringify(data)
     .replaceAll("<", "\\u003c")


## `feat(seo): route별 검색 metadata 정책 추가`

diff --git a/src/lib/site-metadata.test.ts b/src/lib/site-metadata.test.ts
index 971e956..b9d8fb0 100644
--- a/src/lib/site-metadata.test.ts
+++ b/src/lib/site-metadata.test.ts
@@ -26,7 +26,7 @@ describe("site indexing metadata", () => {
     });
 
     expect(metadata.metadataBase).toEqual(metadataBase);
-    expect(metadata.alternates).toEqual({ canonical: "./" });
+    expect(metadata.alternates).toEqual({ canonical: "/" });
     expect(metadata.robots).toEqual({ follow: true, index: true });
     expect(metadata.openGraph).toEqual(
       expect.objectContaining({
diff --git a/src/lib/site-metadata.ts b/src/lib/site-metadata.ts
index 751b958..6386988 100644
--- a/src/lib/site-metadata.ts
+++ b/src/lib/site-metadata.ts
@@ -5,6 +5,18 @@ import type { PortfolioContentMode } from "./content-readiness";
 
 type SiteContent = PortfolioSource["site"];
 
+type RouteMetadataInput = {
+  description: string;
+  path: `/${string}` | "/";
+  site: SiteContent;
+  title: string;
+  type?: "article" | "website";
+};
+
+function routeTitle(path: string, title: string, site: SiteContent) {
+  return path === "/" ? site.title : `${title} | ${site.brand}`;
+}
+
 export function createPortfolioMetadata({
   metadataBase,
   mode,
@@ -20,7 +32,7 @@ export function createPortfolioMetadata({
   const shouldIndex = mode === "production";
 
   return {
-    alternates: { canonical: "./" },
+    alternates: { canonical: "/" },
     description: site.description,
     metadataBase,
     openGraph: {
@@ -28,6 +40,7 @@ export function createPortfolioMetadata({
       images: socialImage ? [{ url: socialImage }] : undefined,
       title: site.title,
       type: "website",
+      url: "/",
     },
     robots: { follow: shouldIndex, index: shouldIndex },
     title: site.title,
@@ -40,6 +53,38 @@ export function createPortfolioMetadata({
   };
 }
 
+export function createRouteMetadata({
+  description,
+  path,
+  site,
+  title,
+  type = "website",
+}: RouteMetadataInput): Metadata {
+  const resolvedTitle = routeTitle(path, title, site);
+  const images = site.socialImage
+    ? [{ alt: site.title, url: site.socialImage }]
+    : undefined;
+
+  return {
+    alternates: { canonical: path },
+    description,
+    openGraph: {
+      description,
+      images,
+      title: resolvedTitle,
+      type,
+      url: path,
+    },
+    title: resolvedTitle,
+    twitter: {
+      card: "summary_large_image",
+      description,
+      images: site.socialImage ? [site.socialImage] : undefined,
+      title: resolvedTitle,
+    },
+  };
+}
+
 export function createRobots({
   mode,
   siteUrl,


## `feat(seo): 홈과 프로젝트 route metadata 연결`

diff --git a/src/app/page.tsx b/src/app/page.tsx
index 3cc1033..94f23a3 100644
--- a/src/app/page.tsx
+++ b/src/app/page.tsx
@@ -1,13 +1,26 @@
+import type { Metadata } from "next";
 import { ClassicHomeRoute } from "@/designs/classic/home-route";
 import { DesignHomeRoute } from "@/designs/design/home-route";
 import { hasDedicatedRouteRenderer, renderDesignRoute } from "@/designs/registry";
-import { type RouteSearchParams } from "@/lib/portfolio";
+import { getPortfolioContent, type RouteSearchParams } from "@/lib/portfolio";
 import { resolvePortfolioPageContext } from "@/lib/portfolio/page-context";
+import { createRouteMetadata } from "@/lib/site-metadata";
 
 type HomePageProps = {
   searchParams?: RouteSearchParams;
 };
 
+export function generateMetadata(): Metadata {
+  const content = getPortfolioContent();
+
+  return createRouteMetadata({
+    description: content.site.description,
+    path: "/",
+    site: content.site,
+    title: content.site.title,
+  });
+}
+
 export default async function Home({ searchParams }: HomePageProps) {
   const { activeTemplate, content, contentDebug } =
     await resolvePortfolioPageContext({
diff --git a/src/app/projects/[projectId]/page.tsx b/src/app/projects/[projectId]/page.tsx
index 7161076..2ed7fbc 100644
--- a/src/app/projects/[projectId]/page.tsx
+++ b/src/app/projects/[projectId]/page.tsx
@@ -1,3 +1,4 @@
+import type { Metadata } from "next";
 import { notFound } from "next/navigation";
 import { ProjectDetailView } from "@/components/portfolio/project-detail-view";
 import { PageShell } from "@/components/portfolio/site-shell";
@@ -9,6 +10,12 @@ import {
   type RouteSearchParams,
 } from "@/lib/portfolio";
 import { resolvePortfolioPageContext } from "@/lib/portfolio/page-context";
+import { createRouteMetadata } from "@/lib/site-metadata";
+
+type ProjectDetailPageProps = {
+  params: Promise<{ projectId: string }>;
+  searchParams?: RouteSearchParams;
+};
 
 export function generateStaticParams() {
   return getPortfolioContent().projects.map((project) => ({
@@ -16,13 +23,30 @@ export function generateStaticParams() {
   }));
 }
 
+export async function generateMetadata({
+  params,
+}: Pick<ProjectDetailPageProps, "params">): Promise<Metadata> {
+  const content = getPortfolioContent();
+  const { projectId } = await params;
+  const project = getProjectById(projectId, content);
+
+  if (!isSitePageEnabled("projects", content) || !project) {
+    notFound();
+  }
+
+  return createRouteMetadata({
+    description: project.summary,
+    path: `/projects/${project.id}`,
+    site: content.site,
+    title: project.title,
+    type: "article",
+  });
+}
+
 export default async function ProjectDetailPage({
   params,
   searchParams,
-}: {
-  params: Promise<{ projectId: string }>;
-  searchParams?: RouteSearchParams;
-}) {
+}: ProjectDetailPageProps) {
   const content = getPortfolioContent();
   if (!isSitePageEnabled("projects", content)) notFound();
   const { projectId } = await params;
diff --git a/src/app/projects/page.tsx b/src/app/projects/page.tsx
index 516190b..9bc6e5c 100644
--- a/src/app/projects/page.tsx
+++ b/src/app/projects/page.tsx
@@ -1,3 +1,4 @@
+import type { Metadata } from "next";
 import { PageShell } from "@/components/portfolio/site-shell";
 import { notFound } from "next/navigation";
 import { ClassicProjectsView } from "@/designs/classic/projects/projects-route";
@@ -11,6 +12,20 @@ import {
 } from "@/lib/portfolio";
 import { resolvePortfolioPageContext } from "@/lib/portfolio/page-context";
 import { groupProjects } from "@/lib/portfolio/project-groups";
+import { createRouteMetadata } from "@/lib/site-metadata";
+
+export function generateMetadata(): Metadata {
+  const content = getPortfolioContent();
+  if (!isSitePageEnabled("projects", content)) notFound();
+  const hero = content.presentation.pages.projects.design.hero;
+
+  return createRouteMetadata({
+    description: hero.body,
+    path: "/projects",
+    site: content.site,
+    title: hero.title,
+  });
+}
 
 export default async function ProjectsPage({
   searchParams,


## `feat(seo): 프로필 route metadata 연결`

diff --git a/src/app/about/page.tsx b/src/app/about/page.tsx
index 4b8594b..dfff0e3 100644
--- a/src/app/about/page.tsx
+++ b/src/app/about/page.tsx
@@ -1,3 +1,4 @@
+import type { Metadata } from "next";
 import Link from "next/link";
 import { notFound } from "next/navigation";
 import { ArrowRightIcon } from "@/components/icons";
@@ -18,6 +19,19 @@ import {
   type RouteSearchParams,
 } from "@/lib/portfolio";
 import { resolvePortfolioPageContext } from "@/lib/portfolio/page-context";
+import { createRouteMetadata } from "@/lib/site-metadata";
+
+export function generateMetadata(): Metadata {
+  const content = getPortfolioContent();
+  if (!isSitePageEnabled("about", content)) notFound();
+
+  return createRouteMetadata({
+    description: content.profile.summary,
+    path: "/about",
+    site: content.site,
+    title: content.presentation.pages.about.hero.title,
+  });
+}
 
 export default async function AboutPage({
   searchParams,
diff --git a/src/app/contact/page.tsx b/src/app/contact/page.tsx
index 3fe4864..1f5898e 100644
--- a/src/app/contact/page.tsx
+++ b/src/app/contact/page.tsx
@@ -1,3 +1,4 @@
+import type { Metadata } from "next";
 import { ArrowRightIcon } from "@/components/icons";
 import { notFound } from "next/navigation";
 import { ContentHint } from "@/components/portfolio/content-hint";
@@ -11,6 +12,19 @@ import {
   type RouteSearchParams,
 } from "@/lib/portfolio";
 import { resolvePortfolioPageContext } from "@/lib/portfolio/page-context";
+import { createRouteMetadata } from "@/lib/site-metadata";
+
+export function generateMetadata(): Metadata {
+  const content = getPortfolioContent();
+  if (!isSitePageEnabled("contact", content)) notFound();
+
+  return createRouteMetadata({
+    description: content.contact.intro,
+    path: "/contact",
+    site: content.site,
+    title: content.contact.title,
+  });
+}
 
 export default async function ContactPage({
   searchParams,
diff --git a/src/app/resume/page.tsx b/src/app/resume/page.tsx
index 6af8fc5..eefae60 100644
--- a/src/app/resume/page.tsx
+++ b/src/app/resume/page.tsx
@@ -1,3 +1,4 @@
+import type { Metadata } from "next";
 import Link from "next/link";
 import { notFound } from "next/navigation";
 import { ArrowRightIcon } from "@/components/icons";
@@ -13,6 +14,20 @@ import {
   type RouteSearchParams,
 } from "@/lib/portfolio";
 import { resolvePortfolioPageContext } from "@/lib/portfolio/page-context";
+import { createRouteMetadata } from "@/lib/site-metadata";
+
+export function generateMetadata(): Metadata {
+  const content = getPortfolioContent();
+  if (!isSitePageEnabled("resume", content)) notFound();
+  const hero = content.presentation.pages.resume.hero;
+
+  return createRouteMetadata({
+    description: hero.body,
+    path: "/resume",
+    site: content.site,
+    title: hero.title,
+  });
+}
 
 export default async function ResumePage({
   searchParams,


