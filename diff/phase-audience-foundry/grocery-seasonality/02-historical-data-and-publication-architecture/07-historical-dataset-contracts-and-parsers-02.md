## `feat(source): seal historical dimensions`

diff --git a/grocery/source/historical_dimensions.py b/grocery/source/historical_dimensions.py
new file mode 100644
index 0000000..7c8fc53
--- /dev/null
+++ b/grocery/source/historical_dimensions.py
@@ -0,0 +1,97 @@
+"""Reviewed dimension contracts and source-month type for historical rows."""
+
+from __future__ import annotations
+
+import re
+import unicodedata
+from collections.abc import Mapping
+from dataclasses import dataclass
+from datetime import datetime
+from types import MappingProxyType
+
+from grocery.source.kamis import ExactIdentityRegistry, KamisParseError
+
+_CODE = re.compile(r"[0-9]{1,20}\Z")
+
+type MarketCodeKey = tuple[str, str]
+
+
+@dataclass(frozen=True, slots=True, order=True)
+class YearMonth:
+    """A source month that does not invent a first-day date."""
+
+    year: int
+    month: int
+
+    @classmethod
+    def from_source(cls, value: object, *, row_index: int) -> YearMonth:
+        if not isinstance(value, str) or re.fullmatch(r"[0-9]{6}", value) is None:
+            raise KamisParseError("invalid_source_month", row_index=row_index, field="exmn_ym")
+        try:
+            parsed = datetime.strptime(value, "%Y%m")
+        except ValueError:
+            raise KamisParseError(
+                "invalid_source_month", row_index=row_index, field="exmn_ym"
+            ) from None
+        return cls(year=parsed.year, month=parsed.month)
+
+    def source_text(self) -> str:
+        return f"{self.year:04d}{self.month:02d}"
+
+
+@dataclass(frozen=True, slots=True)
+class RegionObservation:
+    code: str
+    name: str
+
+
+@dataclass(frozen=True, slots=True)
+class MarketObservation:
+    code: str
+    name: str
+
+
+@dataclass(frozen=True, slots=True)
+class HistoricalDimensionRegistry:
+    """Reviewed identity, region, and market code/name contracts."""
+
+    identity_registry: ExactIdentityRegistry
+    region_names: Mapping[str, str]
+    market_names: Mapping[MarketCodeKey, str]
+    dimension_evidence_revision: str
+
+    def __post_init__(self) -> None:
+        if not isinstance(self.identity_registry, ExactIdentityRegistry):
+            raise TypeError("identity_registry must be an ExactIdentityRegistry")
+        regions = dict(self.region_names)
+        markets = dict(self.market_names)
+        if not regions:
+            raise ValueError("reviewed region registry cannot be empty")
+        if not _is_registry_text(self.dimension_evidence_revision, maximum=200):
+            raise ValueError("dimension_evidence_revision is invalid")
+        for code, name in regions.items():
+            if _CODE.fullmatch(code) is None or not _is_registry_text(name, maximum=100):
+                raise ValueError("region registry contains an invalid code or name")
+        for (region_code, market_code), name in markets.items():
+            if (
+                region_code not in regions
+                or _CODE.fullmatch(market_code) is None
+                or not _is_registry_text(name, maximum=100)
+            ):
+                raise ValueError("market registry contains an invalid code or name")
+        object.__setattr__(self, "region_names", MappingProxyType(regions))
+        object.__setattr__(self, "market_names", MappingProxyType(markets))
+
+
+def is_bounded_source_text(value: object, *, maximum: int) -> bool:
+    return (
+        isinstance(value, str)
+        and bool(value)
+        and value == value.strip()
+        and len(value) <= maximum
+        and all(not unicodedata.category(character).startswith("C") for character in value)
+    )
+
+
+def _is_registry_text(value: object, *, maximum: int) -> bool:
+    return is_bounded_source_text(value, maximum=maximum)
diff --git a/grocery/tests/test_historical_parser_common.py b/grocery/tests/test_historical_parser_common.py
new file mode 100644
index 0000000..a21149e
--- /dev/null
+++ b/grocery/tests/test_historical_parser_common.py
@@ -0,0 +1,60 @@
+"""Focused tests for reviewed historical dimensions and source month typing."""
+
+import pytest
+
+from grocery.source.historical_dimensions import HistoricalDimensionRegistry, YearMonth
+from grocery.source.kamis import KamisParseError
+from grocery.source.registry import INITIAL_RETAIL_IDENTITY_REGISTRY
+
+
+def test_dimension_registry_is_immutable_and_exact() -> None:
+    regions = {"11000": "서울"}
+    markets = {("11000", "110001"): "합성시장"}
+    registry = HistoricalDimensionRegistry(
+        identity_registry=INITIAL_RETAIL_IDENTITY_REGISTRY,
+        region_names=regions,
+        market_names=markets,
+        dimension_evidence_revision="synthetic-reviewed-v1",
+    )
+    regions["26000"] = "부산"
+    markets[("11000", "110002")] = "다른시장"
+
+    assert registry.region_names == {"11000": "서울"}
+    assert registry.market_names == {("11000", "110001"): "합성시장"}
+    with pytest.raises(TypeError):
+        registry.region_names["26000"] = "부산"  # type: ignore[index]
+
+
+@pytest.mark.parametrize(
+    ("regions", "markets"),
+    [
+        ({}, {}),
+        ({"unsafe": "서울"}, {}),
+        ({"11000": "서울\n"}, {}),
+        ({"11000": "서울"}, {("99999", "1"): "시장"}),
+    ],
+)
+def test_invalid_reviewed_dimension_contract_is_rejected(
+    regions: dict[str, str],
+    markets: dict[tuple[str, str], str],
+) -> None:
+    with pytest.raises(ValueError):
+        HistoricalDimensionRegistry(
+            identity_registry=INITIAL_RETAIL_IDENTITY_REGISTRY,
+            region_names=regions,
+            market_names=markets,
+            dimension_evidence_revision="synthetic-reviewed-v1",
+        )
+
+
+def test_year_month_is_typed_without_inventing_a_day() -> None:
+    value = YearMonth.from_source("202602", row_index=0)
+
+    assert (value.year, value.month, value.source_text()) == (2026, 2, "202602")
+    assert not hasattr(value, "day")
+
+
+@pytest.mark.parametrize("value", ["202613", "2026-02", 202602])
+def test_invalid_source_month_is_redacted(value: object) -> None:
+    with pytest.raises(KamisParseError, match=r"invalid_source_month \(row=3, field=exmn_ym\)"):
+        YearMonth.from_source(value, row_index=3)


## `feat(source): validate historical row primitives`

diff --git a/grocery/source/historical_parser.py b/grocery/source/historical_parser.py
new file mode 100644
index 0000000..17fb3c7
--- /dev/null
+++ b/grocery/source/historical_parser.py
@@ -0,0 +1,198 @@
+"""Shared fail-closed validators for approved KAMIS historical row parsers."""
+
+from __future__ import annotations
+
+import hashlib
+import json
+import re
+from collections.abc import Mapping, Sequence
+from dataclasses import dataclass
+from datetime import date, datetime
+from decimal import Decimal
+
+from grocery.source.historical_dimensions import (
+    HistoricalDimensionRegistry,
+    MarketObservation,
+    RegionObservation,
+    is_bounded_source_text,
+)
+from grocery.source.kamis import IdentityObservation, KamisParseError
+
+_CODE = re.compile(r"[0-9]{1,20}\Z")
+_DECIMAL = re.compile(r"[0-9]+(?:\.[0-9]+)?\Z")
+
+
+@dataclass(frozen=True, slots=True)
+class ParsedHistoricalResult[RowT]:
+    rows: tuple[RowT, ...]
+    input_row_count: int
+    result_hash: str
+
+
+class HistoricalRowValidator:
+    """One exact source row plus redacted, index-only validation context."""
+
+    __slots__ = ("_registry", "row", "row_index")
+
+    def __init__(
+        self,
+        raw_row: object,
+        *,
+        row_index: int,
+        expected_fields: frozenset[str],
+        registry: HistoricalDimensionRegistry,
+    ) -> None:
+        if not isinstance(raw_row, Mapping):
+            raise KamisParseError("row_not_object", row_index=row_index)
+        if not all(isinstance(key, str) for key in raw_row):
+            raise KamisParseError("non_string_field_name", row_index=row_index)
+        actual_fields = frozenset(raw_row)
+        if actual_fields != expected_fields:
+            if missing := expected_fields - actual_fields:
+                raise KamisParseError(
+                    "missing_field", row_index=row_index, field=sorted(missing)[0]
+                )
+            raise KamisParseError("unknown_field", row_index=row_index)
+        self.row = {str(key): value for key, value in raw_row.items()}
+        self.row_index = row_index
+        self._registry = registry
+        for field, value in self.row.items():
+            if not isinstance(value, str):
+                raise KamisParseError("field_type_drift", row_index=row_index, field=field)
+
+    def identity(self) -> IdentityObservation:
+        for field in ("se_cd", "ctgry_cd", "item_cd", "vrty_cd", "grd_cd"):
+            self.code(field)
+        observation = IdentityObservation(
+            product_class_code=self.text("se_cd", maximum=20),
+            product_class_name=self.name("se_nm"),
+            category_code=self.text("ctgry_cd", maximum=20),
+            category_name=self.name("ctgry_nm"),
+            item_code=self.text("item_cd", maximum=20),
+            item_name=self.name("item_nm"),
+            variety_code=self.text("vrty_cd", maximum=20),
+            variety_name=self.name("vrty_nm"),
+            grade_code=self.text("grd_cd", maximum=20),
+            grade_name=self.name("grd_nm"),
+            raw_unit=self.text("unit", maximum=30),
+            raw_unit_size=self.text("unit_sz", maximum=30),
+            coverage_identity=self._registry.identity_registry.coverage_identity,
+        )
+        self.positive_decimal("unit_sz")
+        self._registry.identity_registry.validate(observation, row_index=self.row_index)
+        return observation
+
+    def region(self) -> RegionObservation:
+        code = self.code("sgg_cd")
+        name = self.name("sgg_nm")
+        if self._registry.region_names.get(code) != name:
+            raise KamisParseError("region_code_name_drift", row_index=self.row_index)
+        return RegionObservation(code=code, name=name)
+
+    def market(self, region: RegionObservation) -> MarketObservation:
+        code = self.code("mrkt_cd")
+        name = self.name("mrkt_nm")
+        if self._registry.market_names.get((region.code, code)) != name:
+            raise KamisParseError("market_code_name_drift", row_index=self.row_index)
+        return MarketObservation(code=code, name=name)
+
+    def text(self, field: str, *, maximum: int = 100) -> str:
+        value = self.row[field]
+        if not isinstance(value, str) or not is_bounded_source_text(value, maximum=maximum):
+            raise KamisParseError("invalid_source_text", row_index=self.row_index, field=field)
+        return value
+
+    def name(self, field: str) -> str:
+        value = self.text(field, maximum=100)
+        if _CODE.fullmatch(value) is not None:
+            raise KamisParseError("invalid_source_name", row_index=self.row_index, field=field)
+        return value
+
+    def code(self, field: str) -> str:
+        value = self.row[field]
+        if not isinstance(value, str) or _CODE.fullmatch(value) is None:
+            raise KamisParseError("invalid_source_code", row_index=self.row_index, field=field)
+        return value
+
+    def day(self, field: str = "exmn_ymd") -> date:
+        value = self.row[field]
+        if not isinstance(value, str) or re.fullmatch(r"[0-9]{8}", value) is None:
+            raise KamisParseError("invalid_source_date", row_index=self.row_index, field=field)
+        try:
+            return datetime.strptime(value, "%Y%m%d").date()
+        except ValueError:
+            raise KamisParseError(
+                "invalid_source_date", row_index=self.row_index, field=field
+            ) from None
+
+    def positive_decimal(self, field: str) -> Decimal:
+        value = self._decimal(field)
+        if value <= 0:
+            raise KamisParseError(
+                "invalid_positive_decimal", row_index=self.row_index, field=field
+            )
+        return value
+
+    def nonnegative_decimal(self, field: str) -> Decimal:
+        value = self._decimal(field)
+        if value < 0:
+            raise KamisParseError(
+                "invalid_nonnegative_decimal", row_index=self.row_index, field=field
+            )
+        return value
+
+    def _decimal(self, field: str) -> Decimal:
+        value = self.row[field]
+        if not isinstance(value, str) or _DECIMAL.fullmatch(value) is None:
+            raise KamisParseError("invalid_decimal", row_index=self.row_index, field=field)
+        return Decimal(value)
+
+
+def require_items(items: object) -> Sequence[object]:
+    if not isinstance(items, Sequence) or isinstance(items, (str, bytes, bytearray)):
+        raise KamisParseError("items_not_array")
+    return items
+
+
+def require_price_range(
+    low: Decimal,
+    average: Decimal,
+    high: Decimal,
+    *,
+    row_index: int,
+    field: str,
+) -> None:
+    if not low <= average <= high:
+        raise KamisParseError("invalid_price_range", row_index=row_index, field=field)
+
+
+def decimal_text(value: Decimal) -> str:
+    return format(value, "f")
+
+
+def identity_data(value: IdentityObservation) -> dict[str, str]:
+    return {
+        "category_code": value.category_code,
+        "category_name": value.category_name,
+        "coverage_identity": value.coverage_identity,
+        "grade_code": value.grade_code,
+        "grade_name": value.grade_name,
+        "item_code": value.item_code,
+        "item_name": value.item_name,
+        "product_class_code": value.product_class_code,
+        "product_class_name": value.product_class_name,
+        "raw_unit": value.raw_unit,
+        "raw_unit_size": value.raw_unit_size,
+        "variety_code": value.variety_code,
+        "variety_name": value.variety_name,
+    }
+
+
+def canonical_hash(value: object) -> str:
+    canonical = json.dumps(
+        value,
+        ensure_ascii=False,
+        sort_keys=True,
+        separators=(",", ":"),
+    ).encode("utf-8")
+    return hashlib.sha256(canonical).hexdigest()
diff --git a/grocery/tests/test_historical_row_validator.py b/grocery/tests/test_historical_row_validator.py
new file mode 100644
index 0000000..43a8331
--- /dev/null
+++ b/grocery/tests/test_historical_row_validator.py
@@ -0,0 +1,102 @@
+"""Focused contract tests for the shared historical row validator."""
+
+from decimal import Decimal
+
+import pytest
+
+from grocery.source.historical_dimensions import HistoricalDimensionRegistry
+from grocery.source.historical_parser import HistoricalRowValidator
+from grocery.source.kamis import KamisParseError
+from grocery.source.registry import INITIAL_RETAIL_IDENTITY_REGISTRY
+
+
+def _registry() -> HistoricalDimensionRegistry:
+    return HistoricalDimensionRegistry(
+        identity_registry=INITIAL_RETAIL_IDENTITY_REGISTRY,
+        region_names={"11000": "서울"},
+        market_names={("11000", "110001"): "합성시장"},
+        dimension_evidence_revision="synthetic-reviewed-v1",
+    )
+
+
+def _row() -> dict[str, str]:
+    return {
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
+        "sgg_cd": "11000",
+        "sgg_nm": "서울",
+        "exmn_ymd": "20260831",
+        "value": "1000.50",
+    }
+
+
+def _validator(row: object) -> HistoricalRowValidator:
+    return HistoricalRowValidator(
+        row,
+        row_index=2,
+        expected_fields=frozenset(_row()),
+        registry=_registry(),
+    )
+
+
+def test_exact_row_produces_reviewed_typed_values() -> None:
+    validator = _validator(_row())
+
+    assert validator.identity().item_name == "양배추"
+    assert validator.region().name == "서울"
+    assert validator.day().isoformat() == "2026-08-31"
+    assert validator.positive_decimal("value") == Decimal("1000.50")
+
+
+@pytest.mark.parametrize("mutation", ["missing", "unknown", "non_string"])
+def test_shape_and_string_type_drift_fail_without_values(mutation: str) -> None:
+    row: dict[str, object] = {field: value for field, value in _row().items()}
+    if mutation == "missing":
+        del row["value"]
+    elif mutation == "unknown":
+        row["secret-field"] = "secret-value"
+    else:
+        row["value"] = 1000
+
+    with pytest.raises(KamisParseError) as raised:
+        _validator(row)
+
+    assert "secret" not in str(raised.value)
+
+
+@pytest.mark.parametrize(
+    ("field", "value", "method", "error_code"),
+    [
+        ("sgg_nm", "서울\n", "region", "invalid_source_text"),
+        ("exmn_ymd", "20260230", "day", "invalid_source_date"),
+        ("value", "1e3", "decimal", "invalid_decimal"),
+        ("value", "0", "decimal", "invalid_positive_decimal"),
+    ],
+)
+def test_text_date_and_decimal_grammar_is_strict(
+    field: str,
+    value: str,
+    method: str,
+    error_code: str,
+) -> None:
+    row = _row()
+    row[field] = value
+    validator = _validator(row)
+
+    with pytest.raises(KamisParseError, match=error_code):
+        if method == "region":
+            validator.region()
+        elif method == "day":
+            validator.day()
+        else:
+            validator.positive_decimal("value")


