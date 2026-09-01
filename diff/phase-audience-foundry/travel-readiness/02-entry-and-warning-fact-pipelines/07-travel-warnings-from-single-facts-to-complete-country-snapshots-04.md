## `fix(warnings): accept official response variants`

diff --git a/travel_warnings/parser.py b/travel_warnings/parser.py
index 717bfa4..c9c9b7c 100644
--- a/travel_warnings/parser.py
+++ b/travel_warnings/parser.py
@@ -61,11 +61,32 @@ _WRITTEN_DATE_PATH = (
     "[]",
     "written_dt",
 )
+_NULLABLE_LINK_PATHS = frozenset(
+    {
+        (
+            "response",
+            "body",
+            "items",
+            "item",
+            "[]",
+            "flag_download_url",
+        ),
+        (
+            "response",
+            "body",
+            "items",
+            "item",
+            "[]",
+            "map_download_url",
+        ),
+    }
+)
 _SUPPORTED_COUNTRY_IDENTITIES = {
     iso_alpha2: (name_ko, name_en)
     for _country_id, iso_alpha2, name_ko, name_en, _is_public
     in SUPPORTED_COUNTRY_ROWS
 }
+_SUPPORTED_COUNTRY_IDENTITIES["HK"] = ("홍콩", "Hongkong")
 
 
 class _DuplicateKey(ValueError):
@@ -115,6 +136,10 @@ def _reject_non_json_constant(_: str) -> None:
 def _schema_shape(value: Any, path: tuple[str, ...] = ()) -> Any:
     if path == _WRITTEN_DATE_PATH and (value is None or isinstance(value, str)):
         return "null"
+    if path in _NULLABLE_LINK_PATHS and (
+        value is None or isinstance(value, str)
+    ):
+        return "string"
     if isinstance(value, dict):
         return {
             key: _schema_shape(child, (*path, key))
diff --git a/travel_warnings/tests/test_parser.py b/travel_warnings/tests/test_parser.py
index a05540a..042ccdf 100644
--- a/travel_warnings/tests/test_parser.py
+++ b/travel_warnings/tests/test_parser.py
@@ -96,6 +96,21 @@ class TravelAlarmParserTests(SimpleTestCase):
             EXPECTED_SCHEMA_FINGERPRINT_SHA256,
         )
 
+    def test_optional_provider_links_accept_null_without_schema_drift(self):
+        for field in ("flag_download_url", "map_download_url"):
+            with self.subTest(field=field):
+                result = parse_travel_alarm_jp(
+                    self.encode(
+                        self.document(item=self.item(**{field: None}))
+                    )
+                )
+
+                self.assertIsInstance(result, TravelWarningParseSuccess)
+                self.assertEqual(
+                    result.observed_schema_fingerprint_sha256,
+                    EXPECTED_SCHEMA_FINGERPRINT_SHA256,
+                )
+
     def test_projection_and_typed_hash_are_exact_and_deterministic(self):
         document = self.document()
         compact = parse_travel_alarm_jp(self.encode(document))
@@ -153,6 +168,24 @@ class TravelAlarmParserTests(SimpleTestCase):
         self.assertEqual(result.warning.country_name_ko, "대만")
         self.assertEqual(result.warning.country_name_en, "Taiwan")
 
+    def test_supported_hk_identity_uses_exact_provider_english_name(self):
+        payload = self.encode(
+            self.document(
+                item=self.item(
+                    country_iso_alp2="HK",
+                    country_nm="홍콩",
+                    country_eng_nm="Hongkong",
+                    map_download_url=None,
+                )
+            )
+        )
+
+        result = parse_travel_alarm_jp(payload, country_iso2="HK")
+
+        self.assertIsInstance(result, TravelWarningParseSuccess)
+        self.assertEqual(result.warning.country_iso2, "HK")
+        self.assertEqual(result.warning.country_name_en, "Hongkong")
+
     def test_provider_error_and_unknown_envelope_fail_closed(self):
         expected = EXPECTED_SCHEMA_FINGERPRINT_SHA256
         self.assert_failure(


