## `feat(public): connect recent catalog extensions`

diff --git a/grocery/views.py b/grocery/views.py
index 15d0d55..2ab6e62 100644
--- a/grocery/views.py
+++ b/grocery/views.py
@@ -4,8 +4,9 @@ from __future__ import annotations
 
 import logging
 import uuid
+from collections.abc import Mapping
 from decimal import Decimal
-from typing import Final
+from typing import Any, Final
 from urllib.parse import urlencode
 
 from django.conf import settings
@@ -16,10 +17,22 @@ from django.shortcuts import render
 from django.urls import reverse
 from django.views.decorators.http import require_safe
 
-from grocery.forms import QUERY_MAX_LENGTH, SearchForm
+from grocery.forms import (
+    CATEGORY_CHOICES,
+    DIRECTION_CHOICES,
+    PERIOD_CHOICES,
+    QUERY_MAX_LENGTH,
+    SORT_CHOICES,
+    CatalogForm,
+)
+from grocery.historical_public_read import (
+    historical_series_for_recent,
+    load_active_historical_publication,
+)
 from grocery.observability import log_event
 from grocery.presentation import comparison_microbar
 from grocery.public_read import (
+    PUBLIC_PAGE_SIZE,
     catalog_item,
     detail_context,
     load_active_publication,
@@ -51,29 +64,49 @@ _QA_STATE_MESSAGES: Final = {
 
 @require_safe
 def catalog(request: HttpRequest) -> HttpResponse:
-    form = SearchForm(request.GET if request.GET else None)
-    query = ""
-    category = ""
+    form = CatalogForm(request.GET if request.GET else None)
     if form.is_bound and not form.is_valid():
-        query_error = _query_error(form)
-        category_error = "부류 선택을 확인해 주세요." if "category" in form.errors else ""
-        context = _catalog_base_context(category="")
+        context = _catalog_base_context(category="", period="week", direction="all", sort="name")
         context.update(
             {
                 "catalog_state": "validation",
-                "query_error": query_error,
-                "category_error": category_error,
+                "query_error": _query_error(form),
+                "category_error": (
+                    "부류 선택을 확인해 주세요." if "category" in form.errors else ""
+                ),
+                "period_error": "period" in form.errors,
+                "direction_error": "direction" in form.errors,
+                "sort_error": "sort" in form.errors,
+                "filters_open": True,
+                "validation_errors": _validation_errors(
+                    form,
+                    {
+                        "q": "catalog-query",
+                        "period": "catalog-period",
+                        "direction": "catalog-direction",
+                        "sort": "catalog-sort",
+                    },
+                ),
                 "results": [],
             }
         )
         return render(request, "grocery/catalog.html", context, status=400)
-    if form.is_valid():
-        query = form.cleaned_data["q"]
-        category = form.cleaned_data["category"]
+    cleaned = form.cleaned_data if form.is_bound else {}
+    query = str(cleaned.get("q", ""))
+    category = str(cleaned.get("category", ""))
+    period = str(cleaned.get("period", "week"))
+    direction = str(cleaned.get("direction", "all"))
+    sort = str(cleaned.get("sort", "name"))
+    page = int(cleaned.get("page", 1))
 
     try:
         active = load_active_publication()
-        context = _catalog_base_context(category=category)
+        context = _catalog_base_context(
+            category=category,
+            period=period,
+            direction=direction,
+            sort=sort,
+        )
         if active is None:
             context.update(
                 {
@@ -84,7 +117,21 @@ def catalog(request: HttpRequest) -> HttpResponse:
             )
             return render(request, "grocery/catalog.html", context)
 
-        entries = list(publication_entries(active, query=query, category=category))
+        filtered_entries = publication_entries(
+            active,
+            query=query,
+            category=category,
+            period=period,
+            direction=direction,
+            sort=sort,
+        )
+        start = (page - 1) * PUBLIC_PAGE_SIZE
+        entries = (
+            filtered_entries[:PUBLIC_PAGE_SIZE]
+            if query
+            else filtered_entries[start : start + PUBLIC_PAGE_SIZE]
+        )
+        has_next = not query and len(filtered_entries) > start + PUBLIC_PAGE_SIZE
         results = [
             catalog_item(
                 entry,
@@ -93,6 +140,8 @@ def catalog(request: HttpRequest) -> HttpResponse:
             )
             for entry in entries
         ]
+        for item, entry in zip(results, entries, strict=True):
+            item["selection_url"] = _selection_url((entry.snapshot.series_id,))
         context.update(
             {
                 "catalog_state": active.freshness_state if active.stale_message else "ready",
@@ -102,8 +151,22 @@ def catalog(request: HttpRequest) -> HttpResponse:
                     else "품목명을 바꾸거나 다른 부류를 선택하세요."
                 ),
                 "results": results,
-                "result_count_label": f"공개 항목 {len(results)}개",
+                "result_count_label": f"현재 페이지 {len(results)}개",
                 "publication": publication_context(active),
+                "pagination": _pagination_context(
+                    base_url=reverse("grocery:catalog"),
+                    page=page,
+                    has_next=has_next,
+                    parameters=_catalog_parameters(
+                        category=category,
+                        period=period,
+                        direction=direction,
+                        sort=sort,
+                    ),
+                )
+                if not query
+                else None,
+                "selection_url": reverse("grocery:selection"),
             }
         )
         response = render(request, "grocery/catalog.html", context)
@@ -138,16 +201,38 @@ def detail(request: HttpRequest, series_id: uuid.UUID) -> HttpResponse:
         )
         if entry is None:
             raise Http404
+        historical = None
+        historical_series = None
+        try:
+            historical = load_active_historical_publication()
+            historical_series = (
+                historical_series_for_recent(historical, series_id)
+                if historical is not None
+                else None
+            )
+        except DatabaseError, ObjectDoesNotExist, ValidationError:
+            log_event(_LOGGER, "ERROR", "public.detail.history_hidden")
+            historical = None
+        links = _historical_links(series_id) if historical_series is not None else {}
         context = {
             "home_url": reverse("grocery:catalog"),
             "catalog_url": reverse("grocery:catalog"),
+            "selection_url": reverse("grocery:selection"),
+            "selection_add_url": _selection_url((series_id,)),
             "detail_state": active.freshness_state if active.stale_message else "ready",
             "status_message": active.stale_message,
             "publication": publication_context(active),
+            "historical_links": links,
+            "section_nav": _section_navigation(series_id, current="detail", historical=bool(links)),
             **detail_context(entry, active),
         }
         response = render(request, "grocery/detail.html", context)
-        return _publication_response(response, active.revision.typed_fact_set_sha256)
+        response = _publication_response(response, active.revision.typed_fact_set_sha256)
+        if historical is not None and historical_series is not None:
+            response = _historical_publication_response(
+                response, historical.revision.typed_fact_set_sha256
+            )
+        return response
     except Http404:
         raise
     except (DatabaseError, ObjectDoesNotExist, ValidationError):  # fmt: skip
@@ -155,6 +240,7 @@ def detail(request: HttpRequest, series_id: uuid.UUID) -> HttpResponse:
         context = {
             "home_url": reverse("grocery:catalog"),
             "catalog_url": reverse("grocery:catalog"),
+            "selection_url": reverse("grocery:selection"),
             "detail_state": "server_error",
             "status_message": "잠시 후 다시 불러오세요.",
             "retry_url": request.path,
@@ -220,28 +306,140 @@ def qa_detail_state(request: HttpRequest, state: str) -> HttpResponse:
     )
 
 
-def _catalog_base_context(*, category: str) -> dict[str, object]:
+def _catalog_base_context(
+    *, category: str, period: str = "week", direction: str = "all", sort: str = "name"
+) -> dict[str, object]:
     catalog_url = reverse("grocery:catalog")
+    parameters = _catalog_parameters(category="", period=period, direction=direction, sort=sort)
+    period_labels = dict(PERIOD_CHOICES)
+    sort_labels = dict(SORT_CHOICES)
     return {
         "home_url": catalog_url,
         "form_action": catalog_url,
+        "selection_url": reverse("grocery:selection"),
         "selected_category": category,
+        "selected_period": period,
+        "selected_direction": direction,
+        "selected_sort": sort,
+        "selected_period_label": period_labels[period],
+        "selected_period_heading": period_labels[period],
+        "selected_period_missing_label": f"{period_labels[period]}값 없음",
+        "selected_sort_label": sort_labels[sort],
+        "filters_open": period != "week" or direction != "all" or sort != "name",
+        "period_options": _choice_options(PERIOD_CHOICES, period),
+        "direction_options": _choice_options(DIRECTION_CHOICES, direction),
+        "sort_options": _choice_options(SORT_CHOICES, sort),
         "categories": [
             {
                 "label": label,
-                "url": _category_url(catalog_url, category=value),
+                "url": _url(
+                    catalog_url,
+                    {**parameters, **({"category": value} if value else {})},
+                ),
                 "selected": category == value,
             }
-            for value, label in (("", "전체"), ("vegetable", "채소류"), ("fruit", "과일류"))
+            for value, label in CATEGORY_CHOICES
         ],
     }
 
 
-def _category_url(base_url: str, *, category: str) -> str:
+def _choice_options(choices: tuple[tuple[str, str], ...], selected: str) -> list[dict[str, object]]:
+    return [
+        {"value": value, "label": label, "selected": value == selected} for value, label in choices
+    ]
+
+
+def _catalog_parameters(*, category: str, period: str, direction: str, sort: str) -> dict[str, str]:
     parameters = {}
     if category:
         parameters["category"] = category
-    return f"{base_url}?{urlencode(parameters)}" if parameters else base_url
+    if period != "week":
+        parameters["period"] = period
+    if direction != "all":
+        parameters["direction"] = direction
+    if sort != "name":
+        parameters["sort"] = sort
+    return parameters
+
+
+def _url(base_url: str, parameters: dict[str, object]) -> str:
+    return f"{base_url}?{urlencode(parameters, doseq=True)}" if parameters else base_url
+
+
+def _selection_url(series_ids: tuple[uuid.UUID, ...]) -> str:
+    base = reverse("grocery:selection")
+    return _url(base, {"series": [str(series_id) for series_id in series_ids]})
+
+
+def _pagination_context(
+    *,
+    base_url: str,
+    page: int,
+    has_next: bool,
+    parameters: Mapping[str, object],
+) -> dict[str, object]:
+    def page_url(value: int) -> str:
+        values: dict[str, object] = dict(parameters)
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
+
+
+def _historical_links(series_id: uuid.UUID) -> dict[str, str]:
+    return {
+        "history_url": reverse("grocery:history", kwargs={"series_id": series_id}),
+        "regions_url": reverse("grocery:regions", kwargs={"series_id": series_id}),
+    }
+
+
+def _section_navigation(
+    series_id: uuid.UUID, *, current: str, historical: bool
+) -> list[dict[str, object]]:
+    links = [
+        {
+            "label": "최근 조사값",
+            "url": reverse("grocery:detail", kwargs={"series_id": series_id}),
+            "current": current == "detail",
+            "available": True,
+        }
+    ]
+    if historical:
+        links.extend(
+            [
+                {
+                    "label": "월별 기록",
+                    "url": reverse("grocery:history", kwargs={"series_id": series_id}),
+                    "current": current == "history",
+                    "available": True,
+                },
+                {
+                    "label": "지역별 조사값",
+                    "url": reverse("grocery:regions", kwargs={"series_id": series_id}),
+                    "current": current == "regions",
+                    "available": True,
+                },
+            ]
+        )
+    return links
+
+
+def _validation_errors(form: Any, targets: dict[str, str]) -> list[dict[str, str]]:
+    errors: list[dict[str, str]] = []
+    for field_name, field_errors in form.errors.as_data().items():
+        target = targets.get(field_name, "")
+        for error in field_errors:
+            message = str(error.message)
+            if "%" in message and error.params:
+                message %= error.params
+            errors.append({"message": message, "target": target})
+    return errors or [{"message": "요청 내용을 확인하세요.", "target": ""}]
 
 
 def _publication_response(response: HttpResponse, fact_set_sha256: str) -> HttpResponse:
@@ -249,7 +447,12 @@ def _publication_response(response: HttpResponse, fact_set_sha256: str) -> HttpR
     return response
 
 
-def _query_error(form: SearchForm) -> str:
+def _historical_publication_response(response: HttpResponse, fact_set_sha256: str) -> HttpResponse:
+    response.headers["X-Historical-Publication-Fact-Set"] = fact_set_sha256
+    return response
+
+
+def _query_error(form: CatalogForm) -> str:
     errors = form.errors.as_data().get("q", [])
     if not errors:
         return ""


## `test(public): enforce bounded publication reads`

diff --git a/grocery/tests/test_vnext_public_query_bounds.py b/grocery/tests/test_vnext_public_query_bounds.py
new file mode 100644
index 0000000..1c7d03b
--- /dev/null
+++ b/grocery/tests/test_vnext_public_query_bounds.py
@@ -0,0 +1,163 @@
+import uuid
+
+import pytest
+from django.contrib.auth.models import Permission
+from django.core.exceptions import ValidationError
+from django.db import connection
+from django.test.utils import CaptureQueriesContext
+from django.utils import timezone
+
+from grocery.historical_daily_read import markets_context, regions_context
+from grocery.historical_history_read import history_context
+from grocery.historical_public_read import ActiveHistoricalPublication
+from grocery.historical_publications import seal_historical_publication
+from grocery.models import (
+    PriceChangeFact,
+    PublicationActivation,
+    seal_recent_publication,
+    transition_recent_publication,
+)
+from grocery.public_read import (
+    _filter_and_sort_catalog_entries,
+    load_active_publication,
+    publication_entries,
+    publication_entries_for_series,
+)
+from grocery.tests.historical_bundle_factory import create_reviewed_historical_bundle
+from grocery.tests.test_publication_revision_models import create_approved_generation
+
+
+def _activate_recent(count: int):
+    decision, snapshots, publisher = create_approved_generation(snapshot_count=count)
+    publisher.user_permissions.add(
+        Permission.objects.get(content_type__app_label="grocery", codename="publish_publication")
+    )
+    publisher = type(publisher)._default_manager.get(pk=publisher.pk)
+    revision = seal_recent_publication(decision.id, "ko-v4")
+    transition_recent_publication(
+        operation_id=uuid.uuid4(),
+        actor=publisher,
+        operation=PublicationActivation.Operation.ACTIVATE,
+        target_revision_id=revision.id,
+        expected_current_revision_id=None,
+        expected_version=0,
+        reason_code="LOCAL_VNEXT_QUERY_TEST",
+        acceptance_evidence_sha256="a" * 64,
+    )
+    active = load_active_publication()
+    assert active is not None
+    return active, snapshots
+
+
+def _sealed_historical():
+    bundle = create_reviewed_historical_bundle()
+    revision = seal_historical_publication(
+        monthly_review_id=bundle.monthly_review.id,
+        regional_review_id=bundle.regional_review.id,
+        market_review_id=bundle.market_review.id,
+        compatibility_report_sha256="b" * 64,
+    )
+    # Warm only publication metadata; measured calls must account for fact queries.
+    _collections = (
+        revision.monthly_review.collection,
+        revision.regional_review.collection,
+        revision.market_review.collection,
+    )
+    active = ActiveHistoricalPublication(
+        revision=revision,
+        checked_at=timezone.now(),
+        freshness_state="current",
+        freshness_label="KAMIS 자료 확인 완료",
+        stale_message="",
+    )
+    return active, bundle
+
+
+@pytest.mark.django_db
+def test_catalog_materialization_is_two_queries_for_one_or_thirty_rows() -> None:
+    active, _snapshots = _activate_recent(30)
+
+    with CaptureQueriesContext(connection) as one:
+        publication_entries(active, query="품목 212", category="")
+    with CaptureQueriesContext(connection) as thirty:
+        publication_entries(active, query="", category="")
+
+    assert len(one) == len(thirty) == 2
+
+
+@pytest.mark.django_db
+def test_catalog_materialization_rejects_missing_duplicate_and_incomplete_selected_facts() -> None:
+    active, _snapshots = _activate_recent(2)
+    entries = publication_entries(active, query="", category="")
+    entry = entries[0]
+    reference = entry.snapshot.catalog_week_references[0]
+
+    for malformed in ([], [reference, reference]):
+        entry.snapshot.catalog_week_references = malformed
+        with pytest.raises(ValidationError):
+            _filter_and_sort_catalog_entries([entry], direction="all", sort="name")
+
+    entry.snapshot.catalog_week_references = [reference]
+    original_direction = reference.change_fact.direction
+    original_percentage = reference.change_fact.signed_percentage
+    reference.change_fact.direction = PriceChangeFact.Direction.HIGHER
+    reference.change_fact.signed_percentage = None
+    with pytest.raises(ValidationError):
+        _filter_and_sort_catalog_entries([entry], direction="all", sort="name")
+    reference.change_fact.direction = original_direction
+    reference.change_fact.signed_percentage = original_percentage
+
+    entries[1].snapshot.catalog_week_references = []
+    with pytest.raises(ValidationError):
+        _filter_and_sort_catalog_entries(
+            entries,
+            query=entries[0].snapshot.series.item_name,
+            direction="all",
+            sort="name",
+        )
+
+
+@pytest.mark.django_db
+def test_history_and_daily_ledgers_have_row_independent_query_counts() -> None:
+    active, bundle = _sealed_historical()
+
+    with CaptureQueriesContext(connection) as twelve:
+        history_context(
+            active,
+            bundle.series,
+            selected_region_id=bundle.region.id,
+            selected_range=12,
+        )
+    with CaptureQueriesContext(connection) as thirty_six:
+        history_context(
+            active,
+            bundle.series,
+            selected_region_id=bundle.region.id,
+            selected_range=36,
+        )
+    with CaptureQueriesContext(connection) as regions:
+        regions_context(active, bundle.series, selected_date=None)
+    with CaptureQueriesContext(connection) as markets:
+        markets_context(
+            active,
+            bundle.series,
+            region_id=bundle.region.id,
+            selected_date=None,
+            page=1,
+        )
+
+    assert len(twelve) == len(thirty_six) == 1
+    assert len(regions) == 2
+    assert len(markets) == 2
+
+
+@pytest.mark.django_db
+def test_selection_reference_queries_do_not_grow_from_one_to_five_items() -> None:
+    active, snapshots = _activate_recent(5)
+
+    with CaptureQueriesContext(connection) as one:
+        publication_entries_for_series(active, [snapshots[0].series_id])
+    with CaptureQueriesContext(connection) as five:
+        publication_entries_for_series(active, [snapshot.series_id for snapshot in snapshots])
+
+    assert len(one) == len(five) == 2
