## `feat(source): seal reviewed series allowlist`

diff --git a/grocery/source/kamis.py b/grocery/source/kamis.py
index 0627292..150c278 100644
--- a/grocery/source/kamis.py
+++ b/grocery/source/kamis.py
@@ -357,7 +357,7 @@ def parse_recent_price_rows(
         if not isinstance(raw_row, Mapping):
             raise KamisParseError("row_not_object", row_index=row_index)
         row = _validate_row_shape(raw_row, row_index=row_index)
-        if not _is_target_scope(row):
+        if not _is_target_scope(row, identity_registry=identity_registry):
             _validate_contract_types(row, row_index=row_index)
             out_of_scope_row_hashes.append(_canonical_hash(row))
             continue
@@ -422,11 +422,17 @@ def _validate_contract_types(row: Mapping[str, object], *, row_index: int) -> No
         _parse_reference(period, row[field], row_index=row_index, field=field)
 
 
-def _is_target_scope(row: Mapping[str, object]) -> bool:
-    return (
-        row["se_cd"] == KAMIS_RETAIL_PRODUCT_CLASS_CODE
-        and row["ctgry_cd"] in KAMIS_ALLOWED_CATEGORIES
-    )
+def _is_target_scope(
+    row: Mapping[str, object],
+    *,
+    identity_registry: ExactIdentityRegistry,
+) -> bool:
+    if row["se_cd"] != KAMIS_RETAIL_PRODUCT_CLASS_CODE:
+        return False
+    if row["ctgry_cd"] not in KAMIS_ALLOWED_CATEGORIES:
+        return False
+    series_key = (row["ctgry_cd"], row["item_cd"], row["vrty_cd"], row["grd_cd"])
+    return series_key in identity_registry.units
 
 
 def _parse_row(
diff --git a/grocery/source/registry.py b/grocery/source/registry.py
new file mode 100644
index 0000000..b131404
--- /dev/null
+++ b/grocery/source/registry.py
@@ -0,0 +1,89 @@
+"""Reviewed initial KAMIS retail series allowlist.
+
+These ten normalized identity facts were checked against the official code workbook
+and the bounded live canary on 2026-08-30. They are not a copy of an API response.
+Adding a series requires a new reviewed evidence revision.
+"""
+
+import hashlib
+import json
+
+from grocery.source.kamis import (
+    IdentityContractEvidence,
+    build_identity_registry_from_reviewed_evidence,
+)
+
+OFFICIAL_DOCS_ZIP_SHA256 = "07417ea9eb882a33615721256ff8be3b131cdb10bbc9c7b40472bf049a7e0f88"
+IDENTITY_EVIDENCE_REVISION = "15156063-codebook-live-canary-2026-08-30-v1"
+
+ITEM_NAMES = {
+    ("200", "212"): "양배추",
+    ("200", "213"): "시금치",
+    ("200", "214"): "상추",
+    ("200", "215"): "얼갈이배추",
+    ("400", "411"): "사과",
+    ("400", "414"): "포도",
+    ("400", "419"): "참다래",
+    ("400", "420"): "파인애플",
+    ("400", "430"): "아보카도",
+}
+
+VARIETY_NAMES = {
+    ("200", "212", "00"): "양배추",
+    ("200", "213", "00"): "시금치",
+    ("200", "214", "01"): "적",
+    ("200", "214", "02"): "청",
+    ("200", "215", "00"): "얼갈이배추",
+    ("400", "411", "06"): "쓰가루(아오리)",
+    ("400", "414", "12"): "샤인머스켓",
+    ("400", "419", "02"): "그린 뉴질랜드",
+    ("400", "420", "02"): "수입",
+    ("400", "430", "00"): "수입",
+}
+
+GRADE_NAMES = {
+    ("200", "212", "00", "04"): "상품",
+    ("200", "213", "00", "04"): "상품",
+    ("200", "214", "01", "04"): "상품",
+    ("200", "214", "02", "04"): "상품",
+    ("200", "215", "00", "04"): "상품",
+    ("400", "411", "06", "04"): "상품",
+    ("400", "414", "12", "24"): "L과",
+    ("400", "419", "02", "04"): "상품",
+    ("400", "420", "02", "04"): "상품",
+    ("400", "430", "00", "04"): "상품",
+}
+
+UNITS = {
+    ("200", "212", "00", "04"): ("포기", "1"),
+    ("200", "213", "00", "04"): ("g", "100"),
+    ("200", "214", "01", "04"): ("g", "100"),
+    ("200", "214", "02", "04"): ("g", "100"),
+    ("200", "215", "00", "04"): ("kg", "1"),
+    ("400", "411", "06", "04"): ("개", "10"),
+    ("400", "414", "12", "24"): ("kg", "2"),
+    ("400", "419", "02", "04"): ("개", "10"),
+    ("400", "420", "02", "04"): ("개", "1"),
+    ("400", "430", "00", "04"): ("개", "1"),
+}
+
+
+def _unit_contract_hash() -> str:
+    rows = [
+        [*series_key, unit, unit_size] for series_key, (unit, unit_size) in sorted(UNITS.items())
+    ]
+    canonical = json.dumps(rows, ensure_ascii=False, separators=(",", ":")).encode()
+    return hashlib.sha256(canonical).hexdigest()
+
+
+INITIAL_RETAIL_IDENTITY_REGISTRY = build_identity_registry_from_reviewed_evidence(
+    item_names=ITEM_NAMES,
+    variety_names=VARIETY_NAMES,
+    grade_names=GRADE_NAMES,
+    units=UNITS,
+    evidence=IdentityContractEvidence(
+        codebook_sha256=OFFICIAL_DOCS_ZIP_SHA256,
+        unit_contract_sha256=_unit_contract_hash(),
+        coverage_evidence_revision=IDENTITY_EVIDENCE_REVISION,
+    ),
+)
diff --git a/grocery/tests/test_kamis_parser.py b/grocery/tests/test_kamis_parser.py
index 2c31ce8..c204016 100644
--- a/grocery/tests/test_kamis_parser.py
+++ b/grocery/tests/test_kamis_parser.py
@@ -283,6 +283,25 @@ def test_out_of_scope_rows_are_reconciled_without_becoming_facts(
     assert result.input_row_count == result.accepted_row_count + result.out_of_scope_row_count
 
 
+def test_unreviewed_retail_series_is_out_of_scope(
+    identity_registry: ExactIdentityRegistry,
+) -> None:
+    row = deepcopy(SYNTHETIC_ROW)
+    row.update(
+        {
+            "item_cd": "999",
+            "item_nm": "검토되지않은합성품목",
+            "vrty_cd": "01",
+            "vrty_nm": "검토되지않은합성품종",
+        }
+    )
+
+    result = parse_recent_price_rows([row], identity_registry=identity_registry)
+
+    assert result.accepted_row_count == 0
+    assert result.out_of_scope_row_count == 1
+
+
 def test_out_of_scope_order_does_not_change_result_hash(
     identity_registry: ExactIdentityRegistry,
 ) -> None:
diff --git a/grocery/tests/test_kamis_registry.py b/grocery/tests/test_kamis_registry.py
new file mode 100644
index 0000000..d7dc5e9
--- /dev/null
+++ b/grocery/tests/test_kamis_registry.py
@@ -0,0 +1,26 @@
+from grocery.source.registry import (
+    INITIAL_RETAIL_IDENTITY_REGISTRY,
+    OFFICIAL_DOCS_ZIP_SHA256,
+)
+
+
+def test_initial_registry_contains_exactly_reviewed_five_plus_five_series() -> None:
+    registry = INITIAL_RETAIL_IDENTITY_REGISTRY
+
+    vegetable_count = sum(key[0] == "200" for key in registry.units)
+    fruit_count = sum(key[0] == "400" for key in registry.units)
+
+    assert vegetable_count == 5
+    assert fruit_count == 5
+    assert len(registry.units) == 10
+    assert registry.evidence.codebook_sha256 == OFFICIAL_DOCS_ZIP_SHA256
+
+
+def test_reviewed_units_and_names_preserve_exact_identity() -> None:
+    registry = INITIAL_RETAIL_IDENTITY_REGISTRY
+
+    assert registry.item_names[("400", "414")] == "포도"
+    assert registry.variety_names[("400", "414", "12")] == "샤인머스켓"
+    assert registry.grade_names[("400", "414", "12", "24")] == "L과"
+    assert registry.units[("400", "414", "12", "24")] == ("kg", "2")
+    assert registry.units[("200", "212", "00", "04")] == ("포기", "1")


