# 교차 파일 참조·라우트·자산 무결성 검증

## `feat(content): 콘텐츠 validation 오류 모델 추가`

diff --git a/src/lib/content-loader.ts b/src/lib/content-loader.ts
new file mode 100644
index 0000000..d65b299
--- /dev/null
+++ b/src/lib/content-loader.ts
@@ -0,0 +1,95 @@
+import contactJson from "@/content/contact.json";
+import curationJson from "@/content/curation.json";
+import experienceJson from "@/content/experience.json";
+import interviewMapJson from "@/content/interview-map.json";
+import journeyJson from "@/content/journey.json";
+import journeyNarrativeJson from "@/content/journey-narrative.json";
+import linksJson from "@/content/links.json";
+import presentationJson from "@/content/presentation.json";
+import profileJson from "@/content/profile.json";
+import projectsJson from "@/content/projects.json";
+import resumeJson from "@/content/resume.json";
+import siteJson from "@/content/site.json";
+import skillsJson from "@/content/skills.json";
+import techStackJson from "@/content/tech-stack.json";
+import { z } from "zod";
+
+import {
+  contactContentSchema,
+  curationContentSchema,
+  experienceContentSchema,
+  interviewMapContentSchema,
+  journeyContentSchema,
+  journeyNarrativeContentSchema,
+  linksContentSchema,
+  presentationContentSchema,
+  profileContentSchema,
+  projectsContentSchema,
+  resumeContentSchema,
+  siteContentSchema,
+  skillsContentSchema,
+  techStackContentSchema,
+} from "./content-schema";
+
+export type ContentValidationIssue = {
+  file: string;
+  path: string;
+  message: string;
+};
+
+const supportedDesignIdList = [
+  "design",
+  "classic",
+] as const;
+const supportedDesignIds = new Set<string>(supportedDesignIdList);
+
+type NavigablePageId =
+  | "projects"
+  | "about"
+  | "resume"
+  | "contact"
+  | "journey"
+  | "interviewMap";
+
+const internalNavigationPages = new Map<string, NavigablePageId>([
+  ["/projects", "projects"],
+  ["/about", "about"],
+  ["/resume", "resume"],
+  ["/contact", "contact"],
+  ["/journey", "journey"],
+  ["/interview-map", "interviewMap"],
+] as const);
+
+export type PortfolioSourceOverrides = Partial<
+  Record<
+    | "site"
+    | "profile"
+    | "projects"
+    | "presentation"
+    | "skills"
+    | "techStack"
+    | "experience"
+    | "journey"
+    | "links"
+    | "contact"
+    | "resume"
+    | "journeyNarrative"
+    | "interviewMap"
+    | "curation",
+    unknown
+  >
+>;
+
+export class PortfolioContentError extends Error {
+  readonly issues: ContentValidationIssue[];
+
+  constructor(issues: ContentValidationIssue[]) {
+    const details = issues
+      .map(({ file, path, message }) => `- ${file}:${path} ${message}`)
+      .join("\n");
+
+    super(`Portfolio content validation failed:\n${details}`);
+    this.name = "PortfolioContentError";
+    this.issues = issues;
+  }
+}


## `feat(content): JSON 경로 진단 추가`

diff --git a/src/lib/content-loader.ts b/src/lib/content-loader.ts
index d65b299..b9d64b0 100644
--- a/src/lib/content-loader.ts
+++ b/src/lib/content-loader.ts
@@ -93,3 +93,20 @@ export class PortfolioContentError extends Error {
     this.issues = issues;
   }
 }
+
+function jsonPath(path: PropertyKey[]) {
+  if (path.length === 0) {
+    return "$";
+  }
+
+  return path.reduce<string>((result, segment) => {
+    if (typeof segment === "number") {
+      return `${result}[${segment}]`;
+    }
+
+    const key = String(segment);
+    return /^[a-zA-Z_$][\w$]*$/.test(key)
+      ? `${result}.${key}`
+      : `${result}[${JSON.stringify(key)}]`;
+  }, "$" );
+}


## `feat(content): 중복과 참조 진단 helper 추가`

diff --git a/src/lib/content-loader.ts b/src/lib/content-loader.ts
index 255843b..1a4ddce 100644
--- a/src/lib/content-loader.ts
+++ b/src/lib/content-loader.ts
@@ -130,3 +130,51 @@ function parseContentFile<Schema extends z.ZodType>(
 
   return parsed.data;
 }
+
+function findDuplicates(values: string[]) {
+  const seen = new Set<string>();
+  const duplicates = new Set<string>();
+
+  for (const value of values) {
+    if (seen.has(value)) {
+      duplicates.add(value);
+    }
+
+    seen.add(value);
+  }
+
+  return duplicates;
+}
+
+function addDuplicateIssues(
+  issues: ContentValidationIssue[],
+  file: string,
+  path: string,
+  label: string,
+  values: string[],
+) {
+  for (const duplicate of findDuplicates(values)) {
+    issues.push({
+      file,
+      path,
+      message: `Duplicate ${label} "${duplicate}".`,
+    });
+  }
+}
+
+function addMissingReferenceIssue(
+  issues: ContentValidationIssue[],
+  file: string,
+  path: string,
+  referenceType: string,
+  reference: string,
+  knownReferences: Set<string>,
+) {
+  if (!knownReferences.has(reference)) {
+    issues.push({
+      file,
+      path,
+      message: `Unknown ${referenceType} "${reference}".`,
+    });
+  }
+}


## `feat(content): 내부 route 참조 검증 추가`

diff --git a/src/lib/content-loader.ts b/src/lib/content-loader.ts
index 1a4ddce..9d75110 100644
--- a/src/lib/content-loader.ts
+++ b/src/lib/content-loader.ts
@@ -178,3 +178,63 @@ function addMissingReferenceIssue(
     });
   }
 }
+
+function addInternalRouteIssue({
+  enabledProjectIds,
+  file,
+  href,
+  issues,
+  path,
+  routeKind,
+  site,
+}: {
+  enabledProjectIds: Set<string>;
+  file: string;
+  href: string;
+  issues: ContentValidationIssue[];
+  path: string;
+  routeKind: "link" | "navigation";
+  site: z.output<typeof siteContentSchema>;
+}) {
+  if (!href.startsWith("/") || href.startsWith("//")) {
+    return;
+  }
+
+  const pathname = new URL(href, "https://portfolio.invalid").pathname;
+  if (pathname === "/") {
+    return;
+  }
+
+  const projectMatch = pathname.match(/^\/projects\/([^/]+)\/?$/);
+  const pageId = projectMatch
+    ? "projects"
+    : internalNavigationPages.get(pathname);
+
+  if (!pageId) {
+    issues.push({
+      file,
+      path,
+      message: `Unsupported internal ${routeKind} route "${pathname}".`,
+    });
+    return;
+  }
+
+  if (site.pages?.[pageId] === false) {
+    issues.push({
+      file,
+      path,
+      message: `Internal ${routeKind} route "${pathname}" points to disabled page "${pageId}".`,
+    });
+  }
+
+  if (projectMatch) {
+    const projectId = decodeURIComponent(projectMatch[1]);
+    if (!enabledProjectIds.has(projectId)) {
+      issues.push({
+        file,
+        path,
+        message: `Internal ${routeKind} route "${pathname}" points to unknown or disabled project "${projectId}".`,
+      });
+    }
+  }
+}


## `feat(content): 콘텐츠 식별자 중복 검증 추가`

diff --git a/src/lib/content-loader.ts b/src/lib/content-loader.ts
index b4daf4c..67bbc4d 100644
--- a/src/lib/content-loader.ts
+++ b/src/lib/content-loader.ts
@@ -325,6 +325,112 @@ export function loadPortfolioSource(overrides: PortfolioSourceOverrides = {}) {
     input.curation,
   );
 
+  const issues: ContentValidationIssue[] = [];
+  const groupIds = new Set(projects.groups.map((group) => group.id));
+  const enabledProjectIds = new Set(
+    projects.items
+      .filter((project) => project.enabled !== false)
+      .map((project) => project.id),
+  );
+  const stackIds = new Set(techStack.map((item) => item.id));
+  const tagIds = new Set(
+    projects.items
+      .filter((project) => project.enabled !== false)
+      .flatMap((project) => project.tags),
+  );
+  const enabledLinkIds = new Set(
+    links.flatMap((link) =>
+      link.id !== undefined && link.enabled !== false ? [link.id] : [],
+    ),
+  );
+
+  addDuplicateIssues(
+    issues,
+    "src/content/projects.json",
+    "$.groups",
+    "project group id",
+    projects.groups.map((group) => group.id),
+  );
+  addDuplicateIssues(
+    issues,
+    "src/content/projects.json",
+    "$.groups",
+    "project group order",
+    projects.groups.map((group) => String(group.order)),
+  );
+  addDuplicateIssues(
+    issues,
+    "src/content/projects.json",
+    "$.metrics",
+    "project metric id",
+    projects.metrics.map((metric) => metric.id),
+  );
+  addDuplicateIssues(
+    issues,
+    "src/content/projects.json",
+    "$.items",
+    "project id",
+    projects.items.map((project) => project.id),
+  );
+  addDuplicateIssues(
+    issues,
+    "src/content/projects.json",
+    "$.items",
+    "project order",
+    projects.items.map((project) => project.order),
+  );
+  addDuplicateIssues(
+    issues,
+    "src/content/tech-stack.json",
+    "$",
+    "technology id",
+    techStack.map((item) => item.id),
+  );
+  addDuplicateIssues(
+    issues,
+    "src/content/links.json",
+    "$",
+    "link id",
+    links.flatMap((link) => (link.id === undefined ? [] : [link.id])),
+  );
+  addDuplicateIssues(
+    issues,
+    "src/content/journey-narrative.json",
+    "$.milestones",
+    "milestone id",
+    journeyNarrative.milestones.map((milestone) => milestone.id),
+  );
+  addDuplicateIssues(
+    issues,
+    "src/content/interview-map.json",
+    "$.tracks",
+    "interview track id",
+    interviewMap.tracks.map((track) => track.id),
+  );
+  addDuplicateIssues(
+    issues,
+    "src/content/curation.json",
+    "$.categories",
+    "curation category id",
+    curation.categories.map((category) => category.id),
+  );
+  addDuplicateIssues(
+    issues,
+    "src/content/presentation.json",
+    "$.templates",
+    "site design id",
+    presentation.templates.map((template) => template.id),
+  );
+  addDuplicateIssues(
+    issues,
+    "src/content/site.json",
+    "$.navigation",
+    "navigation href",
+    site.navigation.map((item) => item.href),
+  );
+  if (issues.length > 0) {
+    throw new PortfolioContentError(issues);
+  }
 
   return {
     site,


## `feat(content): 지원 디자인 구성 검증 추가`

diff --git a/src/lib/content-loader.ts b/src/lib/content-loader.ts
index 67bbc4d..39a8d5b 100644
--- a/src/lib/content-loader.ts
+++ b/src/lib/content-loader.ts
@@ -428,6 +428,41 @@ export function loadPortfolioSource(overrides: PortfolioSourceOverrides = {}) {
     "navigation href",
     site.navigation.map((item) => item.href),
   );
+
+  if (
+    !presentation.templates.some(
+      (template) => template.id === presentation.defaultHomeTemplate,
+    )
+  ) {
+    issues.push({
+      file: "src/content/presentation.json",
+      path: "$.defaultHomeTemplate",
+      message: `Default site design "${presentation.defaultHomeTemplate}" is not listed in templates.`,
+    });
+  }
+
+  presentation.templates.forEach((template, templateIndex) => {
+    if (!supportedDesignIds.has(template.id)) {
+      issues.push({
+        file: "src/content/presentation.json",
+        path: `$.templates[${templateIndex}].id`,
+        message: `Unsupported site design "${template.id}".`,
+      });
+    }
+  });
+
+  const configuredDesignIds = new Set(
+    presentation.templates.map((template) => template.id),
+  );
+  supportedDesignIdList.forEach((designId) => {
+    if (!configuredDesignIds.has(designId)) {
+      issues.push({
+        file: "src/content/presentation.json",
+        path: "$.templates",
+        message: `Missing supported site design "${designId}".`,
+      });
+    }
+  });
   if (issues.length > 0) {
     throw new PortfolioContentError(issues);
   }


## `feat(content): 사이트와 링크 route 참조 검증 추가`

diff --git a/src/lib/content-loader.ts b/src/lib/content-loader.ts
index 39a8d5b..c006b3e 100644
--- a/src/lib/content-loader.ts
+++ b/src/lib/content-loader.ts
@@ -463,6 +463,30 @@ export function loadPortfolioSource(overrides: PortfolioSourceOverrides = {}) {
       });
     }
   });
+
+  site.navigation.forEach((item, index) =>
+    addInternalRouteIssue({
+      enabledProjectIds,
+      file: "src/content/site.json",
+      href: item.href,
+      issues,
+      path: `$.navigation[${index}].href`,
+      routeKind: "navigation",
+      site,
+    }),
+  );
+
+  links.forEach((link, index) =>
+    addInternalRouteIssue({
+      enabledProjectIds,
+      file: "src/content/links.json",
+      href: link.href,
+      issues,
+      path: `$[${index}].href`,
+      routeKind: "link",
+      site,
+    }),
+  );
   if (issues.length > 0) {
     throw new PortfolioContentError(issues);
   }


## `feat(content): 프로젝트 내부 참조 검증 추가`

diff --git a/src/lib/content-loader.ts b/src/lib/content-loader.ts
index c006b3e..b6b0315 100644
--- a/src/lib/content-loader.ts
+++ b/src/lib/content-loader.ts
@@ -487,6 +487,56 @@ export function loadPortfolioSource(overrides: PortfolioSourceOverrides = {}) {
       site,
     }),
   );
+
+  projects.items.forEach((project, projectIndex) => {
+    addMissingReferenceIssue(
+      issues,
+      "src/content/projects.json",
+      `$.items[${projectIndex}].groupId`,
+      "project group id",
+      project.groupId,
+      groupIds,
+    );
+
+    addDuplicateIssues(
+      issues,
+      "src/content/projects.json",
+      `$.items[${projectIndex}].tags`,
+      "project tag",
+      project.tags,
+    );
+    addDuplicateIssues(
+      issues,
+      "src/content/projects.json",
+      `$.items[${projectIndex}].stack`,
+      "technology reference",
+      project.stack,
+    );
+
+    project.stack.forEach((stackId, stackIndex) =>
+      addMissingReferenceIssue(
+        issues,
+        "src/content/projects.json",
+        `$.items[${projectIndex}].stack[${stackIndex}]`,
+        "technology id",
+        stackId,
+        stackIds,
+      ),
+    );
+
+    project.links.forEach((link, linkIndex) =>
+      addInternalRouteIssue({
+        enabledProjectIds,
+        file: "src/content/projects.json",
+        href: link.href,
+        issues,
+        path: `$.items[${projectIndex}].links[${linkIndex}].href`,
+        routeKind: "link",
+        site,
+      }),
+    );
+
+  });
   if (issues.length > 0) {
     throw new PortfolioContentError(issues);
   }


## `feat(content): 지표와 Resume 참조 검증 추가`

diff --git a/src/lib/content-loader.ts b/src/lib/content-loader.ts
index b6b0315..6eed9a9 100644
--- a/src/lib/content-loader.ts
+++ b/src/lib/content-loader.ts
@@ -537,6 +537,50 @@ export function loadPortfolioSource(overrides: PortfolioSourceOverrides = {}) {
     );
 
   });
+
+  projects.metrics.forEach((metric, metricIndex) => {
+    metric.filter?.projectIds?.forEach((projectId, projectIndex) =>
+      addMissingReferenceIssue(
+        issues,
+        "src/content/projects.json",
+        `$.metrics[${metricIndex}].filter.projectIds[${projectIndex}]`,
+        "project id",
+        projectId,
+        enabledProjectIds,
+      ),
+    );
+    metric.filter?.groupIds?.forEach((groupId, groupIndex) =>
+      addMissingReferenceIssue(
+        issues,
+        "src/content/projects.json",
+        `$.metrics[${metricIndex}].filter.groupIds[${groupIndex}]`,
+        "project group id",
+        groupId,
+        groupIds,
+      ),
+    );
+    metric.filter?.tags?.forEach((tag, tagIndex) =>
+      addMissingReferenceIssue(
+        issues,
+        "src/content/projects.json",
+        `$.metrics[${metricIndex}].filter.tags[${tagIndex}]`,
+        "project tag",
+        tag,
+        tagIds,
+      ),
+    );
+  });
+
+  resume.projectIds.forEach((projectId, index) =>
+    addMissingReferenceIssue(
+      issues,
+      "src/content/resume.json",
+      `$.projectIds[${index}]`,
+      "project id",
+      projectId,
+      enabledProjectIds,
+    ),
+  );
   if (issues.length > 0) {
     throw new PortfolioContentError(issues);
   }


## `feat(content): 여정과 Interview 참조 검증 추가`

diff --git a/src/lib/content-loader.ts b/src/lib/content-loader.ts
index 6eed9a9..4030240 100644
--- a/src/lib/content-loader.ts
+++ b/src/lib/content-loader.ts
@@ -581,6 +581,44 @@ export function loadPortfolioSource(overrides: PortfolioSourceOverrides = {}) {
       enabledProjectIds,
     ),
   );
+  journey.forEach((item, index) => {
+    if (item.projectId !== null) {
+      addMissingReferenceIssue(
+        issues,
+        "src/content/journey.json",
+        `$[${index}].projectId`,
+        "project id",
+        item.projectId,
+        enabledProjectIds,
+      );
+    }
+  });
+  journeyNarrative.milestones.forEach((milestone, milestoneIndex) =>
+    milestone.anchorProjectIds.forEach((projectId, projectIndex) =>
+      addMissingReferenceIssue(
+        issues,
+        "src/content/journey-narrative.json",
+        `$.milestones[${milestoneIndex}].anchorProjectIds[${projectIndex}]`,
+        "project id",
+        projectId,
+        enabledProjectIds,
+      ),
+    ),
+  );
+  interviewMap.tracks.forEach((track, trackIndex) =>
+    track.items.forEach((item, itemIndex) =>
+      item.answers.forEach((answer, answerIndex) =>
+        addMissingReferenceIssue(
+          issues,
+          "src/content/interview-map.json",
+          `$.tracks[${trackIndex}].items[${itemIndex}].answers[${answerIndex}].projectId`,
+          "project id",
+          answer.projectId,
+          enabledProjectIds,
+        ),
+      ),
+    ),
+  );
   if (issues.length > 0) {
     throw new PortfolioContentError(issues);
   }


## `feat(content): 큐레이션과 연락 참조 검증 추가`

diff --git a/src/lib/content-loader.ts b/src/lib/content-loader.ts
index 4030240..f5f20e2 100644
--- a/src/lib/content-loader.ts
+++ b/src/lib/content-loader.ts
@@ -619,6 +619,29 @@ export function loadPortfolioSource(overrides: PortfolioSourceOverrides = {}) {
       ),
     ),
   );
+  curation.categories.forEach((category, categoryIndex) =>
+    category.projectIds.forEach((projectId, projectIndex) =>
+      addMissingReferenceIssue(
+        issues,
+        "src/content/curation.json",
+        `$.categories[${categoryIndex}].projectIds[${projectIndex}]`,
+        "project id",
+        projectId,
+        enabledProjectIds,
+      ),
+    ),
+  );
+  contact.preferred.forEach((linkId, index) =>
+    addMissingReferenceIssue(
+      issues,
+      "src/content/contact.json",
+      `$.preferred[${index}]`,
+      "link id",
+      linkId,
+      enabledLinkIds,
+    ),
+  );
+
   if (issues.length > 0) {
     throw new PortfolioContentError(issues);
   }


## `feat(content): 저장소 자산 참조 경계 검증`

diff --git a/public/template/profile/portrait-placeholder.svg b/public/template/profile/portrait-placeholder.svg
new file mode 100644
index 0000000..84ca61f
--- /dev/null
+++ b/public/template/profile/portrait-placeholder.svg
@@ -0,0 +1,10 @@
+<svg xmlns="http://www.w3.org/2000/svg" width="320" height="400" viewBox="0 0 320 400" role="img" aria-labelledby="title desc">
+  <title id="title">Profile portrait placeholder</title>
+  <desc id="desc">A neutral placeholder frame for a profile portrait image.</desc>
+  <rect width="320" height="400" fill="#f7faf8"/>
+  <rect x="18" y="18" width="284" height="364" rx="24" fill="#ffffff" stroke="#c7d8d2" stroke-width="2"/>
+  <circle cx="160" cy="145" r="58" fill="#dbece6"/>
+  <path d="M78 322c9-57 39-89 82-89s73 32 82 89" fill="#dbece6"/>
+  <path d="M96 326h128" stroke="#2a9d8f" stroke-width="8" stroke-linecap="round"/>
+  <text x="160" y="365" fill="#5d6f82" font-family="Arial, sans-serif" font-size="18" font-weight="700" text-anchor="middle">PHOTO</text>
+</svg>
diff --git a/src/lib/content-assets.ts b/src/lib/content-assets.ts
new file mode 100644
index 0000000..8b3ed1c
--- /dev/null
+++ b/src/lib/content-assets.ts
@@ -0,0 +1,89 @@
+import { existsSync } from "node:fs";
+import { isAbsolute, relative, resolve } from "node:path";
+
+import {
+  PortfolioContentError,
+  type ContentValidationIssue,
+  type PortfolioSource,
+} from "./content-loader";
+
+type AssetReference = {
+  assetPath: string;
+  file: string;
+  path: string;
+};
+
+function collectAssetReferences(content: PortfolioSource): AssetReference[] {
+  const references: AssetReference[] = [];
+
+  if (content.site.socialImage) {
+    references.push({
+      assetPath: content.site.socialImage,
+      file: "src/content/site.json",
+      path: "$.socialImage",
+    });
+  }
+
+  if (content.profile.photo) {
+    references.push({
+      assetPath: content.profile.photo.src,
+      file: "src/content/profile.json",
+      path: "$.photo.src",
+    });
+  }
+
+  if (content.resume.downloadUrl) {
+    references.push({
+      assetPath: content.resume.downloadUrl,
+      file: "src/content/resume.json",
+      path: "$.downloadUrl",
+    });
+  }
+
+  content.projects.items.forEach((project, projectIndex) => {
+    references.push({
+      assetPath: project.screenshot.src,
+      file: "src/content/projects.json",
+      path: `$.items[${projectIndex}].screenshot.src`,
+    });
+    project.screenshots.forEach((screenshot, screenshotIndex) =>
+      references.push({
+        assetPath: screenshot.src,
+        file: "src/content/projects.json",
+        path: `$.items[${projectIndex}].screenshots[${screenshotIndex}].src`,
+      }),
+    );
+  });
+
+  return references;
+}
+
+export function validatePortfolioAssets(
+  content: PortfolioSource,
+  publicRoot: string,
+) {
+  const issues: ContentValidationIssue[] = [];
+
+  for (const reference of collectAssetReferences(content)) {
+    const absoluteAssetPath = resolve(publicRoot, `.${reference.assetPath}`);
+    const pathFromPublic = relative(publicRoot, absoluteAssetPath);
+
+    if (
+      pathFromPublic.startsWith("..") ||
+      isAbsolute(pathFromPublic) ||
+      !existsSync(absoluteAssetPath)
+    ) {
+      issues.push({
+        file: reference.file,
+        path: reference.path,
+        message: `Asset "${reference.assetPath}" does not exist under public/.`,
+      });
+    }
+  }
+
+  if (issues.length > 0) {
+    throw new PortfolioContentError(issues);
+  }
+
+  return content;
+}


