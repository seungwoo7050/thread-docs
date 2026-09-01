## `fix(frontend): compose selection publication states`

diff --git a/grocery/templates/grocery/selection.html b/grocery/templates/grocery/selection.html
index 2f58780..aed4c39 100644
--- a/grocery/templates/grocery/selection.html
+++ b/grocery/templates/grocery/selection.html
@@ -23,6 +23,10 @@
       {% include "grocery/_form_errors.html" with validation_errors=validation_errors %}
       <a class="button button--primary" href="{{ catalog_url|default:'/' }}">공개 목록에서 다시 선택하기</a>
     {% else %}
+      {% if selection_is_stale %}
+        {% include "grocery/_state_notice.html" with state="stale" %}
+      {% endif %}
+
       {% if selection_state == "partial" %}
         {% include "grocery/_state_notice.html" with state="partial" excluded_count=excluded_count %}
       {% endif %}
diff --git a/grocery/tests/test_vnext_selection_template.py b/grocery/tests/test_vnext_selection_template.py
index 0758483..bdd6338 100644
--- a/grocery/tests/test_vnext_selection_template.py
+++ b/grocery/tests/test_vnext_selection_template.py
@@ -89,3 +89,22 @@ def test_selection_add_form_has_limit_and_candidate_empty_states() -> None:
     assert "현재 목록에 더 추가할 수 있는 공개 품목이 없습니다." in empty
     assert 'class="selection-add__form"' not in limit
     assert 'class="selection-add__form"' not in empty
+
+
+def test_selection_renders_stale_publication_and_partial_exclusion_together() -> None:
+    html = render_to_string(
+        "grocery/selection.html",
+        {
+            "selection_state": "partial",
+            "selection_is_stale": True,
+            "excluded_count": 2,
+            "items": [],
+            "publication": {**PUBLICATION, "freshness_state": "stale"},
+            "selection_limit_reached": False,
+            "can_add_selection": False,
+        },
+    )
+
+    assert "마지막 공개 자료를 표시합니다" in html
+    assert "일부 품목을 제외했습니다" in html
+    assert "현재 공개 목록에 없는 품목 2개는 표시하지 않았습니다." in html


## `test(frontend): cover cumulative selection controls`

diff --git a/grocery/tests/test_vnext_selection_template.py b/grocery/tests/test_vnext_selection_template.py
index 3eaa44b..0758483 100644
--- a/grocery/tests/test_vnext_selection_template.py
+++ b/grocery/tests/test_vnext_selection_template.py
@@ -7,6 +7,7 @@ from grocery.tests.test_vnext_public_templates import PUBLICATION, SERIES
 def test_selection_renders_url_order_partial_and_empty_recovery() -> None:
     item = {
         **SERIES,
+        "series_value": "11111111-1111-1111-1111-111111111111",
         "detail_url": "/series/1/",
         "remove_url": "/selection/",
         "current_price_machine": "8000",
@@ -27,3 +28,64 @@ def test_selection_renders_url_order_partial_and_empty_recovery() -> None:
     assert "일부 품목을 제외했습니다" in partial
     assert "1주 전 제공값보다 2,000원 낮음 (-20.0%)" in " ".join(partial.split())
     assert "선택한 품목이 없습니다" in empty and "품목 둘러보기" in empty
+
+
+def test_selection_add_form_appends_to_canonical_url_order() -> None:
+    items = [
+        {
+            **SERIES,
+            "item_name": name,
+            "series_value": value,
+            "detail_url": f"/series/{value}/",
+            "remove_url": "/selection/",
+            "current_price_label": "8,000원",
+            "source_date_iso": "2026-08-29",
+            "source_date_label": "2026년 8월 29일",
+            "comparison": comparison("1주 전 제공값"),
+        }
+        for name, value in (
+            ("배추", "11111111-1111-1111-1111-111111111111"),
+            ("사과", "22222222-2222-2222-2222-222222222222"),
+        )
+    ]
+    html = render_to_string(
+        "grocery/selection.html",
+        {
+            "selection_state": "ready",
+            "items": items,
+            "publication": PUBLICATION,
+            "selection_form_action": "/selection/",
+            "can_add_selection": True,
+            "selection_candidates": [
+                {
+                    "value": "33333333-3333-3333-3333-333333333333",
+                    "label": "양파 · 일반 · 상품 · 1kg",
+                }
+            ],
+        },
+    )
+
+    first = html.index('name="series" value="11111111-1111-1111-1111-111111111111"')
+    second = html.index('name="series" value="22222222-2222-2222-2222-222222222222"')
+    candidate = html.index('id="selection-add-item"')
+    assert first < second < candidate
+    assert 'class="selection-add__form" action="/selection/" method="get"' in html
+    assert 'name="series" required aria-describedby="selection-add-hint"' in html
+    assert "양파 · 일반 · 상품 · 1kg" in html
+
+
+def test_selection_add_form_has_limit_and_candidate_empty_states() -> None:
+    base = {"selection_state": "ready", "items": [], "publication": PUBLICATION}
+    limit = render_to_string(
+        "grocery/selection.html",
+        {**base, "selection_limit_reached": True, "can_add_selection": False},
+    )
+    empty = render_to_string(
+        "grocery/selection.html",
+        {**base, "selection_limit_reached": False, "can_add_selection": False},
+    )
+
+    assert "다섯 품목을 모두 선택했습니다." in limit
+    assert "현재 목록에 더 추가할 수 있는 공개 품목이 없습니다." in empty
+    assert 'class="selection-add__form"' not in limit
+    assert 'class="selection-add__form"' not in empty


## `test(public): lock URL and geometry contracts`

diff --git a/grocery/tests/test_vnext_public_read_contract.py b/grocery/tests/test_vnext_public_read_contract.py
new file mode 100644
index 0000000..bac8dc5
--- /dev/null
+++ b/grocery/tests/test_vnext_public_read_contract.py
@@ -0,0 +1,91 @@
+from decimal import Decimal
+
+import pytest
+from django.core.exceptions import ValidationError
+from django.http import QueryDict
+
+from grocery.forms import CatalogForm, HistoryForm, MarketsForm, RegionsForm, parse_selection_query
+from grocery.security import SECURITY_HEADERS
+from grocery.vnext_presentation import MonthlyChartDatum, build_history_chart, range_meter
+
+
+@pytest.mark.parametrize(
+    "query_string",
+    [
+        "unknown=private-marker",
+        "period=week&period=month",
+        "q=private-marker&page=2",
+        "page=01",
+    ],
+)
+def test_catalog_query_rejects_noncanonical_state_without_reflecting_values(
+    query_string: str,
+) -> None:
+    marker = "private-marker"
+    form = CatalogForm(QueryDict(query_string))
+
+    assert not form.is_valid()
+    assert marker not in str(form.errors)
+
+
+@pytest.mark.parametrize(
+    ("form_type", "query_string"),
+    [
+        (HistoryForm, "region=AAAAAAAA-AAAA-4AAA-8AAA-AAAAAAAAAAAA"),
+        (HistoryForm, "region=aaaaaaaaaaaa4aaa8aaaaaaaaaaaaaaa"),
+        (RegionsForm, "date=2026-8-01"),
+        (RegionsForm, "date=2026-02-30"),
+        (MarketsForm, "date=2026-08-01&page=001"),
+    ],
+)
+def test_historical_forms_require_canonical_uuid_date_and_page(
+    form_type: type, query_string: str
+) -> None:
+    assert not form_type(QueryDict(query_string)).is_valid()
+
+
+def test_selection_preserves_first_seen_order_and_rejects_pre_deduplication_overflow() -> None:
+    first = "11111111-1111-4111-8111-111111111111"
+    second = "22222222-2222-4222-8222-222222222222"
+    parsed = parse_selection_query(QueryDict(f"series={first}&series={second}&series={first}"))
+
+    assert tuple(map(str, parsed.series_ids)) == (first, second)
+    with pytest.raises(ValidationError):
+        parse_selection_query(QueryDict("&".join(f"series={first}" for _ in range(6))))
+
+
+def test_monthly_chart_never_connects_across_a_missing_month() -> None:
+    data = [
+        MonthlyChartDatum("202601", Decimal("100"), Decimal("90"), Decimal("110")),
+        MonthlyChartDatum("202602", Decimal("110"), Decimal("100"), Decimal("120")),
+        MonthlyChartDatum("202604", Decimal("130"), Decimal("120"), Decimal("140")),
+    ]
+
+    chart = build_history_chart(data)
+
+    assert len(chart["mean_segments"]) == 1
+    assert len(chart["range_segments"]) == 1
+    assert len(chart["points"]) == 3
+    isolated_x = chart["points"][2]["x"]
+    assert isolated_x not in chart["mean_segments"][0]["points"]
+    assert isolated_x not in chart["range_segments"][0]["points"]
+    assert chart["gap_markers"] == [{"x": "490.67", "label": "2026.03"}]
+    assert chart["points"] == [
+        {"x": "64", "y": "195.2"},
+        {"x": "277.33", "y": "150.4"},
+        {"x": "704", "y": "60.8"},
+    ]
+
+
+def test_regional_meter_uses_one_server_side_decimal_scale() -> None:
+    assert range_meter(
+        minimum=Decimal("900"),
+        mean=Decimal("1000"),
+        maximum=Decimal("1100"),
+        scale_minimum=Decimal("800"),
+        scale_maximum=Decimal("1200"),
+    ) == {"minimum_x": "25", "mean_x": "50", "maximum_x": "75"}
+
+
+def test_public_referrer_policy_sends_no_query_state() -> None:
+    assert SECURITY_HEADERS["Referrer-Policy"] == "no-referrer"
