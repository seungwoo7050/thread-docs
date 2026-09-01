## `fix(reviews): keep the operator site dormant`

diff --git a/docs/IMPLEMENTATION-PLAN.md b/docs/IMPLEMENTATION-PLAN.md
index 98b8c82..2a65b0c 100644
--- a/docs/IMPLEMENTATION-PLAN.md
+++ b/docs/IMPLEMENTATION-PLAN.md
@@ -105,6 +105,12 @@ typed field와 rights/evidence record만 제품 이력 동안 보존합니다. 
 runtime의 최초 rights row는 이 contract와 승인 근거를 immutable하게 보존하며 Django Admin에서
 수정할 수 없습니다.
 
+현재 검수 Admin은 별도 `DormantOperatorAdminSite`에만 등록하고 root URL에 mount하지 않습니다.
+이 site는 실수로 URL이 추가돼도 모든 request를 거부합니다. 로컬 source→검수→publication
+rehearsal은 동일 publication service를 직접 사용합니다. production platform의 identity-aware
+proxy, MFA 등록·복구, assertion 전달·제거 계약과 실제 operator provisioning이 사람 승인을
+통과하기 전에는 Admin route와 로그인 surface를 활성화하지 않습니다.
+
 ### 여행경보
 
 공식 metadata는 `https://www.data.go.kr/data/15076237/openapi.do`이고 OpenAPI 명은
diff --git a/reviews/admin.py b/reviews/admin.py
index 11f8d3f..f7218e2 100644
--- a/reviews/admin.py
+++ b/reviews/admin.py
@@ -1,4 +1,9 @@
-"""Least-privilege operator boundary for immutable publication history."""
+"""Dormant operator boundary for immutable publication history.
+
+The site is deliberately unmounted and denies every request until a human
+approves the production identity-aware MFA proxy contract. Local rehearsals
+call the same publication service without exposing a login surface.
+"""
 
 from __future__ import annotations
 
@@ -75,6 +80,22 @@ _OUTCOME_MESSAGES = {
 }
 
 
+class DormantOperatorAdminSite(admin.AdminSite):
+    """Fail closed even if the dormant site is mounted accidentally."""
+
+    site_header = "여행준비 검수"
+    site_title = "여행준비 검수"
+    index_title = "검수 대기 항목"
+
+    def has_permission(self, request):
+        return False
+
+
+operator_admin_site = DormantOperatorAdminSite(
+    name="travel_readiness_operator_disabled"
+)
+
+
 def _operator_has_permission(request, permission: str) -> bool:
     try:
         user = request.user
@@ -211,7 +232,7 @@ class _ReadOnlyAdmin(admin.ModelAdmin):
         return True
 
 
-@admin.register(EntryFactRevision)
+@admin.register(EntryFactRevision, site=operator_admin_site)
 class EntryFactRevisionAdmin(_ReadOnlyAdmin):
     fields = (
         "id",
@@ -279,7 +300,7 @@ class EntryFactRevisionAdmin(_ReadOnlyAdmin):
         self._message_action_result(request, result)
 
 
-@admin.register(TravelWarningRevision)
+@admin.register(TravelWarningRevision, site=operator_admin_site)
 class TravelWarningRevisionAdmin(_ReadOnlyAdmin):
     fields = (
         "id",
@@ -339,7 +360,7 @@ class TravelWarningRevisionAdmin(_ReadOnlyAdmin):
         self._message_action_result(request, result)
 
 
-@admin.register(ReviewDecision)
+@admin.register(ReviewDecision, site=operator_admin_site)
 class ReviewDecisionAdmin(_ReadOnlyAdmin):
     fields = (
         "id",
@@ -361,7 +382,7 @@ class ReviewDecisionAdmin(_ReadOnlyAdmin):
         return obj.entry_fact_revision_id or obj.travel_warning_revision_id
 
 
-@admin.register(PublicationRevision)
+@admin.register(PublicationRevision, site=operator_admin_site)
 class PublicationRevisionAdmin(_ReadOnlyAdmin):
     fields = (
         "id",
@@ -516,7 +537,7 @@ class PublicationRevisionAdmin(_ReadOnlyAdmin):
         self._message_action_result(request, result)
 
 
-@admin.register(AuditEvent)
+@admin.register(AuditEvent, site=operator_admin_site)
 class AuditEventAdmin(_ReadOnlyAdmin):
     fields = (
         "id",
diff --git a/reviews/tests/test_admin.py b/reviews/tests/test_admin.py
index 492b666..2f5ff06 100644
--- a/reviews/tests/test_admin.py
+++ b/reviews/tests/test_admin.py
@@ -4,6 +4,7 @@ from unittest.mock import patch
 
 from django.contrib import admin, messages
 from django.test import RequestFactory, SimpleTestCase
+from django.urls import NoReverseMatch, Resolver404, resolve, reverse
 
 from entry_requirements.models import EntryFactRevision
 from travel_warnings.models import TravelWarningRevision
@@ -16,6 +17,7 @@ from reviews.admin import (
     TravelWarningRevisionAdmin,
     _GENERIC_FAILURE_MESSAGE,
     _OUTCOME_MESSAGES,
+    operator_admin_site,
 )
 from reviews.models import (
     AuditEvent,
@@ -79,7 +81,21 @@ class AdminBoundaryTests(SimpleTestCase):
             PublicationRevision,
             AuditEvent,
         ):
-            self.assertIn(model, admin.site._registry)
+            self.assertIn(model, operator_admin_site._registry)
+            self.assertNotIn(model, admin.site._registry)
+
+    def test_operator_site_is_dormant_and_unmounted(self):
+        request = self.request(
+            "reviews.publish_entry",
+            "reviews.publish_warning",
+            "reviews.rollback_entry",
+            "reviews.rollback_warning",
+        )
+        self.assertFalse(operator_admin_site.has_permission(request))
+        with self.assertRaises(NoReverseMatch):
+            reverse("travel_readiness_operator_disabled:index")
+        with self.assertRaises(Resolver404):
+            resolve("/admin/")
 
     def test_admin_surfaces_exclude_transport_and_identity_hash_fields(self):
         forbidden = {


## `fix(reviews): refuse dormant admin routes`

diff --git a/docs/IMPLEMENTATION-PLAN.md b/docs/IMPLEMENTATION-PLAN.md
index 2a65b0c..4c91359 100644
--- a/docs/IMPLEMENTATION-PLAN.md
+++ b/docs/IMPLEMENTATION-PLAN.md
@@ -106,7 +106,8 @@ runtime의 최초 rights row는 이 contract와 승인 근거를 immutable하게
 수정할 수 없습니다.
 
 현재 검수 Admin은 별도 `DormantOperatorAdminSite`에만 등록하고 root URL에 mount하지 않습니다.
-이 site는 실수로 URL이 추가돼도 모든 request를 거부합니다. 로컬 source→검수→publication
+project default AdminSite도 같은 dormant class를 사용하며, 둘 다 URLConf 생성 자체를 거부하므로
+실수로 `site.urls`를 추가해도 애플리케이션이 fail closed합니다. 로컬 source→검수→publication
 rehearsal은 동일 publication service를 직접 사용합니다. production platform의 identity-aware
 proxy, MFA 등록·복구, assertion 전달·제거 계약과 실제 operator provisioning이 사람 승인을
 통과하기 전에는 Admin route와 로그인 surface를 활성화하지 않습니다.
diff --git a/reviews/admin.py b/reviews/admin.py
index f7218e2..6eaf9d4 100644
--- a/reviews/admin.py
+++ b/reviews/admin.py
@@ -8,6 +8,7 @@ call the same publication service without exposing a login surface.
 from __future__ import annotations
 
 from django.contrib import admin, messages
+from django.core.exceptions import ImproperlyConfigured
 
 from entry_requirements.models import EntryFactRevision
 from travel_warnings.models import TravelWarningRevision
@@ -81,7 +82,7 @@ _OUTCOME_MESSAGES = {
 
 
 class DormantOperatorAdminSite(admin.AdminSite):
-    """Fail closed even if the dormant site is mounted accidentally."""
+    """Refuse to construct routes before the MFA proxy checkpoint."""
 
     site_header = "여행준비 검수"
     site_title = "여행준비 검수"
@@ -90,6 +91,11 @@ class DormantOperatorAdminSite(admin.AdminSite):
     def has_permission(self, request):
         return False
 
+    def get_urls(self):
+        raise ImproperlyConfigured(
+            "Operator Admin is disabled pending the approved MFA proxy gate."
+        )
+
 
 operator_admin_site = DormantOperatorAdminSite(
     name="travel_readiness_operator_disabled"
diff --git a/reviews/admin_config.py b/reviews/admin_config.py
new file mode 100644
index 0000000..2aa2907
--- /dev/null
+++ b/reviews/admin_config.py
@@ -0,0 +1,7 @@
+"""Project-wide fail-closed Django Admin configuration."""
+
+from django.contrib.admin.apps import AdminConfig
+
+
+class DormantAdminConfig(AdminConfig):
+    default_site = "reviews.admin.DormantOperatorAdminSite"
diff --git a/reviews/tests/test_admin.py b/reviews/tests/test_admin.py
index 2f5ff06..81344e4 100644
--- a/reviews/tests/test_admin.py
+++ b/reviews/tests/test_admin.py
@@ -3,6 +3,7 @@ from types import SimpleNamespace
 from unittest.mock import patch
 
 from django.contrib import admin, messages
+from django.core.exceptions import ImproperlyConfigured
 from django.test import RequestFactory, SimpleTestCase
 from django.urls import NoReverseMatch, Resolver404, resolve, reverse
 
@@ -92,8 +93,14 @@ class AdminBoundaryTests(SimpleTestCase):
             "reviews.rollback_warning",
         )
         self.assertFalse(operator_admin_site.has_permission(request))
+        with self.assertRaises(ImproperlyConfigured):
+            operator_admin_site.get_urls()
+        with self.assertRaises(ImproperlyConfigured):
+            admin.site.get_urls()
         with self.assertRaises(NoReverseMatch):
             reverse("travel_readiness_operator_disabled:index")
+        with self.assertRaises(NoReverseMatch):
+            reverse("admin:index")
         with self.assertRaises(Resolver404):
             resolve("/admin/")
 
diff --git a/travel_readiness/settings.py b/travel_readiness/settings.py
index 8869428..0699270 100644
--- a/travel_readiness/settings.py
+++ b/travel_readiness/settings.py
@@ -41,7 +41,7 @@ INSTALLED_APPS = [
     "travel_warnings",
     "reviews",
     "public_web",
-    "django.contrib.admin",
+    "reviews.admin_config.DormantAdminConfig",
     "django.contrib.auth",
     "django.contrib.contenttypes",
     "django.contrib.sessions",


## `test(web): prove independent publication rollback`

diff --git a/public_web/tests/test_results.py b/public_web/tests/test_results.py
index d85c6a9..0624454 100644
--- a/public_web/tests/test_results.py
+++ b/public_web/tests/test_results.py
@@ -115,6 +115,12 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
     def checked_at_text(self, value):
         return value.astimezone(UTC).strftime("%Y-%m-%d %H:%M UTC")
 
+    def card_html(self, response, card_id):
+        body = response.content.decode("utf-8")
+        start = body.index(f'<article id="{card_id}"')
+        end = body.index("</article>", start) + len("</article>")
+        return body[start:end]
+
     def test_empty_cards_are_independent_semantic_states(self):
         response = self.client.get(reverse("public_web:results"))
 
@@ -336,6 +342,15 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
         )
         self.publish_entry(period="30일")
         warning = self.publish_warning(scope_text="독립 경보 범위")
+        warning_pointer_before = PublishedTravelWarning.objects.get()
+        warning_boundary_before = (
+            warning_pointer_before.current_publication_id,
+            warning_pointer_before.version,
+        )
+        warning_html_before = self.card_html(
+            self.client.get(reverse("public_web:results")),
+            "warning-card",
+        )
         entry_pointer = PublishedEntryFacts.objects.get()
 
         rolled_back = rollback_publication(
@@ -347,12 +362,28 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
 
         self.assertEqual(rolled_back.code, PublicationCode.ROLLED_BACK)
         response = self.client.get(reverse("public_web:results"))
-        self.assertContains(response, "90일")
-        self.assertNotContains(response, "30일")
-        self.assertContains(response, "독립 경보 범위")
+        entry_html = self.card_html(response, "entry-card")
+        self.assertIn('data-state="stale"', entry_html)
+        self.assertIn("90일", entry_html)
+        self.assertNotIn("30일", entry_html)
+        self.assertIn("일반여권 소지자 : synthetic evidence", entry_html)
+        self.assertIn("publication revision</dt><dd>generation 3", entry_html)
+        self.assertIn("source revision</dt><dd>rights-v1", entry_html)
+        self.assertIn("외교부|공공데이터포털", entry_html)
         self.assertEqual(
-            PublishedTravelWarning.objects.get()
-            .current_publication.travel_warning_revision_id,
+            self.card_html(response, "warning-card"),
+            warning_html_before,
+        )
+        warning_pointer_after = PublishedTravelWarning.objects.get()
+        self.assertEqual(
+            (
+                warning_pointer_after.current_publication_id,
+                warning_pointer_after.version,
+            ),
+            warning_boundary_before,
+        )
+        self.assertEqual(
+            warning_pointer_after.current_publication.travel_warning_revision_id,
             warning.id,
         )
         self.assertEqual(
@@ -360,3 +391,64 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
             .current_publication.entry_fact_revision_id,
             first.id,
         )
+
+    def test_rollback_restores_warning_html_without_changing_entry(self):
+        first = self.publish_warning(scope_text="첫 경보 범위")
+        first_publication_id = (
+            PublishedTravelWarning.objects.get().current_publication_id
+        )
+        self.publish_warning(scope_text="둘째 경보 범위")
+        entry = self.publish_entry(period="90일")
+        entry_pointer_before = PublishedEntryFacts.objects.get()
+        entry_boundary_before = (
+            entry_pointer_before.current_publication_id,
+            entry_pointer_before.version,
+        )
+        entry_html_before = self.card_html(
+            self.client.get(reverse("public_web:results")),
+            "entry-card",
+        )
+        warning_pointer = PublishedTravelWarning.objects.get()
+
+        rolled_back = rollback_publication(
+            module=PublicationModule.TRAVEL_WARNING,
+            target_publication_id=first_publication_id,
+            actor=self.actor,
+            expected_pointer_version=warning_pointer.version,
+        )
+
+        self.assertEqual(rolled_back.code, PublicationCode.ROLLED_BACK)
+        response = self.client.get(reverse("public_web:results"))
+        warning_html = self.card_html(response, "warning-card")
+        self.assertIn('data-state="stale"', warning_html)
+        self.assertIn("첫 경보 범위", warning_html)
+        self.assertNotIn("둘째 경보 범위", warning_html)
+        self.assertIn("source 경보 단계 코드</dt><dd>3", warning_html)
+        self.assertIn("source 범위 유형</dt><dd>일부", warning_html)
+        self.assertIn(
+            "publication revision</dt><dd>generation 3",
+            warning_html,
+        )
+        self.assertIn("source revision</dt><dd>rights-v1", warning_html)
+        self.assertIn("외교부|공공데이터포털", warning_html)
+        self.assertEqual(
+            self.card_html(response, "entry-card"),
+            entry_html_before,
+        )
+        entry_pointer_after = PublishedEntryFacts.objects.get()
+        self.assertEqual(
+            (
+                entry_pointer_after.current_publication_id,
+                entry_pointer_after.version,
+            ),
+            entry_boundary_before,
+        )
+        self.assertEqual(
+            entry_pointer_after.current_publication.entry_fact_revision_id,
+            entry.id,
+        )
+        self.assertEqual(
+            PublishedTravelWarning.objects.get()
+            .current_publication.travel_warning_revision_id,
+            first.id,
+        )


