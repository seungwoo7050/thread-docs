# 월별 이력 완전성과 결측 차트

## `feat(source): parse monthly retail history`

diff --git a/grocery/source/__init__.py b/grocery/source/__init__.py
index b9ba1ea..9dfe591 100644
--- a/grocery/source/__init__.py
+++ b/grocery/source/__init__.py
@@ -1,5 +1,6 @@
 """Source-specific, fail-closed parsers."""
 
+from grocery.source.historical_dimensions import HistoricalDimensionRegistry, YearMonth
 from grocery.source.kamis import (
     KAMIS_RETAIL_COVERAGE_IDENTITY,
     ExactIdentityRegistry,
@@ -10,14 +11,24 @@ from grocery.source.kamis import (
     build_identity_registry_from_reviewed_evidence,
     parse_recent_price_rows,
 )
+from grocery.source.monthly_history import (
+    KAMIS_MONTHLY_PRICE_FIELDS,
+    ParsedMonthlyPriceRow,
+    parse_monthly_price_rows,
+)
 
 __all__ = [
     "KAMIS_RETAIL_COVERAGE_IDENTITY",
     "ExactIdentityRegistry",
+    "HistoricalDimensionRegistry",
     "IdentityContractEvidence",
+    "KAMIS_MONTHLY_PRICE_FIELDS",
     "KamisParseError",
+    "ParsedMonthlyPriceRow",
     "ParsedRecentPriceResult",
     "ParsedRetailPriceRow",
     "build_identity_registry_from_reviewed_evidence",
+    "parse_monthly_price_rows",
     "parse_recent_price_rows",
+    "YearMonth",
 ]
diff --git a/grocery/source/monthly_history.py b/grocery/source/monthly_history.py
new file mode 100644
index 0000000..b9bd46e
--- /dev/null
+++ b/grocery/source/monthly_history.py
@@ -0,0 +1,195 @@
+"""Strict typed parser for public-data API 15156060 monthly retail rows."""
+
+from __future__ import annotations
+
+from dataclasses import dataclass
+from decimal import Decimal
+
+from grocery.source.historical_dimensions import (
+    HistoricalDimensionRegistry,
+    RegionObservation,
+    YearMonth,
+)
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
+KAMIS_MONTHLY_PRICE_FIELDS = frozenset(
+    {
+        "exmn_ym",
+        "sgg_cd",
+        "sgg_nm",
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
+        "unit",
+        "unit_sz",
+        "pmm_avgprc",
+        "pmm_hgprc",
+        "pmm_lwprc",
+        "pmm_stddvtn",
+        "pmm_cfcntvrtn",
+        "pmm_cfcntrng",
+        "pyy_avgprc",
+        "pyy_hgprc",
+        "pyy_lwprc",
+        "pyy_stddvtn",
+        "pyy_cfcntvrtn",
+        "pyy_cfcntrng",
+        "orgnl_reg_dt",
+    }
+)
+
+type MonthlySemanticKey = tuple[str, str, str, str, str, str, str, str, str, str]
+
+
+@dataclass(frozen=True, slots=True)
+class ParsedMonthlyPriceRow:
+    identity: IdentityObservation
+    region: RegionObservation
+    source_effective_month: YearMonth
+    pmm_avgprc: Decimal
+    pmm_hgprc: Decimal
+    pmm_lwprc: Decimal
+    pmm_stddvtn: Decimal
+    pmm_cfcntvrtn: Decimal
+    pmm_cfcntrng: Decimal
+    pyy_avgprc: Decimal
+    pyy_hgprc: Decimal
+    pyy_lwprc: Decimal
+    pyy_stddvtn: Decimal
+    pyy_cfcntvrtn: Decimal
+    pyy_cfcntrng: Decimal
+    source_recorded_at_raw: str
+    source_row_hash: str
+
+    @property
+    def semantic_key(self) -> MonthlySemanticKey:
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
+            self.source_effective_month.source_text(),
+        )
+
+    def canonical_data(self) -> dict[str, object]:
+        return {
+            "identity": identity_data(self.identity),
+            "pmm_avgprc": decimal_text(self.pmm_avgprc),
+            "pmm_cfcntvrtn": decimal_text(self.pmm_cfcntvrtn),
+            "pmm_cfcntrng": decimal_text(self.pmm_cfcntrng),
+            "pmm_hgprc": decimal_text(self.pmm_hgprc),
+            "pmm_lwprc": decimal_text(self.pmm_lwprc),
+            "pmm_stddvtn": decimal_text(self.pmm_stddvtn),
+            "pyy_avgprc": decimal_text(self.pyy_avgprc),
+            "pyy_cfcntvrtn": decimal_text(self.pyy_cfcntvrtn),
+            "pyy_cfcntrng": decimal_text(self.pyy_cfcntrng),
+            "pyy_hgprc": decimal_text(self.pyy_hgprc),
+            "pyy_lwprc": decimal_text(self.pyy_lwprc),
+            "pyy_stddvtn": decimal_text(self.pyy_stddvtn),
+            "region": {"code": self.region.code, "name": self.region.name},
+            "source_effective_month": self.source_effective_month.source_text(),
+            "source_recorded_at_raw": self.source_recorded_at_raw,
+            "source_row_hash": self.source_row_hash,
+        }
+
+
+def parse_monthly_price_rows(
+    items: object,
+    *,
+    registry: HistoricalDimensionRegistry,
+) -> ParsedHistoricalResult[ParsedMonthlyPriceRow]:
+    """Parse only the exact 28-field 15156060 contract, deriving no facts."""
+
+    source_items = require_items(items)
+    parsed_rows: list[ParsedMonthlyPriceRow] = []
+    seen: set[MonthlySemanticKey] = set()
+    for row_index, raw_row in enumerate(source_items):
+        validator = HistoricalRowValidator(
+            raw_row,
+            row_index=row_index,
+            expected_fields=KAMIS_MONTHLY_PRICE_FIELDS,
+            registry=registry,
+        )
+        parsed = _parse_row(validator)
+        if parsed.semantic_key in seen:
+            raise KamisParseError("duplicate_semantic_identity", row_index=row_index)
+        seen.add(parsed.semantic_key)
+        parsed_rows.append(parsed)
+
+    ordered_rows = tuple(sorted(parsed_rows, key=lambda row: row.semantic_key))
+    result_hash = canonical_hash(
+        {
+            "parser_contract": "kamis-15156060-v1",
+            "rows": [row.canonical_data() for row in ordered_rows],
+        }
+    )
+    return ParsedHistoricalResult(
+        rows=ordered_rows,
+        input_row_count=len(source_items),
+        result_hash=result_hash,
+    )
+
+
+def _parse_row(validator: HistoricalRowValidator) -> ParsedMonthlyPriceRow:
+    pmm_avgprc = validator.positive_decimal("pmm_avgprc")
+    pmm_hgprc = validator.positive_decimal("pmm_hgprc")
+    pmm_lwprc = validator.positive_decimal("pmm_lwprc")
+    pyy_avgprc = validator.positive_decimal("pyy_avgprc")
+    pyy_hgprc = validator.positive_decimal("pyy_hgprc")
+    pyy_lwprc = validator.positive_decimal("pyy_lwprc")
+    require_price_range(
+        pmm_lwprc,
+        pmm_avgprc,
+        pmm_hgprc,
+        row_index=validator.row_index,
+        field="pmm_avgprc",
+    )
+    require_price_range(
+        pyy_lwprc,
+        pyy_avgprc,
+        pyy_hgprc,
+        row_index=validator.row_index,
+        field="pyy_avgprc",
+    )
+    return ParsedMonthlyPriceRow(
+        identity=validator.identity(),
+        region=validator.region(),
+        source_effective_month=YearMonth.from_source(
+            validator.row["exmn_ym"], row_index=validator.row_index
+        ),
+        pmm_avgprc=pmm_avgprc,
+        pmm_hgprc=pmm_hgprc,
+        pmm_lwprc=pmm_lwprc,
+        pmm_stddvtn=validator.nonnegative_decimal("pmm_stddvtn"),
+        pmm_cfcntvrtn=validator.nonnegative_decimal("pmm_cfcntvrtn"),
+        pmm_cfcntrng=validator.nonnegative_decimal("pmm_cfcntrng"),
+        pyy_avgprc=pyy_avgprc,
+        pyy_hgprc=pyy_hgprc,
+        pyy_lwprc=pyy_lwprc,
+        pyy_stddvtn=validator.nonnegative_decimal("pyy_stddvtn"),
+        pyy_cfcntvrtn=validator.nonnegative_decimal("pyy_cfcntvrtn"),
+        pyy_cfcntrng=validator.nonnegative_decimal("pyy_cfcntrng"),
+        source_recorded_at_raw=validator.text("orgnl_reg_dt", maximum=64),
+        source_row_hash=canonical_hash(validator.row),
+    )
diff --git a/grocery/tests/historical_fixtures.py b/grocery/tests/historical_fixtures.py
new file mode 100644
index 0000000..9ea5df0
--- /dev/null
+++ b/grocery/tests/historical_fixtures.py
@@ -0,0 +1,49 @@
+"""Synthetic reviewed dimensions and rows shared by historical parser tests."""
+
+from grocery.source.historical_dimensions import HistoricalDimensionRegistry
+from grocery.source.registry import INITIAL_RETAIL_IDENTITY_REGISTRY
+
+
+def historical_registry() -> HistoricalDimensionRegistry:
+    return HistoricalDimensionRegistry(
+        identity_registry=INITIAL_RETAIL_IDENTITY_REGISTRY,
+        region_names={"11000": "서울", "26000": "부산"},
+        market_names={
+            ("11000", "110001"): "합성서울시장",
+            ("26000", "260001"): "합성부산시장",
+        },
+        dimension_evidence_revision="synthetic-reviewed-v1",
+    )
+
+
+def monthly_row() -> dict[str, str]:
+    return {
+        "exmn_ym": "202602",
+        "sgg_cd": "11000",
+        "sgg_nm": "서울",
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
+        "unit": "포기",
+        "unit_sz": "1",
+        "pmm_avgprc": "1000.50",
+        "pmm_hgprc": "1200",
+        "pmm_lwprc": "800",
+        "pmm_stddvtn": "100.25",
+        "pmm_cfcntvrtn": "10.02",
+        "pmm_cfcntrng": "400",
+        "pyy_avgprc": "900",
+        "pyy_hgprc": "1100",
+        "pyy_lwprc": "700",
+        "pyy_stddvtn": "90",
+        "pyy_cfcntvrtn": "10",
+        "pyy_cfcntrng": "400",
+        "orgnl_reg_dt": "2026-08-31 12:00:00",
+    }
diff --git a/grocery/tests/test_monthly_history.py b/grocery/tests/test_monthly_history.py
new file mode 100644
index 0000000..a5adea3
--- /dev/null
+++ b/grocery/tests/test_monthly_history.py
@@ -0,0 +1,122 @@
+"""Synthetic strict parser tests for public-data API 15156060."""
+
+from decimal import Decimal
+
+import pytest
+
+from grocery.source.kamis import KamisParseError
+from grocery.source.monthly_history import (
+    KAMIS_MONTHLY_PRICE_FIELDS,
+    parse_monthly_price_rows,
+)
+from grocery.tests.historical_fixtures import historical_registry, monthly_row
+
+
+def test_monthly_row_parses_all_source_facts_without_derivation() -> None:
+    result = parse_monthly_price_rows([monthly_row()], registry=historical_registry())
+
+    assert len(KAMIS_MONTHLY_PRICE_FIELDS) == 28
+    assert result.input_row_count == 1
+    row = result.rows[0]
+    assert row.source_effective_month.source_text() == "202602"
+    assert row.region.code == "11000"
+    assert row.pmm_avgprc == Decimal("1000.50")
+    assert row.pmm_stddvtn == Decimal("100.25")
+    assert row.pyy_lwprc == Decimal("700")
+    assert row.source_recorded_at_raw == "2026-08-31 12:00:00"
+    assert len(row.source_row_hash) == len(result.result_hash) == 64
+    assert not {
+        "trend",
+        "seasonality",
+        "market_type",
+        "computed_average",
+    } & row.canonical_data().keys()
+
+
+def test_monthly_result_is_deterministic_across_input_and_mapping_order() -> None:
+    first = monthly_row()
+    second = monthly_row()
+    second["exmn_ym"] = "202603"
+    reversed_second = dict(reversed(tuple(second.items())))
+
+    left = parse_monthly_price_rows([first, reversed_second], registry=historical_registry())
+    right = parse_monthly_price_rows([second, first], registry=historical_registry())
+
+    assert left == right
+    assert [row.source_effective_month.source_text() for row in left.rows] == [
+        "202602",
+        "202603",
+    ]
+
+
+@pytest.mark.parametrize("missing_field", sorted(KAMIS_MONTHLY_PRICE_FIELDS))
+def test_every_missing_monthly_field_fails_closed(missing_field: str) -> None:
+    row = monthly_row()
+    del row[missing_field]
+
+    with pytest.raises(KamisParseError, match=rf"missing_field .*field={missing_field}"):
+        parse_monthly_price_rows([row], registry=historical_registry())
+
+
+def test_unknown_or_non_string_monthly_fields_do_not_echo_values() -> None:
+    unknown = monthly_row()
+    unknown["synthetic-secret-field"] = "synthetic-secret-value"
+    with pytest.raises(KamisParseError, match="unknown_field") as raised:
+        parse_monthly_price_rows([unknown], registry=historical_registry())
+    assert "synthetic-secret" not in str(raised.value)
+
+    wrong_type: dict[str, object] = {
+        field: value for field, value in monthly_row().items()
+    }
+    wrong_type["pmm_avgprc"] = 1000
+    with pytest.raises(KamisParseError, match="field_type_drift"):
+        parse_monthly_price_rows([wrong_type], registry=historical_registry())
+
+
+@pytest.mark.parametrize(
+    ("field", "value", "error_code"),
+    [
+        ("se_cd", "02", "unsupported_product_class"),
+        ("ctgry_nm", "과일류", "category_name_drift"),
+        ("unit", "kg", "unit_identity_drift"),
+        ("sgg_nm", "다른지역", "region_code_name_drift"),
+        ("exmn_ym", "202613", "invalid_source_month"),
+        ("orgnl_reg_dt", "2026-08-31\n", "invalid_source_text"),
+        ("pmm_avgprc", "1e3", "invalid_decimal"),
+        ("pmm_avgprc", "0", "invalid_positive_decimal"),
+        ("pmm_stddvtn", "-1", "invalid_decimal"),
+    ],
+)
+def test_monthly_identity_date_text_and_decimal_drift_fails(
+    field: str,
+    value: str,
+    error_code: str,
+) -> None:
+    row = monthly_row()
+    row[field] = value
+
+    with pytest.raises(KamisParseError, match=error_code):
+        parse_monthly_price_rows([row], registry=historical_registry())
+
+
+@pytest.mark.parametrize(
+    ("field", "value"),
+    [("pmm_lwprc", "1100"), ("pmm_hgprc", "900"), ("pyy_avgprc", "1200")],
+)
+def test_monthly_source_range_inversion_fails(field: str, value: str) -> None:
+    row = monthly_row()
+    row[field] = value
+
+    with pytest.raises(KamisParseError, match="invalid_price_range"):
+        parse_monthly_price_rows([row], registry=historical_registry())
+
+
+def test_duplicate_monthly_semantic_identity_fails_even_if_values_change() -> None:
+    changed = monthly_row()
+    changed["pmm_avgprc"] = "1001"
+
+    with pytest.raises(KamisParseError, match="duplicate_semantic_identity"):
+        parse_monthly_price_rows(
+            [monthly_row(), changed],
+            registry=historical_registry(),
+        )


