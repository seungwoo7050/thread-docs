## `feat(frontend): redesign warning publication briefing`

diff --git a/e2e/browser_scenario_server.py b/e2e/browser_scenario_server.py
index 37b6d18..fa6a6f4 100644
--- a/e2e/browser_scenario_server.py
+++ b/e2e/browser_scenario_server.py
@@ -256,8 +256,8 @@ def build_scenario_cards(
                         "공식 출처에서 최신 정보를 재확인해 주세요."
                         if module == "entry"
                         else (
-                            "마지막 검수·게시 사실입니다. 더 최근 조회 또는 "
-                            "source 상태를 재확인해 주세요."
+                            "마지막으로 검수·게시된 여행경보입니다. "
+                            "공식 출처에서 최신 정보를 재확인해 주세요."
                         )
                     ),
                     "checked_at": _stale_checked_at(module, observed),
diff --git a/operations/management/commands/check_publication_rollback.py b/operations/management/commands/check_publication_rollback.py
index 1309335..e3118b4 100644
--- a/operations/management/commands/check_publication_rollback.py
+++ b/operations/management/commands/check_publication_rollback.py
@@ -412,8 +412,8 @@ def _assert_warning_card(
 ) -> None:
     _require(f'data-state="{state}"' in card)
     _require(f"generation {generation}" in card)
-    _require(f"source 경보 단계 코드</dt><dd>{_WARNING_LEVEL}" in card)
-    _require(f"source 범위 유형</dt><dd>{_WARNING_SCOPE_TYPE}" in card)
+    _require(f"출처 경보 단계 코드</dt><dd>{_WARNING_LEVEL}" in card)
+    _require(f"출처 범위 유형</dt><dd>{_WARNING_SCOPE_TYPE}" in card)
     _require(scope_text in card)
     _require("외교부|공공데이터포털" in card)
     if absent_text is not None:
@@ -514,7 +514,7 @@ def run_publication_rollback_rehearsal() -> str:
     _assert_entry_card(entry_card, generation=1, period="90일")
     _require(warning_card_after != warning_card_empty)
     _require('data-state="unavailable"' in warning_card_after)
-    _require("아직 검수·게시된 source 사실이 없습니다" not in warning_card_after)
+    _require("아직 게시된 여행경보 자료가 없습니다" not in warning_card_after)
     _require(_boundary(warning_pointer) == (None, 0))
     warning_card_unavailable = warning_card_after
 
diff --git a/operations/tests/test_browser_scenario_servers.py b/operations/tests/test_browser_scenario_servers.py
index 2b6b886..7344bff 100644
--- a/operations/tests/test_browser_scenario_servers.py
+++ b/operations/tests/test_browser_scenario_servers.py
@@ -58,10 +58,8 @@ def ready_cards() -> dict[str, dict[str, object]]:
             "module": "warning",
             "heading": "여행경보",
             "state": "ready",
-            "status_label": "게시된 source 사실",
-            "message": (
-                "입국요건과 독립된 공식 source의 검수·게시 사실입니다."
-            ),
+            "status_label": "현재 게시본",
+            "message": "입국요건과 별도로 공식 출처에서 수집해 검수·게시한 여행경보입니다.",
             "has_publication": True,
             "country_name": "일본",
             "generation": 1,
@@ -73,7 +71,7 @@ def ready_cards() -> dict[str, dict[str, object]]:
             "checked_at": "2026-08-30 00:00 UTC",
             "alarm_level_code": "1",
             "scope_type": "일부",
-            "scope_text": "실제 게시 source 범위",
+            "scope_text": "실제 게시 출처 범위",
             "written_date": "2025-07-11",
         },
     }
@@ -134,6 +132,8 @@ class BrowserScenarioCardTests(SimpleTestCase):
         self.assertEqual(stale["warning"]["checked_at"], "2026-08-31 03:00 UTC")
         self.assertIn("공식 출처", stale["entry"]["message"])
         self.assertNotIn("source", stale["entry"]["message"])
+        self.assertIn("공식 출처", stale["warning"]["message"])
+        self.assertNotIn("source", stale["warning"]["message"])
         long_cards = scenario_server.build_scenario_cards("long-korean", load)
         self.assertGreaterEqual(
             scenario_server._hangul_count(long_cards["entry"]["basis_text"]),
diff --git a/public_web/results.py b/public_web/results.py
index b9548d1..9e8552a 100644
--- a/public_web/results.py
+++ b/public_web/results.py
@@ -137,16 +137,16 @@ def _state_card(module: str, state: str) -> dict[str, object]:
         if is_entry
         else {
             CARD_EMPTY: (
-                "게시 전",
-                "아직 검수·게시된 source 사실이 없습니다. 공식 source 확인이 필요합니다.",
+                "게시된 자료 없음",
+                "아직 게시된 여행경보 자료가 없습니다. 여행경보 공식 출처에서 확인해 주세요.",
             ),
             CARD_UNAVAILABLE: (
-                "정보 확인 필요",
-                "게시 경계를 확인할 수 없습니다. 공식 source에서 직접 확인해 주세요.",
+                "현재 확인할 수 없음",
+                "여행경보 게시 정보를 현재 확인할 수 없습니다. 여행경보 공식 출처에서 직접 확인해 주세요.",
             ),
             CARD_SERVER_ERROR: (
                 "일시적 오류",
-                "이 정보를 지금 읽을 수 없습니다. 다른 카드는 계속 확인할 수 있습니다.",
+                "여행경보를 지금 읽을 수 없습니다. 입국요건은 다른 구역에서 계속 확인할 수 있습니다.",
             ),
         }
     )
@@ -519,11 +519,11 @@ def _load_warning_card() -> dict[str, object]:
         "module": "warning",
         "heading": "여행경보",
         "state": state,
-        "status_label": "재확인 필요" if stale else "게시된 source 사실",
+        "status_label": "재확인 필요" if stale else "현재 게시본",
         "message": (
-            "마지막 검수·게시 사실입니다. 더 최근 조회 또는 source 상태를 재확인해 주세요."
+            "마지막으로 검수·게시된 여행경보입니다. 공식 출처에서 최신 정보를 재확인해 주세요."
             if stale
-            else "입국요건과 독립된 공식 source의 검수·게시 사실입니다."
+            else "입국요건과 별도로 공식 출처에서 수집해 검수·게시한 여행경보입니다."
         ),
         "has_publication": True,
         "country_name": row["current_publication__scope_country__name_ko"],
diff --git a/public_web/templates/public_web/partials/warning_card.html b/public_web/templates/public_web/partials/warning_card.html
index af60d27..824d4b7 100644
--- a/public_web/templates/public_web/partials/warning_card.html
+++ b/public_web/templates/public_web/partials/warning_card.html
@@ -3,29 +3,32 @@
          aria-describedby="warning-status warning-message">
   <h2 id="warning-heading">{{ warning_card.heading }}</h2>
   <p class="status-line" id="warning-status" role="status">
-    <span class="status-symbol" aria-hidden="true"></span>
     <strong>상태: {{ warning_card.status_label }}</strong>
   </p>
   <p id="warning-message">{{ warning_card.message }}</p>
   {% if warning_card.has_publication %}
-    <dl class="fact-list">
-      <dt>국가</dt><dd>{{ warning_card.country_name }}</dd>
-      <dt>source 경보 단계 코드</dt><dd>{{ warning_card.alarm_level_code }}</dd>
-      <dt>source 범위 유형</dt><dd>{{ warning_card.scope_type }}</dd>
-      <dt>source 범위</dt><dd>{{ warning_card.scope_text }}</dd>
-      <dt>source 작성일</dt><dd>{{ warning_card.written_date|default:"source가 제공하지 않음" }}</dd>
-      <dt>마지막 성공 확인시각</dt><dd>{{ warning_card.checked_at|default:"확인 필요" }}</dd>
-      <dt>publication revision</dt><dd>generation {{ warning_card.generation }}</dd>
-      <dt>게시시각</dt><dd>{{ warning_card.published_at }}</dd>
-      <dt>source revision</dt><dd>{{ warning_card.source_revision }}</dd>
-      <dt>출처</dt>
-      <dd>
-        {{ warning_card.source_owner }} · {{ warning_card.attribution }}
-        <a class="source-link" href="{{ warning_card.source_locator }}"
-           rel="noopener noreferrer"
-           aria-label="외교부 여행경보 source 열기">공식 source</a>
-      </dd>
-    </dl>
-    <p class="verification-note"><strong>확인 필요:</strong> 단계 명칭, 발효·종료 시각과 여행일 적용성은 source에서 다시 확인해 주세요.</p>
+    <section class="publication-section">
+      <h3>출처에서 확인된 사실</h3>
+      <dl class="fact-list">
+        <dt>대상 국가</dt><dd>{{ warning_card.country_name }}</dd>
+        <dt>출처 경보 단계 코드</dt><dd>{{ warning_card.alarm_level_code }}</dd>
+        <dt>출처 범위 유형</dt><dd>{{ warning_card.scope_type }}</dd>
+        <dt>출처 범위</dt><dd>{{ warning_card.scope_text }}</dd>
+        <dt>출처 작성일</dt><dd>{{ warning_card.written_date|default:"출처가 제공하지 않음" }}</dd>
+      </dl>
+    </section>
+    <section class="publication-section">
+      <h3>출처와 게시 이력</h3>
+      <dl class="fact-list">
+        <dt>마지막 성공 확인시각</dt><dd>{{ warning_card.checked_at|default:"확인 필요" }}</dd>
+        <dt>게시 리비전</dt><dd>generation {{ warning_card.generation }}</dd>
+        <dt>게시시각</dt><dd>{{ warning_card.published_at }}</dd>
+        <dt>출처 리비전</dt><dd>{{ warning_card.source_revision }}</dd>
+        <dt>출처</dt><dd>{{ warning_card.source_owner }} · {{ warning_card.attribution }}</dd>
+      </dl>
+    </section>
+    <p class="verification-note"><strong>확인 필요:</strong> 단계 명칭, 발효·종료 시각과 여행일 적용성은 공식 출처에서 다시 확인해 주세요.</p>
+    <a class="source-link" href="{{ warning_card.source_locator }}"
+       rel="noopener noreferrer">여행경보 공식 출처 열기</a>
   {% endif %}
 </article>
diff --git a/public_web/tests/test_accessibility_contract.py b/public_web/tests/test_accessibility_contract.py
index c6692d6..79c4689 100644
--- a/public_web/tests/test_accessibility_contract.py
+++ b/public_web/tests/test_accessibility_contract.py
@@ -112,7 +112,7 @@ class AccessiblePresentationContractTests(SimpleTestCase):
                 "state": "stale",
                 "heading": "여행경보",
                 "status_label": "재확인 필요",
-                "message": "마지막 검수·게시 사실입니다.",
+                "message": "마지막으로 검수·게시된 여행경보입니다. 공식 출처에서 최신 정보를 재확인해 주세요.",
                 "has_publication": False,
             },
         }
@@ -140,7 +140,7 @@ class AccessiblePresentationContractTests(SimpleTestCase):
         )
         self.assertContains(response, "상태: 현재 게시본")
         self.assertContains(response, "상태: 재확인 필요")
-        self.assertContains(response, 'aria-hidden="true"', count=1)
+        self.assertNotContains(response, 'aria-hidden="true"')
 
     def test_styles_encode_responsive_touch_focus_and_wrapping_contracts(self):
         for required in (
diff --git a/public_web/tests/test_results.py b/public_web/tests/test_results.py
index c361b7c..45a2d76 100644
--- a/public_web/tests/test_results.py
+++ b/public_web/tests/test_results.py
@@ -132,7 +132,8 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
         warning_card = self.card_html(response, "warning-card")
         self.assertIn("상태: 게시된 자료 없음", entry_card)
         self.assertIn("아직 게시된 입국요건 자료가 없습니다", entry_card)
-        self.assertIn("아직 검수·게시된 source 사실이 없습니다", warning_card)
+        self.assertIn("상태: 게시된 자료 없음", warning_card)
+        self.assertIn("아직 게시된 여행경보 자료가 없습니다", warning_card)
         self.assertNotIn("sessionid", response.cookies)
 
     def test_entry_ready_marks_only_unpublished_warning_unavailable(self):
@@ -146,7 +147,7 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
         self.assertIn("상태: 현재 게시본", entry_card)
         self.assertIn("90일", entry_card)
         self.assertIn('data-state="unavailable"', warning_card)
-        self.assertIn("게시 경계를 확인할 수 없습니다", warning_card)
+        self.assertIn("상태: 현재 확인할 수 없음", warning_card)
 
     def test_warning_ready_marks_only_unpublished_entry_unavailable(self):
         self.publish_warning(scope_text="독립 경보 범위")
@@ -158,6 +159,7 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
         self.assertIn('data-state="unavailable"', entry_card)
         self.assertIn("상태: 현재 확인할 수 없음", entry_card)
         self.assertIn('data-state="ready"', warning_card)
+        self.assertIn("상태: 현재 게시본", warning_card)
         self.assertIn("독립 경보 범위", warning_card)
 
     def test_ready_cards_render_only_approved_source_facts_and_limits(self):
@@ -173,16 +175,20 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
         self.assertContains(response, "90일")
         self.assertContains(response, "출처 자료 날짜")
         self.assertContains(response, "2025-01-20")
-        self.assertContains(response, "source 경보 단계 코드")
+        self.assertContains(response, "출처 경보 단계 코드")
+        self.assertContains(response, "출처 범위 유형")
+        self.assertContains(response, "출처 범위")
+        self.assertContains(response, "출처 작성일")
         self.assertContains(response, "긴 한국어 검증 범위와 &amp; 기호")
-        self.assertContains(response, "source가 제공하지 않음")
+        self.assertContains(response, "출처가 제공하지 않음")
         self.assertContains(response, "마지막 성공 확인시각", count=2)
-        self.assertContains(response, "게시 리비전")
-        self.assertContains(response, "publication revision")
+        self.assertContains(response, "게시 리비전", count=2)
+        self.assertContains(response, "출처 리비전", count=2)
         self.assertContains(response, "확인 필요")
         self.assertContains(response, "외교부|공공데이터포털", count=2)
         self.assertContains(response, 'rel="noopener noreferrer"', count=2)
         self.assertContains(response, "입국요건 공식 출처 열기")
+        self.assertContains(response, "여행경보 공식 출처 열기")
 
         entry_card = self.card_html(response, "entry-card")
         self.assertLess(entry_card.index("상태:"), entry_card.index("출처에서 확인된 사실"))
@@ -353,7 +359,7 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
         self.assertContains(response, "재확인 필요")
         self.assertContains(response, "90일")
         warning_card = self.card_html(response, "warning-card")
-        self.assertIn("게시 경계를 확인할 수 없습니다", warning_card)
+        self.assertIn("현재 확인할 수 없음", warning_card)
 
     def test_warning_stale_marks_only_unpublished_entry_unavailable(self):
         self.publish_warning(scope_text="독립 경보 범위")
@@ -525,6 +531,10 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
             response,
             'id="warning-card" data-state="server-error"',
         )
+        entry_card = self.card_html(response, "entry-card")
+        warning_card = self.card_html(response, "warning-card")
+        self.assertIn("상태: 현재 확인할 수 없음", entry_card)
+        self.assertIn("상태: 일시적 오류", warning_card)
         self.assertNotContains(response, "synthetic warning read failure")
 
     def test_rollback_restores_entry_html_without_changing_warning(self):
@@ -615,13 +625,13 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
         self.assertIn('data-state="stale"', warning_html)
         self.assertIn("첫 경보 범위", warning_html)
         self.assertNotIn("둘째 경보 범위", warning_html)
-        self.assertIn("source 경보 단계 코드</dt><dd>3", warning_html)
-        self.assertIn("source 범위 유형</dt><dd>일부", warning_html)
+        self.assertIn("출처 경보 단계 코드</dt><dd>3", warning_html)
+        self.assertIn("출처 범위 유형</dt><dd>일부", warning_html)
         self.assertIn(
-            "publication revision</dt><dd>generation 3",
+            "게시 리비전</dt><dd>generation 3",
             warning_html,
         )
-        self.assertIn("source revision</dt><dd>rights-v1", warning_html)
+        self.assertIn("출처 리비전</dt><dd>rights-v1", warning_html)
         self.assertIn("외교부|공공데이터포털", warning_html)
         self.assertEqual(
             self.card_html(response, "entry-card"),


