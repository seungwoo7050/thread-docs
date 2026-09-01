## `fix(web): validate seven-day travel windows`

diff --git a/public_web/forms.py b/public_web/forms.py
index c42d707..45c9953 100644
--- a/public_web/forms.py
+++ b/public_web/forms.py
@@ -6,7 +6,7 @@ from django.utils import timezone
 
 
 SEOUL = ZoneInfo("Asia/Seoul")
-MAXIMUM_SEARCH_HORIZON = timedelta(days=7)
+MAXIMUM_SEARCH_WINDOW = timedelta(days=7)
 DATETIME_LOCAL_FORMAT = "%Y-%m-%dT%H:%M"
 
 
@@ -97,11 +97,15 @@ class TripForm(forms.Form):
                     code="window_order",
                 ),
             )
-        if return_by is not None and return_by > current + MAXIMUM_SEARCH_HORIZON:
+        if (
+            departure_at is not None
+            and return_by is not None
+            and return_by - departure_at > MAXIMUM_SEARCH_WINDOW
+        ):
             self.add_error(
                 "return_by",
                 forms.ValidationError(
-                    "인천 도착 마감 시각은 현재부터 7일 이내로 입력해 주세요.",
+                    "출발 가능한 시각과 인천 도착 마감 시각 사이는 7일 이내여야 합니다.",
                     code="window_too_long",
                 ),
             )
diff --git a/public_web/tests/test_travel_opportunity_web.py b/public_web/tests/test_travel_opportunity_web.py
index e056c09..7b69b78 100644
--- a/public_web/tests/test_travel_opportunity_web.py
+++ b/public_web/tests/test_travel_opportunity_web.py
@@ -32,8 +32,11 @@ class TravelWindowFormTests(SimpleTestCase):
         )
 
     @patch("public_web.forms._now_kst", return_value=NOW)
-    def test_datetime_local_values_are_kst_and_seven_days_is_inclusive(self, _now):
-        form = self._form()
+    def test_far_future_seven_day_window_is_kst_and_inclusive(self, _now):
+        form = self._form(
+            departure_at="2026-12-01T10:00",
+            return_by="2026-12-08T10:00",
+        )
 
         self.assertTrue(form.is_valid(), form.errors)
         self.assertEqual(form.cleaned_data["departure_at"].tzinfo, SEOUL)
@@ -56,7 +59,10 @@ class TravelWindowFormTests(SimpleTestCase):
                 "window_order",
             ),
             (
-                self._form(return_by="2026-09-07T09:01"),
+                self._form(
+                    departure_at="2026-12-01T10:00",
+                    return_by="2026-12-08T10:01",
+                ),
                 "window_too_long",
             ),
             (


## `fix(privacy): narrow transient input schema boundary`

diff --git a/operations/sensitive_absence.py b/operations/sensitive_absence.py
index ad747d7..48da363 100644
--- a/operations/sensitive_absence.py
+++ b/operations/sensitive_absence.py
@@ -1328,6 +1328,9 @@ def _scan_one_database(
             schema, table, name, type_name = column[:4]
             if any("\x00" in item or len(item) > 63 for item in column[:3]):
                 raise ScanFailure()
+            if name.lower() in {"departure_at", "return_by"}:
+                connection.rollback()
+                return False, tuple(normalized), {}
             if patterns.matches("\0".join(column[:4]).encode("utf-8")):
                 connection.rollback()
                 return False, tuple(normalized), {}
diff --git a/operations/source_replay.py b/operations/source_replay.py
index 6a53ddb..1026b97 100644
--- a/operations/source_replay.py
+++ b/operations/source_replay.py
@@ -147,10 +147,11 @@ def _storage_boundary_is_clean() -> bool:
             SELECT count(*)
               FROM information_schema.columns
              WHERE table_schema = 'public'
-               AND lower(column_name) ~
-                   '(raw|secret_value|api_?key|service_?key|credential|'
-                   'destination|departure|return_date|travel_date|trip_date|'
-                   'travel_purpose)'
+               AND (
+                   lower(column_name) ~
+                       '(raw|secret_value|api_?key|service_?key|credential)'
+                   OR lower(column_name) IN ('departure_at', 'return_by')
+               )
                AND NOT (
                    table_name = 'sources_sourceconfiguration'
                    AND column_name = 'secret_reference_name'
@@ -162,14 +163,6 @@ def _storage_boundary_is_clean() -> bool:
                        'raw_retention_seconds'
                    )
                )
-               AND NOT (
-                   column_name = 'destination_airport_id'
-                   AND table_name IN (
-                       'travel_windows_flightschedule',
-                       'travel_windows_routedurationrevision',
-                       'travel_windows_flightpublicationduration'
-                   )
-               )
             """
         )
         forbidden_columns = cursor.fetchone()
diff --git a/operations/tests/test_live_parser_replay.py b/operations/tests/test_live_parser_replay.py
index fc4d152..6ee8499 100644
--- a/operations/tests/test_live_parser_replay.py
+++ b/operations/tests/test_live_parser_replay.py
@@ -280,7 +280,7 @@ class LiveParserReplayTests(SimpleTestCase):
         )
         self.assertEqual(len(_APPROVED_SOURCE_CODES), 8)
 
-    def test_storage_boundary_allows_only_the_three_typed_airport_links(self):
+    def test_storage_boundary_rejects_only_exact_transient_input_fields(self):
         fake_connection = MagicMock()
         cursor = fake_connection.cursor.return_value.__enter__.return_value
         cursor.fetchone.side_effect = ((0,), (0,))
@@ -288,13 +288,12 @@ class LiveParserReplayTests(SimpleTestCase):
             self.assertTrue(_storage_boundary_is_clean())
 
         statement = cursor.execute.call_args_list[0].args[0]
-        self.assertIn("column_name = 'destination_airport_id'", statement)
-        for table in (
-            "travel_windows_flightschedule",
-            "travel_windows_routedurationrevision",
-            "travel_windows_flightpublicationduration",
-        ):
-            self.assertIn(table, statement)
+        self.assertIn(
+            "lower(column_name) IN ('departure_at', 'return_by')",
+            statement,
+        )
+        self.assertNotIn("destination_airport_id", statement)
+        self.assertNotIn("travel_windows_flightschedule", statement)
 
     def test_disposable_database_requires_exact_name_token_and_postgresql_18_6(self):
         database_name = "travel_readiness_replaycheck_abc12345"
diff --git a/operations/tests/test_sensitive_absence.py b/operations/tests/test_sensitive_absence.py
index 8ef1529..75e993d 100644
--- a/operations/tests/test_sensitive_absence.py
+++ b/operations/tests/test_sensitive_absence.py
@@ -839,6 +839,36 @@ class DatabaseBoundaryTests(unittest.TestCase):
             )
             self.assertEqual(len(connection.streams), 3)
 
+    def test_exact_transient_input_columns_are_absent_without_airport_false_positive(self):
+        base_columns, base_rows, integrity = self.values()["travel_readiness"]
+        cases = (
+            ("destination_airport_id", True),
+            ("departure_at", False),
+            ("RETURN_BY", False),
+        )
+        for column_name, expected_clean in cases:
+            with self.subTest(column_name=column_name):
+                connector = FakeConnector(
+                    {
+                        "travel_readiness": (
+                            (*base_columns, ("public", "flight", column_name, "uuid", False, False)),
+                            [*base_rows, [("safe-airport-reference",)]],
+                            integrity,
+                        )
+                    }
+                )
+                clean, _, _ = _scan_one_database(
+                    connector=connector,
+                    host="127.0.0.1",
+                    port=5432,
+                    user="travel_readiness_backup",
+                    password=MAIN_DATABASE_PASSWORD,
+                    database="travel_readiness",
+                    patterns=patterns(),
+                    integrity_query="WITH integrity_lines AS (SELECT 1) SELECT 1",
+                )
+                self.assertEqual(clean, expected_clean)
+
     def test_synthetic_restored_database_value_fails_without_sql_literal(self):
         with tempfile.TemporaryDirectory(dir="/private/tmp") as temporary:
             backup = Path(temporary) / "backup"
diff --git a/scripts/postgresql-common b/scripts/postgresql-common
index 31d8901..3fe8aca 100644
--- a/scripts/postgresql-common
+++ b/scripts/postgresql-common
@@ -381,7 +381,7 @@ database_has_no_forbidden_storage() {
     forbidden_count=$(run_psql \
         "$connection_host" "$connection_port" "$connection_database" "$connection_username" \
         --quiet --tuples-only --no-align \
-        --command="SELECT count(*) FROM information_schema.columns WHERE table_schema = 'public' AND lower(column_name) ~ '(raw|secret_value|api_?key|service_?key|credential|destination|departure|return_date|travel_date|trip_date|travel_purpose)' AND NOT (table_name = 'sources_sourceconfiguration' AND column_name = 'secret_reference_name') AND NOT (table_name = 'sources_sourcerightsdecision' AND column_name IN ('raw_body_storage_allowed', 'raw_retention_seconds'))" \
+        --command="SELECT count(*) FROM information_schema.columns WHERE table_schema = 'public' AND (lower(column_name) ~ '(raw|secret_value|api_?key|service_?key|credential)' OR lower(column_name) IN ('departure_at', 'return_by')) AND NOT (table_name = 'sources_sourceconfiguration' AND column_name = 'secret_reference_name') AND NOT (table_name = 'sources_sourcerightsdecision' AND column_name IN ('raw_body_storage_allowed', 'raw_retention_seconds'))" \
         2>/dev/null) || return 1
     [ "$forbidden_count" = "0" ] || return 1
     large_object_count=$(run_psql \
