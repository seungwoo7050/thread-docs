## `feat(public): expose monthly history route`

diff --git a/grocery/history_views.py b/grocery/history_views.py
new file mode 100644
index 0000000..f8f407c
--- /dev/null
+++ b/grocery/history_views.py
@@ -0,0 +1,155 @@
+"""SSR route for monthly KAMIS history."""
+
+from __future__ import annotations
+
+import logging
+import uuid
+from typing import Final, cast
+
+from django.core.exceptions import ObjectDoesNotExist, ValidationError
+from django.db import DatabaseError
+from django.http import Http404, HttpRequest, HttpResponse
+from django.shortcuts import render
+from django.urls import reverse
+from django.views.decorators.http import require_safe
+
+from grocery.forms import HistoryForm
+from grocery.historical_history_read import history_context
+from grocery.historical_public_read import (
+    PublicParameterError,
+    PublicReadIntegrityError,
+    historical_publication_context,
+    historical_series_context,
+    historical_series_for_recent,
+    load_active_historical_publication,
+)
+from grocery.historical_view_helpers import (
+    active_recent_entry,
+    fixed_parameter_error,
+    historical_base_context,
+    historical_server_error,
+    section_navigation,
+    validation_errors,
+    with_publication_headers,
+)
+from grocery.observability import log_event
+
+_LOGGER: Final = logging.getLogger("grocery.audit")
+
+
+@require_safe
+def history(request: HttpRequest, series_id: uuid.UUID) -> HttpResponse:
+    recent = None
+    historical = None
+    try:
+        recent, entry = active_recent_entry(series_id)
+        form = HistoryForm(request.GET if request.GET else None)
+        form_invalid = form.is_bound and not form.is_valid()
+        historical = load_active_historical_publication()
+        context = historical_base_context(entry, series_id, current="history")
+        if historical is None:
+            if form_invalid:
+                return fixed_parameter_error(request, recent, None)
+            context["history_state"] = "unavailable"
+            return with_publication_headers(
+                render(request, "grocery/history.html", context), recent, None
+            )
+        historical_series = historical_series_for_recent(historical, series_id)
+        if historical_series is None:
+            if form_invalid:
+                return fixed_parameter_error(request, recent, historical)
+            context["history_state"] = "unavailable"
+            return with_publication_headers(
+                render(request, "grocery/history.html", context), recent, historical
+            )
+        context.update(
+            {
+                "series": historical_series_context(historical_series),
+                "section_nav": section_navigation(series_id, current="history", historical=True),
+                "historical_publication": historical_publication_context(historical),
+                "history_form_action": reverse("grocery:history", kwargs={"series_id": series_id}),
+            }
+        )
+        if form_invalid:
+            context.update(
+                history_context(
+                    historical,
+                    historical_series,
+                    selected_region_id=None,
+                    selected_range=36,
+                )
+            )
+            context.update(
+                {
+                    "history_state": "validation",
+                    "region_error": "region" in form.errors,
+                    "range_error": "range" in form.errors,
+                    "validation_errors": validation_errors(
+                        form, {"region": "history-region", "range": "history-range"}
+                    ),
+                }
+            )
+            return with_publication_headers(
+                render(request, "grocery/history.html", context, status=400), recent, historical
+            )
+        cleaned = form.cleaned_data if form.is_bound else {}
+        selected_region_id = cleaned.get("region")
+        selected_range = int(cleaned.get("range", "36"))
+        try:
+            context.update(
+                history_context(
+                    historical,
+                    historical_series,
+                    selected_region_id=selected_region_id,
+                    selected_range=selected_range,
+                )
+            )
+        except PublicParameterError:
+            safe_context = history_context(
+                historical, historical_series, selected_region_id=None, selected_range=36
+            )
+            context.update(safe_context)
+            region_options = cast(list[dict[str, object]], safe_context["region_options"])
+            valid_regions = {str(option["value"]) for option in region_options}
+            range_error = selected_region_id is None or (
+                str(selected_region_id) in valid_regions and selected_range == 60
+            )
+            context.update(
+                {
+                    "history_state": "validation",
+                    "range_error": range_error,
+                    "region_error": not range_error,
+                    "validation_errors": [
+                        {
+                            "message": (
+                                "표시 기간을 확인하세요." if range_error else "지역을 확인하세요."
+                            ),
+                            "target": "history-range" if range_error else "history-region",
+                        }
+                    ],
+                }
+            )
+            return with_publication_headers(
+                render(request, "grocery/history.html", context, status=400), recent, historical
+            )
+        context["history_state"] = (
+            "selection_required"
+            if selected_region_id is None
+            else "stale"
+            if historical.stale_message
+            else "ready"
+        )
+        return with_publication_headers(
+            render(request, "grocery/history.html", context), recent, historical
+        )
+    except Http404:
+        raise
+    except DatabaseError, ObjectDoesNotExist, PublicReadIntegrityError, ValidationError:
+        log_event(_LOGGER, "ERROR", "public.history.unavailable")
+        return historical_server_error(
+            request,
+            template="grocery/history.html",
+            state_name="history_state",
+            recent=recent,
+            historical=historical,
+        )


## `feat(frontend): add monthly price history`

diff --git a/grocery/templates/grocery/history.html b/grocery/templates/grocery/history.html
new file mode 100644
index 0000000..390a679
--- /dev/null
+++ b/grocery/templates/grocery/history.html
@@ -0,0 +1,122 @@
+{% extends "grocery/base.html" %}
+
+{% block title %}{{ series.item_name|default:"품목" }} 월별 기록 | 초록장부{% endblock %}
+
+{% block content %}
+  {% include "grocery/_historical_header.html" with page_eyebrow="월별 과거 가격 패턴" page_summary="선택한 지역에서 KAMIS가 제공한 월별 소매 조사 평균과 조사 범위를 확인합니다." %}
+
+  {% if history_state == "loading" or history_state == "unavailable" or history_state == "server_error" %}
+    {% include "grocery/_state_notice.html" with state=history_state retry_url=retry_url %}
+  {% else %}
+    {% if history_state == "validation" %}
+      {% include "grocery/_form_errors.html" with validation_errors=validation_errors %}
+    {% elif history_state == "stale" %}
+      {% include "grocery/_state_notice.html" with state="stale" %}
+    {% endif %}
+
+    {% include "grocery/_publication_summary.html" with publication=historical_publication publication_label="월별 공개 자료 정보" %}
+
+    <section class="scope-controls" aria-labelledby="history-scope-heading">
+      <div class="scope-controls__heading">
+        <h2 id="history-scope-heading">지역과 기간</h2>
+        <p>지역과 품목 조건이 같은 월별 값만 이어서 표시합니다.</p>
+      </div>
+      <form class="scope-form" action="{{ history_form_action|default:'' }}" method="get">
+        <div class="field-group">
+          <label for="history-region">지역</label>
+          <select id="history-region" name="region"{% if region_error %} aria-invalid="true" aria-describedby="validation-title"{% endif %}>
+            {% if not selected_region %}<option value="" selected>지역을 선택하세요</option>{% endif %}
+            {% for option in region_options %}
+              <option value="{{ option.value }}"{% if option.selected %} selected{% endif %}>{{ option.label }}</option>
+            {% endfor %}
+          </select>
+        </div>
+        <div class="field-group">
+          <label for="history-range">표시 기간</label>
+          <select id="history-range" name="range"{% if range_error %} aria-invalid="true" aria-describedby="validation-title"{% endif %}>
+            {% for option in range_options %}
+              <option value="{{ option.value }}"{% if option.selected %} selected{% endif %}>{{ option.label }}</option>
+            {% endfor %}
+          </select>
+        </div>
+        <button class="button button--primary" type="submit">월별 기록 보기</button>
+      </form>
+    </section>
+
+    {% if history_state == "validation" %}
+    {% elif history_state == "selection_required" %}
+      {% include "grocery/_state_notice.html" with state="selection_required" %}
+    {% elif monthly_points %}
+      <section class="history-section" aria-labelledby="history-heading">
+        <div class="section-heading section-heading--stacked">
+          <p class="eyebrow">{{ selected_region.label }} · {{ selected_range.label }}</p>
+          <h2 id="history-heading">월별 조사값</h2>
+          <p>선은 월 조사 평균, 옅은 범위는 같은 달의 최저·최고 조사값입니다.</p>
+        </div>
+
+        {% if history_chart %}
+          <figure class="history-chart">
+            <svg
+              class="history-chart__svg"
+              viewBox="{{ history_chart.view_box }}"
+              aria-hidden="true"
+              focusable="false"
+            >
+              {% for tick in history_chart.y_ticks %}
+                <line class="history-chart__guide" x1="{{ tick.x1 }}" y1="{{ tick.y }}" x2="{{ tick.x2 }}" y2="{{ tick.y }}"/>
+                <text class="history-chart__label" x="{{ tick.label_x }}" y="{{ tick.label_y }}">{{ tick.label }}</text>
+              {% endfor %}
+              {% for segment in history_chart.range_segments %}
+                <polygon class="history-chart__range" points="{{ segment.points }}"/>
+              {% endfor %}
+              {% for segment in history_chart.mean_segments %}
+                <polyline class="history-chart__mean" points="{{ segment.points }}"/>
+              {% endfor %}
+              {% for point in history_chart.points %}
+                <circle class="history-chart__point" cx="{{ point.x }}" cy="{{ point.y }}" r="3"/>
+              {% endfor %}
+              {% for tick in history_chart.x_ticks %}
+                <text class="history-chart__label history-chart__label--x" x="{{ tick.x }}" y="{{ tick.y }}">{{ tick.label }}</text>
+              {% endfor %}
+            </svg>
+            <figcaption id="history-chart-caption">
+              {{ selected_region.label }}의 {{ selected_range.label }} 월별 KAMIS 소매 조사값
+            </figcaption>
+            <ul class="chart-key" aria-hidden="true">
+              <li><span class="chart-key__line"></span>월 조사 평균</li>
+              <li><span class="chart-key__range"></span>월 최저·최고 조사 범위</li>
+            </ul>
+          </figure>
+        {% endif %}
+
+        <div class="month-ledger">
+          <div class="month-ledger__head" aria-hidden="true">
+            <span>연월</span><span>소매 조사 평균</span><span>월 최저 조사값</span><span>월 최고 조사값</span>
+          </div>
+          <ol class="month-list">
+            {% for point in monthly_points %}
+              <li class="month-row{% if not point.available %} month-row--unavailable{% endif %}">
+                <p class="month-row__date">
+                  {% if point.period_iso %}<time datetime="{{ point.period_iso }}">{% endif %}{{ point.period_label }}{% if point.period_iso %}</time>{% endif %}
+                </p>
+                {% if point.available %}
+                  <dl class="month-row__facts">
+                    <div><dt>소매 조사 평균</dt><dd><data value="{{ point.mean_machine }}">{{ point.mean_label }}</data></dd></div>
+                    <div><dt>월 최저 조사값</dt><dd><data value="{{ point.minimum_machine }}">{{ point.minimum_label }}</data></dd></div>
+                    <div><dt>월 최고 조사값</dt><dd><data value="{{ point.maximum_machine }}">{{ point.maximum_label }}</data></dd></div>
+                  </dl>
+                {% else %}
+                  <p class="month-row__missing">{{ point.unavailable_label }}</p>
+                {% endif %}
+              </li>
+            {% endfor %}
+          </ol>
+        </div>
+      </section>
+    {% else %}
+      {% include "grocery/_state_notice.html" with state="server_error" retry_url=retry_url %}
+    {% endif %}
+  {% endif %}
+{% endblock %}
+
+{% block footer_note %}표시값은 KAMIS가 제공한 월별 소매 조사 평균·최저·최고입니다. 개별 판매처의 실제 판매 금액과 다를 수 있습니다.{% endblock %}


