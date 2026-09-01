## `fix(web): retain canonical freshness evidence`

diff --git a/public_web/results.py b/public_web/results.py
index b0f0ab6..bd815c9 100644
--- a/public_web/results.py
+++ b/public_web/results.py
@@ -169,33 +169,31 @@ def _freshness(
     source_state: str,
     source_enabled: bool,
 ) -> tuple[str, datetime | None]:
-    rights = (
-        SourceRightsDecision.objects.filter(
-            source_id=source_id,
-            source_revision=publication_source_revision,
-        )
-        .order_by("-decision_sequence", "-id")
-        .values(
-            "id",
-            "decision_sequence",
-            "decision",
-            "access_mode",
-            "access_allowed",
-            "automated_collection_allowed",
-            "typed_field_storage_allowed",
-            "raw_body_storage_allowed",
-            "typed_republication_allowed",
-            "raw_retention_seconds",
-            "typed_retention",
-            "evidence_retention",
-            "field_scope_code",
-            "attribution_text",
-            "contract_fingerprint_sha256",
-            "decided_by",
-            "decision_basis_code",
-        )
-        .first()
+    rights_fields = (
+        "id",
+        "decision_sequence",
+        "decision",
+        "access_mode",
+        "access_allowed",
+        "automated_collection_allowed",
+        "typed_field_storage_allowed",
+        "raw_body_storage_allowed",
+        "typed_republication_allowed",
+        "raw_retention_seconds",
+        "typed_retention",
+        "evidence_retention",
+        "field_scope_code",
+        "attribution_text",
+        "contract_fingerprint_sha256",
+        "decided_by",
+        "decision_basis_code",
     )
+    rights_rows = SourceRightsDecision.objects.filter(
+        source_id=source_id,
+        source_revision=publication_source_revision,
+    ).values(*rights_fields)
+    canonical_rights = rights_rows.filter(decision_sequence=1).first()
+    latest_rights = rights_rows.order_by("-decision_sequence", "-id").first()
     publication_contract_exact = (
         publication_source_code == contract.source_code == source_code
         and publication_source_revision
@@ -210,30 +208,36 @@ def _freshness(
         == contract.contract_fingerprint_sha256
         and publication_attribution == contract.attribution
     )
-    rights_exact = (
-        rights is not None
-        and rights["decision_sequence"] == 1
-        and rights["decision"] == SourceRightsDecision.Decision.APPROVED
-        and rights["access_mode"] == contract.access_mode
-        and rights["access_allowed"] is True
-        and rights["automated_collection_allowed"] is True
-        and rights["typed_field_storage_allowed"] is True
-        and rights["raw_body_storage_allowed"] is False
-        and rights["typed_republication_allowed"] is True
-        and rights["raw_retention_seconds"] == 0
-        and rights["typed_retention"]
+    canonical_rights_exact = (
+        canonical_rights is not None
+        and canonical_rights["decision_sequence"] == 1
+        and canonical_rights["decision"]
+        == SourceRightsDecision.Decision.APPROVED
+        and canonical_rights["access_mode"] == contract.access_mode
+        and canonical_rights["access_allowed"] is True
+        and canonical_rights["automated_collection_allowed"] is True
+        and canonical_rights["typed_field_storage_allowed"] is True
+        and canonical_rights["raw_body_storage_allowed"] is False
+        and canonical_rights["typed_republication_allowed"] is True
+        and canonical_rights["raw_retention_seconds"] == 0
+        and canonical_rights["typed_retention"]
         == SourceRightsDecision.Retention.PRODUCT_HISTORY
-        and rights["evidence_retention"]
+        and canonical_rights["evidence_retention"]
         == SourceRightsDecision.Retention.PRODUCT_HISTORY
-        and rights["field_scope_code"] == contract.field_scope
-        and rights["attribution_text"]
+        and canonical_rights["field_scope_code"] == contract.field_scope
+        and canonical_rights["attribution_text"]
         == publication_attribution
         == contract.attribution
-        and rights["contract_fingerprint_sha256"]
+        and canonical_rights["contract_fingerprint_sha256"]
         == publication_contract_fingerprint_sha256
         == contract.contract_fingerprint_sha256
-        and rights["decided_by"] == contract.decided_by
-        and rights["decision_basis_code"] == contract.decision_basis
+        and canonical_rights["decided_by"] == contract.decided_by
+        and canonical_rights["decision_basis_code"] == contract.decision_basis
+    )
+    rights_exact = (
+        canonical_rights_exact
+        and latest_rights is not None
+        and latest_rights["id"] == canonical_rights["id"]
     )
     matching_body = SourceArtifact.objects.filter(
         id=artifact_id,
@@ -257,13 +261,13 @@ def _freshness(
         .first()
     )
     last_matching_success = None
-    if rights is not None:
+    if canonical_rights_exact:
         last_matching_success = (
             terminal_attempts.filter(
                 result=FetchAttempt.Result.SUCCEEDED,
                 http_status=200,
                 provider_code=contract.provider_code,
-                rights_decision_id=rights["id"],
+                rights_decision_id=canonical_rights["id"],
                 matches_publication=True,
             )
             .order_by("-completed_at", "-started_at", "-id")
@@ -272,11 +276,11 @@ def _freshness(
         )
     latest_exact = (
         latest is not None
-        and rights is not None
+        and latest_rights is not None
         and latest["result"] == FetchAttempt.Result.SUCCEEDED
         and latest["http_status"] == 200
         and latest["provider_code"] == contract.provider_code
-        and latest["rights_decision_id"] == rights["id"]
+        and latest["rights_decision_id"] == latest_rights["id"]
         and latest["matches_publication"] is True
     )
     stale = (
diff --git a/public_web/tests/test_results.py b/public_web/tests/test_results.py
index f05511f..d85c6a9 100644
--- a/public_web/tests/test_results.py
+++ b/public_web/tests/test_results.py
@@ -1,4 +1,5 @@
 import uuid
+from datetime import UTC, timedelta
 from unittest.mock import patch
 
 from django.contrib.auth import get_user_model
@@ -81,9 +82,11 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
         rights,
         http_status=200,
         provider_code="",
+        completed_at=None,
     ):
         artifact = subject.parse_run.artifact
         first_attempt = artifact.first_successful_attempt
+        completed_at = completed_at or timezone.now()
         attempt = FetchAttempt.objects.create(
             source=source,
             source_revision=source.revision,
@@ -96,10 +99,10 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
             normalized_request_sha256=(
                 first_attempt.normalized_request_sha256
             ),
-            started_at=timezone.now(),
+            started_at=completed_at,
         )
         FetchAttempt.objects.filter(pk=attempt.pk).update(
-            completed_at=timezone.now(),
+            completed_at=completed_at,
             result=FetchAttempt.Result.SUCCEEDED,
             http_status=http_status,
             provider_code=provider_code,
@@ -109,6 +112,9 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
         attempt.refresh_from_db()
         return attempt
 
+    def checked_at_text(self, value):
+        return value.astimezone(UTC).strftime("%Y-%m-%d %H:%M UTC")
+
     def test_empty_cards_are_independent_semantic_states(self):
         response = self.client.get(reverse("public_web:results"))
 
@@ -182,9 +188,24 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
         self.assertContains(response, "재확인 필요")
         self.assertContains(response, "90일")
 
-    def test_new_contract_same_body_does_not_refresh_old_publication(self):
+    def test_same_revision_second_rights_decision_keeps_canonical_receipt(self):
+        entry = self.publish_entry()
+        artifact = entry.parse_run.artifact
+        original_checked_at = artifact.first_successful_attempt.completed_at
+
+        replacement_rights = self.reject_current_rights(artifact.source.code)
+
+        self.assertEqual(replacement_rights.decision_sequence, 2)
+        response = self.client.get(reverse("public_web:results"))
+        self.assertContains(response, 'id="entry-card" data-state="stale"')
+        self.assertContains(response, self.checked_at_text(original_checked_at))
+        self.assertContains(response, 'id="warning-card" data-state="empty"')
+
+    def test_new_source_revision_same_body_does_not_refresh_old_publication(self):
         entry = self.publish_entry()
         artifact = entry.parse_run.artifact
+        original_checked_at = artifact.first_successful_attempt.completed_at
+        later_checked_at = original_checked_at + timedelta(hours=2)
         source = artifact.source
         approval = source.rights_decisions.get(
             source_revision=source.revision,
@@ -231,15 +252,18 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
             entry,
             source=source,
             rights=replacement_rights,
+            completed_at=later_checked_at,
         )
 
         response = self.client.get(reverse("public_web:results"))
 
         self.assertContains(response, 'id="entry-card" data-state="stale"')
         self.assertContains(response, "source revision</dt><dd>rights-v1")
+        self.assertContains(response, self.checked_at_text(original_checked_at))
+        self.assertNotContains(response, self.checked_at_text(later_checked_at))
         self.assertContains(response, 'id="warning-card" data-state="empty"')
 
-    def test_warning_nonexact_success_receipt_is_stale(self):
+    def test_warning_non_200_success_receipt_is_stale(self):
         warning = self.publish_warning()
         artifact = warning.parse_run.artifact
         source = artifact.source
@@ -252,6 +276,28 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
             source=source,
             rights=rights,
             http_status=204,
+            provider_code=FetchAttempt.ProviderCode.MOFA_SUCCESS_0,
+        )
+
+        response = self.client.get(reverse("public_web:results"))
+
+        self.assertContains(response, 'id="warning-card" data-state="stale"')
+        self.assertContains(response, "재확인 필요")
+        self.assertContains(response, 'id="entry-card" data-state="empty"')
+
+    def test_warning_blank_provider_success_receipt_is_stale(self):
+        warning = self.publish_warning()
+        artifact = warning.parse_run.artifact
+        source = artifact.source
+        rights = source.rights_decisions.get(
+            source_revision=source.revision,
+            decision_sequence=1,
+        )
+        self.append_same_body_success(
+            warning,
+            source=source,
+            rights=rights,
+            http_status=200,
             provider_code="",
         )
 


## `fix(web): expose unavailable publication state`

diff --git a/public_web/results.py b/public_web/results.py
index 1dceeac..3598165 100644
--- a/public_web/results.py
+++ b/public_web/results.py
@@ -547,10 +547,18 @@ def _safe_card(
 def load_publication_cards() -> dict[str, dict[str, object]]:
     """Return the two fixed, independent read models without external I/O."""
 
-    return {
+    cards = {
         "entry": _safe_card("entry", _load_entry_card),
         "warning": _safe_card("warning", _load_warning_card),
     }
+    publication_states = {CARD_READY, CARD_STALE}
+    entry_state = cards["entry"].get("state")
+    warning_state = cards["warning"].get("state")
+    if entry_state in publication_states and warning_state == CARD_EMPTY:
+        cards["warning"] = _state_card("warning", CARD_UNAVAILABLE)
+    elif warning_state in publication_states and entry_state == CARD_EMPTY:
+        cards["entry"] = _state_card("entry", CARD_UNAVAILABLE)
+    return cards
 
 
 @require_GET
diff --git a/public_web/tests/test_results.py b/public_web/tests/test_results.py
index 7ac7f0f..e3e24ab 100644
--- a/public_web/tests/test_results.py
+++ b/public_web/tests/test_results.py
@@ -131,6 +131,30 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
         self.assertContains(response, "아직 검수·게시된 source 사실이 없습니다", count=2)
         self.assertNotIn("sessionid", response.cookies)
 
+    def test_entry_ready_marks_only_unpublished_warning_unavailable(self):
+        self.publish_entry(period="90일")
+
+        response = self.client.get(reverse("public_web:results"))
+
+        entry_card = self.card_html(response, "entry-card")
+        warning_card = self.card_html(response, "warning-card")
+        self.assertIn('data-state="ready"', entry_card)
+        self.assertIn("90일", entry_card)
+        self.assertIn('data-state="unavailable"', warning_card)
+        self.assertIn("게시 경계를 확인할 수 없습니다", warning_card)
+
+    def test_warning_ready_marks_only_unpublished_entry_unavailable(self):
+        self.publish_warning(scope_text="독립 경보 범위")
+
+        response = self.client.get(reverse("public_web:results"))
+
+        entry_card = self.card_html(response, "entry-card")
+        warning_card = self.card_html(response, "warning-card")
+        self.assertIn('data-state="unavailable"', entry_card)
+        self.assertIn("게시 경계를 확인할 수 없습니다", entry_card)
+        self.assertIn('data-state="ready"', warning_card)
+        self.assertIn("독립 경보 범위", warning_card)
+
     def test_ready_cards_render_only_approved_source_facts_and_limits(self):
         self.publish_entry()
         self.publish_warning(scope_text="긴 한국어 검증 범위와 & 기호")
@@ -295,7 +319,7 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
             "&lt;script data-marker=&quot;unsafe&quot;&gt;alert(1)&lt;/script&gt;",
         )
 
-    def test_entry_stale_does_not_change_warning_empty_state(self):
+    def test_entry_stale_marks_only_unpublished_warning_unavailable(self):
         self.publish_entry()
         source = SourceConfiguration.objects.get(code="MOFA_ENTRY_CSV")
         source.state = SourceConfiguration.State.PAUSED
@@ -305,9 +329,33 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
         response = self.client.get(reverse("public_web:results"))
 
         self.assertContains(response, 'id="entry-card" data-state="stale"')
-        self.assertContains(response, 'id="warning-card" data-state="empty"')
+        self.assertContains(
+            response,
+            'id="warning-card" data-state="unavailable"',
+        )
         self.assertContains(response, "재확인 필요")
         self.assertContains(response, "90일")
+        warning_card = self.card_html(response, "warning-card")
+        self.assertIn("게시 경계를 확인할 수 없습니다", warning_card)
+
+    def test_warning_stale_marks_only_unpublished_entry_unavailable(self):
+        self.publish_warning(scope_text="독립 경보 범위")
+        source = SourceConfiguration.objects.get(
+            code="MOFA_TRAVEL_ALARM_API_JP"
+        )
+        source.state = SourceConfiguration.State.PAUSED
+        source.enabled = False
+        source.save(update_fields=("state", "enabled"))
+
+        response = self.client.get(reverse("public_web:results"))
+
+        entry_card = self.card_html(response, "entry-card")
+        warning_card = self.card_html(response, "warning-card")
+        self.assertIn('data-state="unavailable"', entry_card)
+        self.assertIn("게시 경계를 확인할 수 없습니다", entry_card)
+        self.assertIn('data-state="stale"', warning_card)
+        self.assertIn("재확인 필요", warning_card)
+        self.assertIn("독립 경보 범위", warning_card)
 
     def test_same_revision_second_rights_decision_keeps_canonical_receipt(self):
         entry = self.publish_entry()
@@ -320,7 +368,10 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
         response = self.client.get(reverse("public_web:results"))
         self.assertContains(response, 'id="entry-card" data-state="stale"')
         self.assertContains(response, self.checked_at_text(original_checked_at))
-        self.assertContains(response, 'id="warning-card" data-state="empty"')
+        self.assertContains(
+            response,
+            'id="warning-card" data-state="unavailable"',
+        )
 
     def test_new_source_revision_same_body_does_not_refresh_old_publication(self):
         entry = self.publish_entry()
@@ -382,7 +433,10 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
         self.assertContains(response, "source revision</dt><dd>rights-v1")
         self.assertContains(response, self.checked_at_text(original_checked_at))
         self.assertNotContains(response, self.checked_at_text(later_checked_at))
-        self.assertContains(response, 'id="warning-card" data-state="empty"')
+        self.assertContains(
+            response,
+            'id="warning-card" data-state="unavailable"',
+        )
 
     def test_warning_non_200_success_receipt_is_stale(self):
         warning = self.publish_warning()
@@ -404,7 +458,10 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
 
         self.assertContains(response, 'id="warning-card" data-state="stale"')
         self.assertContains(response, "재확인 필요")
-        self.assertContains(response, 'id="entry-card" data-state="empty"')
+        self.assertContains(
+            response,
+            'id="entry-card" data-state="unavailable"',
+        )
 
     def test_warning_blank_provider_success_receipt_is_stale(self):
         warning = self.publish_warning()
@@ -426,7 +483,10 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
 
         self.assertContains(response, 'id="warning-card" data-state="stale"')
         self.assertContains(response, "재확인 필요")
-        self.assertContains(response, 'id="entry-card" data-state="empty"')
+        self.assertContains(
+            response,
+            'id="entry-card" data-state="unavailable"',
+        )
 
     def test_unavailable_and_server_error_are_isolated(self):
         self.publish_warning()


