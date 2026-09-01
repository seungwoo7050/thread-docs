## `feat(design-project): 프로젝트 근거 본문 구성`

diff --git a/src/designs/design/project-detail-route.tsx b/src/designs/design/project-detail-route.tsx
index 10de0a7..789dd71 100644
--- a/src/designs/design/project-detail-route.tsx
+++ b/src/designs/design/project-detail-route.tsx
@@ -4,6 +4,7 @@ import { AvailabilityBadge } from "@/components/portfolio/availability-badge";
 import { ContentHint } from "@/components/portfolio/content-hint";
 import { ProjectLinks } from "@/components/portfolio/project-links";
 import { ProjectScreenshot } from "@/components/portfolio/project-screenshot";
+import { StackList } from "@/components/portfolio/stack-list";
 import {
   getTemplateHref,
   type HomeTemplateId,
@@ -128,3 +129,94 @@ export function ListSection({
     </section>
   );
 }
+
+export function ProjectBody({
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
+        />
+        <div className="rounded-lg border border-line bg-surface p-6">
+          <p className="text-base leading-7 text-foreground">
+            {project.architecture.summary}
+          </p>
+          <ul className="mt-6 grid gap-3">
+            {project.architecture.items.map((item) => (
+              <li className="border-l border-line pl-4 text-sm leading-6 text-muted" key={item}>
+                {item}
+              </li>
+            ))}
+          </ul>
+        </div>
+      </section>
+      <section className="grid gap-6 lg:grid-cols-[0.42fr_0.58fr]">
+        <SectionTitle
+          eyebrow={sections.screenshots.eyebrow}
+          title={sections.screenshots.title}
+        />
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
+        />
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
+  );
+}


## `feat(design-about): 소개와 개발 원칙 구성`

diff --git a/src/designs/design/about-route.tsx b/src/designs/design/about-route.tsx
index 7f96448..31246a6 100644
--- a/src/designs/design/about-route.tsx
+++ b/src/designs/design/about-route.tsx
@@ -1,7 +1,11 @@
 import Link from "next/link";
 import { ArrowRightIcon } from "@/components/icons";
 import { ContentHint } from "@/components/portfolio/content-hint";
+import { ProfilePhoto } from "@/components/portfolio/profile-photo";
 import { SectionHeading } from "@/components/portfolio/section-heading";
+import { PageShell } from "@/components/portfolio/site-shell";
+import { createDesignShellProps } from "@/designs/shell-props";
+import type { PreparedDesignRouteProps as DesignRouteProps } from "@/designs/shell-props";
 import {
   getTemplateHref,
   type HomeTemplateId,
@@ -11,6 +15,73 @@ import type {
   CurationCategoryViewModel,
 } from "@/lib/portfolio/view-models";
 
+export default function AboutRoute({
+  content,
+  contentDebug,
+  currentPath,
+}: DesignRouteProps) {
+  if (content.route !== "about") return null;
+
+  const activeTemplate = "design";
+  const shellProps = createDesignShellProps(
+    content,
+    contentDebug,
+    currentPath,
+    activeTemplate,
+  );
+  const pageCopy = content.presentation.pages.about;
+
+  return (
+    <PageShell {...shellProps}>
+      <section className="border-b border-line">
+        <div className="mx-auto grid max-w-6xl gap-10 px-5 py-20 sm:px-8 lg:grid-cols-[1fr_20rem] lg:items-center">
+          <div>
+            <ContentHint
+              enabled={contentDebug}
+              path="src/content/profile.json > name/handle/summary/photo"
+            />
+            <p className="text-sm font-medium text-muted">
+              {content.profile.name} · {content.profile.handle}
+            </p>
+            <h1 className="mt-5 max-w-3xl text-5xl font-semibold leading-tight text-foreground md:text-6xl">
+              {pageCopy.hero.title}
+            </h1>
+            <p className="mt-6 max-w-2xl text-base leading-7 text-muted">
+              {content.profile.summary}
+            </p>
+          </div>
+          {content.profile.photo ? (
+            <ProfilePhoto photo={content.profile.photo} />
+          ) : null}
+        </div>
+      </section>
+      <section className="border-b border-line">
+        <div className="mx-auto grid max-w-6xl gap-9 px-5 py-16 sm:px-8 lg:grid-cols-[0.8fr_1.2fr]">
+          <SectionHeading
+            contentDebug={contentDebug}
+            contentHint="src/content/presentation.json > pages.about.principles"
+            title={pageCopy.principles.title}
+          />
+          <div className="grid gap-4">
+            {content.profile.principles.map((principle) => (
+              <article className="rounded-lg border border-line bg-surface p-5" key={principle.title}>
+                <ContentHint
+                  enabled={contentDebug}
+                  path={`src/content/profile.json > principles[title=${principle.title}]`}
+                />
+                <h2 className="font-semibold text-foreground">{principle.title}</h2>
+                <p className="mt-3 text-sm leading-6 text-muted">
+                  {principle.body}
+                </p>
+              </article>
+            ))}
+          </div>
+        </div>
+      </section>
+    </PageShell>
+  );
+}
+
 export function CurationSection({
   content,
   contentDebug,


## `feat(design-resume): 이력서 소개와 요약 구성`

diff --git a/src/designs/design/resume-route.tsx b/src/designs/design/resume-route.tsx
new file mode 100644
index 0000000..bc3fa68
--- /dev/null
+++ b/src/designs/design/resume-route.tsx
@@ -0,0 +1,91 @@
+import { ArrowRightIcon } from "@/components/icons";
+import { ContentHint } from "@/components/portfolio/content-hint";
+import { PageShell } from "@/components/portfolio/site-shell";
+import type { PreparedDesignRouteProps as DesignRouteProps } from "@/designs/shell-props";
+import { createDesignShellProps } from "@/designs/shell-props";
+
+export default function ResumeRoute({
+  content,
+  contentDebug,
+  currentPath,
+}: DesignRouteProps) {
+  if (content.route !== "resume") return null;
+
+  const activeTemplate = "design";
+  const shellProps = createDesignShellProps(
+    content,
+    contentDebug,
+    currentPath,
+    activeTemplate,
+  );
+  const pageCopy = content.presentation.pages.resume;
+
+  return (
+    <PageShell {...shellProps}>
+      <section className="border-b border-line">
+        <div className="mx-auto grid max-w-6xl gap-8 px-5 py-20 sm:px-8 lg:grid-cols-[1fr_auto] lg:items-end">
+          <div>
+            <ContentHint
+              enabled={contentDebug}
+              path="src/content/presentation.json > pages.resume.hero + src/content/profile.json + src/content/resume.json > downloadUrl"
+            />
+            <p className="text-sm font-medium text-muted">
+              {content.profile.koreanName} · {content.profile.handle}
+            </p>
+            <h1 className="mt-5 text-5xl font-semibold leading-tight text-foreground md:text-6xl">
+              {pageCopy.hero.title}
+            </h1>
+            <p className="mt-6 max-w-2xl text-base leading-7 text-muted">
+              {pageCopy.hero.body}
+            </p>
+            <dl className="mt-8 grid max-w-2xl gap-4 sm:grid-cols-2">
+              <div className="rounded-lg border border-line bg-surface p-4">
+                <dt className="text-xs font-semibold uppercase tracking-[0.08em] text-muted">
+                  {pageCopy.identity.locationLabel}
+                </dt>
+                <dd className="mt-2 text-sm text-foreground">
+                  {content.profile.location}
+                </dd>
+              </div>
+              <div className="rounded-lg border border-line bg-surface p-4">
+                <dt className="text-xs font-semibold uppercase tracking-[0.08em] text-muted">
+                  {pageCopy.identity.availabilityLabel}
+                </dt>
+                <dd className="mt-2 text-sm leading-6 text-foreground">
+                  {content.profile.availability}
+                </dd>
+              </div>
+            </dl>
+          </div>
+          {content.resume.downloadUrl ? (
+            <a
+              className="inline-flex h-10 items-center gap-2 rounded-md border border-accent bg-accent px-4 text-sm font-semibold text-background"
+              href={content.resume.downloadUrl}
+            >
+              {pageCopy.hero.downloadLabel}
+              <ArrowRightIcon className="-rotate-45" />
+            </a>
+          ) : null}
+        </div>
+      </section>
+      <section className="border-b border-line">
+        <div className="mx-auto grid max-w-6xl gap-9 px-5 py-16 sm:px-8 lg:grid-cols-[0.8fr_1.2fr]">
+          <h2 className="text-3xl font-semibold text-foreground">
+            {pageCopy.summary.title}
+          </h2>
+          <div className="grid gap-4">
+            {content.resume.summary.map((item) => (
+              <p className="text-base leading-7 text-muted" key={item}>
+                <ContentHint
+                  enabled={contentDebug}
+                  path="src/content/resume.json > summary[]"
+                />
+                {item}
+              </p>
+            ))}
+          </div>
+        </div>
+      </section>
+    </PageShell>
+  );
+}


## `feat(design-journey): 여정 마일스톤 카드 추가`

diff --git a/src/designs/design/journey-route.tsx b/src/designs/design/journey-route.tsx
new file mode 100644
index 0000000..b9ca61f
--- /dev/null
+++ b/src/designs/design/journey-route.tsx
@@ -0,0 +1,83 @@
+import Link from "next/link";
+import { ArrowRightIcon } from "@/components/icons";
+import { ContentHint } from "@/components/portfolio/content-hint";
+import {
+  getTemplateHref,
+  type HomeTemplateId,
+  type PresentationContent,
+} from "@/lib/portfolio";
+import type { JourneyMilestoneViewModel } from "@/lib/portfolio/view-models";
+
+export function MilestoneCard({
+  contentDebug,
+  homeTemplate,
+  index,
+  labels,
+  milestone,
+}: {
+  contentDebug: boolean;
+  homeTemplate: HomeTemplateId;
+  index: number;
+  labels: PresentationContent["pages"]["journey"]["narrative"]["labels"];
+  milestone: JourneyMilestoneViewModel;
+}) {
+  return (
+    <li className="rounded-lg border border-line bg-surface p-6">
+      <ContentHint
+        enabled={contentDebug}
+        path={`src/content/journey-narrative.json > milestones[id=${milestone.id}]`}
+      />
+      <div className="flex flex-wrap items-baseline gap-3">
+        <span className="text-xs font-semibold uppercase tracking-[0.08em] text-accent">
+          {String(index + 1).padStart(2, "0")} · {milestone.date}
+        </span>
+      </div>
+      <h3 className="mt-3 text-xl font-semibold text-foreground">
+        {milestone.title}
+      </h3>
+      <dl className="mt-5 grid gap-4">
+        <div>
+          <dt className="text-xs font-semibold uppercase tracking-[0.08em] text-muted">
+            {labels.state}
+          </dt>
+          <dd className="mt-2 text-sm leading-6 text-muted md:text-base md:leading-7">
+            {milestone.state}
+          </dd>
+        </div>
+        <div>
+          <dt className="text-xs font-semibold uppercase tracking-[0.08em] text-muted">
+            {labels.reason}
+          </dt>
+          <dd className="mt-2 text-sm leading-6 text-muted md:text-base md:leading-7">
+            {milestone.reason}
+          </dd>
+        </div>
+        <div>
+          <dt className="text-xs font-semibold uppercase tracking-[0.08em] text-muted">
+            {labels.result}
+          </dt>
+          <dd className="mt-2 text-sm leading-6 text-muted md:text-base md:leading-7">
+            {milestone.result}
+          </dd>
+        </div>
+      </dl>
+      {milestone.anchorProjects.length > 0 ? (
+        <ul className="mt-5 flex flex-wrap gap-2">
+          {milestone.anchorProjects.map((project) => (
+            <li key={project.id}>
+              <Link
+                className="inline-flex items-center gap-2 rounded-md border border-line bg-surface-soft px-3 py-2 text-xs font-semibold text-muted transition hover:border-accent hover:text-foreground"
+                href={getTemplateHref(`/projects/${project.id}`, homeTemplate, {
+                  contentDebug,
+                })}
+              >
+                {project.title}
+                <ArrowRightIcon />
+              </Link>
+            </li>
+          ))}
+        </ul>
+      ) : null}
+    </li>
+  );
+}


## `feat(design-interview): 인터뷰 트랙 표 구조 추가`

diff --git a/src/designs/design/interview-map-route.tsx b/src/designs/design/interview-map-route.tsx
new file mode 100644
index 0000000..3d24e7f
--- /dev/null
+++ b/src/designs/design/interview-map-route.tsx
@@ -0,0 +1,65 @@
+import { ContentHint } from "@/components/portfolio/content-hint";
+import type { HomeTemplateId } from "@/lib/portfolio";
+import type {
+  InterviewMapTrackViewModel,
+  InterviewMapViewModel,
+} from "@/lib/portfolio/view-models";
+
+export function TrackSection({
+  contentDebug,
+  index,
+  pageCopy,
+  track,
+}: {
+  contentDebug: boolean;
+  homeTemplate: HomeTemplateId;
+  index: number;
+  pageCopy: InterviewMapViewModel["presentation"]["pages"]["interviewMap"];
+  track: InterviewMapTrackViewModel;
+}) {
+  return (
+    <section
+      className={`border-b border-line ${index % 2 === 0 ? "" : "bg-background-soft"}`}
+      id={`track-${track.id}`}
+    >
+      <div className="mx-auto grid max-w-6xl gap-9 px-5 py-16 sm:px-8 lg:grid-cols-[0.8fr_1.2fr]">
+        <div>
+          <ContentHint
+            enabled={contentDebug}
+            path={`src/content/interview-map.json > tracks[id=${track.id}]`}
+          />
+          <p className="text-xs font-semibold uppercase tracking-[0.08em] text-accent">
+            {pageCopy.tracks.itemCountTemplate.replace(
+              "{count}",
+              String(track.items.length),
+            )}
+          </p>
+          <h2 className="mt-3 text-3xl font-semibold text-foreground">
+            {track.label}
+          </h2>
+          <p className="mt-4 text-sm leading-6 text-muted md:text-base">
+            {track.body}
+          </p>
+        </div>
+        <div className="overflow-hidden rounded-lg border border-line bg-surface">
+          <table className="w-full border-collapse text-left text-sm">
+            <thead>
+              <tr className="border-b border-line bg-surface-soft text-xs font-semibold uppercase tracking-[0.08em] text-muted">
+                <th className="px-4 py-3" scope="col">
+                  {pageCopy.tracks.questionLabel}
+                </th>
+                <th className="px-4 py-3" scope="col">
+                  {pageCopy.tracks.answerLabel}
+                </th>
+                <th className="px-4 py-3" scope="col">
+                  {pageCopy.tracks.depthLabel}
+                </th>
+              </tr>
+            </thead>
+            <tbody />
+          </table>
+        </div>
+      </div>
+    </section>
+  );
+}


