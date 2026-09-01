## `feat(brutalist): 여정 milestone 구성`

diff --git a/src/designs/brutalist/brutalist-route.tsx b/src/designs/brutalist/brutalist-route.tsx
index ffde642..d9e9dd6 100644
--- a/src/designs/brutalist/brutalist-route.tsx
+++ b/src/designs/brutalist/brutalist-route.tsx
@@ -1299,6 +1299,81 @@ export function ContactView({
   );
 }
 
+export function JourneyView({
+  content,
+  contentDebug,
+}: {
+  content: PortfolioContent;
+  contentDebug: boolean;
+}) {
+  const copy = content.presentation.pages.journey;
+  const narrative = content.journeyNarrative;
+  return (
+    <>
+      <section className={styles.pageHero}>
+        <PageLabel index="01" label={copy.hero.eyebrow} />
+        <h1>{copy.hero.title}</h1>
+        <p>{narrative.intro}</p>
+      </section>
+
+      <section className={`${styles.section} ${styles.yellowSection}`}>
+        <SectionHeader
+          body={copy.narrative.body}
+          number="02"
+          title={copy.narrative.title}
+        />
+        {narrative.milestones.length > 0 ? (
+          <ol className={styles.milestoneList}>
+            {narrative.milestones.map((milestone, index) => {
+            const projects = milestone.anchorProjectIds
+              .map((id) => content.projects.find((project) => project.id === id))
+              .filter((item): item is PortfolioProject => Boolean(item));
+
+            return (
+              <li key={milestone.id}>
+                <header>
+                  <span>{String(index + 1).padStart(2, "0")}</span>
+                  <time>{milestone.date}</time>
+                  <h3>{milestone.title}</h3>
+                </header>
+                <dl>
+                  <div>
+                    <dt>{copy.narrative.labels.state}</dt>
+                    <dd>{milestone.state}</dd>
+                  </div>
+                  <div>
+                    <dt>{copy.narrative.labels.reason}</dt>
+                    <dd>{milestone.reason}</dd>
+                  </div>
+                  <div>
+                    <dt>{copy.narrative.labels.result}</dt>
+                    <dd>{milestone.result}</dd>
+                  </div>
+                </dl>
+                {projects.length > 0 ? (
+                  <div className={styles.anchorLinks}>
+                    {projects.map((project) => (
+                      <Link
+                        href={brutalistHref(`/projects/${project.id}`, contentDebug)}
+                        key={project.id}
+                      >
+                        {project.title} ↗
+                      </Link>
+                    ))}
+                  </div>
+                ) : null}
+              </li>
+            );
+            })}
+          </ol>
+        ) : (
+          <EmptyBlock message={content.presentation.ui.emptyStates.journey} />
+        )}
+      </section>
+    </>
+  );
+}
+
 export function PageLabel({ index, label }: { index: string; label: string }) {
   return (
     <p className={styles.pageLabel}>


## `feat(brutalist): 인터뷰 근거 archive 구성`

diff --git a/src/designs/brutalist/brutalist-route.tsx b/src/designs/brutalist/brutalist-route.tsx
index 2341617..1d9df5c 100644
--- a/src/designs/brutalist/brutalist-route.tsx
+++ b/src/designs/brutalist/brutalist-route.tsx
@@ -1442,6 +1442,10 @@ export function InterviewMapView({
 }) {
   const copy = content.presentation.pages.interviewMap;
   const data = content.interviewMap;
+  const projectsById = new Map(
+    content.projects.map((project) => [project.id, project]),
+  );
+
   return (
     <>
       <section className={styles.pageHero}>
@@ -1477,6 +1481,90 @@ export function InterviewMapView({
         </ol>
       </nav>
 
+      <div className={styles.trackArchive}>
+        {data.tracks.length > 0 ? (
+          data.tracks.map((track, trackIndex) => (
+            <section
+              className={styles.trackSection}
+              id={`track-${track.id}`}
+              key={track.id}
+            >
+              <header>
+                <span>
+                  {copy.tracks.indexLabel}{" "}
+                  {String(trackIndex + 1).padStart(2, "0")}
+                </span>
+                <h2>{track.label}</h2>
+                <p>{track.body}</p>
+                <strong>
+                  {renderCopyTemplate(copy.tracks.itemCountTemplate, {
+                    count: String(track.items.length),
+                  })}
+                </strong>
+              </header>
+              {track.items.length > 0 ? (
+                <ol className={styles.questionList}>
+                  {track.items.map((item, itemIndex) => (
+                    <li key={item.label}>
+                      <div className={styles.questionPrompt}>
+                        <span>
+                          {copy.tracks.questionLabel}{" "}
+                          {String(itemIndex + 1).padStart(2, "0")}
+                        </span>
+                        <h3>{item.label}</h3>
+                        <a href={item.reference} rel="noreferrer" target="_blank">
+                          {copy.tracks.referenceLabel} ↗
+                        </a>
+                      </div>
+                      <div className={styles.answerGrid}>
+                        {item.answers.some((answer) =>
+                          projectsById.has(answer.projectId),
+                        ) ? (
+                          item.answers.flatMap((answer, answerIndex) => {
+                            const project = projectsById.get(answer.projectId);
+
+                            if (!project) {
+                              return [];
+                            }
+
+                            return [
+                              <article
+                                key={`${answer.projectId}-${answerIndex}`}
+                              >
+                                <span>{copy.tracks.answerLabel}</span>
+                                <Link
+                                  href={brutalistHref(
+                                    `/projects/${project.id}`,
+                                    contentDebug,
+                                  )}
+                                >
+                                  {project.title}{" "}
+                                  <span aria-hidden="true">↗</span>
+                                </Link>
+                                <p>
+                                  <b>{copy.tracks.depthLabel}:</b> {answer.depth}
+                                </p>
+                              </article>,
+                            ];
+                          })
+                        ) : (
+                          <p className={styles.emptyAnswer}>
+                            {content.presentation.ui.emptyStates.noMappedEvidence}
+                          </p>
+                        )}
+                      </div>
+                    </li>
+                  ))}
+                </ol>
+              ) : (
+                <p className={styles.emptyAnswer}>{copy.tracks.emptyLabel}</p>
+              )}
+            </section>
+          ))
+        ) : (
+          <EmptyBlock message={copy.tracks.emptyLabel} />
+        )}
+      </div>
     </>
   );
 }


## `refactor(brutalist): 내부 helper 공개 범위 정리`

diff --git a/src/designs/brutalist/brutalist-route.tsx b/src/designs/brutalist/brutalist-route.tsx
index 9d795f6..f477f8f 100644
--- a/src/designs/brutalist/brutalist-route.tsx
+++ b/src/designs/brutalist/brutalist-route.tsx
@@ -32,13 +32,13 @@ type CopyTemplateToken =
   | "title"
   | "year";
 
-export function brutalistHref(path: string, contentDebug: boolean) {
+function brutalistHref(path: string, contentDebug: boolean) {
   return getTemplateHref(path, DESIGN_ID, {
     contentDebug,
   });
 }
 
-export function renderCopyTemplate(
+function renderCopyTemplate(
   template: string,
   values: Partial<Record<CopyTemplateToken, string>>,
 ) {
@@ -48,14 +48,14 @@ export function renderCopyTemplate(
   );
 }
 
-export function getProjectTags(project: PortfolioProject, limit = 4) {
+function getProjectTags(project: PortfolioProject, limit = 4) {
   const tags = project.tags;
   const source = tags && tags.length > 0 ? tags : project.stack;
 
   return source.filter(Boolean).slice(0, limit);
 }
 
-export function groupProjects(content: PortfolioContent): GroupedProjects[] {
+function groupProjects(content: PortfolioContent): GroupedProjects[] {
   return content.projectGroups
     .map((group) => {
       const projects = content.projects.filter((project) => {
@@ -67,7 +67,7 @@ export function groupProjects(content: PortfolioContent): GroupedProjects[] {
     .filter((group) => group.projects.length > 0);
 }
 
-export function getHomeMetrics(content: PortfolioContent) {
+function getHomeMetrics(content: PortfolioContent) {
   return content.projectMetrics.slice(0, 4).map((metric) => ({
     description: metric.description,
     id: metric.id,
@@ -76,7 +76,7 @@ export function getHomeMetrics(content: PortfolioContent) {
   }));
 }
 
-export function isCurrentNavigation(href: string, currentPath: string) {
+function isCurrentNavigation(href: string, currentPath: string) {
   if (href === "/") {
     return currentPath === "/";
   }
@@ -84,7 +84,7 @@ export function isCurrentNavigation(href: string, currentPath: string) {
   return currentPath === href || currentPath.startsWith(`${href}/`);
 }
 
-export function getNavigationLabel(
+function getNavigationLabel(
   content: PortfolioContent,
   href: string,
   fallback: string,
@@ -94,7 +94,7 @@ export function getNavigationLabel(
   );
 }
 
-export function getRouteLabel(
+function getRouteLabel(
   content: PortfolioContent,
   route: DesignRouteProps["route"],
 ) {
@@ -442,7 +442,7 @@ export function HomeView({
   );
 }
 
-export function SignalStrip({ text }: { text: string }) {
+function SignalStrip({ text }: { text: string }) {
   return (
     <div aria-hidden="true" className={styles.signalStrip}>
       <div>
@@ -453,7 +453,7 @@ export function SignalStrip({ text }: { text: string }) {
   );
 }
 
-export function SectionHeader({
+function SectionHeader({
   body,
   number,
   title,
@@ -471,7 +471,7 @@ export function SectionHeader({
   );
 }
 
-export function ProjectIndexRow({
+function ProjectIndexRow({
   contentDebug,
   project,
 }: {
@@ -726,7 +726,7 @@ export function ProjectDetailView({
   );
 }
 
-export function ProjectMedia({
+function ProjectMedia({
   image,
   label,
   priority = false,
@@ -751,7 +751,7 @@ export function ProjectMedia({
   );
 }
 
-export function ProjectActions({
+function ProjectActions({
   contentDebug,
   links,
 }: {
@@ -780,7 +780,7 @@ export function ProjectActions({
 }
 
 
-export function ActionLink({
+function ActionLink({
   children,
   className,
   contentDebug,
@@ -815,7 +815,7 @@ export function ActionLink({
   );
 }
 
-export function DetailTextSection({
+function DetailTextSection({
   body,
   eyebrow,
   number,
@@ -838,7 +838,7 @@ export function DetailTextSection({
 }
 
 
-export function DetailListSection({
+function DetailListSection({
   emptyMessage,
   eyebrow,
   intro,
@@ -1587,7 +1587,7 @@ export function InterviewMapView({
   );
 }
 
-export function PageLabel({ index, label }: { index: string; label: string }) {
+function PageLabel({ index, label }: { index: string; label: string }) {
   return (
     <p className={styles.pageLabel}>
       <span>{index}</span>
@@ -1596,7 +1596,7 @@ export function PageLabel({ index, label }: { index: string; label: string }) {
   );
 }
 
-export function CurationHeading({
+function CurationHeading({
   label,
   title,
 }: {
@@ -1612,7 +1612,7 @@ export function CurationHeading({
 }
 
 
-export function ContactBand({
+function ContactBand({
   content,
   contentDebug,
 }: {
@@ -1634,7 +1634,7 @@ export function ContactBand({
   );
 }
 
-export function EmptyBlock({ message }: { message: string }) {
+function EmptyBlock({ message }: { message: string }) {
   return (
     <div className={styles.emptyBlock} role="status">
       <span aria-hidden="true">□</span>


## `feat(brutalist): 모든 route를 renderer에 통합`

diff --git a/src/designs/brutalist/brutalist-route.tsx b/src/designs/brutalist/brutalist-route.tsx
index f477f8f..b819b3a 100644
--- a/src/designs/brutalist/brutalist-route.tsx
+++ b/src/designs/brutalist/brutalist-route.tsx
@@ -132,8 +132,67 @@ function getRouteLabel(
   }
 }
 
+export function BrutalistRoute({
+  content,
+  contentDebug,
+  currentPath,
+  project,
+  route,
+}: DesignRouteProps) {
+  let view: React.ReactNode;
 
-export function BrutalistShell({
+  switch (route) {
+    case "home":
+      view = <HomeView content={content} contentDebug={contentDebug} />;
+      break;
+    case "projects":
+      view = <ProjectsView content={content} contentDebug={contentDebug} />;
+      break;
+    case "project-detail":
+      view = (
+        <ProjectDetailView
+          content={content}
+          contentDebug={contentDebug}
+          project={project}
+        />
+      );
+      break;
+    case "about":
+      view = <AboutView content={content} contentDebug={contentDebug} />;
+      break;
+    case "resume":
+      view = <ResumeView content={content} contentDebug={contentDebug} />;
+      break;
+    case "contact":
+      view = <ContactView content={content} contentDebug={contentDebug} />;
+      break;
+    case "journey":
+      view = <JourneyView content={content} contentDebug={contentDebug} />;
+      break;
+    case "interview-map":
+      view = (
+        <InterviewMapView
+          content={content}
+          contentDebug={contentDebug}
+          currentPath={currentPath}
+        />
+      );
+      break;
+  }
+
+  return (
+    <BrutalistShell
+      content={content}
+      contentDebug={contentDebug}
+      currentPath={currentPath}
+      route={route}
+    >
+      {view}
+    </BrutalistShell>
+  );
+}
+
+function BrutalistShell({
   children,
   content,
   contentDebug,
@@ -265,7 +324,7 @@ export function BrutalistShell({
   );
 }
 
-export function HomeView({
+function HomeView({
   content,
   contentDebug,
 }: {
@@ -504,8 +563,7 @@ function ProjectIndexRow({
   );
 }
 
-
-export function ProjectsView({
+function ProjectsView({
   content,
   contentDebug,
 }: {
@@ -566,7 +624,7 @@ export function ProjectsView({
   );
 }
 
-export function ProjectDetailView({
+function ProjectDetailView({
   content,
   contentDebug,
   project,
@@ -779,7 +837,6 @@ function ProjectActions({
   );
 }
 
-
 function ActionLink({
   children,
   className,
@@ -837,7 +894,6 @@ function DetailTextSection({
   );
 }
 
-
 function DetailListSection({
   emptyMessage,
   eyebrow,
@@ -884,8 +940,7 @@ function DetailListSection({
   );
 }
 
-
-export function AboutView({
+function AboutView({
   content,
   contentDebug,
 }: {
@@ -1074,8 +1129,7 @@ export function AboutView({
   );
 }
 
-
-export function ResumeView({
+function ResumeView({
   content,
   contentDebug,
 }: {
@@ -1227,7 +1281,7 @@ export function ResumeView({
   );
 }
 
-export function ContactView({
+function ContactView({
   content,
   contentDebug,
 }: {
@@ -1299,7 +1353,7 @@ export function ContactView({
   );
 }
 
-export function JourneyView({
+function JourneyView({
   content,
   contentDebug,
 }: {
@@ -1431,7 +1485,7 @@ export function JourneyView({
   );
 }
 
-export function InterviewMapView({
+function InterviewMapView({
   content,
   contentDebug,
   currentPath,
@@ -1611,7 +1665,6 @@ function CurationHeading({
   );
 }
 
-
 function ContactBand({
   content,
   contentDebug,


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
 
