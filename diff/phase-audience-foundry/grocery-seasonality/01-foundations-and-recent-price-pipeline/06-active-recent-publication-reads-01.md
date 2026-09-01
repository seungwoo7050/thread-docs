# 활성 최근 게시본 읽기

## `feat(public): serve active retail facts`

diff --git a/.env.example b/.env.example
index 5a85a53..edbe21f 100644
--- a/.env.example
+++ b/.env.example
@@ -6,4 +6,6 @@ DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1
 DJANGO_CSRF_TRUSTED_ORIGINS=https://replace-with-approved-domain.example
 DATABASE_URL=postgresql://grocery:local-grocery-only@127.0.0.1:55434/grocery
 KAMIS_API_KEY=
+KAMIS_CONFIRMATION_MAX_AGE_HOURS=36
+QA_STATE_PREVIEWS_ENABLED=0
 DEPLOY_VERSION=0000000
diff --git a/config/settings.py b/config/settings.py
index 59d89c8..2524aff 100644
--- a/config/settings.py
+++ b/config/settings.py
@@ -19,6 +19,19 @@ def env_list(name: str, default: str) -> list[str]:
     return [item.strip() for item in os.environ.get(name, default).split(",") if item.strip()]
 
 
+def env_positive_int(name: str, default: int, *, maximum: int) -> int:
+    raw_value = os.environ.get(name)
+    if raw_value is None:
+        return default
+    try:
+        value = int(raw_value)
+    except ValueError:
+        raise ImproperlyConfigured(f"{name.lower()}_invalid") from None
+    if value < 1 or value > maximum:
+        raise ImproperlyConfigured(f"{name.lower()}_invalid")
+    return value
+
+
 def database_config() -> dict[str, object]:
     value = os.environ.get(
         "DATABASE_URL",
@@ -166,6 +179,12 @@ SECURE_REFERRER_POLICY = "same-origin"
 X_FRAME_OPTIONS = "DENY"
 
 DEPLOY_VERSION = os.environ.get("DEPLOY_VERSION", "0000000")
+KAMIS_CONFIRMATION_MAX_AGE_HOURS = env_positive_int(
+    "KAMIS_CONFIRMATION_MAX_AGE_HOURS",
+    36,
+    maximum=168,
+)
+QA_STATE_PREVIEWS_ENABLED = DEBUG and env_bool("QA_STATE_PREVIEWS_ENABLED", False)
 
 LOGGING = {
     "version": 1,
diff --git a/config/urls.py b/config/urls.py
index 4096fa2..052f697 100644
--- a/config/urls.py
+++ b/config/urls.py
@@ -1,4 +1,7 @@
 from django.contrib import admin
-from django.urls import path
+from django.urls import include, path
 
-urlpatterns = [path("admin/", admin.site.urls)]
+urlpatterns = [
+    path("", include("grocery.urls")),
+    path("admin/", admin.site.urls),
+]
diff --git a/grocery/public_read.py b/grocery/public_read.py
new file mode 100644
index 0000000..6fe1ec2
--- /dev/null
+++ b/grocery/public_read.py
@@ -0,0 +1,252 @@
+"""Read-only presentation data from the active recent-retail publication."""
+
+from __future__ import annotations
+
+from dataclasses import dataclass
+from datetime import datetime, timedelta
+from typing import Final
+
+from django.conf import settings
+from django.core.exceptions import ValidationError
+from django.db.models import QuerySet
+from django.utils import timezone
+
+from grocery.models import (
+    FetchAttempt,
+    PublicationChannel,
+    PublicationEntry,
+    PublicationRevision,
+    ReferencePrice,
+)
+from grocery.presentation import (
+    direction_label,
+    format_absolute_krw,
+    format_korean_date,
+    format_korean_datetime,
+    format_krw,
+    format_signed_percentage,
+    format_unit,
+)
+
+RECENT_RETAIL_CHANNEL: Final = "RECENT_RETAIL"
+PUBLIC_RESULT_LIMIT: Final = 100
+KAMIS_LANDING_URL: Final = "https://www.data.go.kr/data/15156063/openapi.do"
+
+_CATEGORY_CODES: Final = {"vegetable": "200", "fruit": "400"}
+_PERIOD_LABELS: Final = {
+    "WEEK": "1주 전 제공값",
+    "MONTH": "1개월 전 제공값",
+    "YEAR": "1년 전 제공값",
+}
+_PERIOD_ORDER: Final = {"WEEK": 1, "MONTH": 2, "YEAR": 3}
+_COVERAGE_LABELS: Final = {
+    "KAMIS_RETAIL_ALL_REGIONS_22_CITIES_V1": "KAMIS 소매 조사 22개 도시 지역 전체 집계",
+}
+
+
+@dataclass(frozen=True, slots=True)
+class ActivePublication:
+    revision: PublicationRevision
+    checked_at: datetime
+    freshness_state: str
+    freshness_label: str
+    stale_message: str
+
+
+def load_active_publication(*, observed_at: datetime | None = None) -> ActivePublication | None:
+    """Load the only public pointer and derive operational freshness separately."""
+
+    channel = (
+        PublicationChannel.objects.select_related(
+            "current_revision__generation__artifact",
+            "current_revision__review_decision__source_configuration",
+        )
+        .filter(pk=RECENT_RETAIL_CHANNEL)
+        .first()
+    )
+    if channel is None or channel.current_revision is None:
+        return None
+
+    revision = channel.current_revision
+    if revision.sealed_at is None or revision.channel != RECENT_RETAIL_CHANNEL:
+        raise ValidationError("The active publication pointer is not a sealed recent revision.")
+
+    source = revision.review_decision.source_configuration
+    artifact = revision.generation.artifact
+    confirmed_attempt = (
+        FetchAttempt.objects.filter(
+            source_configuration=source,
+            artifact=artifact,
+            state=FetchAttempt.State.SUCCEEDED,
+        )
+        .order_by("-completed_at", "-started_at")
+        .first()
+    )
+    if confirmed_attempt is None or confirmed_attempt.completed_at is None:
+        raise ValidationError("The active publication has no completed source confirmation.")
+
+    latest_attempt = (
+        FetchAttempt.objects.filter(source_configuration=source)
+        .exclude(state=FetchAttempt.State.STARTED)
+        .order_by("-completed_at", "-started_at")
+        .first()
+    )
+    now = observed_at or timezone.now()
+    max_age = timedelta(hours=settings.KAMIS_CONFIRMATION_MAX_AGE_HOURS)
+    newer_content_waits_for_review = bool(
+        latest_attempt is not None
+        and latest_attempt.state == FetchAttempt.State.SUCCEEDED
+        and latest_attempt.artifact_id != artifact.id
+    )
+    newer_attempt_failed = bool(
+        latest_attempt is not None
+        and latest_attempt.state != FetchAttempt.State.SUCCEEDED
+        and latest_attempt.completed_at is not None
+        and latest_attempt.completed_at > confirmed_attempt.completed_at
+    )
+    confirmation_is_old = now - confirmed_attempt.completed_at > max_age
+    stale = newer_content_waits_for_review or newer_attempt_failed or confirmation_is_old
+
+    if stale:
+        return ActivePublication(
+            revision=revision,
+            checked_at=confirmed_attempt.completed_at,
+            freshness_state="stale",
+            freshness_label="마지막 검토 자료 · 새 확인 필요",
+            stale_message=(
+                "아래에는 마지막으로 검토해 공개한 조사값을 표시합니다. "
+                "새 수집 또는 검토 상태를 운영자가 확인하고 있습니다."
+            ),
+        )
+    return ActivePublication(
+        revision=revision,
+        checked_at=confirmed_attempt.completed_at,
+        freshness_state="current",
+        freshness_label="마지막 source 확인 완료",
+        stale_message="",
+    )
+
+
+def publication_entries(
+    active: ActivePublication,
+    *,
+    query: str,
+    category: str,
+) -> QuerySet[PublicationEntry]:
+    entries = active.revision.entries.select_related("snapshot__series").order_by(
+        "snapshot__series__category_code",
+        "snapshot__series__item_name",
+        "snapshot__series__item_code",
+        "snapshot__series__variety_code",
+        "snapshot__series__grade_code",
+        "snapshot__series__raw_unit",
+        "snapshot__series__raw_unit_size",
+    )
+    if category:
+        entries = entries.filter(snapshot__series__category_code=_CATEGORY_CODES[category])
+    if query:
+        entries = entries.filter(snapshot__series__item_name__icontains=query)
+    return entries[:PUBLIC_RESULT_LIMIT]
+
+
+def catalog_item(entry: PublicationEntry, active: ActivePublication, *, url: str) -> dict[str, str]:
+    snapshot = entry.snapshot
+    series = snapshot.series
+    return {
+        "url": url,
+        "category_label": series.category_name,
+        "item_name": series.item_name,
+        "variety_name": series.variety_name,
+        "grade_name": series.grade_name,
+        "unit_label": format_unit(series.raw_unit, series.raw_unit_size),
+        "current_price_label": format_krw(snapshot.current_price),
+        "source_date_iso": snapshot.source_effective_date.isoformat(),
+        "source_date_label": format_korean_date(snapshot.source_effective_date),
+        "freshness_state": active.freshness_state,
+        "freshness_label": active.freshness_label,
+    }
+
+
+def detail_context(entry: PublicationEntry, active: ActivePublication) -> dict[str, object]:
+    snapshot = entry.snapshot
+    series = snapshot.series
+    references = list(snapshot.reference_prices.select_related("change_fact"))
+    if len(references) != 3 or {reference.period for reference in references} != set(_PERIOD_ORDER):
+        raise ValidationError("Published detail requires exact WEEK, MONTH, and YEAR references.")
+    references.sort(key=lambda reference: _PERIOD_ORDER[reference.period])
+
+    source = active.revision.review_decision.source_configuration
+    coverage_label = _COVERAGE_LABELS.get(series.coverage_identity)
+    if coverage_label is None:
+        raise ValidationError("Published detail has an unknown coverage identity.")
+
+    return {
+        "series": {
+            "category_label": series.category_name,
+            "item_name": series.item_name,
+            "variety_name": series.variety_name,
+            "grade_name": series.grade_name,
+            "unit_label": format_unit(series.raw_unit, series.raw_unit_size),
+            "current_price_machine": format(snapshot.current_price, "f"),
+            "current_price_label": format_krw(snapshot.current_price),
+        },
+        "comparisons": [_comparison_context(reference) for reference in references],
+        "provenance": {
+            "source_name": f"{source.source_owner_name} KAMIS 최근일자 도·소매가격정보",
+            "source_url": KAMIS_LANDING_URL,
+            "dataset_id": source.dataset_id,
+            "source_date_iso": snapshot.source_effective_date.isoformat(),
+            "source_date_label": format_korean_date(snapshot.source_effective_date),
+            "coverage_label": coverage_label,
+            "checked_at_iso": active.checked_at.isoformat(),
+            "checked_at_label": format_korean_datetime(active.checked_at),
+            "reviewed_at_iso": active.revision.review_decision.decided_at.isoformat(),
+            "reviewed_at_label": format_korean_datetime(active.revision.review_decision.decided_at),
+            "freshness_state": active.freshness_state,
+            "freshness_label": active.freshness_label,
+        },
+    }
+
+
+def _comparison_context(reference: ReferencePrice) -> dict[str, object]:
+    base: dict[str, object] = {
+        "period_label": _PERIOD_LABELS[reference.period],
+        "reference_date_available": reference.source_reference_date is not None,
+        "reference_date_iso": (
+            reference.source_reference_date.isoformat()
+            if reference.source_reference_date is not None
+            else ""
+        ),
+        "reference_date_label": (
+            format_korean_date(reference.source_reference_date)
+            if reference.source_reference_date is not None
+            else ""
+        ),
+    }
+    if reference.value_status == ReferencePrice.ValueStatus.UNAVAILABLE:
+        base.update(
+            {
+                "available": False,
+                "unavailable_reason_label": "source 응답에 비교 제공값이 없습니다.",
+            }
+        )
+        return base
+
+    change = reference.change_fact
+    if (
+        reference.value is None
+        or change.signed_difference is None
+        or change.signed_percentage is None
+    ):
+        raise ValidationError("An available reference requires a complete change fact.")
+    base.update(
+        {
+            "available": True,
+            "reference_value_label": format_krw(reference.value),
+            "difference_label": format_absolute_krw(change.signed_difference),
+            "percentage_label": format_signed_percentage(change.signed_percentage),
+            "direction_code": change.direction,
+            "direction_label": direction_label(change.direction),
+        }
+    )
+    return base
diff --git a/grocery/static/grocery/app.css b/grocery/static/grocery/app.css
index 07ecb51..5d86bd5 100644
--- a/grocery/static/grocery/app.css
+++ b/grocery/static/grocery/app.css
@@ -119,6 +119,15 @@ h3 {
   padding-block: clamp(2rem, 7vw, 5rem);
 }
 
+.qa-notice {
+  margin-bottom: 1.5rem;
+  padding: 0.65rem 0.8rem;
+  border: 2px dashed var(--color-warning);
+  background: var(--color-warning-soft);
+  color: var(--color-warning);
+  font-weight: 800;
+}
+
 .skip-link {
   position: fixed;
   z-index: 100;
diff --git a/grocery/templates/grocery/base.html b/grocery/templates/grocery/base.html
index c6114bc..9946774 100644
--- a/grocery/templates/grocery/base.html
+++ b/grocery/templates/grocery/base.html
@@ -25,6 +25,9 @@
     </header>
 
     <main id="main-content" class="page-shell page-main" tabindex="-1">
+      {% if qa_preview %}
+        <p class="qa-notice" role="note">로컬 화면 상태 검수용 미리보기</p>
+      {% endif %}
       {% block content %}{% endblock %}
     </main>
 
diff --git a/grocery/tests/test_production_settings.py b/grocery/tests/test_production_settings.py
index 8fc0001..f56c5d1 100644
--- a/grocery/tests/test_production_settings.py
+++ b/grocery/tests/test_production_settings.py
@@ -3,7 +3,7 @@ from collections.abc import Mapping, Sequence
 import pytest
 from django.core.exceptions import ImproperlyConfigured
 
-from config.settings import validate_production_environment
+from config.settings import env_positive_int, validate_production_environment
 
 _SAFE_SECRET = "x" * 50
 _SAFE_HOSTS = ("prices.example",)
@@ -154,3 +154,12 @@ def test_validation_error_never_reflects_supplied_values() -> None:
         )
 
     assert marker not in str(caught.value)
+
+
+def test_positive_integer_environment_bound_is_explicit() -> None:
+    assert env_positive_int("MISSING_TEST_VALUE", 36, maximum=168) == 36
+
+    with pytest.raises(ImproperlyConfigured, match="^bounded_test_value_invalid$"):
+        with pytest.MonkeyPatch.context() as monkeypatch:
+            monkeypatch.setenv("BOUNDED_TEST_VALUE", "0")
+            env_positive_int("BOUNDED_TEST_VALUE", 36, maximum=168)
diff --git a/grocery/tests/test_public_routes.py b/grocery/tests/test_public_routes.py
new file mode 100644
index 0000000..0217e56
--- /dev/null
+++ b/grocery/tests/test_public_routes.py
@@ -0,0 +1,220 @@
+import uuid
+from datetime import timedelta
+from typing import Any
+from unittest.mock import patch
+
+import pytest
+from django.contrib.auth.models import Permission
+from django.db import DatabaseError
+from django.test import Client, override_settings
+from django.urls import reverse
+
+from grocery.models import (
+    PriceSeriesKey,
+    PublicationActivation,
+    PublicationChannel,
+    PublicationRevision,
+    RetailPriceSnapshot,
+    seal_recent_publication,
+    transition_recent_publication,
+)
+from grocery.public_read import load_active_publication
+from grocery.tests.test_price_series_key_models import create_series
+from grocery.tests.test_publication_revision_models import create_approved_generation
+
+_EVIDENCE_HASH = "a" * 64
+
+
+def activate_publication() -> tuple[PublicationRevision, RetailPriceSnapshot, Any]:
+    decision, snapshots, publisher = create_approved_generation()
+    permission = Permission.objects.get(
+        content_type__app_label="grocery",
+        codename="publish_publication",
+    )
+    publisher.user_permissions.add(permission)
+    publisher = type(publisher)._default_manager.get(pk=publisher.pk)
+    revision = seal_recent_publication(decision.id, "ko-v1")
+    transition_recent_publication(
+        operation_id=uuid.uuid4(),
+        actor=publisher,
+        operation=PublicationActivation.Operation.ACTIVATE,
+        target_revision_id=revision.id,
+        expected_current_revision_id=None,
+        expected_version=0,
+        reason_code="LOCAL_PHASE0_ACTIVATE",
+        acceptance_evidence_sha256=_EVIDENCE_HASH,
+    )
+    return revision, snapshots[0], publisher
+
+
+@pytest.mark.django_db
+def test_catalog_is_unavailable_before_activation_and_candidate_detail_is_hidden() -> None:
+    _, snapshots, _ = create_approved_generation()
+    candidate_series_id = snapshots[0].series_id
+
+    catalog_response = Client().get(reverse("grocery:catalog"))
+    detail_response = Client().get(
+        reverse("grocery:detail", kwargs={"series_id": candidate_series_id})
+    )
+
+    assert catalog_response.status_code == 200
+    assert "공개 조사값 없음" in catalog_response.content.decode()
+    assert "X-Publication-Fact-Set" not in catalog_response
+    assert detail_response.status_code == 404
+    assert not PublicationChannel.objects.exists()
+
+
+@pytest.mark.django_db
+def test_catalog_search_and_detail_use_only_the_active_sealed_revision() -> None:
+    revision, snapshot, _ = activate_publication()
+    client = Client()
+
+    catalog_response = client.get(reverse("grocery:catalog"))
+    search_response = client.get(reverse("grocery:catalog"), {"q": snapshot.series.item_name})
+    empty_response = client.get(reverse("grocery:catalog"), {"q": "일치하지않는공식품목명"})
+    detail_response = client.get(
+        reverse("grocery:detail", kwargs={"series_id": snapshot.series_id})
+    )
+
+    assert catalog_response.status_code == 200
+    assert search_response.status_code == 200
+    assert empty_response.status_code == 200
+    assert detail_response.status_code == 200
+    assert catalog_response.headers["X-Publication-Fact-Set"] == revision.typed_fact_set_sha256
+    assert detail_response.headers["X-Publication-Fact-Set"] == revision.typed_fact_set_sha256
+    assert snapshot.series.item_name in search_response.content.decode()
+    assert "조건에 맞는 항목 없음" in empty_response.content.decode()
+    assert "KAMIS 소매 조사 평균" in detail_response.content.decode()
+    assert "2,000원 낮음" in detail_response.content.decode()
+    assert "(+25.0%)" in detail_response.content.decode()
+    assert "같음" in detail_response.content.decode()
+    assert "source가 비교 기준일을 별도로 제공하지 않음" in detail_response.content.decode()
+    assert "데이터셋 15156063" in detail_response.content.decode()
+    assert "sessionid" not in catalog_response.cookies
+
+
+@pytest.mark.django_db
+def test_nonmember_series_is_not_addressable_through_active_publication() -> None:
+    activate_publication()
+    nonmember = create_series(item_code="999", item_name="검토되지않은후보")
+
+    response = Client().get(reverse("grocery:detail", kwargs={"series_id": nonmember.id}))
+
+    assert response.status_code == 404
+    assert "검토되지않은후보" not in response.content.decode()
+
+
+@pytest.mark.django_db
+def test_invalid_mobile_search_input_returns_associated_correction_error() -> None:
+    activate_publication()
+    client = Client()
+
+    invalid_query = "가" * 81
+    invalid_response = client.get(reverse("grocery:catalog"), {"q": invalid_query})
+    corrected_response = client.get(reverse("grocery:catalog"), {"q": "품목"})
+
+    invalid_html = invalid_response.content.decode()
+    assert invalid_response.status_code == 400
+    assert 'role="alert"' in invalid_html
+    assert 'aria-invalid="true"' in invalid_html
+    assert 'aria-describedby="catalog-query-hint search-error"' in invalid_html
+    assert "검색어는 80자 이하여야 합니다." in invalid_html
+    assert corrected_response.status_code == 200
+    assert 'aria-invalid="true"' not in corrected_response.content.decode()
+
+
+@pytest.mark.django_db
+def test_public_request_never_calls_external_source_client() -> None:
+    _, snapshot, _ = activate_publication()
+    client = Client()
+
+    with patch(
+        "grocery.source.client.KamisHttpClient.fetch_recent_prices",
+        side_effect=AssertionError("external source must not be called"),
+    ) as fetch:
+        assert client.get(reverse("grocery:catalog")).status_code == 200
+        assert (
+            client.get(
+                reverse("grocery:detail", kwargs={"series_id": snapshot.series_id})
+            ).status_code
+            == 200
+        )
+
+    fetch.assert_not_called()
+
+
+@pytest.mark.django_db
+def test_confirmation_age_is_separate_and_preserves_last_known_good() -> None:
+    revision, _, _ = activate_publication()
+    current = load_active_publication()
+    assert current is not None
+
+    with override_settings(KAMIS_CONFIRMATION_MAX_AGE_HOURS=1):
+        stale = load_active_publication(observed_at=current.checked_at + timedelta(hours=2))
+
+    assert stale is not None
+    assert stale.revision.id == revision.id
+    assert stale.freshness_state == "stale"
+    assert "새 확인 필요" in stale.freshness_label
+
+
+@pytest.mark.django_db
+def test_database_failure_uses_fixed_server_error_without_exception_reflection() -> None:
+    marker = "must-not-be-reflected"
+    with patch("grocery.views.load_active_publication", side_effect=DatabaseError(marker)):
+        response = Client().get(reverse("grocery:catalog"))
+
+    assert response.status_code == 503
+    assert "자료를 표시하지 못함" in response.content.decode()
+    assert marker not in response.content.decode()
+
+
+@pytest.mark.django_db
+def test_qa_state_routes_are_hard_disabled_unless_local_setting_is_explicit() -> None:
+    disabled = Client().get(reverse("grocery:qa_catalog_state", kwargs={"state": "loading"}))
+    assert disabled.status_code == 404
+
+    with override_settings(QA_STATE_PREVIEWS_ENABLED=True):
+        for state in ("loading", "empty", "unavailable", "stale", "server_error"):
+            response = Client().get(reverse("grocery:qa_catalog_state", kwargs={"state": state}))
+            assert response.status_code == (503 if state == "server_error" else 200)
+            assert "로컬 화면 상태 검수용 미리보기" in response.content.decode()
+
+
+@pytest.mark.django_db
+def test_catalog_response_does_not_expose_internal_actor_or_revision_ids() -> None:
+    revision, _, publisher = activate_publication()
+
+    response = Client().get(reverse("grocery:catalog"))
+    body = response.content.decode()
+
+    assert str(revision.id) not in body
+    assert publisher.username not in body
+    assert revision.review_decision.acceptance_evidence_sha256 not in body
+
+
+@pytest.mark.django_db
+def test_unrelated_unpublished_series_does_not_change_catalog_results() -> None:
+    activate_publication()
+    before = Client().get(reverse("grocery:catalog")).content
+    PriceSeriesKey.get_or_validate(
+        product_class_code="01",
+        product_class_name="소매",
+        category_code="200",
+        category_name="채소류",
+        item_code="998",
+        item_name="공개되지않은긴후보이름",
+        variety_code="00",
+        variety_name="후보품종",
+        grade_code="04",
+        grade_name="상품",
+        raw_unit="단",
+        raw_unit_size="99",
+        coverage_identity="KAMIS_RETAIL_ALL_REGIONS_22_CITIES_V1",
+        identity_evidence_revision="kamis-codebook-20260830-v1",
+    )
+
+    after = Client().get(reverse("grocery:catalog")).content
+
+    assert before == after
+    assert "공개되지않은긴후보이름" not in after.decode()
diff --git a/grocery/urls.py b/grocery/urls.py
new file mode 100644
index 0000000..b1f2408
--- /dev/null
+++ b/grocery/urls.py
@@ -0,0 +1,12 @@
+from django.urls import path
+
+from grocery import views
+
+app_name = "grocery"
+
+urlpatterns = [
+    path("", views.catalog, name="catalog"),
+    path("series/<uuid:series_id>/", views.detail, name="detail"),
+    path("__qa__/catalog/<str:state>/", views.qa_catalog_state, name="qa_catalog_state"),
+    path("__qa__/detail/<str:state>/", views.qa_detail_state, name="qa_detail_state"),
+]
diff --git a/grocery/views.py b/grocery/views.py
new file mode 100644
index 0000000..1e453c1
--- /dev/null
+++ b/grocery/views.py
@@ -0,0 +1,292 @@
+"""Server-rendered public views over the active publication only."""
+
+from __future__ import annotations
+
+import logging
+import uuid
+from typing import Final
+from urllib.parse import urlencode
+
+from django.conf import settings
+from django.core.exceptions import ObjectDoesNotExist, ValidationError
+from django.db import DatabaseError
+from django.http import Http404, HttpRequest, HttpResponse
+from django.shortcuts import render
+from django.urls import reverse
+
+from grocery.forms import QUERY_MAX_LENGTH, SearchForm
+from grocery.observability import log_event
+from grocery.public_read import (
+    catalog_item,
+    detail_context,
+    load_active_publication,
+    publication_entries,
+)
+
+_LOGGER: Final = logging.getLogger("grocery.audit")
+_QA_STATES: Final = frozenset({"loading", "empty", "unavailable", "stale", "server_error"})
+_QA_DETAIL_STATES: Final = frozenset({"loading", "unavailable", "stale", "server_error"})
+
+
+def catalog(request: HttpRequest) -> HttpResponse:
+    form = SearchForm(request.GET if request.GET else None)
+    query = ""
+    category = ""
+    query_error = ""
+    if form.is_bound and not form.is_valid():
+        raw_query = request.GET.get("q", "")
+        query = raw_query[: QUERY_MAX_LENGTH + 1]
+        query_error = str(next(iter(form.errors.values()))[0])
+        context = _catalog_base_context(query=query, category="")
+        context.update(
+            {
+                "catalog_state": "empty",
+                "query_error": query_error,
+                "results": [],
+                "status_message": "입력을 확인하고 다시 검색해 주세요.",
+            }
+        )
+        return render(request, "grocery/catalog.html", context, status=400)
+    if form.is_valid():
+        query = form.cleaned_data["q"]
+        category = form.cleaned_data["category"]
+
+    try:
+        active = load_active_publication()
+        context = _catalog_base_context(query=query, category=category)
+        if active is None:
+            context.update(
+                {
+                    "catalog_state": "unavailable",
+                    "results": [],
+                    "status_message": "현재 공개할 수 있는 검토 완료 자료가 없습니다.",
+                }
+            )
+            return render(request, "grocery/catalog.html", context)
+
+        entries = list(publication_entries(active, query=query, category=category))
+        results = [
+            catalog_item(
+                entry,
+                active,
+                url=reverse("grocery:detail", kwargs={"series_id": entry.snapshot.series_id}),
+            )
+            for entry in entries
+        ]
+        context.update(
+            {
+                "catalog_state": active.freshness_state if active.stale_message else "ready",
+                "status_message": (
+                    active.stale_message
+                    if active.stale_message
+                    else "검색 조건에 맞는 공개 항목이 없습니다."
+                ),
+                "results": results,
+                "result_count_label": f"공개 항목 {len(results)}개",
+            }
+        )
+        response = render(request, "grocery/catalog.html", context)
+        return _publication_response(response, active.revision.typed_fact_set_sha256)
+    except DatabaseError, ValidationError:
+        log_event(_LOGGER, "ERROR", "public.catalog.unavailable")
+        context = _catalog_base_context(query=query, category=category)
+        context.update(
+            {
+                "catalog_state": "server_error",
+                "results": [],
+                "status_message": "잠시 후 다시 시도해 주세요.",
+                "retry_url": reverse("grocery:catalog"),
+            }
+        )
+        return render(request, "grocery/catalog.html", context, status=503)
+
+
+def detail(request: HttpRequest, series_id: uuid.UUID) -> HttpResponse:
+    try:
+        active = load_active_publication()
+        if active is None:
+            raise Http404
+        entry = (
+            active.revision.entries.select_related(
+                "snapshot__series",
+            )
+            .prefetch_related("snapshot__reference_prices__change_fact")
+            .filter(snapshot__series_id=series_id)
+            .first()
+        )
+        if entry is None:
+            raise Http404
+        context = {
+            "home_url": reverse("grocery:catalog"),
+            "catalog_url": reverse("grocery:catalog"),
+            "detail_state": active.freshness_state if active.stale_message else "ready",
+            "status_message": active.stale_message,
+            **detail_context(entry, active),
+        }
+        response = render(request, "grocery/detail.html", context)
+        return _publication_response(response, active.revision.typed_fact_set_sha256)
+    except Http404:
+        raise
+    except DatabaseError, ObjectDoesNotExist, ValidationError:
+        log_event(_LOGGER, "ERROR", "public.detail.unavailable")
+        context = {
+            "home_url": reverse("grocery:catalog"),
+            "catalog_url": reverse("grocery:catalog"),
+            "detail_state": "server_error",
+            "status_message": "잠시 후 다시 시도해 주세요.",
+            "retry_url": request.path,
+        }
+        return render(request, "grocery/detail.html", context, status=503)
+
+
+def qa_catalog_state(request: HttpRequest, state: str) -> HttpResponse:
+    if not settings.QA_STATE_PREVIEWS_ENABLED or state not in _QA_STATES:
+        raise Http404
+    context = _catalog_base_context(query="아주긴한국어공식품목명", category="vegetable")
+    context.update(
+        {
+            "qa_preview": True,
+            "catalog_state": state,
+            "status_message": "화면 상태와 긴 한국어 표시를 검수하는 로컬 전용 자료입니다.",
+            "retry_url": request.path,
+            "results": _qa_results() if state == "stale" else [],
+            "result_count_label": "공개 항목 1개" if state == "stale" else "공개 항목 0개",
+        }
+    )
+    return render(
+        request, "grocery/catalog.html", context, status=503 if state == "server_error" else 200
+    )
+
+
+def qa_detail_state(request: HttpRequest, state: str) -> HttpResponse:
+    if not settings.QA_STATE_PREVIEWS_ENABLED or state not in _QA_DETAIL_STATES:
+        raise Http404
+    context: dict[str, object] = {
+        "qa_preview": True,
+        "home_url": reverse("grocery:catalog"),
+        "catalog_url": reverse("grocery:catalog"),
+        "detail_state": state,
+        "status_message": "화면 상태와 긴 한국어 표시를 검수하는 로컬 전용 자료입니다.",
+        "retry_url": request.path,
+        "series": {
+            "category_label": "채소류",
+            "item_name": "아주긴한국어공식품목명이작은화면에서도잘려서는안되는품목",
+            "variety_name": "아주긴한국어공식품종표시와세부구분",
+            "grade_name": "공식등급표시",
+            "unit_label": "아주긴원문판매단위표시 포기 × 100",
+        },
+    }
+    if state == "stale":
+        context.update(_qa_detail_ready_context())
+    return render(
+        request, "grocery/detail.html", context, status=503 if state == "server_error" else 200
+    )
+
+
+def _catalog_base_context(*, query: str, category: str) -> dict[str, object]:
+    catalog_url = reverse("grocery:catalog")
+    return {
+        "home_url": catalog_url,
+        "form_action": catalog_url,
+        "query": query,
+        "selected_category": category,
+        "categories": [
+            {
+                "label": label,
+                "url": _category_url(catalog_url, query=query, category=value),
+                "selected": category == value,
+            }
+            for value, label in (("", "전체"), ("vegetable", "채소류"), ("fruit", "과일류"))
+        ],
+    }
+
+
+def _category_url(base_url: str, *, query: str, category: str) -> str:
+    parameters = {}
+    if query:
+        parameters["q"] = query
+    if category:
+        parameters["category"] = category
+    return f"{base_url}?{urlencode(parameters)}" if parameters else base_url
+
+
+def _publication_response(response: HttpResponse, fact_set_sha256: str) -> HttpResponse:
+    response.headers["X-Publication-Fact-Set"] = fact_set_sha256
+    response.headers["Cache-Control"] = "public, max-age=60, stale-if-error=3600"
+    return response
+
+
+def _qa_results() -> list[dict[str, str]]:
+    return [
+        {
+            "url": reverse("grocery:qa_detail_state", kwargs={"state": "stale"}),
+            "category_label": "채소류",
+            "item_name": "아주긴한국어공식품목명이작은화면에서도잘려서는안되는품목",
+            "variety_name": "아주긴한국어공식품종표시와세부구분",
+            "grade_name": "공식등급표시",
+            "unit_label": "아주긴원문판매단위표시 포기 × 100",
+            "current_price_label": "123,456원",
+            "source_date_iso": "2026-08-29",
+            "source_date_label": "2026년 8월 29일",
+            "freshness_state": "stale",
+            "freshness_label": "마지막 검토 자료 · 새 확인 필요",
+        }
+    ]
+
+
+def _qa_detail_ready_context() -> dict[str, object]:
+    return {
+        "series": {
+            "category_label": "채소류",
+            "item_name": "아주긴한국어공식품목명이작은화면에서도잘려서는안되는품목",
+            "variety_name": "아주긴한국어공식품종표시와세부구분",
+            "grade_name": "공식등급표시",
+            "unit_label": "아주긴원문판매단위표시 포기 × 100",
+            "current_price_machine": "123456",
+            "current_price_label": "123,456원",
+        },
+        "comparisons": [
+            {
+                "period_label": "1주 전 제공값",
+                "available": True,
+                "reference_value_label": "125,456원",
+                "difference_label": "2,000원",
+                "percentage_label": "-1.6%",
+                "direction_code": "LOWER",
+                "direction_label": "낮음",
+                "reference_date_available": False,
+            },
+            {
+                "period_label": "1개월 전 제공값",
+                "available": False,
+                "unavailable_reason_label": "source 응답에 비교 제공값이 없습니다.",
+                "reference_date_available": False,
+            },
+            {
+                "period_label": "1년 전 제공값",
+                "available": True,
+                "reference_value_label": "123,456원",
+                "difference_label": "0원",
+                "percentage_label": "0.0%",
+                "direction_code": "EQUAL",
+                "direction_label": "같음",
+                "reference_date_available": False,
+            },
+        ],
+        "provenance": {
+            "source_name": (
+                "한국농수산식품유통공사 KAMIS 최근일자 도·소매가격정보 긴 출처 표시 검수"
+            ),
+            "source_url": "https://www.data.go.kr/data/15156063/openapi.do",
+            "dataset_id": "15156063",
+            "source_date_iso": "2026-08-29",
+            "source_date_label": "2026년 8월 29일",
+            "coverage_label": "KAMIS 소매 조사 22개 도시 지역 전체 집계",
+            "checked_at_iso": "2026-08-30T12:00:00+09:00",
+            "checked_at_label": "2026년 8월 30일 12:00",
+            "reviewed_at_iso": "2026-08-30T12:30:00+09:00",
+            "reviewed_at_label": "2026년 8월 30일 12:30",
+            "freshness_state": "stale",
+            "freshness_label": "마지막 검토 자료 · 새 확인 필요",
+        },
+    }


