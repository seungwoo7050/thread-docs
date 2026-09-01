## `feat(brutalist): 대표 작업과 작업 원칙 구성`

diff --git a/src/designs/brutalist/brutalist-route.tsx b/src/designs/brutalist/brutalist-route.tsx
index c9449fb..937ae5e 100644
--- a/src/designs/brutalist/brutalist-route.tsx
+++ b/src/designs/brutalist/brutalist-route.tsx
@@ -267,6 +267,9 @@ export function HomeView({
   contentDebug: boolean;
 }) {
   const homeCopy = content.presentation.home.brutalist;
+  const ui = content.presentation.ui;
+  const featured = content.projects.filter((project) => project.featured);
+  const selected = (featured.length > 0 ? featured : content.projects).slice(0, 5);
   const metrics = getHomeMetrics(content);
   return (
     <>
@@ -322,6 +325,68 @@ export function HomeView({
                 </dl>
               </section>
             );
+          case "signal":
+            return <SignalStrip key={section} text={homeCopy.signalText} />;
+          case "featured":
+            return (
+              <section className={styles.section} key={section}>
+                <SectionHeader
+                  body={homeCopy.featured.body}
+                  number="01"
+                  title={homeCopy.featured.title}
+                />
+                {selected.length > 0 ? (
+                  <ol className={styles.projectIndex}>
+                    {selected.map((project) => (
+                      <ProjectIndexRow
+                        contentDebug={contentDebug}
+                        key={project.id}
+                        project={project}
+                      />
+                    ))}
+                  </ol>
+                ) : (
+                  <EmptyBlock message={ui.emptyStates.projectsHome} />
+                )}
+                <Link
+                  className={styles.fullWidthAction}
+                  href={brutalistHref("/projects", contentDebug)}
+                >
+                  {homeCopy.featured.actionLabel}{" "}
+                  ({String(content.projects.length).padStart(2, "0")})
+                  <span aria-hidden="true">→</span>
+                </Link>
+              </section>
+            );
+          case "system":
+            return (
+              <section
+                className={`${styles.section} ${styles.blueSection}`}
+                key={section}
+              >
+                <SectionHeader
+                  body={homeCopy.system.body}
+                  number="02"
+                  title={homeCopy.system.title}
+                />
+                <div className={styles.principleGrid}>
+                  {content.profile.principles.map((principle, index) => (
+                    <article className={styles.principleCard} key={principle.title}>
+                      <span className={styles.cardNumber}>
+                        {String(index + 1).padStart(2, "0")}
+                      </span>
+                      <h3>{principle.title}</h3>
+                      <p>{principle.body}</p>
+                    </article>
+                  ))}
+                </div>
+                <div className={styles.stackWall} aria-label={homeCopy.system.title}>
+                  {content.techStack.slice(0, 18).map((item) => (
+                    <span key={item.id}>{item.label}</span>
+                  ))}
+                </div>
+              </section>
+            );
         }
       })}
     </>
@@ -391,6 +456,36 @@ export function ProjectIndexRow({
 }
 
 
+export function ProjectsView({
+  content,
+  contentDebug,
+}: {
+  content: PortfolioContent;
+  contentDebug: boolean;
+}) {
+  const pageCopy = content.presentation.pages.projects;
+  const brutalistCopy = pageCopy.brutalist;
+  const metrics = getHomeMetrics(content);
+
+  return (
+    <>
+      <section className={styles.pageHero}>
+        <h1>{brutalistCopy.hero.title}</h1>
+        <p>{brutalistCopy.hero.body}</p>
+        <dl className={styles.inlineMetrics}>
+          {metrics.slice(0, 3).map((metric) => (
+            <div key={metric.id}>
+              <dt>{metric.label}</dt>
+              <dd>{String(metric.value).padStart(2, "0")}</dd>
+            </div>
+          ))}
+        </dl>
+      </section>
+      <ContactBand content={content} contentDebug={contentDebug} />
+    </>
+  );
+}
+
 export function ActionLink({
   children,
   className,


## `feat(brutalist): 홈 여정과 프로젝트 archive 구성`

diff --git a/src/designs/brutalist/brutalist-route.tsx b/src/designs/brutalist/brutalist-route.tsx
index 937ae5e..478ba61 100644
--- a/src/designs/brutalist/brutalist-route.tsx
+++ b/src/designs/brutalist/brutalist-route.tsx
@@ -271,6 +271,8 @@ export function HomeView({
   const featured = content.projects.filter((project) => project.featured);
   const selected = (featured.length > 0 ? featured : content.projects).slice(0, 5);
   const metrics = getHomeMetrics(content);
+  const recentJourney = content.journey.slice(-4).reverse();
+
   return (
     <>
       {homeCopy.sections.map((section) => {
@@ -387,6 +389,47 @@ export function HomeView({
                 </div>
               </section>
             );
+          case "journey":
+            return (
+              <section className={styles.section} key={section}>
+                <SectionHeader
+                  body={content.presentation.home.shared.journey.body}
+                  number="03"
+                  title={content.presentation.home.shared.journey.title}
+                />
+                {recentJourney.length > 0 ? (
+                  <ol className={styles.compactTimeline}>
+                    {recentJourney.map((item, index) => (
+                      <li key={`${item.date}-${item.title}`}>
+                        <span>{String(index + 1).padStart(2, "0")}</span>
+                        <time>
+                          {item.endDate ? `${item.date}—${item.endDate}` : item.date}
+                        </time>
+                        <strong>{item.title}</strong>
+                        <p>{item.body}</p>
+                      </li>
+                    ))}
+                  </ol>
+                ) : (
+                  <EmptyBlock message={ui.emptyStates.journey} />
+                )}
+                <Link
+                  className={styles.fullWidthAction}
+                  href={brutalistHref("/journey", contentDebug)}
+                >
+                  {homeCopy.journeyActionLabel}{" "}
+                  <span aria-hidden="true">→</span>
+                </Link>
+              </section>
+            );
+          case "contact":
+            return (
+              <ContactBand
+                content={content}
+                contentDebug={contentDebug}
+                key={section}
+              />
+            );
         }
       })}
     </>
@@ -463,6 +506,7 @@ export function ProjectsView({
   content: PortfolioContent;
   contentDebug: boolean;
 }) {
+  const groups = groupProjects(content);
   const pageCopy = content.presentation.pages.projects;
   const brutalistCopy = pageCopy.brutalist;
   const metrics = getHomeMetrics(content);
@@ -481,6 +525,35 @@ export function ProjectsView({
           ))}
         </dl>
       </section>
+
+      <div className={styles.groupArchive}>
+        {groups.length > 0 ? (
+          groups.map((group, groupIndex) => (
+            <section className={styles.projectGroup} key={group.id}>
+              <header className={styles.projectGroupHeader}>
+                <span>{String(groupIndex + 1).padStart(2, "0")}</span>
+                <h2>{group.label}</h2>
+                <p>{group.description}</p>
+                <strong>{String(group.projects.length).padStart(2, "0")}</strong>
+              </header>
+              <ol className={styles.groupProjectList}>
+                {group.projects.map((project) => (
+                  <ProjectIndexRow
+                    contentDebug={contentDebug}
+                    key={project.id}
+                    project={project}
+                  />
+                ))}
+              </ol>
+            </section>
+          ))
+        ) : (
+          <EmptyBlock
+            message={content.presentation.ui.emptyStates.projectsArchive}
+          />
+        )}
+      </div>
+
       <ContactBand content={content} contentDebug={contentDebug} />
     </>
   );


## `feat(brutalist): 프로젝트 상세 hero와 소개 구성`

diff --git a/src/designs/brutalist/brutalist-route.tsx b/src/designs/brutalist/brutalist-route.tsx
index 1627a5e..9f6a4cb 100644
--- a/src/designs/brutalist/brutalist-route.tsx
+++ b/src/designs/brutalist/brutalist-route.tsx
@@ -2,6 +2,7 @@ import Image from "next/image";
 import Link from "next/link";
 import { DesignSwitcher } from "@/components/portfolio/design-switcher";
 import {
+  getProjectDetailLinks,
   getProjectMetricValue,
   getTemplateHref,
   type ContentLink,
@@ -562,6 +563,87 @@ export function ProjectsView({
   );
 }
 
+export function ProjectDetailView({
+  content,
+  contentDebug,
+  project,
+}: {
+  content: PortfolioContent;
+  contentDebug: boolean;
+  project?: PortfolioProject;
+}) {
+  const copy = content.presentation.pages.projectDetail;
+
+  if (!project) {
+    return (
+      <section className={styles.notFound}>
+        <span>{copy.missing.eyebrow}</span>
+        <h1>{copy.missing.title}</h1>
+        <p>{copy.missing.body}</p>
+        <Link
+          className={styles.primaryAction}
+          href={brutalistHref("/projects", contentDebug)}
+        >
+          {copy.missing.actionLabel}
+        </Link>
+      </section>
+    );
+  }
+
+
+  return (
+    <>
+      <article>
+        <header className={styles.detailHero}>
+          <div className={styles.detailHeroCopy}>
+            <Link
+              className={styles.backLink}
+              href={brutalistHref("/projects", contentDebug)}
+            >
+              ← {copy.backLabel}
+            </Link>
+            <p className={styles.eyebrow}>
+              {project.order} / {project.category} / {project.period}
+            </p>
+            <h1>{project.title}</h1>
+            <p className={styles.detailLead}>{project.summary}</p>
+            <dl className={styles.detailFacts}>
+              <div>
+                <dt>{copy.facts.roleLabel}</dt>
+                <dd>{project.role}</dd>
+              </div>
+              <div>
+                <dt>{copy.facts.statusLabel}</dt>
+                <dd>{project.deployment.label}</dd>
+              </div>
+            </dl>
+            <ProjectActions
+              contentDebug={contentDebug}
+              links={getProjectDetailLinks(project)}
+            />
+          </div>
+          <ProjectMedia image={project.screenshot} priority />
+        </header>
+
+        <div className={styles.detailIntro}>
+          <span>{copy.caseLabel} / {project.id}</span>
+          <p>{project.description}</p>
+        </div>
+
+      </article>
+      <nav
+        aria-label={content.presentation.ui.projectNavigationAriaLabel}
+        className={styles.nextProject}
+      >
+        <span>{copy.outroLabel}</span>
+        <Link href={brutalistHref("/projects", contentDebug)}>
+          {copy.returnToIndexLabel} <span aria-hidden="true">→</span>
+        </Link>
+      </nav>
+    </>
+  );
+}
+
 export function ProjectMedia({
   image,
   label,


## `feat(brutalist): 프로젝트 상세 본문과 gallery 구성`

diff --git a/src/designs/brutalist/brutalist-route.tsx b/src/designs/brutalist/brutalist-route.tsx
index 9f6a4cb..6898d66 100644
--- a/src/designs/brutalist/brutalist-route.tsx
+++ b/src/designs/brutalist/brutalist-route.tsx
@@ -590,6 +590,7 @@ export function ProjectDetailView({
     );
   }
 
+  const screenshots = project.screenshots;
 
   return (
     <>
@@ -630,6 +631,54 @@ export function ProjectDetailView({
           <p>{project.description}</p>
         </div>
 
+        <div className={styles.detailBody}>
+          <DetailTextSection
+            body={project.problem}
+            eyebrow={copy.sections.problem.eyebrow}
+            number="01"
+            title={copy.sections.problem.title}
+          />
+          <DetailTextSection
+            body={project.solution}
+            eyebrow={copy.sections.solution.eyebrow}
+            number="02"
+            title={copy.sections.solution.title}
+          />
+          <DetailListSection
+            emptyMessage={content.presentation.ui.emptyStates.projectDetails}
+            eyebrow={copy.sections.architecture.eyebrow}
+            intro={project.architecture.summary}
+            items={project.architecture.items}
+            number="03"
+            title={copy.sections.architecture.title}
+          />
+          {screenshots.length > 0 ? (
+            <section className={styles.detailSection}>
+              <SectionHeader
+                number="04"
+                title={copy.sections.screenshots.title}
+              />
+              <div className={styles.galleryGrid}>
+                {screenshots.map((image, index) => (
+                  <ProjectMedia
+                    image={image}
+                    key={`${image.src}-${index}`}
+                    label={`${copy.frameLabel} ${String(index + 1).padStart(2, "0")}`}
+                  />
+                ))}
+              </div>
+            </section>
+          ) : null}
+          <section className={styles.detailSection}>
+            <SectionHeader number="05" title={copy.sections.stack.title} />
+            <div className={styles.stackWall}>
+              {project.stack.map((stackId) => {
+                const stack = content.techStack.find((item) => item.id === stackId);
+                return <span key={stackId}>{stack!.label}</span>;
+              })}
+            </div>
+          </section>
+        </div>
       </article>
       <nav
         aria-label={content.presentation.ui.projectNavigationAriaLabel}
@@ -756,6 +805,53 @@ export function DetailTextSection({
 }
 
 
+export function DetailListSection({
+  emptyMessage,
+  eyebrow,
+  intro,
+  items,
+  number,
+  title,
+  tone,
+}: {
+  emptyMessage: string;
+  eyebrow?: string;
+  intro?: string;
+  items: string[];
+  number: string;
+  title: string;
+  tone?: "blue" | "yellow";
+}) {
+  return (
+    <section className={styles.detailSection}>
+      <SectionHeader number={number} title={title} />
+      {eyebrow ? <p className={styles.detailSectionLabel}>{eyebrow}</p> : null}
+      {intro ? <p className={styles.detailSectionIntro}>{intro}</p> : null}
+      {items.length > 0 ? (
+        <ol
+          className={`${styles.detailList} ${
+            tone === "blue"
+              ? styles.detailListBlue
+              : tone === "yellow"
+                ? styles.detailListYellow
+                : ""
+          }`}
+        >
+          {items.map((item, index) => (
+            <li key={`${index}-${item}`}>
+              <span>{String(index + 1).padStart(2, "0")}</span>
+              <p>{item}</p>
+            </li>
+          ))}
+        </ol>
+      ) : (
+        <EmptyBlock message={emptyMessage} />
+      )}
+    </section>
+  );
+}
+
+
 export function PageLabel({ index, label }: { index: string; label: string }) {
   return (
     <p className={styles.pageLabel}>


## `feat(brutalist): 프로필과 기술 소개 구성`

diff --git a/src/designs/brutalist/brutalist-route.tsx b/src/designs/brutalist/brutalist-route.tsx
index 6898d66..8636987 100644
--- a/src/designs/brutalist/brutalist-route.tsx
+++ b/src/designs/brutalist/brutalist-route.tsx
@@ -852,6 +852,92 @@ export function DetailListSection({
 }
 
 
+export function AboutView({
+  content,
+}: {
+  content: PortfolioContent;
+  contentDebug: boolean;
+}) {
+  const pageCopy = content.presentation.pages.about;
+  const brutalistCopy = pageCopy.brutalist;
+  return (
+    <>
+      <section className={styles.pageHero}>
+        <PageLabel
+          index="01"
+          label={renderCopyTemplate(brutalistCopy.heroEyebrowTemplate, {
+            handle: content.profile.handle,
+          })}
+        />
+        <h1>{pageCopy.hero.title}</h1>
+        <p>{content.profile.summary}</p>
+        <div className={styles.profileAside}>
+          <div className={styles.heroIdentity}>
+            <strong>{content.profile.name}</strong>
+            <span>{content.profile.koreanName}</span>
+            <span>{content.profile.handle}</span>
+            <span>{content.profile.location}</span>
+            <span>{content.profile.role}</span>
+          </div>
+          {content.profile.photo ? (
+            <figure className={styles.profilePortrait}>
+              <Image
+                alt={content.profile.photo.alt}
+                fill
+                sizes="(max-width: 720px) 100vw, 28vw"
+                src={content.profile.photo.src}
+              />
+            </figure>
+          ) : null}
+        </div>
+      </section>
+
+      <section className={`${styles.section} ${styles.yellowSection}`}>
+        <SectionHeader number="02" title={pageCopy.principles.title} />
+        <div className={styles.principleGrid}>
+          {content.profile.principles.map((principle, index) => (
+            <article className={styles.principleCard} key={principle.title}>
+              <span className={styles.cardNumber}>
+                {brutalistCopy.principleItemLabel}{" "}
+                {String(index + 1).padStart(2, "0")}
+              </span>
+              <h3>{principle.title}</h3>
+              <p>{principle.body}</p>
+            </article>
+          ))}
+        </div>
+      </section>
+
+      <section className={styles.section}>
+        <SectionHeader number="03" title={pageCopy.skills.title} />
+        <div className={styles.skillGrid}>
+          {content.skills.focusAreas.map((area, index) => (
+            <article className={styles.focusCard} key={area.title}>
+              <span>
+                {brutalistCopy.focusItemLabel} /{" "}
+                {String(index + 1).padStart(2, "0")}
+              </span>
+              <h3>{area.title}</h3>
+              <p>{area.body}</p>
+            </article>
+          ))}
+          {content.skills.groups.map((group) => (
+            <article className={styles.skillCard} key={group.title}>
+              <h3>{group.title}</h3>
+              <ul>
+                {group.items.map((item) => (
+                  <li key={item}>{item}</li>
+                ))}
+              </ul>
+            </article>
+          ))}
+        </div>
+      </section>
+    </>
+  );
+}
+
+
 export function PageLabel({ index, label }: { index: string; label: string }) {
   return (
     <p className={styles.pageLabel}>


## `feat(brutalist): 이력 hero와 경력 요약 구성`

diff --git a/src/designs/brutalist/brutalist-route.tsx b/src/designs/brutalist/brutalist-route.tsx
index 636e1e0..d54494a 100644
--- a/src/designs/brutalist/brutalist-route.tsx
+++ b/src/designs/brutalist/brutalist-route.tsx
@@ -1043,6 +1043,82 @@ export function AboutView({
 }
 
 
+export function ResumeView({
+  content,
+  contentDebug,
+}: {
+  content: PortfolioContent;
+  contentDebug: boolean;
+}) {
+  const copy = content.presentation.pages.resume;
+  return (
+    <>
+      <section className={`${styles.pageHero} ${styles.resumeHero}`}>
+        <PageLabel
+          index="01"
+          label={renderCopyTemplate(copy.brutalist.heroEyebrowTemplate, {
+            name: content.profile.name,
+          })}
+        />
+        <h1>{copy.hero.title}</h1>
+        <p>{copy.hero.body}</p>
+        <div className={styles.heroIdentity}>
+          <strong>{content.profile.role}</strong>
+          <span>
+            {copy.identity.locationLabel}: {content.profile.location}
+          </span>
+          <span>
+            {copy.identity.availabilityLabel}: {content.profile.availability}
+          </span>
+        </div>
+        {content.resume.downloadUrl ? (
+          <ActionLink
+            className={styles.primaryAction}
+            contentDebug={contentDebug}
+            href={content.resume.downloadUrl}
+          >
+            {copy.hero.downloadLabel} ↗
+          </ActionLink>
+        ) : null}
+      </section>
+
+      <section className={styles.resumeSection}>
+        <header>
+          <span>02</span>
+          <h2>{copy.summary.title}</h2>
+        </header>
+        <div className={styles.resumeSummary}>
+          {content.resume.summary.map((item, index) => (
+            <p key={`${index}-${item}`}>
+              <span>{String(index + 1).padStart(2, "0")}</span>
+              {item}
+            </p>
+          ))}
+        </div>
+      </section>
+
+      {content.experience.length > 0 ? (
+        <section className={styles.resumeSection}>
+          <header>
+            <span>03</span>
+            <h2>{copy.experience.title}</h2>
+          </header>
+          <div className={styles.resumeEntries}>
+            {content.experience.map((item) => (
+              <article key={`${item.period}-${item.title}`}>
+                <time>{item.period}</time>
+                <h3>{item.title}</h3>
+                <p>{item.body}</p>
+              </article>
+            ))}
+          </div>
+        </section>
+      ) : null}
+
+    </>
+  );
+}
+
 export function PageLabel({ index, label }: { index: string; label: string }) {
   return (
     <p className={styles.pageLabel}>


