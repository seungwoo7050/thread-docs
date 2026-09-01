## `feat(public): derive historical chart geometry`

diff --git a/grocery/vnext_presentation.py b/grocery/vnext_presentation.py
new file mode 100644
index 0000000..7754b02
--- /dev/null
+++ b/grocery/vnext_presentation.py
@@ -0,0 +1,188 @@
+"""Template-safe presentation primitives for the historical public ledger."""
+
+from __future__ import annotations
+
+from collections.abc import Sequence
+from dataclasses import dataclass
+from decimal import ROUND_HALF_UP, Decimal
+from typing import Final
+
+_CHART_WIDTH: Final = Decimal("720")
+_CHART_HEIGHT: Final = Decimal("280")
+_PLOT_LEFT: Final = Decimal("64")
+_PLOT_RIGHT: Final = Decimal("704")
+_PLOT_TOP: Final = Decimal("16")
+_PLOT_BOTTOM: Final = Decimal("240")
+
+
+@dataclass(frozen=True, slots=True)
+class MonthlyChartDatum:
+    year_month: str
+    provider_mean: Decimal
+    provider_low: Decimal
+    provider_high: Decimal
+
+
+def decimal_machine(value: Decimal) -> str:
+    """Return one finite, non-exponential decimal for a ``data`` attribute."""
+
+    if not value.is_finite():
+        raise ValueError("Public decimal values must be finite")
+    return format(value, "f")
+
+
+def format_provider_krw(value: Decimal) -> str:
+    """Display an exact provider value while preserving up to two decimals."""
+
+    if not value.is_finite() or value <= 0:
+        raise ValueError("Provider KRW values must be finite and positive")
+    normalized = format(value, ",.2f").rstrip("0").rstrip(".")
+    return f"{normalized}원"
+
+
+def format_year_month(value: str) -> tuple[str, str]:
+    if len(value) != 6 or not value.isdecimal():
+        raise ValueError("Historical month must use YYYYMM")
+    year = int(value[:4])
+    month = int(value[4:])
+    if year < 1 or month < 1 or month > 12:
+        raise ValueError("Historical month is out of range")
+    return f"{year:04d}-{month:02d}", f"{year}년 {month}월"
+
+
+def range_meter(
+    *,
+    minimum: Decimal,
+    mean: Decimal,
+    maximum: Decimal,
+    scale_minimum: Decimal,
+    scale_maximum: Decimal,
+) -> dict[str, str]:
+    """Map one provider range onto a shared neutral 0–100 scale."""
+
+    values = (minimum, mean, maximum, scale_minimum, scale_maximum)
+    if any(not value.is_finite() for value in values):
+        raise ValueError("Range-meter values must be finite")
+    if not (scale_minimum <= minimum <= mean <= maximum <= scale_maximum):
+        raise ValueError("Range-meter values are not ordered")
+    span = scale_maximum - scale_minimum
+    positions: tuple[Decimal, ...]
+    if span == 0:
+        positions = (Decimal("50"),) * 3
+    else:
+        positions = tuple(
+            (value - scale_minimum) * Decimal("100") / span
+            for value in (minimum, mean, maximum)
+        )
+    return {
+        "minimum_x": _svg_number(positions[0]),
+        "mean_x": _svg_number(positions[1]),
+        "maximum_x": _svg_number(positions[2]),
+    }
+
+
+def build_history_chart(data: Sequence[MonthlyChartDatum]) -> dict[str, object]:
+    """Build a supplementary mean line and provider low/high band."""
+
+    if len(data) < 2:
+        raise ValueError("A monthly chart requires at least two points")
+    for datum in data:
+        values = (datum.provider_low, datum.provider_mean, datum.provider_high)
+        if any(not value.is_finite() or value <= 0 for value in values):
+            raise ValueError("Monthly chart values must be finite and positive")
+        if not datum.provider_low <= datum.provider_mean <= datum.provider_high:
+            raise ValueError("Monthly chart ranges are not ordered")
+
+    scale_minimum = min(datum.provider_low for datum in data)
+    scale_maximum = max(datum.provider_high for datum in data)
+    horizontal_span = _PLOT_RIGHT - _PLOT_LEFT
+    vertical_span = _PLOT_BOTTOM - _PLOT_TOP
+
+    def x_position(index: int) -> Decimal:
+        return _PLOT_LEFT + horizontal_span * Decimal(index) / Decimal(len(data) - 1)
+
+    def y_position(value: Decimal) -> Decimal:
+        if scale_maximum == scale_minimum:
+            return (_PLOT_TOP + _PLOT_BOTTOM) / Decimal("2")
+        return _PLOT_BOTTOM - (
+            (value - scale_minimum) * vertical_span / (scale_maximum - scale_minimum)
+        )
+
+    indexed_runs: list[list[tuple[int, MonthlyChartDatum]]] = []
+    for index, datum in enumerate(data):
+        if not indexed_runs or _month_number(datum.year_month) != (
+            _month_number(indexed_runs[-1][-1][1].year_month) + 1
+        ):
+            indexed_runs.append([])
+        indexed_runs[-1].append((index, datum))
+
+    mean_segments: list[dict[str, str]] = []
+    range_segments: list[dict[str, str]] = []
+    for run in indexed_runs:
+        if len(run) < 2:
+            continue
+        mean_points = [
+            f"{_svg_number(x_position(index))},{_svg_number(y_position(datum.provider_mean))}"
+            for index, datum in run
+        ]
+        upper_points = [
+            f"{_svg_number(x_position(index))},{_svg_number(y_position(datum.provider_high))}"
+            for index, datum in run
+        ]
+        lower_points = [
+            f"{_svg_number(x_position(index))},{_svg_number(y_position(datum.provider_low))}"
+            for index, datum in reversed(run)
+        ]
+        mean_segments.append({"points": " ".join(mean_points)})
+        range_segments.append({"points": " ".join((*upper_points, *lower_points))})
+    point_context = [
+        {
+            "x": _svg_number(x_position(index)),
+            "y": _svg_number(y_position(datum.provider_mean)),
+        }
+        for index, datum in enumerate(data)
+    ]
+
+    tick_values = (
+        scale_maximum,
+        (scale_minimum + scale_maximum) / Decimal("2"),
+        scale_minimum,
+    )
+    y_ticks = [
+        {
+            "x1": _svg_number(_PLOT_LEFT),
+            "x2": _svg_number(_PLOT_RIGHT),
+            "y": _svg_number(y_position(value)),
+            "label_x": "4",
+            "label_y": _svg_number(y_position(value) + Decimal("4")),
+            "label": format_provider_krw(value),
+        }
+        for value in tick_values
+    ]
+    tick_indexes = sorted({0, len(data) // 2, len(data) - 1})
+    x_ticks = [
+        {
+            "x": _svg_number(x_position(index)),
+            "y": "268",
+            "label": f"{data[index].year_month[:4]}.{data[index].year_month[4:]}",
+        }
+        for index in tick_indexes
+    ]
+    return {
+        "view_box": f"0 0 {_svg_number(_CHART_WIDTH)} {_svg_number(_CHART_HEIGHT)}",
+        "y_ticks": y_ticks,
+        "x_ticks": x_ticks,
+        "range_segments": range_segments,
+        "mean_segments": mean_segments,
+        "points": point_context,
+    }
+
+
+def _svg_number(value: Decimal) -> str:
+    rounded = value.quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)
+    return format(rounded, "f").rstrip("0").rstrip(".") or "0"
+
+
+def _month_number(value: str) -> int:
+    iso, _label = format_year_month(value)
+    return int(iso[:4]) * 12 + int(iso[5:]) - 1


## `fix(public): preserve monthly chart gaps`

diff --git a/grocery/vnext_presentation.py b/grocery/vnext_presentation.py
index 7754b02..96aad32 100644
--- a/grocery/vnext_presentation.py
+++ b/grocery/vnext_presentation.py
@@ -71,8 +71,7 @@ def range_meter(
         positions = (Decimal("50"),) * 3
     else:
         positions = tuple(
-            (value - scale_minimum) * Decimal("100") / span
-            for value in (minimum, mean, maximum)
+            (value - scale_minimum) * Decimal("100") / span for value in (minimum, mean, maximum)
         )
     return {
         "minimum_x": _svg_number(positions[0]),
@@ -93,13 +92,27 @@ def build_history_chart(data: Sequence[MonthlyChartDatum]) -> dict[str, object]:
         if not datum.provider_low <= datum.provider_mean <= datum.provider_high:
             raise ValueError("Monthly chart ranges are not ordered")
 
+    month_numbers = [_month_number(datum.year_month) for datum in data]
+    if any(
+        current <= previous
+        for previous, current in zip(month_numbers, month_numbers[1:], strict=False)
+    ):
+        raise ValueError("Monthly chart points must be unique and chronological")
+    first_month = month_numbers[0]
+    last_month = month_numbers[-1]
+    month_span = last_month - first_month
+    if month_span < 1:
+        raise ValueError("A monthly chart requires at least two chronological slots")
+
     scale_minimum = min(datum.provider_low for datum in data)
     scale_maximum = max(datum.provider_high for datum in data)
     horizontal_span = _PLOT_RIGHT - _PLOT_LEFT
     vertical_span = _PLOT_BOTTOM - _PLOT_TOP
 
-    def x_position(index: int) -> Decimal:
-        return _PLOT_LEFT + horizontal_span * Decimal(index) / Decimal(len(data) - 1)
+    def x_position(month_number: int) -> Decimal:
+        return _PLOT_LEFT + (
+            horizontal_span * Decimal(month_number - first_month) / Decimal(month_span)
+        )
 
     def y_position(value: Decimal) -> Decimal:
         if scale_maximum == scale_minimum:
@@ -109,12 +122,10 @@ def build_history_chart(data: Sequence[MonthlyChartDatum]) -> dict[str, object]:
         )
 
     indexed_runs: list[list[tuple[int, MonthlyChartDatum]]] = []
-    for index, datum in enumerate(data):
-        if not indexed_runs or _month_number(datum.year_month) != (
-            _month_number(indexed_runs[-1][-1][1].year_month) + 1
-        ):
+    for month_number, datum in zip(month_numbers, data, strict=True):
+        if not indexed_runs or month_number != indexed_runs[-1][-1][0] + 1:
             indexed_runs.append([])
-        indexed_runs[-1].append((index, datum))
+        indexed_runs[-1].append((month_number, datum))
 
     mean_segments: list[dict[str, str]] = []
     range_segments: list[dict[str, str]] = []
@@ -122,25 +133,34 @@ def build_history_chart(data: Sequence[MonthlyChartDatum]) -> dict[str, object]:
         if len(run) < 2:
             continue
         mean_points = [
-            f"{_svg_number(x_position(index))},{_svg_number(y_position(datum.provider_mean))}"
-            for index, datum in run
+            f"{_svg_number(x_position(month_number))},{_svg_number(y_position(datum.provider_mean))}"
+            for month_number, datum in run
         ]
         upper_points = [
-            f"{_svg_number(x_position(index))},{_svg_number(y_position(datum.provider_high))}"
-            for index, datum in run
+            f"{_svg_number(x_position(month_number))},{_svg_number(y_position(datum.provider_high))}"
+            for month_number, datum in run
         ]
         lower_points = [
-            f"{_svg_number(x_position(index))},{_svg_number(y_position(datum.provider_low))}"
-            for index, datum in reversed(run)
+            f"{_svg_number(x_position(month_number))},{_svg_number(y_position(datum.provider_low))}"
+            for month_number, datum in reversed(run)
         ]
         mean_segments.append({"points": " ".join(mean_points)})
         range_segments.append({"points": " ".join((*upper_points, *lower_points))})
     point_context = [
         {
-            "x": _svg_number(x_position(index)),
+            "x": _svg_number(x_position(month_number)),
             "y": _svg_number(y_position(datum.provider_mean)),
         }
-        for index, datum in enumerate(data)
+        for month_number, datum in zip(month_numbers, data, strict=True)
+    ]
+    present_months = set(month_numbers)
+    gap_markers = [
+        {
+            "x": _svg_number(x_position(month_number)),
+            "label": _month_label(month_number),
+        }
+        for month_number in range(first_month, last_month + 1)
+        if month_number not in present_months
     ]
 
     tick_values = (
@@ -159,14 +179,14 @@ def build_history_chart(data: Sequence[MonthlyChartDatum]) -> dict[str, object]:
         }
         for value in tick_values
     ]
-    tick_indexes = sorted({0, len(data) // 2, len(data) - 1})
+    tick_months = sorted({first_month, first_month + month_span // 2, last_month})
     x_ticks = [
         {
-            "x": _svg_number(x_position(index)),
+            "x": _svg_number(x_position(month_number)),
             "y": "268",
-            "label": f"{data[index].year_month[:4]}.{data[index].year_month[4:]}",
+            "label": _month_label(month_number),
         }
-        for index in tick_indexes
+        for month_number in tick_months
     ]
     return {
         "view_box": f"0 0 {_svg_number(_CHART_WIDTH)} {_svg_number(_CHART_HEIGHT)}",
@@ -175,6 +195,7 @@ def build_history_chart(data: Sequence[MonthlyChartDatum]) -> dict[str, object]:
         "range_segments": range_segments,
         "mean_segments": mean_segments,
         "points": point_context,
+        "gap_markers": gap_markers,
     }
 
 
@@ -186,3 +207,8 @@ def _svg_number(value: Decimal) -> str:
 def _month_number(value: str) -> int:
     iso, _label = format_year_month(value)
     return int(iso[:4]) * 12 + int(iso[5:]) - 1
+
+
+def _month_label(value: int) -> str:
+    year, zero_based_month = divmod(value, 12)
+    return f"{year:04d}.{zero_based_month + 1:02d}"


## `feat(public): build complete monthly history context`

diff --git a/grocery/historical_history_read.py b/grocery/historical_history_read.py
new file mode 100644
index 0000000..7ef59e4
--- /dev/null
+++ b/grocery/historical_history_read.py
@@ -0,0 +1,173 @@
+"""Monthly history context from one active historical publication."""
+
+from __future__ import annotations
+
+import uuid
+from collections import defaultdict
+
+from grocery.historical_identity_models import HistoricalRetailSeriesKey, RetailRegionKey
+from grocery.historical_monthly_models import MonthlyRegionalRetailPrice
+from grocery.historical_public_read import (
+    ActiveHistoricalPublication,
+    PublicParameterError,
+    PublicReadIntegrityError,
+)
+from grocery.vnext_presentation import (
+    MonthlyChartDatum,
+    build_history_chart,
+    decimal_machine,
+    format_provider_krw,
+    format_year_month,
+)
+
+
+def history_context(
+    active: ActiveHistoricalPublication,
+    series: HistoricalRetailSeriesKey,
+    *,
+    selected_region_id: uuid.UUID | None,
+    selected_range: int,
+) -> dict[str, object]:
+    facts = list(
+        MonthlyRegionalRetailPrice.objects.filter(
+            collection=active.monthly_collection,
+            series=series,
+        )
+        .select_related("region")
+        .order_by("region__region_name", "region__region_code", "year_month", "id")
+    )
+    by_region: dict[uuid.UUID, list[MonthlyRegionalRetailPrice]] = defaultdict(list)
+    regions: dict[uuid.UUID, RetailRegionKey] = {}
+    for fact in facts:
+        _validate_monthly_fact(fact)
+        by_region[fact.region_id].append(fact)
+        regions[fact.region_id] = fact.region
+
+    complete_36 = {
+        region_id
+        for region_id, rows in by_region.items()
+        if _has_complete_months(rows, active.revision.month_max, 36)
+    }
+    if not complete_36:
+        raise PublicReadIntegrityError("The published series has no complete 36-month region.")
+    ordered_region_ids = sorted(
+        complete_36,
+        key=lambda region_id: (
+            regions[region_id].region_name,
+            regions[region_id].region_code,
+            str(region_id),
+        ),
+    )
+    region_options = [
+        {
+            "value": str(region_id),
+            "label": regions[region_id].region_name,
+            "selected": region_id == selected_region_id,
+        }
+        for region_id in ordered_region_ids
+    ]
+    base: dict[str, object] = {
+        "region_options": region_options,
+        "selected_region": None,
+        "selected_range": {"value": str(selected_range), "label": f"{selected_range}개월"},
+        "monthly_points": [],
+        "history_chart": None,
+    }
+    if selected_region_id is None:
+        if selected_range != 36:
+            raise PublicParameterError("Select a region before changing the historical range.")
+        base["range_options"] = _range_options(selected_range, allow_60=False)
+        return base
+    if selected_region_id not in complete_36:
+        raise PublicParameterError("The selected region is not available for this series.")
+    rows = by_region[selected_region_id]
+    allow_60 = _has_complete_months(rows, active.revision.month_max, 60)
+    if selected_range not in {12, 36} and not (selected_range == 60 and allow_60):
+        raise PublicParameterError("The selected range is not available for this region.")
+    selected_rows = _select_complete_months(rows, active.revision.month_max, selected_range)
+    region = regions[selected_region_id]
+    base.update(
+        {
+            "selected_region": {"value": str(selected_region_id), "label": region.region_name},
+            "range_options": _range_options(selected_range, allow_60=allow_60),
+            "monthly_points": [_monthly_point(row) for row in selected_rows],
+            "history_chart": build_history_chart(
+                [
+                    MonthlyChartDatum(
+                        row.year_month,
+                        row.provider_mean,
+                        row.provider_low,
+                        row.provider_high,
+                    )
+                    for row in selected_rows
+                ]
+            ),
+        }
+    )
+    return base
+
+
+def _validate_monthly_fact(row: MonthlyRegionalRetailPrice) -> None:
+    format_year_month(row.year_month)
+    values = (row.provider_low, row.provider_mean, row.provider_high)
+    if any(not value.is_finite() or value <= 0 for value in values):
+        raise PublicReadIntegrityError("Published provider prices are malformed.")
+    if not row.provider_low <= row.provider_mean <= row.provider_high:
+        raise PublicReadIntegrityError("Published provider ranges are malformed.")
+
+
+def _month_number(value: str) -> int:
+    format_year_month(value)
+    return int(value[:4]) * 12 + int(value[4:]) - 1
+
+
+def _month_value(number: int) -> str:
+    year, month = divmod(number, 12)
+    return f"{year:04d}{month + 1:02d}"
+
+
+def _expected_months(month_max: str, count: int) -> list[str]:
+    maximum = _month_number(month_max)
+    return [_month_value(value) for value in range(maximum - count + 1, maximum + 1)]
+
+
+def _has_complete_months(
+    rows: list[MonthlyRegionalRetailPrice], month_max: str, count: int
+) -> bool:
+    values = [row.year_month for row in rows]
+    expected = set(_expected_months(month_max, count))
+    return len(values) == len(set(values)) and expected <= set(values)
+
+
+def _select_complete_months(
+    rows: list[MonthlyRegionalRetailPrice], month_max: str, count: int
+) -> list[MonthlyRegionalRetailPrice]:
+    by_month = {row.year_month: row for row in rows}
+    expected = _expected_months(month_max, count)
+    if any(month not in by_month for month in expected):
+        raise PublicReadIntegrityError("The selected published monthly range is incomplete.")
+    return [by_month[month] for month in expected]
+
+
+def _range_options(selected_range: int, *, allow_60: bool) -> list[dict[str, object]]:
+    values = (12, 36, 60) if allow_60 else (12, 36)
+    return [
+        {"value": str(value), "label": f"{value}개월", "selected": value == selected_range}
+        for value in values
+    ]
+
+
+def _monthly_point(row: MonthlyRegionalRetailPrice) -> dict[str, object]:
+    period_iso, period_label = format_year_month(row.year_month)
+    return {
+        "available": True,
+        "period_iso": period_iso,
+        "period_label": period_label,
+        "mean_machine": decimal_machine(row.provider_mean),
+        "mean_label": format_provider_krw(row.provider_mean),
+        "minimum_machine": decimal_machine(row.provider_low),
+        "minimum_label": format_provider_krw(row.provider_low),
+        "maximum_machine": decimal_machine(row.provider_high),
+        "maximum_label": format_provider_krw(row.provider_high),
+        "gap_after": False,
+    }


