## `fix(source): parse live schedule item arrays`

diff --git a/travel_windows/parser.py b/travel_windows/parser.py
index c6a7bf3..b413dbd 100644
--- a/travel_windows/parser.py
+++ b/travel_windows/parser.py
@@ -408,6 +408,24 @@ def _documented_response(payload: bytes) -> tuple[dict, dict]:
     return header, body
 
 
+def _item_object_rows(items: object) -> list[dict]:
+    if type(items) is list:
+        rows = items
+    elif type(items) is dict and set(items) == _OFFICIAL_ITEMS_KEYS:
+        raw_items = items["item"]
+        if type(raw_items) is dict:
+            rows = [raw_items]
+        elif type(raw_items) is list:
+            rows = raw_items
+        else:
+            raise ValueError
+    else:
+        raise ValueError
+    if any(type(row) is not dict for row in rows):
+        raise ValueError
+    return rows
+
+
 def _item_rows(items: object, *, exact_keys: set[str]) -> list[dict]:
     if type(items) is not dict or set(items) != _OFFICIAL_ITEMS_KEYS:
         raise ValueError
@@ -617,16 +635,7 @@ def _decode_official_page(payload: bytes) -> tuple[int, int, int, list[dict[str,
         or set(body) != _OFFICIAL_BODY_KEYS
     ):
         raise ValueError
-    items = body["items"]
-    if type(items) is not dict or set(items) != _OFFICIAL_ITEMS_KEYS:
-        raise ValueError
-    raw_items = items["item"]
-    if type(raw_items) is dict:
-        rows = [raw_items]
-    elif type(raw_items) is list:
-        rows = raw_items
-    else:
-        raise ValueError
+    rows = _item_object_rows(body["items"])
     if any(
         type(row) is not dict
         or not _OFFICIAL_REQUIRED_ITEM_KEYS.issubset(row)
diff --git a/travel_windows/tests/test_source_publication.py b/travel_windows/tests/test_source_publication.py
index 0a66bb8..7a74ab8 100644
--- a/travel_windows/tests/test_source_publication.py
+++ b/travel_windows/tests/test_source_publication.py
@@ -90,15 +90,23 @@ def official_row(**overrides):
     return row
 
 
-def official_page(rows, *, page=1, page_size=None, total=None):
+def official_page(
+    rows,
+    *,
+    page=1,
+    page_size=None,
+    total=None,
+    direct_items=False,
+):
     page_size = len(rows) if page_size is None else page_size
     total = len(rows) if total is None else total
+    items = rows if direct_items else {"item": rows}
     return json.dumps(
         {
             "response": {
                 "header": {"resultCode": "00", "resultMsg": "NORMAL SERVICE."},
                 "body": {
-                    "items": {"item": rows},
+                    "items": items,
                     "pageNo": str(page),
                     "numOfRows": str(page_size),
                     "totalCount": str(total),
@@ -258,6 +266,74 @@ def collect_and_stage_fixture(
 
 
 class DataGoScheduleAdapterTests(SimpleTestCase):
+    def test_direct_list_schedule_matches_the_documented_item_wrapper(self):
+        departure_rows = [official_row()]
+        arrival_rows = [
+            official_row(
+                flightId="KE704",
+                masterFlightId="KE704",
+                st="2015",
+            )
+        ]
+        wrapped = adapt_data_go_schedule_pages(
+            departure_pages=(official_page(departure_rows),),
+            arrival_pages=(official_page(arrival_rows),),
+            source_date=date(2026, 8, 31),
+        )
+        direct = adapt_data_go_schedule_pages(
+            departure_pages=(
+                official_page(departure_rows, direct_items=True),
+            ),
+            arrival_pages=(
+                official_page(arrival_rows, direct_items=True),
+            ),
+            source_date=date(2026, 8, 31),
+        )
+
+        self.assertNotIsInstance(direct, AviationParseFailure)
+        self.assertEqual(direct, wrapped)
+
+    def test_schedule_item_envelope_remains_exact(self):
+        arrival = official_page(
+            [official_row(flightId="KE704", masterFlightId="KE704")]
+        )
+        valid_document = json.loads(official_page([official_row()]))
+        row = valid_document["response"]["body"]["items"]["item"][0]
+        invalid_items = (
+            row,
+            {"item": [row], "extra": []},
+            None,
+            [None],
+        )
+
+        for items in invalid_items:
+            with self.subTest(items_type=type(items).__name__):
+                document = json.loads(official_page([official_row()]))
+                document["response"]["body"]["items"] = items
+                result = adapt_data_go_schedule_pages(
+                    departure_pages=(json.dumps(document).encode("utf-8"),),
+                    arrival_pages=(arrival,),
+                    source_date=date(2026, 8, 31),
+                )
+                self.assertIsInstance(result, AviationParseFailure)
+
+    def test_single_object_item_wrapper_remains_supported(self):
+        departure = json.loads(official_page([official_row()]))
+        departure["response"]["body"]["items"]["item"] = departure[
+            "response"
+        ]["body"]["items"]["item"][0]
+        result = adapt_data_go_schedule_pages(
+            departure_pages=(json.dumps(departure).encode("utf-8"),),
+            arrival_pages=(
+                official_page(
+                    [official_row(flightId="KE704", masterFlightId="KE704")]
+                ),
+            ),
+            source_date=date(2026, 8, 31),
+        )
+
+        self.assertNotIsInstance(result, AviationParseFailure)
+
     def test_complete_pages_normalize_and_dedupe_codeshare_by_master(self):
         departure = official_page(
             [


## `fix(source): parse live aviation reference arrays`

diff --git a/travel_windows/parser.py b/travel_windows/parser.py
index b413dbd..8a4d033 100644
--- a/travel_windows/parser.py
+++ b/travel_windows/parser.py
@@ -427,17 +427,9 @@ def _item_object_rows(items: object) -> list[dict]:
 
 
 def _item_rows(items: object, *, exact_keys: set[str]) -> list[dict]:
-    if type(items) is not dict or set(items) != _OFFICIAL_ITEMS_KEYS:
-        raise ValueError
-    raw_items = items["item"]
-    if type(raw_items) is dict:
-        rows = [raw_items]
-    elif type(raw_items) is list:
-        rows = raw_items
-    else:
-        raise ValueError
+    rows = _item_object_rows(items)
     if not rows or any(
-        type(row) is not dict or set(row) != exact_keys for row in rows
+        set(row) != exact_keys for row in rows
     ):
         raise ValueError
     return rows
diff --git a/travel_windows/tests/test_source_publication.py b/travel_windows/tests/test_source_publication.py
index 7a74ab8..484d7a9 100644
--- a/travel_windows/tests/test_source_publication.py
+++ b/travel_windows/tests/test_source_publication.py
@@ -118,7 +118,12 @@ def official_page(
     ).encode("utf-8")
 
 
-def city_reference_payload(*, overrides=None, extra_rows=()):
+def city_reference_payload(
+    *,
+    overrides=None,
+    extra_rows=(),
+    direct_items=False,
+):
     country_names = {
         iso_alpha2: name_ko
         for _country_id, iso_alpha2, name_ko, _name_en, _public in (
@@ -144,6 +149,7 @@ def city_reference_payload(*, overrides=None, extra_rows=()):
         ) in CURATED_AIRPORT_ROWS
     ]
     rows.extend(extra_rows)
+    items = rows if direct_items else {"item": rows}
     return json.dumps(
         {
             "response": {
@@ -151,7 +157,7 @@ def city_reference_payload(*, overrides=None, extra_rows=()):
                     "resultCode": "00",
                     "resultMsg": "NORMAL SERVICE.",
                 },
-                "body": {"items": {"item": rows}},
+                "body": {"items": items},
             }
         },
         ensure_ascii=False,
@@ -181,9 +187,17 @@ def legacy_arrival_row(**overrides):
     return row
 
 
-def legacy_arrival_page(rows, *, page=1, page_size=None, total=None):
+def legacy_arrival_page(
+    rows,
+    *,
+    page=1,
+    page_size=None,
+    total=None,
+    direct_items=False,
+):
     page_size = len(rows) if page_size is None else page_size
     total = len(rows) if total is None else total
+    items = rows if direct_items else {"item": rows}
     return json.dumps(
         {
             "response": {
@@ -192,7 +206,7 @@ def legacy_arrival_page(rows, *, page=1, page_size=None, total=None):
                     "resultMsg": "NORMAL SERVICE.",
                 },
                 "body": {
-                    "items": {"item": rows},
+                    "items": items,
                     "pageNo": str(page),
                     "numOfRows": str(page_size),
                     "totalCount": str(total),
@@ -487,6 +501,30 @@ class DataGoScheduleAdapterTests(SimpleTestCase):
             AviationParseFailure,
         )
 
+    def test_direct_list_references_match_the_documented_item_wrappers(self):
+        wrapped_city = parse_destination_city_reference(
+            city_reference_payload()
+        )
+        direct_city = parse_destination_city_reference(
+            city_reference_payload(direct_items=True)
+        )
+        wrapped_legacy = parse_legacy_arrival_pages(
+            (legacy_arrival_page([legacy_arrival_row()]),)
+        )
+        direct_legacy = parse_legacy_arrival_pages(
+            (
+                legacy_arrival_page(
+                    [legacy_arrival_row()],
+                    direct_items=True,
+                ),
+            )
+        )
+
+        self.assertNotIsInstance(direct_city, AviationParseFailure)
+        self.assertNotIsInstance(direct_legacy, AviationParseFailure)
+        self.assertEqual(direct_city, wrapped_city)
+        self.assertEqual(direct_legacy, wrapped_legacy)
+
     def test_duration_rows_require_the_exact_15151728_dataset_locator(self):
         result = parse_route_durations(
             b"destination_iata,outbound_minutes,inbound_minutes,reference_date,"
@@ -662,9 +700,15 @@ class FlightEvidencePublicationTests(TransactionTestCase):
         )
 
     def test_receipts_and_typed_candidate_persist_without_raw_body_or_publication(self):
-        departure = official_page([official_row()])
+        departure = official_page([official_row()], direct_items=True)
         arrival = official_page(
-            [official_row(flightId="KE704", masterFlightId="KE704", st="2015")]
+            [official_row(flightId="KE704", masterFlightId="KE704", st="2015")],
+            direct_items=True,
+        )
+        city_payload = city_reference_payload(direct_items=True)
+        legacy_payload = legacy_arrival_page(
+            [legacy_arrival_row()],
+            direct_items=True,
         )
         duration = (
             b"destination_iata,outbound_minutes,inbound_minutes,reference_date,"
@@ -675,10 +719,8 @@ class FlightEvidencePublicationTests(TransactionTestCase):
         outcome = collect_and_stage_fixture(
             departure_pages=(departure,),
             arrival_pages=(arrival,),
-            city_payload=city_reference_payload(),
-            legacy_arrival_pages=(
-                legacy_arrival_page([legacy_arrival_row()]),
-            ),
+            city_payload=city_payload,
+            legacy_arrival_pages=(legacy_payload,),
             duration_csv=duration,
             source_date=date(2026, 8, 31),
             source_checked_at=timezone.now(),
@@ -713,6 +755,22 @@ class FlightEvidencePublicationTests(TransactionTestCase):
         revision = FlightScheduleRevision.objects.get(
             pk=outcome.schedule_revision_id
         )
+        parse_runs = (
+            revision.parse_run,
+            revision.city_reference_parse_run,
+            revision.legacy_arrivals_parse_run,
+        )
+        self.assertEqual(
+            [run.parser_version for run in parse_runs],
+            [ParseRun.ParserVersion.V2] * 3,
+        )
+        self.assertTrue(
+            all(
+                run.expected_schema_fingerprint_sha256
+                == run.observed_schema_fingerprint_sha256
+                for run in parse_runs
+            )
+        )
         ordered = list(
             revision.parse_run.ordered_inputs.select_related("artifact").order_by(
                 "ordinal"
@@ -738,7 +796,6 @@ class FlightEvidencePublicationTests(TransactionTestCase):
             revision.parse_run.input_identity_sha256,
             aviation_parse_input_identity(identities),
         )
-        city_payload = city_reference_payload()
         city_artifact = SourceArtifact.objects.get(
             source__code=CITY_SOURCE_CODE
         )


## `fix(source): collapse exact legacy schedule duplicates`

diff --git a/travel_windows/parser.py b/travel_windows/parser.py
index 8a4d033..a21f72c 100644
--- a/travel_windows/parser.py
+++ b/travel_windows/parser.py
@@ -530,8 +530,9 @@ def parse_legacy_arrival_pages(
         if len(raw_rows) != total:
             raise ValueError
 
-        arrivals: list[ParsedLegacyArrival] = []
-        identities: set[tuple[object, ...]] = set()
+        arrivals_by_identity: dict[
+            tuple[object, ...], ParsedLegacyArrival
+        ] = {}
         for row in raw_rows:
             weekdays = []
             for index, key in enumerate(_LEGACY_WEEKDAY_KEYS):
@@ -581,14 +582,14 @@ def parse_legacy_arrival_pages(
                 arrival.valid_until,
                 arrival.weekday_mask,
             )
-            if identity in identities:
+            previous = arrivals_by_identity.get(identity)
+            if previous is not None and previous != arrival:
                 raise ValueError
-            identities.add(identity)
-            arrivals.append(arrival)
+            arrivals_by_identity[identity] = arrival
         return LegacyArrivalParseSuccess(
             arrivals=tuple(
                 sorted(
-                    arrivals,
+                    arrivals_by_identity.values(),
                     key=lambda row: (
                         row.destination_iata,
                         row.icn_event_time,
diff --git a/travel_windows/tests/test_source_publication.py b/travel_windows/tests/test_source_publication.py
index 484d7a9..bcbf9a5 100644
--- a/travel_windows/tests/test_source_publication.py
+++ b/travel_windows/tests/test_source_publication.py
@@ -525,6 +525,40 @@ class DataGoScheduleAdapterTests(SimpleTestCase):
         self.assertEqual(direct_city, wrapped_city)
         self.assertEqual(direct_legacy, wrapped_legacy)
 
+    def test_legacy_arrivals_collapse_only_equal_typed_duplicates(self):
+        row = legacy_arrival_row()
+        normalized_duplicate = legacy_arrival_row(
+            airline=" 대한항공 ",
+            airport=" 도쿄/나리타 ",
+            firstdate="2026-03-29",
+            flightid="KE 704",
+            lastdate="2026-10-24",
+            st="20:15",
+        )
+        repeated = parse_legacy_arrival_pages(
+            (legacy_arrival_page([row, normalized_duplicate]),)
+        )
+        conflicting = parse_legacy_arrival_pages(
+            (
+                legacy_arrival_page(
+                    [row, legacy_arrival_row(airline="다른 항공사")]
+                ),
+            )
+        )
+        distinct = parse_legacy_arrival_pages(
+            (
+                legacy_arrival_page(
+                    [row, legacy_arrival_row(st="2016")]
+                ),
+            )
+        )
+
+        self.assertNotIsInstance(repeated, AviationParseFailure)
+        self.assertEqual(len(repeated.arrivals), 1)
+        self.assertIsInstance(conflicting, AviationParseFailure)
+        self.assertNotIsInstance(distinct, AviationParseFailure)
+        self.assertEqual(len(distinct.arrivals), 2)
+
     def test_duration_rows_require_the_exact_15151728_dataset_locator(self):
         result = parse_route_durations(
             b"destination_iata,outbound_minutes,inbound_minutes,reference_date,"


