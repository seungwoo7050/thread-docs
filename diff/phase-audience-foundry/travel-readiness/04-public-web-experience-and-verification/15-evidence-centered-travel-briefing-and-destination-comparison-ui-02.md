## `feat(frontend): establish briefing design system`

diff --git a/e2e/browser_acceptance.py b/e2e/browser_acceptance.py
index 856654c..96ecd6d 100644
--- a/e2e/browser_acceptance.py
+++ b/e2e/browser_acceptance.py
@@ -173,8 +173,8 @@ WARNING_PUBLIC_SOURCE_LOCATOR: Final = (
 )
 ENTRY_FRESHNESS_MINUTES: Final = 36 * 60
 WARNING_FRESHNESS_MINUTES: Final = 8 * 60
-SITE_CSS_SHA256: Final = "7591bba210c39bdd69c1409b3b2bccb1c829b8f059c601220965884251cce968"
-SITE_CSS_BYTES: Final = 8_478
+SITE_CSS_SHA256: Final = "0b4009a2837437f6af89241fa70a271e99ef2466ba27a38a38de4ed9de161f77"
+SITE_CSS_BYTES: Final = 8_534
 SITE_JS_SHA256: Final = "79754d4ab020672f48ea1d7311fd1583f40e19c50a6af41b4bdf2c1b438c97d4"
 SITE_JS_BYTES: Final = 1_544
 _SIGNAL_INTERRUPTED = False
diff --git a/e2e/browser_scenario_server.py b/e2e/browser_scenario_server.py
index 84dc8cc..3c62766 100644
--- a/e2e/browser_scenario_server.py
+++ b/e2e/browser_scenario_server.py
@@ -49,8 +49,8 @@ SITE_ASSETS: Final = {
     "/static/public_web/site.css": (
         REPOSITORY_ROOT / "public_web" / "static" / "public_web" / "site.css",
         "text/css",
-        8_478,
-        "7591bba210c39bdd69c1409b3b2bccb1c829b8f059c601220965884251cce968",
+        8_534,
+        "0b4009a2837437f6af89241fa70a271e99ef2466ba27a38a38de4ed9de161f77",
     ),
     "/static/public_web/site.js": (
         REPOSITORY_ROOT / "public_web" / "static" / "public_web" / "site.js",
diff --git a/public_web/static/public_web/site.css b/public_web/static/public_web/site.css
index f2671c2..f3b71e2 100644
--- a/public_web/static/public_web/site.css
+++ b/public_web/static/public_web/site.css
@@ -1,20 +1,33 @@
 :root {
   color-scheme: light;
-  --ink: #142f35;
-  --muted-ink: #50656a;
-  --canvas: #f3f7f5;
-  --surface: #ffffff;
-  --surface-soft: #eaf2ef;
-  --line: #b9ccc7;
-  --brand: #123f46;
-  --brand-strong: #082d33;
-  --accent: #d7a441;
-  --focus: #006d8f;
-  --danger: #9f2f24;
-  --shadow: 0 0.75rem 2rem rgb(20 47 53 / 10%);
-  font-family: system-ui, -apple-system, BlinkMacSystemFont, "Apple SD Gothic Neo", sans-serif;
+  --canvas: #F5F3EC;
+  --surface: #FEFDF9;
+  --surface-muted: #ECEBE4;
+  --ink: #17242D;
+  --muted-ink: #4B5A63;
+  --header: #112B36;
+  --link: #1557B0;
+  --focus: #005FCC;
+  --current: #0B6663;
+  --stale: #8A5A00;
+  --error: #A12A2A;
+  --line: #C9CECB;
+  --space-1: 0.25rem;
+  --space-2: 0.5rem;
+  --space-3: 0.75rem;
+  --space-4: 1rem;
+  --space-6: 1.5rem;
+  --space-8: 2rem;
+  --space-12: 3rem;
+  --space-16: 4rem;
+  --space-24: 6rem;
+  --shell-width: 72rem;
+  --reading-width: 44rem;
+  --control-radius: 0.375rem;
+  --section-radius: 0.25rem;
+  font-family: system-ui, "Apple SD Gothic Neo", "Malgun Gothic", sans-serif;
   font-size: 100%;
-  line-height: 1.6;
+  line-height: 1.65;
 }
 
 * {
@@ -31,9 +44,7 @@ body {
   min-height: 100vh;
   margin: 0;
   color: var(--ink);
-  background:
-    linear-gradient(180deg, rgb(215 164 65 / 10%), transparent 18rem),
-    var(--canvas);
+  background: var(--canvas);
 }
 
 :where(p, li, dt, dd, a, strong, label, legend) {
@@ -49,7 +60,6 @@ body {
 .skip-link:focus-visible,
 .site-header a:focus-visible {
   outline-color: white;
-  box-shadow: 0 0 0 0.4375rem #000000;
 }
 
 [hidden] {
@@ -57,7 +67,7 @@ body {
 }
 
 .page-shell {
-  width: min(100% - 2rem, 72rem);
+  width: min(100% - 2rem, var(--shell-width));
   min-width: 0;
   margin-inline: auto;
 }
@@ -72,8 +82,8 @@ body {
   align-items: center;
   padding: 0.5rem 0.875rem;
   color: white;
-  background: var(--brand-strong);
-  border-radius: 0.375rem;
+  background: var(--header);
+  border-radius: var(--control-radius);
   transform: translateY(-150%);
 }
 
@@ -83,15 +93,31 @@ body {
 
 .site-header {
   color: white;
-  background: var(--brand);
-  border-bottom: 0.25rem solid var(--accent);
+  background: var(--header);
+  border-bottom: 0.0625rem solid rgb(255 255 255 / 28%);
+}
+
+.site-header-inner {
+  display: flex;
+  min-height: 4rem;
+  flex-wrap: wrap;
+  gap: var(--space-2) var(--space-6);
+  align-items: center;
+  justify-content: space-between;
+}
+
+.site-brand {
+  padding-block: var(--space-3);
+  font-size: 1.0625rem;
+  font-weight: 800;
+  letter-spacing: -0.02em;
 }
 
 .site-nav {
   display: flex;
   flex-wrap: wrap;
-  gap: 0.5rem;
-  padding-block: 0.625rem;
+  gap: var(--space-1);
+  padding-block: var(--space-2);
 }
 
 .site-nav a {
@@ -103,17 +129,17 @@ body {
   font-weight: 700;
   text-decoration-thickness: 0.08em;
   text-underline-offset: 0.2em;
-  border-radius: 0.4rem;
+  border-radius: var(--control-radius);
 }
 
 .site-nav a[aria-current="page"] {
-  color: var(--brand-strong);
-  background: white;
+  color: white;
+  background: var(--current);
   text-decoration: none;
 }
 
 .page-main {
-  padding-block: clamp(2rem, 7vw, 5rem);
+  padding-block: clamp(var(--space-8), 7vw, var(--space-16));
 }
 
 .page-main > * {
@@ -121,13 +147,13 @@ body {
 }
 
 .page-heading {
-  max-width: 48rem;
+  max-width: var(--reading-width);
   margin-bottom: clamp(1.5rem, 4vw, 2.5rem);
 }
 
 .eyebrow {
   margin: 0 0 0.375rem;
-  color: var(--brand);
+  color: var(--current);
   font-size: 0.875rem;
   font-weight: 800;
   letter-spacing: 0.05em;
@@ -141,8 +167,8 @@ h2 {
 
 h1 {
   margin: 0;
-  font-size: clamp(2rem, 7vw, 3.75rem);
-  letter-spacing: -0.035em;
+  font-size: clamp(2rem, 6vw, 3.25rem);
+  letter-spacing: -0.04em;
 }
 
 h2 {
@@ -152,7 +178,7 @@ h2 {
 }
 
 .page-lead {
-  max-width: 42rem;
+  max-width: var(--reading-width);
   margin: 1rem 0 0;
   color: var(--muted-ink);
   font-size: clamp(1rem, 2.5vw, 1.1875rem);
@@ -162,10 +188,10 @@ h2 {
   max-width: 48rem;
   margin-bottom: 1.5rem;
   padding: clamp(1rem, 4vw, 1.5rem);
-  background: #fff7f5;
-  border: 0.125rem solid var(--danger);
+  background: var(--surface);
+  border: 0.125rem solid var(--error);
   border-left-width: 0.5rem;
-  border-radius: 0.75rem;
+  border-radius: var(--section-radius);
 }
 
 .error-summary:focus {
@@ -187,7 +213,7 @@ h2 {
   display: inline-flex;
   min-height: 44px;
   align-items: center;
-  color: var(--danger);
+  color: var(--error);
   font-weight: 700;
 }
 
@@ -196,8 +222,7 @@ h2 {
   padding: clamp(1rem, 5vw, 2rem);
   background: var(--surface);
   border: 0.0625rem solid var(--line);
-  border-radius: 1rem;
-  box-shadow: var(--shadow);
+  border-radius: var(--section-radius);
 }
 
 .trip-form fieldset {
@@ -238,13 +263,13 @@ h2 {
   color: var(--ink);
   font: inherit;
   font-size: 1rem;
-  background: white;
-  border: 0.125rem solid #718984;
-  border-radius: 0.5rem;
+  background: var(--surface);
+  border: 0.125rem solid #68767E;
+  border-radius: var(--control-radius);
 }
 
 .form-field--error :where(input, select) {
-  border-color: var(--danger);
+  border-color: var(--error);
   border-style: double;
 }
 
@@ -256,7 +281,7 @@ h2 {
 }
 
 .field-errors {
-  color: var(--danger);
+  color: var(--error);
   font-weight: 700;
 }
 
@@ -271,14 +296,20 @@ h2 {
   color: white;
   font: inherit;
   font-weight: 800;
-  background: var(--brand);
-  border: 0.125rem solid var(--brand);
-  border-radius: 0.6rem;
+  background: var(--header);
+  border: 0.125rem solid var(--header);
+  border-radius: var(--control-radius);
   cursor: pointer;
 }
 
 .primary-button:hover {
-  background: var(--brand-strong);
+  background: var(--current);
+  border-color: var(--current);
+}
+
+.primary-button:active {
+  background: var(--ink);
+  border-color: var(--ink);
 }
 
 .primary-button[aria-disabled="true"] {
@@ -303,25 +334,22 @@ h2 {
   min-width: 0;
   padding: clamp(1rem, 4vw, 1.75rem);
   background: var(--surface);
-  border: 0.125rem solid var(--line);
-  border-top: 0.4rem solid var(--brand);
-  border-radius: 1rem;
-  box-shadow: var(--shadow);
+  border: 0.0625rem solid var(--line);
+  border-left: 0.35rem solid var(--current);
+  border-radius: var(--section-radius);
 }
 
 .publication-card[data-state="empty"] {
-  border-style: dashed;
+  border-left-color: var(--muted-ink);
 }
 
 .publication-card[data-state="unavailable"],
 .publication-card[data-state="stale"] {
-  border-top-color: var(--accent);
-  border-top-style: double;
+  border-left-color: var(--stale);
 }
 
 .publication-card[data-state="server-error"] {
-  border-top-color: var(--danger);
-  border-top-style: double;
+  border-left-color: var(--error);
 }
 
 .status-line {
@@ -333,32 +361,7 @@ h2 {
 }
 
 .status-symbol {
-  display: inline-grid;
-  width: 1.75rem;
-  height: 1.75rem;
-  flex: 0 0 auto;
-  place-items: center;
-  color: white;
-  background: var(--brand);
-  border-radius: 50%;
-  font-weight: 900;
-}
-
-[data-state="ready"] .status-symbol::before { content: "✓"; }
-[data-state="empty"] .status-symbol::before { content: "○"; }
-[data-state="unavailable"] .status-symbol::before { content: "?"; }
-[data-state="stale"] .status-symbol::before { content: "!"; }
-[data-state="server-error"] .status-symbol::before { content: "×"; }
-
-[data-state="empty"] .status-symbol,
-[data-state="unavailable"] .status-symbol,
-[data-state="stale"] .status-symbol {
-  color: var(--brand-strong);
-  background: var(--accent);
-}
-
-[data-state="server-error"] .status-symbol {
-  background: var(--danger);
+  display: none;
 }
 
 .fact-list {
@@ -391,22 +394,22 @@ h2 {
   align-items: center;
   margin-inline-start: 0.25rem;
   padding-inline: 0.25rem;
-  color: var(--brand);
+  color: var(--link);
   font-weight: 800;
 }
 
 .verification-note {
   margin-bottom: 0;
   padding: 1rem;
-  background: var(--surface-soft);
-  border-left: 0.35rem solid var(--accent);
-  border-radius: 0.4rem;
+  background: var(--surface-muted);
+  border-left: 0.35rem solid var(--stale);
+  border-radius: var(--section-radius);
 }
 
 .site-footer {
   padding-block: 1.5rem;
   color: white;
-  background: var(--brand-strong);
+  background: var(--header);
 }
 
 .site-footer p {
@@ -430,20 +433,24 @@ h2 {
   }
 }
 
-@media (min-width: 64rem) {
-  .publication-grid {
-    grid-template-columns: repeat(2, minmax(0, 1fr));
-  }
-}
-
 @media (max-width: 30rem) {
   .page-shell {
-    width: min(100% - 1.25rem, 72rem);
+    width: min(100% - 1.25rem, var(--shell-width));
+  }
+
+  .site-header-inner {
+    align-items: stretch;
+    gap: 0;
+  }
+
+  .site-brand,
+  .site-nav {
+    width: 100%;
   }
 
   .site-nav a {
-    flex: 1 1 9rem;
-    justify-content: center;
+    flex: 1 1 auto;
+    justify-content: flex-start;
   }
 }
 
@@ -457,7 +464,8 @@ h2 {
 
 @media (prefers-reduced-motion: no-preference) {
   :where(a, button, input, select) {
-    transition: outline-offset 120ms ease, background-color 120ms ease;
+    transition: outline-offset 120ms ease, background-color 120ms ease,
+      border-color 120ms ease;
   }
 }
 
@@ -467,4 +475,8 @@ h2 {
   .verification-note {
     border: 0.125rem solid CanvasText;
   }
+
+  :where(a, button, input, select, [tabindex="0"]):focus-visible {
+    outline: 0.1875rem solid Highlight;
+  }
 }
diff --git a/public_web/templates/public_web/base.html b/public_web/templates/public_web/base.html
index b4a0ca2..45b40bf 100644
--- a/public_web/templates/public_web/base.html
+++ b/public_web/templates/public_web/base.html
@@ -4,14 +4,15 @@
 <head>
   <meta charset="utf-8">
   <meta name="viewport" content="width=device-width, initial-scale=1">
-  <meta name="theme-color" content="#123f46">
+  <meta name="theme-color" content="#112B36">
   <title>{% block title %}여행준비{% endblock %}</title>
   <link rel="stylesheet" href="{% static 'public_web/site.css' %}">
 </head>
 <body>
   <a class="skip-link" href="#main-content">본문으로 건너뛰기</a>
   <header class="site-header">
-    <div class="page-shell">
+    <div class="page-shell site-header-inner">
+      <span class="site-brand">여행준비</span>
       <nav class="site-nav" aria-label="주요 메뉴">
         {% block nav_links %}{% endblock %}
       </nav>
diff --git a/public_web/tests/test_accessibility_contract.py b/public_web/tests/test_accessibility_contract.py
index 4b0c618..978c25c 100644
--- a/public_web/tests/test_accessibility_contract.py
+++ b/public_web/tests/test_accessibility_contract.py
@@ -30,6 +30,7 @@ class AccessiblePresentationContractTests(SimpleTestCase):
         self.assertEqual(body.count("<h1"), 1)
         self.assertContains(response, 'class="skip-link" href="#main-content"')
         self.assertContains(response, "<header", html=False)
+        self.assertContains(response, '<span class="site-brand">여행준비</span>')
         self.assertContains(response, 'nav class="site-nav" aria-label="주요 메뉴"')
         self.assertContains(response, 'main id="main-content"')
         self.assertContains(response, "<footer", html=False)
@@ -141,14 +142,36 @@ class AccessiblePresentationContractTests(SimpleTestCase):
             "overflow-wrap: anywhere",
             "word-break: keep-all",
             "grid-template-columns: minmax(0, 1fr)",
-            "repeat(2, minmax(0, 1fr))",
             "@media (max-width: 30rem)",
             "@media (pointer: coarse)",
             "@media (forced-colors: active)",
             ".site-header a:focus-visible",
             "outline-color: white",
-            "box-shadow: 0 0 0 0.4375rem #000000",
+            '--canvas: #F5F3EC',
+            '--surface: #FEFDF9',
+            '--ink: #17242D',
+            '--muted-ink: #4B5A63',
+            '--header: #112B36',
+            '--link: #1557B0',
+            '--focus: #005FCC',
+            '--current: #0B6663',
+            '--stale: #8A5A00',
+            '--error: #A12A2A',
+            '--shell-width: 72rem',
+            '--reading-width: 44rem',
+            'system-ui, "Apple SD Gothic Neo", "Malgun Gothic", sans-serif',
+            '.status-symbol {\n  display: none;',
         ):
             self.assertIn(required, self.css)
-        for forbidden in ("@import", "url(http://", "url(https://"):
+        for forbidden in (
+            "@import",
+            "url(http://",
+            "url(https://",
+            "linear-gradient",
+            "radial-gradient",
+            "box-shadow",
+            "filter: blur",
+            "backdrop-filter",
+            "repeat(2, minmax(0, 1fr))",
+        ):
             self.assertNotIn(forbidden, self.css)


