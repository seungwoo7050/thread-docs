## `test(design): 독립 renderer와 design token 경계 검증`

diff --git a/src/designs/design-tokens.test.ts b/src/designs/design-tokens.test.ts
new file mode 100644
index 0000000..47938af
--- /dev/null
+++ b/src/designs/design-tokens.test.ts
@@ -0,0 +1,30 @@
+import { readFileSync } from "node:fs";
+import { resolve } from "node:path";
+import { describe, expect, it } from "vitest";
+
+const stylesheet = readFileSync(resolve(process.cwd(), "src/app/globals.css"), "utf8");
+
+const tokenFamilies = [
+  "--type-display",
+  "--type-body",
+  "--space-section",
+  "--breakpoint-content",
+  "--motion-fast",
+  "--layer-navigation",
+  "--content-width",
+] as const;
+
+describe("design tokens", () => {
+  it.each(tokenFamilies)("defines the %s token", (token) => {
+    expect(stylesheet).toContain(`${token}:`);
+  });
+
+  it.each(["design", "classic"])(
+    "keeps the %s renderer token scope explicit",
+    (designId) => {
+      expect(stylesheet).toMatch(
+        new RegExp(`\\[data-site-design=["']${designId}["']\\][\\s\\S]*?--content-width:`),
+      );
+    },
+  );
+});
diff --git a/src/designs/route-view-models.test.tsx b/src/designs/route-view-models.test.tsx
index dca24a8..de66283 100644
--- a/src/designs/route-view-models.test.tsx
+++ b/src/designs/route-view-models.test.tsx
@@ -4,6 +4,8 @@ import { afterEach, describe, expect, it } from "vitest";
 import AboutPage from "@/app/about/page";
 import ContactPage from "@/app/contact/page";
 import Home from "@/app/page";
+import InterviewMapPage from "@/app/interview-map/page";
+import JourneyPage from "@/app/journey/page";
 import ProjectDetailPage from "@/app/projects/[projectId]/page";
 import ProjectsPage from "@/app/projects/page";
 import ResumePage from "@/app/resume/page";
@@ -58,6 +60,16 @@ const routes = [
     renderPage: (view: SiteDesignId) =>
       ContactPage({ searchParams: Promise.resolve({ view }) }),
   },
+  {
+    id: "journey",
+    renderPage: (view: SiteDesignId) =>
+      JourneyPage({ searchParams: Promise.resolve({ view }) }),
+  },
+  {
+    id: "interview-map",
+    renderPage: (view: SiteDesignId) =>
+      InterviewMapPage({ searchParams: Promise.resolve({ view }) }),
+  },
 ];
 
 afterEach(() => cleanup());
@@ -72,6 +84,11 @@ describe("route view model rendering", () => {
         expect(
           container.querySelector(`[data-site-design="${designId}"]`),
         ).toBeInTheDocument();
+        if (designId === "design" || designId === "classic") {
+          expect(
+            container.querySelector(`[data-route-renderer="${designId}"]`),
+          ).toBeInTheDocument();
+        }
         expect(container.querySelector("h1")).toHaveTextContent(/\S/);
       },
     );
