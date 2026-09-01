## `feat(warnings): persist country warning snapshots`

diff --git a/travel_warnings/migrations/0004_warning_snapshot_facts.py b/travel_warnings/migrations/0004_warning_snapshot_facts.py
new file mode 100644
index 0000000..402c05a
--- /dev/null
+++ b/travel_warnings/migrations/0004_warning_snapshot_facts.py
@@ -0,0 +1,654 @@
+import django.db.models.deletion
+import uuid
+from django.db import migrations, models
+
+
+WARNING_SNAPSHOT_GUARDS_SQL = r"""
+DROP TRIGGER IF EXISTS travel_warnings_revision_change_guard
+    ON travel_warnings_travelwarningrevision;
+DROP TRIGGER IF EXISTS travel_warnings_revision_source_rights_guard
+    ON travel_warnings_travelwarningrevision;
+
+CREATE TRIGGER travel_warnings_revision_change_guard
+BEFORE INSERT ON travel_warnings_travelwarningrevision
+FOR EACH ROW
+WHEN (NEW.observation_attempt_id IS NULL)
+EXECUTE FUNCTION travel_warnings_guard_revision_change();
+
+CREATE TRIGGER travel_warnings_revision_source_rights_guard
+BEFORE INSERT ON travel_warnings_travelwarningrevision
+FOR EACH ROW
+WHEN (NEW.observation_attempt_id IS NULL)
+EXECUTE FUNCTION travel_warnings_validate_revision_source_rights();
+
+CREATE FUNCTION travel_warnings_guard_revision_immutable() RETURNS trigger
+LANGUAGE plpgsql AS $$
+BEGIN
+    RAISE EXCEPTION 'travel warning revisions are immutable'
+        USING ERRCODE = 'check_violation';
+END;
+$$;
+CREATE TRIGGER travel_warnings_revision_immutable_guard
+BEFORE UPDATE OR DELETE ON travel_warnings_travelwarningrevision
+FOR EACH ROW EXECUTE FUNCTION travel_warnings_guard_revision_immutable();
+
+CREATE FUNCTION travel_warnings_validate_snapshot_revision() RETURNS trigger
+LANGUAGE plpgsql AS $$
+DECLARE
+    parse_outcome text;
+    parse_name text;
+    parse_version text;
+    parser_contract text;
+    expected_schema text;
+    observed_schema text;
+    parse_started_at timestamptz;
+    parse_completed_at timestamptz;
+    artifact_source_id uuid;
+    artifact_state text;
+    artifact_byte_count bigint;
+    artifact_body_sha text;
+    artifact_attempt_id uuid;
+    anchor_source_id uuid;
+    anchor_source_revision text;
+    anchor_result text;
+    anchor_http_status integer;
+    anchor_provider_code text;
+    anchor_completed_at timestamptz;
+    anchor_response_sha text;
+    source_code text;
+    active_source_revision text;
+    source_module text;
+    source_state text;
+    source_enabled boolean;
+    source_secret_reference text;
+    attempt_source_id uuid;
+    attempt_source_revision text;
+    attempt_rights_id uuid;
+    attempt_result text;
+    attempt_http_status integer;
+    attempt_provider_code text;
+    attempt_completed_at timestamptz;
+    attempt_response_sha text;
+    attempt_request_revision text;
+    attempt_request_sha text;
+    country_iso2 text;
+    expected_request_sha text;
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
+    SELECT parse_run.outcome, parse_run.parser_name, parse_run.parser_version,
+           parse_run.parser_contract_fingerprint_sha256,
+           parse_run.expected_schema_fingerprint_sha256,
+           parse_run.observed_schema_fingerprint_sha256,
+           parse_run.started_at, parse_run.completed_at,
+           artifact.source_id, artifact.state, artifact.byte_count,
+           artifact.body_sha256, artifact.first_successful_attempt_id,
+           anchor.source_id, anchor.source_revision, anchor.result,
+           anchor.http_status, anchor.provider_code, anchor.completed_at,
+           anchor.response_sha256,
+           source.code, source.revision, source.module, source.state,
+           source.enabled, source.secret_reference_name,
+           attempt.source_id, attempt.source_revision,
+           attempt.rights_decision_id, attempt.result, attempt.http_status,
+           attempt.provider_code, attempt.completed_at,
+           attempt.response_sha256, attempt.request_fingerprint_revision,
+           attempt.normalized_request_sha256, country.iso_alpha2
+      INTO parse_outcome, parse_name, parse_version, parser_contract,
+           expected_schema, observed_schema, parse_started_at,
+           parse_completed_at, artifact_source_id, artifact_state,
+           artifact_byte_count, artifact_body_sha, artifact_attempt_id,
+           anchor_source_id, anchor_source_revision, anchor_result,
+           anchor_http_status, anchor_provider_code, anchor_completed_at,
+           anchor_response_sha,
+           source_code, active_source_revision, source_module, source_state,
+           source_enabled, source_secret_reference, attempt_source_id,
+           attempt_source_revision, attempt_rights_id, attempt_result,
+           attempt_http_status, attempt_provider_code, attempt_completed_at,
+           attempt_response_sha, attempt_request_revision,
+           attempt_request_sha, country_iso2
+      FROM sources_parserun AS parse_run
+      JOIN sources_sourceartifact AS artifact
+        ON artifact.id = parse_run.artifact_id
+      JOIN sources_sourceconfiguration AS source
+        ON source.id = artifact.source_id
+      JOIN sources_fetchattempt AS anchor
+        ON anchor.id = artifact.first_successful_attempt_id
+      JOIN sources_fetchattempt AS attempt
+        ON attempt.id = NEW.observation_attempt_id
+      JOIN countries_country AS country
+        ON country.id = NEW.country_id
+     WHERE parse_run.id = NEW.parse_run_id;
+    IF NOT FOUND THEN
+        RAISE EXCEPTION 'warning snapshot requires exact source evidence'
+            USING ERRCODE = 'check_violation';
+    END IF;
+
+    IF parse_outcome IS DISTINCT FROM 'VALIDATED'
+       OR parse_name IS DISTINCT FROM 'MOFA_TRAVEL_ALARM_JSON'
+       OR parse_version IS DISTINCT FROM 'V2'
+       OR parser_contract IS DISTINCT FROM
+          '23d8a89ec668b856c05f8ebff448322f1fda457407c3b803b96225b990567dd4'
+       OR expected_schema IS DISTINCT FROM
+          '2bb572cd61be540f0e8189ccb29dbd8278671c88af0106a8f16f551bf5118c2d'
+       OR observed_schema IS DISTINCT FROM expected_schema THEN
+        RAISE EXCEPTION 'warning snapshot parser identity is not approved'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    IF source_code IS DISTINCT FROM 'MOFA_TRAVEL_ALARM_API_TRAVEL6_SET'
+       OR active_source_revision IS DISTINCT FROM 'travel6-set-v2'
+       OR source_module IS DISTINCT FROM 'TRAVEL_WARNING'
+       OR source_state IS DISTINCT FROM 'ACTIVE'
+       OR source_enabled IS DISTINCT FROM true
+       OR source_secret_reference IS DISTINCT FROM 'DATA_GO_KR_SERVICE_KEY' THEN
+        RAISE EXCEPTION 'warning snapshot requires the exact active source'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    IF artifact_state IS DISTINCT FROM 'REVIEW_REQUIRED'
+       OR artifact_byte_count < 1
+       OR artifact_byte_count > 262144 THEN
+        RAISE EXCEPTION 'warning snapshot artifact is not a bounded candidate'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    IF artifact_attempt_id IS NULL
+       OR anchor_source_id IS DISTINCT FROM artifact_source_id
+       OR anchor_source_revision IS DISTINCT FROM active_source_revision
+       OR anchor_result IS DISTINCT FROM 'SUCCEEDED'
+       OR anchor_http_status IS DISTINCT FROM 200
+       OR anchor_provider_code IS DISTINCT FROM 'MOFA_SUCCESS_0'
+       OR anchor_completed_at IS NULL
+       OR anchor_response_sha IS DISTINCT FROM artifact_body_sha THEN
+        RAISE EXCEPTION 'warning snapshot artifact anchor is inconsistent'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    expected_request_sha := encode(sha256(convert_to(
+        'GET' || E'\n' ||
+        'https' || E'\n' ||
+        'apis.data.go.kr' || E'\n' ||
+        '/1262000/TravelAlarmService2/getTravelAlarmList2' || E'\n' ||
+        'ServiceKey=%3Credacted%3E&returnType=JSON&numOfRows=100&' ||
+        'pageNo=1&cond%5Bcountry_iso_alp2%3A%3AEQ%5D=' || country_iso2,
+        'UTF8'
+    )), 'hex');
+    IF attempt_source_id IS DISTINCT FROM artifact_source_id
+       OR attempt_source_revision IS DISTINCT FROM active_source_revision
+       OR attempt_result IS DISTINCT FROM 'SUCCEEDED'
+       OR attempt_http_status IS DISTINCT FROM 200
+       OR attempt_provider_code IS DISTINCT FROM 'MOFA_SUCCESS_0'
+       OR attempt_completed_at IS NULL
+       OR attempt_response_sha IS DISTINCT FROM artifact_body_sha
+       OR attempt_request_revision IS DISTINCT FROM 'MOFA_WARNING_V2'
+       OR attempt_request_sha IS DISTINCT FROM expected_request_sha
+       OR NEW.first_observed_at IS DISTINCT FROM attempt_completed_at THEN
+        RAISE EXCEPTION 'warning snapshot observation is inconsistent'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    IF NEW.source_item_count < 0 OR NEW.source_item_count > 100
+       OR NEW.source_alarm_level_code IS NOT NULL
+       OR NEW.source_scope_type IS NOT NULL
+       OR NEW.source_scope_text IS NOT NULL
+       OR NEW.source_written_date IS NOT NULL THEN
+        RAISE EXCEPTION 'warning snapshot root contains scalar fact values'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    IF parse_started_at IS NULL OR parse_completed_at IS NULL
+       OR NEW.created_at IS NULL
+       OR anchor_completed_at > parse_started_at
+       OR parse_started_at > parse_completed_at
+       OR parse_completed_at > NEW.created_at
+       OR attempt_completed_at > NEW.created_at THEN
+        RAISE EXCEPTION 'warning snapshot evidence times are not causal'
+            USING ERRCODE = 'check_violation';
+    END IF;
+
+    SELECT rights.id, rights.source_revision, rights.decision_sequence,
+           rights.decision, rights.access_mode, rights.access_allowed,
+           rights.automated_collection_allowed,
+           rights.typed_field_storage_allowed,
+           rights.raw_body_storage_allowed,
+           rights.typed_republication_allowed, rights.raw_retention_seconds,
+           rights.typed_retention, rights.evidence_retention,
+           rights.field_scope_code, rights.attribution_text,
+           rights.contract_fingerprint_sha256
+      INTO latest_rights_id, latest_rights_revision, latest_rights_sequence,
+           latest_rights_decision, latest_access_mode,
+           latest_access_allowed, latest_automation_allowed,
+           latest_typed_storage_allowed, latest_raw_storage_allowed,
+           latest_typed_republication_allowed,
+           latest_raw_retention_seconds, latest_typed_retention,
+           latest_evidence_retention, latest_field_scope,
+           latest_attribution, latest_contract_fingerprint
+      FROM sources_sourcerightsdecision AS rights
+     WHERE rights.source_id = artifact_source_id
+       AND rights.source_revision = active_source_revision
+     ORDER BY rights.decision_sequence DESC
+     LIMIT 1;
+    IF NOT FOUND
+       OR latest_rights_id IS DISTINCT FROM attempt_rights_id
+       OR latest_rights_revision IS DISTINCT FROM 'travel6-set-v2'
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
+       OR latest_field_scope IS DISTINCT FROM 'SUPPORTED_6_WARNING_SET_V2'
+       OR latest_attribution IS DISTINCT FROM '외교부|공공데이터포털'
+       OR latest_contract_fingerprint IS DISTINCT FROM
+          '23d8a89ec668b856c05f8ebff448322f1fda457407c3b803b96225b990567dd4' THEN
+        RAISE EXCEPTION 'warning snapshot requires the latest exact rights grant'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    RETURN NEW;
+END;
+$$;
+CREATE TRIGGER travel_warnings_snapshot_revision_guard
+BEFORE INSERT ON travel_warnings_travelwarningrevision
+FOR EACH ROW
+WHEN (NEW.observation_attempt_id IS NOT NULL)
+EXECUTE FUNCTION travel_warnings_validate_snapshot_revision();
+
+CREATE FUNCTION travel_warnings_guard_fact_change() RETURNS trigger
+LANGUAGE plpgsql AS $$
+DECLARE
+    root_item_count integer;
+    root_created_at timestamptz;
+    root_observation_attempt_id uuid;
+    parse_version text;
+    source_code text;
+    country_iso2 text;
+    country_name_en text;
+    country_name_ko text;
+    canonical_typed_projection text;
+    expected_typed_fingerprint text;
+BEGIN
+    IF TG_OP IN ('UPDATE', 'DELETE') THEN
+        RAISE EXCEPTION 'travel warning facts are immutable'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    SELECT root.source_item_count, root.created_at,
+           root.observation_attempt_id, parse_run.parser_version,
+           source.code, country.iso_alpha2, country.name_en, country.name_ko
+      INTO root_item_count, root_created_at, root_observation_attempt_id,
+           parse_version, source_code, country_iso2, country_name_en,
+           country_name_ko
+      FROM travel_warnings_travelwarningrevision AS root
+      JOIN sources_parserun AS parse_run ON parse_run.id = root.parse_run_id
+      JOIN sources_sourceartifact AS artifact
+        ON artifact.id = parse_run.artifact_id
+      JOIN sources_sourceconfiguration AS source
+        ON source.id = artifact.source_id
+      JOIN countries_country AS country ON country.id = root.country_id
+     WHERE root.id = NEW.revision_id;
+    IF NOT FOUND
+       OR root_observation_attempt_id IS NULL
+       OR parse_version IS DISTINCT FROM 'V2'
+       OR source_code IS DISTINCT FROM 'MOFA_TRAVEL_ALARM_API_TRAVEL6_SET'
+       OR NEW.source_position < 1
+       OR NEW.source_position > root_item_count
+       OR NEW.created_at IS NULL
+       OR NEW.created_at < root_created_at THEN
+        RAISE EXCEPTION 'warning fact requires an open v2 snapshot position'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    IF NEW.source_alarm_level_code ~ '^[[:space:]]*$'
+       OR NEW.source_scope_type ~ '^[[:space:]]*$'
+       OR NEW.source_scope_text ~ '^[[:space:]]*$' THEN
+        RAISE EXCEPTION 'warning fact values must be non-empty'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    canonical_typed_projection :=
+        '{"country_iso2":' || to_jsonb(country_iso2)::text ||
+        ',"country_name_en":' || to_jsonb(country_name_en)::text ||
+        ',"country_name_ko":' || to_jsonb(country_name_ko)::text ||
+        ',"source_alarm_level_code":' ||
+            to_jsonb(NEW.source_alarm_level_code)::text ||
+        ',"source_scope_text":' || to_jsonb(NEW.source_scope_text)::text ||
+        ',"source_scope_type":' || to_jsonb(NEW.source_scope_type)::text ||
+        ',"source_written_date":' ||
+            CASE
+                WHEN NEW.source_written_date IS NULL THEN 'null'
+                ELSE to_jsonb(
+                    to_char(NEW.source_written_date, 'YYYY-MM-DD')
+                )::text
+            END ||
+        '}';
+    expected_typed_fingerprint := encode(
+        sha256(convert_to(canonical_typed_projection, 'UTF8')),
+        'hex'
+    );
+    IF NEW.typed_fingerprint_sha256 IS DISTINCT FROM expected_typed_fingerprint THEN
+        RAISE EXCEPTION 'warning fact typed fingerprint is inconsistent'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    RETURN NEW;
+END;
+$$;
+CREATE TRIGGER travel_warnings_fact_change_guard
+BEFORE INSERT OR UPDATE OR DELETE ON travel_warnings_travelwarningfact
+FOR EACH ROW EXECUTE FUNCTION travel_warnings_guard_fact_change();
+
+CREATE FUNCTION travel_warnings_validate_snapshot_closure() RETURNS trigger
+LANGUAGE plpgsql AS $$
+DECLARE
+    parse_version text;
+    expected_count integer;
+    observed_count integer;
+    minimum_position integer;
+    maximum_position integer;
+    country_iso2 text;
+    country_name_en text;
+    country_name_ko text;
+    facts_projection text;
+    canonical_snapshot text;
+    expected_fingerprint text;
+BEGIN
+    SELECT parse_run.parser_version, root.source_item_count,
+           country.iso_alpha2, country.name_en, country.name_ko
+      INTO parse_version, expected_count, country_iso2,
+           country_name_en, country_name_ko
+      FROM travel_warnings_travelwarningrevision AS root
+      JOIN sources_parserun AS parse_run ON parse_run.id = root.parse_run_id
+      JOIN countries_country AS country ON country.id = root.country_id
+     WHERE root.id = NEW.id;
+    IF NOT FOUND THEN
+        RAISE EXCEPTION 'warning snapshot root is missing'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    SELECT COUNT(*), MIN(source_position), MAX(source_position)
+      INTO observed_count, minimum_position, maximum_position
+      FROM travel_warnings_travelwarningfact
+     WHERE revision_id = NEW.id;
+    IF parse_version = 'V1' THEN
+        IF observed_count <> 0 THEN
+            RAISE EXCEPTION 'legacy warning revisions cannot have child facts'
+                USING ERRCODE = 'check_violation';
+        END IF;
+        RETURN NEW;
+    END IF;
+    IF parse_version IS DISTINCT FROM 'V2'
+       OR observed_count <> expected_count
+       OR (expected_count = 0 AND (
+            minimum_position IS NOT NULL OR maximum_position IS NOT NULL
+       ))
+       OR (expected_count > 0 AND (
+            minimum_position <> 1 OR maximum_position <> expected_count
+       )) THEN
+        RAISE EXCEPTION 'warning snapshot fact count or order is incomplete'
+            USING ERRCODE = 'check_violation';
+    END IF;
+
+    SELECT string_agg(
+        '{"source_alarm_level_code":' ||
+            to_jsonb(fact.source_alarm_level_code)::text ||
+        ',"source_scope_text":' || to_jsonb(fact.source_scope_text)::text ||
+        ',"source_scope_type":' || to_jsonb(fact.source_scope_type)::text ||
+        ',"source_written_date":' ||
+            CASE
+                WHEN fact.source_written_date IS NULL THEN 'null'
+                ELSE to_jsonb(
+                    to_char(fact.source_written_date, 'YYYY-MM-DD')
+                )::text
+            END ||
+        '}',
+        ',' ORDER BY fact.source_position
+    ) INTO facts_projection
+      FROM travel_warnings_travelwarningfact AS fact
+     WHERE fact.revision_id = NEW.id;
+    canonical_snapshot :=
+        '{"country_iso2":' || to_jsonb(country_iso2)::text ||
+        ',"country_name_en":' || to_jsonb(country_name_en)::text ||
+        ',"country_name_ko":' || to_jsonb(country_name_ko)::text ||
+        ',"facts":[' || COALESCE(facts_projection, '') || ']}';
+    expected_fingerprint := encode(
+        sha256(convert_to(canonical_snapshot, 'UTF8')),
+        'hex'
+    );
+    IF NEW.typed_fingerprint_sha256 IS DISTINCT FROM expected_fingerprint THEN
+        RAISE EXCEPTION 'warning snapshot fingerprint is inconsistent'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    RETURN NEW;
+END;
+$$;
+CREATE CONSTRAINT TRIGGER travel_warnings_snapshot_closure_guard
+AFTER INSERT ON travel_warnings_travelwarningrevision
+DEFERRABLE INITIALLY DEFERRED
+FOR EACH ROW EXECUTE FUNCTION travel_warnings_validate_snapshot_closure();
+"""
+
+
+WARNING_SNAPSHOT_GUARDS_REVERSE_SQL = r"""
+LOCK TABLE travel_warnings_travelwarningrevision,
+           travel_warnings_travelwarningfact IN ACCESS EXCLUSIVE MODE;
+DO $$
+BEGIN
+    IF EXISTS (SELECT 1 FROM travel_warnings_travelwarningfact)
+       OR EXISTS (
+            SELECT 1
+              FROM travel_warnings_travelwarningrevision
+             WHERE observation_attempt_id IS NOT NULL
+                OR source_item_count <> 1
+                OR source_alarm_level_code IS NULL
+                OR source_scope_type IS NULL
+                OR source_scope_text IS NULL
+       ) THEN
+        RAISE EXCEPTION 'warning snapshot rollback requires legacy scalar roots'
+            USING ERRCODE = 'check_violation';
+    END IF;
+END;
+$$;
+
+DROP TRIGGER IF EXISTS travel_warnings_snapshot_closure_guard
+    ON travel_warnings_travelwarningrevision;
+DROP FUNCTION IF EXISTS travel_warnings_validate_snapshot_closure();
+DROP TRIGGER IF EXISTS travel_warnings_fact_change_guard
+    ON travel_warnings_travelwarningfact;
+DROP FUNCTION IF EXISTS travel_warnings_guard_fact_change();
+DROP TRIGGER IF EXISTS travel_warnings_snapshot_revision_guard
+    ON travel_warnings_travelwarningrevision;
+DROP FUNCTION IF EXISTS travel_warnings_validate_snapshot_revision();
+DROP TRIGGER IF EXISTS travel_warnings_revision_immutable_guard
+    ON travel_warnings_travelwarningrevision;
+DROP FUNCTION IF EXISTS travel_warnings_guard_revision_immutable();
+
+DROP TRIGGER IF EXISTS travel_warnings_revision_change_guard
+    ON travel_warnings_travelwarningrevision;
+DROP TRIGGER IF EXISTS travel_warnings_revision_source_rights_guard
+    ON travel_warnings_travelwarningrevision;
+CREATE TRIGGER travel_warnings_revision_change_guard
+BEFORE INSERT OR UPDATE OR DELETE ON travel_warnings_travelwarningrevision
+FOR EACH ROW EXECUTE FUNCTION travel_warnings_guard_revision_change();
+CREATE TRIGGER travel_warnings_revision_source_rights_guard
+BEFORE INSERT ON travel_warnings_travelwarningrevision
+FOR EACH ROW EXECUTE FUNCTION travel_warnings_validate_revision_source_rights();
+"""
+
+
+class Migration(migrations.Migration):
+    dependencies = [
+        ("sources", "0017_warning_snapshot_parser_contract"),
+        ("travel_warnings", "0003_country_scoped_warning_revisions"),
+    ]
+
+    operations = [
+        migrations.AlterField(
+            model_name="travelwarningrevision",
+            name="parse_run",
+            field=models.ForeignKey(
+                on_delete=django.db.models.deletion.PROTECT,
+                related_name="travel_warning_revisions",
+                to="sources.parserun",
+            ),
+        ),
+        migrations.AlterField(
+            model_name="travelwarningrevision",
+            name="source_alarm_level_code",
+            field=models.CharField(blank=True, max_length=32, null=True),
+        ),
+        migrations.AlterField(
+            model_name="travelwarningrevision",
+            name="source_scope_text",
+            field=models.CharField(blank=True, max_length=1000, null=True),
+        ),
+        migrations.AlterField(
+            model_name="travelwarningrevision",
+            name="source_scope_type",
+            field=models.CharField(blank=True, max_length=100, null=True),
+        ),
+        migrations.AddField(
+            model_name="travelwarningrevision",
+            name="observation_attempt",
+            field=models.ForeignKey(
+                blank=True,
+                null=True,
+                on_delete=django.db.models.deletion.PROTECT,
+                related_name="travel_warning_snapshot_revisions",
+                to="sources.fetchattempt",
+            ),
+        ),
+        migrations.AddField(
+            model_name="travelwarningrevision",
+            name="source_item_count",
+            field=models.PositiveSmallIntegerField(default=1),
+        ),
+        migrations.CreateModel(
+            name="TravelWarningFact",
+            fields=[
+                (
+                    "id",
+                    models.UUIDField(
+                        default=uuid.uuid4,
+                        editable=False,
+                        primary_key=True,
+                        serialize=False,
+                    ),
+                ),
+                ("source_position", models.PositiveSmallIntegerField()),
+                ("source_alarm_level_code", models.CharField(max_length=32)),
+                ("source_scope_type", models.CharField(max_length=100)),
+                ("source_scope_text", models.CharField(max_length=1000)),
+                ("source_written_date", models.DateField(blank=True, null=True)),
+                ("typed_fingerprint_sha256", models.CharField(max_length=64)),
+                ("created_at", models.DateTimeField(auto_now_add=True)),
+                (
+                    "revision",
+                    models.ForeignKey(
+                        on_delete=django.db.models.deletion.PROTECT,
+                        related_name="facts",
+                        to="travel_warnings.travelwarningrevision",
+                    ),
+                ),
+            ],
+            options={
+                "ordering": ("source_position",),
+                "constraints": [
+                    models.UniqueConstraint(
+                        fields=("revision", "source_position"),
+                        name="warning_fact_revision_position_unique",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(
+                            source_position__gte=1,
+                            source_position__lte=100,
+                        ),
+                        name="warning_fact_position_bounded",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(
+                            ("source_alarm_level_code", ""), _negated=True
+                        ),
+                        name="warning_fact_level_present",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(
+                            ("source_scope_type", ""), _negated=True
+                        ),
+                        name="warning_fact_scope_type_present",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(
+                            ("source_scope_text", ""), _negated=True
+                        ),
+                        name="warning_fact_scope_text_present",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(
+                            typed_fingerprint_sha256__regex=r"^[0-9a-f]{64}$"
+                        ),
+                        name="warning_fact_typed_fingerprint_format",
+                    ),
+                ],
+            },
+        ),
+        migrations.AddConstraint(
+            model_name="travelwarningrevision",
+            constraint=models.CheckConstraint(
+                condition=models.Q(
+                    source_item_count__gte=0,
+                    source_item_count__lte=100,
+                ),
+                name="warning_revision_item_count_bounded",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="travelwarningrevision",
+            constraint=models.CheckConstraint(
+                condition=models.Q(
+                    models.Q(
+                        observation_attempt__isnull=True,
+                        source_item_count=1,
+                        source_alarm_level_code__isnull=False,
+                        source_scope_type__isnull=False,
+                        source_scope_text__isnull=False,
+                    ),
+                    models.Q(
+                        observation_attempt__isnull=False,
+                        source_alarm_level_code__isnull=True,
+                        source_scope_type__isnull=True,
+                        source_scope_text__isnull=True,
+                        source_written_date__isnull=True,
+                    ),
+                    _connector="OR",
+                ),
+                name="warning_revision_version_shape",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="travelwarningrevision",
+            constraint=models.UniqueConstraint(
+                fields=("parse_run", "country"),
+                name="warning_revision_run_country_unique",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="travelwarningrevision",
+            constraint=models.UniqueConstraint(
+                condition=models.Q(observation_attempt__isnull=True),
+                fields=("parse_run",),
+                name="warning_revision_v1_run_unique",
+            ),
+        ),
+        migrations.RunSQL(
+            WARNING_SNAPSHOT_GUARDS_SQL,
+            WARNING_SNAPSHOT_GUARDS_REVERSE_SQL,
+        ),
+    ]
diff --git a/travel_warnings/models.py b/travel_warnings/models.py
index 582828f..1ab019e 100644
--- a/travel_warnings/models.py
+++ b/travel_warnings/models.py
@@ -18,18 +18,36 @@ class TravelWarningRevision(models.Model):
         on_delete=models.PROTECT,
         related_name="travel_warning_revisions",
     )
-    parse_run = models.OneToOneField(
+    parse_run = models.ForeignKey(
         "sources.ParseRun",
         on_delete=models.PROTECT,
-        related_name="travel_warning_revision",
+        related_name="travel_warning_revisions",
     )
     source_alarm_level_code = models.CharField(
-        max_length=SOURCE_ALARM_LEVEL_CODE_MAX_LENGTH
+        max_length=SOURCE_ALARM_LEVEL_CODE_MAX_LENGTH,
+        null=True,
+        blank=True,
+    )
+    source_scope_type = models.CharField(
+        max_length=SOURCE_SCOPE_TYPE_MAX_LENGTH,
+        null=True,
+        blank=True,
+    )
+    source_scope_text = models.CharField(
+        max_length=SOURCE_SCOPE_TEXT_MAX_LENGTH,
+        null=True,
+        blank=True,
     )
-    source_scope_type = models.CharField(max_length=SOURCE_SCOPE_TYPE_MAX_LENGTH)
-    source_scope_text = models.CharField(max_length=SOURCE_SCOPE_TEXT_MAX_LENGTH)
     source_written_date = models.DateField(null=True, blank=True)
     first_observed_at = models.DateTimeField()
+    source_item_count = models.PositiveSmallIntegerField(default=1)
+    observation_attempt = models.ForeignKey(
+        "sources.FetchAttempt",
+        on_delete=models.PROTECT,
+        related_name="travel_warning_snapshot_revisions",
+        null=True,
+        blank=True,
+    )
     typed_fingerprint_sha256 = models.CharField(max_length=64)
     created_at = models.DateTimeField(auto_now_add=True)
 
@@ -61,4 +79,86 @@ class TravelWarningRevision(models.Model):
                 condition=Q(created_at__gte=F("first_observed_at")),
                 name="warning_created_after_observed",
             ),
+            models.CheckConstraint(
+                condition=Q(
+                    source_item_count__gte=0,
+                    source_item_count__lte=100,
+                ),
+                name="warning_revision_item_count_bounded",
+            ),
+            models.CheckConstraint(
+                condition=(
+                    Q(
+                        observation_attempt__isnull=True,
+                        source_item_count=1,
+                        source_alarm_level_code__isnull=False,
+                        source_scope_type__isnull=False,
+                        source_scope_text__isnull=False,
+                    )
+                    | Q(
+                        observation_attempt__isnull=False,
+                        source_alarm_level_code__isnull=True,
+                        source_scope_type__isnull=True,
+                        source_scope_text__isnull=True,
+                        source_written_date__isnull=True,
+                    )
+                ),
+                name="warning_revision_version_shape",
+            ),
+            models.UniqueConstraint(
+                fields=("parse_run", "country"),
+                name="warning_revision_run_country_unique",
+            ),
+            models.UniqueConstraint(
+                fields=("parse_run",),
+                condition=Q(observation_attempt__isnull=True),
+                name="warning_revision_v1_run_unique",
+            ),
+        ]
+
+
+class TravelWarningFact(models.Model):
+    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
+    revision = models.ForeignKey(
+        TravelWarningRevision,
+        on_delete=models.PROTECT,
+        related_name="facts",
+    )
+    source_position = models.PositiveSmallIntegerField()
+    source_alarm_level_code = models.CharField(
+        max_length=SOURCE_ALARM_LEVEL_CODE_MAX_LENGTH
+    )
+    source_scope_type = models.CharField(max_length=SOURCE_SCOPE_TYPE_MAX_LENGTH)
+    source_scope_text = models.CharField(max_length=SOURCE_SCOPE_TEXT_MAX_LENGTH)
+    source_written_date = models.DateField(null=True, blank=True)
+    typed_fingerprint_sha256 = models.CharField(max_length=64)
+    created_at = models.DateTimeField(auto_now_add=True)
+
+    class Meta:
+        ordering = ("source_position",)
+        constraints = [
+            models.UniqueConstraint(
+                fields=("revision", "source_position"),
+                name="warning_fact_revision_position_unique",
+            ),
+            models.CheckConstraint(
+                condition=Q(source_position__gte=1, source_position__lte=100),
+                name="warning_fact_position_bounded",
+            ),
+            models.CheckConstraint(
+                condition=~Q(source_alarm_level_code=""),
+                name="warning_fact_level_present",
+            ),
+            models.CheckConstraint(
+                condition=~Q(source_scope_type=""),
+                name="warning_fact_scope_type_present",
+            ),
+            models.CheckConstraint(
+                condition=~Q(source_scope_text=""),
+                name="warning_fact_scope_text_present",
+            ),
+            models.CheckConstraint(
+                condition=Q(typed_fingerprint_sha256__regex=r"^[0-9a-f]{64}$"),
+                name="warning_fact_typed_fingerprint_format",
+            ),
         ]
diff --git a/travel_warnings/tests/test_models.py b/travel_warnings/tests/test_models.py
index 306b874..2117c14 100644
--- a/travel_warnings/tests/test_models.py
+++ b/travel_warnings/tests/test_models.py
@@ -231,11 +231,13 @@ class TravelWarningRevisionTests(TravelWarningFixtureMixin, TestCase):
                 "id",
                 "country",
                 "parse_run",
+                "observation_attempt",
                 "source_alarm_level_code",
                 "source_scope_type",
                 "source_scope_text",
                 "source_written_date",
                 "first_observed_at",
+                "source_item_count",
                 "typed_fingerprint_sha256",
                 "created_at",
             },
@@ -269,7 +271,9 @@ class TravelWarningRevisionTests(TravelWarningFixtureMixin, TestCase):
         )
         self.assertIs(fields["country"].remote_field.on_delete, models.PROTECT)
         self.assertIs(fields["parse_run"].remote_field.on_delete, models.PROTECT)
-        self.assertTrue(fields["parse_run"].one_to_one)
+        self.assertFalse(fields["parse_run"].one_to_one)
+        self.assertEqual(revision.source_item_count, 1)
+        self.assertIsNone(revision.observation_attempt_id)
 
     def test_source_written_date_is_nullable(self):
         revision = TravelWarningRevision.objects.create(
diff --git a/travel_warnings/tests/test_snapshot_migration.py b/travel_warnings/tests/test_snapshot_migration.py
new file mode 100644
index 0000000..cbfd589
--- /dev/null
+++ b/travel_warnings/tests/test_snapshot_migration.py
@@ -0,0 +1,93 @@
+from io import StringIO
+
+from django.core.management import call_command
+from django.db import DatabaseError, connection, transaction
+from django.db.migrations.executor import MigrationExecutor
+from django.test import TransactionTestCase
+
+from operations.tests.migration_helpers import restore_canonical_migration_graph
+from travel_warnings.models import TravelWarningRevision
+from travel_warnings.tests.test_models import TravelWarningFixtureMixin
+from travel_warnings.tests.test_snapshot_models import (
+    TravelWarningSnapshotFixtureMixin,
+)
+
+
+class TravelWarningSnapshotMigrationTests(
+    TravelWarningFixtureMixin,
+    TransactionTestCase,
+):
+    latest = [("travel_warnings", "0004_warning_snapshot_facts")]
+    previous = [("travel_warnings", "0003_country_scoped_warning_revisions")]
+
+    def restore_latest(self):
+        restore_canonical_migration_graph(connection)
+
+    def _fixture_teardown(self):
+        try:
+            super()._fixture_teardown()
+        finally:
+            self.restore_latest()
+
+    def setUp(self):
+        self.addCleanup(self.restore_latest)
+        self.restore_latest()
+        call_command("register_approved_sources", "--apply", stdout=StringIO())
+
+    def test_legacy_scalar_revision_survives_reverse_and_reapply(self):
+        run, _, attempt = self.make_parse("SNAPSHOT_MIGRATION_V1")
+        revision = TravelWarningRevision.objects.create(
+            **self.revision_values(run, attempt)
+        )
+
+        executor = MigrationExecutor(connection)
+        executor.migrate(self.previous)
+        old_apps = executor.loader.project_state(self.previous).apps
+        OldRevision = old_apps.get_model(
+            "travel_warnings", "TravelWarningRevision"
+        )
+        historical = OldRevision.objects.get(pk=revision.pk)
+        self.assertEqual(historical.source_alarm_level_code, "3")
+        self.assertEqual(historical.source_scope_text, "합성 검증 범위")
+
+        MigrationExecutor(connection).migrate(self.latest)
+        restored = TravelWarningRevision.objects.get(pk=revision.pk)
+        self.assertEqual(restored.source_item_count, 1)
+        self.assertIsNone(restored.observation_attempt_id)
+        self.assertEqual(
+            restored.typed_fingerprint_sha256,
+            revision.typed_fingerprint_sha256,
+        )
+
+
+class TravelWarningSnapshotAppendOnlyMigrationTests(
+    TravelWarningSnapshotFixtureMixin,
+    TransactionTestCase,
+):
+    latest = [("travel_warnings", "0004_warning_snapshot_facts")]
+    previous = [("travel_warnings", "0003_country_scoped_warning_revisions")]
+
+    def restore_latest(self):
+        restore_canonical_migration_graph(connection)
+
+    def _fixture_teardown(self):
+        try:
+            super()._fixture_teardown()
+        finally:
+            self.restore_latest()
+
+    def setUp(self):
+        self.restore_latest()
+        self.addCleanup(self.restore_latest)
+        super().setUp()
+
+    def test_reverse_refuses_to_discard_snapshot_roots(self):
+        with transaction.atomic():
+            root = TravelWarningRevision.objects.create(
+                **self._root_values(())
+            )
+
+        with self.assertRaises(DatabaseError):
+            MigrationExecutor(connection).migrate(self.previous)
+
+        self.assertTrue(TravelWarningRevision.objects.filter(pk=root.pk).exists())
diff --git a/travel_warnings/tests/test_snapshot_models.py b/travel_warnings/tests/test_snapshot_models.py
new file mode 100644
index 0000000..76874c5
--- /dev/null
+++ b/travel_warnings/tests/test_snapshot_models.py
@@ -0,0 +1,322 @@
+import hashlib
+import json
+import uuid
+from datetime import date, timedelta
+from io import StringIO
+
+from django.core.management import call_command
+from django.db import IntegrityError, transaction
+from django.test import TransactionTestCase
+from django.utils import timezone
+
+from countries.models import Country
+from sources.models import FetchAttempt, ParseRun, SourceArtifact, SourceConfiguration
+from sources.transport import warning_snapshot_request_fingerprint
+from travel_warnings.contracts import (
+    WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256,
+    WARNING_SNAPSHOT_PARSER_CONTRACT_FINGERPRINT_SHA256,
+    WARNING_SNAPSHOT_SOURCE_CODE,
+)
+from travel_warnings.models import TravelWarningFact, TravelWarningRevision
+
+
+def _fingerprint(value):
+    canonical = json.dumps(
+        value,
+        ensure_ascii=False,
+        sort_keys=True,
+        separators=(",", ":"),
+    ).encode("utf-8")
+    return hashlib.sha256(canonical).hexdigest()
+
+
+def _fact_fingerprint(country, values):
+    return _fingerprint(
+        {
+            "country_iso2": country.iso_alpha2,
+            "country_name_en": country.name_en,
+            "country_name_ko": country.name_ko,
+            "source_alarm_level_code": values["source_alarm_level_code"],
+            "source_scope_text": values["source_scope_text"],
+            "source_scope_type": values["source_scope_type"],
+            "source_written_date": (
+                values["source_written_date"].isoformat()
+                if values["source_written_date"] is not None
+                else None
+            ),
+        }
+    )
+
+
+def _snapshot_fingerprint(country, facts):
+    return _fingerprint(
+        {
+            "country_iso2": country.iso_alpha2,
+            "country_name_en": country.name_en,
+            "country_name_ko": country.name_ko,
+            "facts": [
+                {
+                    "source_alarm_level_code": values[
+                        "source_alarm_level_code"
+                    ],
+                    "source_scope_text": values["source_scope_text"],
+                    "source_scope_type": values["source_scope_type"],
+                    "source_written_date": (
+                        values["source_written_date"].isoformat()
+                        if values["source_written_date"] is not None
+                        else None
+                    ),
+                }
+                for values in facts
+            ],
+        }
+    )
+
+
+class TravelWarningSnapshotFixtureMixin:
+    def setUp(self):
+        call_command("register_approved_sources", "--apply", stdout=StringIO())
+        self.country = Country.objects.get(iso_alpha2="TW")
+        self.run, self.attempt = self._make_v2_parse()
+
+    def _make_v2_parse(self):
+        source = SourceConfiguration.objects.get(code=WARNING_SNAPSHOT_SOURCE_CODE)
+        rights = source.rights_decisions.get()
+        started_at = timezone.now() - timedelta(minutes=2)
+        completed_at = started_at + timedelta(seconds=1)
+        body_sha256 = hashlib.sha256(b"warning snapshot").hexdigest()
+        attempt = self._make_success_attempt(
+            source=source,
+            rights=rights,
+            country_iso2="TW",
+            body_sha256=body_sha256,
+            started_at=started_at,
+            completed_at=completed_at,
+        )
+        artifact = SourceArtifact.objects.create(
+            source=source,
+            body_sha256=body_sha256,
+            byte_count=4096,
+            first_successful_attempt=attempt,
+        )
+        run = ParseRun.objects.create(
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
+        ParseRun.objects.filter(pk=run.pk).update(
+            completed_at=run.started_at + timedelta(seconds=1),
+            outcome=ParseRun.Outcome.VALIDATED,
+            observed_schema_fingerprint_sha256=(
+                WARNING_SNAPSHOT_EXPECTED_SCHEMA_FINGERPRINT_SHA256
+            ),
+        )
+        run.refresh_from_db()
+        SourceArtifact.objects.filter(pk=artifact.pk).update(
+            state=SourceArtifact.State.REVIEW_REQUIRED
+        )
+        return run, attempt
+
+    def _make_success_attempt(
+        self,
+        *,
+        source,
+        rights,
+        country_iso2,
+        body_sha256,
+        started_at,
+        completed_at,
+        request_revision=None,
+        request_sha=None,
+    ):
+        request_fingerprint = warning_snapshot_request_fingerprint(country_iso2)
+        attempt = FetchAttempt.objects.create(
+            source=source,
+            source_revision=source.revision,
+            rights_decision=rights,
+            operation_id=uuid.uuid4(),
+            attempt_number=1,
+            request_fingerprint_revision=(
+                request_revision or request_fingerprint.revision
+            ),
+            normalized_request_sha256=(
+                request_sha or request_fingerprint.normalized_request_sha256
+            ),
+            started_at=started_at,
+        )
+        FetchAttempt.objects.filter(pk=attempt.pk).update(
+            completed_at=completed_at,
+            result=FetchAttempt.Result.SUCCEEDED,
+            http_status=200,
+            provider_code=FetchAttempt.ProviderCode.MOFA_SUCCESS_0,
+            response_sha256=body_sha256,
+        )
+        attempt.refresh_from_db()
+        return attempt
+
+    def _root_values(self, facts, *, country=None, attempt=None):
+        country = country or self.country
+        attempt = attempt or self.attempt
+        return {
+            "country": country,
+            "parse_run": self.run,
+            "observation_attempt": attempt,
+            "source_item_count": len(facts),
+            "first_observed_at": attempt.completed_at,
+            "typed_fingerprint_sha256": _snapshot_fingerprint(
+                country, facts
+            ),
+        }
+
+    def _create_fact(self, root, position, values, *, fingerprint=None):
+        return TravelWarningFact.objects.create(
+            revision=root,
+            source_position=position,
+            typed_fingerprint_sha256=(
+                fingerprint or _fact_fingerprint(self.country, values)
+            ),
+            **values,
+        )
+
+
+class TravelWarningSnapshotPersistenceTests(
+    TravelWarningSnapshotFixtureMixin,
+    TransactionTestCase,
+):
+
+    def test_complete_ordered_snapshot_is_persisted_and_immutable(self):
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
+                "source_scope_text": "두 번째 제공 순서",
+                "source_written_date": None,
+            },
+        )
+        with transaction.atomic():
+            root = TravelWarningRevision.objects.create(**self._root_values(facts))
+            first = self._create_fact(root, 1, facts[0])
+            second = self._create_fact(root, 2, facts[1])
+
+        root.refresh_from_db()
+        self.assertEqual(root.source_item_count, 2)
+        self.assertEqual(root.observation_attempt_id, self.attempt.pk)
+        self.assertIsNone(root.source_alarm_level_code)
+        self.assertEqual(
+            list(root.facts.values_list("source_position", flat=True)), [1, 2]
+        )
+        for action in (
+            lambda: TravelWarningRevision.objects.filter(pk=root.pk).update(
+                source_item_count=1
+            ),
+            lambda: TravelWarningFact.objects.filter(pk=first.pk).update(
+                source_scope_text="changed"
+            ),
+            lambda: TravelWarningFact.objects.filter(pk=second.pk).delete(),
+        ):
+            with self.assertRaises(IntegrityError), transaction.atomic():
+                action()
+
+    def test_incomplete_order_wrong_fact_hash_and_empty_hash_are_closed(self):
+        fact = {
+            "source_alarm_level_code": "1",
+            "source_scope_type": "일부",
+            "source_scope_text": "한 개 사실",
+            "source_written_date": None,
+        }
+        with self.assertRaises(IntegrityError), transaction.atomic():
+            root = TravelWarningRevision.objects.create(
+                **self._root_values((fact, fact))
+            )
+            self._create_fact(root, 1, fact)
+        self.assertFalse(TravelWarningRevision.objects.exists())
+
+        with self.assertRaises(IntegrityError), transaction.atomic():
+            root = TravelWarningRevision.objects.create(
+                **self._root_values((fact,))
+            )
+            self._create_fact(root, 1, fact, fingerprint="f" * 64)
+        self.assertFalse(TravelWarningRevision.objects.exists())
+
+        with transaction.atomic():
+            empty = TravelWarningRevision.objects.create(
+                **self._root_values(())
+            )
+        self.assertEqual(empty.source_item_count, 0)
+        self.assertFalse(empty.facts.exists())
+
+    def test_observation_request_must_match_root_country_and_v2_contract(self):
+        source = self.run.artifact.source
+        rights = source.rights_decisions.get()
+        started_at = self.run.completed_at + timedelta(seconds=1)
+        completed_at = started_at + timedelta(seconds=1)
+        macau_attempt = self._make_success_attempt(
+            source=source,
+            rights=rights,
+            country_iso2="MO",
+            body_sha256=self.run.artifact.body_sha256,
+            started_at=started_at,
+            completed_at=completed_at,
+        )
+        with self.assertRaises(IntegrityError), transaction.atomic():
+            TravelWarningRevision.objects.create(
+                **self._root_values((), attempt=macau_attempt)
+            )
+
+        wrong_revision_attempt = self._make_success_attempt(
+            source=source,
+            rights=rights,
+            country_iso2="TW",
+            body_sha256=self.run.artifact.body_sha256,
+            started_at=started_at + timedelta(seconds=2),
+            completed_at=completed_at + timedelta(seconds=2),
+            request_revision="MOFA_WARNING_V1",
+        )
+        with self.assertRaises(IntegrityError), transaction.atomic():
+            TravelWarningRevision.objects.create(
+                **self._root_values((), attempt=wrong_revision_attempt)
+            )
+
+    def test_shared_empty_artifact_accepts_country_specific_observations(self):
+        macau = Country.objects.get(iso_alpha2="MO")
+        source = self.run.artifact.source
+        rights = source.rights_decisions.get()
+        started_at = self.run.completed_at + timedelta(seconds=1)
+        completed_at = started_at + timedelta(seconds=1)
+        macau_attempt = self._make_success_attempt(
+            source=source,
+            rights=rights,
+            country_iso2="MO",
+            body_sha256=self.run.artifact.body_sha256,
+            started_at=started_at,
+            completed_at=completed_at,
+        )
+
+        with transaction.atomic():
+            taiwan_root = TravelWarningRevision.objects.create(
+                **self._root_values(())
+            )
+            macau_root = TravelWarningRevision.objects.create(
+                **self._root_values((), country=macau, attempt=macau_attempt)
+            )
+
+        self.assertEqual(
+            self.run.artifact.first_successful_attempt_id,
+            self.attempt.pk,
+        )
+        self.assertEqual(taiwan_root.observation_attempt_id, self.attempt.pk)
+        self.assertEqual(macau_root.observation_attempt_id, macau_attempt.pk)
+        self.assertLess(self.run.completed_at, macau_attempt.completed_at)


