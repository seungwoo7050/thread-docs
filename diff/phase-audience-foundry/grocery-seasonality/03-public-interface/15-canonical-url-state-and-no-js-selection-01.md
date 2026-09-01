# 정규 URL 상태와 무자바스크립트 선택

## `feat(public): validate bounded URL state`

diff --git a/grocery/forms.py b/grocery/forms.py
index fe829ce..1049594 100644
--- a/grocery/forms.py
+++ b/grocery/forms.py
@@ -1,15 +1,35 @@
+"""Strict, non-reflecting validation for the public GET state."""
+
+from __future__ import annotations
+
 import unicodedata
-from typing import Any
+import uuid
+from collections.abc import Mapping
+from dataclasses import dataclass
+from datetime import date
+from typing import Any, ClassVar
 
 from django import forms
 from django.core.exceptions import ValidationError
+from django.http import QueryDict
 
 QUERY_MAX_LENGTH = 80
-CATEGORY_CHOICES = (
-    ("", "전체"),
-    ("vegetable", "채소류"),
-    ("fruit", "과일류"),
+SELECTION_LIMIT = 5
+CATEGORY_CHOICES = (("", "전체"), ("vegetable", "채소류"), ("fruit", "과일류"))
+PERIOD_CHOICES = (("week", "1주 비교"), ("month", "1개월 비교"), ("year", "1년 비교"))
+DIRECTION_CHOICES = (
+    ("all", "전체"),
+    ("lower", "낮음"),
+    ("equal", "같음"),
+    ("higher", "높음"),
+    ("unavailable", "비교값 없음"),
 )
+SORT_CHOICES = (
+    ("name", "품목명 순"),
+    ("change_asc", "변화율 낮은 순"),
+    ("change_desc", "변화율 높은 순"),
+)
+RANGE_CHOICES = (("12", "12개월"), ("36", "36개월"), ("60", "60개월"))
 
 
 class OfficialItemQueryField(forms.CharField):
@@ -21,8 +41,6 @@ class OfficialItemQueryField(forms.CharField):
     }
 
     def __init__(self, *args: Any, **kwargs: Any) -> None:
-        # Inspect the original value before trimming so an edge CR/LF cannot be
-        # normalized away and accepted as an ordinary query.
         kwargs["strip"] = False
         super().__init__(*args, **kwargs)
 
@@ -38,22 +56,199 @@ class OfficialItemQueryField(forms.CharField):
         return raw_value.strip()
 
 
+class CanonicalPageField(forms.RegexField):
+    def __init__(self, *args: Any, **kwargs: Any) -> None:
+        kwargs.setdefault("regex", r"^(?:[1-9]|[1-9][0-9]|100)$")
+        kwargs.setdefault("required", False)
+        kwargs.setdefault("initial", "1")
+        kwargs.setdefault("error_messages", {"invalid": "페이지를 확인하세요."})
+        super().__init__(*args, **kwargs)
+
+    def clean(self, value: object) -> int:
+        cleaned = super().clean(value)
+        return int(cleaned or "1")
+
+
+class CanonicalUUIDField(forms.RegexField):
+    def __init__(self, *args: Any, **kwargs: Any) -> None:
+        kwargs.setdefault(
+            "regex",
+            r"^[0-9a-f]{8}-[0-9a-f]{4}-[1-5][0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$",
+        )
+        kwargs.setdefault("required", False)
+        kwargs.setdefault("error_messages", {"invalid": "지역을 확인하세요."})
+        super().__init__(*args, **kwargs)
+
+    def clean(self, value: object) -> uuid.UUID | None:
+        cleaned = super().clean(value)
+        return uuid.UUID(cleaned) if cleaned else None
+
+
+class CanonicalDateField(forms.RegexField):
+    def __init__(self, *args: Any, **kwargs: Any) -> None:
+        kwargs.setdefault("regex", r"^[0-9]{4}-(?:0[1-9]|1[0-2])-(?:0[1-9]|[12][0-9]|3[01])$")
+        kwargs.setdefault("required", False)
+        kwargs.setdefault("error_messages", {"invalid": "조사일을 확인하세요."})
+        super().__init__(*args, **kwargs)
+
+    def clean(self, value: object) -> date | None:
+        cleaned = super().clean(value)
+        if not cleaned:
+            return None
+        try:
+            parsed = date.fromisoformat(cleaned)
+        except ValueError:
+            raise ValidationError(self.error_messages["invalid"], code="invalid") from None
+        if parsed.isoformat() != cleaned:
+            raise ValidationError(self.error_messages["invalid"], code="invalid")
+        return parsed
+
+
+class StrictQueryForm(forms.Form):
+    """Reject unknown and duplicate public parameters without echoing their values."""
+
+    allowed_parameters: ClassVar[frozenset[str]] = frozenset()
+
+    def clean(self) -> dict[str, object]:
+        cleaned = super().clean() or {}
+        data = self.data
+        if isinstance(data, QueryDict):
+            if set(data) - self.allowed_parameters:
+                raise ValidationError("요청 항목을 확인하세요.", code="unknown_parameter")
+            for name in self.allowed_parameters:
+                if len(data.getlist(name)) > 1:
+                    self.add_error(
+                        name if name in self.fields else None,
+                        ValidationError("한 번만 선택해 주세요.", code="duplicate_parameter"),
+                    )
+        return cleaned
+
+
+class CatalogForm(StrictQueryForm):
+    allowed_parameters = frozenset({"q", "category", "period", "direction", "sort", "page"})
+
+    q = OfficialItemQueryField(
+        label="공식 품목명",
+        required=False,
+        max_length=QUERY_MAX_LENGTH,
+        error_messages={"max_length": "품목명은 80자 이하로 입력하세요."},
+    )
+    category = forms.ChoiceField(
+        label="부류",
+        required=False,
+        choices=CATEGORY_CHOICES,
+        error_messages={"invalid_choice": "부류 선택을 확인해 주세요."},
+    )
+    period = forms.ChoiceField(
+        label="비교 기간",
+        required=False,
+        choices=PERIOD_CHOICES,
+        initial="week",
+        error_messages={"invalid_choice": "비교 기간을 확인하세요."},
+    )
+    direction = forms.ChoiceField(
+        label="변화 방향",
+        required=False,
+        choices=DIRECTION_CHOICES,
+        initial="all",
+        error_messages={"invalid_choice": "변화 방향을 확인하세요."},
+    )
+    sort = forms.ChoiceField(
+        label="표시 순서",
+        required=False,
+        choices=SORT_CHOICES,
+        initial="name",
+        error_messages={"invalid_choice": "표시 순서를 확인하세요."},
+    )
+    page = CanonicalPageField(label="페이지")
+
+    def clean(self) -> dict[str, object]:
+        cleaned = super().clean()
+        if "period" not in self.errors:
+            cleaned["period"] = cleaned.get("period") or "week"
+        if "direction" not in self.errors:
+            cleaned["direction"] = cleaned.get("direction") or "all"
+        if "sort" not in self.errors:
+            cleaned["sort"] = cleaned.get("sort") or "name"
+        if cleaned.get("q") and cleaned.get("page", 1) != 1:
+            self.add_error("page", "검색 결과는 첫 페이지만 표시합니다.")
+        return cleaned
+
+
 class SearchForm(forms.Form):
-    """Validate the bounded public catalog GET query."""
+    """Preserve the Phase 0 two-field form for internal callers and regression tests."""
 
     q = OfficialItemQueryField(
         label="공식 품목명",
         required=False,
         max_length=QUERY_MAX_LENGTH,
-        error_messages={
-            "max_length": "품목명은 80자 이하로 입력하세요.",
-        },
+        error_messages={"max_length": "품목명은 80자 이하로 입력하세요."},
     )
     category = forms.ChoiceField(
         label="부류",
         required=False,
         choices=CATEGORY_CHOICES,
-        error_messages={
-            "invalid_choice": "부류 선택을 확인해 주세요.",
-        },
+        error_messages={"invalid_choice": "부류 선택을 확인해 주세요."},
     )
+
+
+class HistoryForm(StrictQueryForm):
+    allowed_parameters = frozenset({"range", "region"})
+    range = forms.ChoiceField(
+        required=False,
+        choices=RANGE_CHOICES,
+        initial="36",
+        error_messages={"invalid_choice": "표시 기간을 확인하세요."},
+    )
+    region = CanonicalUUIDField()
+
+    def clean(self) -> dict[str, object]:
+        cleaned = super().clean()
+        cleaned["range"] = cleaned.get("range") or "36"
+        return cleaned
+
+
+class RegionsForm(StrictQueryForm):
+    allowed_parameters = frozenset({"date"})
+    date = CanonicalDateField()
+
+
+class MarketsForm(StrictQueryForm):
+    allowed_parameters = frozenset({"date", "page"})
+    date = CanonicalDateField()
+    page = CanonicalPageField(label="페이지")
+
+
+@dataclass(frozen=True, slots=True)
+class SelectionQuery:
+    series_ids: tuple[uuid.UUID, ...]
+
+
+def parse_selection_query(data: QueryDict | Mapping[str, object]) -> SelectionQuery:
+    """Validate the sole repeatable public parameter and preserve first-seen order."""
+
+    if not isinstance(data, QueryDict):
+        query = QueryDict(mutable=True)
+        for key, value in data.items():
+            query.appendlist(key, str(value))
+        data = query
+    if set(data) - {"series"}:
+        raise ValidationError("요청 항목을 확인하세요.", code="unknown_parameter")
+    raw_values = data.getlist("series")
+    if len(raw_values) > SELECTION_LIMIT:
+        raise ValidationError("품목은 최대 다섯 개까지 선택할 수 있습니다.", code="selection_limit")
+    parsed: list[uuid.UUID] = []
+    seen: set[uuid.UUID] = set()
+    for raw_value in raw_values:
+        try:
+            value = uuid.UUID(raw_value)
+        except (AttributeError, ValueError):
+            raise ValidationError("선택한 품목을 확인하세요.", code="invalid_series") from None
+        if str(value) != raw_value:
+            raise ValidationError("선택한 품목을 확인하세요.", code="invalid_series")
+        if value not in seen:
+            seen.add(value)
+            parsed.append(value)
+    if len(parsed) > SELECTION_LIMIT:
+        raise ValidationError("품목은 최대 다섯 개까지 선택할 수 있습니다.", code="selection_limit")
+    return SelectionQuery(tuple(parsed))


## `feat(public): expose URL selection route`

diff --git a/grocery/selection_views.py b/grocery/selection_views.py
new file mode 100644
index 0000000..c621b3c
--- /dev/null
+++ b/grocery/selection_views.py
@@ -0,0 +1,123 @@
+"""No-JavaScript URL selection route over the active recent publication."""
+
+from __future__ import annotations
+
+import logging
+from typing import Final
+from urllib.parse import urlencode
+
+from django.core.exceptions import ObjectDoesNotExist, ValidationError
+from django.db import DatabaseError
+from django.http import HttpRequest, HttpResponse
+from django.shortcuts import render
+from django.urls import reverse
+from django.views.decorators.http import require_safe
+
+from grocery.forms import parse_selection_query
+from grocery.observability import log_event
+from grocery.public_read import (
+    load_active_publication,
+    publication_candidate_entries,
+    publication_context,
+    publication_entries_for_series,
+    selection_candidate_context,
+    selection_item_context,
+)
+
+_LOGGER: Final = logging.getLogger("grocery.audit")
+
+
+@require_safe
+def selection(request: HttpRequest) -> HttpResponse:
+    try:
+        selection_query = parse_selection_query(request.GET)
+    except ValidationError:
+        return render(
+            request,
+            "grocery/selection.html",
+            {
+                "home_url": reverse("grocery:catalog"),
+                "catalog_url": reverse("grocery:catalog"),
+                "selection_state": "validation",
+                "validation_errors": [{"message": "선택한 품목을 확인하세요.", "target": ""}],
+            },
+            status=400,
+        )
+    try:
+        active = load_active_publication()
+        base = {
+            "home_url": reverse("grocery:catalog"),
+            "catalog_url": reverse("grocery:catalog"),
+            "selection_form_action": reverse("grocery:selection"),
+        }
+        if active is None:
+            return render(
+                request,
+                "grocery/selection.html",
+                {**base, "selection_state": "unavailable", "items": []},
+            )
+        entries = publication_entries_for_series(active, selection_query.series_ids)
+        entry_ids = tuple(entry.snapshot.series_id for entry in entries)
+        excluded_count = len(selection_query.series_ids) - len(entry_ids)
+        items = []
+        for entry in entries:
+            remaining = tuple(
+                series_id for series_id in entry_ids if series_id != entry.snapshot.series_id
+            )
+            items.append(
+                selection_item_context(
+                    entry,
+                    active,
+                    detail_url=reverse(
+                        "grocery:detail", kwargs={"series_id": entry.snapshot.series_id}
+                    ),
+                    remove_url=_selection_url(remaining),
+                )
+            )
+        limit_reached = len(items) >= 5
+        candidates = (
+            []
+            if limit_reached
+            else [
+                selection_candidate_context(entry)
+                for entry in publication_candidate_entries(active, excluded_series_ids=entry_ids)
+            ]
+        )
+        canonical_url = _selection_url(entry_ids)
+        context = {
+            **base,
+            "selection_url": canonical_url,
+            "selection_count": len(items),
+            "selection_state": "partial" if excluded_count else "ready",
+            "selection_is_stale": bool(active.stale_message),
+            "publication": publication_context(active),
+            "items": items,
+            "excluded_count": excluded_count,
+            "result_count_label": f"선택 품목 {len(items)}개",
+            "clear_url": reverse("grocery:selection") if items else "",
+            "selection_limit_reached": limit_reached,
+            "selection_candidates": candidates,
+            "can_add_selection": bool(candidates) and not limit_reached,
+        }
+        response = render(request, "grocery/selection.html", context)
+        response.headers["X-Publication-Fact-Set"] = active.revision.typed_fact_set_sha256
+        return response
+    except DatabaseError, ObjectDoesNotExist, ValidationError:
+        log_event(_LOGGER, "ERROR", "public.selection.unavailable")
+        return render(
+            request,
+            "grocery/selection.html",
+            {
+                "home_url": reverse("grocery:catalog"),
+                "catalog_url": reverse("grocery:catalog"),
+                "selection_state": "server_error",
+                "retry_url": reverse("grocery:selection"),
+            },
+            status=503,
+        )
+
+
+def _selection_url(series_ids: tuple[object, ...]) -> str:
+    base = reverse("grocery:selection")
+    query = urlencode({"series": [str(series_id) for series_id in series_ids]}, doseq=True)
+    return f"{base}?{query}" if query else base


## `feat(frontend): add the URL selection ledger`

diff --git a/grocery/templates/grocery/selection.html b/grocery/templates/grocery/selection.html
new file mode 100644
index 0000000..f3218a8
--- /dev/null
+++ b/grocery/templates/grocery/selection.html
@@ -0,0 +1,95 @@
+{% extends "grocery/base.html" %}
+
+{% block title %}선택한 품목 | 초록장부{% endblock %}
+
+{% block content %}
+  <nav class="breadcrumb" aria-label="목록으로 돌아가기">
+    <a href="{{ catalog_url|default:'/' }}">← 채소·과일 소매 조사값</a>
+  </nav>
+
+  <header class="page-heading selection-heading">
+    <p class="eyebrow">최대 다섯 품목</p>
+    <h1>선택한 품목</h1>
+    <p class="page-heading__summary">
+      품목마다 같은 조건 안에서 최근 조사값과 이전 제공값의 차이를 확인합니다.
+      선택 내용은 이 주소에만 담깁니다.
+    </p>
+  </header>
+
+  {% if selection_state == "loading" or selection_state == "unavailable" or selection_state == "server_error" %}
+    {% include "grocery/_state_notice.html" with state=selection_state retry_url=retry_url %}
+  {% else %}
+    {% if selection_state == "validation" %}
+      {% include "grocery/_form_errors.html" with validation_errors=validation_errors %}
+      <a class="button button--primary" href="{{ catalog_url|default:'/' }}">공개 목록에서 다시 선택하기</a>
+    {% else %}
+      {% if selection_state == "partial" %}
+        {% include "grocery/_state_notice.html" with state="partial" excluded_count=excluded_count %}
+      {% endif %}
+
+      {% include "grocery/_publication_summary.html" with publication=publication publication_label="선택 목록 공개 자료 정보" %}
+
+      {% if items %}
+        <section class="selection-section" aria-labelledby="selection-heading">
+          <div class="section-heading selection-section__heading">
+            <h2 id="selection-heading">최근 조사값</h2>
+            {% if result_count_label %}<p class="result-count">{{ result_count_label }}</p>{% endif %}
+          </div>
+          <div class="selection-actions">
+            <a href="{{ catalog_url|default:'/' }}">품목 더 찾기</a>
+            {% if clear_url %}<a href="{{ clear_url }}">선택 모두 비우기</a>{% endif %}
+          </div>
+
+          <ol class="selection-list">
+            {% for item in items %}
+              <li class="selection-row">
+                <header class="selection-row__identity">
+                  <p class="eyebrow">{{ item.category_label }}</p>
+                  <h3><a href="{{ item.detail_url }}">{{ item.item_name }}</a></h3>
+                  <p>
+                    <span>품종 {{ item.variety_name }}</span>
+                    <span>등급 {{ item.grade_name }}</span>
+                    <span>판매 단위 {{ item.unit_label }}</span>
+                  </p>
+                </header>
+                <dl class="selection-row__price">
+                  <dt>소매 조사 평균</dt>
+                  <dd>{% if item.current_price_machine %}<data value="{{ item.current_price_machine }}">{% endif %}{{ item.current_price_label }}{% if item.current_price_machine %}</data>{% endif %}</dd>
+                </dl>
+                <dl class="selection-row__comparison">
+                  <dt>{{ selected_period_heading|default:"1주 비교" }}</dt>
+                  <dd>
+                    {% with comparison=item.comparison|default:item.week_comparison %}
+                      {% if comparison.available %}
+                        <span class="direction direction--{{ comparison.direction_code|lower }}">
+                          <span class="direction__symbol" aria-hidden="true">{% if comparison.direction_code == "LOWER" %}↓{% elif comparison.direction_code == "HIGHER" %}↑{% else %}＝{% endif %}</span>
+                          <span>{% if comparison.direction_code == "EQUAL" %}{{ comparison.period_label }}과 같음 ({{ comparison.percentage_display }}){% else %}{{ comparison.period_label }}보다 {{ comparison.difference_display }} {{ comparison.direction_label }} ({{ comparison.percentage_display }}){% endif %}</span>
+                        </span>
+                      {% else %}
+                        <span class="status-text status-text--unavailable"><span aria-hidden="true">○</span> 1주 전 비교값 없음</span>
+                      {% endif %}
+                    {% endwith %}
+                  </dd>
+                </dl>
+                <dl class="selection-row__date">
+                  <dt>조사일</dt>
+                  <dd><time datetime="{{ item.source_date_iso }}">{{ item.source_date_label }}</time></dd>
+                </dl>
+                <div class="selection-row__remove"><a href="{{ item.remove_url }}">목록에서 빼기</a></div>
+              </li>
+            {% endfor %}
+          </ol>
+        </section>
+      {% else %}
+        <section class="selection-empty" role="status" aria-labelledby="selection-empty-heading">
+          <p class="selection-empty__mark" aria-hidden="true">○</p>
+          <div>
+            <h2 id="selection-empty-heading">선택한 품목이 없습니다</h2>
+            <p>궁금한 품목을 목록에서 골라 한곳에 모아보세요.</p>
+            <a class="button button--primary" href="{{ catalog_url|default:'/' }}">품목 둘러보기</a>
+          </div>
+        </section>
+      {% endif %}
+    {% endif %}
+  {% endif %}
+{% endblock %}


## `feat(frontend): complete URL selection building`

diff --git a/grocery/static/grocery/app.css b/grocery/static/grocery/app.css
index d7c18ab..59c9291 100644
--- a/grocery/static/grocery/app.css
+++ b/grocery/static/grocery/app.css
@@ -1540,6 +1540,48 @@ select[aria-invalid="true"] {
   margin-bottom: var(--space-6);
 }
 
+.selection-add {
+  min-width: 0;
+  margin-bottom: var(--space-8);
+  padding-block: var(--space-4);
+  border-block: 1px solid var(--color-border-strong);
+}
+
+.selection-add__heading {
+  max-width: var(--measure);
+  margin-bottom: var(--space-4);
+}
+
+.selection-add__heading h2,
+.selection-add__heading p:last-child,
+.selection-add__status {
+  margin-bottom: 0;
+}
+
+.selection-add__heading p:last-child,
+.selection-add__status {
+  color: var(--color-muted);
+}
+
+.selection-add__form {
+  display: grid;
+  min-width: 0;
+  gap: var(--space-3);
+}
+
+.selection-add__form .button {
+  align-self: end;
+}
+
+.selection-add__status {
+  padding-top: var(--space-3);
+  border-top: 1px solid var(--color-border);
+}
+
+.selection-add__status strong {
+  color: var(--color-text);
+}
+
 .selection-actions {
   display: flex;
   min-width: 0;
@@ -1741,6 +1783,11 @@ select[aria-invalid="true"] {
     align-items: end;
   }
 
+  .selection-add__form {
+    grid-template-columns: minmax(0, 1fr) auto;
+    align-items: end;
+  }
+
   .ledger-entry {
     grid-template-columns: repeat(2, minmax(0, 1fr));
     align-items: start;
diff --git a/grocery/templates/grocery/selection.html b/grocery/templates/grocery/selection.html
index 3fad909..2f58780 100644
--- a/grocery/templates/grocery/selection.html
+++ b/grocery/templates/grocery/selection.html
@@ -29,16 +29,51 @@
 
       {% include "grocery/_publication_summary.html" with publication=publication publication_label="선택 목록 공개 자료 정보" %}
 
+      <section class="selection-add" aria-labelledby="selection-add-heading">
+        <div class="selection-add__heading">
+          <p class="eyebrow">현재 목록에 이어서</p>
+          <h2 id="selection-add-heading">품목 추가</h2>
+          <p>품종·등급·판매 단위를 확인하고 한 품목을 더 담으세요.</p>
+        </div>
+
+        {% if selection_limit_reached %}
+          <p class="selection-add__status" role="status">
+            <strong>다섯 품목을 모두 선택했습니다.</strong>
+            다른 품목을 담으려면 먼저 목록에서 하나를 빼세요.
+          </p>
+        {% elif can_add_selection and selection_candidates %}
+          <form class="selection-add__form" action="{{ selection_form_action|default:'/selection/' }}" method="get">
+            {% for item in items %}
+              <input type="hidden" name="series" value="{{ item.series_value }}">
+            {% endfor %}
+            <div class="field-group">
+              <label for="selection-add-item">추가할 품목</label>
+              <p id="selection-add-hint" class="field-hint">현재 목록의 마지막에 추가합니다. 최대 다섯 품목까지 담을 수 있습니다.</p>
+              <select id="selection-add-item" name="series" required aria-describedby="selection-add-hint">
+                <option value="">품목을 선택하세요</option>
+                {% for candidate in selection_candidates %}
+                  <option value="{{ candidate.value }}">{{ candidate.label }}</option>
+                {% endfor %}
+              </select>
+            </div>
+            <button class="button button--primary" type="submit">선택 목록에 추가</button>
+          </form>
+        {% else %}
+          <p class="selection-add__status" role="status">현재 목록에 더 추가할 수 있는 공개 품목이 없습니다.</p>
+        {% endif %}
+      </section>
+
       {% if items %}
         <section class="selection-section" aria-labelledby="selection-heading">
           <div class="section-heading selection-section__heading">
             <h2 id="selection-heading">최근 조사값</h2>
             {% if result_count_label %}<p class="result-count">{{ result_count_label }}</p>{% endif %}
           </div>
-          <div class="selection-actions">
-            <a href="{{ catalog_url|default:'/' }}">품목 더 찾기</a>
-            {% if clear_url %}<a href="{{ clear_url }}">선택 모두 비우기</a>{% endif %}
-          </div>
+          {% if clear_url %}
+            <div class="selection-actions">
+              <a href="{{ clear_url }}">선택 모두 비우기</a>
+            </div>
+          {% endif %}
 
           <ol class="selection-list">
             {% for item in items %}


