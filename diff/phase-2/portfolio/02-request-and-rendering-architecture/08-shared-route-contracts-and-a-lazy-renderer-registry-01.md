# 공통 라우트 계약과 Lazy Renderer Registry

## `feat(designs): site design 정의 registry 추가`

diff --git a/src/designs/config.ts b/src/designs/config.ts
new file mode 100644
index 0000000..32df799
--- /dev/null
+++ b/src/designs/config.ts
@@ -0,0 +1,23 @@
+import type { SiteDesignId } from "@/lib/portfolio";
+
+export type SiteDesignDefinition = {
+  id: SiteDesignId;
+  swatch: [string, string, string];
+};
+
+export const SITE_DESIGNS: SiteDesignDefinition[] = [
+  {
+    id: "design",
+    swatch: ["#f7faf8", "#008c89", "#4f46e5"],
+  },
+  {
+    id: "classic",
+    swatch: ["#1f2023", "#9cc8b1", "#7aa7ff"],
+  },
+];
+
+export const SITE_DESIGN_IDS = SITE_DESIGNS.map((design) => design.id);
+
+export function getSiteDesignDefinition(id: SiteDesignId) {
+  return SITE_DESIGNS.find((design) => design.id === id) ?? SITE_DESIGNS[0];
+}


## `feat(designs): route renderer 계약 추가`

diff --git a/src/designs/types.ts b/src/designs/types.ts
new file mode 100644
index 0000000..7c1f43b
--- /dev/null
+++ b/src/designs/types.ts
@@ -0,0 +1,19 @@
+import type { PortfolioContent, PortfolioProject } from "@/lib/portfolio";
+
+export type PortfolioRouteId =
+  | "home"
+  | "projects"
+  | "project-detail"
+  | "about"
+  | "resume"
+  | "contact"
+  | "journey"
+  | "interview-map";
+
+export type DesignRouteProps = {
+  content: PortfolioContent;
+  contentDebug: boolean;
+  currentPath: string;
+  project?: PortfolioProject;
+  route: PortfolioRouteId;
+};


## `refactor(designs): 확장 renderer lazy registry 추가`

diff --git a/src/designs/registry.tsx b/src/designs/registry.tsx
new file mode 100644
index 0000000..058641e
--- /dev/null
+++ b/src/designs/registry.tsx
@@ -0,0 +1,27 @@
+import type { ComponentType, ReactElement } from "react";
+import type { SiteDesignId } from "@/lib/portfolio";
+import type { DesignRouteProps } from "./types";
+
+type DesignModule = {
+  default: ComponentType<DesignRouteProps>;
+};
+
+const routeLoaders: Partial<Record<SiteDesignId, () => Promise<DesignModule>>> = {};
+
+export function hasDedicatedRouteRenderer(
+  designId: SiteDesignId,
+): designId is "editorial" | "brutalist" | "cinematic" {
+  return designId in routeLoaders;
+}
+
+export async function renderDesignRoute(
+  designId: SiteDesignId,
+  props: DesignRouteProps,
+): Promise<ReactElement | null> {
+  const loader = routeLoaders[designId];
+
+  if (!loader) return null;
+
+  const { default: Renderer } = await loader();
+  return <Renderer {...props} />;
+}


## `refactor(routes): 확장 디자인 renderer 위임 경계 추가`

diff --git a/src/app/about/page.tsx b/src/app/about/page.tsx
index 79d4a7a..6531b82 100644
--- a/src/app/about/page.tsx
+++ b/src/app/about/page.tsx
@@ -7,6 +7,7 @@ import { ProfilePhoto } from "@/components/portfolio/profile-photo";
 import { SectionHeading } from "@/components/portfolio/section-heading";
 import { PageShell } from "@/components/portfolio/site-shell";
 import { StackList } from "@/components/portfolio/stack-list";
+import { hasDedicatedRouteRenderer, renderDesignRoute } from "@/designs/registry";
 import {
   getPortfolioContent,
   getTemplateHref,
@@ -30,6 +31,15 @@ export default async function AboutPage({
   const activeTemplate = resolveHomeTemplateId(params.view, content.presentation);
   const contentDebug = resolveContentDebug(params.debug);
 
+  if (hasDedicatedRouteRenderer(activeTemplate)) {
+    return renderDesignRoute(activeTemplate, {
+      content,
+      contentDebug,
+      currentPath: "/about",
+      route: "about",
+    });
+  }
+
   const pageCopy = content.presentation.pages.about;
 
   return (
diff --git a/src/app/contact/page.tsx b/src/app/contact/page.tsx
index 042c9d6..3e2d286 100644
--- a/src/app/contact/page.tsx
+++ b/src/app/contact/page.tsx
@@ -3,6 +3,7 @@ import { notFound } from "next/navigation";
 import { ContentHint } from "@/components/portfolio/content-hint";
 import { ContentLinkView } from "@/components/portfolio/content-link";
 import { PageShell } from "@/components/portfolio/site-shell";
+import { hasDedicatedRouteRenderer, renderDesignRoute } from "@/designs/registry";
 import {
   getPortfolioContent,
   isSitePageEnabled,
@@ -23,6 +24,15 @@ export default async function ContactPage({
   const activeTemplate = resolveHomeTemplateId(params.view, content.presentation);
   const contentDebug = resolveContentDebug(params.debug);
 
+  if (hasDedicatedRouteRenderer(activeTemplate)) {
+    return renderDesignRoute(activeTemplate, {
+      content,
+      contentDebug,
+      currentPath: "/contact",
+      route: "contact",
+    });
+  }
+
   const pageCopy = content.presentation.pages.contact;
   const preferredLinks = getPreferredContactLinks(content);
 
diff --git a/src/app/interview-map/page.tsx b/src/app/interview-map/page.tsx
index f1c509e..f9d1c5a 100644
--- a/src/app/interview-map/page.tsx
+++ b/src/app/interview-map/page.tsx
@@ -4,6 +4,7 @@ import { ArrowRightIcon } from "@/components/icons";
 import { ContentHint } from "@/components/portfolio/content-hint";
 import { SectionHeading } from "@/components/portfolio/section-heading";
 import { PageShell } from "@/components/portfolio/site-shell";
+import { hasDedicatedRouteRenderer, renderDesignRoute } from "@/designs/registry";
 import {
   getPortfolioContent,
   getTemplateHref,
@@ -27,6 +28,15 @@ export default async function InterviewMapPage({
   const activeTemplate = resolveHomeTemplateId(params.view, content.presentation);
   const contentDebug = resolveContentDebug(params.debug);
 
+  if (hasDedicatedRouteRenderer(activeTemplate)) {
+    return renderDesignRoute(activeTemplate, {
+      content,
+      contentDebug,
+      currentPath: "/interview-map",
+      route: "interview-map",
+    });
+  }
+
   const pageCopy = content.presentation.pages.interviewMap;
   const data = content.interviewMap;
 
diff --git a/src/app/journey/page.tsx b/src/app/journey/page.tsx
index 37419ad..783f0e7 100644
--- a/src/app/journey/page.tsx
+++ b/src/app/journey/page.tsx
@@ -5,6 +5,7 @@ import { ContentHint } from "@/components/portfolio/content-hint";
 import { JourneyList } from "@/components/portfolio/journey-list";
 import { SectionHeading } from "@/components/portfolio/section-heading";
 import { PageShell } from "@/components/portfolio/site-shell";
+import { hasDedicatedRouteRenderer, renderDesignRoute } from "@/designs/registry";
 import {
   getPortfolioContent,
   getTemplateHref,
@@ -29,6 +30,15 @@ export default async function JourneyPage({
   const activeTemplate = resolveHomeTemplateId(params.view, content.presentation);
   const contentDebug = resolveContentDebug(params.debug);
 
+  if (hasDedicatedRouteRenderer(activeTemplate)) {
+    return renderDesignRoute(activeTemplate, {
+      content,
+      contentDebug,
+      currentPath: "/journey",
+      route: "journey",
+    });
+  }
+
   const pageCopy = content.presentation.pages.journey;
   const narrative = content.journeyNarrative;
 
diff --git a/src/app/page.tsx b/src/app/page.tsx
index d7debcc..465fbd6 100644
--- a/src/app/page.tsx
+++ b/src/app/page.tsx
@@ -1,5 +1,6 @@
 import { ClassicHomeRoute } from "@/designs/classic/home-route";
 import { DesignHomeRoute } from "@/designs/design/home-route";
+import { hasDedicatedRouteRenderer, renderDesignRoute } from "@/designs/registry";
 import {
   getPortfolioContent,
   resolveContentDebug,
@@ -17,6 +18,15 @@ export default async function Home({ searchParams }: HomePageProps) {
   const activeTemplate = resolveHomeTemplateId(params.view, content.presentation);
   const contentDebug = resolveContentDebug(params.debug);
 
+  if (hasDedicatedRouteRenderer(activeTemplate)) {
+    return renderDesignRoute(activeTemplate, {
+      content,
+      contentDebug,
+      currentPath: "/",
+      route: "home",
+    });
+  }
+
   if (activeTemplate === "classic") {
     return <ClassicHomeRoute content={content} contentDebug={contentDebug} />;
   }
diff --git a/src/app/projects/[projectId]/page.tsx b/src/app/projects/[projectId]/page.tsx
index 348a969..85d2828 100644
--- a/src/app/projects/[projectId]/page.tsx
+++ b/src/app/projects/[projectId]/page.tsx
@@ -1,6 +1,7 @@
 import { notFound } from "next/navigation";
 import { ProjectDetailView } from "@/components/portfolio/project-detail-view";
 import { PageShell } from "@/components/portfolio/site-shell";
+import { hasDedicatedRouteRenderer, renderDesignRoute } from "@/designs/registry";
 import {
   getPortfolioContent,
   isSitePageEnabled,
@@ -35,6 +36,16 @@ export default async function ProjectDetailPage({
     notFound();
   }
 
+  if (hasDedicatedRouteRenderer(activeTemplate)) {
+    return renderDesignRoute(activeTemplate, {
+      content,
+      contentDebug,
+      currentPath: `/projects/${project.id}`,
+      project,
+      route: "project-detail",
+    });
+  }
+
   return (
     <PageShell
       contentDebug={contentDebug}
diff --git a/src/app/projects/page.tsx b/src/app/projects/page.tsx
index bf048c4..2d92b03 100644
--- a/src/app/projects/page.tsx
+++ b/src/app/projects/page.tsx
@@ -2,6 +2,7 @@ import { PageShell } from "@/components/portfolio/site-shell";
 import { notFound } from "next/navigation";
 import { ClassicProjectsView } from "@/designs/classic/projects/projects-route";
 import { DesignProjectsView } from "@/designs/design/projects/projects-route";
+import { hasDedicatedRouteRenderer, renderDesignRoute } from "@/designs/registry";
 import {
   getPortfolioContent,
   isSitePageEnabled,
@@ -22,6 +23,15 @@ export default async function ProjectsPage({
   const params = searchParams ? await searchParams : {};
   const activeTemplate = resolveHomeTemplateId(params.view, content.presentation);
   const contentDebug = resolveContentDebug(params.debug);
+
+  if (hasDedicatedRouteRenderer(activeTemplate)) {
+    return renderDesignRoute(activeTemplate, {
+      content,
+      contentDebug,
+      currentPath: "/projects",
+      route: "projects",
+    });
+  }
   const pageCopy = content.presentation.pages.projects;
   const featuredProjects = content.projects.filter((project) => project.featured);
   const trackProjects = content.projects.filter((project) => !project.featured);
diff --git a/src/app/resume/page.tsx b/src/app/resume/page.tsx
index 379ffe0..8eeb414 100644
--- a/src/app/resume/page.tsx
+++ b/src/app/resume/page.tsx
@@ -4,6 +4,7 @@ import { ArrowRightIcon } from "@/components/icons";
 import { ContentHint } from "@/components/portfolio/content-hint";
 import { PageShell } from "@/components/portfolio/site-shell";
 import { StackList } from "@/components/portfolio/stack-list";
+import { hasDedicatedRouteRenderer, renderDesignRoute } from "@/designs/registry";
 import {
   getPortfolioContent,
   getResumeProjects,
@@ -25,6 +26,15 @@ export default async function ResumePage({
   const activeTemplate = resolveHomeTemplateId(params.view, content.presentation);
   const contentDebug = resolveContentDebug(params.debug);
 
+  if (hasDedicatedRouteRenderer(activeTemplate)) {
+    return renderDesignRoute(activeTemplate, {
+      content,
+      contentDebug,
+      currentPath: "/resume",
+      route: "resume",
+    });
+  }
+
   const pageCopy = content.presentation.pages.resume;
   const resumeProjects = getResumeProjects(content);
 


## `feat(editorial): renderer를 디자인 registry에 활성화`

diff --git a/src/content/presentation.json b/src/content/presentation.json
index 85b72d2..5658478 100644
--- a/src/content/presentation.json
+++ b/src/content/presentation.json
@@ -1,5 +1,5 @@
 {
-  "defaultHomeTemplate": "design",
+  "defaultHomeTemplate": "editorial",
   "templates": [
     {
       "id": "design",
@@ -10,6 +10,11 @@
       "id": "classic",
       "label": "Classic",
       "description": "A compact terminal-led portfolio with an indexed archive."
+    },
+    {
+      "id": "editorial",
+      "label": "Editorial",
+      "description": "An asymmetric print folio with evidence-led case-study spreads."
     }
   ],
   "ui": {
diff --git a/src/designs/config.ts b/src/designs/config.ts
index 32df799..8582eeb 100644
--- a/src/designs/config.ts
+++ b/src/designs/config.ts
@@ -14,6 +14,10 @@ export const SITE_DESIGNS: SiteDesignDefinition[] = [
     id: "classic",
     swatch: ["#1f2023", "#9cc8b1", "#7aa7ff"],
   },
+  {
+    id: "editorial",
+    swatch: ["#f2ebdd", "#171614", "#d64b32"],
+  },
 ];
 
 export const SITE_DESIGN_IDS = SITE_DESIGNS.map((design) => design.id);
diff --git a/src/designs/editorial/index.ts b/src/designs/editorial/index.ts
new file mode 100644
index 0000000..304b3f0
--- /dev/null
+++ b/src/designs/editorial/index.ts
@@ -0,0 +1,2 @@
+export { EditorialRoute as default, EditorialRoute } from "./editorial-route";
+export type { EditorialRouteName, EditorialRouteProps } from "./editorial-route";
diff --git a/src/designs/registry.tsx b/src/designs/registry.tsx
index 058641e..8545033 100644
--- a/src/designs/registry.tsx
+++ b/src/designs/registry.tsx
@@ -6,7 +6,9 @@ type DesignModule = {
   default: ComponentType<DesignRouteProps>;
 };
 
-const routeLoaders: Partial<Record<SiteDesignId, () => Promise<DesignModule>>> = {};
+const routeLoaders: Partial<Record<SiteDesignId, () => Promise<DesignModule>>> = {
+  editorial: () => import("./editorial"),
+};
 
 export function hasDedicatedRouteRenderer(
   designId: SiteDesignId,
diff --git a/src/lib/content-loader.ts b/src/lib/content-loader.ts
index f5f20e2..c715ca3 100644
--- a/src/lib/content-loader.ts
+++ b/src/lib/content-loader.ts
@@ -40,6 +40,7 @@ export type ContentValidationIssue = {
 const supportedDesignIdList = [
   "design",
   "classic",
+  "editorial",
 ] as const;
 const supportedDesignIds = new Set<string>(supportedDesignIdList);
 


## `feat(designs): Brutalist renderer 활성화`

diff --git a/src/content/presentation.json b/src/content/presentation.json
index 5658478..3338f0d 100644
--- a/src/content/presentation.json
+++ b/src/content/presentation.json
@@ -15,6 +15,11 @@
       "id": "editorial",
       "label": "Editorial",
       "description": "An asymmetric print folio with evidence-led case-study spreads."
+    },
+    {
+      "id": "brutalist",
+      "label": "Brutalist",
+      "description": "An oversized modular grid with hard, expressive contrast."
     }
   ],
   "ui": {
diff --git a/src/designs/brutalist/index.tsx b/src/designs/brutalist/index.tsx
new file mode 100644
index 0000000..77a4161
--- /dev/null
+++ b/src/designs/brutalist/index.tsx
@@ -0,0 +1 @@
+export { BrutalistRoute, BrutalistRoute as default } from "./brutalist-route";
diff --git a/src/designs/config.ts b/src/designs/config.ts
index 8582eeb..c07c670 100644
--- a/src/designs/config.ts
+++ b/src/designs/config.ts
@@ -18,6 +18,10 @@ export const SITE_DESIGNS: SiteDesignDefinition[] = [
     id: "editorial",
     swatch: ["#f2ebdd", "#171614", "#d64b32"],
   },
+  {
+    id: "brutalist",
+    swatch: ["#f4f0e8", "#2e5bff", "#e6ff3f"],
+  },
 ];
 
 export const SITE_DESIGN_IDS = SITE_DESIGNS.map((design) => design.id);
diff --git a/src/designs/registry.tsx b/src/designs/registry.tsx
index 8545033..19b5b93 100644
--- a/src/designs/registry.tsx
+++ b/src/designs/registry.tsx
@@ -8,6 +8,7 @@ type DesignModule = {
 
 const routeLoaders: Partial<Record<SiteDesignId, () => Promise<DesignModule>>> = {
   editorial: () => import("./editorial"),
+  brutalist: () => import("./brutalist"),
 };
 
 export function hasDedicatedRouteRenderer(
diff --git a/src/lib/content-loader.ts b/src/lib/content-loader.ts
index c715ca3..965ea23 100644
--- a/src/lib/content-loader.ts
+++ b/src/lib/content-loader.ts
@@ -41,6 +41,7 @@ const supportedDesignIdList = [
   "design",
   "classic",
   "editorial",
+  "brutalist",
 ] as const;
 const supportedDesignIds = new Set<string>(supportedDesignIdList);
 


## `feat(designs): Cinematic renderer 활성화`

diff --git a/src/content/presentation.json b/src/content/presentation.json
index 3338f0d..9ce2428 100644
--- a/src/content/presentation.json
+++ b/src/content/presentation.json
@@ -20,6 +20,11 @@
       "id": "brutalist",
       "label": "Brutalist",
       "description": "An oversized modular grid with hard, expressive contrast."
+    },
+    {
+      "id": "cinematic",
+      "label": "Cinematic",
+      "description": "A quiet image-led archive arranged as visual chapters."
     }
   ],
   "ui": {
diff --git a/src/designs/cinematic/cinematic-route.tsx b/src/designs/cinematic/cinematic-route.tsx
index 9e20fa3..03d3c0b 100644
--- a/src/designs/cinematic/cinematic-route.tsx
+++ b/src/designs/cinematic/cinematic-route.tsx
@@ -12,16 +12,16 @@ import {
 import type { DesignRouteProps } from "@/designs/types";
 import styles from "./cinematic.module.css";
 
-export function routeHref(href: string, contentDebug = false) {
+function routeHref(href: string, contentDebug = false) {
   return getTemplateHref(href, "cinematic", { contentDebug });
 }
 
-export function isCurrentNavigation(href: string, currentPath: string) {
+function isCurrentNavigation(href: string, currentPath: string) {
   if (href === "/") return currentPath === "/";
   return currentPath === href || currentPath.startsWith(`${href}/`);
 }
 
-export function CinematicLink({
+function CinematicLink({
   children,
   className,
   contentDebug,
@@ -56,7 +56,7 @@ export function CinematicLink({
   );
 }
 
-export function LinkList({
+function LinkList({
   contentDebug,
   links,
 }: {
@@ -88,7 +88,7 @@ export function LinkList({
   );
 }
 
-export function Frame({
+function Frame({
   children,
   content,
   contentDebug,
@@ -156,7 +156,7 @@ export function Frame({
   );
 }
 
-export function Media({
+function Media({
   alt,
   priority = false,
   src,
@@ -172,7 +172,7 @@ export function Media({
   );
 }
 
-export function ChapterLabel({ children, index }: { children: React.ReactNode; index: number }) {
+function ChapterLabel({ children, index }: { children: React.ReactNode; index: number }) {
   return (
     <p className={styles.chapterLabel}>
       <span>{String(index).padStart(2, "0")}</span>
@@ -181,7 +181,7 @@ export function ChapterLabel({ children, index }: { children: React.ReactNode; i
   );
 }
 
-export function ProjectChapter({
+function ProjectChapter({
   actionLabel,
   index,
   openItemAriaTemplate,
@@ -216,7 +216,7 @@ export function ProjectChapter({
   );
 }
 
-export function HomeView({ content, contentDebug }: DesignRouteProps) {
+function HomeView({ content, contentDebug }: DesignRouteProps) {
   const featured = content.projects.filter((project) => project.featured);
   const lead = featured[0] ?? content.projects[0];
   const copy = content.presentation.home.cinematic;
@@ -299,7 +299,7 @@ export function HomeView({ content, contentDebug }: DesignRouteProps) {
   );
 }
 
-export function ProjectsView({ content, contentDebug }: DesignRouteProps) {
+function ProjectsView({ content, contentDebug }: DesignRouteProps) {
   const copy = content.presentation.pages.projects.cinematic.hero;
   const homeCopy = content.presentation.home.cinematic;
 
@@ -327,7 +327,7 @@ export function ProjectsView({ content, contentDebug }: DesignRouteProps) {
   );
 }
 
-export function ProjectDetailView({ content, contentDebug, project }: DesignRouteProps) {
+function ProjectDetailView({ content, contentDebug, project }: DesignRouteProps) {
   const copy = content.presentation.pages.projectDetail;
 
   if (!project) {
@@ -435,7 +435,7 @@ export function ProjectDetailView({ content, contentDebug, project }: DesignRout
   );
 }
 
-export function AboutView({ content, contentDebug }: DesignRouteProps) {
+function AboutView({ content, contentDebug }: DesignRouteProps) {
   const copy = content.presentation.pages.about;
   const curation = content.curation;
 
@@ -565,8 +565,7 @@ export function AboutView({ content, contentDebug }: DesignRouteProps) {
   );
 }
 
-
-export function ResumeView({ content, contentDebug }: DesignRouteProps) {
+function ResumeView({ content, contentDebug }: DesignRouteProps) {
   const copy = content.presentation.pages.resume;
   const selected = content.resume.projectIds
     .map((id) => content.projects.find((project) => project.id === id))
@@ -620,7 +619,7 @@ export function ResumeView({ content, contentDebug }: DesignRouteProps) {
   );
 }
 
-export function ContactView({ content, contentDebug }: DesignRouteProps) {
+function ContactView({ content, contentDebug }: DesignRouteProps) {
   const linksById = new Map(content.links.map((link) => [link.id, link]));
   const preferred = content.contact.preferred
     .map((id) => linksById.get(id))
@@ -645,8 +644,7 @@ export function ContactView({ content, contentDebug }: DesignRouteProps) {
   );
 }
 
-
-export function JourneyView({ content, contentDebug }: DesignRouteProps) {
+function JourneyView({ content, contentDebug }: DesignRouteProps) {
   const copy = content.presentation.pages.journey;
   const narrative = content.journeyNarrative;
 
@@ -732,8 +730,7 @@ export function JourneyView({ content, contentDebug }: DesignRouteProps) {
   );
 }
 
-
-export function InterviewMapView({ content, contentDebug }: DesignRouteProps) {
+function InterviewMapView({ content, contentDebug }: DesignRouteProps) {
   const copy = content.presentation.pages.interviewMap;
   const data = content.interviewMap;
   const projectsById = new Map(content.projects.map((project) => [project.id, project]));
@@ -817,3 +814,24 @@ export function InterviewMapView({ content, contentDebug }: DesignRouteProps) {
     </>
   );
 }
+
+function RouteView(props: DesignRouteProps) {
+  switch (props.route) {
+    case "home": return <HomeView {...props} />;
+    case "projects": return <ProjectsView {...props} />;
+    case "project-detail": return <ProjectDetailView {...props} />;
+    case "about": return <AboutView {...props} />;
+    case "resume": return <ResumeView {...props} />;
+    case "contact": return <ContactView {...props} />;
+    case "journey": return <JourneyView {...props} />;
+    case "interview-map": return <InterviewMapView {...props} />;
+  }
+}
+
+export function CinematicRoute(props: DesignRouteProps) {
+  return (
+    <Frame {...props}>
+      <RouteView {...props} />
+    </Frame>
+  );
+}
diff --git a/src/designs/cinematic/index.ts b/src/designs/cinematic/index.ts
new file mode 100644
index 0000000..e3abbc6
--- /dev/null
+++ b/src/designs/cinematic/index.ts
@@ -0,0 +1 @@
+export { CinematicRoute as default, CinematicRoute } from "./cinematic-route";
diff --git a/src/designs/config.ts b/src/designs/config.ts
index c07c670..5e9e0ad 100644
--- a/src/designs/config.ts
+++ b/src/designs/config.ts
@@ -22,6 +22,10 @@ export const SITE_DESIGNS: SiteDesignDefinition[] = [
     id: "brutalist",
     swatch: ["#f4f0e8", "#2e5bff", "#e6ff3f"],
   },
+  {
+    id: "cinematic",
+    swatch: ["#0a0b0e", "#d9d2c4", "#c98a4a"],
+  },
 ];
 
 export const SITE_DESIGN_IDS = SITE_DESIGNS.map((design) => design.id);
diff --git a/src/designs/registry.tsx b/src/designs/registry.tsx
index 19b5b93..151c916 100644
--- a/src/designs/registry.tsx
+++ b/src/designs/registry.tsx
@@ -9,6 +9,7 @@ type DesignModule = {
 const routeLoaders: Partial<Record<SiteDesignId, () => Promise<DesignModule>>> = {
   editorial: () => import("./editorial"),
   brutalist: () => import("./brutalist"),
+  cinematic: () => import("./cinematic"),
 };
 
 export function hasDedicatedRouteRenderer(
diff --git a/src/lib/content-loader.ts b/src/lib/content-loader.ts
index 965ea23..24b9138 100644
--- a/src/lib/content-loader.ts
+++ b/src/lib/content-loader.ts
@@ -42,6 +42,7 @@ const supportedDesignIdList = [
   "classic",
   "editorial",
   "brutalist",
+  "cinematic",
 ] as const;
 const supportedDesignIds = new Set<string>(supportedDesignIdList);
 


