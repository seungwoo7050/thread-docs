## `refactor(content): 프로젝트 view model 공개 필드 제한`

diff --git a/src/lib/portfolio/view-models.ts b/src/lib/portfolio/view-models.ts
index 8563047..137ae6a 100644
--- a/src/lib/portfolio/view-models.ts
+++ b/src/lib/portfolio/view-models.ts
@@ -74,25 +74,31 @@ export type HomeViewModel = ScopedRouteViewModel<
   }
 >;
 
-export type ProjectIndexViewModel = RouteViewModelBase & {
-  route: "projects";
-  archiveGroupEntries: [string, PortfolioProject[]][];
-  archiveGroups: ProjectGroupViewModel[];
-  archiveProjects: PortfolioProject[];
-  featuredProjects: PortfolioProject[];
-  groupEntries: [string, PortfolioProject[]][];
-  groups: ProjectGroupViewModel[];
-  metricValues: Record<string, number>;
-  metrics: ProjectMetricViewModel[];
-};
+export type ProjectIndexViewModel = ScopedRouteViewModel<
+  "contact" | "projects",
+  {
+    route: "projects";
+    archiveGroupEntries: [string, PortfolioProject[]][];
+    archiveGroups: ProjectGroupViewModel[];
+    archiveProjects: PortfolioProject[];
+    featuredProjects: PortfolioProject[];
+    groupEntries: [string, PortfolioProject[]][];
+    groups: ProjectGroupViewModel[];
+    metricValues: Record<string, number>;
+    metrics: ProjectMetricViewModel[];
+  }
+>;
 
-export type ProjectDetailViewModel = RouteViewModelBase & {
-  route: "project-detail";
-  detailLinks: ContentLink[];
-  project: PortfolioProject;
-  stackItems: TechStackItem[];
-  supportingImages: ProjectImage[];
-};
+export type ProjectDetailViewModel = ScopedRouteViewModel<
+  never,
+  {
+    route: "project-detail";
+    detailLinks: ContentLink[];
+    project: PortfolioProject;
+    stackItems: TechStackItem[];
+    supportingImages: ProjectImage[];
+  }
+>;
 
 export type AboutViewModel = RouteViewModelBase & {
   route: "about";
@@ -267,6 +273,7 @@ export function createProjectIndexViewModel(
     ]),
     archiveGroups,
     archiveProjects,
+    contact: content.contact,
     featuredProjects,
     groupEntries: groups.map((group) => [group.label, group.projects]),
     groups,
@@ -274,8 +281,9 @@ export function createProjectIndexViewModel(
       metrics.map((metric) => [metric.id, metric.value]),
     ),
     metrics,
+    projects: content.projects,
     route: "projects",
-  };
+  } as ProjectIndexViewModel;
 }
 
 export function createProjectDetailViewModel(
@@ -296,7 +304,6 @@ export function createProjectDetailViewModel(
     ...createRouteViewModelBase(content),
     detailLinks: getProjectDetailLinks(project),
     project,
-    projects: [],
     route: "project-detail",
     stackItems: project.stack.map(
       (id) =>
@@ -310,7 +317,7 @@ export function createProjectDetailViewModel(
     supportingImages: project.screenshots.filter(
       (image) => image.src !== project.screenshot.src,
     ),
-  };
+  } as ProjectDetailViewModel;
 }
 
 export function createAboutViewModel(


## `refactor(content): 소개·이력·연락 공개 필드 제한`

diff --git a/src/lib/portfolio/view-models.ts b/src/lib/portfolio/view-models.ts
index 137ae6a..7f9074a 100644
--- a/src/lib/portfolio/view-models.ts
+++ b/src/lib/portfolio/view-models.ts
@@ -100,23 +100,32 @@ export type ProjectDetailViewModel = ScopedRouteViewModel<
   }
 >;
 
-export type AboutViewModel = RouteViewModelBase & {
-  route: "about";
-  curationCategories: CurationCategoryViewModel[];
-};
+export type AboutViewModel = ScopedRouteViewModel<
+  "contact" | "curation" | "experience" | "journey" | "skills",
+  {
+    route: "about";
+    curationCategories: CurationCategoryViewModel[];
+  }
+>;
 
-export type ResumeViewModel = RouteViewModelBase & {
-  route: "resume";
-  resumeProjects: PortfolioProject[];
-};
+export type ResumeViewModel = ScopedRouteViewModel<
+  "experience" | "resume",
+  {
+    route: "resume";
+    resumeProjects: PortfolioProject[];
+  }
+>;
 
-export type ContactViewModel = RouteViewModelBase & {
-  route: "contact";
-  cinematicLinks: ContentLink[];
-  contactPlacementLinks: ContentLink[];
-  preferredLinks: ContentLink[];
-  preferredOrContactLinks: ContentLink[];
-};
+export type ContactViewModel = ScopedRouteViewModel<
+  "contact",
+  {
+    route: "contact";
+    cinematicLinks: ContentLink[];
+    contactPlacementLinks: ContentLink[];
+    preferredLinks: ContentLink[];
+    preferredOrContactLinks: ContentLink[];
+  }
+>;
 
 export type JourneyMilestoneViewModel = JourneyMilestone & {
   anchorProjects: PortfolioProject[];
@@ -329,15 +338,19 @@ export function createAboutViewModel(
 
   return {
     ...createRouteViewModelBase(content),
+    contact: content.contact,
+    curation: content.curation,
     curationCategories: content.curation.categories.map((category) => ({
       ...category,
       projects: category.projectIds
         .map((projectId) => projectById.get(projectId))
         .filter((project): project is PortfolioProject => Boolean(project)),
     })),
-    projects: [],
+    experience: content.experience,
+    journey: content.journey,
     route: "about",
-  };
+    skills: content.skills,
+  } as AboutViewModel;
 }
 
 export function createResumeViewModel(
@@ -349,12 +362,13 @@ export function createResumeViewModel(
 
   return {
     ...createRouteViewModelBase(content),
+    experience: content.experience,
+    resume: content.resume,
     resumeProjects: content.resume.projectIds
       .map((projectId) => projectById.get(projectId))
       .filter((project): project is PortfolioProject => Boolean(project)),
-    projects: [],
     route: "resume",
-  };
+  } as ResumeViewModel;
 }
 
 export function createContactViewModel(
@@ -368,12 +382,12 @@ export function createContactViewModel(
   return {
     ...createRouteViewModelBase(content),
     cinematicLinks: preferredOrContactLinks,
+    contact: content.contact,
     contactPlacementLinks,
     preferredLinks,
     preferredOrContactLinks,
-    projects: [],
     route: "contact",
-  };
+  } as ContactViewModel;
 }
 
 export function createJourneyViewModel(


## `refactor(content): 여정·인터뷰 공개 필드 제한`

diff --git a/src/lib/portfolio/view-models.ts b/src/lib/portfolio/view-models.ts
index 7f9074a..bcc709a 100644
--- a/src/lib/portfolio/view-models.ts
+++ b/src/lib/portfolio/view-models.ts
@@ -135,11 +135,14 @@ export type JourneyItemViewModel = JourneyItem & {
   project: PortfolioProject | null;
 };
 
-export type JourneyViewModel = RouteViewModelBase & {
-  route: "journey";
-  milestones: JourneyMilestoneViewModel[];
-  timelineItems: JourneyItemViewModel[];
-};
+export type JourneyViewModel = ScopedRouteViewModel<
+  "journey" | "journeyNarrative",
+  {
+    route: "journey";
+    milestones: JourneyMilestoneViewModel[];
+    timelineItems: JourneyItemViewModel[];
+  }
+>;
 
 export type InterviewMapAnswerViewModel = InterviewMapAnswer & {
   project: PortfolioProject | null;
@@ -153,10 +156,13 @@ export type InterviewMapTrackViewModel = Omit<InterviewMapTrack, "items"> & {
   items: InterviewMapItemViewModel[];
 };
 
-export type InterviewMapViewModel = RouteViewModelBase & {
-  route: "interview-map";
-  tracks: InterviewMapTrackViewModel[];
-};
+export type InterviewMapViewModel = ScopedRouteViewModel<
+  "interviewMap",
+  {
+    route: "interview-map";
+    tracks: InterviewMapTrackViewModel[];
+  }
+>;
 
 export type PortfolioRouteViewModel =
   | HomeViewModel
@@ -399,19 +405,20 @@ export function createJourneyViewModel(
 
   return {
     ...createRouteViewModelBase(content),
+    journey: content.journey,
+    journeyNarrative: content.journeyNarrative,
     milestones: content.journeyNarrative.milestones.map((milestone) => ({
       ...milestone,
       anchorProjects: milestone.anchorProjectIds
         .map((projectId) => projectById.get(projectId))
         .filter((project): project is PortfolioProject => Boolean(project)),
     })),
-    projects: [],
     route: "journey",
     timelineItems: content.journey.map((item) => ({
       ...item,
       project: item.projectId ? (projectById.get(item.projectId) ?? null) : null,
     })),
-  };
+  } as JourneyViewModel;
 }
 
 export function createInterviewMapViewModel(
@@ -423,7 +430,7 @@ export function createInterviewMapViewModel(
 
   return {
     ...createRouteViewModelBase(content),
-    projects: [],
+    interviewMap: content.interviewMap,
     route: "interview-map",
     tracks: content.interviewMap.tracks.map((track) => ({
       ...track,
@@ -435,5 +442,5 @@ export function createInterviewMapViewModel(
         })),
       })),
     })),
-  };
+  } as InterviewMapViewModel;
 }


## `refactor(content): route view model 공용 경계 제한`

diff --git a/src/lib/portfolio/view-models.ts b/src/lib/portfolio/view-models.ts
index bcc709a..a44f301 100644
--- a/src/lib/portfolio/view-models.ts
+++ b/src/lib/portfolio/view-models.ts
@@ -20,28 +20,27 @@ import type {
   TechStackItem,
 } from "./types";
 
-type RouteViewModelBase = PortfolioContent & {
-  footerLinks: ContentLink[];
-};
-
-type ScopedRouteViewModelBase = {
+type RouteViewModelBase = {
   footerLinks: ContentLink[];
   presentation: PortfolioContent["presentation"];
   profile: PortfolioContent["profile"];
   site: PortfolioContent["site"];
 };
 
-type ScopedSharedContentKey = "presentation" | "profile" | "site";
+type SharedContentKey = "presentation" | "profile" | "site";
 
-type ScopedRouteViewModel<
+// Some legacy renderers still type their local helpers as PortfolioContent.
+// Marking unavailable source fields as never keeps those calls compatible
+// without copying the fields into the runtime view model.
+type RouteViewModel<
   VisibleContentKey extends keyof PortfolioContent,
   RouteFields extends object,
-> = ScopedRouteViewModelBase &
+> = RouteViewModelBase &
   Pick<PortfolioContent, VisibleContentKey> &
   RouteFields & {
     readonly [Key in Exclude<
       keyof PortfolioContent,
-      ScopedSharedContentKey | VisibleContentKey
+      SharedContentKey | VisibleContentKey
     >]: never;
   };
 
@@ -57,7 +56,7 @@ export type CurationCategoryViewModel = CurationCategory & {
   projects: PortfolioProject[];
 };
 
-export type HomeViewModel = ScopedRouteViewModel<
+export type HomeViewModel = RouteViewModel<
   "contact" | "journey" | "journeyNarrative" | "skills" | "techStack",
   {
     route: "home";
@@ -74,7 +73,7 @@ export type HomeViewModel = ScopedRouteViewModel<
   }
 >;
 
-export type ProjectIndexViewModel = ScopedRouteViewModel<
+export type ProjectIndexViewModel = RouteViewModel<
   "contact" | "projects",
   {
     route: "projects";
@@ -89,7 +88,7 @@ export type ProjectIndexViewModel = ScopedRouteViewModel<
   }
 >;
 
-export type ProjectDetailViewModel = ScopedRouteViewModel<
+export type ProjectDetailViewModel = RouteViewModel<
   never,
   {
     route: "project-detail";
@@ -100,7 +99,7 @@ export type ProjectDetailViewModel = ScopedRouteViewModel<
   }
 >;
 
-export type AboutViewModel = ScopedRouteViewModel<
+export type AboutViewModel = RouteViewModel<
   "contact" | "curation" | "experience" | "journey" | "skills",
   {
     route: "about";
@@ -108,7 +107,7 @@ export type AboutViewModel = ScopedRouteViewModel<
   }
 >;
 
-export type ResumeViewModel = ScopedRouteViewModel<
+export type ResumeViewModel = RouteViewModel<
   "experience" | "resume",
   {
     route: "resume";
@@ -116,7 +115,7 @@ export type ResumeViewModel = ScopedRouteViewModel<
   }
 >;
 
-export type ContactViewModel = ScopedRouteViewModel<
+export type ContactViewModel = RouteViewModel<
   "contact",
   {
     route: "contact";
@@ -135,7 +134,7 @@ export type JourneyItemViewModel = JourneyItem & {
   project: PortfolioProject | null;
 };
 
-export type JourneyViewModel = ScopedRouteViewModel<
+export type JourneyViewModel = RouteViewModel<
   "journey" | "journeyNarrative",
   {
     route: "journey";
@@ -156,7 +155,7 @@ export type InterviewMapTrackViewModel = Omit<InterviewMapTrack, "items"> & {
   items: InterviewMapItemViewModel[];
 };
 
-export type InterviewMapViewModel = ScopedRouteViewModel<
+export type InterviewMapViewModel = RouteViewModel<
   "interviewMap",
   {
     route: "interview-map";
@@ -178,11 +177,10 @@ function createRouteViewModelBase(
   content: PortfolioContent,
 ): RouteViewModelBase {
   return {
-    ...content,
     footerLinks: getContentLinksByPlacement("footer", content),
-    links: [],
-    projectGroups: [],
-    projectMetrics: [],
+    presentation: content.presentation,
+    profile: content.profile,
+    site: content.site,
   };
 }
 


