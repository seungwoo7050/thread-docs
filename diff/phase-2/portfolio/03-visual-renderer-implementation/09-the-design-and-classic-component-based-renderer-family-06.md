## `refactor(classic-projects): 프로젝트 목록 renderer를 이동`

diff --git a/src/app/projects/page.tsx b/src/app/projects/page.tsx
index 2436707..ae49935 100644
--- a/src/app/projects/page.tsx
+++ b/src/app/projects/page.tsx
@@ -1,7 +1,6 @@
 import type { Metadata } from "next";
-import { PageShell } from "@/components/portfolio/site-shell";
 import { notFound } from "next/navigation";
-import { ClassicProjectsView } from "@/designs/classic/projects/projects-route";
+import ClassicProjectsRoute from "@/designs/classic/projects-route";
 import DesignProjectsRoute from "@/designs/design/projects-route";
 import { hasDedicatedRouteRenderer, renderDesignRoute } from "@/designs/registry";
 import {
@@ -33,7 +32,7 @@ export default async function ProjectsPage({
 }) {
   const content = getPortfolioContent();
   if (!isSitePageEnabled("projects", content)) notFound();
-  const { activeTemplate, contentDebug, shellProps } =
+  const { activeTemplate, contentDebug } =
     await resolvePortfolioPageContext({
       content,
       currentPath: "/projects",
@@ -49,25 +48,15 @@ export default async function ProjectsPage({
       viewModel,
     });
   }
-  const pageCopy = viewModel.presentation.pages.projects;
-  const featuredProjects = viewModel.featuredProjects;
-  const groupedProjects = viewModel.archiveGroupEntries;
-  const sourceOnlyCount = viewModel.metricValues.sourceOnlyCount ?? 0;
-  const curriculumCount = viewModel.metricValues.curriculumCount ?? 0;
+
   if (activeTemplate === "classic") {
     return (
-      <PageShell {...shellProps}>
-        <ClassicProjectsView
-          activeTemplate={activeTemplate}
-          contentDebug={contentDebug}
-          curriculumCount={curriculumCount}
-          featuredProjects={featuredProjects}
-          groupedProjects={groupedProjects}
-          pageCopy={pageCopy}
-          projects={viewModel.projects}
-          sourceOnlyCount={sourceOnlyCount}
-        />
-      </PageShell>
+      <ClassicProjectsRoute
+        content={viewModel}
+        contentDebug={contentDebug}
+        currentPath="/projects"
+        route="projects"
+      />
     );
   }
 
diff --git a/src/designs/classic/projects-route.tsx b/src/designs/classic/projects-route.tsx
new file mode 100644
index 0000000..2847aaf
--- /dev/null
+++ b/src/designs/classic/projects-route.tsx
@@ -0,0 +1,251 @@
+import Link from "next/link";
+import { ArrowRightIcon } from "@/components/icons";
+import { AvailabilityBadge } from "@/components/portfolio/availability-badge";
+import { ContentHint } from "@/components/portfolio/content-hint";
+import { ProjectCard } from "@/components/portfolio/project-card";
+import { PageShell } from "@/components/portfolio/site-shell";
+import { StackList } from "@/components/portfolio/stack-list";
+import {
+  getTemplateHref,
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
+  const activeTemplate = "classic";
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
+      <ClassicProjectsView
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
+
+function ClassicProjectsView({
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
+  const leadProject = featuredProjects[0];
+  const copy = pageCopy.classic;
+  const groupCopy = new Map(pageCopy.groups.map((group) => [group.category, group.body]));
+  const counts = {
+    curriculumCount,
+    projectCount: projects.length,
+    sourceOnlyCount,
+  };
+
+  return (
+    <>
+      <section className="classic-projects-hero border-b border-line">
+        <div className="mx-auto grid max-w-6xl gap-10 px-5 py-16 sm:px-8 lg:grid-cols-[0.92fr_1.08fr] lg:items-center">
+          <div>
+            <ContentHint
+              enabled={contentDebug}
+              path="src/content/presentation.json > pages.projects.classic.hero"
+            />
+            <p className="text-sm font-medium text-muted">
+              {copy.hero.eyebrow}
+            </p>
+            <h1 className="mt-5 max-w-3xl text-5xl font-semibold leading-tight text-foreground md:text-6xl">
+              {copy.hero.title}
+            </h1>
+            <p className="mt-6 max-w-2xl text-base leading-7 text-muted">
+              {copy.hero.body}
+            </p>
+            <dl className="mt-8 grid max-w-2xl grid-cols-3 overflow-hidden rounded-md border border-line bg-surface">
+              {copy.hero.stats.map((stat, index) => (
+                <div
+                  className={index < copy.hero.stats.length - 1 ? "border-r border-line p-4" : "p-4"}
+                  key={stat.label}
+                >
+                  <dt className="text-xs font-semibold uppercase text-muted">
+                    {stat.label}
+                  </dt>
+                  <dd className="mt-1 text-2xl font-semibold text-foreground">
+                    {counts[stat.countKey]}
+                  </dd>
+                </div>
+              ))}
+            </dl>
+          </div>
+          <aside
+            aria-label={copy.terminal.ariaLabel}
+            className="terminal-window mx-auto w-full max-w-xl"
+          >
+            <ContentHint
+              enabled={contentDebug}
+              path="src/content/presentation.json > pages.projects.classic.terminal"
+            />
+            <div className="terminal-titlebar">
+              <span className="bg-[#ff6b5f]" />
+              <span className="bg-[#f6c76f]" />
+              <span className="bg-[#67d391]" />
+              <p>{copy.terminal.title}</p>
+            </div>
+            <div className="terminal-body">
+              <p className="terminal-line">
+                <span className="text-accent">{copy.terminal.promptUser}</span>
+                <span className="text-muted">:</span>
+                <span className="text-signal">{copy.terminal.promptPath}</span>
+                <span className="text-muted">$ </span>
+                {copy.terminal.command}
+              </p>
+              <div className="mt-5 grid gap-3">
+                {groupedProjects.slice(0, copy.terminal.maxGroups).map(([category, items]) => (
+                  <p className="terminal-line terminal-output" key={category}>
+                    {category.padEnd(26, ".")} {items.length} {copy.terminal.entryLabel}
+                  </p>
+                ))}
+              </div>
+            </div>
+          </aside>
+        </div>
+      </section>
+      {leadProject ? (
+        <section className="border-b border-line bg-background-soft">
+          <div className="mx-auto grid max-w-6xl gap-6 px-5 py-14 sm:px-8">
+            <div className="grid gap-3 lg:grid-cols-[0.4fr_0.6fr] lg:items-end">
+              <div>
+                <p className="text-xs font-semibold uppercase tracking-[0.08em] text-muted">
+                  {copy.selected.eyebrow}
+                </p>
+                <h2 className="mt-3 text-3xl font-semibold text-foreground">
+                  {copy.selected.title}
+                </h2>
+              </div>
+              <p className="max-w-xl text-sm leading-6 text-muted lg:justify-self-end">
+                {copy.selected.body}
+              </p>
+            </div>
+            <ProjectCard
+              contentDebug={contentDebug}
+              homeTemplate={activeTemplate}
+              priority
+              project={leadProject}
+              variant="featured"
+            />
+          </div>
+        </section>
+      ) : null}
+      <section>
+        <div className="mx-auto grid max-w-6xl gap-6 px-5 py-14 sm:px-8">
+          <div className="grid gap-3 lg:grid-cols-[0.4fr_0.6fr] lg:items-end">
+            <div>
+              <ContentHint
+                enabled={contentDebug}
+                path="src/content/presentation.json > pages.projects.classic.grouped"
+              />
+              <p className="text-xs font-semibold uppercase tracking-[0.08em] text-muted">
+                {copy.grouped.eyebrow}
+              </p>
+              <h2 className="mt-3 text-3xl font-semibold text-foreground">
+                {copy.grouped.title}
+              </h2>
+            </div>
+            <p className="max-w-xl text-sm leading-6 text-muted lg:justify-self-end">
+              {copy.grouped.body}
+            </p>
+          </div>
+          <div className="grid gap-3">
+            {groupedProjects.map(([category, items]) => (
+              <article
+                className="rounded-md border border-line bg-surface p-5"
+                key={category}
+              >
+                <div className="grid gap-5 lg:grid-cols-[0.3fr_0.7fr]">
+                  <div>
+                    <ContentHint
+                      enabled={contentDebug}
+                      path={`src/content/presentation.json > pages.projects.groups[category=${category}]`}
+                    />
+                    <p className="text-xs font-semibold uppercase tracking-[0.08em] text-accent">
+                      {items.length} {copy.grouped.countLabel}
+                    </p>
+                    <h3 className="mt-2 text-xl font-semibold text-foreground">
+                      {category}
+                    </h3>
+                    <p className="mt-3 text-sm leading-6 text-muted">
+                      {groupCopy.get(category)}
+                    </p>
+                  </div>
+                  <ul className="grid gap-3">
+                    {items.map((project) => (
+                      <li
+                        className="grid gap-3 border-t border-line pt-3 first:border-t-0 first:pt-0"
+                        key={project.id}
+                      >
+                        <div className="flex flex-wrap items-center justify-between gap-3">
+                          <Link
+                            className="inline-flex items-center gap-2 font-semibold text-foreground transition hover:text-accent-strong"
+                            href={getTemplateHref(
+                              `/projects/${project.id}`,
+                              activeTemplate,
+                              { contentDebug },
+                            )}
+                          >
+                            {project.title}
+                            <ArrowRightIcon />
+                          </Link>
+                          <AvailabilityBadge project={project} />
+                        </div>
+                        <p className="text-sm leading-6 text-muted">
+                          {project.summary}
+                        </p>
+                        <StackList items={project.stack} limit={5} />
+                      </li>
+                    ))}
+                  </ul>
+                </div>
+              </article>
+            ))}
+          </div>
+        </div>
+      </section>
+    </>
+  );
+}
diff --git a/src/designs/classic/projects/projects-route.tsx b/src/designs/classic/projects/projects-route.tsx
deleted file mode 100644
index 94aef43..0000000
--- a/src/designs/classic/projects/projects-route.tsx
+++ /dev/null
@@ -1,214 +0,0 @@
-import Link from "next/link";
-import { ArrowRightIcon } from "@/components/icons";
-import { AvailabilityBadge } from "@/components/portfolio/availability-badge";
-import { ContentHint } from "@/components/portfolio/content-hint";
-import { ProjectCard } from "@/components/portfolio/project-card";
-import { StackList } from "@/components/portfolio/stack-list";
-import { getTemplateHref } from "@/lib/portfolio";
-import type {
-  HomeTemplateId,
-  PortfolioProject,
-  ProjectPageContent,
-} from "@/lib/portfolio";
-import type { GroupedProjects } from "@/lib/portfolio/project-groups";
-
-export function ClassicProjectsView({
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
-  groupedProjects: GroupedProjects;
-  pageCopy: ProjectPageContent;
-  projects: PortfolioProject[];
-  sourceOnlyCount: number;
-}) {
-  const copy = pageCopy.classic;
-  const leadProject = featuredProjects[0];
-  const groupCopy = new Map(pageCopy.groups.map((group) => [group.category, group.body]));
-  const counts = {
-    curriculumCount,
-    projectCount: projects.length,
-    sourceOnlyCount,
-  };
-
-  return (
-    <>
-      <section className="classic-projects-hero border-b border-line">
-        <div className="mx-auto grid max-w-6xl gap-10 px-5 py-16 sm:px-8 lg:grid-cols-[0.92fr_1.08fr] lg:items-center">
-        <div>
-          <ContentHint
-            enabled={contentDebug}
-            path="src/content/presentation.json > pages.projects.classic.hero"
-          />
-          <p className="text-sm font-medium text-muted">
-            {copy.hero.eyebrow}
-          </p>
-          <h1 className="mt-5 max-w-3xl text-5xl font-semibold leading-tight text-foreground md:text-6xl">
-            {copy.hero.title}
-          </h1>
-          <p className="mt-6 max-w-2xl text-base leading-7 text-muted">
-            {copy.hero.body}
-          </p>
-          <dl className="mt-8 grid max-w-2xl grid-cols-3 overflow-hidden rounded-md border border-line bg-surface">
-            {copy.hero.stats.map((stat, index) => (
-              <div
-                className={
-                  index < copy.hero.stats.length - 1 ? "border-r border-line p-4" : "p-4"
-                }
-                key={stat.label}
-              >
-                <dt className="text-xs font-semibold uppercase text-muted">
-                  {stat.label}
-                </dt>
-                <dd className="mt-1 text-2xl font-semibold text-foreground">
-                  {counts[stat.countKey]}
-                </dd>
-              </div>
-            ))}
-          </dl>
-        </div>
-        <aside
-          aria-label={copy.terminal.ariaLabel}
-          className="terminal-window mx-auto w-full max-w-xl"
-        >
-          <ContentHint
-            enabled={contentDebug}
-            path="src/content/presentation.json > pages.projects.classic.terminal"
-          />
-          <div className="terminal-titlebar">
-            <span className="bg-[#ff6b5f]" />
-            <span className="bg-[#f6c76f]" />
-            <span className="bg-[#67d391]" />
-            <p>{copy.terminal.title}</p>
-          </div>
-          <div className="terminal-body">
-            <p className="terminal-line">
-              <span className="text-accent">{copy.terminal.promptUser}</span>
-              <span className="text-muted">:</span>
-              <span className="text-signal">{copy.terminal.promptPath}</span>
-              <span className="text-muted">$ </span>
-              {copy.terminal.command}
-            </p>
-            <div className="mt-5 grid gap-3">
-              {groupedProjects.slice(0, copy.terminal.maxGroups).map(([category, items]) => (
-                <p className="terminal-line terminal-output" key={category}>
-                  {category.padEnd(26, ".")} {items.length} {copy.terminal.entryLabel}
-                </p>
-              ))}
-            </div>
-          </div>
-        </aside>
-        </div>
-      </section>
-      {leadProject ? (
-        <section className="border-b border-line bg-background-soft">
-          <div className="mx-auto grid max-w-6xl gap-6 px-5 py-14 sm:px-8">
-            <div className="grid gap-3 lg:grid-cols-[0.4fr_0.6fr] lg:items-end">
-              <div>
-                <p className="text-xs font-semibold uppercase tracking-[0.08em] text-muted">
-                  {copy.selected.eyebrow}
-                </p>
-                <h2 className="mt-3 text-3xl font-semibold text-foreground">
-                  {copy.selected.title}
-                </h2>
-              </div>
-              <p className="max-w-xl text-sm leading-6 text-muted lg:justify-self-end">
-                {copy.selected.body}
-              </p>
-            </div>
-            <ProjectCard
-              contentDebug={contentDebug}
-              homeTemplate={activeTemplate}
-              priority
-              project={leadProject}
-              variant="featured"
-            />
-          </div>
-        </section>
-      ) : null}
-      <section>
-        <div className="mx-auto grid max-w-6xl gap-6 px-5 py-14 sm:px-8">
-          <div className="grid gap-3 lg:grid-cols-[0.4fr_0.6fr] lg:items-end">
-            <div>
-              <ContentHint
-                enabled={contentDebug}
-                path="src/content/presentation.json > pages.projects.classic.grouped"
-              />
-              <p className="text-xs font-semibold uppercase tracking-[0.08em] text-muted">
-                {copy.grouped.eyebrow}
-              </p>
-              <h2 className="mt-3 text-3xl font-semibold text-foreground">
-                {copy.grouped.title}
-              </h2>
-            </div>
-            <p className="max-w-xl text-sm leading-6 text-muted lg:justify-self-end">
-              {copy.grouped.body}
-            </p>
-          </div>
-          <div className="grid gap-3">
-            {groupedProjects.map(([category, items]) => (
-              <article
-                className="rounded-md border border-line bg-surface p-5"
-                key={category}
-              >
-                <div className="grid gap-5 lg:grid-cols-[0.3fr_0.7fr]">
-                  <div>
-                    <ContentHint
-                      enabled={contentDebug}
-                      path={`src/content/presentation.json > pages.projects.groups[category=${category}]`}
-                    />
-                    <p className="text-xs font-semibold uppercase tracking-[0.08em] text-accent">
-                      {items.length} {copy.grouped.countLabel}
-                    </p>
-                    <h3 className="mt-2 text-xl font-semibold text-foreground">
-                      {category}
-                    </h3>
-                    <p className="mt-3 text-sm leading-6 text-muted">
-                      {groupCopy.get(category)}
-                    </p>
-                  </div>
-                  <ul className="grid gap-3">
-                    {items.map((project) => (
-                      <li
-                        className="grid gap-3 border-t border-line pt-3 first:border-t-0 first:pt-0"
-                        key={project.id}
-                      >
-                        <div className="flex flex-wrap items-center justify-between gap-3">
-                          <Link
-                            className="inline-flex items-center gap-2 font-semibold text-foreground transition hover:text-accent-strong"
-                            href={getTemplateHref(
-                              `/projects/${project.id}`,
-                              activeTemplate,
-                              { contentDebug },
-                            )}
-                          >
-                            {project.title}
-                            <ArrowRightIcon />
-                          </Link>
-                          <AvailabilityBadge project={project} />
-                        </div>
-                        <p className="text-sm leading-6 text-muted">
-                          {project.summary}
-                        </p>
-                        <StackList items={project.stack} limit={5} />
-                      </li>
-                    ))}
-                  </ul>
-                </div>
-              </article>
-            ))}
-          </div>
-        </div>
-      </section>
-    </>
-  );
-}


