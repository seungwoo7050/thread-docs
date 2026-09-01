# Design·Classic 컴포넌트형 렌더러 계열

## `style(theme): 포트폴리오 기본 디자인 토큰 추가`

diff --git a/src/app/globals.css b/src/app/globals.css
new file mode 100644
index 0000000..4b66820
--- /dev/null
+++ b/src/app/globals.css
@@ -0,0 +1,63 @@
+@import "tailwindcss";
+
+:root {
+  --background: #f7faf8;
+  --background-soft: #edf5f2;
+  --foreground: #14213d;
+  --muted: #5f6d7d;
+  --line: #ccd9d8;
+  --surface: #ffffff;
+  --surface-soft: #f2f7f6;
+  --surface-hover: #f8fbfb;
+  --accent: #008c89;
+  --accent-strong: #006b69;
+  --accent-soft: rgba(0, 140, 137, 0.12);
+  --signal: #4f46e5;
+  --signal-strong: #3730a3;
+  --signal-soft: rgba(79, 70, 229, 0.12);
+  --warm: #f59e0b;
+}
+
+@theme inline {
+  --color-background: var(--background);
+  --color-background-soft: var(--background-soft);
+  --color-foreground: var(--foreground);
+  --color-muted: var(--muted);
+  --color-line: var(--line);
+  --color-surface: var(--surface);
+  --color-surface-soft: var(--surface-soft);
+  --color-surface-hover: var(--surface-hover);
+  --color-accent: var(--accent);
+  --color-accent-strong: var(--accent-strong);
+  --color-accent-soft: var(--accent-soft);
+  --color-signal: var(--signal);
+  --color-signal-strong: var(--signal-strong);
+  --color-signal-soft: var(--signal-soft);
+  --color-warm: var(--warm);
+  --font-sans: "Geist", "Apple SD Gothic Neo", "Noto Sans KR", ui-sans-serif,
+    system-ui, sans-serif;
+  --font-mono: "Geist Mono", "SFMono-Regular", Consolas, ui-monospace,
+    monospace;
+}
+
+html {
+  background: var(--background);
+  scroll-behavior: smooth;
+}
+
+body {
+  background: var(--background);
+  color: var(--foreground);
+  font-family: var(--font-geist-sans), "Apple SD Gothic Neo", "Noto Sans KR",
+    ui-sans-serif, system-ui, sans-serif;
+  overflow-x: hidden;
+}
+
+::selection {
+  background: #bff4e5;
+  color: #10212c;
+}
+
+a {
+  text-underline-offset: 3px;
+}


## `feat(home): 디자인 홈 소개 영역 구성`

diff --git a/src/designs/design/home-route.tsx b/src/designs/design/home-route.tsx
new file mode 100644
index 0000000..469cb2d
--- /dev/null
+++ b/src/designs/design/home-route.tsx
@@ -0,0 +1,89 @@
+import { ArrowRightIcon } from "@/components/icons";
+import { ContentHint } from "@/components/portfolio/content-hint";
+import { ContentLinkView } from "@/components/portfolio/content-link";
+import { ProfilePhoto } from "@/components/portfolio/profile-photo";
+import { Reveal } from "@/components/portfolio/reveal";
+import { PageShell } from "@/components/portfolio/site-shell";
+import type { HomeTemplateId, PortfolioContent } from "@/lib/portfolio";
+
+export function DesignHomeRoute({
+  content,
+  contentDebug,
+}: {
+  content: PortfolioContent;
+  contentDebug: boolean;
+}) {
+  const activeTemplate: HomeTemplateId = "design";
+
+  return (
+    <PageShell
+      contentDebug={contentDebug}
+      homeTemplate={activeTemplate}
+      profile={content.profile}
+      site={content.site}
+    >
+      <HeroSection
+        activeTemplate={activeTemplate}
+        content={content}
+        contentDebug={contentDebug}
+      />
+    </PageShell>
+  );
+}
+
+function HeroSection({
+  activeTemplate,
+  content,
+  contentDebug,
+}: {
+  activeTemplate: HomeTemplateId;
+  content: PortfolioContent;
+  contentDebug: boolean;
+}) {
+  const { profile } = content;
+  const links = content.links.filter((link) =>
+    ["github", "resume", "website"].includes(link.type),
+  );
+
+  return (
+    <section className="hero-section border-b border-line">
+      <div className="mx-auto grid max-w-6xl gap-10 px-5 py-12 sm:px-8 md:py-16 lg:min-h-[calc(88vh-4rem)] lg:grid-cols-[0.86fr_1.14fr] lg:items-center">
+        <Reveal className="max-w-3xl">
+          <ContentHint
+            enabled={contentDebug}
+            path="src/content/profile.json > name/koreanName/photo/role/headline/summary + src/content/presentation.json > home.design.hero"
+          />
+          <div className="flex items-center gap-4">
+            {profile.photo ? <ProfilePhoto photo={profile.photo} /> : null}
+            <p className="text-sm font-medium text-muted">
+              {profile.name} · {profile.koreanName}
+            </p>
+          </div>
+          <h1 className="mt-6 text-5xl font-semibold leading-[0.98] tracking-normal text-foreground sm:text-6xl md:text-7xl">
+            {profile.role}
+          </h1>
+          <p className="mt-6 max-w-2xl text-xl leading-8 text-foreground md:text-2xl md:leading-9">
+            {profile.headline}
+          </p>
+          <p className="mt-5 max-w-2xl text-base leading-7 text-muted">
+            {profile.summary}
+          </p>
+          <div className="mt-9 flex flex-wrap gap-3">
+            {links.map((link) => (
+              <ContentLinkView
+                className="inline-flex h-10 items-center gap-2 rounded-md border border-line bg-surface px-4 text-sm font-semibold text-muted transition hover:-translate-y-0.5 hover:border-accent hover:text-foreground focus:outline-none focus:ring-2 focus:ring-accent focus:ring-offset-2 focus:ring-offset-background"
+                contentDebug={contentDebug}
+                homeTemplate={activeTemplate}
+                key={link.id ?? link.href}
+                link={link}
+              >
+                {link.label}
+                <ArrowRightIcon className="-rotate-45" />
+              </ContentLinkView>
+            ))}
+          </div>
+        </Reveal>
+      </div>
+    </section>
+  );
+}


## `feat(app): 콘텐츠 기반 디자인 홈 연결`

diff --git a/src/app/favicon.ico b/src/app/favicon.ico
new file mode 100644
index 0000000..718d6fe
Binary files /dev/null and b/src/app/favicon.ico differ
diff --git a/src/app/layout.tsx b/src/app/layout.tsx
index 23f1fdf..4d9da82 100644
--- a/src/app/layout.tsx
+++ b/src/app/layout.tsx
@@ -1,8 +1,21 @@
 import type { Metadata } from "next";
+import { Geist, Geist_Mono } from "next/font/google";
+import site from "@/content/site.json";
+import "./globals.css";
+
+const geistSans = Geist({
+  variable: "--font-geist-sans",
+  subsets: ["latin"],
+});
+
+const geistMono = Geist_Mono({
+  variable: "--font-geist-mono",
+  subsets: ["latin"],
+});
 
 export const metadata: Metadata = {
-  title: "Portfolio",
-  description: "Content-driven portfolio",
+  title: site.title,
+  description: site.description,
 };
 
 export default function RootLayout({
@@ -11,8 +24,11 @@ export default function RootLayout({
   children: React.ReactNode;
 }>) {
   return (
-    <html lang="ko">
-      <body>{children}</body>
+    <html
+      lang={site.language}
+      className={`${geistSans.variable} ${geistMono.variable} h-full antialiased`}
+    >
+      <body className="min-h-full flex flex-col">{children}</body>
     </html>
   );
 }
diff --git a/src/app/page.tsx b/src/app/page.tsx
index 881d940..64965b9 100644
--- a/src/app/page.tsx
+++ b/src/app/page.tsx
@@ -1,8 +1,8 @@
+import { DesignHomeRoute } from "@/designs/design/home-route";
+import { getPortfolioContent } from "@/lib/portfolio";
+
 export default function Home() {
-  return (
-    <main>
-      <h1>Portfolio</h1>
-      <p>프로젝트와 기술적 의사결정을 정리하는 공간입니다.</p>
-    </main>
-  );
+  const content = getPortfolioContent();
+
+  return <DesignHomeRoute content={content} contentDebug={false} />;
 }


## `feat(home): 애니메이션 터미널 상호작용 추가`

diff --git a/src/components/portfolio/animated-terminal.tsx b/src/components/portfolio/animated-terminal.tsx
new file mode 100644
index 0000000..0616aa0
--- /dev/null
+++ b/src/components/portfolio/animated-terminal.tsx
@@ -0,0 +1,131 @@
+"use client";
+
+import { useEffect, useMemo, useState } from "react";
+import type { ProfileContent, TerminalPresentation } from "@/lib/portfolio";
+
+function formatTerminalLine(
+  line: string,
+  {
+    profile,
+    projectCount,
+    stackCount,
+  }: {
+    profile: ProfileContent;
+    projectCount: number;
+    stackCount: number;
+  },
+) {
+  return line
+    .replaceAll("{handle}", profile.handle.toLowerCase())
+    .replaceAll("{role}", profile.role.toLowerCase())
+    .replaceAll("{location}", profile.location)
+    .replaceAll("{projectCount}", String(projectCount))
+    .replaceAll("{stackCount}", String(stackCount));
+}
+
+export function AnimatedTerminal({
+  profile,
+  projectCount,
+  stackCount,
+  terminal,
+}: {
+  profile: ProfileContent;
+  projectCount: number;
+  stackCount: number;
+  terminal: TerminalPresentation;
+}) {
+  const commands = useMemo(
+    () =>
+      terminal.commands.map((command) => ({
+        ...command,
+        output: command.output.map((line) =>
+          formatTerminalLine(line, { profile, projectCount, stackCount }),
+        ),
+      })),
+    [profile, projectCount, stackCount, terminal.commands],
+  );
+  const [commandIndex, setCommandIndex] = useState(0);
+  const [typedCommand, setTypedCommand] = useState(commands[0]?.command ?? "");
+  const [phase, setPhase] = useState<"typing" | "hold" | "erase">("hold");
+  const activeCommand = commands[commandIndex];
+
+  useEffect(() => {
+    const reduceMotion =
+      typeof window !== "undefined" &&
+      typeof window.matchMedia === "function" &&
+      window.matchMedia("(prefers-reduced-motion: reduce)").matches;
+
+    if (reduceMotion) {
+      return;
+    }
+
+    let timeout: ReturnType<typeof setTimeout>;
+
+    if (phase === "typing") {
+      if (typedCommand.length < activeCommand.command.length) {
+        timeout = setTimeout(() => {
+          setTypedCommand(activeCommand.command.slice(0, typedCommand.length + 1));
+        }, 42);
+      } else {
+        timeout = setTimeout(() => setPhase("hold"), 520);
+      }
+    }
+
+    if (phase === "hold") {
+      timeout = setTimeout(() => setPhase("erase"), 1700);
+    }
+
+    if (phase === "erase") {
+      if (typedCommand.length > 0) {
+        timeout = setTimeout(() => {
+          setTypedCommand(activeCommand.command.slice(0, typedCommand.length - 1));
+        }, 24);
+      } else {
+        timeout = setTimeout(() => {
+          setCommandIndex((current) => (current + 1) % commands.length);
+          setPhase("typing");
+        }, 220);
+      }
+    }
+
+    return () => clearTimeout(timeout);
+  }, [activeCommand.command, commands.length, phase, typedCommand]);
+
+  const shouldShowOutput = phase !== "typing" || typedCommand === activeCommand.command;
+
+  return (
+    <aside
+      aria-label="Animated portfolio terminal preview"
+      className="terminal-window mx-auto w-full max-w-xl"
+    >
+      <div className="terminal-titlebar">
+        <span className="bg-[#ff6b5f]" />
+        <span className="bg-[#f6c76f]" />
+        <span className="bg-[#67d391]" />
+        <p>{terminal.title}</p>
+      </div>
+      <div className="terminal-body">
+        <p className="terminal-line text-muted">
+          {terminal.bootLine}
+        </p>
+        <p className="terminal-line">
+          <span className="text-accent">{terminal.promptUser}</span>
+          <span className="text-muted">:</span>
+          <span className="text-signal">{terminal.promptPath}</span>
+          <span className="text-muted">$ </span>
+          <span>{typedCommand}</span>
+          <span aria-hidden="true" className="terminal-caret" />
+        </p>
+        <div className="mt-4 min-h-[4.75rem] space-y-2">
+          {shouldShowOutput
+            ? activeCommand.output.map((line) => (
+                <p className="terminal-line terminal-output" key={line}>
+                  {line}
+                </p>
+              ))
+            : null}
+        </div>
+      </div>
+    </aside>
+  );
+}


## `feat(home): 클래식 홈 히어로 구성`

diff --git a/src/designs/classic/home-route.tsx b/src/designs/classic/home-route.tsx
new file mode 100644
index 0000000..50bbfe1
--- /dev/null
+++ b/src/designs/classic/home-route.tsx
@@ -0,0 +1,102 @@
+import { ArrowRightIcon } from "@/components/icons";
+import { AnimatedTerminal } from "@/components/portfolio/animated-terminal";
+import { ContentHint } from "@/components/portfolio/content-hint";
+import { ContentLinkView } from "@/components/portfolio/content-link";
+import { ProfilePhoto } from "@/components/portfolio/profile-photo";
+import { Reveal } from "@/components/portfolio/reveal";
+import { PageShell } from "@/components/portfolio/site-shell";
+import type { HomeTemplateId, PortfolioContent } from "@/lib/portfolio";
+
+export function ClassicHomeRoute({
+  content,
+  contentDebug,
+}: {
+  content: PortfolioContent;
+  contentDebug: boolean;
+}) {
+  const activeTemplate: HomeTemplateId = "classic";
+
+  return (
+    <PageShell
+      contentDebug={contentDebug}
+      homeTemplate={activeTemplate}
+      profile={content.profile}
+      site={content.site}
+    >
+      <ClassicHeroSection
+        activeTemplate={activeTemplate}
+        content={content}
+        contentDebug={contentDebug}
+      />
+    </PageShell>
+  );
+}
+
+function ClassicHeroSection({
+  activeTemplate,
+  content,
+  contentDebug,
+}: {
+  activeTemplate: HomeTemplateId;
+  content: PortfolioContent;
+  contentDebug: boolean;
+}) {
+  const { profile } = content;
+  const links = content.links.filter((link) =>
+    ["github", "resume", "website"].includes(link.type),
+  );
+
+  return (
+    <section className="hero-section border-b border-line">
+      <div className="mx-auto grid max-w-6xl gap-10 px-5 py-12 sm:px-8 md:py-16 lg:min-h-[calc(88vh-4rem)] lg:grid-cols-[0.9fr_1.1fr] lg:items-center">
+        <Reveal className="max-w-3xl">
+          <ContentHint
+            enabled={contentDebug}
+            path="src/content/profile.json > name/koreanName/photo/role/headline/summary + src/content/presentation.json > home.classic.hero"
+          />
+          <div className="flex items-center gap-4">
+            {profile.photo ? <ProfilePhoto photo={profile.photo} /> : null}
+            <p className="text-sm font-medium text-muted">
+              {profile.name} · {profile.koreanName}
+            </p>
+          </div>
+          <h1 className="mt-6 text-5xl font-semibold leading-[0.98] tracking-normal text-foreground sm:text-6xl md:text-7xl">
+            {profile.role}
+          </h1>
+          <p className="mt-6 max-w-2xl text-xl leading-8 text-foreground md:text-2xl md:leading-9">
+            {profile.headline}
+          </p>
+          <p className="mt-5 max-w-2xl text-base leading-7 text-muted">
+            {profile.summary}
+          </p>
+          <div className="mt-9 flex flex-wrap gap-3">
+            {links.map((link) => (
+              <ContentLinkView
+                className="inline-flex h-10 items-center gap-2 rounded-md border border-line bg-surface px-4 text-sm font-semibold text-muted transition hover:-translate-y-0.5 hover:border-accent hover:text-foreground focus:outline-none focus:ring-2 focus:ring-accent focus:ring-offset-2 focus:ring-offset-background"
+                contentDebug={contentDebug}
+                homeTemplate={activeTemplate}
+                key={link.id ?? link.href}
+                link={link}
+              >
+                {link.label}
+                <ArrowRightIcon className="-rotate-45" />
+              </ContentLinkView>
+            ))}
+          </div>
+        </Reveal>
+        <div className="hero-terminal-wrap">
+          <ContentHint
+            enabled={contentDebug}
+            path="src/content/presentation.json > home.classic.terminal"
+          />
+          <AnimatedTerminal
+            profile={profile}
+            projectCount={content.projects.length}
+            stackCount={content.techStack.length}
+            terminal={content.presentation.home.classic.terminal}
+          />
+        </div>
+      </div>
+    </section>
+  );
+}


## `style(home): 클래식 홈 테마 적용`

diff --git a/src/app/globals.css b/src/app/globals.css
index ba8c2c3..afe9380 100644
--- a/src/app/globals.css
+++ b/src/app/globals.css
@@ -18,6 +18,24 @@
   --warm: #f59e0b;
 }
 
+main[data-home-template="classic"] {
+  --background: #1f2023;
+  --background-soft: #242528;
+  --foreground: #f2efe7;
+  --muted: #a4abb2;
+  --line: #3a3d43;
+  --surface: #292b2f;
+  --surface-soft: #303338;
+  --surface-hover: #2f3237;
+  --accent: #9cc8b1;
+  --accent-strong: #c0ddcd;
+  --accent-soft: rgba(156, 200, 177, 0.12);
+  --signal: #7aa7ff;
+  --signal-strong: #9db9ff;
+  --signal-soft: rgba(122, 167, 255, 0.16);
+  --warm: #f6c76f;
+}
+
 @theme inline {
   --color-background: var(--background);
   --color-background-soft: var(--background-soft);
@@ -109,6 +127,26 @@ a {
   z-index: 1;
 }
 
+main[data-home-template="classic"] .hero-section::before {
+  background:
+    radial-gradient(circle at 82% 14%, rgba(122, 167, 255, 0.24), transparent 24rem),
+    linear-gradient(135deg, transparent 0 58%, rgba(122, 167, 255, 0.16) 58.2% 58.55%, transparent 58.8%),
+    linear-gradient(155deg, transparent 0 70%, rgba(156, 200, 177, 0.14) 70.1% 70.38%, transparent 70.7%);
+}
+
+main[data-home-template="classic"] .hero-section::after {
+  background-image:
+    linear-gradient(rgba(122, 167, 255, 0.08) 1px, transparent 1px),
+    linear-gradient(90deg, rgba(156, 200, 177, 0.07) 1px, transparent 1px);
+  opacity: 0.8;
+}
+
+main[data-home-template="classic"] .profile-photo-frame {
+  border-color: rgba(156, 200, 177, 0.34);
+  box-shadow: 0 18px 44px rgba(0, 0, 0, 0.28);
+  height: 4.8rem;
+}
+
 .hero-terminal-wrap {
   animation: terminal-float 7s ease-in-out infinite;
   position: relative;


## `feat(home): 디자인 대표 프로젝트 섹션 추가`

diff --git a/src/designs/design/home-route.tsx b/src/designs/design/home-route.tsx
index de98fde..803f6ce 100644
--- a/src/designs/design/home-route.tsx
+++ b/src/designs/design/home-route.tsx
@@ -3,8 +3,10 @@ import { ArrowRightIcon } from "@/components/icons";
 import { ContentHint } from "@/components/portfolio/content-hint";
 import { ContentLinkView } from "@/components/portfolio/content-link";
 import { ProfilePhoto } from "@/components/portfolio/profile-photo";
+import { ProjectCard } from "@/components/portfolio/project-card";
 import { ProjectScreenshot } from "@/components/portfolio/project-screenshot";
 import { Reveal } from "@/components/portfolio/reveal";
+import { SectionHeading } from "@/components/portfolio/section-heading";
 import { PageShell } from "@/components/portfolio/site-shell";
 import {
   getFeaturedProjects,
@@ -42,10 +44,72 @@ export function DesignHomeRoute({
         contentDebug={contentDebug}
         projects={featuredProjects}
       />
+      {content.presentation.home.design.sections.includes("featured") ? (
+        <FeaturedProjectsSection
+          activeTemplate={activeTemplate}
+          content={content}
+          contentDebug={contentDebug}
+          projects={featuredProjects}
+        />
+      ) : null}
     </PageShell>
   );
 }
 
+function FeaturedProjectsSection({
+  activeTemplate,
+  content,
+  contentDebug,
+  projects,
+}: {
+  activeTemplate: HomeTemplateId;
+  content: PortfolioContent;
+  contentDebug: boolean;
+  projects: ReturnType<typeof getFeaturedProjects>;
+}) {
+  const copy = content.presentation.home.design.featured;
+
+  return (
+    <section className="border-b border-line bg-background-soft">
+      <div className="mx-auto grid max-w-6xl gap-6 px-5 py-10 sm:px-8 md:py-12">
+        <div className="flex flex-col gap-5 sm:flex-row sm:items-end sm:justify-between">
+          <SectionHeading
+            body={copy.body}
+            contentDebug={contentDebug}
+            contentHint="src/content/presentation.json > home.design.featured"
+            title={copy.title}
+          />
+        </div>
+        <div className="grid gap-5">
+          {projects.slice(0, 1).map((project) => (
+            <Reveal key={project.id}>
+              <ProjectCard
+                contentDebug={contentDebug}
+                homeTemplate={activeTemplate}
+                priority
+                project={project}
+                variant="featured"
+              />
+            </Reveal>
+          ))}
+          <div className="grid gap-5 lg:grid-cols-2">
+            {projects.slice(1, 3).map((project, index) => (
+              <Reveal delay={index * 80} key={project.id}>
+                <ProjectCard
+                  contentDebug={contentDebug}
+                  homeTemplate={activeTemplate}
+                  priority
+                  project={project}
+                />
+              </Reveal>
+            ))}
+          </div>
+        </div>
+      </div>
+    </section>
+  );
+}
+
 function getWorkMapStats(content: PortfolioContent) {
   const curriculumCount = content.projects.filter((project) =>
     project.screenshot.src.startsWith("/projects/42/") || project.id === "pong-pong",


