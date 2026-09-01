## `fix(public): keep searches private and uncached`

diff --git a/grocery/security.py b/grocery/security.py
index 671fdf9..df38a30 100644
--- a/grocery/security.py
+++ b/grocery/security.py
@@ -24,6 +24,7 @@ CONTENT_SECURITY_POLICY: Final = (
 )
 
 SECURITY_HEADERS: Final[dict[str, str]] = {
+    "Cache-Control": "no-store",
     "Content-Security-Policy": CONTENT_SECURITY_POLICY,
     "Permissions-Policy": "camera=(), geolocation=(), microphone=(), payment=()",
     "Cross-Origin-Opener-Policy": "same-origin",
diff --git a/grocery/templates/grocery/catalog.html b/grocery/templates/grocery/catalog.html
index c0137be..3c01cf6 100644
--- a/grocery/templates/grocery/catalog.html
+++ b/grocery/templates/grocery/catalog.html
@@ -14,12 +14,19 @@
 
   <section class="search-panel" aria-labelledby="catalog-search-heading">
     <h2 id="catalog-search-heading">품목 찾기</h2>
-    {% if query_error %}
+    {% if query_error or category_error %}
       <div id="search-error" class="form-error" role="alert">
         <span aria-hidden="true">!</span>
         <div class="form-error__content">
-          <p><strong>입력 확인:</strong> {{ query_error }}</p>
-          <a class="form-error__link" href="#catalog-query">검색어 입력으로 이동</a>
+          <p><strong>입력 확인:</strong> 입력 내용을 고친 뒤 다시 검색해 주세요.</p>
+          <ul>
+            {% if query_error %}
+              <li>{{ query_error }} <a class="form-error__link" href="#catalog-query">검색어 입력으로 이동</a></li>
+            {% endif %}
+            {% if category_error %}
+              <li>{{ category_error }} <a class="form-error__link" href="{{ form_action }}">부류 선택 초기화</a></li>
+            {% endif %}
+          </ul>
         </div>
       </div>
     {% endif %}
@@ -31,7 +38,6 @@
           id="catalog-query"
           name="q"
           type="search"
-          value="{{ query|default:'' }}"
           maxlength="80"
           autocomplete="off"
           enterkeyhint="search"
@@ -65,7 +71,8 @@
     {% endif %}
   </section>
 
-  {% if catalog_state == "loading" or catalog_state == "unavailable" or catalog_state == "server_error" %}
+  {% if catalog_state == "validation" %}
+  {% elif catalog_state == "loading" or catalog_state == "unavailable" or catalog_state == "server_error" %}
     {% include "grocery/_state_notice.html" with state=catalog_state state_message=status_message retry_url=retry_url %}
   {% else %}
     {% if catalog_state == "stale" %}
diff --git a/grocery/tests/test_public_routes.py b/grocery/tests/test_public_routes.py
index 0217e56..cb22812 100644
--- a/grocery/tests/test_public_routes.py
+++ b/grocery/tests/test_public_routes.py
@@ -119,10 +119,46 @@ def test_invalid_mobile_search_input_returns_associated_correction_error() -> No
     assert 'aria-invalid="true"' in invalid_html
     assert 'aria-describedby="catalog-query-hint search-error"' in invalid_html
     assert "검색어는 80자 이하여야 합니다." in invalid_html
+    assert invalid_query not in invalid_html
+    assert "조건에 맞는 항목 없음" not in invalid_html
     assert corrected_response.status_code == 200
     assert 'aria-invalid="true"' not in corrected_response.content.decode()
 
 
+@pytest.mark.django_db
+def test_category_validation_is_distinct_and_never_reflects_query_or_choice() -> None:
+    activate_publication()
+    query_marker = "응답에나오면안되는검색표시"
+    category_marker = "응답에나오면안되는부류표시"
+
+    response = Client().get(
+        reverse("grocery:catalog"),
+        {"q": query_marker, "category": category_marker},
+    )
+    html = response.content.decode()
+
+    assert response.status_code == 400
+    assert "부류 선택을 확인해 주세요." in html
+    assert "부류 선택 초기화" in html
+    assert 'aria-invalid="true"' not in html
+    assert query_marker not in html
+    assert category_marker not in html
+    assert "조건에 맞는 항목 없음" not in html
+
+
+@pytest.mark.django_db
+def test_valid_unmatched_query_is_not_echoed_or_propagated_to_category_urls() -> None:
+    activate_publication()
+    marker = "응답에나오면안되는정상검색표시"
+
+    response = Client().get(reverse("grocery:catalog"), {"q": marker})
+    html = response.content.decode()
+
+    assert response.status_code == 200
+    assert marker not in html
+    assert "q=" not in html
+
+
 @pytest.mark.django_db
 def test_public_request_never_calls_external_source_client() -> None:
     _, snapshot, _ = activate_publication()
@@ -218,3 +254,37 @@ def test_unrelated_unpublished_series_does_not_change_catalog_results() -> None:
 
     assert before == after
     assert "공개되지않은긴후보이름" not in after.decode()
+
+
+@pytest.mark.django_db
+def test_public_html_is_no_store_for_success_validation_not_found_and_failure() -> None:
+    _, snapshot, _ = activate_publication()
+    client = Client()
+    responses = [
+        client.get(reverse("grocery:catalog")),
+        client.get(reverse("grocery:catalog"), {"q": "가" * 81}),
+        client.get(reverse("grocery:detail", kwargs={"series_id": uuid.uuid4()})),
+        client.get(reverse("grocery:detail", kwargs={"series_id": snapshot.series_id})),
+    ]
+    with patch("grocery.views.load_active_publication", side_effect=DatabaseError("hidden")):
+        responses.append(client.get(reverse("grocery:catalog")))
+
+    assert [response.status_code for response in responses] == [200, 400, 404, 200, 503]
+    assert all(response.headers["Cache-Control"] == "no-store" for response in responses)
+
+
+@pytest.mark.django_db
+def test_public_views_reject_unsafe_methods_before_reading_publication() -> None:
+    client = Client()
+    paths = (
+        reverse("grocery:catalog"),
+        reverse("grocery:detail", kwargs={"series_id": uuid.uuid4()}),
+    )
+
+    with patch("grocery.views.load_active_publication") as publication_read:
+        for path in paths:
+            for method in (client.post, client.put, client.patch, client.delete):
+                response = method(path)
+                assert response.status_code == 405
+
+    publication_read.assert_not_called()
diff --git a/grocery/tests/test_public_templates.py b/grocery/tests/test_public_templates.py
index 2a3f7a0..6235561 100644
--- a/grocery/tests/test_public_templates.py
+++ b/grocery/tests/test_public_templates.py
@@ -142,9 +142,14 @@ def test_catalog_renders_semantic_search_and_long_identity() -> None:
 
 
 def test_catalog_validation_error_is_associated_with_input() -> None:
+    private_query = "잘못된 입력"
     html = render(
         "grocery/catalog.html",
-        catalog_context(query="잘못된 입력", query_error="검색어는 80자 이하여야 합니다."),
+        catalog_context(
+            catalog_state="validation",
+            query=private_query,
+            query_error="검색어는 80자 이하여야 합니다.",
+        ),
     )
 
     assert 'id="search-error"' in html
@@ -153,6 +158,8 @@ def test_catalog_validation_error_is_associated_with_input() -> None:
     assert 'aria-describedby="catalog-query-hint search-error"' in html
     assert 'href="#catalog-query">검색어 입력으로 이동</a>' in html
     assert "검색어는 80자 이하여야 합니다." in html
+    assert private_query not in html
+    assert "조건에 맞는 항목 없음" not in html
 
 
 @pytest.mark.parametrize(
diff --git a/grocery/tests/test_security.py b/grocery/tests/test_security.py
index 307750d..b1a7fe8 100644
--- a/grocery/tests/test_security.py
+++ b/grocery/tests/test_security.py
@@ -56,6 +56,7 @@ def test_privileged_browser_capabilities_and_cross_origin_access_are_denied() ->
     assert response.headers["Cross-Origin-Resource-Policy"] == "same-origin"
     assert response.headers["X-Frame-Options"] == "DENY"
     assert response.headers["X-Content-Type-Options"] == "nosniff"
+    assert response.headers["Cache-Control"] == "no-store"
 
 
 @pytest.mark.parametrize("path", ["/admin", "/admin/", "/admin/auth/user/"])
diff --git a/grocery/views.py b/grocery/views.py
index 1e453c1..d73d9f5 100644
--- a/grocery/views.py
+++ b/grocery/views.py
@@ -13,6 +13,7 @@ from django.db import DatabaseError
 from django.http import Http404, HttpRequest, HttpResponse
 from django.shortcuts import render
 from django.urls import reverse
+from django.views.decorators.http import require_safe
 
 from grocery.forms import QUERY_MAX_LENGTH, SearchForm
 from grocery.observability import log_event
@@ -26,24 +27,27 @@ from grocery.public_read import (
 _LOGGER: Final = logging.getLogger("grocery.audit")
 _QA_STATES: Final = frozenset({"loading", "empty", "unavailable", "stale", "server_error"})
 _QA_DETAIL_STATES: Final = frozenset({"loading", "unavailable", "stale", "server_error"})
+_QUERY_ERROR_MESSAGES: Final = {
+    "max_length": f"검색어는 {QUERY_MAX_LENGTH}자 이하여야 합니다.",
+    "unsafe": "검색어에는 줄바꿈이나 제어 문자를 사용할 수 없습니다.",
+}
 
 
+@require_safe
 def catalog(request: HttpRequest) -> HttpResponse:
     form = SearchForm(request.GET if request.GET else None)
     query = ""
     category = ""
-    query_error = ""
     if form.is_bound and not form.is_valid():
-        raw_query = request.GET.get("q", "")
-        query = raw_query[: QUERY_MAX_LENGTH + 1]
-        query_error = str(next(iter(form.errors.values()))[0])
-        context = _catalog_base_context(query=query, category="")
+        query_error = _query_error(form)
+        category_error = "부류 선택을 확인해 주세요." if "category" in form.errors else ""
+        context = _catalog_base_context(category="")
         context.update(
             {
-                "catalog_state": "empty",
+                "catalog_state": "validation",
                 "query_error": query_error,
+                "category_error": category_error,
                 "results": [],
-                "status_message": "입력을 확인하고 다시 검색해 주세요.",
             }
         )
         return render(request, "grocery/catalog.html", context, status=400)
@@ -53,7 +57,7 @@ def catalog(request: HttpRequest) -> HttpResponse:
 
     try:
         active = load_active_publication()
-        context = _catalog_base_context(query=query, category=category)
+        context = _catalog_base_context(category=category)
         if active is None:
             context.update(
                 {
@@ -89,7 +93,7 @@ def catalog(request: HttpRequest) -> HttpResponse:
         return _publication_response(response, active.revision.typed_fact_set_sha256)
     except DatabaseError, ValidationError:
         log_event(_LOGGER, "ERROR", "public.catalog.unavailable")
-        context = _catalog_base_context(query=query, category=category)
+        context = _catalog_base_context(category=category)
         context.update(
             {
                 "catalog_state": "server_error",
@@ -101,6 +105,7 @@ def catalog(request: HttpRequest) -> HttpResponse:
         return render(request, "grocery/catalog.html", context, status=503)
 
 
+@require_safe
 def detail(request: HttpRequest, series_id: uuid.UUID) -> HttpResponse:
     try:
         active = load_active_publication()
@@ -139,10 +144,11 @@ def detail(request: HttpRequest, series_id: uuid.UUID) -> HttpResponse:
         return render(request, "grocery/detail.html", context, status=503)
 
 
+@require_safe
 def qa_catalog_state(request: HttpRequest, state: str) -> HttpResponse:
     if not settings.QA_STATE_PREVIEWS_ENABLED or state not in _QA_STATES:
         raise Http404
-    context = _catalog_base_context(query="아주긴한국어공식품목명", category="vegetable")
+    context = _catalog_base_context(category="vegetable")
     context.update(
         {
             "qa_preview": True,
@@ -158,6 +164,7 @@ def qa_catalog_state(request: HttpRequest, state: str) -> HttpResponse:
     )
 
 
+@require_safe
 def qa_detail_state(request: HttpRequest, state: str) -> HttpResponse:
     if not settings.QA_STATE_PREVIEWS_ENABLED or state not in _QA_DETAIL_STATES:
         raise Http404
@@ -183,17 +190,16 @@ def qa_detail_state(request: HttpRequest, state: str) -> HttpResponse:
     )
 
 
-def _catalog_base_context(*, query: str, category: str) -> dict[str, object]:
+def _catalog_base_context(*, category: str) -> dict[str, object]:
     catalog_url = reverse("grocery:catalog")
     return {
         "home_url": catalog_url,
         "form_action": catalog_url,
-        "query": query,
         "selected_category": category,
         "categories": [
             {
                 "label": label,
-                "url": _category_url(catalog_url, query=query, category=value),
+                "url": _category_url(catalog_url, category=value),
                 "selected": category == value,
             }
             for value, label in (("", "전체"), ("vegetable", "채소류"), ("fruit", "과일류"))
@@ -201,10 +207,8 @@ def _catalog_base_context(*, query: str, category: str) -> dict[str, object]:
     }
 
 
-def _category_url(base_url: str, *, query: str, category: str) -> str:
+def _category_url(base_url: str, *, category: str) -> str:
     parameters = {}
-    if query:
-        parameters["q"] = query
     if category:
         parameters["category"] = category
     return f"{base_url}?{urlencode(parameters)}" if parameters else base_url
@@ -212,10 +216,16 @@ def _category_url(base_url: str, *, query: str, category: str) -> str:
 
 def _publication_response(response: HttpResponse, fact_set_sha256: str) -> HttpResponse:
     response.headers["X-Publication-Fact-Set"] = fact_set_sha256
-    response.headers["Cache-Control"] = "public, max-age=60, stale-if-error=3600"
     return response
 
 
+def _query_error(form: SearchForm) -> str:
+    errors = form.errors.as_data().get("q", [])
+    if not errors:
+        return ""
+    return _QUERY_ERROR_MESSAGES.get(errors[0].code or "", "검색어를 확인해 주세요.")
+
+
 def _qa_results() -> list[dict[str, str]]:
     return [
         {


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


