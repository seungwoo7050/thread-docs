# 최근 게시본 수명 주기

## `feat(publication): hash canonical retail facts`

diff --git a/grocery/publication_facts.py b/grocery/publication_facts.py
new file mode 100644
index 0000000..d145421
--- /dev/null
+++ b/grocery/publication_facts.py
@@ -0,0 +1,172 @@
+"""Canonical hashes for the fixed recent-retail publication fact set."""
+
+from __future__ import annotations
+
+import hashlib
+import json
+import uuid
+from collections.abc import Sequence
+from dataclasses import dataclass
+from datetime import date
+from decimal import Decimal
+
+from django.core.exceptions import ValidationError
+
+from grocery.models import PriceChangeFact, ReferencePrice, RetailPriceSnapshot
+
+ENTRY_HASH_VERSION = "recent-retail-entry-v1"
+FACT_SET_HASH_VERSION = "recent-retail-fact-set-v1"
+_PERIOD_ORDER = ("WEEK", "MONTH", "YEAR")
+
+
+@dataclass(frozen=True, slots=True)
+class CanonicalPublicationEntry:
+    ordinal: int
+    snapshot_id: uuid.UUID
+    fact_sha256: str
+
+
+@dataclass(frozen=True, slots=True)
+class CanonicalPublicationFactSet:
+    entries: tuple[CanonicalPublicationEntry, ...]
+    typed_fact_set_sha256: str
+    source_effective_date_min: date
+    source_effective_date_max: date
+
+
+def canonical_snapshot_data(snapshot: RetailPriceSnapshot) -> dict[str, object]:
+    """Build the complete, source-bounded public fact for one immutable snapshot."""
+
+    series = snapshot.series
+    references = list(snapshot.reference_prices.select_related("change_fact").order_by("period"))
+    by_period = {reference.period: reference for reference in references}
+    if len(references) != len(_PERIOD_ORDER) or set(by_period) != set(_PERIOD_ORDER):
+        raise ValidationError("Publication requires exactly WEEK, MONTH, and YEAR references.")
+
+    return {
+        "entry_hash_version": ENTRY_HASH_VERSION,
+        "series": {
+            "product_class_code": series.product_class_code,
+            "product_class_name": series.product_class_name,
+            "category_code": series.category_code,
+            "category_name": series.category_name,
+            "item_code": series.item_code,
+            "item_name": series.item_name,
+            "variety_code": series.variety_code,
+            "variety_name": series.variety_name,
+            "grade_code": series.grade_code,
+            "grade_name": series.grade_name,
+            "raw_unit": series.raw_unit,
+            "raw_unit_size": series.raw_unit_size,
+            "coverage_identity": series.coverage_identity,
+            "identity_evidence_revision": series.identity_evidence_revision,
+        },
+        "snapshot": {
+            "source_effective_date": snapshot.source_effective_date.isoformat(),
+            "source_recorded_at": (
+                snapshot.source_recorded_at.isoformat()
+                if snapshot.source_recorded_at is not None
+                else None
+            ),
+            "current_price": _decimal_text(snapshot.current_price),
+            "currency": snapshot.currency,
+            "source_row_sha256": snapshot.source_row_sha256,
+            "source_contract_revision": snapshot.source_contract_revision,
+        },
+        "references": [_canonical_reference_data(by_period[period]) for period in _PERIOD_ORDER],
+    }
+
+
+def snapshot_fact_sha256(snapshot: RetailPriceSnapshot) -> str:
+    canonical = _canonical_json(canonical_snapshot_data(snapshot))
+    return hashlib.sha256(f"{ENTRY_HASH_VERSION}\n".encode("ascii") + canonical).hexdigest()
+
+
+def build_publication_fact_set(
+    snapshots: Sequence[RetailPriceSnapshot],
+) -> CanonicalPublicationFactSet:
+    if not snapshots:
+        raise ValidationError("A publication fact set cannot be empty.")
+
+    parse_run_ids = {snapshot.parse_run_id for snapshot in snapshots}
+    snapshot_ids = {snapshot.id for snapshot in snapshots}
+    if len(parse_run_ids) != 1:
+        raise ValidationError("A publication fact set must use one parse generation.")
+    if len(snapshot_ids) != len(snapshots):
+        raise ValidationError("A publication fact set cannot repeat a snapshot.")
+
+    ordered = sorted(snapshots, key=_snapshot_order_key)
+    entries = tuple(
+        CanonicalPublicationEntry(
+            ordinal=index,
+            snapshot_id=snapshot.id,
+            fact_sha256=snapshot_fact_sha256(snapshot),
+        )
+        for index, snapshot in enumerate(ordered, start=1)
+    )
+    hash_lines = "\n".join(entry.fact_sha256 for entry in entries)
+    set_hash = hashlib.sha256(f"{FACT_SET_HASH_VERSION}\n{hash_lines}".encode("ascii")).hexdigest()
+    source_dates = [snapshot.source_effective_date for snapshot in ordered]
+    return CanonicalPublicationFactSet(
+        entries=entries,
+        typed_fact_set_sha256=set_hash,
+        source_effective_date_min=min(source_dates),
+        source_effective_date_max=max(source_dates),
+    )
+
+
+def _canonical_reference_data(reference: ReferencePrice) -> dict[str, object]:
+    try:
+        change = reference.change_fact
+    except PriceChangeFact.DoesNotExist as error:
+        raise ValidationError("Every publication reference requires a change fact.") from error
+    return {
+        "period": reference.period,
+        "value_status": reference.value_status,
+        "value": _nullable_decimal_text(reference.value),
+        "unavailable_reason": reference.unavailable_reason,
+        "reference_date_status": reference.reference_date_status,
+        "source_reference_date": (
+            reference.source_reference_date.isoformat()
+            if reference.source_reference_date is not None
+            else None
+        ),
+        "change": {
+            "direction": change.direction,
+            "signed_difference": _nullable_decimal_text(change.signed_difference),
+            "signed_percentage": _nullable_decimal_text(change.signed_percentage),
+            "calculation_revision": change.calculation_revision,
+            "rounding_mode": change.rounding_mode,
+        },
+    }
+
+
+def _snapshot_order_key(snapshot: RetailPriceSnapshot) -> tuple[str, ...]:
+    series = snapshot.series
+    return (
+        series.product_class_code,
+        series.category_code,
+        series.item_code,
+        series.variety_code,
+        series.grade_code,
+        series.raw_unit,
+        series.raw_unit_size,
+        series.coverage_identity,
+    )
+
+
+def _decimal_text(value: Decimal) -> str:
+    return format(value, "f")
+
+
+def _nullable_decimal_text(value: Decimal | None) -> str | None:
+    return _decimal_text(value) if value is not None else None
+
+
+def _canonical_json(value: object) -> bytes:
+    return json.dumps(
+        value,
+        ensure_ascii=False,
+        sort_keys=True,
+        separators=(",", ":"),
+    ).encode("utf-8")
diff --git a/grocery/tests/test_publication_facts.py b/grocery/tests/test_publication_facts.py
new file mode 100644
index 0000000..76cfc47
--- /dev/null
+++ b/grocery/tests/test_publication_facts.py
@@ -0,0 +1,143 @@
+import uuid
+from datetime import date, timedelta
+from decimal import Decimal
+from typing import Any, cast
+
+import pytest
+from django.core.exceptions import ValidationError
+from django.utils import timezone
+
+from grocery.models import ParseRun, RetailPriceSnapshot, persist_reference_price_facts
+from grocery.publication_facts import (
+    ENTRY_HASH_VERSION,
+    FACT_SET_HASH_VERSION,
+    build_publication_fact_set,
+    canonical_snapshot_data,
+    snapshot_fact_sha256,
+)
+from grocery.tests.test_artifact_parse_models import create_artifact
+from grocery.tests.test_price_series_key_models import create_series
+from grocery.tests.test_retail_price_snapshot_models import (
+    create_validated_parse_run,
+    replay_snapshot,
+)
+
+pytestmark = pytest.mark.django_db
+
+
+def create_distinct_validated_parse_run() -> ParseRun:
+    completed_at = timezone.now()
+    return ParseRun.objects.create(
+        artifact=create_artifact(),
+        parser_revision="kamis-recent-v1",
+        configuration_hash=uuid.uuid4().hex * 2,
+        result_hash=uuid.uuid4().hex * 2,
+        status=ParseRun.Status.VALIDATED,
+        started_at=completed_at - timedelta(seconds=1),
+        completed_at=completed_at,
+        total_row_count=1,
+        accepted_row_count=1,
+    )
+
+
+def make_publishable_snapshot(
+    *,
+    parse_run: ParseRun | None = None,
+    item_code: str = "212",
+    source_date: date = date(2026, 8, 29),
+    current_price: Decimal = Decimal("8000"),
+) -> RetailPriceSnapshot:
+    selected_parse_run = parse_run or create_distinct_validated_parse_run()
+    series = create_series(item_code=item_code, item_name=f"품목 {item_code}")
+    snapshot = replay_snapshot(
+        parse_run=selected_parse_run,
+        series=series,
+        source_effective_date=source_date,
+        current_price=current_price,
+        source_row_sha256=item_code.zfill(64),
+    )
+    persist_reference_price_facts(
+        snapshot_id=snapshot.id,
+        reference_values={
+            "WEEK": Decimal("10000"),
+            "MONTH": None,
+            "YEAR": Decimal("8000"),
+        },
+    )
+    return snapshot
+
+
+def test_canonical_entry_contains_only_complete_typed_public_facts() -> None:
+    snapshot = make_publishable_snapshot()
+
+    data = canonical_snapshot_data(snapshot)
+
+    assert data["entry_hash_version"] == ENTRY_HASH_VERSION
+    series = cast(dict[str, Any], data["series"])
+    snapshot_data = cast(dict[str, Any], data["snapshot"])
+    references = cast(list[dict[str, Any]], data["references"])
+    assert series["item_code"] == "212"
+    assert snapshot_data["current_price"] == "8000"
+    assert [reference["period"] for reference in references] == [
+        "WEEK",
+        "MONTH",
+        "YEAR",
+    ]
+    assert references[1]["value"] is None
+    assert references[1]["change"]["direction"] == "UNAVAILABLE"
+    serialized = str(data)
+    assert "fetch" not in serialized
+    assert "serviceKey" not in serialized
+
+
+def test_fact_and_set_hashes_are_deterministic_and_order_independent() -> None:
+    parse_run = create_validated_parse_run()
+    first = make_publishable_snapshot(parse_run=parse_run, item_code="212")
+    second = make_publishable_snapshot(
+        parse_run=parse_run,
+        item_code="213",
+        source_date=date(2026, 8, 28),
+        current_price=Decimal("9000"),
+    )
+
+    forward = build_publication_fact_set([first, second])
+    reverse = build_publication_fact_set([second, first])
+
+    assert forward == reverse
+    assert len(forward.entries) == 2
+    assert [entry.ordinal for entry in forward.entries] == [1, 2]
+    assert forward.entries[0].snapshot_id == first.id
+    assert forward.typed_fact_set_sha256 == reverse.typed_fact_set_sha256
+    assert len(forward.typed_fact_set_sha256) == 64
+    assert forward.source_effective_date_min == date(2026, 8, 28)
+    assert forward.source_effective_date_max == date(2026, 8, 29)
+    assert snapshot_fact_sha256(first) == forward.entries[0].fact_sha256
+    assert FACT_SET_HASH_VERSION == "recent-retail-fact-set-v1"
+
+
+def test_hash_changes_when_a_semantic_public_fact_changes() -> None:
+    first = make_publishable_snapshot(item_code="212", current_price=Decimal("8000"))
+    second = make_publishable_snapshot(item_code="213", current_price=Decimal("8001"))
+
+    assert snapshot_fact_sha256(first) != snapshot_fact_sha256(second)
+
+
+def test_set_rejects_empty_duplicate_or_mixed_generation_membership() -> None:
+    first = make_publishable_snapshot(item_code="212")
+    second = make_publishable_snapshot(item_code="213")
+
+    with pytest.raises(ValidationError, match="empty"):
+        build_publication_fact_set([])
+    with pytest.raises(ValidationError, match="repeat"):
+        build_publication_fact_set([first, first])
+    with pytest.raises(ValidationError, match="one parse"):
+        build_publication_fact_set([first, second])
+
+
+def test_entry_rejects_missing_reference_or_change_fact() -> None:
+    parse_run = create_validated_parse_run()
+    series = create_series(item_code="212")
+    snapshot = replay_snapshot(parse_run=parse_run, series=series)
+
+    with pytest.raises(ValidationError, match="exactly"):
+        canonical_snapshot_data(snapshot)


