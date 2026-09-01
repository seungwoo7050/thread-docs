# 지역·시장 일별 원장

## `feat(source): parse regional daily history`

diff --git a/grocery/source/__init__.py b/grocery/source/__init__.py
index 9dfe591..9db9981 100644
--- a/grocery/source/__init__.py
+++ b/grocery/source/__init__.py
@@ -16,6 +16,11 @@ from grocery.source.monthly_history import (
     ParsedMonthlyPriceRow,
     parse_monthly_price_rows,
 )
+from grocery.source.regional_history import (
+    KAMIS_REGIONAL_PRICE_FIELDS,
+    ParsedRegionalPriceRow,
+    parse_regional_price_rows,
+)
 
 __all__ = [
     "KAMIS_RETAIL_COVERAGE_IDENTITY",
@@ -23,12 +28,15 @@ __all__ = [
     "HistoricalDimensionRegistry",
     "IdentityContractEvidence",
     "KAMIS_MONTHLY_PRICE_FIELDS",
+    "KAMIS_REGIONAL_PRICE_FIELDS",
     "KamisParseError",
     "ParsedMonthlyPriceRow",
+    "ParsedRegionalPriceRow",
     "ParsedRecentPriceResult",
     "ParsedRetailPriceRow",
     "build_identity_registry_from_reviewed_evidence",
     "parse_monthly_price_rows",
     "parse_recent_price_rows",
+    "parse_regional_price_rows",
     "YearMonth",
 ]
diff --git a/grocery/source/regional_history.py b/grocery/source/regional_history.py
new file mode 100644
index 0000000..13beabf
--- /dev/null
+++ b/grocery/source/regional_history.py
@@ -0,0 +1,161 @@
+"""Strict typed parser for public-data API 15156062 regional daily rows."""
+
+from __future__ import annotations
+
+from dataclasses import dataclass
+from datetime import date
+from decimal import Decimal
+
+from grocery.source.historical_dimensions import HistoricalDimensionRegistry, RegionObservation
+from grocery.source.historical_parser import (
+    HistoricalRowValidator,
+    ParsedHistoricalResult,
+    canonical_hash,
+    decimal_text,
+    identity_data,
+    require_items,
+    require_price_range,
+)
+from grocery.source.kamis import IdentityObservation, KamisParseError
+
+KAMIS_REGIONAL_PRICE_FIELDS = frozenset(
+    {
+        "exmn_ymd",
+        "se_cd",
+        "se_nm",
+        "ctgry_cd",
+        "ctgry_nm",
+        "item_cd",
+        "item_nm",
+        "vrty_cd",
+        "vrty_nm",
+        "grd_cd",
+        "grd_nm",
+        "sgg_cd",
+        "sgg_nm",
+        "unit",
+        "unit_sz",
+        "exmn_dd_min_prc",
+        "exmn_dd_cnvs_min_prc",
+        "exmn_dd_avg_prc",
+        "exmn_dd_cnvs_avg_prc",
+        "exmn_dd_max_prc",
+        "exmn_dd_cnvs_max_prc",
+    }
+)
+
+type RegionalSemanticKey = tuple[str, str, str, str, str, str, str, str, str, date]
+
+
+@dataclass(frozen=True, slots=True)
+class ParsedRegionalPriceRow:
+    identity: IdentityObservation
+    region: RegionObservation
+    source_effective_date: date
+    raw_min_price: Decimal
+    raw_average_price: Decimal
+    raw_max_price: Decimal
+    converted_min_price: Decimal
+    converted_average_price: Decimal
+    converted_max_price: Decimal
+    source_row_hash: str
+
+    @property
+    def semantic_key(self) -> RegionalSemanticKey:
+        return (
+            self.identity.product_class_code,
+            self.identity.category_code,
+            self.identity.item_code,
+            self.identity.variety_code,
+            self.identity.grade_code,
+            self.identity.raw_unit,
+            self.identity.raw_unit_size,
+            self.identity.coverage_identity,
+            self.region.code,
+            self.source_effective_date,
+        )
+
+    def canonical_data(self) -> dict[str, object]:
+        return {
+            "exmn_dd_avg_prc": decimal_text(self.raw_average_price),
+            "exmn_dd_cnvs_avg_prc": decimal_text(self.converted_average_price),
+            "exmn_dd_cnvs_max_prc": decimal_text(self.converted_max_price),
+            "exmn_dd_cnvs_min_prc": decimal_text(self.converted_min_price),
+            "exmn_dd_max_prc": decimal_text(self.raw_max_price),
+            "exmn_dd_min_prc": decimal_text(self.raw_min_price),
+            "identity": identity_data(self.identity),
+            "region": {"code": self.region.code, "name": self.region.name},
+            "source_effective_date": self.source_effective_date.isoformat(),
+            "source_row_hash": self.source_row_hash,
+        }
+
+
+def parse_regional_price_rows(
+    items: object,
+    *,
+    registry: HistoricalDimensionRegistry,
+) -> ParsedHistoricalResult[ParsedRegionalPriceRow]:
+    """Parse only exact 21-field 15156062 rows without reconstructing an average."""
+
+    source_items = require_items(items)
+    parsed_rows: list[ParsedRegionalPriceRow] = []
+    seen: set[RegionalSemanticKey] = set()
+    for row_index, raw_row in enumerate(source_items):
+        validator = HistoricalRowValidator(
+            raw_row,
+            row_index=row_index,
+            expected_fields=KAMIS_REGIONAL_PRICE_FIELDS,
+            registry=registry,
+        )
+        parsed = _parse_row(validator)
+        if parsed.semantic_key in seen:
+            raise KamisParseError("duplicate_semantic_identity", row_index=row_index)
+        seen.add(parsed.semantic_key)
+        parsed_rows.append(parsed)
+
+    ordered_rows = tuple(sorted(parsed_rows, key=lambda row: row.semantic_key))
+    return ParsedHistoricalResult(
+        rows=ordered_rows,
+        input_row_count=len(source_items),
+        result_hash=canonical_hash(
+            {
+                "parser_contract": "kamis-15156062-v1",
+                "rows": [row.canonical_data() for row in ordered_rows],
+            }
+        ),
+    )
+
+
+def _parse_row(validator: HistoricalRowValidator) -> ParsedRegionalPriceRow:
+    raw_min = validator.positive_decimal("exmn_dd_min_prc")
+    raw_average = validator.positive_decimal("exmn_dd_avg_prc")
+    raw_max = validator.positive_decimal("exmn_dd_max_prc")
+    converted_min = validator.positive_decimal("exmn_dd_cnvs_min_prc")
+    converted_average = validator.positive_decimal("exmn_dd_cnvs_avg_prc")
+    converted_max = validator.positive_decimal("exmn_dd_cnvs_max_prc")
+    require_price_range(
+        raw_min,
+        raw_average,
+        raw_max,
+        row_index=validator.row_index,
+        field="exmn_dd_avg_prc",
+    )
+    require_price_range(
+        converted_min,
+        converted_average,
+        converted_max,
+        row_index=validator.row_index,
+        field="exmn_dd_cnvs_avg_prc",
+    )
+    return ParsedRegionalPriceRow(
+        identity=validator.identity(),
+        region=validator.region(),
+        source_effective_date=validator.day(),
+        raw_min_price=raw_min,
+        raw_average_price=raw_average,
+        raw_max_price=raw_max,
+        converted_min_price=converted_min,
+        converted_average_price=converted_average,
+        converted_max_price=converted_max,
+        source_row_hash=canonical_hash(validator.row),
+    )
diff --git a/grocery/tests/historical_fixtures.py b/grocery/tests/historical_fixtures.py
index 9ea5df0..daa803f 100644
--- a/grocery/tests/historical_fixtures.py
+++ b/grocery/tests/historical_fixtures.py
@@ -47,3 +47,29 @@ def monthly_row() -> dict[str, str]:
         "pyy_cfcntrng": "400",
         "orgnl_reg_dt": "2026-08-31 12:00:00",
     }
+
+
+def regional_row() -> dict[str, str]:
+    return {
+        "exmn_ymd": "20260831",
+        "se_cd": "01",
+        "se_nm": "소매",
+        "ctgry_cd": "200",
+        "ctgry_nm": "채소류",
+        "item_cd": "212",
+        "item_nm": "양배추",
+        "vrty_cd": "00",
+        "vrty_nm": "양배추",
+        "grd_cd": "04",
+        "grd_nm": "상품",
+        "sgg_cd": "11000",
+        "sgg_nm": "서울",
+        "unit": "포기",
+        "unit_sz": "1",
+        "exmn_dd_min_prc": "800",
+        "exmn_dd_cnvs_min_prc": "80.5",
+        "exmn_dd_avg_prc": "1000",
+        "exmn_dd_cnvs_avg_prc": "100.25",
+        "exmn_dd_max_prc": "1200",
+        "exmn_dd_cnvs_max_prc": "120.75",
+    }
diff --git a/grocery/tests/test_regional_history.py b/grocery/tests/test_regional_history.py
new file mode 100644
index 0000000..92d5e4d
--- /dev/null
+++ b/grocery/tests/test_regional_history.py
@@ -0,0 +1,99 @@
+"""Synthetic strict parser tests for public-data API 15156062."""
+
+from decimal import Decimal
+
+import pytest
+
+from grocery.source.kamis import KamisParseError
+from grocery.source.regional_history import (
+    KAMIS_REGIONAL_PRICE_FIELDS,
+    parse_regional_price_rows,
+)
+from grocery.tests.historical_fixtures import historical_registry, regional_row
+
+
+def test_regional_row_preserves_raw_and_converted_provider_ranges() -> None:
+    result = parse_regional_price_rows([regional_row()], registry=historical_registry())
+
+    assert len(KAMIS_REGIONAL_PRICE_FIELDS) == 21
+    row = result.rows[0]
+    assert row.source_effective_date.isoformat() == "2026-08-31"
+    assert (row.raw_min_price, row.raw_average_price, row.raw_max_price) == (
+        Decimal("800"),
+        Decimal("1000"),
+        Decimal("1200"),
+    )
+    assert (
+        row.converted_min_price,
+        row.converted_average_price,
+        row.converted_max_price,
+    ) == (Decimal("80.5"), Decimal("100.25"), Decimal("120.75"))
+    assert len(result.result_hash) == len(row.source_row_hash) == 64
+
+
+@pytest.mark.parametrize("missing_field", sorted(KAMIS_REGIONAL_PRICE_FIELDS))
+def test_every_missing_regional_field_fails_closed(missing_field: str) -> None:
+    row = regional_row()
+    del row[missing_field]
+
+    with pytest.raises(KamisParseError, match=rf"missing_field .*field={missing_field}"):
+        parse_regional_price_rows([row], registry=historical_registry())
+
+
+@pytest.mark.parametrize(
+    ("field", "value", "error_code"),
+    [
+        ("exmn_ymd", "20260230", "invalid_source_date"),
+        ("sgg_nm", "다른지역", "region_code_name_drift"),
+        ("item_nm", "다른품목", "item_code_name_drift"),
+        ("exmn_dd_avg_prc", "0", "invalid_positive_decimal"),
+        ("exmn_dd_cnvs_avg_prc", "1,000", "invalid_decimal"),
+    ],
+)
+def test_regional_identity_date_and_decimal_drift_fails(
+    field: str,
+    value: str,
+    error_code: str,
+) -> None:
+    row = regional_row()
+    row[field] = value
+
+    with pytest.raises(KamisParseError, match=error_code):
+        parse_regional_price_rows([row], registry=historical_registry())
+
+
+@pytest.mark.parametrize(
+    ("field", "value"),
+    [
+        ("exmn_dd_min_prc", "1100"),
+        ("exmn_dd_max_prc", "900"),
+        ("exmn_dd_cnvs_avg_prc", "130"),
+    ],
+)
+def test_raw_and_converted_range_inversions_fail_independently(field: str, value: str) -> None:
+    row = regional_row()
+    row[field] = value
+
+    with pytest.raises(KamisParseError, match="invalid_price_range"):
+        parse_regional_price_rows([row], registry=historical_registry())
+
+
+def test_regional_result_is_order_stable_and_duplicate_safe() -> None:
+    next_day = regional_row()
+    next_day["exmn_ymd"] = "20260830"
+    left = parse_regional_price_rows(
+        [regional_row(), next_day], registry=historical_registry()
+    )
+    right = parse_regional_price_rows(
+        [dict(reversed(tuple(next_day.items()))), regional_row()],
+        registry=historical_registry(),
+    )
+    assert left == right
+
+    changed = regional_row()
+    changed["exmn_dd_avg_prc"] = "1001"
+    with pytest.raises(KamisParseError, match="duplicate_semantic_identity"):
+        parse_regional_price_rows(
+            [regional_row(), changed],
+            registry=historical_registry(),
+        )


## `feat(source): parse market daily history`

diff --git a/grocery/source/__init__.py b/grocery/source/__init__.py
index 9db9981..aaa8348 100644
--- a/grocery/source/__init__.py
+++ b/grocery/source/__init__.py
@@ -11,6 +11,11 @@ from grocery.source.kamis import (
     build_identity_registry_from_reviewed_evidence,
     parse_recent_price_rows,
 )
+from grocery.source.market_history import (
+    KAMIS_MARKET_PRICE_FIELDS,
+    ParsedMarketPriceRow,
+    parse_market_price_rows,
+)
 from grocery.source.monthly_history import (
     KAMIS_MONTHLY_PRICE_FIELDS,
     ParsedMonthlyPriceRow,
@@ -28,14 +33,17 @@ __all__ = [
     "HistoricalDimensionRegistry",
     "IdentityContractEvidence",
     "KAMIS_MONTHLY_PRICE_FIELDS",
+    "KAMIS_MARKET_PRICE_FIELDS",
     "KAMIS_REGIONAL_PRICE_FIELDS",
     "KamisParseError",
     "ParsedMonthlyPriceRow",
+    "ParsedMarketPriceRow",
     "ParsedRegionalPriceRow",
     "ParsedRecentPriceResult",
     "ParsedRetailPriceRow",
     "build_identity_registry_from_reviewed_evidence",
     "parse_monthly_price_rows",
+    "parse_market_price_rows",
     "parse_recent_price_rows",
     "parse_regional_price_rows",
     "YearMonth",
diff --git a/grocery/source/market_history.py b/grocery/source/market_history.py
new file mode 100644
index 0000000..f10039c
--- /dev/null
+++ b/grocery/source/market_history.py
@@ -0,0 +1,139 @@
+"""Strict typed parser for public-data API 15156065 market daily rows."""
+
+from __future__ import annotations
+
+from dataclasses import dataclass
+from datetime import date
+from decimal import Decimal
+
+from grocery.source.historical_dimensions import (
+    HistoricalDimensionRegistry,
+    MarketObservation,
+    RegionObservation,
+)
+from grocery.source.historical_parser import (
+    HistoricalRowValidator,
+    ParsedHistoricalResult,
+    canonical_hash,
+    decimal_text,
+    identity_data,
+    require_items,
+)
+from grocery.source.kamis import IdentityObservation, KamisParseError
+
+KAMIS_MARKET_PRICE_FIELDS = frozenset(
+    {
+        "exmn_ymd",
+        "se_cd",
+        "se_nm",
+        "ctgry_cd",
+        "ctgry_nm",
+        "item_cd",
+        "item_nm",
+        "vrty_cd",
+        "vrty_nm",
+        "grd_cd",
+        "grd_nm",
+        "sgg_cd",
+        "sgg_nm",
+        "unit",
+        "unit_sz",
+        "mrkt_cd",
+        "mrkt_nm",
+        "exmn_dd_prc",
+        "exmn_dd_cnvs_prc",
+        "orgnl_reg_dt",
+    }
+)
+
+type MarketSemanticKey = tuple[str, str, str, str, str, str, str, str, str, str, date]
+
+
+@dataclass(frozen=True, slots=True)
+class ParsedMarketPriceRow:
+    identity: IdentityObservation
+    region: RegionObservation
+    market: MarketObservation
+    source_effective_date: date
+    raw_observed_price: Decimal
+    converted_observed_price: Decimal
+    source_recorded_at_raw: str
+    source_row_hash: str
+
+    @property
+    def semantic_key(self) -> MarketSemanticKey:
+        return (
+            self.identity.product_class_code,
+            self.identity.category_code,
+            self.identity.item_code,
+            self.identity.variety_code,
+            self.identity.grade_code,
+            self.identity.raw_unit,
+            self.identity.raw_unit_size,
+            self.identity.coverage_identity,
+            self.region.code,
+            self.market.code,
+            self.source_effective_date,
+        )
+
+    def canonical_data(self) -> dict[str, object]:
+        return {
+            "exmn_dd_cnvs_prc": decimal_text(self.converted_observed_price),
+            "exmn_dd_prc": decimal_text(self.raw_observed_price),
+            "identity": identity_data(self.identity),
+            "market": {"code": self.market.code, "name": self.market.name},
+            "region": {"code": self.region.code, "name": self.region.name},
+            "source_effective_date": self.source_effective_date.isoformat(),
+            "source_recorded_at_raw": self.source_recorded_at_raw,
+            "source_row_hash": self.source_row_hash,
+        }
+
+
+def parse_market_price_rows(
+    items: object,
+    *,
+    registry: HistoricalDimensionRegistry,
+) -> ParsedHistoricalResult[ParsedMarketPriceRow]:
+    """Parse exact 20-field 15156065 observations without inferring market type."""
+
+    source_items = require_items(items)
+    parsed_rows: list[ParsedMarketPriceRow] = []
+    seen: set[MarketSemanticKey] = set()
+    for row_index, raw_row in enumerate(source_items):
+        validator = HistoricalRowValidator(
+            raw_row,
+            row_index=row_index,
+            expected_fields=KAMIS_MARKET_PRICE_FIELDS,
+            registry=registry,
+        )
+        parsed = _parse_row(validator)
+        if parsed.semantic_key in seen:
+            raise KamisParseError("duplicate_semantic_identity", row_index=row_index)
+        seen.add(parsed.semantic_key)
+        parsed_rows.append(parsed)
+
+    ordered_rows = tuple(sorted(parsed_rows, key=lambda row: row.semantic_key))
+    return ParsedHistoricalResult(
+        rows=ordered_rows,
+        input_row_count=len(source_items),
+        result_hash=canonical_hash(
+            {
+                "parser_contract": "kamis-15156065-v1",
+                "rows": [row.canonical_data() for row in ordered_rows],
+            }
+        ),
+    )
+
+
+def _parse_row(validator: HistoricalRowValidator) -> ParsedMarketPriceRow:
+    region = validator.region()
+    return ParsedMarketPriceRow(
+        identity=validator.identity(),
+        region=region,
+        market=validator.market(region),
+        source_effective_date=validator.day(),
+        raw_observed_price=validator.positive_decimal("exmn_dd_prc"),
+        converted_observed_price=validator.positive_decimal("exmn_dd_cnvs_prc"),
+        source_recorded_at_raw=validator.text("orgnl_reg_dt", maximum=64),
+        source_row_hash=canonical_hash(validator.row),
+    )
diff --git a/grocery/tests/historical_fixtures.py b/grocery/tests/historical_fixtures.py
index daa803f..7c23b33 100644
--- a/grocery/tests/historical_fixtures.py
+++ b/grocery/tests/historical_fixtures.py
@@ -73,3 +73,28 @@ def regional_row() -> dict[str, str]:
         "exmn_dd_max_prc": "1200",
         "exmn_dd_cnvs_max_prc": "120.75",
     }
+
+
+def market_row() -> dict[str, str]:
+    return {
+        "exmn_ymd": "20260831",
+        "se_cd": "01",
+        "se_nm": "소매",
+        "ctgry_cd": "200",
+        "ctgry_nm": "채소류",
+        "item_cd": "212",
+        "item_nm": "양배추",
+        "vrty_cd": "00",
+        "vrty_nm": "양배추",
+        "grd_cd": "04",
+        "grd_nm": "상품",
+        "sgg_cd": "11000",
+        "sgg_nm": "서울",
+        "unit": "포기",
+        "unit_sz": "1",
+        "mrkt_cd": "110001",
+        "mrkt_nm": "합성서울시장",
+        "exmn_dd_prc": "1000.50",
+        "exmn_dd_cnvs_prc": "77.25",
+        "orgnl_reg_dt": "2026-08-31 12:00:00",
+    }
diff --git a/grocery/tests/test_market_history.py b/grocery/tests/test_market_history.py
new file mode 100644
index 0000000..0fbec41
--- /dev/null
+++ b/grocery/tests/test_market_history.py
@@ -0,0 +1,82 @@
+"""Synthetic strict parser tests for public-data API 15156065."""
+
+from decimal import Decimal
+
+import pytest
+
+from grocery.source.kamis import KamisParseError
+from grocery.source.market_history import KAMIS_MARKET_PRICE_FIELDS, parse_market_price_rows
+from grocery.tests.historical_fixtures import historical_registry, market_row
+
+
+def test_market_row_preserves_observed_values_without_market_inference() -> None:
+    result = parse_market_price_rows([market_row()], registry=historical_registry())
+
+    assert len(KAMIS_MARKET_PRICE_FIELDS) == 20
+    row = result.rows[0]
+    assert row.source_effective_date.isoformat() == "2026-08-31"
+    assert (row.region.code, row.market.code) == ("11000", "110001")
+    assert row.raw_observed_price == Decimal("1000.50")
+    assert row.converted_observed_price == Decimal("77.25")
+    assert row.source_recorded_at_raw == "2026-08-31 12:00:00"
+    assert not {"market_type", "computed_average", "trend"} & row.canonical_data().keys()
+    assert len(result.result_hash) == len(row.source_row_hash) == 64
+
+
+@pytest.mark.parametrize("missing_field", sorted(KAMIS_MARKET_PRICE_FIELDS))
+def test_every_missing_market_field_fails_closed(missing_field: str) -> None:
+    row = market_row()
+    del row[missing_field]
+
+    with pytest.raises(KamisParseError, match=rf"missing_field .*field={missing_field}"):
+        parse_market_price_rows([row], registry=historical_registry())
+
+
+@pytest.mark.parametrize(
+    ("field", "value", "error_code"),
+    [
+        ("exmn_ymd", "20260832", "invalid_source_date"),
+        ("sgg_nm", "다른지역", "region_code_name_drift"),
+        ("mrkt_nm", "다른시장", "market_code_name_drift"),
+        ("mrkt_cd", "260001", "market_code_name_drift"),
+        ("exmn_dd_prc", "0", "invalid_positive_decimal"),
+        ("exmn_dd_cnvs_prc", "+77", "invalid_decimal"),
+        ("orgnl_reg_dt", "2026-08-31\u0000", "invalid_source_text"),
+    ],
+)
+def test_market_dimension_date_decimal_and_text_drift_fails(
+    field: str,
+    value: str,
+    error_code: str,
+) -> None:
+    row = market_row()
+    row[field] = value
+
+    with pytest.raises(KamisParseError, match=error_code):
+        parse_market_price_rows([row], registry=historical_registry())
+
+
+def test_market_result_is_order_stable_and_duplicate_safe() -> None:
+    busan = market_row()
+    busan.update(
+        {
+            "sgg_cd": "26000",
+            "sgg_nm": "부산",
+            "mrkt_cd": "260001",
+            "mrkt_nm": "합성부산시장",
+        }
+    )
+    left = parse_market_price_rows([market_row(), busan], registry=historical_registry())
+    right = parse_market_price_rows(
+        [dict(reversed(tuple(busan.items()))), market_row()],
+        registry=historical_registry(),
+    )
+    assert left == right
+
+    changed = market_row()
+    changed["exmn_dd_prc"] = "1001"
+    with pytest.raises(KamisParseError, match="duplicate_semantic_identity"):
+        parse_market_price_rows(
+            [market_row(), changed],
+            registry=historical_registry(),
+        )


