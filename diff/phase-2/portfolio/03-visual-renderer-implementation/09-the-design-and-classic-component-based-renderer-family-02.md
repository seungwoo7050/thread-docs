## `feat(home): 클래식 대표 프로젝트 섹션 추가`

diff --git a/src/designs/classic/home-route.tsx b/src/designs/classic/home-route.tsx
index 028e79d..13fd10f 100644
--- a/src/designs/classic/home-route.tsx
+++ b/src/designs/classic/home-route.tsx
@@ -3,9 +3,15 @@ import { AnimatedTerminal } from "@/components/portfolio/animated-terminal";
 import { ContentHint } from "@/components/portfolio/content-hint";
 import { ContentLinkView } from "@/components/portfolio/content-link";
 import { ProfilePhoto } from "@/components/portfolio/profile-photo";
+import { ProjectCard } from "@/components/portfolio/project-card";
 import { Reveal } from "@/components/portfolio/reveal";
+import { SectionHeading } from "@/components/portfolio/section-heading";
 import { PageShell } from "@/components/portfolio/site-shell";
-import type { HomeTemplateId, PortfolioContent } from "@/lib/portfolio";
+import {
+  getFeaturedProjects,
+  type HomeTemplateId,
+  type PortfolioContent,
+} from "@/lib/portfolio";
 
 export function ClassicHomeRoute({
   content,
@@ -15,6 +21,7 @@ export function ClassicHomeRoute({
   contentDebug: boolean;
 }) {
   const activeTemplate: HomeTemplateId = "classic";
+  const featuredProjects = getFeaturedProjects(content);
 
   return (
     <PageShell
@@ -34,10 +41,60 @@ export function ClassicHomeRoute({
         content={content}
         contentDebug={contentDebug}
       />
+      {content.presentation.home.classic.sections.includes("featured") ? (
+        <ClassicFeaturedProjectsSection
+          activeTemplate={activeTemplate}
+          content={content}
+          contentDebug={contentDebug}
+          projects={featuredProjects}
+        />
+      ) : null}
     </PageShell>
   );
 }
 
+function ClassicFeaturedProjectsSection({
+  activeTemplate,
+  content,
+  contentDebug,
+  projects,
+}: {
+  activeTemplate: HomeTemplateId;
+  content: PortfolioContent;
+  contentDebug: boolean;
+  projects: ReturnType<typeof getFeaturedProjects>;
+}) {
+  const copy = content.presentation.home.classic.featured;
+
+  return (
+    <section className="border-b border-line bg-background-soft">
+      <div className="mx-auto grid max-w-6xl gap-6 px-5 py-10 sm:px-8 md:py-12">
+        <div className="flex flex-col gap-5 sm:flex-row sm:items-end sm:justify-between">
+          <SectionHeading
+            body={copy.body}
+            contentDebug={contentDebug}
+            contentHint="src/content/presentation.json > home.classic.featured"
+            title={copy.title}
+          />
+        </div>
+        <div className="grid gap-5">
+          {projects.slice(0, 1).map((project) => (
+            <Reveal key={project.id}>
+              <ProjectCard
+                contentDebug={contentDebug}
+                homeTemplate={activeTemplate}
+                priority
+                project={project}
+                variant="featured"
+              />
+            </Reveal>
+          ))}
+        </div>
+      </div>
+    </section>
+  );
+}
+
 function ClassicHeroSection({
   activeTemplate,
   content,


## `feat(projects): 디자인 프로젝트 그룹 목록 추가`

diff --git a/src/designs/design/projects/projects-route.tsx b/src/designs/design/projects/projects-route.tsx
index 4492828..83db75a 100644
--- a/src/designs/design/projects/projects-route.tsx
+++ b/src/designs/design/projects/projects-route.tsx
@@ -5,12 +5,14 @@ import type {
   PortfolioProject,
   ProjectPageContent,
 } from "@/lib/portfolio";
+import type { GroupedProjects } from "@/lib/portfolio/project-groups";
 
 export function DesignProjectsView({
   activeTemplate,
   contentDebug,
   curriculumCount,
   featuredProjects,
+  groupedProjects,
   pageCopy,
   projects,
   sourceOnlyCount,
@@ -19,11 +21,13 @@ export function DesignProjectsView({
   contentDebug: boolean;
   curriculumCount: number;
   featuredProjects: PortfolioProject[];
+  groupedProjects: GroupedProjects;
   pageCopy: ProjectPageContent;
   projects: PortfolioProject[];
   sourceOnlyCount: number;
 }) {
   const copy = pageCopy.design;
+  const groupCopy = new Map(pageCopy.groups.map((group) => [group.category, group.body]));
 
   return (
     <>
@@ -78,6 +82,46 @@ export function DesignProjectsView({
           </div>
         </div>
       </section>
+      <div>
+        {groupedProjects.map(([category, groupedItems], groupIndex) => (
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
+                    {groupedItems.length} {copy.group.countLabel}
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
+                {groupedItems.map((project) => (
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
     </>
   );
 }


## `feat(projects): 클래식 프로젝트 소개와 터미널 추가`

diff --git a/src/designs/classic/projects/projects-route.tsx b/src/designs/classic/projects/projects-route.tsx
new file mode 100644
index 0000000..d0a0e2f
--- /dev/null
+++ b/src/designs/classic/projects/projects-route.tsx
@@ -0,0 +1,96 @@
+import { ContentHint } from "@/components/portfolio/content-hint";
+import type { PortfolioProject, ProjectPageContent } from "@/lib/portfolio";
+import type { GroupedProjects } from "@/lib/portfolio/project-groups";
+
+export function ClassicProjectsView({
+  contentDebug,
+  curriculumCount,
+  groupedProjects,
+  pageCopy,
+  projects,
+  sourceOnlyCount,
+}: {
+  contentDebug: boolean;
+  curriculumCount: number;
+  groupedProjects: GroupedProjects;
+  pageCopy: ProjectPageContent;
+  projects: PortfolioProject[];
+  sourceOnlyCount: number;
+}) {
+  const copy = pageCopy.classic;
+  const counts = {
+    curriculumCount,
+    projectCount: projects.length,
+    sourceOnlyCount,
+  };
+
+  return (
+    <section className="classic-projects-hero border-b border-line">
+      <div className="mx-auto grid max-w-6xl gap-10 px-5 py-16 sm:px-8 lg:grid-cols-[0.92fr_1.08fr] lg:items-center">
+        <div>
+          <ContentHint
+            enabled={contentDebug}
+            path="src/content/presentation.json > pages.projects.classic.hero"
+          />
+          <p className="text-sm font-medium text-muted">
+            {copy.hero.eyebrow}
+          </p>
+          <h1 className="mt-5 max-w-3xl text-5xl font-semibold leading-tight text-foreground md:text-6xl">
+            {copy.hero.title}
+          </h1>
+          <p className="mt-6 max-w-2xl text-base leading-7 text-muted">
+            {copy.hero.body}
+          </p>
+          <dl className="mt-8 grid max-w-2xl grid-cols-3 overflow-hidden rounded-md border border-line bg-surface">
+            {copy.hero.stats.map((stat, index) => (
+              <div
+                className={
+                  index < copy.hero.stats.length - 1 ? "border-r border-line p-4" : "p-4"
+                }
+                key={stat.label}
+              >
+                <dt className="text-xs font-semibold uppercase text-muted">
+                  {stat.label}
+                </dt>
+                <dd className="mt-1 text-2xl font-semibold text-foreground">
+                  {counts[stat.countKey]}
+                </dd>
+              </div>
+            ))}
+          </dl>
+        </div>
+        <aside
+          aria-label={copy.terminal.ariaLabel}
+          className="terminal-window mx-auto w-full max-w-xl"
+        >
+          <ContentHint
+            enabled={contentDebug}
+            path="src/content/presentation.json > pages.projects.classic.terminal"
+          />
+          <div className="terminal-titlebar">
+            <span className="bg-[#ff6b5f]" />
+            <span className="bg-[#f6c76f]" />
+            <span className="bg-[#67d391]" />
+            <p>{copy.terminal.title}</p>
+          </div>
+          <div className="terminal-body">
+            <p className="terminal-line">
+              <span className="text-accent">{copy.terminal.promptUser}</span>
+              <span className="text-muted">:</span>
+              <span className="text-signal">{copy.terminal.promptPath}</span>
+              <span className="text-muted">$ </span>
+              {copy.terminal.command}
+            </p>
+            <div className="mt-5 grid gap-3">
+              {groupedProjects.slice(0, copy.terminal.maxGroups).map(([category, items]) => (
+                <p className="terminal-line terminal-output" key={category}>
+                  {category.padEnd(26, ".")} {items.length} {copy.terminal.entryLabel}
+                </p>
+              ))}
+            </div>
+          </div>
+        </aside>
+      </div>
+    </section>
+  );
+}


## `feat(projects): 클래식 그룹 인덱스 추가`

diff --git a/src/designs/classic/projects/projects-route.tsx b/src/designs/classic/projects/projects-route.tsx
index 1a52e69..94aef43 100644
--- a/src/designs/classic/projects/projects-route.tsx
+++ b/src/designs/classic/projects/projects-route.tsx
@@ -1,5 +1,10 @@
+import Link from "next/link";
+import { ArrowRightIcon } from "@/components/icons";
+import { AvailabilityBadge } from "@/components/portfolio/availability-badge";
 import { ContentHint } from "@/components/portfolio/content-hint";
 import { ProjectCard } from "@/components/portfolio/project-card";
+import { StackList } from "@/components/portfolio/stack-list";
+import { getTemplateHref } from "@/lib/portfolio";
 import type {
   HomeTemplateId,
   PortfolioProject,
@@ -28,6 +33,7 @@ export function ClassicProjectsView({
 }) {
   const copy = pageCopy.classic;
   const leadProject = featuredProjects[0];
+  const groupCopy = new Map(pageCopy.groups.map((group) => [group.category, group.body]));
   const counts = {
     curriculumCount,
     projectCount: projects.length,
@@ -129,6 +135,80 @@ export function ClassicProjectsView({
           </div>
         </section>
       ) : null}
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
     </>
   );
 }


## `feat(projects): 프로젝트 목록 route 연결`

diff --git a/src/app/projects/page.tsx b/src/app/projects/page.tsx
new file mode 100644
index 0000000..b7c94ba
--- /dev/null
+++ b/src/app/projects/page.tsx
@@ -0,0 +1,75 @@
+import { PageShell } from "@/components/portfolio/site-shell";
+import { ClassicProjectsView } from "@/designs/classic/projects/projects-route";
+import { DesignProjectsView } from "@/designs/design/projects/projects-route";
+import {
+  getPortfolioContent,
+  resolveContentDebug,
+  resolveHomeTemplateId,
+  type RouteSearchParams,
+} from "@/lib/portfolio";
+import { groupProjects } from "@/lib/portfolio/project-groups";
+
+export default async function ProjectsPage({
+  searchParams,
+}: {
+  searchParams?: RouteSearchParams;
+}) {
+  const content = getPortfolioContent();
+  const params = searchParams ? await searchParams : {};
+  const activeTemplate = resolveHomeTemplateId(params.view, content.presentation);
+  const contentDebug = resolveContentDebug(params.debug);
+  const pageCopy = content.presentation.pages.projects;
+  const featuredProjects = content.projects.filter((project) => project.featured);
+  const trackProjects = content.projects.filter((project) => !project.featured);
+  const groupedProjects = groupProjects(trackProjects, pageCopy.groups);
+  const sourceOnlyCount = content.projects.filter(
+    (project) => project.deployment.status === "source-only",
+  ).length;
+  const curriculumCount = content.projects.filter((project) =>
+    project.screenshot.src.startsWith("/projects/42/") || project.id === "pong-pong",
+  ).length;
+  const shellProps = {
+    contentDebug,
+    homeTemplate: activeTemplate,
+    profile: content.profile,
+    site: content.site,
+    templateSwitcher: {
+      activeId: activeTemplate,
+      contentDebug,
+      currentPath: "/projects",
+      templates: content.presentation.templates,
+    },
+  };
+
+  if (activeTemplate === "classic") {
+    return (
+      <PageShell {...shellProps}>
+        <ClassicProjectsView
+          activeTemplate={activeTemplate}
+          contentDebug={contentDebug}
+          curriculumCount={curriculumCount}
+          featuredProjects={featuredProjects}
+          groupedProjects={groupedProjects}
+          pageCopy={pageCopy}
+          projects={content.projects}
+          sourceOnlyCount={sourceOnlyCount}
+        />
+      </PageShell>
+    );
+  }
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
diff --git a/src/content/site.json b/src/content/site.json
index 2c963c2..9d9a999 100644
--- a/src/content/site.json
+++ b/src/content/site.json
@@ -7,6 +7,10 @@
     {
       "label": "Home",
       "href": "/"
+    },
+    {
+      "label": "Projects",
+      "href": "/projects"
     }
   ],
   "footer": {


## `feat(project): 프로젝트 상세 route 연결`

diff --git a/src/app/projects/[projectId]/page.tsx b/src/app/projects/[projectId]/page.tsx
new file mode 100644
index 0000000..c18e625
--- /dev/null
+++ b/src/app/projects/[projectId]/page.tsx
@@ -0,0 +1,57 @@
+import { notFound } from "next/navigation";
+import { ProjectDetailView } from "@/components/portfolio/project-detail-view";
+import { PageShell } from "@/components/portfolio/site-shell";
+import {
+  getPortfolioContent,
+  getProjectById,
+  resolveContentDebug,
+  resolveHomeTemplateId,
+  type RouteSearchParams,
+} from "@/lib/portfolio";
+
+export function generateStaticParams() {
+  return getPortfolioContent().projects.map((project) => ({
+    projectId: project.id,
+  }));
+}
+
+export default async function ProjectDetailPage({
+  params,
+  searchParams,
+}: {
+  params: Promise<{ projectId: string }>;
+  searchParams?: RouteSearchParams;
+}) {
+  const content = getPortfolioContent();
+  const { projectId } = await params;
+  const query = searchParams ? await searchParams : {};
+  const activeTemplate = resolveHomeTemplateId(query.view, content.presentation);
+  const contentDebug = resolveContentDebug(query.debug);
+  const project = getProjectById(projectId, content);
+
+  if (!project) {
+    notFound();
+  }
+
+  return (
+    <PageShell
+      contentDebug={contentDebug}
+      homeTemplate={activeTemplate}
+      profile={content.profile}
+      site={content.site}
+      templateSwitcher={{
+        activeId: activeTemplate,
+        contentDebug,
+        currentPath: `/projects/${project.id}`,
+        templates: content.presentation.templates,
+      }}
+    >
+      <ProjectDetailView
+        contentDebug={contentDebug}
+        homeTemplate={activeTemplate}
+        pageCopy={content.presentation.pages.projectDetail}
+        project={project}
+      />
+    </PageShell>
+  );
+}


## `feat(design-home): 홈과 대표 프로젝트 행동 동선 추가`

diff --git a/src/designs/design/home-route.tsx b/src/designs/design/home-route.tsx
index 9e48276..d58b751 100644
--- a/src/designs/design/home-route.tsx
+++ b/src/designs/design/home-route.tsx
@@ -111,6 +111,15 @@ function FeaturedProjectsSection({
             contentHint="src/content/presentation.json > home.design.featured"
             title={copy.title}
           />
+          <Link
+            className="inline-flex h-10 w-fit items-center gap-2 rounded-md border border-line bg-surface px-4 text-sm font-semibold text-muted transition hover:border-accent hover:text-foreground"
+            href={getTemplateHref("/projects", activeTemplate, {
+              contentDebug,
+            })}
+          >
+            {copy.actionLabel}
+            <ArrowRightIcon />
+          </Link>
         </div>
         <div className="grid gap-5">
           {projects.slice(0, 1).map((project) => (
@@ -201,6 +210,15 @@ function HeroSection({
             ))}
           </dl>
           <div className="mt-9 flex flex-wrap gap-3">
+            <Link
+              className="inline-flex h-10 items-center gap-2 rounded-md border border-accent bg-accent px-4 text-sm font-semibold text-background transition hover:-translate-y-0.5 hover:bg-accent-strong focus:outline-none focus:ring-2 focus:ring-accent focus:ring-offset-2 focus:ring-offset-background"
+              href={getTemplateHref("/projects", activeTemplate, {
+                contentDebug,
+              })}
+            >
+              {copy.primaryActionLabel}
+              <ArrowRightIcon />
+            </Link>
             {links.map((link) => (
               <ContentLinkView
                 className="inline-flex h-10 items-center gap-2 rounded-md border border-line bg-surface px-4 text-sm font-semibold text-muted transition hover:-translate-y-0.5 hover:border-accent hover:text-foreground focus:outline-none focus:ring-2 focus:ring-accent focus:ring-offset-2 focus:ring-offset-background"


