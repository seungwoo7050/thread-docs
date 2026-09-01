# 결정적 최근 가격 정규화

## `feat(source): parse kamis recent rows`

diff --git a/grocery/source/__init__.py b/grocery/source/__init__.py
new file mode 100644
index 0000000..b9ba1ea
--- /dev/null
+++ b/grocery/source/__init__.py
@@ -0,0 +1,23 @@
+"""Source-specific, fail-closed parsers."""
+
+from grocery.source.kamis import (
+    KAMIS_RETAIL_COVERAGE_IDENTITY,
+    ExactIdentityRegistry,
+    IdentityContractEvidence,
+    KamisParseError,
+    ParsedRecentPriceResult,
+    ParsedRetailPriceRow,
+    build_identity_registry_from_reviewed_evidence,
+    parse_recent_price_rows,
+)
+
+__all__ = [
+    "KAMIS_RETAIL_COVERAGE_IDENTITY",
+    "ExactIdentityRegistry",
+    "IdentityContractEvidence",
+    "KamisParseError",
+    "ParsedRecentPriceResult",
+    "ParsedRetailPriceRow",
+    "build_identity_registry_from_reviewed_evidence",
+    "parse_recent_price_rows",
+]
diff --git a/grocery/source/kamis.py b/grocery/source/kamis.py
new file mode 100644
index 0000000..0627292
--- /dev/null
+++ b/grocery/source/kamis.py
@@ -0,0 +1,577 @@
+"""Deterministic parser for the KAMIS recent-price row contract.
+
+Transport, credentials, pagination, and persistence deliberately live outside this
+module. The parser accepts the JSON ``items.item`` list only and emits immutable,
+typed facts. It never includes source values in exception messages.
+"""
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
+from enum import StrEnum
+from types import MappingProxyType
+
+KAMIS_RETAIL_COVERAGE_IDENTITY = "KAMIS_RETAIL_ALL_REGIONS_22_CITIES_V1"
+KAMIS_RETAIL_PRODUCT_CLASS_CODE = "01"
+KAMIS_RETAIL_PRODUCT_CLASS_NAME = "소매"
+KAMIS_ALLOWED_CATEGORIES = MappingProxyType({"200": "채소류", "400": "과일류"})
+
+KAMIS_RECENT_PRICE_FIELDS = frozenset(
+    {
+        "ctgry_cd",
+        "ctgry_nm",
+        "dd1_bfr_cnvs_prc",
+        "dd1_bfr_prc",
+        "exmn_dd_cnvs_prc",
+        "exmn_dd_prc",
+        "exmn_ymd",
+        "grd_cd",
+        "grd_nm",
+        "item_cd",
+        "item_nm",
+        "mm1_bfr_cnvs_prc",
+        "mm1_bfr_prc",
+        "se_cd",
+        "se_nm",
+        "unit",
+        "unit_sz",
+        "vrty_cd",
+        "vrty_nm",
+        "ww1_bfr_cnvs_prc",
+        "ww1_bfr_prc",
+        "yy1_bfr_cnvs_prc",
+        "yy1_bfr_prc",
+    }
+)
+
+_IDENTITY_FIELDS = (
+    "se_cd",
+    "se_nm",
+    "ctgry_cd",
+    "ctgry_nm",
+    "item_cd",
+    "item_nm",
+    "vrty_cd",
+    "vrty_nm",
+    "grd_cd",
+    "grd_nm",
+    "unit",
+    "unit_sz",
+)
+_IGNORED_PRICE_FIELDS = (
+    "exmn_dd_cnvs_prc",
+    "dd1_bfr_prc",
+    "dd1_bfr_cnvs_prc",
+    "ww1_bfr_cnvs_prc",
+    "mm1_bfr_cnvs_prc",
+    "yy1_bfr_cnvs_prc",
+)
+_OPTIONAL_PRICE_FIELDS = frozenset(
+    {
+        "dd1_bfr_cnvs_prc",
+        "dd1_bfr_prc",
+        "mm1_bfr_cnvs_prc",
+        "mm1_bfr_prc",
+        "ww1_bfr_cnvs_prc",
+        "ww1_bfr_prc",
+        "yy1_bfr_cnvs_prc",
+        "yy1_bfr_prc",
+    }
+)
+_INTEGER_PRICE = re.compile(r"[1-9][0-9]*\Z")
+_SHA256 = re.compile(r"[0-9a-f]{64}\Z")
+
+type ItemCodeKey = tuple[str, str]
+type VarietyCodeKey = tuple[str, str, str]
+type GradeCodeKey = tuple[str, str, str, str]
+type SeriesCodeKey = tuple[str, str, str, str]
+type SemanticIdentityKey = tuple[str, str, str, str, str, str, str, str]
+
+
+class ReferencePeriod(StrEnum):
+    WEEK = "WEEK"
+    MONTH = "MONTH"
+    YEAR = "YEAR"
+
+
+class ValueStatus(StrEnum):
+    AVAILABLE = "AVAILABLE"
+    UNAVAILABLE = "UNAVAILABLE"
+
+
+class ReferenceDateStatus(StrEnum):
+    SOURCE_REFERENCE_DATE_UNAVAILABLE = "SOURCE_REFERENCE_DATE_UNAVAILABLE"
+
+
+class KamisParseError(ValueError):
+    """A redacted parser failure safe for operational logs."""
+
+    def __init__(self, code: str, *, row_index: int | None = None, field: str | None = None):
+        self.code = code
+        self.row_index = row_index
+        self.field = field
+        context = []
+        if row_index is not None:
+            context.append(f"row={row_index}")
+        if field is not None:
+            context.append(f"field={field}")
+        suffix = f" ({', '.join(context)})" if context else ""
+        super().__init__(f"{code}{suffix}")
+
+
+@dataclass(frozen=True, slots=True)
+class IdentityObservation:
+    product_class_code: str
+    product_class_name: str
+    category_code: str
+    category_name: str
+    item_code: str
+    item_name: str
+    variety_code: str
+    variety_name: str
+    grade_code: str
+    grade_name: str
+    raw_unit: str
+    raw_unit_size: str
+    coverage_identity: str
+
+    @property
+    def series_code_key(self) -> SeriesCodeKey:
+        return (
+            self.category_code,
+            self.item_code,
+            self.variety_code,
+            self.grade_code,
+        )
+
+
+@dataclass(frozen=True, slots=True)
+class IdentityContractEvidence:
+    """Hashes/revision proving a registry was reviewed outside the live parser."""
+
+    codebook_sha256: str
+    unit_contract_sha256: str
+    coverage_evidence_revision: str
+
+    def __post_init__(self) -> None:
+        if _SHA256.fullmatch(self.codebook_sha256) is None:
+            raise ValueError("codebook_sha256 must be a lowercase SHA-256")
+        if _SHA256.fullmatch(self.unit_contract_sha256) is None:
+            raise ValueError("unit_contract_sha256 must be a lowercase SHA-256")
+        if not self.coverage_evidence_revision.strip():
+            raise ValueError("coverage_evidence_revision is required")
+
+
+@dataclass(frozen=True, slots=True)
+class ExactIdentityRegistry:
+    """An immutable, exact code/name/unit contract supplied by reviewed evidence."""
+
+    item_names: Mapping[ItemCodeKey, str]
+    variety_names: Mapping[VarietyCodeKey, str]
+    grade_names: Mapping[GradeCodeKey, str]
+    units: Mapping[SeriesCodeKey, tuple[str, str]]
+    evidence: IdentityContractEvidence
+    coverage_identity: str = KAMIS_RETAIL_COVERAGE_IDENTITY
+
+    def __post_init__(self) -> None:
+        object.__setattr__(self, "item_names", MappingProxyType(dict(self.item_names)))
+        object.__setattr__(self, "variety_names", MappingProxyType(dict(self.variety_names)))
+        object.__setattr__(self, "grade_names", MappingProxyType(dict(self.grade_names)))
+        object.__setattr__(self, "units", MappingProxyType(dict(self.units)))
+        if not self.coverage_identity.strip():
+            raise ValueError("coverage_identity is required")
+        if not self.item_names or not self.variety_names or not self.grade_names or not self.units:
+            raise ValueError("reviewed identity registry cannot be empty")
+        for series_key in self.units:
+            if series_key not in self.grade_names:
+                raise ValueError("unit contract has no matching reviewed grade code")
+
+    def validate(self, observation: IdentityObservation, *, row_index: int) -> None:
+        if observation.coverage_identity != self.coverage_identity:
+            raise KamisParseError("coverage_identity_drift", row_index=row_index)
+        if observation.product_class_code != KAMIS_RETAIL_PRODUCT_CLASS_CODE:
+            raise KamisParseError("unsupported_product_class", row_index=row_index, field="se_cd")
+        if observation.product_class_name != KAMIS_RETAIL_PRODUCT_CLASS_NAME:
+            raise KamisParseError("product_class_name_drift", row_index=row_index, field="se_nm")
+
+        expected_category_name = KAMIS_ALLOWED_CATEGORIES.get(observation.category_code)
+        if expected_category_name is None:
+            raise KamisParseError("unsupported_category", row_index=row_index, field="ctgry_cd")
+        if observation.category_name != expected_category_name:
+            raise KamisParseError("category_name_drift", row_index=row_index, field="ctgry_nm")
+
+        item_key = (observation.category_code, observation.item_code)
+        if self.item_names.get(item_key) != observation.item_name:
+            raise KamisParseError("item_code_name_drift", row_index=row_index)
+
+        variety_key = (*item_key, observation.variety_code)
+        if self.variety_names.get(variety_key) != observation.variety_name:
+            raise KamisParseError("variety_code_name_drift", row_index=row_index)
+
+        grade_key = (*variety_key, observation.grade_code)
+        if self.grade_names.get(grade_key) != observation.grade_name:
+            raise KamisParseError("grade_code_name_drift", row_index=row_index)
+
+        if self.units.get(observation.series_code_key) != (
+            observation.raw_unit,
+            observation.raw_unit_size,
+        ):
+            raise KamisParseError("unit_identity_drift", row_index=row_index)
+
+
+def build_identity_registry_from_reviewed_evidence(
+    *,
+    item_names: Mapping[ItemCodeKey, str],
+    variety_names: Mapping[VarietyCodeKey, str],
+    grade_names: Mapping[GradeCodeKey, str],
+    units: Mapping[SeriesCodeKey, tuple[str, str]],
+    evidence: IdentityContractEvidence,
+    coverage_identity: str = KAMIS_RETAIL_COVERAGE_IDENTITY,
+) -> ExactIdentityRegistry:
+    """Seal an explicitly reviewed codebook/unit contract for repeated parse runs.
+
+    Callers must not populate these mappings directly from the live response being
+    parsed. The evidence hashes belong to the independently reviewed codebook and
+    unit-contract artifact; subsequent parses then fail on any observed drift.
+    """
+
+    return ExactIdentityRegistry(
+        item_names=item_names,
+        variety_names=variety_names,
+        grade_names=grade_names,
+        units=units,
+        evidence=evidence,
+        coverage_identity=coverage_identity,
+    )
+
+
+@dataclass(frozen=True, slots=True)
+class ParsedReferencePrice:
+    period: ReferencePeriod
+    value_status: ValueStatus
+    value: Decimal | None
+    unavailable_reason: str | None
+    reference_date_status: ReferenceDateStatus
+    source_reference_date: None = None
+
+    def canonical_data(self) -> dict[str, str | None]:
+        return {
+            "period": self.period.value,
+            "reference_date_status": self.reference_date_status.value,
+            "source_reference_date": None,
+            "unavailable_reason": self.unavailable_reason,
+            "value": _decimal_text(self.value) if self.value is not None else None,
+            "value_status": self.value_status.value,
+        }
+
+
+@dataclass(frozen=True, slots=True)
+class ParsedRetailPriceRow:
+    product_class_code: str
+    product_class_name: str
+    category_code: str
+    category_name: str
+    item_code: str
+    item_name: str
+    variety_code: str
+    variety_name: str
+    grade_code: str
+    grade_name: str
+    raw_unit: str
+    raw_unit_size: str
+    coverage_identity: str
+    source_effective_date: date
+    current_price: Decimal
+    references: tuple[ParsedReferencePrice, ...]
+    source_row_hash: str
+
+    @property
+    def semantic_identity_key(self) -> SemanticIdentityKey:
+        return (
+            self.product_class_code,
+            self.category_code,
+            self.item_code,
+            self.variety_code,
+            self.grade_code,
+            self.raw_unit,
+            self.raw_unit_size,
+            self.coverage_identity,
+        )
+
+    def canonical_data(self) -> dict[str, object]:
+        return {
+            "category_code": self.category_code,
+            "category_name": self.category_name,
+            "coverage_identity": self.coverage_identity,
+            "currency": "KRW",
+            "current_price": _decimal_text(self.current_price),
+            "grade_code": self.grade_code,
+            "grade_name": self.grade_name,
+            "item_code": self.item_code,
+            "item_name": self.item_name,
+            "product_class_code": self.product_class_code,
+            "product_class_name": self.product_class_name,
+            "raw_unit": self.raw_unit,
+            "raw_unit_size": self.raw_unit_size,
+            "references": [reference.canonical_data() for reference in self.references],
+            "source_effective_date": self.source_effective_date.isoformat(),
+            "source_row_hash": self.source_row_hash,
+            "variety_code": self.variety_code,
+            "variety_name": self.variety_name,
+        }
+
+
+@dataclass(frozen=True, slots=True)
+class ParsedRecentPriceResult:
+    rows: tuple[ParsedRetailPriceRow, ...]
+    input_row_count: int
+    out_of_scope_row_count: int
+    out_of_scope_row_hashes: tuple[str, ...]
+    result_hash: str
+
+    @property
+    def accepted_row_count(self) -> int:
+        return len(self.rows)
+
+
+def parse_recent_price_rows(
+    items: object,
+    *,
+    identity_registry: ExactIdentityRegistry,
+) -> ParsedRecentPriceResult:
+    """Parse a complete KAMIS ``items.item`` list and return a canonical result."""
+
+    if not isinstance(items, Sequence) or isinstance(items, (str, bytes, bytearray)):
+        raise KamisParseError("items_not_array")
+
+    parsed_rows: list[ParsedRetailPriceRow] = []
+    out_of_scope_row_hashes: list[str] = []
+    seen_identities: set[SemanticIdentityKey] = set()
+    for row_index, raw_row in enumerate(items):
+        if not isinstance(raw_row, Mapping):
+            raise KamisParseError("row_not_object", row_index=row_index)
+        row = _validate_row_shape(raw_row, row_index=row_index)
+        if not _is_target_scope(row):
+            _validate_contract_types(row, row_index=row_index)
+            out_of_scope_row_hashes.append(_canonical_hash(row))
+            continue
+        parsed = _parse_row(row, row_index=row_index, identity_registry=identity_registry)
+        if parsed.semantic_identity_key in seen_identities:
+            raise KamisParseError("duplicate_semantic_identity", row_index=row_index)
+        seen_identities.add(parsed.semantic_identity_key)
+        parsed_rows.append(parsed)
+
+    ordered_rows = tuple(sorted(parsed_rows, key=lambda row: row.semantic_identity_key))
+    ordered_out_of_scope_hashes = tuple(sorted(out_of_scope_row_hashes))
+    result_hash = _canonical_hash(
+        {
+            "accepted_rows": [row.canonical_data() for row in ordered_rows],
+            "out_of_scope_row_hashes": ordered_out_of_scope_hashes,
+            "parser_contract": "kamis-recent-price-v1",
+        }
+    )
+    return ParsedRecentPriceResult(
+        rows=ordered_rows,
+        input_row_count=len(items),
+        out_of_scope_row_count=len(ordered_out_of_scope_hashes),
+        out_of_scope_row_hashes=ordered_out_of_scope_hashes,
+        result_hash=result_hash,
+    )
+
+
+def _validate_row_shape(raw_row: Mapping[object, object], *, row_index: int) -> dict[str, object]:
+    if not all(isinstance(key, str) for key in raw_row):
+        raise KamisParseError("non_string_field_name", row_index=row_index)
+    actual_fields = frozenset(raw_row)
+    if actual_fields != KAMIS_RECENT_PRICE_FIELDS:
+        if missing_fields := KAMIS_RECENT_PRICE_FIELDS - actual_fields:
+            field = sorted(missing_fields)[0]
+            raise KamisParseError("missing_field", row_index=row_index, field=field)
+        raise KamisParseError("unknown_field", row_index=row_index)
+    return {str(key): value for key, value in raw_row.items()}
+
+
+def _validate_contract_types(row: Mapping[str, object], *, row_index: int) -> None:
+    required_string_fields = KAMIS_RECENT_PRICE_FIELDS - _OPTIONAL_PRICE_FIELDS
+    for field in required_string_fields:
+        value = row[field]
+        if not isinstance(value, str):
+            raise KamisParseError("field_type_drift", row_index=row_index, field=field)
+    for field in _OPTIONAL_PRICE_FIELDS:
+        value = row[field]
+        if value is not None and not isinstance(value, str):
+            raise KamisParseError("field_type_drift", row_index=row_index, field=field)
+
+    for field in _IDENTITY_FIELDS:
+        _required_text(row[field], row_index=row_index, field=field)
+    _parse_effective_date(row["exmn_ymd"], row_index=row_index)
+    _parse_required_price(row["exmn_dd_prc"], row_index=row_index, field="exmn_dd_prc")
+    for field in _IGNORED_PRICE_FIELDS:
+        _validate_ignored_price(row[field], row_index=row_index, field=field)
+    for field, period in (
+        ("ww1_bfr_prc", ReferencePeriod.WEEK),
+        ("mm1_bfr_prc", ReferencePeriod.MONTH),
+        ("yy1_bfr_prc", ReferencePeriod.YEAR),
+    ):
+        _parse_reference(period, row[field], row_index=row_index, field=field)
+
+
+def _is_target_scope(row: Mapping[str, object]) -> bool:
+    return (
+        row["se_cd"] == KAMIS_RETAIL_PRODUCT_CLASS_CODE
+        and row["ctgry_cd"] in KAMIS_ALLOWED_CATEGORIES
+    )
+
+
+def _parse_row(
+    row: Mapping[str, object],
+    *,
+    row_index: int,
+    identity_registry: ExactIdentityRegistry,
+) -> ParsedRetailPriceRow:
+    identity_values = {
+        field: _required_text(row[field], row_index=row_index, field=field)
+        for field in _IDENTITY_FIELDS
+    }
+    observation = IdentityObservation(
+        product_class_code=identity_values["se_cd"],
+        product_class_name=identity_values["se_nm"],
+        category_code=identity_values["ctgry_cd"],
+        category_name=identity_values["ctgry_nm"],
+        item_code=identity_values["item_cd"],
+        item_name=identity_values["item_nm"],
+        variety_code=identity_values["vrty_cd"],
+        variety_name=identity_values["vrty_nm"],
+        grade_code=identity_values["grd_cd"],
+        grade_name=identity_values["grd_nm"],
+        raw_unit=identity_values["unit"],
+        raw_unit_size=identity_values["unit_sz"],
+        coverage_identity=KAMIS_RETAIL_COVERAGE_IDENTITY,
+    )
+    identity_registry.validate(observation, row_index=row_index)
+
+    effective_date = _parse_effective_date(row["exmn_ymd"], row_index=row_index)
+    current_price = _parse_required_price(
+        row["exmn_dd_prc"], row_index=row_index, field="exmn_dd_prc"
+    )
+    references = (
+        _parse_reference(
+            ReferencePeriod.WEEK,
+            row["ww1_bfr_prc"],
+            row_index=row_index,
+            field="ww1_bfr_prc",
+        ),
+        _parse_reference(
+            ReferencePeriod.MONTH,
+            row["mm1_bfr_prc"],
+            row_index=row_index,
+            field="mm1_bfr_prc",
+        ),
+        _parse_reference(
+            ReferencePeriod.YEAR,
+            row["yy1_bfr_prc"],
+            row_index=row_index,
+            field="yy1_bfr_prc",
+        ),
+    )
+
+    # These fields are in the observed provider contract but outside the product
+    # decision. Type-check them so drift fails closed; do not publish or persist them.
+    for field in _IGNORED_PRICE_FIELDS:
+        _validate_ignored_price(row[field], row_index=row_index, field=field)
+
+    return ParsedRetailPriceRow(
+        product_class_code=observation.product_class_code,
+        product_class_name=observation.product_class_name,
+        category_code=observation.category_code,
+        category_name=observation.category_name,
+        item_code=observation.item_code,
+        item_name=observation.item_name,
+        variety_code=observation.variety_code,
+        variety_name=observation.variety_name,
+        grade_code=observation.grade_code,
+        grade_name=observation.grade_name,
+        raw_unit=observation.raw_unit,
+        raw_unit_size=observation.raw_unit_size,
+        coverage_identity=observation.coverage_identity,
+        source_effective_date=effective_date,
+        current_price=current_price,
+        references=references,
+        source_row_hash=_canonical_hash(row),
+    )
+
+
+def _required_text(value: object, *, row_index: int, field: str) -> str:
+    if not isinstance(value, str) or not value or value != value.strip():
+        raise KamisParseError("invalid_identity_text", row_index=row_index, field=field)
+    return value
+
+
+def _parse_effective_date(value: object, *, row_index: int) -> date:
+    if not isinstance(value, str) or not re.fullmatch(r"[0-9]{8}", value):
+        raise KamisParseError("invalid_effective_date", row_index=row_index, field="exmn_ymd")
+    try:
+        return datetime.strptime(value, "%Y%m%d").date()
+    except ValueError as error:
+        raise KamisParseError(
+            "invalid_effective_date", row_index=row_index, field="exmn_ymd"
+        ) from error
+
+
+def _parse_required_price(value: object, *, row_index: int, field: str) -> Decimal:
+    if not isinstance(value, str) or _INTEGER_PRICE.fullmatch(value) is None:
+        raise KamisParseError("invalid_positive_scale_zero_price", row_index=row_index, field=field)
+    parsed = Decimal(value)
+    if parsed <= 0 or parsed.as_tuple().exponent != 0:
+        raise KamisParseError("invalid_positive_scale_zero_price", row_index=row_index, field=field)
+    return parsed
+
+
+def _parse_reference(
+    period: ReferencePeriod,
+    value: object,
+    *,
+    row_index: int,
+    field: str,
+) -> ParsedReferencePrice:
+    if value is None:
+        return ParsedReferencePrice(
+            period=period,
+            value_status=ValueStatus.UNAVAILABLE,
+            value=None,
+            unavailable_reason="SOURCE_VALUE_MISSING",
+            reference_date_status=ReferenceDateStatus.SOURCE_REFERENCE_DATE_UNAVAILABLE,
+        )
+    return ParsedReferencePrice(
+        period=period,
+        value_status=ValueStatus.AVAILABLE,
+        value=_parse_required_price(value, row_index=row_index, field=field),
+        unavailable_reason=None,
+        reference_date_status=ReferenceDateStatus.SOURCE_REFERENCE_DATE_UNAVAILABLE,
+    )
+
+
+def _validate_ignored_price(value: object, *, row_index: int, field: str) -> None:
+    is_invalid = not isinstance(value, str) or _INTEGER_PRICE.fullmatch(value) is None
+    if value is not None and is_invalid:
+        raise KamisParseError("invalid_ignored_price", row_index=row_index, field=field)
+
+
+def _decimal_text(value: Decimal) -> str:
+    return format(value, "f")
+
+
+def _canonical_hash(value: object) -> str:
+    canonical = json.dumps(
+        value,
+        ensure_ascii=False,
+        sort_keys=True,
+        separators=(",", ":"),
+    ).encode("utf-8")
+    return hashlib.sha256(canonical).hexdigest()
diff --git a/grocery/tests/test_kamis_parser.py b/grocery/tests/test_kamis_parser.py
new file mode 100644
index 0000000..2c31ce8
--- /dev/null
+++ b/grocery/tests/test_kamis_parser.py
@@ -0,0 +1,319 @@
+"""Synthetic contract tests; these fixtures are not live source evidence."""
+
+from __future__ import annotations
+
+from copy import deepcopy
+from decimal import Decimal
+
+import pytest
+
+from grocery.source.kamis import (
+    KAMIS_RETAIL_COVERAGE_IDENTITY,
+    ExactIdentityRegistry,
+    IdentityContractEvidence,
+    KamisParseError,
+    ReferenceDateStatus,
+    ReferencePeriod,
+    ValueStatus,
+    parse_recent_price_rows,
+)
+
+SYNTHETIC_EVIDENCE = IdentityContractEvidence(
+    codebook_sha256="1" * 64,
+    unit_contract_sha256="2" * 64,
+    coverage_evidence_revision="synthetic-test-only-v1",
+)
+
+SYNTHETIC_ROW: dict[str, object] = {
+    "ctgry_cd": "200",
+    "ctgry_nm": "채소류",
+    "dd1_bfr_cnvs_prc": "2300",
+    "dd1_bfr_prc": "2300",
+    "exmn_dd_cnvs_prc": "2400",
+    "exmn_dd_prc": "2400",
+    "exmn_ymd": "20260829",
+    "grd_cd": "04",
+    "grd_nm": "상품",
+    "item_cd": "212",
+    "item_nm": "합성채소",
+    "mm1_bfr_cnvs_prc": "2200",
+    "mm1_bfr_prc": None,
+    "se_cd": "01",
+    "se_nm": "소매",
+    "unit": "포기",
+    "unit_sz": "1",
+    "vrty_cd": "00",
+    "vrty_nm": "합성품종",
+    "ww1_bfr_cnvs_prc": "2100",
+    "ww1_bfr_prc": "2100",
+    "yy1_bfr_cnvs_prc": "2000",
+    "yy1_bfr_prc": "2000",
+}
+
+
+@pytest.fixture
+def identity_registry() -> ExactIdentityRegistry:
+    return ExactIdentityRegistry(
+        item_names={("200", "212"): "합성채소"},
+        variety_names={("200", "212", "00"): "합성품종"},
+        grade_names={("200", "212", "00", "04"): "상품"},
+        units={("200", "212", "00", "04"): ("포기", "1")},
+        evidence=SYNTHETIC_EVIDENCE,
+    )
+
+
+def test_parses_typed_current_and_reference_values(
+    identity_registry: ExactIdentityRegistry,
+) -> None:
+    result = parse_recent_price_rows([SYNTHETIC_ROW], identity_registry=identity_registry)
+
+    assert len(result.result_hash) == 64
+    row = result.rows[0]
+    assert row.semantic_identity_key == (
+        "01",
+        "200",
+        "212",
+        "00",
+        "04",
+        "포기",
+        "1",
+        KAMIS_RETAIL_COVERAGE_IDENTITY,
+    )
+    assert row.source_effective_date.isoformat() == "2026-08-29"
+    assert row.current_price == Decimal("2400")
+    assert [reference.period for reference in row.references] == [
+        ReferencePeriod.WEEK,
+        ReferencePeriod.MONTH,
+        ReferencePeriod.YEAR,
+    ]
+    assert row.references[0].value == Decimal("2100")
+    assert row.references[1].value_status is ValueStatus.UNAVAILABLE
+    assert row.references[1].value is None
+    assert row.references[1].unavailable_reason == "SOURCE_VALUE_MISSING"
+    assert all(
+        reference.reference_date_status is ReferenceDateStatus.SOURCE_REFERENCE_DATE_UNAVAILABLE
+        for reference in row.references
+    )
+    assert all(reference.source_reference_date is None for reference in row.references)
+
+
+def test_hashes_are_deterministic_and_result_is_identity_sorted(
+    identity_registry: ExactIdentityRegistry,
+) -> None:
+    second_row = deepcopy(SYNTHETIC_ROW)
+    second_row.update(
+        {
+            "item_cd": "213",
+            "item_nm": "합성시금치",
+            "vrty_nm": "합성시금치품종",
+            "unit": "g",
+            "unit_sz": "100",
+        }
+    )
+    registry = ExactIdentityRegistry(
+        item_names={**identity_registry.item_names, ("200", "213"): "합성시금치"},
+        variety_names={
+            **identity_registry.variety_names,
+            ("200", "213", "00"): "합성시금치품종",
+        },
+        grade_names={
+            **identity_registry.grade_names,
+            ("200", "213", "00", "04"): "상품",
+        },
+        units={
+            **identity_registry.units,
+            ("200", "213", "00", "04"): ("g", "100"),
+        },
+        evidence=SYNTHETIC_EVIDENCE,
+    )
+
+    forward = parse_recent_price_rows([second_row, SYNTHETIC_ROW], identity_registry=registry)
+    reverse = parse_recent_price_rows([SYNTHETIC_ROW, second_row], identity_registry=registry)
+
+    assert forward == reverse
+    assert [row.item_code for row in forward.rows] == ["212", "213"]
+
+
+@pytest.mark.parametrize("field", sorted(SYNTHETIC_ROW))
+def test_missing_field_fails_closed(
+    identity_registry: ExactIdentityRegistry,
+    field: str,
+) -> None:
+    row = deepcopy(SYNTHETIC_ROW)
+    del row[field]
+
+    with pytest.raises(KamisParseError, match="missing_field") as raised:
+        parse_recent_price_rows([row], identity_registry=identity_registry)
+
+    assert raised.value.field == field
+
+
+def test_extra_field_fails_without_echoing_it(identity_registry: ExactIdentityRegistry) -> None:
+    row = deepcopy(SYNTHETIC_ROW)
+    row["provider_new_sensitive_value"] = "must-not-appear"
+
+    with pytest.raises(KamisParseError, match="unknown_field") as raised:
+        parse_recent_price_rows([row], identity_registry=identity_registry)
+
+    assert "provider_new_sensitive_value" not in str(raised.value)
+    assert "must-not-appear" not in str(raised.value)
+
+
+@pytest.mark.parametrize(
+    ("field", "value", "code"),
+    [
+        ("se_nm", "소매변경", "product_class_name_drift"),
+        ("ctgry_nm", "채소", "category_name_drift"),
+        ("item_nm", "변경된이름", "item_code_name_drift"),
+        ("vrty_nm", "변경된품종", "variety_code_name_drift"),
+        ("grd_nm", "변경된등급", "grade_code_name_drift"),
+        ("unit", "kg", "unit_identity_drift"),
+        ("unit_sz", "2", "unit_identity_drift"),
+    ],
+)
+def test_identity_or_unit_drift_fails_closed(
+    identity_registry: ExactIdentityRegistry,
+    field: str,
+    value: str,
+    code: str,
+) -> None:
+    row = deepcopy(SYNTHETIC_ROW)
+    row[field] = value
+
+    with pytest.raises(KamisParseError, match=code):
+        parse_recent_price_rows([row], identity_registry=identity_registry)
+
+
+def test_coverage_revision_drift_fails_closed() -> None:
+    registry = ExactIdentityRegistry(
+        item_names={("200", "212"): "합성채소"},
+        variety_names={("200", "212", "00"): "합성품종"},
+        grade_names={("200", "212", "00", "04"): "상품"},
+        units={("200", "212", "00", "04"): ("포기", "1")},
+        evidence=SYNTHETIC_EVIDENCE,
+        coverage_identity="UNREVIEWED_COVERAGE_REVISION",
+    )
+
+    with pytest.raises(KamisParseError, match="coverage_identity_drift"):
+        parse_recent_price_rows([SYNTHETIC_ROW], identity_registry=registry)
+
+
+@pytest.mark.parametrize("value", [0, "0", "-1", "1.0", " 1", "1 ", "", None])
+def test_invalid_current_price_fails_closed(
+    identity_registry: ExactIdentityRegistry,
+    value: object,
+) -> None:
+    row = deepcopy(SYNTHETIC_ROW)
+    row["exmn_dd_prc"] = value
+
+    with pytest.raises(KamisParseError, match="invalid_positive_scale_zero_price"):
+        parse_recent_price_rows([row], identity_registry=identity_registry)
+
+
+@pytest.mark.parametrize("value", [0, "0", "-", "1.5", "", [], {}])
+def test_only_json_null_marks_a_reference_unavailable(
+    identity_registry: ExactIdentityRegistry,
+    value: object,
+) -> None:
+    row = deepcopy(SYNTHETIC_ROW)
+    row["ww1_bfr_prc"] = value
+
+    with pytest.raises(KamisParseError, match="invalid_positive_scale_zero_price"):
+        parse_recent_price_rows([row], identity_registry=identity_registry)
+
+
+@pytest.mark.parametrize("value", ["20260230", "2026-08-29", 20260829, None])
+def test_invalid_effective_date_fails_closed(
+    identity_registry: ExactIdentityRegistry,
+    value: object,
+) -> None:
+    row = deepcopy(SYNTHETIC_ROW)
+    row["exmn_ymd"] = value
+
+    with pytest.raises(KamisParseError, match="invalid_effective_date"):
+        parse_recent_price_rows([row], identity_registry=identity_registry)
+
+
+def test_duplicate_semantic_identity_fails_closed(
+    identity_registry: ExactIdentityRegistry,
+) -> None:
+    later = deepcopy(SYNTHETIC_ROW)
+    later["exmn_ymd"] = "20260830"
+
+    with pytest.raises(KamisParseError, match="duplicate_semantic_identity"):
+        parse_recent_price_rows([SYNTHETIC_ROW, later], identity_registry=identity_registry)
+
+
+@pytest.mark.parametrize("items", [None, {}, "row", b"row", 1])
+def test_items_must_be_an_array(identity_registry: ExactIdentityRegistry, items: object) -> None:
+    with pytest.raises(KamisParseError, match="items_not_array"):
+        parse_recent_price_rows(items, identity_registry=identity_registry)
+
+
+def test_empty_items_have_a_stable_result_hash(identity_registry: ExactIdentityRegistry) -> None:
+    first = parse_recent_price_rows([], identity_registry=identity_registry)
+    second = parse_recent_price_rows([], identity_registry=identity_registry)
+
+    assert first == second
+    assert first.rows == ()
+    assert first.input_row_count == 0
+    assert first.accepted_row_count == 0
+    assert first.out_of_scope_row_count == 0
+
+
+def test_out_of_scope_rows_are_reconciled_without_becoming_facts(
+    identity_registry: ExactIdentityRegistry,
+) -> None:
+    wholesale_row = deepcopy(SYNTHETIC_ROW)
+    wholesale_row["se_cd"] = "02"
+    wholesale_row["se_nm"] = "합성중도매"
+    other_category_row = deepcopy(SYNTHETIC_ROW)
+    other_category_row["ctgry_cd"] = "100"
+    other_category_row["ctgry_nm"] = "합성식량작물"
+
+    result = parse_recent_price_rows(
+        [other_category_row, SYNTHETIC_ROW, wholesale_row],
+        identity_registry=identity_registry,
+    )
+
+    assert result.input_row_count == 3
+    assert result.accepted_row_count == 1
+    assert result.out_of_scope_row_count == 2
+    assert len(result.out_of_scope_row_hashes) == 2
+    assert result.input_row_count == result.accepted_row_count + result.out_of_scope_row_count
+
+
+def test_out_of_scope_order_does_not_change_result_hash(
+    identity_registry: ExactIdentityRegistry,
+) -> None:
+    first = deepcopy(SYNTHETIC_ROW)
+    first["se_cd"] = "02"
+    second = deepcopy(SYNTHETIC_ROW)
+    second["ctgry_cd"] = "600"
+
+    forward = parse_recent_price_rows([first, second], identity_registry=identity_registry)
+    reverse = parse_recent_price_rows([second, first], identity_registry=identity_registry)
+
+    assert forward == reverse
+
+
+def test_out_of_scope_schema_type_drift_still_fails_closed(
+    identity_registry: ExactIdentityRegistry,
+) -> None:
+    row = deepcopy(SYNTHETIC_ROW)
+    row["se_cd"] = "02"
+    row["exmn_ymd"] = 20260829
+
+    with pytest.raises(KamisParseError, match="field_type_drift"):
+        parse_recent_price_rows([row], identity_registry=identity_registry)
+
+
+def test_out_of_scope_price_sentinel_still_fails_closed(
+    identity_registry: ExactIdentityRegistry,
+) -> None:
+    row = deepcopy(SYNTHETIC_ROW)
+    row["se_cd"] = "02"
+    row["ww1_bfr_prc"] = "-"
+
+    with pytest.raises(KamisParseError, match="invalid_positive_scale_zero_price"):
+        parse_recent_price_rows([row], identity_registry=identity_registry)


