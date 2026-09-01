## `refactor(classic-project): 상세 renderer를 독립 모듈로 완성`

diff --git a/src/app/projects/[projectId]/page.tsx b/src/app/projects/[projectId]/page.tsx
index 27326a2..9215cd9 100644
--- a/src/app/projects/[projectId]/page.tsx
+++ b/src/app/projects/[projectId]/page.tsx
@@ -1,8 +1,7 @@
 import type { Metadata } from "next";
 import { notFound } from "next/navigation";
-import { ProjectDetailView } from "@/designs/classic/project-detail-route";
-import { PageShell } from "@/components/portfolio/site-shell";
 import { StructuredData } from "@/components/portfolio/structured-data";
+import ClassicProjectDetailRoute from "@/designs/classic/project-detail-route";
 import DesignProjectDetailRoute from "@/designs/design/project-detail-route";
 import { hasDedicatedRouteRenderer, renderDesignRoute } from "@/designs/registry";
 import {
@@ -60,7 +59,7 @@ export default async function ProjectDetailPage({
   const content = getPortfolioContent();
   if (!isSitePageEnabled("projects", content)) notFound();
   const { projectId } = await params;
-  const { activeTemplate, contentDebug, shellProps } =
+  const { activeTemplate, contentDebug } =
     await resolvePortfolioPageContext({
       content,
       currentPath: `/projects/${projectId}`,
@@ -118,14 +117,12 @@ export default async function ProjectDetailPage({
   return (
     <>
       {structuredData ? <StructuredData data={structuredData} /> : null}
-      <PageShell {...shellProps}>
-        <ProjectDetailView
-          contentDebug={contentDebug}
-          homeTemplate={activeTemplate}
-          pageCopy={viewModel.presentation.pages.projectDetail}
-          project={project}
-        />
-      </PageShell>
+      <ClassicProjectDetailRoute
+        content={viewModel}
+        contentDebug={contentDebug}
+        currentPath={`/projects/${project.id}`}
+        route="project-detail"
+      />
     </>
   );
 }
diff --git a/src/designs/classic/project-detail-route.tsx b/src/designs/classic/project-detail-route.tsx
index 516a3f5..e19ce56 100644
--- a/src/designs/classic/project-detail-route.tsx
+++ b/src/designs/classic/project-detail-route.tsx
@@ -1,18 +1,55 @@
 import Link from "next/link";
 import { ArrowRightIcon } from "@/components/icons";
-import {
-  getTemplateHref,
-  type HomeTemplateId,
-  type PortfolioProject,
-  type ProjectDetailPageContent,
-} from "@/lib/portfolio";
 import { AvailabilityBadge } from "@/components/portfolio/availability-badge";
 import { ContentHint } from "@/components/portfolio/content-hint";
 import { ProjectLinks } from "@/components/portfolio/project-links";
 import { ProjectScreenshot } from "@/components/portfolio/project-screenshot";
+import { PageShell } from "@/components/portfolio/site-shell";
 import { StackList } from "@/components/portfolio/stack-list";
+import {
+  getTemplateHref,
+  type HomeTemplateId,
+  type ProjectDetailPageContent,
+  type PortfolioProject,
+} from "@/lib/portfolio";
+import { createDesignShellProps } from "@/designs/shell-props";
+import type { PreparedDesignRouteProps as DesignRouteProps } from "@/designs/shell-props";
+
+export default function ProjectDetailRoute({
+  content,
+  contentDebug,
+  currentPath,
+}: DesignRouteProps) {
+  if (content.route !== "project-detail") return null;
 
-export function ProjectDetailView({
+  const activeTemplate = "classic";
+  const shellProps = createDesignShellProps(
+    content,
+    contentDebug,
+    currentPath,
+    activeTemplate,
+  );
+  const project = content.project;
+  const pageCopy = content.presentation.pages.projectDetail;
+
+  return (
+    <PageShell {...shellProps}>
+      <ProjectHero
+        contentDebug={contentDebug}
+        homeTemplate={activeTemplate}
+        pageCopy={pageCopy}
+        project={project}
+      />
+      <ProjectBody
+        contentDebug={contentDebug}
+        pageCopy={pageCopy}
+        project={project}
+      />
+    </PageShell>
+  );
+}
+
+function ProjectHero({
   contentDebug,
   homeTemplate,
   pageCopy,
@@ -23,12 +60,9 @@ export function ProjectDetailView({
   pageCopy: ProjectDetailPageContent;
   project: PortfolioProject;
 }) {
-  const { sections } = pageCopy;
-
   return (
-    <>
-      <section className="border-b border-line">
-        <div className="mx-auto grid max-w-6xl gap-10 px-5 py-16 sm:px-8 lg:grid-cols-[0.9fr_1.1fr] lg:items-center">
+    <section className="border-b border-line">
+      <div className="mx-auto grid max-w-6xl gap-10 px-5 py-16 sm:px-8 lg:grid-cols-[0.9fr_1.1fr] lg:items-center">
         <div>
           <Link
             className="inline-flex items-center gap-2 text-sm font-semibold text-muted transition hover:text-foreground"
@@ -68,88 +102,99 @@ export function ProjectDetailView({
         <div className="group">
           <ProjectScreenshot image={project.screenshot} priority />
         </div>
-        </div>
-      </section>
-      <div className="mx-auto grid max-w-6xl gap-14 px-5 py-16 sm:px-8">
-        <ContentHint
-          enabled={contentDebug}
-          path={`src/content/presentation.json > pages.projectDetail.sections + src/content/projects.json > projects[id=${project.id}] detail fields`}
-        />
-        <TwoColumnSection
-          body={project.problem}
-          eyebrow={sections.problem.eyebrow}
-          title={sections.problem.title}
-        />
-        <TwoColumnSection
-          body={project.solution}
-          eyebrow={sections.solution.eyebrow}
-          title={sections.solution.title}
+      </div>
+    </section>
+  );
+}
+
+function ProjectBody({
+  contentDebug,
+  pageCopy,
+  project,
+}: {
+  contentDebug: boolean;
+  pageCopy: ProjectDetailPageContent;
+  project: PortfolioProject;
+}) {
+  const { sections } = pageCopy;
+
+  return (
+    <div className="mx-auto grid max-w-6xl gap-14 px-5 py-16 sm:px-8">
+      <ContentHint
+        enabled={contentDebug}
+        path={`src/content/presentation.json > pages.projectDetail.sections + src/content/projects.json > projects[id=${project.id}] detail fields`}
+      />
+      <TwoColumnSection
+        body={project.problem}
+        eyebrow={sections.problem.eyebrow}
+        title={sections.problem.title}
+      />
+      <TwoColumnSection
+        body={project.solution}
+        eyebrow={sections.solution.eyebrow}
+        title={sections.solution.title}
+      />
+      <section className="grid gap-6 lg:grid-cols-[0.42fr_0.58fr]">
+        <SectionTitle
+          eyebrow={sections.architecture.eyebrow}
+          title={sections.architecture.title}
         />
-        <section className="grid gap-6 lg:grid-cols-[0.42fr_0.58fr]">
-          <SectionTitle
-            eyebrow={sections.architecture.eyebrow}
-            title={sections.architecture.title}
-          />
-          <div className="rounded-lg border border-line bg-surface p-6">
-            <p className="text-base leading-7 text-foreground">
-              {project.architecture.summary}
-            </p>
-            <ul className="mt-6 grid gap-3">
-              {project.architecture.items.map((item) => (
-                <li
-                  className="border-l border-line pl-4 text-sm leading-6 text-muted"
-                  key={item}
-                >
-                  {item}
-                </li>
-              ))}
-            </ul>
-          </div>
-        </section>
-        <section className="grid gap-6 lg:grid-cols-[0.42fr_0.58fr]">
-          <SectionTitle
-            eyebrow={sections.screenshots.eyebrow}
-            title={sections.screenshots.title}
-          />
-          <div className="grid gap-4">
-            {project.screenshots.map((image) => (
-              <div className="group" key={image.src}>
-                <ProjectScreenshot image={image} />
-              </div>
+        <div className="rounded-lg border border-line bg-surface p-6">
+          <p className="text-base leading-7 text-foreground">
+            {project.architecture.summary}
+          </p>
+          <ul className="mt-6 grid gap-3">
+            {project.architecture.items.map((item) => (
+              <li className="border-l border-line pl-4 text-sm leading-6 text-muted" key={item}>
+                {item}
+              </li>
             ))}
-          </div>
-        </section>
-        <section className="grid gap-6 lg:grid-cols-[0.42fr_0.58fr]">
-          <SectionTitle
-            eyebrow={sections.stack.eyebrow}
-            title={sections.stack.title}
-          />
-          <div className="rounded-lg border border-line bg-surface p-6">
-            <StackList items={project.stack} />
-          </div>
-        </section>
-        <ListSection
-          eyebrow={sections.decisions.eyebrow}
-          items={project.decisions}
-          title={sections.decisions.title}
-        />
-        <ListSection
-          eyebrow={sections.highlights.eyebrow}
-          items={project.highlights}
-          title={sections.highlights.title}
-        />
-        <ListSection
-          eyebrow={sections.tradeoffs.eyebrow}
-          items={project.tradeoffs}
-          title={sections.tradeoffs.title}
+          </ul>
+        </div>
+      </section>
+      <section className="grid gap-6 lg:grid-cols-[0.42fr_0.58fr]">
+        <SectionTitle
+          eyebrow={sections.screenshots.eyebrow}
+          title={sections.screenshots.title}
         />
-        <ListSection
-          eyebrow={sections.result.eyebrow}
-          items={project.results}
-          title={sections.result.title}
+        <div className="grid gap-4">
+          {project.screenshots.map((image) => (
+            <div className="group" key={image.src}>
+              <ProjectScreenshot image={image} />
+            </div>
+          ))}
+        </div>
+      </section>
+      <section className="grid gap-6 lg:grid-cols-[0.42fr_0.58fr]">
+        <SectionTitle
+          eyebrow={sections.stack.eyebrow}
+          title={sections.stack.title}
         />
-      </div>
-    </>
+        <div className="rounded-lg border border-line bg-surface p-6">
+          <StackList items={project.stack} />
+        </div>
+      </section>
+      <ListSection
+        eyebrow={sections.decisions.eyebrow}
+        items={project.decisions}
+        title={sections.decisions.title}
+      />
+      <ListSection
+        eyebrow={sections.highlights.eyebrow}
+        items={project.highlights}
+        title={sections.highlights.title}
+      />
+      <ListSection
+        eyebrow={sections.tradeoffs.eyebrow}
+        items={project.tradeoffs}
+        title={sections.tradeoffs.title}
+      />
+      <ListSection
+        eyebrow={sections.result.eyebrow}
+        items={project.results}
+        title={sections.result.title}
+      />
+    </div>
   );
 }
 


