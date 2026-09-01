## `refactor(journey): 모든 renderer에 여정 view model 적용`

diff --git a/src/app/journey/page.tsx b/src/app/journey/page.tsx
index b7a9196..0dcb5c3 100644
--- a/src/app/journey/page.tsx
+++ b/src/app/journey/page.tsx
@@ -37,17 +37,17 @@ export default async function JourneyPage({
       currentPath: "/journey",
       searchParams,
     });
+  const viewModel = createJourneyViewModel(content);
 
   if (hasDedicatedRouteRenderer(activeTemplate)) {
     return renderDesignRoute(activeTemplate, {
-      content,
       contentDebug,
       currentPath: "/journey",
       route: "journey",
+      viewModel,
     });
   }
 
-  const viewModel = createJourneyViewModel(content);
   const JourneyRoute = activeTemplate === "design" ? DesignJourneyRoute : ClassicJourneyRoute;
 
   return (
diff --git a/src/designs/brutalist/brutalist-route.tsx b/src/designs/brutalist/brutalist-route.tsx
index c6c60f6..f7fa7b3 100644
--- a/src/designs/brutalist/brutalist-route.tsx
+++ b/src/designs/brutalist/brutalist-route.tsx
@@ -12,6 +12,7 @@ import type {
   AboutViewModel,
   ContactViewModel,
   HomeViewModel,
+  JourneyViewModel,
   ProjectDetailViewModel,
   ProjectIndexViewModel,
   ResumeViewModel,
@@ -143,7 +144,12 @@ export function BrutalistRoute({
       view = <ContactView content={content} contentDebug={contentDebug} />;
       break;
     case "journey":
-      view = <JourneyView content={content} contentDebug={contentDebug} />;
+      view = (
+        <JourneyView
+          content={content as JourneyViewModel}
+          contentDebug={contentDebug}
+        />
+      );
       break;
     case "interview-map":
       view = (
@@ -1331,14 +1337,11 @@ function JourneyView({
   content,
   contentDebug,
 }: {
-  content: PortfolioContent;
+  content: JourneyViewModel;
   contentDebug: boolean;
 }) {
   const copy = content.presentation.pages.journey;
   const narrative = content.journeyNarrative;
-  const projectsById = new Map(
-    content.projects.map((project) => [project.id, project]),
-  );
 
   return (
     <>
@@ -1354,13 +1357,9 @@ function JourneyView({
           number="02"
           title={copy.narrative.title}
         />
-        {narrative.milestones.length > 0 ? (
+        {content.milestones.length > 0 ? (
           <ol className={styles.milestoneList}>
-            {narrative.milestones.map((milestone, index) => {
-            const projects = milestone.anchorProjectIds
-              .map((id) => content.projects.find((project) => project.id === id))
-              .filter((item): item is PortfolioProject => Boolean(item));
-
+            {content.milestones.map((milestone, index) => {
             return (
               <li key={milestone.id}>
                 <header>
@@ -1382,9 +1381,9 @@ function JourneyView({
                     <dd>{milestone.result}</dd>
                   </div>
                 </dl>
-                {projects.length > 0 ? (
+                {milestone.anchorProjects.length > 0 ? (
                   <div className={styles.anchorLinks}>
-                    {projects.map((project) => (
+                    {milestone.anchorProjects.map((project) => (
                       <Link
                         href={brutalistHref(`/projects/${project.id}`, contentDebug)}
                         key={project.id}
@@ -1409,13 +1408,9 @@ function JourneyView({
           number="03"
           title={copy.timeline.title}
         />
-        {content.journey.length > 0 ? (
+        {content.timelineItems.length > 0 ? (
           <ol className={styles.archiveTimeline}>
-            {content.journey.map((item, index) => {
-              const linkedProject = item.projectId
-                ? projectsById.get(item.projectId)
-                : undefined;
-
+            {content.timelineItems.map((item, index) => {
               return (
                 <li key={`${item.date}-${item.title}`}>
                   <span>{String(index + 1).padStart(2, "0")}</span>
@@ -1427,14 +1422,14 @@ function JourneyView({
                     <h3>{item.title}</h3>
                     <p>{item.body}</p>
                   </div>
-                  {linkedProject ? (
+                  {item.project ? (
                     <Link
                       aria-label={renderCopyTemplate(
                         content.presentation.ui.openItemAriaTemplate,
                         { title: item.title },
                       )}
                       href={brutalistHref(
-                        `/projects/${linkedProject.id}`,
+                        `/projects/${item.project.id}`,
                         contentDebug,
                       )}
                     >
diff --git a/src/designs/cinematic/cinematic-route.tsx b/src/designs/cinematic/cinematic-route.tsx
index 1f96871..71ba761 100644
--- a/src/designs/cinematic/cinematic-route.tsx
+++ b/src/designs/cinematic/cinematic-route.tsx
@@ -12,6 +12,7 @@ import type {
   AboutViewModel,
   ContactViewModel,
   HomeViewModel,
+  JourneyViewModel,
   ProjectDetailViewModel,
   ResumeViewModel,
 } from "@/lib/portfolio/view-models";
@@ -646,6 +647,7 @@ function ContactView({ content, contentDebug }: DesignRouteProps) {
 }
 
 function JourneyView({ content, contentDebug }: DesignRouteProps) {
+  const viewModel = content as JourneyViewModel;
   const copy = content.presentation.pages.journey;
   const narrative = content.journeyNarrative;
 
@@ -663,11 +665,7 @@ function JourneyView({ content, contentDebug }: DesignRouteProps) {
           <p>{copy.narrative.body}</p>
         </div>
       <ol className={styles.timeline}>
-        {narrative.milestones.map((milestone, index) => {
-          const projects = milestone.anchorProjectIds
-            .map((projectId) => content.projects.find((project) => project.id === projectId))
-            .filter((project): project is PortfolioProject => Boolean(project));
-
+        {viewModel.milestones.map((milestone, index) => {
           return (
           <li key={milestone.id}>
             <span>{String(index + 1).padStart(2, "0")}</span>
@@ -679,9 +677,9 @@ function JourneyView({ content, contentDebug }: DesignRouteProps) {
                 <div><dt>{copy.narrative.labels.reason}</dt><dd>{milestone.reason}</dd></div>
                 <div><dt>{copy.narrative.labels.result}</dt><dd>{milestone.result}</dd></div>
               </dl>
-              {projects.length > 0 ? (
+              {milestone.anchorProjects.length > 0 ? (
                 <div className={styles.evidenceLinks}>
-                  {projects.map((project) => (
+                  {milestone.anchorProjects.map((project) => (
                     <Link
                       href={routeHref(`/projects/${project.id}`, contentDebug)}
                       key={project.id}
diff --git a/src/designs/editorial/editorial-route.tsx b/src/designs/editorial/editorial-route.tsx
index acbe1f5..cf3b433 100644
--- a/src/designs/editorial/editorial-route.tsx
+++ b/src/designs/editorial/editorial-route.tsx
@@ -15,6 +15,7 @@ import type {
   AboutViewModel,
   ContactViewModel,
   HomeViewModel,
+  JourneyViewModel,
   ProjectDetailViewModel,
   ProjectIndexViewModel,
   ResumeViewModel,
@@ -1147,6 +1148,7 @@ function ContactRoute({ content, contentDebug }: EditorialRouteProps) {
 }
 
 function JourneyRoute({ content, contentDebug }: EditorialRouteProps) {
+  const viewModel = content as JourneyViewModel;
   const copy = content.presentation.pages.journey;
   const narrative = content.journeyNarrative;
   const ui = content.presentation.ui;
@@ -1169,11 +1171,7 @@ function JourneyRoute({ content, contentDebug }: EditorialRouteProps) {
         <SectionKicker number="01">{copy.narrative.title}</SectionKicker>
         <p className={styles.sectionLead}>{copy.narrative.body}</p>
         <ol>
-          {narrative.milestones.length > 0 ? narrative.milestones.map((milestone, index) => {
-            const anchorProjects = milestone.anchorProjectIds
-              .map((id) => content.projects.find((project) => project.id === id))
-              .filter((item): item is PortfolioProject => Boolean(item));
-
+          {viewModel.milestones.length > 0 ? viewModel.milestones.map((milestone, index) => {
             return (
               <li key={milestone.id}>
                 <div className={styles.milestoneDate}>
@@ -1193,12 +1191,12 @@ function JourneyRoute({ content, contentDebug }: EditorialRouteProps) {
                       <dd>{milestone.result}</dd>
                     </div>
                   </dl>
-                  {anchorProjects.length > 0 ? (
+                  {milestone.anchorProjects.length > 0 ? (
                     <nav
                       aria-label={ui.projectNavigationAriaLabel}
                       className={styles.milestoneLinks}
                     >
-                      {anchorProjects.map((project) => (
+                      {milestone.anchorProjects.map((project) => (
                         <Link
                           href={editorialHref(`/projects/${project.id}`, contentDebug)}
                           key={project.id}
@@ -1224,26 +1222,22 @@ function JourneyRoute({ content, contentDebug }: EditorialRouteProps) {
           <p>{copy.timeline.body}</p>
         </div>
         <ol>
-          {content.journey.length > 0 ? content.journey.map((item, index) => {
-            const linkedProject = item.projectId
-              ? content.projects.find((project) => project.id === item.projectId)
-              : null;
-
+          {viewModel.timelineItems.length > 0 ? viewModel.timelineItems.map((item, index) => {
             return (
               <li key={`${item.date}-${item.title}-${index}`}>
                 <time>{item.date}{item.endDate ? ` — ${item.endDate}` : ""}</time>
                 <p>{item.category}</p>
                 <h3>{item.title}</h3>
                 <span>{item.body}</span>
-                {linkedProject ? (
+                {item.project ? (
                   <Link
                     className={styles.timelineProjectLink}
                     href={editorialHref(
-                      `/projects/${linkedProject.id}`,
+                      `/projects/${item.project.id}`,
                       contentDebug,
                     )}
                   >
-                    {copy.now.anchorLabel} · {linkedProject.title} <Arrow />
+                    {copy.now.anchorLabel} · {item.project.title} <Arrow />
                   </Link>
                 ) : null}
               </li>


## `refactor(interview): 모든 renderer에 인터뷰 view model 적용`

diff --git a/src/app/interview-map/page.tsx b/src/app/interview-map/page.tsx
index 39dd00c..39eb3d4 100644
--- a/src/app/interview-map/page.tsx
+++ b/src/app/interview-map/page.tsx
@@ -37,17 +37,17 @@ export default async function InterviewMapPage({
       currentPath: "/interview-map",
       searchParams,
     });
+  const viewModel = createInterviewMapViewModel(content);
 
   if (hasDedicatedRouteRenderer(activeTemplate)) {
     return renderDesignRoute(activeTemplate, {
-      content,
       contentDebug,
       currentPath: "/interview-map",
       route: "interview-map",
+      viewModel,
     });
   }
 
-  const viewModel = createInterviewMapViewModel(content);
   const InterviewMapRoute =
     activeTemplate === "design" ? DesignInterviewMapRoute : ClassicInterviewMapRoute;
 
diff --git a/src/designs/brutalist/brutalist-route.tsx b/src/designs/brutalist/brutalist-route.tsx
index f7fa7b3..9481993 100644
--- a/src/designs/brutalist/brutalist-route.tsx
+++ b/src/designs/brutalist/brutalist-route.tsx
@@ -12,6 +12,7 @@ import type {
   AboutViewModel,
   ContactViewModel,
   HomeViewModel,
+  InterviewMapViewModel,
   JourneyViewModel,
   ProjectDetailViewModel,
   ProjectIndexViewModel,
@@ -154,7 +155,7 @@ export function BrutalistRoute({
     case "interview-map":
       view = (
         <InterviewMapView
-          content={content}
+          content={content as InterviewMapViewModel}
           contentDebug={contentDebug}
           currentPath={currentPath}
         />
@@ -1459,15 +1460,12 @@ function InterviewMapView({
   contentDebug,
   currentPath,
 }: {
-  content: PortfolioContent;
+  content: InterviewMapViewModel;
   contentDebug: boolean;
   currentPath: string;
 }) {
   const copy = content.presentation.pages.interviewMap;
   const data = content.interviewMap;
-  const projectsById = new Map(
-    content.projects.map((project) => [project.id, project]),
-  );
 
   return (
     <>
@@ -1488,7 +1486,7 @@ function InterviewMapView({
       <nav aria-label={copy.tracks.title} className={styles.trackNavigation}>
         <span>{copy.tracks.indexLabel}</span>
         <ol>
-          {data.tracks.map((track, index) => (
+          {content.tracks.map((track, index) => (
             <li key={track.id}>
               <Link
                 href={brutalistHref(
@@ -1506,7 +1504,7 @@ function InterviewMapView({
 
       <div className={styles.trackArchive}>
         {data.tracks.length > 0 ? (
-          data.tracks.map((track, trackIndex) => (
+          content.tracks.map((track, trackIndex) => (
             <section
               className={styles.trackSection}
               id={`track-${track.id}`}
@@ -1540,11 +1538,9 @@ function InterviewMapView({
                         </a>
                       </div>
                       <div className={styles.answerGrid}>
-                        {item.answers.some((answer) =>
-                          projectsById.has(answer.projectId),
-                        ) ? (
+                        {item.answers.some((answer) => answer.project) ? (
                           item.answers.flatMap((answer, answerIndex) => {
-                            const project = projectsById.get(answer.projectId);
+                            const project = answer.project;
 
                             if (!project) {
                               return [];
diff --git a/src/designs/cinematic/cinematic-route.tsx b/src/designs/cinematic/cinematic-route.tsx
index 71ba761..cbf7c8f 100644
--- a/src/designs/cinematic/cinematic-route.tsx
+++ b/src/designs/cinematic/cinematic-route.tsx
@@ -12,6 +12,7 @@ import type {
   AboutViewModel,
   ContactViewModel,
   HomeViewModel,
+  InterviewMapViewModel,
   JourneyViewModel,
   ProjectDetailViewModel,
   ResumeViewModel,
@@ -730,9 +731,9 @@ function JourneyView({ content, contentDebug }: DesignRouteProps) {
 }
 
 function InterviewMapView({ content, contentDebug }: DesignRouteProps) {
+  const viewModel = content as InterviewMapViewModel;
   const copy = content.presentation.pages.interviewMap;
   const data = content.interviewMap;
-  const projectsById = new Map(content.projects.map((project) => [project.id, project]));
 
   return (
     <>
@@ -756,7 +757,7 @@ function InterviewMapView({ content, contentDebug }: DesignRouteProps) {
         </div>
       </section>
       <section className={styles.trackGrid}>
-        {data.tracks.map((track, trackIndex) => (
+        {viewModel.tracks.map((track, trackIndex) => (
           <article key={track.id}>
             <ChapterLabel index={trackIndex + 3}>{track.label}</ChapterLabel>
             <p>{track.body}</p>
@@ -773,7 +774,7 @@ function InterviewMapView({ content, contentDebug }: DesignRouteProps) {
                 {item.answers.length > 0 ? (
                   <div className={styles.answerList}>
                     {item.answers.map((answer) => {
-                      const project = projectsById.get(answer.projectId);
+                      const project = answer.project;
 
                       return (
                         <article key={`${item.label}-${answer.projectId}`}>
diff --git a/src/designs/editorial/editorial-route.tsx b/src/designs/editorial/editorial-route.tsx
index cf3b433..d778c59 100644
--- a/src/designs/editorial/editorial-route.tsx
+++ b/src/designs/editorial/editorial-route.tsx
@@ -15,6 +15,7 @@ import type {
   AboutViewModel,
   ContactViewModel,
   HomeViewModel,
+  InterviewMapViewModel,
   JourneyViewModel,
   ProjectDetailViewModel,
   ProjectIndexViewModel,
@@ -1261,9 +1262,9 @@ function JourneyRoute({ content, contentDebug }: EditorialRouteProps) {
 }
 
 function InterviewMapRoute({ content, contentDebug }: EditorialRouteProps) {
+  const viewModel = content as InterviewMapViewModel;
   const copy = content.presentation.pages.interviewMap;
   const data = content.interviewMap;
-  const projectById = new Map(content.projects.map((project) => [project.id, project]));
   const ui = content.presentation.ui;
 
   return (
@@ -1292,7 +1293,7 @@ function InterviewMapRoute({ content, contentDebug }: EditorialRouteProps) {
 
       <nav aria-label={copy.tracks.title} className={styles.chapterNav}>
         <strong className={styles.chapterNavLabel}>{copy.tracks.indexLabel}</strong>
-        {data.tracks.map((track, index) => (
+        {viewModel.tracks.map((track, index) => (
           <a href={`#editorial-track-${track.id}`} key={track.id}>
             <span>{twoDigits(index)}</span>
             {track.label}
@@ -1301,7 +1302,7 @@ function InterviewMapRoute({ content, contentDebug }: EditorialRouteProps) {
       </nav>
 
       <div className={styles.interviewTracks}>
-        {data.tracks.length > 0 ? data.tracks.map((track, trackIndex) => (
+        {viewModel.tracks.length > 0 ? viewModel.tracks.map((track, trackIndex) => (
           <section id={`editorial-track-${track.id}`} key={track.id}>
             <header>
               <span>{twoDigits(trackIndex)}</span>
@@ -1320,7 +1321,7 @@ function InterviewMapRoute({ content, contentDebug }: EditorialRouteProps) {
                   </div>
                   <div className={styles.answerEvidence}>
                     {item.answers.length > 0 ? item.answers.map((answer) => {
-                      const answerProject = projectById.get(answer.projectId);
+                      const answerProject = answer.project;
 
                       if (!answerProject) {
                         return (


## `refactor(content): 홈 view model 공개 필드 제한`

diff --git a/src/lib/portfolio/view-models.ts b/src/lib/portfolio/view-models.ts
index 52c314c..8563047 100644
--- a/src/lib/portfolio/view-models.ts
+++ b/src/lib/portfolio/view-models.ts
@@ -24,6 +24,27 @@ type RouteViewModelBase = PortfolioContent & {
   footerLinks: ContentLink[];
 };
 
+type ScopedRouteViewModelBase = {
+  footerLinks: ContentLink[];
+  presentation: PortfolioContent["presentation"];
+  profile: PortfolioContent["profile"];
+  site: PortfolioContent["site"];
+};
+
+type ScopedSharedContentKey = "presentation" | "profile" | "site";
+
+type ScopedRouteViewModel<
+  VisibleContentKey extends keyof PortfolioContent,
+  RouteFields extends object,
+> = ScopedRouteViewModelBase &
+  Pick<PortfolioContent, VisibleContentKey> &
+  RouteFields & {
+    readonly [Key in Exclude<
+      keyof PortfolioContent,
+      ScopedSharedContentKey | VisibleContentKey
+    >]: never;
+  };
+
 export type ProjectGroupViewModel = ProjectGroup & {
   projects: PortfolioProject[];
 };
@@ -36,19 +57,22 @@ export type CurationCategoryViewModel = CurationCategory & {
   projects: PortfolioProject[];
 };
 
-export type HomeViewModel = RouteViewModelBase & {
-  route: "home";
-  currentYear: number;
-  featuredProjects: PortfolioProject[];
-  featuredOrAllProjects: PortfolioProject[];
-  heroLinks: ContentLink[];
-  leadProject: PortfolioProject | null;
-  metricValues: Record<string, number>;
-  metrics: ProjectMetricViewModel[];
-  preferredContactLinks: ContentLink[];
-  projectCount: number;
-  recentJourney: PortfolioContent["journey"];
-};
+export type HomeViewModel = ScopedRouteViewModel<
+  "contact" | "journey" | "journeyNarrative" | "skills" | "techStack",
+  {
+    route: "home";
+    currentYear: number;
+    featuredProjects: PortfolioProject[];
+    featuredOrAllProjects: PortfolioProject[];
+    heroLinks: ContentLink[];
+    leadProject: PortfolioProject | null;
+    metricValues: Record<string, number>;
+    metrics: ProjectMetricViewModel[];
+    preferredContactLinks: ContentLink[];
+    projectCount: number;
+    recentJourney: PortfolioContent["journey"];
+  }
+>;
 
 export type ProjectIndexViewModel = RouteViewModelBase & {
   route: "projects";
@@ -199,6 +223,7 @@ export function createHomeViewModel(
 
   return {
     ...createRouteViewModelBase(content),
+    contact: content.contact,
     currentYear: now.getFullYear(),
     featuredOrAllProjects,
     featuredProjects,
@@ -208,10 +233,13 @@ export function createHomeViewModel(
     metrics,
     preferredContactLinks: getPreferredContactLinks(content),
     projectCount: content.projects.length,
-    projects: [],
+    journey: content.journey,
+    journeyNarrative: content.journeyNarrative,
     recentJourney: content.journey.slice(-4).reverse(),
     route: "home",
-  };
+    skills: content.skills,
+    techStack: content.techStack,
+  } as HomeViewModel;
 }
 
 export function createProjectIndexViewModel(


