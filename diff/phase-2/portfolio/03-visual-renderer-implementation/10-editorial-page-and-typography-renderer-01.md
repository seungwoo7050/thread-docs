# Editorial 지면·타이포그래피 렌더러

## `style(editorial): 지면과 masthead 토큰 구성`

diff --git a/src/designs/editorial/editorial-route.module.css b/src/designs/editorial/editorial-route.module.css
new file mode 100644
index 0000000..605f837
--- /dev/null
+++ b/src/designs/editorial/editorial-route.module.css
@@ -0,0 +1,103 @@
+.root {
+  --paper: #f2ebdd;
+  --paper-deep: #e6ddcc;
+  --ink: #171614;
+  --ink-soft: #5f5a52;
+  --vermilion: #d64b32;
+  --vermilion-text: #a93222;
+  --hairline: rgb(23 22 20 / 24%);
+  min-height: 100vh;
+  overflow: clip;
+  background:
+    linear-gradient(rgb(23 22 20 / 2%) 1px, transparent 1px),
+    var(--paper);
+  background-size: 100% 32px;
+  color: var(--ink);
+  font-family: Geist, "Helvetica Neue", Arial, sans-serif;
+}
+
+.root *,
+.root *::before,
+.root *::after {
+  box-sizing: border-box;
+}
+
+.root :where(a, button, summary) {
+  -webkit-tap-highlight-color: transparent;
+}
+
+.root :where(a, button, summary):focus-visible {
+  outline: 3px solid var(--vermilion);
+  outline-offset: 4px;
+}
+
+.root a {
+  color: inherit;
+  text-decoration: none;
+}
+
+.root h1,
+.root h2,
+.root h3,
+.root p {
+  margin: 0;
+}
+
+.root ul,
+.root ol,
+.root dl,
+.root figure {
+  margin: 0;
+  padding: 0;
+}
+
+.root li {
+  list-style: none;
+}
+
+.skipLink {
+  position: fixed;
+  top: 12px;
+  left: 12px;
+  z-index: 100;
+  transform: translateY(-160%);
+  border: 2px solid var(--ink);
+  background: var(--paper);
+  padding: 12px 18px;
+  font-size: 0.8rem;
+  font-weight: 750;
+  text-transform: uppercase;
+  letter-spacing: 0.08em;
+}
+
+.skipLink:focus {
+  transform: translateY(0);
+}
+
+.masthead {
+  position: relative;
+  z-index: 30;
+  border-bottom: 1px solid var(--ink);
+  background: rgb(242 235 221 / 94%);
+  backdrop-filter: blur(12px);
+}
+
+.mastheadRule {
+  display: flex;
+  justify-content: space-between;
+  border-bottom: 1px solid var(--hairline);
+  padding: 8px clamp(20px, 4vw, 64px);
+  color: var(--ink-soft);
+  font-size: 0.63rem;
+  font-weight: 700;
+  text-transform: uppercase;
+  letter-spacing: 0.16em;
+}
+
+.mastheadMain {
+  display: grid;
+  grid-template-columns: minmax(180px, 0.72fr) minmax(400px, 1.3fr) auto;
+  align-items: stretch;
+  min-height: 94px;
+  padding-inline: clamp(20px, 4vw, 64px);
+}


## `style(editorial): wordmark와 navigation 계층 구성`

diff --git a/src/designs/editorial/editorial-route.module.css b/src/designs/editorial/editorial-route.module.css
index 605f837..f2b8b91 100644
--- a/src/designs/editorial/editorial-route.module.css
+++ b/src/designs/editorial/editorial-route.module.css
@@ -101,3 +101,113 @@
   min-height: 94px;
   padding-inline: clamp(20px, 4vw, 64px);
 }
+
+.wordmark {
+  display: flex;
+  flex-direction: column;
+  justify-content: center;
+  min-height: 48px;
+  padding-right: 28px;
+}
+
+.wordmark > span {
+  font-family: "Noto Serif KR", "Iowan Old Style", Baskerville, Georgia, serif;
+  font-size: clamp(1.45rem, 2vw, 2rem);
+  font-weight: 700;
+  letter-spacing: -0.045em;
+}
+
+.wordmark > small {
+  margin-top: 4px;
+  color: var(--ink-soft);
+  font-size: 0.62rem;
+  font-weight: 650;
+  text-transform: uppercase;
+  letter-spacing: 0.1em;
+}
+
+.desktopNav {
+  display: flex;
+  align-items: stretch;
+  border-inline: 1px solid var(--hairline);
+}
+
+.navLink {
+  display: flex;
+  flex: 1;
+  min-width: 92px;
+  min-height: 48px;
+  align-items: center;
+  justify-content: center;
+  gap: 8px;
+  padding: 10px 14px;
+  font-size: 0.74rem;
+  font-weight: 750;
+  transition: background-color 180ms ease, color 180ms ease;
+}
+
+.navLink + .navLink {
+  border-left: 1px solid var(--hairline);
+}
+
+.navLink span {
+  color: var(--vermilion-text);
+  font-size: 0.55rem;
+  vertical-align: super;
+}
+
+.navLink:hover {
+  background: var(--ink);
+  color: var(--paper);
+}
+
+.switcherSlot {
+  display: flex;
+  min-width: 164px;
+  align-items: center;
+  justify-content: flex-end;
+  padding-left: 24px;
+}
+
+.mobileMenu {
+  display: none;
+}
+
+.footer {
+  border-top: 1px solid var(--ink);
+  padding: clamp(48px, 7vw, 100px) clamp(20px, 5vw, 80px) 24px;
+}
+
+.footerLead {
+  display: grid;
+  grid-template-columns: 1fr 1.3fr;
+  align-items: end;
+  gap: 40px;
+  padding-bottom: 70px;
+}
+
+.footerLead p {
+  max-width: 34rem;
+  color: var(--ink-soft);
+  font-family: "Noto Serif KR", "Iowan Old Style", Georgia, serif;
+  font-size: clamp(1.15rem, 2vw, 1.75rem);
+  line-height: 1.45;
+}
+
+.footerLead a {
+  display: flex;
+  align-items: flex-end;
+  justify-content: space-between;
+  border-bottom: 3px solid var(--vermilion);
+  padding: 0 0 12px;
+  font-family: "Noto Serif KR", "Iowan Old Style", Georgia, serif;
+  font-size: clamp(2rem, 5vw, 5rem);
+  line-height: 0.96;
+  letter-spacing: -0.055em;
+}
+
+.footerLead a span {
+  color: var(--vermilion-text);
+  font-family: Geist, sans-serif;
+  font-size: 0.6em;
+}


## `style(editorial): footer와 hero 활자 체계 구성`

diff --git a/src/designs/editorial/editorial-route.module.css b/src/designs/editorial/editorial-route.module.css
index f2b8b91..45c231c 100644
--- a/src/designs/editorial/editorial-route.module.css
+++ b/src/designs/editorial/editorial-route.module.css
@@ -211,3 +211,110 @@
   font-family: Geist, sans-serif;
   font-size: 0.6em;
 }
+
+.footerFineprint {
+  display: flex;
+  justify-content: space-between;
+  border-top: 1px solid var(--hairline);
+  padding-top: 18px;
+  color: var(--ink-soft);
+  font-size: 0.68rem;
+  text-transform: uppercase;
+  letter-spacing: 0.12em;
+}
+
+.debugNote {
+  display: inline-flex;
+  margin-bottom: 12px;
+  border: 1px dashed var(--vermilion);
+  padding: 5px 8px;
+  color: var(--vermilion-text);
+  font-family: ui-monospace, SFMono-Regular, Menlo, monospace;
+  font-size: 0.62rem;
+  line-height: 1.3;
+}
+
+.sectionKicker {
+  display: grid;
+  grid-template-columns: 52px 1fr;
+  align-items: center;
+  gap: 20px;
+  border-top: 1px solid var(--ink);
+  padding-top: 12px;
+}
+
+.sectionKicker span {
+  color: var(--vermilion-text);
+  font-family: "Noto Serif KR", Georgia, serif;
+  font-size: 0.78rem;
+  font-style: italic;
+}
+
+.sectionKicker p {
+  font-size: 0.68rem;
+  font-weight: 800;
+  text-transform: uppercase;
+  letter-spacing: 0.16em;
+}
+
+.overline,
+.sidebarLabel {
+  color: var(--vermilion-text);
+  font-size: 0.67rem;
+  font-weight: 800;
+  text-transform: uppercase;
+  letter-spacing: 0.15em;
+}
+
+.standfirst {
+  font-family: "Noto Serif KR", "Iowan Old Style", Georgia, serif;
+  font-size: clamp(1.2rem, 2.1vw, 1.85rem);
+  line-height: 1.48;
+  letter-spacing: -0.025em;
+}
+
+.homeHero {
+  display: grid;
+  grid-template-columns: repeat(12, minmax(0, 1fr));
+  grid-template-rows: auto auto auto;
+  gap: clamp(24px, 4vw, 64px) 20px;
+  min-height: min(850px, calc(100vh - 126px));
+  padding: clamp(38px, 7vw, 110px) clamp(20px, 5vw, 80px) clamp(56px, 8vw, 120px);
+  border-bottom: 1px solid var(--ink);
+}
+
+.heroIssue {
+  grid-column: 1 / 3;
+  display: flex;
+  flex-direction: column;
+  gap: 6px;
+  align-self: start;
+  color: var(--ink-soft);
+  font-size: 0.65rem;
+  font-weight: 700;
+  text-transform: uppercase;
+  letter-spacing: 0.13em;
+}
+
+.heroTitleBlock {
+  grid-column: 3 / 12;
+}
+
+.heroRole {
+  margin-bottom: 16px !important;
+  color: var(--vermilion-text);
+  font-size: 0.72rem;
+  font-weight: 800;
+  text-transform: uppercase;
+  letter-spacing: 0.16em;
+}
+
+.heroTitleBlock h1 {
+  max-width: 12ch;
+  font-family: "Noto Serif KR", "Iowan Old Style", Baskerville, Georgia, serif;
+  font-size: clamp(3.8rem, 8.5vw, 9.5rem);
+  font-weight: 500;
+  line-height: 0.94;
+  letter-spacing: -0.072em;
+  text-wrap: balance;
+}


## `style(editorial): hero spread 레이아웃 구성`

diff --git a/src/designs/editorial/editorial-route.module.css b/src/designs/editorial/editorial-route.module.css
index 45c231c..1a1fa31 100644
--- a/src/designs/editorial/editorial-route.module.css
+++ b/src/designs/editorial/editorial-route.module.css
@@ -318,3 +318,110 @@
   letter-spacing: -0.072em;
   text-wrap: balance;
 }
+
+.heroTitleBlock h1::first-letter {
+  color: var(--vermilion-text);
+}
+
+.heroSummary {
+  grid-column: 3 / 8;
+  max-width: 42rem;
+  align-self: end;
+  font-family: "Noto Serif KR", "Iowan Old Style", Georgia, serif;
+  font-size: clamp(1.1rem, 1.7vw, 1.45rem);
+  line-height: 1.65;
+}
+
+.heroByline {
+  grid-column: 9 / 12;
+  display: flex;
+  align-items: center;
+  gap: 16px;
+  align-self: end;
+  border-top: 1px solid var(--hairline);
+  padding-top: 15px;
+}
+
+.heroByline > div {
+  display: flex;
+  min-width: 0;
+  flex-direction: column;
+  gap: 5px;
+}
+
+.heroByline strong {
+  font-family: "Noto Serif KR", Georgia, serif;
+  font-size: 1rem;
+}
+
+.heroByline span:not(.portraitFallback) {
+  overflow: hidden;
+  color: var(--ink-soft);
+  font-size: 0.67rem;
+  text-overflow: ellipsis;
+  white-space: nowrap;
+}
+
+.portrait,
+.portraitFallback {
+  width: 58px;
+  height: 58px;
+  flex: 0 0 auto;
+  border: 1px solid var(--ink);
+  border-radius: 50%;
+  object-fit: cover;
+  filter: grayscale(0.85) contrast(1.08);
+}
+
+.portraitFallback {
+  display: grid;
+  place-items: center;
+  background: var(--vermilion);
+  color: var(--paper);
+  font-family: "Noto Serif KR", Georgia, serif;
+  font-size: 1.6rem;
+}
+
+.heroAction {
+  grid-column: 9 / 13;
+  display: flex;
+  min-height: 52px;
+  align-items: center;
+  justify-content: space-between;
+  align-self: end;
+  border-bottom: 2px solid var(--ink);
+  padding: 10px 0;
+  font-size: 0.76rem;
+  font-weight: 800;
+  transition: border-color 180ms ease, color 180ms ease;
+}
+
+.heroAction span {
+  color: var(--vermilion-text);
+  font-size: 1.3rem;
+}
+
+.heroAction:hover {
+  border-color: var(--vermilion);
+  color: var(--vermilion-text);
+}
+
+.leadStory,
+.selectedStories,
+.principlesSpread,
+.experienceSpread,
+.evidenceGallery,
+.decisionSpread,
+.highlightsSpread,
+.milestoneSpread {
+  padding: clamp(56px, 8vw, 120px) clamp(20px, 5vw, 80px);
+  border-bottom: 1px solid var(--ink);
+}
+
+.leadStoryGrid {
+  display: grid;
+  grid-template-columns: minmax(250px, 0.75fr) minmax(450px, 1.25fr);
+  gap: clamp(30px, 5vw, 80px);
+  margin-top: clamp(36px, 5vw, 72px);
+  align-items: start;
+}


## `style(editorial): lead story와 매체 표현 구성`

diff --git a/src/designs/editorial/editorial-route.module.css b/src/designs/editorial/editorial-route.module.css
index 1a1fa31..9acc881 100644
--- a/src/designs/editorial/editorial-route.module.css
+++ b/src/designs/editorial/editorial-route.module.css
@@ -425,3 +425,106 @@
   margin-top: clamp(36px, 5vw, 72px);
   align-items: start;
 }
+
+.leadStoryCopy {
+  display: flex;
+  flex-direction: column;
+  align-items: flex-start;
+}
+
+.leadStoryCopy h2 {
+  margin-top: 16px;
+  font-family: "Noto Serif KR", "Iowan Old Style", Georgia, serif;
+  font-size: clamp(2.8rem, 5.6vw, 6.7rem);
+  font-weight: 500;
+  line-height: 0.95;
+  letter-spacing: -0.06em;
+}
+
+.leadStoryCopy h2 a {
+  background-image: linear-gradient(var(--vermilion), var(--vermilion));
+  background-position: 0 100%;
+  background-repeat: no-repeat;
+  background-size: 0 4px;
+  transition: background-size 300ms ease;
+}
+
+.leadStoryCopy h2 a:hover {
+  background-size: 100% 4px;
+}
+
+.leadStoryCopy .standfirst {
+  margin-top: 30px;
+}
+
+.leadStoryCopy > p:not(.overline, .standfirst) {
+  margin-top: 20px;
+  color: var(--ink-soft);
+  font-size: 0.94rem;
+  line-height: 1.75;
+}
+
+.inlineFacts {
+  display: grid;
+  gap: 0;
+  width: 100%;
+  margin-top: 30px !important;
+  border-top: 1px solid var(--hairline);
+}
+
+.inlineFacts li {
+  border-bottom: 1px solid var(--hairline);
+  padding: 12px 0;
+  color: var(--ink-soft);
+  font-size: 0.75rem;
+  line-height: 1.45;
+}
+
+.leadVisualLink {
+  display: block;
+}
+
+.leadVisualLink > span {
+  display: flex;
+  align-items: center;
+  justify-content: space-between;
+  margin-top: 12px;
+  border-top: 1px solid var(--ink);
+  padding-top: 10px;
+  font-size: 0.72rem;
+  font-weight: 800;
+  text-transform: uppercase;
+  letter-spacing: 0.08em;
+}
+
+.imageFrame {
+  position: relative;
+  overflow: hidden;
+  border: 1px solid var(--ink);
+  background: var(--paper-deep);
+}
+
+.image {
+  display: block;
+  width: 100%;
+  height: auto;
+  min-height: 260px;
+  max-height: 760px;
+  object-fit: cover;
+  filter: saturate(0.76) contrast(1.03);
+  transition: transform 700ms cubic-bezier(0.2, 0.8, 0.2, 1), filter 400ms ease;
+}
+
+.imageFrame:hover .image,
+.leadVisualLink:hover .image {
+  transform: scale(1.018);
+  filter: saturate(1) contrast(1.01);
+}
+
+.imageFrame figcaption {
+  border-top: 1px solid var(--ink);
+  padding: 8px 11px;
+  color: var(--ink-soft);
+  font-size: 0.62rem;
+  line-height: 1.4;
+}


## `style(editorial): 원칙 목록과 contact strip 구성`

diff --git a/src/designs/editorial/editorial-route.module.css b/src/designs/editorial/editorial-route.module.css
index 5214df8..5b8e7b1 100644
--- a/src/designs/editorial/editorial-route.module.css
+++ b/src/designs/editorial/editorial-route.module.css
@@ -636,3 +636,112 @@
 .columnFeature {
   padding: clamp(56px, 8vw, 120px) clamp(20px, 5vw, 80px);
 }
+
+.principleList {
+  display: grid;
+  grid-template-columns: repeat(2, minmax(0, 1fr));
+  gap: 0 42px;
+  margin-top: 52px;
+}
+
+.principleList article {
+  min-height: 210px;
+  border-top: 1px solid var(--hairline);
+  padding: 18px 0 30px;
+}
+
+.principleList article span,
+.principlesSpread article > span,
+.resumeSections > section > span,
+.skillsIntro > span {
+  color: var(--vermilion-text);
+  font-family: "Noto Serif KR", Georgia, serif;
+  font-size: 0.72rem;
+  font-style: italic;
+}
+
+.principleList h3 {
+  margin-top: 22px;
+  font-family: "Noto Serif KR", Georgia, serif;
+  font-size: 1.45rem;
+  font-weight: 600;
+}
+
+.principleList p {
+  margin-top: 12px;
+  color: var(--ink-soft);
+  font-size: 0.83rem;
+  line-height: 1.7;
+}
+
+.sidebarFeature {
+  padding: clamp(56px, 8vw, 120px) clamp(24px, 4vw, 64px);
+  border-left: 1px solid var(--ink);
+  background: var(--paper-deep);
+}
+
+.sidebarFeature h2 {
+  margin-top: 20px;
+  font-family: "Noto Serif KR", Georgia, serif;
+  font-size: clamp(2rem, 3.2vw, 3.8rem);
+  font-weight: 500;
+  line-height: 1.02;
+  letter-spacing: -0.05em;
+}
+
+.sidebarFeature > p:not(.sidebarLabel) {
+  margin-top: 20px;
+  color: var(--ink-soft);
+  font-family: "Noto Serif KR", Georgia, serif;
+  font-size: 0.95rem;
+  line-height: 1.7;
+}
+
+.sidebarFeature > a {
+  display: flex;
+  min-height: 44px;
+  align-items: center;
+  justify-content: space-between;
+  margin-top: 30px;
+  border-bottom: 1px solid var(--ink);
+  font-size: 0.72rem;
+  font-weight: 800;
+  text-transform: uppercase;
+  letter-spacing: 0.08em;
+}
+
+.sidebarRule {
+  height: 1px;
+  margin: 58px 0 30px;
+  background: var(--ink);
+}
+
+.textTags {
+  display: flex;
+  flex-wrap: wrap;
+  gap: 8px;
+  margin-top: 18px !important;
+}
+
+.textTags li {
+  border: 1px solid var(--hairline);
+  padding: 7px 9px;
+  font-size: 0.64rem;
+}
+
+.contactStrip {
+  display: grid;
+  grid-template-columns: 0.7fr 1.15fr 1fr;
+  gap: 30px;
+  align-items: center;
+  padding: clamp(50px, 7vw, 90px) clamp(20px, 5vw, 80px);
+  border-bottom: 1px solid var(--ink);
+  background: var(--vermilion);
+  color: #fff9ec;
+}
+
+.contactStrip > p {
+  max-width: 23rem;
+  font-size: 0.75rem;
+  line-height: 1.6;
+}


## `feat(editorial): route 계약과 navigation helper 추가`

diff --git a/src/designs/editorial/editorial-route.tsx b/src/designs/editorial/editorial-route.tsx
new file mode 100644
index 0000000..d123e95
--- /dev/null
+++ b/src/designs/editorial/editorial-route.tsx
@@ -0,0 +1,69 @@
+import Image from "next/image";
+import Link from "next/link";
+import type { ReactNode } from "react";
+
+import { DesignSwitcher } from "@/components/portfolio/design-switcher";
+import {
+  getPreferredContactLinks,
+  getProjectDetailLinks,
+  getProjectMetricValue,
+  getResumeProjects,
+  getTemplateHref,
+  isSitePageEnabled,
+  type ContentLink,
+  type PortfolioContent,
+  type PortfolioProject,
+  type ProjectImage,
+} from "@/lib/portfolio";
+
+import styles from "./editorial-route.module.css";
+
+export type EditorialRouteName =
+  | "home"
+  | "projects"
+  | "project-detail"
+  | "about"
+  | "resume"
+  | "contact"
+  | "journey"
+  | "interview-map";
+
+export type EditorialRouteProps = {
+  route: EditorialRouteName;
+  content: PortfolioContent;
+  project?: PortfolioProject;
+  currentPath: string;
+  contentDebug: boolean;
+};
+
+const DESIGN_ID = "editorial" as const;
+
+const routeNumbers: Record<EditorialRouteName, string> = {
+  home: "00",
+  projects: "01",
+  "project-detail": "01",
+  about: "02",
+  resume: "03",
+  contact: "04",
+  journey: "05",
+  "interview-map": "06",
+};
+
+function editorialHref(path: string, contentDebug: boolean) {
+  return getTemplateHref(path, DESIGN_ID, {
+    contentDebug,
+  });
+}
+
+function isCurrentNavigation(href: string, currentPath: string) {
+  if (href === "/") return currentPath === "/";
+  return currentPath === href || currentPath.startsWith(`${href}/`);
+}
+
+function twoDigits(index: number) {
+  return String(index + 1).padStart(2, "0");
+}
+
+function getProjectTags(project: PortfolioProject) {
+  return project.tags.slice(0, 4);
+}


## `feat(editorial): masthead와 footer shell 추가`

diff --git a/src/designs/editorial/editorial-route.tsx b/src/designs/editorial/editorial-route.tsx
index ef191d4..3f733fb 100644
--- a/src/designs/editorial/editorial-route.tsx
+++ b/src/designs/editorial/editorial-route.tsx
@@ -150,3 +150,105 @@ function EditorialContentLink({
 function Arrow() {
   return <span aria-hidden="true">↗</span>;
 }
+
+function EditorialShell({
+  children,
+  content,
+  contentDebug,
+  currentPath,
+  route,
+}: EditorialRouteProps & { children: ReactNode }) {
+  const primaryNavigation = content.site.navigation;
+  const ui = content.presentation.ui;
+  const shellCopy = content.presentation.editorial.shell;
+  const footerLinks = content.links.filter((link) =>
+    link.placements?.includes("footer"),
+  );
+
+  return (
+    <div className={styles.root} data-site-design="editorial">
+      <a className={styles.skipLink} href="#editorial-main">
+        {ui.skipLinkLabel}
+      </a>
+      <header className={styles.masthead}>
+        <div className={styles.mastheadRule}>
+          <span>{shellCopy.kicker}</span>
+          <span aria-hidden="true">{shellCopy.volumeLabel} {routeNumbers[route]}</span>
+        </div>
+        <div className={styles.mastheadMain}>
+          <Link
+            className={styles.wordmark}
+            href={editorialHref("/", contentDebug)}
+          >
+            <span>{content.profile.name}</span>
+            <small>{content.profile.role}</small>
+          </Link>
+          <nav aria-label={ui.primaryNavigationAriaLabel} className={styles.desktopNav}>
+            {primaryNavigation.map((item, index) => (
+              <Link
+                aria-current={
+                  isCurrentNavigation(item.href, currentPath) ? "page" : undefined
+                }
+                className={styles.navLink}
+                href={editorialHref(item.href, contentDebug)}
+                key={`${item.href}-${index}`}
+              >
+                <span>{twoDigits(index)}</span>
+                {item.label}
+              </Link>
+            ))}
+          </nav>
+          <div className={styles.switcherSlot}>
+            <DesignSwitcher
+              activeId={DESIGN_ID}
+              contentDebug={contentDebug}
+              currentPath={currentPath}
+              templates={content.presentation.templates}
+              ui={ui}
+            />
+          </div>
+          <details className={styles.mobileMenu}>
+            <summary>{ui.menuLabel}</summary>
+            <nav aria-label={ui.mobileNavigationAriaLabel}>
+              {primaryNavigation.map((item, index) => (
+                <Link
+                  aria-current={
+                    isCurrentNavigation(item.href, currentPath)
+                      ? "page"
+                      : undefined
+                  }
+                  href={editorialHref(item.href, contentDebug)}
+                  key={`${item.href}-mobile-${index}`}
+                >
+                  <span>{twoDigits(index)}</span>
+                  {item.label}
+                </Link>
+              ))}
+            </nav>
+          </details>
+        </div>
+      </header>
+      <main id="editorial-main">{children}</main>
+      <footer className={styles.footer}>
+        <div className={styles.footerLead}>
+          <p>{content.site.footer.note}</p>
+          {footerLinks.map((link) => (
+            <EditorialContentLink
+              contentDebug={contentDebug}
+              key={link.id ?? link.href}
+              link={link}
+            >
+              {link.label} <Arrow />
+            </EditorialContentLink>
+          ))}
+        </div>
+        <div className={styles.footerFineprint}>
+          <span>{content.site.footer.copyright}</span>
+          <span>
+            {content.profile.location} · {content.profile.handle}
+          </span>
+        </div>
+      </footer>
+    </div>
+  );
+}


