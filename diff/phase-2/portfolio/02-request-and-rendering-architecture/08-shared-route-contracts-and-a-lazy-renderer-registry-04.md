## `refactor(designs): 모든 route를 registry renderer로 위임`

diff --git a/src/app/about/page.tsx b/src/app/about/page.tsx
index 2131105..84e767d 100644
--- a/src/app/about/page.tsx
+++ b/src/app/about/page.tsx
@@ -1,15 +1,13 @@
 import type { Metadata } from "next";
 import { notFound } from "next/navigation";
-import ClassicAboutRoute from "@/designs/classic/about-route";
-import DesignAboutRoute from "@/designs/design/about-route";
-import { hasDedicatedRouteRenderer, renderDesignRoute } from "@/designs/registry";
+import { renderDesignRoute } from "@/designs/registry";
+import { resolvePortfolioPageContext } from "@/lib/portfolio/page-context";
+import { createAboutViewModel } from "@/lib/portfolio/view-models";
 import {
   getPortfolioContent,
   isSitePageEnabled,
   type RouteSearchParams,
 } from "@/lib/portfolio";
-import { resolvePortfolioPageContext } from "@/lib/portfolio/page-context";
-import { createAboutViewModel } from "@/lib/portfolio/view-models";
 import { createRouteMetadata } from "@/lib/site-metadata";
 
 export function generateMetadata(): Metadata {
@@ -29,33 +27,20 @@ export default async function AboutPage({
 }: {
   searchParams?: RouteSearchParams;
 }) {
-  const contentSource = getPortfolioContent();
-  if (!isSitePageEnabled("about", contentSource)) notFound();
+  const content = getPortfolioContent();
+  if (!isSitePageEnabled("about", content)) notFound();
+
   const { activeTemplate, contentDebug } =
     await resolvePortfolioPageContext({
-      content: contentSource,
+      content,
       currentPath: "/about",
       searchParams,
     });
-  const content = createAboutViewModel(contentSource);
-
-  if (hasDedicatedRouteRenderer(activeTemplate)) {
-    return renderDesignRoute(activeTemplate, {
-      contentDebug,
-      currentPath: "/about",
-      route: "about",
-      viewModel: content,
-    });
-  }
-
-  const AboutRoute = activeTemplate === "design" ? DesignAboutRoute : ClassicAboutRoute;
 
-  return (
-    <AboutRoute
-      content={content}
-      contentDebug={contentDebug}
-      currentPath="/about"
-      route="about"
-    />
-  );
+  return renderDesignRoute(activeTemplate, {
+    contentDebug,
+    currentPath: "/about",
+    route: "about",
+    viewModel: createAboutViewModel(content),
+  });
 }
diff --git a/src/app/contact/page.tsx b/src/app/contact/page.tsx
index e6148ce..894b753 100644
--- a/src/app/contact/page.tsx
+++ b/src/app/contact/page.tsx
@@ -1,15 +1,13 @@
 import type { Metadata } from "next";
 import { notFound } from "next/navigation";
-import ClassicContactRoute from "@/designs/classic/contact-route";
-import DesignContactRoute from "@/designs/design/contact-route";
-import { hasDedicatedRouteRenderer, renderDesignRoute } from "@/designs/registry";
+import { renderDesignRoute } from "@/designs/registry";
+import { resolvePortfolioPageContext } from "@/lib/portfolio/page-context";
+import { createContactViewModel } from "@/lib/portfolio/view-models";
 import {
   getPortfolioContent,
   isSitePageEnabled,
   type RouteSearchParams,
 } from "@/lib/portfolio";
-import { resolvePortfolioPageContext } from "@/lib/portfolio/page-context";
-import { createContactViewModel } from "@/lib/portfolio/view-models";
 import { createRouteMetadata } from "@/lib/site-metadata";
 
 export function generateMetadata(): Metadata {
@@ -29,33 +27,20 @@ export default async function ContactPage({
 }: {
   searchParams?: RouteSearchParams;
 }) {
-  const contentSource = getPortfolioContent();
-  if (!isSitePageEnabled("contact", contentSource)) notFound();
+  const content = getPortfolioContent();
+  if (!isSitePageEnabled("contact", content)) notFound();
+
   const { activeTemplate, contentDebug } =
     await resolvePortfolioPageContext({
-      content: contentSource,
+      content,
       currentPath: "/contact",
       searchParams,
     });
-  const content = createContactViewModel(contentSource);
-
-  if (hasDedicatedRouteRenderer(activeTemplate)) {
-    return renderDesignRoute(activeTemplate, {
-      contentDebug,
-      currentPath: "/contact",
-      route: "contact",
-      viewModel: content,
-    });
-  }
-
-  const ContactRoute = activeTemplate === "design" ? DesignContactRoute : ClassicContactRoute;
 
-  return (
-    <ContactRoute
-      content={content}
-      contentDebug={contentDebug}
-      currentPath="/contact"
-      route="contact"
-    />
-  );
+  return renderDesignRoute(activeTemplate, {
+    contentDebug,
+    currentPath: "/contact",
+    route: "contact",
+    viewModel: createContactViewModel(content),
+  });
 }
diff --git a/src/app/interview-map/page.tsx b/src/app/interview-map/page.tsx
index 39eb3d4..cf81800 100644
--- a/src/app/interview-map/page.tsx
+++ b/src/app/interview-map/page.tsx
@@ -1,14 +1,12 @@
 import type { Metadata } from "next";
 import { notFound } from "next/navigation";
-import ClassicInterviewMapRoute from "@/designs/classic/interview-map-route";
-import DesignInterviewMapRoute from "@/designs/design/interview-map-route";
-import { hasDedicatedRouteRenderer, renderDesignRoute } from "@/designs/registry";
+import { renderDesignRoute } from "@/designs/registry";
+import { resolvePortfolioPageContext } from "@/lib/portfolio/page-context";
 import {
   getPortfolioContent,
   isSitePageEnabled,
   type RouteSearchParams,
 } from "@/lib/portfolio";
-import { resolvePortfolioPageContext } from "@/lib/portfolio/page-context";
 import { createInterviewMapViewModel } from "@/lib/portfolio/view-models";
 import { createRouteMetadata } from "@/lib/site-metadata";
 
@@ -31,32 +29,18 @@ export default async function InterviewMapPage({
 }) {
   const content = getPortfolioContent();
   if (!isSitePageEnabled("interviewMap", content)) notFound();
+
   const { activeTemplate, contentDebug } =
     await resolvePortfolioPageContext({
       content,
       currentPath: "/interview-map",
       searchParams,
     });
-  const viewModel = createInterviewMapViewModel(content);
-
-  if (hasDedicatedRouteRenderer(activeTemplate)) {
-    return renderDesignRoute(activeTemplate, {
-      contentDebug,
-      currentPath: "/interview-map",
-      route: "interview-map",
-      viewModel,
-    });
-  }
 
-  const InterviewMapRoute =
-    activeTemplate === "design" ? DesignInterviewMapRoute : ClassicInterviewMapRoute;
-
-  return (
-    <InterviewMapRoute
-      content={viewModel}
-      contentDebug={contentDebug}
-      currentPath="/interview-map"
-      route="interview-map"
-    />
-  );
+  return renderDesignRoute(activeTemplate, {
+    contentDebug,
+    currentPath: "/interview-map",
+    route: "interview-map",
+    viewModel: createInterviewMapViewModel(content),
+  });
 }
diff --git a/src/app/journey/page.tsx b/src/app/journey/page.tsx
index 0dcb5c3..985af72 100644
--- a/src/app/journey/page.tsx
+++ b/src/app/journey/page.tsx
@@ -1,14 +1,12 @@
 import type { Metadata } from "next";
 import { notFound } from "next/navigation";
-import ClassicJourneyRoute from "@/designs/classic/journey-route";
-import DesignJourneyRoute from "@/designs/design/journey-route";
-import { hasDedicatedRouteRenderer, renderDesignRoute } from "@/designs/registry";
+import { renderDesignRoute } from "@/designs/registry";
+import { resolvePortfolioPageContext } from "@/lib/portfolio/page-context";
 import {
   getPortfolioContent,
   isSitePageEnabled,
   type RouteSearchParams,
 } from "@/lib/portfolio";
-import { resolvePortfolioPageContext } from "@/lib/portfolio/page-context";
 import { createJourneyViewModel } from "@/lib/portfolio/view-models";
 import { createRouteMetadata } from "@/lib/site-metadata";
 
@@ -31,31 +29,18 @@ export default async function JourneyPage({
 }) {
   const content = getPortfolioContent();
   if (!isSitePageEnabled("journey", content)) notFound();
+
   const { activeTemplate, contentDebug } =
     await resolvePortfolioPageContext({
       content,
       currentPath: "/journey",
       searchParams,
     });
-  const viewModel = createJourneyViewModel(content);
-
-  if (hasDedicatedRouteRenderer(activeTemplate)) {
-    return renderDesignRoute(activeTemplate, {
-      contentDebug,
-      currentPath: "/journey",
-      route: "journey",
-      viewModel,
-    });
-  }
 
-  const JourneyRoute = activeTemplate === "design" ? DesignJourneyRoute : ClassicJourneyRoute;
-
-  return (
-    <JourneyRoute
-      content={viewModel}
-      contentDebug={contentDebug}
-      currentPath="/journey"
-      route="journey"
-    />
-  );
+  return renderDesignRoute(activeTemplate, {
+    contentDebug,
+    currentPath: "/journey",
+    route: "journey",
+    viewModel: createJourneyViewModel(content),
+  });
 }
diff --git a/src/app/page.tsx b/src/app/page.tsx
index 9e92678..ba2215e 100644
--- a/src/app/page.tsx
+++ b/src/app/page.tsx
@@ -1,10 +1,11 @@
 import type { Metadata } from "next";
-import ClassicHomeRoute from "@/designs/classic/home-route";
-import DesignHomeRoute from "@/designs/design/home-route";
-import { hasDedicatedRouteRenderer, renderDesignRoute } from "@/designs/registry";
-import { getPortfolioContent, type RouteSearchParams } from "@/lib/portfolio";
+import { renderDesignRoute } from "@/designs/registry";
 import { resolvePortfolioPageContext } from "@/lib/portfolio/page-context";
 import { createHomeViewModel } from "@/lib/portfolio/view-models";
+import {
+  getPortfolioContent,
+  type RouteSearchParams,
+} from "@/lib/portfolio";
 import { createRouteMetadata } from "@/lib/site-metadata";
 
 type HomePageProps = {
@@ -28,34 +29,11 @@ export default async function Home({ searchParams }: HomePageProps) {
       currentPath: "/",
       searchParams,
     });
-  const viewModel = createHomeViewModel(content);
 
-  if (hasDedicatedRouteRenderer(activeTemplate)) {
-    return renderDesignRoute(activeTemplate, {
-      contentDebug,
-      currentPath: "/",
-      route: "home",
-      viewModel,
-    });
-  }
-
-  if (activeTemplate === "classic") {
-    return (
-      <ClassicHomeRoute
-        content={viewModel}
-        contentDebug={contentDebug}
-        currentPath="/"
-        route="home"
-      />
-    );
-  }
-
-  return (
-    <DesignHomeRoute
-      content={viewModel}
-      contentDebug={contentDebug}
-      currentPath="/"
-      route="home"
-    />
-  );
+  return renderDesignRoute(activeTemplate, {
+    contentDebug,
+    currentPath: "/",
+    route: "home",
+    viewModel: createHomeViewModel(content),
+  });
 }
diff --git a/src/app/projects/[projectId]/page.tsx b/src/app/projects/[projectId]/page.tsx
index 9215cd9..eb2eedf 100644
--- a/src/app/projects/[projectId]/page.tsx
+++ b/src/app/projects/[projectId]/page.tsx
@@ -1,21 +1,19 @@
 import type { Metadata } from "next";
 import { notFound } from "next/navigation";
 import { StructuredData } from "@/components/portfolio/structured-data";
-import ClassicProjectDetailRoute from "@/designs/classic/project-detail-route";
-import DesignProjectDetailRoute from "@/designs/design/project-detail-route";
-import { hasDedicatedRouteRenderer, renderDesignRoute } from "@/designs/registry";
+import { renderDesignRoute } from "@/designs/registry";
 import {
   resolvePortfolioContentMode,
   resolveProductionSiteUrl,
 } from "@/lib/content-readiness";
+import { resolvePortfolioPageContext } from "@/lib/portfolio/page-context";
+import { createProjectDetailViewModel } from "@/lib/portfolio/view-models";
 import {
   getPortfolioContent,
-  isSitePageEnabled,
   getProjectById,
+  isSitePageEnabled,
   type RouteSearchParams,
 } from "@/lib/portfolio";
-import { resolvePortfolioPageContext } from "@/lib/portfolio/page-context";
-import { createProjectDetailViewModel } from "@/lib/portfolio/view-models";
 import {
   createProjectStructuredData,
   createRouteMetadata,
@@ -58,19 +56,17 @@ export default async function ProjectDetailPage({
 }: ProjectDetailPageProps) {
   const content = getPortfolioContent();
   if (!isSitePageEnabled("projects", content)) notFound();
+
   const { projectId } = await params;
+  const viewModel = createProjectDetailViewModel(content, projectId);
+  if (!viewModel) notFound();
+
   const { activeTemplate, contentDebug } =
     await resolvePortfolioPageContext({
       content,
       currentPath: `/projects/${projectId}`,
       searchParams,
     });
-  const viewModel = createProjectDetailViewModel(content, projectId);
-
-  if (!viewModel) {
-    notFound();
-  }
-  const project = viewModel.project;
 
   const mode = resolvePortfolioContentMode(
     process.env.PORTFOLIO_CONTENT_MODE,
@@ -79,50 +75,22 @@ export default async function ProjectDetailPage({
     mode === "production"
       ? createProjectStructuredData({
           content,
-          project,
+          project: viewModel.project,
           siteUrl: resolveProductionSiteUrl(process.env.SITE_URL),
         })
       : undefined;
 
-  if (hasDedicatedRouteRenderer(activeTemplate)) {
-    const designRoute = await renderDesignRoute(activeTemplate, {
-      contentDebug,
-      currentPath: `/projects/${project.id}`,
-      route: "project-detail",
-      viewModel,
-    });
-
-    return (
-      <>
-        {structuredData ? <StructuredData data={structuredData} /> : null}
-        {designRoute}
-      </>
-    );
-  }
-
-  if (activeTemplate === "design") {
-    return (
-      <>
-        {structuredData ? <StructuredData data={structuredData} /> : null}
-        <DesignProjectDetailRoute
-          content={viewModel}
-          contentDebug={contentDebug}
-          currentPath={`/projects/${project.id}`}
-          route="project-detail"
-        />
-      </>
-    );
-  }
+  const designRoute = await renderDesignRoute(activeTemplate, {
+    contentDebug,
+    currentPath: `/projects/${viewModel.project.id}`,
+    route: "project-detail",
+    viewModel,
+  });
 
   return (
     <>
       {structuredData ? <StructuredData data={structuredData} /> : null}
-      <ClassicProjectDetailRoute
-        content={viewModel}
-        contentDebug={contentDebug}
-        currentPath={`/projects/${project.id}`}
-        route="project-detail"
-      />
+      {designRoute}
     </>
   );
 }
diff --git a/src/app/projects/page.tsx b/src/app/projects/page.tsx
index ae49935..c2e2ffc 100644
--- a/src/app/projects/page.tsx
+++ b/src/app/projects/page.tsx
@@ -1,15 +1,13 @@
 import type { Metadata } from "next";
 import { notFound } from "next/navigation";
-import ClassicProjectsRoute from "@/designs/classic/projects-route";
-import DesignProjectsRoute from "@/designs/design/projects-route";
-import { hasDedicatedRouteRenderer, renderDesignRoute } from "@/designs/registry";
+import { renderDesignRoute } from "@/designs/registry";
+import { resolvePortfolioPageContext } from "@/lib/portfolio/page-context";
+import { createProjectIndexViewModel } from "@/lib/portfolio/view-models";
 import {
   getPortfolioContent,
   isSitePageEnabled,
   type RouteSearchParams,
 } from "@/lib/portfolio";
-import { resolvePortfolioPageContext } from "@/lib/portfolio/page-context";
-import { createProjectIndexViewModel } from "@/lib/portfolio/view-models";
 import { createRouteMetadata } from "@/lib/site-metadata";
 
 export function generateMetadata(): Metadata {
@@ -32,40 +30,18 @@ export default async function ProjectsPage({
 }) {
   const content = getPortfolioContent();
   if (!isSitePageEnabled("projects", content)) notFound();
+
   const { activeTemplate, contentDebug } =
     await resolvePortfolioPageContext({
       content,
       currentPath: "/projects",
       searchParams,
     });
-  const viewModel = createProjectIndexViewModel(content);
 
-  if (hasDedicatedRouteRenderer(activeTemplate)) {
-    return renderDesignRoute(activeTemplate, {
-      contentDebug,
-      currentPath: "/projects",
-      route: "projects",
-      viewModel,
-    });
-  }
-
-  if (activeTemplate === "classic") {
-    return (
-      <ClassicProjectsRoute
-        content={viewModel}
-        contentDebug={contentDebug}
-        currentPath="/projects"
-        route="projects"
-      />
-    );
-  }
-
-  return (
-    <DesignProjectsRoute
-      content={viewModel}
-      contentDebug={contentDebug}
-      currentPath="/projects"
-      route="projects"
-    />
-  );
+  return renderDesignRoute(activeTemplate, {
+    contentDebug,
+    currentPath: "/projects",
+    route: "projects",
+    viewModel: createProjectIndexViewModel(content),
+  });
 }
diff --git a/src/app/resume/page.tsx b/src/app/resume/page.tsx
index b6ce384..a2d47ef 100644
--- a/src/app/resume/page.tsx
+++ b/src/app/resume/page.tsx
@@ -1,15 +1,13 @@
 import type { Metadata } from "next";
 import { notFound } from "next/navigation";
-import ClassicResumeRoute from "@/designs/classic/resume-route";
-import DesignResumeRoute from "@/designs/design/resume-route";
-import { hasDedicatedRouteRenderer, renderDesignRoute } from "@/designs/registry";
+import { renderDesignRoute } from "@/designs/registry";
+import { resolvePortfolioPageContext } from "@/lib/portfolio/page-context";
+import { createResumeViewModel } from "@/lib/portfolio/view-models";
 import {
   getPortfolioContent,
   isSitePageEnabled,
   type RouteSearchParams,
 } from "@/lib/portfolio";
-import { resolvePortfolioPageContext } from "@/lib/portfolio/page-context";
-import { createResumeViewModel } from "@/lib/portfolio/view-models";
 import { createRouteMetadata } from "@/lib/site-metadata";
 
 export function generateMetadata(): Metadata {
@@ -30,33 +28,20 @@ export default async function ResumePage({
 }: {
   searchParams?: RouteSearchParams;
 }) {
-  const contentSource = getPortfolioContent();
-  if (!isSitePageEnabled("resume", contentSource)) notFound();
+  const content = getPortfolioContent();
+  if (!isSitePageEnabled("resume", content)) notFound();
+
   const { activeTemplate, contentDebug } =
     await resolvePortfolioPageContext({
-      content: contentSource,
+      content,
       currentPath: "/resume",
       searchParams,
     });
-  const content = createResumeViewModel(contentSource);
-
-  if (hasDedicatedRouteRenderer(activeTemplate)) {
-    return renderDesignRoute(activeTemplate, {
-      contentDebug,
-      currentPath: "/resume",
-      route: "resume",
-      viewModel: content,
-    });
-  }
-
-  const ResumeRoute = activeTemplate === "design" ? DesignResumeRoute : ClassicResumeRoute;
 
-  return (
-    <ResumeRoute
-      content={content}
-      contentDebug={contentDebug}
-      currentPath="/resume"
-      route="resume"
-    />
-  );
+  return renderDesignRoute(activeTemplate, {
+    contentDebug,
+    currentPath: "/resume",
+    route: "resume",
+    viewModel: createResumeViewModel(content),
+  });
 }
diff --git a/src/designs/registry.tsx b/src/designs/registry.tsx
index 5a53b14..9eb674f 100644
--- a/src/designs/registry.tsx
+++ b/src/designs/registry.tsx
@@ -6,40 +6,31 @@ type DesignModule = {
   default: ComponentType<DesignRouteProps>;
 };
 
-const routeLoaders: Partial<Record<SiteDesignId, () => Promise<DesignModule>>> = {
+const routeLoaders: Record<SiteDesignId, () => Promise<DesignModule>> = {
+  design: () => import("./design"),
+  classic: () => import("./classic"),
   editorial: () => import("./editorial"),
   brutalist: () => import("./brutalist"),
   cinematic: () => import("./cinematic"),
 };
 
-export function hasDedicatedRouteRenderer(
-  designId: SiteDesignId,
-): designId is "editorial" | "brutalist" | "cinematic" {
-  return designId in routeLoaders;
-}
-
 export async function renderDesignRoute(
   designId: SiteDesignId,
   props: DesignRouteRequestProps,
 ): Promise<ReactElement | null> {
   const loader = routeLoaders[designId];
 
-  if (!loader) return null;
-
   const { default: Renderer } = await loader();
-  const rendererProps: DesignRouteProps =
-    "viewModel" in props
-      ? {
-          content: props.viewModel,
-          contentDebug: props.contentDebug,
-          currentPath: props.currentPath,
-          project:
-            props.viewModel.route === "project-detail"
-              ? props.viewModel.project
-              : undefined,
-          route: props.route,
-        }
-      : props;
+  const rendererProps: DesignRouteProps = {
+    content: props.viewModel,
+    contentDebug: props.contentDebug,
+    currentPath: props.currentPath,
+    project:
+      props.viewModel.route === "project-detail"
+        ? props.viewModel.project
+        : undefined,
+    route: props.route,
+  };
 
   return <Renderer {...rendererProps} />;
 }


