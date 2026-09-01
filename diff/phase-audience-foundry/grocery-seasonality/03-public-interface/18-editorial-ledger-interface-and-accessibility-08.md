## `fix(frontend): polish responsive and state surfaces`

diff --git a/grocery/static/grocery/app.css b/grocery/static/grocery/app.css
index 72486b5..0de68a9 100644
--- a/grocery/static/grocery/app.css
+++ b/grocery/static/grocery/app.css
@@ -29,6 +29,7 @@
   --color-lower: #1c5d75;
   --color-higher: #1c5d75;
   --color-focus: #075d92;
+  --color-focus-on-dark: #8ecfe4;
   --color-on-brand: #fffdf6;
   --color-harvest: #a93426;
   --color-forest: #123f2d;
@@ -108,6 +109,10 @@ a:hover {
   outline-offset: 3px;
 }
 
+.site-header :focus-visible {
+  outline-color: var(--color-focus-on-dark);
+}
+
 h1,
 h2,
 h3,
@@ -119,7 +124,6 @@ dd {
 }
 
 h1,
-h2,
 .brand__name {
   font-family: "Gowun Batang", serif;
   font-weight: 700;
@@ -127,6 +131,7 @@ h2,
   text-wrap: balance;
 }
 
+h2,
 h3,
 h4 {
   font-family: inherit;
@@ -144,8 +149,25 @@ h1 {
 
 h2 {
   margin-bottom: var(--space-4);
+  font-size: clamp(1.3rem, 3vw, 1.65rem);
+  letter-spacing: -0.025em;
+}
+
+.catalog-results__heading h2,
+.history-section > .section-heading h2,
+.region-section__intro h2,
+.market-section > .section-heading h2,
+.selection-section__heading h2 {
+  font-family: "Gowun Batang", serif;
   font-size: clamp(1.55rem, 4.2vw, 2.35rem);
+  font-weight: 700;
   letter-spacing: -0.04em;
+  line-height: 1.18;
+}
+
+.error-page h1 {
+  font-family: inherit;
+  font-weight: 850;
 }
 
 h3 {
@@ -1405,10 +1427,6 @@ select[aria-invalid="true"] {
   font-size: 15px;
 }
 
-.history-chart__label--x {
-  text-anchor: middle;
-}
-
 .history-chart figcaption {
   margin-top: var(--space-2);
   color: var(--color-muted);
@@ -2110,6 +2128,12 @@ select[aria-invalid="true"] {
     background: var(--color-brand-soft);
   }
 
+  .segment--selected:hover,
+  .site-actions a:hover {
+    background: var(--color-harvest);
+    color: var(--color-on-brand);
+  }
+
   .region-row__action a:hover {
     background: var(--color-brand-soft);
   }
@@ -2473,13 +2497,10 @@ select[aria-invalid="true"] {
     padding: var(--space-3);
   }
 
-  .ledger-entry__heading,
-  .ledger-entry__actions {
-    grid-column: auto;
-  }
-
   .ledger-entry__heading {
     display: block;
+    grid-row: 1;
+    grid-column: 1;
   }
 
   .ledger-fact {
@@ -2487,11 +2508,19 @@ select[aria-invalid="true"] {
     border-top: 0;
   }
 
-  .ledger-fact--price,
-  .ledger-fact--comparison,
+  .ledger-fact--price {
+    grid-row: 1;
+    grid-column: 2;
+  }
+
+  .ledger-fact--comparison {
+    grid-row: 1;
+    grid-column: 3;
+  }
+
   .ledger-fact--date {
-    grid-row: auto;
-    grid-column: auto;
+    grid-row: 1;
+    grid-column: 4;
   }
 
   .ledger-fact dt {
@@ -2507,7 +2536,8 @@ select[aria-invalid="true"] {
   }
 
   .ledger-entry__actions {
-    grid-row: auto;
+    grid-row: 1;
+    grid-column: 5;
     border-top: 0;
   }
 
diff --git a/grocery/templates/grocery/_historical_header.html b/grocery/templates/grocery/_historical_header.html
index 6dfb10c..6842b49 100644
--- a/grocery/templates/grocery/_historical_header.html
+++ b/grocery/templates/grocery/_historical_header.html
@@ -1,16 +1,19 @@
+{% firstof back_url catalog_url home_url "/" as recovery_url %}
 <nav class="breadcrumb" aria-label="이전 화면으로 돌아가기">
-  <a href="{{ back_url|default:catalog_url }}">← {{ back_label|default:"채소·과일 소매 조사값" }}</a>
+  <a href="{{ recovery_url }}">← {{ back_label|default:"채소·과일 소매 조사값" }}</a>
 </nav>
 
 <header class="record-heading">
   <p class="eyebrow">{{ page_eyebrow }}</p>
-  <h1>{{ series.item_name }}</h1>
+  <h1>{% firstof series.item_name page_eyebrow "조사 자료" %}</h1>
   <p class="record-heading__summary">{{ page_summary }}</p>
-  <dl class="detail-signature" aria-label="품목 조건">
-    <div><dt>품종</dt><dd>{{ series.variety_name }}</dd></div>
-    <div><dt>등급</dt><dd>{{ series.grade_name }}</dd></div>
-    <div><dt>판매 단위</dt><dd>{{ series.unit_label }}</dd></div>
-  </dl>
+  {% if series.variety_name or series.grade_name or series.unit_label %}
+    <dl class="detail-signature" aria-label="품목 조건">
+      {% if series.variety_name %}<div><dt>품종</dt><dd>{{ series.variety_name }}</dd></div>{% endif %}
+      {% if series.grade_name %}<div><dt>등급</dt><dd>{{ series.grade_name }}</dd></div>{% endif %}
+      {% if series.unit_label %}<div><dt>판매 단위</dt><dd>{{ series.unit_label }}</dd></div>{% endif %}
+    </dl>
+  {% endif %}
 </header>
 
 {% include "grocery/_series_nav.html" with section_nav=section_nav series=series %}
diff --git a/grocery/templates/grocery/_series_nav.html b/grocery/templates/grocery/_series_nav.html
index 977a6b4..4c17e29 100644
--- a/grocery/templates/grocery/_series_nav.html
+++ b/grocery/templates/grocery/_series_nav.html
@@ -6,7 +6,9 @@
           {% if item.current %}
             <span class="series-nav__item series-nav__item--current" aria-current="page">{{ item.label }}</span>
           {% elif item.available and item.url %}
-            <a class="series-nav__item" href="{{ item.url }}">{{ item.label }}</a>
+            <a class="series-nav__item" href="{{ item.url }}">
+              {{ item.label }}{% if market_continuation and item.label == "지역별 조사값" %} · 지역 선택 후 시장별 조사값{% endif %}
+            </a>
           {% endif %}
         </li>
       {% endfor %}
diff --git a/grocery/templates/grocery/detail.html b/grocery/templates/grocery/detail.html
index 6fda882..adcaaf7 100644
--- a/grocery/templates/grocery/detail.html
+++ b/grocery/templates/grocery/detail.html
@@ -48,13 +48,13 @@
 
   <div class="detail-pathways">
     {% if section_nav %}
-      {% include "grocery/_series_nav.html" with section_nav=section_nav series=series %}
+      {% include "grocery/_series_nav.html" with section_nav=section_nav series=series market_continuation=historical_links.regions_url %}
     {% elif historical_links %}
       <nav class="series-nav" aria-label="{{ series.item_name }} 자료 보기">
         <ul>
           <li><span class="series-nav__item series-nav__item--current" aria-current="page">최근 조사값</span></li>
           {% if historical_links.history_url %}<li><a class="series-nav__item" href="{{ historical_links.history_url }}">월별 기록</a></li>{% endif %}
-          {% if historical_links.regions_url %}<li><a class="series-nav__item" href="{{ historical_links.regions_url }}">지역별 조사값</a></li>{% endif %}
+          {% if historical_links.regions_url %}<li><a class="series-nav__item" href="{{ historical_links.regions_url }}">지역별 조사값 · 지역 선택 후 시장별 조사값</a></li>{% endif %}
         </ul>
       </nav>
     {% endif %}
diff --git a/grocery/templates/grocery/history.html b/grocery/templates/grocery/history.html
index 01e2523..512d77f 100644
--- a/grocery/templates/grocery/history.html
+++ b/grocery/templates/grocery/history.html
@@ -114,7 +114,7 @@
                 <circle class="history-chart__point" cx="{{ point.x }}" cy="{{ point.y }}" r="3"/>
               {% endfor %}
               {% for tick in history_chart.x_ticks %}
-                <text class="history-chart__label history-chart__label--x" x="{{ tick.x }}" y="{{ tick.y }}">{{ tick.label }}</text>
+                <text class="history-chart__label history-chart__label--x" x="{{ tick.x }}" y="{{ tick.y }}" text-anchor="{{ tick.anchor }}">{{ tick.label }}</text>
               {% endfor %}
             </svg>
             <figcaption id="history-chart-caption">
@@ -128,7 +128,7 @@
         {% endif %}
 
         {% if history_year_groups %}
-          <div class="history-year-groups" aria-label="연도별 월 조사 기록">
+          <div class="history-year-groups">
             {% for group in history_year_groups %}
               <details class="history-year{% if group.is_latest %} history-year--latest{% endif %}"{% if group.open %} open{% endif %}>
                 <summary>
diff --git a/grocery/templates/grocery/markets.html b/grocery/templates/grocery/markets.html
index 2152cbf..7a2ccee 100644
--- a/grocery/templates/grocery/markets.html
+++ b/grocery/templates/grocery/markets.html
@@ -3,7 +3,11 @@
 {% block title %}{{ series.item_name|default:"품목" }} 시장별 조사값 | 초록장부{% endblock %}
 
 {% block content %}
-  {% include "grocery/_historical_header.html" with page_eyebrow="시장별 소매 조사값" page_summary="선택한 지역과 조사일에 KAMIS가 기록한 시장별 소매 조사값을 확인합니다." back_url=regions_url back_label="지역별 조사값" %}
+  {% if regions_url %}
+    {% include "grocery/_historical_header.html" with page_eyebrow="시장별 소매 조사값" page_summary="선택한 지역과 조사일에 KAMIS가 기록한 시장별 소매 조사값을 확인합니다." back_url=regions_url back_label="지역별 조사값" %}
+  {% else %}
+    {% include "grocery/_historical_header.html" with page_eyebrow="시장별 소매 조사값" page_summary="선택한 지역과 조사일에 KAMIS가 기록한 시장별 소매 조사값을 확인합니다." back_label="채소·과일 소매 조사값" %}
+  {% endif %}
 
   {% if markets_state == "loading" or markets_state == "unavailable" or markets_state == "server_error" %}
     {% include "grocery/_state_notice.html" with state=markets_state retry_url=retry_url %}
diff --git a/grocery/tests/test_accessibility_contrast.py b/grocery/tests/test_accessibility_contrast.py
index 3017d3f..d842aee 100644
--- a/grocery/tests/test_accessibility_contrast.py
+++ b/grocery/tests/test_accessibility_contrast.py
@@ -58,6 +58,7 @@ def test_focus_and_interactive_boundaries_meet_non_text_contrast() -> None:
         (colors["color-focus"], colors["color-canvas"]),
         (colors["color-focus"], colors["color-surface"]),
         (colors["color-focus"], colors["color-brand-soft"]),
+        (colors["color-focus-on-dark"], colors["color-brand-strong"]),
         (colors["color-border-strong"], colors["color-canvas"]),
         (colors["color-border-strong"], colors["color-surface"]),
     )
@@ -65,6 +66,13 @@ def test_focus_and_interactive_boundaries_meet_non_text_contrast() -> None:
     assert min(_contrast(foreground, background) for foreground, background in pairs) >= 3
 
 
+def test_selected_segment_and_header_link_hover_text_remains_legible() -> None:
+    colors = _colors()
+
+    # Both selected segments and the masthead selection link use this hover pair.
+    assert _contrast(colors["color-on-brand"], colors["color-brand"]) >= 4.5
+
+
 def test_price_direction_tokens_use_one_neutral_data_color() -> None:
     colors = _colors()
 
diff --git a/grocery/tests/test_public_templates.py b/grocery/tests/test_public_templates.py
index bfb96f5..5f850a6 100644
--- a/grocery/tests/test_public_templates.py
+++ b/grocery/tests/test_public_templates.py
@@ -307,6 +307,34 @@ def test_detail_direction_is_not_conveyed_by_symbol_color_or_chart_alone() -> No
     assert 'aria-hidden="true"' in html
 
 
+def test_detail_navigation_explains_the_market_path_without_inventing_a_route() -> None:
+    html = render(
+        "grocery/detail.html",
+        detail_context(
+            historical_links={"regions_url": "/series/1/regions/"},
+            section_nav=[
+                {
+                    "label": "최근 조사값",
+                    "url": "/series/1/",
+                    "current": True,
+                    "available": True,
+                },
+                {
+                    "label": "지역별 조사값",
+                    "url": "/series/1/regions/",
+                    "current": False,
+                    "available": True,
+                },
+            ],
+        ),
+    )
+
+    compact_html = " ".join(html.split())
+    assert 'href="/series/1/regions/"' in html
+    assert "지역별 조사값 · 지역 선택 후 시장별 조사값" in compact_html
+    assert "/markets/" not in html
+
+
 @pytest.mark.parametrize(
     ("template_name", "heading"),
     [
@@ -380,5 +408,5 @@ def test_styles_define_ledger_tokens_responsive_interaction_and_user_preferences
     assert "linear-gradient(" not in css
     assert "radial-gradient(" not in css
     assert "box-shadow" not in css
-    assert "--color-lower: #245b73" in css
-    assert "--color-higher: #245b73" in css
+    assert "--color-lower: #1c5d75" in css
+    assert "--color-higher: #1c5d75" in css
diff --git a/grocery/tests/test_vnext_public_read_contract.py b/grocery/tests/test_vnext_public_read_contract.py
index 8fab8d1..0a4649b 100644
--- a/grocery/tests/test_vnext_public_read_contract.py
+++ b/grocery/tests/test_vnext_public_read_contract.py
@@ -73,6 +73,7 @@ def test_monthly_chart_never_connects_across_a_missing_month() -> None:
     mean_segments = cast(list[dict[str, str]], chart["mean_segments"])
     range_segments = cast(list[dict[str, str]], chart["range_segments"])
     points = cast(list[dict[str, str]], chart["points"])
+    x_ticks = cast(list[dict[str, str]], chart["x_ticks"])
 
     assert len(mean_segments) == 1
     assert len(range_segments) == 1
@@ -81,6 +82,7 @@ def test_monthly_chart_never_connects_across_a_missing_month() -> None:
     assert isolated_x not in mean_segments[0]["points"]
     assert isolated_x not in range_segments[0]["points"]
     assert chart["gap_markers"] == [{"x": "490.67", "label": "2026.03"}]
+    assert [tick["anchor"] for tick in x_ticks] == ["start", "middle", "end"]
     assert chart["points"] == [
         {"x": "64", "y": "195.2"},
         {"x": "277.33", "y": "150.4"},
diff --git a/grocery/tests/test_vnext_public_templates.py b/grocery/tests/test_vnext_public_templates.py
index b8c7474..d657733 100644
--- a/grocery/tests/test_vnext_public_templates.py
+++ b/grocery/tests/test_vnext_public_templates.py
@@ -142,6 +142,7 @@ def test_history_leads_with_summary_and_opens_only_the_latest_year() -> None:
     assert html.index("2026년") < html.index("2025년")
     assert html.count("<details") == 2
     assert html.count(" open>") == 1
+    assert 'class="history-year-groups" aria-label=' not in html
 
 
 def test_region_and_market_pages_keep_provider_facts_distinct() -> None:
@@ -215,6 +216,29 @@ def test_historical_blocking_states_hide_controls_and_fact_ledgers() -> None:
         assert "-ledger" not in html
 
 
+def test_historical_server_errors_without_series_keep_truthful_recovery() -> None:
+    cases = (
+        ("grocery/history.html", "history_state", "월별 조사 기록"),
+        ("grocery/regions.html", "regions_state", "지역별 소매 조사값"),
+        ("grocery/markets.html", "markets_state", "시장별 소매 조사값"),
+    )
+    rendered: dict[str, str] = {}
+
+    for template, state_key, heading in cases:
+        html = render_to_string(
+            template,
+            {"home_url": "/", state_key: "server_error", "retry_url": "/safe-retry/"},
+        )
+        rendered[template] = html
+        assert html.count("<h1>") == 1
+        assert f"<h1>{heading}</h1>" in html
+        assert "조사 자료를 불러오지 못했습니다" in html
+
+    markets_html = rendered["grocery/markets.html"]
+    assert '<a href="/">← 채소·과일 소매 조사값</a>' in markets_html
+    assert "← 지역별 조사값" not in markets_html
+
+
 def test_catalog_validation_reveals_and_associates_advanced_controls() -> None:
     context = catalog_context(
         catalog_state="validation",
diff --git a/grocery/vnext_presentation.py b/grocery/vnext_presentation.py
index ac5a93a..e5ce5d1 100644
--- a/grocery/vnext_presentation.py
+++ b/grocery/vnext_presentation.py
@@ -266,6 +266,13 @@ def build_history_chart(data: Sequence[MonthlyChartDatum]) -> dict[str, object]:
             "x": _svg_number(x_position(month_number)),
             "y": "268",
             "label": _month_label(month_number),
+            "anchor": (
+                "start"
+                if month_number == first_month
+                else "end"
+                if month_number == last_month
+                else "middle"
+            ),
         }
         for month_number in tick_months
     ]


