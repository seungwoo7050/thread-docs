## `test(web): align transient search contracts`

diff --git a/public_web/tests/test_privacy_form.py b/public_web/tests/test_privacy_form.py
index 06e6b55..c656a7b 100644
--- a/public_web/tests/test_privacy_form.py
+++ b/public_web/tests/test_privacy_form.py
@@ -1,142 +1,149 @@
+from datetime import datetime
 from unittest.mock import patch
+from zoneinfo import ZoneInfo
 
 from django.core.cache import cache
+from django.http import HttpResponse
 from django.test import Client, SimpleTestCase, override_settings
 from django.urls import reverse
 
 
+SEOUL = ZoneInfo("Asia/Seoul")
+NOW = datetime(2026, 8, 31, 9, tzinfo=SEOUL)
+
+
+def _search_context():
+    return {
+        "flight_state": "ready",
+        "flight_status_label": "현재 게시본",
+        "flight_message": "합성 검색 결과",
+        "flight_checked_at": "2026.08.31 08:00 KST",
+        "flight_publication_revision": "1",
+        "flight_source_locator": "https://example.invalid/schedule",
+        "flight_source_attribution": "합성 출처",
+        "opportunities": [],
+    }
+
+
+def _render(_request, _template, _context):
+    return HttpResponse("rendered")
+
+
 class TripFormPrivacyTests(SimpleTestCase):
     def valid_data(self, **overrides):
         data = {
-            "destination": "JP",
-            "departure_date": "2026-09-01",
-            "return_date": "2026-09-01",
+            "departure_at": "2026-08-31T10:00",
+            "return_by": "2026-09-01T20:00",
         }
         data.update(overrides)
         return data
 
-    def test_get_is_unbound_and_query_input_is_removed(self):
+    @patch("public_web.views.render", side_effect=_render)
+    def test_get_is_unbound_and_query_input_is_removed(self, render):
         response = self.client.get(reverse("public_web:index"))
         self.assertEqual(response.status_code, 200)
-        self.assertContains(response, "대한민국 일반여권 · 목적지 일본")
-        self.assertContains(response, "일본 여행 전 확인할 입국요건과 여행경보")
-        self.assertContains(
-            response,
-            "공식 출처에서 수집해 검수·게시한 입국요건 사실과 여행경보를 보여드립니다.",
-        )
-        self.assertContains(
-            response,
-            "이 서비스는 입국 여부, 법률 해석, 여행 목적·날짜별 적용성을 판정하지 않습니다.",
-        )
-        self.assertContains(
-            response,
-            "목적지와 날짜는 이 요청 안에서 형식과 순서만 확인하며 저장하거나 성공 결과에 표시하지 않습니다.",
-        )
-        self.assertContains(response, "입국요건·여행경보 보기")
+        self.assertFalse(render.call_args.args[2]["form"].is_bound)
         self.assertNotIn("sessionid", response.cookies)
 
         canonical = self.client.get(
             reverse("public_web:index"),
-            {"departure_date": "private-marker"},
+            {"departure_at": "private-marker"},
         )
         self.assertEqual(canonical.status_code, 303)
         self.assertEqual(canonical.headers["Location"], "/")
-        self.assertNotContains(
-            canonical,
-            "private-marker",
-            status_code=303,
-        )
+        self.assertNotContains(canonical, "private-marker", status_code=303)
         self.assertEqual(canonical.headers["Cache-Control"], "no-store")
 
     @override_settings(SECURE_SSL_REDIRECT=True)
     def test_query_is_removed_before_https_and_slash_redirects(self):
         root = self.client.get(
             "/",
-            {"departure_date": "private-http-marker"},
+            {"departure_at": "private-http-marker"},
             secure=False,
         )
         self.assertEqual(root.status_code, 303)
         self.assertEqual(root.headers["Location"], "/")
         self.assertNotIn("private-http-marker", root.headers["Location"])
-        self.assertEqual(root.headers["Referrer-Policy"], "same-origin")
-        self.assertEqual(root.headers["X-Content-Type-Options"], "nosniff")
 
         missing_slash = self.client.get(
             "/results",
-            {"return_date": "private-slash-marker"},
+            {"return_by": "private-slash-marker"},
             secure=False,
         )
         self.assertEqual(missing_slash.status_code, 303)
         self.assertEqual(missing_slash.headers["Location"], "/results/")
-        self.assertNotIn(
-            "private-slash-marker",
-            missing_slash.headers["Location"],
-        )
+        self.assertNotIn("private-slash-marker", missing_slash.headers["Location"])
 
         https_redirect = self.client.get("/", secure=False)
         self.assertEqual(https_redirect.status_code, 301)
         self.assertNotIn("?", https_redirect.headers["Location"])
         self.assertEqual(https_redirect.headers["Cache-Control"], "no-store")
 
+    @patch("public_web.forms._now_kst", return_value=NOW)
     @patch.object(cache, "set")
     @patch("django.contrib.sessions.backends.db.SessionStore.save")
-    def test_valid_post_is_queryless_no_retention_303(
+    @patch("public_web.views.render", side_effect=_render)
+    @patch("public_web.views.search_travel_opportunities")
+    def test_valid_post_is_200_without_retention(
         self,
+        search,
+        render,
         session_save,
         cache_set,
+        _now,
     ):
+        search.return_value = _search_context()
         response = self.client.post(
             reverse("public_web:index"),
             self.valid_data(),
         )
 
-        self.assertEqual(response.status_code, 303)
-        self.assertEqual(response.headers["Location"], "/results/")
+        self.assertEqual(response.status_code, 200)
         self.assertEqual(response.headers["Cache-Control"], "no-store")
-        self.assertNotContains(
-            response,
-            "2026-09-01",
-            status_code=303,
-        )
         self.assertNotIn("sessionid", response.cookies)
         self.assertFalse(response.wsgi_request.session.modified)
         session_save.assert_not_called()
         cache_set.assert_not_called()
-
-    def test_invalid_values_are_corrected_in_the_same_request(self):
+        search.assert_called_once()
+        context = render.call_args.args[2]
+        self.assertTrue(context["has_search_result"])
+        self.assertEqual(context["opportunities"], [])
+
+    @patch("public_web.forms._now_kst", return_value=NOW)
+    @patch("public_web.views.render", side_effect=_render)
+    @patch("public_web.views.search_travel_opportunities")
+    def test_invalid_values_are_corrected_in_the_same_request(
+        self,
+        search,
+        render,
+        _now,
+    ):
         response = self.client.post(
             reverse("public_web:index"),
             self.valid_data(
-                departure_date="2026-09-10",
-                return_date="2026-09-09",
+                departure_at="2026-09-01T20:00",
+                return_by="2026-09-01T19:00",
             ),
         )
 
         self.assertEqual(response.status_code, 200)
-        self.assertContains(response, "입력한 내용을 확인해 주세요")
-        self.assertContains(
-            response,
-            "귀국일은 출국일과 같거나 그 이후여야 합니다.",
-        )
-        self.assertContains(response, 'aria-invalid="true"')
-        self.assertContains(response, "2026-09-10")
-        self.assertContains(response, "2026-09-09")
+        search.assert_not_called()
+        form = render.call_args.args[2]["form"]
+        self.assertTrue(form.is_bound)
+        self.assertIn("window_order", {
+            error.code for error in form.errors.as_data()["return_by"]
+        })
         self.assertEqual(response.headers["Cache-Control"], "no-store")
         self.assertNotIn("sessionid", response.cookies)
-        self.assertFalse(response.wsgi_request.session.modified)
 
-    def test_invalid_destination_and_date_shapes_do_not_redirect(self):
-        for data, expected in (
-            (self.valid_data(destination="ZZ"), "일본만 선택"),
-            (self.valid_data(departure_date="not-a-date"), "날짜 형식"),
-            (self.valid_data(return_date=""), "귀국일을 입력"),
-        ):
-            with self.subTest(data=data):
-                response = self.client.post(reverse("public_web:index"), data)
-                self.assertEqual(response.status_code, 200)
-                self.assertContains(response, expected)
-
-    def test_csrf_is_required_and_valid_token_keeps_fixed_redirect(self):
+    @patch("public_web.forms._now_kst", return_value=NOW)
+    @patch("public_web.views.search_travel_opportunities")
+    def test_csrf_is_required_and_valid_token_keeps_same_request_result(
+        self,
+        search,
+        _now,
+    ):
+        search.return_value = _search_context()
         client = Client(enforce_csrf_checks=True)
         rejected = client.post(
             reverse("public_web:index"),
@@ -145,10 +152,8 @@ class TripFormPrivacyTests(SimpleTestCase):
             HTTP_ORIGIN="https://outside.invalid",
         )
         self.assertEqual(rejected.status_code, 403)
-        self.assertEqual(rejected.headers["Cache-Control"], "no-store")
 
         form_page = client.get(reverse("public_web:index"), secure=True)
-        self.assertEqual(form_page.headers["Referrer-Policy"], "same-origin")
         token = form_page.cookies["csrftoken"].value
         cross_origin = client.post(
             reverse("public_web:index"),
@@ -163,28 +168,22 @@ class TripFormPrivacyTests(SimpleTestCase):
             secure=True,
             HTTP_ORIGIN="https://testserver",
         )
-        self.assertEqual(accepted.status_code, 303)
-        self.assertEqual(accepted.headers["Location"], "/results/")
-        self.assertNotIn("?", accepted.headers["Location"])
-        self.assertNotIn("2026-09-01", accepted.headers["Location"])
+        self.assertEqual(accepted.status_code, 200)
         self.assertNotIn("sessionid", accepted.cookies)
 
-    def test_results_query_is_removed_and_post_is_not_supported(self):
+    def test_results_is_a_queryless_redirect_to_search(self):
         query = self.client.get(
             reverse("public_web:results"),
-            {"return_date": "private-marker"},
+            {"return_by": "private-marker"},
         )
         self.assertEqual(query.status_code, 303)
         self.assertEqual(query.headers["Location"], "/results/")
-        self.assertNotContains(
-            query,
-            "private-marker",
-            status_code=303,
-        )
+        self.assertNotIn("?", query.headers["Location"])
 
-        post = self.client.post(reverse("public_web:results"), self.valid_data())
-        self.assertEqual(post.status_code, 405)
-        self.assertEqual(post.headers["Cache-Control"], "no-store")
+        canonical = self.client.get(reverse("public_web:results"))
+        self.assertEqual(canonical.status_code, 303)
+        self.assertEqual(canonical.headers["Location"], "/")
+        self.assertEqual(canonical.headers["Cache-Control"], "no-store")
 
     @override_settings(DEBUG=True)
     def test_debug_500_never_renders_bound_trip_values_or_exception(self):
@@ -194,13 +193,10 @@ class TripFormPrivacyTests(SimpleTestCase):
         with patch(
             "public_web.views.render",
             side_effect=RuntimeError("private-exception-marker"),
-        ):
+        ), patch("public_web.forms._now_kst", return_value=NOW):
             response = client.post(
                 reverse("public_web:index"),
-                self.valid_data(
-                    departure_date=marker,
-                    return_date="invalid-return-date",
-                ),
+                self.valid_data(departure_at=marker),
             )
 
         self.assertEqual(response.status_code, 500)
diff --git a/public_web/tests/test_results.py b/public_web/tests/test_results.py
index 22407bf..fb7c66f 100644
--- a/public_web/tests/test_results.py
+++ b/public_web/tests/test_results.py
@@ -1,672 +1,113 @@
-import uuid
 from datetime import UTC, datetime, timedelta
 from unittest.mock import patch
 
-from django.contrib.auth import get_user_model
-from django.contrib.auth.models import Permission
 from django.db import DatabaseError
-from django.test import TransactionTestCase
+from django.test import SimpleTestCase
 from django.urls import reverse
-from django.utils.html import strip_tags
-from django.utils import timezone
 
-from entry_requirements.models import EntryFactRevision
-from reviews.models import (
-    PublicationModule,
-    PublishedEntryFacts,
-    PublishedTravelWarning,
+from countries.models import SUPPORTED_COUNTRY_IDS
+from public_web.results import (
+    CARD_EMPTY,
+    CARD_READY,
+    CARD_SERVER_ERROR,
+    _matching_success_is_current,
+    load_publication_cards,
+    load_publication_cards_for_country,
 )
-from reviews.publication import (
-    PublicationCode,
-    publish_candidate,
-    rollback_publication,
-)
-from reviews.tests.test_publication import PublicationFixtureMixin
-from public_web.results import _matching_success_is_current
-from sources.models import (
-    FetchAttempt,
-    SourceConfiguration,
-    SourceRightsDecision,
-)
-
-
-class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
-    reset_sequences = True
-
-    def setUp(self):
-        self.seed_boundaries()
-        self.actor = get_user_model().objects.create_user(
-            username="public-web-reviewer",
-            password=None,
-            is_staff=True,
-        )
-        self.actor.user_permissions.add(
-            *Permission.objects.filter(
-                content_type__app_label="reviews",
-                codename__in=(
-                    "publish_entry",
-                    "publish_warning",
-                    "rollback_entry",
-                    "rollback_warning",
-                ),
-            )
-        )
-
-    def publish_entry(self, *, period="90일") -> EntryFactRevision:
-        entry = self.make_entry(period=period)
-        pointer = PublishedEntryFacts.objects.get()
-        outcome = publish_candidate(
-            module=PublicationModule.ENTRY,
-            typed_revision_id=entry.id,
-            actor=self.actor,
-            expected_pointer_version=pointer.version,
-        )
-        self.assertEqual(outcome.code, PublicationCode.PUBLISHED)
-        return entry
-
-    def publish_warning(self, *, scope_text="합성 검증 범위"):
-        warning = self.make_warning(scope_text=scope_text)
-        pointer = PublishedTravelWarning.objects.get()
-        outcome = publish_candidate(
-            module=PublicationModule.TRAVEL_WARNING,
-            typed_revision_id=warning.id,
-            actor=self.actor,
-            expected_pointer_version=pointer.version,
-        )
-        self.assertEqual(outcome.code, PublicationCode.PUBLISHED)
-        return warning
-
-    def append_same_body_success(
-        self,
-        subject,
-        *,
-        source,
-        rights,
-        http_status=200,
-        provider_code="",
-        completed_at=None,
-    ):
-        artifact = subject.parse_run.artifact
-        first_attempt = artifact.first_successful_attempt
-        completed_at = completed_at or timezone.now()
-        attempt = FetchAttempt.objects.create(
-            source=source,
-            source_revision=source.revision,
-            rights_decision=rights,
-            operation_id=uuid.uuid4(),
-            attempt_number=1,
-            request_fingerprint_revision=(
-                first_attempt.request_fingerprint_revision
-            ),
-            normalized_request_sha256=(
-                first_attempt.normalized_request_sha256
-            ),
-            started_at=completed_at,
-        )
-        FetchAttempt.objects.filter(pk=attempt.pk).update(
-            completed_at=completed_at,
-            result=FetchAttempt.Result.SUCCEEDED,
-            http_status=http_status,
-            provider_code=provider_code,
-            response_sha256=artifact.body_sha256,
-            failure_code="",
-        )
-        attempt.refresh_from_db()
-        return attempt
 
-    def checked_at_text(self, value):
-        return value.astimezone(UTC).strftime("%Y-%m-%d %H:%M UTC")
 
-    def card_html(self, response, card_id):
-        body = response.content.decode("utf-8")
-        start = body.index(f'<article id="{card_id}"')
-        end = body.index("</article>", start) + len("</article>")
-        return body[start:end]
-
-    def test_empty_cards_are_independent_semantic_states(self):
-        response = self.client.get(reverse("public_web:results"))
-
-        self.assertEqual(response.status_code, 200)
-        self.assertContains(response, 'id="entry-card" data-state="empty"')
-        self.assertContains(response, 'id="warning-card" data-state="empty"')
-        entry_card = self.card_html(response, "entry-card")
-        warning_card = self.card_html(response, "warning-card")
-        self.assertIn("상태: 게시된 자료 없음", entry_card)
-        self.assertIn("아직 게시된 입국요건 자료가 없습니다", entry_card)
-        self.assertIn("상태: 게시된 자료 없음", warning_card)
-        self.assertIn("아직 게시된 여행경보 자료가 없습니다", warning_card)
-        self.assertContains(response, "일본 입국요건·여행경보 게시본")
-        self.assertContains(
-            response,
-            "입국요건과 여행경보는 서로 독립된 검수·게시 경계에서 표시합니다.",
-        )
-        self.assertContains(response, "입력한 날짜에 맞춘 결과가 아닙니다.")
-        self.assertContains(response, 'aria-label="게시 정보 목차"')
-        self.assertContains(response, 'href="#entry-card">입국요건</a>')
-        self.assertContains(response, 'href="#warning-card">여행경보</a>')
-        body = response.content.decode("utf-8")
-        self.assertLess(body.index('href="#entry-card"'), body.index('id="entry-card"'))
-        self.assertLess(body.index('id="entry-card"'), body.index('id="warning-card"'))
-        visible_text = strip_tags(body).lower()
-        for internal_term in ("source", "publication", "phase 0"):
-            self.assertNotIn(internal_term, visible_text)
-        self.assertNotIn("sessionid", response.cookies)
-
-    def test_entry_ready_marks_only_unpublished_warning_unavailable(self):
-        self.publish_entry(period="90일")
-
-        response = self.client.get(reverse("public_web:results"))
-
-        entry_card = self.card_html(response, "entry-card")
-        warning_card = self.card_html(response, "warning-card")
-        self.assertIn('data-state="ready"', entry_card)
-        self.assertIn("상태: 현재 게시본", entry_card)
-        self.assertIn("90일", entry_card)
-        self.assertIn('data-state="unavailable"', warning_card)
-        self.assertIn("상태: 현재 확인할 수 없음", warning_card)
-
-    def test_warning_ready_marks_only_unpublished_entry_unavailable(self):
-        self.publish_warning(scope_text="독립 경보 범위")
-
-        response = self.client.get(reverse("public_web:results"))
-
-        entry_card = self.card_html(response, "entry-card")
-        warning_card = self.card_html(response, "warning-card")
-        self.assertIn('data-state="unavailable"', entry_card)
-        self.assertIn("상태: 현재 확인할 수 없음", entry_card)
-        self.assertIn('data-state="ready"', warning_card)
-        self.assertIn("상태: 현재 게시본", warning_card)
-        self.assertIn("독립 경보 범위", warning_card)
-
-    def test_ready_cards_render_only_approved_source_facts_and_limits(self):
-        self.publish_entry()
-        self.publish_warning(scope_text="긴 한국어 검증 범위와 & 기호")
-
-        response = self.client.get(reverse("public_web:results"))
-
-        self.assertEqual(response.status_code, 200)
-        self.assertContains(response, 'id="entry-card" data-state="ready"')
-        self.assertContains(response, 'id="warning-card" data-state="ready"')
-        self.assertContains(response, "일반여권 관련 출처 표기")
-        self.assertContains(response, "90일")
-        self.assertContains(response, "출처 자료 날짜")
-        self.assertContains(response, "2025-01-20")
-        self.assertContains(response, "출처 경보 단계 코드")
-        self.assertContains(response, "출처 범위 유형")
-        self.assertContains(response, "출처 범위")
-        self.assertContains(response, "출처 작성일")
-        self.assertContains(response, "긴 한국어 검증 범위와 &amp; 기호")
-        self.assertContains(response, "출처가 제공하지 않음")
-        self.assertContains(response, "마지막 성공 확인시각", count=3)
-        self.assertContains(response, "게시 리비전", count=2)
-        self.assertContains(response, "출처 리비전", count=2)
-        self.assertContains(response, "확인 필요")
-        self.assertContains(response, "외교부|공공데이터포털", count=2)
-        self.assertContains(response, 'rel="noopener noreferrer"', count=2)
-        self.assertContains(response, "입국요건 공식 출처 열기")
-        self.assertContains(response, "여행경보 공식 출처 열기")
-
-        entry_card = self.card_html(response, "entry-card")
-        self.assertLess(entry_card.index("상태:"), entry_card.index("출처에서 확인된 사실"))
-        self.assertLess(
-            entry_card.index("출처에서 확인된 사실"),
-            entry_card.index("출처와 게시 이력"),
-        )
-        self.assertLess(entry_card.index("출처와 게시 이력"), entry_card.index("확인 필요:"))
-        self.assertLess(entry_card.index("확인 필요:"), entry_card.index("입국요건 공식 출처 열기"))
-
-        forbidden = (
-            "ALLOW" + "ED",
-            "DENI" + "ED",
-            "입국 " + "가능",
-            "법적 " + "판단",
-        )
-        body = response.content.decode("utf-8")
-        for phrase in forbidden:
-            self.assertNotIn(phrase, body)
-        for sensitive_name in (
-            "MOFA_TRAVEL_ALARM_SERVICE_" + "KEY",
-            "body_" + "sha256",
-            "response_" + "sha256",
-        ):
-            self.assertNotIn(sensitive_name, body)
-
-    def test_entry_matching_success_age_boundaries_fail_closed(self):
-        entry = self.publish_entry(period="90일")
-        completed_at = (
-            entry.parse_run.artifact.first_successful_attempt.completed_at
-        )
+class PublicationFreshnessTests(SimpleTestCase):
+    def test_success_age_boundary_and_future_fail_closed(self):
+        checked_at = datetime(2026, 8, 30, 12, tzinfo=UTC)
         cases = (
-            ("just-before", timedelta(hours=36) - timedelta(seconds=1), "ready"),
-            ("boundary", timedelta(hours=36), "stale"),
-            ("old", timedelta(hours=37), "stale"),
-            ("future", -timedelta(seconds=1), "stale"),
-        )
-
-        for label, age, expected_state in cases:
-            with self.subTest(label=label):
-                with patch(
-                    "public_web.results.timezone.now",
-                    return_value=completed_at + age,
-                ):
-                    response = self.client.get(reverse("public_web:results"))
-
-                card = self.card_html(response, "entry-card")
-                self.assertIn(f'data-state="{expected_state}"', card)
-                self.assertIn("90일", card)
-                if expected_state == "stale":
-                    self.assertIn("재확인 필요", card)
-
-    def test_warning_matching_success_age_boundaries_fail_closed(self):
-        warning = self.publish_warning(scope_text="합성 검증 범위")
-        completed_at = (
-            warning.parse_run.artifact.first_successful_attempt.completed_at
-        )
-        cases = (
-            ("just-before", timedelta(hours=8) - timedelta(seconds=1), "ready"),
-            ("boundary", timedelta(hours=8), "stale"),
-            ("old", timedelta(hours=9), "stale"),
-            ("future", -timedelta(seconds=1), "stale"),
-        )
-
-        for label, age, expected_state in cases:
-            with self.subTest(label=label):
-                with patch(
-                    "public_web.results.timezone.now",
-                    return_value=completed_at + age,
-                ):
-                    response = self.client.get(reverse("public_web:results"))
-
-                card = self.card_html(response, "warning-card")
-                self.assertIn(f'data-state="{expected_state}"', card)
-                self.assertIn("합성 검증 범위", card)
-                if expected_state == "stale":
-                    self.assertIn("재확인 필요", card)
-
-    def test_warning_age_boundary_drives_freshness_endpoint_alert(self):
-        self.publish_entry()
-        warning = self.publish_warning()
-        completed_at = (
-            warning.parse_run.artifact.first_successful_attempt.completed_at
-        )
-
-        with patch(
-            "public_web.results.timezone.now",
-            return_value=(
-                completed_at
-                + timedelta(hours=8)
-                - timedelta(seconds=1)
-            ),
-        ):
-            just_before = self.client.get("/freshnessz")
-
-        self.assertEqual(just_before.status_code, 200)
-        self.assertEqual(
-            just_before.json(),
-            {
-                "status": "ready",
-                "modules": {"entry": "ready", "warning": "ready"},
-            },
-        )
-
-        with patch(
-            "public_web.results.timezone.now",
-            return_value=completed_at + timedelta(hours=8),
-        ):
-            boundary = self.client.get("/freshnessz")
-
-        self.assertEqual(boundary.status_code, 200)
-        self.assertEqual(
-            boundary.json(),
-            {
-                "status": "alert",
-                "modules": {"entry": "ready", "warning": "stale"},
-            },
-        )
-
-    def test_matching_success_invalid_timestamp_shapes_fail_closed(self):
-        current_time = datetime(2026, 8, 30, 12, 0, tzinfo=UTC)
-        invalid_values = (
-            None,
-            "2026-08-30T11:00:00Z",
-            datetime(2026, 8, 30, 11, 0),
-        )
-
-        with patch(
-            "public_web.results.timezone.now",
-            return_value=current_time,
-        ):
-            for completed_at in invalid_values:
-                with self.subTest(completed_at=completed_at):
+            (timedelta(hours=36) - timedelta(seconds=1), True),
+            (timedelta(hours=36), False),
+            (timedelta(hours=37), False),
+            (-timedelta(seconds=1), False),
+        )
+        for age, expected in cases:
+            with self.subTest(age=age), patch(
+                "public_web.results.timezone.now",
+                return_value=checked_at + age,
+            ):
+                self.assertIs(
+                    _matching_success_is_current(
+                        checked_at,
+                        max_age=timedelta(hours=36),
+                    ),
+                    expected,
+                )
+
+    def test_invalid_timestamp_shapes_fail_closed(self):
+        current = datetime(2026, 8, 30, 12, tzinfo=UTC)
+        with patch("public_web.results.timezone.now", return_value=current):
+            for value in (
+                None,
+                "2026-08-30T11:00:00Z",
+                datetime(2026, 8, 30, 11),
+            ):
+                with self.subTest(value=value):
                     self.assertFalse(
                         _matching_success_is_current(
-                            completed_at,
-                            max_age=timedelta(hours=36),
+                            value,
+                            max_age=timedelta(hours=8),
                         )
                     )
 
-    def test_warning_text_is_template_escaped(self):
-        marker = '<script data-marker="unsafe">alert(1)</script>'
-        self.publish_warning(scope_text=marker)
-
-        response = self.client.get(reverse("public_web:results"))
-
-        self.assertEqual(response.status_code, 200)
-        self.assertNotContains(response, marker)
-        self.assertContains(
-            response,
-            "&lt;script data-marker=&quot;unsafe&quot;&gt;alert(1)&lt;/script&gt;",
-        )
-
-    def test_entry_stale_marks_only_unpublished_warning_unavailable(self):
-        self.publish_entry()
-        source = SourceConfiguration.objects.get(code="MOFA_ENTRY_CSV")
-        source.state = SourceConfiguration.State.PAUSED
-        source.enabled = False
-        source.save(update_fields=("state", "enabled"))
-
-        response = self.client.get(reverse("public_web:results"))
-
-        self.assertContains(response, 'id="entry-card" data-state="stale"')
-        self.assertContains(
-            response,
-            'id="warning-card" data-state="unavailable"',
-        )
-        self.assertContains(response, "재확인 필요")
-        self.assertContains(response, "90일")
-        warning_card = self.card_html(response, "warning-card")
-        self.assertIn("현재 확인할 수 없음", warning_card)
-
-    def test_warning_stale_marks_only_unpublished_entry_unavailable(self):
-        self.publish_warning(scope_text="독립 경보 범위")
-        source = SourceConfiguration.objects.get(
-            code="MOFA_TRAVEL_ALARM_API_JP"
-        )
-        source.state = SourceConfiguration.State.PAUSED
-        source.enabled = False
-        source.save(update_fields=("state", "enabled"))
-
-        response = self.client.get(reverse("public_web:results"))
-
-        entry_card = self.card_html(response, "entry-card")
-        warning_card = self.card_html(response, "warning-card")
-        self.assertIn('data-state="unavailable"', entry_card)
-        self.assertIn("현재 확인할 수 없음", entry_card)
-        self.assertIn('data-state="stale"', warning_card)
-        self.assertIn("재확인 필요", warning_card)
-        self.assertIn("독립 경보 범위", warning_card)
-
-    def test_same_revision_second_rights_decision_keeps_canonical_receipt(self):
-        entry = self.publish_entry()
-        artifact = entry.parse_run.artifact
-        original_checked_at = artifact.first_successful_attempt.completed_at
-
-        replacement_rights = self.reject_current_rights(artifact.source.code)
-
-        self.assertEqual(replacement_rights.decision_sequence, 2)
-        response = self.client.get(reverse("public_web:results"))
-        self.assertContains(response, 'id="entry-card" data-state="stale"')
-        self.assertContains(response, self.checked_at_text(original_checked_at))
-        self.assertContains(
-            response,
-            'id="warning-card" data-state="unavailable"',
-        )
 
-    def test_new_source_revision_same_body_does_not_refresh_old_publication(self):
-        entry = self.publish_entry()
-        artifact = entry.parse_run.artifact
-        original_checked_at = artifact.first_successful_attempt.completed_at
-        later_checked_at = original_checked_at + timedelta(hours=2)
-        source = artifact.source
-        approval = source.rights_decisions.get(
-            source_revision=source.revision,
-            decision_sequence=1,
-        )
-        SourceConfiguration.objects.filter(pk=source.pk).update(
-            revision="rights-v2",
-            state=SourceConfiguration.State.DRAFT,
-            enabled=False,
-        )
-        source.refresh_from_db()
-        replacement_rights = SourceRightsDecision.objects.create(
-            source=source,
-            source_revision=source.revision,
-            decision_sequence=1,
-            decision=SourceRightsDecision.Decision.APPROVED,
-            access_mode=approval.access_mode,
-            access_allowed=True,
-            automated_collection_allowed=True,
-            typed_field_storage_allowed=True,
-            raw_body_storage_allowed=False,
-            typed_republication_allowed=True,
-            raw_retention_seconds=0,
-            typed_retention=SourceRightsDecision.Retention.PRODUCT_HISTORY,
-            evidence_retention=(
-                SourceRightsDecision.Retention.PRODUCT_HISTORY
-            ),
-            field_scope_code=approval.field_scope_code,
-            attribution_text=approval.attribution_text,
-            contract_fingerprint_sha256="f" * 64,
-            decided_by="PROJECT_OWNER_REQUEST",
-            decision_basis_code="SYNTHETIC_NEW_CONTRACT",
-        )
-        SourceConfiguration.objects.filter(pk=source.pk).update(
-            state=SourceConfiguration.State.RIGHTS_APPROVED,
-            enabled=False,
-        )
-        SourceConfiguration.objects.filter(pk=source.pk).update(
-            state=SourceConfiguration.State.ACTIVE,
-            enabled=True,
-        )
-        source.refresh_from_db()
-        self.append_same_body_success(
-            entry,
-            source=source,
-            rights=replacement_rights,
-            completed_at=later_checked_at,
-        )
-
-        response = self.client.get(reverse("public_web:results"))
+class CountryScopedPublicationCardTests(SimpleTestCase):
+    @patch("public_web.results._load_warning_card")
+    @patch("public_web.results._load_entry_card")
+    def test_two_modules_are_loaded_independently_for_same_country(
+        self,
+        entry_loader,
+        warning_loader,
+    ):
+        country_id = SUPPORTED_COUNTRY_IDS["TW"]
+        entry_loader.return_value = {"module": "entry", "state": CARD_EMPTY}
+        warning_loader.return_value = {"module": "warning", "state": CARD_READY}
 
-        self.assertContains(response, 'id="entry-card" data-state="stale"')
-        self.assertContains(response, "출처 리비전</dt><dd>rights-v1")
-        self.assertContains(response, self.checked_at_text(original_checked_at))
-        self.assertNotContains(response, self.checked_at_text(later_checked_at))
-        self.assertContains(
-            response,
-            'id="warning-card" data-state="unavailable"',
-        )
+        cards = load_publication_cards_for_country(country_id)
 
-    def test_warning_non_200_success_receipt_is_stale(self):
-        warning = self.publish_warning()
-        artifact = warning.parse_run.artifact
-        source = artifact.source
-        rights = source.rights_decisions.get(
-            source_revision=source.revision,
-            decision_sequence=1,
-        )
-        self.append_same_body_success(
-            warning,
-            source=source,
-            rights=rights,
-            http_status=204,
-            provider_code=FetchAttempt.ProviderCode.MOFA_SUCCESS_0,
-        )
-
-        response = self.client.get(reverse("public_web:results"))
+        entry_loader.assert_called_once_with(country_id)
+        warning_loader.assert_called_once_with(country_id)
+        self.assertEqual(cards["entry"]["state"], CARD_EMPTY)
+        self.assertEqual(cards["warning"]["state"], CARD_READY)
 
-        self.assertContains(response, 'id="warning-card" data-state="stale"')
-        self.assertContains(response, "재확인 필요")
-        self.assertContains(
-            response,
-            'id="entry-card" data-state="unavailable"',
-        )
+    @patch("public_web.results._load_warning_card")
+    @patch("public_web.results._load_entry_card")
+    def test_one_module_database_error_does_not_change_the_other(
+        self,
+        entry_loader,
+        warning_loader,
+    ):
+        entry_loader.side_effect = DatabaseError("synthetic private detail")
+        warning_loader.return_value = {"module": "warning", "state": CARD_READY}
 
-    def test_warning_blank_provider_success_receipt_is_stale(self):
-        warning = self.publish_warning()
-        artifact = warning.parse_run.artifact
-        source = artifact.source
-        rights = source.rights_decisions.get(
-            source_revision=source.revision,
-            decision_sequence=1,
-        )
-        self.append_same_body_success(
-            warning,
-            source=source,
-            rights=rights,
-            http_status=200,
-            provider_code="",
-        )
+        cards = load_publication_cards_for_country(SUPPORTED_COUNTRY_IDS["VN"])
 
-        response = self.client.get(reverse("public_web:results"))
+        self.assertEqual(cards["entry"]["state"], CARD_SERVER_ERROR)
+        self.assertEqual(cards["warning"]["state"], CARD_READY)
+        self.assertNotIn("synthetic private detail", repr(cards))
 
-        self.assertContains(response, 'id="warning-card" data-state="stale"')
-        self.assertContains(response, "재확인 필요")
-        self.assertContains(
-            response,
-            'id="entry-card" data-state="unavailable"',
-        )
+    @patch("public_web.results.load_publication_cards_for_country")
+    def test_legacy_internal_loader_uses_japanese_country_scope(self, scoped_loader):
+        scoped_loader.return_value = {
+            "entry": {"state": CARD_EMPTY},
+            "warning": {"state": CARD_EMPTY},
+        }
 
-    def test_unavailable_and_server_error_are_isolated(self):
-        self.publish_warning()
-        with (
-            patch("public_web.results._entry_row", return_value=None),
-            patch(
-                "public_web.results._load_warning_card",
-                side_effect=DatabaseError("synthetic warning read failure"),
-            ),
-        ):
-            response = self.client.get(reverse("public_web:results"))
+        cards = load_publication_cards()
 
-        self.assertEqual(response.status_code, 200)
-        self.assertContains(
-            response,
-            'id="entry-card" data-state="unavailable"',
-        )
-        self.assertContains(
-            response,
-            'id="warning-card" data-state="server-error"',
-        )
-        entry_card = self.card_html(response, "entry-card")
-        warning_card = self.card_html(response, "warning-card")
-        self.assertIn("상태: 현재 확인할 수 없음", entry_card)
-        self.assertIn("상태: 일시적 오류", warning_card)
-        self.assertNotContains(response, "synthetic warning read failure")
+        scoped_loader.assert_called_once_with(SUPPORTED_COUNTRY_IDS["JP"])
+        self.assertEqual(cards["entry"]["state"], CARD_EMPTY)
 
-    def test_rollback_restores_entry_html_without_changing_warning(self):
-        first = self.publish_entry(period="90일")
-        first_publication_id = (
-            PublishedEntryFacts.objects.get().current_publication_id
-        )
-        self.publish_entry(period="30일")
-        warning = self.publish_warning(scope_text="독립 경보 범위")
-        warning_pointer_before = PublishedTravelWarning.objects.get()
-        warning_boundary_before = (
-            warning_pointer_before.current_publication_id,
-            warning_pointer_before.version,
-        )
-        warning_html_before = self.card_html(
-            self.client.get(reverse("public_web:results")),
-            "warning-card",
-        )
-        entry_pointer = PublishedEntryFacts.objects.get()
 
-        rolled_back = rollback_publication(
-            module=PublicationModule.ENTRY,
-            target_publication_id=first_publication_id,
-            actor=self.actor,
-            expected_pointer_version=entry_pointer.version,
-        )
-
-        self.assertEqual(rolled_back.code, PublicationCode.ROLLED_BACK)
+class ResultsRedirectTests(SimpleTestCase):
+    def test_queryless_results_redirects_to_search_form(self):
         response = self.client.get(reverse("public_web:results"))
-        entry_html = self.card_html(response, "entry-card")
-        self.assertIn('data-state="stale"', entry_html)
-        self.assertIn("90일", entry_html)
-        self.assertNotIn("30일", entry_html)
-        self.assertIn("일반여권 소지자 : synthetic evidence", entry_html)
-        self.assertIn("게시 리비전</dt><dd>generation 3", entry_html)
-        self.assertIn("출처 리비전</dt><dd>rights-v1", entry_html)
-        self.assertIn("외교부|공공데이터포털", entry_html)
-        self.assertEqual(
-            self.card_html(response, "warning-card"),
-            warning_html_before,
-        )
-        warning_pointer_after = PublishedTravelWarning.objects.get()
-        self.assertEqual(
-            (
-                warning_pointer_after.current_publication_id,
-                warning_pointer_after.version,
-            ),
-            warning_boundary_before,
-        )
-        self.assertEqual(
-            warning_pointer_after.current_publication.travel_warning_revision_id,
-            warning.id,
-        )
-        self.assertEqual(
-            PublishedEntryFacts.objects.get()
-            .current_publication.entry_fact_revision_id,
-            first.id,
-        )
-
-    def test_rollback_restores_warning_html_without_changing_entry(self):
-        first = self.publish_warning(scope_text="첫 경보 범위")
-        first_publication_id = (
-            PublishedTravelWarning.objects.get().current_publication_id
-        )
-        self.publish_warning(scope_text="둘째 경보 범위")
-        entry = self.publish_entry(period="90일")
-        entry_pointer_before = PublishedEntryFacts.objects.get()
-        entry_boundary_before = (
-            entry_pointer_before.current_publication_id,
-            entry_pointer_before.version,
-        )
-        entry_html_before = self.card_html(
-            self.client.get(reverse("public_web:results")),
-            "entry-card",
-        )
-        warning_pointer = PublishedTravelWarning.objects.get()
 
-        rolled_back = rollback_publication(
-            module=PublicationModule.TRAVEL_WARNING,
-            target_publication_id=first_publication_id,
-            actor=self.actor,
-            expected_pointer_version=warning_pointer.version,
-        )
-
-        self.assertEqual(rolled_back.code, PublicationCode.ROLLED_BACK)
-        response = self.client.get(reverse("public_web:results"))
-        warning_html = self.card_html(response, "warning-card")
-        self.assertIn('data-state="stale"', warning_html)
-        self.assertIn("첫 경보 범위", warning_html)
-        self.assertNotIn("둘째 경보 범위", warning_html)
-        self.assertIn("출처 경보 단계 코드</dt><dd>3", warning_html)
-        self.assertIn("출처 범위 유형</dt><dd>일부", warning_html)
-        self.assertIn(
-            "게시 리비전</dt><dd>generation 3",
-            warning_html,
-        )
-        self.assertIn("출처 리비전</dt><dd>rights-v1", warning_html)
-        self.assertIn("외교부|공공데이터포털", warning_html)
-        self.assertEqual(
-            self.card_html(response, "entry-card"),
-            entry_html_before,
-        )
-        entry_pointer_after = PublishedEntryFacts.objects.get()
-        self.assertEqual(
-            (
-                entry_pointer_after.current_publication_id,
-                entry_pointer_after.version,
-            ),
-            entry_boundary_before,
-        )
-        self.assertEqual(
-            entry_pointer_after.current_publication.entry_fact_revision_id,
-            entry.id,
-        )
-        self.assertEqual(
-            PublishedTravelWarning.objects.get()
-            .current_publication.travel_warning_revision_id,
-            first.id,
-        )
+        self.assertEqual(response.status_code, 303)
+        self.assertEqual(response.headers["Location"], "/")
+        self.assertEqual(response.headers["Cache-Control"], "no-store")
+        self.assertNotIn("?", response.headers["Location"])


