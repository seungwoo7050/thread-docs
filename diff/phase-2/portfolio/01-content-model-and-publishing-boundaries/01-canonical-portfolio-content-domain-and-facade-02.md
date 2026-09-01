## `feat(portfolio): 기술과 프로젝트 조회기 추가`

diff --git a/src/lib/portfolio/selectors.ts b/src/lib/portfolio/selectors.ts
index 08e613c..41bb940 100644
--- a/src/lib/portfolio/selectors.ts
+++ b/src/lib/portfolio/selectors.ts
@@ -1,7 +1,14 @@
-import { portfolioPresentation } from "./content";
+import {
+  getPortfolioContent,
+  portfolioPresentation,
+  portfolioTechStackById,
+} from "./content";
 import type {
   HomeTemplateId,
+  PortfolioContent,
+  PortfolioProject,
   PresentationContent,
+  TechStackItem,
 } from "./types";
 import { createTemplateHref } from "./template-href";
 
@@ -37,3 +44,37 @@ export function getTemplateHref(
     options,
   );
 }
+
+export function resolveTechStackItem(id: string): TechStackItem {
+  return (
+    portfolioTechStackById.get(id) ?? {
+      id,
+      label: id,
+      icon: "tool",
+      color: "#9cc8b1",
+    }
+  );
+}
+
+export function getFeaturedProjects(
+  content: PortfolioContent = getPortfolioContent(),
+) {
+  return content.projects.filter((project) => project.featured);
+}
+
+export function getProjectById(
+  projectId: string,
+  content: PortfolioContent = getPortfolioContent(),
+) {
+  return content.projects.find((project) => project.id === projectId) ?? null;
+}
+
+export function getResumeProjects(
+  content: PortfolioContent = getPortfolioContent(),
+) {
+  const byId = new Map(content.projects.map((project) => [project.id, project]));
+
+  return content.resume.projectIds
+    .map((projectId) => byId.get(projectId))
+    .filter((project): project is PortfolioProject => Boolean(project));
+}


## `feat(portfolio): 연락과 프로젝트 링크 선택기 추가`

diff --git a/src/lib/portfolio.ts b/src/lib/portfolio.ts
new file mode 100644
index 0000000..0160216
--- /dev/null
+++ b/src/lib/portfolio.ts
@@ -0,0 +1,16 @@
+export * from "./portfolio/types";
+export { getEnabledLinks, getPortfolioContent } from "./portfolio/content";
+export {
+  getExternalLinkProps,
+  getFeaturedProjects,
+  getPreferredContactLinks,
+  getProjectById,
+  getProjectCardLinks,
+  getProjectLink,
+  getResumeProjects,
+  getTemplateHref,
+  isProjectLive,
+  resolveContentDebug,
+  resolveHomeTemplateId,
+  resolveTechStackItem,
+} from "./portfolio/selectors";
diff --git a/src/lib/portfolio/selectors.ts b/src/lib/portfolio/selectors.ts
index 41bb940..ecb2fab 100644
--- a/src/lib/portfolio/selectors.ts
+++ b/src/lib/portfolio/selectors.ts
@@ -4,7 +4,9 @@ import {
   portfolioTechStackById,
 } from "./content";
 import type {
+  ContentLink,
   HomeTemplateId,
+  LinkType,
   PortfolioContent,
   PortfolioProject,
   PresentationContent,
@@ -78,3 +80,44 @@ export function getResumeProjects(
     .map((projectId) => byId.get(projectId))
     .filter((project): project is PortfolioProject => Boolean(project));
 }
+
+export function getPreferredContactLinks(
+  content: PortfolioContent = getPortfolioContent(),
+) {
+  const byId = new Map(content.links.map((link) => [link.id, link]));
+
+  return content.contact.preferred
+    .map((id) => byId.get(id))
+    .filter((link): link is ContentLink => Boolean(link));
+}
+
+export function getProjectLink(project: PortfolioProject, type: LinkType) {
+  return project.links.find((link) => link.type === type) ?? null;
+}
+
+export function isProjectLive(project: PortfolioProject) {
+  return Boolean(
+    project.deployment.status === "live" && getProjectLink(project, "demo"),
+  );
+}
+
+export function getProjectCardLinks(project: PortfolioProject) {
+  return project.links.filter((link) => {
+    if (link.type === "demo") {
+      return isProjectLive(project);
+    }
+
+    return link.type === "github" || link.type === "case-study";
+  });
+}
+
+export function getExternalLinkProps(link: ContentLink) {
+  if (!link.external) {
+    return {};
+  }
+
+  return {
+    rel: "noreferrer",
+    target: "_blank",
+  };
+}


## `refactor(content): 검증된 콘텐츠를 portfolio facade에 연결`

diff --git a/src/lib/portfolio/content.ts b/src/lib/portfolio/content.ts
index e7df0db..108bfc0 100644
--- a/src/lib/portfolio/content.ts
+++ b/src/lib/portfolio/content.ts
@@ -1,46 +1,55 @@
-import contactJson from "@/content/contact.json";
-import experienceJson from "@/content/experience.json";
-import journeyJson from "@/content/journey.json";
-import linksJson from "@/content/links.json";
-import presentationJson from "@/content/presentation.json";
-import profileJson from "@/content/profile.json";
-import projectsJson from "@/content/projects.json";
-import resumeJson from "@/content/resume.json";
-import siteJson from "@/content/site.json";
-import skillsJson from "@/content/skills.json";
-import techStackJson from "@/content/tech-stack.json";
+import { portfolioSource } from "../content-loader";
 import type {
   ContactContent,
   ContentLink,
+  CurationContent,
   ExperienceItem,
+  InterviewMapContent,
   JourneyItem,
+  JourneyNarrativeContent,
   PortfolioContent,
-  PortfolioEnv,
   PortfolioProject,
   PresentationContent,
   ProfileContent,
+  ProjectGroup,
+  ProjectMetric,
   ResumeContent,
   SiteContent,
   SkillsContent,
   TechStackItem,
 } from "./types";
 
-const site = siteJson as SiteContent;
-const profile = profileJson as ProfileContent;
-export const portfolioPresentation =
-  presentationJson as PresentationContent;
-type ProjectContentFile =
-  | PortfolioProject[]
-  | { items: PortfolioProject[] };
-
-const projectContentFile = projectsJson as unknown as ProjectContentFile;
-const projects = Array.isArray(projectContentFile)
-  ? projectContentFile
-  : projectContentFile.items;
-const skills = skillsJson as SkillsContent;
-const techStack = techStackJson as TechStackItem[];
-const experience = experienceJson as ExperienceItem[];
-const journey = (journeyJson as JourneyItem[]).slice().sort((left, right) => {
+const site = portfolioSource.site as SiteContent;
+const profile = portfolioSource.profile as ProfileContent;
+const projectGroups = portfolioSource.projects.groups
+  .slice()
+  .sort((left, right) => left.order - right.order) as ProjectGroup[];
+const projectMetrics = portfolioSource.projects.metrics as ProjectMetric[];
+const projectGroupById = new Map(
+  projectGroups.map((group) => [group.id, group]),
+);
+const projects = portfolioSource.projects.items.map((project) => ({
+  ...project,
+  category: projectGroupById.get(project.groupId)?.label ?? project.groupId,
+})) as PortfolioProject[];
+const presentationSource = portfolioSource.presentation as unknown as PresentationContent;
+export const portfolioPresentation = {
+  ...presentationSource,
+  pages: {
+    ...presentationSource.pages,
+    projects: {
+      ...presentationSource.pages.projects,
+      groups: projectGroups.map((group) => ({
+        category: group.label,
+        body: group.description,
+      })),
+    },
+  },
+} satisfies PresentationContent;
+const skills = portfolioSource.skills as SkillsContent;
+const techStack = portfolioSource.techStack as TechStackItem[];
+const experience = portfolioSource.experience as ExperienceItem[];
+const journey = (portfolioSource.journey as JourneyItem[]).slice().sort((left, right) => {
   const dateOrder = left.date.localeCompare(right.date);
 
   if (dateOrder !== 0) {
@@ -49,9 +58,12 @@ const journey = (journeyJson as JourneyItem[]).slice().sort((left, right) => {
 
   return left.title.localeCompare(right.title);
 });
-const links = linksJson as ContentLink[];
-const contact = contactJson as ContactContent;
-const resume = resumeJson as ResumeContent;
+const links = portfolioSource.links as ContentLink[];
+const contact = portfolioSource.contact as ContactContent;
+const resume = portfolioSource.resume as ResumeContent;
+const journeyNarrative = portfolioSource.journeyNarrative as JourneyNarrativeContent;
+const interviewMap = portfolioSource.interviewMap as InterviewMapContent;
+const curation = portfolioSource.curation as CurationContent;
 export const portfolioTechStackById = new Map(
   techStack.map((item) => [item.id, item]),
 );
@@ -60,39 +72,32 @@ export function getEnabledLinks(contentLinks: ContentLink[] = links) {
   return contentLinks.filter((link) => link.enabled !== false);
 }
 
-function withEnvHref(link: ContentLink, env: PortfolioEnv): ContentLink {
-  const envValue = link.envKey ? env[link.envKey]?.trim() : undefined;
-
-  return {
-    ...link,
-    href: envValue && envValue.length > 0 ? envValue : link.href,
-  };
-}
-
 export function getPortfolioContent(
-  env: PortfolioEnv = {
-    NEXT_PUBLIC_DASHBOARD_URL: process.env.NEXT_PUBLIC_DASHBOARD_URL,
-    NEXT_PUBLIC_SEOUL_APP_URL: process.env.NEXT_PUBLIC_SEOUL_APP_URL,
-  },
+  _legacyEnvironment?: Readonly<Record<string, string | undefined>>,
 ): PortfolioContent {
+  void _legacyEnvironment;
+
   const resolvedProjects = projects
     .filter((project) => project.enabled !== false)
     .map((project) => ({
       ...project,
-      links: project.links
-        .filter((link) => link.enabled !== false)
-        .map((link) => withEnvHref(link, env)),
+      links: project.links.filter((link) => link.enabled !== false),
     }));
 
   return {
     site,
     profile,
     projects: resolvedProjects,
+    projectGroups,
+    projectMetrics,
     presentation: portfolioPresentation,
     skills,
     techStack,
     experience,
     journey,
+    journeyNarrative,
+    interviewMap,
+    curation,
     links: getEnabledLinks(),
     contact,
     resume,
diff --git a/src/lib/portfolio/types.ts b/src/lib/portfolio/types.ts
index 492b384..cf77ac3 100644
--- a/src/lib/portfolio/types.ts
+++ b/src/lib/portfolio/types.ts
@@ -1,4 +1,9 @@
-import type { PresentationContentSource } from "../content-schema";
+import type {
+  PresentationContentSource,
+  ProjectGroup,
+  ProjectMetric,
+  ProjectMetricFilter,
+} from "../content-schema";
 
 export type {
   PortfolioProjectSource,
@@ -68,14 +73,11 @@ export type LinkType =
   | "source"
   | "website";
 
-export type EnvKey = "NEXT_PUBLIC_DASHBOARD_URL" | "NEXT_PUBLIC_SEOUL_APP_URL";
-
 export type ContentLink = {
   id?: string;
   type: LinkType;
   label: string;
   href: string;
-  envKey?: EnvKey;
   external?: boolean;
   enabled?: boolean;
   placements?: Array<"hero" | "contact" | "card" | "detail" | "footer">;
@@ -329,7 +331,6 @@ export type ResumeContent = {
   notes: string[];
 };
 
-
 export type JourneyMilestone = {
   id: string;
   date: string;
@@ -422,17 +423,21 @@ export type PortfolioContent = {
   site: SiteContent;
   profile: ProfileContent;
   projects: PortfolioProject[];
+  projectGroups: ProjectGroup[];
+  projectMetrics: ProjectMetric[];
   presentation: PresentationContent;
   skills: SkillsContent;
   techStack: TechStackItem[];
   experience: ExperienceItem[];
   journey: JourneyItem[];
+  journeyNarrative: JourneyNarrativeContent;
+  interviewMap: InterviewMapContent;
+  curation: CurationContent;
   links: ContentLink[];
   contact: ContactContent;
   resume: ResumeContent;
 };
 
-export type PortfolioEnv = Partial<Record<EnvKey, string | undefined>>;
 export type RouteSearchParams = Promise<
   Record<string, string | string[] | undefined>
 >;


## `refactor(project): 상세 링크를 배치 기준으로 선택`

diff --git a/src/components/portfolio/project-links.tsx b/src/components/portfolio/project-links.tsx
index ed4b9f4..34e675a 100644
--- a/src/components/portfolio/project-links.tsx
+++ b/src/components/portfolio/project-links.tsx
@@ -1,6 +1,7 @@
 import { ArrowUpRightIcon, ExternalLinkIcon } from "@/components/icons";
 import {
   getProjectCardLinks,
+  getProjectDetailLinks,
   isProjectLive,
   type ContentLink,
   type HomeTemplateId,
@@ -27,13 +28,11 @@ export function ProjectLinks({
   homeTemplate?: HomeTemplateId;
   project: PortfolioProject;
 }) {
-  const links = project.links.filter((link) => {
-    if (excludeCaseStudy && link.type === "case-study") {
-      return false;
-    }
-
-    return isVisibleProjectLink(project, link);
-  });
+  const links = getProjectDetailLinks(project).filter(
+    (link) =>
+      (!excludeCaseStudy || link.type !== "case-study") &&
+      isVisibleProjectLink(project, link),
+  );
 
   if (!links.length) {
     return null;
