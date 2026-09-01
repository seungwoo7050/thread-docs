## `refactor(classic-home): 홈 renderer를 독립 모듈로 이동`

diff --git a/src/app/page.tsx b/src/app/page.tsx
index b98713f..9e92678 100644
--- a/src/app/page.tsx
+++ b/src/app/page.tsx
@@ -1,5 +1,5 @@
 import type { Metadata } from "next";
-import { ClassicHomeRoute } from "@/designs/classic/home-route";
+import ClassicHomeRoute from "@/designs/classic/home-route";
 import DesignHomeRoute from "@/designs/design/home-route";
 import { hasDedicatedRouteRenderer, renderDesignRoute } from "@/designs/registry";
 import { getPortfolioContent, type RouteSearchParams } from "@/lib/portfolio";
@@ -40,7 +40,14 @@ export default async function Home({ searchParams }: HomePageProps) {
   }
 
   if (activeTemplate === "classic") {
-    return <ClassicHomeRoute content={viewModel} contentDebug={contentDebug} />;
+    return (
+      <ClassicHomeRoute
+        content={viewModel}
+        contentDebug={contentDebug}
+        currentPath="/"
+        route="home"
+      />
+    );
   }
 
   return (
diff --git a/src/components/portfolio/home-contact-preview.tsx b/src/components/portfolio/home-contact-preview.tsx
deleted file mode 100644
index 66f937b..0000000
--- a/src/components/portfolio/home-contact-preview.tsx
+++ /dev/null
@@ -1,51 +0,0 @@
-import { ArrowRightIcon } from "@/components/icons";
-import {
-  type HomeTemplateId,
-} from "@/lib/portfolio";
-import type { HomeViewModel } from "@/lib/portfolio/view-models";
-import { ContentHint } from "./content-hint";
-import { ContentLinkView } from "./content-link";
-
-export function HomeContactPreview({
-  activeTemplate,
-  content,
-  contentDebug,
-}: {
-  activeTemplate: HomeTemplateId;
-  content: HomeViewModel;
-  contentDebug: boolean;
-}) {
-  const preferredLinks = content.preferredContactLinks;
-  const copy = content.presentation.home.shared.contact;
-
-  return (
-    <section>
-      <div className="mx-auto grid max-w-6xl gap-8 px-5 py-20 sm:px-8 lg:grid-cols-[1fr_auto] lg:items-center">
-        <div>
-          <ContentHint
-            enabled={contentDebug}
-            path="src/content/presentation.json > home.shared.contact + src/content/contact.json > availability"
-          />
-          <h2 className="text-3xl font-semibold text-foreground">{copy.title}</h2>
-          <p className="mt-4 max-w-2xl text-sm leading-6 text-muted md:text-base">
-            {content.contact.availability}
-          </p>
-        </div>
-        <div className="flex flex-wrap gap-3">
-          {preferredLinks.map((link) => (
-            <ContentLinkView
-              className="inline-flex h-10 items-center gap-2 rounded-md border border-line bg-surface px-4 text-sm font-semibold text-muted transition hover:border-accent hover:text-foreground"
-              contentDebug={contentDebug}
-              homeTemplate={activeTemplate}
-              key={link.id ?? link.href}
-              link={link}
-            >
-              {link.label}
-              <ArrowRightIcon className="-rotate-45" />
-            </ContentLinkView>
-          ))}
-        </div>
-      </div>
-    </section>
-  );
-}
diff --git a/src/components/portfolio/home-journey-section.tsx b/src/components/portfolio/home-journey-section.tsx
deleted file mode 100644
index cb0fadc..0000000
--- a/src/components/portfolio/home-journey-section.tsx
+++ /dev/null
@@ -1,36 +0,0 @@
-import type { HomeTemplateId, PortfolioContent } from "@/lib/portfolio";
-import { JourneyList } from "./journey-list";
-import { SectionHeading } from "./section-heading";
-
-export function HomeJourneySection({
-  activeTemplate,
-  content,
-  contentDebug,
-}: {
-  activeTemplate: HomeTemplateId;
-  content: PortfolioContent;
-  contentDebug: boolean;
-}) {
-  const copy = content.presentation.home.shared.journey;
-
-  return (
-    <section className="border-b border-line">
-      <div className="mx-auto grid max-w-6xl gap-9 px-5 py-20 sm:px-8">
-        <SectionHeading
-          body={copy.body}
-          contentDebug={contentDebug}
-          contentHint="src/content/presentation.json > home.shared.journey"
-          title={copy.title}
-        />
-        <JourneyList
-          animated
-          caseStudyLabel={content.presentation.ui.journeyCaseStudyLabel}
-          contentDebug={contentDebug}
-          homeTemplate={activeTemplate}
-          items={content.journey}
-          variant="paired-centerline"
-        />
-      </div>
-    </section>
-  );
-}
diff --git a/src/components/portfolio/selected-stack-section.tsx b/src/components/portfolio/selected-stack-section.tsx
deleted file mode 100644
index f5b2c1a..0000000
--- a/src/components/portfolio/selected-stack-section.tsx
+++ /dev/null
@@ -1,57 +0,0 @@
-import type { PortfolioContent } from "@/lib/portfolio";
-import { ContentHint } from "./content-hint";
-import { Reveal } from "./reveal";
-import { SectionHeading } from "./section-heading";
-import { StackList } from "./stack-list";
-import { TechMarquee } from "./tech-marquee";
-
-export function SelectedStackSection({
-  content,
-  contentDebug,
-}: {
-  content: PortfolioContent;
-  contentDebug: boolean;
-}) {
-  const stackIds = new Set(content.skills.groups.flatMap((group) => group.items));
-  const visibleStackItems = content.techStack.filter((item) => stackIds.has(item.id));
-  const copy = content.presentation.home.shared.stack;
-
-  return (
-    <section className="border-b border-line bg-background-soft">
-      <div className="mx-auto grid max-w-6xl gap-10 px-5 py-20 sm:px-8">
-        <div className="grid gap-9 lg:grid-cols-[0.8fr_1.2fr]">
-          <SectionHeading
-            body={copy.body}
-            contentDebug={contentDebug}
-            contentHint="src/content/presentation.json > home.shared.stack"
-            title={copy.title}
-          />
-          <div className="grid gap-6">
-            <TechMarquee
-              ariaLabel={content.presentation.ui.techMarqueeAriaLabel}
-              items={visibleStackItems}
-            />
-            <div className="grid overflow-hidden rounded-lg border border-line bg-surface sm:grid-cols-2">
-              {content.skills.groups.map((group, index) => (
-                <Reveal delay={index * 80} key={group.title}>
-                  <div className="h-full border-b border-line p-5 sm:border-r">
-                    <ContentHint
-                      enabled={contentDebug}
-                      path={`src/content/skills.json > groups[title=${group.title}]`}
-                    />
-                    <h3 className="text-sm font-semibold text-foreground">
-                      {group.title}
-                    </h3>
-                    <div className="mt-4">
-                      <StackList items={group.items} />
-                    </div>
-                  </div>
-                </Reveal>
-              ))}
-            </div>
-          </div>
-        </div>
-      </div>
-    </section>
-  );
-}
diff --git a/src/components/portfolio/technical-focus-section.tsx b/src/components/portfolio/technical-focus-section.tsx
deleted file mode 100644
index 2e54700..0000000
--- a/src/components/portfolio/technical-focus-section.tsx
+++ /dev/null
@@ -1,45 +0,0 @@
-import type { PortfolioContent } from "@/lib/portfolio";
-import { ContentHint } from "./content-hint";
-import { Reveal } from "./reveal";
-import { SectionHeading } from "./section-heading";
-
-export function TechnicalFocusSection({
-  content,
-  contentDebug,
-}: {
-  content: PortfolioContent;
-  contentDebug: boolean;
-}) {
-  const copy = content.presentation.home.shared.technicalFocus;
-
-  return (
-    <section className="border-b border-line">
-      <div className="mx-auto grid max-w-6xl gap-9 px-5 py-20 sm:px-8">
-        <SectionHeading
-          body={copy.body}
-          contentDebug={contentDebug}
-          contentHint="src/content/presentation.json > home.shared.technicalFocus"
-          title={copy.title}
-        />
-        <div className="grid gap-4 sm:grid-cols-2">
-          {content.skills.focusAreas.map((area, index) => (
-            <Reveal delay={index * 70} key={area.title}>
-              <article
-                className="motion-card h-full rounded-lg border border-line bg-surface p-5 transition duration-300 hover:border-accent/45 hover:bg-surface-hover"
-              >
-                <ContentHint
-                  enabled={contentDebug}
-                  path={`src/content/skills.json > focusAreas[title=${area.title}]`}
-                />
-                <h3 className="text-base font-semibold text-foreground">
-                  {area.title}
-                </h3>
-                <p className="mt-3 text-sm leading-6 text-muted">{area.body}</p>
-              </article>
-            </Reveal>
-          ))}
-        </div>
-      </div>
-    </section>
-  );
-}
diff --git a/src/components/portfolio/work-map-section.tsx b/src/components/portfolio/work-map-section.tsx
deleted file mode 100644
index c153096..0000000
--- a/src/components/portfolio/work-map-section.tsx
+++ /dev/null
@@ -1,65 +0,0 @@
-import type { HomeViewModel } from "@/lib/portfolio/view-models";
-import { Reveal } from "./reveal";
-import { SectionHeading } from "./section-heading";
-
-export function getWorkMapStats(content: HomeViewModel) {
-  return {
-    curriculumCount: content.metricValues.curriculumCount ?? 0,
-    productCount: content.metricValues.productCount ?? 0,
-    reliabilityCount: content.metricValues.reliabilityCount ?? 0,
-  };
-}
-
-export function WorkMapSection({
-  content,
-  contentDebug,
-}: {
-  content: HomeViewModel;
-  contentDebug: boolean;
-}) {
-  const stats = getWorkMapStats(content);
-  const copy = content.presentation.home.shared.workMap;
-
-  return (
-    <section className="border-b border-line">
-      <div className="mx-auto grid max-w-6xl gap-8 px-5 py-16 sm:px-8 lg:grid-cols-[0.8fr_1.2fr]">
-        <SectionHeading
-          body={copy.body}
-          contentDebug={contentDebug}
-          contentHint="src/content/presentation.json > home.shared.workMap"
-          title={copy.title}
-        />
-        <div className="grid gap-4 sm:grid-cols-3">
-          {copy.cards.map((card) => (
-            <ArchiveStat
-              body={card.body}
-              count={stats[card.countKey]}
-              key={card.id}
-              label={card.label}
-            />
-          ))}
-        </div>
-      </div>
-    </section>
-  );
-}
-
-function ArchiveStat({
-  body,
-  count,
-  label,
-}: {
-  body: string;
-  count: number;
-  label: string;
-}) {
-  return (
-    <Reveal>
-      <article className="h-full rounded-lg border border-line bg-surface p-5">
-        <p className="text-sm font-semibold text-muted">{label}</p>
-        <p className="mt-4 text-5xl font-semibold text-foreground">{count}</p>
-        <p className="mt-4 text-sm leading-6 text-muted">{body}</p>
-      </article>
-    </Reveal>
-  );
-}
diff --git a/src/designs/classic/home-route.tsx b/src/designs/classic/home-route.tsx
index d79f6c4..6dba64b 100644
--- a/src/designs/classic/home-route.tsx
+++ b/src/designs/classic/home-route.tsx
@@ -1,129 +1,153 @@
+import Link from "next/link";
 import { ArrowRightIcon } from "@/components/icons";
 import { AnimatedTerminal } from "@/components/portfolio/animated-terminal";
 import { ContentHint } from "@/components/portfolio/content-hint";
 import { ContentLinkView } from "@/components/portfolio/content-link";
-import { HomeContactPreview } from "@/components/portfolio/home-contact-preview";
-import { HomeJourneySection } from "@/components/portfolio/home-journey-section";
+import { JourneyList } from "@/components/portfolio/journey-list";
 import { ProfilePhoto } from "@/components/portfolio/profile-photo";
 import { ProjectCard } from "@/components/portfolio/project-card";
 import { Reveal } from "@/components/portfolio/reveal";
 import { SectionHeading } from "@/components/portfolio/section-heading";
-import { SelectedStackSection } from "@/components/portfolio/selected-stack-section";
 import { PageShell } from "@/components/portfolio/site-shell";
-import { TechnicalFocusSection } from "@/components/portfolio/technical-focus-section";
-import { WorkMapSection } from "@/components/portfolio/work-map-section";
+import { StackList } from "@/components/portfolio/stack-list";
+import { TechMarquee } from "@/components/portfolio/tech-marquee";
 import {
+  type HomeViewModel,
+} from "@/lib/portfolio/view-models";
+import {
+  getTemplateHref,
+  type HomeSectionId,
   type HomeTemplateId,
   type PortfolioProject,
 } from "@/lib/portfolio";
-import type { HomeViewModel } from "@/lib/portfolio/view-models";
+import type { PreparedDesignRouteProps as DesignRouteProps } from "@/designs/shell-props";
+import { createDesignShellProps } from "@/designs/shell-props";
+
+export default function HomeRoute({
+  content,
+  contentDebug,
+  currentPath,
+}: DesignRouteProps) {
+  if (content.route !== "home") return null;
+
+  const activeTemplate = "classic";
+  return (
+    <HomeView
+      activeTemplate={activeTemplate}
+      content={content}
+      contentDebug={contentDebug}
+      shellProps={createDesignShellProps(
+        content,
+        contentDebug,
+        currentPath,
+        activeTemplate,
+      )}
+    />
+  );
+}
 
-export function ClassicHomeRoute({
+function HomeView({
+  activeTemplate,
   content,
   contentDebug,
+  shellProps,
 }: {
+  activeTemplate: HomeTemplateId;
   content: HomeViewModel;
   contentDebug: boolean;
+  shellProps: ReturnType<typeof createDesignShellProps>;
 }) {
-  const activeTemplate: HomeTemplateId = "classic";
   const featuredProjects = content.featuredProjects;
-
+  const sections = content.presentation.home.classic.sections;
   return (
-    <PageShell
-      contentDebug={contentDebug}
-      homeTemplate={activeTemplate}
-      profile={content.profile}
-      site={content.site}
-      ui={content.presentation.ui}
-      templateSwitcher={{
-        activeId: activeTemplate,
-        contentDebug,
-        currentPath: "/",
-        templates: content.presentation.templates,
-      }}
-    >
+    <PageShell {...shellProps}>
       <ClassicHeroSection
         activeTemplate={activeTemplate}
         content={content}
         contentDebug={contentDebug}
       />
-      {content.presentation.home.classic.sections.includes("featured") ? (
-        <ClassicFeaturedProjectsSection
-          activeTemplate={activeTemplate}
-          content={content}
-          contentDebug={contentDebug}
-          projects={featuredProjects}
-        />
-      ) : null}
-      {content.presentation.home.classic.sections.includes("workMap") ? (
-        <WorkMapSection content={content} contentDebug={contentDebug} />
-      ) : null}
-      {content.presentation.home.classic.sections.includes("technicalFocus") ? (
-        <TechnicalFocusSection content={content} contentDebug={contentDebug} />
-      ) : null}
-      {content.presentation.home.classic.sections.includes("stack") ? (
-        <SelectedStackSection content={content} contentDebug={contentDebug} />
-      ) : null}
-      {content.presentation.home.classic.sections.includes("journey") ? (
-        <HomeJourneySection
-          activeTemplate={activeTemplate}
-          content={content}
-          contentDebug={contentDebug}
-        />
-      ) : null}
-      {content.presentation.home.classic.sections.includes("contact") ? (
-        <HomeContactPreview
+      {sections.map((sectionId) => (
+        <HomeSection
           activeTemplate={activeTemplate}
           content={content}
           contentDebug={contentDebug}
+          featuredProjects={featuredProjects}
+          key={sectionId}
+          sectionId={sectionId}
         />
-      ) : null}
+      ))}
     </PageShell>
   );
 }
 
-function ClassicFeaturedProjectsSection({
+function HomeSection({
   activeTemplate,
   content,
   contentDebug,
-  projects,
+  featuredProjects,
+  sectionId,
 }: {
   activeTemplate: HomeTemplateId;
   content: HomeViewModel;
   contentDebug: boolean;
-  projects: PortfolioProject[];
+  featuredProjects: PortfolioProject[];
+  sectionId: HomeSectionId;
 }) {
-  const copy = content.presentation.home.classic.featured;
+  if (sectionId === "featured") {
+    return (
+      <ClassicFeaturedProjectsSection
+        activeTemplate={activeTemplate}
+        content={content}
+        contentDebug={contentDebug}
+        projects={featuredProjects}
+      />
+    );
+  }
 
-  return (
-    <section className="border-b border-line bg-background-soft">
-      <div className="mx-auto grid max-w-6xl gap-6 px-5 py-10 sm:px-8 md:py-12">
-        <div className="flex flex-col gap-5 sm:flex-row sm:items-end sm:justify-between">
-          <SectionHeading
-            body={copy.body}
-            contentDebug={contentDebug}
-            contentHint="src/content/presentation.json > home.classic.featured"
-            title={copy.title}
-          />
-        </div>
-        <div className="grid gap-5">
-          {projects.slice(0, 1).map((project) => (
-            <Reveal key={project.id}>
-              <ProjectCard
-                contentDebug={contentDebug}
-                homeTemplate={activeTemplate}
-                priority
-                project={project}
-                variant="featured"
-              />
-            </Reveal>
-          ))}
-        </div>
-      </div>
-    </section>
-  );
+  if (sectionId === "workMap") {
+    return <WorkMapSection content={content} contentDebug={contentDebug} />;
+  }
+
+  if (sectionId === "technicalFocus") {
+    return <TechnicalFocusSection content={content} contentDebug={contentDebug} />;
+  }
+
+  if (sectionId === "stack") {
+    return <SelectedStackSection content={content} contentDebug={contentDebug} />;
+  }
+
+  if (sectionId === "journey") {
+    return (
+      <JourneySection
+        activeTemplate={activeTemplate}
+        content={content}
+        contentDebug={contentDebug}
+      />
+    );
+  }
+
+  if (sectionId === "contact") {
+    return (
+      <ContactPreview
+        activeTemplate={activeTemplate}
+        content={content}
+        contentDebug={contentDebug}
+      />
+    );
+  }
+
+  return null;
+}
+
+function getWorkMapStats(content: HomeViewModel) {
+  return {
+    curriculumCount: content.metricValues.curriculumCount ?? 0,
+    productCount: content.metricValues.productCount ?? 0,
+    reliabilityCount: content.metricValues.reliabilityCount ?? 0,
+  };
 }
 
+
 function ClassicHeroSection({
   activeTemplate,
   content,
@@ -134,6 +158,7 @@ function ClassicHeroSection({
   contentDebug: boolean;
 }) {
   const { profile } = content;
+  const copy = content.presentation.home.classic.hero;
   const links = content.heroLinks;
 
   return (
@@ -160,6 +185,15 @@ function ClassicHeroSection({
             {profile.summary}
           </p>
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
@@ -191,3 +225,285 @@ function ClassicHeroSection({
     </section>
   );
 }
+
+
+function ClassicFeaturedProjectsSection({
+  activeTemplate,
+  content,
+  contentDebug,
+  projects,
+}: {
+  activeTemplate: HomeTemplateId;
+  content: HomeViewModel;
+  contentDebug: boolean;
+  projects: PortfolioProject[];
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
+          <Link
+            className="inline-flex h-10 w-fit items-center gap-2 rounded-md border border-line bg-surface px-4 text-sm font-semibold text-muted transition hover:border-accent hover:text-foreground"
+            href={getTemplateHref("/projects", activeTemplate, {
+              contentDebug,
+            })}
+          >
+            {copy.actionLabel}
+            <ArrowRightIcon />
+          </Link>
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
+function WorkMapSection({
+  content,
+  contentDebug,
+}: {
+  content: HomeViewModel;
+  contentDebug: boolean;
+}) {
+  const stats = getWorkMapStats(content);
+  const copy = content.presentation.home.shared.workMap;
+
+  return (
+    <section className="border-b border-line">
+      <div className="mx-auto grid max-w-6xl gap-8 px-5 py-16 sm:px-8 lg:grid-cols-[0.8fr_1.2fr]">
+        <SectionHeading
+          body={copy.body}
+          contentDebug={contentDebug}
+          contentHint="src/content/presentation.json > home.shared.workMap"
+          title={copy.title}
+        />
+        <div className="grid gap-4 sm:grid-cols-3">
+          {copy.cards.map((card) => (
+            <ArchiveStat
+              body={card.body}
+              count={stats[card.countKey]}
+              key={card.id}
+              label={card.label}
+            />
+          ))}
+        </div>
+      </div>
+    </section>
+  );
+}
+
+function ArchiveStat({
+  body,
+  count,
+  label,
+}: {
+  body: string;
+  count: number;
+  label: string;
+}) {
+  return (
+    <Reveal>
+      <article className="h-full rounded-lg border border-line bg-surface p-5">
+        <p className="text-sm font-semibold text-muted">{label}</p>
+        <p className="mt-4 text-5xl font-semibold text-foreground">{count}</p>
+        <p className="mt-4 text-sm leading-6 text-muted">{body}</p>
+      </article>
+    </Reveal>
+  );
+}
+
+function TechnicalFocusSection({
+  content,
+  contentDebug,
+}: {
+  content: HomeViewModel;
+  contentDebug: boolean;
+}) {
+  const copy = content.presentation.home.shared.technicalFocus;
+
+  return (
+    <section className="border-b border-line">
+      <div className="mx-auto grid max-w-6xl gap-9 px-5 py-20 sm:px-8">
+        <SectionHeading
+          body={copy.body}
+          contentDebug={contentDebug}
+          contentHint="src/content/presentation.json > home.shared.technicalFocus"
+          title={copy.title}
+        />
+        <div className="grid gap-4 sm:grid-cols-2">
+          {content.skills.focusAreas.map((area, index) => (
+            <Reveal delay={index * 70} key={area.title}>
+              <article
+                className="motion-card h-full rounded-lg border border-line bg-surface p-5 transition duration-300 hover:border-accent/45 hover:bg-surface-hover"
+              >
+                <ContentHint
+                  enabled={contentDebug}
+                  path={`src/content/skills.json > focusAreas[title=${area.title}]`}
+                />
+                <h3 className="text-base font-semibold text-foreground">
+                  {area.title}
+                </h3>
+                <p className="mt-3 text-sm leading-6 text-muted">{area.body}</p>
+              </article>
+            </Reveal>
+          ))}
+        </div>
+      </div>
+    </section>
+  );
+}
+
+function SelectedStackSection({
+  content,
+  contentDebug,
+}: {
+  content: HomeViewModel;
+  contentDebug: boolean;
+}) {
+  const stackIds = new Set(content.skills.groups.flatMap((group) => group.items));
+  const visibleStackItems = content.techStack.filter((item) => stackIds.has(item.id));
+  const copy = content.presentation.home.shared.stack;
+
+  return (
+    <section className="border-b border-line bg-background-soft">
+      <div className="mx-auto grid max-w-6xl gap-10 px-5 py-20 sm:px-8">
+        <div className="grid gap-9 lg:grid-cols-[0.8fr_1.2fr]">
+          <SectionHeading
+            body={copy.body}
+            contentDebug={contentDebug}
+            contentHint="src/content/presentation.json > home.shared.stack"
+            title={copy.title}
+          />
+          <div className="grid gap-6">
+            <TechMarquee
+              ariaLabel={content.presentation.ui.techMarqueeAriaLabel}
+              items={visibleStackItems}
+            />
+            <div className="grid overflow-hidden rounded-lg border border-line bg-surface sm:grid-cols-2">
+              {content.skills.groups.map((group, index) => (
+                <Reveal delay={index * 80} key={group.title}>
+                  <div className="h-full border-b border-line p-5 sm:border-r">
+                    <ContentHint
+                      enabled={contentDebug}
+                      path={`src/content/skills.json > groups[title=${group.title}]`}
+                    />
+                    <h3 className="text-sm font-semibold text-foreground">
+                      {group.title}
+                    </h3>
+                    <div className="mt-4">
+                      <StackList items={group.items} />
+                    </div>
+                  </div>
+                </Reveal>
+              ))}
+            </div>
+          </div>
+        </div>
+      </div>
+    </section>
+  );
+}
+
+function JourneySection({
+  activeTemplate,
+  content,
+  contentDebug,
+}: {
+  activeTemplate: HomeTemplateId;
+  content: HomeViewModel;
+  contentDebug: boolean;
+}) {
+  const copy = content.presentation.home.shared.journey;
+
+  return (
+    <section className="border-b border-line">
+      <div className="mx-auto grid max-w-6xl gap-9 px-5 py-20 sm:px-8">
+        <SectionHeading
+          body={copy.body}
+          contentDebug={contentDebug}
+          contentHint="src/content/presentation.json > home.shared.journey"
+          title={copy.title}
+        />
+        <JourneyList
+          animated
+          caseStudyLabel={content.presentation.ui.journeyCaseStudyLabel}
+          contentDebug={contentDebug}
+          homeTemplate={activeTemplate}
+          items={content.journey}
+          variant="paired-centerline"
+        />
+      </div>
+    </section>
+  );
+}
+
+function ContactPreview({
+  activeTemplate,
+  content,
+  contentDebug,
+}: {
+  activeTemplate: HomeTemplateId;
+  content: HomeViewModel;
+  contentDebug: boolean;
+}) {
+  const preferredLinks = content.preferredContactLinks;
+  const copy = content.presentation.home.shared.contact;
+
+  return (
+    <section>
+      <div className="mx-auto grid max-w-6xl gap-8 px-5 py-20 sm:px-8 lg:grid-cols-[1fr_auto] lg:items-center">
+        <div>
+          <ContentHint
+            enabled={contentDebug}
+            path="src/content/presentation.json > home.shared.contact + src/content/contact.json > availability"
+          />
+          <h2 className="text-3xl font-semibold text-foreground">{copy.title}</h2>
+          <p className="mt-4 max-w-2xl text-sm leading-6 text-muted md:text-base">
+            {content.contact.availability}
+          </p>
+        </div>
+        <div className="flex flex-wrap gap-3">
+          {preferredLinks.map((link) => (
+            <ContentLinkView
+              className="inline-flex h-10 items-center gap-2 rounded-md border border-line bg-surface px-4 text-sm font-semibold text-muted transition hover:border-accent hover:text-foreground"
+              contentDebug={contentDebug}
+              homeTemplate={activeTemplate}
+              key={link.id ?? link.href}
+              link={link}
+            >
+              {link.label}
+              <ArrowRightIcon className="-rotate-45" />
+            </ContentLinkView>
+          ))}
+          <Link
+            className="inline-flex h-10 items-center gap-2 rounded-md border border-accent bg-accent px-4 text-sm font-semibold text-background transition hover:bg-accent-strong"
+            href={getTemplateHref("/contact", activeTemplate, { contentDebug })}
+          >
+            {copy.actionLabel}
+            <ArrowRightIcon />
+          </Link>
+        </div>
+      </div>
+    </section>
+  );
+}


