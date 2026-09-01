## `feat(web): render transient opportunity searches`

diff --git a/public_web/forms.py b/public_web/forms.py
index 8171fb3..c42d707 100644
--- a/public_web/forms.py
+++ b/public_web/forms.py
@@ -1,68 +1,108 @@
+from datetime import datetime, timedelta
+from zoneinfo import ZoneInfo
+
 from django import forms
+from django.utils import timezone
+
+
+SEOUL = ZoneInfo("Asia/Seoul")
+MAXIMUM_SEARCH_HORIZON = timedelta(days=7)
+DATETIME_LOCAL_FORMAT = "%Y-%m-%dT%H:%M"
+
+
+def _now_kst() -> datetime:
+    return timezone.now().astimezone(SEOUL)
+
+
+class _KSTDateTimeField(forms.DateTimeField):
+    def to_python(self, value):
+        if value in self.empty_values:
+            return None
+        if isinstance(value, datetime):
+            parsed = value
+        elif isinstance(value, str):
+            try:
+                parsed = datetime.strptime(value, DATETIME_LOCAL_FORMAT)
+            except (TypeError, ValueError) as exc:
+                raise forms.ValidationError(
+                    self.error_messages["invalid"],
+                    code="invalid",
+                ) from exc
+        else:
+            raise forms.ValidationError(
+                self.error_messages["invalid"],
+                code="invalid",
+            )
+        if timezone.is_aware(parsed):
+            return parsed.astimezone(SEOUL)
+        return timezone.make_aware(parsed, SEOUL)
 
 
 class TripForm(forms.Form):
-    destination = forms.ChoiceField(
-        label="목적지",
-        choices=(("JP", "일본"),),
-        error_messages={
-            "required": "목적지를 선택해 주세요.",
-            "invalid_choice": "목적지는 일본만 선택할 수 있습니다.",
-        },
-        widget=forms.Select(
-            attrs={
-                "autocomplete": "off",
-                "aria-describedby": "id_destination_hint",
-            }
-        ),
-    )
-    departure_date = forms.DateField(
-        label="출국일",
-        input_formats=("%Y-%m-%d",),
-        widget=forms.DateInput(
-            format="%Y-%m-%d",
+    departure_at = _KSTDateTimeField(
+        label="출발 가능한 시각",
+        widget=forms.DateTimeInput(
+            format=DATETIME_LOCAL_FORMAT,
             attrs={
-                "type": "date",
+                "type": "datetime-local",
+                "step": "60",
                 "autocomplete": "off",
-                "aria-describedby": "id_departure_date_hint",
+                "aria-describedby": "id_departure_at_hint",
             },
         ),
         error_messages={
-            "required": "출국일을 입력해 주세요.",
-            "invalid": "출국일을 날짜 형식으로 입력해 주세요.",
+            "required": "출발 가능한 시각을 입력해 주세요.",
+            "invalid": "출발 가능한 시각을 날짜와 시각 형식으로 입력해 주세요.",
         },
     )
-    return_date = forms.DateField(
-        label="귀국일",
-        input_formats=("%Y-%m-%d",),
-        widget=forms.DateInput(
-            format="%Y-%m-%d",
+    return_by = _KSTDateTimeField(
+        label="인천 도착 마감 시각",
+        widget=forms.DateTimeInput(
+            format=DATETIME_LOCAL_FORMAT,
             attrs={
-                "type": "date",
+                "type": "datetime-local",
+                "step": "60",
                 "autocomplete": "off",
-                "aria-describedby": "id_return_date_hint",
+                "aria-describedby": "id_return_by_hint",
             },
         ),
         error_messages={
-            "required": "귀국일을 입력해 주세요.",
-            "invalid": "귀국일을 날짜 형식으로 입력해 주세요.",
+            "required": "인천 도착 마감 시각을 입력해 주세요.",
+            "invalid": "인천 도착 마감 시각을 날짜와 시각 형식으로 입력해 주세요.",
         },
     )
 
     def clean(self):
         cleaned = super().clean()
-        departure = cleaned.get("departure_date")
-        return_date = cleaned.get("return_date")
+        departure_at = cleaned.get("departure_at")
+        return_by = cleaned.get("return_by")
+        current = _now_kst()
+        if departure_at is not None and departure_at <= current:
+            self.add_error(
+                "departure_at",
+                forms.ValidationError(
+                    "출발 가능한 시각은 현재보다 늦게 입력해 주세요.",
+                    code="departure_not_future",
+                ),
+            )
         if (
-            departure is not None
-            and return_date is not None
-            and return_date < departure
+            departure_at is not None
+            and return_by is not None
+            and return_by <= departure_at
         ):
             self.add_error(
-                "return_date",
+                "return_by",
+                forms.ValidationError(
+                    "인천 도착 마감 시각은 출발 가능한 시각보다 늦어야 합니다.",
+                    code="window_order",
+                ),
+            )
+        if return_by is not None and return_by > current + MAXIMUM_SEARCH_HORIZON:
+            self.add_error(
+                "return_by",
                 forms.ValidationError(
-                    "귀국일은 출국일과 같거나 그 이후여야 합니다.",
-                    code="date_order",
+                    "인천 도착 마감 시각은 현재부터 7일 이내로 입력해 주세요.",
+                    code="window_too_long",
                 ),
             )
         return cleaned
diff --git a/public_web/results.py b/public_web/results.py
index 9e8552a..967a2fd 100644
--- a/public_web/results.py
+++ b/public_web/results.py
@@ -3,10 +3,10 @@ from __future__ import annotations
 from dataclasses import dataclass
 from datetime import UTC, date, datetime, timedelta
 from typing import Callable
+from uuid import UUID
 
 from django.db.models import Exists, OuterRef
 from django.http import HttpRequest, HttpResponse
-from django.shortcuts import render
 from django.urls import reverse
 from django.utils import timezone
 from django.views.decorators.http import require_GET
@@ -22,9 +22,8 @@ from entry_requirements.ingestion import (
     ENTRY_SOURCE_OWNER,
     ENTRY_SOURCE_REVISION,
 )
+from countries.models import SUPPORTED_COUNTRY_IDS
 from reviews.models import (
-    ENTRY_POINTER_ID,
-    WARNING_POINTER_ID,
     PublishedEntryFacts,
     PublishedTravelWarning,
 )
@@ -349,9 +348,9 @@ def _freshness(
     return (CARD_STALE if stale else CARD_READY), checked_at
 
 
-def _entry_row() -> dict | None:
+def _entry_row(country_id: UUID) -> dict | None:
     return (
-        PublishedEntryFacts.objects.filter(id=ENTRY_POINTER_ID)
+        PublishedEntryFacts.objects.filter(country_id=country_id)
         .values(
             "current_publication_id",
             "current_publication__generation",
@@ -380,9 +379,9 @@ def _entry_row() -> dict | None:
     )
 
 
-def _warning_row() -> dict | None:
+def _warning_row(country_id: UUID) -> dict | None:
     return (
-        PublishedTravelWarning.objects.filter(id=WARNING_POINTER_ID)
+        PublishedTravelWarning.objects.filter(country_id=country_id)
         .values(
             "current_publication_id",
             "current_publication__generation",
@@ -412,8 +411,8 @@ def _warning_row() -> dict | None:
     )
 
 
-def _load_entry_card() -> dict[str, object]:
-    row = _entry_row()
+def _load_entry_card(country_id: UUID) -> dict[str, object]:
+    row = _entry_row(country_id)
     if row is None:
         return _state_card("entry", CARD_UNAVAILABLE)
     if row["current_publication_id"] is None:
@@ -479,8 +478,8 @@ def _load_entry_card() -> dict[str, object]:
     }
 
 
-def _load_warning_card() -> dict[str, object]:
-    row = _warning_row()
+def _load_warning_card(country_id: UUID) -> dict[str, object]:
+    row = _warning_row(country_id)
     if row is None:
         return _state_card("warning", CARD_UNAVAILABLE)
     if row["current_publication_id"] is None:
@@ -561,35 +560,25 @@ def _safe_card(
         return _state_card(module, CARD_SERVER_ERROR)
 
 
-def load_publication_cards() -> dict[str, dict[str, object]]:
-    """Return the two fixed, independent read models without external I/O."""
+def load_publication_cards_for_country(
+    country_id: UUID,
+) -> dict[str, dict[str, object]]:
+    """Return independent country-scoped read models without external I/O."""
 
-    cards = {
-        "entry": _safe_card("entry", _load_entry_card),
-        "warning": _safe_card("warning", _load_warning_card),
+    return {
+        "entry": _safe_card("entry", lambda: _load_entry_card(country_id)),
+        "warning": _safe_card(
+            "warning", lambda: _load_warning_card(country_id)
+        ),
     }
-    publication_states = {CARD_READY, CARD_STALE}
-    entry_state = cards["entry"].get("state")
-    warning_state = cards["warning"].get("state")
-    if entry_state in publication_states and warning_state == CARD_EMPTY:
-        cards["warning"] = _state_card("warning", CARD_UNAVAILABLE)
-    elif warning_state in publication_states and entry_state == CARD_EMPTY:
-        cards["entry"] = _state_card("entry", CARD_UNAVAILABLE)
-    return cards
+
+
+def load_publication_cards() -> dict[str, dict[str, object]]:
+    """Backward-compatible Japanese card loader for internal checks."""
+
+    return load_publication_cards_for_country(SUPPORTED_COUNTRY_IDS["JP"])
 
 
 @require_GET
 def results(request: HttpRequest) -> HttpResponse:
-    if request.GET:
-        return _fixed_redirect("public_web:results")
-    cards = load_publication_cards()
-    response = render(
-        request,
-        "public_web/results.html",
-        {
-            "entry_card": cards["entry"],
-            "warning_card": cards["warning"],
-        },
-    )
-    response.headers["Cache-Control"] = "no-store"
-    return response
+    return _fixed_redirect("public_web:index")
diff --git a/public_web/tests/test_travel_opportunity_web.py b/public_web/tests/test_travel_opportunity_web.py
new file mode 100644
index 0000000..e056c09
--- /dev/null
+++ b/public_web/tests/test_travel_opportunity_web.py
@@ -0,0 +1,280 @@
+from datetime import date, datetime
+from unittest.mock import patch
+from uuid import UUID
+from zoneinfo import ZoneInfo
+
+from django.http import HttpResponse
+from django.test import SimpleTestCase
+from django.urls import reverse
+
+from countries.models import SUPPORTED_COUNTRY_IDS
+from public_web.forms import TripForm
+from public_web.results import (
+    CARD_READY,
+    CARD_SERVER_ERROR,
+    _load_entry_card,
+    _load_warning_card,
+    load_publication_cards_for_country,
+)
+
+
+SEOUL = ZoneInfo("Asia/Seoul")
+NOW = datetime(2026, 8, 31, 9, tzinfo=SEOUL)
+
+
+class TravelWindowFormTests(SimpleTestCase):
+    def _form(self, *, departure_at="2026-08-31T10:00", return_by="2026-09-07T09:00"):
+        return TripForm(
+            {
+                "departure_at": departure_at,
+                "return_by": return_by,
+            }
+        )
+
+    @patch("public_web.forms._now_kst", return_value=NOW)
+    def test_datetime_local_values_are_kst_and_seven_days_is_inclusive(self, _now):
+        form = self._form()
+
+        self.assertTrue(form.is_valid(), form.errors)
+        self.assertEqual(form.cleaned_data["departure_at"].tzinfo, SEOUL)
+        self.assertEqual(form.cleaned_data["return_by"].tzinfo, SEOUL)
+        self.assertEqual(
+            form.fields["departure_at"].widget.input_type,
+            "datetime-local",
+        )
+        self.assertEqual(form.fields["return_by"].widget.attrs["step"], "60")
+
+    @patch("public_web.forms._now_kst", return_value=NOW)
+    def test_past_order_horizon_and_shape_fail_closed(self, _now):
+        cases = (
+            (
+                self._form(departure_at="2026-08-31T09:00"),
+                "departure_not_future",
+            ),
+            (
+                self._form(return_by="2026-08-31T10:00"),
+                "window_order",
+            ),
+            (
+                self._form(return_by="2026-09-07T09:01"),
+                "window_too_long",
+            ),
+            (
+                self._form(departure_at="2026-08-31T10:00:30"),
+                "invalid",
+            ),
+        )
+        for form, error_code in cases:
+            with self.subTest(error_code=error_code):
+                self.assertFalse(form.is_valid())
+                codes = {
+                    error.code
+                    for errors in form.errors.as_data().values()
+                    for error in errors
+                }
+                self.assertIn(error_code, codes)
+
+
+def _search_result():
+    return {
+        "flight_state": "ready",
+        "flight_status_label": "현재 게시본",
+        "flight_message": "합성 검색 결과",
+        "flight_checked_at": "2026.08.31 08:30 KST",
+        "flight_publication_revision": "7",
+        "flight_source_locator": "https://example.invalid/schedule",
+        "flight_source_attribution": "합성 출처",
+        "opportunities": [{"rank": 1}],
+    }
+
+
+class OpportunityViewContractTests(SimpleTestCase):
+    def _render_response(self, _request, _template, _context):
+        return HttpResponse("rendered")
+
+    @patch("public_web.forms._now_kst", return_value=NOW)
+    @patch("public_web.views.render")
+    @patch("public_web.views.search_travel_opportunities")
+    def test_valid_post_is_transient_200_with_frozen_context(
+        self,
+        search,
+        render,
+        _now,
+    ):
+        search.return_value = _search_result()
+        render.side_effect = self._render_response
+
+        response = self.client.post(
+            reverse("public_web:index"),
+            {
+                "departure_at": "2026-08-31T10:00",
+                "return_by": "2026-09-01T20:00",
+            },
+        )
+
+        self.assertEqual(response.status_code, 200)
+        self.assertEqual(response.headers["Cache-Control"], "no-store")
+        self.assertNotIn("sessionid", response.cookies)
+        search.assert_called_once()
+        call = search.call_args.kwargs
+        self.assertEqual(call["departure_at"].tzinfo, SEOUL)
+        self.assertEqual(call["return_by"].tzinfo, SEOUL)
+        context = render.call_args.args[2]
+        self.assertEqual(
+            set(context),
+            {
+                "form",
+                "has_search_result",
+                "search_window",
+                "flight_state",
+                "flight_status_label",
+                "flight_message",
+                "flight_checked_at",
+                "flight_publication_revision",
+                "flight_source_locator",
+                "flight_source_attribution",
+                "opportunities",
+            },
+        )
+        self.assertTrue(context["form"].is_bound)
+        self.assertTrue(context["has_search_result"])
+        self.assertEqual(
+            context["search_window"],
+            {
+                "departure_at_label": "2026.08.31 10:00 KST",
+                "return_by_label": "2026.09.01 20:00 KST",
+            },
+        )
+        self.assertEqual(context["opportunities"], [{"rank": 1}])
+
+    @patch("public_web.forms._now_kst", return_value=NOW)
+    @patch("public_web.views.render")
+    @patch("public_web.views.search_travel_opportunities")
+    def test_invalid_post_keeps_bound_values_and_does_not_search(
+        self,
+        search,
+        render,
+        _now,
+    ):
+        render.side_effect = self._render_response
+        response = self.client.post(
+            reverse("public_web:index"),
+            {
+                "departure_at": "2026-09-01T20:00",
+                "return_by": "2026-09-01T19:00",
+            },
+        )
+
+        self.assertEqual(response.status_code, 200)
+        search.assert_not_called()
+        context = render.call_args.args[2]
+        self.assertFalse(context["has_search_result"])
+        self.assertEqual(context["opportunities"], [])
+        self.assertTrue(context["form"].is_bound)
+        self.assertIn("window_order", {
+            error.code
+            for error in context["form"].errors.as_data()["return_by"]
+        })
+
+    @patch("public_web.views.render")
+    def test_get_is_unbound_and_results_route_redirects_queryless(self, render):
+        render.side_effect = self._render_response
+        index = self.client.get(reverse("public_web:index"))
+        context = render.call_args.args[2]
+        self.assertEqual(index.status_code, 200)
+        self.assertFalse(context["form"].is_bound)
+        self.assertFalse(context["has_search_result"])
+
+        results = self.client.get(reverse("public_web:results"))
+        self.assertEqual(results.status_code, 303)
+        self.assertEqual(results.headers["Location"], "/")
+        self.assertNotIn("?", results.headers["Location"])
+
+
+def _publication_base_row() -> dict[str, object]:
+    return {
+        "current_publication_id": UUID("6fc3778f-53a5-4dc0-9177-ac6b7f03299e"),
+        "current_publication__generation": 3,
+        "current_publication__published_at": NOW,
+        "current_publication__scope_country__name_ko": "대만",
+        "current_publication__source_code_snapshot": "SYNTHETIC",
+        "current_publication__source_revision": "synthetic-v1",
+        "current_publication__source_owner_snapshot": "합성 기관",
+        "current_publication__source_locator_snapshot": "https://example.invalid/source",
+        "current_publication__attribution_text_snapshot": "합성 출처",
+        "current_publication__source_contract_fingerprint_sha256": "a" * 64,
+    }
+
+
+class CountryPublicationCardTests(SimpleTestCase):
+    @patch("public_web.results._freshness", return_value=(CARD_READY, NOW))
+    @patch("public_web.results._entry_row")
+    def test_entry_card_contains_complete_typed_facts(self, entry_row, _freshness):
+        row = _publication_base_row()
+        prefix = "current_publication__entry_fact_revision__parse_run__artifact"
+        row.update(
+            {
+                "current_publication__entry_fact_revision__ordinary_passport_period_text": "90일",
+                "current_publication__entry_fact_revision__basis_text": "일반여권 소지자: 90일",
+                "current_publication__entry_fact_revision__snapshot_date": date(2026, 8, 1),
+                f"{prefix}_id": UUID("57a68d56-4e7c-4a06-a8bd-f106e0ed4a61"),
+                f"{prefix}__source_id": UUID("5ca7e28c-81df-40ea-b77b-69b95860949f"),
+                f"{prefix}__source__code": "SYNTHETIC",
+                f"{prefix}__source__revision": "synthetic-v1",
+                f"{prefix}__source__module": "ENTRY_REQUIREMENT",
+                f"{prefix}__source__owner": "합성 기관",
+                f"{prefix}__source__official_locator": "https://example.invalid/source",
+                f"{prefix}__source__state": "ACTIVE",
+                f"{prefix}__source__enabled": True,
+            }
+        )
+        entry_row.return_value = row
+
+        card = _load_entry_card(SUPPORTED_COUNTRY_IDS["TW"])
+
+        entry_row.assert_called_once_with(SUPPORTED_COUNTRY_IDS["TW"])
+        self.assertEqual(card["period_text"], "90일")
+        self.assertEqual(card["basis_text"], "일반여권 소지자: 90일")
+        self.assertEqual(card["snapshot_date"], "2026-08-01")
+
+    @patch("public_web.results._freshness", return_value=(CARD_READY, NOW))
+    @patch("public_web.results._warning_row")
+    def test_warning_card_contains_complete_typed_facts(self, warning_row, _freshness):
+        row = _publication_base_row()
+        prefix = "current_publication__travel_warning_revision__parse_run__artifact"
+        row.update(
+            {
+                "current_publication__travel_warning_revision__source_alarm_level_code": "1",
+                "current_publication__travel_warning_revision__source_scope_type": "일부",
+                "current_publication__travel_warning_revision__source_scope_text": "합성 범위",
+                "current_publication__travel_warning_revision__source_written_date": date(2026, 8, 2),
+                f"{prefix}_id": UUID("57a68d56-4e7c-4a06-a8bd-f106e0ed4a61"),
+                f"{prefix}__source_id": UUID("5ca7e28c-81df-40ea-b77b-69b95860949f"),
+                f"{prefix}__source__code": "SYNTHETIC",
+                f"{prefix}__source__revision": "synthetic-v1",
+                f"{prefix}__source__module": "TRAVEL_WARNING",
+                f"{prefix}__source__owner": "합성 기관",
+                f"{prefix}__source__official_locator": "https://example.invalid/source",
+                f"{prefix}__source__state": "ACTIVE",
+                f"{prefix}__source__enabled": True,
+            }
+        )
+        warning_row.return_value = row
+
+        card = _load_warning_card(SUPPORTED_COUNTRY_IDS["TW"])
+
+        warning_row.assert_called_once_with(SUPPORTED_COUNTRY_IDS["TW"])
+        self.assertEqual(card["alarm_level_code"], "1")
+        self.assertEqual(card["scope_type"], "일부")
+        self.assertEqual(card["scope_text"], "합성 범위")
+        self.assertEqual(card["written_date"], "2026-08-02")
+
+    @patch("public_web.results._load_warning_card")
+    @patch("public_web.results._load_entry_card", side_effect=RuntimeError)
+    def test_entry_and_warning_fail_independently(self, _entry, warning):
+        warning.return_value = {"module": "warning", "state": CARD_READY}
+
+        cards = load_publication_cards_for_country(SUPPORTED_COUNTRY_IDS["TW"])
+
+        self.assertEqual(cards["entry"]["state"], CARD_SERVER_ERROR)
+        self.assertEqual(cards["warning"]["state"], CARD_READY)
diff --git a/public_web/views.py b/public_web/views.py
index b328c3e..cce910f 100644
--- a/public_web/views.py
+++ b/public_web/views.py
@@ -1,10 +1,18 @@
+from datetime import datetime
+from uuid import UUID
+
 from django.http import HttpRequest, HttpResponse
 from django.shortcuts import render
 from django.urls import reverse
 from django.views.decorators.debug import sensitive_post_parameters
 from django.views.decorators.http import require_http_methods
 
-from .forms import TripForm
+from travel_windows.search import (
+    search_travel_opportunities as _calculate_travel_opportunities,
+)
+
+from .forms import SEOUL, TripForm
+from .results import load_publication_cards_for_country
 
 
 def _fixed_redirect(route_name: str) -> HttpResponse:
@@ -17,26 +25,76 @@ def _fixed_redirect(route_name: str) -> HttpResponse:
     )
 
 
-@sensitive_post_parameters(
-    "destination",
-    "departure_date",
-    "return_date",
-)
+def _display_kst(value: datetime) -> str:
+    return value.astimezone(SEOUL).strftime("%Y.%m.%d %H:%M KST")
+
+
+def _publication_cards(country_id: UUID) -> dict[str, dict[str, object]]:
+    return load_publication_cards_for_country(country_id)
+
+
+def search_travel_opportunities(
+    *, departure_at: datetime, return_by: datetime
+) -> dict[str, object]:
+    """Patchable request boundary around the DB-only opportunity calculator."""
+
+    return _calculate_travel_opportunities(
+        departure_at=departure_at,
+        return_by=return_by,
+        publication_card_loader=_publication_cards,
+    )
+
+
+def _initial_context(form: TripForm) -> dict[str, object]:
+    return {
+        "form": form,
+        "has_search_result": False,
+        "search_window": {
+            "departure_at_label": "",
+            "return_by_label": "",
+        },
+        "flight_state": "empty",
+        "flight_status_label": "검색 전",
+        "flight_message": "여행 가능 시간을 입력하면 현재 게시된 운항 자료로 계산합니다.",
+        "flight_checked_at": None,
+        "flight_publication_revision": None,
+        "flight_source_locator": None,
+        "flight_source_attribution": None,
+        "opportunities": [],
+    }
+
+
+@sensitive_post_parameters("departure_at", "return_by")
 @require_http_methods(("GET", "POST"))
 def index(request: HttpRequest) -> HttpResponse:
     if request.method == "GET" and request.GET:
         return _fixed_redirect("public_web:index")
+
     if request.method == "POST":
         form = TripForm(request.POST)
         if form.is_valid():
-            return _fixed_redirect("public_web:results")
-        form.mark_errors_for_accessibility()
+            departure_at = form.cleaned_data["departure_at"]
+            return_by = form.cleaned_data["return_by"]
+            search_context = search_travel_opportunities(
+                departure_at=departure_at,
+                return_by=return_by,
+            )
+            context = {
+                "form": form,
+                "has_search_result": True,
+                "search_window": {
+                    "departure_at_label": _display_kst(departure_at),
+                    "return_by_label": _display_kst(return_by),
+                },
+                **search_context,
+            }
+        else:
+            form.mark_errors_for_accessibility()
+            context = _initial_context(form)
     else:
         form = TripForm()
-    response = render(
-        request,
-        "public_web/index.html",
-        {"form": form},
-    )
+        context = _initial_context(form)
+
+    response = render(request, "public_web/index.html", context)
     response.headers["Cache-Control"] = "no-store"
     return response


