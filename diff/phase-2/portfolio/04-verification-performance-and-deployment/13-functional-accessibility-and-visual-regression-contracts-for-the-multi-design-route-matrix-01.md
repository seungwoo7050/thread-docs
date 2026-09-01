# 다중 디자인 Route Matrix의 기능·접근성·시각 회귀 계약

## `test(routes): 홈과 route presentation 계약 검증`

diff --git a/src/app/journey/page.test.tsx b/src/app/journey/page.test.tsx
new file mode 100644
index 0000000..fba5077
--- /dev/null
+++ b/src/app/journey/page.test.tsx
@@ -0,0 +1,40 @@
+import { cleanup, render, screen } from "@testing-library/react";
+import { afterEach, describe, expect, it } from "vitest";
+
+import { getPortfolioContent } from "@/lib/portfolio";
+
+import JourneyPage from "./page";
+
+afterEach(() => cleanup());
+
+describe("JourneyPage", () => {
+  it("uses content-owned shell and milestone labels", async () => {
+    const content = getPortfolioContent();
+    const ui = content.presentation.ui;
+    const labels = content.presentation.pages.journey.narrative.labels;
+
+    render(
+      await JourneyPage({
+        searchParams: Promise.resolve({ view: "design" }),
+      }),
+    );
+
+    expect(screen.getByText(ui.skipLinkLabel, { exact: true })).toBeInTheDocument();
+    expect(
+      screen.getByRole("navigation", {
+        name: ui.primaryNavigationAriaLabel,
+      }),
+    ).toBeInTheDocument();
+    expect(
+      screen.getByRole("navigation", {
+        hidden: true,
+        name: ui.mobileNavigationAriaLabel,
+      }),
+    ).toBeInTheDocument();
+    expect(screen.getByText(ui.menuLabel, { exact: true })).toBeInTheDocument();
+
+    for (const label of [labels.state, labels.reason, labels.result]) {
+      expect(screen.getAllByText(label, { exact: true }).length).toBeGreaterThan(0);
+    }
+  });
+});
diff --git a/src/app/page.test.tsx b/src/app/page.test.tsx
new file mode 100644
index 0000000..ee9df21
--- /dev/null
+++ b/src/app/page.test.tsx
@@ -0,0 +1,170 @@
+import { cleanup, render, screen, within } from "@testing-library/react";
+import { afterEach, describe, expect, it } from "vitest";
+
+import { getFeaturedProjects, getPortfolioContent } from "@/lib/portfolio";
+
+import Home from "./page";
+
+const content = getPortfolioContent();
+const designIds = [
+  "design",
+  "classic",
+  "editorial",
+  "brutalist",
+  "cinematic",
+] as const;
+
+afterEach(() => cleanup());
+
+describe("Home", () => {
+  it("renders the same journey evidence in the two original presentations", async () => {
+    const journeyTitles = content.journey.map((item) => item.title);
+
+    render(
+      await Home({
+        searchParams: Promise.resolve({ view: "design" }),
+      }),
+    );
+
+    for (const title of journeyTitles) {
+      expect(screen.getAllByText(title, { exact: true }).length).toBeGreaterThan(
+        0,
+      );
+    }
+
+    cleanup();
+    render(
+      await Home({
+        searchParams: Promise.resolve({ view: "classic" }),
+      }),
+    );
+
+    for (const title of journeyTitles) {
+      expect(screen.getAllByText(title, { exact: true }).length).toBeGreaterThan(
+        0,
+      );
+    }
+  });
+
+  it("uses editorial when no design is requested", async () => {
+    const { container } = render(await Home({}));
+
+    expect(
+      container.querySelector('[data-site-design="editorial"]'),
+    ).toBeInTheDocument();
+    expect(
+      screen.getByLabelText(
+        "Change site design. Current design: Editorial",
+      ),
+    ).toHaveTextContent("Design 03/05");
+  });
+
+  it.each(designIds)(
+    "renders shared content through the %s full-site design",
+    async (designId) => {
+      const { container } = render(
+        await Home({
+          searchParams: Promise.resolve({ view: designId }),
+        }),
+      );
+      const design = content.presentation.templates.find(
+        (template) => template.id === designId,
+      );
+      const root = container.querySelector(
+        `[data-site-design="${designId}"]`,
+      );
+      const featuredProject = getFeaturedProjects(content)[0] ?? content.projects[0];
+      const projectsNavItem = content.site.navigation.find(
+        (item) => item.href === "/projects",
+      );
+      const expectedProjectsHref =
+        designId === "editorial" ? "/projects" : `/projects?view=${designId}`;
+
+      expect(design).toBeDefined();
+      expect(root).toBeInTheDocument();
+      expect(
+        screen.getAllByRole("heading", { level: 1 })[0],
+      ).toHaveTextContent(/\S/);
+      expect(
+        screen.getByLabelText(
+          `Change site design. Current design: ${design?.label}`,
+        ),
+      ).toBeInTheDocument();
+
+      if (featuredProject) {
+        expect(
+          screen.getAllByText(featuredProject.title, { exact: true }).length,
+        ).toBeGreaterThan(0);
+      }
+
+      expect(projectsNavItem).toBeDefined();
+      const projectLinks = Array.from(root?.querySelectorAll("a") ?? []);
+      expect(
+        projectLinks.some(
+          (link) => link.getAttribute("href") === expectedProjectsHref,
+        ),
+      ).toBe(true);
+
+      const designNavigation = screen.getByRole("navigation", {
+        hidden: true,
+        name: "Site design",
+      });
+      expect(
+        within(designNavigation).getAllByRole("link", { hidden: true }),
+      ).toHaveLength(designIds.length);
+    },
+  );
+
+  it("falls back to editorial for an unknown design", async () => {
+    const { container } = render(
+      await Home({
+        searchParams: Promise.resolve({ view: "not-a-design" }),
+      }),
+    );
+
+    expect(
+      container.querySelector('[data-site-design="editorial"]'),
+    ).toBeInTheDocument();
+  });
+
+  it("preserves content debug state across navigation and design changes", async () => {
+    render(
+      await Home({
+        searchParams: Promise.resolve({
+          debug: "content",
+          view: "brutalist",
+        }),
+      }),
+    );
+
+    const designNavigation = screen.getByRole("navigation", {
+      hidden: true,
+      name: "Site design",
+    });
+    const editorialLink = within(designNavigation).getByRole("link", {
+      hidden: true,
+      name: /Editorial/,
+    });
+    const cinematicLink = within(designNavigation).getByRole("link", {
+      hidden: true,
+      name: /Cinematic/,
+    });
+    const brutalistRoot = document.querySelector(
+      '[data-site-design="brutalist"]',
+    );
+
+    expect(editorialLink).toHaveAttribute("href", "/?debug=content");
+    expect(cinematicLink).toHaveAttribute(
+      "href",
+      "/?view=cinematic&debug=content",
+    );
+    expect(
+      Array.from(brutalistRoot?.querySelectorAll("a") ?? [])
+        .some(
+          (link) =>
+            link.getAttribute("href") ===
+            "/projects?view=brutalist&debug=content",
+        ),
+    ).toBe(true);
+  });
+});
diff --git a/src/app/routes.test.tsx b/src/app/routes.test.tsx
new file mode 100644
index 0000000..e8ff5b2
--- /dev/null
+++ b/src/app/routes.test.tsx
@@ -0,0 +1,152 @@
+import { cleanup, render, screen, within } from "@testing-library/react";
+import { afterEach, describe, expect, it } from "vitest";
+
+import { getPortfolioContent } from "@/lib/portfolio";
+
+import AboutPage from "./about/page";
+import ContactPage from "./contact/page";
+import InterviewMapPage from "./interview-map/page";
+import JourneyPage from "./journey/page";
+import Home from "./page";
+import ProjectDetailPage from "./projects/[projectId]/page";
+import ProjectsPage from "./projects/page";
+import ResumePage from "./resume/page";
+
+afterEach(() => cleanup());
+
+const content = getPortfolioContent();
+const firstProject = content.projects[0];
+
+if (!firstProject) {
+  throw new Error("Route characterization requires at least one enabled project.");
+}
+
+const routes = [
+  {
+    currentPath: "/",
+    heading: content.profile.role,
+    renderPage: () =>
+      Home({
+        searchParams: Promise.resolve({ debug: "content", view: "classic" }),
+      }),
+  },
+  {
+    currentPath: "/about",
+    heading: "About",
+    renderPage: () =>
+      AboutPage({
+        searchParams: Promise.resolve({ debug: "content", view: "classic" }),
+      }),
+  },
+  {
+    currentPath: "/contact",
+    heading: "Contact",
+    renderPage: () =>
+      ContactPage({
+        searchParams: Promise.resolve({ debug: "content", view: "classic" }),
+      }),
+  },
+  {
+    currentPath: "/interview-map",
+    heading: "Interview Map",
+    renderPage: () =>
+      InterviewMapPage({
+        searchParams: Promise.resolve({ debug: "content", view: "classic" }),
+      }),
+  },
+  {
+    currentPath: "/journey",
+    heading: "Journey",
+    renderPage: () =>
+      JourneyPage({
+        searchParams: Promise.resolve({ debug: "content", view: "classic" }),
+      }),
+  },
+  {
+    currentPath: "/projects",
+    heading: "Project archive",
+    renderPage: () =>
+      ProjectsPage({
+        searchParams: Promise.resolve({ debug: "content", view: "classic" }),
+      }),
+  },
+  {
+    currentPath: `/projects/${firstProject.id}`,
+    heading: firstProject.title,
+    renderPage: () =>
+      ProjectDetailPage({
+        params: Promise.resolve({ projectId: firstProject.id }),
+        searchParams: Promise.resolve({ debug: "content", view: "classic" }),
+      }),
+  },
+  {
+    currentPath: "/resume",
+    heading: "Resume",
+    renderPage: () =>
+      ResumePage({
+        searchParams: Promise.resolve({ debug: "content", view: "classic" }),
+      }),
+  },
+];
+
+describe("portfolio routes", () => {
+  it.each(routes)(
+    "preserves the classic shell contract for $currentPath",
+    async ({ currentPath, heading, renderPage }) => {
+      const { container } = render(await renderPage());
+
+      expect(
+        screen.getByRole("heading", { level: 1, name: heading }),
+      ).toBeInTheDocument();
+      expect(container.querySelector("main")).toHaveAttribute(
+        "data-home-template",
+        "classic",
+      );
+      expect(
+        screen.getAllByLabelText(/^Content source:/).length,
+      ).toBeGreaterThan(0);
+      expect(
+        screen.getAllByRole("link", { name: "Projects" })[0],
+      ).toHaveAttribute("href", "/projects?view=classic&debug=content");
+
+      const designNavigation = screen.getByRole("navigation", {
+        hidden: true,
+        name: "Site design",
+      });
+      expect(
+        within(designNavigation).getByRole("link", {
+          hidden: true,
+          name: /Classic/,
+        }),
+      ).toHaveAttribute("aria-current", "page");
+      expect(
+        within(designNavigation).getByRole("link", {
+          hidden: true,
+          name: /Design/,
+        }),
+      ).toHaveAttribute(
+        "href",
+        `${currentPath}?view=design&debug=content`,
+      );
+    },
+  );
+
+  it("uses the first value from repeated view and debug queries", async () => {
+    const { container } = render(
+      await AboutPage({
+        searchParams: Promise.resolve({
+          debug: ["content", "off"],
+          view: ["classic", "editorial"],
+        }),
+      }),
+    );
+
+    expect(container.querySelector("main")).toHaveAttribute(
+      "data-home-template",
+      "classic",
+    );
+    expect(screen.getAllByLabelText(/^Content source:/).length).toBeGreaterThan(
+      0,
+    );
+  });
+});


## `test(ui): 디자인 선택과 프로젝트 링크 계약 검증`

diff --git a/src/components/portfolio/design-switcher.test.tsx b/src/components/portfolio/design-switcher.test.tsx
new file mode 100644
index 0000000..9145496
--- /dev/null
+++ b/src/components/portfolio/design-switcher.test.tsx
@@ -0,0 +1,61 @@
+import { cleanup, fireEvent, render, screen, within } from "@testing-library/react";
+import { afterEach, describe, expect, it } from "vitest";
+
+import { getPortfolioContent } from "@/lib/portfolio";
+
+import { DesignSwitcher } from "./design-switcher";
+
+afterEach(() => cleanup());
+
+describe("DesignSwitcher", () => {
+  it("renders selector copy from content and clears native open state", () => {
+    const content = getPortfolioContent();
+    const ui = {
+      ...content.presentation.ui,
+      designSwitcherAriaTemplate: "Choose presentation: {label}",
+      designSwitcherCountTemplate: "View {index} of {total}",
+      designNavigationAriaLabel: "Presentation choices",
+    };
+
+    render(
+      <DesignSwitcher
+        activeId="editorial"
+        contentDebug
+        currentPath="/projects"
+        templates={content.presentation.templates}
+        ui={ui}
+      />,
+    );
+
+    const summary = screen.getByLabelText("Choose presentation: Editorial");
+    expect(summary).toHaveTextContent("View 03 of 05");
+
+    const navigation = screen.getByRole("navigation", {
+      hidden: true,
+      name: "Presentation choices",
+    });
+    const classicLink = within(navigation).getByRole("link", {
+      hidden: true,
+      name: /Classic/,
+    });
+    const details = summary.closest("details");
+    const closeButton = screen.getByRole("button", {
+      hidden: true,
+      name: ui.designSwitcherCloseLabel,
+    });
+
+    expect(details).not.toBeNull();
+    details?.setAttribute("open", "");
+    fireEvent.click(closeButton);
+    expect(details).not.toHaveAttribute("open");
+    expect(summary).toHaveFocus();
+
+    details?.setAttribute("open", "");
+    document.addEventListener("click", (event) => event.preventDefault(), {
+      capture: true,
+      once: true,
+    });
+    fireEvent.click(classicLink);
+    expect(details).not.toHaveAttribute("open");
+  });
+});
diff --git a/src/components/portfolio/project-links.test.tsx b/src/components/portfolio/project-links.test.tsx
new file mode 100644
index 0000000..cd1c4c0
--- /dev/null
+++ b/src/components/portfolio/project-links.test.tsx
@@ -0,0 +1,157 @@
+import { render, screen } from "@testing-library/react";
+import { describe, expect, it } from "vitest";
+
+import type { PortfolioProject } from "@/lib/portfolio";
+
+import { ProjectCardLinks, ProjectLinks } from "./project-links";
+
+const project: PortfolioProject = {
+  id: "sample-project",
+  order: "999",
+  title: "Sample Project",
+  groupId: "featured",
+  tags: ["sample"],
+  category: "Web App",
+  period: "2026",
+  role: "Developer",
+  summary: "Summary",
+  description: "Description",
+  deployment: {
+    label: "Live",
+    status: "live",
+  },
+  screenshot: {
+    alt: "Sample project",
+    src: "/content/projects/sample.svg",
+  },
+  screenshots: [],
+  stack: [],
+  links: [
+    {
+      href: "/projects/sample-project",
+      label: "Case Study",
+      placements: ["card", "detail"],
+      type: "case-study",
+    },
+    {
+      external: true,
+      href: "https://github.com/example/sample",
+      label: "GitHub",
+      placements: ["card", "detail"],
+      type: "github",
+    },
+    {
+      external: true,
+      href: "https://example.com/demo",
+      label: "Live Demo",
+      placements: ["card", "detail"],
+      type: "demo",
+    },
+    {
+      external: true,
+      href: "https://example.com/source",
+      label: "Source",
+      placements: ["detail"],
+      type: "source",
+    },
+  ],
+  highlights: [],
+  problem: "Problem",
+  solution: "Solution",
+  architecture: {
+    items: [],
+    summary: "Architecture",
+  },
+  decisions: [],
+  tradeoffs: [],
+  results: [],
+};
+
+describe("project links", () => {
+  it("renders detail links in source order", () => {
+    render(
+      <ProjectLinks
+        contentDebug
+        homeTemplate="classic"
+        project={project}
+      />,
+    );
+
+    const links = screen.getAllByRole("link");
+
+    expect(links.map((link) => link.textContent)).toEqual([
+      "Case Study",
+      "GitHub",
+      "Live Demo",
+      "Source",
+    ]);
+    expect(links[0]).toHaveAttribute(
+      "href",
+      "/projects/sample-project?view=classic&debug=content",
+    );
+    expect(links[0]).not.toHaveAttribute("target");
+    expect(links[1]).toHaveAttribute("target", "_blank");
+    expect(links[1]).toHaveAttribute("rel", "noreferrer");
+  });
+
+  it("applies detail filtering without hiding source links", () => {
+    render(
+      <ProjectLinks
+        excludeCaseStudy
+        project={{
+          ...project,
+          deployment: { label: "Offline", status: "offline" },
+        }}
+      />,
+    );
+
+    expect(
+      screen.queryByRole("link", { name: "Case Study" }),
+    ).not.toBeInTheDocument();
+    expect(
+      screen.queryByRole("link", { name: "Live Demo" }),
+    ).not.toBeInTheDocument();
+    expect(screen.getAllByRole("link").map((link) => link.textContent)).toEqual([
+      "GitHub",
+      "Source",
+    ]);
+  });
+
+  it("limits card links to their declared placement", () => {
+    render(<ProjectCardLinks project={project} />);
+
+    expect(screen.getAllByRole("link").map((link) => link.textContent)).toEqual([
+      "Case Study",
+      "GitHub",
+      "Live Demo",
+    ]);
+    expect(
+      screen.queryByRole("link", { name: "Source" }),
+    ).not.toBeInTheDocument();
+  });
+
+  it("renders no wrapper when filtering leaves no links", () => {
+    const { container, rerender } = render(
+      <ProjectCardLinks
+        project={{
+          ...project,
+          links: [project.links[3]],
+        }}
+      />,
+    );
+
+    expect(container).toBeEmptyDOMElement();
+
+    rerender(
+      <ProjectLinks
+        excludeCaseStudy
+        project={{
+          ...project,
+          links: [project.links[0]],
+        }}
+      />,
+    );
+
+    expect(container).toBeEmptyDOMElement();
+  });
+});


