# Cinematic 장면·챕터 렌더러

## `style(cinematic): 암실 palette와 shell 기초 구성`

diff --git a/src/designs/cinematic/cinematic.module.css b/src/designs/cinematic/cinematic.module.css
new file mode 100644
index 0000000..fcc38c4
--- /dev/null
+++ b/src/designs/cinematic/cinematic.module.css
@@ -0,0 +1,26 @@
+.site {
+  --ink: #0a0b0e;
+  --paper: #d9d2c4;
+  --muted: #8f8a82;
+  --amber: #c98a4a;
+  background: var(--ink);
+  color: var(--paper);
+  font-family: var(--font-geist-sans), sans-serif;
+  min-height: 100vh;
+}
+
+.site ::selection { background: var(--amber); color: var(--ink); }
+.site a { color: inherit; }
+.site :where(a, summary):focus-visible { outline: 2px solid var(--amber); outline-offset: 4px; }
+.skipLink { background: var(--paper); color: var(--ink) !important; left: 1rem; padding: .75rem 1rem; position: fixed; top: -5rem; z-index: 100; }
+.skipLink:focus { top: 1rem; }
+.header { align-items: center; backdrop-filter: blur(18px); background: rgba(10,11,14,.82); border-bottom: 1px solid rgba(217,210,196,.16); display: grid; gap: 1.5rem; grid-template-columns: 1fr auto auto; min-height: 5rem; padding: 0 3vw; position: sticky; top: 0; z-index: 40; }
+.brand { display: flex; flex-direction: column; justify-content: center; min-height: 2.75rem; text-decoration: none; width: fit-content; }
+.brand span { font-size: .82rem; font-weight: 700; letter-spacing: .12em; text-transform: uppercase; }
+.brand small { color: var(--muted); font-size: .62rem; letter-spacing: .16em; margin-top: .2rem; text-transform: uppercase; }
+.desktopNav { display: flex; gap: 1.5rem; }
+.switcher { display: flex; justify-content: flex-end; }
+.desktopNav a { align-items: center; color: var(--muted); display: inline-flex; font-size: .68rem; letter-spacing: .1em; min-height: 2.75rem; padding: .8rem .15rem; text-decoration: none; text-transform: uppercase; }
+.desktopNav a:hover,.desktopNav a[aria-current="page"] { color: var(--paper); }
+.mobileNav { display: none; position: relative; }
+.mobileNav summary { align-items: center; cursor: pointer; display: inline-flex; font-size: .68rem; font-weight: 700; justify-content: center; letter-spacing: .12em; list-style: none; min-height: 2.75rem; min-width: 4rem; text-transform: uppercase; }


## `feat(cinematic): 링크와 chapter 표기 프리미티브 추가`

diff --git a/src/designs/cinematic/cinematic-route.tsx b/src/designs/cinematic/cinematic-route.tsx
new file mode 100644
index 0000000..2a61af4
--- /dev/null
+++ b/src/designs/cinematic/cinematic-route.tsx
@@ -0,0 +1,91 @@
+import Link from "next/link";
+import {
+  getTemplateHref,
+  type ContentLink,
+} from "@/lib/portfolio";
+import styles from "./cinematic.module.css";
+
+export function routeHref(href: string, contentDebug = false) {
+  return getTemplateHref(href, "cinematic", { contentDebug });
+}
+
+export function isCurrentNavigation(href: string, currentPath: string) {
+  if (href === "/") return currentPath === "/";
+  return currentPath === href || currentPath.startsWith(`${href}/`);
+}
+
+export function CinematicLink({
+  children,
+  className,
+  contentDebug,
+  external,
+  href,
+}: {
+  children: React.ReactNode;
+  className?: string;
+  contentDebug: boolean;
+  external?: boolean;
+  href: string;
+}) {
+  if (href.startsWith("/") && !href.startsWith("//")) {
+    return (
+      <Link className={className} href={routeHref(href, contentDebug)}>
+        {children}
+      </Link>
+    );
+  }
+
+  const opensNewTab = external || href.startsWith("http://") || href.startsWith("https://");
+
+  return (
+    <a
+      className={className}
+      href={href}
+      rel={opensNewTab ? "noreferrer" : undefined}
+      target={opensNewTab ? "_blank" : undefined}
+    >
+      {children}
+    </a>
+  );
+}
+
+export function LinkList({
+  contentDebug,
+  links,
+}: {
+  contentDebug: boolean;
+  links: ContentLink[];
+}) {
+  return (
+    <div className={styles.linkList}>
+      {links.filter((link) => link.enabled !== false).map((link) => {
+        const children = (
+          <>
+            {link.label}
+            <span aria-hidden="true">↗</span>
+          </>
+        );
+
+        return (
+          <CinematicLink
+            contentDebug={contentDebug}
+            external={link.external}
+            href={link.href}
+            key={link.id ?? `${link.label}-${link.href}`}
+          >
+            {children}
+          </CinematicLink>
+        );
+      })}
+    </div>
+  );
+}
+
+export function ChapterLabel({ children, index }: { children: React.ReactNode; index: number }) {
+  return (
+    <p className={styles.chapterLabel}>
+      <span>{String(index).padStart(2, "0")}</span>
+      {children}
+    </p>
+  );
+}


## `style(cinematic): 모바일 탐색과 hero 매체 구성`

diff --git a/src/designs/cinematic/cinematic.module.css b/src/designs/cinematic/cinematic.module.css
index fcc38c4..2d6443c 100644
--- a/src/designs/cinematic/cinematic.module.css
+++ b/src/designs/cinematic/cinematic.module.css
@@ -24,3 +24,21 @@
 .desktopNav a:hover,.desktopNav a[aria-current="page"] { color: var(--paper); }
 .mobileNav { display: none; position: relative; }
 .mobileNav summary { align-items: center; cursor: pointer; display: inline-flex; font-size: .68rem; font-weight: 700; justify-content: center; letter-spacing: .12em; list-style: none; min-height: 2.75rem; min-width: 4rem; text-transform: uppercase; }
+.mobileNav summary::-webkit-details-marker { display: none; }
+.mobileNav nav { background: #111318; border: 1px solid rgba(217,210,196,.28); display: grid; max-height: min(70vh, 32rem); min-width: min(22rem, calc(100vw - 2rem)); overflow-y: auto; padding: .6rem; position: absolute; right: 0; top: calc(100% + .75rem); }
+.mobileNav nav a { align-items: center; border-bottom: 1px solid rgba(217,210,196,.14); color: var(--muted); display: flex; font-size: .72rem; letter-spacing: .1em; min-height: 3rem; padding: .65rem .8rem; text-decoration: none; text-transform: uppercase; }
+.mobileNav nav a:last-child { border-bottom: 0; }
+.mobileNav nav a:hover,.mobileNav nav a[aria-current="page"] { color: var(--paper); }
+.footer { border-top: 1px solid rgba(217,210,196,.16); color: var(--muted); display: flex; font-size: .7rem; justify-content: space-between; letter-spacing: .08em; padding: 3rem 4vw; text-transform: uppercase; }
+
+.hero { display: grid; grid-template-columns: minmax(20rem,.72fr) minmax(0,1.28fr); min-height: calc(100vh - 5rem); }
+.heroCopy { align-self: center; padding: 8vw 4vw; }
+.kicker,.chapterLabel { color: var(--muted); font-size: .66rem; letter-spacing: .16em; text-transform: uppercase; }
+.hero h1 { font-size: clamp(3.8rem,6vw,6.5rem); font-weight: 500; letter-spacing: -.065em; line-height: .88; margin: 2.3rem 0; max-width: 9ch; }
+.lede { color: var(--paper); font-size: clamp(1.25rem,2.1vw,2.1rem); line-height: 1.28; max-width: 28ch; }
+.summary { color: var(--muted); line-height: 1.8; margin-top: 1.5rem; max-width: 36rem; }
+.heroActions,.linkList { display: flex; flex-wrap: wrap; gap: .85rem 1.5rem; margin-top: 2.5rem; }
+.heroActions a,.linkList a,.textLink { align-items: center; border-bottom: 1px solid var(--amber); display: inline-flex; font-size: .75rem; gap: .75rem; letter-spacing: .09em; min-height: 2.75rem; text-decoration: none; text-transform: uppercase; }
+.heroMedia { min-height: 45rem; min-width: 0; position: relative; }
+.heroMedia>.media { height: 100%; min-height: 45rem; width: 100%; }
+.heroMedia>p { background: rgba(10,11,14,.85); bottom: 1.5rem; font-size: .67rem; left: 1.5rem; letter-spacing: .1em; padding: .75rem 1rem; position: absolute; text-transform: uppercase; }


## `feat(cinematic): 공용 frame과 media 추가`

diff --git a/src/designs/cinematic/cinematic-route.tsx b/src/designs/cinematic/cinematic-route.tsx
index 2a61af4..0aaf812 100644
--- a/src/designs/cinematic/cinematic-route.tsx
+++ b/src/designs/cinematic/cinematic-route.tsx
@@ -1,8 +1,11 @@
+import Image from "next/image";
 import Link from "next/link";
+import { DesignSwitcher } from "@/components/portfolio/design-switcher";
 import {
   getTemplateHref,
   type ContentLink,
 } from "@/lib/portfolio";
+import type { DesignRouteProps } from "@/designs/types";
 import styles from "./cinematic.module.css";
 
 export function routeHref(href: string, contentDebug = false) {
@@ -81,6 +84,90 @@ export function LinkList({
   );
 }
 
+export function Frame({
+  children,
+  content,
+  contentDebug,
+  currentPath,
+}: DesignRouteProps & { children: React.ReactNode }) {
+  const footerLinks = content.links.filter((link) =>
+    link.placements?.includes("footer"),
+  );
+  const ui = content.presentation.ui;
+
+  return (
+    <div className={styles.site} data-site-design="cinematic">
+      <a className={styles.skipLink} href="#cinematic-content">
+        {ui.skipLinkLabel}
+      </a>
+      <header className={styles.header}>
+        <Link className={styles.brand} href={routeHref("/", contentDebug)}>
+          <span>{content.site.brand}</span>
+          <small>{content.presentation.cinematic.shell.brandSubtitle}</small>
+        </Link>
+        <nav aria-label={ui.primaryNavigationAriaLabel} className={styles.desktopNav}>
+          {content.site.navigation.map((item) => (
+            <Link
+              aria-current={isCurrentNavigation(item.href, currentPath) ? "page" : undefined}
+              href={routeHref(item.href, contentDebug)}
+              key={item.href}
+            >
+              {item.label}
+            </Link>
+          ))}
+        </nav>
+        <div className={styles.switcher}>
+          <DesignSwitcher
+            activeId="cinematic"
+            contentDebug={contentDebug}
+            currentPath={currentPath}
+            templates={content.presentation.templates}
+            ui={ui}
+          />
+        </div>
+        <details className={styles.mobileNav}>
+          <summary>{ui.menuLabel}</summary>
+          <nav aria-label={ui.mobileNavigationAriaLabel}>
+            {content.site.navigation.map((item) => (
+              <Link
+                aria-current={isCurrentNavigation(item.href, currentPath) ? "page" : undefined}
+                href={routeHref(item.href, contentDebug)}
+                key={`${item.href}-mobile`}
+              >
+                {item.label}
+              </Link>
+            ))}
+          </nav>
+        </details>
+      </header>
+      <main id="cinematic-content">{children}</main>
+      <footer className={styles.footer}>
+        <p>{content.site.footer.note}</p>
+        {footerLinks.length > 0 ? (
+          <LinkList contentDebug={contentDebug} links={footerLinks} />
+        ) : null}
+        <p>{content.site.footer.copyright}</p>
+      </footer>
+    </div>
+  );
+}
+
+export function Media({
+  alt,
+  priority = false,
+  src,
+}: {
+  alt: string;
+  priority?: boolean;
+  src: string;
+}) {
+  return (
+    <figure className={styles.media}>
+      <Image alt={alt} fill priority={priority} sizes="(max-width: 900px) 100vw, 72vw" src={src} />
+    </figure>
+  );
+}
+
 export function ChapterLabel({ children, index }: { children: React.ReactNode; index: number }) {
   return (
     <p className={styles.chapterLabel}>


## `feat(cinematic): 프로젝트 chapter 추가`

diff --git a/src/designs/cinematic/cinematic-route.tsx b/src/designs/cinematic/cinematic-route.tsx
index 0aaf812..db68dc2 100644
--- a/src/designs/cinematic/cinematic-route.tsx
+++ b/src/designs/cinematic/cinematic-route.tsx
@@ -4,6 +4,7 @@ import { DesignSwitcher } from "@/components/portfolio/design-switcher";
 import {
   getTemplateHref,
   type ContentLink,
+  type PortfolioProject,
 } from "@/lib/portfolio";
 import type { DesignRouteProps } from "@/designs/types";
 import styles from "./cinematic.module.css";
@@ -176,3 +177,38 @@ export function ChapterLabel({ children, index }: { children: React.ReactNode; i
     </p>
   );
 }
+
+export function ProjectChapter({
+  actionLabel,
+  index,
+  openItemAriaTemplate,
+  priority,
+  project,
+  contentDebug,
+}: {
+  actionLabel: string;
+  contentDebug: boolean;
+  index: number;
+  openItemAriaTemplate: string;
+  priority?: boolean;
+  project: PortfolioProject;
+}) {
+  return (
+    <article className={styles.projectChapter}>
+      <div className={styles.stickyCopy}>
+        <ChapterLabel index={index}>{project.category}</ChapterLabel>
+        <h2>{project.title}</h2>
+        <p>{project.summary}</p>
+        <Link className={styles.textLink} href={routeHref(`/projects/${project.id}`, contentDebug)}>
+          {actionLabel} <span aria-hidden="true">→</span>
+        </Link>
+      </div>
+      <Link
+        aria-label={openItemAriaTemplate.replace("{title}", project.title)}
+        href={routeHref(`/projects/${project.id}`, contentDebug)}
+      >
+        <Media alt={project.screenshot.alt} priority={priority} src={project.screenshot.src} />
+      </Link>
+    </article>
+  );
+}


## `style(cinematic): chapter와 archive 지면 구성`

diff --git a/src/designs/cinematic/cinematic.module.css b/src/designs/cinematic/cinematic.module.css
index 2d6443c..b5f4aa1 100644
--- a/src/designs/cinematic/cinematic.module.css
+++ b/src/designs/cinematic/cinematic.module.css
@@ -42,3 +42,24 @@
 .heroMedia { min-height: 45rem; min-width: 0; position: relative; }
 .heroMedia>.media { height: 100%; min-height: 45rem; width: 100%; }
 .heroMedia>p { background: rgba(10,11,14,.85); bottom: 1.5rem; font-size: .67rem; left: 1.5rem; letter-spacing: .1em; padding: .75rem 1rem; position: absolute; text-transform: uppercase; }
+.media { aspect-ratio: 16/10; background: #17191e; margin: 0; overflow: hidden; position: relative; }
+.media img { filter: saturate(.74) contrast(1.04); object-fit: cover; transition: filter .7s ease, transform 1.2s cubic-bezier(.2,.8,.2,1); }
+a:hover .media img { filter: saturate(1); transform: scale(1.025); }
+.statement { align-items: start; border-bottom: 1px solid rgba(217,210,196,.16); border-top: 1px solid rgba(217,210,196,.16); display: grid; gap: 4rem; grid-template-columns: 16rem 1fr; padding: 10vw 4vw; }
+.statement>p { font-size: clamp(2rem,5vw,5.8rem); letter-spacing: -.055em; line-height: .98; margin: 0; max-width: 19ch; }
+.chapterLabel { align-items: center; display: flex; gap: .7rem; margin: 0; }
+.chapterLabel span { color: var(--amber); font-family: var(--font-geist-mono),monospace; }
+.chapters { display: grid; }
+.projectChapter { border-bottom: 1px solid rgba(217,210,196,.16); display: grid; gap: 5vw; grid-template-columns: minmax(18rem,.7fr) minmax(0,1.3fr); min-height: 80vh; padding: 6vw 4vw; }
+.stickyCopy { align-self: start; padding-top: 2rem; position: sticky; top: 8rem; }
+.stickyCopy h2 { font-size: clamp(2.8rem,5.8vw,6.5rem); font-weight: 500; letter-spacing: -.06em; line-height: .9; margin: 2rem 0; max-width: 9ch; }
+.stickyCopy>p { color: var(--muted); line-height: 1.75; max-width: 34rem; }
+.projectChapter>a:last-child { align-self: center; }
+.dualPanel { display: grid; gap: 8vw; grid-template-columns: 1.35fr .65fr; padding: 10vw 4vw; }
+.dualPanel h2,.essayGrid h2,.resumeGrid h2 { font-size: clamp(2rem,4vw,4.5rem); font-weight: 500; letter-spacing: -.05em; margin: 2rem 0 3rem; }
+.focusGrid { display: grid; gap: 1px; grid-template-columns: repeat(2,1fr); background: rgba(217,210,196,.16); }
+.focusGrid article { background: var(--ink); padding: 2rem; }
+.focusGrid h3,.essayGrid h3,.resumeGrid h3 { font-size: 1rem; letter-spacing: .04em; }
+.focusGrid p,.essayGrid p,.resumeGrid p { color: var(--muted); line-height: 1.7; }
+.indexHero,.textHero,.contactHero,.caseHero { min-height: 62vh; padding: 10vw 4vw 7vw; }
+.indexHero p { color: var(--amber); letter-spacing: .12em; text-transform: uppercase; }


## `feat(cinematic-home): 소개와 대표 프로젝트 구성`

diff --git a/src/designs/cinematic/cinematic-route.tsx b/src/designs/cinematic/cinematic-route.tsx
index db68dc2..b874098 100644
--- a/src/designs/cinematic/cinematic-route.tsx
+++ b/src/designs/cinematic/cinematic-route.tsx
@@ -1,5 +1,6 @@
 import Image from "next/image";
 import Link from "next/link";
+import { Fragment } from "react";
 import { DesignSwitcher } from "@/components/portfolio/design-switcher";
 import {
   getTemplateHref,
@@ -212,3 +213,86 @@ export function ProjectChapter({
     </article>
   );
 }
+
+export function HomeView({ content, contentDebug }: DesignRouteProps) {
+  const featured = content.projects.filter((project) => project.featured);
+  const lead = featured[0] ?? content.projects[0];
+  const copy = content.presentation.home.cinematic;
+  const ui = content.presentation.ui;
+  const sectionNodes: Record<(typeof copy.sections)[number], React.ReactNode> = {
+    hero: (
+      <section className={styles.hero}>
+        <div className={styles.heroCopy}>
+          <p className={styles.kicker}>{content.profile.name} · {content.profile.location}</p>
+          <h1>{content.profile.role}</h1>
+          <p className={styles.lede}>{content.profile.headline}</p>
+          <p className={styles.summary}>{content.profile.summary}</p>
+          <div className={styles.heroActions}>
+            <Link href={routeHref("/projects", contentDebug)}>{copy.hero.primaryActionLabel}</Link>
+            <Link href={routeHref("/contact", contentDebug)}>{copy.hero.secondaryActionLabel}</Link>
+          </div>
+        </div>
+        {lead ? (
+          <div className={styles.heroMedia}>
+            <Media alt={lead.screenshot.alt} priority src={lead.screenshot.src} />
+            <p>{lead.title} · {lead.period}</p>
+          </div>
+        ) : null}
+      </section>
+    ),
+    statement: (
+      <section className={styles.statement}>
+        <ChapterLabel index={1}>{copy.statementLabel}</ChapterLabel>
+        <p>{content.presentation.home.shared.technicalFocus.body}</p>
+      </section>
+    ),
+    projects: (
+      <section className={styles.chapters}>
+        {(featured.length > 0 ? featured : content.projects.slice(0, 4)).map(
+          (project, index) => (
+            <ProjectChapter
+              actionLabel={copy.caseStudyActionLabel}
+              contentDebug={contentDebug}
+              index={index + 2}
+              key={project.id}
+              openItemAriaTemplate={ui.openItemAriaTemplate}
+              project={project}
+            />
+          ),
+        )}
+      </section>
+    ),
+    focusContact: (
+      <section className={styles.dualPanel}>
+        <div>
+          <ChapterLabel index={featured.length + 2}>{copy.focusLabel}</ChapterLabel>
+          <h2>{content.presentation.home.shared.technicalFocus.title}</h2>
+          <div className={styles.focusGrid}>
+            {content.skills.focusAreas.map((area) => (
+              <article key={area.title}>
+                <h3>{area.title}</h3>
+                <p>{area.body}</p>
+              </article>
+            ))}
+          </div>
+        </div>
+        <div>
+          <ChapterLabel index={featured.length + 3}>{ui.nowLabel}</ChapterLabel>
+          <h2>{content.contact.title}</h2>
+          <p>{content.contact.availability}</p>
+          <Link className={styles.textLink} href={routeHref("/contact", contentDebug)}>
+            {copy.contactActionLabel} <span aria-hidden="true">→</span>
+          </Link>
+        </div>
+      </section>
+    ),
+  };
+
+  return (
+    <>
+      {copy.sections.map((sectionId) => (
+        <Fragment key={sectionId}>{sectionNodes[sectionId]}</Fragment>
+      ))}
+    </>
+  );
+}


## `feat(cinematic-projects): 프로젝트 archive 구성`

diff --git a/src/designs/cinematic/cinematic-route.tsx b/src/designs/cinematic/cinematic-route.tsx
index b874098..27c7c86 100644
--- a/src/designs/cinematic/cinematic-route.tsx
+++ b/src/designs/cinematic/cinematic-route.tsx
@@ -296,3 +296,31 @@ export function HomeView({ content, contentDebug }: DesignRouteProps) {
     </>
   );
 }
+
+export function ProjectsView({ content, contentDebug }: DesignRouteProps) {
+  const copy = content.presentation.pages.projects.cinematic.hero;
+  const homeCopy = content.presentation.home.cinematic;
+
+  return (
+    <>
+      <section className={styles.indexHero}>
+        <p>{copy.eyebrow} / {String(content.projects.length).padStart(2, "0")} {copy.entryLabel}</p>
+        <h1>{copy.title}</h1>
+        <p className={styles.indexSummary}>{copy.body}</p>
+      </section>
+      <section className={styles.chapters}>
+        {content.projects.map((project, index) => (
+          <ProjectChapter
+            actionLabel={homeCopy.caseStudyActionLabel}
+            contentDebug={contentDebug}
+            index={index + 1}
+            key={project.id}
+            openItemAriaTemplate={content.presentation.ui.openItemAriaTemplate}
+            priority={index === 0}
+            project={project}
+          />
+        ))}
+      </section>
+    </>
+  );
+}


