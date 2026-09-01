## `fix(warnings): bind candidates to current rights`

diff --git a/travel_warnings/migrations/0002_revision_source_rights_guard.py b/travel_warnings/migrations/0002_revision_source_rights_guard.py
new file mode 100644
index 0000000..58ac745
--- /dev/null
+++ b/travel_warnings/migrations/0002_revision_source_rights_guard.py
@@ -0,0 +1,177 @@
+from django.db import migrations
+
+
+TRAVEL_WARNING_SOURCE_RIGHTS_GUARD_SQL = """
+CREATE FUNCTION travel_warnings_validate_revision_source_rights() RETURNS trigger
+LANGUAGE plpgsql AS $$
+DECLARE
+    artifact_source_id uuid;
+    artifact_body_sha text;
+    attempt_source_id uuid;
+    attempt_source_revision text;
+    attempt_rights_id uuid;
+    attempt_result text;
+    attempt_http_status integer;
+    attempt_provider_code text;
+    attempt_completed_at timestamptz;
+    attempt_response_sha text;
+    parse_started_at timestamptz;
+    parse_completed_at timestamptz;
+    source_code text;
+    active_source_revision text;
+    source_module text;
+    source_state text;
+    source_enabled boolean;
+    source_secret_reference text;
+    latest_rights_id uuid;
+    latest_rights_revision text;
+    latest_rights_sequence integer;
+    latest_rights_decision text;
+    latest_access_mode text;
+    latest_access_allowed boolean;
+    latest_automation_allowed boolean;
+    latest_typed_storage_allowed boolean;
+    latest_raw_storage_allowed boolean;
+    latest_typed_republication_allowed boolean;
+    latest_raw_retention_seconds integer;
+    latest_typed_retention text;
+    latest_evidence_retention text;
+    latest_field_scope text;
+    latest_attribution text;
+    latest_contract_fingerprint text;
+BEGIN
+    SELECT artifact.source_id, artifact.body_sha256,
+           attempt.source_id, attempt.source_revision,
+           attempt.rights_decision_id, attempt.result, attempt.http_status,
+           attempt.provider_code, attempt.completed_at, attempt.response_sha256,
+           parse_run.started_at, parse_run.completed_at
+    INTO artifact_source_id, artifact_body_sha, attempt_source_id,
+         attempt_source_revision, attempt_rights_id, attempt_result,
+         attempt_http_status, attempt_provider_code, attempt_completed_at,
+         attempt_response_sha, parse_started_at, parse_completed_at
+    FROM sources_parserun AS parse_run
+    JOIN sources_sourceartifact AS artifact
+      ON artifact.id = parse_run.artifact_id
+    JOIN sources_fetchattempt AS attempt
+      ON attempt.id = artifact.first_successful_attempt_id
+    WHERE parse_run.id = NEW.parse_run_id;
+    IF NOT FOUND THEN
+        RAISE EXCEPTION 'travel warning source-rights evidence is missing'
+            USING ERRCODE = 'check_violation';
+    END IF;
+
+    SELECT code, revision, module, state, enabled, secret_reference_name
+    INTO source_code, active_source_revision, source_module, source_state,
+         source_enabled, source_secret_reference
+    FROM sources_sourceconfiguration
+    WHERE id = artifact_source_id
+    FOR UPDATE;
+    IF NOT FOUND
+       OR source_code IS DISTINCT FROM 'MOFA_TRAVEL_ALARM_API_JP'
+       OR active_source_revision IS DISTINCT FROM 'rights-v1'
+       OR source_module IS DISTINCT FROM 'TRAVEL_WARNING'
+       OR source_state IS DISTINCT FROM 'ACTIVE'
+       OR source_enabled IS DISTINCT FROM true
+       OR source_secret_reference IS DISTINCT FROM
+          'MOFA_TRAVEL_ALARM_SERVICE_KEY' THEN
+        RAISE EXCEPTION 'travel warning candidate requires the exact active source'
+            USING ERRCODE = 'check_violation';
+    END IF;
+
+    SELECT id, source_revision, decision_sequence, decision, access_mode,
+           access_allowed, automated_collection_allowed,
+           typed_field_storage_allowed, raw_body_storage_allowed,
+           typed_republication_allowed, raw_retention_seconds,
+           typed_retention, evidence_retention, field_scope_code,
+           attribution_text, contract_fingerprint_sha256
+    INTO latest_rights_id, latest_rights_revision, latest_rights_sequence,
+         latest_rights_decision, latest_access_mode, latest_access_allowed,
+         latest_automation_allowed, latest_typed_storage_allowed,
+         latest_raw_storage_allowed, latest_typed_republication_allowed,
+         latest_raw_retention_seconds, latest_typed_retention,
+         latest_evidence_retention, latest_field_scope, latest_attribution,
+         latest_contract_fingerprint
+    FROM sources_sourcerightsdecision AS rights
+    WHERE rights.source_id = artifact_source_id
+      AND rights.source_revision = active_source_revision
+    ORDER BY rights.decision_sequence DESC
+    LIMIT 1;
+    IF NOT FOUND
+       OR latest_rights_id IS DISTINCT FROM attempt_rights_id
+       OR latest_rights_revision IS DISTINCT FROM 'rights-v1'
+       OR latest_rights_sequence IS DISTINCT FROM 1
+       OR latest_rights_decision IS DISTINCT FROM 'APPROVED'
+       OR latest_access_mode IS DISTINCT FROM 'CREDENTIAL_REFERENCE'
+       OR latest_access_allowed IS DISTINCT FROM true
+       OR latest_automation_allowed IS DISTINCT FROM true
+       OR latest_typed_storage_allowed IS DISTINCT FROM true
+       OR latest_raw_storage_allowed IS DISTINCT FROM false
+       OR latest_typed_republication_allowed IS DISTINCT FROM true
+       OR latest_raw_retention_seconds IS DISTINCT FROM 0
+       OR latest_typed_retention IS DISTINCT FROM 'PRODUCT_HISTORY'
+       OR latest_evidence_retention IS DISTINCT FROM 'PRODUCT_HISTORY'
+       OR latest_field_scope IS DISTINCT FROM 'JP_WARNING_V1'
+       OR latest_attribution IS DISTINCT FROM '외교부|공공데이터포털'
+       OR latest_contract_fingerprint IS DISTINCT FROM
+          'c43c5f7e7e6f37f13ba7e5d9e2448b29b24d5524012b41fc5e851abb907c55f6' THEN
+        RAISE EXCEPTION 'travel warning candidate requires the latest exact rights grant'
+            USING ERRCODE = 'check_violation';
+    END IF;
+
+    IF attempt_source_id IS DISTINCT FROM artifact_source_id
+       OR attempt_source_revision IS DISTINCT FROM active_source_revision
+       OR attempt_result IS DISTINCT FROM 'SUCCEEDED'
+       OR attempt_http_status IS DISTINCT FROM 200
+       OR attempt_provider_code IS DISTINCT FROM 'MOFA_SUCCESS_0'
+       OR attempt_completed_at IS NULL
+       OR attempt_response_sha IS DISTINCT FROM artifact_body_sha
+       OR NEW.first_observed_at IS DISTINCT FROM attempt_completed_at THEN
+        RAISE EXCEPTION 'travel warning candidate fetch evidence is inconsistent'
+            USING ERRCODE = 'check_violation';
+    END IF;
+
+    IF parse_started_at IS NULL
+       OR parse_completed_at IS NULL
+       OR NEW.created_at IS NULL
+       OR attempt_completed_at > parse_started_at
+       OR parse_started_at > parse_completed_at
+       OR parse_completed_at > NEW.created_at THEN
+        RAISE EXCEPTION 'travel warning candidate evidence times are not causal'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    RETURN NEW;
+END;
+$$;
+CREATE TRIGGER travel_warnings_revision_source_rights_guard
+BEFORE INSERT ON travel_warnings_travelwarningrevision
+FOR EACH ROW EXECUTE FUNCTION travel_warnings_validate_revision_source_rights();
+"""
+
+
+TRAVEL_WARNING_SOURCE_RIGHTS_GUARD_REVERSE_SQL = """
+LOCK TABLE travel_warnings_travelwarningrevision IN ACCESS EXCLUSIVE MODE;
+DO $$
+BEGIN
+    IF EXISTS (SELECT 1 FROM travel_warnings_travelwarningrevision) THEN
+        RAISE EXCEPTION 'travel warning source-rights guard rollback requires an empty table'
+            USING ERRCODE = 'check_violation';
+    END IF;
+END;
+$$;
+DROP TRIGGER IF EXISTS travel_warnings_revision_source_rights_guard
+    ON travel_warnings_travelwarningrevision;
+DROP FUNCTION IF EXISTS travel_warnings_validate_revision_source_rights();
+"""
+
+
+class Migration(migrations.Migration):
+    dependencies = [
+        ("travel_warnings", "0001_initial"),
+    ]
+
+    operations = [
+        migrations.RunSQL(
+            TRAVEL_WARNING_SOURCE_RIGHTS_GUARD_SQL,
+            TRAVEL_WARNING_SOURCE_RIGHTS_GUARD_REVERSE_SQL,
+        ),
+    ]
diff --git a/travel_warnings/models.py b/travel_warnings/models.py
index 9bfba12..28b63d2 100644
--- a/travel_warnings/models.py
+++ b/travel_warnings/models.py
@@ -4,6 +4,11 @@ from django.db import models
 from django.db.models import F, Q
 
 
+SOURCE_ALARM_LEVEL_CODE_MAX_LENGTH = 32
+SOURCE_SCOPE_TYPE_MAX_LENGTH = 100
+SOURCE_SCOPE_TEXT_MAX_LENGTH = 1000
+
+
 class TravelWarningRevision(models.Model):
     id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
     country = models.ForeignKey(
@@ -16,9 +21,11 @@ class TravelWarningRevision(models.Model):
         on_delete=models.PROTECT,
         related_name="travel_warning_revision",
     )
-    source_alarm_level_code = models.CharField(max_length=32)
-    source_scope_type = models.CharField(max_length=100)
-    source_scope_text = models.CharField(max_length=1000)
+    source_alarm_level_code = models.CharField(
+        max_length=SOURCE_ALARM_LEVEL_CODE_MAX_LENGTH
+    )
+    source_scope_type = models.CharField(max_length=SOURCE_SCOPE_TYPE_MAX_LENGTH)
+    source_scope_text = models.CharField(max_length=SOURCE_SCOPE_TEXT_MAX_LENGTH)
     source_written_date = models.DateField(null=True, blank=True)
     first_observed_at = models.DateTimeField()
     typed_fingerprint_sha256 = models.CharField(max_length=64)
diff --git a/travel_warnings/parser.py b/travel_warnings/parser.py
index 7566d40..e3c013e 100644
--- a/travel_warnings/parser.py
+++ b/travel_warnings/parser.py
@@ -8,6 +8,11 @@ from datetime import date
 from typing import Any
 
 from sources.models import ParseRun
+from travel_warnings.models import (
+    SOURCE_ALARM_LEVEL_CODE_MAX_LENGTH,
+    SOURCE_SCOPE_TEXT_MAX_LENGTH,
+    SOURCE_SCOPE_TYPE_MAX_LENGTH,
+)
 
 
 MAX_RESPONSE_BYTES = 4096
@@ -216,6 +221,12 @@ def parse_travel_alarm_jp(payload: bytes) -> TravelWarningParseResult:
         for field in ("alarm_lvl", "region_ty", "remark")
     ):
         return _failure(ParseRun.FailureCode.REQUIRED_VALUE_MISSING, observed_schema)
+    if (
+        len(item["alarm_lvl"]) > SOURCE_ALARM_LEVEL_CODE_MAX_LENGTH
+        or len(item["region_ty"]) > SOURCE_SCOPE_TYPE_MAX_LENGTH
+        or len(item["remark"]) > SOURCE_SCOPE_TEXT_MAX_LENGTH
+    ):
+        return _failure(ParseRun.FailureCode.INVALID_VALUE, observed_schema)
     try:
         written_date = _parse_written_date(item["written_dt"])
     except ValueError:
diff --git a/travel_warnings/tests/test_models.py b/travel_warnings/tests/test_models.py
index d3e27fc..bfb9fa6 100644
--- a/travel_warnings/tests/test_models.py
+++ b/travel_warnings/tests/test_models.py
@@ -13,7 +13,13 @@ from django.test import TestCase, TransactionTestCase
 from django.utils import timezone
 
 from countries.models import Country, JP_COUNTRY_ID
-from sources.models import FetchAttempt, ParseRun, SourceArtifact, SourceConfiguration
+from sources.models import (
+    FetchAttempt,
+    ParseRun,
+    SourceArtifact,
+    SourceConfiguration,
+    SourceRightsDecision,
+)
 from travel_warnings.models import TravelWarningRevision
 
 
@@ -34,6 +40,50 @@ class TravelWarningFixtureMixin:
     def register_sources(self):
         call_command("register_approved_sources", "--apply", stdout=StringIO())
 
+    def create_warning_source(self, **rights_overrides):
+        source = SourceConfiguration.objects.create(
+            code="MOFA_TRAVEL_ALARM_API_JP",
+            revision="rights-v1",
+            module=SourceConfiguration.Module.TRAVEL_WARNING,
+            owner="대한민국 외교부",
+            official_locator=(
+                "https://apis.data.go.kr/1262000/TravelAlarmService2/"
+                "getTravelAlarmList2"
+            ),
+            secret_reference_name="MOFA_TRAVEL_ALARM_SERVICE_KEY",
+        )
+        rights_values = {
+            "source": source,
+            "source_revision": "rights-v1",
+            "decision_sequence": 1,
+            "decision": SourceRightsDecision.Decision.APPROVED,
+            "access_mode": SourceRightsDecision.AccessMode.CREDENTIAL_REFERENCE,
+            "access_allowed": True,
+            "automated_collection_allowed": True,
+            "typed_field_storage_allowed": True,
+            "raw_body_storage_allowed": False,
+            "typed_republication_allowed": True,
+            "raw_retention_seconds": 0,
+            "typed_retention": SourceRightsDecision.Retention.PRODUCT_HISTORY,
+            "evidence_retention": SourceRightsDecision.Retention.PRODUCT_HISTORY,
+            "field_scope_code": "JP_WARNING_V1",
+            "attribution_text": "외교부|공공데이터포털",
+            "contract_fingerprint_sha256": self.WARNING_CONTRACT,
+            "decided_by": "PROJECT_OWNER_REQUEST",
+            "decision_basis_code": "USER_FOLLOWUP_20260830",
+        }
+        rights_values.update(rights_overrides)
+        rights = SourceRightsDecision.objects.create(**rights_values)
+        SourceConfiguration.objects.filter(pk=source.pk).update(
+            state=SourceConfiguration.State.RIGHTS_APPROVED
+        )
+        SourceConfiguration.objects.filter(pk=source.pk).update(
+            state=SourceConfiguration.State.ACTIVE,
+            enabled=True,
+        )
+        source.refresh_from_db()
+        return source, rights
+
     def make_parse(
         self,
         suffix,
@@ -42,6 +92,10 @@ class TravelWarningFixtureMixin:
         close=True,
         review_artifact=True,
         byte_count=512,
+        parse_start_delta=timedelta(seconds=1),
+        parse_duration=timedelta(seconds=1),
+        provider_code=FetchAttempt.ProviderCode.MOFA_SUCCESS_0,
+        http_status=200,
     ):
         source = SourceConfiguration.objects.get(code=source_code)
         rights = source.rights_decisions.get()
@@ -63,8 +117,8 @@ class TravelWarningFixtureMixin:
         FetchAttempt.objects.filter(pk=attempt.pk).update(
             completed_at=completed_at,
             result=FetchAttempt.Result.SUCCEEDED,
-            http_status=200,
-            provider_code=FetchAttempt.ProviderCode.MOFA_SUCCESS_0,
+            http_status=http_status,
+            provider_code=provider_code,
             response_sha256=body_sha,
         )
         attempt.refresh_from_db()
@@ -89,12 +143,12 @@ class TravelWarningFixtureMixin:
             expected_schema_fingerprint_sha256=(
                 self.ENTRY_SCHEMA if is_entry else self.WARNING_SCHEMA
             ),
-            started_at=completed_at + timedelta(seconds=1),
+            started_at=completed_at + parse_start_delta,
         )
         if close:
             expected = run.expected_schema_fingerprint_sha256
             ParseRun.objects.filter(pk=run.pk).update(
-                completed_at=run.started_at + timedelta(seconds=1),
+                completed_at=run.started_at + parse_duration,
                 outcome=ParseRun.Outcome.VALIDATED,
                 observed_schema_fingerprint_sha256=expected,
             )
@@ -357,10 +411,134 @@ class TravelWarningRevisionTests(TravelWarningFixtureMixin, TestCase):
                 ],
             )
 
+    def test_candidate_requires_an_active_source_at_insert_time(self):
+        SourceConfiguration.objects.filter(
+            code="MOFA_TRAVEL_ALARM_API_JP"
+        ).update(state=SourceConfiguration.State.PAUSED, enabled=False)
+
+        self.assert_integrity_error(
+            lambda: TravelWarningRevision.objects.create(
+                **self.revision_values(self.run, self.attempt)
+            )
+        )
+
+    def test_later_rights_rejection_closes_candidate_creation(self):
+        source = SourceConfiguration.objects.get(code="MOFA_TRAVEL_ALARM_API_JP")
+        SourceRightsDecision.objects.create(
+            source=source,
+            source_revision=source.revision,
+            decision_sequence=2,
+            decision=SourceRightsDecision.Decision.REJECTED,
+            access_mode=SourceRightsDecision.AccessMode.NO_ACCESS,
+            access_allowed=False,
+            automated_collection_allowed=False,
+            typed_field_storage_allowed=False,
+            raw_body_storage_allowed=False,
+            typed_republication_allowed=False,
+            raw_retention_seconds=0,
+            typed_retention=SourceRightsDecision.Retention.NONE,
+            evidence_retention=SourceRightsDecision.Retention.PRODUCT_HISTORY,
+            field_scope_code="",
+            attribution_text="",
+            contract_fingerprint_sha256=self.WARNING_CONTRACT,
+            decided_by="PROJECT_OWNER_REQUEST",
+            decision_basis_code="RIGHTS_REVOKED_20260830",
+        )
+        source.refresh_from_db()
+        self.assertEqual(source.state, SourceConfiguration.State.REJECTED)
+        self.assertFalse(source.enabled)
+
+        self.assert_integrity_error(
+            lambda: TravelWarningRevision.objects.create(
+                **self.revision_values(self.run, self.attempt)
+            )
+        )
+
+    def test_candidate_times_must_follow_the_fetch_parse_create_sequence(self):
+        early_run, _, early_attempt = self.make_parse(
+            "PARSE_BEFORE_FETCH",
+            parse_start_delta=timedelta(seconds=-2),
+        )
+        self.assert_integrity_error(
+            lambda: TravelWarningRevision.objects.create(
+                **self.revision_values(early_run, early_attempt)
+            )
+        )
+
+        future_run, _, future_attempt = self.make_parse(
+            "PARSE_AFTER_CREATE",
+            parse_start_delta=timedelta(minutes=5),
+        )
+        self.assert_integrity_error(
+            lambda: TravelWarningRevision.objects.create(
+                **self.revision_values(future_run, future_attempt)
+            )
+        )
+
+    def test_candidate_requires_the_success_provider_envelope(self):
+        run, _, attempt = self.make_parse("NO_PROVIDER_CODE", provider_code="")
+        self.assert_integrity_error(
+            lambda: TravelWarningRevision.objects.create(
+                **self.revision_values(run, attempt)
+            )
+        )
+
+        nonexact_run, _, nonexact_attempt = self.make_parse(
+            "NONEXACT_HTTP_SUCCESS",
+            http_status=201,
+        )
+        self.assert_integrity_error(
+            lambda: TravelWarningRevision.objects.create(
+                **self.revision_values(nonexact_run, nonexact_attempt)
+            )
+        )
+
+
+class TravelWarningRightsGuardTests(TravelWarningFixtureMixin, TestCase):
+    def assert_custom_rights_rejected(self, suffix, **rights_overrides):
+        self.create_warning_source(**rights_overrides)
+        run, _, attempt = self.make_parse(suffix)
+        self.assert_integrity_error(
+            lambda: TravelWarningRevision.objects.create(
+                **self.revision_values(run, attempt)
+            )
+        )
+
+    def test_candidate_requires_typed_republication_capability(self):
+        self.assert_custom_rights_rejected(
+            "NO_REPUBLICATION",
+            typed_republication_allowed=False,
+        )
+
+    def test_candidate_requires_credential_reference_access(self):
+        self.assert_custom_rights_rejected(
+            "ANONYMOUS_ACCESS",
+            access_mode=SourceRightsDecision.AccessMode.ANONYMOUS_PUBLIC,
+        )
+
+    def test_candidate_rejects_raw_storage_and_retention(self):
+        self.assert_custom_rights_rejected(
+            "RAW_RETENTION",
+            raw_body_storage_allowed=True,
+            raw_retention_seconds=60,
+        )
+
+    def test_candidate_requires_the_exact_typed_scope(self):
+        self.assert_custom_rights_rejected(
+            "SCOPE_DRIFT",
+            field_scope_code="JP_WARNING_V2",
+        )
+
+    def test_candidate_requires_the_exact_rights_contract(self):
+        self.assert_custom_rights_rejected(
+            "CONTRACT_DRIFT",
+            contract_fingerprint_sha256="b" * 64,
+        )
+
 
 class TravelWarningMigrationTests(TravelWarningFixtureMixin, TransactionTestCase):
-    latest = [("travel_warnings", "0001_initial")]
-    previous = [("travel_warnings", None)]
+    latest = [("travel_warnings", "0002_revision_source_rights_guard")]
+    previous = [("travel_warnings", "0001_initial")]
 
     def restore_latest(self):
         MigrationExecutor(connection).migrate(self.latest)
@@ -397,30 +575,46 @@ class TravelWarningMigrationTests(TravelWarningFixtureMixin, TransactionTestCase
 
         self.assertTrue(TravelWarningRevision.objects.filter(pk=revision.pk).exists())
         self.assertIn(
-            ("travel_warnings", "0001_initial"),
+            ("travel_warnings", "0002_revision_source_rights_guard"),
             MigrationRecorder(connection).applied_migrations(),
         )
         with connection.cursor() as cursor:
             cursor.execute(
-                "SELECT to_regprocedure('travel_warnings_guard_revision_change()')"
+                "SELECT to_regprocedure("
+                "'travel_warnings_validate_revision_source_rights()')"
             )
             self.assertIsNotNone(cursor.fetchone()[0])
 
     def test_empty_reverse_and_reapply_restore_the_guard(self):
         self.assertFalse(TravelWarningRevision.objects.exists())
         MigrationExecutor(connection).migrate(self.previous)
-        self.assertNotIn(
+        self.assertIn(
             "travel_warnings_travelwarningrevision",
             connection.introspection.table_names(),
         )
         self.assertNotIn(
+            ("travel_warnings", "0002_revision_source_rights_guard"),
+            MigrationRecorder(connection).applied_migrations(),
+        )
+        self.assertIn(
             ("travel_warnings", "0001_initial"),
             MigrationRecorder(connection).applied_migrations(),
         )
+        with connection.cursor() as cursor:
+            cursor.execute(
+                "SELECT to_regprocedure("
+                "'travel_warnings_validate_revision_source_rights()')"
+            )
+            self.assertIsNone(cursor.fetchone()[0])
+            cursor.execute(
+                "SELECT to_regprocedure('travel_warnings_guard_revision_change()')"
+            )
+            self.assertIsNotNone(cursor.fetchone()[0])
 
         self.restore_latest()
         with connection.cursor() as cursor:
             cursor.execute(
-                "SELECT to_regprocedure('travel_warnings_guard_revision_change()')"
+                "SELECT to_regprocedure("
+                "'travel_warnings_validate_revision_source_rights()')"
             )
             self.assertIsNotNone(cursor.fetchone()[0])
diff --git a/travel_warnings/tests/test_parser.py b/travel_warnings/tests/test_parser.py
index 3c406be..b76f66d 100644
--- a/travel_warnings/tests/test_parser.py
+++ b/travel_warnings/tests/test_parser.py
@@ -6,6 +6,11 @@ from datetime import date
 from django.test import SimpleTestCase
 
 from sources.models import ParseRun
+from travel_warnings.models import (
+    SOURCE_ALARM_LEVEL_CODE_MAX_LENGTH,
+    SOURCE_SCOPE_TEXT_MAX_LENGTH,
+    SOURCE_SCOPE_TYPE_MAX_LENGTH,
+)
 from travel_warnings.parser import (
     EXPECTED_SCHEMA_FINGERPRINT_SHA256,
     ITEM_KEYS,
@@ -164,6 +169,43 @@ class TravelAlarmParserTests(SimpleTestCase):
                     observed=expected,
                 )
 
+    def test_model_character_limits_accept_boundaries_and_reject_overflow(self):
+        cases = (
+            (
+                "alarm_lvl",
+                "source_alarm_level_code",
+                SOURCE_ALARM_LEVEL_CODE_MAX_LENGTH,
+            ),
+            ("region_ty", "source_scope_type", SOURCE_SCOPE_TYPE_MAX_LENGTH),
+            ("remark", "source_scope_text", SOURCE_SCOPE_TEXT_MAX_LENGTH),
+        )
+        for provider_field, projection_field, limit in cases:
+            with self.subTest(field=provider_field, boundary=limit):
+                boundary_value = "가" * limit
+                result = parse_travel_alarm_jp(
+                    self.encode(
+                        self.document(
+                            item=self.item(**{provider_field: boundary_value})
+                        )
+                    )
+                )
+                self.assertIsInstance(result, TravelWarningParseSuccess)
+                self.assertEqual(
+                    getattr(result.warning, projection_field),
+                    boundary_value,
+                )
+
+            with self.subTest(field=provider_field, overflow=limit + 1):
+                self.assert_failure(
+                    self.encode(
+                        self.document(
+                            item=self.item(**{provider_field: "가" * (limit + 1)})
+                        )
+                    ),
+                    ParseRun.FailureCode.INVALID_VALUE,
+                    observed=EXPECTED_SCHEMA_FINGERPRINT_SHA256,
+                )
+
     def test_duplicate_keys_and_items_are_rejected(self):
         payload = self.encode(self.document())
         duplicate_key = payload.replace(


