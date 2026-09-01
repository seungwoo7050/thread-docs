# 입국요건·여행경보의 독립 게시 읽기 모델과 신선도 판정

## `feat(web): render independent publication cards`

diff --git a/public_web/results.py b/public_web/results.py
new file mode 100644
index 0000000..f859077
--- /dev/null
+++ b/public_web/results.py
@@ -0,0 +1,295 @@
+from __future__ import annotations
+
+from datetime import UTC, date, datetime
+from typing import Callable
+
+from django.db.models import Exists, OuterRef
+from django.http import HttpRequest, HttpResponse
+from django.shortcuts import render
+from django.urls import reverse
+from django.views.decorators.http import require_GET
+
+from reviews.models import (
+    ENTRY_POINTER_ID,
+    WARNING_POINTER_ID,
+    PublishedEntryFacts,
+    PublishedTravelWarning,
+)
+from sources.models import FetchAttempt, SourceArtifact, SourceConfiguration
+
+
+CARD_READY = "ready"
+CARD_EMPTY = "empty"
+CARD_UNAVAILABLE = "unavailable"
+CARD_STALE = "stale"
+CARD_SERVER_ERROR = "server-error"
+
+
+def _fixed_redirect(route_name: str) -> HttpResponse:
+    return HttpResponse(
+        status=303,
+        headers={
+            "Location": reverse(route_name),
+            "Cache-Control": "no-store",
+        },
+    )
+
+
+def _state_card(module: str, state: str) -> dict[str, object]:
+    is_entry = module == "entry"
+    heading = "입국요건 사실" if is_entry else "여행경보"
+    labels = {
+        CARD_EMPTY: (
+            "게시 전",
+            "아직 검수·게시된 source 사실이 없습니다. 공식 source 확인이 필요합니다.",
+        ),
+        CARD_UNAVAILABLE: (
+            "정보 확인 필요",
+            "게시 경계를 확인할 수 없습니다. 공식 source에서 직접 확인해 주세요.",
+        ),
+        CARD_SERVER_ERROR: (
+            "일시적 오류",
+            "이 정보를 지금 읽을 수 없습니다. 다른 카드는 계속 확인할 수 있습니다.",
+        ),
+    }
+    status_label, message = labels[state]
+    return {
+        "module": module,
+        "heading": heading,
+        "state": state,
+        "status_label": status_label,
+        "message": message,
+        "has_publication": False,
+    }
+
+
+def _iso_date(value: date | None) -> str | None:
+    return value.isoformat() if value is not None else None
+
+
+def _utc_minute(value: datetime | None) -> str | None:
+    if value is None:
+        return None
+    return value.astimezone(UTC).strftime("%Y-%m-%d %H:%M UTC")
+
+
+def _freshness(
+    *,
+    artifact_id,
+    source_id,
+    source_revision: str,
+    source_state: str,
+    source_enabled: bool,
+) -> tuple[str, datetime | None]:
+    matching_body = SourceArtifact.objects.filter(
+        id=artifact_id,
+        body_sha256=OuterRef("response_sha256"),
+    )
+    terminal_attempts = FetchAttempt.objects.filter(
+        source_id=source_id,
+        source_revision=source_revision,
+        completed_at__isnull=False,
+    ).annotate(matches_publication=Exists(matching_body))
+    latest = (
+        terminal_attempts.order_by("-completed_at", "-started_at", "-id")
+        .values("id", "result", "matches_publication")
+        .first()
+    )
+    last_matching_success = (
+        terminal_attempts.filter(
+            result=FetchAttempt.Result.SUCCEEDED,
+            matches_publication=True,
+        )
+        .order_by("-completed_at", "-started_at", "-id")
+        .values("id", "completed_at")
+        .first()
+    )
+    stale = (
+        source_state != SourceConfiguration.State.ACTIVE
+        or not source_enabled
+        or latest is None
+        or last_matching_success is None
+        or latest["id"] != last_matching_success["id"]
+    )
+    checked_at = (
+        None
+        if last_matching_success is None
+        else last_matching_success["completed_at"]
+    )
+    return (CARD_STALE if stale else CARD_READY), checked_at
+
+
+def _entry_row() -> dict | None:
+    return (
+        PublishedEntryFacts.objects.filter(id=ENTRY_POINTER_ID)
+        .values(
+            "current_publication_id",
+            "current_publication__generation",
+            "current_publication__published_at",
+            "current_publication__scope_country__name_ko",
+            "current_publication__source_revision",
+            "current_publication__source_owner_snapshot",
+            "current_publication__source_locator_snapshot",
+            "current_publication__attribution_text_snapshot",
+            "current_publication__entry_fact_revision__ordinary_passport_period_text",
+            "current_publication__entry_fact_revision__basis_text",
+            "current_publication__entry_fact_revision__snapshot_date",
+            "current_publication__entry_fact_revision__parse_run__artifact_id",
+            "current_publication__entry_fact_revision__parse_run__artifact__source_id",
+            "current_publication__entry_fact_revision__parse_run__artifact__source__revision",
+            "current_publication__entry_fact_revision__parse_run__artifact__source__state",
+            "current_publication__entry_fact_revision__parse_run__artifact__source__enabled",
+        )
+        .first()
+    )
+
+
+def _warning_row() -> dict | None:
+    return (
+        PublishedTravelWarning.objects.filter(id=WARNING_POINTER_ID)
+        .values(
+            "current_publication_id",
+            "current_publication__generation",
+            "current_publication__published_at",
+            "current_publication__scope_country__name_ko",
+            "current_publication__source_revision",
+            "current_publication__source_owner_snapshot",
+            "current_publication__source_locator_snapshot",
+            "current_publication__attribution_text_snapshot",
+            "current_publication__travel_warning_revision__source_alarm_level_code",
+            "current_publication__travel_warning_revision__source_scope_type",
+            "current_publication__travel_warning_revision__source_scope_text",
+            "current_publication__travel_warning_revision__source_written_date",
+            "current_publication__travel_warning_revision__parse_run__artifact_id",
+            "current_publication__travel_warning_revision__parse_run__artifact__source_id",
+            "current_publication__travel_warning_revision__parse_run__artifact__source__revision",
+            "current_publication__travel_warning_revision__parse_run__artifact__source__state",
+            "current_publication__travel_warning_revision__parse_run__artifact__source__enabled",
+        )
+        .first()
+    )
+
+
+def _load_entry_card() -> dict[str, object]:
+    row = _entry_row()
+    if row is None:
+        return _state_card("entry", CARD_UNAVAILABLE)
+    if row["current_publication_id"] is None:
+        return _state_card("entry", CARD_EMPTY)
+    prefix = "current_publication__entry_fact_revision__parse_run__artifact"
+    state, checked_at = _freshness(
+        artifact_id=row[f"{prefix}_id"],
+        source_id=row[f"{prefix}__source_id"],
+        source_revision=row[f"{prefix}__source__revision"],
+        source_state=row[f"{prefix}__source__state"],
+        source_enabled=row[f"{prefix}__source__enabled"],
+    )
+    stale = state == CARD_STALE
+    return {
+        "module": "entry",
+        "heading": "입국요건 사실",
+        "state": state,
+        "status_label": "재확인 필요" if stale else "게시된 source 사실",
+        "message": (
+            "마지막 검수·게시 사실입니다. 더 최근 조회 또는 source 상태를 재확인해 주세요."
+            if stale
+            else "공식 source의 검수·게시 사실입니다."
+        ),
+        "has_publication": True,
+        "country_name": row["current_publication__scope_country__name_ko"],
+        "generation": row["current_publication__generation"],
+        "published_at": _utc_minute(row["current_publication__published_at"]),
+        "source_revision": row["current_publication__source_revision"],
+        "source_owner": row["current_publication__source_owner_snapshot"],
+        "source_locator": row["current_publication__source_locator_snapshot"],
+        "attribution": row["current_publication__attribution_text_snapshot"],
+        "checked_at": _utc_minute(checked_at),
+        "period_text": row[
+            "current_publication__entry_fact_revision__ordinary_passport_period_text"
+        ],
+        "basis_text": row[
+            "current_publication__entry_fact_revision__basis_text"
+        ],
+        "snapshot_date": _iso_date(
+            row["current_publication__entry_fact_revision__snapshot_date"]
+        ),
+    }
+
+
+def _load_warning_card() -> dict[str, object]:
+    row = _warning_row()
+    if row is None:
+        return _state_card("warning", CARD_UNAVAILABLE)
+    if row["current_publication_id"] is None:
+        return _state_card("warning", CARD_EMPTY)
+    prefix = "current_publication__travel_warning_revision__parse_run__artifact"
+    state, checked_at = _freshness(
+        artifact_id=row[f"{prefix}_id"],
+        source_id=row[f"{prefix}__source_id"],
+        source_revision=row[f"{prefix}__source__revision"],
+        source_state=row[f"{prefix}__source__state"],
+        source_enabled=row[f"{prefix}__source__enabled"],
+    )
+    stale = state == CARD_STALE
+    return {
+        "module": "warning",
+        "heading": "여행경보",
+        "state": state,
+        "status_label": "재확인 필요" if stale else "게시된 source 사실",
+        "message": (
+            "마지막 검수·게시 사실입니다. 더 최근 조회 또는 source 상태를 재확인해 주세요."
+            if stale
+            else "입국요건과 독립된 공식 source의 검수·게시 사실입니다."
+        ),
+        "has_publication": True,
+        "country_name": row["current_publication__scope_country__name_ko"],
+        "generation": row["current_publication__generation"],
+        "published_at": _utc_minute(row["current_publication__published_at"]),
+        "source_revision": row["current_publication__source_revision"],
+        "source_owner": row["current_publication__source_owner_snapshot"],
+        "source_locator": row["current_publication__source_locator_snapshot"],
+        "attribution": row["current_publication__attribution_text_snapshot"],
+        "checked_at": _utc_minute(checked_at),
+        "alarm_level_code": row[
+            "current_publication__travel_warning_revision__source_alarm_level_code"
+        ],
+        "scope_type": row[
+            "current_publication__travel_warning_revision__source_scope_type"
+        ],
+        "scope_text": row[
+            "current_publication__travel_warning_revision__source_scope_text"
+        ],
+        "written_date": _iso_date(
+            row[
+                "current_publication__travel_warning_revision__source_written_date"
+            ]
+        ),
+    }
+
+
+def _safe_card(
+    module: str,
+    loader: Callable[[], dict[str, object]],
+) -> dict[str, object]:
+    try:
+        return loader()
+    except Exception:
+        return _state_card(module, CARD_SERVER_ERROR)
+
+
+@require_GET
+def results(request: HttpRequest) -> HttpResponse:
+    if request.GET:
+        return _fixed_redirect("public_web:results")
+    entry_card = _safe_card("entry", _load_entry_card)
+    warning_card = _safe_card("warning", _load_warning_card)
+    response = render(
+        request,
+        "public_web/results.html",
+        {
+            "entry_card": entry_card,
+            "warning_card": warning_card,
+        },
+    )
+    response.headers["Cache-Control"] = "no-store"
+    return response
diff --git a/public_web/templates/public_web/results.html b/public_web/templates/public_web/results.html
new file mode 100644
index 0000000..b48a972
--- /dev/null
+++ b/public_web/templates/public_web/results.html
@@ -0,0 +1,79 @@
+<!doctype html>
+<html lang="ko">
+<head>
+  <meta charset="utf-8">
+  <meta name="viewport" content="width=device-width, initial-scale=1">
+  <title>여행준비 — 일본 게시 정보</title>
+</head>
+<body>
+  <header>
+    <nav aria-label="주요 메뉴">
+      <a href="/">처음으로</a>
+      <a href="{% url 'public_web:results' %}" aria-current="page">게시 정보</a>
+    </nav>
+  </header>
+  <main id="main-content">
+    <h1>일본 게시 정보</h1>
+    <p>
+      고정된 일본 publication만 표시합니다. 여행 목적과 날짜에 대한 적용 여부는
+      계산하지 않으므로 공식 기관 확인이 필요합니다.
+    </p>
+
+    <section aria-label="독립 publication 결과">
+      <article id="entry-card" data-state="{{ entry_card.state }}" aria-labelledby="entry-heading">
+        <h2 id="entry-heading">{{ entry_card.heading }}</h2>
+        <p role="status"><strong>상태: {{ entry_card.status_label }}</strong></p>
+        <p>{{ entry_card.message }}</p>
+        {% if entry_card.has_publication %}
+          <dl>
+            <dt>국가</dt><dd>{{ entry_card.country_name }}</dd>
+            <dt>일반여권 source 표기</dt><dd>{{ entry_card.period_text }}</dd>
+            <dt>source 근거 문구</dt><dd>{{ entry_card.basis_text }}</dd>
+            <dt>snapshot date</dt><dd>{{ entry_card.snapshot_date }}</dd>
+            <dt>마지막 성공 확인시각</dt><dd>{{ entry_card.checked_at|default:"확인 필요" }}</dd>
+            <dt>publication revision</dt><dd>generation {{ entry_card.generation }}</dd>
+            <dt>게시시각</dt><dd>{{ entry_card.published_at }}</dd>
+            <dt>source revision</dt><dd>{{ entry_card.source_revision }}</dd>
+            <dt>출처</dt>
+            <dd>
+              {{ entry_card.source_owner }} · {{ entry_card.attribution }}
+              <a href="{{ entry_card.source_locator }}" rel="noopener noreferrer"
+                 aria-label="외교부 입국요건 source 열기">공식 source</a>
+            </dd>
+          </dl>
+          <p><strong>확인 필요:</strong> 여행 목적·날짜 적용성과 최신 조건은 source에서 다시 확인해 주세요.</p>
+        {% endif %}
+      </article>
+
+      <article id="warning-card" data-state="{{ warning_card.state }}" aria-labelledby="warning-heading">
+        <h2 id="warning-heading">{{ warning_card.heading }}</h2>
+        <p role="status"><strong>상태: {{ warning_card.status_label }}</strong></p>
+        <p>{{ warning_card.message }}</p>
+        {% if warning_card.has_publication %}
+          <dl>
+            <dt>국가</dt><dd>{{ warning_card.country_name }}</dd>
+            <dt>source 경보 단계 코드</dt><dd>{{ warning_card.alarm_level_code }}</dd>
+            <dt>source 범위 유형</dt><dd>{{ warning_card.scope_type }}</dd>
+            <dt>source 범위</dt><dd>{{ warning_card.scope_text }}</dd>
+            <dt>source 작성일</dt><dd>{{ warning_card.written_date|default:"source가 제공하지 않음" }}</dd>
+            <dt>마지막 성공 확인시각</dt><dd>{{ warning_card.checked_at|default:"확인 필요" }}</dd>
+            <dt>publication revision</dt><dd>generation {{ warning_card.generation }}</dd>
+            <dt>게시시각</dt><dd>{{ warning_card.published_at }}</dd>
+            <dt>source revision</dt><dd>{{ warning_card.source_revision }}</dd>
+            <dt>출처</dt>
+            <dd>
+              {{ warning_card.source_owner }} · {{ warning_card.attribution }}
+              <a href="{{ warning_card.source_locator }}" rel="noopener noreferrer"
+                 aria-label="외교부 여행경보 source 열기">공식 source</a>
+            </dd>
+          </dl>
+          <p><strong>확인 필요:</strong> 단계 명칭, 발효·종료 시각과 여행일 적용성은 source에서 다시 확인해 주세요.</p>
+        {% endif %}
+      </article>
+    </section>
+  </main>
+  <footer>
+    <p>두 카드는 서로 독립된 검수·게시 경계를 사용합니다.</p>
+  </footer>
+</body>
+</html>
diff --git a/public_web/tests/__init__.py b/public_web/tests/__init__.py
new file mode 100644
index 0000000..e69de29
diff --git a/public_web/tests/test_results.py b/public_web/tests/test_results.py
new file mode 100644
index 0000000..1091639
--- /dev/null
+++ b/public_web/tests/test_results.py
@@ -0,0 +1,195 @@
+from unittest.mock import patch
+
+from django.contrib.auth import get_user_model
+from django.contrib.auth.models import Permission
+from django.db import DatabaseError
+from django.test import TransactionTestCase
+from django.urls import reverse
+
+from entry_requirements.models import EntryFactRevision
+from reviews.models import (
+    PublicationModule,
+    PublishedEntryFacts,
+    PublishedTravelWarning,
+)
+from reviews.publication import (
+    PublicationCode,
+    publish_candidate,
+    rollback_publication,
+)
+from reviews.tests.test_publication import PublicationFixtureMixin
+from sources.models import SourceConfiguration
+
+
+class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
+    reset_sequences = True
+
+    def setUp(self):
+        self.seed_boundaries()
+        self.actor = get_user_model().objects.create_user(
+            username="public-web-reviewer",
+            password=None,
+            is_staff=True,
+        )
+        self.actor.user_permissions.add(
+            *Permission.objects.filter(
+                content_type__app_label="reviews",
+                codename__in=(
+                    "publish_entry",
+                    "publish_warning",
+                    "rollback_entry",
+                    "rollback_warning",
+                ),
+            )
+        )
+
+    def publish_entry(self, *, period="90일") -> EntryFactRevision:
+        entry = self.make_entry(period=period)
+        pointer = PublishedEntryFacts.objects.get()
+        outcome = publish_candidate(
+            module=PublicationModule.ENTRY,
+            typed_revision_id=entry.id,
+            actor=self.actor,
+            expected_pointer_version=pointer.version,
+        )
+        self.assertEqual(outcome.code, PublicationCode.PUBLISHED)
+        return entry
+
+    def publish_warning(self, *, scope_text="합성 검증 범위"):
+        warning = self.make_warning(scope_text=scope_text)
+        pointer = PublishedTravelWarning.objects.get()
+        outcome = publish_candidate(
+            module=PublicationModule.TRAVEL_WARNING,
+            typed_revision_id=warning.id,
+            actor=self.actor,
+            expected_pointer_version=pointer.version,
+        )
+        self.assertEqual(outcome.code, PublicationCode.PUBLISHED)
+        return warning
+
+    def test_empty_cards_are_independent_semantic_states(self):
+        response = self.client.get(reverse("public_web:results"))
+
+        self.assertEqual(response.status_code, 200)
+        self.assertContains(response, 'id="entry-card" data-state="empty"')
+        self.assertContains(response, 'id="warning-card" data-state="empty"')
+        self.assertContains(response, "아직 검수·게시된 source 사실이 없습니다", count=2)
+        self.assertNotIn("sessionid", response.cookies)
+
+    def test_ready_cards_render_only_approved_source_facts_and_limits(self):
+        self.publish_entry()
+        self.publish_warning(scope_text="긴 한국어 검증 범위와 & 기호")
+
+        response = self.client.get(reverse("public_web:results"))
+
+        self.assertEqual(response.status_code, 200)
+        self.assertContains(response, 'id="entry-card" data-state="ready"')
+        self.assertContains(response, 'id="warning-card" data-state="ready"')
+        self.assertContains(response, "일반여권 source 표기")
+        self.assertContains(response, "90일")
+        self.assertContains(response, "snapshot date")
+        self.assertContains(response, "2025-01-20")
+        self.assertContains(response, "source 경보 단계 코드")
+        self.assertContains(response, "긴 한국어 검증 범위와 &amp; 기호")
+        self.assertContains(response, "source가 제공하지 않음")
+        self.assertContains(response, "마지막 성공 확인시각", count=2)
+        self.assertContains(response, "publication revision", count=2)
+        self.assertContains(response, "확인 필요")
+        self.assertContains(response, "외교부|공공데이터포털", count=2)
+
+        forbidden = (
+            "ALLOW" + "ED",
+            "DENI" + "ED",
+            "입국 " + "가능",
+            "법적 " + "판단",
+        )
+        body = response.content.decode("utf-8")
+        for phrase in forbidden:
+            self.assertNotIn(phrase, body)
+        for sensitive_name in (
+            "MOFA_TRAVEL_ALARM_SERVICE_" + "KEY",
+            "body_" + "sha256",
+            "response_" + "sha256",
+        ):
+            self.assertNotIn(sensitive_name, body)
+
+    def test_warning_text_is_template_escaped(self):
+        marker = '<script data-marker="unsafe">alert(1)</script>'
+        self.publish_warning(scope_text=marker)
+
+        response = self.client.get(reverse("public_web:results"))
+
+        self.assertEqual(response.status_code, 200)
+        self.assertNotContains(response, marker)
+        self.assertContains(
+            response,
+            "&lt;script data-marker=&quot;unsafe&quot;&gt;alert(1)&lt;/script&gt;",
+        )
+
+    def test_entry_stale_does_not_change_warning_empty_state(self):
+        self.publish_entry()
+        source = SourceConfiguration.objects.get(code="MOFA_ENTRY_CSV")
+        source.state = SourceConfiguration.State.PAUSED
+        source.enabled = False
+        source.save(update_fields=("state", "enabled"))
+
+        response = self.client.get(reverse("public_web:results"))
+
+        self.assertContains(response, 'id="entry-card" data-state="stale"')
+        self.assertContains(response, 'id="warning-card" data-state="empty"')
+        self.assertContains(response, "재확인 필요")
+        self.assertContains(response, "90일")
+
+    def test_unavailable_and_server_error_are_isolated(self):
+        self.publish_warning()
+        with (
+            patch("public_web.results._entry_row", return_value=None),
+            patch(
+                "public_web.results._load_warning_card",
+                side_effect=DatabaseError("synthetic warning read failure"),
+            ),
+        ):
+            response = self.client.get(reverse("public_web:results"))
+
+        self.assertEqual(response.status_code, 200)
+        self.assertContains(
+            response,
+            'id="entry-card" data-state="unavailable"',
+        )
+        self.assertContains(
+            response,
+            'id="warning-card" data-state="server-error"',
+        )
+        self.assertNotContains(response, "synthetic warning read failure")
+
+    def test_rollback_restores_entry_html_without_changing_warning(self):
+        first = self.publish_entry(period="90일")
+        first_publication_id = (
+            PublishedEntryFacts.objects.get().current_publication_id
+        )
+        self.publish_entry(period="30일")
+        warning = self.publish_warning(scope_text="독립 경보 범위")
+        entry_pointer = PublishedEntryFacts.objects.get()
+
+        rolled_back = rollback_publication(
+            module=PublicationModule.ENTRY,
+            target_publication_id=first_publication_id,
+            actor=self.actor,
+            expected_pointer_version=entry_pointer.version,
+        )
+
+        self.assertEqual(rolled_back.code, PublicationCode.ROLLED_BACK)
+        response = self.client.get(reverse("public_web:results"))
+        self.assertContains(response, "90일")
+        self.assertNotContains(response, "30일")
+        self.assertContains(response, "독립 경보 범위")
+        self.assertEqual(
+            PublishedTravelWarning.objects.get()
+            .current_publication.travel_warning_revision_id,
+            warning.id,
+        )
+        self.assertEqual(
+            PublishedEntryFacts.objects.get()
+            .current_publication.entry_fact_revision_id,
+            first.id,
+        )
diff --git a/public_web/urls.py b/public_web/urls.py
new file mode 100644
index 0000000..5223dea
--- /dev/null
+++ b/public_web/urls.py
@@ -0,0 +1,10 @@
+from django.urls import path
+
+from . import results
+
+
+app_name = "public_web"
+
+urlpatterns = [
+    path("results/", results.results, name="results"),
+]
diff --git a/travel_readiness/urls.py b/travel_readiness/urls.py
index 8650156..a147719 100644
--- a/travel_readiness/urls.py
+++ b/travel_readiness/urls.py
@@ -1,3 +1,6 @@
 from django.urls import include, path
 
-urlpatterns = [path("", include("operations.urls"))]
+urlpatterns = [
+    path("", include("public_web.urls")),
+    path("", include("operations.urls")),
+]


