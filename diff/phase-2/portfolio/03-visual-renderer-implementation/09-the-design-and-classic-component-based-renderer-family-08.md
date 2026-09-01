## `refactor(classic-about): 소개 renderer를 독립 모듈로 이동`

diff --git a/src/app/about/page.tsx b/src/app/about/page.tsx
index 2dcd0a2..2131105 100644
--- a/src/app/about/page.tsx
+++ b/src/app/about/page.tsx
@@ -1,28 +1,15 @@
 import type { Metadata } from "next";
-import Link from "next/link";
 import { notFound } from "next/navigation";
-import { ArrowRightIcon } from "@/components/icons";
-import { ContentHint } from "@/components/portfolio/content-hint";
-import { JourneyList } from "@/components/portfolio/journey-list";
-import { ProfilePhoto } from "@/components/portfolio/profile-photo";
-import { SectionHeading } from "@/components/portfolio/section-heading";
-import { PageShell } from "@/components/portfolio/site-shell";
-import { StackList } from "@/components/portfolio/stack-list";
+import ClassicAboutRoute from "@/designs/classic/about-route";
 import DesignAboutRoute from "@/designs/design/about-route";
 import { hasDedicatedRouteRenderer, renderDesignRoute } from "@/designs/registry";
 import {
   getPortfolioContent,
-  getTemplateHref,
   isSitePageEnabled,
-  type HomeTemplateId,
   type RouteSearchParams,
 } from "@/lib/portfolio";
 import { resolvePortfolioPageContext } from "@/lib/portfolio/page-context";
-import {
-  createAboutViewModel,
-  type AboutViewModel,
-  type CurationCategoryViewModel,
-} from "@/lib/portfolio/view-models";
+import { createAboutViewModel } from "@/lib/portfolio/view-models";
 import { createRouteMetadata } from "@/lib/site-metadata";
 
 export function generateMetadata(): Metadata {
@@ -44,7 +31,7 @@ export default async function AboutPage({
 }) {
   const contentSource = getPortfolioContent();
   if (!isSitePageEnabled("about", contentSource)) notFound();
-  const { activeTemplate, contentDebug, shellProps } =
+  const { activeTemplate, contentDebug } =
     await resolvePortfolioPageContext({
       content: contentSource,
       currentPath: "/about",
@@ -61,302 +48,14 @@ export default async function AboutPage({
     });
   }
 
-  if (activeTemplate === "design") {
-    return (
-      <DesignAboutRoute
-        content={content}
-        contentDebug={contentDebug}
-        currentPath="/about"
-        route="about"
-      />
-    );
-  }
-
-  const pageCopy = content.presentation.pages.about;
-
-  return (
-    <PageShell {...shellProps}>
-      <section className="border-b border-line">
-        <div className="mx-auto grid max-w-6xl gap-10 px-5 py-20 sm:px-8 lg:grid-cols-[1fr_20rem] lg:items-center">
-          <div>
-            <ContentHint
-              enabled={contentDebug}
-              path="src/content/profile.json > name/handle/summary/photo"
-            />
-            <p className="text-sm font-medium text-muted">
-              {content.profile.name} · {content.profile.handle}
-            </p>
-            <h1 className="mt-5 max-w-3xl text-5xl font-semibold leading-tight text-foreground md:text-6xl">
-              {pageCopy.hero.title}
-            </h1>
-            <p className="mt-6 max-w-2xl text-base leading-7 text-muted">
-              {content.profile.summary}
-            </p>
-          </div>
-          {content.profile.photo ? (
-            <ProfilePhoto photo={content.profile.photo} />
-          ) : null}
-        </div>
-      </section>
-      <section className="border-b border-line">
-        <div className="mx-auto grid max-w-6xl gap-9 px-5 py-16 sm:px-8 lg:grid-cols-[0.8fr_1.2fr]">
-          <SectionHeading
-            contentDebug={contentDebug}
-            contentHint="src/content/presentation.json > pages.about.principles"
-            title={pageCopy.principles.title}
-          />
-          <div className="grid gap-4">
-            {content.profile.principles.map((principle) => (
-              <article className="rounded-lg border border-line bg-surface p-5" key={principle.title}>
-                <ContentHint
-                  enabled={contentDebug}
-                  path={`src/content/profile.json > principles[title=${principle.title}]`}
-                />
-                <h2 className="font-semibold text-foreground">{principle.title}</h2>
-                <p className="mt-3 text-sm leading-6 text-muted">
-                  {principle.body}
-                </p>
-              </article>
-            ))}
-          </div>
-        </div>
-      </section>
-      <section className="border-b border-line bg-background-soft">
-        <div className="mx-auto grid max-w-6xl gap-9 px-5 py-16 sm:px-8 lg:grid-cols-[0.8fr_1.2fr]">
-          <SectionHeading
-            contentDebug={contentDebug}
-            contentHint="src/content/presentation.json > pages.about.journey + src/content/journey.json > journey[]"
-            title={pageCopy.journey.title}
-          />
-          <JourneyList
-            caseStudyLabel={content.presentation.ui.journeyCaseStudyLabel}
-            contentDebug={contentDebug}
-            homeTemplate={activeTemplate}
-            items={content.journey}
-          />
-        </div>
-      </section>
-      <section className="border-b border-line">
-        <div className="mx-auto grid max-w-6xl gap-9 px-5 py-16 sm:px-8 lg:grid-cols-[0.8fr_1.2fr]">
-          <SectionHeading
-            contentDebug={contentDebug}
-            contentHint="src/content/presentation.json > pages.about.skills"
-            title={pageCopy.skills.title}
-          />
-          <div className="grid gap-8">
-            <div className="grid gap-4 sm:grid-cols-2">
-              {content.skills.focusAreas.map((area) => (
-                <article
-                  className="rounded-lg border border-line bg-surface p-5"
-                  key={area.title}
-                >
-                  <ContentHint
-                    enabled={contentDebug}
-                    path={`src/content/skills.json > focusAreas[title=${area.title}]`}
-                  />
-                  <h2 className="text-sm font-semibold text-foreground">
-                    {area.title}
-                  </h2>
-                  <p className="mt-3 text-sm leading-6 text-muted">
-                    {area.body}
-                  </p>
-                </article>
-              ))}
-            </div>
-            <div className="grid gap-4 sm:grid-cols-2">
-              {content.skills.groups.map((group) => (
-                <article
-                  className="rounded-lg border border-line bg-surface p-5"
-                  key={group.title}
-                >
-                  <ContentHint
-                    enabled={contentDebug}
-                    path={`src/content/skills.json > groups[title=${group.title}]`}
-                  />
-                  <h2 className="text-sm font-semibold text-foreground">
-                    {group.title}
-                  </h2>
-                  <div className="mt-4">
-                    <StackList items={group.items} />
-                  </div>
-                </article>
-              ))}
-            </div>
-          </div>
-        </div>
-      </section>
-      <section className="border-b border-line bg-background-soft">
-        <div className="mx-auto grid max-w-6xl gap-9 px-5 py-16 sm:px-8 lg:grid-cols-[0.8fr_1.2fr]">
-          <SectionHeading
-            contentDebug={contentDebug}
-            contentHint="src/content/experience.json"
-            title={pageCopy.journey.title}
-          />
-          <ol className="grid gap-4">
-            {content.experience.map((item) => (
-              <li
-                className="rounded-lg border border-line bg-surface p-5"
-                key={`${item.period}-${item.title}`}
-              >
-                <p className="text-xs font-semibold uppercase tracking-[0.08em] text-accent">
-                  {item.period}
-                </p>
-                <h2 className="mt-3 text-base font-semibold text-foreground">
-                  {item.title}
-                </h2>
-                <p className="mt-3 text-sm leading-6 text-muted">{item.body}</p>
-              </li>
-            ))}
-          </ol>
-        </div>
-      </section>
-      {isSitePageEnabled("curation", content) ? (
-        <CurationSection
-          content={content}
-          contentDebug={contentDebug}
-          homeTemplate={activeTemplate}
-        />
-      ) : null}
-    </PageShell>
-  );
-}
-
-function CurationSection({
-  content,
-  contentDebug,
-  homeTemplate,
-}: {
-  content: AboutViewModel;
-  contentDebug: boolean;
-  homeTemplate: HomeTemplateId;
-}) {
-  const pageCopy = content.presentation.pages.about.curation;
-  const data = content.curation;
-
-  return (
-    <section aria-label={pageCopy.title} className="bg-background-soft">
-      <div className="mx-auto grid max-w-6xl gap-9 px-5 py-16 sm:px-8">
-        <SectionHeading
-          body={pageCopy.body}
-          contentDebug={contentDebug}
-          contentHint="src/content/curation.json > intro/criteria + src/content/presentation.json > pages.about.curation"
-          title={pageCopy.title}
-        />
-        <p className="max-w-3xl text-sm leading-6 text-muted md:text-base md:leading-7">
-          {data.intro}
-        </p>
-        <div className="grid gap-9 lg:grid-cols-[0.8fr_1.2fr]">
-          <h3 className="text-xl font-semibold text-foreground">
-            {pageCopy.criteriaTitle}
-          </h3>
-          <div className="grid gap-4 sm:grid-cols-2">
-            {data.criteria.items.map((item) => (
-              <article
-                className="rounded-lg border border-line bg-surface p-5"
-                key={item.title}
-              >
-                <ContentHint
-                  enabled={contentDebug}
-                  path={`src/content/curation.json > criteria.items[title=${item.title}]`}
-                />
-                <h4 className="text-sm font-semibold text-foreground">
-                  {item.title}
-                </h4>
-                <p className="mt-3 text-sm leading-6 text-muted">{item.body}</p>
-              </article>
-            ))}
-          </div>
-        </div>
-        <div className="grid gap-9 lg:grid-cols-[0.8fr_1.2fr]">
-          <h3 className="text-xl font-semibold text-foreground">
-            {pageCopy.categoriesTitle}
-          </h3>
-          <ul className="grid gap-4">
-            {content.curationCategories.map((category) => (
-              <CurationCategoryCard
-                category={category}
-                contentDebug={contentDebug}
-                homeTemplate={homeTemplate}
-                key={category.id}
-              />
-            ))}
-          </ul>
-        </div>
-        <div className="grid gap-9 lg:grid-cols-[0.8fr_1.2fr]">
-          <div>
-            <h3 className="text-xl font-semibold text-foreground">
-              {pageCopy.omissionsTitle}
-            </h3>
-            <p className="mt-3 text-sm leading-6 text-muted">{data.omissions.body}</p>
-          </div>
-          <ul className="grid gap-3">
-            {data.omissions.items.map((item) => (
-              <li
-                className="rounded-lg border border-line bg-surface p-4"
-                key={item.title}
-              >
-                <h4 className="text-sm font-semibold text-foreground">
-                  {item.title}
-                </h4>
-                <p className="mt-2 text-sm leading-6 text-muted">{item.body}</p>
-              </li>
-            ))}
-          </ul>
-        </div>
-        <div className="grid gap-9 lg:grid-cols-[0.8fr_1.2fr]">
-          <h3 className="text-xl font-semibold text-foreground">
-            {pageCopy.nextReviewTitle}
-          </h3>
-          <div className="rounded-lg border border-line bg-surface p-5">
-            <h4 className="text-sm font-semibold text-foreground">
-              {data.nextReview.title}
-            </h4>
-            <p className="mt-3 text-sm leading-6 text-muted">{data.nextReview.body}</p>
-          </div>
-        </div>
-      </div>
-    </section>
-  );
-}
-
-function CurationCategoryCard({
-  category,
-  contentDebug,
-  homeTemplate,
-}: {
-  category: CurationCategoryViewModel;
-  contentDebug: boolean;
-  homeTemplate: HomeTemplateId;
-}) {
-  const projects = category.projects;
+  const AboutRoute = activeTemplate === "design" ? DesignAboutRoute : ClassicAboutRoute;
 
   return (
-    <li className="rounded-lg border border-line bg-surface p-5">
-      <ContentHint
-        enabled={contentDebug}
-        path={`src/content/curation.json > categories[id=${category.id}]`}
-      />
-      <h4 className="text-base font-semibold text-foreground">{category.label}</h4>
-      <p className="mt-3 text-sm leading-6 text-muted md:leading-7">
-        {category.rationale}
-      </p>
-      {projects.length > 0 ? (
-        <ul className="mt-4 flex flex-wrap gap-2">
-          {projects.map((project) => (
-            <li key={project.id}>
-              <Link
-                className="inline-flex items-center gap-2 rounded-md border border-line bg-surface-soft px-3 py-1.5 text-xs font-semibold text-muted transition hover:border-accent hover:text-foreground"
-                href={getTemplateHref(`/projects/${project.id}`, homeTemplate, {
-                  contentDebug,
-                })}
-              >
-                {project.title}
-                <ArrowRightIcon />
-              </Link>
-            </li>
-          ))}
-        </ul>
-      ) : null}
-    </li>
+    <AboutRoute
+      content={content}
+      contentDebug={contentDebug}
+      currentPath="/about"
+      route="about"
+    />
   );
 }
diff --git a/src/designs/classic/about-route.tsx b/src/designs/classic/about-route.tsx
new file mode 100644
index 0000000..ca0c9a0
--- /dev/null
+++ b/src/designs/classic/about-route.tsx
@@ -0,0 +1,322 @@
+import Link from "next/link";
+import { ArrowRightIcon } from "@/components/icons";
+import { ContentHint } from "@/components/portfolio/content-hint";
+import { JourneyList } from "@/components/portfolio/journey-list";
+import { ProfilePhoto } from "@/components/portfolio/profile-photo";
+import { SectionHeading } from "@/components/portfolio/section-heading";
+import { PageShell } from "@/components/portfolio/site-shell";
+import { StackList } from "@/components/portfolio/stack-list";
+import {
+  type AboutViewModel,
+  type CurationCategoryViewModel,
+} from "@/lib/portfolio/view-models";
+import {
+  getTemplateHref,
+  isSitePageEnabled,
+  type HomeTemplateId,
+} from "@/lib/portfolio";
+import type { PreparedDesignRouteProps as DesignRouteProps } from "@/designs/shell-props";
+import { createDesignShellProps } from "@/designs/shell-props";
+
+export default function AboutRoute({
+  content,
+  contentDebug,
+  currentPath,
+}: DesignRouteProps) {
+  if (content.route !== "about") return null;
+
+  const activeTemplate = "classic";
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
+      <section className="border-b border-line bg-background-soft">
+        <div className="mx-auto grid max-w-6xl gap-9 px-5 py-16 sm:px-8 lg:grid-cols-[0.8fr_1.2fr]">
+          <SectionHeading
+            contentDebug={contentDebug}
+            contentHint="src/content/presentation.json > pages.about.journey + src/content/journey.json > journey[]"
+            title={pageCopy.journey.title}
+          />
+          <JourneyList
+            caseStudyLabel={content.presentation.ui.journeyCaseStudyLabel}
+            contentDebug={contentDebug}
+            homeTemplate={activeTemplate}
+            items={content.journey}
+          />
+        </div>
+      </section>
+      <section className="border-b border-line">
+        <div className="mx-auto grid max-w-6xl gap-9 px-5 py-16 sm:px-8 lg:grid-cols-[0.8fr_1.2fr]">
+          <SectionHeading
+            contentDebug={contentDebug}
+            contentHint="src/content/presentation.json > pages.about.skills"
+            title={pageCopy.skills.title}
+          />
+          <div className="grid gap-8">
+            <div className="grid gap-4 sm:grid-cols-2">
+              {content.skills.focusAreas.map((area) => (
+                <article
+                  className="rounded-lg border border-line bg-surface p-5"
+                  key={area.title}
+                >
+                  <ContentHint
+                    enabled={contentDebug}
+                    path={`src/content/skills.json > focusAreas[title=${area.title}]`}
+                  />
+                  <h2 className="text-sm font-semibold text-foreground">
+                    {area.title}
+                  </h2>
+                  <p className="mt-3 text-sm leading-6 text-muted">
+                    {area.body}
+                  </p>
+                </article>
+              ))}
+            </div>
+            <div className="grid gap-4 sm:grid-cols-2">
+              {content.skills.groups.map((group) => (
+                <article
+                  className="rounded-lg border border-line bg-surface p-5"
+                  key={group.title}
+                >
+                  <ContentHint
+                    enabled={contentDebug}
+                    path={`src/content/skills.json > groups[title=${group.title}]`}
+                  />
+                  <h2 className="text-sm font-semibold text-foreground">
+                    {group.title}
+                  </h2>
+                  <div className="mt-4">
+                    <StackList items={group.items} />
+                  </div>
+                </article>
+              ))}
+            </div>
+          </div>
+        </div>
+      </section>
+      <section className="border-b border-line bg-background-soft">
+        <div className="mx-auto grid max-w-6xl gap-9 px-5 py-16 sm:px-8 lg:grid-cols-[0.8fr_1.2fr]">
+          <SectionHeading
+            contentDebug={contentDebug}
+            contentHint="src/content/experience.json"
+            title={pageCopy.journey.title}
+          />
+          <ol className="grid gap-4">
+            {content.experience.map((item) => (
+              <li
+                className="rounded-lg border border-line bg-surface p-5"
+                key={`${item.period}-${item.title}`}
+              >
+                <p className="text-xs font-semibold uppercase tracking-[0.08em] text-accent">
+                  {item.period}
+                </p>
+                <h2 className="mt-3 text-base font-semibold text-foreground">
+                  {item.title}
+                </h2>
+                <p className="mt-3 text-sm leading-6 text-muted">{item.body}</p>
+              </li>
+            ))}
+          </ol>
+        </div>
+      </section>
+      {isSitePageEnabled("curation", content) ? (
+        <CurationSection
+          content={content}
+          contentDebug={contentDebug}
+          homeTemplate={activeTemplate}
+        />
+      ) : null}
+    </PageShell>
+  );
+}
+
+function CurationSection({
+  content,
+  contentDebug,
+  homeTemplate,
+}: {
+  content: AboutViewModel;
+  contentDebug: boolean;
+  homeTemplate: HomeTemplateId;
+}) {
+  const pageCopy = content.presentation.pages.about.curation;
+  const data = content.curation;
+
+  return (
+    <section aria-label={pageCopy.title} className="bg-background-soft">
+      <div className="mx-auto grid max-w-6xl gap-9 px-5 py-16 sm:px-8">
+        <SectionHeading
+          body={pageCopy.body}
+          contentDebug={contentDebug}
+          contentHint="src/content/curation.json > intro/criteria + src/content/presentation.json > pages.about.curation"
+          title={pageCopy.title}
+        />
+        <p className="max-w-3xl text-sm leading-6 text-muted md:text-base md:leading-7">
+          {data.intro}
+        </p>
+        <div className="grid gap-9 lg:grid-cols-[0.8fr_1.2fr]">
+          <h3 className="text-xl font-semibold text-foreground">
+            {pageCopy.criteriaTitle}
+          </h3>
+          <div className="grid gap-4 sm:grid-cols-2">
+            {data.criteria.items.map((item) => (
+              <article
+                className="rounded-lg border border-line bg-surface p-5"
+                key={item.title}
+              >
+                <ContentHint
+                  enabled={contentDebug}
+                  path={`src/content/curation.json > criteria.items[title=${item.title}]`}
+                />
+                <h4 className="text-sm font-semibold text-foreground">
+                  {item.title}
+                </h4>
+                <p className="mt-3 text-sm leading-6 text-muted">{item.body}</p>
+              </article>
+            ))}
+          </div>
+        </div>
+        <div className="grid gap-9 lg:grid-cols-[0.8fr_1.2fr]">
+          <h3 className="text-xl font-semibold text-foreground">
+            {pageCopy.categoriesTitle}
+          </h3>
+          <ul className="grid gap-4">
+            {content.curationCategories.map((category) => (
+              <CurationCategoryCard
+                category={category}
+                contentDebug={contentDebug}
+                homeTemplate={homeTemplate}
+                key={category.id}
+              />
+            ))}
+          </ul>
+        </div>
+        <div className="grid gap-9 lg:grid-cols-[0.8fr_1.2fr]">
+          <div>
+            <h3 className="text-xl font-semibold text-foreground">
+              {pageCopy.omissionsTitle}
+            </h3>
+            <p className="mt-3 text-sm leading-6 text-muted">{data.omissions.body}</p>
+          </div>
+          <ul className="grid gap-3">
+            {data.omissions.items.map((item) => (
+              <li
+                className="rounded-lg border border-line bg-surface p-4"
+                key={item.title}
+              >
+                <h4 className="text-sm font-semibold text-foreground">
+                  {item.title}
+                </h4>
+                <p className="mt-2 text-sm leading-6 text-muted">{item.body}</p>
+              </li>
+            ))}
+          </ul>
+        </div>
+        <div className="grid gap-9 lg:grid-cols-[0.8fr_1.2fr]">
+          <h3 className="text-xl font-semibold text-foreground">
+            {pageCopy.nextReviewTitle}
+          </h3>
+          <div className="rounded-lg border border-line bg-surface p-5">
+            <h4 className="text-sm font-semibold text-foreground">
+              {data.nextReview.title}
+            </h4>
+            <p className="mt-3 text-sm leading-6 text-muted">{data.nextReview.body}</p>
+          </div>
+        </div>
+      </div>
+    </section>
+  );
+}
+
+function CurationCategoryCard({
+  category,
+  contentDebug,
+  homeTemplate,
+}: {
+  category: CurationCategoryViewModel;
+  contentDebug: boolean;
+  homeTemplate: HomeTemplateId;
+}) {
+  const projects = category.projects;
+
+  return (
+    <li className="rounded-lg border border-line bg-surface p-5">
+      <ContentHint
+        enabled={contentDebug}
+        path={`src/content/curation.json > categories[id=${category.id}]`}
+      />
+      <h4 className="text-base font-semibold text-foreground">{category.label}</h4>
+      <p className="mt-3 text-sm leading-6 text-muted md:leading-7">
+        {category.rationale}
+      </p>
+      {projects.length > 0 ? (
+        <ul className="mt-4 flex flex-wrap gap-2">
+          {projects.map((project) => (
+            <li key={project.id}>
+              <Link
+                className="inline-flex items-center gap-2 rounded-md border border-line bg-surface-soft px-3 py-1.5 text-xs font-semibold text-muted transition hover:border-accent hover:text-foreground"
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


