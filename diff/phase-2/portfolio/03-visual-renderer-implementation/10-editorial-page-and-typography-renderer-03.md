## `feat(editorial): Interview Map 소개와 chapter 추가`

diff --git a/src/designs/editorial/editorial-route.tsx b/src/designs/editorial/editorial-route.tsx
index 749119b..94d8d7e 100644
--- a/src/designs/editorial/editorial-route.tsx
+++ b/src/designs/editorial/editorial-route.tsx
@@ -1265,3 +1265,45 @@ function JourneyRoute({ content, contentDebug }: EditorialRouteProps) {
     </>
   );
 }
+function InterviewMapRoute({ content, contentDebug }: EditorialRouteProps) {
+  const copy = content.presentation.pages.interviewMap;
+  const data = content.interviewMap;
+  const projectById = new Map(content.projects.map((project) => [project.id, project]));
+  const ui = content.presentation.ui;
+
+  return (
+    <>
+      <section className={styles.pageHero}>
+        <div className={styles.pageHeroNumber}>06</div>
+        <div>
+          <p className={styles.overline}>{copy.hero.eyebrow}</p>
+          <DebugNote enabled={contentDebug} prefix={ui.debugPrefix}>
+            interview-map.json
+          </DebugNote>
+          <h1>{copy.hero.title}</h1>
+        </div>
+        <div>
+          <p>{data.intro}</p>
+          <a
+            className={styles.referenceLink}
+            href={data.referenceRepo.href}
+            rel="noreferrer"
+            target="_blank"
+          >
+            {data.referenceRepo.label} <Arrow />
+          </a>
+        </div>
+      </section>
+
+      <nav aria-label={copy.tracks.title} className={styles.chapterNav}>
+        <strong className={styles.chapterNavLabel}>{copy.tracks.indexLabel}</strong>
+        {data.tracks.map((track, index) => (
+          <a href={`#editorial-track-${track.id}`} key={track.id}>
+            <span>{twoDigits(index)}</span>
+            {track.label}
+          </a>
+        ))}
+      </nav>
+    </>
+  );
+}


## `feat(editorial): route dispatcher 추가`

diff --git a/src/designs/editorial/editorial-route.tsx b/src/designs/editorial/editorial-route.tsx
index 513cd18..8802208 100644
--- a/src/designs/editorial/editorial-route.tsx
+++ b/src/designs/editorial/editorial-route.tsx
@@ -1385,3 +1385,28 @@ function InterviewMapRoute({ content, contentDebug }: EditorialRouteProps) {
     </>
   );
 }
+
+function renderRoute(props: EditorialRouteProps) {
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
+
+export function EditorialRoute(props: EditorialRouteProps) {
+  return <EditorialShell {...props}>{renderRoute(props)}</EditorialShell>;
+}


## `style(editorial): 반응형 media rule 정렬`

diff --git a/src/designs/editorial/editorial-route.module.css b/src/designs/editorial/editorial-route.module.css
index 0b0395b..4ef3793 100644
--- a/src/designs/editorial/editorial-route.module.css
+++ b/src/designs/editorial/editorial-route.module.css
@@ -2403,28 +2403,28 @@
 }
 
 @media (max-width: 900px) {
-.mastheadMain {
+  .mastheadMain {
     grid-template-columns: 1fr auto auto;
     min-height: 78px;
   }
 
-.desktopNav {
+  .desktopNav {
     display: none;
   }
 
-.switcherSlot {
+  .switcherSlot {
     min-width: 0;
     padding: 0 16px;
     border-left: 1px solid var(--hairline);
   }
 
-.mobileMenu {
+  .mobileMenu {
     position: relative;
     display: block;
     border-left: 1px solid var(--hairline);
   }
 
-.mobileMenu summary {
+  .mobileMenu summary {
     display: grid;
     min-width: 74px;
     min-height: 78px;
@@ -2437,16 +2437,16 @@
     list-style: none;
   }
 
-.mobileMenu summary::-webkit-details-marker {
+  .mobileMenu summary::-webkit-details-marker {
     display: none;
   }
 
-.mobileMenu[open] summary {
+  .mobileMenu[open] summary {
     background: var(--ink);
     color: var(--paper);
   }
 
-.mobileMenu nav {
+  .mobileMenu nav {
     position: absolute;
     top: 78px;
     right: 0;
@@ -2456,7 +2456,7 @@
     box-shadow: 10px 12px 0 rgb(23 22 20 / 16%);
   }
 
-.mobileMenu nav a {
+  .mobileMenu nav a {
     display: flex;
     min-height: 54px;
     align-items: center;
@@ -2467,43 +2467,41 @@
     font-size: 1.05rem;
   }
 
-.mobileMenu nav a:last-child {
+  .mobileMenu nav a:last-child {
     border-bottom: 0;
   }
 
-.mobileMenu nav span {
+  .mobileMenu nav span {
     color: var(--vermilion-text);
     font-family: Geist, sans-serif;
     font-size: 0.62rem;
   }
 
-.homeHero {
+  .homeHero {
     grid-template-columns: repeat(8, minmax(0, 1fr));
   }
 
-.heroIssue {
+  .heroIssue {
     grid-column: 1 / 3;
   }
 
-.heroTitleBlock {
+  .heroTitleBlock {
     grid-column: 2 / 9;
   }
 
-.heroSummary {
+  .heroSummary {
     grid-column: 2 / 7;
   }
 
-.heroByline {
+  .heroByline {
     grid-column: 2 / 6;
   }
 
-.heroAction {
+  .heroAction {
     grid-column: 6 / 9;
   }
-}
 
-@media (max-width: 900px) {
-.leadStoryGrid,
+  .leadStoryGrid,
   .editorialColumns,
   .archiveGroup,
   .profileHero,
@@ -2517,135 +2515,133 @@
     grid-template-columns: 1fr;
   }
 
-.leadStoryGrid {
+  .leadStoryGrid {
     gap: 45px;
   }
 
-.sidebarFeature {
+  .sidebarFeature {
     border-top: 1px solid var(--ink);
     border-left: 0;
   }
 
-.contactStrip {
+  .contactStrip {
     grid-template-columns: 0.55fr 1fr;
   }
 
-.contactStrip > div {
+  .contactStrip > div {
     grid-column: 1 / -1;
   }
 
-.pageHero {
+  .pageHero {
     grid-template-columns: 0.23fr 1fr;
     min-height: 500px;
   }
 
-.pageHero > p,
+  .pageHero > p,
   .pageHero > div:last-child {
     grid-column: 2;
   }
 
-.archiveGroup > header,
+  .archiveGroup > header,
   .resumeIdentity,
   .interviewTracks > section > header {
     position: static;
   }
 
-.caseHero {
+  .caseHero {
     grid-template-columns: 0.25fr 1fr;
   }
 
-.caseDescription,
+  .caseDescription,
   .caseLinks {
     grid-column: 2;
   }
 
-.caseNarrative {
+  .caseNarrative {
     grid-template-columns: 0.22fr 1fr;
   }
 
-.caseNarrative > div:last-child {
+  .caseNarrative > div:last-child {
     grid-column: 2;
   }
 
-.architectureSpread {
+  .architectureSpread {
     grid-template-columns: 0.7fr 1.3fr;
   }
 
-.architectureSpread aside {
+  .architectureSpread aside {
     grid-column: 2;
     border-top: 1px solid rgb(242 235 221 / 28%);
     border-left: 0;
     padding: 24px 0 0;
   }
 
-.profileHero {
+  .profileHero {
     padding-bottom: 0;
   }
 
-.profileHero > div,
+  .profileHero > div,
   .profileSummary {
     max-width: 46rem;
   }
 
-.profileSummary {
+  .profileSummary {
     margin-bottom: 0 !important;
   }
 
-.curationIntro {
+  .curationIntro {
     max-width: 48rem;
   }
 
-.profilePortrait {
+  .profilePortrait {
     width: min(65vw, 470px);
     justify-self: end;
   }
 
-.resumeIdentity {
+  .resumeIdentity {
     display: grid;
     grid-template-columns: 1fr 1fr;
     gap: 20px;
   }
 
-.resumeIdentity dl {
+  .resumeIdentity dl {
     margin-top: 0;
   }
 
-.contactNotes {
+  .contactNotes {
     border-top: 1px solid var(--hairline);
     border-left: 0;
     padding: 25px 0 0;
   }
-}
 
-@media (max-width: 900px) {
-.timelineSpread > div {
+  .timelineSpread > div {
     max-width: 36rem;
   }
 }
 
 @media (max-width: 640px) {
-.masthead {
+  .masthead {
     backdrop-filter: none;
   }
 
-.mastheadRule span:first-child {
+  .mastheadRule span:first-child {
     display: none;
   }
 
-.mastheadRule {
+  .mastheadRule {
     justify-content: flex-end;
   }
 
-.wordmark > small {
+  .wordmark > small {
     display: none;
   }
 
-.switcherSlot {
+  .switcherSlot {
     max-width: 132px;
     padding-inline: 8px;
   }
 
-.footerLead,
+  .footerLead,
   .pageHero,
   .caseHero,
   .caseNarrative,
@@ -2656,74 +2652,74 @@
     grid-template-columns: 1fr;
   }
 
-.footerLead {
+  .footerLead {
     gap: 42px;
   }
 
-.footerFineprint {
+  .footerFineprint {
     flex-direction: column;
     gap: 8px;
   }
 
-.homeHero {
+  .homeHero {
     display: flex;
     min-height: auto;
     flex-direction: column;
     gap: 32px;
   }
 
-.heroIssue {
+  .heroIssue {
     flex-direction: row;
     justify-content: space-between;
   }
 
-.heroTitleBlock h1 {
+  .heroTitleBlock h1 {
     font-size: clamp(3.3rem, 16vw, 5.5rem);
   }
 
-.heroSummary {
+  .heroSummary {
     font-size: 1.05rem;
   }
 
-.heroByline {
+  .heroByline {
     width: 100%;
   }
 
-.heroAction {
+  .heroAction {
     width: 100%;
   }
 
-.leadStoryGrid {
+  .leadStoryGrid {
     grid-template-columns: 1fr;
   }
 
-.leadStoryCopy h2 {
+  .leadStoryCopy h2 {
     font-size: 3.2rem;
   }
 
-.projectIndexItem {
+  .projectIndexItem {
     grid-template-columns: 35px 1fr 44px;
     gap: 13px;
     min-height: 0;
     padding: 20px 0;
   }
 
-.projectIndexSummary,
+  .projectIndexSummary,
   .projectIndexMeta {
     grid-column: 2 / 4;
   }
 
-.projectIndexSummary {
+  .projectIndexSummary {
     display: block;
     font-size: 0.82rem;
   }
 
-.indexArrow {
+  .indexArrow {
     grid-column: 3;
     grid-row: 1;
   }
 
-.principleList,
+  .principleList,
   .principlesSpread > div:last-child,
   .focusAreas,
   .curationGrid,
@@ -2734,63 +2730,61 @@
     grid-template-columns: 1fr;
   }
 
-.principlesSpread > div:last-child,
+  .principlesSpread > div:last-child,
   .timelineSpread > ol {
     border-left: 0;
   }
-}
 
-@media (max-width: 640px) {
-.principlesSpread article,
+  .principlesSpread article,
   .timelineSpread > ol li {
     border-left: 1px solid var(--ink);
   }
 
-.curationGrid article:nth-child(odd) {
+  .curationGrid article:nth-child(odd) {
     border-right: 0;
   }
 
-.contactStrip {
+  .contactStrip {
     grid-template-columns: 1fr;
   }
 
-.contactStrip > div {
+  .contactStrip > div {
     grid-column: auto;
   }
 
-.pageHero {
+  .pageHero {
     min-height: auto;
     align-items: start;
   }
 
-.pageHeroNumber {
+  .pageHeroNumber {
     font-size: 3rem;
   }
 
-.pageHero h1,
+  .pageHero h1,
   .resumeHeader h1,
   .contactHero h1 {
     font-size: clamp(3.3rem, 16vw, 5.5rem);
   }
 
-.pageHero > p,
+  .pageHero > p,
   .pageHero > div:last-child {
     grid-column: auto;
   }
 
-.archiveOverview {
+  .archiveOverview {
     grid-template-columns: 1fr 1fr;
   }
 
-.archiveOverview > div:nth-child(3) {
+  .archiveOverview > div:nth-child(3) {
     border-top: 1px solid var(--hairline);
   }
 
-.archiveOverview > div:nth-child(4) {
+  .archiveOverview > div:nth-child(4) {
     border-top: 1px solid var(--hairline);
   }
 
-.caseMetaRail {
+  .caseMetaRail {
     display: grid;
     grid-template-columns: 1fr 1fr;
     border-right: 0;
@@ -2798,50 +2792,50 @@
     padding: 0 0 20px;
   }
 
-.caseMetaRail a {
+  .caseMetaRail a {
     grid-column: 1 / -1;
     margin-bottom: 14px;
   }
 
-.caseDescription,
+  .caseDescription,
   .caseLinks {
     grid-column: auto;
   }
 
-.caseLinks {
+  .caseLinks {
     margin-top: 14px;
   }
 
-.caseNarrative > div:last-child,
+  .caseNarrative > div:last-child,
   .architectureSpread aside {
     grid-column: auto;
   }
 
-.offsetImage {
+  .offsetImage {
     margin-top: 0 !important;
   }
 
-.resultsSpread {
+  .resultsSpread {
     gap: 30px;
   }
 
-.profileHero {
+  .profileHero {
     min-height: auto;
   }
 
-.profileHero .standfirst {
+  .profileHero .standfirst {
     margin-bottom: 20px;
   }
 
-.profileFacts {
+  .profileFacts {
     margin-bottom: 38px !important;
   }
 
-.profilePortrait {
+  .profilePortrait {
     width: 80%;
   }
 
-.skillGroups article,
+  .skillGroups article,
   .experienceSpread > ol > li,
   .resumeSections > section,
   .resumeIdentity,
@@ -2854,71 +2848,69 @@
     grid-template-columns: 1fr;
   }
 
-.sectionLead {
+  .sectionLead {
     margin-left: 0 !important;
   }
-}
 
-@media (max-width: 640px) {
-.curationPanelHeader {
+  .curationPanelHeader {
     grid-template-columns: 36px 1fr;
   }
 
-.experienceSpread li > b {
+  .experienceSpread li > b {
     display: none;
   }
 
-.resumeIdentity h2 {
+  .resumeIdentity h2 {
     margin-bottom: 20px;
   }
 
-.resumeSections {
+  .resumeSections {
     gap: 65px;
   }
 
-.resumeSections > section > h2 {
+  .resumeSections > section > h2 {
     margin-bottom: 20px;
   }
 
-.contactHero {
+  .contactHero {
     min-height: 480px;
   }
 
-.contactHero > p:last-child {
+  .contactHero > p:last-child {
     margin-top: 35px;
   }
 
-.milestoneSpread > ol > li {
+  .milestoneSpread > ol > li {
     min-height: 0;
   }
 
-.milestoneDate {
+  .milestoneDate {
     justify-content: flex-start;
     gap: 25px;
   }
 
-.milestoneStory dl {
+  .milestoneStory dl {
     grid-template-columns: 1fr;
   }
 
-.timelineSpread > ol {
+  .timelineSpread > ol {
     border-top: 0;
   }
 
-.currentPosition {
+  .currentPosition {
     align-items: start;
   }
 
-.chapterNav {
+  .chapterNav {
     padding-inline: 0;
   }
 
-.interviewTracks > section > header {
+  .interviewTracks > section > header {
     padding-bottom: 30px;
     border-bottom: 1px solid var(--ink);
   }
 
-.questionTitle a {
+  .questionTitle a {
     grid-column: auto;
   }
 }


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
 
