## `feat(frontend): refine itinerary travel briefs`

diff --git a/e2e/browser_acceptance.py b/e2e/browser_acceptance.py
index c528a0c..b43b6a6 100644
--- a/e2e/browser_acceptance.py
+++ b/e2e/browser_acceptance.py
@@ -185,8 +185,8 @@ WARNING_PUBLIC_SOURCE_LOCATOR: Final = (
 )
 ENTRY_FRESHNESS_MINUTES: Final = 36 * 60
 WARNING_FRESHNESS_MINUTES: Final = 8 * 60
-SITE_CSS_SHA256: Final = "2d1dcfb4ce3e2d908ac7a5f8f0f314c624a6a8484b6c986a2e6c314d0cd249aa"
-SITE_CSS_BYTES: Final = 28_687
+SITE_CSS_SHA256: Final = "9651b8ee4bfb220189693b8bc5fb473427070e28df46299797dfe65a8aa36e5f"
+SITE_CSS_BYTES: Final = 29_295
 SITE_JS_SHA256: Final = "1a5a95b286d6e12f1765351d5a0b1528c833fb46af16d747b469b61323cc7f44"
 SITE_JS_BYTES: Final = 1_628
 MARU_BURI_SHA256: Final = "5c8b39b683595d0ddcf2554148f4d2fb14c55cb967e5bff2e282b2936034fc75"
diff --git a/e2e/browser_scenario_server.py b/e2e/browser_scenario_server.py
index 88a4c7e..c2d256d 100644
--- a/e2e/browser_scenario_server.py
+++ b/e2e/browser_scenario_server.py
@@ -57,8 +57,8 @@ SITE_ASSETS: Final = {
     "/static/public_web/site.css": (
         REPOSITORY_ROOT / "public_web" / "static" / "public_web" / "site.css",
         "text/css",
-        28_687,
-        "2d1dcfb4ce3e2d908ac7a5f8f0f314c624a6a8484b6c986a2e6c314d0cd249aa",
+        29_295,
+        "9651b8ee4bfb220189693b8bc5fb473427070e28df46299797dfe65a8aa36e5f",
     ),
     "/static/public_web/site.js": (
         REPOSITORY_ROOT / "public_web" / "static" / "public_web" / "site.js",
diff --git a/public_web/static/public_web/site.css b/public_web/static/public_web/site.css
index d80ae83..64d7843 100644
--- a/public_web/static/public_web/site.css
+++ b/public_web/static/public_web/site.css
@@ -1248,9 +1248,21 @@ h3 {
   justify-content: space-between;
 }
 
+.publication-brief-header > div {
+  min-width: 0;
+}
+
+.brief-kind {
+  margin: 0 0 var(--space-1);
+  color: var(--muted-ink);
+  font-size: 0.6875rem;
+  font-weight: 700;
+  letter-spacing: 0.08em;
+}
+
 .publication-brief h5 {
   margin: 0;
-  font-size: 1.2rem;
+  font-size: 1.35rem;
 }
 
 .publication-status,
@@ -1262,6 +1274,19 @@ h3 {
   font-size: 0.875rem;
 }
 
+.publication-brief[data-state="ready"] .publication-status {
+  color: var(--current);
+}
+
+.publication-brief[data-state="unavailable"] .publication-status,
+.publication-brief[data-state="stale"] .publication-status {
+  color: var(--stale);
+}
+
+.publication-brief[data-state="server-error"] .publication-status {
+  color: var(--error);
+}
+
 .publication-message {
   margin-top: var(--space-2);
   color: var(--muted-ink);
@@ -1346,6 +1371,11 @@ h3 {
   font-weight: 800;
 }
 
+.source-link::after {
+  margin-left: var(--space-2);
+  content: "↗";
+}
+
 .site-footer {
   padding-block: var(--space-6);
   color: white;
diff --git a/public_web/templates/public_web/partials/entry_card.html b/public_web/templates/public_web/partials/entry_card.html
index a9095f4..37f7682 100644
--- a/public_web/templates/public_web/partials/entry_card.html
+++ b/public_web/templates/public_web/partials/entry_card.html
@@ -2,7 +2,10 @@
          data-module="entry" data-state="{{ entry_card.state }}" class="publication-brief"
          aria-labelledby="{% if opportunity %}destination-{{ opportunity.rank }}-entry-heading{% else %}entry-heading{% endif %}">
   <header class="publication-brief-header">
-    <h5 id="{% if opportunity %}destination-{{ opportunity.rank }}-entry-heading{% else %}entry-heading{% endif %}">{{ entry_card.heading|default:"입국요건" }}</h5>
+    <div>
+      <p class="brief-kind">공식 출처 사실</p>
+      <h5 id="{% if opportunity %}destination-{{ opportunity.rank }}-entry-heading{% else %}entry-heading{% endif %}">{{ entry_card.heading|default:"입국요건" }}</h5>
+    </div>
     <p class="publication-status" role="status"><strong>상태: {{ entry_card.status_label }}</strong></p>
   </header>
   <p class="publication-message">{{ entry_card.message }}</p>
@@ -17,12 +20,13 @@
       <dl>
         <div><dt>대상 국가</dt><dd>{{ entry_card.country_name }}</dd></div>
         <div><dt>마지막 성공 확인시각</dt><dd>{{ entry_card.checked_at|default:"확인 필요" }}</dd></div>
-        <div><dt>게시 리비전</dt><dd>{{ entry_card.generation }} · {{ entry_card.published_at }}</dd></div>
-        <div><dt>출처 리비전</dt><dd>{{ entry_card.source_revision }}</dd></div>
+        <div><dt>정보 버전</dt><dd>{{ entry_card.generation }}</dd></div>
+        <div><dt>정보 게시시각</dt><dd>{{ entry_card.published_at }}</dd></div>
+        <div><dt>출처 버전</dt><dd>{{ entry_card.source_revision }}</dd></div>
         <div><dt>출처</dt><dd>{{ entry_card.source_owner }} · {{ entry_card.attribution }}</dd></div>
       </dl>
     </details>
-    <p class="verification-note"><strong>확인 필요:</strong> 여행 목적·날짜에 따른 적용성과 최신 조건</p>
+    <p class="verification-note"><strong>출발 전 확인:</strong> 여행 목적·날짜에 따른 적용성과 최신 조건</p>
     <a class="source-link" href="{{ entry_card.source_locator }}"
        rel="noopener noreferrer">입국요건 공식 출처 열기</a>
   {% endif %}
diff --git a/public_web/templates/public_web/partials/warning_card.html b/public_web/templates/public_web/partials/warning_card.html
index e751e61..e7d2af4 100644
--- a/public_web/templates/public_web/partials/warning_card.html
+++ b/public_web/templates/public_web/partials/warning_card.html
@@ -2,7 +2,10 @@
          data-module="warning" data-state="{{ warning_card.state }}" class="publication-brief"
          aria-labelledby="{% if opportunity %}destination-{{ opportunity.rank }}-warning-heading{% else %}warning-heading{% endif %}">
   <header class="publication-brief-header">
-    <h5 id="{% if opportunity %}destination-{{ opportunity.rank }}-warning-heading{% else %}warning-heading{% endif %}">{{ warning_card.heading|default:"여행경보" }}</h5>
+    <div>
+      <p class="brief-kind">공식 출처 사실</p>
+      <h5 id="{% if opportunity %}destination-{{ opportunity.rank }}-warning-heading{% else %}warning-heading{% endif %}">{{ warning_card.heading|default:"여행경보" }}</h5>
+    </div>
     <p class="publication-status" role="status"><strong>상태: {{ warning_card.status_label }}</strong></p>
   </header>
   <p class="publication-message">{{ warning_card.message }}</p>
@@ -35,12 +38,13 @@
       <dl>
         <div><dt>대상 국가</dt><dd>{{ warning_card.country_name }}</dd></div>
         <div><dt>마지막 성공 확인시각</dt><dd>{{ warning_card.checked_at|default:"확인 필요" }}</dd></div>
-        <div><dt>게시 리비전</dt><dd>{{ warning_card.generation }} · {{ warning_card.published_at }}</dd></div>
-        <div><dt>출처 리비전</dt><dd>{{ warning_card.source_revision }}</dd></div>
+        <div><dt>정보 버전</dt><dd>{{ warning_card.generation }}</dd></div>
+        <div><dt>정보 게시시각</dt><dd>{{ warning_card.published_at }}</dd></div>
+        <div><dt>출처 버전</dt><dd>{{ warning_card.source_revision }}</dd></div>
         <div><dt>출처</dt><dd>{{ warning_card.source_owner }} · {{ warning_card.attribution }}</dd></div>
       </dl>
     </details>
-    <p class="verification-note"><strong>확인 필요:</strong> 단계 명칭, 발효·종료 시각과 여행일 적용성</p>
+    <p class="verification-note"><strong>출발 전 확인:</strong> 단계 명칭, 발효·종료 시각과 여행일 적용성</p>
     <a class="source-link" href="{{ warning_card.source_locator }}"
        rel="noopener noreferrer">여행경보 공식 출처 열기</a>
   {% endif %}


## `fix(frontend): reveal results after search`

diff --git a/e2e/browser_acceptance.py b/e2e/browser_acceptance.py
index 3d95ac3..2d01af9 100644
--- a/e2e/browser_acceptance.py
+++ b/e2e/browser_acceptance.py
@@ -185,10 +185,10 @@ WARNING_PUBLIC_SOURCE_LOCATOR: Final = (
 )
 ENTRY_FRESHNESS_MINUTES: Final = 36 * 60
 WARNING_FRESHNESS_MINUTES: Final = 8 * 60
-SITE_CSS_SHA256: Final = "1b804c5e8f5a670cec0917a734ca79c6d61417ea2c2a44a67e9d328a093498ee"
-SITE_CSS_BYTES: Final = 30_305
-SITE_JS_SHA256: Final = "1a5a95b286d6e12f1765351d5a0b1528c833fb46af16d747b469b61323cc7f44"
-SITE_JS_BYTES: Final = 1_628
+SITE_CSS_SHA256: Final = "71cd8e9842d32eb74f9bd281dfb40663fbadcef143e9fce3b3fff17a4f9556b3"
+SITE_CSS_BYTES: Final = 30_441
+SITE_JS_SHA256: Final = "928e5388a63fe3c8bf1c13d8156e1b336d6ab7708533c0372013518fe27f7611"
+SITE_JS_BYTES: Final = 2_096
 WANTED_SANS_VARIABLE_SHA256: Final = "4259e7e9a172e634c2cb419d793b84148990316341e910443e5d10965b2c8f16"
 WANTED_SANS_VARIABLE_BYTES: Final = 1_289_292
 _SIGNAL_INTERRUPTED = False
@@ -3651,6 +3651,11 @@ def _focused_result_javascript(
   {submission}
   const result = page.locator('#recommendations[data-state={json.dumps(state)}]');
   if (await result.count() !== 1) fail('result-state');
+  await page.waitForFunction(() => document.activeElement?.id === 'results-heading');
+  const resultHeading = page.locator('#results-heading[data-results-heading][tabindex="-1"]');
+  if (await resultHeading.count() !== 1) fail('result-focus-handoff');
+  const resultHeadingTop = await resultHeading.evaluate((node) => node.getBoundingClientRect().top);
+  if (resultHeadingTop < 0 || resultHeadingTop > 96) fail('result-focus-handoff');
   const opportunities = page.locator('.opportunity');
   const count = await opportunities.count();
   if ({str(expect_opportunities).lower()} ? !(count >= 1 && count <= 6) : count !== 0) fail('opportunity-count');
diff --git a/e2e/browser_scenario_server.py b/e2e/browser_scenario_server.py
index c7bdfb9..5216665 100644
--- a/e2e/browser_scenario_server.py
+++ b/e2e/browser_scenario_server.py
@@ -57,14 +57,14 @@ SITE_ASSETS: Final = {
     "/static/public_web/site.css": (
         REPOSITORY_ROOT / "public_web" / "static" / "public_web" / "site.css",
         "text/css",
-        30_305,
-        "1b804c5e8f5a670cec0917a734ca79c6d61417ea2c2a44a67e9d328a093498ee",
+        30_441,
+        "71cd8e9842d32eb74f9bd281dfb40663fbadcef143e9fce3b3fff17a4f9556b3",
     ),
     "/static/public_web/site.js": (
         REPOSITORY_ROOT / "public_web" / "static" / "public_web" / "site.js",
         "text/javascript",
-        1_628,
-        "1a5a95b286d6e12f1765351d5a0b1528c833fb46af16d747b469b61323cc7f44",
+        2_096,
+        "928e5388a63fe3c8bf1c13d8156e1b336d6ab7708533c0372013518fe27f7611",
     ),
     "/static/public_web/fonts/wanted-sans-variable.woff2": (
         REPOSITORY_ROOT
diff --git a/operations/tests/test_browser_acceptance_harness.py b/operations/tests/test_browser_acceptance_harness.py
index 338102e..ae1bfd2 100644
--- a/operations/tests/test_browser_acceptance_harness.py
+++ b/operations/tests/test_browser_acceptance_harness.py
@@ -1125,6 +1125,7 @@ class BrowserAcceptanceHarnessTests(unittest.TestCase):
         self.assertIn("verified-empty-warning", result)
         self.assertIn("warning-fact-count", result)
         self.assertIn("warning-fact-order", result)
+        self.assertIn("result-focus-handoff", result)
         self.assertIn(json.dumps(acceptance.FOCUSED_LONG_AIRPORT), accessibility)
         self.assertIn(
             json.dumps(acceptance.FOCUSED_LONG_ENTRY_BASIS), accessibility
diff --git a/public_web/static/public_web/site.css b/public_web/static/public_web/site.css
index 738a6ca..2029b64 100644
--- a/public_web/static/public_web/site.css
+++ b/public_web/static/public_web/site.css
@@ -573,6 +573,12 @@ h3 {
 .results-header h2 {
   max-width: 18ch;
   font-size: clamp(2.05rem, 4.5vw, 3.65rem);
+  scroll-margin-top: var(--space-6);
+}
+
+.results-header h2:focus {
+  outline: 0.1875rem solid var(--focus);
+  outline-offset: 0.375rem;
 }
 
 .results-header h2 .heading-translation {
diff --git a/public_web/static/public_web/site.js b/public_web/static/public_web/site.js
index c2f2a6a..e7e0ef0 100644
--- a/public_web/static/public_web/site.js
+++ b/public_web/static/public_web/site.js
@@ -44,4 +44,16 @@
   if (errorSummary) {
     window.requestAnimationFrame(() => errorSummary.focus());
   }
+
+  const resultsHeading = document.querySelector("[data-results-heading]");
+  if (resultsHeading) {
+    window.requestAnimationFrame(() => {
+      const root = document.documentElement;
+      const inlineScrollBehavior = root.style.scrollBehavior;
+      root.style.scrollBehavior = "auto";
+      resultsHeading.focus({ preventScroll: true });
+      resultsHeading.scrollIntoView({ block: "start" });
+      root.style.scrollBehavior = inlineScrollBehavior;
+    });
+  }
 })();
diff --git a/public_web/templates/public_web/partials/search_results.html b/public_web/templates/public_web/partials/search_results.html
index 947e6be..dccfa37 100644
--- a/public_web/templates/public_web/partials/search_results.html
+++ b/public_web/templates/public_web/partials/search_results.html
@@ -5,16 +5,17 @@
       <div>
         <p class="section-kicker" lang="en">SEARCH RESULTS · ICN</p>
         {% if opportunities %}
-          <h2 id="results-heading" class="bilingual-heading"
+          <h2 id="results-heading" class="bilingual-heading" tabindex="-1"
+              data-results-heading
               aria-label="갈 수 있는 도시를 찾았습니다">
             <span lang="en">Destinations within reach</span>
             <span class="heading-translation" aria-hidden="true">갈 수 있는 도시를 찾았습니다</span>
           </h2>
           <p class="results-count">현재 조건으로 비교할 수 있는 도시 {{ opportunities|length }}곳</p>
         {% elif flight_state == "ready" or flight_state == "stale" %}
-          <h2 id="results-heading">현재 조건으로 표시할 수 있는 도시가 없습니다</h2>
+          <h2 id="results-heading" tabindex="-1" data-results-heading>현재 조건으로 표시할 수 있는 도시가 없습니다</h2>
         {% else %}
-          <h2 id="results-heading">운항 자료 상태를 확인해 주세요</h2>
+          <h2 id="results-heading" tabindex="-1" data-results-heading>운항 자료 상태를 확인해 주세요</h2>
         {% endif %}
       </div>
       <a class="edit-search" href="#trip-search">시간 다시 입력하기</a>
diff --git a/public_web/tests/test_accessibility_contract.py b/public_web/tests/test_accessibility_contract.py
index f47a4f6..1c4db15 100644
--- a/public_web/tests/test_accessibility_contract.py
+++ b/public_web/tests/test_accessibility_contract.py
@@ -254,6 +254,17 @@ class AccessiblePresentationContractTests(SimpleTestCase):
             response,
             '<span lang="en">Destinations within reach</span>',
         )
+        self.assertContains(response, 'tabindex="-1"')
+        self.assertContains(response, "data-results-heading")
+        self.assertIn(
+            'resultsHeading.focus({ preventScroll: true })',
+            self.javascript,
+        )
+        self.assertIn(
+            'resultsHeading.scrollIntoView({ block: "start" })',
+            self.javascript,
+        )
+        self.assertIn(".results-header h2:focus {", self.css)
         self.assertContains(
             response,
             'nav class="destination-index" aria-labelledby="destination-index-heading"',
