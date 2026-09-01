## `feat(frontend): add destination comparison index`

diff --git a/e2e/browser_acceptance.py b/e2e/browser_acceptance.py
index 5309e1e..c528a0c 100644
--- a/e2e/browser_acceptance.py
+++ b/e2e/browser_acceptance.py
@@ -185,8 +185,8 @@ WARNING_PUBLIC_SOURCE_LOCATOR: Final = (
 )
 ENTRY_FRESHNESS_MINUTES: Final = 36 * 60
 WARNING_FRESHNESS_MINUTES: Final = 8 * 60
-SITE_CSS_SHA256: Final = "ca317ac24714a9404cdaeb0a37a301ebb3781304c3aa210ba4adbe80d288f893"
-SITE_CSS_BYTES: Final = 21_361
+SITE_CSS_SHA256: Final = "2d1dcfb4ce3e2d908ac7a5f8f0f314c624a6a8484b6c986a2e6c314d0cd249aa"
+SITE_CSS_BYTES: Final = 28_687
 SITE_JS_SHA256: Final = "1a5a95b286d6e12f1765351d5a0b1528c833fb46af16d747b469b61323cc7f44"
 SITE_JS_BYTES: Final = 1_628
 MARU_BURI_SHA256: Final = "5c8b39b683595d0ddcf2554148f4d2fb14c55cb967e5bff2e282b2936034fc75"
@@ -3652,7 +3652,7 @@ def _focused_result_javascript(
     }}
     for (const opportunity of await opportunities.all()) {{
       if (await opportunity.locator('.alternatives > ol > li').count() > 2) fail('alternative-count');
-      if (!(await opportunity.textContent()).includes('공식 운항 일정') || !(await opportunity.textContent()).includes('제품 예상')) fail('fact-estimate-separation');
+      if (!(await opportunity.textContent()).includes('공식 운항 일정') || !(await opportunity.textContent()).includes('예상 시각')) fail('fact-estimate-separation');
     }}
     const externalLinks = page.locator('a[rel="noopener noreferrer"]');
     if (await externalLinks.count() < 3) fail('source-links');
diff --git a/e2e/browser_scenario_server.py b/e2e/browser_scenario_server.py
index 1238ea5..88a4c7e 100644
--- a/e2e/browser_scenario_server.py
+++ b/e2e/browser_scenario_server.py
@@ -57,8 +57,8 @@ SITE_ASSETS: Final = {
     "/static/public_web/site.css": (
         REPOSITORY_ROOT / "public_web" / "static" / "public_web" / "site.css",
         "text/css",
-        21_361,
-        "ca317ac24714a9404cdaeb0a37a301ebb3781304c3aa210ba4adbe80d288f893",
+        28_687,
+        "2d1dcfb4ce3e2d908ac7a5f8f0f314c624a6a8484b6c986a2e6c314d0cd249aa",
     ),
     "/static/public_web/site.js": (
         REPOSITORY_ROOT / "public_web" / "static" / "public_web" / "site.js",
diff --git a/operations/tests/test_browser_scenario_servers.py b/operations/tests/test_browser_scenario_servers.py
index 1f77569..3858605 100644
--- a/operations/tests/test_browser_scenario_servers.py
+++ b/operations/tests/test_browser_scenario_servers.py
@@ -359,7 +359,8 @@ class BrowserScenarioCardTests(SimpleTestCase):
 
         for public_path, (payload, content_type) in assets.items():
             response = views["static"](
-                factory.get(public_path), asset=public_path.rsplit("/", 1)[1]
+                factory.get(public_path),
+                asset=public_path.removeprefix("/static/public_web/"),
             )
             self.assertEqual(response.status_code, 200)
             self.assertEqual(response.content, payload)
@@ -394,7 +395,7 @@ class BrowserScenarioCardTests(SimpleTestCase):
         self.assertIn("갈 수 있는 도시를 찾았습니다".encode(), valid.content)
         self.assertIn("후쿠오카".encode(), valid.content)
         self.assertIn("공식 운항 일정".encode(), valid.content)
-        self.assertIn("제품 예상".encode(), valid.content)
+        self.assertIn("예상 시각".encode(), valid.content)
         self.assertEqual(delays, ["delay"])
         self.assertEqual(calls, ["read"])
 
diff --git a/public_web/static/public_web/site.css b/public_web/static/public_web/site.css
index 324eacf..d80ae83 100644
--- a/public_web/static/public_web/site.css
+++ b/public_web/static/public_web/site.css
@@ -27,6 +27,7 @@
   --canvas: #F4F8FB;
   --surface: #FFFFFF;
   --surface-muted: #EAF3F8;
+  --surface-warm: #FFF7E3;
   --ink: #0B1F33;
   --muted-ink: #4B6072;
   --header: #073B66;
@@ -537,15 +538,47 @@ h3 {
 }
 
 .results-header {
-  max-width: var(--reading-width);
-  margin-left: auto;
-  padding-bottom: var(--space-8);
+  max-width: none;
+  padding-bottom: clamp(var(--space-8), 6vw, var(--space-12));
   border-bottom: 0.125rem solid var(--header);
 }
 
+.results-title-row {
+  display: flex;
+  min-width: 0;
+  gap: var(--space-6);
+  align-items: flex-end;
+  justify-content: space-between;
+}
+
+.results-title-row > div {
+  min-width: 0;
+}
+
+.results-header h2 {
+  max-width: 18ch;
+  font-size: clamp(2.05rem, 4.5vw, 3.65rem);
+}
+
+.results-count {
+  margin: var(--space-2) 0 0;
+  color: var(--muted-ink);
+}
+
+.edit-search {
+  display: inline-flex;
+  flex: 0 0 auto;
+  min-height: 44px;
+  align-items: center;
+  color: var(--link);
+  font-weight: 700;
+  text-underline-offset: 0.25em;
+}
+
 .search-window {
   display: grid;
   min-width: 0;
+  max-width: 52rem;
   margin: var(--space-8) 0 0;
   border-block: 0.0625rem solid var(--line-strong);
   grid-template-columns: repeat(2, minmax(0, 1fr));
@@ -569,10 +602,12 @@ h3 {
 .search-window dd {
   margin: var(--space-1) 0 0;
   font-size: 1.0625rem;
-  font-weight: 850;
+  font-variant-numeric: tabular-nums;
+  font-weight: 700;
 }
 
 .flight-status {
+  max-width: 52rem;
   margin-top: var(--space-6);
   padding-left: var(--space-4);
   border-left: 0.35rem solid var(--current);
@@ -600,23 +635,202 @@ h3 {
   color: var(--muted-ink);
 }
 
+.flight-status-heading {
+  display: flex;
+  min-width: 0;
+  flex-wrap: wrap;
+  gap: var(--space-2) var(--space-4);
+  align-items: baseline;
+}
+
+.flight-status-heading span {
+  color: var(--muted-ink);
+  font-size: 0.75rem;
+  font-weight: 700;
+  letter-spacing: 0.07em;
+}
+
 .flight-source {
+  max-width: 52rem;
   margin: var(--space-4) 0 0;
   color: var(--muted-ink);
   font-size: 0.875rem;
 }
 
+.flight-source summary {
+  display: inline-flex;
+  min-height: 44px;
+  align-items: center;
+  color: var(--link);
+  font-weight: 700;
+  cursor: pointer;
+}
+
+.flight-source p {
+  margin: 0;
+}
+
+.flight-source p + p {
+  margin-top: var(--space-1);
+}
+
 .flight-source a {
   display: inline-flex;
   min-height: 44px;
   align-items: center;
   color: var(--link);
-  font-weight: 800;
+  font-weight: 700;
+}
+
+.empty-guidance {
+  max-width: 52rem;
+  margin-top: var(--space-6);
+  padding: var(--space-4) var(--space-6);
+  background: var(--surface-muted);
+  border-left: 0.25rem solid var(--line-strong);
+}
+
+.empty-guidance p {
+  margin: 0;
+}
+
+.results-layout {
+  display: grid;
+  min-width: 0;
+  gap: clamp(var(--space-8), 5vw, var(--space-16));
+  align-items: start;
+  grid-template-columns: minmax(18rem, 22rem) minmax(0, 1fr);
+}
+
+.destination-index {
+  position: sticky;
+  top: var(--space-6);
+  min-width: 0;
+  padding-top: var(--space-8);
+}
+
+.destination-index-heading {
+  padding-bottom: var(--space-4);
+  border-bottom: 0.125rem solid var(--header);
+}
+
+.index-kicker {
+  margin: 0 0 var(--space-1);
+  color: var(--current);
+  font-size: 0.75rem;
+  font-weight: 700;
+  letter-spacing: 0.08em;
+}
+
+.destination-index h3 {
+  font-family: "NanumSquare Neo", system-ui, sans-serif;
+  font-size: 1rem;
+  font-weight: 700;
+  letter-spacing: 0;
+}
+
+.destination-index ol {
+  margin: 0;
+  padding: 0;
+  list-style: none;
+}
+
+.destination-index li {
+  min-width: 0;
+  border-bottom: 0.0625rem solid var(--line);
+}
+
+.destination-index a {
+  display: grid;
+  min-width: 0;
+  gap: var(--space-3) var(--space-3);
+  padding-block: var(--space-4);
+  color: var(--ink);
+  text-decoration: none;
+  grid-template-columns: 2rem minmax(0, 1fr) auto;
+}
+
+.destination-index a:hover .index-place > strong,
+.destination-index a:focus-visible .index-place > strong {
+  color: var(--link);
+  text-decoration: underline;
+  text-underline-offset: 0.2em;
+}
+
+.index-rank {
+  color: var(--current);
+  font-size: 0.75rem;
+  font-variant-numeric: tabular-nums;
+  font-weight: 700;
+  letter-spacing: 0.08em;
+}
+
+.index-place {
+  min-width: 0;
+}
+
+.index-place > strong,
+.index-place > small,
+.index-module-states,
+.index-flight small,
+.index-flight strong,
+.index-stay small,
+.index-stay strong {
+  display: block;
+}
+
+.index-place > strong {
+  font-family: "Maru Buri", "Batang", serif;
+  font-size: 1.2rem;
+  font-weight: 600;
+  line-height: 1.3;
+}
+
+.index-place > small,
+.index-module-states {
+  color: var(--muted-ink);
+  font-size: 0.6875rem;
+}
+
+.index-module-states {
+  margin-top: var(--space-1);
+}
+
+.index-flight {
+  display: flex;
+  min-width: 0;
+  gap: var(--space-2);
+  align-items: baseline;
+  justify-content: space-between;
+  grid-column: 2 / -1;
+}
+
+.index-flight small,
+.index-stay small {
+  color: var(--muted-ink);
+  font-size: 0.6875rem;
+}
+
+.index-flight strong,
+.index-stay strong {
+  font-size: 0.8125rem;
+  font-variant-numeric: tabular-nums;
+  font-weight: 700;
+}
+
+.index-stay {
+  min-width: 5.75rem;
+  text-align: right;
+}
+
+.index-stay strong {
+  color: var(--header);
+  font-size: 1rem;
 }
 
 .opportunity-list {
-  max-width: var(--reading-width);
-  margin: 0 0 0 auto;
+  max-width: 52rem;
+  margin: 0;
   padding: 0;
   list-style: none;
 }
@@ -628,23 +842,46 @@ h3 {
 .opportunity {
   min-width: 0;
   padding-block: clamp(var(--space-12), 8vw, var(--space-16));
-  border-bottom: 0.125rem solid var(--header);
+  border-top: 0.1875rem solid var(--header);
+  scroll-margin-top: var(--space-6);
+}
+
+.opportunity-list > li:last-child .opportunity {
+  border-bottom: 0.1875rem solid var(--header);
+}
+
+.opportunity-route {
+  display: flex;
+  min-width: 0;
+  gap: var(--space-4);
+  align-items: baseline;
+  justify-content: space-between;
 }
 
 .opportunity-rank {
-  margin: 0 0 var(--space-2);
+  margin: 0;
   color: var(--current);
   font-size: 0.8125rem;
-  font-weight: 850;
+  font-weight: 700;
   letter-spacing: 0.08em;
 }
 
+.route-code {
+  margin: 0;
+  color: var(--muted-ink);
+  font-size: 0.75rem;
+  font-variant-numeric: tabular-nums;
+  font-weight: 700;
+  letter-spacing: 0.06em;
+}
+
 .destination-heading {
   display: flex;
   min-width: 0;
   gap: var(--space-6);
   align-items: flex-end;
   justify-content: space-between;
+  margin-top: var(--space-2);
 }
 
 .destination-heading > div {
@@ -652,8 +889,8 @@ h3 {
 }
 
 .destination-heading h3 {
-  font-size: clamp(2rem, 7vw, 3.25rem);
-  letter-spacing: -0.055em;
+  font-size: clamp(2.25rem, 6vw, 3.75rem);
+  letter-spacing: -0.05em;
 }
 
 .destination-heading > div p {
@@ -671,7 +908,7 @@ h3 {
   display: block;
   color: var(--muted-ink);
   font-size: 0.8125rem;
-  font-weight: 750;
+  font-weight: 700;
 }
 
 .stay-time strong {
@@ -679,12 +916,27 @@ h3 {
   font-size: clamp(1.5rem, 5vw, 2.2rem);
   letter-spacing: -0.04em;
   line-height: 1.15;
+  font-variant-numeric: tabular-nums;
 }
 
 .main-itinerary {
   margin-top: var(--space-8);
 }
 
+.section-heading-row {
+  display: flex;
+  min-width: 0;
+  gap: var(--space-4);
+  align-items: baseline;
+  justify-content: space-between;
+}
+
+.section-heading-row p {
+  margin: 0;
+  color: var(--muted-ink);
+  font-size: 0.75rem;
+}
+
 .main-itinerary > h4,
 .calculation-note h4,
 .travel-briefs > h4 {
@@ -726,7 +978,7 @@ h3 {
 
 .flight-leg-heading .leg-direction {
   color: var(--ink);
-  font-weight: 850;
+  font-weight: 700;
 }
 
 .event-list {
@@ -735,16 +987,6 @@ h3 {
   padding-left: var(--space-6);
 }
 
-.event-list::before {
-  position: absolute;
-  top: 0.4rem;
-  bottom: 0.4rem;
-  left: 0.25rem;
-  width: 0.125rem;
-  content: "";
-  background: var(--line-strong);
-}
-
 .event {
   position: relative;
 }
@@ -765,15 +1007,29 @@ h3 {
   border-radius: 50%;
 }
 
+.event:not(:last-child)::after {
+  position: absolute;
+  top: 1rem;
+  bottom: calc(-1 * var(--space-6) - 0.4rem);
+  left: calc(-1.5rem + 0.05rem);
+  content: "";
+  border-left: 0.125rem solid var(--current);
+}
+
 .event--estimated::before {
-  border-color: var(--muted-ink);
+  border-color: var(--stale);
   border-style: dashed;
 }
 
+.event--estimated:not(:last-child)::after {
+  border-left-color: var(--stale);
+  border-left-style: dashed;
+}
+
 .event dt {
   color: var(--muted-ink);
   font-size: 0.8125rem;
-  font-weight: 750;
+  font-weight: 700;
 }
 
 .event dt span {
@@ -787,10 +1043,16 @@ h3 {
   color: var(--current);
 }
 
+.event--estimated dt span {
+  color: var(--stale);
+  border-left-style: dashed;
+}
+
 .event dd {
   margin: 0;
   font-size: 1.125rem;
-  font-weight: 850;
+  font-variant-numeric: tabular-nums;
+  font-weight: 700;
 }
 
 .alternatives {
@@ -804,10 +1066,25 @@ h3 {
   align-items: center;
   padding-block: var(--space-3);
   color: var(--link);
-  font-weight: 850;
+  font-weight: 700;
   cursor: pointer;
 }
 
+.alternatives summary::after,
+.source-history summary::after,
+.flight-source summary::after {
+  margin-left: var(--space-2);
+  content: "+";
+  color: var(--muted-ink);
+  font-weight: 700;
+}
+
+.alternatives[open] summary::after,
+.source-history[open] summary::after,
+.flight-source[open] summary::after {
+  content: "−";
+}
+
 .alternatives > ol {
   margin: 0;
   padding: 0 0 var(--space-4);
@@ -815,17 +1092,30 @@ h3 {
 }
 
 .alternatives > ol > li {
-  padding: var(--space-4);
-  background: var(--surface);
-  border-left: 0.25rem solid var(--line-strong);
+  padding-block: var(--space-4);
+  border-top: 0.0625rem solid var(--line);
 }
 
 .alternatives > ol > li + li {
-  margin-top: var(--space-3);
+  margin-top: 0;
 }
 
 .alternative-stay {
-  margin: 0 0 var(--space-2);
+  display: flex;
+  min-width: 0;
+  gap: var(--space-3);
+  align-items: baseline;
+  justify-content: space-between;
+  margin: 0 0 var(--space-3);
+}
+
+.alternative-stay span {
+  color: var(--muted-ink);
+  font-size: 0.75rem;
+}
+
+.alternative-stay strong {
+  font-variant-numeric: tabular-nums;
 }
 
 .alternative-schedule {
@@ -836,7 +1126,8 @@ h3 {
   display: grid;
   min-width: 0;
   gap: var(--space-2);
-  grid-template-columns: 4rem minmax(0, 1fr);
+  padding-block: var(--space-2);
+  grid-template-columns: 4.5rem minmax(0, 1fr);
 }
 
 .alternative-schedule div + div {
@@ -854,11 +1145,27 @@ h3 {
   margin: 0;
 }
 
+.alternative-schedule dd strong,
+.alternative-schedule dd span {
+  display: block;
+}
+
+.alternative-schedule dd strong {
+  font-size: 0.875rem;
+}
+
+.alternative-schedule dd span {
+  margin-top: var(--space-1);
+  color: var(--muted-ink);
+  font-size: 0.8125rem;
+  font-variant-numeric: tabular-nums;
+}
+
 .calculation-note {
   margin-top: var(--space-8);
   padding: var(--space-4) var(--space-6);
   color: var(--muted-ink);
-  background: var(--surface-muted);
+  background: var(--surface-warm);
   border-left: 0.25rem solid var(--stale);
 }
 
@@ -866,9 +1173,32 @@ h3 {
   margin: 0;
 }
 
-.calculation-note p + p {
-  margin-top: var(--space-2);
-  font-size: 0.8125rem;
+.calculation-note dl {
+  display: flex;
+  min-width: 0;
+  flex-wrap: wrap;
+  gap: var(--space-2) var(--space-6);
+  margin: var(--space-3) 0 0;
+}
+
+.calculation-note dl > div {
+  display: flex;
+  min-width: 0;
+  gap: var(--space-2);
+}
+
+.calculation-note dt,
+.calculation-note dd {
+  margin: 0;
+  font-size: 0.75rem;
+}
+
+.calculation-note dt {
+  font-weight: 700;
+}
+
+.calculation-note dd {
+  font-variant-numeric: tabular-nums;
 }
 
 .travel-briefs {
@@ -1039,6 +1369,39 @@ h3 {
   font-weight: 850;
 }
 
+@media (max-width: 64rem) {
+  .results-layout {
+    grid-template-columns: minmax(0, 1fr);
+  }
+
+  .destination-index {
+    position: static;
+  }
+
+  .destination-index ol {
+    display: grid;
+    grid-template-columns: repeat(3, minmax(0, 1fr));
+  }
+
+  .destination-index li {
+    border-right: 0.0625rem solid var(--line);
+  }
+
+  .destination-index li:nth-child(3n) {
+    border-right: 0;
+  }
+
+  .destination-index a {
+    height: 100%;
+    padding-inline: var(--space-4);
+  }
+
+  .opportunity-list {
+    width: min(100%, 52rem);
+    margin-left: auto;
+  }
+}
+
 @media (max-width: 52rem) {
   .search-section {
     grid-template-columns: minmax(0, 1fr);
@@ -1052,8 +1415,27 @@ h3 {
     max-width: 15ch;
   }
 
-  .results-header,
+  .results-title-row {
+    align-items: flex-start;
+    flex-direction: column;
+    gap: var(--space-4);
+  }
+
+  .destination-index ol {
+    grid-template-columns: minmax(0, 1fr);
+  }
+
+  .destination-index li,
+  .destination-index li:nth-child(3n) {
+    border-right: 0;
+  }
+
+  .destination-index a {
+    padding-inline: 0;
+  }
+
   .opportunity-list {
+    width: 100%;
     margin-left: 0;
   }
 }
@@ -1122,6 +1504,19 @@ h3 {
     gap: var(--space-4);
   }
 
+  .opportunity-route,
+  .section-heading-row {
+    align-items: flex-start;
+    flex-direction: column;
+    gap: var(--space-1);
+  }
+
+  .index-flight {
+    align-items: flex-start;
+    flex-direction: column;
+    gap: 0;
+  }
+
   .stay-time {
     text-align: left;
   }
@@ -1130,6 +1525,12 @@ h3 {
     grid-template-columns: minmax(0, 1fr);
   }
 
+  .alternative-stay {
+    align-items: flex-start;
+    flex-direction: column;
+    gap: 0;
+  }
+
   .publication-brief {
     padding-left: var(--space-4);
   }
@@ -1143,6 +1544,8 @@ h3 {
 @media (pointer: coarse) {
   .site-brand,
   .primary-button,
+  .edit-search,
+  .destination-index a,
   .error-summary a,
   .flight-source a,
   .alternatives summary,
@@ -1174,6 +1577,8 @@ h3 {
   }
 
   .flight-status,
+  .destination-index-heading,
+  .destination-index li,
   .opportunity,
   .itinerary-legs,
   .flight-leg,
diff --git a/public_web/templates/public_web/partials/opportunity.html b/public_web/templates/public_web/partials/opportunity.html
index d31cd62..5f2679f 100644
--- a/public_web/templates/public_web/partials/opportunity.html
+++ b/public_web/templates/public_web/partials/opportunity.html
@@ -1,7 +1,10 @@
 <article id="destination-{{ opportunity.rank }}" class="opportunity"
          aria-labelledby="destination-{{ opportunity.rank }}-heading">
   <header class="opportunity-header">
-    <p class="opportunity-rank">추천 {{ opportunity.rank }}</p>
+    <div class="opportunity-route">
+      <p class="opportunity-rank">추천 {{ opportunity.rank|stringformat:"02d" }}</p>
+      <p class="route-code">ICN ↔ {{ opportunity.destination.airport_code }} · 직항</p>
+    </div>
     <div class="destination-heading">
       <div>
         <h3 id="destination-{{ opportunity.rank }}-heading">{{ opportunity.destination.city_name }}</h3>
@@ -15,7 +18,10 @@
   </header>
 
   <section class="main-itinerary" aria-labelledby="destination-{{ opportunity.rank }}-itinerary">
-    <h4 id="destination-{{ opportunity.rank }}-itinerary">추천 일정</h4>
+    <div class="section-heading-row">
+      <h4 id="destination-{{ opportunity.rank }}-itinerary">추천 일정</h4>
+      <p>공식 인천 운항시각과 예상 현지시각</p>
+    </div>
     <div class="itinerary-legs">
       <section class="flight-leg" aria-label="출국 운항 예정편">
         <div class="flight-leg-heading">
@@ -28,7 +34,7 @@
             <dd>{{ opportunity.outbound_schedule.icn_event_label }}</dd>
           </div>
           <div class="event event--estimated">
-            <dt>현지 도착 <span>제품 예상</span></dt>
+            <dt>현지 도착 <span>예상 시각</span></dt>
             <dd>{{ opportunity.outbound_schedule.estimated_destination_event_label }}</dd>
           </div>
         </dl>
@@ -41,7 +47,7 @@
         </div>
         <dl class="event-list">
           <div class="event event--estimated">
-            <dt>현지 출발 <span>제품 예상</span></dt>
+            <dt>현지 출발 <span>예상 시각</span></dt>
             <dd>{{ opportunity.inbound_schedule.estimated_destination_event_label }}</dd>
           </div>
           <div class="event event--official">
@@ -59,15 +65,21 @@
       <ol>
         {% for alternative in opportunity.alternatives %}
           <li>
-            <p class="alternative-stay">예상 현지 체류시간 <strong>{{ alternative.estimated_local_stay_label }}</strong></p>
+            <p class="alternative-stay"><span>예상 현지 체류시간</span><strong>{{ alternative.estimated_local_stay_label }}</strong></p>
             <dl class="alternative-schedule">
               <div>
                 <dt>가는 편</dt>
-                <dd>{{ alternative.outbound_schedule.carrier_name }} {{ alternative.outbound_schedule.flight_number }} · 인천 출발 {{ alternative.outbound_schedule.icn_event_label }}</dd>
+                <dd>
+                  <strong>{{ alternative.outbound_schedule.carrier_name }} {{ alternative.outbound_schedule.flight_number }}</strong>
+                  <span>인천 출발 {{ alternative.outbound_schedule.icn_event_label }} · 현지 도착 예상 {{ alternative.outbound_schedule.estimated_destination_event_label }}</span>
+                </dd>
               </div>
               <div>
                 <dt>오는 편</dt>
-                <dd>{{ alternative.inbound_schedule.carrier_name }} {{ alternative.inbound_schedule.flight_number }} · 인천 도착 {{ alternative.inbound_schedule.icn_event_label }}</dd>
+                <dd>
+                  <strong>{{ alternative.inbound_schedule.carrier_name }} {{ alternative.inbound_schedule.flight_number }}</strong>
+                  <span>현지 출발 예상 {{ alternative.inbound_schedule.estimated_destination_event_label }} · 인천 도착 {{ alternative.inbound_schedule.icn_event_label }}</span>
+                </dd>
               </div>
             </dl>
           </li>
@@ -79,14 +91,20 @@
   <aside class="calculation-note" aria-label="예상 체류시간 계산 기준">
     <h4>계산 기준</h4>
     <p>{{ opportunity.calculation_basis.notice }}</p>
-    <p>공식 운항 일정 {{ opportunity.calculation_basis.schedule_source_date|default:"날짜 없음" }} · 비행시간 참고자료 {{ opportunity.calculation_basis.duration_reference_date|default:"날짜 없음" }}</p>
+    <dl>
+      <div><dt>운항 자료일</dt><dd>{{ opportunity.calculation_basis.schedule_source_date|default:"날짜 없음" }}</dd></div>
+      <div><dt>비행시간 참고일</dt><dd>{{ opportunity.calculation_basis.duration_reference_date|default:"날짜 없음" }}</dd></div>
+    </dl>
   </aside>
 
   <section class="travel-briefs" aria-labelledby="destination-{{ opportunity.rank }}-briefs">
-    <h4 id="destination-{{ opportunity.rank }}-briefs">여행 전 확인</h4>
+    <div class="section-heading-row">
+      <h4 id="destination-{{ opportunity.rank }}-briefs">입국과 여행 정보</h4>
+      <p>각 정보는 서로 독립적으로 확인합니다</p>
+    </div>
     <p class="travel-briefs-intro">
-      아래 정보는 추천 계산과 별도로 검수·게시됩니다. 실제 입국 여부와
-      여행일 적용성은 공식 기관에서 다시 확인해 주세요.
+      추천 계산과 별도로 확인한 공식 출처 사실입니다. 실제 입국 여부와 여행일
+      적용성은 출발 전 공식 기관에서 다시 확인해 주세요.
     </p>
     {% include "public_web/partials/entry_card.html" with entry_card=opportunity.entry_card %}
     {% include "public_web/partials/warning_card.html" with warning_card=opportunity.warning_card %}
diff --git a/public_web/templates/public_web/partials/search_results.html b/public_web/templates/public_web/partials/search_results.html
index 595b653..faa4c46 100644
--- a/public_web/templates/public_web/partials/search_results.html
+++ b/public_web/templates/public_web/partials/search_results.html
@@ -1,14 +1,20 @@
 <section id="recommendations" class="results-section" data-state="{{ flight_state }}"
          aria-labelledby="results-heading">
   <header class="results-header">
-    <p class="section-kicker">검색 결과</p>
-    {% if opportunities %}
-      <h2 id="results-heading">갈 수 있는 도시를 찾았습니다</h2>
-    {% elif flight_state == "ready" or flight_state == "stale" %}
-      <h2 id="results-heading">이 시간 안에 안내할 수 있는 도시를 찾지 못했습니다</h2>
-    {% else %}
-      <h2 id="results-heading">찾은 결과를 확인해 주세요</h2>
-    {% endif %}
+    <div class="results-title-row">
+      <div>
+        <p class="section-kicker">검색 결과 · ICN</p>
+        {% if opportunities %}
+          <h2 id="results-heading">갈 수 있는 도시를 찾았습니다</h2>
+          <p class="results-count">현재 조건으로 비교할 수 있는 도시 {{ opportunities|length }}곳</p>
+        {% elif flight_state == "ready" or flight_state == "stale" %}
+          <h2 id="results-heading">현재 조건으로 표시할 수 있는 도시가 없습니다</h2>
+        {% else %}
+          <h2 id="results-heading">운항 자료 상태를 확인해 주세요</h2>
+        {% endif %}
+      </div>
+      <a class="edit-search" href="#trip-search">시간 다시 입력하기</a>
+    </div>
     <dl class="search-window" aria-label="입력한 여행 가능 시간">
       <div>
         <dt>출발 가능</dt>
@@ -20,36 +26,74 @@
       </div>
     </dl>
     <div class="flight-status" role="status" aria-live="polite">
-      <p><strong>운항 자료 상태: {{ flight_status_label }}</strong></p>
+      <p class="flight-status-heading"><span>운항 자료</span><strong>상태: {{ flight_status_label }}</strong></p>
       <p>{{ flight_message }}</p>
     </div>
     {% if not opportunities %}
       {% if flight_state == "ready" or flight_state == "stale" %}
-        <div class="service-boundary">
+        <div class="empty-guidance">
           <p>
-            현지에서 40시간 이상 머물 수 있는 직항 일정이 없습니다.<br>
-            출발 가능한 시각을 앞당기거나 인천 도착 마감 시각을 늦춰 다시 찾아보세요.
+            현재 게시된 운항·여행 정보와 입력한 시간 조건을 모두 만족해 표시할 수 있는
+            도시가 없습니다. 출발 가능한 시각을 앞당기거나 인천 도착 마감 시각을 늦춰
+            다시 찾아보세요.
           </p>
         </div>
       {% endif %}
     {% endif %}
     {% if flight_source_locator %}
-      <p class="flight-source">
-        운항 일정 마지막 확인 {{ flight_checked_at|default:"확인시각 없음" }} ·
-        운항 자료 리비전 {{ flight_publication_revision|default:"없음" }}<br>
-        <a href="{{ flight_source_locator }}" rel="noopener noreferrer">운항 일정 공식 출처 열기</a>
-        {% if flight_source_attribution %}<span> — {{ flight_source_attribution }}</span>{% endif %}
-      </p>
+      <details class="flight-source">
+        <summary>운항 자료 확인 정보</summary>
+        <p>마지막 성공 확인 {{ flight_checked_at|default:"확인시각 없음" }} · 정보 버전 {{ flight_publication_revision|default:"없음" }}</p>
+        <p>
+          <a href="{{ flight_source_locator }}" rel="noopener noreferrer">운항 일정 공식 출처 열기</a>
+          {% if flight_source_attribution %}<span> — {{ flight_source_attribution }}</span>{% endif %}
+        </p>
+      </details>
     {% endif %}
   </header>
 
   {% if opportunities %}
-    <ol class="opportunity-list" aria-label="추천 도시">
-      {% for opportunity in opportunities %}
-        <li>
-          {% include "public_web/partials/opportunity.html" %}
-        </li>
-      {% endfor %}
-    </ol>
+    <div class="results-layout">
+      <nav class="destination-index" aria-labelledby="destination-index-heading">
+        <div class="destination-index-heading">
+          <p class="index-kicker">추천 순서</p>
+          <h3 id="destination-index-heading">도시 한눈에 비교</h3>
+        </div>
+        <ol>
+          {% for opportunity in opportunities %}
+            <li>
+              <a href="#destination-{{ opportunity.rank }}">
+                <span class="index-rank" aria-hidden="true">{{ opportunity.rank|stringformat:"02d" }}</span>
+                <span class="index-place">
+                  <strong>{{ opportunity.destination.city_name }}</strong>
+                  <small>{{ opportunity.destination.country_name }} · {{ opportunity.destination.airport_code }}</small>
+                  <span class="index-module-states">입국요건 {{ opportunity.entry_card.status_label }} · 여행경보 {{ opportunity.warning_card.status_label }}</span>
+                </span>
+                <span class="index-flight">
+                  <small>가는 편 · {{ opportunity.outbound_schedule.flight_number }}</small>
+                  <strong>{{ opportunity.outbound_schedule.icn_event_label }}</strong>
+                </span>
+                <span class="index-flight">
+                  <small>오는 편 · {{ opportunity.inbound_schedule.flight_number }}</small>
+                  <strong>{{ opportunity.inbound_schedule.icn_event_label }}</strong>
+                </span>
+                <span class="index-stay">
+                  <small>예상 체류</small>
+                  <strong>{{ opportunity.estimated_local_stay_label }}</strong>
+                </span>
+              </a>
+            </li>
+          {% endfor %}
+        </ol>
+      </nav>
+
+      <ol class="opportunity-list" aria-label="추천 도시">
+        {% for opportunity in opportunities %}
+          <li>
+            {% include "public_web/partials/opportunity.html" %}
+          </li>
+        {% endfor %}
+      </ol>
+    </div>
   {% endif %}
 </section>
diff --git a/public_web/tests/test_accessibility_contract.py b/public_web/tests/test_accessibility_contract.py
index 74ec79c..6944dc4 100644
--- a/public_web/tests/test_accessibility_contract.py
+++ b/public_web/tests/test_accessibility_contract.py
@@ -240,7 +240,13 @@ class AccessiblePresentationContractTests(SimpleTestCase):
         self.assertContains(response, "예상 현지 체류시간")
         self.assertContains(response, "51시간 35분")
         self.assertContains(response, "공식 운항 일정")
-        self.assertContains(response, "제품 예상")
+        self.assertContains(response, "예상 시각")
+        self.assertContains(
+            response,
+            'nav class="destination-index" aria-labelledby="destination-index-heading"',
+        )
+        self.assertContains(response, 'href="#destination-1"')
+        self.assertContains(response, "도시 한눈에 비교")
         self.assertContains(response, 'id="destination-1-entry"')
         self.assertContains(response, 'data-module="entry" data-state="ready"')
         self.assertContains(response, 'id="destination-1-warning"')
@@ -297,11 +303,11 @@ class AccessiblePresentationContractTests(SimpleTestCase):
         self.assertEqual(response.status_code, 200)
         self.assertContains(
             response,
-            "이 시간 안에 안내할 수 있는 도시를 찾지 못했습니다",
+            "현재 조건으로 표시할 수 있는 도시가 없습니다",
         )
         self.assertContains(
             response,
-            "현지에서 40시간 이상 머물 수 있는 직항 일정이 없습니다.",
+            "현재 게시된 운항·여행 정보와 입력한 시간 조건을 모두 만족해 표시할 수 있는",
         )
         self.assertContains(
             response,
@@ -329,7 +335,9 @@ class AccessiblePresentationContractTests(SimpleTestCase):
             "flex-wrap: wrap",
             ".opportunity-list {",
             ".publication-brief {",
-            "max-width: var(--reading-width)",
+            ".results-layout {",
+            ".destination-index {",
+            "max-width: 52rem",
             "@media (max-width: 30rem)",
             "@media (max-width: 52rem)",
             "@media (pointer: coarse)",


