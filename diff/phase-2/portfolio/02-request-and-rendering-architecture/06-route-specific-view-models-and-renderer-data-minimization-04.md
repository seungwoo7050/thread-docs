## `refactor(designs): renderer 입력을 route view model로 제한`

diff --git a/src/designs/brutalist/brutalist-route.tsx b/src/designs/brutalist/brutalist-route.tsx
index 9481993..dea950d 100644
--- a/src/designs/brutalist/brutalist-route.tsx
+++ b/src/designs/brutalist/brutalist-route.tsx
@@ -16,6 +16,7 @@ import type {
   JourneyViewModel,
   ProjectDetailViewModel,
   ProjectIndexViewModel,
+  PortfolioRouteViewModel,
   ResumeViewModel,
 } from "@/lib/portfolio/view-models";
 import type { DesignRouteProps } from "@/designs/types";
@@ -183,17 +184,14 @@ function BrutalistShell({
   route,
 }: {
   children: React.ReactNode;
-  content: PortfolioContent;
+  content: PortfolioRouteViewModel;
   contentDebug: boolean;
   currentPath: string;
   route: DesignRouteProps["route"];
 }) {
   const shellCopy = content.presentation.brutalist.shell;
   const ui = content.presentation.ui;
-  const footerLinks =
-    "footerLinks" in content
-      ? (content.footerLinks as ContentLink[])
-      : content.links.filter((link) => link.placements?.includes("footer"));
+  const footerLinks = content.footerLinks;
 
   return (
     <div
diff --git a/src/designs/cinematic/cinematic-route.tsx b/src/designs/cinematic/cinematic-route.tsx
index cbf7c8f..1d4451d 100644
--- a/src/designs/cinematic/cinematic-route.tsx
+++ b/src/designs/cinematic/cinematic-route.tsx
@@ -102,10 +102,7 @@ function Frame({
   contentDebug,
   currentPath,
 }: DesignRouteProps & { children: React.ReactNode }) {
-  const footerLinks =
-    "footerLinks" in content
-      ? (content.footerLinks as ContentLink[])
-      : content.links.filter((link) => link.placements?.includes("footer"));
+  const footerLinks = content.footerLinks;
   const ui = content.presentation.ui;
 
   return (
diff --git a/src/designs/classic/about-route.tsx b/src/designs/classic/about-route.tsx
index ca0c9a0..572bbca 100644
--- a/src/designs/classic/about-route.tsx
+++ b/src/designs/classic/about-route.tsx
@@ -15,7 +15,7 @@ import {
   isSitePageEnabled,
   type HomeTemplateId,
 } from "@/lib/portfolio";
-import type { PreparedDesignRouteProps as DesignRouteProps } from "@/designs/shell-props";
+import type { DesignRouteProps } from "@/designs/types";
 import { createDesignShellProps } from "@/designs/shell-props";
 
 export default function AboutRoute({
diff --git a/src/designs/classic/contact-route.tsx b/src/designs/classic/contact-route.tsx
index 0aee0fe..8946a0e 100644
--- a/src/designs/classic/contact-route.tsx
+++ b/src/designs/classic/contact-route.tsx
@@ -2,7 +2,7 @@ import { ArrowRightIcon } from "@/components/icons";
 import { ContentHint } from "@/components/portfolio/content-hint";
 import { ContentLinkView } from "@/components/portfolio/content-link";
 import { PageShell } from "@/components/portfolio/site-shell";
-import type { PreparedDesignRouteProps as DesignRouteProps } from "@/designs/shell-props";
+import type { DesignRouteProps } from "@/designs/types";
 import { createDesignShellProps } from "@/designs/shell-props";
 
 export default function ContactRoute({
diff --git a/src/designs/classic/home-route.tsx b/src/designs/classic/home-route.tsx
index 6dba64b..1e19b2e 100644
--- a/src/designs/classic/home-route.tsx
+++ b/src/designs/classic/home-route.tsx
@@ -20,7 +20,7 @@ import {
   type HomeTemplateId,
   type PortfolioProject,
 } from "@/lib/portfolio";
-import type { PreparedDesignRouteProps as DesignRouteProps } from "@/designs/shell-props";
+import type { DesignRouteProps } from "@/designs/types";
 import { createDesignShellProps } from "@/designs/shell-props";
 
 export default function HomeRoute({
diff --git a/src/designs/classic/index.tsx b/src/designs/classic/index.tsx
index ba74527..909447c 100644
--- a/src/designs/classic/index.tsx
+++ b/src/designs/classic/index.tsx
@@ -1,4 +1,4 @@
-import type { PreparedDesignRouteProps as DesignRouteProps } from "@/designs/shell-props";
+import type { DesignRouteProps } from "@/designs/types";
 
 import AboutRoute from "./about-route";
 import ContactRoute from "./contact-route";
diff --git a/src/designs/classic/interview-map-route.tsx b/src/designs/classic/interview-map-route.tsx
index 03bbdd0..33a4c15 100644
--- a/src/designs/classic/interview-map-route.tsx
+++ b/src/designs/classic/interview-map-route.tsx
@@ -11,7 +11,7 @@ import {
   type InterviewMapTrackViewModel,
   type InterviewMapViewModel,
 } from "@/lib/portfolio/view-models";
-import type { PreparedDesignRouteProps as DesignRouteProps } from "@/designs/shell-props";
+import type { DesignRouteProps } from "@/designs/types";
 import { createDesignShellProps } from "@/designs/shell-props";
 
 export default function InterviewMapRoute({
diff --git a/src/designs/classic/journey-route.tsx b/src/designs/classic/journey-route.tsx
index 92a80bc..052b554 100644
--- a/src/designs/classic/journey-route.tsx
+++ b/src/designs/classic/journey-route.tsx
@@ -12,7 +12,7 @@ import {
 import {
   type JourneyMilestoneViewModel,
 } from "@/lib/portfolio/view-models";
-import type { PreparedDesignRouteProps as DesignRouteProps } from "@/designs/shell-props";
+import type { DesignRouteProps } from "@/designs/types";
 import { createDesignShellProps } from "@/designs/shell-props";
 
 export default function JourneyRoute({
diff --git a/src/designs/classic/project-detail-route.tsx b/src/designs/classic/project-detail-route.tsx
index e19ce56..e5c4f76 100644
--- a/src/designs/classic/project-detail-route.tsx
+++ b/src/designs/classic/project-detail-route.tsx
@@ -13,7 +13,7 @@ import {
   type PortfolioProject,
 } from "@/lib/portfolio";
 import { createDesignShellProps } from "@/designs/shell-props";
-import type { PreparedDesignRouteProps as DesignRouteProps } from "@/designs/shell-props";
+import type { DesignRouteProps } from "@/designs/types";
 
 export default function ProjectDetailRoute({
   content,
diff --git a/src/designs/classic/projects-route.tsx b/src/designs/classic/projects-route.tsx
index 2847aaf..9019aad 100644
--- a/src/designs/classic/projects-route.tsx
+++ b/src/designs/classic/projects-route.tsx
@@ -11,7 +11,7 @@ import {
   type ProjectPageContent,
   type PortfolioProject,
 } from "@/lib/portfolio";
-import type { PreparedDesignRouteProps as DesignRouteProps } from "@/designs/shell-props";
+import type { DesignRouteProps } from "@/designs/types";
 import { createDesignShellProps } from "@/designs/shell-props";
 
 export default function ProjectsRoute({
diff --git a/src/designs/classic/resume-route.tsx b/src/designs/classic/resume-route.tsx
index bf7bf31..0930bf2 100644
--- a/src/designs/classic/resume-route.tsx
+++ b/src/designs/classic/resume-route.tsx
@@ -6,7 +6,7 @@ import { StackList } from "@/components/portfolio/stack-list";
 import {
   getTemplateHref,
 } from "@/lib/portfolio";
-import type { PreparedDesignRouteProps as DesignRouteProps } from "@/designs/shell-props";
+import type { DesignRouteProps } from "@/designs/types";
 import { createDesignShellProps } from "@/designs/shell-props";
 
 export default function ResumeRoute({
diff --git a/src/designs/design/about-route.tsx b/src/designs/design/about-route.tsx
index e93ee76..4f4cb7d 100644
--- a/src/designs/design/about-route.tsx
+++ b/src/designs/design/about-route.tsx
@@ -15,7 +15,7 @@ import {
   isSitePageEnabled,
   type HomeTemplateId,
 } from "@/lib/portfolio";
-import type { PreparedDesignRouteProps as DesignRouteProps } from "@/designs/shell-props";
+import type { DesignRouteProps } from "@/designs/types";
 import { createDesignShellProps } from "@/designs/shell-props";
 
 export default function AboutRoute({
diff --git a/src/designs/design/contact-route.tsx b/src/designs/design/contact-route.tsx
index b8a608a..db909b5 100644
--- a/src/designs/design/contact-route.tsx
+++ b/src/designs/design/contact-route.tsx
@@ -2,7 +2,7 @@ import { ArrowRightIcon } from "@/components/icons";
 import { ContentHint } from "@/components/portfolio/content-hint";
 import { ContentLinkView } from "@/components/portfolio/content-link";
 import { PageShell } from "@/components/portfolio/site-shell";
-import type { PreparedDesignRouteProps as DesignRouteProps } from "@/designs/shell-props";
+import type { DesignRouteProps } from "@/designs/types";
 import { createDesignShellProps } from "@/designs/shell-props";
 
 export default function ContactRoute({
diff --git a/src/designs/design/home-route.tsx b/src/designs/design/home-route.tsx
index 5893dfd..7724eb1 100644
--- a/src/designs/design/home-route.tsx
+++ b/src/designs/design/home-route.tsx
@@ -20,7 +20,7 @@ import {
   type HomeTemplateId,
   type PortfolioProject,
 } from "@/lib/portfolio";
-import type { PreparedDesignRouteProps as DesignRouteProps } from "@/designs/shell-props";
+import type { DesignRouteProps } from "@/designs/types";
 import { createDesignShellProps } from "@/designs/shell-props";
 
 export default function HomeRoute({
diff --git a/src/designs/design/index.tsx b/src/designs/design/index.tsx
index b0b70c8..261072c 100644
--- a/src/designs/design/index.tsx
+++ b/src/designs/design/index.tsx
@@ -1,4 +1,4 @@
-import type { PreparedDesignRouteProps as DesignRouteProps } from "@/designs/shell-props";
+import type { DesignRouteProps } from "@/designs/types";
 
 import AboutRoute from "./about-route";
 import ContactRoute from "./contact-route";
diff --git a/src/designs/design/interview-map-route.tsx b/src/designs/design/interview-map-route.tsx
index f5170ee..c7943cc 100644
--- a/src/designs/design/interview-map-route.tsx
+++ b/src/designs/design/interview-map-route.tsx
@@ -11,7 +11,7 @@ import {
   type InterviewMapTrackViewModel,
   type InterviewMapViewModel,
 } from "@/lib/portfolio/view-models";
-import type { PreparedDesignRouteProps as DesignRouteProps } from "@/designs/shell-props";
+import type { DesignRouteProps } from "@/designs/types";
 import { createDesignShellProps } from "@/designs/shell-props";
 
 export default function InterviewMapRoute({
diff --git a/src/designs/design/journey-route.tsx b/src/designs/design/journey-route.tsx
index 4c303df..a836816 100644
--- a/src/designs/design/journey-route.tsx
+++ b/src/designs/design/journey-route.tsx
@@ -12,7 +12,7 @@ import {
 import {
   type JourneyMilestoneViewModel,
 } from "@/lib/portfolio/view-models";
-import type { PreparedDesignRouteProps as DesignRouteProps } from "@/designs/shell-props";
+import type { DesignRouteProps } from "@/designs/types";
 import { createDesignShellProps } from "@/designs/shell-props";
 
 export default function JourneyRoute({
diff --git a/src/designs/design/project-detail-route.tsx b/src/designs/design/project-detail-route.tsx
index bffeaaa..7b2fdda 100644
--- a/src/designs/design/project-detail-route.tsx
+++ b/src/designs/design/project-detail-route.tsx
@@ -13,7 +13,7 @@ import {
   type PortfolioProject,
 } from "@/lib/portfolio";
 import { createDesignShellProps } from "@/designs/shell-props";
-import type { PreparedDesignRouteProps as DesignRouteProps } from "@/designs/shell-props";
+import type { DesignRouteProps } from "@/designs/types";
 
 export default function ProjectDetailRoute({
   content,
diff --git a/src/designs/design/projects-route.tsx b/src/designs/design/projects-route.tsx
index 65deb8a..d5e9e72 100644
--- a/src/designs/design/projects-route.tsx
+++ b/src/designs/design/projects-route.tsx
@@ -6,7 +6,7 @@ import {
   type ProjectPageContent,
   type PortfolioProject,
 } from "@/lib/portfolio";
-import type { PreparedDesignRouteProps as DesignRouteProps } from "@/designs/shell-props";
+import type { DesignRouteProps } from "@/designs/types";
 import { createDesignShellProps } from "@/designs/shell-props";
 
 export default function ProjectsRoute({
diff --git a/src/designs/design/resume-route.tsx b/src/designs/design/resume-route.tsx
index eed9da2..c20045d 100644
--- a/src/designs/design/resume-route.tsx
+++ b/src/designs/design/resume-route.tsx
@@ -6,7 +6,7 @@ import { StackList } from "@/components/portfolio/stack-list";
 import {
   getTemplateHref,
 } from "@/lib/portfolio";
-import type { PreparedDesignRouteProps as DesignRouteProps } from "@/designs/shell-props";
+import type { DesignRouteProps } from "@/designs/types";
 import { createDesignShellProps } from "@/designs/shell-props";
 
 export default function ResumeRoute({
diff --git a/src/designs/editorial/editorial-route.tsx b/src/designs/editorial/editorial-route.tsx
index d778c59..42a5eb5 100644
--- a/src/designs/editorial/editorial-route.tsx
+++ b/src/designs/editorial/editorial-route.tsx
@@ -7,7 +7,6 @@ import {
   getTemplateHref,
   isSitePageEnabled,
   type ContentLink,
-  type PortfolioContent,
   type PortfolioProject,
   type ProjectImage,
 } from "@/lib/portfolio";
@@ -19,6 +18,7 @@ import type {
   JourneyViewModel,
   ProjectDetailViewModel,
   ProjectIndexViewModel,
+  PortfolioRouteViewModel,
   ResumeViewModel,
 } from "@/lib/portfolio/view-models";
 
@@ -36,7 +36,7 @@ export type EditorialRouteName =
 
 export type EditorialRouteProps = {
   route: EditorialRouteName;
-  content: PortfolioContent;
+  content: PortfolioRouteViewModel;
   project?: PortfolioProject;
   currentPath: string;
   contentDebug: boolean;
@@ -167,10 +167,7 @@ function EditorialShell({
   const primaryNavigation = content.site.navigation;
   const ui = content.presentation.ui;
   const shellCopy = content.presentation.editorial.shell;
-  const footerLinks =
-    "footerLinks" in content
-      ? (content.footerLinks as ContentLink[])
-      : content.links.filter((link) => link.placements?.includes("footer"));
+  const footerLinks = content.footerLinks;
 
   return (
     <div className={styles.root} data-site-design="editorial">
diff --git a/src/designs/shell-props.ts b/src/designs/shell-props.ts
index b89a418..ebc7c08 100644
--- a/src/designs/shell-props.ts
+++ b/src/designs/shell-props.ts
@@ -1,15 +1,6 @@
-import type { PortfolioProject, SiteDesignId } from "@/lib/portfolio";
-import type { PortfolioRouteId } from "@/designs/types";
+import type { SiteDesignId } from "@/lib/portfolio";
 import type { PortfolioRouteViewModel } from "@/lib/portfolio/view-models";
 
-export type PreparedDesignRouteProps = {
-  content: PortfolioRouteViewModel;
-  contentDebug: boolean;
-  currentPath: string;
-  project?: PortfolioProject;
-  route: PortfolioRouteId;
-};
-
 export function createDesignShellProps(
   content: PortfolioRouteViewModel,
   contentDebug: boolean,
diff --git a/src/designs/types.ts b/src/designs/types.ts
index 82584f0..d0c2861 100644
--- a/src/designs/types.ts
+++ b/src/designs/types.ts
@@ -1,4 +1,4 @@
-import type { PortfolioContent, PortfolioProject } from "@/lib/portfolio";
+import type { PortfolioProject } from "@/lib/portfolio";
 import type { PortfolioRouteViewModel } from "@/lib/portfolio/view-models";
 
 export type PortfolioRouteId =
@@ -12,7 +12,7 @@ export type PortfolioRouteId =
   | "interview-map";
 
 export type DesignRouteProps = {
-  content: PortfolioContent;
+  content: PortfolioRouteViewModel;
   contentDebug: boolean;
   currentPath: string;
   project?: PortfolioProject;
@@ -28,11 +28,4 @@ type ViewModelDesignRouteRequest = {
   };
 }[PortfolioRouteViewModel["route"]];
 
-export type DesignRouteRequestProps =
-  | ViewModelDesignRouteRequest
-  | {
-      content: PortfolioContent;
-      contentDebug: boolean;
-      currentPath: string;
-      route: "journey" | "interview-map";
-    };
+export type DesignRouteRequestProps = ViewModelDesignRouteRequest;


## `test(content): scoped view model과 연락처 회귀 검증`

diff --git a/src/lib/portfolio/view-models.test.ts b/src/lib/portfolio/view-models.test.ts
index 83ffff5..c645724 100644
--- a/src/lib/portfolio/view-models.test.ts
+++ b/src/lib/portfolio/view-models.test.ts
@@ -1,3 +1,6 @@
+import { readFileSync } from "node:fs";
+import { resolve } from "node:path";
+
 import { describe, expect, it } from "vitest";
 
 import { getPortfolioContent } from "./content";
@@ -5,12 +8,115 @@ import {
   createAboutViewModel,
   createContactViewModel,
   createHomeViewModel,
+  createInterviewMapViewModel,
+  createJourneyViewModel,
   createProjectDetailViewModel,
   createProjectIndexViewModel,
   createResumeViewModel,
 } from "./view-models";
 
 describe("portfolio route view models", () => {
+  it("exposes only the shared shell fields and route-specific data", () => {
+    const content = getPortfolioContent();
+    const project = content.projects[0];
+
+    expect(project).toBeDefined();
+
+    const viewModels = [
+      {
+        sourceFields: [
+          "contact",
+          "journey",
+          "journeyNarrative",
+          "presentation",
+          "profile",
+          "site",
+          "skills",
+          "techStack",
+        ],
+        viewModel: createHomeViewModel(content),
+      },
+      {
+        sourceFields: ["contact", "presentation", "profile", "projects", "site"],
+        viewModel: createProjectIndexViewModel(content),
+      },
+      {
+        sourceFields: ["presentation", "profile", "site"],
+        viewModel: createProjectDetailViewModel(content, project.id),
+      },
+      {
+        sourceFields: [
+          "contact",
+          "curation",
+          "experience",
+          "journey",
+          "presentation",
+          "profile",
+          "site",
+          "skills",
+        ],
+        viewModel: createAboutViewModel(content),
+      },
+      {
+        sourceFields: [
+          "experience",
+          "presentation",
+          "profile",
+          "resume",
+          "site",
+        ],
+        viewModel: createResumeViewModel(content),
+      },
+      {
+        sourceFields: ["contact", "presentation", "profile", "site"],
+        viewModel: createContactViewModel(content),
+      },
+      {
+        sourceFields: [
+          "journey",
+          "journeyNarrative",
+          "presentation",
+          "profile",
+          "site",
+        ],
+        viewModel: createJourneyViewModel(content),
+      },
+      {
+        sourceFields: ["interviewMap", "presentation", "profile", "site"],
+        viewModel: createInterviewMapViewModel(content),
+      },
+    ];
+
+    for (const { sourceFields, viewModel } of viewModels) {
+      expect(viewModel).not.toBeNull();
+      if (!viewModel) throw new Error("expected a route view model");
+
+      expect(viewModel).toEqual(
+        expect.objectContaining({
+          footerLinks: expect.any(Array),
+          presentation: content.presentation,
+          profile: content.profile,
+          site: content.site,
+        }),
+      );
+      expect(
+        Object.keys(viewModel)
+          .filter((key) => Object.hasOwn(content, key))
+          .sort(),
+      ).toEqual(sourceFields);
+    }
+  });
+
+  it("does not build route models by spreading the full content object", () => {
+    const source = readFileSync(
+      resolve(process.cwd(), "src/lib/portfolio/view-models.ts"),
+      "utf8",
+    );
+
+    expect(source).not.toMatch(/PortfolioContent\s*&/);
+    expect(source).not.toMatch(/\.\.\.content\b/);
+  });
+
   it("prepares home selections, metrics, and links before rendering", () => {
     const content = getPortfolioContent();
     const viewModel = createHomeViewModel(
@@ -51,6 +157,7 @@ describe("portfolio route view models", () => {
     );
 
     expect(viewModel.route).toBe("projects");
+    expect(viewModel.contact).toBe(content.contact);
     expect(viewModel.groups.map((group) => group.id)).toEqual(
       [...viewModel.groups.map((group) => group.id)].sort(
         (left, right) => groupOrder.indexOf(left) - groupOrder.indexOf(right),
@@ -92,6 +199,7 @@ describe("portfolio route view models", () => {
     const viewModel = createAboutViewModel(content);
 
     expect(viewModel.route).toBe("about");
+    expect(viewModel.contact).toBe(content.contact);
     expect(viewModel.curationCategories).toHaveLength(
       content.curation.categories.length,
     );
@@ -137,4 +245,43 @@ describe("portfolio route view models", () => {
         : viewModel.contactPlacementLinks,
     );
   });
+
+  it("resolves journey milestone projects before rendering", () => {
+    const content = structuredClone(getPortfolioContent());
+    const firstMilestone = content.journeyNarrative.milestones[0];
+
+    expect(firstMilestone).toBeDefined();
+    firstMilestone.anchorProjectIds = [
+      ...firstMilestone.anchorProjectIds,
+      "missing-project",
+    ];
+
+    const viewModel = createJourneyViewModel(content);
+
+    expect(viewModel.route).toBe("journey");
+    expect(viewModel.milestones[0]?.anchorProjects.map((project) => project.id)).toEqual(
+      firstMilestone.anchorProjectIds.filter((projectId) =>
+        content.projects.some((project) => project.id === projectId),
+      ),
+    );
+  });
+
+  it("resolves interview-map answers before rendering", () => {
+    const content = structuredClone(getPortfolioContent());
+    const firstAnswer = content.interviewMap.tracks[0]?.items[0]?.answers[0];
+
+    expect(firstAnswer).toBeDefined();
+    if (firstAnswer) firstAnswer.projectId = "missing-project";
+
+    const viewModel = createInterviewMapViewModel(content);
+    const answer = viewModel.tracks[0]?.items[0]?.answers[0];
+
+    expect(viewModel.route).toBe("interview-map");
+    expect(answer).toEqual(
+      expect.objectContaining({
+        project: null,
+        projectId: "missing-project",
+      }),
+    );
+  });
 });
