## `build(content): 콘텐츠 검사 명령 추가`

diff --git a/package.json b/package.json
index ce9c6c4..20e9fe6 100644
--- a/package.json
+++ b/package.json
@@ -7,7 +7,8 @@
     "build": "next build",
     "start": "next start -p 3100",
     "lint": "eslint .",
-    "typecheck": "tsc --noEmit"
+    "typecheck": "tsc --noEmit",
+    "content:check": "node --import tsx scripts/validate-content.ts"
   },
   "dependencies": {
     "next": "16.2.4",
diff --git a/scripts/validate-content.ts b/scripts/validate-content.ts
new file mode 100644
index 0000000..a48fc73
--- /dev/null
+++ b/scripts/validate-content.ts
@@ -0,0 +1,13 @@
+import { resolve } from "node:path";
+
+import { validatePortfolioAssets } from "../src/lib/content-assets";
+import { loadPortfolioSource } from "../src/lib/content-loader";
+
+const content = validatePortfolioAssets(
+  loadPortfolioSource(),
+  resolve(process.cwd(), "public"),
+);
+
+console.log(
+  `Content valid: ${content.projects.items.length} projects, ${content.presentation.templates.length} designs.`,
+);


## `build(content): 콘텐츠 검사를 prebuild에 연결`

diff --git a/package.json b/package.json
index 20e9fe6..34d8cee 100644
--- a/package.json
+++ b/package.json
@@ -4,6 +4,7 @@
   "private": true,
   "scripts": {
     "dev": "next dev --webpack -p 3100",
+    "prebuild": "npm run content:check",
     "build": "next build",
     "start": "next start -p 3100",
     "lint": "eslint .",


## `feat(routes): 비활성 페이지 route 차단`

diff --git a/src/app/about/page.tsx b/src/app/about/page.tsx
index f7a22ed..d701f02 100644
--- a/src/app/about/page.tsx
+++ b/src/app/about/page.tsx
@@ -1,10 +1,12 @@
 import { ContentHint } from "@/components/portfolio/content-hint";
+import { notFound } from "next/navigation";
 import { JourneyList } from "@/components/portfolio/journey-list";
 import { SectionHeading } from "@/components/portfolio/section-heading";
 import { PageShell } from "@/components/portfolio/site-shell";
 import { StackList } from "@/components/portfolio/stack-list";
 import {
   getPortfolioContent,
+  isSitePageEnabled,
   resolveContentDebug,
   resolveHomeTemplateId,
   type RouteSearchParams,
@@ -16,6 +18,7 @@ export default async function AboutPage({
   searchParams?: RouteSearchParams;
 }) {
   const content = getPortfolioContent();
+  if (!isSitePageEnabled("about", content)) notFound();
   const params = searchParams ? await searchParams : {};
   const activeTemplate = resolveHomeTemplateId(params.view, content.presentation);
   const contentDebug = resolveContentDebug(params.debug);
diff --git a/src/app/contact/page.tsx b/src/app/contact/page.tsx
index 9e1837e..e09d603 100644
--- a/src/app/contact/page.tsx
+++ b/src/app/contact/page.tsx
@@ -1,9 +1,11 @@
 import { ArrowRightIcon } from "@/components/icons";
+import { notFound } from "next/navigation";
 import { ContentHint } from "@/components/portfolio/content-hint";
 import { ContentLinkView } from "@/components/portfolio/content-link";
 import { PageShell } from "@/components/portfolio/site-shell";
 import {
   getPortfolioContent,
+  isSitePageEnabled,
   getPreferredContactLinks,
   resolveContentDebug,
   resolveHomeTemplateId,
@@ -16,6 +18,7 @@ export default async function ContactPage({
   searchParams?: RouteSearchParams;
 }) {
   const content = getPortfolioContent();
+  if (!isSitePageEnabled("contact", content)) notFound();
   const params = searchParams ? await searchParams : {};
   const activeTemplate = resolveHomeTemplateId(params.view, content.presentation);
   const contentDebug = resolveContentDebug(params.debug);
diff --git a/src/app/projects/[projectId]/page.tsx b/src/app/projects/[projectId]/page.tsx
index c18e625..5c5ac53 100644
--- a/src/app/projects/[projectId]/page.tsx
+++ b/src/app/projects/[projectId]/page.tsx
@@ -3,6 +3,7 @@ import { ProjectDetailView } from "@/components/portfolio/project-detail-view";
 import { PageShell } from "@/components/portfolio/site-shell";
 import {
   getPortfolioContent,
+  isSitePageEnabled,
   getProjectById,
   resolveContentDebug,
   resolveHomeTemplateId,
@@ -23,6 +24,7 @@ export default async function ProjectDetailPage({
   searchParams?: RouteSearchParams;
 }) {
   const content = getPortfolioContent();
+  if (!isSitePageEnabled("projects", content)) notFound();
   const { projectId } = await params;
   const query = searchParams ? await searchParams : {};
   const activeTemplate = resolveHomeTemplateId(query.view, content.presentation);
diff --git a/src/app/projects/page.tsx b/src/app/projects/page.tsx
index 659c40b..b82d1f6 100644
--- a/src/app/projects/page.tsx
+++ b/src/app/projects/page.tsx
@@ -1,8 +1,10 @@
 import { PageShell } from "@/components/portfolio/site-shell";
+import { notFound } from "next/navigation";
 import { ClassicProjectsView } from "@/designs/classic/projects/projects-route";
 import { DesignProjectsView } from "@/designs/design/projects/projects-route";
 import {
   getPortfolioContent,
+  isSitePageEnabled,
   getProjectMetricValue,
   resolveContentDebug,
   resolveHomeTemplateId,
@@ -16,6 +18,7 @@ export default async function ProjectsPage({
   searchParams?: RouteSearchParams;
 }) {
   const content = getPortfolioContent();
+  if (!isSitePageEnabled("projects", content)) notFound();
   const params = searchParams ? await searchParams : {};
   const activeTemplate = resolveHomeTemplateId(params.view, content.presentation);
   const contentDebug = resolveContentDebug(params.debug);
diff --git a/src/app/resume/page.tsx b/src/app/resume/page.tsx
index fc07099..b011604 100644
--- a/src/app/resume/page.tsx
+++ b/src/app/resume/page.tsx
@@ -1,10 +1,12 @@
 import Link from "next/link";
+import { notFound } from "next/navigation";
 import { ArrowRightIcon } from "@/components/icons";
 import { ContentHint } from "@/components/portfolio/content-hint";
 import { PageShell } from "@/components/portfolio/site-shell";
 import { StackList } from "@/components/portfolio/stack-list";
 import {
   getPortfolioContent,
+  isSitePageEnabled,
   getResumeProjects,
   getTemplateHref,
   resolveContentDebug,
@@ -18,6 +20,7 @@ export default async function ResumePage({
   searchParams?: RouteSearchParams;
 }) {
   const content = getPortfolioContent();
+  if (!isSitePageEnabled("resume", content)) notFound();
   const params = searchParams ? await searchParams : {};
   const activeTemplate = resolveHomeTemplateId(params.view, content.presentation);
   const contentDebug = resolveContentDebug(params.debug);


