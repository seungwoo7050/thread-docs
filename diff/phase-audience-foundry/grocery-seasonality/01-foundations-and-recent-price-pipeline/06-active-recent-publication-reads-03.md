## `feat(public): extend active recent reads`

diff --git a/grocery/public_read.py b/grocery/public_read.py
index 523e297..5b6bb19 100644
--- a/grocery/public_read.py
+++ b/grocery/public_read.py
@@ -2,13 +2,16 @@
 
 from __future__ import annotations
 
+import uuid
+from collections.abc import Sequence
 from dataclasses import dataclass
 from datetime import datetime, timedelta
+from decimal import Decimal
 from typing import Final
 
 from django.conf import settings
 from django.core.exceptions import ObjectDoesNotExist, ValidationError
-from django.db.models import Prefetch, QuerySet
+from django.db.models import Prefetch
 from django.utils import timezone
 
 from grocery.models import (
@@ -32,6 +35,7 @@ from grocery.presentation import (
 
 RECENT_RETAIL_CHANNEL: Final = "RECENT_RETAIL"
 PUBLIC_RESULT_LIMIT: Final = 100
+PUBLIC_PAGE_SIZE: Final = 30
 KAMIS_LANDING_URL: Final = "https://www.data.go.kr/data/15156063/openapi.do"
 
 _CATEGORY_CODES: Final = {"vegetable": "200", "fruit": "400"}
@@ -41,6 +45,13 @@ _PERIOD_LABELS: Final = {
     "YEAR": "1년 전 제공값",
 }
 _PERIOD_ORDER: Final = {"WEEK": 1, "MONTH": 2, "YEAR": 3}
+_PUBLIC_PERIODS: Final = {"week": "WEEK", "month": "MONTH", "year": "YEAR"}
+_PUBLIC_DIRECTIONS: Final = {
+    "lower": PriceChangeFact.Direction.LOWER,
+    "equal": PriceChangeFact.Direction.EQUAL,
+    "higher": PriceChangeFact.Direction.HIGHER,
+    "unavailable": PriceChangeFact.Direction.UNAVAILABLE,
+}
 _COVERAGE_LABELS: Final = {
     "KAMIS_RETAIL_ALL_REGIONS_22_CITIES_V1": "KAMIS 소매 조사 22개 도시 지역 전체 집계",
 }
@@ -144,34 +155,56 @@ def publication_entries(
     *,
     query: str,
     category: str,
-) -> QuerySet[PublicationEntry]:
-    catalog_week_references = ReferencePrice.objects.filter(
-        period=ReferencePrice.Period.WEEK
-    ).select_related("change_fact")
-    entries = (
-        active.revision.entries.select_related("snapshot__series")
-        .prefetch_related(
-            Prefetch(
-                "snapshot__reference_prices",
-                queryset=catalog_week_references,
-                to_attr="catalog_week_references",
-            )
-        )
-        .order_by(
-            "snapshot__series__category_code",
-            "snapshot__series__item_name",
-            "snapshot__series__item_code",
-            "snapshot__series__variety_code",
-            "snapshot__series__grade_code",
-            "snapshot__series__raw_unit",
-            "snapshot__series__raw_unit_size",
+    period: str = "week",
+    direction: str = "all",
+    sort: str = "name",
+) -> list[PublicationEntry]:
+    try:
+        selected_period = _PUBLIC_PERIODS[period]
+    except KeyError as error:
+        raise ValidationError("Unknown public comparison period.") from error
+    if direction != "all" and direction not in _PUBLIC_DIRECTIONS:
+        raise ValidationError("Unknown public comparison direction.")
+    if sort not in {"name", "change_asc", "change_desc"}:
+        raise ValidationError("Unknown public catalog sort.")
+    if category and category not in _CATEGORY_CODES:
+        raise ValidationError("Unknown public category.")
+
+    catalog_references = ReferencePrice.objects.filter(period=selected_period).select_related(
+        "change_fact"
+    )
+    comparison_attribute = (
+        "catalog_week_references"
+        if selected_period == ReferencePrice.Period.WEEK
+        else "catalog_comparison_references"
+    )
+    entries = active.revision.entries.select_related("snapshot__series").prefetch_related(
+        Prefetch(
+            "snapshot__reference_prices",
+            queryset=catalog_references,
+            to_attr=comparison_attribute,
         )
     )
-    if category:
-        entries = entries.filter(snapshot__series__category_code=_CATEGORY_CODES[category])
-    if query:
-        entries = entries.filter(snapshot__series__item_name__icontains=query)
-    return entries[:PUBLIC_RESULT_LIMIT]
+    entries = entries.order_by(
+        "snapshot__series__category_code",
+        "snapshot__series__item_name",
+        "snapshot__series__item_code",
+        "snapshot__series__variety_code",
+        "snapshot__series__grade_code",
+        "snapshot__series__raw_unit",
+        "snapshot__series__raw_unit_size",
+        "snapshot__series_id",
+    )
+    materialized = list(entries[: PUBLIC_RESULT_LIMIT + 1])
+    if len(materialized) > PUBLIC_RESULT_LIMIT:
+        raise ValidationError("The active catalog exceeds its public result limit.")
+    return _filter_and_sort_catalog_entries(
+        materialized,
+        query=query,
+        category=category,
+        direction=direction,
+        sort=sort,
+    )
 
 
 def catalog_item(
@@ -189,12 +222,92 @@ def catalog_item(
         "variety_name": series.variety_name,
         "grade_name": series.grade_name,
         "unit_label": format_unit(series.raw_unit, series.raw_unit_size),
+        "current_price_machine": format(snapshot.current_price, "f"),
         "current_price_label": format_krw(snapshot.current_price),
         "source_date_iso": snapshot.source_effective_date.isoformat(),
         "source_date_label": format_korean_date(snapshot.source_effective_date),
         "freshness_state": active.freshness_state,
         "freshness_label": active.freshness_label,
-        "week_comparison": _catalog_week_comparison(snapshot),
+        "comparison": _catalog_comparison(snapshot),
+        # Kept for the Phase 0 template until the vNext frontend commit lands.
+        "week_comparison": _catalog_comparison(snapshot),
+    }
+
+
+def publication_entries_for_series(
+    active: ActivePublication, series_ids: Sequence[uuid.UUID]
+) -> list[PublicationEntry]:
+    """Materialize at most five selected members with one bounded reference prefetch."""
+
+    if len(series_ids) > 5:
+        raise ValidationError("A public selection cannot exceed five series.")
+    if not series_ids:
+        return []
+    week_references = ReferencePrice.objects.filter(
+        period=ReferencePrice.Period.WEEK
+    ).select_related("change_fact")
+    entries = list(
+        active.revision.entries.select_related("snapshot__series")
+        .prefetch_related(
+            Prefetch(
+                "snapshot__reference_prices",
+                queryset=week_references,
+                to_attr="catalog_comparison_references",
+            )
+        )
+        .filter(snapshot__series_id__in=series_ids)
+    )
+    by_series = {entry.snapshot.series_id: entry for entry in entries}
+    return [by_series[series_id] for series_id in series_ids if series_id in by_series]
+
+
+def publication_candidate_entries(
+    active: ActivePublication, *, excluded_series_ids: Sequence[uuid.UUID]
+) -> list[PublicationEntry]:
+    """Return only current publication members for the no-JS add control."""
+
+    return list(
+        active.revision.entries.select_related("snapshot__series")
+        .exclude(snapshot__series_id__in=excluded_series_ids)
+        .order_by(
+            "snapshot__series__category_code",
+            "snapshot__series__item_name",
+            "snapshot__series__item_code",
+            "snapshot__series__variety_code",
+            "snapshot__series__grade_code",
+            "snapshot__series__raw_unit",
+            "snapshot__series__raw_unit_size",
+            "snapshot__series_id",
+        )[:PUBLIC_RESULT_LIMIT]
+    )
+
+
+def selection_item_context(
+    entry: PublicationEntry,
+    active: ActivePublication,
+    *,
+    detail_url: str,
+    remove_url: str,
+) -> dict[str, object]:
+    item = catalog_item(entry, active, url=detail_url)
+    item.update(
+        {
+            "series_value": str(entry.snapshot.series_id),
+            "detail_url": detail_url,
+            "remove_url": remove_url,
+        }
+    )
+    return item
+
+
+def selection_candidate_context(entry: PublicationEntry) -> dict[str, str]:
+    series = entry.snapshot.series
+    return {
+        "value": str(series.id),
+        "label": (
+            f"{series.item_name} · {series.variety_name} · {series.grade_name} · "
+            f"{format_unit(series.raw_unit, series.raw_unit_size)}"
+        ),
     }
 
 
@@ -240,14 +353,85 @@ def detail_context(entry: PublicationEntry, active: ActivePublication) -> dict[s
     }
 
 
-def _catalog_week_comparison(snapshot: object) -> dict[str, object]:
-    references = getattr(snapshot, "catalog_week_references", None)
+def _catalog_comparison(snapshot: object) -> dict[str, object]:
+    return _comparison_context(_catalog_reference(snapshot))
+
+
+def _catalog_reference(snapshot: object) -> ReferencePrice:
+    references = getattr(snapshot, "catalog_comparison_references", None)
+    if references is None:
+        references = getattr(snapshot, "catalog_week_references", None)
     if not isinstance(references, list) or len(references) != 1:
-        raise ValidationError("Published catalog requires exactly one WEEK reference.")
+        raise ValidationError("Published catalog requires exactly one selected reference.")
     reference = references[0]
-    if not isinstance(reference, ReferencePrice) or reference.period != ReferencePrice.Period.WEEK:
-        raise ValidationError("Published catalog WEEK reference is malformed.")
-    return _comparison_context(reference)
+    if not isinstance(reference, ReferencePrice) or reference.period not in _PERIOD_ORDER:
+        raise ValidationError("Published catalog reference is malformed.")
+    return reference
+
+
+def _filter_and_sort_catalog_entries(
+    entries: Sequence[PublicationEntry],
+    *,
+    query: str = "",
+    category: str = "",
+    direction: str,
+    sort: str,
+) -> list[PublicationEntry]:
+    validated: list[tuple[PublicationEntry, ReferencePrice]] = []
+    for entry in entries:
+        reference = _catalog_reference(entry.snapshot)
+        _comparison_context(reference)
+        validated.append((entry, reference))
+    if category:
+        category_code = _CATEGORY_CODES[category]
+        validated = [
+            (entry, reference)
+            for entry, reference in validated
+            if entry.snapshot.series.category_code == category_code
+        ]
+    if query:
+        folded_query = query.casefold()
+        validated = [
+            (entry, reference)
+            for entry, reference in validated
+            if folded_query in entry.snapshot.series.item_name.casefold()
+        ]
+    if direction != "all":
+        expected = _PUBLIC_DIRECTIONS[direction]
+        validated = [
+            (entry, reference)
+            for entry, reference in validated
+            if reference.change_fact.direction == expected
+        ]
+    if sort == "name":
+        return [entry for entry, _reference in validated]
+
+    descending = sort == "change_desc"
+
+    def comparison_key(
+        value: tuple[PublicationEntry, ReferencePrice],
+    ) -> tuple[bool, Decimal, tuple[object, ...]]:
+        entry, reference = value
+        change = reference.change_fact
+        unavailable = change.direction == PriceChangeFact.Direction.UNAVAILABLE
+        percentage = change.signed_percentage or Decimal("0")
+        if descending:
+            percentage = -percentage
+        series = entry.snapshot.series
+        identity = (
+            series.category_code,
+            series.item_name,
+            series.item_code,
+            series.variety_code,
+            series.grade_code,
+            series.raw_unit,
+            series.raw_unit_size,
+            str(series.id),
+        )
+        return unavailable, percentage, identity
+
+    validated.sort(key=comparison_key)
+    return [entry for entry, _reference in validated]
 
 
 def _comparison_context(reference: ReferencePrice) -> dict[str, object]:


