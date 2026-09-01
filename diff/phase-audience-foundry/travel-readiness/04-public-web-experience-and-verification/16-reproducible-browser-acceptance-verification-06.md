## `test(frontend): align live warning browser contract`

diff --git a/e2e/browser_acceptance.py b/e2e/browser_acceptance.py
index 499893a..3d95ac3 100644
--- a/e2e/browser_acceptance.py
+++ b/e2e/browser_acceptance.py
@@ -3662,9 +3662,9 @@ def _focused_result_javascript(
     const warningStates = await warningModules.evaluateAll((nodes) => nodes.map((node) => node.dataset.state));
     if ({json.dumps(state)} === 'stale') {{
       if (!entryStates.every((value, index) => value === (index === 0 ? 'stale' : 'ready'))) fail('entry-state-isolation');
-      if (!warningStates.every((value, index) => value === (index === 0 ? 'unavailable' : 'ready'))) fail('warning-state-isolation');
+      if (!warningStates.every((value, index) => value === (index === 0 ? 'server-error' : 'ready'))) fail('warning-state-isolation');
       const first = opportunities.first();
-      if (await first.locator('[data-module="entry"][data-state="stale"]').count() !== 1 || await first.locator('[data-module="warning"][data-state="unavailable"]').count() !== 1) fail('composite-state-boundary');
+      if (await first.locator('[data-module="entry"][data-state="stale"]').count() !== 1 || await first.locator('[data-module="warning"][data-state="server-error"]').count() !== 1) fail('composite-state-boundary');
     }} else if (!entryStates.every((value) => value === 'ready') || !warningStates.every((value) => value === 'ready')) {{
       fail('ready-module-state');
     }}
@@ -3690,6 +3690,16 @@ def _focused_result_javascript(
       }});
     }});
     if (!indexContract) fail('destination-index-contract');
+    const verifiedEmpty = page.locator('#destination-2-warning');
+    if (await verifiedEmpty.locator('.verified-empty-note').count() !== 1
+      || await verifiedEmpty.locator('.warning-fact-list').count() !== 0
+      || !(await verifiedEmpty.textContent()).includes('안전함이나 여행 가능 여부를 뜻하지 않습니다.')
+      || await verifiedEmpty.locator('details.source-history').count() !== 1
+      || await verifiedEmpty.getByRole('link', {{ name: '여행경보 공식 출처 열기' }}).count() !== 1) fail('verified-empty-warning');
+    const orderedFacts = page.locator('#destination-3-warning .warning-fact-list > li');
+    if (await orderedFacts.count() !== 2) fail('warning-fact-count');
+    const warningFactText = await orderedFacts.allTextContents();
+    if (!warningFactText[0].includes('베트남 전역') || !warningFactText[1].includes('국경 인접 일부 지역')) fail('warning-fact-order');
     const externalLinks = page.locator('a[rel="noopener noreferrer"]');
     if (await externalLinks.count() < 3) fail('source-links');
   }} else if (await page.locator('[data-module="entry"], [data-module="warning"]').count() !== 0) {{
diff --git a/e2e/browser_scenario_server.py b/e2e/browser_scenario_server.py
index 81720fd..c7bdfb9 100644
--- a/e2e/browser_scenario_server.py
+++ b/e2e/browser_scenario_server.py
@@ -171,6 +171,42 @@ def _stale_checked_at(module: str, current_time: datetime) -> str:
     )
 
 
+def _valid_warning_display_shape(card: Mapping[str, object]) -> bool:
+    """Accept the current snapshot shape or the supported legacy scalar shape."""
+
+    facts = card.get("facts")
+    verified_empty = card.get("verified_empty")
+    if type(verified_empty) is bool and isinstance(facts, (tuple, list)):
+        if verified_empty != (len(facts) == 0) or len(facts) > 100:
+            return False
+        for expected_position, fact in enumerate(facts, start=1):
+            if (
+                not isinstance(fact, dict)
+                or fact.get("source_position") != expected_position
+                or not isinstance(fact.get("alarm_level_code"), str)
+                or not fact["alarm_level_code"].strip()
+                or not isinstance(fact.get("scope_type"), str)
+                or not fact["scope_type"].strip()
+                or not isinstance(fact.get("scope_text"), str)
+                or not fact["scope_text"].strip()
+                or (
+                    fact.get("written_date") is not None
+                    and not isinstance(fact.get("written_date"), str)
+                )
+            ):
+                return False
+        return True
+    return all(
+        key in card
+        for key in (
+            "alarm_level_code",
+            "scope_type",
+            "scope_text",
+            "written_date",
+        )
+    )
+
+
 def _validate_actual_cards(cards: object) -> dict[str, dict[str, object]]:
     """Require exact typed publication identities before making a clone."""
 
@@ -204,10 +240,6 @@ def _validate_actual_cards(cards: object) -> dict[str, dict[str, object]]:
             "generation",
         },
         "warning": {
-            "alarm_level_code",
-            "scope_type",
-            "scope_text",
-            "written_date",
             "checked_at",
             "published_at",
             "generation",
@@ -229,6 +261,7 @@ def _validate_actual_cards(cards: object) -> dict[str, dict[str, object]]:
             or card.get("attribution") != attribution
             or card.get("source_revision") != "rights-v1"
             or any(key not in card for key in required[module])
+            or (module == "warning" and not _valid_warning_display_shape(card))
         ):
             raise ScenarioFailure("publication-not-ready")
         validated[module] = copy.deepcopy(card)
@@ -295,13 +328,19 @@ def _e2e_ready_cards(
             "source_locator": WARNING_SOURCE_LOCATOR,
             "attribution": WARNING_ATTRIBUTION,
             "checked_at": checked_at,
-            "alarm_level_code": "3",
-            "scope_type": "일부",
-            "scope_text": (
-                "후쿠시마 원전 반경 30km 이내 및 일본 정부 지정 "
-                "피난지시구역"
+            "verified_empty": False,
+            "facts": (
+                {
+                    "source_position": 1,
+                    "alarm_level_code": "3",
+                    "scope_type": "일부",
+                    "scope_text": (
+                        "후쿠시마 원전 반경 30km 이내 및 일본 정부 지정 "
+                        "피난지시구역"
+                    ),
+                    "written_date": None,
+                },
             ),
-            "written_date": None,
         },
     }
     return _validate_actual_cards(cards)
@@ -372,7 +411,13 @@ def build_scenario_cards(
     raise ScenarioFailure("unknown-scenario")
 
 
-def _travel_card(module: str, *, country_name: str, state: str) -> dict[str, object]:
+def _travel_card(
+    module: str,
+    *,
+    country_name: str,
+    state: str,
+    warning_variant: str = "facts",
+) -> dict[str, object]:
     """Return local, typed publication display data without a source response."""
 
     from entry_requirements.ingestion import ENTRY_ATTRIBUTION, ENTRY_SOURCE_OWNER
@@ -393,7 +438,11 @@ def _travel_card(module: str, *, country_name: str, state: str) -> dict[str, obj
         "stale": "마지막 게시 자료입니다. 최신 정보는 공식 출처에서 다시 확인해 주세요.",
         "server-error": "이 정보를 불러오는 중 오류가 발생했습니다. 잠시 후 다시 확인해 주세요.",
     }
-    if state not in labels or module not in {"entry", "warning"}:
+    if (
+        state not in labels
+        or module not in {"entry", "warning"}
+        or warning_variant not in {"facts", "multi-fact", "verified-empty"}
+    ):
         raise ScenarioFailure("publication-state")
     ready = state in {"ready", "stale"}
     common: dict[str, object] = {
@@ -426,12 +475,36 @@ def _travel_card(module: str, *, country_name: str, state: str) -> dict[str, obj
                 "source_owner": WARNING_SOURCE_OWNER,
                 "source_locator": WARNING_SOURCE_LOCATOR,
                 "attribution": WARNING_ATTRIBUTION,
-                "alarm_level_code": "1",
-                "scope_type": "국가",
-                "scope_text": country_name,
-                "written_date": "2026-08-29",
+                "verified_empty": ready and warning_variant == "verified-empty",
+                "facts": (),
             }
         )
+        if ready and warning_variant != "verified-empty":
+            facts = [
+                {
+                    "source_position": 1,
+                    "alarm_level_code": "1",
+                    "scope_type": "국가",
+                    "scope_text": f"{country_name} 전역",
+                    "written_date": "2026-08-29",
+                }
+            ]
+            if warning_variant == "multi-fact":
+                facts.append(
+                    {
+                        "source_position": 2,
+                        "alarm_level_code": "2",
+                        "scope_type": "일부",
+                        "scope_text": "국경 인접 일부 지역",
+                        "written_date": None,
+                    }
+                )
+            common["facts"] = tuple(facts)
+        if ready and warning_variant == "verified-empty":
+            common["message"] = (
+                "공식 출처의 국가 전체 조회에서 게시할 여행경보 항목이 0건으로 "
+                "확인됐습니다. 안전함이나 여행 가능 여부를 뜻하지 않습니다."
+            )
     return common
 
 
@@ -449,6 +522,7 @@ def _schedule(
 def _opportunity(
     *, rank: int, city: str, country: str, airport: str, code: str,
     stay: str, entry_state: str = "ready", warning_state: str = "ready",
+    warning_variant: str = "facts",
 ) -> dict[str, object]:
     outbound = _schedule(
         "대한항공", f"KE{700 + rank}", "9월 18일 (금) 19:30",
@@ -495,7 +569,10 @@ def _opportunity(
             "entry", country_name=country, state=entry_state
         ),
         "warning_card": _travel_card(
-            "warning", country_name=country, state=warning_state
+            "warning",
+            country_name=country,
+            state=warning_state,
+            warning_variant=warning_variant,
         ),
     }
 
@@ -523,7 +600,7 @@ def build_scenario_context(scenario: str) -> dict[str, object]:
     opportunities: list[dict[str, object]] = []
     if state in {"ready", "stale"}:
         module_states = (
-            ("stale", "unavailable") if state == "stale" else ("ready", "ready")
+            ("stale", "server-error") if state == "stale" else ("ready", "ready")
         )
         opportunities = [
             _opportunity(
@@ -534,10 +611,12 @@ def build_scenario_context(scenario: str) -> dict[str, object]:
             _opportunity(
                 rank=2, city="타이베이", country="대만",
                 airport="타오위안 국제공항", code="TPE", stay="51시간 35분",
+                warning_variant="verified-empty",
             ),
             _opportunity(
                 rank=3, city="다낭", country="베트남",
                 airport="다낭 국제공항", code="DAD", stay="48시간 40분",
+                warning_variant="multi-fact",
             ),
         ]
     if scenario == "long-korean" and opportunities:
diff --git a/operations/tests/test_browser_acceptance_harness.py b/operations/tests/test_browser_acceptance_harness.py
index 420cd86..338102e 100644
--- a/operations/tests/test_browser_acceptance_harness.py
+++ b/operations/tests/test_browser_acceptance_harness.py
@@ -1121,7 +1121,10 @@ class BrowserAcceptanceHarnessTests(unittest.TestCase):
         self.assertIn("composite-state-boundary", result)
         self.assertIn("destination-index-contract", result)
         self.assertIn("index === 0 ? 'stale' : 'ready'", result)
-        self.assertIn("index === 0 ? 'unavailable' : 'ready'", result)
+        self.assertIn("index === 0 ? 'server-error' : 'ready'", result)
+        self.assertIn("verified-empty-warning", result)
+        self.assertIn("warning-fact-count", result)
+        self.assertIn("warning-fact-order", result)
         self.assertIn(json.dumps(acceptance.FOCUSED_LONG_AIRPORT), accessibility)
         self.assertIn(
             json.dumps(acceptance.FOCUSED_LONG_ENTRY_BASIS), accessibility
diff --git a/operations/tests/test_browser_scenario_servers.py b/operations/tests/test_browser_scenario_servers.py
index 825bf01..bbda6cf 100644
--- a/operations/tests/test_browser_scenario_servers.py
+++ b/operations/tests/test_browser_scenario_servers.py
@@ -147,11 +147,15 @@ class BrowserScenarioCardTests(SimpleTestCase):
             ("2026-08-31 12:33 UTC", "2026-08-31 12:33 UTC"),
         )
         self.assertEqual(cards["entry"]["snapshot_date"], "2025-01-20")
-        self.assertIsNone(cards["warning"]["written_date"])
+        self.assertFalse(cards["warning"]["verified_empty"])
+        self.assertEqual(len(cards["warning"]["facts"]), 1)
         self.assertEqual(cards["entry"]["basis_text"], "일반여권 소지자 : 상호주의")
-        self.assertEqual(cards["warning"]["alarm_level_code"], "3")
+        warning_fact = cards["warning"]["facts"][0]
+        self.assertIsNone(warning_fact["written_date"])
+        self.assertEqual(warning_fact["source_position"], 1)
+        self.assertEqual(warning_fact["alarm_level_code"], "3")
         self.assertEqual(
-            cards["warning"]["scope_text"],
+            warning_fact["scope_text"],
             "후쿠시마 원전 반경 30km 이내 및 일본 정부 지정 피난지시구역",
         )
         for module, owner, locator, attribution in (
@@ -193,6 +197,33 @@ class BrowserScenarioCardTests(SimpleTestCase):
         ):
             self.assertNotIn(forbidden, serve_source)
 
+    def test_focused_context_uses_snapshot_facts_and_verified_empty(self):
+        opportunities = scenario_server.build_scenario_context("ready")[
+            "opportunities"
+        ]
+        self.assertEqual(len(opportunities), 3)
+
+        first_warning = opportunities[0]["warning_card"]
+        empty_warning = opportunities[1]["warning_card"]
+        multi_warning = opportunities[2]["warning_card"]
+        self.assertFalse(first_warning["verified_empty"])
+        self.assertEqual(
+            tuple(fact["source_position"] for fact in first_warning["facts"]),
+            (1,),
+        )
+        self.assertTrue(empty_warning["verified_empty"])
+        self.assertEqual(empty_warning["facts"], ())
+        self.assertIn("안전함이나 여행 가능 여부", empty_warning["message"])
+        self.assertFalse(multi_warning["verified_empty"])
+        self.assertEqual(
+            tuple(fact["source_position"] for fact in multi_warning["facts"]),
+            (1, 2),
+        )
+        self.assertEqual(
+            tuple(fact["scope_text"] for fact in multi_warning["facts"]),
+            ("베트남 전역", "국경 인접 일부 지역"),
+        )
+
     def test_e2e_ready_provider_rechecks_marker_loopback_and_key_absence(self):
         for changed in (
             {"TRAVEL_READINESS_E2E_SCENARIO_MARKER": ""},
@@ -447,7 +478,7 @@ class BrowserScenarioCardTests(SimpleTestCase):
                     )
                     self.assertEqual(
                         response.content.count(
-                            b'data-module="warning" data-state="unavailable"'
+                            b'data-module="warning" data-state="server-error"'
                         ),
                         1,
                     )
@@ -463,6 +494,19 @@ class BrowserScenarioCardTests(SimpleTestCase):
                         ),
                         2,
                     )
+                if name == "ready":
+                    self.assertEqual(
+                        response.content.count(b'class="warning-fact-list"'),
+                        2,
+                    )
+                    self.assertEqual(
+                        response.content.count(b'class="verified-empty-note"'),
+                        1,
+                    )
+                    self.assertLess(
+                        response.content.index("베트남 전역".encode()),
+                        response.content.index("국경 인접 일부 지역".encode()),
+                    )
                 if name == "long-korean":
                     self.assertIn(
                         scenario_server.LONG_KOREAN_AIRPORT_NAME.encode(),
diff --git a/public_web/tests/test_accessibility_contract.py b/public_web/tests/test_accessibility_contract.py
index 90d23f4..f47a4f6 100644
--- a/public_web/tests/test_accessibility_contract.py
+++ b/public_web/tests/test_accessibility_contract.py
@@ -315,6 +315,116 @@ class AccessiblePresentationContractTests(SimpleTestCase):
         self.assertContains(response, "출처와 확인 이력 보기")
         self.assertContains(response, "여행경보 공식 출처 열기")
 
+    @patch("public_web.views.search_travel_opportunities")
+    def test_warning_snapshot_renders_all_one_hundred_facts_in_source_order(
+        self,
+        search,
+    ):
+        opportunity = deepcopy(self._opportunity())
+        warning = opportunity["warning_card"]
+        warning["state"] = "ready"
+        warning["status_label"] = "현재 게시본"
+        warning["facts"] = tuple(
+            {
+                "source_position": position,
+                "alarm_level_code": str((position % 4) + 1),
+                "scope_type": "일부",
+                "scope_text": f"경보 범위 {position:03d}",
+                "written_date": None,
+            }
+            for position in range(1, 101)
+        )
+        search.return_value = self._search_result(opportunities=[opportunity])
+
+        response = self.client.post(reverse("public_web:index"), self.valid_data())
+        body = response.content.decode("utf-8")
+        start = body.index('<article id="destination-1-warning"')
+        fragment = body[start : body.index("</article>", start)]
+
+        self.assertEqual(fragment.count("<li>"), 100)
+        self.assertLess(
+            fragment.index("경보 범위 001"),
+            fragment.index("경보 범위 100"),
+        )
+
+    @patch("public_web.views.search_travel_opportunities")
+    def test_legacy_warning_scalar_fallback_remains_readable(self, search):
+        opportunity = deepcopy(self._opportunity())
+        warning = opportunity["warning_card"]
+        warning.pop("facts")
+        warning.pop("verified_empty")
+        warning.update(
+            {
+                "alarm_level_code": "LEGACY-1",
+                "scope_type": "기존 범위 유형",
+                "scope_text": "기존 단일 경보 범위",
+                "written_date": "2026-08-29",
+            }
+        )
+        search.return_value = self._search_result(opportunities=[opportunity])
+
+        response = self.client.post(reverse("public_web:index"), self.valid_data())
+
+        for value in (
+            "LEGACY-1",
+            "기존 범위 유형",
+            "기존 단일 경보 범위",
+            "2026-08-29",
+        ):
+            self.assertContains(response, value)
+        self.assertNotContains(response, "확인된 항목 수: 0건")
+
+    @patch("public_web.views.search_travel_opportunities")
+    def test_warning_five_states_do_not_change_the_entry_boundary(
+        self,
+        search,
+    ):
+        states = {
+            "ready": ("현재 게시본", True),
+            "empty": ("게시된 자료 없음", False),
+            "unavailable": ("현재 확인할 수 없음", False),
+            "stale": ("재확인 필요", True),
+            "server-error": ("일시적 오류", False),
+        }
+        for state, (status_label, has_publication) in states.items():
+            with self.subTest(state=state):
+                opportunity = deepcopy(self._opportunity())
+                warning = opportunity["warning_card"]
+                warning.update(
+                    {
+                        "state": state,
+                        "status_label": status_label,
+                        "message": f"경보 상태 설명 {state}",
+                        "has_publication": has_publication,
+                        "source_owner": "경보 전용 기관",
+                    }
+                )
+                search.return_value = self._search_result(
+                    opportunities=[opportunity]
+                )
+
+                response = self.client.post(
+                    reverse("public_web:index"), self.valid_data()
+                )
+                body = response.content.decode("utf-8")
+
+                self.assertContains(
+                    response,
+                    f'data-module="warning" data-state="{state}"',
+                )
+                self.assertContains(response, f"상태: {status_label}")
+                self.assertContains(
+                    response,
+                    'data-module="entry" data-state="ready"',
+                )
+                self.assertContains(response, "입국요건 공식 출처 열기")
+                if has_publication:
+                    self.assertIn("경보 전용 기관", body)
+                    self.assertContains(response, "여행경보 공식 출처 열기")
+                else:
+                    self.assertNotIn("경보 전용 기관", body)
+                    self.assertNotContains(response, "여행경보 공식 출처 열기")
+
     @patch("public_web.views.search_travel_opportunities")
     def test_ready_search_without_matches_explains_how_to_try_again(self, search):
         search.return_value = self._search_result(opportunities=[])
