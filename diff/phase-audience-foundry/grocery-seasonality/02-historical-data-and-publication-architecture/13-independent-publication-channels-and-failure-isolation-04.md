## `feat(public): validate active historical bundle`

diff --git a/config/settings.py b/config/settings.py
index 8da0f19..1f6f023 100644
--- a/config/settings.py
+++ b/config/settings.py
@@ -232,7 +232,7 @@ validate_hsts_configuration(
     preload=SECURE_HSTS_PRELOAD,
 )
 SECURE_CONTENT_TYPE_NOSNIFF = True
-SECURE_REFERRER_POLICY = "same-origin"
+SECURE_REFERRER_POLICY = "no-referrer"
 X_FRAME_OPTIONS = "DENY"
 
 KAMIS_CONFIRMATION_MAX_AGE_HOURS = env_positive_int(
@@ -240,6 +240,16 @@ KAMIS_CONFIRMATION_MAX_AGE_HOURS = env_positive_int(
     36,
     maximum=168,
 )
+KAMIS_HISTORICAL_MONTHLY_MAX_AGE_HOURS = env_positive_int(
+    "KAMIS_HISTORICAL_MONTHLY_MAX_AGE_HOURS",
+    192,
+    maximum=744,
+)
+KAMIS_HISTORICAL_DAILY_MAX_AGE_HOURS = env_positive_int(
+    "KAMIS_HISTORICAL_DAILY_MAX_AGE_HOURS",
+    36,
+    maximum=168,
+)
 QA_STATE_PREVIEWS_ENABLED = DEBUG and env_bool("QA_STATE_PREVIEWS_ENABLED", False)
 
 LOGGING = {
diff --git a/grocery/historical_public_read.py b/grocery/historical_public_read.py
new file mode 100644
index 0000000..b31980a
--- /dev/null
+++ b/grocery/historical_public_read.py
@@ -0,0 +1,197 @@
+"""Active historical publication and exact recent-series membership boundary."""
+
+from __future__ import annotations
+
+import re
+import uuid
+from dataclasses import dataclass
+from datetime import datetime, timedelta
+from typing import Final, cast
+
+from django.conf import settings
+from django.core.exceptions import ValidationError
+from django.utils import timezone
+
+from grocery.historical_activation_models import HistoricalRetailPublicationChannel
+from grocery.historical_collection_models import HistoricalSourceCollection
+from grocery.historical_daily_models import DailyMarketRetailPrice, DailyRegionalRetailPrice
+from grocery.historical_identity_models import HistoricalRetailSeriesKey
+from grocery.historical_monthly_models import MonthlyRegionalRetailPrice
+from grocery.historical_publication_models import HistoricalRetailPublicationRevision
+from grocery.historical_review_models import HistoricalCollectionReviewDecision
+from grocery.presentation import format_korean_datetime, format_unit
+
+HISTORICAL_RETAIL_CHANNEL: Final = "HISTORICAL_RETAIL"
+_SHA256_RE: Final = re.compile(r"^[0-9a-f]{64}$")
+_KIND_REVIEW_FIELDS: Final = {
+    HistoricalSourceCollection.Kind.MONTHLY: "monthly_review",
+    HistoricalSourceCollection.Kind.REGIONAL_DAILY: "regional_review",
+    HistoricalSourceCollection.Kind.MARKET_DAILY: "market_review",
+}
+
+
+class PublicReadIntegrityError(ValidationError):
+    """An active publication cannot be presented without inventing facts."""
+
+
+class PublicParameterError(ValidationError):
+    """A syntactically valid public value is not in the active allowlist."""
+
+
+@dataclass(frozen=True, slots=True)
+class ActiveHistoricalPublication:
+    revision: HistoricalRetailPublicationRevision
+    checked_at: datetime
+    freshness_state: str
+    freshness_label: str
+    stale_message: str
+
+    @property
+    def monthly_collection(self) -> HistoricalSourceCollection:
+        return self.revision.monthly_review.collection
+
+    @property
+    def regional_collection(self) -> HistoricalSourceCollection:
+        return self.revision.regional_review.collection
+
+    @property
+    def market_collection(self) -> HistoricalSourceCollection:
+        return self.revision.market_review.collection
+
+
+def load_active_historical_publication(
+    *, observed_at: datetime | None = None
+) -> ActiveHistoricalPublication | None:
+    """Load and validate the historical pointer independently of recent retail."""
+
+    channel = (
+        HistoricalRetailPublicationChannel.objects.select_related(
+            "current_revision__monthly_review__collection__source_configuration",
+            "current_revision__regional_review__collection__source_configuration",
+            "current_revision__market_review__collection__source_configuration",
+        )
+        .filter(pk=HISTORICAL_RETAIL_CHANNEL)
+        .first()
+    )
+    if channel is None or channel.current_revision is None:
+        return None
+    revision = channel.current_revision
+    if (
+        revision.sealed_at is None
+        or revision.public_copy_revision != HistoricalRetailPublicationRevision.COPY_REVISION
+        or not _SHA256_RE.fullmatch(revision.typed_fact_set_sha256)
+    ):
+        raise PublicReadIntegrityError("The historical pointer is not a sealed public revision.")
+
+    collections: list[HistoricalSourceCollection] = []
+    for expected_kind, review_field in _KIND_REVIEW_FIELDS.items():
+        decision = getattr(revision, review_field)
+        collection = decision.collection
+        if (
+            decision.decision != HistoricalCollectionReviewDecision.Decision.APPROVE
+            or collection.kind != expected_kind
+            or collection.state != HistoricalSourceCollection.State.VALIDATED
+            or collection.completed_at is None
+            or collection.code_manifest_sha256 != revision.code_manifest_sha256
+            or decision.approved_result_sha256 != collection.result_sha256
+            or decision.approved_partition_manifest_sha256 != collection.partition_manifest_sha256
+        ):
+            raise PublicReadIntegrityError(
+                "The historical revision has an invalid reviewed source."
+            )
+        collections.append(collection)
+
+    now = observed_at or timezone.now()
+    monthly, regional, market = collections
+    monthly_completed = cast(datetime, monthly.completed_at)
+    regional_completed = cast(datetime, regional.completed_at)
+    market_completed = cast(datetime, market.completed_at)
+    monthly_age = timedelta(hours=settings.KAMIS_HISTORICAL_MONTHLY_MAX_AGE_HOURS)
+    daily_age = timedelta(hours=settings.KAMIS_HISTORICAL_DAILY_MAX_AGE_HOURS)
+    stale = bool(
+        now - monthly_completed > monthly_age
+        or now - regional_completed > daily_age
+        or now - market_completed > daily_age
+    )
+    completed_by_source = {
+        collection.source_configuration_id: cast(datetime, collection.completed_at)
+        for collection in collections
+    }
+    newer_outcome_exists = any(
+        completed_at is not None and completed_at > completed_by_source[source_configuration_id]
+        for source_configuration_id, completed_at in HistoricalSourceCollection.objects.filter(
+            source_configuration_id__in=completed_by_source,
+            completed_at__isnull=False,
+        )
+        .exclude(id__in={collection.id for collection in collections})
+        .values_list("source_configuration_id", "completed_at")
+    )
+    stale = stale or newer_outcome_exists
+    checked_at = min(monthly_completed, regional_completed, market_completed)
+    if stale:
+        return ActiveHistoricalPublication(
+            revision=revision,
+            checked_at=checked_at,
+            freshness_state="stale",
+            freshness_label="마지막 공개 자료 · 최근 확인 필요",
+            stale_message=(
+                "최근 자료 확인이 필요합니다. 마지막으로 검토를 마친 조사값을 표시합니다."
+            ),
+        )
+    return ActiveHistoricalPublication(
+        revision=revision,
+        checked_at=checked_at,
+        freshness_state="current",
+        freshness_label="KAMIS 자료 확인 완료",
+        stale_message="",
+    )
+
+
+def historical_publication_context(active: ActiveHistoricalPublication) -> dict[str, str]:
+    return {
+        "checked_at_iso": active.checked_at.isoformat(),
+        "checked_at_display": format_korean_datetime(active.checked_at),
+        "freshness_state": active.freshness_state,
+        "freshness_label": active.freshness_label,
+    }
+
+
+def historical_series_for_recent(
+    active: ActiveHistoricalPublication, recent_series_id: uuid.UUID
+) -> HistoricalRetailSeriesKey | None:
+    series = (
+        HistoricalRetailSeriesKey.objects.select_related("recent_series")
+        .filter(recent_series_id=recent_series_id)
+        .first()
+    )
+    if series is None:
+        return None
+    if series.code_manifest_sha256 != active.revision.code_manifest_sha256:
+        raise PublicReadIntegrityError("Historical series identity uses a different code manifest.")
+    memberships = (
+        MonthlyRegionalRetailPrice.objects.filter(
+            collection=active.monthly_collection, series=series
+        ).exists(),
+        DailyRegionalRetailPrice.objects.filter(
+            collection=active.regional_collection, series=series
+        ).exists(),
+        DailyMarketRetailPrice.objects.filter(
+            collection=active.market_collection, series=series
+        ).exists(),
+    )
+    if len(set(memberships)) != 1:
+        raise PublicReadIntegrityError("Historical series membership is incomplete.")
+    if not all(memberships):
+        return None
+    return series
+
+
+def historical_series_context(series: HistoricalRetailSeriesKey) -> dict[str, str]:
+    recent = series.recent_series
+    return {
+        "category_label": recent.category_name,
+        "item_name": recent.item_name,
+        "variety_name": recent.variety_name,
+        "grade_name": recent.grade_name,
+        "unit_label": format_unit(recent.raw_unit, recent.raw_unit_size),
+    }


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
