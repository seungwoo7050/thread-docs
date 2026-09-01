## `feat(public): expose regional and market routes`

diff --git a/grocery/daily_views.py b/grocery/daily_views.py
new file mode 100644
index 0000000..b73027a
--- /dev/null
+++ b/grocery/daily_views.py
@@ -0,0 +1,305 @@
+"""SSR routes for regional ranges and market observations."""
+
+from __future__ import annotations
+
+import logging
+import uuid
+from typing import Any, Final, cast
+from urllib.parse import urlencode
+
+from django.core.exceptions import ObjectDoesNotExist, ValidationError
+from django.db import DatabaseError
+from django.http import Http404, HttpRequest, HttpResponse
+from django.shortcuts import render
+from django.urls import reverse
+from django.views.decorators.http import require_safe
+
+from grocery.forms import MarketsForm, RegionsForm
+from grocery.historical_daily_read import markets_context, regions_context
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
+    market_recovery_context,
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
+def regions(request: HttpRequest, series_id: uuid.UUID) -> HttpResponse:
+    recent = None
+    historical = None
+    try:
+        recent, entry = active_recent_entry(series_id)
+        form = RegionsForm(request.GET if request.GET else None)
+        form_invalid = form.is_bound and not form.is_valid()
+        historical = load_active_historical_publication()
+        context = historical_base_context(entry, series_id, current="regions")
+        if historical is None:
+            if form_invalid:
+                return fixed_parameter_error(request, recent, None)
+            context["regions_state"] = "unavailable"
+            return with_publication_headers(
+                render(request, "grocery/regions.html", context), recent, None
+            )
+        historical_series = historical_series_for_recent(historical, series_id)
+        if historical_series is None:
+            if form_invalid:
+                return fixed_parameter_error(request, recent, historical)
+            context["regions_state"] = "unavailable"
+            return with_publication_headers(
+                render(request, "grocery/regions.html", context), recent, historical
+            )
+        context.update(
+            {
+                "series": historical_series_context(historical_series),
+                "section_nav": section_navigation(series_id, current="regions", historical=True),
+                "historical_publication": historical_publication_context(historical),
+                "regions_form_action": reverse("grocery:regions", kwargs={"series_id": series_id}),
+            }
+        )
+        if form_invalid:
+            context.update(regions_context(historical, historical_series, selected_date=None))
+            context.update(
+                {
+                    "regions_state": "validation",
+                    "date_error": "date" in form.errors,
+                    "validation_errors": validation_errors(form, {"date": "regions-date"}),
+                }
+            )
+            return with_publication_headers(
+                render(request, "grocery/regions.html", context, status=400), recent, historical
+            )
+        selected_date = form.cleaned_data.get("date") if form.is_bound else None
+        try:
+            context.update(
+                regions_context(historical, historical_series, selected_date=selected_date)
+            )
+        except PublicParameterError:
+            context.update(regions_context(historical, historical_series, selected_date=None))
+            context.update(
+                {
+                    "regions_state": "validation",
+                    "date_error": True,
+                    "validation_errors": [
+                        {"message": "조사일을 확인하세요.", "target": "regions-date"}
+                    ],
+                }
+            )
+            return with_publication_headers(
+                render(request, "grocery/regions.html", context, status=400), recent, historical
+            )
+        regional_rows = cast(list[dict[str, Any]], context["regional_rows"])
+        selected_date_context = cast(dict[str, str], context["selected_date"])
+        for row in regional_rows:
+            row["markets_url"] = (
+                _url(
+                    reverse(
+                        "grocery:markets",
+                        kwargs={"series_id": series_id, "region_id": row["region_id"]},
+                    ),
+                    {"date": selected_date_context["iso"]},
+                )
+                if row.pop("market_available")
+                else ""
+            )
+        context["regions_state"] = "stale" if historical.stale_message else "ready"
+        return with_publication_headers(
+            render(request, "grocery/regions.html", context), recent, historical
+        )
+    except Http404:
+        raise
+    except DatabaseError, ObjectDoesNotExist, PublicReadIntegrityError, ValidationError:
+        log_event(_LOGGER, "ERROR", "public.regions.unavailable")
+        return historical_server_error(
+            request,
+            template="grocery/regions.html",
+            state_name="regions_state",
+            recent=recent,
+            historical=historical,
+        )
+
+
+@require_safe
+def markets(request: HttpRequest, series_id: uuid.UUID, region_id: uuid.UUID) -> HttpResponse:
+    recent = None
+    historical = None
+    try:
+        recent, entry = active_recent_entry(series_id)
+        form = MarketsForm(request.GET if request.GET else None)
+        form_invalid = form.is_bound and not form.is_valid()
+        historical = load_active_historical_publication()
+        context = historical_base_context(entry, series_id, current="regions")
+        context["regions_url"] = reverse("grocery:regions", kwargs={"series_id": series_id})
+        if historical is None:
+            if form_invalid:
+                return fixed_parameter_error(request, recent, None)
+            context["markets_state"] = "unavailable"
+            return with_publication_headers(
+                render(request, "grocery/markets.html", context), recent, None
+            )
+        historical_series = historical_series_for_recent(historical, series_id)
+        if historical_series is None:
+            if form_invalid:
+                return fixed_parameter_error(request, recent, historical)
+            context["markets_state"] = "unavailable"
+            return with_publication_headers(
+                render(request, "grocery/markets.html", context), recent, historical
+            )
+        context.update(
+            {
+                "series": historical_series_context(historical_series),
+                "section_nav": section_navigation(series_id, current="regions", historical=True),
+                "historical_publication": historical_publication_context(historical),
+            }
+        )
+        if form_invalid:
+            safe_context, recovery_region_id, _region_available = _market_recovery(
+                historical, historical_series, region_id
+            )
+            context.update(safe_context)
+            context["markets_form_action"] = _market_url(series_id, recovery_region_id)
+            context.update(
+                {
+                    "markets_state": "validation",
+                    "date_error": "date" in form.errors,
+                    "validation_errors": validation_errors(form, {"date": "markets-date"}),
+                }
+            )
+            return with_publication_headers(
+                render(request, "grocery/markets.html", context, status=400), recent, historical
+            )
+        cleaned = form.cleaned_data if form.is_bound else {}
+        selected_date = cleaned.get("date")
+        page = int(cleaned.get("page", 1))
+        try:
+            context.update(
+                markets_context(
+                    historical,
+                    historical_series,
+                    region_id=region_id,
+                    selected_date=selected_date,
+                    page=page,
+                )
+            )
+            context["markets_form_action"] = _market_url(series_id, region_id)
+        except PublicParameterError:
+            safe_context, recovery_region_id, region_available = _market_recovery(
+                historical, historical_series, region_id
+            )
+            context.update(safe_context)
+            context["markets_form_action"] = _market_url(series_id, recovery_region_id)
+            date_options = cast(list[dict[str, object]], safe_context["date_options"])
+            available_dates = {str(option["value"]) for option in date_options}
+            date_error = (
+                region_available
+                and selected_date is not None
+                and selected_date.isoformat() not in available_dates
+            )
+            context.update(
+                {
+                    "markets_state": "validation",
+                    "date_error": date_error,
+                    "validation_errors": [
+                        {
+                            "message": (
+                                "조사일을 확인하세요."
+                                if date_error
+                                else "지역·조사일·페이지를 확인하세요."
+                            ),
+                            "target": "markets-date" if date_error else "",
+                        }
+                    ],
+                }
+            )
+            return with_publication_headers(
+                render(request, "grocery/markets.html", context, status=400), recent, historical
+            )
+        selected_date_value = cast(Any, context["selected_date_value"])
+        current_page = cast(int, context["page"])
+        total_pages = cast(int, context["total_pages"])
+        context.update(
+            {
+                "markets_state": "stale" if historical.stale_message else "ready",
+                "result_count_label": f"공개 시장 {context['total_count']}곳",
+                "pagination": _pagination_context(
+                    base_url=cast(str, context["markets_form_action"]),
+                    page=current_page,
+                    has_next=current_page < total_pages,
+                    parameters={"date": selected_date_value.isoformat()},
+                ),
+            }
+        )
+        return with_publication_headers(
+            render(request, "grocery/markets.html", context), recent, historical
+        )
+    except Http404:
+        raise
+    except DatabaseError, ObjectDoesNotExist, PublicReadIntegrityError, ValidationError:
+        log_event(_LOGGER, "ERROR", "public.markets.unavailable")
+        return historical_server_error(
+            request,
+            template="grocery/markets.html",
+            state_name="markets_state",
+            recent=recent,
+            historical=historical,
+        )
+
+
+def _market_url(series_id: uuid.UUID, region_id: uuid.UUID) -> str:
+    return reverse("grocery:markets", kwargs={"series_id": series_id, "region_id": region_id})
+
+
+def _market_recovery(
+    historical: Any, historical_series: Any, region_id: uuid.UUID
+) -> tuple[dict[str, object], uuid.UUID, bool]:
+    try:
+        return (
+            markets_context(
+                historical,
+                historical_series,
+                region_id=region_id,
+                selected_date=None,
+                page=1,
+            ),
+            region_id,
+            True,
+        )
+    except PublicParameterError:
+        context, recovery_region_id = market_recovery_context(historical, historical_series)
+        return context, recovery_region_id, False
+
+
+def _url(base_url: str, parameters: dict[str, object]) -> str:
+    return f"{base_url}?{urlencode(parameters)}" if parameters else base_url
+
+
+def _pagination_context(
+    *, base_url: str, page: int, has_next: bool, parameters: dict[str, object]
+) -> dict[str, object]:
+    def page_url(value: int) -> str:
+        values = dict(parameters)
+        if value != 1:
+            values["page"] = value
+        return _url(base_url, values)
+
+    return {
+        "has_multiple_pages": page > 1 or has_next,
+        "previous_url": page_url(page - 1) if page > 1 else "",
+        "next_url": page_url(page + 1) if has_next else "",
+        "page_label": f"{page}페이지",
+    }


## `feat(frontend): add regional price ledger`

diff --git a/grocery/templates/grocery/regions.html b/grocery/templates/grocery/regions.html
new file mode 100644
index 0000000..dc8042c
--- /dev/null
+++ b/grocery/templates/grocery/regions.html
@@ -0,0 +1,85 @@
+{% extends "grocery/base.html" %}
+
+{% block title %}{{ series.item_name|default:"품목" }} 지역별 조사값 | 초록장부{% endblock %}
+
+{% block content %}
+  {% include "grocery/_historical_header.html" with page_eyebrow="지역별 소매 조사값" page_summary="같은 조사일과 품목 조건에서 KAMIS가 제공한 지역별 평균과 조사 범위를 살펴봅니다." %}
+
+  {% if regions_state == "loading" or regions_state == "unavailable" or regions_state == "server_error" %}
+    {% include "grocery/_state_notice.html" with state=regions_state retry_url=retry_url %}
+  {% else %}
+    {% if regions_state == "validation" %}
+      {% include "grocery/_form_errors.html" with validation_errors=validation_errors %}
+    {% elif regions_state == "stale" %}
+      {% include "grocery/_state_notice.html" with state="stale" %}
+    {% endif %}
+
+    {% include "grocery/_publication_summary.html" with publication=historical_publication publication_label="지역별 공개 자료 정보" %}
+
+    <section class="scope-controls scope-controls--compact" aria-labelledby="regions-date-heading">
+      <div class="scope-controls__heading">
+        <h2 id="regions-date-heading">조사일</h2>
+        <p>지역과 시장 자료가 함께 확인된 실제 조사일만 선택할 수 있습니다.</p>
+      </div>
+      <form class="scope-form scope-form--date" action="{{ regions_form_action|default:'' }}" method="get">
+        <div class="field-group">
+          <label for="regions-date">조사일 선택</label>
+          <select id="regions-date" name="date"{% if date_error %} aria-invalid="true"{% endif %}>
+            {% for option in date_options %}
+              <option value="{{ option.value }}"{% if option.selected %} selected{% endif %}>{{ option.label }}</option>
+            {% endfor %}
+          </select>
+        </div>
+        <button class="button button--primary" type="submit">조사일 적용</button>
+      </form>
+    </section>
+
+    {% if regions_state == "validation" %}
+    {% elif regional_rows %}
+      <section class="region-section" aria-labelledby="regions-heading">
+        <div class="section-heading section-heading--stacked">
+          <p class="eyebrow">{{ selected_date.label }}</p>
+          <h2 id="regions-heading">지역별 조사 범위</h2>
+          <p>지역은 KAMIS 공식 지역명 순서로 표시합니다. 선의 양 끝은 최저·최고, 점은 평균입니다.</p>
+        </div>
+
+        <div class="region-ledger">
+          <div class="region-ledger__head" aria-hidden="true">
+            <span>지역</span><span>소매 조사 평균</span><span>최저·최고 조사 범위</span><span>시장 조사값</span>
+          </div>
+          <ul class="region-list">
+            {% for row in regional_rows %}
+              <li class="region-row">
+                <h3>{{ row.region_label }}</h3>
+                <dl class="region-row__average">
+                  <dt>소매 조사 평균</dt>
+                  <dd><data value="{{ row.mean_machine }}">{{ row.mean_label }}</data></dd>
+                </dl>
+                <div class="region-row__range">
+                  <dl>
+                    <div><dt>최저 조사값</dt><dd>{{ row.minimum_label }}</dd></div>
+                    <div><dt>최고 조사값</dt><dd>{{ row.maximum_label }}</dd></div>
+                  </dl>
+                  {% if row.meter %}
+                    <svg class="range-meter" viewBox="0 0 100 8" aria-hidden="true" focusable="false">
+                      <line class="range-meter__rail" x1="0" y1="4" x2="100" y2="4"/>
+                      <line class="range-meter__span" x1="{{ row.meter.minimum_x }}" y1="4" x2="{{ row.meter.maximum_x }}" y2="4"/>
+                      <circle class="range-meter__point" cx="{{ row.meter.mean_x }}" cy="4" r="2.5"/>
+                    </svg>
+                  {% endif %}
+                </div>
+                <div class="region-row__action">
+                  {% if row.markets_url %}<a href="{{ row.markets_url }}">시장별 값 보기 <span aria-hidden="true">→</span></a>{% else %}<span class="status-text status-text--unavailable">시장 조사값 없음</span>{% endif %}
+                </div>
+              </li>
+            {% endfor %}
+          </ul>
+        </div>
+      </section>
+    {% else %}
+      {% include "grocery/_state_notice.html" with state="server_error" retry_url=retry_url %}
+    {% endif %}
+  {% endif %}
+{% endblock %}
+
+{% block footer_note %}표시값은 KAMIS가 제공한 지역별 소매 조사 평균·최저·최고입니다. 개별 판매처의 실제 판매 금액과 다를 수 있습니다.{% endblock %}


## `feat(frontend): add market observation ledger`

diff --git a/grocery/templates/grocery/markets.html b/grocery/templates/grocery/markets.html
new file mode 100644
index 0000000..a1a2d53
--- /dev/null
+++ b/grocery/templates/grocery/markets.html
@@ -0,0 +1,71 @@
+{% extends "grocery/base.html" %}
+
+{% block title %}{{ series.item_name|default:"품목" }} 시장별 조사값 | 초록장부{% endblock %}
+
+{% block content %}
+  {% include "grocery/_historical_header.html" with page_eyebrow="시장별 소매 조사값" page_summary="선택한 지역과 조사일에 KAMIS가 기록한 시장별 소매 조사값을 확인합니다." back_url=regions_url back_label="지역별 조사값" %}
+
+  {% if markets_state == "loading" or markets_state == "unavailable" or markets_state == "server_error" %}
+    {% include "grocery/_state_notice.html" with state=markets_state retry_url=retry_url %}
+  {% else %}
+    {% if markets_state == "validation" %}
+      {% include "grocery/_form_errors.html" with validation_errors=validation_errors %}
+    {% elif markets_state == "stale" %}
+      {% include "grocery/_state_notice.html" with state="stale" %}
+    {% endif %}
+
+    {% include "grocery/_publication_summary.html" with publication=historical_publication publication_label="시장별 공개 자료 정보" %}
+
+    <section class="market-scope" aria-labelledby="market-scope-heading">
+      <div>
+        <p class="eyebrow">선택 지역</p>
+        <h2 id="market-scope-heading">{{ selected_region.label }}</h2>
+      </div>
+      <form class="scope-form scope-form--date" action="{{ markets_form_action|default:'' }}" method="get">
+        <div class="field-group">
+          <label for="markets-date">조사일</label>
+          <select id="markets-date" name="date"{% if date_error %} aria-invalid="true"{% endif %}>
+            {% for option in date_options %}
+              <option value="{{ option.value }}"{% if option.selected %} selected{% endif %}>{{ option.label }}</option>
+            {% endfor %}
+          </select>
+        </div>
+        <button class="button button--primary" type="submit">조사일 적용</button>
+      </form>
+    </section>
+
+    {% if markets_state == "validation" %}
+    {% elif market_rows %}
+      <section class="market-section" aria-labelledby="markets-heading">
+        <div class="section-heading">
+          <div>
+            <p class="eyebrow">{{ selected_date.label }}</p>
+            <h2 id="markets-heading">시장별 조사값</h2>
+          </div>
+          {% if result_count_label %}<p class="result-count">{{ result_count_label }}</p>{% endif %}
+        </div>
+        <p class="section-note">시장명은 KAMIS 공식 이름 순서로 표시합니다. 각 값은 시장별 관측이며 지역 평균이 아닙니다.</p>
+
+        <div class="market-ledger">
+          <div class="market-ledger__head" aria-hidden="true">
+            <span>시장</span><span>소매 조사값</span><span>조사일</span>
+          </div>
+          <ol class="market-list">
+            {% for row in market_rows %}
+              <li class="market-row">
+                <h3>{{ row.market_name }}</h3>
+                <dl class="market-row__price"><dt>소매 조사값</dt><dd><data value="{{ row.price_machine }}">{{ row.price_label }}</data></dd></dl>
+                <dl class="market-row__date"><dt>조사일</dt><dd><time datetime="{{ row.survey_date_iso }}">{{ row.survey_date_label }}</time></dd></dl>
+              </li>
+            {% endfor %}
+          </ol>
+        </div>
+        {% include "grocery/_pagination.html" with pagination=pagination %}
+      </section>
+    {% else %}
+      {% include "grocery/_state_notice.html" with state="server_error" retry_url=retry_url %}
+    {% endif %}
+  {% endif %}
+{% endblock %}
+
+{% block footer_note %}표시값은 KAMIS가 기록한 시장별 소매 조사값입니다. 개별 판매처의 실제 판매 금액과 다를 수 있습니다.{% endblock %}


