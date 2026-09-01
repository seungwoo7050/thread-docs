## `refactor(design): Design route dispatcher 추가`

diff --git a/src/app/projects/page.tsx b/src/app/projects/page.tsx
index 057e78e..2436707 100644
--- a/src/app/projects/page.tsx
+++ b/src/app/projects/page.tsx
@@ -2,7 +2,7 @@ import type { Metadata } from "next";
 import { PageShell } from "@/components/portfolio/site-shell";
 import { notFound } from "next/navigation";
 import { ClassicProjectsView } from "@/designs/classic/projects/projects-route";
-import DesignProjectsRoute from "@/designs/design/projects/projects-route";
+import DesignProjectsRoute from "@/designs/design/projects-route";
 import { hasDedicatedRouteRenderer, renderDesignRoute } from "@/designs/registry";
 import {
   getPortfolioContent,
diff --git a/src/designs/design/index.tsx b/src/designs/design/index.tsx
new file mode 100644
index 0000000..b0b70c8
--- /dev/null
+++ b/src/designs/design/index.tsx
@@ -0,0 +1,31 @@
+import type { PreparedDesignRouteProps as DesignRouteProps } from "@/designs/shell-props";
+
+import AboutRoute from "./about-route";
+import ContactRoute from "./contact-route";
+import HomeRoute from "./home-route";
+import InterviewMapRoute from "./interview-map-route";
+import JourneyRoute from "./journey-route";
+import ProjectDetailRoute from "./project-detail-route";
+import ProjectsRoute from "./projects-route";
+import ResumeRoute from "./resume-route";
+
+export default function DesignRoute(props: DesignRouteProps) {
+  switch (props.route) {
+    case "home":
+      return <HomeRoute {...props} />;
+    case "projects":
+      return <ProjectsRoute {...props} />;
+    case "project-detail":
+      return <ProjectDetailRoute {...props} />;
+    case "about":
+      return <AboutRoute {...props} />;
+    case "resume":
+      return <ResumeRoute {...props} />;
+    case "contact":
+      return <ContactRoute {...props} />;
+    case "journey":
+      return <JourneyRoute {...props} />;
+    case "interview-map":
+      return <InterviewMapRoute {...props} />;
+  }
+}
diff --git a/src/designs/design/projects-route.tsx b/src/designs/design/projects-route.tsx
new file mode 100644
index 0000000..65deb8a
--- /dev/null
+++ b/src/designs/design/projects-route.tsx
@@ -0,0 +1,163 @@
+import { ContentHint } from "@/components/portfolio/content-hint";
+import { ProjectCard } from "@/components/portfolio/project-card";
+import { PageShell } from "@/components/portfolio/site-shell";
+import {
+  type HomeTemplateId,
+  type ProjectPageContent,
+  type PortfolioProject,
+} from "@/lib/portfolio";
+import type { PreparedDesignRouteProps as DesignRouteProps } from "@/designs/shell-props";
+import { createDesignShellProps } from "@/designs/shell-props";
+
+export default function ProjectsRoute({
+  content,
+  contentDebug,
+  currentPath,
+}: DesignRouteProps) {
+  if (content.route !== "projects") return null;
+
+  const activeTemplate = "design";
+  const shellProps = createDesignShellProps(
+    content,
+    contentDebug,
+    currentPath,
+    activeTemplate,
+  );
+  const pageCopy = content.presentation.pages.projects;
+  const featuredProjects = content.featuredProjects;
+  const groupedProjects = content.archiveGroupEntries;
+  const sourceOnlyCount = content.metricValues.sourceOnlyCount ?? 0;
+  const curriculumCount = content.metricValues.curriculumCount ?? 0;
+
+  return (
+    <PageShell {...shellProps}>
+      <DesignProjectsView
+        activeTemplate={activeTemplate}
+        contentDebug={contentDebug}
+        curriculumCount={curriculumCount}
+        featuredProjects={featuredProjects}
+        groupedProjects={groupedProjects}
+        pageCopy={pageCopy}
+        projects={content.projects}
+        sourceOnlyCount={sourceOnlyCount}
+      />
+    </PageShell>
+  );
+}
+
+function DesignProjectsView({
+  activeTemplate,
+  contentDebug,
+  curriculumCount,
+  featuredProjects,
+  groupedProjects,
+  pageCopy,
+  projects,
+  sourceOnlyCount,
+}: {
+  activeTemplate: HomeTemplateId;
+  contentDebug: boolean;
+  curriculumCount: number;
+  featuredProjects: PortfolioProject[];
+  groupedProjects: [string, PortfolioProject[]][];
+  pageCopy: ProjectPageContent;
+  projects: PortfolioProject[];
+  sourceOnlyCount: number;
+}) {
+  const copy = pageCopy.design;
+  const groupCopy = new Map(pageCopy.groups.map((group) => [group.category, group.body]));
+
+  return (
+    <>
+      <section className="border-b border-line">
+        <div className="mx-auto max-w-6xl px-5 py-20 sm:px-8">
+          <ContentHint
+            enabled={contentDebug}
+            path="src/content/projects.json > projects[]"
+          />
+          <p className="text-sm font-medium text-muted">
+            {projects.length} {copy.hero.stats.visibleEntries} · {curriculumCount} {copy.hero.stats.archive} · {sourceOnlyCount} {copy.hero.stats.sourceFirst}
+          </p>
+          <h1 className="mt-5 max-w-3xl text-5xl font-semibold leading-tight text-foreground md:text-6xl">
+            {copy.hero.title}
+          </h1>
+          <p className="mt-6 max-w-2xl text-base leading-7 text-muted">
+            {copy.hero.body}
+          </p>
+        </div>
+      </section>
+      <section className="border-b border-line bg-background-soft">
+        <div className="mx-auto grid max-w-6xl gap-6 px-5 py-14 sm:px-8">
+          <div className="flex flex-col gap-3 sm:flex-row sm:items-end sm:justify-between">
+            <div>
+              <ContentHint
+                enabled={contentDebug}
+                path="src/content/presentation.json > pages.projects.design.featured"
+              />
+              <p className="text-xs font-semibold uppercase tracking-[0.08em] text-muted">
+                {copy.featured.eyebrow}
+              </p>
+              <h2 className="mt-3 text-3xl font-semibold text-foreground">
+                {copy.featured.title}
+              </h2>
+            </div>
+            <p className="max-w-xl text-sm leading-6 text-muted">
+              {copy.featured.body}
+            </p>
+          </div>
+          <div className="grid gap-6 lg:grid-cols-2">
+            {featuredProjects.map((project, index) => (
+              <ProjectCard
+                contentDebug={contentDebug}
+                homeTemplate={activeTemplate}
+                key={project.id}
+                priority={index < 2}
+                project={project}
+              />
+            ))}
+          </div>
+        </div>
+      </section>
+      <div>
+        {groupedProjects.map(([category, projects], groupIndex) => (
+          <section
+            className={`border-b border-line ${
+              groupIndex % 2 === 0 ? "" : "bg-background-soft"
+            }`}
+            key={category}
+          >
+            <div className="mx-auto grid max-w-6xl gap-6 px-5 py-14 sm:px-8">
+              <div className="grid gap-4 lg:grid-cols-[0.38fr_0.62fr] lg:items-end">
+                <div>
+                  <ContentHint
+                    enabled={contentDebug}
+                    path={`src/content/presentation.json > pages.projects.groups[category=${category}]`}
+                  />
+                  <p className="text-xs font-semibold uppercase tracking-[0.08em] text-muted">
+                    {projects.length} {copy.group.countLabel}
+                  </p>
+                  <h2 className="mt-3 text-3xl font-semibold text-foreground">
+                    {category}
+                  </h2>
+                </div>
+                <p className="max-w-2xl text-sm leading-6 text-muted lg:justify-self-end">
+                  {groupCopy.get(category)}
+                </p>
+              </div>
+              <div className="grid gap-6 lg:grid-cols-2">
+                {projects.map((project) => (
+                  <ProjectCard
+                    contentDebug={contentDebug}
+                    homeTemplate={activeTemplate}
+                    key={project.id}
+                    project={project}
+                  />
+                ))}
+              </div>
+            </div>
+          </section>
+        ))}
+      </div>
+    </>
+  );
+}
diff --git a/src/designs/design/projects/projects-route.tsx b/src/designs/design/projects/projects-route.tsx
deleted file mode 100644
index 65deb8a..0000000
--- a/src/designs/design/projects/projects-route.tsx
+++ /dev/null
@@ -1,163 +0,0 @@
-import { ContentHint } from "@/components/portfolio/content-hint";
-import { ProjectCard } from "@/components/portfolio/project-card";
-import { PageShell } from "@/components/portfolio/site-shell";
-import {
-  type HomeTemplateId,
-  type ProjectPageContent,
-  type PortfolioProject,
-} from "@/lib/portfolio";
-import type { PreparedDesignRouteProps as DesignRouteProps } from "@/designs/shell-props";
-import { createDesignShellProps } from "@/designs/shell-props";
-
-export default function ProjectsRoute({
-  content,
-  contentDebug,
-  currentPath,
-}: DesignRouteProps) {
-  if (content.route !== "projects") return null;
-
-  const activeTemplate = "design";
-  const shellProps = createDesignShellProps(
-    content,
-    contentDebug,
-    currentPath,
-    activeTemplate,
-  );
-  const pageCopy = content.presentation.pages.projects;
-  const featuredProjects = content.featuredProjects;
-  const groupedProjects = content.archiveGroupEntries;
-  const sourceOnlyCount = content.metricValues.sourceOnlyCount ?? 0;
-  const curriculumCount = content.metricValues.curriculumCount ?? 0;
-
-  return (
-    <PageShell {...shellProps}>
-      <DesignProjectsView
-        activeTemplate={activeTemplate}
-        contentDebug={contentDebug}
-        curriculumCount={curriculumCount}
-        featuredProjects={featuredProjects}
-        groupedProjects={groupedProjects}
-        pageCopy={pageCopy}
-        projects={content.projects}
-        sourceOnlyCount={sourceOnlyCount}
-      />
-    </PageShell>
-  );
-}
-
-function DesignProjectsView({
-  activeTemplate,
-  contentDebug,
-  curriculumCount,
-  featuredProjects,
-  groupedProjects,
-  pageCopy,
-  projects,
-  sourceOnlyCount,
-}: {
-  activeTemplate: HomeTemplateId;
-  contentDebug: boolean;
-  curriculumCount: number;
-  featuredProjects: PortfolioProject[];
-  groupedProjects: [string, PortfolioProject[]][];
-  pageCopy: ProjectPageContent;
-  projects: PortfolioProject[];
-  sourceOnlyCount: number;
-}) {
-  const copy = pageCopy.design;
-  const groupCopy = new Map(pageCopy.groups.map((group) => [group.category, group.body]));
-
-  return (
-    <>
-      <section className="border-b border-line">
-        <div className="mx-auto max-w-6xl px-5 py-20 sm:px-8">
-          <ContentHint
-            enabled={contentDebug}
-            path="src/content/projects.json > projects[]"
-          />
-          <p className="text-sm font-medium text-muted">
-            {projects.length} {copy.hero.stats.visibleEntries} · {curriculumCount} {copy.hero.stats.archive} · {sourceOnlyCount} {copy.hero.stats.sourceFirst}
-          </p>
-          <h1 className="mt-5 max-w-3xl text-5xl font-semibold leading-tight text-foreground md:text-6xl">
-            {copy.hero.title}
-          </h1>
-          <p className="mt-6 max-w-2xl text-base leading-7 text-muted">
-            {copy.hero.body}
-          </p>
-        </div>
-      </section>
-      <section className="border-b border-line bg-background-soft">
-        <div className="mx-auto grid max-w-6xl gap-6 px-5 py-14 sm:px-8">
-          <div className="flex flex-col gap-3 sm:flex-row sm:items-end sm:justify-between">
-            <div>
-              <ContentHint
-                enabled={contentDebug}
-                path="src/content/presentation.json > pages.projects.design.featured"
-              />
-              <p className="text-xs font-semibold uppercase tracking-[0.08em] text-muted">
-                {copy.featured.eyebrow}
-              </p>
-              <h2 className="mt-3 text-3xl font-semibold text-foreground">
-                {copy.featured.title}
-              </h2>
-            </div>
-            <p className="max-w-xl text-sm leading-6 text-muted">
-              {copy.featured.body}
-            </p>
-          </div>
-          <div className="grid gap-6 lg:grid-cols-2">
-            {featuredProjects.map((project, index) => (
-              <ProjectCard
-                contentDebug={contentDebug}
-                homeTemplate={activeTemplate}
-                key={project.id}
-                priority={index < 2}
-                project={project}
-              />
-            ))}
-          </div>
-        </div>
-      </section>
-      <div>
-        {groupedProjects.map(([category, projects], groupIndex) => (
-          <section
-            className={`border-b border-line ${
-              groupIndex % 2 === 0 ? "" : "bg-background-soft"
-            }`}
-            key={category}
-          >
-            <div className="mx-auto grid max-w-6xl gap-6 px-5 py-14 sm:px-8">
-              <div className="grid gap-4 lg:grid-cols-[0.38fr_0.62fr] lg:items-end">
-                <div>
-                  <ContentHint
-                    enabled={contentDebug}
-                    path={`src/content/presentation.json > pages.projects.groups[category=${category}]`}
-                  />
-                  <p className="text-xs font-semibold uppercase tracking-[0.08em] text-muted">
-                    {projects.length} {copy.group.countLabel}
-                  </p>
-                  <h2 className="mt-3 text-3xl font-semibold text-foreground">
-                    {category}
-                  </h2>
-                </div>
-                <p className="max-w-2xl text-sm leading-6 text-muted lg:justify-self-end">
-                  {groupCopy.get(category)}
-                </p>
-              </div>
-              <div className="grid gap-6 lg:grid-cols-2">
-                {projects.map((project) => (
-                  <ProjectCard
-                    contentDebug={contentDebug}
-                    homeTemplate={activeTemplate}
-                    key={project.id}
-                    project={project}
-                  />
-                ))}
-              </div>
-            </div>
-          </section>
-        ))}
-      </div>
-    </>
-  );
-}


