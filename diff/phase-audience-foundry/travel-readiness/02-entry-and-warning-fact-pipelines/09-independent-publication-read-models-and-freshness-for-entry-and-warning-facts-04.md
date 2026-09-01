## `fix(operations): expire stale source observations`

diff --git a/public_web/results.py b/public_web/results.py
index db37b50..1dceeac 100644
--- a/public_web/results.py
+++ b/public_web/results.py
@@ -1,13 +1,14 @@
 from __future__ import annotations
 
 from dataclasses import dataclass
-from datetime import UTC, date, datetime
+from datetime import UTC, date, datetime, timedelta
 from typing import Callable
 
 from django.db.models import Exists, OuterRef
 from django.http import HttpRequest, HttpResponse
 from django.shortcuts import render
 from django.urls import reverse
+from django.utils import timezone
 from django.views.decorators.http import require_GET
 
 from entry_requirements.ingestion import (
@@ -68,6 +69,7 @@ class _FreshnessContract:
     decided_by: str
     decision_basis: str
     provider_code: str
+    max_success_age: timedelta
 
 
 _FRESHNESS_CONTRACTS = {
@@ -84,6 +86,7 @@ _FRESHNESS_CONTRACTS = {
         decided_by=ENTRY_DECIDED_BY,
         decision_basis=ENTRY_DECISION_BASIS,
         provider_code="",
+        max_success_age=timedelta(hours=36),
     ),
     "warning": _FreshnessContract(
         source_code=WARNING_SOURCE_CODE,
@@ -98,6 +101,7 @@ _FRESHNESS_CONTRACTS = {
         decided_by=WARNING_DECIDED_BY,
         decision_basis=WARNING_DECISION_BASIS,
         provider_code=FetchAttempt.ProviderCode.MOFA_SUCCESS_0,
+        max_success_age=timedelta(hours=8),
     ),
 }
 
@@ -150,6 +154,30 @@ def _utc_minute(value: datetime | None) -> str | None:
     return value.astimezone(UTC).strftime("%Y-%m-%d %H:%M UTC")
 
 
+def _aware_utc(value: object) -> datetime | None:
+    try:
+        if not isinstance(value, datetime) or timezone.is_naive(value):
+            return None
+        return value.astimezone(UTC)
+    except (AttributeError, OverflowError, TypeError, ValueError):
+        return None
+
+
+def _matching_success_is_current(
+    completed_at: object,
+    *,
+    max_age: timedelta,
+) -> bool:
+    """Fail closed unless a successful observation has a valid current age."""
+
+    observation_time = _aware_utc(completed_at)
+    current_time = _aware_utc(timezone.now())
+    if observation_time is None or current_time is None:
+        return False
+    age = current_time - observation_time
+    return timedelta(0) <= age < max_age
+
+
 def _freshness(
     *,
     contract: _FreshnessContract,
@@ -291,8 +319,12 @@ def _freshness(
         or not latest_exact
         or last_matching_success is None
         or latest["id"] != last_matching_success["id"]
+        or not _matching_success_is_current(
+            last_matching_success["completed_at"],
+            max_age=contract.max_success_age,
+        )
     )
-    checked_at = (
+    checked_at = _aware_utc(
         None
         if last_matching_success is None
         else last_matching_success["completed_at"]
diff --git a/public_web/tests/test_results.py b/public_web/tests/test_results.py
index 0624454..7ac7f0f 100644
--- a/public_web/tests/test_results.py
+++ b/public_web/tests/test_results.py
@@ -1,5 +1,5 @@
 import uuid
-from datetime import UTC, timedelta
+from datetime import UTC, datetime, timedelta
 from unittest.mock import patch
 
 from django.contrib.auth import get_user_model
@@ -21,6 +21,7 @@ from reviews.publication import (
     rollback_publication,
 )
 from reviews.tests.test_publication import PublicationFixtureMixin
+from public_web.results import _matching_success_is_current
 from sources.models import (
     FetchAttempt,
     SourceConfiguration,
@@ -167,6 +168,120 @@ class PublicationCardTests(PublicationFixtureMixin, TransactionTestCase):
         ):
             self.assertNotIn(sensitive_name, body)
 
+    def test_entry_matching_success_age_boundaries_fail_closed(self):
+        entry = self.publish_entry(period="90일")
+        completed_at = (
+            entry.parse_run.artifact.first_successful_attempt.completed_at
+        )
+        cases = (
+            ("just-before", timedelta(hours=36) - timedelta(seconds=1), "ready"),
+            ("boundary", timedelta(hours=36), "stale"),
+            ("old", timedelta(hours=37), "stale"),
+            ("future", -timedelta(seconds=1), "stale"),
+        )
+
+        for label, age, expected_state in cases:
+            with self.subTest(label=label):
+                with patch(
+                    "public_web.results.timezone.now",
+                    return_value=completed_at + age,
+                ):
+                    response = self.client.get(reverse("public_web:results"))
+
+                card = self.card_html(response, "entry-card")
+                self.assertIn(f'data-state="{expected_state}"', card)
+                self.assertIn("90일", card)
+                if expected_state == "stale":
+                    self.assertIn("재확인 필요", card)
+
+    def test_warning_matching_success_age_boundaries_fail_closed(self):
+        warning = self.publish_warning(scope_text="합성 검증 범위")
+        completed_at = (
+            warning.parse_run.artifact.first_successful_attempt.completed_at
+        )
+        cases = (
+            ("just-before", timedelta(hours=8) - timedelta(seconds=1), "ready"),
+            ("boundary", timedelta(hours=8), "stale"),
+            ("old", timedelta(hours=9), "stale"),
+            ("future", -timedelta(seconds=1), "stale"),
+        )
+
+        for label, age, expected_state in cases:
+            with self.subTest(label=label):
+                with patch(
+                    "public_web.results.timezone.now",
+                    return_value=completed_at + age,
+                ):
+                    response = self.client.get(reverse("public_web:results"))
+
+                card = self.card_html(response, "warning-card")
+                self.assertIn(f'data-state="{expected_state}"', card)
+                self.assertIn("합성 검증 범위", card)
+                if expected_state == "stale":
+                    self.assertIn("재확인 필요", card)
+
+    def test_warning_age_boundary_drives_freshness_endpoint_alert(self):
+        self.publish_entry()
+        warning = self.publish_warning()
+        completed_at = (
+            warning.parse_run.artifact.first_successful_attempt.completed_at
+        )
+
+        with patch(
+            "public_web.results.timezone.now",
+            return_value=(
+                completed_at
+                + timedelta(hours=8)
+                - timedelta(seconds=1)
+            ),
+        ):
+            just_before = self.client.get("/freshnessz")
+
+        self.assertEqual(just_before.status_code, 200)
+        self.assertEqual(
+            just_before.json(),
+            {
+                "status": "ready",
+                "modules": {"entry": "ready", "warning": "ready"},
+            },
+        )
+
+        with patch(
+            "public_web.results.timezone.now",
+            return_value=completed_at + timedelta(hours=8),
+        ):
+            boundary = self.client.get("/freshnessz")
+
+        self.assertEqual(boundary.status_code, 200)
+        self.assertEqual(
+            boundary.json(),
+            {
+                "status": "alert",
+                "modules": {"entry": "ready", "warning": "stale"},
+            },
+        )
+
+    def test_matching_success_invalid_timestamp_shapes_fail_closed(self):
+        current_time = datetime(2026, 8, 30, 12, 0, tzinfo=UTC)
+        invalid_values = (
+            None,
+            "2026-08-30T11:00:00Z",
+            datetime(2026, 8, 30, 11, 0),
+        )
+
+        with patch(
+            "public_web.results.timezone.now",
+            return_value=current_time,
+        ):
+            for completed_at in invalid_values:
+                with self.subTest(completed_at=completed_at):
+                    self.assertFalse(
+                        _matching_success_is_current(
+                            completed_at,
+                            max_age=timedelta(hours=36),
+                        )
+                    )
+
     def test_warning_text_is_template_escaped(self):
         marker = '<script data-marker="unsafe">alert(1)</script>'
         self.publish_warning(scope_text=marker)


## `fix(browser): accept post-publication freshness checks`

diff --git a/e2e/browser_acceptance.py b/e2e/browser_acceptance.py
index 2cfc4c5..0e21672 100644
--- a/e2e/browser_acceptance.py
+++ b/e2e/browser_acceptance.py
@@ -2132,7 +2132,7 @@ def results_javascript(*, origin: str, state: str, check: str) -> str:
     const ageMinutes = (nowMillis - observedMillis) / 60000;
     if (!Number.isFinite(ageMinutes) || ageMinutes < -5 || (stateValue === 'ready' && ageMinutes > contract.freshnessMinutes + 2) || (stateValue === 'stale' && ageMinutes <= contract.freshnessMinutes)) fail(`${{module}}-freshness-age`);
     const publishedMillis = parseUtcMinute(metadata.values['게시시각']);
-    if (!/^generation [1-9]\\d*$/.test(metadata.values['publication revision']) || !Number.isFinite(publishedMillis) || publishedMillis > nowMillis + 300000 || publishedMillis + 60000 < observedMillis || metadata.values['source revision'] !== 'rights-v1') fail(`${{module}}-revision-metadata`);
+    if (!/^generation [1-9]\\d*$/.test(metadata.values['publication revision']) || !Number.isFinite(publishedMillis) || publishedMillis > nowMillis + 300000 || metadata.values['source revision'] !== 'rights-v1') fail(`${{module}}-revision-metadata`);
     if (module === 'entry') {{ const snapshotMillis = parseDateOnly(metadata.values['snapshot date']); if (!Number.isFinite(snapshotMillis) || snapshotMillis > nowMillis + 86400000 || !/^[1-9]\\d{{0,2}}일$/.test(metadata.values['일반여권 source 표기']) || unsafeSourceValue(metadata.values['source 근거 문구'], 1000)) fail('entry-facts'); }}
     if (module === 'warning') {{ const written = metadata.values['source 작성일']; const writtenMillis = written === 'source가 제공하지 않음' ? null : parseDateOnly(written); if (unsafeSourceValue(metadata.values['source 경보 단계 코드'], 32) || unsafeSourceValue(metadata.values['source 범위 유형'], 100) || unsafeSourceValue(metadata.values['source 범위'], 1000) || (writtenMillis !== null && (!Number.isFinite(writtenMillis) || writtenMillis > nowMillis + 86400000))) fail('warning-facts'); }}
     if (metadata.values['출처'].replace(/\\s+/g, ' ') !== `${{contract.owner}} 공식 source`) fail(`${{module}}-source-name`);
diff --git a/operations/tests/test_browser_acceptance_harness.py b/operations/tests/test_browser_acceptance_harness.py
index b0db2a3..9b5dfa0 100644
--- a/operations/tests/test_browser_acceptance_harness.py
+++ b/operations/tests/test_browser_acceptance_harness.py
@@ -556,6 +556,7 @@ class BrowserAcceptanceHarnessTests(unittest.TestCase):
         self.assertIn("parseDateOnly", source)
         self.assertIn("getUTCDate() === parts[2]", source)
         self.assertIn("publishedMillis > nowMillis + 300000", source)
+        self.assertNotIn("publishedMillis + 60000 < observedMillis", source)
 
     def test_pinned_static_assets_match_the_reviewed_bytes(self):
         assets = (


## `fix(web): preserve legacy entry freshness`

diff --git a/public_web/results.py b/public_web/results.py
index de4cd0d..4c25d78 100644
--- a/public_web/results.py
+++ b/public_web/results.py
@@ -110,7 +110,7 @@ _FRESHNESS_CONTRACTS = {
         ),
         attribution=ENTRY_ATTRIBUTION,
         decided_by="PROJECT_OWNER_REQUEST",
-        decision_basis="USER_FOLLOWUP_20260830",
+        decision_basis="USER_DIRECTIVE_20260830",
         provider_code="",
         max_success_age=timedelta(hours=36),
     ),
diff --git a/public_web/tests/test_travel_opportunity_web.py b/public_web/tests/test_travel_opportunity_web.py
index c4267ea..0dd6193 100644
--- a/public_web/tests/test_travel_opportunity_web.py
+++ b/public_web/tests/test_travel_opportunity_web.py
@@ -322,7 +322,9 @@ class CountryPublicationCardTests(SimpleTestCase):
         self.assertIsNotNone(entry)
         self.assertIsNotNone(warning)
         self.assertEqual(entry.source_code, LEGACY_ENTRY_SOURCE_CODE)
+        self.assertEqual(entry.decision_basis, "USER_DIRECTIVE_20260830")
         self.assertEqual(warning.source_code, LEGACY_WARNING_SOURCE_CODE)
+        self.assertEqual(warning.decision_basis, "USER_FOLLOWUP_20260830")
         self.assertIsNone(
             _freshness_contract_for_publication(
                 module="entry",


