# 클라이언트 비용 최적화와 성능 예산

## `refactor(ui): reveal 콘텐츠를 server에서 즉시 표시`

diff --git a/src/components/portfolio/reveal.tsx b/src/components/portfolio/reveal.tsx
index 01c418e..f8b4c9b 100644
--- a/src/components/portfolio/reveal.tsx
+++ b/src/components/portfolio/reveal.tsx
@@ -1,7 +1,3 @@
-"use client";
-
-import { useEffect, useRef, useState, type Ref } from "react";
-
 export function Reveal({
   as = "div",
   children,
@@ -13,43 +9,11 @@ export function Reveal({
   className?: string;
   delay?: number;
 }) {
-  const ref = useRef<HTMLElement>(null);
-  const [visible, setVisible] = useState(
-    () => typeof window !== "undefined" && !("IntersectionObserver" in window),
-  );
-
-  useEffect(() => {
-    const node = ref.current;
-
-    if (!node) {
-      return;
-    }
-
-    if (!("IntersectionObserver" in window)) {
-      return;
-    }
-
-    const observer = new IntersectionObserver(
-      ([entry]) => {
-        if (entry.isIntersecting) {
-          setVisible(true);
-          observer.disconnect();
-        }
-      },
-      { rootMargin: "0px 0px -12% 0px", threshold: 0.12 },
-    );
-
-    observer.observe(node);
-
-    return () => observer.disconnect();
-  }, []);
-
   const Component = as;
 
   return (
     <Component
-      className={`reveal-item ${visible ? "is-visible" : ""} ${className}`}
-      ref={ref as Ref<HTMLDivElement & HTMLLIElement>}
+      className={`reveal-item is-visible ${className}`}
       style={{ transitionDelay: `${delay}ms` }}
     >
       {children}


## `perf(font): route별 글꼴 로딩 비용 축소`

diff --git a/src/app/layout.tsx b/src/app/layout.tsx
index 54241f5..5ef7d97 100644
--- a/src/app/layout.tsx
+++ b/src/app/layout.tsx
@@ -1,6 +1,7 @@
 import type { Metadata } from "next";
 import localFont from "next/font/local";
 import { headers } from "next/headers";
+import type { CSSProperties } from "react";
 import { StructuredData } from "@/components/portfolio/structured-data";
 import {
   resolvePortfolioContentMode,
@@ -14,21 +15,22 @@ import {
 import "./globals.css";
 
 const geistSans = localFont({
-  display: "swap",
+  display: "optional",
   src: "./fonts/Geist-Variable.woff2",
   variable: "--font-geist-sans",
   weight: "100 900",
 });
 
 const geistMono = localFont({
-  display: "swap",
+  display: "optional",
+  preload: false,
   src: "./fonts/GeistMono-Variable.woff2",
   variable: "--font-geist-mono",
   weight: "100 900",
 });
 
 const koreanSerif = localFont({
-  display: "swap",
+  display: "optional",
   preload: false,
   src: "./fonts/SourceHanSerifKR-Variable.woff2",
   variable: "--font-noto-serif-kr",
@@ -36,6 +38,10 @@ const koreanSerif = localFont({
 });
 
 const { site } = getPortfolioContent();
+const usesKoreanSerif = site.language.toLowerCase().startsWith("ko");
+const serifFallback = {
+  "--font-noto-serif-kr": '"Iowan Old Style", Baskerville, Georgia, serif',
+} as CSSProperties;
 
 export async function generateMetadata(): Promise<Metadata> {
   const mode = resolvePortfolioContentMode(
@@ -82,7 +88,8 @@ export default function RootLayout({
     <html
       lang={site.language}
       data-scroll-behavior="smooth"
-      className={`${geistSans.variable} ${geistMono.variable} ${koreanSerif.variable} h-full antialiased`}
+      className={`${geistSans.variable} ${geistMono.variable} ${usesKoreanSerif ? koreanSerif.variable : ""} h-full antialiased`}
+      style={usesKoreanSerif ? undefined : serifFallback}
     >
       <body className="min-h-full flex flex-col">
         {siteStructuredData ? <StructuredData data={siteStructuredData} /> : null}
diff --git a/src/components/portfolio/design-switcher.module.css b/src/components/portfolio/design-switcher.module.css
index 9f04800..5a3edf5 100644
--- a/src/components/portfolio/design-switcher.module.css
+++ b/src/components/portfolio/design-switcher.module.css
@@ -24,7 +24,7 @@
 }
 
 .count {
-  font-family: var(--font-geist-mono), ui-monospace, monospace;
+  font-family: ui-monospace, "SFMono-Regular", Consolas, monospace;
   font-size: 0.68rem;
   letter-spacing: 0.08em;
   opacity: 0.72;
@@ -107,7 +107,7 @@
 }
 
 .number {
-  font-family: var(--font-geist-mono), ui-monospace, monospace;
+  font-family: ui-monospace, "SFMono-Regular", Consolas, monospace;
   font-size: 0.65rem;
 }
 


## `perf(navigation): 유휴 route prefetch 비활성화`

diff --git a/src/components/portfolio/content-link.tsx b/src/components/portfolio/content-link.tsx
index 4e937ae..228bcaf 100644
--- a/src/components/portfolio/content-link.tsx
+++ b/src/components/portfolio/content-link.tsx
@@ -33,6 +33,7 @@ export function ContentLinkView({
     <Link
       className={className}
       href={getTemplateHref(link.href, homeTemplate, { contentDebug })}
+      prefetch={false}
     >
       {children}
     </Link>
diff --git a/src/components/portfolio/design-switcher.tsx b/src/components/portfolio/design-switcher.tsx
index b85a1e4..9537dd2 100644
--- a/src/components/portfolio/design-switcher.tsx
+++ b/src/components/portfolio/design-switcher.tsx
@@ -67,6 +67,7 @@ export function DesignSwitcher({
                     defaultId,
                     { contentDebug },
                   )}
+                  prefetch={false}
                 >
                   <span aria-hidden="true" className={styles.swatch}>
                     {design.swatch.map((color) => (
diff --git a/src/components/portfolio/project-card.tsx b/src/components/portfolio/project-card.tsx
index 30ae66e..afc4d79 100644
--- a/src/components/portfolio/project-card.tsx
+++ b/src/components/portfolio/project-card.tsx
@@ -39,6 +39,7 @@ export function ProjectCard({
         aria-label={`${project.title} case study`}
         className="block focus:outline-none focus:ring-2 focus:ring-accent focus:ring-offset-2 focus:ring-offset-background"
         href={caseStudyHref}
+        prefetch={false}
       >
         <ProjectScreenshot image={project.screenshot} priority={priority} />
       </Link>
@@ -57,6 +58,7 @@ export function ProjectCard({
           <Link
             className="group/title inline-flex items-center gap-2 text-xl font-semibold text-foreground hover:text-accent-strong"
             href={caseStudyHref}
+            prefetch={false}
           >
             {project.title}
             <ArrowUpRightIcon className="opacity-0 transition group-hover/title:opacity-100" />
diff --git a/src/components/portfolio/site-shell.tsx b/src/components/portfolio/site-shell.tsx
index 7153d03..dc671e7 100644
--- a/src/components/portfolio/site-shell.tsx
+++ b/src/components/portfolio/site-shell.tsx
@@ -46,6 +46,7 @@ export function SiteHeader({
           href={getTemplateHref("/", templateSwitcher?.activeId, {
             contentDebug: templateSwitcher?.contentDebug,
           })}
+          prefetch={false}
         >
           {profile.handle}
         </Link>
@@ -65,6 +66,7 @@ export function SiteHeader({
                 contentDebug: templateSwitcher?.contentDebug,
               })}
               key={item.href}
+              prefetch={false}
             >
               {item.label}
             </Link>
@@ -90,6 +92,7 @@ export function SiteHeader({
                   contentDebug: templateSwitcher?.contentDebug,
                 })}
                 key={item.href}
+                prefetch={false}
               >
                 {item.label}
               </Link>
@@ -127,6 +130,7 @@ export function SiteFooter({
         <Link
           className="inline-flex items-center gap-2 font-semibold text-foreground transition hover:text-accent-strong"
           href={getTemplateHref("/", homeTemplate, { contentDebug })}
+          prefetch={false}
         >
           {site.footer.copyright}
           <ArrowRightIcon className="-rotate-45" />
diff --git a/src/designs/brutalist/brutalist-route.tsx b/src/designs/brutalist/brutalist-route.tsx
index 01e52ba..bbacd12 100644
--- a/src/designs/brutalist/brutalist-route.tsx
+++ b/src/designs/brutalist/brutalist-route.tsx
@@ -207,6 +207,7 @@ function BrutalistShell({
           <Link
             className={styles.brand}
             href={brutalistHref("/", contentDebug)}
+            prefetch={false}
           >
             <span className={styles.brandMark} aria-hidden="true">
               ■
@@ -242,6 +243,7 @@ function BrutalistShell({
                   }
                   className={styles.navigationLink}
                   href={brutalistHref(item.href, contentDebug)}
+                  prefetch={false}
                 >
                   <span className={styles.navigationIndex}>
                     {String(index + 1).padStart(2, "0")}
@@ -262,6 +264,7 @@ function BrutalistShell({
                 }
                 href={brutalistHref(item.href, contentDebug)}
                 key={`${item.href}-mobile`}
+                prefetch={false}
               >
                 <span>{String(index + 1).padStart(2, "0")}</span>
                 {item.label}
@@ -350,6 +353,7 @@ function HomeView({
                     <Link
                       className={styles.primaryAction}
                       href={brutalistHref("/projects", contentDebug)}
+                      prefetch={false}
                     >
                       {homeCopy.hero.primaryActionLabel}{" "}
                       <span aria-hidden="true">↗</span>
@@ -357,6 +361,7 @@ function HomeView({
                     <Link
                       className={styles.secondaryAction}
                       href={brutalistHref("/contact", contentDebug)}
+                      prefetch={false}
                     >
                       {homeCopy.hero.secondaryActionLabel}
                     </Link>
@@ -405,6 +410,7 @@ function HomeView({
                 <Link
                   className={styles.fullWidthAction}
                   href={brutalistHref("/projects", contentDebug)}
+                  prefetch={false}
                 >
                   {homeCopy.featured.actionLabel}{" "}
                   ({String(viewModel.projectCount).padStart(2, "0")})
@@ -468,6 +474,7 @@ function HomeView({
                 <Link
                   className={styles.fullWidthAction}
                   href={brutalistHref("/journey", contentDebug)}
+                  prefetch={false}
                 >
                   {homeCopy.journeyActionLabel}{" "}
                   <span aria-hidden="true">→</span>
@@ -650,6 +657,7 @@ function ProjectDetailView({
             <Link
               className={styles.backLink}
               href={brutalistHref("/projects", contentDebug)}
+              prefetch={false}
             >
               ← {copy.backLabel}
             </Link>
diff --git a/src/designs/cinematic/cinematic-route.tsx b/src/designs/cinematic/cinematic-route.tsx
index 865999f..df88663 100644
--- a/src/designs/cinematic/cinematic-route.tsx
+++ b/src/designs/cinematic/cinematic-route.tsx
@@ -44,7 +44,11 @@ function CinematicLink({
 }) {
   if (href.startsWith("/") && !href.startsWith("//")) {
     return (
-      <Link className={className} href={routeHref(href, contentDebug)}>
+      <Link
+        className={className}
+        href={routeHref(href, contentDebug)}
+        prefetch={false}
+      >
         {children}
       </Link>
     );
@@ -111,7 +115,11 @@ function Frame({
         {ui.skipLinkLabel}
       </a>
       <header className={styles.header}>
-        <Link className={styles.brand} href={routeHref("/", contentDebug)}>
+        <Link
+          className={styles.brand}
+          href={routeHref("/", contentDebug)}
+          prefetch={false}
+        >
           <span>{content.site.brand}</span>
           <small>{content.presentation.cinematic.shell.brandSubtitle}</small>
         </Link>
@@ -121,6 +129,7 @@ function Frame({
               aria-current={isCurrentNavigation(item.href, currentPath) ? "page" : undefined}
               href={routeHref(item.href, contentDebug)}
               key={item.href}
+              prefetch={false}
             >
               {item.label}
             </Link>
@@ -144,6 +153,7 @@ function Frame({
                 aria-current={isCurrentNavigation(item.href, currentPath) ? "page" : undefined}
                 href={routeHref(item.href, contentDebug)}
                 key={`${item.href}-mobile`}
+                prefetch={false}
               >
                 {item.label}
               </Link>
@@ -211,13 +221,18 @@ function ProjectChapter({
         <ChapterLabel index={index}>{project.category}</ChapterLabel>
         <h2>{project.title}</h2>
         <p>{project.summary}</p>
-        <Link className={styles.textLink} href={routeHref(`/projects/${project.id}`, contentDebug)}>
+        <Link
+          className={styles.textLink}
+          href={routeHref(`/projects/${project.id}`, contentDebug)}
+          prefetch={false}
+        >
           {actionLabel} <span aria-hidden="true">→</span>
         </Link>
       </div>
       <Link
         aria-label={openItemAriaTemplate.replace("{title}", project.title)}
         href={routeHref(`/projects/${project.id}`, contentDebug)}
+        prefetch={false}
       >
         <Media alt={project.screenshot.alt} priority={priority} src={project.screenshot.src} />
       </Link>
@@ -240,8 +255,12 @@ function HomeView({ content, contentDebug }: DesignRouteProps) {
           <p className={styles.lede}>{content.profile.headline}</p>
           <p className={styles.summary}>{content.profile.summary}</p>
           <div className={styles.heroActions}>
-            <Link href={routeHref("/projects", contentDebug)}>{copy.hero.primaryActionLabel}</Link>
-            <Link href={routeHref("/contact", contentDebug)}>{copy.hero.secondaryActionLabel}</Link>
+            <Link href={routeHref("/projects", contentDebug)} prefetch={false}>
+              {copy.hero.primaryActionLabel}
+            </Link>
+            <Link href={routeHref("/contact", contentDebug)} prefetch={false}>
+              {copy.hero.secondaryActionLabel}
+            </Link>
           </div>
         </div>
         {lead ? (
@@ -368,7 +387,9 @@ function ProjectDetailView({ content, contentDebug, project }: DesignRouteProps)
   return (
     <article className={styles.caseStudy}>
       <header className={styles.caseHero}>
-        <Link href={routeHref("/projects", contentDebug)}>← {copy.backLabel}</Link>
+        <Link href={routeHref("/projects", contentDebug)} prefetch={false}>
+          ← {copy.backLabel}
+        </Link>
         <p>{project.category} · {project.period}</p>
         <h1>{project.title}</h1>
         <p className={styles.lede}>{project.summary}</p>
diff --git a/src/designs/classic/home-route.tsx b/src/designs/classic/home-route.tsx
index 1e19b2e..2b05050 100644
--- a/src/designs/classic/home-route.tsx
+++ b/src/designs/classic/home-route.tsx
@@ -190,6 +190,7 @@ function ClassicHeroSection({
               href={getTemplateHref("/projects", activeTemplate, {
                 contentDebug,
               })}
+              prefetch={false}
             >
               {copy.primaryActionLabel}
               <ArrowRightIcon />
@@ -255,6 +256,7 @@ function ClassicFeaturedProjectsSection({
             href={getTemplateHref("/projects", activeTemplate, {
               contentDebug,
             })}
+            prefetch={false}
           >
             {copy.actionLabel}
             <ArrowRightIcon />
diff --git a/src/designs/classic/project-detail-route.tsx b/src/designs/classic/project-detail-route.tsx
index e5c4f76..362bb6e 100644
--- a/src/designs/classic/project-detail-route.tsx
+++ b/src/designs/classic/project-detail-route.tsx
@@ -67,6 +67,7 @@ function ProjectHero({
           <Link
             className="inline-flex items-center gap-2 text-sm font-semibold text-muted transition hover:text-foreground"
             href={getTemplateHref("/projects", homeTemplate, { contentDebug })}
+            prefetch={false}
           >
             <ArrowRightIcon className="rotate-180" />
             {pageCopy.backLabel}
diff --git a/src/designs/design/home-route.tsx b/src/designs/design/home-route.tsx
index 7724eb1..f2e2d43 100644
--- a/src/designs/design/home-route.tsx
+++ b/src/designs/design/home-route.tsx
@@ -212,6 +212,7 @@ function HeroSection({
               href={getTemplateHref("/projects", activeTemplate, {
                 contentDebug,
               })}
+              prefetch={false}
             >
               {copy.primaryActionLabel}
               <ArrowRightIcon />
@@ -253,6 +254,7 @@ function HeroSection({
                     activeTemplate,
                     { contentDebug },
                   )}
+                  prefetch={false}
                 >
                   {copy.leadActionLabel}
                   <ArrowRightIcon />
@@ -271,6 +273,7 @@ function HeroSection({
                       { contentDebug },
                     )}
                     key={project.id}
+                    prefetch={false}
                   >
                     <p className="text-xs font-semibold uppercase text-muted">
                       {project.category}
@@ -321,6 +324,7 @@ function FeaturedProjectsSection({
             href={getTemplateHref("/projects", activeTemplate, {
               contentDebug,
             })}
+            prefetch={false}
           >
             {copy.actionLabel}
             <ArrowRightIcon />
diff --git a/src/designs/design/project-detail-route.tsx b/src/designs/design/project-detail-route.tsx
index 7b2fdda..0ab6c1b 100644
--- a/src/designs/design/project-detail-route.tsx
+++ b/src/designs/design/project-detail-route.tsx
@@ -67,6 +67,7 @@ function ProjectHero({
           <Link
             className="inline-flex items-center gap-2 text-sm font-semibold text-muted transition hover:text-foreground"
             href={getTemplateHref("/projects", homeTemplate, { contentDebug })}
+            prefetch={false}
           >
             <ArrowRightIcon className="rotate-180" />
             {pageCopy.backLabel}
diff --git a/src/designs/editorial/editorial-route.tsx b/src/designs/editorial/editorial-route.tsx
index cf53255..9fba3fc 100644
--- a/src/designs/editorial/editorial-route.tsx
+++ b/src/designs/editorial/editorial-route.tsx
@@ -135,6 +135,7 @@ function EditorialContentLink({
       <Link
         className={className}
         href={editorialHref(link.href, contentDebug)}
+        prefetch={false}
       >
         {children ?? link.label}
       </Link>
@@ -183,6 +184,7 @@ function EditorialShell({
           <Link
             className={styles.wordmark}
             href={editorialHref("/", contentDebug)}
+            prefetch={false}
           >
             <span>{content.profile.name}</span>
             <small>{content.profile.role}</small>
@@ -196,6 +198,7 @@ function EditorialShell({
                 className={styles.navLink}
                 href={editorialHref(item.href, contentDebug)}
                 key={`${item.href}-${index}`}
+                prefetch={false}
               >
                 <span>{twoDigits(index)}</span>
                 {item.label}
@@ -224,6 +227,7 @@ function EditorialShell({
                   }
                   href={editorialHref(item.href, contentDebug)}
                   key={`${item.href}-mobile-${index}`}
+                  prefetch={false}
                 >
                   <span>{twoDigits(index)}</span>
                   {item.label}
@@ -286,7 +290,10 @@ function ProjectIndexItem({
       <div className={styles.projectIndexTitle}>
         <p>{project.category}</p>
         <h3>
-          <Link href={editorialHref(`/projects/${project.id}`, contentDebug)}>
+          <Link
+            href={editorialHref(`/projects/${project.id}`, contentDebug)}
+            prefetch={false}
+          >
             {project.title}
           </Link>
         </h3>
@@ -303,6 +310,7 @@ function ProjectIndexItem({
         aria-label={readCaseStudyAriaTemplate.replace("{title}", project.title)}
         className={styles.indexArrow}
         href={editorialHref(`/projects/${project.id}`, contentDebug)}
+        prefetch={false}
       >
         <Arrow />
       </Link>
@@ -371,6 +379,7 @@ function HomeRoute({ content, contentDebug }: EditorialRouteProps) {
                 <Link
                   className={styles.heroAction}
                   href={editorialHref("/projects", contentDebug)}
+                  prefetch={false}
                 >
                   {homeCopy.hero.primaryActionLabel} <Arrow />
                 </Link>
@@ -387,7 +396,10 @@ function HomeRoute({ content, contentDebug }: EditorialRouteProps) {
                         {lead.category} · {lead.period}
                       </p>
                       <h2>
-                        <Link href={editorialHref(`/projects/${lead.id}`, contentDebug)}>
+                        <Link
+                          href={editorialHref(`/projects/${lead.id}`, contentDebug)}
+                          prefetch={false}
+                        >
                           {lead.title}
                         </Link>
                       </h2>
@@ -406,6 +418,7 @@ function HomeRoute({ content, contentDebug }: EditorialRouteProps) {
                       )}
                       className={styles.leadVisualLink}
                       href={editorialHref(`/projects/${lead.id}`, contentDebug)}
+                      prefetch={false}
                     >
                       <EditorialImage image={lead.screenshot} priority />
                       <span>{homeCopy.lead.actionLabel} <Arrow /></span>
@@ -457,7 +470,10 @@ function HomeRoute({ content, contentDebug }: EditorialRouteProps) {
                   <p className={styles.sidebarLabel}>{sharedCopy.journey.title}</p>
                   <h2>{content.journeyNarrative.currentPosition.title}</h2>
                   <p>{content.journeyNarrative.currentPosition.body}</p>
-                  <Link href={editorialHref("/journey", contentDebug)}>
+                  <Link
+                    href={editorialHref("/journey", contentDebug)}
+                    prefetch={false}
+                  >
                     {homeCopy.current.actionLabel} <Arrow />
                   </Link>
                   <div className={styles.sidebarRule} />
@@ -610,7 +626,10 @@ function ProjectDetailRoute({ content, contentDebug, project }: EditorialRoutePr
     <article className={styles.caseStudy}>
       <header className={styles.caseHero}>
         <div className={styles.caseMetaRail}>
-          <Link href={editorialHref("/projects", contentDebug)}>
+          <Link
+            href={editorialHref("/projects", contentDebug)}
+            prefetch={false}
+          >
             ← {copy.backLabel}
           </Link>
           <span>{project.category}</span>


