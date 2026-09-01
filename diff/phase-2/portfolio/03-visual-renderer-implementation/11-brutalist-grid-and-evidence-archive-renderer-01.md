# Brutalist 그리드·근거 아카이브 렌더러

## `style(brutalist): 화면 토큰과 brand mark 구성`

diff --git a/src/designs/brutalist/brutalist.module.css b/src/designs/brutalist/brutalist.module.css
new file mode 100644
index 0000000..9c0b07d
--- /dev/null
+++ b/src/designs/brutalist/brutalist.module.css
@@ -0,0 +1,97 @@
+.root {
+  --paper: #f4f0e8;
+  --ink: #101010;
+  --blue: #2e5bff;
+  --yellow: #e6ff3f;
+  --white: #fffdf6;
+  background: var(--paper);
+  color: var(--ink);
+  font-family: var(--font-geist-sans), "Apple SD Gothic Neo", "Noto Sans KR",
+    sans-serif;
+  min-height: 100vh;
+  overflow: clip;
+  width: 100%;
+}
+
+.root,
+.root * {
+  box-sizing: border-box;
+}
+
+.root :where(a, button, summary) {
+  -webkit-tap-highlight-color: transparent;
+}
+
+.root :where(a, button, summary):focus-visible {
+  outline: 4px solid var(--yellow);
+  outline-offset: 4px;
+}
+
+.root ::selection {
+  background: var(--yellow);
+  color: var(--ink);
+}
+
+.skipLink {
+  background: var(--yellow);
+  border: 3px solid var(--ink);
+  color: var(--ink);
+  font-family: var(--font-geist-mono), monospace;
+  font-size: 0.75rem;
+  font-weight: 800;
+  left: 1rem;
+  padding: 0.8rem 1rem;
+  position: fixed;
+  text-decoration: none;
+  top: 1rem;
+  transform: translateY(-180%);
+  z-index: 200;
+}
+
+.skipLink:focus {
+  transform: translateY(0);
+}
+
+.header {
+  background: var(--paper);
+  border-bottom: 3px solid var(--ink);
+  position: relative;
+  z-index: 30;
+}
+
+.headerBar {
+  align-items: stretch;
+  display: grid;
+  grid-template-columns: minmax(11rem, 0.7fr) minmax(16rem, 1fr) auto;
+  min-height: 4.8rem;
+}
+
+.brand,
+.headerStatus,
+.switcher {
+  align-items: center;
+  display: flex;
+  min-width: 0;
+  padding: 0.85rem 1.35rem;
+}
+
+.brand {
+  color: var(--ink);
+  font-size: clamp(1rem, 2vw, 1.35rem);
+  font-weight: 900;
+  gap: 0.75rem;
+  letter-spacing: -0.04em;
+  text-decoration: none;
+  text-transform: uppercase;
+}
+
+.brand:hover .brandMark {
+  color: var(--blue);
+  transform: rotate(45deg);
+}
+
+.brandMark {
+  color: var(--ink);
+  font-size: 1rem;
+  transition: color 140ms linear, transform 140ms linear;
+}


## `style(brutalist): header 상태와 home hero 구성`

diff --git a/src/designs/brutalist/brutalist.module.css b/src/designs/brutalist/brutalist.module.css
index 9c0b07d..dc1d0bb 100644
--- a/src/designs/brutalist/brutalist.module.css
+++ b/src/designs/brutalist/brutalist.module.css
@@ -95,3 +95,112 @@
   font-size: 1rem;
   transition: color 140ms linear, transform 140ms linear;
 }
+
+.headerStatus {
+  border-left: 3px solid var(--ink);
+  border-right: 3px solid var(--ink);
+  font-family: var(--font-geist-mono), monospace;
+  font-size: 0.66rem;
+  font-weight: 700;
+  gap: 0.7rem;
+  justify-content: space-between;
+  letter-spacing: 0.07em;
+  text-transform: uppercase;
+}
+
+.switcher {
+  justify-content: flex-end;
+  padding-block: 0;
+}
+
+.navigation {
+  border-top: 3px solid var(--ink);
+  overflow-x: auto;
+  scrollbar-width: thin;
+}
+
+.navigationList {
+  display: grid;
+  grid-auto-columns: minmax(8.5rem, 1fr);
+  grid-auto-flow: column;
+  list-style: none;
+  margin: 0;
+  min-width: max-content;
+  padding: 0;
+  width: 100%;
+}
+
+.navigationList li {
+  border-right: 3px solid var(--ink);
+}
+
+.navigationList li:last-child {
+  border-right: 0;
+}
+
+.navigationLink {
+  align-items: center;
+  color: var(--ink);
+  display: flex;
+  font-family: var(--font-geist-mono), monospace;
+  font-size: 0.72rem;
+  font-weight: 750;
+  gap: 0.7rem;
+  min-height: 3.35rem;
+  padding: 0.75rem 1rem;
+  text-decoration: none;
+  text-transform: uppercase;
+  transition: background 100ms linear, color 100ms linear;
+}
+
+.navigationLink:hover,
+.navigationLink[aria-current="page"] {
+  background: var(--ink);
+  color: var(--yellow);
+}
+
+.navigationIndex {
+  color: var(--blue);
+  font-size: 0.6rem;
+  letter-spacing: 0.04em;
+}
+
+.navigationLink:hover .navigationIndex,
+.navigationLink[aria-current="page"] .navigationIndex {
+  color: var(--yellow);
+}
+
+.mobileMenu {
+  display: none;
+}
+
+.debugBanner {
+  align-items: center;
+  background: var(--yellow);
+  border-bottom: 3px solid var(--ink);
+  display: flex;
+  font-family: var(--font-geist-mono), monospace;
+  font-size: 0.7rem;
+  gap: 1rem;
+  justify-content: space-between;
+  letter-spacing: 0.04em;
+  padding: 0.75rem 1.35rem;
+  text-transform: uppercase;
+}
+
+.debugBanner strong {
+  background: var(--ink);
+  color: var(--yellow);
+  padding: 0.25rem 0.45rem;
+}
+
+.main {
+  display: block;
+}
+
+.homeHero {
+  display: grid;
+  grid-template-columns: minmax(0, 1.35fr) minmax(18rem, 0.65fr);
+  grid-template-rows: auto auto auto;
+  min-height: min(53rem, calc(100svh - 8rem));
+}


## `style(brutalist): section header와 프로젝트 지표 구성`

diff --git a/src/designs/brutalist/brutalist.module.css b/src/designs/brutalist/brutalist.module.css
index 66820a0..f47429d 100644
--- a/src/designs/brutalist/brutalist.module.css
+++ b/src/designs/brutalist/brutalist.module.css
@@ -409,3 +409,109 @@
   border-bottom: 3px solid var(--ink);
   padding: clamp(3.5rem, 7vw, 7rem) clamp(1.2rem, 4vw, 4rem);
 }
+
+.sectionHeader {
+  align-items: start;
+  border: 3px solid var(--ink);
+  display: grid;
+  grid-template-columns: 5rem minmax(0, 1.1fr) minmax(16rem, 0.9fr);
+  margin-bottom: clamp(2.5rem, 5vw, 5rem);
+}
+
+.sectionHeader > * {
+  margin: 0;
+  min-height: 6.5rem;
+  padding: 1.2rem;
+}
+
+.sectionHeader > * + * {
+  border-left: 3px solid var(--ink);
+}
+
+.sectionNumber {
+  align-items: flex-start;
+  background: var(--ink);
+  color: var(--yellow);
+  display: flex;
+  font-family: var(--font-geist-mono), monospace;
+  font-size: 0.8rem;
+  font-weight: 800;
+}
+
+.sectionHeader h2 {
+  font-size: clamp(2.1rem, 4.2vw, 5rem);
+  font-weight: 950;
+  letter-spacing: -0.075em;
+  line-height: 0.9;
+  overflow-wrap: anywhere;
+  text-transform: uppercase;
+}
+
+.sectionHeader p {
+  font-size: 0.88rem;
+  line-height: 1.55;
+}
+
+.projectIndex,
+.groupProjectList {
+  display: grid;
+  gap: 1rem;
+  list-style: none;
+  margin: 0;
+  padding: 0;
+}
+
+.projectIndexItem {
+  min-width: 0;
+}
+
+.projectIndexItem:nth-child(even) {
+  margin-left: clamp(0rem, 4vw, 4rem);
+}
+
+.projectIndexItem > a {
+  align-items: stretch;
+  background: var(--white);
+  border: 3px solid var(--ink);
+  color: var(--ink);
+  display: grid;
+  grid-template-columns: 5rem minmax(0, 1.6fr) minmax(11rem, 0.75fr) 4rem;
+  min-height: 10rem;
+  text-decoration: none;
+  transition: background 100ms linear, box-shadow 120ms linear,
+    transform 120ms linear;
+}
+
+.projectIndexItem > a > * {
+  min-width: 0;
+  padding: 1.15rem;
+}
+
+.projectIndexItem > a > * + * {
+  border-left: 3px solid var(--ink);
+}
+
+.projectIndexItem > a:hover,
+.projectIndexItem > a:focus-visible {
+  background: var(--yellow);
+  box-shadow: 8px 8px 0 var(--ink);
+  transform: translate(-4px, -4px);
+}
+
+.projectIndexNumber {
+  font-family: var(--font-geist-mono), monospace;
+  font-size: 0.72rem;
+  font-weight: 800;
+}
+
+.projectIndexMain {
+  display: flex;
+  flex-direction: column;
+  justify-content: space-between;
+}
+
+.projectIndexMeta,
+.projectIndexSummary {
+  font-size: 0.7rem;
+  line-height: 1.45;
+}


## `feat(brutalist): 콘텐츠와 탐색 조회 도우미 추가`

diff --git a/src/designs/brutalist/brutalist-route.tsx b/src/designs/brutalist/brutalist-route.tsx
new file mode 100644
index 0000000..e090c71
--- /dev/null
+++ b/src/designs/brutalist/brutalist-route.tsx
@@ -0,0 +1,85 @@
+import {
+  getProjectMetricValue,
+  getTemplateHref,
+  type PortfolioContent,
+  type PortfolioProject,
+} from "@/lib/portfolio";
+
+const DESIGN_ID = "brutalist" as const;
+
+type GroupedProjects = {
+  description: string;
+  id: string;
+  label: string;
+  projects: PortfolioProject[];
+};
+
+type CopyTemplateToken =
+  | "count"
+  | "handle"
+  | "name"
+  | "number"
+  | "title"
+  | "year";
+
+export function brutalistHref(path: string, contentDebug: boolean) {
+  return getTemplateHref(path, DESIGN_ID, {
+    contentDebug,
+  });
+}
+
+export function renderCopyTemplate(
+  template: string,
+  values: Partial<Record<CopyTemplateToken, string>>,
+) {
+  return Object.entries(values).reduce(
+    (copy, [token, value]) => copy.replaceAll(`{${token}}`, value),
+    template,
+  );
+}
+
+export function getProjectTags(project: PortfolioProject, limit = 4) {
+  const tags = project.tags;
+  const source = tags && tags.length > 0 ? tags : project.stack;
+
+  return source.filter(Boolean).slice(0, limit);
+}
+
+export function groupProjects(content: PortfolioContent): GroupedProjects[] {
+  return content.projectGroups
+    .map((group) => {
+      const projects = content.projects.filter((project) => {
+        return project.groupId === group.id;
+      });
+
+      return { ...group, projects };
+    })
+    .filter((group) => group.projects.length > 0);
+}
+
+export function getHomeMetrics(content: PortfolioContent) {
+  return content.projectMetrics.slice(0, 4).map((metric) => ({
+    description: metric.description,
+    id: metric.id,
+    label: metric.label,
+    value: getProjectMetricValue(metric.id, content),
+  }));
+}
+
+export function isCurrentNavigation(href: string, currentPath: string) {
+  if (href === "/") {
+    return currentPath === "/";
+  }
+
+  return currentPath === href || currentPath.startsWith(`${href}/`);
+}
+
+export function getNavigationLabel(
+  content: PortfolioContent,
+  href: string,
+  fallback: string,
+) {
+  return (
+    content.site.navigation.find((item) => item.href === href)?.label ?? fallback
+  );
+}


## `feat(brutalist): route 레이블과 기본 shell 구성`

diff --git a/src/designs/brutalist/brutalist-route.tsx b/src/designs/brutalist/brutalist-route.tsx
index e090c71..2bdcc1f 100644
--- a/src/designs/brutalist/brutalist-route.tsx
+++ b/src/designs/brutalist/brutalist-route.tsx
@@ -1,9 +1,13 @@
+import Link from "next/link";
+import { DesignSwitcher } from "@/components/portfolio/design-switcher";
 import {
   getProjectMetricValue,
   getTemplateHref,
   type PortfolioContent,
   type PortfolioProject,
 } from "@/lib/portfolio";
+import type { DesignRouteProps } from "@/designs/types";
+import styles from "./brutalist.module.css";
 
 const DESIGN_ID = "brutalist" as const;
 
@@ -83,3 +87,99 @@ export function getNavigationLabel(
     content.site.navigation.find((item) => item.href === href)?.label ?? fallback
   );
 }
+
+export function getRouteLabel(
+  content: PortfolioContent,
+  route: DesignRouteProps["route"],
+) {
+  const pages = content.presentation.pages;
+
+  switch (route) {
+    case "home":
+      return getNavigationLabel(
+        content,
+        "/",
+        content.presentation.home.brutalist.stampLabel,
+      );
+    case "projects":
+      return getNavigationLabel(
+        content,
+        "/projects",
+        pages.projects.brutalist.hero.title,
+      );
+    case "project-detail":
+      return pages.projectDetail.caseLabel;
+    case "about":
+      return getNavigationLabel(content, "/about", pages.about.hero.title);
+    case "resume":
+      return getNavigationLabel(content, "/resume", pages.resume.hero.title);
+    case "contact":
+      return getNavigationLabel(content, "/contact", content.contact.title);
+    case "journey":
+      return getNavigationLabel(content, "/journey", pages.journey.hero.title);
+    case "interview-map":
+      return getNavigationLabel(
+        content,
+        "/interview-map",
+        pages.interviewMap.hero.title,
+      );
+  }
+}
+
+
+export function BrutalistShell({
+  children,
+  content,
+  contentDebug,
+  currentPath,
+  route,
+}: {
+  children: React.ReactNode;
+  content: PortfolioContent;
+  contentDebug: boolean;
+  currentPath: string;
+  route: DesignRouteProps["route"];
+}) {
+  const ui = content.presentation.ui;
+  return (
+    <div
+      className={styles.root}
+      data-content-debug={contentDebug ? "true" : "false"}
+      data-site-design={DESIGN_ID}
+    >
+      <a className={styles.skipLink} href="#brutalist-main">
+        {ui.skipLinkLabel}
+      </a>
+      <header className={styles.header}>
+        <div className={styles.headerBar}>
+          <Link
+            className={styles.brand}
+            href={brutalistHref("/", contentDebug)}
+          >
+            <span className={styles.brandMark} aria-hidden="true">
+              ■
+            </span>
+            <span>{content.site.brand}</span>
+          </Link>
+          <div className={styles.headerStatus}>
+            <span>{getRouteLabel(content, route)}</span>
+            <span aria-hidden="true">/</span>
+            <span>{content.profile.location}</span>
+          </div>
+          <div className={styles.switcher}>
+            <DesignSwitcher
+              activeId={DESIGN_ID}
+              contentDebug={contentDebug}
+              currentPath={currentPath}
+              templates={content.presentation.templates}
+              ui={content.presentation.ui}
+            />
+          </div>
+        </div>
+      </header>
+      <main className={styles.main} id="brutalist-main">
+        {children}
+      </main>
+    </div>
+  );
+}


## `feat(brutalist): 주 탐색과 모바일 메뉴 추가`

diff --git a/src/designs/brutalist/brutalist-route.tsx b/src/designs/brutalist/brutalist-route.tsx
index 2bdcc1f..a617a9a 100644
--- a/src/designs/brutalist/brutalist-route.tsx
+++ b/src/designs/brutalist/brutalist-route.tsx
@@ -140,6 +140,7 @@ export function BrutalistShell({
   currentPath: string;
   route: DesignRouteProps["route"];
 }) {
+  const shellCopy = content.presentation.brutalist.shell;
   const ui = content.presentation.ui;
   return (
     <div
@@ -176,10 +177,93 @@ export function BrutalistShell({
             />
           </div>
         </div>
+        <nav
+          aria-label={ui.primaryNavigationAriaLabel}
+          className={styles.navigation}
+        >
+          <ol className={styles.navigationList}>
+            {content.site.navigation.map((item, index) => (
+              <li key={`${item.href}-${item.label}`}>
+                <Link
+                  aria-current={
+                    isCurrentNavigation(item.href, currentPath) ? "page" : undefined
+                  }
+                  className={styles.navigationLink}
+                  href={brutalistHref(item.href, contentDebug)}
+                >
+                  <span className={styles.navigationIndex}>
+                    {String(index + 1).padStart(2, "0")}
+                  </span>
+                  <span>{item.label}</span>
+                </Link>
+              </li>
+            ))}
+          </ol>
+        </nav>
+        <details className={styles.mobileMenu}>
+          <summary>{ui.menuLabel}</summary>
+          <nav aria-label={ui.mobileNavigationAriaLabel}>
+            {content.site.navigation.map((item, index) => (
+              <Link
+                aria-current={
+                  isCurrentNavigation(item.href, currentPath) ? "page" : undefined
+                }
+                href={brutalistHref(item.href, contentDebug)}
+                key={`${item.href}-mobile`}
+              >
+                <span>{String(index + 1).padStart(2, "0")}</span>
+                {item.label}
+              </Link>
+            ))}
+          </nav>
+        </details>
       </header>
+      {contentDebug ? (
+        <aside className={styles.debugBanner} role="status">
+          <strong>{shellCopy.debugLabel}</strong>
+          <span>
+            {ui.debugPrefix}: {shellCopy.debugHint}
+          </span>
+        </aside>
+      ) : null}
       <main className={styles.main} id="brutalist-main">
         {children}
       </main>
     </div>
   );
 }
+
+export function ActionLink({
+  children,
+  className,
+  contentDebug,
+  href,
+  isExternal,
+}: {
+  children: React.ReactNode;
+  className: string;
+  contentDebug: boolean;
+  href: string;
+  isExternal?: boolean;
+}) {
+  const external = isExternal || /^https?:\/\//.test(href) || href.startsWith("mailto:");
+
+  if (external) {
+    return (
+      <a
+        className={className}
+        href={href}
+        rel={href.startsWith("mailto:") ? undefined : "noreferrer"}
+        target={href.startsWith("mailto:") ? undefined : "_blank"}
+      >
+        {children}
+      </a>
+    );
+  }
+
+  return (
+    <Link className={className} href={brutalistHref(href, contentDebug)}>
+      {children}
+    </Link>
+  );
+}


## `feat(brutalist): footer와 홈 히어로 연결`

diff --git a/src/designs/brutalist/brutalist-route.tsx b/src/designs/brutalist/brutalist-route.tsx
index a617a9a..065ecf7 100644
--- a/src/designs/brutalist/brutalist-route.tsx
+++ b/src/designs/brutalist/brutalist-route.tsx
@@ -142,6 +142,10 @@ export function BrutalistShell({
 }) {
   const shellCopy = content.presentation.brutalist.shell;
   const ui = content.presentation.ui;
+  const footerLinks = content.links.filter((link) =>
+    link.placements?.includes("footer"),
+  );
+
   return (
     <div
       className={styles.root}
@@ -229,10 +233,101 @@ export function BrutalistShell({
       <main className={styles.main} id="brutalist-main">
         {children}
       </main>
+      <footer className={styles.footer}>
+        <div className={styles.footerLead}>
+          <span className={styles.footerSymbol} aria-hidden="true">
+            ↳
+          </span>
+          <p>{content.site.footer.note}</p>
+        </div>
+        <div className={styles.footerMeta}>
+          <span>{content.site.footer.copyright}</span>
+          {footerLinks.map((link) => (
+            <ActionLink
+              className=""
+              contentDebug={contentDebug}
+              href={link.href}
+              isExternal={link.external}
+              key={link.id ?? `${link.type}-${link.href}`}
+            >
+              {link.label} <span aria-hidden="true">↗</span>
+            </ActionLink>
+          ))}
+        </div>
+      </footer>
     </div>
   );
 }
 
+export function HomeView({
+  content,
+  contentDebug,
+}: {
+  content: PortfolioContent;
+  contentDebug: boolean;
+}) {
+  const homeCopy = content.presentation.home.brutalist;
+  const metrics = getHomeMetrics(content);
+  return (
+    <>
+      {homeCopy.sections.map((section) => {
+        switch (section) {
+          case "hero":
+            return (
+              <section className={styles.homeHero} key={section}>
+                <div className={styles.heroStamp}>
+                  <span>
+                    {homeCopy.stampLabel} / {new Date().getFullYear()}
+                  </span>
+                  <span>{content.profile.availability}</span>
+                </div>
+                <div className={styles.heroCopy}>
+                  <p className={styles.eyebrow}>{content.profile.role}</p>
+                  <h1 className={styles.megaTitle}>
+                    <span>{content.profile.name}</span>
+                    <span className={styles.megaTitleAccent}>
+                      {content.profile.handle}
+                    </span>
+                  </h1>
+                  <p className={styles.heroHeadline}>{content.profile.headline}</p>
+                </div>
+                <div className={styles.heroSummary}>
+                  <p>{content.profile.summary}</p>
+                  <div className={styles.actionRow}>
+                    <Link
+                      className={styles.primaryAction}
+                      href={brutalistHref("/projects", contentDebug)}
+                    >
+                      {homeCopy.hero.primaryActionLabel}{" "}
+                      <span aria-hidden="true">↗</span>
+                    </Link>
+                    <Link
+                      className={styles.secondaryAction}
+                      href={brutalistHref("/contact", contentDebug)}
+                    >
+                      {homeCopy.hero.secondaryActionLabel}
+                    </Link>
+                  </div>
+                </div>
+                <dl className={styles.metricsGrid}>
+                  {metrics.map((metric, index) => (
+                    <div className={styles.metricBlock} key={metric.id}>
+                      <dt>
+                        {String(index + 1).padStart(2, "0")} / {metric.label}
+                      </dt>
+                      <dd>{String(metric.value).padStart(2, "0")}</dd>
+                      {metric.description ? <p>{metric.description}</p> : null}
+                    </div>
+                  ))}
+                </dl>
+              </section>
+            );
+        }
+      })}
+    </>
+  );
+}
+
 export function ActionLink({
   children,
   className,


