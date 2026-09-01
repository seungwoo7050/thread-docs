## `refactor(content): 프로젝트 컬렉션 migration 경계 추가`

diff --git a/src/lib/portfolio/content.ts b/src/lib/portfolio/content.ts
index 80104ff..e7df0db 100644
--- a/src/lib/portfolio/content.ts
+++ b/src/lib/portfolio/content.ts
@@ -29,7 +29,14 @@ const site = siteJson as SiteContent;
 const profile = profileJson as ProfileContent;
 export const portfolioPresentation =
   presentationJson as PresentationContent;
-const projects = projectsJson as PortfolioProject[];
+type ProjectContentFile =
+  | PortfolioProject[]
+  | { items: PortfolioProject[] };
+
+const projectContentFile = projectsJson as unknown as ProjectContentFile;
+const projects = Array.isArray(projectContentFile)
+  ? projectContentFile
+  : projectContentFile.items;
 const skills = skillsJson as SkillsContent;
 const techStack = techStackJson as TechStackItem[];
 const experience = experienceJson as ExperienceItem[];


## `feat(content): JSON schema 파싱 경계 추가`

diff --git a/src/lib/content-loader.ts b/src/lib/content-loader.ts
index b9d64b0..255843b 100644
--- a/src/lib/content-loader.ts
+++ b/src/lib/content-loader.ts
@@ -110,3 +110,23 @@ function jsonPath(path: PropertyKey[]) {
       : `${result}[${JSON.stringify(key)}]`;
   }, "$" );
 }
+
+function parseContentFile<Schema extends z.ZodType>(
+  file: string,
+  schema: Schema,
+  input: unknown,
+): z.output<Schema> {
+  const parsed = schema.safeParse(input);
+
+  if (!parsed.success) {
+    throw new PortfolioContentError(
+      parsed.error.issues.map((issue) => ({
+        file,
+        path: jsonPath(issue.path),
+        message: issue.message,
+      })),
+    );
+  }
+
+  return parsed.data;
+}


## `feat(content): 콘텐츠 파일 schema 파싱 연결`

diff --git a/src/lib/content-loader.ts b/src/lib/content-loader.ts
index 9d75110..b4daf4c 100644
--- a/src/lib/content-loader.ts
+++ b/src/lib/content-loader.ts
@@ -238,3 +238,112 @@ function addInternalRouteIssue({
     }
   }
 }
+
+export function loadPortfolioSource(overrides: PortfolioSourceOverrides = {}) {
+  const input = {
+    site: siteJson,
+    profile: profileJson,
+    projects: projectsJson,
+    presentation: presentationJson,
+    skills: skillsJson,
+    techStack: techStackJson,
+    experience: experienceJson,
+    journey: journeyJson,
+    links: linksJson,
+    contact: contactJson,
+    resume: resumeJson,
+    journeyNarrative: journeyNarrativeJson,
+    interviewMap: interviewMapJson,
+    curation: curationJson,
+    ...overrides,
+  };
+
+  const site = parseContentFile("src/content/site.json", siteContentSchema, input.site);
+  const profile = parseContentFile(
+    "src/content/profile.json",
+    profileContentSchema,
+    input.profile,
+  );
+  const projects = parseContentFile(
+    "src/content/projects.json",
+    projectsContentSchema,
+    input.projects,
+  );
+  const presentation = parseContentFile(
+    "src/content/presentation.json",
+    presentationContentSchema,
+    input.presentation,
+  );
+  const skills = parseContentFile(
+    "src/content/skills.json",
+    skillsContentSchema,
+    input.skills,
+  );
+  const techStack = parseContentFile(
+    "src/content/tech-stack.json",
+    techStackContentSchema,
+    input.techStack,
+  );
+  const experience = parseContentFile(
+    "src/content/experience.json",
+    experienceContentSchema,
+    input.experience,
+  );
+  const journey = parseContentFile(
+    "src/content/journey.json",
+    journeyContentSchema,
+    input.journey,
+  );
+  const links = parseContentFile(
+    "src/content/links.json",
+    linksContentSchema,
+    input.links,
+  );
+  const contact = parseContentFile(
+    "src/content/contact.json",
+    contactContentSchema,
+    input.contact,
+  );
+  const resume = parseContentFile(
+    "src/content/resume.json",
+    resumeContentSchema,
+    input.resume,
+  );
+  const journeyNarrative = parseContentFile(
+    "src/content/journey-narrative.json",
+    journeyNarrativeContentSchema,
+    input.journeyNarrative,
+  );
+  const interviewMap = parseContentFile(
+    "src/content/interview-map.json",
+    interviewMapContentSchema,
+    input.interviewMap,
+  );
+  const curation = parseContentFile(
+    "src/content/curation.json",
+    curationContentSchema,
+    input.curation,
+  );
+
+
+  return {
+    site,
+    profile,
+    projects,
+    presentation,
+    skills,
+    techStack,
+    experience,
+    journey,
+    journeyNarrative,
+    interviewMap,
+    curation,
+    links,
+    contact,
+    resume,
+  };
+}
+
+export const portfolioSource = loadPortfolioSource();
+
+export type PortfolioSource = ReturnType<typeof loadPortfolioSource>;


## `refactor(content): schema 기반 핵심 콘텐츠 타입 연결`

diff --git a/src/lib/portfolio/types.ts b/src/lib/portfolio/types.ts
index c3958eb..49a781c 100644
--- a/src/lib/portfolio/types.ts
+++ b/src/lib/portfolio/types.ts
@@ -1,3 +1,13 @@
+import type { PresentationContentSource } from "../content-schema";
+
+export type {
+  PortfolioProjectSource,
+  ProjectGroup,
+  ProjectMetric,
+  ProjectMetricFilter,
+  ProjectsContentSource,
+} from "../content-schema";
+
 export type NavigationItem = {
   label: string;
   href: string;
@@ -8,6 +18,8 @@ export type SiteContent = {
   description: string;
   language: string;
   brand: string;
+  socialImage?: string;
+  pages?: Record<SitePageId, boolean>;
   navigation: NavigationItem[];
   footer: {
     note: string;
@@ -15,6 +27,15 @@ export type SiteContent = {
   };
 };
 
+export type SitePageId =
+  | "projects"
+  | "about"
+  | "resume"
+  | "contact"
+  | "journey"
+  | "interviewMap"
+  | "curation";
+
 export type ProfilePrinciple = {
   title: string;
   body: string;
@@ -57,6 +78,7 @@ export type ContentLink = {
   envKey?: EnvKey;
   external?: boolean;
   enabled?: boolean;
+  placements?: Array<"hero" | "contact" | "card" | "detail" | "footer">;
 };
 
 export type DeploymentStatus =
@@ -87,6 +109,8 @@ export type PortfolioProject = {
   id: string;
   order: string;
   title: string;
+  groupId: string;
+  tags: string[];
   category: string;
   featured?: boolean;
   enabled?: boolean;
@@ -108,7 +132,14 @@ export type PortfolioProject = {
   results: string[];
 };
 
-export type HomeTemplateId = "design" | "classic";
+export type SiteDesignId =
+  | "design"
+  | "classic"
+  | "editorial"
+  | "brutalist"
+  | "cinematic";
+
+export type HomeTemplateId = SiteDesignId;
 
 export type PresentationTemplate = {
   id: HomeTemplateId;
@@ -175,29 +206,7 @@ export type TerminalPresentation = {
   commands: TerminalCommand[];
 };
 
-export type HomePresentation = {
-  design: {
-    hero: DesignHomeHeroPresentation;
-    sections: HomeSectionId[];
-    featured: SectionCopy;
-  };
-  classic: {
-    hero: ClassicHomeHeroPresentation;
-    sections: HomeSectionId[];
-    featured: SectionCopy;
-    terminal: TerminalPresentation;
-  };
-  shared: {
-    workMap: WorkMapPresentation;
-    technicalFocus: SectionCopy;
-    stack: SectionCopy;
-    journey: SectionCopy;
-    contact: {
-      actionLabel: string;
-      title: string;
-    };
-  };
-};
+export type HomePresentation = PresentationContentSource["home"];
 
 export type ProjectGroupPresentation = {
   category: string;
@@ -209,116 +218,23 @@ export type ProjectPageCountKey =
   | "projectCount"
   | "sourceOnlyCount";
 
-export type ProjectPageContent = {
-  groups: ProjectGroupPresentation[];
-  design: {
-    hero: {
-      title: string;
-      body: string;
-      stats: {
-        visibleEntries: string;
-        archive: string;
-        sourceFirst: string;
-      };
-    };
-    featured: {
-      eyebrow: string;
-      title: string;
-      body: string;
-    };
-    group: {
-      countLabel: string;
-    };
-  };
-  classic: {
-    hero: {
-      eyebrow: string;
-      title: string;
-      body: string;
-      stats: Array<{
-        label: string;
-        countKey: ProjectPageCountKey;
-      }>;
-    };
-    terminal: {
-      ariaLabel: string;
-      title: string;
-      promptUser: string;
-      promptPath: string;
-      command: string;
-      entryLabel: string;
-      maxGroups: number;
-    };
-    selected: {
-      eyebrow: string;
-      title: string;
-      body: string;
-    };
-    grouped: {
-      eyebrow: string;
-      title: string;
-      body: string;
-      countLabel: string;
-    };
-  };
-};
+export type ProjectPageContent = PresentationContentSource["pages"]["projects"];
 
-export type ProjectDetailPageContent = {
-  backLabel: string;
-  sections: Record<
-    | "architecture"
-    | "decisions"
-    | "problem"
-    | "result"
-    | "screenshots"
-    | "solution"
-    | "stack"
-    | "tradeoffs",
-    {
-      eyebrow: string;
-      title: string;
-    }
-  >;
-};
+export type ProjectDetailPageContent =
+  PresentationContentSource["pages"]["projectDetail"];
 
-export type AboutPageContent = {
-  hero: { title: string };
-  principles: { title: string };
-  journey: { title: string };
-  skills: { title: string };
-};
+export type AboutPageContent = PresentationContentSource["pages"]["about"];
 
-export type ResumePageContent = {
-  hero: {
-    title: string;
-    body: string;
-    downloadLabel: string;
-  };
-  summary: { title: string };
-  projects: {
-    title: string;
-    caseStudyLabel: string;
-  };
-  training: { title: string };
-};
+export type JourneyPageContent = PresentationContentSource["pages"]["journey"];
 
-export type ContactPageContent = {
-  availability: { title: string };
-  notes: { title: string };
-};
+export type InterviewMapPageContent =
+  PresentationContentSource["pages"]["interviewMap"];
 
-export type PresentationContent = {
-  defaultHomeTemplate: HomeTemplateId;
-  templates: PresentationTemplate[];
-  home: HomePresentation;
-  pages: {
-    about: AboutPageContent;
-    contact: ContactPageContent;
-    projectDetail: ProjectDetailPageContent;
-    projects: ProjectPageContent;
-    resume: ResumePageContent;
-  };
-};
+export type ResumePageContent = PresentationContentSource["pages"]["resume"];
+
+export type ContactPageContent = PresentationContentSource["pages"]["contact"];
+
+export type PresentationContent = PresentationContentSource;
 
 export type TechStackIcon =
   | "api"
@@ -413,6 +329,7 @@ export type ResumeContent = {
   notes: string[];
 };
 
+
 export type PortfolioContent = {
   site: SiteContent;
   profile: ProfileContent;


## `refactor(content): schema type import 경계 정리`

diff --git a/src/lib/portfolio/types.ts b/src/lib/portfolio/types.ts
index cf77ac3..ff049c3 100644
--- a/src/lib/portfolio/types.ts
+++ b/src/lib/portfolio/types.ts
@@ -2,7 +2,6 @@ import type {
   PresentationContentSource,
   ProjectGroup,
   ProjectMetric,
-  ProjectMetricFilter,
 } from "../content-schema";
 
 export type {
