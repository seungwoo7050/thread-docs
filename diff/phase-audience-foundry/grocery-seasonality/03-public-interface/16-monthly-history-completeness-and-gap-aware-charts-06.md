## `feat(presentation): add consumer display summaries`

diff --git a/grocery/historical_daily_read.py b/grocery/historical_daily_read.py
index 8a596d0..1a6e89e 100644
--- a/grocery/historical_daily_read.py
+++ b/grocery/historical_daily_read.py
@@ -15,7 +15,12 @@ from grocery.historical_public_read import (
     PublicReadIntegrityError,
 )
 from grocery.presentation import format_korean_date
-from grocery.vnext_presentation import decimal_machine, format_provider_krw, range_meter
+from grocery.vnext_presentation import (
+    build_market_summary,
+    decimal_machine,
+    format_provider_krw,
+    range_meter,
+)
 
 HISTORICAL_PAGE_SIZE: Final = 30
 
@@ -127,6 +132,10 @@ def markets_context(
             or row.market.region_id != region_id
         ):
             raise PublicReadIntegrityError("Published market prices are malformed.")
+    try:
+        market_summary = build_market_summary([row.provider_price for row in rows])
+    except ValueError as exc:
+        raise PublicReadIntegrityError("Published market summary facts are malformed.") from exc
     total = len(rows)
     total_pages = max(1, (total + HISTORICAL_PAGE_SIZE - 1) // HISTORICAL_PAGE_SIZE)
     if page > total_pages:
@@ -148,6 +157,7 @@ def markets_context(
             }
             for row in selected_rows
         ],
+        "market_summary": market_summary,
         "total_count": total,
         "page": page,
         "total_pages": total_pages,
diff --git a/grocery/historical_history_read.py b/grocery/historical_history_read.py
index 7ef59e4..56ed981 100644
--- a/grocery/historical_history_read.py
+++ b/grocery/historical_history_read.py
@@ -15,9 +15,10 @@ from grocery.historical_public_read import (
 from grocery.vnext_presentation import (
     MonthlyChartDatum,
     build_history_chart,
-    decimal_machine,
-    format_provider_krw,
+    build_history_summary,
+    build_history_year_groups,
     format_year_month,
+    monthly_display_point,
 )
 
 
@@ -86,22 +87,32 @@ def history_context(
         raise PublicParameterError("The selected range is not available for this region.")
     selected_rows = _select_complete_months(rows, active.revision.month_max, selected_range)
     region = regions[selected_region_id]
+    presentation_data = [
+        MonthlyChartDatum(
+            row.year_month,
+            row.provider_mean,
+            row.provider_low,
+            row.provider_high,
+        )
+        for row in selected_rows
+    ]
+    try:
+        monthly_points = [monthly_display_point(datum) for datum in presentation_data]
+        history_summary = build_history_summary(presentation_data)
+        history_year_groups = build_history_year_groups(presentation_data)
+        history_chart = build_history_chart(presentation_data)
+    except ValueError as exc:
+        raise PublicReadIntegrityError(
+            "Published monthly presentation facts are malformed."
+        ) from exc
     base.update(
         {
             "selected_region": {"value": str(selected_region_id), "label": region.region_name},
             "range_options": _range_options(selected_range, allow_60=allow_60),
-            "monthly_points": [_monthly_point(row) for row in selected_rows],
-            "history_chart": build_history_chart(
-                [
-                    MonthlyChartDatum(
-                        row.year_month,
-                        row.provider_mean,
-                        row.provider_low,
-                        row.provider_high,
-                    )
-                    for row in selected_rows
-                ]
-            ),
+            "monthly_points": monthly_points,
+            "history_summary": history_summary,
+            "history_year_groups": history_year_groups,
+            "history_chart": history_chart,
         }
     )
     return base
@@ -155,19 +166,3 @@ def _range_options(selected_range: int, *, allow_60: bool) -> list[dict[str, obj
         {"value": str(value), "label": f"{value}개월", "selected": value == selected_range}
         for value in values
     ]
-
-
-def _monthly_point(row: MonthlyRegionalRetailPrice) -> dict[str, object]:
-    period_iso, period_label = format_year_month(row.year_month)
-    return {
-        "available": True,
-        "period_iso": period_iso,
-        "period_label": period_label,
-        "mean_machine": decimal_machine(row.provider_mean),
-        "mean_label": format_provider_krw(row.provider_mean),
-        "minimum_machine": decimal_machine(row.provider_low),
-        "minimum_label": format_provider_krw(row.provider_low),
-        "maximum_machine": decimal_machine(row.provider_high),
-        "maximum_label": format_provider_krw(row.provider_high),
-        "gap_after": False,
-    }
diff --git a/grocery/tests/test_vnext_public_query_bounds.py b/grocery/tests/test_vnext_public_query_bounds.py
index f98777e..10105fc 100644
--- a/grocery/tests/test_vnext_public_query_bounds.py
+++ b/grocery/tests/test_vnext_public_query_bounds.py
@@ -141,14 +141,14 @@ def test_history_and_daily_ledgers_have_row_independent_query_counts() -> None:
     active, bundle = _sealed_historical()
 
     with CaptureQueriesContext(connection) as twelve:
-        history_context(
+        twelve_context = history_context(
             active,
             bundle.series,
             selected_region_id=bundle.region.id,
             selected_range=12,
         )
     with CaptureQueriesContext(connection) as thirty_six:
-        history_context(
+        thirty_six_context = history_context(
             active,
             bundle.series,
             selected_region_id=bundle.region.id,
@@ -157,7 +157,7 @@ def test_history_and_daily_ledgers_have_row_independent_query_counts() -> None:
     with CaptureQueriesContext(connection) as regions:
         regions_context(active, bundle.series, selected_date=None)
     with CaptureQueriesContext(connection) as markets:
-        markets_context(
+        market_context = markets_context(
             active,
             bundle.series,
             region_id=bundle.region.id,
@@ -168,6 +168,40 @@ def test_history_and_daily_ledgers_have_row_independent_query_counts() -> None:
     assert len(twelve) == len(thirty_six) == 1
     assert len(regions) == 2
     assert len(markets) == 2
+    assert twelve_context["history_summary"] == {
+        "latest": {
+            "period_iso": "2025-12",
+            "period_label": "2025년 12월",
+            "mean_machine": "1000.00",
+            "mean_label": "1,000원",
+        },
+        "lowest": {
+            "period_iso": "2025-01",
+            "period_label": "2025년 1월",
+            "mean_machine": "989.00",
+            "mean_label": "989원",
+        },
+        "highest": {
+            "period_iso": "2025-12",
+            "period_label": "2025년 12월",
+            "mean_machine": "1000.00",
+            "mean_label": "1,000원",
+        },
+    }
+    year_groups = cast(list[dict[str, object]], thirty_six_context["history_year_groups"])
+    assert [(group["year"], group["open"]) for group in year_groups] == [
+        ("2025", True),
+        ("2024", False),
+        ("2023", False),
+    ]
+    assert market_context["market_summary"] == {
+        "total_count": 1,
+        "total_count_label": "1곳",
+        "minimum_machine": "1000.00",
+        "minimum_label": "1,000원",
+        "maximum_machine": "1000.00",
+        "maximum_label": "1,000원",
+    }
 
 
 @pytest.mark.django_db
diff --git a/grocery/tests/test_vnext_public_read_contract.py b/grocery/tests/test_vnext_public_read_contract.py
index 1571e24..8fab8d1 100644
--- a/grocery/tests/test_vnext_public_read_contract.py
+++ b/grocery/tests/test_vnext_public_read_contract.py
@@ -7,7 +7,14 @@ from django.http import QueryDict
 
 from grocery.forms import CatalogForm, HistoryForm, MarketsForm, RegionsForm, parse_selection_query
 from grocery.security import SECURITY_HEADERS
-from grocery.vnext_presentation import MonthlyChartDatum, build_history_chart, range_meter
+from grocery.vnext_presentation import (
+    MonthlyChartDatum,
+    build_history_chart,
+    build_history_summary,
+    build_history_year_groups,
+    build_market_summary,
+    range_meter,
+)
 
 
 @pytest.mark.parametrize(
@@ -91,5 +98,71 @@ def test_regional_meter_uses_one_server_side_decimal_scale() -> None:
     ) == {"minimum_x": "25", "mean_x": "50", "maximum_x": "75"}
 
 
+def test_monthly_summaries_and_year_groups_are_server_prepared_and_deterministic() -> None:
+    data = [
+        MonthlyChartDatum("202412", Decimal("100"), Decimal("90"), Decimal("110")),
+        MonthlyChartDatum("202501", Decimal("100"), Decimal("80"), Decimal("120")),
+        MonthlyChartDatum("202502", Decimal("130"), Decimal("120"), Decimal("140")),
+    ]
+
+    assert build_history_summary(data) == {
+        "latest": {
+            "period_iso": "2025-02",
+            "period_label": "2025년 2월",
+            "mean_machine": "130",
+            "mean_label": "130원",
+        },
+        "lowest": {
+            "period_iso": "2024-12",
+            "period_label": "2024년 12월",
+            "mean_machine": "100",
+            "mean_label": "100원",
+        },
+        "highest": {
+            "period_iso": "2025-02",
+            "period_label": "2025년 2월",
+            "mean_machine": "130",
+            "mean_label": "130원",
+        },
+    }
+    groups = build_history_year_groups(data)
+    assert [(group["year"], group["is_latest"], group["open"]) for group in groups] == [
+        ("2025", True, True),
+        ("2024", False, False),
+    ]
+    newest_points = cast(list[dict[str, object]], groups[0]["points"])
+    assert [point["period_iso"] for point in newest_points] == [
+        "2025-01",
+        "2025-02",
+    ]
+
+
+def test_market_summary_uses_all_exact_provider_observations() -> None:
+    assert build_market_summary(
+        [Decimal("1250.50"), Decimal("900"), Decimal("900.00"), Decimal("1800")]
+    ) == {
+        "total_count": 4,
+        "total_count_label": "4곳",
+        "minimum_machine": "900",
+        "minimum_label": "900원",
+        "maximum_machine": "1800",
+        "maximum_label": "1,800원",
+    }
+
+
+def test_presentation_summaries_fail_closed_for_missing_or_noncanonical_facts() -> None:
+    with pytest.raises(ValueError):
+        build_history_summary([])
+    with pytest.raises(ValueError):
+        build_history_year_groups(
+            [
+                MonthlyChartDatum("202501", Decimal("100"), Decimal("90"), Decimal("110")),
+                MonthlyChartDatum("202501", Decimal("100"), Decimal("90"), Decimal("110")),
+            ]
+        )
+    with pytest.raises(ValueError):
+        build_market_summary([Decimal("NaN")])
+
+
 def test_public_referrer_policy_sends_no_query_state() -> None:
     assert SECURITY_HEADERS["Referrer-Policy"] == "no-referrer"
diff --git a/grocery/vnext_presentation.py b/grocery/vnext_presentation.py
index 96aad32..ac5a93a 100644
--- a/grocery/vnext_presentation.py
+++ b/grocery/vnext_presentation.py
@@ -50,6 +50,87 @@ def format_year_month(value: str) -> tuple[str, str]:
     return f"{year:04d}-{month:02d}", f"{year}년 {month}월"
 
 
+def monthly_display_point(datum: MonthlyChartDatum) -> dict[str, object]:
+    """Format one validated monthly provider fact for an SSR ledger."""
+
+    _validate_monthly_data((datum,))
+    period_iso, period_label = format_year_month(datum.year_month)
+    return {
+        "available": True,
+        "period_iso": period_iso,
+        "period_label": period_label,
+        "mean_machine": decimal_machine(datum.provider_mean),
+        "mean_label": format_provider_krw(datum.provider_mean),
+        "minimum_machine": decimal_machine(datum.provider_low),
+        "minimum_label": format_provider_krw(datum.provider_low),
+        "maximum_machine": decimal_machine(datum.provider_high),
+        "maximum_label": format_provider_krw(datum.provider_high),
+        "gap_after": False,
+    }
+
+
+def build_history_summary(data: Sequence[MonthlyChartDatum]) -> dict[str, object]:
+    """Prepare latest and extrema-of-mean facts without template arithmetic.
+
+    Input must be unique and chronological. Stable ``min``/``max`` selection means
+    that the earliest month represents an exact tie.
+    """
+
+    _validate_monthly_data(data, require_chronological=True)
+    if not data:
+        raise ValueError("A monthly summary requires at least one point")
+    latest = data[-1]
+    lowest = min(data, key=lambda datum: datum.provider_mean)
+    highest = max(data, key=lambda datum: datum.provider_mean)
+    return {
+        "latest": _mean_summary_item(latest),
+        "lowest": _mean_summary_item(lowest),
+        "highest": _mean_summary_item(highest),
+    }
+
+
+def build_history_year_groups(data: Sequence[MonthlyChartDatum]) -> list[dict[str, object]]:
+    """Group chronological monthly facts by year, newest year first."""
+
+    _validate_monthly_data(data, require_chronological=True)
+    if not data:
+        raise ValueError("History year groups require at least one point")
+    points_by_year: dict[str, list[dict[str, object]]] = {}
+    for datum in data:
+        year = datum.year_month[:4]
+        points_by_year.setdefault(year, []).append(monthly_display_point(datum))
+    latest_year = data[-1].year_month[:4]
+    return [
+        {
+            "year": year,
+            "label": f"{year}년",
+            "is_latest": year == latest_year,
+            "open": year == latest_year,
+            "points": points_by_year[year],
+        }
+        for year in reversed(points_by_year)
+    ]
+
+
+def build_market_summary(values: Sequence[Decimal]) -> dict[str, object]:
+    """Summarize the full validated market result set before pagination."""
+
+    if not values:
+        raise ValueError("A market summary requires at least one observation")
+    if any(not value.is_finite() or value <= 0 for value in values):
+        raise ValueError("Market observations must be finite and positive")
+    minimum = min(values)
+    maximum = max(values)
+    return {
+        "total_count": len(values),
+        "total_count_label": f"{len(values)}곳",
+        "minimum_machine": decimal_machine(minimum),
+        "minimum_label": format_provider_krw(minimum),
+        "maximum_machine": decimal_machine(maximum),
+        "maximum_label": format_provider_krw(maximum),
+    }
+
+
 def range_meter(
     *,
     minimum: Decimal,
@@ -199,6 +280,34 @@ def build_history_chart(data: Sequence[MonthlyChartDatum]) -> dict[str, object]:
     }
 
 
+def _validate_monthly_data(
+    data: Sequence[MonthlyChartDatum], *, require_chronological: bool = False
+) -> None:
+    month_numbers: list[int] = []
+    for datum in data:
+        values = (datum.provider_low, datum.provider_mean, datum.provider_high)
+        if any(not value.is_finite() or value <= 0 for value in values):
+            raise ValueError("Monthly provider values must be finite and positive")
+        if not datum.provider_low <= datum.provider_mean <= datum.provider_high:
+            raise ValueError("Monthly provider ranges are not ordered")
+        month_numbers.append(_month_number(datum.year_month))
+    if require_chronological and any(
+        current <= previous
+        for previous, current in zip(month_numbers, month_numbers[1:], strict=False)
+    ):
+        raise ValueError("Monthly points must be unique and chronological")
+
+
+def _mean_summary_item(datum: MonthlyChartDatum) -> dict[str, str]:
+    period_iso, period_label = format_year_month(datum.year_month)
+    return {
+        "period_iso": period_iso,
+        "period_label": period_label,
+        "mean_machine": decimal_machine(datum.provider_mean),
+        "mean_label": format_provider_krw(datum.provider_mean),
+    }
+
+
 def _svg_number(value: Decimal) -> str:
     rounded = value.quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)
     return format(rounded, "f").rstrip("0").rstrip(".") or "0"
