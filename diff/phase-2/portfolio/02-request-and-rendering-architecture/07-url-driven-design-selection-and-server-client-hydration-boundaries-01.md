# URL 기반 디자인 선택과 Server/Client Hydration 경계

## `feat(navigation): 템플릿 URL과 쿼리 해석 추가`

diff --git a/src/lib/portfolio/selectors.ts b/src/lib/portfolio/selectors.ts
new file mode 100644
index 0000000..08e613c
--- /dev/null
+++ b/src/lib/portfolio/selectors.ts
@@ -0,0 +1,39 @@
+import { portfolioPresentation } from "./content";
+import type {
+  HomeTemplateId,
+  PresentationContent,
+} from "./types";
+import { createTemplateHref } from "./template-href";
+
+export function resolveHomeTemplateId(
+  value: string | string[] | undefined,
+  content: PresentationContent = portfolioPresentation,
+): HomeTemplateId {
+  const templateId = Array.isArray(value) ? value[0] : value;
+
+  if (
+    templateId &&
+    content.templates.some((template) => template.id === templateId)
+  ) {
+    return templateId as HomeTemplateId;
+  }
+
+  return content.defaultHomeTemplate;
+}
+
+export function resolveContentDebug(value: string | string[] | undefined) {
+  return (Array.isArray(value) ? value[0] : value) === "content";
+}
+
+export function getTemplateHref(
+  href: string,
+  templateId?: HomeTemplateId,
+  options: { alwaysInclude?: boolean; contentDebug?: boolean } = {},
+) {
+  return createTemplateHref(
+    href,
+    templateId,
+    portfolioPresentation.defaultHomeTemplate,
+    options,
+  );
+}
diff --git a/src/lib/portfolio/template-href.ts b/src/lib/portfolio/template-href.ts
new file mode 100644
index 0000000..1c8bd2d
--- /dev/null
+++ b/src/lib/portfolio/template-href.ts
@@ -0,0 +1,34 @@
+import type { HomeTemplateId } from "./types";
+
+export function createTemplateHref(
+  href: string,
+  templateId: HomeTemplateId | undefined,
+  defaultTemplateId: HomeTemplateId,
+  options: { alwaysInclude?: boolean; contentDebug?: boolean } = {},
+) {
+  if (!templateId || !href.startsWith("/") || href.startsWith("//")) {
+    return href;
+  }
+
+  const hashIndex = href.indexOf("#");
+  const withoutHash = hashIndex === -1 ? href : href.slice(0, hashIndex);
+  const hash = hashIndex === -1 ? "" : href.slice(hashIndex);
+  const [pathname, query] = withoutHash.split("?", 2);
+  const params = new URLSearchParams(query);
+  const shouldIncludeView =
+    options.alwaysInclude || templateId !== defaultTemplateId;
+
+  if (shouldIncludeView) {
+    params.set("view", templateId);
+  } else {
+    params.delete("view");
+  }
+
+  if (options.contentDebug) {
+    params.set("debug", "content");
+  }
+
+  const queryString = params.toString();
+
+  return `${pathname}${queryString ? `?${queryString}` : ""}${hash}`;
+}


## `feat(home): 쿼리 기반 디자인 전환 연결`

diff --git a/src/app/page.tsx b/src/app/page.tsx
index 64965b9..d7debcc 100644
--- a/src/app/page.tsx
+++ b/src/app/page.tsx
@@ -1,8 +1,25 @@
+import { ClassicHomeRoute } from "@/designs/classic/home-route";
 import { DesignHomeRoute } from "@/designs/design/home-route";
-import { getPortfolioContent } from "@/lib/portfolio";
+import {
+  getPortfolioContent,
+  resolveContentDebug,
+  resolveHomeTemplateId,
+  type RouteSearchParams,
+} from "@/lib/portfolio";
 
-export default function Home() {
+type HomePageProps = {
+  searchParams?: RouteSearchParams;
+};
+
+export default async function Home({ searchParams }: HomePageProps) {
   const content = getPortfolioContent();
+  const params = searchParams ? await searchParams : {};
+  const activeTemplate = resolveHomeTemplateId(params.view, content.presentation);
+  const contentDebug = resolveContentDebug(params.debug);
+
+  if (activeTemplate === "classic") {
+    return <ClassicHomeRoute content={content} contentDebug={contentDebug} />;
+  }
 
-  return <DesignHomeRoute content={content} contentDebug={false} />;
+  return <DesignHomeRoute content={content} contentDebug={contentDebug} />;
 }
diff --git a/src/designs/classic/home-route.tsx b/src/designs/classic/home-route.tsx
index 50bbfe1..028e79d 100644
--- a/src/designs/classic/home-route.tsx
+++ b/src/designs/classic/home-route.tsx
@@ -22,6 +22,12 @@ export function ClassicHomeRoute({
       homeTemplate={activeTemplate}
       profile={content.profile}
       site={content.site}
+      templateSwitcher={{
+        activeId: activeTemplate,
+        contentDebug,
+        currentPath: "/",
+        templates: content.presentation.templates,
+      }}
     >
       <ClassicHeroSection
         activeTemplate={activeTemplate}
diff --git a/src/designs/design/home-route.tsx b/src/designs/design/home-route.tsx
index e359880..de98fde 100644
--- a/src/designs/design/home-route.tsx
+++ b/src/designs/design/home-route.tsx
@@ -29,6 +29,12 @@ export function DesignHomeRoute({
       homeTemplate={activeTemplate}
       profile={content.profile}
       site={content.site}
+      templateSwitcher={{
+        activeId: activeTemplate,
+        contentDebug,
+        currentPath: "/",
+        templates: content.presentation.templates,
+      }}
     >
       <HeroSection
         activeTemplate={activeTemplate}


## `feat(designs): 디자인 선택기 상태와 trigger 추가`

diff --git a/src/components/portfolio/design-switcher.tsx b/src/components/portfolio/design-switcher.tsx
new file mode 100644
index 0000000..169f660
--- /dev/null
+++ b/src/components/portfolio/design-switcher.tsx
@@ -0,0 +1,54 @@
+"use client";
+
+import Link from "next/link";
+import { useRef } from "react";
+import { SITE_DESIGNS } from "@/designs/config";
+import {
+  getTemplateHref,
+  type PresentationContent,
+  type PresentationTemplate,
+  type SiteDesignId,
+} from "@/lib/portfolio";
+import styles from "./design-switcher.module.css";
+
+export function DesignSwitcher({
+  activeId,
+  contentDebug,
+  currentPath,
+  templates,
+  ui,
+}: {
+  activeId: SiteDesignId;
+  contentDebug?: boolean;
+  currentPath: string;
+  templates: PresentationTemplate[];
+  ui: PresentationContent["ui"];
+}) {
+  const detailsRef = useRef<HTMLDetailsElement>(null);
+  const summaryRef = useRef<HTMLElement>(null);
+  const templateCopy = new Map(
+    templates.map((template) => [template.id, template]),
+  );
+  const activeIndex = SITE_DESIGNS.findIndex((design) => design.id === activeId);
+  const active = SITE_DESIGNS[activeIndex] ?? SITE_DESIGNS[0];
+  const activeCopy = templateCopy.get(active.id);
+  const activeLabel = activeCopy?.label ?? active.id;
+  const countLabel = ui.designSwitcherCountTemplate
+    .replace("{index}", String(activeIndex + 1).padStart(2, "0"))
+    .replace("{total}", String(SITE_DESIGNS.length).padStart(2, "0"));
+
+  return (
+    <details className={styles.root} ref={detailsRef}>
+      <summary
+        aria-label={ui.designSwitcherAriaTemplate.replace(
+          "{label}",
+          activeLabel,
+        )}
+        ref={summaryRef}
+      >
+        <span className={styles.count}>{countLabel}</span>
+        <span className={styles.label}>{activeLabel}</span>
+      </summary>
+    </details>
+  );
+}


## `feat(designs): 디자인 선택 목록과 닫기 동작 추가`

diff --git a/src/components/portfolio/design-switcher.tsx b/src/components/portfolio/design-switcher.tsx
index 169f660..46e522a 100644
--- a/src/components/portfolio/design-switcher.tsx
+++ b/src/components/portfolio/design-switcher.tsx
@@ -49,6 +49,53 @@ export function DesignSwitcher({
         <span className={styles.count}>{countLabel}</span>
         <span className={styles.label}>{activeLabel}</span>
       </summary>
+      <nav aria-label={ui.designNavigationAriaLabel} className={styles.panel}>
+        <div className={styles.sheetHeader}>
+          <strong>{ui.designNavigationAriaLabel}</strong>
+          <button
+            aria-label={ui.designSwitcherCloseLabel}
+            onClick={() => {
+              detailsRef.current?.removeAttribute("open");
+              summaryRef.current?.focus();
+            }}
+            type="button"
+          >
+            <span aria-hidden="true">×</span>
+          </button>
+        </div>
+        <ul className={styles.list}>
+          {SITE_DESIGNS.map((design, index) => {
+            const copy = templateCopy.get(design.id);
+            const isActive = design.id === activeId;
+
+            return (
+              <li key={design.id}>
+                <Link
+                  aria-current={isActive ? "page" : undefined}
+                  className={`${styles.link} ${isActive ? styles.active : ""}`}
+                  href={getTemplateHref(currentPath, design.id, {
+                    contentDebug,
+                  })}
+                  onClick={() => detailsRef.current?.removeAttribute("open")}
+                >
+                  <span aria-hidden="true" className={styles.swatch}>
+                    {design.swatch.map((color) => (
+                      <span key={color} style={{ background: color }} />
+                    ))}
+                  </span>
+                  <span className={styles.copy}>
+                    <strong>{copy?.label ?? design.id}</strong>
+                    <small>{copy?.description}</small>
+                  </span>
+                  <span className={styles.number}>
+                    {String(index + 1).padStart(2, "0")}
+                  </span>
+                </Link>
+              </li>
+            );
+          })}
+        </ul>
+      </nav>
     </details>
   );
 }


## `feat(shell): 디자인 선택기를 공용 shell에 연결`

diff --git a/src/app/about/page.tsx b/src/app/about/page.tsx
index c9646f5..79d4a7a 100644
--- a/src/app/about/page.tsx
+++ b/src/app/about/page.tsx
@@ -38,6 +38,7 @@ export default async function AboutPage({
       homeTemplate={activeTemplate}
       profile={content.profile}
       site={content.site}
+      ui={content.presentation.ui}
       templateSwitcher={{
         activeId: activeTemplate,
         contentDebug,
diff --git a/src/app/contact/page.tsx b/src/app/contact/page.tsx
index e09d603..042c9d6 100644
--- a/src/app/contact/page.tsx
+++ b/src/app/contact/page.tsx
@@ -32,6 +32,7 @@ export default async function ContactPage({
       homeTemplate={activeTemplate}
       profile={content.profile}
       site={content.site}
+      ui={content.presentation.ui}
       templateSwitcher={{
         activeId: activeTemplate,
         contentDebug,
diff --git a/src/app/interview-map/page.tsx b/src/app/interview-map/page.tsx
index 3bafca8..f1c509e 100644
--- a/src/app/interview-map/page.tsx
+++ b/src/app/interview-map/page.tsx
@@ -36,6 +36,7 @@ export default async function InterviewMapPage({
       homeTemplate={activeTemplate}
       profile={content.profile}
       site={content.site}
+      ui={content.presentation.ui}
       templateSwitcher={{
         activeId: activeTemplate,
         contentDebug,
diff --git a/src/app/journey/page.tsx b/src/app/journey/page.tsx
index de403ca..37419ad 100644
--- a/src/app/journey/page.tsx
+++ b/src/app/journey/page.tsx
@@ -38,6 +38,7 @@ export default async function JourneyPage({
       homeTemplate={activeTemplate}
       profile={content.profile}
       site={content.site}
+      ui={content.presentation.ui}
       templateSwitcher={{
         activeId: activeTemplate,
         contentDebug,
diff --git a/src/app/projects/[projectId]/page.tsx b/src/app/projects/[projectId]/page.tsx
index 5c5ac53..348a969 100644
--- a/src/app/projects/[projectId]/page.tsx
+++ b/src/app/projects/[projectId]/page.tsx
@@ -41,6 +41,7 @@ export default async function ProjectDetailPage({
       homeTemplate={activeTemplate}
       profile={content.profile}
       site={content.site}
+      ui={content.presentation.ui}
       templateSwitcher={{
         activeId: activeTemplate,
         contentDebug,
diff --git a/src/app/projects/page.tsx b/src/app/projects/page.tsx
index b82d1f6..bf048c4 100644
--- a/src/app/projects/page.tsx
+++ b/src/app/projects/page.tsx
@@ -33,6 +33,7 @@ export default async function ProjectsPage({
     homeTemplate: activeTemplate,
     profile: content.profile,
     site: content.site,
+    ui: content.presentation.ui,
     templateSwitcher: {
       activeId: activeTemplate,
       contentDebug,
diff --git a/src/app/resume/page.tsx b/src/app/resume/page.tsx
index 6434fad..379ffe0 100644
--- a/src/app/resume/page.tsx
+++ b/src/app/resume/page.tsx
@@ -34,6 +34,7 @@ export default async function ResumePage({
       homeTemplate={activeTemplate}
       profile={content.profile}
       site={content.site}
+      ui={content.presentation.ui}
       templateSwitcher={{
         activeId: activeTemplate,
         contentDebug,
diff --git a/src/components/portfolio/site-shell.tsx b/src/components/portfolio/site-shell.tsx
index fa3a991..e9bc83a 100644
--- a/src/components/portfolio/site-shell.tsx
+++ b/src/components/portfolio/site-shell.tsx
@@ -1,8 +1,10 @@
 import Link from "next/link";
 import { ArrowRightIcon } from "@/components/icons";
+import { DesignSwitcher } from "@/components/portfolio/design-switcher";
 import {
   getTemplateHref,
   type HomeTemplateId,
+  type PresentationContent,
   type PresentationTemplate,
   type ProfileContent,
   type SiteContent,
@@ -25,10 +27,12 @@ export function SiteHeader({
   templateSwitcher,
   profile,
   site,
+  ui,
 }: {
   templateSwitcher?: TemplateSwitcherProps;
   profile: ProfileContent;
   site: SiteContent;
+  ui: PresentationContent["ui"];
 }) {
   return (
     <header
@@ -45,7 +49,7 @@ export function SiteHeader({
           {profile.handle}
         </Link>
         <nav
-          aria-label="Primary navigation"
+          aria-label={ui.primaryNavigationAriaLabel}
           className="hidden items-center gap-6 md:flex"
         >
           {site.navigation.map((item) => (
@@ -67,10 +71,10 @@ export function SiteHeader({
         </nav>
         <details className="relative md:hidden">
           <summary className="flex min-h-11 cursor-pointer list-none items-center rounded border border-line px-3 text-xs font-semibold uppercase tracking-wide text-foreground focus:outline-none focus-visible:ring-2 focus-visible:ring-accent">
-            Menu
+            {ui.menuLabel}
           </summary>
           <nav
-            aria-label="Mobile navigation"
+            aria-label={ui.mobileNavigationAriaLabel}
             className="absolute right-0 top-[calc(100%+0.5rem)] grid min-w-52 gap-1 border border-line bg-surface p-2 shadow-xl"
           >
             {site.navigation.map((item) => (
@@ -91,6 +95,15 @@ export function SiteHeader({
             ))}
           </nav>
         </details>
+        {templateSwitcher ? (
+          <DesignSwitcher
+            activeId={templateSwitcher.activeId}
+            contentDebug={templateSwitcher.contentDebug}
+            currentPath={templateSwitcher.currentPath}
+            templates={templateSwitcher.templates}
+            ui={ui}
+          />
+        ) : null}
       </div>
     </header>
   );
@@ -128,6 +141,7 @@ export function PageShell({
   profile,
   site,
   templateSwitcher,
+  ui,
 }: {
   children: React.ReactNode;
   contentDebug?: boolean;
@@ -135,24 +149,33 @@ export function PageShell({
   profile: ProfileContent;
   site: SiteContent;
   templateSwitcher?: TemplateSwitcherProps;
+  ui: PresentationContent["ui"];
 }) {
   return (
-    <main
+    <div
       className="min-h-screen bg-background text-foreground"
-      data-home-template={homeTemplate}
       data-site-design={homeTemplate}
     >
+      <a
+        className="fixed left-4 top-[-5rem] z-[100] bg-foreground px-4 py-3 text-sm font-semibold text-background focus:top-4"
+        href="#main-content"
+      >
+        {ui.skipLinkLabel}
+      </a>
       <SiteHeader
         profile={profile}
         site={site}
         templateSwitcher={templateSwitcher}
+        ui={ui}
       />
-      {children}
+      <main data-home-template={homeTemplate} id="main-content">
+        {children}
+      </main>
       <SiteFooter
         contentDebug={contentDebug}
         homeTemplate={homeTemplate}
         site={site}
       />
-    </main>
+    </div>
   );
 }
diff --git a/src/designs/classic/home-route.tsx b/src/designs/classic/home-route.tsx
index ae7dbe7..4e43cdb 100644
--- a/src/designs/classic/home-route.tsx
+++ b/src/designs/classic/home-route.tsx
@@ -34,6 +34,7 @@ export function ClassicHomeRoute({
       homeTemplate={activeTemplate}
       profile={content.profile}
       site={content.site}
+      ui={content.presentation.ui}
       templateSwitcher={{
         activeId: activeTemplate,
         contentDebug,
diff --git a/src/designs/design/home-route.tsx b/src/designs/design/home-route.tsx
index 1767bce..542b88c 100644
--- a/src/designs/design/home-route.tsx
+++ b/src/designs/design/home-route.tsx
@@ -39,6 +39,7 @@ export function DesignHomeRoute({
       homeTemplate={activeTemplate}
       profile={content.profile}
       site={content.site}
+      ui={content.presentation.ui}
       templateSwitcher={{
         activeId: activeTemplate,
         contentDebug,


## `refactor(routes): 홈 page context 통합`

diff --git a/src/app/page.tsx b/src/app/page.tsx
index 465fbd6..3cc1033 100644
--- a/src/app/page.tsx
+++ b/src/app/page.tsx
@@ -1,22 +1,19 @@
 import { ClassicHomeRoute } from "@/designs/classic/home-route";
 import { DesignHomeRoute } from "@/designs/design/home-route";
 import { hasDedicatedRouteRenderer, renderDesignRoute } from "@/designs/registry";
-import {
-  getPortfolioContent,
-  resolveContentDebug,
-  resolveHomeTemplateId,
-  type RouteSearchParams,
-} from "@/lib/portfolio";
+import { type RouteSearchParams } from "@/lib/portfolio";
+import { resolvePortfolioPageContext } from "@/lib/portfolio/page-context";
 
 type HomePageProps = {
   searchParams?: RouteSearchParams;
 };
 
 export default async function Home({ searchParams }: HomePageProps) {
-  const content = getPortfolioContent();
-  const params = searchParams ? await searchParams : {};
-  const activeTemplate = resolveHomeTemplateId(params.view, content.presentation);
-  const contentDebug = resolveContentDebug(params.debug);
+  const { activeTemplate, content, contentDebug } =
+    await resolvePortfolioPageContext({
+      currentPath: "/",
+      searchParams,
+    });
 
   if (hasDedicatedRouteRenderer(activeTemplate)) {
     return renderDesignRoute(activeTemplate, {
diff --git a/src/lib/portfolio/page-context.ts b/src/lib/portfolio/page-context.ts
new file mode 100644
index 0000000..2a807aa
--- /dev/null
+++ b/src/lib/portfolio/page-context.ts
@@ -0,0 +1,52 @@
+import { getPortfolioContent } from "./content";
+import { resolveContentDebug, resolveHomeTemplateId } from "./selectors";
+import type {
+  PortfolioContent,
+  RouteSearchParams,
+} from "./types";
+
+export type PortfolioPagePath =
+  | "/"
+  | "/about"
+  | "/contact"
+  | "/interview-map"
+  | "/journey"
+  | "/projects"
+  | "/resume"
+  | `/projects/${string}`;
+
+export async function resolvePortfolioPageContext({
+  content = getPortfolioContent(),
+  currentPath,
+  searchParams,
+}: {
+  content?: PortfolioContent;
+  currentPath: PortfolioPagePath;
+  searchParams?: RouteSearchParams;
+}) {
+  const query = searchParams ? await searchParams : {};
+  const activeTemplate = resolveHomeTemplateId(
+    query.view,
+    content.presentation,
+  );
+  const contentDebug = resolveContentDebug(query.debug);
+
+  return {
+    activeTemplate,
+    content,
+    contentDebug,
+    shellProps: {
+      contentDebug,
+      homeTemplate: activeTemplate,
+      profile: content.profile,
+      site: content.site,
+      ui: content.presentation.ui,
+      templateSwitcher: {
+        activeId: activeTemplate,
+        contentDebug,
+        currentPath,
+        templates: content.presentation.templates,
+      },
+    },
+  };
+}


## `refactor(projects): 프로젝트 page context 통합`

diff --git a/src/app/projects/[projectId]/page.tsx b/src/app/projects/[projectId]/page.tsx
index 85d2828..7161076 100644
--- a/src/app/projects/[projectId]/page.tsx
+++ b/src/app/projects/[projectId]/page.tsx
@@ -6,10 +6,9 @@ import {
   getPortfolioContent,
   isSitePageEnabled,
   getProjectById,
-  resolveContentDebug,
-  resolveHomeTemplateId,
   type RouteSearchParams,
 } from "@/lib/portfolio";
+import { resolvePortfolioPageContext } from "@/lib/portfolio/page-context";
 
 export function generateStaticParams() {
   return getPortfolioContent().projects.map((project) => ({
@@ -27,9 +26,12 @@ export default async function ProjectDetailPage({
   const content = getPortfolioContent();
   if (!isSitePageEnabled("projects", content)) notFound();
   const { projectId } = await params;
-  const query = searchParams ? await searchParams : {};
-  const activeTemplate = resolveHomeTemplateId(query.view, content.presentation);
-  const contentDebug = resolveContentDebug(query.debug);
+  const { activeTemplate, contentDebug, shellProps } =
+    await resolvePortfolioPageContext({
+      content,
+      currentPath: `/projects/${projectId}`,
+      searchParams,
+    });
   const project = getProjectById(projectId, content);
 
   if (!project) {
@@ -47,19 +49,7 @@ export default async function ProjectDetailPage({
   }
 
   return (
-    <PageShell
-      contentDebug={contentDebug}
-      homeTemplate={activeTemplate}
-      profile={content.profile}
-      site={content.site}
-      ui={content.presentation.ui}
-      templateSwitcher={{
-        activeId: activeTemplate,
-        contentDebug,
-        currentPath: `/projects/${project.id}`,
-        templates: content.presentation.templates,
-      }}
-    >
+    <PageShell {...shellProps}>
       <ProjectDetailView
         contentDebug={contentDebug}
         homeTemplate={activeTemplate}
diff --git a/src/app/projects/page.tsx b/src/app/projects/page.tsx
index 2d92b03..516190b 100644
--- a/src/app/projects/page.tsx
+++ b/src/app/projects/page.tsx
@@ -7,10 +7,9 @@ import {
   getPortfolioContent,
   isSitePageEnabled,
   getProjectMetricValue,
-  resolveContentDebug,
-  resolveHomeTemplateId,
   type RouteSearchParams,
 } from "@/lib/portfolio";
+import { resolvePortfolioPageContext } from "@/lib/portfolio/page-context";
 import { groupProjects } from "@/lib/portfolio/project-groups";
 
 export default async function ProjectsPage({
@@ -20,9 +19,12 @@ export default async function ProjectsPage({
 }) {
   const content = getPortfolioContent();
   if (!isSitePageEnabled("projects", content)) notFound();
-  const params = searchParams ? await searchParams : {};
-  const activeTemplate = resolveHomeTemplateId(params.view, content.presentation);
-  const contentDebug = resolveContentDebug(params.debug);
+  const { activeTemplate, contentDebug, shellProps } =
+    await resolvePortfolioPageContext({
+      content,
+      currentPath: "/projects",
+      searchParams,
+    });
 
   if (hasDedicatedRouteRenderer(activeTemplate)) {
     return renderDesignRoute(activeTemplate, {
@@ -38,20 +40,6 @@ export default async function ProjectsPage({
   const groupedProjects = groupProjects(trackProjects, pageCopy.groups);
   const sourceOnlyCount = getProjectMetricValue("sourceOnlyCount", content);
   const curriculumCount = getProjectMetricValue("curriculumCount", content);
-  const shellProps = {
-    contentDebug,
-    homeTemplate: activeTemplate,
-    profile: content.profile,
-    site: content.site,
-    ui: content.presentation.ui,
-    templateSwitcher: {
-      activeId: activeTemplate,
-      contentDebug,
-      currentPath: "/projects",
-      templates: content.presentation.templates,
-    },
-  };
-
   if (activeTemplate === "classic") {
     return (
       <PageShell {...shellProps}>


