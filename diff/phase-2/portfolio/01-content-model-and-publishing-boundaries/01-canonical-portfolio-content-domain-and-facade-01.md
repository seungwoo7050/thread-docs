# 정규화된 포트폴리오 콘텐츠 도메인과 Facade

## `feat(content): 사이트와 프로필 콘텐츠 기반 추가`

diff --git a/src/content/profile.json b/src/content/profile.json
new file mode 100644
index 0000000..866d5ef
--- /dev/null
+++ b/src/content/profile.json
@@ -0,0 +1,11 @@
+{
+  "name": "Profile Name",
+  "koreanName": "프로필 이름",
+  "handle": "handle",
+  "role": "Web Developer",
+  "headline": "Portfolio headline placeholder.",
+  "summary": "Portfolio summary placeholder.",
+  "location": "Seoul, Korea",
+  "availability": "Available for portfolio review.",
+  "principles": []
+}
diff --git a/src/content/site.json b/src/content/site.json
new file mode 100644
index 0000000..2c963c2
--- /dev/null
+++ b/src/content/site.json
@@ -0,0 +1,16 @@
+{
+  "title": "Portfolio Site",
+  "description": "Content-driven portfolio placeholder.",
+  "language": "ko",
+  "brand": "Portfolio",
+  "navigation": [
+    {
+      "label": "Home",
+      "href": "/"
+    }
+  ],
+  "footer": {
+    "note": "Content placeholder.",
+    "copyright": "Portfolio"
+  }
+}
diff --git a/src/lib/portfolio/types.ts b/src/lib/portfolio/types.ts
new file mode 100644
index 0000000..c6d7105
--- /dev/null
+++ b/src/lib/portfolio/types.ts
@@ -0,0 +1,39 @@
+export type NavigationItem = {
+  label: string;
+  href: string;
+};
+
+export type SiteContent = {
+  title: string;
+  description: string;
+  language: string;
+  brand: string;
+  navigation: NavigationItem[];
+  footer: {
+    note: string;
+    copyright: string;
+  };
+};
+
+export type ProfilePrinciple = {
+  title: string;
+  body: string;
+};
+
+export type ProfilePhoto = {
+  src: string;
+  alt: string;
+};
+
+export type ProfileContent = {
+  name: string;
+  koreanName: string;
+  handle: string;
+  role: string;
+  headline: string;
+  summary: string;
+  location: string;
+  availability: string;
+  photo?: ProfilePhoto;
+  principles: ProfilePrinciple[];
+};


## `feat(content): 링크와 프로젝트 도메인 정의`

diff --git a/src/content/links.json b/src/content/links.json
new file mode 100644
index 0000000..fe51488
--- /dev/null
+++ b/src/content/links.json
@@ -0,0 +1 @@
+[]
diff --git a/src/content/projects.json b/src/content/projects.json
new file mode 100644
index 0000000..fe51488
--- /dev/null
+++ b/src/content/projects.json
@@ -0,0 +1 @@
+[]
diff --git a/src/lib/portfolio/types.ts b/src/lib/portfolio/types.ts
index c6d7105..da87b4f 100644
--- a/src/lib/portfolio/types.ts
+++ b/src/lib/portfolio/types.ts
@@ -37,3 +37,73 @@ export type ProfileContent = {
   photo?: ProfilePhoto;
   principles: ProfilePrinciple[];
 };
+
+export type LinkType =
+  | "case-study"
+  | "demo"
+  | "email"
+  | "github"
+  | "resume"
+  | "source"
+  | "website";
+
+export type EnvKey = "NEXT_PUBLIC_DASHBOARD_URL" | "NEXT_PUBLIC_SEOUL_APP_URL";
+
+export type ContentLink = {
+  id?: string;
+  type: LinkType;
+  label: string;
+  href: string;
+  envKey?: EnvKey;
+  external?: boolean;
+  enabled?: boolean;
+};
+
+export type DeploymentStatus =
+  | "archived"
+  | "case-study-only"
+  | "live"
+  | "offline"
+  | "private"
+  | "source-only";
+
+export type DeploymentState = {
+  status: DeploymentStatus;
+  label: string;
+  showBadge?: boolean;
+};
+
+export type ProjectImage = {
+  src: string;
+  alt: string;
+};
+
+export type ProjectArchitecture = {
+  summary: string;
+  items: string[];
+};
+
+export type PortfolioProject = {
+  id: string;
+  order: string;
+  title: string;
+  category: string;
+  featured?: boolean;
+  enabled?: boolean;
+  period: string;
+  role: string;
+  summary: string;
+  description: string;
+  deployment: DeploymentState;
+  screenshot: ProjectImage;
+  screenshots: ProjectImage[];
+  stack: string[];
+  links: ContentLink[];
+  highlights: string[];
+  problem: string;
+  solution: string;
+  architecture: ProjectArchitecture;
+  decisions: string[];
+  tradeoffs: string[];
+  results: string[];
+};


## `feat(content): 디자인 홈 표현 모델 추가`

diff --git a/src/content/presentation.json b/src/content/presentation.json
new file mode 100644
index 0000000..fc04de8
--- /dev/null
+++ b/src/content/presentation.json
@@ -0,0 +1,44 @@
+{
+  "defaultHomeTemplate": "design",
+  "templates": [
+    {
+      "id": "design",
+      "label": "Design",
+      "description": "Design template placeholder."
+    },
+    {
+      "id": "classic",
+      "label": "Classic",
+      "description": "Classic template placeholder."
+    }
+  ],
+  "home": {
+    "design": {
+      "hero": {
+        "primaryActionLabel": "Projects",
+        "leadLabel": "Lead case study",
+        "leadActionLabel": "Open",
+        "stats": [
+          {
+            "label": "Product",
+            "countKey": "productCount"
+          },
+          {
+            "label": "42 archive",
+            "countKey": "curriculumCount"
+          },
+          {
+            "label": "Reliability",
+            "countKey": "reliabilityCount"
+          }
+        ]
+      },
+      "sections": [],
+      "featured": {
+        "title": "Featured",
+        "body": "Featured placeholder.",
+        "actionLabel": "View all projects"
+      }
+    }
+  }
+}
diff --git a/src/lib/portfolio/types.ts b/src/lib/portfolio/types.ts
index da87b4f..5a96e85 100644
--- a/src/lib/portfolio/types.ts
+++ b/src/lib/portfolio/types.ts
@@ -107,3 +107,53 @@ export type PortfolioProject = {
   tradeoffs: string[];
   results: string[];
 };
+
+export type HomeTemplateId = "design" | "classic";
+
+export type PresentationTemplate = {
+  id: HomeTemplateId;
+  label: string;
+  description: string;
+};
+
+export type HomeSectionId =
+  | "contact"
+  | "featured"
+  | "journey"
+  | "stack"
+  | "technicalFocus"
+  | "workMap";
+
+export type SectionCopy = {
+  actionLabel?: string;
+  title: string;
+  body?: string;
+};
+
+export type WorkMapCountKey =
+  | "curriculumCount"
+  | "productCount"
+  | "reliabilityCount";
+
+export type WorkMapCard = {
+  id: string;
+  label: string;
+  body: string;
+  countKey: WorkMapCountKey;
+};
+
+export type WorkMapPresentation = SectionCopy & {
+  cards: WorkMapCard[];
+};
+
+export type HomeStatPresentation = {
+  label: string;
+  countKey: WorkMapCountKey;
+};
+
+export type DesignHomeHeroPresentation = {
+  primaryActionLabel: string;
+  leadLabel: string;
+  leadActionLabel: string;
+  stats: HomeStatPresentation[];
+};


## `feat(content): 클래식과 공용 홈 표현 추가`

diff --git a/src/content/presentation.json b/src/content/presentation.json
index fc04de8..f565c05 100644
--- a/src/content/presentation.json
+++ b/src/content/presentation.json
@@ -39,6 +39,73 @@
         "body": "Featured placeholder.",
         "actionLabel": "View all projects"
       }
+    },
+    "classic": {
+      "hero": {
+        "primaryActionLabel": "Projects"
+      },
+      "sections": [],
+      "featured": {
+        "title": "Selected work",
+        "body": "Selected work placeholder.",
+        "actionLabel": "View all projects"
+      },
+      "terminal": {
+        "title": "portfolio.sh",
+        "bootLine": "boot placeholder",
+        "promptUser": "user",
+        "promptPath": "~/portfolio",
+        "commands": [
+          {
+            "command": "status",
+            "output": [
+              "placeholder"
+            ]
+          }
+        ]
+      }
+    },
+    "shared": {
+      "workMap": {
+        "title": "Work map",
+        "body": "Work map placeholder.",
+        "cards": [
+          {
+            "id": "product",
+            "label": "Product",
+            "body": "Product placeholder.",
+            "countKey": "productCount"
+          },
+          {
+            "id": "curriculum",
+            "label": "Curriculum",
+            "body": "Curriculum placeholder.",
+            "countKey": "curriculumCount"
+          },
+          {
+            "id": "reliability",
+            "label": "Reliability",
+            "body": "Reliability placeholder.",
+            "countKey": "reliabilityCount"
+          }
+        ]
+      },
+      "technicalFocus": {
+        "title": "Technical focus",
+        "body": "Technical focus placeholder."
+      },
+      "stack": {
+        "title": "Stack",
+        "body": "Stack placeholder."
+      },
+      "journey": {
+        "title": "Journey",
+        "body": "Journey placeholder."
+      },
+      "contact": {
+        "title": "Contact",
+        "actionLabel": "Contact page"
+      }
     }
   }
 }
diff --git a/src/lib/portfolio/types.ts b/src/lib/portfolio/types.ts
index 5a96e85..16fe86d 100644
--- a/src/lib/portfolio/types.ts
+++ b/src/lib/portfolio/types.ts
@@ -157,3 +157,44 @@ export type DesignHomeHeroPresentation = {
   leadActionLabel: string;
   stats: HomeStatPresentation[];
 };
+
+export type ClassicHomeHeroPresentation = {
+  primaryActionLabel: string;
+};
+
+export type TerminalCommand = {
+  command: string;
+  output: string[];
+};
+
+export type TerminalPresentation = {
+  title: string;
+  bootLine: string;
+  promptUser: string;
+  promptPath: string;
+  commands: TerminalCommand[];
+};
+
+export type HomePresentation = {
+  design: {
+    hero: DesignHomeHeroPresentation;
+    sections: HomeSectionId[];
+    featured: SectionCopy;
+  };
+  classic: {
+    hero: ClassicHomeHeroPresentation;
+    sections: HomeSectionId[];
+    featured: SectionCopy;
+    terminal: TerminalPresentation;
+  };
+  shared: {
+    workMap: WorkMapPresentation;
+    technicalFocus: SectionCopy;
+    stack: SectionCopy;
+    journey: SectionCopy;
+    contact: {
+      actionLabel: string;
+      title: string;
+    };
+  };
+};


## `feat(content): 프로젝트 목록 표현 계약 정의`

diff --git a/src/lib/portfolio/types.ts b/src/lib/portfolio/types.ts
index 16fe86d..3e491ec 100644
--- a/src/lib/portfolio/types.ts
+++ b/src/lib/portfolio/types.ts
@@ -198,3 +198,67 @@ export type HomePresentation = {
     };
   };
 };
+
+export type ProjectGroupPresentation = {
+  category: string;
+  body: string;
+};
+
+export type ProjectPageCountKey =
+  | "curriculumCount"
+  | "projectCount"
+  | "sourceOnlyCount";
+
+export type ProjectPageContent = {
+  groups: ProjectGroupPresentation[];
+  design: {
+    hero: {
+      title: string;
+      body: string;
+      stats: {
+        visibleEntries: string;
+        archive: string;
+        sourceFirst: string;
+      };
+    };
+    featured: {
+      eyebrow: string;
+      title: string;
+      body: string;
+    };
+    group: {
+      countLabel: string;
+    };
+  };
+  classic: {
+    hero: {
+      eyebrow: string;
+      title: string;
+      body: string;
+      stats: Array<{
+        label: string;
+        countKey: ProjectPageCountKey;
+      }>;
+    };
+    terminal: {
+      ariaLabel: string;
+      title: string;
+      promptUser: string;
+      promptPath: string;
+      command: string;
+      entryLabel: string;
+      maxGroups: number;
+    };
+    selected: {
+      eyebrow: string;
+      title: string;
+      body: string;
+    };
+    grouped: {
+      eyebrow: string;
+      title: string;
+      body: string;
+      countLabel: string;
+    };
+  };
+};


## `feat(content): 보조 페이지 표현 계약 정의`

diff --git a/src/lib/portfolio/types.ts b/src/lib/portfolio/types.ts
index 3e491ec..f354dac 100644
--- a/src/lib/portfolio/types.ts
+++ b/src/lib/portfolio/types.ts
@@ -262,3 +262,60 @@ export type ProjectPageContent = {
     };
   };
 };
+
+export type ProjectDetailPageContent = {
+  backLabel: string;
+  sections: Record<
+    | "architecture"
+    | "decisions"
+    | "problem"
+    | "result"
+    | "screenshots"
+    | "solution"
+    | "stack"
+    | "tradeoffs",
+    {
+      eyebrow: string;
+      title: string;
+    }
+  >;
+};
+
+export type AboutPageContent = {
+  hero: { title: string };
+  principles: { title: string };
+  journey: { title: string };
+  skills: { title: string };
+};
+
+export type ResumePageContent = {
+  hero: {
+    title: string;
+    body: string;
+    downloadLabel: string;
+  };
+  summary: { title: string };
+  projects: {
+    title: string;
+    caseStudyLabel: string;
+  };
+  training: { title: string };
+};
+
+export type ContactPageContent = {
+  availability: { title: string };
+  notes: { title: string };
+};
+
+export type PresentationContent = {
+  defaultHomeTemplate: HomeTemplateId;
+  templates: PresentationTemplate[];
+  home: HomePresentation;
+  pages: {
+    about: AboutPageContent;
+    contact: ContactPageContent;
+    projectDetail: ProjectDetailPageContent;
+    projects: ProjectPageContent;
+    resume: ResumePageContent;
+  };
+};


## `feat(content): 기술과 여정 콘텐츠 모델 추가`

diff --git a/src/content/experience.json b/src/content/experience.json
new file mode 100644
index 0000000..fe51488
--- /dev/null
+++ b/src/content/experience.json
@@ -0,0 +1 @@
+[]
diff --git a/src/content/journey.json b/src/content/journey.json
new file mode 100644
index 0000000..fe51488
--- /dev/null
+++ b/src/content/journey.json
@@ -0,0 +1 @@
+[]
diff --git a/src/content/skills.json b/src/content/skills.json
new file mode 100644
index 0000000..c04e1f8
--- /dev/null
+++ b/src/content/skills.json
@@ -0,0 +1,4 @@
+{
+  "focusAreas": [],
+  "groups": []
+}
diff --git a/src/content/tech-stack.json b/src/content/tech-stack.json
new file mode 100644
index 0000000..fe51488
--- /dev/null
+++ b/src/content/tech-stack.json
@@ -0,0 +1 @@
+[]
diff --git a/src/lib/portfolio/types.ts b/src/lib/portfolio/types.ts
index f354dac..40fa53a 100644
--- a/src/lib/portfolio/types.ts
+++ b/src/lib/portfolio/types.ts
@@ -319,3 +319,67 @@ export type PresentationContent = {
     resume: ResumePageContent;
   };
 };
+
+export type TechStackIcon =
+  | "api"
+  | "box"
+  | "c"
+  | "check"
+  | "cmake"
+  | "cplusplus"
+  | "database"
+  | "docker"
+  | "eslint"
+  | "flow"
+  | "json"
+  | "nextjs"
+  | "nodejs"
+  | "playwright"
+  | "postgresql"
+  | "prisma"
+  | "react"
+  | "redis"
+  | "shield"
+  | "tailwind"
+  | "terminal"
+  | "tool"
+  | "typescript"
+  | "vitest";
+
+export type TechStackItem = {
+  id: string;
+  label: string;
+  icon: TechStackIcon;
+  color: string;
+};
+
+export type SkillFocusArea = {
+  title: string;
+  body: string;
+};
+
+export type SkillGroup = {
+  title: string;
+  items: string[];
+};
+
+export type SkillsContent = {
+  focusAreas: SkillFocusArea[];
+  groups: SkillGroup[];
+};
+
+export type ExperienceItem = {
+  period: string;
+  title: string;
+  body: string;
+};
+
+export type JourneyItem = {
+  date: string;
+  endDate: string | null;
+  title: string;
+  category: string;
+  body: string;
+  projectId: string | null;
+  sourcePath: string | null;
+};


## `feat(content): 연락과 이력 집계 모델 완성`

diff --git a/src/content/contact.json b/src/content/contact.json
new file mode 100644
index 0000000..ed4efe2
--- /dev/null
+++ b/src/content/contact.json
@@ -0,0 +1,7 @@
+{
+  "title": "Contact",
+  "intro": "Contact intro placeholder.",
+  "availability": "Availability placeholder.",
+  "preferred": [],
+  "notes": []
+}
diff --git a/src/content/resume.json b/src/content/resume.json
new file mode 100644
index 0000000..7c793d7
--- /dev/null
+++ b/src/content/resume.json
@@ -0,0 +1,8 @@
+{
+  "downloadUrl": null,
+  "summary": [],
+  "projectIds": [],
+  "training": [],
+  "education": [],
+  "notes": []
+}
diff --git a/src/lib/portfolio/types.ts b/src/lib/portfolio/types.ts
index 40fa53a..c3958eb 100644
--- a/src/lib/portfolio/types.ts
+++ b/src/lib/portfolio/types.ts
@@ -383,3 +383,51 @@ export type JourneyItem = {
   projectId: string | null;
   sourcePath: string | null;
 };
+
+export type ContactContent = {
+  title: string;
+  intro: string;
+  availability: string;
+  preferred: string[];
+  notes: string[];
+};
+
+export type ResumeTraining = {
+  name: string;
+  period: string;
+  description: string;
+};
+
+export type ResumeEducation = {
+  name: string;
+  period: string;
+  description: string;
+};
+
+export type ResumeContent = {
+  downloadUrl: string | null;
+  summary: string[];
+  projectIds: string[];
+  training: ResumeTraining[];
+  education: ResumeEducation[];
+  notes: string[];
+};
+
+export type PortfolioContent = {
+  site: SiteContent;
+  profile: ProfileContent;
+  projects: PortfolioProject[];
+  presentation: PresentationContent;
+  skills: SkillsContent;
+  techStack: TechStackItem[];
+  experience: ExperienceItem[];
+  journey: JourneyItem[];
+  links: ContentLink[];
+  contact: ContactContent;
+  resume: ResumeContent;
+};
+
+export type PortfolioEnv = Partial<Record<EnvKey, string | undefined>>;
+export type RouteSearchParams = Promise<
+  Record<string, string | string[] | undefined>
+>;


## `feat(content): 정적 포트폴리오 콘텐츠 로딩`

diff --git a/src/lib/portfolio/content.ts b/src/lib/portfolio/content.ts
new file mode 100644
index 0000000..2fafe3f
--- /dev/null
+++ b/src/lib/portfolio/content.ts
@@ -0,0 +1,25 @@
+import experienceJson from "@/content/experience.json";
+import presentationJson from "@/content/presentation.json";
+import profileJson from "@/content/profile.json";
+import projectsJson from "@/content/projects.json";
+import siteJson from "@/content/site.json";
+import skillsJson from "@/content/skills.json";
+import techStackJson from "@/content/tech-stack.json";
+import type {
+  ExperienceItem,
+  PortfolioProject,
+  PresentationContent,
+  ProfileContent,
+  SiteContent,
+  SkillsContent,
+  TechStackItem,
+} from "./types";
+
+const site = siteJson as SiteContent;
+const profile = profileJson as ProfileContent;
+export const portfolioPresentation =
+  presentationJson as PresentationContent;
+const projects = projectsJson as PortfolioProject[];
+const skills = skillsJson as SkillsContent;
+const techStack = techStackJson as TechStackItem[];
+const experience = experienceJson as ExperienceItem[];


## `feat(content): 여정 정렬과 콘텐츠 인덱스 구성`

diff --git a/src/lib/portfolio/content.ts b/src/lib/portfolio/content.ts
index 2fafe3f..94190a8 100644
--- a/src/lib/portfolio/content.ts
+++ b/src/lib/portfolio/content.ts
@@ -1,15 +1,23 @@
+import contactJson from "@/content/contact.json";
 import experienceJson from "@/content/experience.json";
+import journeyJson from "@/content/journey.json";
+import linksJson from "@/content/links.json";
 import presentationJson from "@/content/presentation.json";
 import profileJson from "@/content/profile.json";
 import projectsJson from "@/content/projects.json";
+import resumeJson from "@/content/resume.json";
 import siteJson from "@/content/site.json";
 import skillsJson from "@/content/skills.json";
 import techStackJson from "@/content/tech-stack.json";
 import type {
+  ContactContent,
+  ContentLink,
   ExperienceItem,
+  JourneyItem,
   PortfolioProject,
   PresentationContent,
   ProfileContent,
+  ResumeContent,
   SiteContent,
   SkillsContent,
   TechStackItem,
@@ -23,3 +31,22 @@ const projects = projectsJson as PortfolioProject[];
 const skills = skillsJson as SkillsContent;
 const techStack = techStackJson as TechStackItem[];
 const experience = experienceJson as ExperienceItem[];
+const journey = (journeyJson as JourneyItem[]).slice().sort((left, right) => {
+  const dateOrder = left.date.localeCompare(right.date);
+
+  if (dateOrder !== 0) {
+    return dateOrder;
+  }
+
+  return left.title.localeCompare(right.title);
+});
+const links = linksJson as ContentLink[];
+const contact = contactJson as ContactContent;
+const resume = resumeJson as ResumeContent;
+export const portfolioTechStackById = new Map(
+  techStack.map((item) => [item.id, item]),
+);
+
+export function getEnabledLinks(contentLinks: ContentLink[] = links) {
+  return contentLinks.filter((link) => link.enabled !== false);
+}


## `feat(content): 환경 링크를 반영한 콘텐츠 집계`

diff --git a/src/lib/portfolio/content.ts b/src/lib/portfolio/content.ts
index 94190a8..80104ff 100644
--- a/src/lib/portfolio/content.ts
+++ b/src/lib/portfolio/content.ts
@@ -14,6 +14,8 @@ import type {
   ContentLink,
   ExperienceItem,
   JourneyItem,
+  PortfolioContent,
+  PortfolioEnv,
   PortfolioProject,
   PresentationContent,
   ProfileContent,
@@ -50,3 +52,42 @@ export const portfolioTechStackById = new Map(
 export function getEnabledLinks(contentLinks: ContentLink[] = links) {
   return contentLinks.filter((link) => link.enabled !== false);
 }
+
+function withEnvHref(link: ContentLink, env: PortfolioEnv): ContentLink {
+  const envValue = link.envKey ? env[link.envKey]?.trim() : undefined;
+
+  return {
+    ...link,
+    href: envValue && envValue.length > 0 ? envValue : link.href,
+  };
+}
+
+export function getPortfolioContent(
+  env: PortfolioEnv = {
+    NEXT_PUBLIC_DASHBOARD_URL: process.env.NEXT_PUBLIC_DASHBOARD_URL,
+    NEXT_PUBLIC_SEOUL_APP_URL: process.env.NEXT_PUBLIC_SEOUL_APP_URL,
+  },
+): PortfolioContent {
+  const resolvedProjects = projects
+    .filter((project) => project.enabled !== false)
+    .map((project) => ({
+      ...project,
+      links: project.links
+        .filter((link) => link.enabled !== false)
+        .map((link) => withEnvHref(link, env)),
+    }));
+
+  return {
+    site,
+    profile,
+    projects: resolvedProjects,
+    presentation: portfolioPresentation,
+    skills,
+    techStack,
+    experience,
+    journey,
+    links: getEnabledLinks(),
+    contact,
+    resume,
+  };
+}


