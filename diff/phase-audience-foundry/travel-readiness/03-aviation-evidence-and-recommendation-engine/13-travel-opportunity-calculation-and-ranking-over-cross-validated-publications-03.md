## `fix(search): surface stale flight publications`

diff --git a/docs/TRAVEL-OPPORTUNITY-CONTRACT.md b/docs/TRAVEL-OPPORTUNITY-CONTRACT.md
index d2c4631..0e6b488 100644
--- a/docs/TRAVEL-OPPORTUNITY-CONTRACT.md
+++ b/docs/TRAVEL-OPPORTUNITY-CONTRACT.md
@@ -23,6 +23,9 @@
 - 사용자 입력과 계산 결과는 response scope 밖의 URL, DB, session, cookie, cache,
   log, metric, trace, audit와 backup에 저장하지 않습니다.
 - 공개 request path는 외부 source를 호출하지 않고 current publication만 읽습니다.
+- 운항 source는 7일 수집 주기와 1일 grace를 적용합니다. 마지막 성공 확인시각의
+  age가 8일 이상이거나 없거나 미래이면 `stale`로 표시하되, 마지막 검수·게시본으로
+  계산한 기회는 숨기지 않고 공식 출처 재확인을 안내합니다.
 
 ## 추천 계약
 
diff --git a/travel_windows/search.py b/travel_windows/search.py
index 3403bc0..cdc78c7 100644
--- a/travel_windows/search.py
+++ b/travel_windows/search.py
@@ -30,6 +30,7 @@ SEOUL = ZoneInfo("Asia/Seoul")
 MINIMUM_LOCAL_STAY_MINUTES = 40 * 60
 MAXIMUM_CITY_RESULTS = 6
 MAXIMUM_ITINERARIES_PER_CITY = 2
+FLIGHT_PUBLICATION_STALE_AGE = timedelta(days=8)
 
 FLIGHT_READY = "ready"
 FLIGHT_EMPTY = "empty"
@@ -581,6 +582,20 @@ def _flight_context(
     }
 
 
+def _current_publication_state(evidence: _CurrentFlightEvidence) -> str:
+    checked_at = evidence.source_checked_at
+    current = timezone.now()
+    if (
+        not isinstance(checked_at, datetime)
+        or timezone.is_naive(checked_at)
+        or timezone.is_naive(current)
+        or checked_at > current
+        or current - checked_at >= FLIGHT_PUBLICATION_STALE_AGE
+    ):
+        return FLIGHT_STALE
+    return FLIGHT_READY
+
+
 def search_travel_opportunities(
     *,
     departure_at: datetime,
@@ -616,7 +631,7 @@ def search_travel_opportunities(
             ),
         )
         return _flight_context(
-            state=FLIGHT_READY,
+            state=_current_publication_state(evidence),
             evidence=evidence,
             opportunities=opportunities,
         )
diff --git a/travel_windows/tests/test_search.py b/travel_windows/tests/test_search.py
index fc7535d..9204194 100644
--- a/travel_windows/tests/test_search.py
+++ b/travel_windows/tests/test_search.py
@@ -23,6 +23,7 @@ from travel_windows.models import (
 )
 from travel_windows.search import (
     FLIGHT_READY,
+    FLIGHT_STALE,
     _AirportEvidence,
     _CurrentFlightEvidence,
     _DurationEvidence,
@@ -107,7 +108,7 @@ def _evidence(*, schedules, durations):
         source_revision="synthetic-search-v1",
         source_locator="https://example.invalid/synthetic-schedule",
         source_attribution="합성 검색 계약 fixture",
-        source_checked_at=datetime(2026, 8, 31, 12, tzinfo=SEOUL),
+        source_checked_at=datetime(2026, 8, 31, 0, tzinfo=SEOUL),
         schedule_source_date=SOURCE_DATE,
         schedules=tuple(schedules),
         durations=tuple(durations),
@@ -378,6 +379,46 @@ class TravelOpportunityCalculationTests(SimpleTestCase):
             ["HKG", "NRT", "KIX", "FUK"],
         )
 
+    def test_eight_day_old_and_future_checks_are_stale_but_keep_results(self):
+        nrt = _airport_evidence("NRT")
+        evidence = _evidence(
+            schedules=(
+                _schedule(
+                    nrt,
+                    direction="OUTBOUND",
+                    event_time=time(9),
+                    flight_number="KE701",
+                    weekday=1,
+                ),
+                _schedule(
+                    nrt,
+                    direction="INBOUND",
+                    event_time=time(5),
+                    flight_number="KE704",
+                    weekday=3,
+                ),
+            ),
+            durations=(_duration(nrt),),
+        )
+        search_kwargs = {
+            "departure_at": datetime(2026, 9, 1, 8, tzinfo=SEOUL),
+            "return_by": datetime(2026, 9, 3, 5, tzinfo=SEOUL),
+        }
+        for now in (
+            datetime(2026, 9, 8, 0, tzinfo=SEOUL),
+            datetime(2026, 8, 30, 23, 59, tzinfo=SEOUL),
+        ):
+            with self.subTest(now=now), patch(
+                "travel_windows.search.timezone.now", return_value=now
+            ):
+                result = _search(
+                    evidence,
+                    {SUPPORTED_COUNTRY_IDS["JP"]},
+                    **search_kwargs,
+                )
+            self.assertEqual(result["flight_state"], FLIGHT_STALE)
+            self.assertEqual(len(result["opportunities"]), 1)
+
 
 class PublicationEligibilityTests(SimpleTestCase):
     def test_country_requires_both_current_entry_and_warning_publications(self):


## `fix(search): require usable travel publications`

diff --git a/travel_windows/search.py b/travel_windows/search.py
index cdc78c7..fe4899e 100644
--- a/travel_windows/search.py
+++ b/travel_windows/search.py
@@ -37,6 +37,7 @@ FLIGHT_EMPTY = "empty"
 FLIGHT_UNAVAILABLE = "unavailable"
 FLIGHT_STALE = "stale"
 FLIGHT_SERVER_ERROR = "server-error"
+ELIGIBLE_PUBLICATION_CARD_STATES = frozenset((FLIGHT_READY, FLIGHT_STALE))
 
 _CALCULATION_NOTICE = (
     "공식 인천 출발·도착 시각에 검수된 노선별 비행시간을 더하고 빼서 계산한 "
@@ -488,6 +489,16 @@ def _itinerary_context(candidate: _Candidate) -> dict[str, object]:
     }
 
 
+def _has_usable_card(
+    cards: dict[str, dict[str, object]], module: str
+) -> bool:
+    card = cards.get(module)
+    return (
+        isinstance(card, dict)
+        and card.get("state") in ELIGIBLE_PUBLICATION_CARD_STATES
+    )
+
+
 def _opportunities_context(
     city_candidates: tuple[tuple[_Candidate, ...], ...],
     *,
@@ -496,7 +507,7 @@ def _opportunities_context(
 ) -> list[dict[str, object]]:
     opportunities: list[dict[str, object]] = []
     card_cache: dict[UUID, dict[str, dict[str, object]]] = {}
-    for rank, candidates in enumerate(city_candidates, start=1):
+    for candidates in city_candidates:
         primary = candidates[0]
         cards = card_cache.get(primary.airport.country_id)
         if cards is None:
@@ -508,6 +519,11 @@ def _opportunities_context(
                     "warning": _state_card("warning", FLIGHT_SERVER_ERROR),
                 }
             card_cache[primary.airport.country_id] = cards
+        if not _has_usable_card(cards, "entry") or not _has_usable_card(
+            cards, "warning"
+        ):
+            continue
+        rank = len(opportunities) + 1
         itinerary = _itinerary_context(primary)
         opportunities.append(
             {
diff --git a/travel_windows/tests/test_search.py b/travel_windows/tests/test_search.py
index 9204194..d2a7d1e 100644
--- a/travel_windows/tests/test_search.py
+++ b/travel_windows/tests/test_search.py
@@ -419,6 +419,50 @@ class TravelOpportunityCalculationTests(SimpleTestCase):
             self.assertEqual(result["flight_state"], FLIGHT_STALE)
             self.assertEqual(len(result["opportunities"]), 1)
 
+    def test_city_requires_two_usable_independent_publication_cards(self):
+        nrt = _airport_evidence("NRT")
+        evidence = _evidence(
+            schedules=(
+                _schedule(
+                    nrt,
+                    direction="OUTBOUND",
+                    event_time=time(9),
+                    flight_number="KE701",
+                    weekday=1,
+                ),
+                _schedule(
+                    nrt,
+                    direction="INBOUND",
+                    event_time=time(5),
+                    flight_number="KE704",
+                    weekday=3,
+                ),
+            ),
+            durations=(_duration(nrt),),
+        )
+
+        with (
+            patch(
+                "travel_windows.search._load_current_flight_evidence",
+                return_value=evidence,
+            ),
+            patch(
+                "travel_windows.search._load_eligible_country_ids",
+                return_value=frozenset({SUPPORTED_COUNTRY_IDS["JP"]}),
+            ),
+        ):
+            result = search_travel_opportunities(
+                departure_at=datetime(2026, 9, 1, 8, tzinfo=SEOUL),
+                return_by=datetime(2026, 9, 3, 5, tzinfo=SEOUL),
+                publication_card_loader=lambda _country_id: {
+                    "entry": {"state": "server-error"},
+                    "warning": {"state": "ready"},
+                },
+            )
+
+        self.assertEqual(result["flight_state"], FLIGHT_READY)
+        self.assertEqual(result["opportunities"], [])
+
 
 class PublicationEligibilityTests(SimpleTestCase):
     def test_country_requires_both_current_entry_and_warning_publications(self):


