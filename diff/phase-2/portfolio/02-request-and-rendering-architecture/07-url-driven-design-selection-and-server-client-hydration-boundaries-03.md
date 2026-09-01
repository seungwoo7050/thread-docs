## `refactor(navigation): 디자인 전환 URL 기본값 명시`

diff --git a/src/components/portfolio/design-switcher.test.tsx b/src/components/portfolio/design-switcher.test.tsx
index ca128f0..1c2f8b2 100644
--- a/src/components/portfolio/design-switcher.test.tsx
+++ b/src/components/portfolio/design-switcher.test.tsx
@@ -33,6 +33,7 @@ describe("DesignSwitcher", () => {
         activeId="editorial"
         contentDebug
         currentPath="/projects"
+        defaultId={content.presentation.defaultHomeTemplate}
         templates={content.presentation.templates}
         ui={content.presentation.ui}
       />
@@ -87,6 +88,7 @@ describe("DesignSwitcher", () => {
         activeId="editorial"
         contentDebug
         currentPath="/projects"
+        defaultId={content.presentation.defaultHomeTemplate}
         templates={content.presentation.templates}
         ui={ui}
       />,
diff --git a/src/components/portfolio/design-switcher.tsx b/src/components/portfolio/design-switcher.tsx
index 492caa9..b85a1e4 100644
--- a/src/components/portfolio/design-switcher.tsx
+++ b/src/components/portfolio/design-switcher.tsx
@@ -1,6 +1,6 @@
 import Link from "next/link";
 import { SITE_DESIGNS } from "@/designs/config";
-import { getTemplateHref } from "@/lib/portfolio";
+import { createTemplateHref } from "@/lib/portfolio/template-href";
 import type {
   PresentationContent,
   PresentationTemplate,
@@ -13,12 +13,14 @@ export function DesignSwitcher({
   activeId,
   contentDebug,
   currentPath,
+  defaultId,
   templates,
   ui,
 }: {
   activeId: SiteDesignId;
   contentDebug?: boolean;
   currentPath: string;
+  defaultId: SiteDesignId;
   templates: PresentationTemplate[];
   ui: PresentationContent["ui"];
 }) {
@@ -59,9 +61,12 @@ export function DesignSwitcher({
                 <Link
                   aria-current={isActive ? "page" : undefined}
                   className={`${styles.link} ${isActive ? styles.active : ""}`}
-                  href={getTemplateHref(currentPath, design.id, {
-                    contentDebug,
-                  })}
+                  href={createTemplateHref(
+                    currentPath,
+                    design.id,
+                    defaultId,
+                    { contentDebug },
+                  )}
                 >
                   <span aria-hidden="true" className={styles.swatch}>
                     {design.swatch.map((color) => (
diff --git a/src/components/portfolio/site-shell.tsx b/src/components/portfolio/site-shell.tsx
index a2b863d..7153d03 100644
--- a/src/components/portfolio/site-shell.tsx
+++ b/src/components/portfolio/site-shell.tsx
@@ -14,6 +14,7 @@ type TemplateSwitcherProps = {
   activeId: HomeTemplateId;
   contentDebug?: boolean;
   currentPath: string;
+  defaultId: HomeTemplateId;
   templates: PresentationTemplate[];
 };
 
@@ -100,6 +101,7 @@ export function SiteHeader({
             activeId={templateSwitcher.activeId}
             contentDebug={templateSwitcher.contentDebug}
             currentPath={templateSwitcher.currentPath}
+            defaultId={templateSwitcher.defaultId}
             templates={templateSwitcher.templates}
             ui={ui}
           />
diff --git a/src/designs/brutalist/brutalist-route.tsx b/src/designs/brutalist/brutalist-route.tsx
index dea950d..01e52ba 100644
--- a/src/designs/brutalist/brutalist-route.tsx
+++ b/src/designs/brutalist/brutalist-route.tsx
@@ -223,6 +223,7 @@ function BrutalistShell({
               activeId={DESIGN_ID}
               contentDebug={contentDebug}
               currentPath={currentPath}
+              defaultId={content.presentation.defaultHomeTemplate}
               templates={content.presentation.templates}
               ui={content.presentation.ui}
             />
diff --git a/src/designs/cinematic/cinematic-route.tsx b/src/designs/cinematic/cinematic-route.tsx
index 1d4451d..865999f 100644
--- a/src/designs/cinematic/cinematic-route.tsx
+++ b/src/designs/cinematic/cinematic-route.tsx
@@ -131,6 +131,7 @@ function Frame({
             activeId="cinematic"
             contentDebug={contentDebug}
             currentPath={currentPath}
+            defaultId={content.presentation.defaultHomeTemplate}
             templates={content.presentation.templates}
             ui={ui}
           />
diff --git a/src/designs/editorial/editorial-route.tsx b/src/designs/editorial/editorial-route.tsx
index 42a5eb5..cf53255 100644
--- a/src/designs/editorial/editorial-route.tsx
+++ b/src/designs/editorial/editorial-route.tsx
@@ -207,6 +207,7 @@ function EditorialShell({
               activeId={DESIGN_ID}
               contentDebug={contentDebug}
               currentPath={currentPath}
+              defaultId={content.presentation.defaultHomeTemplate}
               templates={content.presentation.templates}
               ui={ui}
             />
diff --git a/src/designs/shell-props.ts b/src/designs/shell-props.ts
index ebc7c08..045d229 100644
--- a/src/designs/shell-props.ts
+++ b/src/designs/shell-props.ts
@@ -17,6 +17,7 @@ export function createDesignShellProps(
       activeId: designId,
       contentDebug,
       currentPath,
+      defaultId: content.presentation.defaultHomeTemplate,
       templates: content.presentation.templates,
     },
     ui: content.presentation.ui,
diff --git a/src/lib/portfolio/page-context.ts b/src/lib/portfolio/page-context.ts
index 2a807aa..e0f2668 100644
--- a/src/lib/portfolio/page-context.ts
+++ b/src/lib/portfolio/page-context.ts
@@ -45,6 +45,7 @@ export async function resolvePortfolioPageContext({
         activeId: activeTemplate,
         contentDebug,
         currentPath,
+        defaultId: content.presentation.defaultHomeTemplate,
         templates: content.presentation.templates,
       },
     },


## `refactor(ui): reveal 콘텐츠를 server에서 즉시 표시`

diff --git a/src/components/portfolio/reveal.tsx b/src/components/portfolio/reveal.tsx
index 01c418e..f8b4c9b 100644
--- a/src/components/portfolio/reveal.tsx
+++ b/src/components/portfolio/reveal.tsx
@@ -1,7 +1,3 @@
-"use client";
-
-import { useEffect, useRef, useState, type Ref } from "react";
-
 export function Reveal({
   as = "div",
   children,
@@ -13,43 +9,11 @@ export function Reveal({
   className?: string;
   delay?: number;
 }) {
-  const ref = useRef<HTMLElement>(null);
-  const [visible, setVisible] = useState(
-    () => typeof window !== "undefined" && !("IntersectionObserver" in window),
-  );
-
-  useEffect(() => {
-    const node = ref.current;
-
-    if (!node) {
-      return;
-    }
-
-    if (!("IntersectionObserver" in window)) {
-      return;
-    }
-
-    const observer = new IntersectionObserver(
-      ([entry]) => {
-        if (entry.isIntersecting) {
-          setVisible(true);
-          observer.disconnect();
-        }
-      },
-      { rootMargin: "0px 0px -12% 0px", threshold: 0.12 },
-    );
-
-    observer.observe(node);
-
-    return () => observer.disconnect();
-  }, []);
-
   const Component = as;
 
   return (
     <Component
-      className={`reveal-item ${visible ? "is-visible" : ""} ${className}`}
-      ref={ref as Ref<HTMLDivElement & HTMLLIElement>}
+      className={`reveal-item is-visible ${className}`}
       style={{ transitionDelay: `${delay}ms` }}
     >
       {children}


