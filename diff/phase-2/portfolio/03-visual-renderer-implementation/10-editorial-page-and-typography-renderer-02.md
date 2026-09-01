## `feat(editorial): 홈 hero spread 추가`

diff --git a/src/designs/editorial/editorial-route.tsx b/src/designs/editorial/editorial-route.tsx
index f22321c..5cc35af 100644
--- a/src/designs/editorial/editorial-route.tsx
+++ b/src/designs/editorial/editorial-route.tsx
@@ -302,3 +302,74 @@ function ProjectIndexItem({
     </article>
   );
 }
+
+function HomeRoute({ content, contentDebug }: EditorialRouteProps) {
+  const projects = content.projects;
+  const featured = projects.filter((project) => project.featured);
+  const selected = featured.length > 0 ? featured : projects.slice(0, 4);
+  const lead = selected[0];
+  const preferredLinks = getPreferredContactLinks(content);
+  const homeCopy = content.presentation.home.editorial;
+  const sharedCopy = content.presentation.home.shared;
+  const ui = content.presentation.ui;
+
+  return (
+    <>
+      {homeCopy.sections.map((section) => {
+        switch (section) {
+          case "hero":
+            return (
+              <section className={styles.homeHero} key={section}>
+                <div className={styles.heroIssue}>
+                  <span>
+                    {homeCopy.hero.issueTemplate.replace(
+                      "{year}",
+                      String(new Date().getFullYear()),
+                    )}
+                  </span>
+                  <span>{content.profile.location}</span>
+                </div>
+                <div className={styles.heroTitleBlock}>
+                  <DebugNote
+                    enabled={contentDebug}
+                    prefix={ui.debugPrefix}
+                  >
+                    profile.json
+                  </DebugNote>
+                  <p className={styles.heroRole}>{content.profile.role}</p>
+                  <h1>{content.profile.headline}</h1>
+                </div>
+                <p className={styles.heroSummary}>{content.profile.summary}</p>
+                <div className={styles.heroByline}>
+                  {content.profile.photo ? (
+                    <Image
+                      alt={content.profile.photo.alt}
+                      className={styles.portrait}
+                      height={160}
+                      priority
+                      src={content.profile.photo.src}
+                      width={160}
+                    />
+                  ) : (
+                    <span className={styles.portraitFallback} aria-hidden="true">
+                      {content.profile.name.slice(0, 1)}
+                    </span>
+                  )}
+                  <div>
+                    <strong>{content.profile.name}</strong>
+                    <span>{content.profile.availability}</span>
+                  </div>
+                </div>
+                <Link
+                  className={styles.heroAction}
+                  href={editorialHref("/projects", contentDebug)}
+                >
+                  {homeCopy.hero.primaryActionLabel} <Arrow />
+                </Link>
+              </section>
+            );
+        }
+      })}
+    </>
+  );
+}


## `feat(editorial): 프로젝트 archive route 추가`

diff --git a/src/designs/editorial/editorial-route.tsx b/src/designs/editorial/editorial-route.tsx
index bc381b8..3f823b7 100644
--- a/src/designs/editorial/editorial-route.tsx
+++ b/src/designs/editorial/editorial-route.tsx
@@ -486,3 +486,95 @@ function HomeRoute({ content, contentDebug }: EditorialRouteProps) {
     </>
   );
 }
+
+function ProjectsRoute({ content, contentDebug }: EditorialRouteProps) {
+  const projects = content.projects;
+  const grouped = content.projectGroups
+    .map((group) => ({
+      group,
+      items: projects.filter((project) => project.groupId === group.id),
+    }))
+    .filter(({ items }) => items.length > 0);
+  const copy = content.presentation.pages.projects.editorial;
+  const ui = content.presentation.ui;
+
+  return (
+    <>
+      <section className={styles.pageHero}>
+        <div className={styles.pageHeroNumber}>01</div>
+        <div>
+          <DebugNote enabled={contentDebug} prefix={ui.debugPrefix}>
+            projects.json / presentation.pages.projects
+          </DebugNote>
+          <h1>{copy.hero.title}</h1>
+        </div>
+        <p>{copy.hero.body}</p>
+      </section>
+
+      <section className={styles.archiveOverview} aria-label={copy.archiveAriaLabel}>
+        {content.projectMetrics.map((metric) => (
+          <div key={metric.id}>
+            <strong>{getProjectMetricValue(metric.id, content)}</strong>
+            <span>{metric.label}</span>
+          </div>
+        ))}
+      </section>
+
+      <div className={styles.archiveSections}>
+        {grouped.length > 0 ? grouped.map(({ group, items }, groupIndex) => (
+          <section className={styles.archiveGroup} key={group.id}>
+            <header>
+              <span>
+                {copy.groupKickerTemplate.replace(
+                  "{number}",
+                  twoDigits(groupIndex),
+                )}
+              </span>
+              <h2>{group.label}</h2>
+              <p>{group.description}</p>
+            </header>
+            <div className={styles.projectIndex}>
+              {items.map((project) => (
+                <ProjectIndexItem
+                  contentDebug={contentDebug}
+                  key={project.id}
+                  project={project}
+                  readCaseStudyAriaTemplate={ui.readCaseStudyAriaTemplate}
+                />
+              ))}
+            </div>
+          </section>
+        )) : (
+          <p className={styles.emptyCopy}>{ui.emptyStates.projectsArchive}</p>
+        )}
+      </div>
+    </>
+  );
+}
+
+function EvidenceList({
+  emptyLabel,
+  items,
+  ordered = false,
+}: {
+  emptyLabel: string;
+  items: string[];
+  ordered?: boolean;
+}) {
+  if (items.length === 0) {
+    return <p className={styles.emptyCopy}>{emptyLabel}</p>;
+  }
+
+  const List = ordered ? "ol" : "ul";
+
+  return (
+    <List className={styles.evidenceList}>
+      {items.map((item, index) => (
+        <li key={`${item}-${index}`}>
+          <span>{twoDigits(index)}</span>
+          <p>{item}</p>
+        </li>
+      ))}
+    </List>
+  );
+}


## `feat(editorial): 프로젝트 상세 서사와 구조 추가`

diff --git a/src/designs/editorial/editorial-route.tsx b/src/designs/editorial/editorial-route.tsx
index 3f823b7..89d5739 100644
--- a/src/designs/editorial/editorial-route.tsx
+++ b/src/designs/editorial/editorial-route.tsx
@@ -578,3 +578,114 @@ function EvidenceList({
     </List>
   );
 }
+function ProjectDetailRoute({ content, contentDebug, project }: EditorialRouteProps) {
+  const copy = content.presentation.pages.projectDetail;
+  const ui = content.presentation.ui;
+
+  if (!project) {
+    return (
+      <section className={styles.missingPage}>
+        <p className={styles.overline}>{copy.missing.eyebrow}</p>
+        <h1>{copy.missing.title}</h1>
+        <p>{copy.missing.body}</p>
+        <Link href={editorialHref("/projects", contentDebug)}>
+          {copy.missing.actionLabel}
+        </Link>
+      </section>
+    );
+  }
+
+  const supportingImages = project.screenshots.filter(
+    (image) => image.src !== project.screenshot.src,
+  );
+  const detailLinks = getProjectDetailLinks(project);
+  const stackById = new Map(content.techStack.map((item) => [item.id, item]));
+
+  return (
+    <article className={styles.caseStudy}>
+      <header className={styles.caseHero}>
+        <div className={styles.caseMetaRail}>
+          <Link href={editorialHref("/projects", contentDebug)}>
+            ← {copy.backLabel}
+          </Link>
+          <span>{project.category}</span>
+          <span>{project.period}</span>
+          <span>{copy.facts.roleLabel} · {project.role}</span>
+          <span>{copy.facts.statusLabel} · {project.deployment.label}</span>
+        </div>
+        <div className={styles.caseTitle}>
+          <DebugNote enabled={contentDebug} prefix={ui.debugPrefix}>
+            {`projects.items[id=${project.id}]`}
+          </DebugNote>
+          <p>{copy.caseLabel} {project.order} · {project.role}</p>
+          <h1>{project.title}</h1>
+          <p className={styles.caseStandfirst}>{project.summary}</p>
+        </div>
+        <p className={styles.caseDescription}>{project.description}</p>
+        {detailLinks.length > 0 ? (
+          <nav
+            aria-label={ui.projectNavigationAriaLabel}
+            className={styles.caseLinks}
+          >
+            {detailLinks.map((link) => (
+              <EditorialContentLink
+                contentDebug={contentDebug}
+                key={`${link.type}-${link.href}`}
+                link={link}
+              >
+                {link.label} <Arrow />
+              </EditorialContentLink>
+            ))}
+          </nav>
+        ) : null}
+      </header>
+
+      <div className={styles.caseCover}>
+        <EditorialImage image={project.screenshot} priority sizes="100vw" />
+      </div>
+
+      <section className={styles.caseNarrative}>
+        <aside>
+          <span>I</span>
+          <p>{copy.sections.problem.eyebrow}</p>
+        </aside>
+        <div>
+          <h2>{copy.sections.problem.title}</h2>
+          <p className={styles.dropcap}>{project.problem}</p>
+        </div>
+        <div>
+          <p className={styles.overline}>{copy.sections.solution.eyebrow}</p>
+          <h2>{copy.sections.solution.title}</h2>
+          <p>{project.solution}</p>
+        </div>
+      </section>
+
+      <section className={styles.architectureSpread}>
+        <div className={styles.darkSectionTitle}>
+          <span>II</span>
+          <p>{copy.sections.architecture.eyebrow}</p>
+          <h2>{copy.sections.architecture.title}</h2>
+        </div>
+        <div className={styles.architectureBody}>
+          <p>{project.architecture.summary}</p>
+          <EvidenceList
+            emptyLabel={ui.emptyStates.additionalNotes}
+            items={project.architecture.items}
+            ordered
+          />
+        </div>
+        <aside>
+          <span>{copy.sections.stack.eyebrow}</span>
+          <p>{copy.sections.stack.title}</p>
+          <ul>
+            {project.stack.map((stackId) => (
+              <li key={stackId}>
+                {stackById.get(stackId)?.label ?? stackId}
+              </li>
+            ))}
+          </ul>
+        </aside>
+      </section>
+    </article>
+  );
+}


## `feat(editorial): About 정체성과 원칙 소개 추가`

diff --git a/src/designs/editorial/editorial-route.tsx b/src/designs/editorial/editorial-route.tsx
index 71a156d..08a8127 100644
--- a/src/designs/editorial/editorial-route.tsx
+++ b/src/designs/editorial/editorial-route.tsx
@@ -761,3 +761,59 @@ function ProjectDetailRoute({ content, contentDebug, project }: EditorialRoutePr
     </article>
   );
 }
+function AboutRoute({ content, contentDebug }: EditorialRouteProps) {
+  const pageCopy = content.presentation.pages.about;
+  const ui = content.presentation.ui;
+
+  return (
+    <>
+      <section className={styles.profileHero}>
+        <div>
+          <p className={styles.overline}>
+            {pageCopy.editorial.heroEyebrowTemplate.replace(
+              "{handle}",
+              content.profile.handle,
+            )}
+          </p>
+          <DebugNote enabled={contentDebug} prefix={ui.debugPrefix}>
+            profile.json
+          </DebugNote>
+          <h1>{pageCopy.hero.title}</h1>
+          <p className={styles.standfirst}>{content.profile.headline}</p>
+          <ul className={styles.profileFacts} aria-label={pageCopy.hero.title}>
+            <li>{content.profile.name} · {content.profile.koreanName}</li>
+            <li>{content.profile.role}</li>
+            <li>{content.profile.location}</li>
+            <li>{content.profile.availability}</li>
+          </ul>
+        </div>
+        <p className={styles.profileSummary}>{content.profile.summary}</p>
+        {content.profile.photo ? (
+          <figure className={styles.profilePortrait}>
+            <Image
+              alt={content.profile.photo.alt}
+              height={1000}
+              priority
+              sizes="(max-width: 768px) 90vw, 35vw"
+              src={content.profile.photo.src}
+              width={800}
+            />
+          </figure>
+        ) : null}
+      </section>
+
+      <section className={styles.principlesSpread}>
+        <SectionKicker number="01">{pageCopy.principles.title}</SectionKicker>
+        <div>
+          {content.profile.principles.map((principle, index) => (
+            <article key={`${principle.title}-${index}`}>
+              <span>{twoDigits(index)}</span>
+              <h2>{principle.title}</h2>
+              <p>{principle.body}</p>
+            </article>
+          ))}
+        </div>
+      </section>
+    </>
+  );
+}


## `feat(editorial): Resume 정체성과 프로젝트 경력 추가`

diff --git a/src/designs/editorial/editorial-route.tsx b/src/designs/editorial/editorial-route.tsx
index a5599c9..5e99a75 100644
--- a/src/designs/editorial/editorial-route.tsx
+++ b/src/designs/editorial/editorial-route.tsx
@@ -963,3 +963,80 @@ function AboutRoute({ content, contentDebug }: EditorialRouteProps) {
     </>
   );
 }
+function ResumeRoute({ content, contentDebug }: EditorialRouteProps) {
+  const pageCopy = content.presentation.pages.resume;
+  const projects = getResumeProjects(content);
+  const ui = content.presentation.ui;
+
+  return (
+    <>
+      <section className={styles.resumeHeader}>
+        <div>
+          <p className={styles.overline}>{pageCopy.editorial.heroEyebrow}</p>
+          <DebugNote enabled={contentDebug} prefix={ui.debugPrefix}>
+            resume.json
+          </DebugNote>
+          <h1>{pageCopy.hero.title}</h1>
+        </div>
+        <p>{pageCopy.hero.body}</p>
+        {content.resume.downloadUrl ? (
+          <a className={styles.downloadLink} href={content.resume.downloadUrl}>
+            {pageCopy.hero.downloadLabel} <Arrow />
+          </a>
+        ) : null}
+      </section>
+
+      <div className={styles.resumeBody}>
+        <aside className={styles.resumeIdentity}>
+          <p>{content.profile.name} · {content.profile.koreanName}</p>
+          <h2>{content.profile.role}</h2>
+          <dl>
+            <div>
+              <dt>{pageCopy.identity.locationLabel}</dt>
+              <dd>{content.profile.location}</dd>
+            </div>
+            <div>
+              <dt>{pageCopy.identity.availabilityLabel}</dt>
+              <dd>{content.profile.availability}</dd>
+            </div>
+          </dl>
+        </aside>
+        <div className={styles.resumeSections}>
+          <section>
+            <span>01</span>
+            <h2>{pageCopy.summary.title}</h2>
+            <div className={styles.resumeSummary}>
+              {content.resume.summary.map((paragraph, index) => (
+                <p key={`${paragraph}-${index}`}>{paragraph}</p>
+              ))}
+            </div>
+          </section>
+          <section>
+            <span>02</span>
+            <h2>{pageCopy.projects.title}</h2>
+            <div className={styles.resumeProjects}>
+              {projects.length > 0 ? (
+                projects.map((project, index) => (
+                  <article key={project.id}>
+                    <p>{twoDigits(index)} · {project.period} · {project.role}</p>
+                    <h3>{project.title}</h3>
+                    <p>{project.summary}</p>
+                    <span>{getProjectTags(project).join(" · ")}</span>
+                    <Link
+                      className={styles.resumeCaseLink}
+                      href={editorialHref(`/projects/${project.id}`, contentDebug)}
+                    >
+                      {pageCopy.projects.caseStudyLabel} <Arrow />
+                    </Link>
+                  </article>
+                ))
+              ) : (
+                <p className={styles.emptyCopy}>{ui.emptyStates.projectsArchive}</p>
+              )}
+            </div>
+          </section>
+        </div>
+      </div>
+    </>
+  );
+}


## `feat(editorial): Contact desk route 추가`

diff --git a/src/designs/editorial/editorial-route.tsx b/src/designs/editorial/editorial-route.tsx
index 52c547e..8dbb838 100644
--- a/src/designs/editorial/editorial-route.tsx
+++ b/src/designs/editorial/editorial-route.tsx
@@ -1088,3 +1088,60 @@ function ResumeRoute({ content, contentDebug }: EditorialRouteProps) {
     </>
   );
 }
+
+function ContactRoute({ content, contentDebug }: EditorialRouteProps) {
+  const preferredLinks = getPreferredContactLinks(content);
+  const pageCopy = content.presentation.pages.contact;
+  const ui = content.presentation.ui;
+
+  return (
+    <>
+      <section className={styles.contactHero}>
+        <p className={styles.overline}>
+          {pageCopy.editorial.heroEyebrowTemplate.replace(
+            "{location}",
+            content.profile.location,
+          )}
+        </p>
+        <DebugNote enabled={contentDebug} prefix={ui.debugPrefix}>
+          contact.json / links.json
+        </DebugNote>
+        <h1>{content.contact.title}</h1>
+        <p>{content.contact.intro}</p>
+      </section>
+      <section className={styles.contactDesk}>
+        <div className={styles.availabilityCard}>
+          <span>{ui.nowLabel}</span>
+          <h2>{pageCopy.availability.title}</h2>
+          <p>{content.contact.availability}</p>
+        </div>
+        <div className={styles.contactLinks}>
+          {preferredLinks.length > 0 ? (
+            preferredLinks.map((link, index) => (
+              <EditorialContentLink
+                className={styles.contactLinkCard}
+                contentDebug={contentDebug}
+                key={link.id ?? link.href}
+                link={link}
+              >
+                <span>{twoDigits(index)}</span>
+                <strong>{link.label}</strong>
+                <Arrow />
+              </EditorialContentLink>
+            ))
+          ) : (
+            <p className={styles.emptyCopy}>{ui.emptyStates.contactLinks}</p>
+          )}
+        </div>
+        <aside className={styles.contactNotes}>
+          <p className={styles.overline}>{pageCopy.notes.title}</p>
+          <ul>
+            {content.contact.notes.map((note) => (
+              <li key={note}>{note}</li>
+            ))}
+          </ul>
+        </aside>
+      </section>
+    </>
+  );
+}


## `feat(editorial): Journey milestone spread 추가`

diff --git a/src/designs/editorial/editorial-route.tsx b/src/designs/editorial/editorial-route.tsx
index 8dbb838..a13ec79 100644
--- a/src/designs/editorial/editorial-route.tsx
+++ b/src/designs/editorial/editorial-route.tsx
@@ -1145,3 +1145,76 @@ function ContactRoute({ content, contentDebug }: EditorialRouteProps) {
     </>
   );
 }
+function JourneyRoute({ content, contentDebug }: EditorialRouteProps) {
+  const copy = content.presentation.pages.journey;
+  const narrative = content.journeyNarrative;
+  const ui = content.presentation.ui;
+
+  return (
+    <>
+      <section className={styles.pageHero}>
+        <div className={styles.pageHeroNumber}>05</div>
+        <div>
+          <p className={styles.overline}>{copy.hero.eyebrow}</p>
+          <DebugNote enabled={contentDebug} prefix={ui.debugPrefix}>
+            journey-narrative.json
+          </DebugNote>
+          <h1>{copy.hero.title}</h1>
+        </div>
+        <p>{narrative.intro}</p>
+      </section>
+
+      <section className={styles.milestoneSpread}>
+        <SectionKicker number="01">{copy.narrative.title}</SectionKicker>
+        <p className={styles.sectionLead}>{copy.narrative.body}</p>
+        <ol>
+          {narrative.milestones.length > 0 ? narrative.milestones.map((milestone, index) => {
+            const anchorProjects = milestone.anchorProjectIds
+              .map((id) => content.projects.find((project) => project.id === id))
+              .filter((item): item is PortfolioProject => Boolean(item));
+
+            return (
+              <li key={milestone.id}>
+                <div className={styles.milestoneDate}>
+                  <b>{twoDigits(index)}</b>
+                  <span>{milestone.date}</span>
+                </div>
+                <div className={styles.milestoneStory}>
+                  <p>{copy.narrative.labels.state} · {milestone.state}</p>
+                  <h2>{milestone.title}</h2>
+                  <dl>
+                    <div>
+                      <dt>{copy.narrative.labels.reason}</dt>
+                      <dd>{milestone.reason}</dd>
+                    </div>
+                    <div>
+                      <dt>{copy.narrative.labels.result}</dt>
+                      <dd>{milestone.result}</dd>
+                    </div>
+                  </dl>
+                  {anchorProjects.length > 0 ? (
+                    <nav
+                      aria-label={ui.projectNavigationAriaLabel}
+                      className={styles.milestoneLinks}
+                    >
+                      {anchorProjects.map((project) => (
+                        <Link
+                          href={editorialHref(`/projects/${project.id}`, contentDebug)}
+                          key={project.id}
+                        >
+                          {project.title} <Arrow />
+                        </Link>
+                      ))}
+                    </nav>
+                  ) : null}
+                </div>
+              </li>
+            );
+          }) : (
+            <li className={styles.emptyCopy}>{ui.emptyStates.journey}</li>
+          )}
+        </ol>
+      </section>
+    </>
+  );
+}


