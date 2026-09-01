# 라우트별 View Model과 렌더러 데이터 최소화

## `refactor(content): 홈 route view model 경계 추가`

diff --git a/src/lib/portfolio/view-models.ts b/src/lib/portfolio/view-models.ts
new file mode 100644
index 0000000..c3ad116
--- /dev/null
+++ b/src/lib/portfolio/view-models.ts
@@ -0,0 +1,92 @@
+import {
+  getContentLinksByPlacement,
+  getPreferredContactLinks,
+  getProjectMetricValue,
+} from "./selectors";
+import type {
+  ContentLink,
+  CurationCategory,
+  PortfolioContent,
+  PortfolioProject,
+  ProjectGroup,
+  ProjectMetric,
+} from "./types";
+
+type RouteViewModelBase = PortfolioContent & {
+  footerLinks: ContentLink[];
+};
+
+export type ProjectGroupViewModel = ProjectGroup & {
+  projects: PortfolioProject[];
+};
+
+export type ProjectMetricViewModel = ProjectMetric & {
+  value: number;
+};
+
+export type CurationCategoryViewModel = CurationCategory & {
+  projects: PortfolioProject[];
+};
+
+export type HomeViewModel = RouteViewModelBase & {
+  route: "home";
+  currentYear: number;
+  featuredProjects: PortfolioProject[];
+  featuredOrAllProjects: PortfolioProject[];
+  heroLinks: ContentLink[];
+  leadProject: PortfolioProject | null;
+  metricValues: Record<string, number>;
+  metrics: ProjectMetricViewModel[];
+  preferredContactLinks: ContentLink[];
+  projectCount: number;
+  recentJourney: PortfolioContent["journey"];
+};
+
+function createRouteViewModelBase(
+  content: PortfolioContent,
+): RouteViewModelBase {
+  return {
+    ...content,
+    footerLinks: getContentLinksByPlacement("footer", content),
+    links: [],
+    projectGroups: [],
+    projectMetrics: [],
+  };
+}
+
+export function createHomeViewModel(
+  content: PortfolioContent,
+  now: Date = new Date(),
+): HomeViewModel {
+  const featuredProjects = content.projects.filter(
+    (project) => project.featured,
+  );
+  const featuredOrAllProjects =
+    featuredProjects.length > 0 ? featuredProjects : content.projects;
+  const metricValues = Object.fromEntries(
+    content.projectMetrics.map((metric) => [
+      metric.id,
+      getProjectMetricValue(metric.id, content),
+    ]),
+  );
+  const metrics = content.projectMetrics.map((metric) => ({
+    ...metric,
+    value: metricValues[metric.id] ?? 0,
+  }));
+
+  return {
+    ...createRouteViewModelBase(content),
+    currentYear: now.getFullYear(),
+    featuredOrAllProjects,
+    featuredProjects,
+    heroLinks: getContentLinksByPlacement("hero", content),
+    leadProject: featuredOrAllProjects[0] ?? null,
+    metricValues,
+    metrics,
+    preferredContactLinks: getPreferredContactLinks(content),
+    projectCount: content.projects.length,
+    projects: [],
+    recentJourney: content.journey.slice(-4).reverse(),
+    route: "home",
+  };
+}


## `refactor(content): 프로젝트 목록 파생 모델 추가`

diff --git a/src/lib/portfolio/view-models.ts b/src/lib/portfolio/view-models.ts
index c3ad116..721642b 100644
--- a/src/lib/portfolio/view-models.ts
+++ b/src/lib/portfolio/view-models.ts
@@ -42,6 +42,18 @@ export type HomeViewModel = RouteViewModelBase & {
   recentJourney: PortfolioContent["journey"];
 };
 
+export type ProjectIndexViewModel = RouteViewModelBase & {
+  route: "projects";
+  archiveGroupEntries: [string, PortfolioProject[]][];
+  archiveGroups: ProjectGroupViewModel[];
+  archiveProjects: PortfolioProject[];
+  featuredProjects: PortfolioProject[];
+  groupEntries: [string, PortfolioProject[]][];
+  groups: ProjectGroupViewModel[];
+  metricValues: Record<string, number>;
+  metrics: ProjectMetricViewModel[];
+};
+
 function createRouteViewModelBase(
   content: PortfolioContent,
 ): RouteViewModelBase {
@@ -54,6 +66,42 @@ function createRouteViewModelBase(
   };
 }
 
+function resolveProjectGroups(
+  content: PortfolioContent,
+  projects: PortfolioProject[],
+) {
+  const projectsByGroup = new Map<string, PortfolioProject[]>();
+
+  for (const project of projects) {
+    projectsByGroup.set(project.groupId, [
+      ...(projectsByGroup.get(project.groupId) ?? []),
+      project,
+    ]);
+  }
+
+  const configuredGroups = content.projectGroups
+    .map((group) => ({
+      ...group,
+      projects: projectsByGroup.get(group.id) ?? [],
+    }))
+    .filter((group) => group.projects.length > 0);
+  const configuredGroupIds = new Set(
+    configuredGroups.map((group) => group.id),
+  );
+  const unconfiguredGroups = [...projectsByGroup.entries()]
+    .filter(([groupId]) => !configuredGroupIds.has(groupId))
+    .sort(([left], [right]) => left.localeCompare(right))
+    .map(([groupId, groupedProjects], index) => ({
+      description: "",
+      id: groupId,
+      label: groupedProjects[0]?.category ?? groupId,
+      order: content.projectGroups.length + index,
+      projects: groupedProjects,
+    }));
+
+  return [...configuredGroups, ...unconfiguredGroups];
+}
+
 export function createHomeViewModel(
   content: PortfolioContent,
   now: Date = new Date(),
@@ -90,3 +138,39 @@ export function createHomeViewModel(
     route: "home",
   };
 }
+
+export function createProjectIndexViewModel(
+  content: PortfolioContent,
+): ProjectIndexViewModel {
+  const featuredProjects = content.projects.filter(
+    (project) => project.featured,
+  );
+  const archiveProjects = content.projects.filter(
+    (project) => !project.featured,
+  );
+
+  const archiveGroups = resolveProjectGroups(content, archiveProjects);
+  const groups = resolveProjectGroups(content, content.projects);
+  const metrics = content.projectMetrics.map((metric) => ({
+    ...metric,
+    value: getProjectMetricValue(metric.id, content),
+  }));
+
+  return {
+    ...createRouteViewModelBase(content),
+    archiveGroupEntries: archiveGroups.map((group) => [
+      group.label,
+      group.projects,
+    ]),
+    archiveGroups,
+    archiveProjects,
+    featuredProjects,
+    groupEntries: groups.map((group) => [group.label, group.projects]),
+    groups,
+    metricValues: Object.fromEntries(
+      metrics.map((metric) => [metric.id, metric.value]),
+    ),
+    metrics,
+    route: "projects",
+  };
+}


## `refactor(content): 상세와 소개 파생 모델 추가`

diff --git a/src/lib/portfolio/view-models.ts b/src/lib/portfolio/view-models.ts
index 721642b..1d46c3b 100644
--- a/src/lib/portfolio/view-models.ts
+++ b/src/lib/portfolio/view-models.ts
@@ -1,6 +1,7 @@
 import {
   getContentLinksByPlacement,
   getPreferredContactLinks,
+  getProjectDetailLinks,
   getProjectMetricValue,
 } from "./selectors";
 import type {
@@ -9,7 +10,9 @@ import type {
   PortfolioContent,
   PortfolioProject,
   ProjectGroup,
+  ProjectImage,
   ProjectMetric,
+  TechStackItem,
 } from "./types";
 
 type RouteViewModelBase = PortfolioContent & {
@@ -54,6 +57,19 @@ export type ProjectIndexViewModel = RouteViewModelBase & {
   metrics: ProjectMetricViewModel[];
 };
 
+export type ProjectDetailViewModel = RouteViewModelBase & {
+  route: "project-detail";
+  detailLinks: ContentLink[];
+  project: PortfolioProject;
+  stackItems: TechStackItem[];
+  supportingImages: ProjectImage[];
+};
+
+export type AboutViewModel = RouteViewModelBase & {
+  route: "about";
+  curationCategories: CurationCategoryViewModel[];
+};
+
 function createRouteViewModelBase(
   content: PortfolioContent,
 ): RouteViewModelBase {
@@ -174,3 +190,58 @@ export function createProjectIndexViewModel(
     route: "projects",
   };
 }
+
+export function createProjectDetailViewModel(
+  content: PortfolioContent,
+  projectId: string,
+): ProjectDetailViewModel | null {
+  const project = content.projects.find((item) => item.id === projectId);
+
+  if (!project) {
+    return null;
+  }
+
+  const stackById = new Map(
+    content.techStack.map((item) => [item.id, item]),
+  );
+
+  return {
+    ...createRouteViewModelBase(content),
+    detailLinks: getProjectDetailLinks(project),
+    project,
+    projects: [],
+    route: "project-detail",
+    stackItems: project.stack.map(
+      (id) =>
+        stackById.get(id) ?? {
+          color: "#9cc8b1",
+          icon: "tool",
+          id,
+          label: id,
+        },
+    ),
+    supportingImages: project.screenshots.filter(
+      (image) => image.src !== project.screenshot.src,
+    ),
+  };
+}
+
+export function createAboutViewModel(
+  content: PortfolioContent,
+): AboutViewModel {
+  const projectById = new Map(
+    content.projects.map((project) => [project.id, project]),
+  );
+
+  return {
+    ...createRouteViewModelBase(content),
+    curationCategories: content.curation.categories.map((category) => ({
+      ...category,
+      projects: category.projectIds
+        .map((projectId) => projectById.get(projectId))
+        .filter((project): project is PortfolioProject => Boolean(project)),
+    })),
+    projects: [],
+    route: "about",
+  };
+}


## `refactor(content): 이력과 연락 파생 모델 추가`

diff --git a/src/lib/portfolio/view-models.ts b/src/lib/portfolio/view-models.ts
index 1d46c3b..ff7f3d1 100644
--- a/src/lib/portfolio/view-models.ts
+++ b/src/lib/portfolio/view-models.ts
@@ -70,6 +70,27 @@ export type AboutViewModel = RouteViewModelBase & {
   curationCategories: CurationCategoryViewModel[];
 };
 
+export type ResumeViewModel = RouteViewModelBase & {
+  route: "resume";
+  resumeProjects: PortfolioProject[];
+};
+
+export type ContactViewModel = RouteViewModelBase & {
+  route: "contact";
+  cinematicLinks: ContentLink[];
+  contactPlacementLinks: ContentLink[];
+  preferredLinks: ContentLink[];
+  preferredOrContactLinks: ContentLink[];
+};
+
+export type PortfolioRouteViewModel =
+  | HomeViewModel
+  | ProjectIndexViewModel
+  | ProjectDetailViewModel
+  | AboutViewModel
+  | ResumeViewModel
+  | ContactViewModel;
+
 function createRouteViewModelBase(
   content: PortfolioContent,
 ): RouteViewModelBase {
@@ -245,3 +266,39 @@ export function createAboutViewModel(
     route: "about",
   };
 }
+
+export function createResumeViewModel(
+  content: PortfolioContent,
+): ResumeViewModel {
+  const projectById = new Map(
+    content.projects.map((project) => [project.id, project]),
+  );
+
+  return {
+    ...createRouteViewModelBase(content),
+    resumeProjects: content.resume.projectIds
+      .map((projectId) => projectById.get(projectId))
+      .filter((project): project is PortfolioProject => Boolean(project)),
+    projects: [],
+    route: "resume",
+  };
+}
+
+export function createContactViewModel(
+  content: PortfolioContent,
+): ContactViewModel {
+  const contactPlacementLinks = getContentLinksByPlacement("contact", content);
+  const preferredLinks = getPreferredContactLinks(content);
+  const preferredOrContactLinks =
+    preferredLinks.length > 0 ? preferredLinks : contactPlacementLinks;
+
+  return {
+    ...createRouteViewModelBase(content),
+    cinematicLinks: preferredOrContactLinks,
+    contactPlacementLinks,
+    preferredLinks,
+    preferredOrContactLinks,
+    projects: [],
+    route: "contact",
+  };
+}


## `test(content): route view model 파생 규칙 검증`

diff --git a/src/lib/portfolio/view-models.test.ts b/src/lib/portfolio/view-models.test.ts
new file mode 100644
index 0000000..83ffff5
--- /dev/null
+++ b/src/lib/portfolio/view-models.test.ts
@@ -0,0 +1,140 @@
+import { describe, expect, it } from "vitest";
+
+import { getPortfolioContent } from "./content";
+import {
+  createAboutViewModel,
+  createContactViewModel,
+  createHomeViewModel,
+  createProjectDetailViewModel,
+  createProjectIndexViewModel,
+  createResumeViewModel,
+} from "./view-models";
+
+describe("portfolio route view models", () => {
+  it("prepares home selections, metrics, and links before rendering", () => {
+    const content = getPortfolioContent();
+    const viewModel = createHomeViewModel(
+      content,
+      new Date("2026-07-23T00:00:00.000Z"),
+    );
+
+    expect(viewModel.route).toBe("home");
+    expect(viewModel.currentYear).toBe(2026);
+    expect(viewModel).not.toHaveProperty("content");
+    expect(viewModel.featuredProjects).toEqual(
+      content.projects.filter((project) => project.featured),
+    );
+    expect(viewModel.leadProject).toBe(
+      viewModel.featuredProjects[0] ?? content.projects[0] ?? null,
+    );
+    expect(viewModel.heroLinks.every((link) =>
+      link.placements?.includes("hero"),
+    )).toBe(true);
+    expect(viewModel.footerLinks.every((link) =>
+      link.placements?.includes("footer"),
+    )).toBe(true);
+    expect(viewModel.metricValues).toEqual(
+      expect.objectContaining({
+        curriculumCount: expect.any(Number),
+        productCount: expect.any(Number),
+        reliabilityCount: expect.any(Number),
+      }),
+    );
+  });
+
+  it("groups the project index in configured order and resolves metric values", () => {
+    const content = getPortfolioContent();
+    const viewModel = createProjectIndexViewModel(content);
+    const groupOrder = content.projectGroups.map((group) => group.id);
+    const renderedProjectIds = viewModel.groups.flatMap((group) =>
+      group.projects.map((project) => project.id),
+    );
+
+    expect(viewModel.route).toBe("projects");
+    expect(viewModel.groups.map((group) => group.id)).toEqual(
+      [...viewModel.groups.map((group) => group.id)].sort(
+        (left, right) => groupOrder.indexOf(left) - groupOrder.indexOf(right),
+      ),
+    );
+    expect(renderedProjectIds).toEqual(
+      expect.arrayContaining(content.projects.map((project) => project.id)),
+    );
+    expect(new Set(renderedProjectIds).size).toBe(content.projects.length);
+    expect(viewModel.metrics.map((metric) => metric.value)).toEqual(
+      expect.arrayContaining(viewModel.metrics.map(() => expect.any(Number))),
+    );
+  });
+
+  it("resolves project-detail links, stack labels, and supporting images", () => {
+    const content = getPortfolioContent();
+    const project = content.projects[0];
+
+    expect(project).toBeDefined();
+    const viewModel = createProjectDetailViewModel(content, project.id);
+
+    expect(viewModel).not.toBeNull();
+    expect(viewModel?.route).toBe("project-detail");
+    expect(viewModel?.project).toBe(project);
+    expect(viewModel?.detailLinks.every((link) =>
+      link.placements?.includes("detail"),
+    )).toBe(true);
+    expect(viewModel?.stackItems.map((item) => item.id)).toEqual(project.stack);
+    expect(
+      viewModel?.supportingImages.every(
+        (image) => image.src !== project.screenshot.src,
+      ),
+    ).toBe(true);
+    expect(createProjectDetailViewModel(content, "missing-project")).toBeNull();
+  });
+
+  it("resolves about curation references without making renderers search projects", () => {
+    const content = getPortfolioContent();
+    const viewModel = createAboutViewModel(content);
+
+    expect(viewModel.route).toBe("about");
+    expect(viewModel.curationCategories).toHaveLength(
+      content.curation.categories.length,
+    );
+    for (const category of viewModel.curationCategories) {
+      expect(category.projects.map((project) => project.id)).toEqual(
+        category.projectIds.filter((projectId) =>
+          content.projects.some((project) => project.id === projectId),
+        ),
+      );
+    }
+  });
+
+  it("keeps the resume project order and omits unknown references", () => {
+    const content = structuredClone(getPortfolioContent());
+    content.resume.projectIds = [
+      ...content.resume.projectIds.slice().reverse(),
+      "missing-project",
+    ];
+
+    const viewModel = createResumeViewModel(content);
+
+    expect(viewModel.route).toBe("resume");
+    expect(viewModel.resumeProjects.map((project) => project.id)).toEqual(
+      content.resume.projectIds.filter((projectId) =>
+        content.projects.some((project) => project.id === projectId),
+      ),
+    );
+  });
+
+  it("keeps preferred contact order and prepares the cinematic fallback", () => {
+    const content = getPortfolioContent();
+    const viewModel = createContactViewModel(content);
+
+    expect(viewModel.route).toBe("contact");
+    expect(viewModel.preferredLinks.map((link) => link.id)).toEqual(
+      content.contact.preferred.filter((id) =>
+        content.links.some((link) => link.id === id),
+      ),
+    );
+    expect(viewModel.cinematicLinks).toEqual(
+      viewModel.preferredLinks.length > 0
+        ? viewModel.preferredLinks
+        : viewModel.contactPlacementLinks,
+    );
+  });
+});


## `refactor(content): 여정 근거 view model 추가`

diff --git a/src/lib/portfolio/view-models.ts b/src/lib/portfolio/view-models.ts
index ff7f3d1..d8eb376 100644
--- a/src/lib/portfolio/view-models.ts
+++ b/src/lib/portfolio/view-models.ts
@@ -7,6 +7,8 @@ import {
 import type {
   ContentLink,
   CurationCategory,
+  JourneyItem,
+  JourneyMilestone,
   PortfolioContent,
   PortfolioProject,
   ProjectGroup,
@@ -83,13 +85,28 @@ export type ContactViewModel = RouteViewModelBase & {
   preferredOrContactLinks: ContentLink[];
 };
 
+export type JourneyMilestoneViewModel = JourneyMilestone & {
+  anchorProjects: PortfolioProject[];
+};
+
+export type JourneyItemViewModel = JourneyItem & {
+  project: PortfolioProject | null;
+};
+
+export type JourneyViewModel = RouteViewModelBase & {
+  route: "journey";
+  milestones: JourneyMilestoneViewModel[];
+  timelineItems: JourneyItemViewModel[];
+};
+
 export type PortfolioRouteViewModel =
   | HomeViewModel
   | ProjectIndexViewModel
   | ProjectDetailViewModel
   | AboutViewModel
   | ResumeViewModel
-  | ContactViewModel;
+  | ContactViewModel
+  | JourneyViewModel;
 
 function createRouteViewModelBase(
   content: PortfolioContent,
@@ -302,3 +319,27 @@ export function createContactViewModel(
     route: "contact",
   };
 }
+
+export function createJourneyViewModel(
+  content: PortfolioContent,
+): JourneyViewModel {
+  const projectById = new Map(
+    content.projects.map((project) => [project.id, project]),
+  );
+
+  return {
+    ...createRouteViewModelBase(content),
+    milestones: content.journeyNarrative.milestones.map((milestone) => ({
+      ...milestone,
+      anchorProjects: milestone.anchorProjectIds
+        .map((projectId) => projectById.get(projectId))
+        .filter((project): project is PortfolioProject => Boolean(project)),
+    })),
+    projects: [],
+    route: "journey",
+    timelineItems: content.journey.map((item) => ({
+      ...item,
+      project: item.projectId ? (projectById.get(item.projectId) ?? null) : null,
+    })),
+  };
+}


## `refactor(content): 인터뷰 근거 view model 추가`

diff --git a/src/lib/portfolio/view-models.ts b/src/lib/portfolio/view-models.ts
index d8eb376..52c314c 100644
--- a/src/lib/portfolio/view-models.ts
+++ b/src/lib/portfolio/view-models.ts
@@ -7,6 +7,9 @@ import {
 import type {
   ContentLink,
   CurationCategory,
+  InterviewMapAnswer,
+  InterviewMapItem,
+  InterviewMapTrack,
   JourneyItem,
   JourneyMilestone,
   PortfolioContent,
@@ -99,6 +102,23 @@ export type JourneyViewModel = RouteViewModelBase & {
   timelineItems: JourneyItemViewModel[];
 };
 
+export type InterviewMapAnswerViewModel = InterviewMapAnswer & {
+  project: PortfolioProject | null;
+};
+
+export type InterviewMapItemViewModel = Omit<InterviewMapItem, "answers"> & {
+  answers: InterviewMapAnswerViewModel[];
+};
+
+export type InterviewMapTrackViewModel = Omit<InterviewMapTrack, "items"> & {
+  items: InterviewMapItemViewModel[];
+};
+
+export type InterviewMapViewModel = RouteViewModelBase & {
+  route: "interview-map";
+  tracks: InterviewMapTrackViewModel[];
+};
+
 export type PortfolioRouteViewModel =
   | HomeViewModel
   | ProjectIndexViewModel
@@ -106,7 +126,8 @@ export type PortfolioRouteViewModel =
   | AboutViewModel
   | ResumeViewModel
   | ContactViewModel
-  | JourneyViewModel;
+  | JourneyViewModel
+  | InterviewMapViewModel;
 
 function createRouteViewModelBase(
   content: PortfolioContent,
@@ -343,3 +364,27 @@ export function createJourneyViewModel(
     })),
   };
 }
+
+export function createInterviewMapViewModel(
+  content: PortfolioContent,
+): InterviewMapViewModel {
+  const projectById = new Map(
+    content.projects.map((project) => [project.id, project]),
+  );
+
+  return {
+    ...createRouteViewModelBase(content),
+    projects: [],
+    route: "interview-map",
+    tracks: content.interviewMap.tracks.map((track) => ({
+      ...track,
+      items: track.items.map((item) => ({
+        ...item,
+        answers: item.answers.map((answer) => ({
+          ...answer,
+          project: projectById.get(answer.projectId) ?? null,
+        })),
+      })),
+    })),
+  };
+}


