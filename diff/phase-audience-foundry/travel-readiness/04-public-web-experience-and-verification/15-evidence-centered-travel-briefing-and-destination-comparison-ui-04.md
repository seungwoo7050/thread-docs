## `feat(frontend): redesign entry publication briefing`

diff --git a/e2e/browser_acceptance.py b/e2e/browser_acceptance.py
index 15f3cb5..037a4a1 100644
--- a/e2e/browser_acceptance.py
+++ b/e2e/browser_acceptance.py
@@ -173,8 +173,8 @@ WARNING_PUBLIC_SOURCE_LOCATOR: Final = (
 )
 ENTRY_FRESHNESS_MINUTES: Final = 36 * 60
 WARNING_FRESHNESS_MINUTES: Final = 8 * 60
-SITE_CSS_SHA256: Final = "5d878aba398b9692bff17096d4f53f9197cc4b3a149f6da797d6f10f2b1d3aca"
-SITE_CSS_BYTES: Final = 9_174
+SITE_CSS_SHA256: Final = "2c939acf9506c27d296a7c0d881d2c9b5ab737468596e086e0779052cd7462c1"
+SITE_CSS_BYTES: Final = 9_382
 SITE_JS_SHA256: Final = "b890036fb3ce127e71f7e71c3af93209c1b381078f6896a627ec59b8ab9b217e"
 SITE_JS_BYTES: Final = 1_620
 _SIGNAL_INTERRUPTED = False
diff --git a/e2e/browser_scenario_server.py b/e2e/browser_scenario_server.py
index 6ed22b0..37b6d18 100644
--- a/e2e/browser_scenario_server.py
+++ b/e2e/browser_scenario_server.py
@@ -49,8 +49,8 @@ SITE_ASSETS: Final = {
     "/static/public_web/site.css": (
         REPOSITORY_ROOT / "public_web" / "static" / "public_web" / "site.css",
         "text/css",
-        9_174,
-        "5d878aba398b9692bff17096d4f53f9197cc4b3a149f6da797d6f10f2b1d3aca",
+        9_382,
+        "2c939acf9506c27d296a7c0d881d2c9b5ab737468596e086e0779052cd7462c1",
     ),
     "/static/public_web/site.js": (
         REPOSITORY_ROOT / "public_web" / "static" / "public_web" / "site.js",
@@ -252,8 +252,13 @@ def build_scenario_cards(
                     "state": "stale",
                     "status_label": "재확인 필요",
                     "message": (
-                        "마지막 검수·게시 사실입니다. 더 최근 조회 또는 "
-                        "source 상태를 재확인해 주세요."
+                        "마지막으로 검수·게시된 입국요건 사실입니다. "
+                        "공식 출처에서 최신 정보를 재확인해 주세요."
+                        if module == "entry"
+                        else (
+                            "마지막 검수·게시 사실입니다. 더 최근 조회 또는 "
+                            "source 상태를 재확인해 주세요."
+                        )
                     ),
                     "checked_at": _stale_checked_at(module, observed),
                 }
diff --git a/operations/tests/test_browser_scenario_servers.py b/operations/tests/test_browser_scenario_servers.py
index 0d3ee8e..2b6b886 100644
--- a/operations/tests/test_browser_scenario_servers.py
+++ b/operations/tests/test_browser_scenario_servers.py
@@ -37,10 +37,10 @@ def ready_cards() -> dict[str, dict[str, object]]:
     return {
         "entry": {
             "module": "entry",
-            "heading": "입국요건 사실",
+            "heading": "입국요건",
             "state": "ready",
-            "status_label": "게시된 source 사실",
-            "message": "공식 source의 검수·게시 사실입니다.",
+            "status_label": "현재 게시본",
+            "message": "공식 출처에서 수집해 검수·게시한 입국요건 사실입니다.",
             "has_publication": True,
             "country_name": "일본",
             "generation": 1,
@@ -51,7 +51,7 @@ def ready_cards() -> dict[str, dict[str, object]]:
             "attribution": ENTRY_ATTRIBUTION,
             "checked_at": "2026-08-30 00:00 UTC",
             "period_text": "90일",
-            "basis_text": "일반여권 소지자: 실제 게시 source 근거",
+            "basis_text": "일반여권 소지자: 실제 게시 출처 근거",
             "snapshot_date": "2025-07-11",
         },
         "warning": {
@@ -132,6 +132,8 @@ class BrowserScenarioCardTests(SimpleTestCase):
         )
         self.assertEqual(stale["entry"]["checked_at"], "2026-08-29 23:00 UTC")
         self.assertEqual(stale["warning"]["checked_at"], "2026-08-31 03:00 UTC")
+        self.assertIn("공식 출처", stale["entry"]["message"])
+        self.assertNotIn("source", stale["entry"]["message"])
         long_cards = scenario_server.build_scenario_cards("long-korean", load)
         self.assertGreaterEqual(
             scenario_server._hangul_count(long_cards["entry"]["basis_text"]),
diff --git a/public_web/results.py b/public_web/results.py
index 3598165..b9548d1 100644
--- a/public_web/results.py
+++ b/public_web/results.py
@@ -118,21 +118,38 @@ def _fixed_redirect(route_name: str) -> HttpResponse:
 
 def _state_card(module: str, state: str) -> dict[str, object]:
     is_entry = module == "entry"
-    heading = "입국요건 사실" if is_entry else "여행경보"
-    labels = {
-        CARD_EMPTY: (
-            "게시 전",
-            "아직 검수·게시된 source 사실이 없습니다. 공식 source 확인이 필요합니다.",
-        ),
-        CARD_UNAVAILABLE: (
-            "정보 확인 필요",
-            "게시 경계를 확인할 수 없습니다. 공식 source에서 직접 확인해 주세요.",
-        ),
-        CARD_SERVER_ERROR: (
-            "일시적 오류",
-            "이 정보를 지금 읽을 수 없습니다. 다른 카드는 계속 확인할 수 있습니다.",
-        ),
-    }
+    heading = "입국요건" if is_entry else "여행경보"
+    labels = (
+        {
+            CARD_EMPTY: (
+                "게시된 자료 없음",
+                "아직 게시된 입국요건 자료가 없습니다. 입국요건 공식 출처에서 확인해 주세요.",
+            ),
+            CARD_UNAVAILABLE: (
+                "현재 확인할 수 없음",
+                "입국요건 게시 정보를 현재 확인할 수 없습니다. 입국요건 공식 출처에서 직접 확인해 주세요.",
+            ),
+            CARD_SERVER_ERROR: (
+                "일시적 오류",
+                "입국요건을 지금 읽을 수 없습니다. 여행경보는 다른 구역에서 계속 확인할 수 있습니다.",
+            ),
+        }
+        if is_entry
+        else {
+            CARD_EMPTY: (
+                "게시 전",
+                "아직 검수·게시된 source 사실이 없습니다. 공식 source 확인이 필요합니다.",
+            ),
+            CARD_UNAVAILABLE: (
+                "정보 확인 필요",
+                "게시 경계를 확인할 수 없습니다. 공식 source에서 직접 확인해 주세요.",
+            ),
+            CARD_SERVER_ERROR: (
+                "일시적 오류",
+                "이 정보를 지금 읽을 수 없습니다. 다른 카드는 계속 확인할 수 있습니다.",
+            ),
+        }
+    )
     status_label, message = labels[state]
     return {
         "module": module,
@@ -433,13 +450,13 @@ def _load_entry_card() -> dict[str, object]:
     stale = state == CARD_STALE
     return {
         "module": "entry",
-        "heading": "입국요건 사실",
+        "heading": "입국요건",
         "state": state,
-        "status_label": "재확인 필요" if stale else "게시된 source 사실",
+        "status_label": "재확인 필요" if stale else "현재 게시본",
         "message": (
-            "마지막 검수·게시 사실입니다. 더 최근 조회 또는 source 상태를 재확인해 주세요."
+            "마지막으로 검수·게시된 입국요건 사실입니다. 공식 출처에서 최신 정보를 재확인해 주세요."
             if stale
-            else "공식 source의 검수·게시 사실입니다."
+            else "공식 출처에서 수집해 검수·게시한 입국요건 사실입니다."
         ),
         "has_publication": True,
         "country_name": row["current_publication__scope_country__name_ko"],
diff --git a/public_web/static/public_web/site.css b/public_web/static/public_web/site.css
index b27b15e..41bb424 100644
--- a/public_web/static/public_web/site.css
+++ b/public_web/static/public_web/site.css
@@ -412,6 +412,20 @@ h2 {
   border-top: 0.0625rem solid var(--line);
 }
 
+.publication-section {
+  margin-top: var(--space-8);
+}
+
+.publication-section h3 {
+  margin: 0;
+  font-size: 1rem;
+  line-height: 1.4;
+}
+
+.publication-section .fact-list {
+  margin-block: var(--space-3) 0;
+}
+
 .fact-list dt,
 .fact-list dd {
   min-width: 0;
@@ -434,8 +448,8 @@ h2 {
   display: inline-flex;
   min-height: 44px;
   align-items: center;
-  margin-inline-start: 0.25rem;
-  padding-inline: 0.25rem;
+  margin-top: var(--space-4);
+  padding: var(--space-2) 0;
   color: var(--link);
   font-weight: 800;
 }
diff --git a/public_web/templates/public_web/partials/entry_card.html b/public_web/templates/public_web/partials/entry_card.html
index 3ca93c0..ccfd893 100644
--- a/public_web/templates/public_web/partials/entry_card.html
+++ b/public_web/templates/public_web/partials/entry_card.html
@@ -3,28 +3,31 @@
          aria-describedby="entry-status entry-message">
   <h2 id="entry-heading">{{ entry_card.heading }}</h2>
   <p class="status-line" id="entry-status" role="status">
-    <span class="status-symbol" aria-hidden="true"></span>
     <strong>상태: {{ entry_card.status_label }}</strong>
   </p>
   <p id="entry-message">{{ entry_card.message }}</p>
   {% if entry_card.has_publication %}
-    <dl class="fact-list">
-      <dt>국가</dt><dd>{{ entry_card.country_name }}</dd>
-      <dt>일반여권 source 표기</dt><dd>{{ entry_card.period_text }}</dd>
-      <dt>source 근거 문구</dt><dd>{{ entry_card.basis_text }}</dd>
-      <dt>snapshot date</dt><dd>{{ entry_card.snapshot_date }}</dd>
-      <dt>마지막 성공 확인시각</dt><dd>{{ entry_card.checked_at|default:"확인 필요" }}</dd>
-      <dt>publication revision</dt><dd>generation {{ entry_card.generation }}</dd>
-      <dt>게시시각</dt><dd>{{ entry_card.published_at }}</dd>
-      <dt>source revision</dt><dd>{{ entry_card.source_revision }}</dd>
-      <dt>출처</dt>
-      <dd>
-        {{ entry_card.source_owner }} · {{ entry_card.attribution }}
-        <a class="source-link" href="{{ entry_card.source_locator }}"
-           rel="noopener noreferrer"
-           aria-label="외교부 입국요건 source 열기">공식 source</a>
-      </dd>
-    </dl>
-    <p class="verification-note"><strong>확인 필요:</strong> 여행 목적·날짜 적용성과 최신 조건은 source에서 다시 확인해 주세요.</p>
+    <section class="publication-section">
+      <h3>출처에서 확인된 사실</h3>
+      <dl class="fact-list">
+        <dt>대상 국가</dt><dd>{{ entry_card.country_name }}</dd>
+        <dt>일반여권 관련 출처 표기</dt><dd>{{ entry_card.period_text }}</dd>
+        <dt>출처 근거 문구</dt><dd>{{ entry_card.basis_text }}</dd>
+        <dt>출처 자료 날짜</dt><dd>{{ entry_card.snapshot_date|default:"출처가 제공하지 않음" }}</dd>
+      </dl>
+    </section>
+    <section class="publication-section">
+      <h3>출처와 게시 이력</h3>
+      <dl class="fact-list">
+        <dt>마지막 성공 확인시각</dt><dd>{{ entry_card.checked_at|default:"확인 필요" }}</dd>
+        <dt>게시 리비전</dt><dd>generation {{ entry_card.generation }}</dd>
+        <dt>게시시각</dt><dd>{{ entry_card.published_at }}</dd>
+        <dt>출처 리비전</dt><dd>{{ entry_card.source_revision }}</dd>
+        <dt>출처</dt><dd>{{ entry_card.source_owner }} · {{ entry_card.attribution }}</dd>
+      </dl>
+    </section>
+    <p class="verification-note"><strong>확인 필요:</strong> 여행 목적·날짜에 따른 적용성과 최신 조건은 공식 출처에서 다시 확인해 주세요.</p>
+    <a class="source-link" href="{{ entry_card.source_locator }}"
+       rel="noopener noreferrer">입국요건 공식 출처 열기</a>
   {% endif %}
 </article>
diff --git a/public_web/tests/test_accessibility_contract.py b/public_web/tests/test_accessibility_contract.py
index e452ebc..c6692d6 100644
--- a/public_web/tests/test_accessibility_contract.py
+++ b/public_web/tests/test_accessibility_contract.py
@@ -103,9 +103,9 @@ class AccessiblePresentationContractTests(SimpleTestCase):
         cards = {
             "entry": {
                 "state": "ready",
-                "heading": "입국요건 사실",
-                "status_label": "게시된 source 사실",
-                "message": "공식 source의 검수·게시 사실입니다.",
+                "heading": "입국요건",
+                "status_label": "현재 게시본",
+                "message": "공식 출처에서 수집해 검수·게시한 입국요건 사실입니다.",
                 "has_publication": False,
             },
             "warning": {
@@ -138,9 +138,9 @@ class AccessiblePresentationContractTests(SimpleTestCase):
             response,
             'aria-describedby="warning-status warning-message"',
         )
-        self.assertContains(response, "상태: 게시된 source 사실")
+        self.assertContains(response, "상태: 현재 게시본")
         self.assertContains(response, "상태: 재확인 필요")
-        self.assertContains(response, 'aria-hidden="true"', count=2)
+        self.assertContains(response, 'aria-hidden="true"', count=1)
 
     def test_styles_encode_responsive_touch_focus_and_wrapping_contracts(self):
         for required in (
diff --git a/public_web/tests/test_results.py b/public_web/tests/test_results.py
index 610889e..c361b7c 100644
--- a/public_web/tests/test_results.py
+++ b/public_web/tests/test_results.py
@@ -128,7 +128,11 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
         self.assertEqual(response.status_code, 200)
         self.assertContains(response, 'id="entry-card" data-state="empty"')
         self.assertContains(response, 'id="warning-card" data-state="empty"')
-        self.assertContains(response, "아직 검수·게시된 source 사실이 없습니다", count=2)
+        entry_card = self.card_html(response, "entry-card")
+        warning_card = self.card_html(response, "warning-card")
+        self.assertIn("상태: 게시된 자료 없음", entry_card)
+        self.assertIn("아직 게시된 입국요건 자료가 없습니다", entry_card)
+        self.assertIn("아직 검수·게시된 source 사실이 없습니다", warning_card)
         self.assertNotIn("sessionid", response.cookies)
 
     def test_entry_ready_marks_only_unpublished_warning_unavailable(self):
@@ -139,6 +143,7 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
         entry_card = self.card_html(response, "entry-card")
         warning_card = self.card_html(response, "warning-card")
         self.assertIn('data-state="ready"', entry_card)
+        self.assertIn("상태: 현재 게시본", entry_card)
         self.assertIn("90일", entry_card)
         self.assertIn('data-state="unavailable"', warning_card)
         self.assertIn("게시 경계를 확인할 수 없습니다", warning_card)
@@ -151,7 +156,7 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
         entry_card = self.card_html(response, "entry-card")
         warning_card = self.card_html(response, "warning-card")
         self.assertIn('data-state="unavailable"', entry_card)
-        self.assertIn("게시 경계를 확인할 수 없습니다", entry_card)
+        self.assertIn("상태: 현재 확인할 수 없음", entry_card)
         self.assertIn('data-state="ready"', warning_card)
         self.assertIn("독립 경보 범위", warning_card)
 
@@ -164,18 +169,29 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
         self.assertEqual(response.status_code, 200)
         self.assertContains(response, 'id="entry-card" data-state="ready"')
         self.assertContains(response, 'id="warning-card" data-state="ready"')
-        self.assertContains(response, "일반여권 source 표기")
+        self.assertContains(response, "일반여권 관련 출처 표기")
         self.assertContains(response, "90일")
-        self.assertContains(response, "snapshot date")
+        self.assertContains(response, "출처 자료 날짜")
         self.assertContains(response, "2025-01-20")
         self.assertContains(response, "source 경보 단계 코드")
         self.assertContains(response, "긴 한국어 검증 범위와 &amp; 기호")
         self.assertContains(response, "source가 제공하지 않음")
         self.assertContains(response, "마지막 성공 확인시각", count=2)
-        self.assertContains(response, "publication revision", count=2)
+        self.assertContains(response, "게시 리비전")
+        self.assertContains(response, "publication revision")
         self.assertContains(response, "확인 필요")
         self.assertContains(response, "외교부|공공데이터포털", count=2)
         self.assertContains(response, 'rel="noopener noreferrer"', count=2)
+        self.assertContains(response, "입국요건 공식 출처 열기")
+
+        entry_card = self.card_html(response, "entry-card")
+        self.assertLess(entry_card.index("상태:"), entry_card.index("출처에서 확인된 사실"))
+        self.assertLess(
+            entry_card.index("출처에서 확인된 사실"),
+            entry_card.index("출처와 게시 이력"),
+        )
+        self.assertLess(entry_card.index("출처와 게시 이력"), entry_card.index("확인 필요:"))
+        self.assertLess(entry_card.index("확인 필요:"), entry_card.index("입국요건 공식 출처 열기"))
 
         forbidden = (
             "ALLOW" + "ED",
@@ -353,7 +369,7 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
         entry_card = self.card_html(response, "entry-card")
         warning_card = self.card_html(response, "warning-card")
         self.assertIn('data-state="unavailable"', entry_card)
-        self.assertIn("게시 경계를 확인할 수 없습니다", entry_card)
+        self.assertIn("현재 확인할 수 없음", entry_card)
         self.assertIn('data-state="stale"', warning_card)
         self.assertIn("재확인 필요", warning_card)
         self.assertIn("독립 경보 범위", warning_card)
@@ -431,7 +447,7 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
         response = self.client.get(reverse("public_web:results"))
 
         self.assertContains(response, 'id="entry-card" data-state="stale"')
-        self.assertContains(response, "source revision</dt><dd>rights-v1")
+        self.assertContains(response, "출처 리비전</dt><dd>rights-v1")
         self.assertContains(response, self.checked_at_text(original_checked_at))
         self.assertNotContains(response, self.checked_at_text(later_checked_at))
         self.assertContains(
@@ -543,8 +559,8 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
         self.assertIn("90일", entry_html)
         self.assertNotIn("30일", entry_html)
         self.assertIn("일반여권 소지자 : synthetic evidence", entry_html)
-        self.assertIn("publication revision</dt><dd>generation 3", entry_html)
-        self.assertIn("source revision</dt><dd>rights-v1", entry_html)
+        self.assertIn("게시 리비전</dt><dd>generation 3", entry_html)
+        self.assertIn("출처 리비전</dt><dd>rights-v1", entry_html)
         self.assertIn("외교부|공공데이터포털", entry_html)
         self.assertEqual(
             self.card_html(response, "warning-card"),


