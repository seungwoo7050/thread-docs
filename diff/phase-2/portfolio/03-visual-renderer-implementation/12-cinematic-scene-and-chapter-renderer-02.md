## `feat(cinematic-project): 상세 hero와 매체 구성`

diff --git a/src/designs/cinematic/cinematic-route.tsx b/src/designs/cinematic/cinematic-route.tsx
index 27c7c86..4f205f6 100644
--- a/src/designs/cinematic/cinematic-route.tsx
+++ b/src/designs/cinematic/cinematic-route.tsx
@@ -324,3 +324,43 @@ export function ProjectsView({ content, contentDebug }: DesignRouteProps) {
     </>
   );
 }
+
+export function ProjectDetailView({ content, contentDebug, project }: DesignRouteProps) {
+  const copy = content.presentation.pages.projectDetail;
+
+  if (!project) {
+    return (
+      <section className={styles.contactHero}>
+        <ChapterLabel index={1}>{copy.missing.eyebrow}</ChapterLabel>
+        <h1>{copy.missing.title}</h1>
+        <p>{copy.missing.body}</p>
+        <Link className={styles.textLink} href={routeHref("/projects", contentDebug)}>
+          {copy.missing.actionLabel} <span aria-hidden="true">→</span>
+        </Link>
+      </section>
+    );
+  }
+
+  return (
+    <article className={styles.caseStudy}>
+      <header className={styles.caseHero}>
+        <Link href={routeHref("/projects", contentDebug)}>← {copy.backLabel}</Link>
+        <p>{project.category} · {project.period}</p>
+        <h1>{project.title}</h1>
+        <p className={styles.lede}>{project.summary}</p>
+        <p>{project.description}</p>
+        <dl className={styles.identityList}>
+          <div>
+            <dt>{copy.facts.roleLabel}</dt>
+            <dd>{project.role}</dd>
+          </div>
+          <div>
+            <dt>{copy.facts.statusLabel}</dt>
+            <dd>{project.deployment.label}</dd>
+          </div>
+        </dl>
+      </header>
+      <Media alt={project.screenshot.alt} priority src={project.screenshot.src} />
+    </article>
+  );
+}


## `feat(cinematic-project): 상세 서사와 gallery 구성`

diff --git a/src/designs/cinematic/cinematic-route.tsx b/src/designs/cinematic/cinematic-route.tsx
index 4f205f6..a032d66 100644
--- a/src/designs/cinematic/cinematic-route.tsx
+++ b/src/designs/cinematic/cinematic-route.tsx
@@ -3,6 +3,7 @@ import Link from "next/link";
 import { Fragment } from "react";
 import { DesignSwitcher } from "@/components/portfolio/design-switcher";
 import {
+  getProjectDetailLinks,
   getTemplateHref,
   type ContentLink,
   type PortfolioProject,
@@ -341,6 +342,17 @@ export function ProjectDetailView({ content, contentDebug, project }: DesignRout
     );
   }
 
+  const detailSections = [
+    { label: copy.sections.problem.title, body: project.problem },
+    { label: copy.sections.solution.title, body: project.solution },
+    { label: copy.sections.architecture.title, body: project.architecture.summary },
+  ].filter((section) => section.body);
+  const stackById = new Map(content.techStack.map((item) => [item.id, item]));
+  const detailLinks = getProjectDetailLinks(project);
+  const supportingImages = project.screenshots.filter(
+    (image) => image.src !== project.screenshot.src,
+  );
+
   return (
     <article className={styles.caseStudy}>
       <header className={styles.caseHero}>
@@ -361,6 +373,63 @@ export function ProjectDetailView({ content, contentDebug, project }: DesignRout
         </dl>
       </header>
       <Media alt={project.screenshot.alt} priority src={project.screenshot.src} />
+      <div className={styles.caseBody}>
+        {detailSections.map((section, index) => (
+          <section key={section.label}>
+            <ChapterLabel index={index + 1}>{section.label}</ChapterLabel>
+            <p>{section.body}</p>
+          </section>
+        ))}
+        {project.architecture?.items?.length ? (
+          <section>
+            <ChapterLabel index={4}>{copy.sections.architecture.eyebrow}</ChapterLabel>
+            <ul>{project.architecture.items.map((item) => <li key={item}>{item}</li>)}</ul>
+          </section>
+        ) : null}
+        {project.decisions.length ? (
+          <section>
+            <ChapterLabel index={5}>{copy.sections.decisions.title}</ChapterLabel>
+            <ul>{project.decisions.map((item) => <li key={item}>{item}</li>)}</ul>
+          </section>
+        ) : null}
+        {project.highlights.length ? (
+          <section>
+            <ChapterLabel index={6}>{copy.sections.highlights.title}</ChapterLabel>
+            <ul>{project.highlights.map((item) => <li key={item}>{item}</li>)}</ul>
+          </section>
+        ) : null}
+        {project.tradeoffs.length ? (
+          <section>
+            <ChapterLabel index={7}>{copy.sections.tradeoffs.title}</ChapterLabel>
+            <ul>{project.tradeoffs.map((item) => <li key={item}>{item}</li>)}</ul>
+          </section>
+        ) : null}
+        {project.results.length ? (
+          <section>
+            <ChapterLabel index={8}>{copy.sections.result.title}</ChapterLabel>
+            <ul>{project.results.map((item) => <li key={item}>{item}</li>)}</ul>
+          </section>
+        ) : null}
+        <section>
+          <ChapterLabel index={9}>{copy.sections.stack.title}</ChapterLabel>
+          <p className={styles.stack}>
+            {project.stack
+              .map((stackId) => stackById.get(stackId)?.label ?? stackId)
+              .join(" · ")}
+          </p>
+          <LinkList
+            contentDebug={contentDebug}
+            links={detailLinks}
+          />
+        </section>
+      </div>
+      {supportingImages.length > 0 ? (
+        <div className={styles.gallery}>
+          {supportingImages.map((image) => (
+            <Media alt={image.alt} key={image.src} src={image.src} />
+          ))}
+        </div>
+      ) : null}
     </article>
   );
 }


## `feat(cinematic-about): 프로필과 경력 소개 구성`

diff --git a/src/designs/cinematic/cinematic-route.tsx b/src/designs/cinematic/cinematic-route.tsx
index a032d66..a0bc382 100644
--- a/src/designs/cinematic/cinematic-route.tsx
+++ b/src/designs/cinematic/cinematic-route.tsx
@@ -433,3 +433,67 @@ export function ProjectDetailView({ content, contentDebug, project }: DesignRout
     </article>
   );
 }
+
+export function AboutView({ content }: DesignRouteProps) {
+  const copy = content.presentation.pages.about;
+  return (
+    <>
+      <section className={styles.textHero}>
+        <ChapterLabel index={1}>{copy.hero.title}</ChapterLabel>
+        <h1>{content.profile.headline}</h1>
+        <p className={styles.lede}>{content.profile.summary}</p>
+        <div className={styles.aboutIdentity}>
+          {content.profile.photo ? (
+            <Media
+              alt={content.profile.photo.alt}
+              src={content.profile.photo.src}
+            />
+          ) : null}
+          <ul className={styles.profileFacts}>
+            <li>{content.profile.name}</li>
+            <li>{content.profile.koreanName}</li>
+            <li>{content.profile.handle}</li>
+            <li>{content.profile.location}</li>
+          </ul>
+        </div>
+      </section>
+      <section className={styles.essayGrid}>
+        <div>
+          <h2>{copy.principles.title}</h2>
+          {content.profile.principles.map((principle) => (
+            <article key={principle.title}><h3>{principle.title}</h3><p>{principle.body}</p></article>
+          ))}
+        </div>
+        <div>
+          <h2>{copy.skills.title}</h2>
+          <div className={styles.focusGrid}>
+            {content.skills.focusAreas.map((area) => (
+              <article key={area.title}>
+                <h3>{area.title}</h3>
+                <p>{area.body}</p>
+              </article>
+            ))}
+          </div>
+          {content.skills.groups.map((group) => (
+            <article key={group.title}><h3>{group.title}</h3><p>{group.items.join(" · ")}</p></article>
+          ))}
+        </div>
+      </section>
+      <section className={styles.contentSection}>
+        <ChapterLabel index={2}>{copy.journey.title}</ChapterLabel>
+        <div className={styles.sectionHeading}>
+          <h2>{copy.journey.title}</h2>
+        </div>
+        <div className={styles.entryList}>
+          {content.experience.map((item) => (
+            <article key={`${item.period}-${item.title}`}>
+              <p>{item.period}</p>
+              <h3>{item.title}</h3>
+              <p>{item.body}</p>
+            </article>
+          ))}
+        </div>
+      </section>
+    </>
+  );
+}


## `feat(cinematic): 이력과 연락 route 구성`

diff --git a/src/designs/cinematic/cinematic-route.tsx b/src/designs/cinematic/cinematic-route.tsx
index 1cddcf2..7d3870b 100644
--- a/src/designs/cinematic/cinematic-route.tsx
+++ b/src/designs/cinematic/cinematic-route.tsx
@@ -564,3 +564,83 @@ export function AboutView({ content, contentDebug }: DesignRouteProps) {
     </>
   );
 }
+
+
+export function ResumeView({ content, contentDebug }: DesignRouteProps) {
+  const copy = content.presentation.pages.resume;
+  const selected = content.resume.projectIds
+    .map((id) => content.projects.find((project) => project.id === id))
+    .filter((project): project is PortfolioProject => Boolean(project));
+
+  return (
+    <>
+      <section className={styles.textHero}>
+        <ChapterLabel index={1}>{copy.hero.title}</ChapterLabel>
+        <p className={styles.kicker}>{content.profile.role}</p>
+        <h1>{content.profile.name}</h1>
+        <p className={styles.lede}>{copy.hero.body}</p>
+        <dl className={styles.identityList}>
+          <div><dt>{copy.identity.locationLabel}</dt><dd>{content.profile.location}</dd></div>
+          <div><dt>{copy.identity.availabilityLabel}</dt><dd>{content.profile.availability}</dd></div>
+        </dl>
+        {content.resume.downloadUrl ? (
+          <a className={styles.textLink} href={content.resume.downloadUrl}>
+            {copy.hero.downloadLabel} <span aria-hidden="true">↗</span>
+          </a>
+        ) : null}
+      </section>
+      <section className={styles.resumeGrid}>
+        <div><h2>{copy.summary.title}</h2>{content.resume.summary.map((line) => <p key={line}>{line}</p>)}</div>
+        <div>
+          <h2>{copy.projects.title}</h2>
+          {selected.map((project) => (
+            <article key={project.id}>
+              <p>{project.period} · {project.role}</p>
+              <h3>{project.title}</h3>
+              <p>{project.summary}</p>
+              <Link href={routeHref(`/projects/${project.id}`, contentDebug)}>
+                {copy.projects.caseStudyLabel}<span aria-hidden="true">↗</span>
+              </Link>
+            </article>
+          ))}
+        </div>
+        <div><h2>{copy.experience.title}</h2>{content.experience.map((item) => <article key={`${item.title}-${item.period}`}><h3>{item.title}</h3><p>{item.period}</p><p>{item.body}</p></article>)}</div>
+        <div><h2>{copy.training.title}</h2>{content.resume.training.map((item) => <article key={`${item.name}-${item.period}`}><h3>{item.name}</h3><p>{item.period}</p><p>{item.description}</p></article>)}</div>
+        <div><h2>{copy.education.title}</h2>{content.resume.education.map((item) => <article key={`${item.name}-${item.period}`}><h3>{item.name}</h3><p>{item.period}</p><p>{item.description}</p></article>)}</div>
+        <div>
+          <h2>{copy.notes.title}</h2>
+          {content.resume.notes.length > 0 ? (
+            <ul>{content.resume.notes.map((note) => <li key={note}>{note}</li>)}</ul>
+          ) : (
+            <p>{content.presentation.ui.emptyStates.additionalNotes}</p>
+          )}
+        </div>
+      </section>
+    </>
+  );
+}
+
+export function ContactView({ content, contentDebug }: DesignRouteProps) {
+  const linksById = new Map(content.links.map((link) => [link.id, link]));
+  const preferred = content.contact.preferred
+    .map((id) => linksById.get(id))
+    .filter((link): link is ContentLink => Boolean(link));
+  const links = preferred.length > 0
+    ? preferred
+    : content.links.filter((link) => link.placements?.includes("contact"));
+
+  return (
+    <section className={styles.contactHero}>
+      <ChapterLabel index={1}>{content.contact.title}</ChapterLabel>
+      <h1>{content.contact.title}</h1>
+      <p className={styles.lede}>{content.contact.intro}</p>
+      <p>{content.contact.availability}</p>
+      {links.length > 0 ? (
+        <LinkList contentDebug={contentDebug} links={links} />
+      ) : (
+        <p>{content.presentation.ui.emptyStates.contactLinks}</p>
+      )}
+      <ul>{content.contact.notes.map((note) => <li key={note}>{note}</li>)}</ul>
+    </section>
+  );
+}


## `style(cinematic): 인터뷰 근거와 반응형 동작 구성`

diff --git a/src/designs/cinematic/cinematic.module.css b/src/designs/cinematic/cinematic.module.css
index a217f5c..6074765 100644
--- a/src/designs/cinematic/cinematic.module.css
+++ b/src/designs/cinematic/cinematic.module.css
@@ -130,3 +130,43 @@ a:hover .media img { filter: saturate(1); transform: scale(1.025); }
 .answerList { display: grid; gap: 1rem; }
 .answerList article { border-left: 1px solid var(--amber); border-top: 0 !important; margin-top: 0 !important; padding: 0 0 0 1rem !important; }
 .answerList article>span { color: var(--amber); display: block; font-size: .62rem; letter-spacing: .1em; margin-bottom: .5rem; text-transform: uppercase; }
+.answerList article>a { display: inline-flex; font-weight: 650; min-height: 2.75rem; text-decoration: none; }
+.answerList article p { margin: .5rem 0 0; }
+.answerList article strong { color: var(--paper); display: block; font-size: .62rem; letter-spacing: .08em; text-transform: uppercase; }
+.gapsSection { background: #111318; display: grid; gap: 5vw; grid-template-columns: minmax(16rem,.75fr) minmax(18rem,1.25fr); }
+.gapsSection ul { display: grid; gap: 0; list-style: none; margin: 0; padding: 0; }
+.gapsSection li { border-top: 1px solid rgba(217,210,196,.16); color: var(--muted); line-height: 1.7; padding: 1.25rem 0; }
+
+@media (max-width: 980px) {
+  .header { grid-template-columns: minmax(0,1fr) auto auto; padding: .75rem 1rem; }
+  .desktopNav { display: none; }
+  .mobileNav { display: block; }
+  .hero,.projectChapter,.statement,.dualPanel { grid-template-columns: 1fr; }
+  .heroMedia,.heroMedia>.media { min-height: 55vw; }
+  .projectChapter { min-height: 0; }
+  .stickyCopy { position: static; }
+  .statement { gap: 2rem; }
+  .entryList article,.currentChapter,.gapsSection { grid-template-columns: 1fr; }
+}
+
+@media (max-width: 640px) {
+  .header { backdrop-filter: none; grid-template-columns: minmax(0,1fr) auto; }
+  .brand { grid-column: 1 / -1; grid-row: 1; }
+  .switcher { grid-column: 1; grid-row: 2; justify-content: flex-start; }
+  .mobileNav { grid-column: 2; grid-row: 2; }
+  .heroCopy,.indexHero,.textHero,.contactHero,.caseHero { padding: 6rem 1.25rem 4rem; }
+  .hero h1,.indexHero h1,.textHero h1,.contactHero h1,.caseHero h1 { font-size: clamp(3.2rem,18vw,5.4rem); }
+  .statement,.projectChapter,.dualPanel { padding: 5rem 1.25rem; }
+  .focusGrid,.caseBody,.gallery,.essayGrid,.resumeGrid,.trackGrid,.contentGrid,.milestoneFacts { grid-template-columns: 1fr; }
+  .aboutIdentity { grid-template-columns: 1fr; }
+  .caseBody,.essayGrid,.resumeGrid,.trackGrid { padding-left: 1.25rem; padding-right: 1.25rem; }
+  .caseBody section,.essayGrid>div,.resumeGrid>div,.trackGrid>article { min-height: 0; padding: 1.5rem; }
+  .contentSection,.gapsSection,.currentChapter { padding: 5rem 1.25rem; }
+  .timeline li,.archiveTimeline li { gap: 1rem; grid-template-columns: 2rem 1fr; }
+  .footer { align-items: flex-start; flex-direction: column; gap: 1rem; }
+}
+
+@media (prefers-reduced-motion: reduce) {
+  .site * { animation: none !important; scroll-behavior: auto !important; transition: none !important; }
+  .site a:hover .media img { transform: none; }
+}


## `feat(cinematic-journey): 여정 archive 구성`

diff --git a/src/designs/cinematic/cinematic-route.tsx b/src/designs/cinematic/cinematic-route.tsx
index 7d3870b..fcf1ad1 100644
--- a/src/designs/cinematic/cinematic-route.tsx
+++ b/src/designs/cinematic/cinematic-route.tsx
@@ -644,3 +644,90 @@ export function ContactView({ content, contentDebug }: DesignRouteProps) {
     </section>
   );
 }
+
+
+export function JourneyView({ content, contentDebug }: DesignRouteProps) {
+  const copy = content.presentation.pages.journey;
+  const narrative = content.journeyNarrative;
+
+  return (
+    <>
+      <section className={styles.textHero}>
+        <ChapterLabel index={1}>{copy.hero.eyebrow}</ChapterLabel>
+        <h1>{copy.hero.title}</h1>
+        <p>{narrative.intro}</p>
+      </section>
+      <section className={styles.contentSection}>
+        <div className={styles.sectionHeading}>
+          <ChapterLabel index={2}>{copy.narrative.title}</ChapterLabel>
+          <h2>{copy.narrative.title}</h2>
+          <p>{copy.narrative.body}</p>
+        </div>
+      <ol className={styles.timeline}>
+        {narrative.milestones.map((milestone, index) => {
+          const projects = milestone.anchorProjectIds
+            .map((projectId) => content.projects.find((project) => project.id === projectId))
+            .filter((project): project is PortfolioProject => Boolean(project));
+
+          return (
+          <li key={milestone.id}>
+            <span>{String(index + 1).padStart(2, "0")}</span>
+            <div>
+              <p>{milestone.date}</p>
+              <h2>{milestone.title}</h2>
+              <dl className={styles.milestoneFacts}>
+                <div><dt>{copy.narrative.labels.state}</dt><dd>{milestone.state}</dd></div>
+                <div><dt>{copy.narrative.labels.reason}</dt><dd>{milestone.reason}</dd></div>
+                <div><dt>{copy.narrative.labels.result}</dt><dd>{milestone.result}</dd></div>
+              </dl>
+              {projects.length > 0 ? (
+                <div className={styles.evidenceLinks}>
+                  {projects.map((project) => (
+                    <Link
+                      href={routeHref(`/projects/${project.id}`, contentDebug)}
+                      key={project.id}
+                    >
+                      {copy.now.anchorLabel}: {project.title} <span aria-hidden="true">↗</span>
+                    </Link>
+                  ))}
+                </div>
+              ) : null}
+            </div>
+          </li>
+          );
+        })}
+      </ol>
+      </section>
+      <section className={styles.contentSection}>
+        <div className={styles.sectionHeading}>
+          <ChapterLabel index={3}>{copy.timeline.title}</ChapterLabel>
+          <h2>{copy.timeline.title}</h2>
+          <p>{copy.timeline.body}</p>
+        </div>
+        <ol className={styles.archiveTimeline}>
+          {content.journey.map((item, index) => (
+            <li key={`${item.date}-${item.title}`}>
+              <span>{String(index + 1).padStart(2, "0")}</span>
+              <div>
+                <time>{item.endDate ? `${item.date} — ${item.endDate}` : item.date}</time>
+                <p>{item.category}</p>
+                <h3>{item.title}</h3>
+                <p>{item.body}</p>
+                {item.projectId ? (
+                  <Link href={routeHref(`/projects/${item.projectId}`, contentDebug)}>
+                    {copy.now.anchorLabel} <span aria-hidden="true">↗</span>
+                  </Link>
+                ) : null}
+              </div>
+            </li>
+          ))}
+        </ol>
+      </section>
+      <section className={styles.currentChapter}>
+        <ChapterLabel index={4}>{copy.now.title}</ChapterLabel>
+        <h2>{narrative.currentPosition.title}</h2>
+        <p>{narrative.currentPosition.body}</p>
+      </section>
+    </>
+  );
+}


## `feat(cinematic-interview): 인터뷰 근거 map 구성`

diff --git a/src/designs/cinematic/cinematic-route.tsx b/src/designs/cinematic/cinematic-route.tsx
index fcf1ad1..9e20fa3 100644
--- a/src/designs/cinematic/cinematic-route.tsx
+++ b/src/designs/cinematic/cinematic-route.tsx
@@ -731,3 +731,89 @@ export function JourneyView({ content, contentDebug }: DesignRouteProps) {
     </>
   );
 }
+
+
+export function InterviewMapView({ content, contentDebug }: DesignRouteProps) {
+  const copy = content.presentation.pages.interviewMap;
+  const data = content.interviewMap;
+  const projectsById = new Map(content.projects.map((project) => [project.id, project]));
+
+  return (
+    <>
+      <section className={styles.textHero}>
+        <ChapterLabel index={1}>{copy.hero.title}</ChapterLabel>
+        <h1>{copy.hero.title}</h1>
+        <p className={styles.lede}>{data.intro}</p>
+        <CinematicLink
+          className={styles.textLink}
+          contentDebug={contentDebug}
+          external
+          href={data.referenceRepo.href}
+        >
+          {data.referenceRepo.label} <span aria-hidden="true">↗</span>
+        </CinematicLink>
+      </section>
+      <section className={styles.contentSection}>
+        <div className={styles.sectionHeading}>
+          <ChapterLabel index={2}>{copy.tracks.indexLabel}</ChapterLabel>
+          <h2>{copy.tracks.title}</h2>
+        </div>
+      </section>
+      <section className={styles.trackGrid}>
+        {data.tracks.map((track, trackIndex) => (
+          <article key={track.id}>
+            <ChapterLabel index={trackIndex + 3}>{track.label}</ChapterLabel>
+            <p>{track.body}</p>
+            {track.items.map((item) => (
+              <div key={item.label}>
+                <h3>{item.label}</h3>
+                <CinematicLink
+                  className={styles.referenceLink}
+                  contentDebug={contentDebug}
+                  href={item.reference}
+                >
+                  {copy.tracks.referenceLabel} <span aria-hidden="true">↗</span>
+                </CinematicLink>
+                {item.answers.length > 0 ? (
+                  <div className={styles.answerList}>
+                    {item.answers.map((answer) => {
+                      const project = projectsById.get(answer.projectId);
+
+                      return (
+                        <article key={`${item.label}-${answer.projectId}`}>
+                          <span>{copy.tracks.answerLabel}</span>
+                          {project ? (
+                            <Link href={routeHref(`/projects/${project.id}`, contentDebug)}>
+                              {project.title} <span aria-hidden="true">↗</span>
+                            </Link>
+                          ) : (
+                            <p>{content.presentation.ui.emptyStates.noMappedEvidence}</p>
+                          )}
+                          <p><strong>{copy.tracks.depthLabel}</strong> {answer.depth}</p>
+                        </article>
+                      );
+                    })}
+                  </div>
+                ) : (
+                  <p>{copy.tracks.emptyLabel}</p>
+                )}
+              </div>
+            ))}
+          </article>
+        ))}
+      </section>
+      <section className={styles.gapsSection} aria-label={copy.gaps.ariaLabel}>
+        <div className={styles.sectionHeading}>
+          <ChapterLabel index={data.tracks.length + 3}>{copy.gaps.eyebrow}</ChapterLabel>
+          <h2>{data.gaps.title}</h2>
+          <p>{data.gaps.body}</p>
+        </div>
+        {data.gaps.items.length > 0 ? (
+          <ul>{data.gaps.items.map((item) => <li key={item}>{item}</li>)}</ul>
+        ) : (
+          <p>{content.presentation.ui.emptyStates.additionalNotes}</p>
+        )}
+      </section>
+    </>
+  );
+}


