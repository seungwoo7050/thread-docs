## `feat(reviews): publish warning snapshots`

diff --git a/reviews/admin.py b/reviews/admin.py
index 628d069..638eb00 100644
--- a/reviews/admin.py
+++ b/reviews/admin.py
@@ -9,9 +9,10 @@ from __future__ import annotations
 
 from django.contrib import admin, messages
 from django.core.exceptions import ImproperlyConfigured
+from django.utils.html import format_html, format_html_join
 
 from entry_requirements.models import EntryFactRevision
-from travel_warnings.models import TravelWarningRevision
+from travel_warnings.models import TravelWarningFact, TravelWarningRevision
 
 from .models import (
     AuditEvent,
@@ -298,6 +299,76 @@ class _ReadOnlyAdmin(admin.ModelAdmin):
         return True
 
 
+class TravelWarningFactInline(admin.TabularInline):
+    """Show every source fact in provider order without permitting edits."""
+
+    model = TravelWarningFact
+    fields = (
+        "source_position",
+        "source_alarm_level_code",
+        "source_scope_type",
+        "source_scope_text",
+        "source_written_date",
+    )
+    readonly_fields = fields
+    ordering = ("source_position",)
+    extra = 0
+    can_delete = False
+    show_change_link = False
+
+    def has_add_permission(self, request, obj=None):
+        return False
+
+    def has_change_permission(self, request, obj=None):
+        return False
+
+    def has_delete_permission(self, request, obj=None):
+        return False
+
+
+def _warning_contract_label(revision) -> str:
+    if revision is None:
+        return "—"
+    if revision.observation_attempt_id is None:
+        return "기존 단일 출처 항목 (v1 이력)"
+    return "국가 전체 출처 조회 (v2)"
+
+
+def _warning_source_facts(revision):
+    if revision is None:
+        return "—"
+    if revision.observation_attempt_id is None:
+        written_date = revision.source_written_date or "출처가 제공하지 않음"
+        return format_html(
+            "<div><strong>기존 출처 항목 1</strong> — 단계 코드 {} · "
+            "범위 유형 {} · 범위 {} · 작성일 {}</div>",
+            revision.source_alarm_level_code,
+            revision.source_scope_type,
+            revision.source_scope_text,
+            written_date,
+        )
+    facts = list(revision.facts.all().order_by("source_position"))
+    if revision.source_item_count == 0 and not facts:
+        return "공식 출처의 국가 전체 조회 결과 항목 0건"
+    return format_html_join(
+        "",
+        (
+            "<div><strong>출처 항목 {}</strong> — 단계 코드 {} · "
+            "범위 유형 {} · 범위 {} · 작성일 {}</div>"
+        ),
+        (
+            (
+                fact.source_position,
+                fact.source_alarm_level_code,
+                fact.source_scope_type,
+                fact.source_scope_text,
+                fact.source_written_date or "출처가 제공하지 않음",
+            )
+            for fact in facts
+        ),
+    )
+
+
 @admin.register(EntryFactRevision, site=operator_admin_site)
 class EntryFactRevisionAdmin(_ReadOnlyAdmin):
     fields = (
@@ -371,6 +442,8 @@ class TravelWarningRevisionAdmin(_ReadOnlyAdmin):
     fields = (
         "id",
         "country",
+        "warning_contract",
+        "source_item_count",
         "source_alarm_level_code",
         "source_scope_type",
         "source_scope_text",
@@ -379,10 +452,25 @@ class TravelWarningRevisionAdmin(_ReadOnlyAdmin):
         "created_at",
     )
     readonly_fields = fields
-    list_display = fields
-    list_select_related = ("country",)
+    list_display = (
+        "id",
+        "country",
+        "warning_contract",
+        "source_item_count",
+        "first_observed_at",
+        "created_at",
+    )
+    list_select_related = ("country", "observation_attempt")
     ordering = ("-created_at", "-id")
     actions = ("publish_warning_candidate", "reject_warning_candidate")
+    inlines = (TravelWarningFactInline,)
+
+    def get_queryset(self, request):
+        return super().get_queryset(request).prefetch_related("facts")
+
+    @admin.display(description="출처 자료 구조")
+    def warning_contract(self, obj):
+        return _warning_contract_label(obj)
 
     def has_publish_warning_permission(self, request):
         return _operator_has_permission(request, "reviews.publish_warning")
@@ -460,6 +548,9 @@ class PublicationRevisionAdmin(_ReadOnlyAdmin):
         "entry_period_text",
         "entry_basis_text",
         "entry_snapshot_date",
+        "warning_contract",
+        "warning_source_item_count",
+        "warning_source_facts",
         "warning_alarm_level_code",
         "warning_scope_type",
         "warning_scope_text",
@@ -529,6 +620,19 @@ class PublicationRevisionAdmin(_ReadOnlyAdmin):
         revision = obj.entry_fact_revision
         return revision.snapshot_date if revision else None
 
+    @admin.display(description="여행경보 출처 자료 구조")
+    def warning_contract(self, obj):
+        return _warning_contract_label(obj.travel_warning_revision)
+
+    @admin.display(description="여행경보 출처 항목 수")
+    def warning_source_item_count(self, obj):
+        revision = obj.travel_warning_revision
+        return revision.source_item_count if revision else None
+
+    @admin.display(description="여행경보 출처 사실")
+    def warning_source_facts(self, obj):
+        return _warning_source_facts(obj.travel_warning_revision)
+
     @admin.display(description="여행경보 단계")
     def warning_alarm_level_code(self, obj):
         revision = obj.travel_warning_revision
diff --git a/reviews/migrations/0004_warning_snapshot_publications.py b/reviews/migrations/0004_warning_snapshot_publications.py
new file mode 100644
index 0000000..c16b9df
--- /dev/null
+++ b/reviews/migrations/0004_warning_snapshot_publications.py
@@ -0,0 +1,427 @@
+import uuid
+
+from django.db import migrations, models
+
+
+FUNCTION_NAME = "reviews_guard_publication_revision"
+V1_WARNING_TAIL = """        AND rights_contract IS NOT DISTINCT FROM
+          '5a6c879394b8b25c56ad53eaaf3f294f87ae71956c5a9397d830972b426f85a7'
+    ) THEN"""
+V2_WARNING_TAIL = """        AND rights_contract IS NOT DISTINCT FROM
+          '5a6c879394b8b25c56ad53eaaf3f294f87ae71956c5a9397d830972b426f85a7'
+    ) AND NOT (
+        source_code IS NOT DISTINCT FROM
+          'MOFA_TRAVEL_ALARM_API_TRAVEL6_SET'
+        AND chain_source_revision IS NOT DISTINCT FROM 'travel6-set-v2'
+        AND source_module IS NOT DISTINCT FROM 'TRAVEL_WARNING'
+        AND source_owner IS NOT DISTINCT FROM '대한민국 외교부'
+        AND source_locator IS NOT DISTINCT FROM
+          'https://apis.data.go.kr/1262000/TravelAlarmService2/getTravelAlarmList2'
+        AND source_secret_reference IS NOT DISTINCT FROM
+          'DATA_GO_KR_SERVICE_KEY'
+        AND artifact_byte_count <= 262144
+        AND rights_decided_by IS NOT DISTINCT FROM 'PROJECT_OWNER_REQUEST'
+        AND rights_decision_basis IS NOT DISTINCT FROM
+          'LIVE_WARNING_SET_SHAPES_20260831'
+        AND parse_name IS NOT DISTINCT FROM 'MOFA_TRAVEL_ALARM_JSON'
+        AND parse_version IS NOT DISTINCT FROM 'V2'
+        AND parse_contract IS NOT DISTINCT FROM
+          '23d8a89ec668b856c05f8ebff448322f1fda457407c3b803b96225b990567dd4'
+        AND parse_schema IS NOT DISTINCT FROM
+          '2bb572cd61be540f0e8189ccb29dbd8278671c88af0106a8f16f551bf5118c2d'
+        AND attempt_provider_code IS NOT DISTINCT FROM 'MOFA_SUCCESS_0'
+        AND rights_access_mode IS NOT DISTINCT FROM 'CREDENTIAL_REFERENCE'
+        AND rights_scope IS NOT DISTINCT FROM 'SUPPORTED_6_WARNING_SET_V2'
+        AND rights_attribution IS NOT DISTINCT FROM '외교부|공공데이터포털'
+        AND rights_contract IS NOT DISTINCT FROM
+          '23d8a89ec668b856c05f8ebff448322f1fda457407c3b803b96225b990567dd4'
+    ) THEN"""
+
+
+WARNING_SNAPSHOT_PUBLICATION_SQL = r"""
+CREATE FUNCTION reviews_guard_warning_snapshot_publication() RETURNS trigger
+LANGUAGE plpgsql AS $$
+DECLARE
+    root_country_id uuid;
+    root_observation_id uuid;
+    root_item_count integer;
+    root_level text;
+    root_scope_type text;
+    root_scope_text text;
+    root_written_date date;
+    root_observed_at timestamptz;
+    root_typed_hash text;
+    parse_outcome text;
+    parse_name text;
+    parse_version text;
+    parse_contract text;
+    parse_expected_schema text;
+    parse_observed_schema text;
+    artifact_source_id uuid;
+    artifact_state text;
+    artifact_byte_count bigint;
+    artifact_body_hash text;
+    source_code text;
+    active_source_revision text;
+    source_module text;
+    source_owner text;
+    source_locator text;
+    source_state text;
+    source_enabled boolean;
+    source_secret_reference text;
+    observation_source_id uuid;
+    observation_source_revision text;
+    observation_rights_id uuid;
+    observation_result text;
+    observation_http_status integer;
+    observation_provider_code text;
+    observation_completed_at timestamptz;
+    observation_response_hash text;
+    observation_request_revision text;
+    observation_request_hash text;
+    country_iso2 text;
+    expected_request_hash text;
+    latest_rights_id uuid;
+    rights_revision text;
+    rights_sequence integer;
+    rights_decision text;
+    rights_access_mode text;
+    rights_access boolean;
+    rights_automation boolean;
+    rights_typed_storage boolean;
+    rights_raw_storage boolean;
+    rights_republication boolean;
+    rights_raw_seconds integer;
+    rights_typed_retention text;
+    rights_evidence_retention text;
+    rights_scope text;
+    rights_attribution text;
+    rights_contract text;
+    rights_decided_by text;
+    rights_decision_basis text;
+    fact_count integer;
+    minimum_position integer;
+    maximum_position integer;
+BEGIN
+    SELECT root.country_id, root.observation_attempt_id,
+           root.source_item_count, root.source_alarm_level_code,
+           root.source_scope_type, root.source_scope_text,
+           root.source_written_date, root.first_observed_at,
+           root.typed_fingerprint_sha256,
+           parse_run.outcome, parse_run.parser_name,
+           parse_run.parser_version,
+           parse_run.parser_contract_fingerprint_sha256,
+           parse_run.expected_schema_fingerprint_sha256,
+           parse_run.observed_schema_fingerprint_sha256,
+           artifact.source_id, artifact.state, artifact.byte_count,
+           artifact.body_sha256,
+           source.code, source.revision, source.module, source.owner,
+           source.official_locator, source.state, source.enabled,
+           source.secret_reference_name,
+           observation.source_id, observation.source_revision,
+           observation.rights_decision_id, observation.result,
+           observation.http_status, observation.provider_code,
+           observation.completed_at, observation.response_sha256,
+           observation.request_fingerprint_revision,
+           observation.normalized_request_sha256, country.iso_alpha2
+      INTO root_country_id, root_observation_id, root_item_count,
+           root_level, root_scope_type, root_scope_text, root_written_date,
+           root_observed_at, root_typed_hash,
+           parse_outcome, parse_name, parse_version, parse_contract,
+           parse_expected_schema, parse_observed_schema,
+           artifact_source_id, artifact_state, artifact_byte_count,
+           artifact_body_hash,
+           source_code, active_source_revision, source_module, source_owner,
+           source_locator, source_state, source_enabled,
+           source_secret_reference,
+           observation_source_id, observation_source_revision,
+           observation_rights_id, observation_result,
+           observation_http_status, observation_provider_code,
+           observation_completed_at, observation_response_hash,
+           observation_request_revision, observation_request_hash,
+           country_iso2
+      FROM travel_warnings_travelwarningrevision AS root
+      JOIN sources_parserun AS parse_run ON parse_run.id = root.parse_run_id
+      JOIN sources_sourceartifact AS artifact
+        ON artifact.id = parse_run.artifact_id
+      JOIN sources_sourceconfiguration AS source
+        ON source.id = artifact.source_id
+      JOIN sources_fetchattempt AS observation
+        ON observation.id = root.observation_attempt_id
+      JOIN countries_country AS country ON country.id = root.country_id
+     WHERE root.id = NEW.travel_warning_revision_id
+     FOR UPDATE OF source, observation;
+    IF NOT FOUND THEN
+        RAISE EXCEPTION 'warning snapshot publication requires exact evidence'
+            USING ERRCODE = 'check_violation';
+    END IF;
+
+    SELECT id, source_revision, decision_sequence, decision, access_mode,
+           access_allowed, automated_collection_allowed,
+           typed_field_storage_allowed, raw_body_storage_allowed,
+           typed_republication_allowed, raw_retention_seconds,
+           typed_retention, evidence_retention, field_scope_code,
+           attribution_text, contract_fingerprint_sha256, decided_by,
+           decision_basis_code
+      INTO latest_rights_id, rights_revision, rights_sequence,
+           rights_decision, rights_access_mode, rights_access,
+           rights_automation, rights_typed_storage, rights_raw_storage,
+           rights_republication, rights_raw_seconds,
+           rights_typed_retention, rights_evidence_retention,
+           rights_scope, rights_attribution, rights_contract,
+           rights_decided_by, rights_decision_basis
+      FROM sources_sourcerightsdecision AS rights
+     WHERE rights.source_id = artifact_source_id
+       AND rights.source_revision = active_source_revision
+     ORDER BY decision_sequence DESC, id DESC
+     LIMIT 1 FOR UPDATE;
+
+    SELECT COUNT(*), MIN(source_position), MAX(source_position)
+      INTO fact_count, minimum_position, maximum_position
+      FROM travel_warnings_travelwarningfact
+     WHERE revision_id = NEW.travel_warning_revision_id;
+
+    expected_request_hash := encode(sha256(convert_to(
+        'GET' || E'\n' || 'https' || E'\n' || 'apis.data.go.kr' || E'\n' ||
+        '/1262000/TravelAlarmService2/getTravelAlarmList2' || E'\n' ||
+        'ServiceKey=%3Credacted%3E&returnType=JSON&numOfRows=100&' ||
+        'pageNo=1&cond%5Bcountry_iso_alp2%3A%3AEQ%5D=' || country_iso2,
+        'UTF8'
+    )), 'hex');
+
+    IF root_country_id IS DISTINCT FROM NEW.scope_country_id
+       OR root_observation_id IS NULL
+       OR root_item_count < 0 OR root_item_count > 100
+       OR root_level IS NOT NULL OR root_scope_type IS NOT NULL
+       OR root_scope_text IS NOT NULL OR root_written_date IS NOT NULL
+       OR fact_count <> root_item_count
+       OR (root_item_count = 0 AND (
+            minimum_position IS NOT NULL OR maximum_position IS NOT NULL
+       ))
+       OR (root_item_count > 0 AND (
+            minimum_position <> 1 OR maximum_position <> root_item_count
+       ))
+       OR parse_outcome IS DISTINCT FROM 'VALIDATED'
+       OR parse_name IS DISTINCT FROM 'MOFA_TRAVEL_ALARM_JSON'
+       OR parse_version IS DISTINCT FROM 'V2'
+       OR parse_contract IS DISTINCT FROM
+          '23d8a89ec668b856c05f8ebff448322f1fda457407c3b803b96225b990567dd4'
+       OR parse_expected_schema IS DISTINCT FROM
+          '2bb572cd61be540f0e8189ccb29dbd8278671c88af0106a8f16f551bf5118c2d'
+       OR parse_observed_schema IS DISTINCT FROM parse_expected_schema
+       OR artifact_state IS DISTINCT FROM 'REVIEW_REQUIRED'
+       OR artifact_byte_count < 1 OR artifact_byte_count > 262144
+       OR source_code IS DISTINCT FROM 'MOFA_TRAVEL_ALARM_API_TRAVEL6_SET'
+       OR active_source_revision IS DISTINCT FROM 'travel6-set-v2'
+       OR source_module IS DISTINCT FROM 'TRAVEL_WARNING'
+       OR source_owner IS DISTINCT FROM '대한민국 외교부'
+       OR source_locator IS DISTINCT FROM
+          'https://apis.data.go.kr/1262000/TravelAlarmService2/getTravelAlarmList2'
+       OR source_state IS DISTINCT FROM 'ACTIVE'
+       OR source_enabled IS DISTINCT FROM true
+       OR source_secret_reference IS DISTINCT FROM 'DATA_GO_KR_SERVICE_KEY'
+       OR observation_source_id IS DISTINCT FROM artifact_source_id
+       OR observation_source_revision IS DISTINCT FROM active_source_revision
+       OR observation_rights_id IS DISTINCT FROM latest_rights_id
+       OR observation_result IS DISTINCT FROM 'SUCCEEDED'
+       OR observation_http_status IS DISTINCT FROM 200
+       OR observation_provider_code IS DISTINCT FROM 'MOFA_SUCCESS_0'
+       OR observation_completed_at IS NULL
+       OR observation_response_hash IS DISTINCT FROM artifact_body_hash
+       OR observation_request_revision IS DISTINCT FROM 'MOFA_WARNING_V2'
+       OR observation_request_hash IS DISTINCT FROM expected_request_hash
+       OR root_observed_at IS DISTINCT FROM observation_completed_at
+       OR rights_revision IS DISTINCT FROM 'travel6-set-v2'
+       OR rights_sequence IS DISTINCT FROM 1
+       OR rights_decision IS DISTINCT FROM 'APPROVED'
+       OR rights_access_mode IS DISTINCT FROM 'CREDENTIAL_REFERENCE'
+       OR rights_access IS DISTINCT FROM true
+       OR rights_automation IS DISTINCT FROM true
+       OR rights_typed_storage IS DISTINCT FROM true
+       OR rights_raw_storage IS DISTINCT FROM false
+       OR rights_republication IS DISTINCT FROM true
+       OR rights_raw_seconds IS DISTINCT FROM 0
+       OR rights_typed_retention IS DISTINCT FROM 'PRODUCT_HISTORY'
+       OR rights_evidence_retention IS DISTINCT FROM 'PRODUCT_HISTORY'
+       OR rights_scope IS DISTINCT FROM 'SUPPORTED_6_WARNING_SET_V2'
+       OR rights_attribution IS DISTINCT FROM '외교부|공공데이터포털'
+       OR rights_contract IS DISTINCT FROM
+          '23d8a89ec668b856c05f8ebff448322f1fda457407c3b803b96225b990567dd4'
+       OR rights_decided_by IS DISTINCT FROM 'PROJECT_OWNER_REQUEST'
+       OR rights_decision_basis IS DISTINCT FROM
+          'LIVE_WARNING_SET_SHAPES_20260831'
+       OR NEW.source_code_snapshot IS DISTINCT FROM source_code
+       OR NEW.source_revision IS DISTINCT FROM active_source_revision
+       OR NEW.source_owner_snapshot IS DISTINCT FROM source_owner
+       OR NEW.source_locator_snapshot IS DISTINCT FROM source_locator
+       OR NEW.attribution_text_snapshot IS DISTINCT FROM rights_attribution
+       OR NEW.source_contract_fingerprint_sha256 IS DISTINCT FROM rights_contract
+       OR NEW.parser_name IS DISTINCT FROM parse_name
+       OR NEW.parser_version IS DISTINCT FROM parse_version
+       OR NEW.parser_contract_fingerprint_sha256 IS DISTINCT FROM parse_contract
+       OR NEW.schema_fingerprint_sha256 IS DISTINCT FROM parse_observed_schema
+       OR NEW.typed_fingerprint_sha256 IS DISTINCT FROM root_typed_hash
+       OR NEW.source_first_observed_at IS DISTINCT FROM root_observed_at THEN
+        RAISE EXCEPTION 'warning snapshot publication is not canonical'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    RETURN NEW;
+END;
+$$;
+CREATE TRIGGER reviews_warning_snapshot_publication_guard
+BEFORE INSERT ON reviews_publicationrevision
+FOR EACH ROW
+WHEN (
+    NEW.module = 'TRAVEL_WARNING'
+    AND NEW.source_code_snapshot = 'MOFA_TRAVEL_ALARM_API_TRAVEL6_SET'
+)
+EXECUTE FUNCTION reviews_guard_warning_snapshot_publication();
+"""
+
+
+WARNING_SNAPSHOT_PUBLICATION_REVERSE_SQL = r"""
+DROP TRIGGER IF EXISTS reviews_warning_snapshot_publication_guard
+    ON reviews_publicationrevision;
+DROP FUNCTION IF EXISTS reviews_guard_warning_snapshot_publication();
+"""
+
+
+def _rewrite_publication_guard(schema_editor, old, new):
+    with schema_editor.connection.cursor() as cursor:
+        cursor.execute(
+            """
+            SELECT procedure.prosrc
+              FROM pg_proc AS procedure
+              JOIN pg_namespace AS namespace
+                ON namespace.oid = procedure.pronamespace
+             WHERE namespace.nspname = current_schema()
+               AND procedure.proname = %s
+               AND procedure.pronargs = 0
+            """,
+            [FUNCTION_NAME],
+        )
+        rows = cursor.fetchall()
+        if len(rows) != 1:
+            raise RuntimeError(f"expected one trigger function named {FUNCTION_NAME}")
+        body = rows[0][0]
+        if body.count(old) != 1:
+            raise RuntimeError(f"unexpected {FUNCTION_NAME} trigger definition")
+        body = body.replace(old, new)
+        quoted_name = schema_editor.quote_name(FUNCTION_NAME)
+        cursor.execute(
+            f"""
+            CREATE OR REPLACE FUNCTION {quoted_name}() RETURNS trigger
+            LANGUAGE plpgsql AS $reviews_warning_snapshot_contract$
+            {body}
+            $reviews_warning_snapshot_contract$
+            """
+        )
+
+
+def allow_warning_snapshot_publications(apps, schema_editor):
+    _rewrite_publication_guard(schema_editor, V1_WARNING_TAIL, V2_WARNING_TAIL)
+    with schema_editor.connection.cursor() as cursor:
+        cursor.execute(WARNING_SNAPSHOT_PUBLICATION_SQL)
+
+
+def restore_warning_v1_publications(apps, schema_editor):
+    PublicationRevision = apps.get_model("reviews", "PublicationRevision")
+    if PublicationRevision.objects.filter(
+        source_code_snapshot="MOFA_TRAVEL_ALARM_API_TRAVEL6_SET"
+    ).exists():
+        raise RuntimeError(
+            "warning snapshot publication rollback requires no v2 history"
+        )
+    with schema_editor.connection.cursor() as cursor:
+        cursor.execute(WARNING_SNAPSHOT_PUBLICATION_REVERSE_SQL)
+    _rewrite_publication_guard(schema_editor, V2_WARNING_TAIL, V1_WARNING_TAIL)
+
+
+class Migration(migrations.Migration):
+    dependencies = [
+        ("reviews", "0003_country_conditional_source_contracts"),
+        ("travel_warnings", "0004_warning_snapshot_facts"),
+    ]
+
+    operations = [
+        migrations.RemoveConstraint(
+            model_name="publicationrevision",
+            name="publication_exact_typed_scope",
+        ),
+        migrations.AddConstraint(
+            model_name="publicationrevision",
+            constraint=models.CheckConstraint(
+                condition=(
+                    (
+                        models.Q(
+                            module="ENTRY",
+                            scope_passport_policy_id=uuid.UUID(
+                                "f461f7a7-18f7-5b0d-9831-bfbd47b695e5"
+                            ),
+                            entry_fact_revision__isnull=False,
+                            travel_warning_revision__isnull=True,
+                            limitation_code=(
+                                "ENTRY_PURPOSE_DATE_APPLICABILITY_UNPROVEN"
+                            ),
+                            parser_name="MOFA_ENTRY_CSV",
+                            parser_version="V1",
+                        )
+                        & (
+                            models.Q(
+                                source_code_snapshot="MOFA_ENTRY_CSV_TRAVEL6"
+                            )
+                            | models.Q(
+                                scope_country_id=uuid.UUID(
+                                    "575fa8b9-14f9-526e-9464-ebd1dea76da9"
+                                ),
+                                source_code_snapshot="MOFA_ENTRY_CSV",
+                            )
+                        )
+                    )
+                    | (
+                        models.Q(
+                            module="TRAVEL_WARNING",
+                            scope_passport_policy__isnull=True,
+                            entry_fact_revision__isnull=True,
+                            travel_warning_revision__isnull=False,
+                            limitation_code=(
+                                "WARNING_EFFECTIVE_PERIOD_UNPROVEN"
+                            ),
+                            parser_name="MOFA_TRAVEL_ALARM_JSON",
+                        )
+                        & (
+                            models.Q(
+                                parser_version="V2",
+                                source_code_snapshot=(
+                                    "MOFA_TRAVEL_ALARM_API_TRAVEL6_SET"
+                                ),
+                            )
+                            | (
+                                models.Q(parser_version="V1")
+                                & (
+                                    models.Q(
+                                        source_code_snapshot=(
+                                            "MOFA_TRAVEL_ALARM_API_TRAVEL6"
+                                        )
+                                    )
+                                    | models.Q(
+                                        scope_country_id=uuid.UUID(
+                                            "575fa8b9-14f9-526e-9464-ebd1dea76da9"
+                                        ),
+                                        source_code_snapshot=(
+                                            "MOFA_TRAVEL_ALARM_API_JP"
+                                        ),
+                                    )
+                                )
+                            )
+                        )
+                    )
+                ),
+                name="publication_exact_typed_scope",
+            ),
+        ),
+        migrations.RunPython(
+            allow_warning_snapshot_publications,
+            restore_warning_v1_publications,
+        ),
+    ]
diff --git a/reviews/models.py b/reviews/models.py
index 9c68fe3..9496a09 100644
--- a/reviews/models.py
+++ b/reviews/models.py
@@ -264,20 +264,30 @@ class PublicationRevision(models.Model):
                                 "WARNING_EFFECTIVE_PERIOD_UNPROVEN"
                             ),
                             parser_name="MOFA_TRAVEL_ALARM_JSON",
-                            parser_version="V1",
                         )
                         & (
                             Q(
+                                parser_version="V2",
                                 source_code_snapshot=(
-                                    "MOFA_TRAVEL_ALARM_API_TRAVEL6"
-                                )
-                            )
-                            | Q(
-                                scope_country_id=JP_COUNTRY_ID,
-                                source_code_snapshot=(
-                                    "MOFA_TRAVEL_ALARM_API_JP"
+                                    "MOFA_TRAVEL_ALARM_API_TRAVEL6_SET"
                                 ),
                             )
+                            | (
+                                Q(parser_version="V1")
+                                & (
+                                    Q(
+                                        source_code_snapshot=(
+                                            "MOFA_TRAVEL_ALARM_API_TRAVEL6"
+                                        )
+                                    )
+                                    | Q(
+                                        scope_country_id=JP_COUNTRY_ID,
+                                        source_code_snapshot=(
+                                            "MOFA_TRAVEL_ALARM_API_JP"
+                                        ),
+                                    )
+                                )
+                            )
                         )
                     )
                 ),
diff --git a/reviews/publication.py b/reviews/publication.py
index 853a7aa..cf197d8 100644
--- a/reviews/publication.py
+++ b/reviews/publication.py
@@ -43,6 +43,16 @@ from sources.transport import (
     ENTRY_SOURCE_LOCATOR,
     WARNING_SECRET_REFERENCE,
     WARNING_SOURCE_LOCATOR,
+    warning_snapshot_request_fingerprint,
+)
+from travel_warnings.contracts import (
+    WARNING_SNAPSHOT_DECISION_BASIS,
+    WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256,
+    WARNING_SNAPSHOT_FIELD_SCOPE,
+    WARNING_SNAPSHOT_MAX_RESPONSE_BYTES,
+    WARNING_SNAPSHOT_PARSER_CONTRACT_FINGERPRINT_SHA256,
+    WARNING_SNAPSHOT_SOURCE_CODE,
+    WARNING_SNAPSHOT_SOURCE_REVISION,
 )
 from travel_warnings.ingestion import (
     WARNING_ATTRIBUTION,
@@ -140,6 +150,7 @@ class _ModuleSpec:
     access_mode: str
     secret_reference_name: str
     provider_code: str
+    max_artifact_bytes: int
 
 
 _SPECS = {
@@ -171,6 +182,7 @@ _SPECS = {
         access_mode=SourceRightsDecision.AccessMode.ANONYMOUS_PUBLIC,
         secret_reference_name="",
         provider_code="",
+        max_artifact_bytes=262_144,
     ),
     PublicationModule.TRAVEL_WARNING: _ModuleSpec(
         module=PublicationModule.TRAVEL_WARNING,
@@ -200,9 +212,24 @@ _SPECS = {
         access_mode=SourceRightsDecision.AccessMode.CREDENTIAL_REFERENCE,
         secret_reference_name=WARNING_SECRET_REFERENCE,
         provider_code=FetchAttempt.ProviderCode.MOFA_SUCCESS_0,
+        max_artifact_bytes=4_096,
     ),
 }
 
+_WARNING_SNAPSHOT_SPEC = replace(
+    _SPECS[PublicationModule.TRAVEL_WARNING],
+    source_code=WARNING_SNAPSHOT_SOURCE_CODE,
+    source_revision=WARNING_SNAPSHOT_SOURCE_REVISION,
+    field_scope=WARNING_SNAPSHOT_FIELD_SCOPE,
+    contract_fingerprint=(
+        WARNING_SNAPSHOT_PARSER_CONTRACT_FINGERPRINT_SHA256
+    ),
+    decision_basis=WARNING_SNAPSHOT_DECISION_BASIS,
+    parser_version=ParseRun.ParserVersion.V2,
+    schema_fingerprint=WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256,
+    max_artifact_bytes=WARNING_SNAPSHOT_MAX_RESPONSE_BYTES,
+)
+
 _LEGACY_SPECS = {
     PublicationModule.ENTRY: replace(
         _SPECS[PublicationModule.ENTRY],
@@ -360,12 +387,16 @@ def _lock_module(spec: _ModuleSpec) -> None:
 
 def _load_subject(spec: _ModuleSpec, subject_id: object):
     try:
+        related = [
+            "country",
+            "parse_run__artifact__source",
+            "parse_run__artifact__first_successful_attempt",
+        ]
+        if spec.module == PublicationModule.TRAVEL_WARNING:
+            related.append("observation_attempt")
         return (
-            spec.typed_model.objects.select_for_update()
-            .select_related(
-                "parse_run__artifact__source",
-                "parse_run__artifact__first_successful_attempt",
-            )
+            spec.typed_model.objects.select_for_update(of=("self",))
+            .select_related(*related)
             .get(pk=subject_id)
         )
     except Exception:
@@ -389,17 +420,102 @@ def _lock_subject_source(spec: _ModuleSpec, subject) -> None:
     _latest_rights(source)
 
 
-def _validate_source_chain(spec: _ModuleSpec, subject) -> dict[str, object]:
+def _subject_spec(spec: _ModuleSpec, subject) -> _ModuleSpec:
+    """Select an exact append-only contract for this typed revision."""
+
     source_code = subject.parse_run.artifact.source.code
+    if spec.module == PublicationModule.TRAVEL_WARNING:
+        if source_code == WARNING_SNAPSHOT_SOURCE_CODE:
+            return _WARNING_SNAPSHOT_SPEC
+        if source_code == spec.source_code:
+            return spec
+    elif source_code == spec.source_code:
+        return spec
+    legacy = _LEGACY_SPECS[spec.module]
+    if subject.country_id == JP_COUNTRY_ID and source_code == legacy.source_code:
+        return legacy
+    return spec
+
+
+def _locked_warning_attempt(spec: _ModuleSpec, subject) -> FetchAttempt:
+    """Lock and validate the warning root/fact shape without interpreting it."""
+
+    try:
+        facts = list(
+            subject.facts.select_for_update()
+            .order_by("source_position")
+            .only(
+                "source_position",
+                "source_alarm_level_code",
+                "source_scope_type",
+                "source_scope_text",
+                "source_written_date",
+                "typed_fingerprint_sha256",
+            )
+        )
+    except Exception:
+        raise _ClosedFailure(PublicationCode.SOURCE_GATE_FAILED) from None
+
+    if spec.source_code == WARNING_SNAPSHOT_SOURCE_CODE:
+        expected_positions = list(range(1, subject.source_item_count + 1))
+        if (
+            subject.observation_attempt_id is None
+            or subject.source_item_count < 0
+            or subject.source_item_count > 100
+            or subject.source_alarm_level_code is not None
+            or subject.source_scope_type is not None
+            or subject.source_scope_text is not None
+            or subject.source_written_date is not None
+            or len(facts) != subject.source_item_count
+            or [fact.source_position for fact in facts] != expected_positions
+        ):
+            raise _ClosedFailure(PublicationCode.SOURCE_GATE_FAILED)
+        try:
+            attempt = FetchAttempt.objects.select_for_update().get(
+                pk=subject.observation_attempt_id
+            )
+            expected_request = warning_snapshot_request_fingerprint(
+                subject.country.iso_alpha2
+            )
+        except Exception:
+            raise _ClosedFailure(PublicationCode.SOURCE_GATE_FAILED) from None
+        if (
+            attempt.request_fingerprint_revision != expected_request.revision
+            or attempt.normalized_request_sha256
+            != expected_request.normalized_request_sha256
+        ):
+            raise _ClosedFailure(PublicationCode.SOURCE_GATE_FAILED)
+        return attempt
+
     if (
-        subject.country_id == JP_COUNTRY_ID
-        and source_code == _LEGACY_SPECS[spec.module].source_code
+        subject.observation_attempt_id is not None
+        or subject.source_item_count != 1
+        or not subject.source_alarm_level_code
+        or not subject.source_scope_type
+        or not subject.source_scope_text
+        or facts
     ):
-        spec = _LEGACY_SPECS[spec.module]
+        raise _ClosedFailure(PublicationCode.SOURCE_GATE_FAILED)
+    try:
+        return FetchAttempt.objects.select_for_update().get(
+            pk=subject.parse_run.artifact.first_successful_attempt_id
+        )
+    except Exception:
+        raise _ClosedFailure(PublicationCode.SOURCE_GATE_FAILED) from None
+
+
+def _validate_source_chain(spec: _ModuleSpec, subject) -> dict[str, object]:
+    spec = _subject_spec(spec, subject)
     parse_run = subject.parse_run
     artifact = parse_run.artifact
     source = artifact.source
-    attempt = artifact.first_successful_attempt
+    attempt = (
+        _locked_warning_attempt(spec, subject)
+        if spec.module == PublicationModule.TRAVEL_WARNING
+        else FetchAttempt.objects.select_for_update().get(
+            pk=artifact.first_successful_attempt_id
+        )
+    )
     source = source.__class__.objects.select_for_update().get(pk=source.pk)
     rights = _latest_rights(source)
     if (
@@ -425,8 +541,7 @@ def _validate_source_chain(spec: _ModuleSpec, subject) -> dict[str, object]:
         != spec.schema_fingerprint
         or artifact.state != SourceArtifact.State.REVIEW_REQUIRED
         or artifact.byte_count < 1
-        or artifact.byte_count
-        > (262_144 if spec.module == PublicationModule.ENTRY else 4_096)
+        or artifact.byte_count > spec.max_artifact_bytes
         or attempt.source_id != source.id
         or attempt.source_revision != source.revision
         or attempt.rights_decision_id != getattr(rights, "id", None)
diff --git a/reviews/tests/test_admin.py b/reviews/tests/test_admin.py
index a5746e5..8c25f63 100644
--- a/reviews/tests/test_admin.py
+++ b/reviews/tests/test_admin.py
@@ -15,6 +15,7 @@ from reviews.admin import (
     EntryFactRevisionAdmin,
     PublicationRevisionAdmin,
     ReviewDecisionAdmin,
+    TravelWarningFactInline,
     TravelWarningRevisionAdmin,
     _GENERIC_FAILURE_MESSAGE,
     _OUTCOME_MESSAGES,
@@ -142,6 +143,28 @@ class AdminBoundaryTests(SimpleTestCase):
             self.assertFalse(model_admin.has_change_permission(request))
             self.assertFalse(model_admin.has_delete_permission(request))
 
+    def test_warning_candidates_expose_ordered_read_only_source_facts(self):
+        self.assertEqual(self.warning_admin.inlines, (TravelWarningFactInline,))
+        inline = TravelWarningFactInline(
+            TravelWarningRevision,
+            self.site,
+        )
+        self.assertEqual(
+            inline.fields,
+            (
+                "source_position",
+                "source_alarm_level_code",
+                "source_scope_type",
+                "source_scope_text",
+                "source_written_date",
+            ),
+        )
+        request = self.request("reviews.publish_warning")
+        self.assertFalse(inline.has_add_permission(request))
+        self.assertFalse(inline.has_change_permission(request))
+        self.assertFalse(inline.has_delete_permission(request))
+        self.assertIn("warning_source_facts", self.publication_admin.fields)
+
     def test_action_visibility_requires_exact_permission(self):
         request = self.request("reviews.publish_entry")
         actions = self.entry_admin.get_actions(request)
diff --git a/reviews/tests/test_publication.py b/reviews/tests/test_publication.py
index d7f471d..06791d6 100644
--- a/reviews/tests/test_publication.py
+++ b/reviews/tests/test_publication.py
@@ -465,7 +465,15 @@ class LegacyJpPublicationCompatibilityTests(TransactionTestCase):
             typed_fingerprint_sha256=typed_fingerprint_sha256(period, basis),
         )
 
-    def _legacy_warning(self, source, rights, *, suffix, scope_text):
+    def _legacy_warning(
+        self,
+        source,
+        rights,
+        *,
+        suffix,
+        scope_text,
+        model=TravelWarningRevision,
+    ):
         parse_run, attempt = self._legacy_parse_run(
             source=source,
             rights=rights,
@@ -480,9 +488,9 @@ class LegacyJpPublicationCompatibilityTests(TransactionTestCase):
         )
         self.assertIsInstance(parsed, TravelWarningParseSuccess)
         warning = parsed.warning
-        return TravelWarningRevision.objects.create(
+        return model.objects.create(
             country_id=JP_COUNTRY_ID,
-            parse_run=parse_run,
+            parse_run_id=parse_run.pk,
             source_alarm_level_code=warning.source_alarm_level_code,
             source_scope_type=warning.source_scope_type,
             source_scope_text=warning.source_scope_text,
@@ -493,12 +501,18 @@ class LegacyJpPublicationCompatibilityTests(TransactionTestCase):
 
     def test_legacy_jp_history_survives_forward_migrations_and_can_rollback(self):
         self.addCleanup(lambda: restore_canonical_migration_graph(connection))
-        MigrationExecutor(connection).migrate(
-            [
-                ("entry_requirements", "0001_initial"),
-                ("travel_warnings", "0002_revision_source_rights_guard"),
-                ("reviews", "0002_country_scoped_publications"),
-            ]
+        historical_targets = [
+            ("entry_requirements", "0001_initial"),
+            ("travel_warnings", "0002_revision_source_rights_guard"),
+            ("reviews", "0002_country_scoped_publications"),
+        ]
+        executor = MigrationExecutor(connection)
+        executor.migrate(historical_targets)
+        historical_apps = executor.loader.project_state(
+            historical_targets
+        ).apps
+        historical_warning_model = historical_apps.get_model(
+            "travel_warnings", "TravelWarningRevision"
         )
         Country.objects.get_or_create(
             id=JP_COUNTRY_ID,
@@ -539,12 +553,14 @@ class LegacyJpPublicationCompatibilityTests(TransactionTestCase):
             warning_rights,
             suffix="WARNING_FIRST",
             scope_text="합성 기존 경보 첫 번째",
+            model=historical_warning_model,
         )
         warning_second = self._legacy_warning(
             warning_source,
             warning_rights,
             suffix="WARNING_SECOND",
             scope_text="합성 기존 경보 두 번째",
+            model=historical_warning_model,
         )
         legacy_ids = {
             entry_first.pk,
@@ -552,6 +568,12 @@ class LegacyJpPublicationCompatibilityTests(TransactionTestCase):
             warning_first.pk,
             warning_second.pk,
         }
+        restore_canonical_migration_graph(connection)
+
+        entry_first = EntryFactRevision.objects.get(pk=entry_first.pk)
+        entry_second = EntryFactRevision.objects.get(pk=entry_second.pk)
+        warning_first = TravelWarningRevision.objects.get(pk=warning_first.pk)
+        warning_second = TravelWarningRevision.objects.get(pk=warning_second.pk)
         actor = get_user_model().objects.create_superuser(
             username="legacy-jp-publication-operator",
             password=None,
@@ -575,8 +597,6 @@ class LegacyJpPublicationCompatibilityTests(TransactionTestCase):
                 generation=1,
             ).pk
 
-        restore_canonical_migration_graph(connection)
-
         self.assertEqual(
             legacy_ids,
             {
@@ -586,19 +606,6 @@ class LegacyJpPublicationCompatibilityTests(TransactionTestCase):
                 TravelWarningRevision.objects.get(pk=warning_second.pk).pk,
             },
         )
-        self.assertEqual(
-            PublishedEntryFacts.objects.get(
-                country_id=JP_COUNTRY_ID,
-                passport_policy_id=PASSPORT_POLICY_ID,
-            ).current_publication_id,
-            first_publications[PublicationModule.ENTRY],
-        )
-        self.assertEqual(
-            PublishedTravelWarning.objects.get(
-                country_id=JP_COUNTRY_ID,
-            ).current_publication_id,
-            first_publications[PublicationModule.TRAVEL_WARNING],
-        )
         for module, second in (
             (PublicationModule.ENTRY, entry_second),
             (PublicationModule.TRAVEL_WARNING, warning_second),
diff --git a/reviews/tests/test_warning_snapshot_publication.py b/reviews/tests/test_warning_snapshot_publication.py
new file mode 100644
index 0000000..bd0c770
--- /dev/null
+++ b/reviews/tests/test_warning_snapshot_publication.py
@@ -0,0 +1,309 @@
+import hashlib
+import json
+import uuid
+from datetime import date, timedelta
+from io import StringIO
+
+from django.contrib import admin
+from django.contrib.auth import get_user_model
+from django.core.management import call_command
+from django.db import transaction
+from django.test import TransactionTestCase
+from django.utils import timezone
+
+from countries.models import SUPPORTED_COUNTRY_ROWS, Country
+from reviews.admin import PublicationRevisionAdmin
+from reviews.models import (
+    AuditEvent,
+    PublicationModule,
+    PublicationRevision,
+    PublishedTravelWarning,
+)
+from reviews.publication import (
+    PublicationCode,
+    publish_candidate,
+    rollback_publication,
+)
+from sources.models import (
+    FetchAttempt,
+    ParseRun,
+    SourceArtifact,
+    SourceConfiguration,
+    SourceRightsDecision,
+)
+from sources.transport import warning_snapshot_request_fingerprint
+from travel_warnings.contracts import (
+    WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256,
+    WARNING_SNAPSHOT_PARSER_CONTRACT_FINGERPRINT_SHA256,
+    WARNING_SNAPSHOT_SOURCE_CODE,
+)
+from travel_warnings.models import TravelWarningFact, TravelWarningRevision
+
+
+def _fingerprint(value) -> str:
+    payload = json.dumps(
+        value,
+        ensure_ascii=False,
+        sort_keys=True,
+        separators=(",", ":"),
+    ).encode("utf-8")
+    return hashlib.sha256(payload).hexdigest()
+
+
+def _fact_fingerprint(country, fact) -> str:
+    return _fingerprint(
+        {
+            "country_iso2": country.iso_alpha2,
+            "country_name_en": country.name_en,
+            "country_name_ko": country.name_ko,
+            "source_alarm_level_code": fact["source_alarm_level_code"],
+            "source_scope_text": fact["source_scope_text"],
+            "source_scope_type": fact["source_scope_type"],
+            "source_written_date": (
+                fact["source_written_date"].isoformat()
+                if fact["source_written_date"] is not None
+                else None
+            ),
+        }
+    )
+
+
+def _snapshot_fingerprint(country, facts) -> str:
+    return _fingerprint(
+        {
+            "country_iso2": country.iso_alpha2,
+            "country_name_en": country.name_en,
+            "country_name_ko": country.name_ko,
+            "facts": [
+                {
+                    "source_alarm_level_code": fact[
+                        "source_alarm_level_code"
+                    ],
+                    "source_scope_text": fact["source_scope_text"],
+                    "source_scope_type": fact["source_scope_type"],
+                    "source_written_date": (
+                        fact["source_written_date"].isoformat()
+                        if fact["source_written_date"] is not None
+                        else None
+                    ),
+                }
+                for fact in facts
+            ],
+        }
+    )
+
+
+class WarningSnapshotPublicationTests(TransactionTestCase):
+    def setUp(self):
+        for country_id, iso2, name_ko, name_en, is_public in (
+            SUPPORTED_COUNTRY_ROWS
+        ):
+            Country.objects.get_or_create(
+                id=country_id,
+                defaults={
+                    "iso_alpha2": iso2,
+                    "name_ko": name_ko,
+                    "name_en": name_en,
+                    "is_public": is_public,
+                },
+            )
+        call_command("register_approved_sources", "--apply", stdout=StringIO())
+        self.country = Country.objects.get(iso_alpha2="TW")
+        self.source = SourceConfiguration.objects.get(
+            code=WARNING_SNAPSHOT_SOURCE_CODE
+        )
+        self.rights = self.source.rights_decisions.get(decision_sequence=1)
+        self.actor = get_user_model().objects.create_superuser(
+            username="warning-snapshot-publication-operator",
+            password=None,
+        )
+
+    def _candidate(self, label, facts):
+        started_at = timezone.now() - timedelta(minutes=2)
+        completed_at = started_at + timedelta(seconds=1)
+        body_hash = hashlib.sha256(label.encode("ascii")).hexdigest()
+        request = warning_snapshot_request_fingerprint("TW")
+        attempt = FetchAttempt.objects.create(
+            source=self.source,
+            source_revision=self.source.revision,
+            rights_decision=self.rights,
+            operation_id=uuid.uuid4(),
+            attempt_number=1,
+            request_fingerprint_revision=request.revision,
+            normalized_request_sha256=request.normalized_request_sha256,
+            started_at=started_at,
+        )
+        FetchAttempt.objects.filter(pk=attempt.pk).update(
+            completed_at=completed_at,
+            result=FetchAttempt.Result.SUCCEEDED,
+            http_status=200,
+            provider_code=FetchAttempt.ProviderCode.MOFA_SUCCESS_0,
+            response_sha256=body_hash,
+        )
+        attempt.refresh_from_db()
+        artifact = SourceArtifact.objects.create(
+            source=self.source,
+            body_sha256=body_hash,
+            byte_count=16_384,
+            first_successful_attempt=attempt,
+        )
+        parse_run = ParseRun.objects.create(
+            artifact=artifact,
+            parser_name=ParseRun.ParserName.MOFA_TRAVEL_ALARM_JSON,
+            parser_version=ParseRun.ParserVersion.V2,
+            parser_contract_fingerprint_sha256=(
+                WARNING_SNAPSHOT_PARSER_CONTRACT_FINGERPRINT_SHA256
+            ),
+            expected_schema_fingerprint_sha256=(
+                WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256
+            ),
+            started_at=completed_at + timedelta(seconds=1),
+        )
+        ParseRun.objects.filter(pk=parse_run.pk).update(
+            completed_at=parse_run.started_at + timedelta(seconds=1),
+            outcome=ParseRun.Outcome.VALIDATED,
+            observed_schema_fingerprint_sha256=(
+                WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256
+            ),
+        )
+        SourceArtifact.objects.filter(pk=artifact.pk).update(
+            state=SourceArtifact.State.REVIEW_REQUIRED
+        )
+        with transaction.atomic():
+            revision = TravelWarningRevision.objects.create(
+                country=self.country,
+                parse_run=parse_run,
+                observation_attempt=attempt,
+                source_item_count=len(facts),
+                first_observed_at=attempt.completed_at,
+                typed_fingerprint_sha256=_snapshot_fingerprint(
+                    self.country, facts
+                ),
+            )
+            for position, fact in enumerate(facts, start=1):
+                TravelWarningFact.objects.create(
+                    revision=revision,
+                    source_position=position,
+                    typed_fingerprint_sha256=_fact_fingerprint(
+                        self.country, fact
+                    ),
+                    **fact,
+                )
+        return revision
+
+    def test_verified_empty_and_ordered_facts_publish_and_rollback(self):
+        verified_empty = self._candidate("EMPTY", ())
+        first = publish_candidate(
+            module=PublicationModule.TRAVEL_WARNING,
+            typed_revision_id=verified_empty.pk,
+            actor=self.actor,
+            expected_pointer_version=0,
+        )
+        self.assertEqual(first.code, PublicationCode.PUBLISHED)
+
+        facts = (
+            {
+                "source_alarm_level_code": "1",
+                "source_scope_type": "일부",
+                "source_scope_text": "타이베이 일부 지역",
+                "source_written_date": date(2026, 8, 30),
+            },
+            {
+                "source_alarm_level_code": "2",
+                "source_scope_type": "전역",
+                "source_scope_text": "출처가 제공한 두 번째 범위",
+                "source_written_date": None,
+            },
+        )
+        ordered = self._candidate("ORDERED", facts)
+        second = publish_candidate(
+            module=PublicationModule.TRAVEL_WARNING,
+            typed_revision_id=ordered.pk,
+            actor=self.actor,
+            expected_pointer_version=1,
+        )
+        self.assertEqual(second.code, PublicationCode.PUBLISHED)
+
+        first_publication = PublicationRevision.objects.get(generation=1)
+        restored = rollback_publication(
+            module=PublicationModule.TRAVEL_WARNING,
+            target_publication_id=first_publication.pk,
+            actor=self.actor,
+            expected_pointer_version=2,
+        )
+        self.assertEqual(restored.code, PublicationCode.ROLLED_BACK)
+
+        publications = list(PublicationRevision.objects.order_by("generation"))
+        self.assertEqual(
+            [item.travel_warning_revision_id for item in publications],
+            [verified_empty.pk, ordered.pk, verified_empty.pk],
+        )
+        self.assertTrue(
+            all(
+                item.parser_version == ParseRun.ParserVersion.V2
+                for item in publications
+            )
+        )
+        self.assertTrue(
+            all(
+                item.source_code_snapshot == WARNING_SNAPSHOT_SOURCE_CODE
+                for item in publications
+            )
+        )
+        pointer = PublishedTravelWarning.objects.get(country=self.country)
+        self.assertEqual(pointer.version, 3)
+        self.assertEqual(
+            pointer.current_publication.travel_warning_revision_id,
+            verified_empty.pk,
+        )
+        self.assertEqual(AuditEvent.objects.filter(outcome="SUCCEEDED").count(), 3)
+
+        publication_admin = PublicationRevisionAdmin(
+            PublicationRevision, admin.AdminSite(name="warning-history-test")
+        )
+        self.assertIn(
+            "공식 출처의 국가 전체 조회 결과 항목 0건",
+            str(publication_admin.warning_source_facts(publications[0])),
+        )
+        ordered_summary = str(
+            publication_admin.warning_source_facts(publications[1])
+        )
+        self.assertLess(
+            ordered_summary.index("타이베이 일부 지역"),
+            ordered_summary.index("출처가 제공한 두 번째 범위"),
+        )
+
+    def test_latest_exact_v2_rights_are_required(self):
+        candidate = self._candidate("REJECTED_RIGHTS", ())
+        SourceRightsDecision.objects.create(
+            source=self.source,
+            source_revision=self.source.revision,
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
+            contract_fingerprint_sha256=(
+                WARNING_SNAPSHOT_PARSER_CONTRACT_FINGERPRINT_SHA256
+            ),
+            decided_by="PROJECT_OWNER_REQUEST",
+            decision_basis_code="SYNTHETIC_REJECTION_TEST",
+        )
+
+        outcome = publish_candidate(
+            module=PublicationModule.TRAVEL_WARNING,
+            typed_revision_id=candidate.pk,
+            actor=self.actor,
+            expected_pointer_version=0,
+        )
+
+        self.assertEqual(outcome.code, PublicationCode.SOURCE_GATE_FAILED)
+        self.assertFalse(PublicationRevision.objects.exists())
