## `feat(content): 링크와 배포 상태 schema 추가`

diff --git a/src/lib/content-schema.ts b/src/lib/content-schema.ts
index 430f5b9..278458d 100644
--- a/src/lib/content-schema.ts
+++ b/src/lib/content-schema.ts
@@ -87,3 +87,43 @@ export const profileContentSchema = z
     ),
   })
   .strict();
+
+export const linkTypeSchema = z.enum([
+  "case-study",
+  "demo",
+  "email",
+  "github",
+  "resume",
+  "source",
+  "website",
+]);
+
+export const contentLinkSchema = z
+  .object({
+    id: contentId.optional(),
+    type: linkTypeSchema,
+    label: nonEmptyString,
+    href: contentHrefSchema,
+    external: z.boolean().optional(),
+    enabled: z.boolean().optional(),
+    placements: z
+      .array(z.enum(["hero", "contact", "card", "detail", "footer"]))
+      .optional(),
+  })
+  .strict();
+
+export const deploymentStatusSchema = z.enum([
+  "archived",
+  "case-study-only",
+  "live",
+  "offline",
+  "private",
+  "source-only",
+]);
+
+const projectImageSchema = z
+  .object({
+    src: contentAssetPathSchema,
+    alt: nonEmptyString,
+  })
+  .strict();


## `feat(content): 프로젝트 분류와 지표 schema 추가`

diff --git a/src/lib/content-schema.ts b/src/lib/content-schema.ts
index 278458d..bcb446b 100644
--- a/src/lib/content-schema.ts
+++ b/src/lib/content-schema.ts
@@ -127,3 +127,32 @@ const projectImageSchema = z
     alt: nonEmptyString,
   })
   .strict();
+
+export const projectGroupSchema = z
+  .object({
+    id: contentId,
+    label: nonEmptyString,
+    description: nonEmptyString,
+    order: z.number().int().nonnegative(),
+  })
+  .strict();
+
+export const projectMetricFilterSchema = z
+  .object({
+    projectIds: z.array(contentId).min(1).optional(),
+    groupIds: z.array(contentId).min(1).optional(),
+    tags: z.array(contentId).min(1).optional(),
+    featured: z.boolean().optional(),
+    deploymentStatuses: z.array(deploymentStatusSchema).min(1).optional(),
+  })
+  .strict();
+
+export const projectMetricSchema = z
+  .object({
+    id: contentId,
+    label: nonEmptyString,
+    description: nonEmptyString.optional(),
+    aggregate: z.enum(["projects", "highlights"]),
+    filter: projectMetricFilterSchema.optional(),
+  })
+  .strict();


## `feat(content): 프로젝트 사례 schema 추가`

diff --git a/src/lib/content-schema.ts b/src/lib/content-schema.ts
index bcb446b..9c6563e 100644
--- a/src/lib/content-schema.ts
+++ b/src/lib/content-schema.ts
@@ -156,3 +156,50 @@ export const projectMetricSchema = z
     filter: projectMetricFilterSchema.optional(),
   })
   .strict();
+
+export const portfolioProjectSourceSchema = z
+  .object({
+    id: contentId,
+    order: nonEmptyString,
+    title: nonEmptyString,
+    groupId: contentId,
+    tags: z.array(contentId),
+    featured: z.boolean().optional(),
+    enabled: z.boolean().optional(),
+    period: nonEmptyString,
+    role: nonEmptyString,
+    summary: nonEmptyString,
+    description: nonEmptyString,
+    deployment: z
+      .object({
+        status: deploymentStatusSchema,
+        label: nonEmptyString,
+        showBadge: z.boolean().optional(),
+      })
+      .strict(),
+    screenshot: projectImageSchema,
+    screenshots: z.array(projectImageSchema),
+    stack: z.array(contentId),
+    links: z.array(contentLinkSchema),
+    highlights: z.array(nonEmptyString),
+    problem: nonEmptyString,
+    solution: nonEmptyString,
+    architecture: z
+      .object({
+        summary: nonEmptyString,
+        items: z.array(nonEmptyString),
+      })
+      .strict(),
+    decisions: z.array(nonEmptyString),
+    tradeoffs: z.array(nonEmptyString),
+    results: z.array(nonEmptyString),
+  })
+  .strict();
+
+export const projectsContentSchema = z
+  .object({
+    groups: z.array(projectGroupSchema).min(1),
+    metrics: z.array(projectMetricSchema),
+    items: z.array(portfolioProjectSourceSchema).min(1),
+  })
+  .strict();


## `feat(content): 표현 공용 UI schema 추가`

diff --git a/src/lib/content-schema.ts b/src/lib/content-schema.ts
index db7fc9f..d301a75 100644
--- a/src/lib/content-schema.ts
+++ b/src/lib/content-schema.ts
@@ -376,3 +376,49 @@ const projectPageContentSchema = z
       .strict(),
   })
   .passthrough();
+
+export const presentationContentSchema = z
+  .object({
+    defaultHomeTemplate: siteDesignIdSchema,
+    templates: z.array(
+      z
+        .object({
+          id: siteDesignIdSchema,
+          label: nonEmptyString,
+          description: nonEmptyString,
+        })
+        .passthrough(),
+    ),
+    ui: z
+      .object({
+        debugPrefix: nonEmptyString,
+        skipLinkLabel: nonEmptyString,
+        primaryNavigationAriaLabel: nonEmptyString,
+        mobileNavigationAriaLabel: nonEmptyString,
+        menuLabel: nonEmptyString,
+        designSwitcherAriaTemplate: nonEmptyString,
+        designSwitcherCountTemplate: nonEmptyString,
+        designSwitcherCloseLabel: nonEmptyString,
+        designNavigationAriaLabel: nonEmptyString,
+        journeyCaseStudyLabel: nonEmptyString,
+        techMarqueeAriaLabel: nonEmptyString,
+        animatedTerminalAriaLabel: nonEmptyString,
+        projectNavigationAriaLabel: nonEmptyString,
+        readCaseStudyAriaTemplate: nonEmptyString,
+        openItemAriaTemplate: nonEmptyString,
+        nowLabel: nonEmptyString,
+        emptyStates: z
+          .object({
+            projectsHome: nonEmptyString,
+            projectsArchive: nonEmptyString,
+            journey: nonEmptyString,
+            projectDetails: nonEmptyString,
+            noMappedEvidence: nonEmptyString,
+            additionalNotes: nonEmptyString,
+            contactLinks: nonEmptyString,
+          })
+          .strict(),
+      })
+      .strict(),
+  })
+  .passthrough();


## `feat(content): Design과 Classic 홈 표현 schema 추가`

diff --git a/src/lib/content-schema.ts b/src/lib/content-schema.ts
index d301a75..27cd6af 100644
--- a/src/lib/content-schema.ts
+++ b/src/lib/content-schema.ts
@@ -420,5 +420,53 @@ export const presentationContentSchema = z
           .strict(),
       })
       .strict(),
+    home: z
+      .object({
+        design: z
+          .object({
+            hero: z
+              .object({
+                primaryActionLabel: nonEmptyString,
+                leadLabel: nonEmptyString,
+                leadActionLabel: nonEmptyString,
+                stats: z.array(
+                  z
+                    .object({
+                      label: nonEmptyString,
+                      countKey: workMapCountKeySchema,
+                    })
+                    .strict(),
+                ),
+              })
+              .strict(),
+            sections: z.array(homeSectionIdSchema),
+            featured: sectionCopySchema,
+          })
+          .strict(),
+        classic: z
+          .object({
+            hero: z.object({ primaryActionLabel: nonEmptyString }).strict(),
+            sections: z.array(homeSectionIdSchema),
+            featured: sectionCopySchema,
+            terminal: z
+              .object({
+                title: nonEmptyString,
+                bootLine: nonEmptyString,
+                promptUser: nonEmptyString,
+                promptPath: nonEmptyString,
+                commands: z.array(
+                  z
+                    .object({
+                      command: nonEmptyString,
+                      output: z.array(nonEmptyString),
+                    })
+                    .strict(),
+                ),
+              })
+              .strict(),
+          })
+          .strict(),
+      })
+      .passthrough(),
   })
   .passthrough();


## `feat(content): Editorial 홈 표현 schema 추가`

diff --git a/src/lib/content-schema.ts b/src/lib/content-schema.ts
index 27cd6af..7177ff0 100644
--- a/src/lib/content-schema.ts
+++ b/src/lib/content-schema.ts
@@ -420,6 +420,13 @@ export const presentationContentSchema = z
           .strict(),
       })
       .strict(),
+    editorial: z
+      .object({
+        shell: z
+          .object({ kicker: nonEmptyString, volumeLabel: nonEmptyString })
+          .strict(),
+      })
+      .strict(),
     home: z
       .object({
         design: z
@@ -466,6 +473,22 @@ export const presentationContentSchema = z
               .strict(),
           })
           .strict(),
+        editorial: z
+          .object({
+            sections: editorialHomeSectionsSchema,
+            hero: z
+              .object({
+                issueTemplate: nonEmptyString,
+                primaryActionLabel: nonEmptyString,
+              })
+              .strict(),
+            lead: z
+              .object({ label: nonEmptyString, actionLabel: nonEmptyString })
+              .strict(),
+            featured: z.object({ title: nonEmptyString }).strict(),
+            current: z.object({ actionLabel: nonEmptyString }).strict(),
+          })
+          .strict(),
       })
       .passthrough(),
   })


## `feat(content): Brutalist 홈 표현 schema 추가`

diff --git a/src/lib/content-schema.ts b/src/lib/content-schema.ts
index 7177ff0..7ef4536 100644
--- a/src/lib/content-schema.ts
+++ b/src/lib/content-schema.ts
@@ -427,6 +427,13 @@ export const presentationContentSchema = z
           .strict(),
       })
       .strict(),
+    brutalist: z
+      .object({
+        shell: z
+          .object({ debugLabel: nonEmptyString, debugHint: nonEmptyString })
+          .strict(),
+      })
+      .strict(),
     home: z
       .object({
         design: z
@@ -489,6 +496,31 @@ export const presentationContentSchema = z
             current: z.object({ actionLabel: nonEmptyString }).strict(),
           })
           .strict(),
+        brutalist: z
+          .object({
+            sections: brutalistHomeSectionsSchema,
+            stampLabel: nonEmptyString,
+            signalText: nonEmptyString,
+            hero: z
+              .object({
+                primaryActionLabel: nonEmptyString,
+                secondaryActionLabel: nonEmptyString,
+              })
+              .strict(),
+            featured: z
+              .object({
+                title: nonEmptyString,
+                body: nonEmptyString,
+                actionLabel: nonEmptyString,
+              })
+              .strict(),
+            system: z
+              .object({ title: nonEmptyString, body: nonEmptyString })
+              .strict(),
+            journeyActionLabel: nonEmptyString,
+            contactActionLabel: nonEmptyString,
+          })
+          .strict(),
       })
       .passthrough(),
   })


## `feat(content): Cinematic 홈 표현 schema 추가`

diff --git a/src/lib/content-schema.ts b/src/lib/content-schema.ts
index 7ef4536..c5c750a 100644
--- a/src/lib/content-schema.ts
+++ b/src/lib/content-schema.ts
@@ -434,6 +434,11 @@ export const presentationContentSchema = z
           .strict(),
       })
       .strict(),
+    cinematic: z
+      .object({
+        shell: z.object({ brandSubtitle: nonEmptyString }).strict(),
+      })
+      .strict(),
     home: z
       .object({
         design: z
@@ -521,6 +526,21 @@ export const presentationContentSchema = z
             contactActionLabel: nonEmptyString,
           })
           .strict(),
+        cinematic: z
+          .object({
+            sections: cinematicHomeSectionsSchema,
+            hero: z
+              .object({
+                primaryActionLabel: nonEmptyString,
+                secondaryActionLabel: nonEmptyString,
+              })
+              .strict(),
+            statementLabel: nonEmptyString,
+            focusLabel: nonEmptyString,
+            contactActionLabel: nonEmptyString,
+            caseStudyActionLabel: nonEmptyString,
+          })
+          .strict(),
       })
       .passthrough(),
   })


## `feat(content): 프로젝트 상세 표현 schema 추가`

diff --git a/src/lib/content-schema.ts b/src/lib/content-schema.ts
index 747924b..fb7a2f0 100644
--- a/src/lib/content-schema.ts
+++ b/src/lib/content-schema.ts
@@ -656,6 +656,43 @@ export const presentationContentSchema = z
               .strict(),
           })
           .passthrough(),
+        projectDetail: z
+          .object({
+            backLabel: nonEmptyString,
+            caseLabel: nonEmptyString,
+            missing: z
+              .object({
+                eyebrow: nonEmptyString,
+                title: nonEmptyString,
+                body: nonEmptyString,
+                actionLabel: nonEmptyString,
+              })
+              .strict(),
+            facts: z
+              .object({ roleLabel: nonEmptyString, statusLabel: nonEmptyString })
+              .strict(),
+            outroLabel: nonEmptyString,
+            returnToIndexLabel: nonEmptyString,
+            frameLabel: nonEmptyString,
+            editorial: z
+              .object({ decisionSpreadTitle: nonEmptyString })
+              .strict(),
+            sections: z
+              .object({
+                architecture: projectDetailSectionSchema,
+                decisions: projectDetailSectionSchema,
+                highlights: projectDetailSectionSchema,
+                problem: projectDetailSectionSchema,
+                result: projectDetailSectionSchema,
+                screenshots: projectDetailSectionSchema,
+                solution: projectDetailSectionSchema,
+                stack: projectDetailSectionSchema,
+                tradeoffs: projectDetailSectionSchema,
+              })
+              .strict(),
+          })
+          .passthrough(),
+        projects: projectPageContentSchema,
       })
       .passthrough(),
   })


## `feat(content): Resume 표현 schema 추가`

diff --git a/src/lib/content-schema.ts b/src/lib/content-schema.ts
index fb7a2f0..4510927 100644
--- a/src/lib/content-schema.ts
+++ b/src/lib/content-schema.ts
@@ -693,6 +693,38 @@ export const presentationContentSchema = z
           })
           .passthrough(),
         projects: projectPageContentSchema,
+        resume: z
+          .object({
+            hero: z
+              .object({
+                title: nonEmptyString,
+                body: nonEmptyString,
+                downloadLabel: nonEmptyString,
+              })
+              .strict(),
+            summary: presentationPageTitleSchema,
+            projects: z
+              .object({
+                title: nonEmptyString,
+                caseStudyLabel: nonEmptyString,
+              })
+              .strict(),
+            training: presentationPageTitleSchema,
+            experience: presentationPageTitleSchema,
+            education: presentationPageTitleSchema,
+            notes: presentationPageTitleSchema,
+            identity: z
+              .object({
+                locationLabel: nonEmptyString,
+                availabilityLabel: nonEmptyString,
+              })
+              .strict(),
+            editorial: z.object({ heroEyebrow: nonEmptyString }).strict(),
+            brutalist: z
+              .object({ heroEyebrowTemplate: nonEmptyString })
+              .strict(),
+          })
+          .passthrough(),
       })
       .passthrough(),
   })


## `feat(content): 기술과 경력 schema 추가`

diff --git a/src/lib/content-schema.ts b/src/lib/content-schema.ts
index 4510927..9570460 100644
--- a/src/lib/content-schema.ts
+++ b/src/lib/content-schema.ts
@@ -729,3 +729,62 @@ export const presentationContentSchema = z
       .passthrough(),
   })
   .passthrough();
+
+export const techStackIconSchema = z.enum([
+  "api",
+  "box",
+  "c",
+  "check",
+  "cmake",
+  "cplusplus",
+  "database",
+  "docker",
+  "eslint",
+  "flow",
+  "json",
+  "nextjs",
+  "nodejs",
+  "playwright",
+  "postgresql",
+  "prisma",
+  "react",
+  "redis",
+  "shield",
+  "tailwind",
+  "terminal",
+  "tool",
+  "typescript",
+  "vitest",
+]);
+
+export const techStackContentSchema = z.array(
+  z
+    .object({
+      id: contentId,
+      label: nonEmptyString,
+      icon: techStackIconSchema,
+      color,
+    })
+    .strict(),
+);
+
+export const skillsContentSchema = z
+  .object({
+    focusAreas: z.array(
+      z.object({ title: nonEmptyString, body: nonEmptyString }).strict(),
+    ),
+    groups: z.array(
+      z.object({ title: nonEmptyString, items: z.array(nonEmptyString) }).strict(),
+    ),
+  })
+  .strict();
+
+export const experienceContentSchema = z.array(
+  z
+    .object({
+      period: nonEmptyString,
+      title: nonEmptyString,
+      body: nonEmptyString,
+    })
+    .strict(),
+);


## `feat(content): 여정과 연락 schema 추가`

diff --git a/src/lib/content-schema.ts b/src/lib/content-schema.ts
index 9570460..bccd140 100644
--- a/src/lib/content-schema.ts
+++ b/src/lib/content-schema.ts
@@ -788,3 +788,29 @@ export const experienceContentSchema = z.array(
     })
     .strict(),
 );
+
+export const journeyContentSchema = z.array(
+  z
+    .object({
+      date: nonEmptyString,
+      endDate: nonEmptyString.nullable(),
+      title: nonEmptyString,
+      category: nonEmptyString,
+      body: nonEmptyString,
+      projectId: contentId.nullable(),
+      sourcePath: nonEmptyString.nullable(),
+    })
+    .strict(),
+);
+
+export const linksContentSchema = z.array(contentLinkSchema);
+
+export const contactContentSchema = z
+  .object({
+    title: nonEmptyString,
+    intro: nonEmptyString,
+    availability: nonEmptyString,
+    preferred: z.array(contentId),
+    notes: z.array(nonEmptyString),
+  })
+  .strict();


## `feat(content): 여정 narrative schema 추가`

diff --git a/src/lib/content-schema.ts b/src/lib/content-schema.ts
index 8c3be1c..0db0781 100644
--- a/src/lib/content-schema.ts
+++ b/src/lib/content-schema.ts
@@ -841,3 +841,25 @@ export const resumeContentSchema = z
     notes: z.array(nonEmptyString),
   })
   .strict();
+
+export const journeyNarrativeContentSchema = z
+  .object({
+    intro: nonEmptyString,
+    milestones: z.array(
+      z
+        .object({
+          id: contentId,
+          date: nonEmptyString,
+          title: nonEmptyString,
+          state: nonEmptyString,
+          reason: nonEmptyString,
+          result: nonEmptyString,
+          anchorProjectIds: z.array(contentId),
+        })
+        .strict(),
+    ),
+    currentPosition: z
+      .object({ title: nonEmptyString, body: nonEmptyString })
+      .strict(),
+  })
+  .strict();


## `feat(content): Interview Map 콘텐츠 schema 추가`

diff --git a/src/lib/content-schema.ts b/src/lib/content-schema.ts
index 0db0781..d65e02d 100644
--- a/src/lib/content-schema.ts
+++ b/src/lib/content-schema.ts
@@ -863,3 +863,44 @@ export const journeyNarrativeContentSchema = z
       .strict(),
   })
   .strict();
+
+export const interviewMapContentSchema = z
+  .object({
+    intro: nonEmptyString,
+    referenceRepo: z
+      .object({ label: nonEmptyString, href: contentHrefSchema })
+      .strict(),
+    tracks: z.array(
+      z
+        .object({
+          id: contentId,
+          label: nonEmptyString,
+          body: nonEmptyString,
+          items: z.array(
+            z
+              .object({
+                label: nonEmptyString,
+                reference: contentHrefSchema,
+                answers: z.array(
+                  z
+                    .object({
+                      projectId: contentId,
+                      depth: nonEmptyString,
+                    })
+                    .strict(),
+                ),
+              })
+              .strict(),
+          ),
+        })
+        .strict(),
+    ),
+    gaps: z
+      .object({
+        title: nonEmptyString,
+        body: nonEmptyString,
+        items: z.array(nonEmptyString),
+      })
+      .strict(),
+  })
+  .strict();


## `feat(content): 큐레이션 schema와 타입 export 추가`

diff --git a/src/lib/content-schema.ts b/src/lib/content-schema.ts
index d65e02d..ca72598 100644
--- a/src/lib/content-schema.ts
+++ b/src/lib/content-schema.ts
@@ -904,3 +904,50 @@ export const interviewMapContentSchema = z
       .strict(),
   })
   .strict();
+
+export const curationContentSchema = z
+  .object({
+    intro: nonEmptyString,
+    criteria: z
+      .object({
+        title: nonEmptyString,
+        items: z.array(
+          z.object({ title: nonEmptyString, body: nonEmptyString }).strict(),
+        ),
+      })
+      .strict(),
+    categories: z.array(
+      z
+        .object({
+          id: contentId,
+          label: nonEmptyString,
+          rationale: nonEmptyString,
+          projectIds: z.array(contentId),
+        })
+        .strict(),
+    ),
+    omissions: z
+      .object({
+        title: nonEmptyString,
+        body: nonEmptyString,
+        items: z.array(
+          z.object({ title: nonEmptyString, body: nonEmptyString }).strict(),
+        ),
+      })
+      .strict(),
+    nextReview: z
+      .object({ title: nonEmptyString, body: nonEmptyString })
+      .strict(),
+  })
+  .strict();
+
+export type ProjectGroup = z.infer<typeof projectGroupSchema>;
+export type ProjectMetric = z.infer<typeof projectMetricSchema>;
+export type ProjectMetricFilter = z.infer<typeof projectMetricFilterSchema>;
+export type PortfolioProjectSource = z.infer<
+  typeof portfolioProjectSourceSchema
+>;
+export type ProjectsContentSource = z.infer<typeof projectsContentSchema>;
+export type PresentationContentSource = z.infer<
+  typeof presentationContentSchema
+>;


