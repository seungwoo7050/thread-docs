## `refactor(routes): 소개와 학습 route context 통합`

diff --git a/src/app/about/page.tsx b/src/app/about/page.tsx
index 6531b82..4b8594b 100644
--- a/src/app/about/page.tsx
+++ b/src/app/about/page.tsx
@@ -12,13 +12,12 @@ import {
   getPortfolioContent,
   getTemplateHref,
   isSitePageEnabled,
-  resolveContentDebug,
-  resolveHomeTemplateId,
   type CurationCategory,
   type HomeTemplateId,
   type PortfolioContent,
   type RouteSearchParams,
 } from "@/lib/portfolio";
+import { resolvePortfolioPageContext } from "@/lib/portfolio/page-context";
 
 export default async function AboutPage({
   searchParams,
@@ -27,9 +26,12 @@ export default async function AboutPage({
 }) {
   const content = getPortfolioContent();
   if (!isSitePageEnabled("about", content)) notFound();
-  const params = searchParams ? await searchParams : {};
-  const activeTemplate = resolveHomeTemplateId(params.view, content.presentation);
-  const contentDebug = resolveContentDebug(params.debug);
+  const { activeTemplate, contentDebug, shellProps } =
+    await resolvePortfolioPageContext({
+      content,
+      currentPath: "/about",
+      searchParams,
+    });
 
   if (hasDedicatedRouteRenderer(activeTemplate)) {
     return renderDesignRoute(activeTemplate, {
@@ -43,19 +45,7 @@ export default async function AboutPage({
   const pageCopy = content.presentation.pages.about;
 
   return (
-    <PageShell
-      contentDebug={contentDebug}
-      homeTemplate={activeTemplate}
-      profile={content.profile}
-      site={content.site}
-      ui={content.presentation.ui}
-      templateSwitcher={{
-        activeId: activeTemplate,
-        contentDebug,
-        currentPath: "/about",
-        templates: content.presentation.templates,
-      }}
-    >
+    <PageShell {...shellProps}>
       <section className="border-b border-line">
         <div className="mx-auto grid max-w-6xl gap-10 px-5 py-20 sm:px-8 lg:grid-cols-[1fr_20rem] lg:items-center">
           <div>
diff --git a/src/app/interview-map/page.tsx b/src/app/interview-map/page.tsx
index f9d1c5a..4b855cf 100644
--- a/src/app/interview-map/page.tsx
+++ b/src/app/interview-map/page.tsx
@@ -9,13 +9,12 @@ import {
   getPortfolioContent,
   getTemplateHref,
   isSitePageEnabled,
-  resolveContentDebug,
-  resolveHomeTemplateId,
   type HomeTemplateId,
   type InterviewMapTrack,
   type PortfolioContent,
   type RouteSearchParams,
 } from "@/lib/portfolio";
+import { resolvePortfolioPageContext } from "@/lib/portfolio/page-context";
 
 export default async function InterviewMapPage({
   searchParams,
@@ -24,9 +23,12 @@ export default async function InterviewMapPage({
 }) {
   const content = getPortfolioContent();
   if (!isSitePageEnabled("interviewMap", content)) notFound();
-  const params = searchParams ? await searchParams : {};
-  const activeTemplate = resolveHomeTemplateId(params.view, content.presentation);
-  const contentDebug = resolveContentDebug(params.debug);
+  const { activeTemplate, contentDebug, shellProps } =
+    await resolvePortfolioPageContext({
+      content,
+      currentPath: "/interview-map",
+      searchParams,
+    });
 
   if (hasDedicatedRouteRenderer(activeTemplate)) {
     return renderDesignRoute(activeTemplate, {
@@ -41,19 +43,7 @@ export default async function InterviewMapPage({
   const data = content.interviewMap;
 
   return (
-    <PageShell
-      contentDebug={contentDebug}
-      homeTemplate={activeTemplate}
-      profile={content.profile}
-      site={content.site}
-      ui={content.presentation.ui}
-      templateSwitcher={{
-        activeId: activeTemplate,
-        contentDebug,
-        currentPath: "/interview-map",
-        templates: content.presentation.templates,
-      }}
-    >
+    <PageShell {...shellProps}>
       <section className="border-b border-line">
         <div className="mx-auto max-w-6xl px-5 py-20 sm:px-8">
           <ContentHint
diff --git a/src/app/journey/page.tsx b/src/app/journey/page.tsx
index 783f0e7..edece0c 100644
--- a/src/app/journey/page.tsx
+++ b/src/app/journey/page.tsx
@@ -10,14 +10,13 @@ import {
   getPortfolioContent,
   getTemplateHref,
   isSitePageEnabled,
-  resolveContentDebug,
-  resolveHomeTemplateId,
   type HomeTemplateId,
   type JourneyMilestone,
   type PortfolioContent,
   type PresentationContent,
   type RouteSearchParams,
 } from "@/lib/portfolio";
+import { resolvePortfolioPageContext } from "@/lib/portfolio/page-context";
 
 export default async function JourneyPage({
   searchParams,
@@ -26,9 +25,12 @@ export default async function JourneyPage({
 }) {
   const content = getPortfolioContent();
   if (!isSitePageEnabled("journey", content)) notFound();
-  const params = searchParams ? await searchParams : {};
-  const activeTemplate = resolveHomeTemplateId(params.view, content.presentation);
-  const contentDebug = resolveContentDebug(params.debug);
+  const { activeTemplate, contentDebug, shellProps } =
+    await resolvePortfolioPageContext({
+      content,
+      currentPath: "/journey",
+      searchParams,
+    });
 
   if (hasDedicatedRouteRenderer(activeTemplate)) {
     return renderDesignRoute(activeTemplate, {
@@ -43,19 +45,7 @@ export default async function JourneyPage({
   const narrative = content.journeyNarrative;
 
   return (
-    <PageShell
-      contentDebug={contentDebug}
-      homeTemplate={activeTemplate}
-      profile={content.profile}
-      site={content.site}
-      ui={content.presentation.ui}
-      templateSwitcher={{
-        activeId: activeTemplate,
-        contentDebug,
-        currentPath: "/journey",
-        templates: content.presentation.templates,
-      }}
-    >
+    <PageShell {...shellProps}>
       <section className="border-b border-line">
         <div className="mx-auto max-w-6xl px-5 py-20 sm:px-8">
           <ContentHint


## `refactor(routes): 이력과 연락 context 통합`

diff --git a/src/app/contact/page.tsx b/src/app/contact/page.tsx
index 3e2d286..3fe4864 100644
--- a/src/app/contact/page.tsx
+++ b/src/app/contact/page.tsx
@@ -8,10 +8,9 @@ import {
   getPortfolioContent,
   isSitePageEnabled,
   getPreferredContactLinks,
-  resolveContentDebug,
-  resolveHomeTemplateId,
   type RouteSearchParams,
 } from "@/lib/portfolio";
+import { resolvePortfolioPageContext } from "@/lib/portfolio/page-context";
 
 export default async function ContactPage({
   searchParams,
@@ -20,9 +19,12 @@ export default async function ContactPage({
 }) {
   const content = getPortfolioContent();
   if (!isSitePageEnabled("contact", content)) notFound();
-  const params = searchParams ? await searchParams : {};
-  const activeTemplate = resolveHomeTemplateId(params.view, content.presentation);
-  const contentDebug = resolveContentDebug(params.debug);
+  const { activeTemplate, contentDebug, shellProps } =
+    await resolvePortfolioPageContext({
+      content,
+      currentPath: "/contact",
+      searchParams,
+    });
 
   if (hasDedicatedRouteRenderer(activeTemplate)) {
     return renderDesignRoute(activeTemplate, {
@@ -37,19 +39,7 @@ export default async function ContactPage({
   const preferredLinks = getPreferredContactLinks(content);
 
   return (
-    <PageShell
-      contentDebug={contentDebug}
-      homeTemplate={activeTemplate}
-      profile={content.profile}
-      site={content.site}
-      ui={content.presentation.ui}
-      templateSwitcher={{
-        activeId: activeTemplate,
-        contentDebug,
-        currentPath: "/contact",
-        templates: content.presentation.templates,
-      }}
-    >
+    <PageShell {...shellProps}>
       <section className="border-b border-line">
         <div className="mx-auto max-w-6xl px-5 py-20 sm:px-8">
           <ContentHint
diff --git a/src/app/resume/page.tsx b/src/app/resume/page.tsx
index 8eeb414..6af8fc5 100644
--- a/src/app/resume/page.tsx
+++ b/src/app/resume/page.tsx
@@ -10,10 +10,9 @@ import {
   getResumeProjects,
   getTemplateHref,
   isSitePageEnabled,
-  resolveContentDebug,
-  resolveHomeTemplateId,
   type RouteSearchParams,
 } from "@/lib/portfolio";
+import { resolvePortfolioPageContext } from "@/lib/portfolio/page-context";
 
 export default async function ResumePage({
   searchParams,
@@ -22,9 +21,12 @@ export default async function ResumePage({
 }) {
   const content = getPortfolioContent();
   if (!isSitePageEnabled("resume", content)) notFound();
-  const params = searchParams ? await searchParams : {};
-  const activeTemplate = resolveHomeTemplateId(params.view, content.presentation);
-  const contentDebug = resolveContentDebug(params.debug);
+  const { activeTemplate, contentDebug, shellProps } =
+    await resolvePortfolioPageContext({
+      content,
+      currentPath: "/resume",
+      searchParams,
+    });
 
   if (hasDedicatedRouteRenderer(activeTemplate)) {
     return renderDesignRoute(activeTemplate, {
@@ -39,19 +41,7 @@ export default async function ResumePage({
   const resumeProjects = getResumeProjects(content);
 
   return (
-    <PageShell
-      contentDebug={contentDebug}
-      homeTemplate={activeTemplate}
-      profile={content.profile}
-      site={content.site}
-      ui={content.presentation.ui}
-      templateSwitcher={{
-        activeId: activeTemplate,
-        contentDebug,
-        currentPath: "/resume",
-        templates: content.presentation.templates,
-      }}
-    >
+    <PageShell {...shellProps}>
       <section className="border-b border-line">
         <div className="mx-auto grid max-w-6xl gap-8 px-5 py-20 sm:px-8 lg:grid-cols-[1fr_auto] lg:items-end">
           <div>


## `fix(ui): hydration 중 native details 상태 보존`

diff --git a/src/components/portfolio/design-switcher.tsx b/src/components/portfolio/design-switcher.tsx
index 46e522a..d7807b8 100644
--- a/src/components/portfolio/design-switcher.tsx
+++ b/src/components/portfolio/design-switcher.tsx
@@ -38,7 +38,7 @@ export function DesignSwitcher({
     .replace("{total}", String(SITE_DESIGNS.length).padStart(2, "0"));
 
   return (
-    <details className={styles.root} ref={detailsRef}>
+    <details className={styles.root} ref={detailsRef} suppressHydrationWarning>
       <summary
         aria-label={ui.designSwitcherAriaTemplate.replace(
           "{label}",


## `test(ui): details hydration 경쟁 조건 검증`

diff --git a/src/components/portfolio/design-switcher.test.tsx b/src/components/portfolio/design-switcher.test.tsx
index 9145496..ccd57ea 100644
--- a/src/components/portfolio/design-switcher.test.tsx
+++ b/src/components/portfolio/design-switcher.test.tsx
@@ -1,5 +1,8 @@
 import { cleanup, fireEvent, render, screen, within } from "@testing-library/react";
-import { afterEach, describe, expect, it } from "vitest";
+import { act } from "react";
+import { hydrateRoot } from "react-dom/client";
+import { renderToString } from "react-dom/server";
+import { afterEach, describe, expect, it, vi } from "vitest";
 
 import { getPortfolioContent } from "@/lib/portfolio";
 
@@ -8,6 +11,53 @@ import { DesignSwitcher } from "./design-switcher";
 afterEach(() => cleanup());
 
 describe("DesignSwitcher", () => {
+  it("tolerates native open state changed before hydration", async () => {
+    const content = getPortfolioContent();
+    const switcher = (
+      <DesignSwitcher
+        activeId="editorial"
+        contentDebug
+        currentPath="/projects"
+        templates={content.presentation.templates}
+        ui={content.presentation.ui}
+      />
+    );
+    const container = document.createElement("div");
+    container.innerHTML = renderToString(switcher);
+    document.body.append(container);
+
+    const details = container.querySelector("details");
+    if (!details) {
+      throw new Error("DesignSwitcher must render a details element.");
+    }
+
+    details.open = true;
+    const consoleError = vi.spyOn(console, "error").mockImplementation(() => {});
+    let root: ReturnType<typeof hydrateRoot> | undefined;
+
+    try {
+      await act(async () => {
+        root = hydrateRoot(container, switcher);
+        await Promise.resolve();
+      });
+
+      const hydrationErrors = consoleError.mock.calls
+        .flatMap((call) => call.map(String))
+        .filter((message) =>
+          /hydration|server rendered HTML|did not match/i.test(message),
+        );
+
+      expect(details).toHaveAttribute("open");
+      expect(hydrationErrors).toEqual([]);
+    } finally {
+      if (root) {
+        await act(async () => root?.unmount());
+      }
+      consoleError.mockRestore();
+      container.remove();
+    }
+  });
+
   it("renders selector copy from content and clears native open state", () => {
     const content = getPortfolioContent();
     const ui = {


## `refactor(ui): 디자인 선택기를 server markup으로 전환`

diff --git a/src/components/portfolio/design-switcher-close.tsx b/src/components/portfolio/design-switcher-close.tsx
new file mode 100644
index 0000000..1b0670f
--- /dev/null
+++ b/src/components/portfolio/design-switcher-close.tsx
@@ -0,0 +1,19 @@
+"use client";
+
+export function DesignSwitcherClose({ label }: { label: string }) {
+  return (
+    <button
+      aria-label={label}
+      onClick={(event) => {
+        const details = event.currentTarget.closest("details");
+        const summary = details?.querySelector<HTMLElement>(":scope > summary");
+
+        details?.removeAttribute("open");
+        summary?.focus();
+      }}
+      type="button"
+    >
+      <span aria-hidden="true">×</span>
+    </button>
+  );
+}
diff --git a/src/components/portfolio/design-switcher.test.tsx b/src/components/portfolio/design-switcher.test.tsx
index ccd57ea..e37783f 100644
--- a/src/components/portfolio/design-switcher.test.tsx
+++ b/src/components/portfolio/design-switcher.test.tsx
@@ -58,7 +58,7 @@ describe("DesignSwitcher", () => {
     }
   });
 
-  it("renders selector copy from content and clears native open state", () => {
+  it("renders selector copy and restores focus when explicitly closed", () => {
     const content = getPortfolioContent();
     const ui = {
       ...content.presentation.ui,
@@ -95,17 +95,13 @@ describe("DesignSwitcher", () => {
     });
 
     expect(details).not.toBeNull();
+    expect(classicLink).toHaveAttribute(
+      "href",
+      "/projects?view=classic&debug=content",
+    );
     details?.setAttribute("open", "");
     fireEvent.click(closeButton);
     expect(details).not.toHaveAttribute("open");
     expect(summary).toHaveFocus();
-
-    details?.setAttribute("open", "");
-    document.addEventListener("click", (event) => event.preventDefault(), {
-      capture: true,
-      once: true,
-    });
-    fireEvent.click(classicLink);
-    expect(details).not.toHaveAttribute("open");
   });
 });
diff --git a/src/components/portfolio/design-switcher.tsx b/src/components/portfolio/design-switcher.tsx
index d7807b8..492caa9 100644
--- a/src/components/portfolio/design-switcher.tsx
+++ b/src/components/portfolio/design-switcher.tsx
@@ -1,14 +1,12 @@
-"use client";
-
 import Link from "next/link";
-import { useRef } from "react";
 import { SITE_DESIGNS } from "@/designs/config";
-import {
-  getTemplateHref,
-  type PresentationContent,
-  type PresentationTemplate,
-  type SiteDesignId,
-} from "@/lib/portfolio";
+import { getTemplateHref } from "@/lib/portfolio";
+import type {
+  PresentationContent,
+  PresentationTemplate,
+  SiteDesignId,
+} from "@/lib/portfolio/types";
+import { DesignSwitcherClose } from "./design-switcher-close";
 import styles from "./design-switcher.module.css";
 
 export function DesignSwitcher({
@@ -24,8 +22,6 @@ export function DesignSwitcher({
   templates: PresentationTemplate[];
   ui: PresentationContent["ui"];
 }) {
-  const detailsRef = useRef<HTMLDetailsElement>(null);
-  const summaryRef = useRef<HTMLElement>(null);
   const templateCopy = new Map(
     templates.map((template) => [template.id, template]),
   );
@@ -38,13 +34,12 @@ export function DesignSwitcher({
     .replace("{total}", String(SITE_DESIGNS.length).padStart(2, "0"));
 
   return (
-    <details className={styles.root} ref={detailsRef} suppressHydrationWarning>
+    <details className={styles.root} suppressHydrationWarning>
       <summary
         aria-label={ui.designSwitcherAriaTemplate.replace(
           "{label}",
           activeLabel,
         )}
-        ref={summaryRef}
       >
         <span className={styles.count}>{countLabel}</span>
         <span className={styles.label}>{activeLabel}</span>
@@ -52,16 +47,7 @@ export function DesignSwitcher({
       <nav aria-label={ui.designNavigationAriaLabel} className={styles.panel}>
         <div className={styles.sheetHeader}>
           <strong>{ui.designNavigationAriaLabel}</strong>
-          <button
-            aria-label={ui.designSwitcherCloseLabel}
-            onClick={() => {
-              detailsRef.current?.removeAttribute("open");
-              summaryRef.current?.focus();
-            }}
-            type="button"
-          >
-            <span aria-hidden="true">×</span>
-          </button>
+          <DesignSwitcherClose label={ui.designSwitcherCloseLabel} />
         </div>
         <ul className={styles.list}>
           {SITE_DESIGNS.map((design, index) => {
@@ -76,7 +62,6 @@ export function DesignSwitcher({
                   href={getTemplateHref(currentPath, design.id, {
                     contentDebug,
                   })}
-                  onClick={() => detailsRef.current?.removeAttribute("open")}
                 >
                   <span aria-hidden="true" className={styles.swatch}>
                     {design.swatch.map((color) => (


## `test(ui): server 선택기와 focus 복원 검증`

diff --git a/package-lock.json b/package-lock.json
index 95624f5..c83751e 100644
--- a/package-lock.json
+++ b/package-lock.json
@@ -7672,15 +7672,15 @@
       }
     },
     "node_modules/side-channel": {
-      "version": "1.1.0",
-      "resolved": "https://registry.npmjs.org/side-channel/-/side-channel-1.1.0.tgz",
-      "integrity": "sha512-ZX99e6tRweoUXqR+VBrslhda51Nh5MTQwou5tnUDgbtyM0dBgmhEDtWGP/xbKn6hqfPRHujUNwz5fy/wbbhnpw==",
+      "version": "1.1.1",
+      "resolved": "https://registry.npmjs.org/side-channel/-/side-channel-1.1.1.tgz",
+      "integrity": "sha512-6x6dK6zJdpTzF4sQeNYxwtvBzf6Eg4GtlesS94HOvTudUeyK2WXAaIfmDgsyslYrRBeFIlsi54AYsFGUuhmvrQ==",
       "dev": true,
       "license": "MIT",
       "dependencies": {
         "es-errors": "^1.3.0",
-        "object-inspect": "^1.13.3",
-        "side-channel-list": "^1.0.0",
+        "object-inspect": "^1.13.4",
+        "side-channel-list": "^1.0.1",
         "side-channel-map": "^1.0.1",
         "side-channel-weakmap": "^1.0.2"
       },
diff --git a/src/components/portfolio/design-switcher.test.tsx b/src/components/portfolio/design-switcher.test.tsx
index e37783f..ca128f0 100644
--- a/src/components/portfolio/design-switcher.test.tsx
+++ b/src/components/portfolio/design-switcher.test.tsx
@@ -1,4 +1,6 @@
 import { cleanup, fireEvent, render, screen, within } from "@testing-library/react";
+import { readFile } from "node:fs/promises";
+import path from "node:path";
 import { act } from "react";
 import { hydrateRoot } from "react-dom/client";
 import { renderToString } from "react-dom/server";
@@ -11,6 +13,19 @@ import { DesignSwitcher } from "./design-switcher";
 afterEach(() => cleanup());
 
 describe("DesignSwitcher", () => {
+  it("keeps the selector markup in a server component", async () => {
+    const source = await readFile(
+      path.join(
+        process.cwd(),
+        "src/components/portfolio/design-switcher.tsx",
+      ),
+      "utf8",
+    );
+
+    expect(source).not.toContain('"use client"');
+    expect(source).not.toContain("useRef");
+  });
+
   it("tolerates native open state changed before hydration", async () => {
     const content = getPortfolioContent();
     const switcher = (


