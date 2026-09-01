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
 
