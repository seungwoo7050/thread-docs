# 입국요건·여행경보의 운영자 검수와 원자적 게시·롤백

## `feat(reviews): publish approved facts atomically`

diff --git a/reviews/__init__.py b/reviews/__init__.py
new file mode 100644
index 0000000..8b13789
--- /dev/null
+++ b/reviews/__init__.py
@@ -0,0 +1 @@
+
diff --git a/reviews/apps.py b/reviews/apps.py
new file mode 100644
index 0000000..31ca513
--- /dev/null
+++ b/reviews/apps.py
@@ -0,0 +1,6 @@
+from django.apps import AppConfig
+
+
+class ReviewsConfig(AppConfig):
+    default_auto_field = "django.db.models.BigAutoField"
+    name = "reviews"
diff --git a/reviews/migrations/0001_initial.py b/reviews/migrations/0001_initial.py
new file mode 100644
index 0000000..a2a27e6
--- /dev/null
+++ b/reviews/migrations/0001_initial.py
@@ -0,0 +1,1683 @@
+import uuid
+
+import django.db.models.deletion
+import django.utils.timezone
+from django.conf import settings
+from django.db import migrations, models
+from django.db.models import F, Q
+
+
+JP_COUNTRY_ID = uuid.UUID("575fa8b9-14f9-526e-9464-ebd1dea76da9")
+PASSPORT_POLICY_ID = uuid.UUID("f461f7a7-18f7-5b0d-9831-bfbd47b695e5")
+ENTRY_POINTER_ID = uuid.UUID("6c3602d7-a7ec-53e2-9628-59c09922f332")
+WARNING_POINTER_ID = uuid.UUID("ec830f45-da57-5dfd-a289-f11a5acde9e2")
+JP_COUNTRY = (JP_COUNTRY_ID, "JP", "일본", "Japan", True)
+PASSPORT_POLICY = (
+    PASSPORT_POLICY_ID,
+    "KOR_ORDINARY_SHORT_TOURISM",
+    "V1",
+)
+
+
+def seed_module_pointers(apps, schema_editor):
+    Country = apps.get_model("countries", "Country")
+    PassportPolicy = apps.get_model("entry_requirements", "PassportPolicy")
+    Publication = apps.get_model("reviews", "PublicationRevision")
+    EntryPointer = apps.get_model("reviews", "PublishedEntryFacts")
+    WarningPointer = apps.get_model("reviews", "PublishedTravelWarning")
+    alias = schema_editor.connection.alias
+
+    # A test database can reach this migration after an app-scoped migration
+    # exercise has flushed data from an already-created prerequisite table.
+    # Restore and verify the fixed identities before inserting pointer FKs.
+    country, _ = Country.objects.using(alias).get_or_create(
+        id=JP_COUNTRY_ID,
+        defaults={
+            "iso_alpha2": JP_COUNTRY[1],
+            "name_ko": JP_COUNTRY[2],
+            "name_en": JP_COUNTRY[3],
+            "is_public": JP_COUNTRY[4],
+        },
+    )
+    if (
+        country.id,
+        country.iso_alpha2,
+        country.name_ko,
+        country.name_en,
+        country.is_public,
+    ) != JP_COUNTRY:
+        raise RuntimeError("JP country identity conflicts with the fixed scope")
+
+    policy, _ = PassportPolicy.objects.using(alias).get_or_create(
+        id=PASSPORT_POLICY_ID,
+        defaults={
+            "code": PASSPORT_POLICY[1],
+            "revision": PASSPORT_POLICY[2],
+        },
+    )
+    if (policy.id, policy.code, policy.revision) != PASSPORT_POLICY:
+        raise RuntimeError("passport policy identity conflicts with the fixed scope")
+
+    latest_entry = (
+        Publication.objects.using(alias)
+        .filter(module="ENTRY")
+        .order_by("-generation")
+        .values("id", "generation")
+        .first()
+    )
+    latest_warning = (
+        Publication.objects.using(alias)
+        .filter(module="TRAVEL_WARNING")
+        .order_by("-generation")
+        .values("id", "generation")
+        .first()
+    )
+    entry_history = latest_entry is not None
+    warning_history = latest_warning is not None
+    entry = EntryPointer.objects.using(alias).filter(
+        id=ENTRY_POINTER_ID
+    ).first()
+    warning = WarningPointer.objects.using(alias).filter(
+        id=WARNING_POINTER_ID
+    ).first()
+    if entry is None:
+        if entry_history:
+            raise RuntimeError("entry publication history has no current pointer")
+        entry = EntryPointer.objects.using(alias).create(
+            id=ENTRY_POINTER_ID,
+            country_id=JP_COUNTRY_ID,
+            passport_policy_id=PASSPORT_POLICY_ID,
+            current_publication_id=None,
+            version=0,
+        )
+    if warning is None:
+        if warning_history:
+            raise RuntimeError("warning publication history has no current pointer")
+        warning = WarningPointer.objects.using(alias).create(
+            id=WARNING_POINTER_ID,
+            country_id=JP_COUNTRY_ID,
+            current_publication_id=None,
+            version=0,
+        )
+    if (
+        entry_history
+        and (entry.current_publication_id is None or entry.version < 1)
+    ):
+        raise RuntimeError("entry publication history has an initial pointer")
+    if (
+        warning_history
+        and (warning.current_publication_id is None or warning.version < 1)
+    ):
+        raise RuntimeError("warning publication history has an initial pointer")
+    entry_current_is_exact = (
+        not entry_history
+        and entry.current_publication_id is None
+        and entry.version == 0
+    ) or (
+        entry_history
+        and entry.current_publication_id == latest_entry["id"]
+        and entry.version == latest_entry["generation"]
+    )
+    warning_current_is_exact = (
+        not warning_history
+        and warning.current_publication_id is None
+        and warning.version == 0
+    ) or (
+        warning_history
+        and warning.current_publication_id == latest_warning["id"]
+        and warning.version == latest_warning["generation"]
+    )
+    if (
+        entry.country_id != JP_COUNTRY_ID
+        or entry.passport_policy_id != PASSPORT_POLICY_ID
+        or not entry_current_is_exact
+        or warning.country_id != JP_COUNTRY_ID
+        or not warning_current_is_exact
+    ):
+        raise RuntimeError("publication pointer seed conflicts with fixed scope")
+
+
+def unseed_module_pointers(apps, schema_editor):
+    EntryPointer = apps.get_model("reviews", "PublishedEntryFacts")
+    WarningPointer = apps.get_model("reviews", "PublishedTravelWarning")
+    alias = schema_editor.connection.alias
+    EntryPointer.objects.using(alias).filter(id=ENTRY_POINTER_ID).delete()
+    WarningPointer.objects.using(alias).filter(id=WARNING_POINTER_ID).delete()
+
+
+REVIEWS_GUARDS_SQL = r"""
+CREATE FUNCTION reviews_guard_review_decision() RETURNS trigger
+LANGUAGE plpgsql AS $$
+DECLARE
+    subject_created_at timestamptz;
+    previous_sequence bigint;
+    actor_username text;
+    actor_active boolean;
+    actor_staff boolean;
+    actor_superuser boolean;
+    subject_key text;
+    required_permission text;
+BEGIN
+    IF TG_OP = 'UPDATE' OR TG_OP = 'DELETE' THEN
+        RAISE EXCEPTION 'review decisions are append-only'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    IF NEW.module = 'ENTRY' THEN
+        SELECT created_at INTO subject_created_at
+          FROM entry_requirements_entryfactrevision
+         WHERE id = NEW.entry_fact_revision_id;
+        subject_key := NEW.entry_fact_revision_id::text;
+    ELSIF NEW.module = 'TRAVEL_WARNING' THEN
+        SELECT created_at INTO subject_created_at
+          FROM travel_warnings_travelwarningrevision
+         WHERE id = NEW.travel_warning_revision_id;
+        subject_key := NEW.travel_warning_revision_id::text;
+    ELSE
+        RAISE EXCEPTION 'review module is outside the publication allowlist'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    IF subject_created_at IS NULL OR NEW.decided_at < subject_created_at THEN
+        RAISE EXCEPTION 'review decision time is not causal'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    SELECT username, is_active, is_staff, is_superuser
+      INTO actor_username, actor_active, actor_staff, actor_superuser
+      FROM auth_user WHERE id = NEW.actor_id FOR UPDATE;
+    IF NOT FOUND OR NOT actor_active OR NOT actor_staff
+       OR actor_username IS DISTINCT FROM NEW.actor_principal THEN
+        RAISE EXCEPTION 'review decision requires the exact active staff actor'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    required_permission := CASE
+        WHEN NEW.module = 'ENTRY' AND NEW.decision = 'REJECTED'
+            THEN 'reject_entry'
+        WHEN NEW.module = 'ENTRY' AND NEW.reason_code = 'ROLLBACK_ENTRY'
+            THEN 'rollback_entry'
+        WHEN NEW.module = 'ENTRY' THEN 'publish_entry'
+        WHEN NEW.module = 'TRAVEL_WARNING' AND NEW.decision = 'REJECTED'
+            THEN 'reject_warning'
+        WHEN NEW.module = 'TRAVEL_WARNING'
+             AND NEW.reason_code = 'ROLLBACK_WARNING'
+            THEN 'rollback_warning'
+        ELSE 'publish_warning'
+    END;
+    IF NOT actor_superuser AND NOT EXISTS (
+        SELECT 1
+          FROM auth_permission AS permission
+          JOIN django_content_type AS content_type
+            ON content_type.id = permission.content_type_id
+         WHERE content_type.app_label = 'reviews'
+           AND permission.codename = required_permission
+           AND (
+                EXISTS (
+                    SELECT 1 FROM auth_user_user_permissions AS direct_permission
+                     WHERE direct_permission.user_id = NEW.actor_id
+                       AND direct_permission.permission_id = permission.id
+                )
+                OR EXISTS (
+                    SELECT 1
+                      FROM auth_user_groups AS membership
+                      JOIN auth_group_permissions AS group_permission
+                        ON group_permission.group_id = membership.group_id
+                     WHERE membership.user_id = NEW.actor_id
+                       AND group_permission.permission_id = permission.id
+                )
+           )
+    ) THEN
+        RAISE EXCEPTION 'review decision requires the module action permission'
+            USING ERRCODE = 'insufficient_privilege';
+    END IF;
+    PERFORM pg_advisory_xact_lock(
+        hashtextextended(NEW.module || ':' || subject_key, 22007)
+    );
+    IF NEW.module = 'ENTRY' THEN
+        SELECT decision_sequence INTO previous_sequence
+          FROM reviews_reviewdecision
+         WHERE module = 'ENTRY'
+           AND entry_fact_revision_id = NEW.entry_fact_revision_id
+         ORDER BY decision_sequence DESC LIMIT 1 FOR UPDATE;
+    ELSE
+        SELECT decision_sequence INTO previous_sequence
+          FROM reviews_reviewdecision
+         WHERE module = 'TRAVEL_WARNING'
+           AND travel_warning_revision_id = NEW.travel_warning_revision_id
+         ORDER BY decision_sequence DESC LIMIT 1 FOR UPDATE;
+    END IF;
+    IF NEW.decision_sequence IS DISTINCT FROM
+       COALESCE(previous_sequence + 1, 1) THEN
+        RAISE EXCEPTION 'review decision sequence must be contiguous'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    RETURN NEW;
+END;
+$$;
+CREATE TRIGGER reviews_review_decision_guard
+BEFORE INSERT OR UPDATE OR DELETE ON reviews_reviewdecision
+FOR EACH ROW EXECUTE FUNCTION reviews_guard_review_decision();
+
+CREATE FUNCTION reviews_guard_publication_revision() RETURNS trigger
+LANGUAGE plpgsql AS $$
+DECLARE
+    review_module text;
+    review_entry_id uuid;
+    review_warning_id uuid;
+    review_decision text;
+    review_decided_at timestamptz;
+    latest_review_id uuid;
+    typed_country_id uuid;
+    typed_policy_id uuid;
+    typed_hash text;
+    typed_observed_at timestamptz;
+    parse_outcome text;
+    parse_name text;
+    parse_version text;
+    parse_contract text;
+    parse_schema text;
+    artifact_state text;
+    artifact_body_hash text;
+    artifact_byte_count bigint;
+    chain_source_id uuid;
+    source_code text;
+    chain_source_revision text;
+    source_module text;
+    source_owner text;
+    source_locator text;
+    source_state text;
+    source_enabled boolean;
+    source_secret_reference text;
+    source_connect_timeout integer;
+    source_read_timeout integer;
+    source_max_retries integer;
+    attempt_rights_id uuid;
+    attempt_result text;
+    attempt_http_status integer;
+    attempt_provider_code text;
+    attempt_response_hash text;
+    latest_rights_id uuid;
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
+    pointer_current_id uuid;
+    pointer_version bigint;
+    target_module text;
+    target_country_id uuid;
+    target_policy_id uuid;
+    target_entry_id uuid;
+    target_warning_id uuid;
+    target_generation bigint;
+    target_values_match boolean;
+BEGIN
+    IF TG_OP = 'UPDATE' OR TG_OP = 'DELETE' THEN
+        RAISE EXCEPTION 'publication revisions are immutable'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    SELECT module, entry_fact_revision_id, travel_warning_revision_id,
+           decision, decided_at
+      INTO review_module, review_entry_id, review_warning_id,
+           review_decision, review_decided_at
+      FROM reviews_reviewdecision
+     WHERE id = NEW.review_decision_id;
+    IF NOT FOUND OR review_module IS DISTINCT FROM NEW.module
+       OR review_decision IS DISTINCT FROM 'APPROVED'
+       OR review_decided_at > NEW.created_at
+       OR review_entry_id IS DISTINCT FROM NEW.entry_fact_revision_id
+       OR review_warning_id IS DISTINCT FROM NEW.travel_warning_revision_id THEN
+        RAISE EXCEPTION 'publication requires its exact approved review'
+            USING ERRCODE = 'check_violation';
+    END IF;
+
+    IF NEW.module = 'ENTRY' THEN
+        SELECT id INTO latest_review_id
+          FROM reviews_reviewdecision
+         WHERE module = 'ENTRY'
+           AND entry_fact_revision_id = NEW.entry_fact_revision_id
+         ORDER BY decision_sequence DESC LIMIT 1 FOR UPDATE;
+        SELECT typed.country_id, typed.passport_policy_id,
+               typed.typed_fingerprint_sha256, typed.first_observed_at,
+               parse_run.outcome, parse_run.parser_name,
+               parse_run.parser_version,
+               parse_run.parser_contract_fingerprint_sha256,
+               parse_run.observed_schema_fingerprint_sha256,
+               artifact.state, artifact.body_sha256, artifact.byte_count,
+               source.id,
+               source.code, source.revision, source.module, source.owner,
+               source.official_locator, source.state, source.enabled,
+               source.secret_reference_name, source.connect_timeout_seconds,
+               source.read_timeout_seconds, source.max_retries,
+               attempt.rights_decision_id,
+               attempt.result, attempt.http_status, attempt.provider_code,
+               attempt.response_sha256
+          INTO typed_country_id, typed_policy_id, typed_hash,
+               typed_observed_at, parse_outcome, parse_name, parse_version,
+               parse_contract, parse_schema, artifact_state,
+               artifact_body_hash, artifact_byte_count, chain_source_id,
+               source_code,
+               chain_source_revision,
+               source_module, source_owner, source_locator, source_state,
+               source_enabled, source_secret_reference,
+               source_connect_timeout, source_read_timeout,
+               source_max_retries, attempt_rights_id,
+               attempt_result, attempt_http_status, attempt_provider_code,
+               attempt_response_hash
+          FROM entry_requirements_entryfactrevision AS typed
+          JOIN sources_parserun AS parse_run
+            ON parse_run.id = typed.parse_run_id
+          JOIN sources_sourceartifact AS artifact
+            ON artifact.id = parse_run.artifact_id
+          JOIN sources_sourceconfiguration AS source
+            ON source.id = artifact.source_id
+          JOIN sources_fetchattempt AS attempt
+            ON attempt.id = artifact.first_successful_attempt_id
+         WHERE typed.id = NEW.entry_fact_revision_id
+         FOR UPDATE OF source;
+        SELECT current_publication_id, version
+          INTO pointer_current_id, pointer_version
+          FROM reviews_publishedentryfacts
+         WHERE id = '6c3602d7-a7ec-53e2-9628-59c09922f332'::uuid
+         FOR UPDATE;
+    ELSIF NEW.module = 'TRAVEL_WARNING' THEN
+        SELECT id INTO latest_review_id
+          FROM reviews_reviewdecision
+         WHERE module = 'TRAVEL_WARNING'
+           AND travel_warning_revision_id = NEW.travel_warning_revision_id
+         ORDER BY decision_sequence DESC LIMIT 1 FOR UPDATE;
+        SELECT typed.country_id, NULL::uuid,
+               typed.typed_fingerprint_sha256, typed.first_observed_at,
+               parse_run.outcome, parse_run.parser_name,
+               parse_run.parser_version,
+               parse_run.parser_contract_fingerprint_sha256,
+               parse_run.observed_schema_fingerprint_sha256,
+               artifact.state, artifact.body_sha256, artifact.byte_count,
+               source.id,
+               source.code, source.revision, source.module, source.owner,
+               source.official_locator, source.state, source.enabled,
+               source.secret_reference_name, source.connect_timeout_seconds,
+               source.read_timeout_seconds, source.max_retries,
+               attempt.rights_decision_id,
+               attempt.result, attempt.http_status, attempt.provider_code,
+               attempt.response_sha256
+          INTO typed_country_id, typed_policy_id, typed_hash,
+               typed_observed_at, parse_outcome, parse_name, parse_version,
+               parse_contract, parse_schema, artifact_state,
+               artifact_body_hash, artifact_byte_count, chain_source_id,
+               source_code,
+               chain_source_revision,
+               source_module, source_owner, source_locator, source_state,
+               source_enabled, source_secret_reference,
+               source_connect_timeout, source_read_timeout,
+               source_max_retries, attempt_rights_id,
+               attempt_result, attempt_http_status, attempt_provider_code,
+               attempt_response_hash
+          FROM travel_warnings_travelwarningrevision AS typed
+          JOIN sources_parserun AS parse_run
+            ON parse_run.id = typed.parse_run_id
+          JOIN sources_sourceartifact AS artifact
+            ON artifact.id = parse_run.artifact_id
+          JOIN sources_sourceconfiguration AS source
+            ON source.id = artifact.source_id
+          JOIN sources_fetchattempt AS attempt
+            ON attempt.id = artifact.first_successful_attempt_id
+         WHERE typed.id = NEW.travel_warning_revision_id
+         FOR UPDATE OF source;
+        SELECT current_publication_id, version
+          INTO pointer_current_id, pointer_version
+          FROM reviews_publishedtravelwarning
+         WHERE id = 'ec830f45-da57-5dfd-a289-f11a5acde9e2'::uuid
+         FOR UPDATE;
+    ELSE
+        RAISE EXCEPTION 'publication module is outside the allowlist'
+            USING ERRCODE = 'check_violation';
+    END IF;
+
+    SELECT id, decision_sequence, decision, access_mode, access_allowed,
+           automated_collection_allowed, typed_field_storage_allowed,
+           raw_body_storage_allowed, typed_republication_allowed,
+           raw_retention_seconds, typed_retention, evidence_retention,
+           field_scope_code, attribution_text,
+           contract_fingerprint_sha256, decided_by, decision_basis_code
+      INTO latest_rights_id, rights_sequence, rights_decision,
+           rights_access_mode, rights_access, rights_automation,
+           rights_typed_storage, rights_raw_storage, rights_republication,
+           rights_raw_seconds, rights_typed_retention,
+           rights_evidence_retention, rights_scope, rights_attribution,
+           rights_contract, rights_decided_by, rights_decision_basis
+      FROM sources_sourcerightsdecision AS rights
+     WHERE rights.source_id = chain_source_id
+       AND rights.source_revision = chain_source_revision
+     ORDER BY decision_sequence DESC LIMIT 1 FOR UPDATE;
+
+    IF latest_review_id IS DISTINCT FROM NEW.review_decision_id
+       OR typed_country_id IS DISTINCT FROM
+          '575fa8b9-14f9-526e-9464-ebd1dea76da9'::uuid
+       OR parse_outcome IS DISTINCT FROM 'VALIDATED'
+       OR artifact_state IS DISTINCT FROM 'REVIEW_REQUIRED'
+       OR artifact_byte_count < 1
+       OR source_state IS DISTINCT FROM 'ACTIVE' OR NOT source_enabled
+       OR source_connect_timeout IS DISTINCT FROM 5
+       OR source_read_timeout IS DISTINCT FROM 15
+       OR source_max_retries IS DISTINCT FROM 2
+       OR attempt_rights_id IS DISTINCT FROM latest_rights_id
+       OR attempt_result IS DISTINCT FROM 'SUCCEEDED'
+       OR attempt_http_status IS DISTINCT FROM 200
+       OR attempt_response_hash IS DISTINCT FROM artifact_body_hash
+       OR rights_sequence IS DISTINCT FROM 1
+       OR rights_decision IS DISTINCT FROM 'APPROVED'
+       OR NOT rights_access OR NOT rights_automation
+       OR NOT rights_typed_storage OR rights_raw_storage
+       OR NOT rights_republication OR rights_raw_seconds <> 0
+       OR rights_typed_retention IS DISTINCT FROM 'PRODUCT_HISTORY'
+       OR rights_evidence_retention IS DISTINCT FROM 'PRODUCT_HISTORY'
+       OR NEW.source_code_snapshot IS DISTINCT FROM source_code
+       OR NEW.source_revision IS DISTINCT FROM chain_source_revision
+       OR NEW.source_owner_snapshot IS DISTINCT FROM source_owner
+       OR NEW.source_locator_snapshot IS DISTINCT FROM source_locator
+       OR NEW.attribution_text_snapshot IS DISTINCT FROM rights_attribution
+       OR NEW.source_contract_fingerprint_sha256 IS DISTINCT FROM rights_contract
+       OR NEW.parser_name IS DISTINCT FROM parse_name
+       OR NEW.parser_version IS DISTINCT FROM parse_version
+       OR NEW.parser_contract_fingerprint_sha256 IS DISTINCT FROM parse_contract
+       OR NEW.schema_fingerprint_sha256 IS DISTINCT FROM parse_schema
+       OR NEW.typed_fingerprint_sha256 IS DISTINCT FROM typed_hash
+       OR NEW.source_first_observed_at IS DISTINCT FROM typed_observed_at
+       OR pointer_version IS NULL
+       OR NEW.generation IS DISTINCT FROM pointer_version + 1
+       OR NEW.supersedes_id IS DISTINCT FROM pointer_current_id THEN
+        RAISE EXCEPTION 'publication evidence or generation is not canonical'
+            USING ERRCODE = 'check_violation';
+    END IF;
+
+    IF NEW.module = 'ENTRY' AND (
+        typed_policy_id IS DISTINCT FROM
+          'f461f7a7-18f7-5b0d-9831-bfbd47b695e5'::uuid
+        OR source_code IS DISTINCT FROM 'MOFA_ENTRY_CSV'
+        OR chain_source_revision IS DISTINCT FROM 'rights-v1'
+        OR source_module IS DISTINCT FROM 'ENTRY'
+        OR source_owner IS DISTINCT FROM '대한민국 외교부 정보화담당관실'
+        OR source_locator IS DISTINCT FROM
+          'https://www.data.go.kr/cmm/cmm/fileDownload.do?atchFileId=FILE_000000003090472&fileDetailSn=1&insertDataPrcus=N'
+        OR source_secret_reference IS DISTINCT FROM ''
+        OR artifact_byte_count > 262144
+        OR rights_decided_by IS DISTINCT FROM 'PROJECT_OWNER_REQUEST'
+        OR rights_decision_basis IS DISTINCT FROM 'USER_DIRECTIVE_20260830'
+        OR parse_name IS DISTINCT FROM 'MOFA_ENTRY_CSV'
+        OR parse_version IS DISTINCT FROM 'V1'
+        OR parse_contract IS DISTINCT FROM
+          '622a399317ab730f9e9780f51b3ac837073cd99939f07f248323636e97676021'
+        OR parse_schema IS DISTINCT FROM
+          '46bb5428a08a94810e6d381541cb7a3aed3f2f5110a529c2ae32e101cd5e1e0b'
+        OR attempt_provider_code IS DISTINCT FROM ''
+        OR rights_access_mode IS DISTINCT FROM 'ANONYMOUS_PUBLIC'
+        OR rights_scope IS DISTINCT FROM 'JP_ORDINARY_TEXT_V1'
+        OR rights_attribution IS DISTINCT FROM '외교부|공공데이터포털'
+        OR rights_contract IS DISTINCT FROM
+          '622a399317ab730f9e9780f51b3ac837073cd99939f07f248323636e97676021'
+    ) THEN
+        RAISE EXCEPTION 'entry publication is outside the approved contract'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    IF NEW.module = 'TRAVEL_WARNING' AND (
+        source_code IS DISTINCT FROM 'MOFA_TRAVEL_ALARM_API_JP'
+        OR chain_source_revision IS DISTINCT FROM 'rights-v1'
+        OR source_module IS DISTINCT FROM 'TRAVEL_WARNING'
+        OR source_owner IS DISTINCT FROM '대한민국 외교부'
+        OR source_locator IS DISTINCT FROM
+          'https://apis.data.go.kr/1262000/TravelAlarmService2/getTravelAlarmList2'
+        OR source_secret_reference IS DISTINCT FROM
+          'MOFA_TRAVEL_ALARM_SERVICE_KEY'
+        OR artifact_byte_count > 4096
+        OR rights_decided_by IS DISTINCT FROM 'PROJECT_OWNER_REQUEST'
+        OR rights_decision_basis IS DISTINCT FROM 'USER_FOLLOWUP_20260830'
+        OR parse_name IS DISTINCT FROM 'MOFA_TRAVEL_ALARM_JSON'
+        OR parse_version IS DISTINCT FROM 'V1'
+        OR parse_contract IS DISTINCT FROM
+          'c43c5f7e7e6f37f13ba7e5d9e2448b29b24d5524012b41fc5e851abb907c55f6'
+        OR parse_schema IS DISTINCT FROM
+          '64a98b89c39d6c04e0daf2bbac9353de053320b835a62076a37f620cc07c4f0b'
+        OR attempt_provider_code IS DISTINCT FROM 'MOFA_SUCCESS_0'
+        OR rights_access_mode IS DISTINCT FROM 'CREDENTIAL_REFERENCE'
+        OR rights_scope IS DISTINCT FROM 'JP_WARNING_V1'
+        OR rights_attribution IS DISTINCT FROM '외교부|공공데이터포털'
+        OR rights_contract IS DISTINCT FROM
+          'c43c5f7e7e6f37f13ba7e5d9e2448b29b24d5524012b41fc5e851abb907c55f6'
+    ) THEN
+        RAISE EXCEPTION 'warning publication is outside the approved contract'
+            USING ERRCODE = 'check_violation';
+    END IF;
+
+    IF NEW.rollback_target_id IS NOT NULL THEN
+        SELECT module, scope_country_id, scope_passport_policy_id,
+               entry_fact_revision_id, travel_warning_revision_id, generation,
+               (source_code_snapshot, source_revision, source_owner_snapshot,
+                source_locator_snapshot, attribution_text_snapshot,
+                source_contract_fingerprint_sha256,
+                parser_name, parser_version,
+                parser_contract_fingerprint_sha256,
+                schema_fingerprint_sha256, typed_fingerprint_sha256,
+                source_first_observed_at)
+               IS NOT DISTINCT FROM
+               (NEW.source_code_snapshot, NEW.source_revision,
+                NEW.source_owner_snapshot, NEW.source_locator_snapshot,
+                NEW.attribution_text_snapshot,
+                NEW.source_contract_fingerprint_sha256,
+                NEW.parser_name,
+                NEW.parser_version,
+                NEW.parser_contract_fingerprint_sha256,
+                NEW.schema_fingerprint_sha256,
+                NEW.typed_fingerprint_sha256,
+                NEW.source_first_observed_at)
+          INTO target_module, target_country_id, target_policy_id,
+               target_entry_id, target_warning_id, target_generation,
+               target_values_match
+          FROM reviews_publicationrevision
+         WHERE id = NEW.rollback_target_id;
+        IF NOT FOUND OR NEW.supersedes_id IS NULL
+           OR NEW.rollback_target_id = NEW.supersedes_id
+           OR target_module IS DISTINCT FROM NEW.module
+           OR target_country_id IS DISTINCT FROM NEW.scope_country_id
+           OR target_policy_id IS DISTINCT FROM NEW.scope_passport_policy_id
+           OR target_entry_id IS DISTINCT FROM NEW.entry_fact_revision_id
+           OR target_warning_id IS DISTINCT FROM NEW.travel_warning_revision_id
+           OR target_generation >= NEW.generation
+           OR NOT target_values_match THEN
+            RAISE EXCEPTION 'rollback target is not an exact historical publication'
+                USING ERRCODE = 'check_violation';
+        END IF;
+    END IF;
+    RETURN NEW;
+END;
+$$;
+CREATE TRIGGER reviews_publication_revision_guard
+BEFORE INSERT OR UPDATE OR DELETE ON reviews_publicationrevision
+FOR EACH ROW EXECUTE FUNCTION reviews_guard_publication_revision();
+
+CREATE FUNCTION reviews_guard_audit_event() RETURNS trigger
+LANGUAGE plpgsql AS $$
+DECLARE
+    actor_username text;
+    review_actor_id integer;
+    review_actor_principal text;
+    review_module text;
+    review_decision text;
+    review_entry_id uuid;
+    review_warning_id uuid;
+    review_decided_at timestamptz;
+    publication_module text;
+    publication_entry_id uuid;
+    publication_warning_id uuid;
+    publication_review_id uuid;
+    publication_supersedes_id uuid;
+    publication_rollback_target_id uuid;
+    publication_time timestamptz;
+    canonical text;
+BEGIN
+    IF TG_OP = 'UPDATE' OR TG_OP = 'DELETE' THEN
+        RAISE EXCEPTION 'audit events are append-only'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    IF NEW.actor_id IS NULL THEN
+        IF NEW.actor_principal <> 'UNAUTHENTICATED' THEN
+            RAISE EXCEPTION 'anonymous audit principal is not canonical'
+                USING ERRCODE = 'check_violation';
+        END IF;
+    ELSE
+        SELECT username INTO actor_username FROM auth_user
+         WHERE id = NEW.actor_id;
+        IF NOT FOUND OR actor_username IS DISTINCT FROM NEW.actor_principal THEN
+            RAISE EXCEPTION 'audit actor principal is not canonical'
+                USING ERRCODE = 'check_violation';
+        END IF;
+    END IF;
+
+    IF NEW.outcome = 'SUCCEEDED' THEN
+        SELECT actor_id, actor_principal, module, decision,
+               entry_fact_revision_id, travel_warning_revision_id, decided_at
+          INTO review_actor_id, review_actor_principal, review_module,
+               review_decision, review_entry_id, review_warning_id,
+               review_decided_at
+          FROM reviews_reviewdecision WHERE id = NEW.review_decision_id;
+        IF NOT FOUND OR review_actor_id IS DISTINCT FROM NEW.actor_id
+           OR review_actor_principal IS DISTINCT FROM NEW.actor_principal
+           OR review_module IS DISTINCT FROM NEW.module
+           OR review_entry_id IS DISTINCT FROM NEW.entry_fact_revision_id
+           OR review_warning_id IS DISTINCT FROM NEW.travel_warning_revision_id
+           OR review_decided_at > NEW.occurred_at THEN
+            RAISE EXCEPTION 'successful audit review chain is inconsistent'
+                USING ERRCODE = 'check_violation';
+        END IF;
+        IF NEW.action = 'REVIEW_REJECT' THEN
+            IF review_decision <> 'REJECTED' THEN
+                RAISE EXCEPTION 'review rejection audit requires a rejection'
+                    USING ERRCODE = 'check_violation';
+            END IF;
+        ELSE
+            SELECT module, entry_fact_revision_id,
+                   travel_warning_revision_id, review_decision_id,
+                   supersedes_id, rollback_target_id, published_at
+              INTO publication_module, publication_entry_id,
+                   publication_warning_id, publication_review_id,
+                   publication_supersedes_id,
+                   publication_rollback_target_id, publication_time
+              FROM reviews_publicationrevision
+             WHERE id = NEW.publication_revision_id;
+            IF NOT FOUND OR review_decision <> 'APPROVED'
+               OR publication_module IS DISTINCT FROM NEW.module
+               OR publication_entry_id IS DISTINCT FROM NEW.entry_fact_revision_id
+               OR publication_warning_id IS DISTINCT FROM
+                  NEW.travel_warning_revision_id
+               OR publication_review_id IS DISTINCT FROM NEW.review_decision_id
+               OR publication_supersedes_id IS DISTINCT FROM
+                  NEW.prior_publication_revision_id
+               OR publication_rollback_target_id IS DISTINCT FROM
+                  NEW.rollback_target_publication_revision_id
+               OR publication_time > NEW.occurred_at THEN
+                RAISE EXCEPTION 'successful publication audit chain is inconsistent'
+                    USING ERRCODE = 'check_violation';
+            END IF;
+        END IF;
+    END IF;
+
+    canonical := concat_ws(E'\n', NEW.action, NEW.module,
+        COALESCE(NEW.entry_fact_revision_id::text, ''),
+        COALESCE(NEW.travel_warning_revision_id::text, ''),
+        COALESCE(NEW.review_decision_id::text, ''),
+        COALESCE(NEW.publication_revision_id::text, ''),
+        COALESCE(NEW.prior_publication_revision_id::text, ''),
+        COALESCE(NEW.rollback_target_publication_revision_id::text, ''),
+        NEW.outcome, NEW.failure_code);
+    IF NEW.input_identity_sha256 IS DISTINCT FROM
+       encode(sha256(convert_to(canonical, 'UTF8')), 'hex') THEN
+        RAISE EXCEPTION 'audit input identity is not canonical'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    RETURN NEW;
+END;
+$$;
+CREATE TRIGGER reviews_audit_event_guard
+BEFORE INSERT OR UPDATE OR DELETE ON reviews_auditevent
+FOR EACH ROW EXECUTE FUNCTION reviews_guard_audit_event();
+
+CREATE FUNCTION reviews_guard_entry_pointer() RETURNS trigger
+LANGUAGE plpgsql AS $$
+DECLARE
+    publication_module text;
+    publication_country_id uuid;
+    publication_policy_id uuid;
+    publication_generation bigint;
+    publication_supersedes_id uuid;
+    publication_rollback_target_id uuid;
+    publication_review_reason text;
+    publication_audit_action text;
+BEGIN
+    IF TG_OP = 'DELETE' THEN
+        RAISE EXCEPTION 'entry publication pointer cannot be deleted'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    IF TG_OP = 'INSERT' THEN
+        IF NEW.id <> '6c3602d7-a7ec-53e2-9628-59c09922f332'::uuid
+           OR NEW.current_publication_id IS NOT NULL OR NEW.version <> 0 THEN
+            RAISE EXCEPTION 'entry pointer insert is not the fixed seed'
+                USING ERRCODE = 'check_violation';
+        END IF;
+        RETURN NEW;
+    END IF;
+    IF (NEW.id, NEW.country_id, NEW.passport_policy_id)
+       IS DISTINCT FROM (OLD.id, OLD.country_id, OLD.passport_policy_id)
+       OR NEW.version <> OLD.version + 1
+       OR NEW.updated_at < OLD.updated_at
+       OR NEW.current_publication_id IS NULL THEN
+        RAISE EXCEPTION 'entry pointer CAS shape is invalid'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    SELECT publication.module, publication.scope_country_id,
+           publication.scope_passport_policy_id, publication.generation,
+           publication.supersedes_id, publication.rollback_target_id,
+           review.reason_code, audit.action
+      INTO publication_module, publication_country_id,
+           publication_policy_id, publication_generation,
+           publication_supersedes_id, publication_rollback_target_id,
+           publication_review_reason, publication_audit_action
+      FROM reviews_publicationrevision AS publication
+      JOIN reviews_reviewdecision AS review
+        ON review.id = publication.review_decision_id
+      JOIN reviews_auditevent AS audit
+        ON audit.publication_revision_id = publication.id
+       AND audit.outcome = 'SUCCEEDED'
+     WHERE publication.id = NEW.current_publication_id;
+    IF NOT FOUND OR publication_module <> 'ENTRY'
+       OR publication_country_id IS DISTINCT FROM NEW.country_id
+       OR publication_policy_id IS DISTINCT FROM NEW.passport_policy_id
+       OR publication_generation <> NEW.version
+       OR publication_supersedes_id IS DISTINCT FROM OLD.current_publication_id
+       OR NOT (
+            (publication_audit_action = 'PUBLISH'
+             AND publication_rollback_target_id IS NULL
+             AND publication_review_reason = 'APPROVE_ENTRY')
+            OR
+            (publication_audit_action = 'ROLLBACK'
+             AND publication_rollback_target_id IS NOT NULL
+             AND publication_review_reason = 'ROLLBACK_ENTRY')
+       )
+       OR NOT EXISTS (
+            SELECT 1 FROM reviews_auditevent
+             WHERE publication_revision_id = NEW.current_publication_id
+               AND outcome = 'SUCCEEDED'
+               AND action IN ('PUBLISH', 'ROLLBACK')
+               AND prior_publication_revision_id
+                   IS NOT DISTINCT FROM OLD.current_publication_id
+       ) THEN
+        RAISE EXCEPTION 'entry pointer target is not an audited publication'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    RETURN NEW;
+END;
+$$;
+CREATE TRIGGER reviews_entry_pointer_guard
+BEFORE INSERT OR UPDATE OR DELETE ON reviews_publishedentryfacts
+FOR EACH ROW EXECUTE FUNCTION reviews_guard_entry_pointer();
+
+CREATE FUNCTION reviews_guard_warning_pointer() RETURNS trigger
+LANGUAGE plpgsql AS $$
+DECLARE
+    publication_module text;
+    publication_country_id uuid;
+    publication_generation bigint;
+    publication_supersedes_id uuid;
+    publication_rollback_target_id uuid;
+    publication_review_reason text;
+    publication_audit_action text;
+BEGIN
+    IF TG_OP = 'DELETE' THEN
+        RAISE EXCEPTION 'warning publication pointer cannot be deleted'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    IF TG_OP = 'INSERT' THEN
+        IF NEW.id <> 'ec830f45-da57-5dfd-a289-f11a5acde9e2'::uuid
+           OR NEW.current_publication_id IS NOT NULL OR NEW.version <> 0 THEN
+            RAISE EXCEPTION 'warning pointer insert is not the fixed seed'
+                USING ERRCODE = 'check_violation';
+        END IF;
+        RETURN NEW;
+    END IF;
+    IF (NEW.id, NEW.country_id) IS DISTINCT FROM (OLD.id, OLD.country_id)
+       OR NEW.version <> OLD.version + 1
+       OR NEW.updated_at < OLD.updated_at
+       OR NEW.current_publication_id IS NULL THEN
+        RAISE EXCEPTION 'warning pointer CAS shape is invalid'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    SELECT publication.module, publication.scope_country_id,
+           publication.generation, publication.supersedes_id,
+           publication.rollback_target_id, review.reason_code, audit.action
+      INTO publication_module, publication_country_id,
+           publication_generation, publication_supersedes_id,
+           publication_rollback_target_id, publication_review_reason,
+           publication_audit_action
+      FROM reviews_publicationrevision AS publication
+      JOIN reviews_reviewdecision AS review
+        ON review.id = publication.review_decision_id
+      JOIN reviews_auditevent AS audit
+        ON audit.publication_revision_id = publication.id
+       AND audit.outcome = 'SUCCEEDED'
+     WHERE publication.id = NEW.current_publication_id;
+    IF NOT FOUND OR publication_module <> 'TRAVEL_WARNING'
+       OR publication_country_id IS DISTINCT FROM NEW.country_id
+       OR publication_generation <> NEW.version
+       OR publication_supersedes_id IS DISTINCT FROM OLD.current_publication_id
+       OR NOT (
+            (publication_audit_action = 'PUBLISH'
+             AND publication_rollback_target_id IS NULL
+             AND publication_review_reason = 'APPROVE_WARNING')
+            OR
+            (publication_audit_action = 'ROLLBACK'
+             AND publication_rollback_target_id IS NOT NULL
+             AND publication_review_reason = 'ROLLBACK_WARNING')
+       )
+       OR NOT EXISTS (
+            SELECT 1 FROM reviews_auditevent
+             WHERE publication_revision_id = NEW.current_publication_id
+               AND outcome = 'SUCCEEDED'
+               AND action IN ('PUBLISH', 'ROLLBACK')
+               AND prior_publication_revision_id
+                   IS NOT DISTINCT FROM OLD.current_publication_id
+       ) THEN
+        RAISE EXCEPTION 'warning pointer target is not an audited publication'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    RETURN NEW;
+END;
+$$;
+CREATE TRIGGER reviews_warning_pointer_guard
+BEFORE INSERT OR UPDATE OR DELETE ON reviews_publishedtravelwarning
+FOR EACH ROW EXECUTE FUNCTION reviews_guard_warning_pointer();
+
+CREATE FUNCTION reviews_prelock_review_insert() RETURNS trigger
+LANGUAGE plpgsql AS $$
+DECLARE
+    pointer_publication_id uuid;
+    current_entry_id uuid;
+    current_warning_id uuid;
+    chain_source_id uuid;
+    chain_source_revision text;
+BEGIN
+    IF NEW.module = 'ENTRY' THEN
+        PERFORM pg_advisory_xact_lock(1414678614, 2007001);
+        SELECT current_publication_id INTO pointer_publication_id
+          FROM reviews_publishedentryfacts
+         WHERE id = '6c3602d7-a7ec-53e2-9628-59c09922f332'::uuid
+         FOR UPDATE;
+        IF NOT FOUND THEN
+            RAISE EXCEPTION 'entry publication pointer is unavailable'
+                USING ERRCODE = 'check_violation';
+        END IF;
+        SELECT source.id, source.revision
+          INTO chain_source_id, chain_source_revision
+          FROM entry_requirements_entryfactrevision AS typed
+          JOIN sources_parserun AS parse_run ON parse_run.id = typed.parse_run_id
+          JOIN sources_sourceartifact AS artifact
+            ON artifact.id = parse_run.artifact_id
+          JOIN sources_sourceconfiguration AS source
+            ON source.id = artifact.source_id
+         WHERE typed.id = NEW.entry_fact_revision_id
+         FOR UPDATE OF typed, source;
+        IF NOT FOUND THEN
+            RAISE EXCEPTION 'entry review subject is unavailable'
+                USING ERRCODE = 'check_violation';
+        END IF;
+        IF NEW.decision = 'REJECTED' AND pointer_publication_id IS NOT NULL THEN
+            SELECT entry_fact_revision_id INTO current_entry_id
+              FROM reviews_publicationrevision
+             WHERE id = pointer_publication_id;
+            IF current_entry_id IS NOT DISTINCT FROM NEW.entry_fact_revision_id THEN
+                RAISE EXCEPTION 'the current entry publication cannot be rejected'
+                    USING ERRCODE = 'check_violation';
+            END IF;
+        END IF;
+    ELSIF NEW.module = 'TRAVEL_WARNING' THEN
+        PERFORM pg_advisory_xact_lock(1414678614, 2007002);
+        SELECT current_publication_id INTO pointer_publication_id
+          FROM reviews_publishedtravelwarning
+         WHERE id = 'ec830f45-da57-5dfd-a289-f11a5acde9e2'::uuid
+         FOR UPDATE;
+        IF NOT FOUND THEN
+            RAISE EXCEPTION 'warning publication pointer is unavailable'
+                USING ERRCODE = 'check_violation';
+        END IF;
+        SELECT source.id, source.revision
+          INTO chain_source_id, chain_source_revision
+          FROM travel_warnings_travelwarningrevision AS typed
+          JOIN sources_parserun AS parse_run ON parse_run.id = typed.parse_run_id
+          JOIN sources_sourceartifact AS artifact
+            ON artifact.id = parse_run.artifact_id
+          JOIN sources_sourceconfiguration AS source
+            ON source.id = artifact.source_id
+         WHERE typed.id = NEW.travel_warning_revision_id
+         FOR UPDATE OF typed, source;
+        IF NOT FOUND THEN
+            RAISE EXCEPTION 'warning review subject is unavailable'
+                USING ERRCODE = 'check_violation';
+        END IF;
+        IF NEW.decision = 'REJECTED' AND pointer_publication_id IS NOT NULL THEN
+            SELECT travel_warning_revision_id INTO current_warning_id
+              FROM reviews_publicationrevision
+             WHERE id = pointer_publication_id;
+            IF current_warning_id IS NOT DISTINCT FROM
+               NEW.travel_warning_revision_id THEN
+                RAISE EXCEPTION 'the current warning publication cannot be rejected'
+                    USING ERRCODE = 'check_violation';
+            END IF;
+        END IF;
+    ELSE
+        RAISE EXCEPTION 'review module is outside the publication allowlist'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    PERFORM 1 FROM sources_sourcerightsdecision
+     WHERE source_id = chain_source_id
+       AND source_revision = chain_source_revision
+     ORDER BY decision_sequence DESC LIMIT 1 FOR UPDATE;
+    PERFORM 1 FROM auth_user WHERE id = NEW.actor_id FOR UPDATE;
+    RETURN NEW;
+END;
+$$;
+CREATE TRIGGER reviews_00_review_insert_prelock
+BEFORE INSERT ON reviews_reviewdecision
+FOR EACH ROW EXECUTE FUNCTION reviews_prelock_review_insert();
+
+CREATE FUNCTION reviews_prelock_publication_insert() RETURNS trigger
+LANGUAGE plpgsql AS $$
+DECLARE
+    chain_source_id uuid;
+    chain_source_revision text;
+    review_reason text;
+    review_actor_id integer;
+BEGIN
+    IF NEW.module = 'ENTRY' THEN
+        PERFORM pg_advisory_xact_lock(1414678614, 2007001);
+        PERFORM 1 FROM reviews_publishedentryfacts
+         WHERE id = '6c3602d7-a7ec-53e2-9628-59c09922f332'::uuid
+         FOR UPDATE;
+        IF NOT FOUND THEN
+            RAISE EXCEPTION 'entry publication pointer is unavailable'
+                USING ERRCODE = 'check_violation';
+        END IF;
+        SELECT source.id, source.revision
+          INTO chain_source_id, chain_source_revision
+          FROM entry_requirements_entryfactrevision AS typed
+          JOIN sources_parserun AS parse_run ON parse_run.id = typed.parse_run_id
+          JOIN sources_sourceartifact AS artifact
+            ON artifact.id = parse_run.artifact_id
+          JOIN sources_sourceconfiguration AS source
+            ON source.id = artifact.source_id
+         WHERE typed.id = NEW.entry_fact_revision_id
+         FOR UPDATE OF typed, source;
+        IF NEW.rollback_target_id IS NULL AND EXISTS (
+            SELECT 1 FROM reviews_publicationrevision
+             WHERE module = 'ENTRY'
+               AND entry_fact_revision_id = NEW.entry_fact_revision_id
+        ) THEN
+            RAISE EXCEPTION 'entry history can only be restored by rollback'
+                USING ERRCODE = 'check_violation';
+        END IF;
+    ELSIF NEW.module = 'TRAVEL_WARNING' THEN
+        PERFORM pg_advisory_xact_lock(1414678614, 2007002);
+        PERFORM 1 FROM reviews_publishedtravelwarning
+         WHERE id = 'ec830f45-da57-5dfd-a289-f11a5acde9e2'::uuid
+         FOR UPDATE;
+        IF NOT FOUND THEN
+            RAISE EXCEPTION 'warning publication pointer is unavailable'
+                USING ERRCODE = 'check_violation';
+        END IF;
+        SELECT source.id, source.revision
+          INTO chain_source_id, chain_source_revision
+          FROM travel_warnings_travelwarningrevision AS typed
+          JOIN sources_parserun AS parse_run ON parse_run.id = typed.parse_run_id
+          JOIN sources_sourceartifact AS artifact
+            ON artifact.id = parse_run.artifact_id
+          JOIN sources_sourceconfiguration AS source
+            ON source.id = artifact.source_id
+         WHERE typed.id = NEW.travel_warning_revision_id
+         FOR UPDATE OF typed, source;
+        IF NEW.rollback_target_id IS NULL AND EXISTS (
+            SELECT 1 FROM reviews_publicationrevision
+             WHERE module = 'TRAVEL_WARNING'
+               AND travel_warning_revision_id = NEW.travel_warning_revision_id
+        ) THEN
+            RAISE EXCEPTION 'warning history can only be restored by rollback'
+                USING ERRCODE = 'check_violation';
+        END IF;
+    ELSE
+        RAISE EXCEPTION 'publication module is outside the allowlist'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    PERFORM 1 FROM sources_sourcerightsdecision
+     WHERE source_id = chain_source_id
+       AND source_revision = chain_source_revision
+     ORDER BY decision_sequence DESC LIMIT 1 FOR UPDATE;
+    SELECT reason_code, actor_id INTO review_reason, review_actor_id
+      FROM reviews_reviewdecision
+     WHERE id = NEW.review_decision_id FOR UPDATE;
+    IF NOT FOUND OR NOT (
+        (NEW.module = 'ENTRY' AND NEW.rollback_target_id IS NULL
+         AND review_reason = 'APPROVE_ENTRY')
+        OR (NEW.module = 'ENTRY' AND NEW.rollback_target_id IS NOT NULL
+            AND review_reason = 'ROLLBACK_ENTRY')
+        OR (NEW.module = 'TRAVEL_WARNING' AND NEW.rollback_target_id IS NULL
+            AND review_reason = 'APPROVE_WARNING')
+        OR (NEW.module = 'TRAVEL_WARNING'
+            AND NEW.rollback_target_id IS NOT NULL
+            AND review_reason = 'ROLLBACK_WARNING')
+    ) THEN
+        RAISE EXCEPTION 'publication action is not bound to review permission'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    PERFORM 1 FROM auth_user WHERE id = review_actor_id FOR UPDATE;
+    RETURN NEW;
+END;
+$$;
+CREATE TRIGGER reviews_00_publication_insert_prelock
+BEFORE INSERT ON reviews_publicationrevision
+FOR EACH ROW EXECUTE FUNCTION reviews_prelock_publication_insert();
+
+CREATE FUNCTION reviews_prelock_audit_insert() RETURNS trigger
+LANGUAGE plpgsql AS $$
+DECLARE
+    chain_source_id uuid;
+    chain_source_revision text;
+    review_reason text;
+    review_actor_id integer;
+    review_correlation_id uuid;
+    publication_rollback_target_id uuid;
+BEGIN
+    IF NEW.module = 'ENTRY' THEN
+        PERFORM pg_advisory_xact_lock(1414678614, 2007001);
+        PERFORM 1 FROM reviews_publishedentryfacts
+         WHERE id = '6c3602d7-a7ec-53e2-9628-59c09922f332'::uuid
+         FOR UPDATE;
+        IF NEW.entry_fact_revision_id IS NOT NULL THEN
+            SELECT source.id, source.revision
+              INTO chain_source_id, chain_source_revision
+              FROM entry_requirements_entryfactrevision AS typed
+              JOIN sources_parserun AS parse_run
+                ON parse_run.id = typed.parse_run_id
+              JOIN sources_sourceartifact AS artifact
+                ON artifact.id = parse_run.artifact_id
+              JOIN sources_sourceconfiguration AS source
+                ON source.id = artifact.source_id
+             WHERE typed.id = NEW.entry_fact_revision_id
+             FOR UPDATE OF typed, source;
+        END IF;
+    ELSIF NEW.module = 'TRAVEL_WARNING' THEN
+        PERFORM pg_advisory_xact_lock(1414678614, 2007002);
+        PERFORM 1 FROM reviews_publishedtravelwarning
+         WHERE id = 'ec830f45-da57-5dfd-a289-f11a5acde9e2'::uuid
+         FOR UPDATE;
+        IF NEW.travel_warning_revision_id IS NOT NULL THEN
+            SELECT source.id, source.revision
+              INTO chain_source_id, chain_source_revision
+              FROM travel_warnings_travelwarningrevision AS typed
+              JOIN sources_parserun AS parse_run
+                ON parse_run.id = typed.parse_run_id
+              JOIN sources_sourceartifact AS artifact
+                ON artifact.id = parse_run.artifact_id
+              JOIN sources_sourceconfiguration AS source
+                ON source.id = artifact.source_id
+             WHERE typed.id = NEW.travel_warning_revision_id
+             FOR UPDATE OF typed, source;
+        END IF;
+    ELSE
+        RAISE EXCEPTION 'audit module is outside the publication allowlist'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    IF chain_source_id IS NOT NULL THEN
+        PERFORM 1 FROM sources_sourcerightsdecision
+         WHERE source_id = chain_source_id
+           AND source_revision = chain_source_revision
+         ORDER BY decision_sequence DESC LIMIT 1 FOR UPDATE;
+    END IF;
+    IF NEW.outcome = 'SUCCEEDED' THEN
+        SELECT reason_code, actor_id, correlation_id
+          INTO review_reason, review_actor_id, review_correlation_id
+          FROM reviews_reviewdecision
+         WHERE id = NEW.review_decision_id FOR UPDATE;
+        IF NEW.publication_revision_id IS NOT NULL THEN
+            SELECT rollback_target_id INTO publication_rollback_target_id
+              FROM reviews_publicationrevision
+             WHERE id = NEW.publication_revision_id FOR UPDATE;
+        END IF;
+        IF NEW.correlation_id IS DISTINCT FROM review_correlation_id
+           OR NOT (
+                (NEW.module = 'ENTRY' AND NEW.action = 'REVIEW_REJECT'
+                 AND review_reason = 'REJECT_ENTRY'
+                 AND NEW.publication_revision_id IS NULL)
+                OR (NEW.module = 'TRAVEL_WARNING'
+                    AND NEW.action = 'REVIEW_REJECT'
+                    AND review_reason = 'REJECT_WARNING'
+                    AND NEW.publication_revision_id IS NULL)
+                OR (NEW.module = 'ENTRY' AND NEW.action = 'PUBLISH'
+                    AND review_reason = 'APPROVE_ENTRY'
+                    AND publication_rollback_target_id IS NULL)
+                OR (NEW.module = 'TRAVEL_WARNING'
+                    AND NEW.action = 'PUBLISH'
+                    AND review_reason = 'APPROVE_WARNING'
+                    AND publication_rollback_target_id IS NULL)
+                OR (NEW.module = 'ENTRY' AND NEW.action = 'ROLLBACK'
+                    AND review_reason = 'ROLLBACK_ENTRY'
+                    AND publication_rollback_target_id IS NOT NULL)
+                OR (NEW.module = 'TRAVEL_WARNING'
+                    AND NEW.action = 'ROLLBACK'
+                    AND review_reason = 'ROLLBACK_WARNING'
+                    AND publication_rollback_target_id IS NOT NULL)
+           ) THEN
+            RAISE EXCEPTION 'audit action is not bound to review permission'
+                USING ERRCODE = 'check_violation';
+        END IF;
+    ELSE
+        review_actor_id := NEW.actor_id;
+    END IF;
+    IF review_actor_id IS NOT NULL THEN
+        PERFORM 1 FROM auth_user WHERE id = review_actor_id FOR UPDATE;
+    END IF;
+    RETURN NEW;
+END;
+$$;
+CREATE TRIGGER reviews_00_audit_insert_prelock
+BEFORE INSERT ON reviews_auditevent
+FOR EACH ROW EXECUTE FUNCTION reviews_prelock_audit_insert();
+
+CREATE FUNCTION reviews_prelock_pointer_statement() RETURNS trigger
+LANGUAGE plpgsql AS $$
+BEGIN
+    IF TG_TABLE_NAME = 'reviews_publishedentryfacts' THEN
+        PERFORM pg_advisory_xact_lock(1414678614, 2007001);
+    ELSIF TG_TABLE_NAME = 'reviews_publishedtravelwarning' THEN
+        PERFORM pg_advisory_xact_lock(1414678614, 2007002);
+    ELSE
+        RAISE EXCEPTION 'publication pointer table is outside the allowlist'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    RETURN NULL;
+END;
+$$;
+CREATE TRIGGER reviews_00_entry_pointer_statement_lock
+BEFORE INSERT OR UPDATE OR DELETE ON reviews_publishedentryfacts
+FOR EACH STATEMENT EXECUTE FUNCTION reviews_prelock_pointer_statement();
+CREATE TRIGGER reviews_00_warning_pointer_statement_lock
+BEFORE INSERT OR UPDATE OR DELETE ON reviews_publishedtravelwarning
+FOR EACH STATEMENT EXECUTE FUNCTION reviews_prelock_pointer_statement();
+
+CREATE FUNCTION reviews_enforce_deferred_closure() RETURNS trigger
+LANGUAGE plpgsql AS $$
+DECLARE
+    closure_review_id uuid;
+    review_module text;
+    review_decision text;
+    review_reason text;
+    review_actor_id integer;
+    review_actor_principal text;
+    review_entry_id uuid;
+    review_warning_id uuid;
+    review_correlation_id uuid;
+    publication_count integer;
+    publication_id uuid;
+    publication_generation bigint;
+    publication_supersedes_id uuid;
+    publication_rollback_target_id uuid;
+    publication_country_id uuid;
+    publication_policy_id uuid;
+    prior_module text;
+    prior_generation bigint;
+    prior_country_id uuid;
+    prior_policy_id uuid;
+    audit_count integer;
+    audit_action text;
+    audit_actor_id integer;
+    audit_actor_principal text;
+    audit_module text;
+    audit_entry_id uuid;
+    audit_warning_id uuid;
+    audit_publication_id uuid;
+    audit_prior_id uuid;
+    audit_rollback_target_id uuid;
+    audit_correlation_id uuid;
+    pointer_publication_id uuid;
+    pointer_entry_id uuid;
+    pointer_warning_id uuid;
+    latest_subject_review_id uuid;
+    duplicate_subject_count integer;
+    rollback_is_ancestor boolean;
+BEGIN
+    IF TG_TABLE_NAME = 'reviews_reviewdecision' THEN
+        closure_review_id := NEW.id;
+    ELSIF TG_TABLE_NAME = 'reviews_publicationrevision' THEN
+        closure_review_id := NEW.review_decision_id;
+    ELSIF TG_TABLE_NAME = 'reviews_auditevent' THEN
+        IF NEW.outcome <> 'SUCCEEDED' THEN
+            RETURN NULL;
+        END IF;
+        closure_review_id := NEW.review_decision_id;
+    ELSIF TG_TABLE_NAME = 'reviews_publishedentryfacts' THEN
+        SELECT current_publication_id INTO pointer_publication_id
+          FROM reviews_publishedentryfacts
+         WHERE id = '6c3602d7-a7ec-53e2-9628-59c09922f332'::uuid;
+        IF pointer_publication_id IS NULL THEN
+            RETURN NULL;
+        END IF;
+        SELECT review_decision_id INTO closure_review_id
+          FROM reviews_publicationrevision WHERE id = pointer_publication_id;
+    ELSIF TG_TABLE_NAME = 'reviews_publishedtravelwarning' THEN
+        SELECT current_publication_id INTO pointer_publication_id
+          FROM reviews_publishedtravelwarning
+         WHERE id = 'ec830f45-da57-5dfd-a289-f11a5acde9e2'::uuid;
+        IF pointer_publication_id IS NULL THEN
+            RETURN NULL;
+        END IF;
+        SELECT review_decision_id INTO closure_review_id
+          FROM reviews_publicationrevision WHERE id = pointer_publication_id;
+    END IF;
+
+    SELECT module, decision, reason_code, actor_id, actor_principal,
+           entry_fact_revision_id, travel_warning_revision_id, correlation_id
+      INTO review_module, review_decision, review_reason, review_actor_id,
+           review_actor_principal, review_entry_id, review_warning_id,
+           review_correlation_id
+      FROM reviews_reviewdecision WHERE id = closure_review_id;
+    IF NOT FOUND THEN
+        RAISE EXCEPTION 'publication closure has no review decision'
+            USING ERRCODE = 'check_violation';
+    END IF;
+
+    SELECT count(*) INTO publication_count
+      FROM reviews_publicationrevision
+     WHERE review_decision_id = closure_review_id;
+    IF publication_count = 1 THEN
+        SELECT id, generation, supersedes_id, rollback_target_id,
+               scope_country_id, scope_passport_policy_id
+          INTO publication_id, publication_generation,
+               publication_supersedes_id, publication_rollback_target_id,
+               publication_country_id, publication_policy_id
+          FROM reviews_publicationrevision
+         WHERE review_decision_id = closure_review_id;
+    END IF;
+    SELECT count(*) INTO audit_count
+      FROM reviews_auditevent
+     WHERE review_decision_id = closure_review_id
+       AND outcome = 'SUCCEEDED';
+    IF audit_count = 1 THEN
+        SELECT action, actor_id, actor_principal, module,
+               entry_fact_revision_id, travel_warning_revision_id,
+               publication_revision_id, prior_publication_revision_id,
+               rollback_target_publication_revision_id, correlation_id
+          INTO audit_action, audit_actor_id, audit_actor_principal,
+               audit_module, audit_entry_id, audit_warning_id,
+               audit_publication_id, audit_prior_id,
+               audit_rollback_target_id, audit_correlation_id
+          FROM reviews_auditevent
+         WHERE review_decision_id = closure_review_id
+           AND outcome = 'SUCCEEDED';
+    END IF;
+
+    IF review_decision = 'REJECTED' THEN
+        IF review_module = 'ENTRY' THEN
+            SELECT publication.entry_fact_revision_id
+              INTO pointer_entry_id
+              FROM reviews_publishedentryfacts AS pointer
+              JOIN reviews_publicationrevision AS publication
+                ON publication.id = pointer.current_publication_id
+             WHERE pointer.id =
+                '6c3602d7-a7ec-53e2-9628-59c09922f332'::uuid;
+            IF pointer_entry_id IS NOT DISTINCT FROM review_entry_id THEN
+                RAISE EXCEPTION 'current entry publication cannot close rejected'
+                    USING ERRCODE = 'check_violation';
+            END IF;
+        ELSE
+            SELECT publication.travel_warning_revision_id
+              INTO pointer_warning_id
+              FROM reviews_publishedtravelwarning AS pointer
+              JOIN reviews_publicationrevision AS publication
+                ON publication.id = pointer.current_publication_id
+             WHERE pointer.id =
+                'ec830f45-da57-5dfd-a289-f11a5acde9e2'::uuid;
+            IF pointer_warning_id IS NOT DISTINCT FROM review_warning_id THEN
+                RAISE EXCEPTION 'current warning publication cannot close rejected'
+                    USING ERRCODE = 'check_violation';
+            END IF;
+        END IF;
+        IF publication_count <> 0 OR audit_count <> 1
+           OR audit_action IS DISTINCT FROM 'REVIEW_REJECT'
+           OR audit_publication_id IS NOT NULL
+           OR audit_prior_id IS NOT NULL
+           OR audit_rollback_target_id IS NOT NULL
+           OR NOT (
+                (review_module = 'ENTRY' AND review_reason = 'REJECT_ENTRY')
+                OR (review_module = 'TRAVEL_WARNING'
+                    AND review_reason = 'REJECT_WARNING')
+           ) THEN
+            RAISE EXCEPTION 'rejected review is not atomically closed'
+                USING ERRCODE = 'check_violation';
+        END IF;
+    ELSIF review_decision = 'APPROVED' THEN
+        IF publication_count <> 1 OR audit_count <> 1 THEN
+            RAISE EXCEPTION 'approved review is not atomically closed'
+                USING ERRCODE = 'check_violation';
+        END IF;
+        IF review_module = 'ENTRY' THEN
+            SELECT current_publication_id INTO pointer_publication_id
+              FROM reviews_publishedentryfacts
+             WHERE id = '6c3602d7-a7ec-53e2-9628-59c09922f332'::uuid;
+            SELECT id INTO latest_subject_review_id
+              FROM reviews_reviewdecision
+             WHERE module = 'ENTRY'
+               AND entry_fact_revision_id = review_entry_id
+             ORDER BY decision_sequence DESC LIMIT 1;
+        ELSE
+            SELECT current_publication_id INTO pointer_publication_id
+              FROM reviews_publishedtravelwarning
+             WHERE id = 'ec830f45-da57-5dfd-a289-f11a5acde9e2'::uuid;
+            SELECT id INTO latest_subject_review_id
+              FROM reviews_reviewdecision
+             WHERE module = 'TRAVEL_WARNING'
+               AND travel_warning_revision_id = review_warning_id
+             ORDER BY decision_sequence DESC LIMIT 1;
+        END IF;
+        IF pointer_publication_id IS DISTINCT FROM publication_id
+           OR latest_subject_review_id IS DISTINCT FROM closure_review_id
+           OR audit_publication_id IS DISTINCT FROM publication_id
+           OR audit_prior_id IS DISTINCT FROM publication_supersedes_id
+           OR audit_rollback_target_id IS DISTINCT FROM
+              publication_rollback_target_id
+           OR NOT (
+                (review_module = 'ENTRY' AND review_reason = 'APPROVE_ENTRY'
+                 AND publication_rollback_target_id IS NULL
+                 AND audit_action = 'PUBLISH')
+                OR (review_module = 'ENTRY' AND review_reason = 'ROLLBACK_ENTRY'
+                    AND publication_rollback_target_id IS NOT NULL
+                    AND audit_action = 'ROLLBACK')
+                OR (review_module = 'TRAVEL_WARNING'
+                    AND review_reason = 'APPROVE_WARNING'
+                    AND publication_rollback_target_id IS NULL
+                    AND audit_action = 'PUBLISH')
+                OR (review_module = 'TRAVEL_WARNING'
+                    AND review_reason = 'ROLLBACK_WARNING'
+                    AND publication_rollback_target_id IS NOT NULL
+                    AND audit_action = 'ROLLBACK')
+           ) THEN
+            RAISE EXCEPTION 'approved review action is not atomically closed'
+                USING ERRCODE = 'check_violation';
+        END IF;
+    ELSE
+        RAISE EXCEPTION 'publication closure has an unsupported review decision'
+            USING ERRCODE = 'check_violation';
+    END IF;
+
+    IF audit_actor_id IS DISTINCT FROM review_actor_id
+       OR audit_actor_principal IS DISTINCT FROM review_actor_principal
+       OR audit_module IS DISTINCT FROM review_module
+       OR audit_entry_id IS DISTINCT FROM review_entry_id
+       OR audit_warning_id IS DISTINCT FROM review_warning_id
+       OR audit_correlation_id IS DISTINCT FROM review_correlation_id THEN
+        RAISE EXCEPTION 'publication closure audit identity is not exact'
+            USING ERRCODE = 'check_violation';
+    END IF;
+
+    IF publication_count = 1 THEN
+        IF publication_generation = 1 THEN
+            IF publication_supersedes_id IS NOT NULL THEN
+                RAISE EXCEPTION 'first publication cannot supersede history'
+                    USING ERRCODE = 'check_violation';
+            END IF;
+        ELSE
+            SELECT module, generation, scope_country_id,
+                   scope_passport_policy_id
+              INTO prior_module, prior_generation, prior_country_id,
+                   prior_policy_id
+              FROM reviews_publicationrevision
+             WHERE id = publication_supersedes_id;
+            IF NOT FOUND OR prior_module IS DISTINCT FROM review_module
+               OR prior_generation IS DISTINCT FROM publication_generation - 1
+               OR prior_country_id IS DISTINCT FROM publication_country_id
+               OR prior_policy_id IS DISTINCT FROM publication_policy_id THEN
+                RAISE EXCEPTION 'publication predecessor is not contiguous'
+                    USING ERRCODE = 'check_violation';
+            END IF;
+        END IF;
+        IF publication_rollback_target_id IS NOT NULL THEN
+            WITH RECURSIVE ancestors AS (
+                SELECT id, supersedes_id
+                  FROM reviews_publicationrevision
+                 WHERE id = publication_supersedes_id
+                UNION ALL
+                SELECT prior.id, prior.supersedes_id
+                  FROM reviews_publicationrevision AS prior
+                  JOIN ancestors ON prior.id = ancestors.supersedes_id
+            )
+            SELECT EXISTS (
+                SELECT 1 FROM ancestors
+                 WHERE id = publication_rollback_target_id
+            ) INTO rollback_is_ancestor;
+            IF NOT rollback_is_ancestor
+               OR publication_rollback_target_id = publication_supersedes_id THEN
+                RAISE EXCEPTION 'rollback target is not a strict ancestor'
+                    USING ERRCODE = 'check_violation';
+            END IF;
+        ELSE
+            IF review_module = 'ENTRY' THEN
+                SELECT count(*) INTO duplicate_subject_count
+                  FROM reviews_publicationrevision
+                 WHERE module = 'ENTRY'
+                   AND entry_fact_revision_id = review_entry_id;
+            ELSE
+                SELECT count(*) INTO duplicate_subject_count
+                  FROM reviews_publicationrevision
+                 WHERE module = 'TRAVEL_WARNING'
+                   AND travel_warning_revision_id = review_warning_id;
+            END IF;
+            IF duplicate_subject_count <> 1 THEN
+                RAISE EXCEPTION 'published history requires explicit rollback'
+                    USING ERRCODE = 'check_violation';
+            END IF;
+        END IF;
+    END IF;
+    RETURN NULL;
+END;
+$$;
+CREATE CONSTRAINT TRIGGER reviews_review_deferred_closure
+AFTER INSERT ON reviews_reviewdecision
+DEFERRABLE INITIALLY DEFERRED
+FOR EACH ROW EXECUTE FUNCTION reviews_enforce_deferred_closure();
+CREATE CONSTRAINT TRIGGER reviews_publication_deferred_closure
+AFTER INSERT ON reviews_publicationrevision
+DEFERRABLE INITIALLY DEFERRED
+FOR EACH ROW EXECUTE FUNCTION reviews_enforce_deferred_closure();
+CREATE CONSTRAINT TRIGGER reviews_audit_deferred_closure
+AFTER INSERT ON reviews_auditevent
+DEFERRABLE INITIALLY DEFERRED
+FOR EACH ROW EXECUTE FUNCTION reviews_enforce_deferred_closure();
+CREATE CONSTRAINT TRIGGER reviews_entry_pointer_deferred_closure
+AFTER UPDATE ON reviews_publishedentryfacts
+DEFERRABLE INITIALLY DEFERRED
+FOR EACH ROW EXECUTE FUNCTION reviews_enforce_deferred_closure();
+CREATE CONSTRAINT TRIGGER reviews_warning_pointer_deferred_closure
+AFTER UPDATE ON reviews_publishedtravelwarning
+DEFERRABLE INITIALLY DEFERRED
+FOR EACH ROW EXECUTE FUNCTION reviews_enforce_deferred_closure();
+"""
+
+
+REVIEWS_GUARDS_REVERSE_SQL = r"""
+LOCK TABLE reviews_auditevent, reviews_publishedentryfacts,
+    reviews_publishedtravelwarning, reviews_publicationrevision,
+    reviews_reviewdecision IN ACCESS EXCLUSIVE MODE;
+DO $$
+BEGIN
+    IF EXISTS (SELECT 1 FROM reviews_auditevent)
+       OR EXISTS (SELECT 1 FROM reviews_publicationrevision)
+       OR EXISTS (SELECT 1 FROM reviews_reviewdecision)
+       OR (SELECT count(*) FROM reviews_publishedentryfacts) <> 1
+       OR (SELECT count(*) FROM reviews_publishedtravelwarning) <> 1
+       OR EXISTS (
+            SELECT 1 FROM reviews_publishedentryfacts
+             WHERE id <> '6c3602d7-a7ec-53e2-9628-59c09922f332'::uuid
+                OR country_id <>
+                   '575fa8b9-14f9-526e-9464-ebd1dea76da9'::uuid
+                OR passport_policy_id <>
+                   'f461f7a7-18f7-5b0d-9831-bfbd47b695e5'::uuid
+                OR current_publication_id IS NOT NULL OR version <> 0
+       )
+       OR EXISTS (
+            SELECT 1 FROM reviews_publishedtravelwarning
+             WHERE id <> 'ec830f45-da57-5dfd-a289-f11a5acde9e2'::uuid
+                OR country_id <>
+                   '575fa8b9-14f9-526e-9464-ebd1dea76da9'::uuid
+                OR current_publication_id IS NOT NULL OR version <> 0
+       ) THEN
+        RAISE EXCEPTION 'publication rollback requires empty immutable history and untouched pointers'
+            USING ERRCODE = 'check_violation';
+    END IF;
+END;
+$$;
+DROP TRIGGER IF EXISTS reviews_warning_pointer_deferred_closure
+    ON reviews_publishedtravelwarning;
+DROP TRIGGER IF EXISTS reviews_entry_pointer_deferred_closure
+    ON reviews_publishedentryfacts;
+DROP TRIGGER IF EXISTS reviews_audit_deferred_closure ON reviews_auditevent;
+DROP TRIGGER IF EXISTS reviews_publication_deferred_closure
+    ON reviews_publicationrevision;
+DROP TRIGGER IF EXISTS reviews_review_deferred_closure
+    ON reviews_reviewdecision;
+DROP FUNCTION IF EXISTS reviews_enforce_deferred_closure();
+DROP TRIGGER IF EXISTS reviews_00_warning_pointer_statement_lock
+    ON reviews_publishedtravelwarning;
+DROP TRIGGER IF EXISTS reviews_00_entry_pointer_statement_lock
+    ON reviews_publishedentryfacts;
+DROP FUNCTION IF EXISTS reviews_prelock_pointer_statement();
+DROP TRIGGER IF EXISTS reviews_00_audit_insert_prelock ON reviews_auditevent;
+DROP FUNCTION IF EXISTS reviews_prelock_audit_insert();
+DROP TRIGGER IF EXISTS reviews_00_publication_insert_prelock
+    ON reviews_publicationrevision;
+DROP FUNCTION IF EXISTS reviews_prelock_publication_insert();
+DROP TRIGGER IF EXISTS reviews_00_review_insert_prelock
+    ON reviews_reviewdecision;
+DROP FUNCTION IF EXISTS reviews_prelock_review_insert();
+DROP TRIGGER IF EXISTS reviews_warning_pointer_guard
+    ON reviews_publishedtravelwarning;
+DROP FUNCTION IF EXISTS reviews_guard_warning_pointer();
+DROP TRIGGER IF EXISTS reviews_entry_pointer_guard
+    ON reviews_publishedentryfacts;
+DROP FUNCTION IF EXISTS reviews_guard_entry_pointer();
+DROP TRIGGER IF EXISTS reviews_audit_event_guard ON reviews_auditevent;
+DROP FUNCTION IF EXISTS reviews_guard_audit_event();
+DROP TRIGGER IF EXISTS reviews_publication_revision_guard
+    ON reviews_publicationrevision;
+DROP FUNCTION IF EXISTS reviews_guard_publication_revision();
+DROP TRIGGER IF EXISTS reviews_review_decision_guard
+    ON reviews_reviewdecision;
+DROP FUNCTION IF EXISTS reviews_guard_review_decision();
+"""
+
+
+class Migration(migrations.Migration):
+    initial = True
+
+    dependencies = [
+        ("countries", "0001_initial"),
+        ("entry_requirements", "0001_initial"),
+        ("travel_warnings", "0002_revision_source_rights_guard"),
+        migrations.swappable_dependency(settings.AUTH_USER_MODEL),
+    ]
+
+    operations = [
+        migrations.CreateModel(
+            name="ReviewDecision",
+            fields=[
+                ("id", models.UUIDField(default=uuid.uuid4, editable=False, primary_key=True, serialize=False)),
+                ("module", models.CharField(choices=[("ENTRY", "Entry requirements"), ("TRAVEL_WARNING", "Travel warnings")], max_length=24)),
+                ("decision", models.CharField(choices=[("APPROVED", "Approved"), ("REJECTED", "Rejected")], max_length=16)),
+                ("decision_sequence", models.PositiveBigIntegerField()),
+                ("reason_code", models.CharField(choices=[("APPROVE_ENTRY", "Approve entry evidence"), ("APPROVE_WARNING", "Approve warning evidence"), ("REJECT_ENTRY", "Reject entry evidence"), ("REJECT_WARNING", "Reject warning evidence"), ("ROLLBACK_ENTRY", "Restore entry history"), ("ROLLBACK_WARNING", "Restore warning history")], max_length=64)),
+                ("actor_principal", models.CharField(max_length=150)),
+                ("correlation_id", models.UUIDField(unique=True)),
+                ("decided_at", models.DateTimeField(default=django.utils.timezone.now)),
+                ("actor", models.ForeignKey(on_delete=django.db.models.deletion.PROTECT, related_name="travel_review_decisions", to=settings.AUTH_USER_MODEL)),
+                ("entry_fact_revision", models.ForeignKey(blank=True, null=True, on_delete=django.db.models.deletion.PROTECT, related_name="review_decisions", to="entry_requirements.entryfactrevision")),
+                ("travel_warning_revision", models.ForeignKey(blank=True, null=True, on_delete=django.db.models.deletion.PROTECT, related_name="review_decisions", to="travel_warnings.travelwarningrevision")),
+            ],
+            options={"permissions": [("publish_entry", "Can publish entry facts"), ("publish_warning", "Can publish travel warnings"), ("rollback_entry", "Can rollback entry facts"), ("rollback_warning", "Can rollback travel warnings"), ("reject_entry", "Can reject entry facts"), ("reject_warning", "Can reject travel warnings")]},
+        ),
+        migrations.CreateModel(
+            name="PublicationRevision",
+            fields=[
+                ("id", models.UUIDField(default=uuid.uuid4, editable=False, primary_key=True, serialize=False)),
+                ("module", models.CharField(choices=[("ENTRY", "Entry requirements"), ("TRAVEL_WARNING", "Travel warnings")], max_length=24)),
+                ("generation", models.PositiveBigIntegerField()),
+                ("completeness", models.CharField(choices=[("LIMITED", "Limited")], default="LIMITED", max_length=16)),
+                ("limitation_code", models.CharField(choices=[("ENTRY_PURPOSE_DATE_APPLICABILITY_UNPROVEN", "Entry purpose and date applicability unproven"), ("WARNING_EFFECTIVE_PERIOD_UNPROVEN", "Warning effective period unproven")], max_length=64)),
+                ("state", models.CharField(choices=[("PUBLISHED", "Published")], default="PUBLISHED", max_length=16)),
+                ("source_code_snapshot", models.CharField(max_length=64)),
+                ("source_revision", models.CharField(max_length=64)),
+                ("source_owner_snapshot", models.CharField(max_length=200)),
+                ("source_locator_snapshot", models.URLField(max_length=500)),
+                ("attribution_text_snapshot", models.CharField(max_length=300)),
+                ("source_contract_fingerprint_sha256", models.CharField(max_length=64)),
+                ("parser_name", models.CharField(max_length=64)),
+                ("parser_version", models.CharField(max_length=64)),
+                ("parser_contract_fingerprint_sha256", models.CharField(max_length=64)),
+                ("schema_fingerprint_sha256", models.CharField(max_length=64)),
+                ("typed_fingerprint_sha256", models.CharField(max_length=64)),
+                ("source_first_observed_at", models.DateTimeField()),
+                ("created_at", models.DateTimeField(default=django.utils.timezone.now)),
+                ("published_at", models.DateTimeField(default=django.utils.timezone.now)),
+                ("entry_fact_revision", models.ForeignKey(blank=True, null=True, on_delete=django.db.models.deletion.PROTECT, related_name="publication_revisions", to="entry_requirements.entryfactrevision")),
+                ("rollback_target", models.ForeignKey(blank=True, null=True, on_delete=django.db.models.deletion.PROTECT, related_name="rollback_restorations", to="reviews.publicationrevision")),
+                ("scope_country", models.ForeignKey(on_delete=django.db.models.deletion.PROTECT, related_name="publication_revisions", to="countries.country")),
+                ("scope_passport_policy", models.ForeignKey(blank=True, null=True, on_delete=django.db.models.deletion.PROTECT, related_name="publication_revisions", to="entry_requirements.passportpolicy")),
+                ("supersedes", models.ForeignKey(blank=True, null=True, on_delete=django.db.models.deletion.PROTECT, related_name="successors", to="reviews.publicationrevision")),
+                ("travel_warning_revision", models.ForeignKey(blank=True, null=True, on_delete=django.db.models.deletion.PROTECT, related_name="publication_revisions", to="travel_warnings.travelwarningrevision")),
+                ("review_decision", models.OneToOneField(on_delete=django.db.models.deletion.PROTECT, related_name="publication_revision", to="reviews.reviewdecision")),
+            ],
+        ),
+        migrations.CreateModel(
+            name="AuditEvent",
+            fields=[
+                ("id", models.UUIDField(default=uuid.uuid4, editable=False, primary_key=True, serialize=False)),
+                ("actor_principal", models.CharField(max_length=150)),
+                ("action", models.CharField(choices=[("REVIEW_REJECT", "Review reject"), ("PUBLISH", "Publish"), ("ROLLBACK", "Rollback"), ("AUTHORIZATION_FAILURE", "Authorization failure"), ("PUBLICATION_FAILURE", "Publication failure")], max_length=32)),
+                ("module", models.CharField(choices=[("ENTRY", "Entry requirements"), ("TRAVEL_WARNING", "Travel warnings")], max_length=24)),
+                ("logical_transaction_id", models.UUIDField(unique=True)),
+                ("correlation_id", models.UUIDField(unique=True)),
+                ("input_identity_sha256", models.CharField(max_length=64)),
+                ("occurred_at", models.DateTimeField(default=django.utils.timezone.now)),
+                ("outcome", models.CharField(choices=[("SUCCEEDED", "Succeeded"), ("FAILED", "Failed")], max_length=16)),
+                ("failure_code", models.CharField(blank=True, choices=[("NOT_AUTHORIZED", "Not authorized"), ("STALE_POINTER", "Stale pointer"), ("INVALID_TARGET", "Invalid target"), ("SOURCE_GATE_FAILED", "Source gate failed"), ("TRANSACTION_ABORTED", "Transaction aborted")], max_length=32)),
+                ("redaction_state", models.CharField(default="REDACTED", max_length=16)),
+                ("actor", models.ForeignKey(blank=True, null=True, on_delete=django.db.models.deletion.PROTECT, related_name="travel_audit_events", to=settings.AUTH_USER_MODEL)),
+                ("entry_fact_revision", models.ForeignKey(blank=True, null=True, on_delete=django.db.models.deletion.PROTECT, related_name="audit_events", to="entry_requirements.entryfactrevision")),
+                ("travel_warning_revision", models.ForeignKey(blank=True, null=True, on_delete=django.db.models.deletion.PROTECT, related_name="audit_events", to="travel_warnings.travelwarningrevision")),
+                ("prior_publication_revision", models.ForeignKey(blank=True, null=True, on_delete=django.db.models.deletion.PROTECT, related_name="prior_audit_events", to="reviews.publicationrevision")),
+                ("publication_revision", models.OneToOneField(blank=True, null=True, on_delete=django.db.models.deletion.PROTECT, related_name="success_audit_event", to="reviews.publicationrevision")),
+                ("rollback_target_publication_revision", models.ForeignKey(blank=True, null=True, on_delete=django.db.models.deletion.PROTECT, related_name="rollback_target_audit_events", to="reviews.publicationrevision")),
+                ("review_decision", models.OneToOneField(blank=True, null=True, on_delete=django.db.models.deletion.PROTECT, related_name="success_audit_event", to="reviews.reviewdecision")),
+            ],
+        ),
+        migrations.CreateModel(
+            name="PublishedEntryFacts",
+            fields=[
+                ("id", models.UUIDField(default=ENTRY_POINTER_ID, editable=False, primary_key=True, serialize=False)),
+                ("version", models.PositiveBigIntegerField(default=0)),
+                ("updated_at", models.DateTimeField(default=django.utils.timezone.now)),
+                ("country", models.ForeignKey(on_delete=django.db.models.deletion.PROTECT, to="countries.country")),
+                ("current_publication", models.OneToOneField(blank=True, null=True, on_delete=django.db.models.deletion.PROTECT, related_name="entry_current_pointer", to="reviews.publicationrevision")),
+                ("passport_policy", models.ForeignKey(on_delete=django.db.models.deletion.PROTECT, to="entry_requirements.passportpolicy")),
+            ],
+            options={"constraints": [models.CheckConstraint(condition=Q(id=ENTRY_POINTER_ID, country_id=JP_COUNTRY_ID, passport_policy_id=PASSPORT_POLICY_ID), name="published_entry_singleton_scope"), models.CheckConstraint(condition=Q(Q(current_publication__isnull=True, version=0), Q(current_publication__isnull=False, version__gte=1), _connector="OR"), name="published_entry_pointer_shape")]},
+        ),
+        migrations.CreateModel(
+            name="PublishedTravelWarning",
+            fields=[
+                ("id", models.UUIDField(default=WARNING_POINTER_ID, editable=False, primary_key=True, serialize=False)),
+                ("version", models.PositiveBigIntegerField(default=0)),
+                ("updated_at", models.DateTimeField(default=django.utils.timezone.now)),
+                ("country", models.ForeignKey(on_delete=django.db.models.deletion.PROTECT, to="countries.country")),
+                ("current_publication", models.OneToOneField(blank=True, null=True, on_delete=django.db.models.deletion.PROTECT, related_name="warning_current_pointer", to="reviews.publicationrevision")),
+            ],
+            options={"constraints": [models.CheckConstraint(condition=Q(id=WARNING_POINTER_ID, country_id=JP_COUNTRY_ID), name="published_warning_singleton_scope"), models.CheckConstraint(condition=Q(Q(current_publication__isnull=True, version=0), Q(current_publication__isnull=False, version__gte=1), _connector="OR"), name="published_warning_pointer_shape")]},
+        ),
+        migrations.AddConstraint(model_name="reviewdecision", constraint=models.CheckConstraint(condition=Q(Q(entry_fact_revision__isnull=False, module="ENTRY", travel_warning_revision__isnull=True), Q(entry_fact_revision__isnull=True, module="TRAVEL_WARNING", travel_warning_revision__isnull=False), _connector="OR"), name="review_exact_typed_subject")),
+        migrations.AddConstraint(model_name="reviewdecision", constraint=models.CheckConstraint(condition=Q(decision_sequence__gte=1), name="review_sequence_positive")),
+        migrations.AddConstraint(model_name="reviewdecision", constraint=models.CheckConstraint(condition=Q(Q(decision="APPROVED", module="ENTRY", reason_code__in=["APPROVE_ENTRY", "ROLLBACK_ENTRY"]), Q(decision="REJECTED", module="ENTRY", reason_code="REJECT_ENTRY"), Q(decision="APPROVED", module="TRAVEL_WARNING", reason_code__in=["APPROVE_WARNING", "ROLLBACK_WARNING"]), Q(decision="REJECTED", module="TRAVEL_WARNING", reason_code="REJECT_WARNING"), _connector="OR"), name="review_decision_reason_closed")),
+        migrations.AddConstraint(model_name="reviewdecision", constraint=models.CheckConstraint(condition=~Q(actor_principal=""), name="review_actor_principal_present")),
+        migrations.AddConstraint(model_name="reviewdecision", constraint=models.UniqueConstraint(condition=Q(module="ENTRY"), fields=("entry_fact_revision", "decision_sequence"), name="review_entry_sequence_unique")),
+        migrations.AddConstraint(model_name="reviewdecision", constraint=models.UniqueConstraint(condition=Q(module="TRAVEL_WARNING"), fields=("travel_warning_revision", "decision_sequence"), name="review_warning_sequence_unique")),
+        migrations.AddConstraint(model_name="publicationrevision", constraint=models.CheckConstraint(condition=Q(scope_country_id=JP_COUNTRY_ID), name="publication_scope_country_jp")),
+        migrations.AddConstraint(model_name="publicationrevision", constraint=models.CheckConstraint(condition=Q(Q(entry_fact_revision__isnull=False, limitation_code="ENTRY_PURPOSE_DATE_APPLICABILITY_UNPROVEN", module="ENTRY", parser_name="MOFA_ENTRY_CSV", parser_version="V1", scope_passport_policy_id=PASSPORT_POLICY_ID, source_code_snapshot="MOFA_ENTRY_CSV", travel_warning_revision__isnull=True), Q(entry_fact_revision__isnull=True, limitation_code="WARNING_EFFECTIVE_PERIOD_UNPROVEN", module="TRAVEL_WARNING", parser_name="MOFA_TRAVEL_ALARM_JSON", parser_version="V1", scope_passport_policy__isnull=True, source_code_snapshot="MOFA_TRAVEL_ALARM_API_JP", travel_warning_revision__isnull=False), _connector="OR"), name="publication_exact_typed_scope")),
+        migrations.AddConstraint(model_name="publicationrevision", constraint=models.CheckConstraint(condition=Q(generation__gte=1), name="publication_generation_positive")),
+        migrations.AddConstraint(model_name="publicationrevision", constraint=models.CheckConstraint(condition=Q(completeness="LIMITED"), name="publication_phase0_limited")),
+        migrations.AddConstraint(model_name="publicationrevision", constraint=models.CheckConstraint(condition=Q(state="PUBLISHED"), name="publication_inserted_final")),
+        migrations.AddConstraint(model_name="publicationrevision", constraint=models.CheckConstraint(condition=(Q(published_at__gte=F("created_at")) & Q(created_at__gte=F("source_first_observed_at"))), name="publication_time_causal")),
+        migrations.AddConstraint(model_name="publicationrevision", constraint=models.CheckConstraint(condition=Q(source_revision__regex=r"^[A-Za-z0-9][A-Za-z0-9._-]{0,63}$"), name="publication_source_revision_format")),
+        migrations.AddConstraint(model_name="publicationrevision", constraint=models.CheckConstraint(condition=(~Q(source_owner_snapshot="") & Q(source_locator_snapshot__startswith="https://") & ~Q(attribution_text_snapshot="")), name="publication_attribution_present")),
+        migrations.AddConstraint(model_name="publicationrevision", constraint=models.CheckConstraint(condition=~Q(id=F("supersedes_id")), name="publication_not_self_superseding")),
+        migrations.AddConstraint(model_name="publicationrevision", constraint=models.CheckConstraint(condition=~Q(id=F("rollback_target_id")), name="publication_not_self_rollback")),
+        *[migrations.AddConstraint(model_name="publicationrevision", constraint=models.CheckConstraint(condition=Q(**{f"{field}__regex": r"^[0-9a-f]{64}$"}), name=name)) for field, name in (("source_contract_fingerprint_sha256", "publication_source_contract_hash"), ("parser_contract_fingerprint_sha256", "publication_parser_contract_hash"), ("schema_fingerprint_sha256", "publication_schema_hash"), ("typed_fingerprint_sha256", "publication_typed_hash"))],
+        migrations.AddConstraint(model_name="publicationrevision", constraint=models.UniqueConstraint(condition=Q(module="ENTRY"), fields=("module", "scope_country", "scope_passport_policy", "generation"), name="publication_entry_generation_unique")),
+        migrations.AddConstraint(model_name="publicationrevision", constraint=models.UniqueConstraint(condition=Q(module="TRAVEL_WARNING"), fields=("module", "scope_country", "generation"), name="publication_warning_generation_unique")),
+        migrations.AddConstraint(model_name="publicationrevision", constraint=models.UniqueConstraint(condition=Q(supersedes__isnull=False), fields=("supersedes",), name="publication_linear_successor")),
+        migrations.AddConstraint(model_name="auditevent", constraint=models.CheckConstraint(condition=Q(Q(entry_fact_revision__isnull=False, module="ENTRY", travel_warning_revision__isnull=True), Q(entry_fact_revision__isnull=True, module="TRAVEL_WARNING", travel_warning_revision__isnull=False), Q(entry_fact_revision__isnull=True, outcome="FAILED", travel_warning_revision__isnull=True), _connector="OR"), name="audit_typed_subject_shape")),
+        migrations.AddConstraint(model_name="auditevent", constraint=models.CheckConstraint(condition=~Q(actor_principal=""), name="audit_actor_principal_present")),
+        migrations.AddConstraint(model_name="auditevent", constraint=models.CheckConstraint(condition=Q(input_identity_sha256__regex=r"^[0-9a-f]{64}$"), name="audit_input_identity_hash")),
+        migrations.AddConstraint(model_name="auditevent", constraint=models.CheckConstraint(condition=Q(redaction_state="REDACTED"), name="audit_redaction_fixed")),
+        migrations.AddConstraint(model_name="auditevent", constraint=models.CheckConstraint(condition=Q(Q(action="REVIEW_REJECT", failure_code="", outcome="SUCCEEDED", prior_publication_revision__isnull=True, publication_revision__isnull=True, review_decision__isnull=False, rollback_target_publication_revision__isnull=True), Q(action="PUBLISH", failure_code="", outcome="SUCCEEDED", publication_revision__isnull=False, review_decision__isnull=False, rollback_target_publication_revision__isnull=True), Q(action="ROLLBACK", failure_code="", outcome="SUCCEEDED", prior_publication_revision__isnull=False, publication_revision__isnull=False, review_decision__isnull=False, rollback_target_publication_revision__isnull=False), Q(action="AUTHORIZATION_FAILURE", failure_code="NOT_AUTHORIZED", outcome="FAILED", prior_publication_revision__isnull=True, publication_revision__isnull=True, review_decision__isnull=True, rollback_target_publication_revision__isnull=True), Q(action="PUBLICATION_FAILURE", failure_code__in=["STALE_POINTER", "INVALID_TARGET", "SOURCE_GATE_FAILED", "TRANSACTION_ABORTED"], outcome="FAILED", prior_publication_revision__isnull=True, publication_revision__isnull=True, review_decision__isnull=True, rollback_target_publication_revision__isnull=True), _connector="OR"), name="audit_action_outcome_shape")),
+        migrations.RunPython(seed_module_pointers, unseed_module_pointers),
+        migrations.RunSQL(REVIEWS_GUARDS_SQL, REVIEWS_GUARDS_REVERSE_SQL),
+    ]
diff --git a/reviews/migrations/__init__.py b/reviews/migrations/__init__.py
new file mode 100644
index 0000000..8b13789
--- /dev/null
+++ b/reviews/migrations/__init__.py
@@ -0,0 +1 @@
+
diff --git a/reviews/models.py b/reviews/models.py
new file mode 100644
index 0000000..12e564c
--- /dev/null
+++ b/reviews/models.py
@@ -0,0 +1,607 @@
+import uuid
+
+from django.conf import settings
+from django.db import models
+from django.db.models import F, Q
+from django.utils import timezone
+
+from countries.models import JP_COUNTRY_ID
+from entry_requirements.models import PASSPORT_POLICY_ID
+
+
+ENTRY_POINTER_ID = uuid.UUID("6c3602d7-a7ec-53e2-9628-59c09922f332")
+WARNING_POINTER_ID = uuid.UUID("ec830f45-da57-5dfd-a289-f11a5acde9e2")
+
+
+class PublicationModule(models.TextChoices):
+    ENTRY = "ENTRY", "Entry requirements"
+    TRAVEL_WARNING = "TRAVEL_WARNING", "Travel warnings"
+
+
+class ReviewDecision(models.Model):
+    class Decision(models.TextChoices):
+        APPROVED = "APPROVED", "Approved"
+        REJECTED = "REJECTED", "Rejected"
+
+    class ReasonCode(models.TextChoices):
+        APPROVE_ENTRY = "APPROVE_ENTRY", "Approve entry evidence"
+        APPROVE_WARNING = "APPROVE_WARNING", "Approve warning evidence"
+        REJECT_ENTRY = "REJECT_ENTRY", "Reject entry evidence"
+        REJECT_WARNING = "REJECT_WARNING", "Reject warning evidence"
+        ROLLBACK_ENTRY = "ROLLBACK_ENTRY", "Restore entry history"
+        ROLLBACK_WARNING = "ROLLBACK_WARNING", "Restore warning history"
+
+    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
+    module = models.CharField(max_length=24, choices=PublicationModule.choices)
+    entry_fact_revision = models.ForeignKey(
+        "entry_requirements.EntryFactRevision",
+        on_delete=models.PROTECT,
+        related_name="review_decisions",
+        null=True,
+        blank=True,
+    )
+    travel_warning_revision = models.ForeignKey(
+        "travel_warnings.TravelWarningRevision",
+        on_delete=models.PROTECT,
+        related_name="review_decisions",
+        null=True,
+        blank=True,
+    )
+    decision = models.CharField(max_length=16, choices=Decision.choices)
+    decision_sequence = models.PositiveBigIntegerField()
+    reason_code = models.CharField(max_length=64, choices=ReasonCode.choices)
+    actor = models.ForeignKey(
+        settings.AUTH_USER_MODEL,
+        on_delete=models.PROTECT,
+        related_name="travel_review_decisions",
+    )
+    actor_principal = models.CharField(max_length=150)
+    correlation_id = models.UUIDField(unique=True)
+    decided_at = models.DateTimeField(default=timezone.now)
+
+    class Meta:
+        permissions = [
+            ("publish_entry", "Can publish entry facts"),
+            ("publish_warning", "Can publish travel warnings"),
+            ("rollback_entry", "Can rollback entry facts"),
+            ("rollback_warning", "Can rollback travel warnings"),
+            ("reject_entry", "Can reject entry facts"),
+            ("reject_warning", "Can reject travel warnings"),
+        ]
+        constraints = [
+            models.CheckConstraint(
+                condition=(
+                    Q(
+                        module=PublicationModule.ENTRY,
+                        entry_fact_revision__isnull=False,
+                        travel_warning_revision__isnull=True,
+                    )
+                    | Q(
+                        module=PublicationModule.TRAVEL_WARNING,
+                        entry_fact_revision__isnull=True,
+                        travel_warning_revision__isnull=False,
+                    )
+                ),
+                name="review_exact_typed_subject",
+            ),
+            models.CheckConstraint(
+                condition=Q(decision_sequence__gte=1),
+                name="review_sequence_positive",
+            ),
+            models.CheckConstraint(
+                condition=(
+                    Q(
+                        module=PublicationModule.ENTRY,
+                        decision="APPROVED",
+                        reason_code__in=[
+                            "APPROVE_ENTRY",
+                            "ROLLBACK_ENTRY",
+                        ],
+                    )
+                    | Q(
+                        module=PublicationModule.ENTRY,
+                        decision="REJECTED",
+                        reason_code="REJECT_ENTRY",
+                    )
+                    | Q(
+                        module=PublicationModule.TRAVEL_WARNING,
+                        decision="APPROVED",
+                        reason_code__in=[
+                            "APPROVE_WARNING",
+                            "ROLLBACK_WARNING",
+                        ],
+                    )
+                    | Q(
+                        module=PublicationModule.TRAVEL_WARNING,
+                        decision="REJECTED",
+                        reason_code="REJECT_WARNING",
+                    )
+                ),
+                name="review_decision_reason_closed",
+            ),
+            models.CheckConstraint(
+                condition=~Q(actor_principal=""),
+                name="review_actor_principal_present",
+            ),
+            models.UniqueConstraint(
+                fields=("entry_fact_revision", "decision_sequence"),
+                condition=Q(module=PublicationModule.ENTRY),
+                name="review_entry_sequence_unique",
+            ),
+            models.UniqueConstraint(
+                fields=("travel_warning_revision", "decision_sequence"),
+                condition=Q(module=PublicationModule.TRAVEL_WARNING),
+                name="review_warning_sequence_unique",
+            ),
+        ]
+
+
+class PublicationRevision(models.Model):
+    class Completeness(models.TextChoices):
+        LIMITED = "LIMITED", "Limited"
+
+    class State(models.TextChoices):
+        PUBLISHED = "PUBLISHED", "Published"
+
+    class LimitationCode(models.TextChoices):
+        ENTRY = (
+            "ENTRY_PURPOSE_DATE_APPLICABILITY_UNPROVEN",
+            "Entry purpose and date applicability unproven",
+        )
+        TRAVEL_WARNING = (
+            "WARNING_EFFECTIVE_PERIOD_UNPROVEN",
+            "Warning effective period unproven",
+        )
+
+    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
+    module = models.CharField(max_length=24, choices=PublicationModule.choices)
+    scope_country = models.ForeignKey(
+        "countries.Country",
+        on_delete=models.PROTECT,
+        related_name="publication_revisions",
+    )
+    scope_passport_policy = models.ForeignKey(
+        "entry_requirements.PassportPolicy",
+        on_delete=models.PROTECT,
+        related_name="publication_revisions",
+        null=True,
+        blank=True,
+    )
+    entry_fact_revision = models.ForeignKey(
+        "entry_requirements.EntryFactRevision",
+        on_delete=models.PROTECT,
+        related_name="publication_revisions",
+        null=True,
+        blank=True,
+    )
+    travel_warning_revision = models.ForeignKey(
+        "travel_warnings.TravelWarningRevision",
+        on_delete=models.PROTECT,
+        related_name="publication_revisions",
+        null=True,
+        blank=True,
+    )
+    review_decision = models.OneToOneField(
+        ReviewDecision,
+        on_delete=models.PROTECT,
+        related_name="publication_revision",
+    )
+    generation = models.PositiveBigIntegerField()
+    completeness = models.CharField(
+        max_length=16,
+        choices=Completeness.choices,
+        default=Completeness.LIMITED,
+    )
+    limitation_code = models.CharField(
+        max_length=64,
+        choices=LimitationCode.choices,
+    )
+    state = models.CharField(
+        max_length=16,
+        choices=State.choices,
+        default=State.PUBLISHED,
+    )
+    supersedes = models.ForeignKey(
+        "self",
+        on_delete=models.PROTECT,
+        related_name="successors",
+        null=True,
+        blank=True,
+    )
+    rollback_target = models.ForeignKey(
+        "self",
+        on_delete=models.PROTECT,
+        related_name="rollback_restorations",
+        null=True,
+        blank=True,
+    )
+    source_code_snapshot = models.CharField(max_length=64)
+    source_revision = models.CharField(max_length=64)
+    source_owner_snapshot = models.CharField(max_length=200)
+    source_locator_snapshot = models.URLField(max_length=500)
+    attribution_text_snapshot = models.CharField(max_length=300)
+    source_contract_fingerprint_sha256 = models.CharField(max_length=64)
+    parser_name = models.CharField(max_length=64)
+    parser_version = models.CharField(max_length=64)
+    parser_contract_fingerprint_sha256 = models.CharField(max_length=64)
+    schema_fingerprint_sha256 = models.CharField(max_length=64)
+    typed_fingerprint_sha256 = models.CharField(max_length=64)
+    source_first_observed_at = models.DateTimeField()
+    created_at = models.DateTimeField(default=timezone.now)
+    published_at = models.DateTimeField(default=timezone.now)
+
+    class Meta:
+        constraints = [
+            models.CheckConstraint(
+                condition=Q(scope_country_id=JP_COUNTRY_ID),
+                name="publication_scope_country_jp",
+            ),
+            models.CheckConstraint(
+                condition=(
+                    Q(
+                        module=PublicationModule.ENTRY,
+                        scope_passport_policy_id=PASSPORT_POLICY_ID,
+                        entry_fact_revision__isnull=False,
+                        travel_warning_revision__isnull=True,
+                        limitation_code=(
+                            "ENTRY_PURPOSE_DATE_APPLICABILITY_UNPROVEN"
+                        ),
+                        source_code_snapshot="MOFA_ENTRY_CSV",
+                        parser_name="MOFA_ENTRY_CSV",
+                        parser_version="V1",
+                    )
+                    | Q(
+                        module=PublicationModule.TRAVEL_WARNING,
+                        scope_passport_policy__isnull=True,
+                        entry_fact_revision__isnull=True,
+                        travel_warning_revision__isnull=False,
+                        limitation_code="WARNING_EFFECTIVE_PERIOD_UNPROVEN",
+                        source_code_snapshot="MOFA_TRAVEL_ALARM_API_JP",
+                        parser_name="MOFA_TRAVEL_ALARM_JSON",
+                        parser_version="V1",
+                    )
+                ),
+                name="publication_exact_typed_scope",
+            ),
+            models.CheckConstraint(
+                condition=Q(generation__gte=1),
+                name="publication_generation_positive",
+            ),
+            models.CheckConstraint(
+                condition=Q(completeness="LIMITED"),
+                name="publication_phase0_limited",
+            ),
+            models.CheckConstraint(
+                condition=Q(state="PUBLISHED"),
+                name="publication_inserted_final",
+            ),
+            models.CheckConstraint(
+                condition=(
+                    Q(published_at__gte=F("created_at"))
+                    & Q(created_at__gte=F("source_first_observed_at"))
+                ),
+                name="publication_time_causal",
+            ),
+            models.CheckConstraint(
+                condition=Q(
+                    source_revision__regex=(
+                        r"^[A-Za-z0-9][A-Za-z0-9._-]{0,63}$"
+                    )
+                ),
+                name="publication_source_revision_format",
+            ),
+            models.CheckConstraint(
+                condition=(
+                    ~Q(source_owner_snapshot="")
+                    & Q(source_locator_snapshot__startswith="https://")
+                    & ~Q(attribution_text_snapshot="")
+                ),
+                name="publication_attribution_present",
+            ),
+            models.CheckConstraint(
+                condition=~Q(id=F("supersedes_id")),
+                name="publication_not_self_superseding",
+            ),
+            models.CheckConstraint(
+                condition=~Q(id=F("rollback_target_id")),
+                name="publication_not_self_rollback",
+            ),
+            *[
+                models.CheckConstraint(
+                    condition=Q(
+                        **{f"{field_name}__regex": r"^[0-9a-f]{64}$"}
+                    ),
+                    name=constraint_name,
+                )
+                for field_name, constraint_name in (
+                    (
+                        "source_contract_fingerprint_sha256",
+                        "publication_source_contract_hash",
+                    ),
+                    (
+                        "parser_contract_fingerprint_sha256",
+                        "publication_parser_contract_hash",
+                    ),
+                    (
+                        "schema_fingerprint_sha256",
+                        "publication_schema_hash",
+                    ),
+                    (
+                        "typed_fingerprint_sha256",
+                        "publication_typed_hash",
+                    ),
+                )
+            ],
+            models.UniqueConstraint(
+                fields=(
+                    "module",
+                    "scope_country",
+                    "scope_passport_policy",
+                    "generation",
+                ),
+                condition=Q(module=PublicationModule.ENTRY),
+                name="publication_entry_generation_unique",
+            ),
+            models.UniqueConstraint(
+                fields=("module", "scope_country", "generation"),
+                condition=Q(module=PublicationModule.TRAVEL_WARNING),
+                name="publication_warning_generation_unique",
+            ),
+            models.UniqueConstraint(
+                fields=("supersedes",),
+                condition=Q(supersedes__isnull=False),
+                name="publication_linear_successor",
+            ),
+        ]
+
+
+class PublishedEntryFacts(models.Model):
+    id = models.UUIDField(
+        primary_key=True,
+        default=ENTRY_POINTER_ID,
+        editable=False,
+    )
+    country = models.ForeignKey("countries.Country", on_delete=models.PROTECT)
+    passport_policy = models.ForeignKey(
+        "entry_requirements.PassportPolicy",
+        on_delete=models.PROTECT,
+    )
+    current_publication = models.OneToOneField(
+        PublicationRevision,
+        on_delete=models.PROTECT,
+        null=True,
+        blank=True,
+        related_name="entry_current_pointer",
+    )
+    version = models.PositiveBigIntegerField(default=0)
+    updated_at = models.DateTimeField(default=timezone.now)
+
+    class Meta:
+        constraints = [
+            models.CheckConstraint(
+                condition=Q(
+                    id=ENTRY_POINTER_ID,
+                    country_id=JP_COUNTRY_ID,
+                    passport_policy_id=PASSPORT_POLICY_ID,
+                ),
+                name="published_entry_singleton_scope",
+            ),
+            models.CheckConstraint(
+                condition=(
+                    Q(current_publication__isnull=True, version=0)
+                    | Q(current_publication__isnull=False, version__gte=1)
+                ),
+                name="published_entry_pointer_shape",
+            ),
+        ]
+
+
+class PublishedTravelWarning(models.Model):
+    id = models.UUIDField(
+        primary_key=True,
+        default=WARNING_POINTER_ID,
+        editable=False,
+    )
+    country = models.ForeignKey("countries.Country", on_delete=models.PROTECT)
+    current_publication = models.OneToOneField(
+        PublicationRevision,
+        on_delete=models.PROTECT,
+        null=True,
+        blank=True,
+        related_name="warning_current_pointer",
+    )
+    version = models.PositiveBigIntegerField(default=0)
+    updated_at = models.DateTimeField(default=timezone.now)
+
+    class Meta:
+        constraints = [
+            models.CheckConstraint(
+                condition=Q(id=WARNING_POINTER_ID, country_id=JP_COUNTRY_ID),
+                name="published_warning_singleton_scope",
+            ),
+            models.CheckConstraint(
+                condition=(
+                    Q(current_publication__isnull=True, version=0)
+                    | Q(current_publication__isnull=False, version__gte=1)
+                ),
+                name="published_warning_pointer_shape",
+            ),
+        ]
+
+
+class AuditEvent(models.Model):
+    class Action(models.TextChoices):
+        REVIEW_REJECT = "REVIEW_REJECT", "Review reject"
+        PUBLISH = "PUBLISH", "Publish"
+        ROLLBACK = "ROLLBACK", "Rollback"
+        AUTHORIZATION_FAILURE = (
+            "AUTHORIZATION_FAILURE",
+            "Authorization failure",
+        )
+        PUBLICATION_FAILURE = "PUBLICATION_FAILURE", "Publication failure"
+
+    class Outcome(models.TextChoices):
+        SUCCEEDED = "SUCCEEDED", "Succeeded"
+        FAILED = "FAILED", "Failed"
+
+    class FailureCode(models.TextChoices):
+        NOT_AUTHORIZED = "NOT_AUTHORIZED", "Not authorized"
+        STALE_POINTER = "STALE_POINTER", "Stale pointer"
+        INVALID_TARGET = "INVALID_TARGET", "Invalid target"
+        SOURCE_GATE_FAILED = "SOURCE_GATE_FAILED", "Source gate failed"
+        TRANSACTION_ABORTED = "TRANSACTION_ABORTED", "Transaction aborted"
+
+    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
+    actor = models.ForeignKey(
+        settings.AUTH_USER_MODEL,
+        on_delete=models.PROTECT,
+        related_name="travel_audit_events",
+        null=True,
+        blank=True,
+    )
+    actor_principal = models.CharField(max_length=150)
+    action = models.CharField(max_length=32, choices=Action.choices)
+    module = models.CharField(max_length=24, choices=PublicationModule.choices)
+    entry_fact_revision = models.ForeignKey(
+        "entry_requirements.EntryFactRevision",
+        on_delete=models.PROTECT,
+        related_name="audit_events",
+        null=True,
+        blank=True,
+    )
+    travel_warning_revision = models.ForeignKey(
+        "travel_warnings.TravelWarningRevision",
+        on_delete=models.PROTECT,
+        related_name="audit_events",
+        null=True,
+        blank=True,
+    )
+    review_decision = models.OneToOneField(
+        ReviewDecision,
+        on_delete=models.PROTECT,
+        related_name="success_audit_event",
+        null=True,
+        blank=True,
+    )
+    publication_revision = models.OneToOneField(
+        PublicationRevision,
+        on_delete=models.PROTECT,
+        related_name="success_audit_event",
+        null=True,
+        blank=True,
+    )
+    prior_publication_revision = models.ForeignKey(
+        PublicationRevision,
+        on_delete=models.PROTECT,
+        related_name="prior_audit_events",
+        null=True,
+        blank=True,
+    )
+    rollback_target_publication_revision = models.ForeignKey(
+        PublicationRevision,
+        on_delete=models.PROTECT,
+        related_name="rollback_target_audit_events",
+        null=True,
+        blank=True,
+    )
+    logical_transaction_id = models.UUIDField(unique=True)
+    correlation_id = models.UUIDField(unique=True)
+    input_identity_sha256 = models.CharField(max_length=64)
+    occurred_at = models.DateTimeField(default=timezone.now)
+    outcome = models.CharField(max_length=16, choices=Outcome.choices)
+    failure_code = models.CharField(
+        max_length=32,
+        choices=FailureCode.choices,
+        blank=True,
+    )
+    redaction_state = models.CharField(max_length=16, default="REDACTED")
+
+    class Meta:
+        constraints = [
+            models.CheckConstraint(
+                condition=(
+                    Q(
+                        module=PublicationModule.ENTRY,
+                        entry_fact_revision__isnull=False,
+                        travel_warning_revision__isnull=True,
+                    )
+                    | Q(
+                        module=PublicationModule.TRAVEL_WARNING,
+                        entry_fact_revision__isnull=True,
+                        travel_warning_revision__isnull=False,
+                    )
+                    | Q(
+                        outcome="FAILED",
+                        entry_fact_revision__isnull=True,
+                        travel_warning_revision__isnull=True,
+                    )
+                ),
+                name="audit_typed_subject_shape",
+            ),
+            models.CheckConstraint(
+                condition=~Q(actor_principal=""),
+                name="audit_actor_principal_present",
+            ),
+            models.CheckConstraint(
+                condition=Q(input_identity_sha256__regex=r"^[0-9a-f]{64}$"),
+                name="audit_input_identity_hash",
+            ),
+            models.CheckConstraint(
+                condition=Q(redaction_state="REDACTED"),
+                name="audit_redaction_fixed",
+            ),
+            models.CheckConstraint(
+                condition=(
+                    Q(
+                        action="REVIEW_REJECT",
+                        outcome="SUCCEEDED",
+                        failure_code="",
+                        review_decision__isnull=False,
+                        publication_revision__isnull=True,
+                        prior_publication_revision__isnull=True,
+                        rollback_target_publication_revision__isnull=True,
+                    )
+                    | Q(
+                        action="PUBLISH",
+                        outcome="SUCCEEDED",
+                        failure_code="",
+                        review_decision__isnull=False,
+                        publication_revision__isnull=False,
+                        rollback_target_publication_revision__isnull=True,
+                    )
+                    | Q(
+                        action="ROLLBACK",
+                        outcome="SUCCEEDED",
+                        failure_code="",
+                        review_decision__isnull=False,
+                        publication_revision__isnull=False,
+                        prior_publication_revision__isnull=False,
+                        rollback_target_publication_revision__isnull=False,
+                    )
+                    | Q(
+                        action="AUTHORIZATION_FAILURE",
+                        outcome="FAILED",
+                        failure_code="NOT_AUTHORIZED",
+                        review_decision__isnull=True,
+                        publication_revision__isnull=True,
+                        prior_publication_revision__isnull=True,
+                        rollback_target_publication_revision__isnull=True,
+                    )
+                    | Q(
+                        action="PUBLICATION_FAILURE",
+                        outcome="FAILED",
+                        failure_code__in=[
+                            "STALE_POINTER",
+                            "INVALID_TARGET",
+                            "SOURCE_GATE_FAILED",
+                            "TRANSACTION_ABORTED",
+                        ],
+                        review_decision__isnull=True,
+                        publication_revision__isnull=True,
+                        prior_publication_revision__isnull=True,
+                        rollback_target_publication_revision__isnull=True,
+                    )
+                ),
+                name="audit_action_outcome_shape",
+            ),
+        ]
diff --git a/reviews/publication.py b/reviews/publication.py
new file mode 100644
index 0000000..bec2786
--- /dev/null
+++ b/reviews/publication.py
@@ -0,0 +1,1011 @@
+"""Atomic, redacted publication services for the two approved modules."""
+
+from __future__ import annotations
+
+import hashlib
+import uuid
+from dataclasses import dataclass
+from enum import Enum
+from typing import Any, NoReturn
+
+from django.contrib.auth import get_user_model
+from django.db import connection, transaction
+from django.db.models import F
+from django.utils import timezone
+
+from countries.models import JP_COUNTRY_ID
+from entry_requirements.ingestion import (
+    ENTRY_ATTRIBUTION,
+    ENTRY_CONTRACT_FINGERPRINT_SHA256,
+    ENTRY_DECIDED_BY,
+    ENTRY_DECISION_BASIS,
+    ENTRY_FIELD_SCOPE,
+    ENTRY_PARSER_NAME,
+    ENTRY_PARSER_VERSION,
+    ENTRY_SOURCE_CODE,
+    ENTRY_SOURCE_MODULE,
+    ENTRY_SOURCE_OWNER,
+    ENTRY_SOURCE_REVISION,
+)
+from entry_requirements.models import (
+    PASSPORT_POLICY_ID,
+    EntryFactRevision,
+)
+from entry_requirements.parser import ENTRY_SCHEMA_FINGERPRINT_SHA256
+from sources.models import (
+    FetchAttempt,
+    ParseRun,
+    SourceArtifact,
+    SourceRightsDecision,
+)
+from sources.transport import ENTRY_SOURCE_LOCATOR, WARNING_SOURCE_LOCATOR
+from travel_warnings.ingestion import (
+    WARNING_ATTRIBUTION,
+    WARNING_DECIDED_BY,
+    WARNING_DECISION_BASIS,
+    WARNING_FIELD_SCOPE,
+    WARNING_PARSER_NAME,
+    WARNING_PARSER_VERSION,
+    WARNING_SOURCE_CODE,
+    WARNING_SOURCE_MODULE,
+    WARNING_SOURCE_OWNER,
+    WARNING_SOURCE_REVISION,
+)
+from travel_warnings.models import TravelWarningRevision
+from travel_warnings.parser import (
+    EXPECTED_SCHEMA_FINGERPRINT_SHA256,
+    PARSER_CONTRACT_FINGERPRINT_SHA256,
+)
+
+from .models import (
+    AuditEvent,
+    PublicationModule,
+    PublicationRevision,
+    PublishedEntryFacts,
+    PublishedTravelWarning,
+    ReviewDecision,
+)
+
+
+PUBLICATION_LOCK_NAMESPACE = 1_414_678_614
+PUBLICATION_LOCK_KEYS = {
+    PublicationModule.ENTRY: 2_007_001,
+    PublicationModule.TRAVEL_WARNING: 2_007_002,
+}
+
+
+class PublicationCode:
+    PUBLISHED = "PUBLISHED"
+    REJECTED = "REJECTED"
+    ROLLED_BACK = "ROLLED_BACK"
+    NOT_AUTHORIZED = "NOT_AUTHORIZED"
+    STALE_POINTER = "STALE_POINTER"
+    INVALID_TARGET = "INVALID_TARGET"
+    SOURCE_GATE_FAILED = "SOURCE_GATE_FAILED"
+    TRANSACTION_ABORTED = "TRANSACTION_ABORTED"
+    AUDIT_UNAVAILABLE = "AUDIT_UNAVAILABLE"
+
+
+@dataclass(frozen=True, slots=True)
+class PublicationOutcome:
+    code: str
+    module: str
+    generation: int | None = None
+    pointer_version: int | None = None
+
+    @property
+    def succeeded(self) -> bool:
+        return self.code in {
+            PublicationCode.PUBLISHED,
+            PublicationCode.REJECTED,
+            PublicationCode.ROLLED_BACK,
+        }
+
+
+@dataclass(frozen=True, slots=True)
+class _ModuleSpec:
+    module: str
+    typed_model: type
+    typed_field: str
+    pointer_model: type
+    permission_publish: str
+    permission_reject: str
+    permission_rollback: str
+    approve_reason: str
+    reject_reason: str
+    rollback_reason: str
+    limitation_code: str
+    source_code: str
+    source_revision: str
+    source_module: str
+    source_owner: str
+    source_locator: str
+    field_scope: str
+    attribution: str
+    contract_fingerprint: str
+    decided_by: str
+    decision_basis: str
+    parser_name: str
+    parser_version: str
+    schema_fingerprint: str
+    access_mode: str
+    secret_reference_name: str
+    provider_code: str
+
+
+_SPECS = {
+    PublicationModule.ENTRY: _ModuleSpec(
+        module=PublicationModule.ENTRY,
+        typed_model=EntryFactRevision,
+        typed_field="entry_fact_revision",
+        pointer_model=PublishedEntryFacts,
+        permission_publish="reviews.publish_entry",
+        permission_reject="reviews.reject_entry",
+        permission_rollback="reviews.rollback_entry",
+        approve_reason=ReviewDecision.ReasonCode.APPROVE_ENTRY,
+        reject_reason=ReviewDecision.ReasonCode.REJECT_ENTRY,
+        rollback_reason=ReviewDecision.ReasonCode.ROLLBACK_ENTRY,
+        limitation_code=PublicationRevision.LimitationCode.ENTRY,
+        source_code=ENTRY_SOURCE_CODE,
+        source_revision=ENTRY_SOURCE_REVISION,
+        source_module=ENTRY_SOURCE_MODULE,
+        source_owner=ENTRY_SOURCE_OWNER,
+        source_locator=ENTRY_SOURCE_LOCATOR,
+        field_scope=ENTRY_FIELD_SCOPE,
+        attribution=ENTRY_ATTRIBUTION,
+        contract_fingerprint=ENTRY_CONTRACT_FINGERPRINT_SHA256,
+        decided_by=ENTRY_DECIDED_BY,
+        decision_basis=ENTRY_DECISION_BASIS,
+        parser_name=ENTRY_PARSER_NAME,
+        parser_version=ENTRY_PARSER_VERSION,
+        schema_fingerprint=ENTRY_SCHEMA_FINGERPRINT_SHA256,
+        access_mode=SourceRightsDecision.AccessMode.ANONYMOUS_PUBLIC,
+        secret_reference_name="",
+        provider_code="",
+    ),
+    PublicationModule.TRAVEL_WARNING: _ModuleSpec(
+        module=PublicationModule.TRAVEL_WARNING,
+        typed_model=TravelWarningRevision,
+        typed_field="travel_warning_revision",
+        pointer_model=PublishedTravelWarning,
+        permission_publish="reviews.publish_warning",
+        permission_reject="reviews.reject_warning",
+        permission_rollback="reviews.rollback_warning",
+        approve_reason=ReviewDecision.ReasonCode.APPROVE_WARNING,
+        reject_reason=ReviewDecision.ReasonCode.REJECT_WARNING,
+        rollback_reason=ReviewDecision.ReasonCode.ROLLBACK_WARNING,
+        limitation_code=PublicationRevision.LimitationCode.TRAVEL_WARNING,
+        source_code=WARNING_SOURCE_CODE,
+        source_revision=WARNING_SOURCE_REVISION,
+        source_module=WARNING_SOURCE_MODULE,
+        source_owner=WARNING_SOURCE_OWNER,
+        source_locator=WARNING_SOURCE_LOCATOR,
+        field_scope=WARNING_FIELD_SCOPE,
+        attribution=WARNING_ATTRIBUTION,
+        contract_fingerprint=PARSER_CONTRACT_FINGERPRINT_SHA256,
+        decided_by=WARNING_DECIDED_BY,
+        decision_basis=WARNING_DECISION_BASIS,
+        parser_name=WARNING_PARSER_NAME,
+        parser_version=WARNING_PARSER_VERSION,
+        schema_fingerprint=EXPECTED_SCHEMA_FINGERPRINT_SHA256,
+        access_mode=SourceRightsDecision.AccessMode.CREDENTIAL_REFERENCE,
+        secret_reference_name="MOFA_TRAVEL_ALARM_SERVICE_KEY",
+        provider_code=FetchAttempt.ProviderCode.MOFA_SUCCESS_0,
+    ),
+}
+
+
+class _ClosedFailure(Exception):
+    def __init__(self, code: str):
+        self.code = code
+        super().__init__(code)
+
+
+class _ProcessControlKind(Enum):
+    KEYBOARD_INTERRUPT = "KEYBOARD_INTERRUPT"
+    SYSTEM_EXIT = "SYSTEM_EXIT"
+    GENERATOR_EXIT = "GENERATOR_EXIT"
+
+
+def _spec(module: str) -> _ModuleSpec:
+    try:
+        return _SPECS[module]
+    except (KeyError, TypeError):
+        raise _ClosedFailure(PublicationCode.INVALID_TARGET) from None
+
+
+def _actor_principal(actor: Any) -> str:
+    try:
+        if not getattr(actor, "is_authenticated", False):
+            return "UNAUTHENTICATED"
+        principal = str(actor.get_username())
+    except Exception:
+        return "AUTHENTICATED_OPERATOR"
+    if not principal or any(ord(character) < 32 for character in principal):
+        return "AUTHENTICATED_OPERATOR"
+    return principal[:150]
+
+
+def _raise_sanitized_process_control(kind: _ProcessControlKind) -> NoReturn:
+    if kind is _ProcessControlKind.KEYBOARD_INTERRUPT:
+        raise KeyboardInterrupt() from None
+    if kind is _ProcessControlKind.SYSTEM_EXIT:
+        raise SystemExit() from None
+    if kind is _ProcessControlKind.GENERATOR_EXIT:
+        raise GeneratorExit() from None
+    raise RuntimeError("invalid process-control sentinel") from None
+
+
+def _lock_authorized_actor(actor: Any, permission: str):
+    try:
+        authenticated = getattr(actor, "is_authenticated", False)
+        actor_id = getattr(actor, "pk", None)
+    except Exception:
+        return None
+    if not authenticated or actor_id is None:
+        return None
+    user_model = get_user_model()
+    try:
+        locked_actor = user_model.objects.select_for_update().get(pk=actor_id)
+    except user_model.DoesNotExist:
+        return None
+    except Exception:
+        raise _ClosedFailure(PublicationCode.TRANSACTION_ABORTED) from None
+    try:
+        if not locked_actor.is_active or not locked_actor.is_staff:
+            return None
+        if locked_actor.is_superuser or locked_actor.has_perm(permission):
+            return locked_actor
+    except Exception:
+        raise _ClosedFailure(PublicationCode.TRANSACTION_ABORTED) from None
+    return None
+
+
+def _lock_audit_actor(actor: Any):
+    try:
+        if not getattr(actor, "is_authenticated", False):
+            return None
+        actor_id = getattr(actor, "pk", None)
+        if actor_id is None:
+            raise _ClosedFailure(PublicationCode.AUDIT_UNAVAILABLE)
+        return get_user_model().objects.select_for_update().get(pk=actor_id)
+    except _ClosedFailure:
+        raise
+    except Exception:
+        raise _ClosedFailure(PublicationCode.AUDIT_UNAVAILABLE) from None
+
+
+def _audit_identity_hash(
+    *,
+    action: str,
+    module: str,
+    entry_id: object | None,
+    warning_id: object | None,
+    review_id: object | None,
+    publication_id: object | None,
+    prior_id: object | None,
+    rollback_target_id: object | None,
+    outcome: str,
+    failure_code: str,
+) -> str:
+    canonical = "\n".join(
+        str(value) if value is not None else ""
+        for value in (
+            action,
+            module,
+            entry_id,
+            warning_id,
+            review_id,
+            publication_id,
+            prior_id,
+            rollback_target_id,
+            outcome,
+            failure_code,
+        )
+    ).encode("utf-8")
+    return hashlib.sha256(canonical).hexdigest()
+
+
+def _subject_values(spec: _ModuleSpec, subject: object | None) -> dict:
+    return {
+        "entry_fact_revision": (
+            subject if spec.module == PublicationModule.ENTRY else None
+        ),
+        "travel_warning_revision": (
+            subject
+            if spec.module == PublicationModule.TRAVEL_WARNING
+            else None
+        ),
+    }
+
+
+def _lock_module(spec: _ModuleSpec) -> None:
+    if connection.vendor != "postgresql":
+        raise _ClosedFailure(PublicationCode.TRANSACTION_ABORTED)
+    with connection.cursor() as cursor:
+        cursor.execute(
+            "SELECT pg_advisory_xact_lock(%s, %s)",
+            [PUBLICATION_LOCK_NAMESPACE, PUBLICATION_LOCK_KEYS[spec.module]],
+        )
+
+
+def _load_subject(spec: _ModuleSpec, subject_id: object):
+    try:
+        return (
+            spec.typed_model.objects.select_for_update()
+            .select_related(
+                "parse_run__artifact__source",
+                "parse_run__artifact__first_successful_attempt",
+            )
+            .get(pk=subject_id)
+        )
+    except Exception:
+        raise _ClosedFailure(PublicationCode.INVALID_TARGET) from None
+
+
+def _latest_rights(source):
+    return (
+        SourceRightsDecision.objects.select_for_update()
+        .filter(source=source, source_revision=source.revision)
+        .order_by("-decision_sequence", "-id")
+        .first()
+    )
+
+
+def _lock_subject_source(spec: _ModuleSpec, subject) -> None:
+    """Lock source and exact-revision rights without judging their state."""
+
+    source = subject.parse_run.artifact.source
+    source = source.__class__.objects.select_for_update().get(pk=source.pk)
+    _latest_rights(source)
+
+
+def _validate_source_chain(spec: _ModuleSpec, subject) -> dict[str, object]:
+    parse_run = subject.parse_run
+    artifact = parse_run.artifact
+    source = artifact.source
+    attempt = artifact.first_successful_attempt
+    source = source.__class__.objects.select_for_update().get(pk=source.pk)
+    rights = _latest_rights(source)
+    if (
+        source.code != spec.source_code
+        or source.revision != spec.source_revision
+        or source.module != spec.source_module
+        or source.owner != spec.source_owner
+        or source.official_locator != spec.source_locator
+        or source.state != source.State.ACTIVE
+        or not source.enabled
+        or source.secret_reference_name != spec.secret_reference_name
+        or source.connect_timeout_seconds != 5
+        or source.read_timeout_seconds != 15
+        or source.max_retries != 2
+        or parse_run.outcome != ParseRun.Outcome.VALIDATED
+        or parse_run.parser_name != spec.parser_name
+        or parse_run.parser_version != spec.parser_version
+        or parse_run.parser_contract_fingerprint_sha256
+        != spec.contract_fingerprint
+        or parse_run.expected_schema_fingerprint_sha256
+        != spec.schema_fingerprint
+        or parse_run.observed_schema_fingerprint_sha256
+        != spec.schema_fingerprint
+        or artifact.state != SourceArtifact.State.REVIEW_REQUIRED
+        or artifact.byte_count < 1
+        or artifact.byte_count
+        > (262_144 if spec.module == PublicationModule.ENTRY else 4_096)
+        or attempt.source_id != source.id
+        or attempt.source_revision != source.revision
+        or attempt.rights_decision_id != getattr(rights, "id", None)
+        or attempt.result != FetchAttempt.Result.SUCCEEDED
+        or attempt.http_status != 200
+        or attempt.provider_code != spec.provider_code
+        or attempt.response_sha256 != artifact.body_sha256
+        or subject.first_observed_at != attempt.completed_at
+    ):
+        raise _ClosedFailure(PublicationCode.SOURCE_GATE_FAILED)
+    if (
+        rights is None
+        or rights.source_revision != spec.source_revision
+        or rights.decision_sequence != 1
+        or rights.decision != SourceRightsDecision.Decision.APPROVED
+        or rights.access_mode != spec.access_mode
+        or not rights.access_allowed
+        or not rights.automated_collection_allowed
+        or not rights.typed_field_storage_allowed
+        or rights.raw_body_storage_allowed
+        or not rights.typed_republication_allowed
+        or rights.raw_retention_seconds != 0
+        or rights.typed_retention
+        != SourceRightsDecision.Retention.PRODUCT_HISTORY
+        or rights.evidence_retention
+        != SourceRightsDecision.Retention.PRODUCT_HISTORY
+        or rights.field_scope_code != spec.field_scope
+        or rights.attribution_text != spec.attribution
+        or rights.contract_fingerprint_sha256 != spec.contract_fingerprint
+        or rights.decided_by != spec.decided_by
+        or rights.decision_basis_code != spec.decision_basis
+    ):
+        raise _ClosedFailure(PublicationCode.SOURCE_GATE_FAILED)
+    return {
+        "source_code_snapshot": source.code,
+        "source_revision": source.revision,
+        "source_owner_snapshot": source.owner,
+        "source_locator_snapshot": source.official_locator,
+        "attribution_text_snapshot": rights.attribution_text,
+        "source_contract_fingerprint_sha256": (
+            rights.contract_fingerprint_sha256
+        ),
+        "parser_name": parse_run.parser_name,
+        "parser_version": parse_run.parser_version,
+        "parser_contract_fingerprint_sha256": (
+            parse_run.parser_contract_fingerprint_sha256
+        ),
+        "schema_fingerprint_sha256": (
+            parse_run.observed_schema_fingerprint_sha256
+        ),
+        "typed_fingerprint_sha256": subject.typed_fingerprint_sha256,
+        "source_first_observed_at": subject.first_observed_at,
+    }
+
+
+def _next_review_sequence(spec: _ModuleSpec, subject) -> int:
+    filters = {spec.typed_field: subject, "module": spec.module}
+    decisions = list(
+        ReviewDecision.objects.select_for_update()
+        .filter(**filters)
+        .order_by("decision_sequence")
+        .only("decision_sequence")
+    )
+    return 1 if not decisions else decisions[-1].decision_sequence + 1
+
+
+def _create_review(
+    *,
+    spec: _ModuleSpec,
+    subject,
+    actor,
+    decision: str,
+    reason_code: str,
+    correlation_id: uuid.UUID,
+) -> ReviewDecision:
+    return ReviewDecision.objects.create(
+        module=spec.module,
+        decision=decision,
+        decision_sequence=_next_review_sequence(spec, subject),
+        reason_code=reason_code,
+        actor=actor,
+        actor_principal=_actor_principal(actor),
+        correlation_id=correlation_id,
+        **_subject_values(spec, subject),
+    )
+
+
+def _pointer(spec: _ModuleSpec):
+    try:
+        return (
+            spec.pointer_model.objects.select_for_update(of=("self",))
+            .select_related("current_publication")
+            .get()
+        )
+    except Exception:
+        raise _ClosedFailure(PublicationCode.TRANSACTION_ABORTED) from None
+
+
+def _pointer_has_subject(pointer, spec: _ModuleSpec, subject) -> bool:
+    current = pointer.current_publication
+    return bool(
+        current is not None
+        and getattr(current, f"{spec.typed_field}_id") == subject.id
+    )
+
+
+def _subject_was_published(spec: _ModuleSpec, subject) -> bool:
+    return PublicationRevision.objects.select_for_update().filter(
+        module=spec.module,
+        **{spec.typed_field: subject},
+    ).exists()
+
+
+def _audit_success(
+    *,
+    action: str,
+    spec: _ModuleSpec,
+    subject,
+    actor,
+    review: ReviewDecision,
+    publication: PublicationRevision | None = None,
+    prior: PublicationRevision | None = None,
+    rollback_target: PublicationRevision | None = None,
+    logical_transaction_id: uuid.UUID,
+) -> AuditEvent:
+    subject_values = _subject_values(spec, subject)
+    return AuditEvent.objects.create(
+        actor=actor,
+        actor_principal=_actor_principal(actor),
+        action=action,
+        module=spec.module,
+        review_decision=review,
+        publication_revision=publication,
+        prior_publication_revision=prior,
+        rollback_target_publication_revision=rollback_target,
+        logical_transaction_id=logical_transaction_id,
+        correlation_id=review.correlation_id,
+        input_identity_sha256=_audit_identity_hash(
+            action=action,
+            module=spec.module,
+            entry_id=getattr(
+                subject_values["entry_fact_revision"], "id", None
+            ),
+            warning_id=getattr(
+                subject_values["travel_warning_revision"], "id", None
+            ),
+            review_id=review.id,
+            publication_id=getattr(publication, "id", None),
+            prior_id=getattr(prior, "id", None),
+            rollback_target_id=getattr(rollback_target, "id", None),
+            outcome=AuditEvent.Outcome.SUCCEEDED,
+            failure_code="",
+        ),
+        outcome=AuditEvent.Outcome.SUCCEEDED,
+        failure_code="",
+        **subject_values,
+    )
+
+
+def _failure_audit(
+    *,
+    spec: _ModuleSpec,
+    actor,
+    failure_code: str,
+    subject_id: object | None,
+    authorization: bool,
+) -> bool:
+    try:
+        with transaction.atomic(durable=True):
+            _lock_module(spec)
+            _pointer(spec)
+            subject = None
+            if subject_id is not None:
+                try:
+                    subject = _load_subject(spec, subject_id)
+                    _lock_subject_source(spec, subject)
+                except _ClosedFailure:
+                    subject = None
+            actor_value = _lock_audit_actor(actor)
+            subject_values = _subject_values(spec, subject)
+            action = (
+                AuditEvent.Action.AUTHORIZATION_FAILURE
+                if authorization
+                else AuditEvent.Action.PUBLICATION_FAILURE
+            )
+            AuditEvent.objects.create(
+                actor=actor_value,
+                actor_principal=_actor_principal(actor),
+                action=action,
+                module=spec.module,
+                logical_transaction_id=uuid.uuid4(),
+                correlation_id=uuid.uuid4(),
+                input_identity_sha256=_audit_identity_hash(
+                    action=action,
+                    module=spec.module,
+                    entry_id=getattr(
+                        subject_values["entry_fact_revision"], "id", None
+                    ),
+                    warning_id=getattr(
+                        subject_values["travel_warning_revision"], "id", None
+                    ),
+                    review_id=None,
+                    publication_id=None,
+                    prior_id=None,
+                    rollback_target_id=None,
+                    outcome=AuditEvent.Outcome.FAILED,
+                    failure_code=failure_code,
+                ),
+                outcome=AuditEvent.Outcome.FAILED,
+                failure_code=failure_code,
+                **subject_values,
+            )
+        return True
+    except Exception:
+        return False
+
+
+def _failure_outcome(
+    *,
+    spec: _ModuleSpec,
+    actor,
+    code: str,
+    subject_id: object | None,
+    authorization: bool = False,
+) -> PublicationOutcome | _ProcessControlKind:
+    audit_code = {
+        PublicationCode.NOT_AUTHORIZED: AuditEvent.FailureCode.NOT_AUTHORIZED,
+        PublicationCode.STALE_POINTER: AuditEvent.FailureCode.STALE_POINTER,
+        PublicationCode.INVALID_TARGET: AuditEvent.FailureCode.INVALID_TARGET,
+        PublicationCode.SOURCE_GATE_FAILED: (
+            AuditEvent.FailureCode.SOURCE_GATE_FAILED
+        ),
+    }.get(code, AuditEvent.FailureCode.TRANSACTION_ABORTED)
+    try:
+        audit_written = _failure_audit(
+            spec=spec,
+            actor=actor,
+            failure_code=audit_code,
+            subject_id=subject_id,
+            authorization=authorization,
+        )
+    except KeyboardInterrupt:
+        return _ProcessControlKind.KEYBOARD_INTERRUPT
+    except SystemExit:
+        return _ProcessControlKind.SYSTEM_EXIT
+    except GeneratorExit:
+        return _ProcessControlKind.GENERATOR_EXIT
+    if not audit_written:
+        return PublicationOutcome(
+            code=PublicationCode.AUDIT_UNAVAILABLE,
+            module=spec.module,
+        )
+    return PublicationOutcome(code=code, module=spec.module)
+
+
+def _publish_candidate_inner(
+    *,
+    module: str,
+    typed_revision_id: object,
+    actor,
+    expected_pointer_version: int,
+) -> PublicationOutcome | _ProcessControlKind:
+    try:
+        spec = _spec(module)
+    except _ClosedFailure:
+        return PublicationOutcome(PublicationCode.INVALID_TARGET, "INVALID")
+    if type(expected_pointer_version) is not int or expected_pointer_version < 0:
+        return _failure_outcome(
+            spec=spec,
+            actor=actor,
+            code=PublicationCode.INVALID_TARGET,
+            subject_id=typed_revision_id,
+        )
+    try:
+        with transaction.atomic(durable=True):
+            _lock_module(spec)
+            pointer = _pointer(spec)
+            if pointer.version != expected_pointer_version:
+                raise _ClosedFailure(PublicationCode.STALE_POINTER)
+            subject = _load_subject(spec, typed_revision_id)
+            snapshots = _validate_source_chain(spec, subject)
+            if _subject_was_published(spec, subject):
+                raise _ClosedFailure(PublicationCode.INVALID_TARGET)
+            locked_actor = _lock_authorized_actor(
+                actor,
+                spec.permission_publish,
+            )
+            if locked_actor is None:
+                raise _ClosedFailure(PublicationCode.NOT_AUTHORIZED)
+            logical_id = uuid.uuid4()
+            review_correlation = uuid.uuid4()
+            review = _create_review(
+                spec=spec,
+                subject=subject,
+                actor=locked_actor,
+                decision=ReviewDecision.Decision.APPROVED,
+                reason_code=spec.approve_reason,
+                correlation_id=review_correlation,
+            )
+            prior = pointer.current_publication
+            now = timezone.now()
+            publication = PublicationRevision.objects.create(
+                module=spec.module,
+                scope_country_id=JP_COUNTRY_ID,
+                scope_passport_policy_id=(
+                    PASSPORT_POLICY_ID
+                    if spec.module == PublicationModule.ENTRY
+                    else None
+                ),
+                review_decision=review,
+                generation=pointer.version + 1,
+                completeness=PublicationRevision.Completeness.LIMITED,
+                limitation_code=spec.limitation_code,
+                state=PublicationRevision.State.PUBLISHED,
+                supersedes=prior,
+                rollback_target=None,
+                created_at=now,
+                published_at=now,
+                **_subject_values(spec, subject),
+                **snapshots,
+            )
+            _audit_success(
+                action=AuditEvent.Action.PUBLISH,
+                spec=spec,
+                subject=subject,
+                actor=locked_actor,
+                review=review,
+                publication=publication,
+                prior=prior,
+                logical_transaction_id=logical_id,
+            )
+            updated = spec.pointer_model.objects.filter(
+                pk=pointer.pk,
+                version=expected_pointer_version,
+            ).update(
+                current_publication=publication,
+                version=F("version") + 1,
+                updated_at=timezone.now(),
+            )
+            if updated != 1:
+                raise _ClosedFailure(PublicationCode.STALE_POINTER)
+            return PublicationOutcome(
+                PublicationCode.PUBLISHED,
+                spec.module,
+                generation=publication.generation,
+                pointer_version=expected_pointer_version + 1,
+            )
+    except _ClosedFailure as failure:
+        return _failure_outcome(
+            spec=spec,
+            actor=actor,
+            code=failure.code,
+            subject_id=typed_revision_id,
+            authorization=failure.code == PublicationCode.NOT_AUTHORIZED,
+        )
+    except KeyboardInterrupt:
+        return _ProcessControlKind.KEYBOARD_INTERRUPT
+    except SystemExit:
+        return _ProcessControlKind.SYSTEM_EXIT
+    except GeneratorExit:
+        return _ProcessControlKind.GENERATOR_EXIT
+    except Exception:
+        return _failure_outcome(
+            spec=spec,
+            actor=actor,
+            code=PublicationCode.TRANSACTION_ABORTED,
+            subject_id=typed_revision_id,
+        )
+
+
+def _reject_candidate_inner(
+    *,
+    module: str,
+    typed_revision_id: object,
+    actor,
+) -> PublicationOutcome | _ProcessControlKind:
+    try:
+        spec = _spec(module)
+    except _ClosedFailure:
+        return PublicationOutcome(PublicationCode.INVALID_TARGET, "INVALID")
+    try:
+        with transaction.atomic(durable=True):
+            _lock_module(spec)
+            pointer = _pointer(spec)
+            subject = _load_subject(spec, typed_revision_id)
+            _lock_subject_source(spec, subject)
+            if _pointer_has_subject(pointer, spec, subject):
+                raise _ClosedFailure(PublicationCode.INVALID_TARGET)
+            locked_actor = _lock_authorized_actor(
+                actor,
+                spec.permission_reject,
+            )
+            if locked_actor is None:
+                raise _ClosedFailure(PublicationCode.NOT_AUTHORIZED)
+            logical_id = uuid.uuid4()
+            review = _create_review(
+                spec=spec,
+                subject=subject,
+                actor=locked_actor,
+                decision=ReviewDecision.Decision.REJECTED,
+                reason_code=spec.reject_reason,
+                correlation_id=uuid.uuid4(),
+            )
+            _audit_success(
+                action=AuditEvent.Action.REVIEW_REJECT,
+                spec=spec,
+                subject=subject,
+                actor=locked_actor,
+                review=review,
+                logical_transaction_id=logical_id,
+            )
+            return PublicationOutcome(PublicationCode.REJECTED, spec.module)
+    except _ClosedFailure as failure:
+        return _failure_outcome(
+            spec=spec,
+            actor=actor,
+            code=failure.code,
+            subject_id=typed_revision_id,
+            authorization=failure.code == PublicationCode.NOT_AUTHORIZED,
+        )
+    except KeyboardInterrupt:
+        return _ProcessControlKind.KEYBOARD_INTERRUPT
+    except SystemExit:
+        return _ProcessControlKind.SYSTEM_EXIT
+    except GeneratorExit:
+        return _ProcessControlKind.GENERATOR_EXIT
+    except Exception:
+        return _failure_outcome(
+            spec=spec,
+            actor=actor,
+            code=PublicationCode.TRANSACTION_ABORTED,
+            subject_id=typed_revision_id,
+        )
+
+
+def _rollback_publication_inner(
+    *,
+    module: str,
+    target_publication_id: object,
+    actor,
+    expected_pointer_version: int,
+) -> PublicationOutcome | _ProcessControlKind:
+    try:
+        spec = _spec(module)
+    except _ClosedFailure:
+        return PublicationOutcome(PublicationCode.INVALID_TARGET, "INVALID")
+    if type(expected_pointer_version) is not int or expected_pointer_version < 1:
+        return _failure_outcome(
+            spec=spec,
+            actor=actor,
+            code=PublicationCode.INVALID_TARGET,
+            subject_id=None,
+        )
+    try:
+        with transaction.atomic(durable=True):
+            _lock_module(spec)
+            pointer = _pointer(spec)
+            if pointer.version != expected_pointer_version:
+                raise _ClosedFailure(PublicationCode.STALE_POINTER)
+            current = pointer.current_publication
+            if current is None:
+                raise _ClosedFailure(PublicationCode.INVALID_TARGET)
+            try:
+                target = PublicationRevision.objects.select_for_update().get(
+                    pk=target_publication_id,
+                    module=spec.module,
+                )
+            except Exception:
+                raise _ClosedFailure(PublicationCode.INVALID_TARGET) from None
+            if (
+                target.id == current.id
+                or target.generation >= current.generation
+                or target.scope_country_id != current.scope_country_id
+                or target.scope_passport_policy_id
+                != current.scope_passport_policy_id
+            ):
+                raise _ClosedFailure(PublicationCode.INVALID_TARGET)
+            subject = _load_subject(
+                spec,
+                getattr(target, f"{spec.typed_field}_id"),
+            )
+            snapshots = _validate_source_chain(spec, subject)
+            locked_actor = _lock_authorized_actor(
+                actor,
+                spec.permission_rollback,
+            )
+            if locked_actor is None:
+                raise _ClosedFailure(PublicationCode.NOT_AUTHORIZED)
+            review = _create_review(
+                spec=spec,
+                subject=subject,
+                actor=locked_actor,
+                decision=ReviewDecision.Decision.APPROVED,
+                reason_code=spec.rollback_reason,
+                correlation_id=uuid.uuid4(),
+            )
+            now = timezone.now()
+            publication = PublicationRevision.objects.create(
+                module=spec.module,
+                scope_country_id=JP_COUNTRY_ID,
+                scope_passport_policy_id=(
+                    PASSPORT_POLICY_ID
+                    if spec.module == PublicationModule.ENTRY
+                    else None
+                ),
+                review_decision=review,
+                generation=pointer.version + 1,
+                completeness=PublicationRevision.Completeness.LIMITED,
+                limitation_code=spec.limitation_code,
+                state=PublicationRevision.State.PUBLISHED,
+                supersedes=current,
+                rollback_target=target,
+                created_at=now,
+                published_at=now,
+                **_subject_values(spec, subject),
+                **snapshots,
+            )
+            _audit_success(
+                action=AuditEvent.Action.ROLLBACK,
+                spec=spec,
+                subject=subject,
+                actor=locked_actor,
+                review=review,
+                publication=publication,
+                prior=current,
+                rollback_target=target,
+                logical_transaction_id=uuid.uuid4(),
+            )
+            updated = spec.pointer_model.objects.filter(
+                pk=pointer.pk,
+                version=expected_pointer_version,
+            ).update(
+                current_publication=publication,
+                version=F("version") + 1,
+                updated_at=timezone.now(),
+            )
+            if updated != 1:
+                raise _ClosedFailure(PublicationCode.STALE_POINTER)
+            return PublicationOutcome(
+                PublicationCode.ROLLED_BACK,
+                spec.module,
+                generation=publication.generation,
+                pointer_version=expected_pointer_version + 1,
+            )
+    except _ClosedFailure as failure:
+        return _failure_outcome(
+            spec=spec,
+            actor=actor,
+            code=failure.code,
+            subject_id=None,
+            authorization=failure.code == PublicationCode.NOT_AUTHORIZED,
+        )
+    except KeyboardInterrupt:
+        return _ProcessControlKind.KEYBOARD_INTERRUPT
+    except SystemExit:
+        return _ProcessControlKind.SYSTEM_EXIT
+    except GeneratorExit:
+        return _ProcessControlKind.GENERATOR_EXIT
+    except Exception:
+        return _failure_outcome(
+            spec=spec,
+            actor=actor,
+            code=PublicationCode.TRANSACTION_ABORTED,
+            subject_id=None,
+        )
+
+
+def publish_candidate(
+    *,
+    module: str,
+    typed_revision_id: object,
+    actor,
+    expected_pointer_version: int,
+) -> PublicationOutcome:
+    result = _publish_candidate_inner(
+        module=module,
+        typed_revision_id=typed_revision_id,
+        actor=actor,
+        expected_pointer_version=expected_pointer_version,
+    )
+    if isinstance(result, _ProcessControlKind):
+        _raise_sanitized_process_control(result)
+    return result
+
+
+def reject_candidate(
+    *,
+    module: str,
+    typed_revision_id: object,
+    actor,
+) -> PublicationOutcome:
+    result = _reject_candidate_inner(
+        module=module,
+        typed_revision_id=typed_revision_id,
+        actor=actor,
+    )
+    if isinstance(result, _ProcessControlKind):
+        _raise_sanitized_process_control(result)
+    return result
+
+
+def rollback_publication(
+    *,
+    module: str,
+    target_publication_id: object,
+    actor,
+    expected_pointer_version: int,
+) -> PublicationOutcome:
+    result = _rollback_publication_inner(
+        module=module,
+        target_publication_id=target_publication_id,
+        actor=actor,
+        expected_pointer_version=expected_pointer_version,
+    )
+    if isinstance(result, _ProcessControlKind):
+        _raise_sanitized_process_control(result)
+    return result
diff --git a/reviews/tests/__init__.py b/reviews/tests/__init__.py
new file mode 100644
index 0000000..8b13789
--- /dev/null
+++ b/reviews/tests/__init__.py
@@ -0,0 +1 @@
+
diff --git a/reviews/tests/test_publication.py b/reviews/tests/test_publication.py
new file mode 100644
index 0000000..0cc25e9
--- /dev/null
+++ b/reviews/tests/test_publication.py
@@ -0,0 +1,1580 @@
+import uuid
+from concurrent.futures import ThreadPoolExecutor
+from importlib import import_module
+from threading import Barrier
+from types import SimpleNamespace
+from unittest.mock import patch
+
+import psycopg
+from django.apps import apps as django_apps
+from django.contrib.auth import get_user_model
+from django.contrib.auth.models import Permission
+from django.db import (
+    DatabaseError,
+    IntegrityError,
+    close_old_connections,
+    connection,
+    transaction,
+)
+from django.db.migrations.executor import MigrationExecutor
+from django.test import SimpleTestCase, TransactionTestCase
+from django.utils import timezone
+
+from countries.models import JP_COUNTRY_ID, Country
+from entry_requirements.ingestion import (
+    EntryIngestionCode,
+    ingest_entry_snapshot,
+)
+from entry_requirements.models import (
+    PASSPORT_POLICY_CODE,
+    PASSPORT_POLICY_ID,
+    PASSPORT_POLICY_REVISION,
+    EntryFactRevision,
+    PassportPolicy,
+)
+from entry_requirements.tests.test_ingestion import (
+    entry_payload,
+    successful_result as successful_entry_result,
+)
+from operations.tests.migration_helpers import (
+    restore_canonical_migration_graph,
+)
+from sources.management.commands.register_approved_sources import (
+    register_approved_sources,
+)
+from sources.models import SourceConfiguration, SourceRightsDecision
+from travel_warnings.ingestion import (
+    TravelWarningIngestionCode,
+    ingest_travel_warning,
+)
+from travel_warnings.models import TravelWarningRevision
+from travel_warnings.tests.test_ingestion import (
+    successful_result as successful_warning_result,
+    warning_item,
+    warning_payload,
+)
+
+from reviews.models import (
+    ENTRY_POINTER_ID,
+    WARNING_POINTER_ID,
+    AuditEvent,
+    PublicationModule,
+    PublicationRevision,
+    PublishedEntryFacts,
+    PublishedTravelWarning,
+    ReviewDecision,
+)
+from reviews.publication import (
+    PUBLICATION_LOCK_KEYS,
+    PUBLICATION_LOCK_NAMESPACE,
+    PublicationCode,
+    _SPECS,
+    _audit_identity_hash,
+    _lock_module,
+    publish_candidate,
+    reject_candidate,
+    rollback_publication,
+)
+
+
+migration_module = import_module("reviews.migrations.0001_initial")
+
+
+class PublicationFixtureMixin:
+    def seed_boundaries(self):
+        Country.objects.get_or_create(
+            id=JP_COUNTRY_ID,
+            defaults={
+                "iso_alpha2": "JP",
+                "name_ko": "일본",
+                "name_en": "Japan",
+                "is_public": True,
+            },
+        )
+        PassportPolicy.objects.get_or_create(
+            id=PASSPORT_POLICY_ID,
+            defaults={
+                "code": PASSPORT_POLICY_CODE,
+                "revision": PASSPORT_POLICY_REVISION,
+            },
+        )
+        PublishedEntryFacts.objects.get_or_create(
+            id=ENTRY_POINTER_ID,
+            defaults={
+                "country_id": JP_COUNTRY_ID,
+                "passport_policy_id": PASSPORT_POLICY_ID,
+                "version": 0,
+            },
+        )
+        PublishedTravelWarning.objects.get_or_create(
+            id=WARNING_POINTER_ID,
+            defaults={"country_id": JP_COUNTRY_ID, "version": 0},
+        )
+        register_approved_sources(apply=True)
+
+    def make_entry(self, *, period="90일"):
+        payload = entry_payload(period=period)
+        outcome = ingest_entry_snapshot(
+            transport=lambda **_kwargs: successful_entry_result(payload),
+            retry_wait=lambda _attempt: None,
+        )
+        self.assertEqual(outcome.code, EntryIngestionCode.REVIEW_REQUIRED)
+        return EntryFactRevision.objects.latest("created_at")
+
+    def make_warning(self, *, scope_text="합성 검증 범위"):
+        payload = warning_payload(item=warning_item(remark=scope_text))
+        outcome = ingest_travel_warning(
+            transport=lambda **_kwargs: successful_warning_result(payload),
+            secret_loader=lambda _reference: "synthetic-test-credential",
+            retry_wait=lambda _attempt: None,
+        )
+        self.assertEqual(
+            outcome.code,
+            TravelWarningIngestionCode.REVIEW_REQUIRED,
+        )
+        return TravelWarningRevision.objects.latest("created_at")
+
+    def reject_current_rights(self, source_code):
+        source = SourceConfiguration.objects.get(code=source_code)
+        approval = source.rights_decisions.get(decision_sequence=1)
+        return SourceRightsDecision.objects.create(
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
+            evidence_retention=(
+                SourceRightsDecision.Retention.PRODUCT_HISTORY
+            ),
+            field_scope_code="",
+            attribution_text="",
+            contract_fingerprint_sha256=(
+                approval.contract_fingerprint_sha256
+            ),
+            decided_by="PROJECT_OWNER_REQUEST",
+            decision_basis_code="SYNTHETIC_REJECTION_TEST",
+        )
+
+    def review_values(
+        self,
+        subject,
+        *,
+        actor=None,
+        decision=ReviewDecision.Decision.APPROVED,
+        reason=ReviewDecision.ReasonCode.APPROVE_ENTRY,
+        sequence=1,
+        correlation_id=None,
+    ):
+        actor = actor or self.actor
+        is_entry = isinstance(subject, EntryFactRevision)
+        return {
+            "module": (
+                PublicationModule.ENTRY
+                if is_entry
+                else PublicationModule.TRAVEL_WARNING
+            ),
+            "entry_fact_revision": subject if is_entry else None,
+            "travel_warning_revision": None if is_entry else subject,
+            "decision": decision,
+            "decision_sequence": sequence,
+            "reason_code": reason,
+            "actor": actor,
+            "actor_principal": actor.username,
+            "correlation_id": correlation_id or uuid.uuid4(),
+        }
+
+    def publication_values(
+        self,
+        subject,
+        review,
+        *,
+        generation,
+        supersedes=None,
+        rollback_target=None,
+    ):
+        parse_run = subject.parse_run
+        artifact = parse_run.artifact
+        source = artifact.source
+        rights = artifact.first_successful_attempt.rights_decision
+        is_entry = isinstance(subject, EntryFactRevision)
+        now = timezone.now()
+        return {
+            "module": (
+                PublicationModule.ENTRY
+                if is_entry
+                else PublicationModule.TRAVEL_WARNING
+            ),
+            "scope_country_id": JP_COUNTRY_ID,
+            "scope_passport_policy_id": (
+                PASSPORT_POLICY_ID if is_entry else None
+            ),
+            "entry_fact_revision": subject if is_entry else None,
+            "travel_warning_revision": None if is_entry else subject,
+            "review_decision": review,
+            "generation": generation,
+            "limitation_code": (
+                PublicationRevision.LimitationCode.ENTRY
+                if is_entry
+                else PublicationRevision.LimitationCode.TRAVEL_WARNING
+            ),
+            "supersedes": supersedes,
+            "rollback_target": rollback_target,
+            "source_code_snapshot": source.code,
+            "source_revision": source.revision,
+            "source_owner_snapshot": source.owner,
+            "source_locator_snapshot": source.official_locator,
+            "attribution_text_snapshot": rights.attribution_text,
+            "source_contract_fingerprint_sha256": (
+                rights.contract_fingerprint_sha256
+            ),
+            "parser_name": parse_run.parser_name,
+            "parser_version": parse_run.parser_version,
+            "parser_contract_fingerprint_sha256": (
+                parse_run.parser_contract_fingerprint_sha256
+            ),
+            "schema_fingerprint_sha256": (
+                parse_run.observed_schema_fingerprint_sha256
+            ),
+            "typed_fingerprint_sha256": subject.typed_fingerprint_sha256,
+            "source_first_observed_at": subject.first_observed_at,
+            "created_at": now,
+            "published_at": now,
+        }
+
+    def audit_values(
+        self,
+        *,
+        action,
+        subject,
+        review,
+        publication=None,
+        prior=None,
+        rollback_target=None,
+        correlation_id=None,
+    ):
+        is_entry = isinstance(subject, EntryFactRevision)
+        module = (
+            PublicationModule.ENTRY
+            if is_entry
+            else PublicationModule.TRAVEL_WARNING
+        )
+        return {
+            "actor": review.actor,
+            "actor_principal": review.actor_principal,
+            "action": action,
+            "module": module,
+            "entry_fact_revision": subject if is_entry else None,
+            "travel_warning_revision": None if is_entry else subject,
+            "review_decision": review,
+            "publication_revision": publication,
+            "prior_publication_revision": prior,
+            "rollback_target_publication_revision": rollback_target,
+            "logical_transaction_id": uuid.uuid4(),
+            "correlation_id": correlation_id or review.correlation_id,
+            "input_identity_sha256": _audit_identity_hash(
+                action=action,
+                module=module,
+                entry_id=subject.id if is_entry else None,
+                warning_id=None if is_entry else subject.id,
+                review_id=review.id,
+                publication_id=getattr(publication, "id", None),
+                prior_id=getattr(prior, "id", None),
+                rollback_target_id=getattr(rollback_target, "id", None),
+                outcome=AuditEvent.Outcome.SUCCEEDED,
+                failure_code="",
+            ),
+            "outcome": AuditEvent.Outcome.SUCCEEDED,
+            "failure_code": "",
+        }
+
+    def force_deferred_checks(self):
+        with connection.cursor() as cursor:
+            cursor.execute("SET CONSTRAINTS ALL IMMEDIATE")
+
+    def actor_with_permission(self, username, codename):
+        actor = get_user_model().objects.create_user(
+            username=username,
+            password=None,
+            is_staff=True,
+        )
+        actor.user_permissions.add(
+            Permission.objects.get(
+                content_type__app_label="reviews",
+                codename=codename,
+            )
+        )
+        return actor
+
+
+class PublicationServiceTests(PublicationFixtureMixin, TransactionTestCase):
+    reset_sequences = True
+
+    def setUp(self):
+        self.seed_boundaries()
+        self.actor = get_user_model().objects.create_superuser(
+            username="publication-operator",
+            password=None,
+        )
+
+    def test_entry_and_warning_publish_as_independent_boundaries(self):
+        entry = self.make_entry()
+        warning = self.make_warning()
+
+        entry_outcome = publish_candidate(
+            module=PublicationModule.ENTRY,
+            typed_revision_id=entry.id,
+            actor=self.actor,
+            expected_pointer_version=0,
+        )
+        warning_outcome = publish_candidate(
+            module=PublicationModule.TRAVEL_WARNING,
+            typed_revision_id=warning.id,
+            actor=self.actor,
+            expected_pointer_version=0,
+        )
+
+        self.assertEqual(entry_outcome.code, PublicationCode.PUBLISHED)
+        self.assertEqual(warning_outcome.code, PublicationCode.PUBLISHED)
+        entry_pointer = PublishedEntryFacts.objects.select_related(
+            "current_publication"
+        ).get()
+        warning_pointer = PublishedTravelWarning.objects.select_related(
+            "current_publication"
+        ).get()
+        self.assertEqual(entry_pointer.version, 1)
+        self.assertEqual(warning_pointer.version, 1)
+        self.assertEqual(
+            entry_pointer.current_publication.entry_fact_revision_id,
+            entry.id,
+        )
+        self.assertIsNone(
+            entry_pointer.current_publication.travel_warning_revision_id
+        )
+        self.assertEqual(
+            warning_pointer.current_publication.travel_warning_revision_id,
+            warning.id,
+        )
+        self.assertIsNone(
+            warning_pointer.current_publication.entry_fact_revision_id
+        )
+        self.assertEqual(PublicationRevision.objects.count(), 2)
+
+    def test_reject_is_append_only_review_evidence_without_publication(self):
+        entry = self.make_entry()
+        warning = self.make_warning()
+
+        entry_outcome = reject_candidate(
+            module=PublicationModule.ENTRY,
+            typed_revision_id=entry.id,
+            actor=self.actor,
+        )
+        warning_outcome = reject_candidate(
+            module=PublicationModule.TRAVEL_WARNING,
+            typed_revision_id=warning.id,
+            actor=self.actor,
+        )
+
+        self.assertEqual(entry_outcome.code, PublicationCode.REJECTED)
+        self.assertEqual(warning_outcome.code, PublicationCode.REJECTED)
+        reviews = ReviewDecision.objects.order_by("module")
+        self.assertEqual(reviews.count(), 2)
+        self.assertTrue(
+            all(
+                review.decision == ReviewDecision.Decision.REJECTED
+                and review.decision_sequence == 1
+                for review in reviews
+            )
+        )
+        self.assertEqual(PublicationRevision.objects.count(), 0)
+        self.assertEqual(PublishedEntryFacts.objects.get().version, 0)
+        self.assertEqual(PublishedTravelWarning.objects.get().version, 0)
+        audits = AuditEvent.objects.all()
+        self.assertEqual(audits.count(), 2)
+        self.assertTrue(
+            all(
+                audit.action == AuditEvent.Action.REVIEW_REJECT
+                and audit.outcome == AuditEvent.Outcome.SUCCEEDED
+                for audit in audits
+            )
+        )
+
+    def test_db_trigger_requires_the_latest_review_to_be_approved(self):
+        entry = self.make_entry()
+        with self.assertRaises(DatabaseError), transaction.atomic():
+            approved = ReviewDecision.objects.create(
+                **self.review_values(entry)
+            )
+            rejected = ReviewDecision.objects.create(
+                **self.review_values(
+                    entry,
+                    decision=ReviewDecision.Decision.REJECTED,
+                    reason=ReviewDecision.ReasonCode.REJECT_ENTRY,
+                    sequence=2,
+                )
+            )
+            AuditEvent.objects.create(
+                **self.audit_values(
+                    action=AuditEvent.Action.REVIEW_REJECT,
+                    subject=entry,
+                    review=rejected,
+                )
+            )
+            PublicationRevision.objects.create(
+                **self.publication_values(
+                    entry,
+                    approved,
+                    generation=1,
+                )
+            )
+        self.assertFalse(PublicationRevision.objects.exists())
+        self.assertFalse(ReviewDecision.objects.exists())
+
+    def test_publish_is_latest_approved_review_and_snapshots_ssr_evidence(self):
+        entry = self.make_entry()
+        reject_candidate(
+            module=PublicationModule.ENTRY,
+            typed_revision_id=entry.id,
+            actor=self.actor,
+        )
+
+        outcome = publish_candidate(
+            module=PublicationModule.ENTRY,
+            typed_revision_id=entry.id,
+            actor=self.actor,
+            expected_pointer_version=0,
+        )
+
+        self.assertEqual(outcome.code, PublicationCode.PUBLISHED)
+        reviews = list(
+            ReviewDecision.objects.filter(entry_fact_revision=entry).order_by(
+                "decision_sequence"
+            )
+        )
+        self.assertEqual(
+            [review.decision for review in reviews],
+            [ReviewDecision.Decision.REJECTED, ReviewDecision.Decision.APPROVED],
+        )
+        publication = PublicationRevision.objects.get()
+        self.assertEqual(publication.review_decision_id, reviews[-1].id)
+        self.assertEqual(publication.source_code_snapshot, "MOFA_ENTRY_CSV")
+        self.assertTrue(publication.source_locator_snapshot.startswith("https://"))
+        self.assertEqual(
+            publication.attribution_text_snapshot,
+            "외교부|공공데이터포털",
+        )
+        self.assertEqual(
+            publication.source_first_observed_at,
+            entry.first_observed_at,
+        )
+        self.assertGreaterEqual(
+            publication.published_at,
+            publication.source_first_observed_at,
+        )
+
+    def test_unauthorized_actor_is_fixed_failure_and_db_bypass_is_blocked(self):
+        entry = self.make_entry()
+        unauthorized = get_user_model().objects.create_user(
+            username="staff-without-publication-permission",
+            password=None,
+            is_staff=True,
+        )
+
+        outcome = publish_candidate(
+            module=PublicationModule.ENTRY,
+            typed_revision_id=entry.id,
+            actor=unauthorized,
+            expected_pointer_version=0,
+        )
+
+        self.assertEqual(outcome.code, PublicationCode.NOT_AUTHORIZED)
+        self.assertEqual(PublicationRevision.objects.count(), 0)
+        self.assertEqual(ReviewDecision.objects.count(), 0)
+        failures = AuditEvent.objects.filter(outcome=AuditEvent.Outcome.FAILED)
+        self.assertEqual(failures.count(), 1)
+        self.assertEqual(
+            failures.get().failure_code, AuditEvent.FailureCode.NOT_AUTHORIZED
+        )
+        with self.assertRaises(DatabaseError), transaction.atomic():
+            ReviewDecision.objects.create(
+                module=PublicationModule.ENTRY,
+                entry_fact_revision=entry,
+                decision=ReviewDecision.Decision.APPROVED,
+                decision_sequence=1,
+                reason_code=ReviewDecision.ReasonCode.APPROVE_ENTRY,
+                actor=unauthorized,
+                actor_principal=unauthorized.username,
+                correlation_id=uuid.uuid4(),
+            )
+
+    def test_inactive_source_and_latest_rights_rejection_fail_closed(self):
+        entry = self.make_entry()
+        self.reject_current_rights("MOFA_ENTRY_CSV")
+
+        rights_outcome = publish_candidate(
+            module=PublicationModule.ENTRY,
+            typed_revision_id=entry.id,
+            actor=self.actor,
+            expected_pointer_version=0,
+        )
+
+        self.assertEqual(
+            rights_outcome.code,
+            PublicationCode.SOURCE_GATE_FAILED,
+        )
+        self.assertEqual(PublicationRevision.objects.count(), 0)
+        self.assertEqual(ReviewDecision.objects.count(), 0)
+
+        warning = self.make_warning()
+        warning_source = SourceConfiguration.objects.get(
+            code="MOFA_TRAVEL_ALARM_API_JP"
+        )
+        warning_source.state = SourceConfiguration.State.PAUSED
+        warning_source.enabled = False
+        warning_source.save(update_fields=("state", "enabled"))
+        source_outcome = publish_candidate(
+            module=PublicationModule.TRAVEL_WARNING,
+            typed_revision_id=warning.id,
+            actor=self.actor,
+            expected_pointer_version=0,
+        )
+        self.assertEqual(
+            source_outcome.code,
+            PublicationCode.SOURCE_GATE_FAILED,
+        )
+        self.assertEqual(PublishedTravelWarning.objects.get().version, 0)
+        self.assertEqual(
+            AuditEvent.objects.filter(
+                outcome=AuditEvent.Outcome.FAILED,
+                failure_code=AuditEvent.FailureCode.SOURCE_GATE_FAILED,
+            ).count(),
+            2,
+        )
+
+    def test_cross_module_subjects_and_shapes_are_rejected(self):
+        entry = self.make_entry()
+        warning = self.make_warning()
+
+        outcome = publish_candidate(
+            module=PublicationModule.ENTRY,
+            typed_revision_id=warning.id,
+            actor=self.actor,
+            expected_pointer_version=0,
+        )
+
+        self.assertEqual(outcome.code, PublicationCode.INVALID_TARGET)
+        with self.assertRaises((DatabaseError, IntegrityError)), transaction.atomic():
+            ReviewDecision.objects.create(
+                module=PublicationModule.ENTRY,
+                travel_warning_revision=warning,
+                decision=ReviewDecision.Decision.APPROVED,
+                decision_sequence=1,
+                reason_code=ReviewDecision.ReasonCode.APPROVE_ENTRY,
+                actor=self.actor,
+                actor_principal=self.actor.username,
+                correlation_id=uuid.uuid4(),
+            )
+        self.assertFalse(
+            ReviewDecision.objects.filter(entry_fact_revision=entry).exists()
+        )
+
+    def test_stale_cas_does_not_change_current_pointer(self):
+        first = self.make_entry(period="90일")
+        second = self.make_entry(period="30일")
+        first_outcome = publish_candidate(
+            module=PublicationModule.ENTRY,
+            typed_revision_id=first.id,
+            actor=self.actor,
+            expected_pointer_version=0,
+        )
+        self.assertEqual(first_outcome.code, PublicationCode.PUBLISHED)
+        before = PublishedEntryFacts.objects.get()
+
+        stale = publish_candidate(
+            module=PublicationModule.ENTRY,
+            typed_revision_id=second.id,
+            actor=self.actor,
+            expected_pointer_version=0,
+        )
+
+        self.assertEqual(stale.code, PublicationCode.STALE_POINTER)
+        after = PublishedEntryFacts.objects.get()
+        self.assertEqual(after.version, 1)
+        self.assertEqual(
+            after.current_publication_id,
+            before.current_publication_id,
+        )
+        self.assertEqual(PublicationRevision.objects.count(), 1)
+        self.assertEqual(
+            AuditEvent.objects.filter(
+                outcome=AuditEvent.Outcome.FAILED,
+                failure_code=AuditEvent.FailureCode.STALE_POINTER,
+            ).count(),
+            1,
+        )
+
+    def test_module_advisory_lock_serializes_a_second_database_session(self):
+        contender = psycopg.connect(**connection.get_connection_params())
+        try:
+            with transaction.atomic():
+                _lock_module(_SPECS[PublicationModule.ENTRY])
+                with contender.cursor() as cursor:
+                    cursor.execute(
+                        "SELECT pg_try_advisory_xact_lock(%s, %s)",
+                        [
+                            PUBLICATION_LOCK_NAMESPACE,
+                            PUBLICATION_LOCK_KEYS[PublicationModule.ENTRY],
+                        ],
+                    )
+                    self.assertIs(cursor.fetchone()[0], False)
+                    cursor.execute(
+                        "SELECT pg_try_advisory_xact_lock(%s, %s)",
+                        [
+                            PUBLICATION_LOCK_NAMESPACE,
+                            PUBLICATION_LOCK_KEYS[
+                                PublicationModule.TRAVEL_WARNING
+                            ],
+                        ],
+                    )
+                    self.assertIs(cursor.fetchone()[0], True)
+                contender.rollback()
+            with contender.cursor() as cursor:
+                cursor.execute(
+                    "SELECT pg_try_advisory_xact_lock(%s, %s)",
+                    [
+                        PUBLICATION_LOCK_NAMESPACE,
+                        PUBLICATION_LOCK_KEYS[PublicationModule.ENTRY],
+                    ],
+                )
+                self.assertIs(cursor.fetchone()[0], True)
+            contender.rollback()
+        finally:
+            contender.close()
+
+    def test_failure_audit_unavailability_is_an_explicit_fixed_outcome(self):
+        entry = self.make_entry()
+
+        with patch(
+            "reviews.publication.AuditEvent.objects.create",
+            side_effect=RuntimeError("synthetic audit store failure"),
+        ):
+            outcome = publish_candidate(
+                module=PublicationModule.ENTRY,
+                typed_revision_id=entry.id,
+                actor=self.actor,
+                expected_pointer_version=-1,
+            )
+
+        self.assertEqual(outcome.code, PublicationCode.AUDIT_UNAVAILABLE)
+        self.assertFalse(AuditEvent.objects.exists())
+        self.assertFalse(ReviewDecision.objects.exists())
+        self.assertFalse(PublicationRevision.objects.exists())
+
+    def test_process_control_exceptions_are_sanitized_and_not_outcomes(self):
+        entry = self.make_entry()
+
+        def assert_redacted_process_control(exception, marker):
+            self.assertEqual(exception.args, ())
+            self.assertIsNone(exception.__cause__)
+            self.assertIsNone(exception.__context__)
+            self.assertNotIn(marker, repr(exception))
+            traceback = exception.__traceback__
+            while traceback is not None:
+                frame = traceback.tb_frame
+                if frame.f_code.co_filename.endswith("reviews/publication.py"):
+                    for value in frame.f_locals.values():
+                        self.assertNotIn(marker, repr(value))
+                traceback = traceback.tb_next
+
+        with patch(
+            "reviews.publication._audit_success",
+            side_effect=KeyboardInterrupt("sensitive-marker"),
+        ):
+            with self.assertRaises(KeyboardInterrupt) as interrupted:
+                publish_candidate(
+                    module=PublicationModule.ENTRY,
+                    typed_revision_id=entry.id,
+                    actor=self.actor,
+                    expected_pointer_version=0,
+                )
+        assert_redacted_process_control(
+            interrupted.exception,
+            "sensitive-marker",
+        )
+        self.assertFalse(ReviewDecision.objects.exists())
+        self.assertFalse(PublicationRevision.objects.exists())
+        self.assertFalse(AuditEvent.objects.exists())
+        self.assertEqual(PublishedEntryFacts.objects.get().version, 0)
+
+        with patch(
+            "reviews.publication._failure_audit",
+            side_effect=SystemExit("second-sensitive-marker"),
+        ):
+            with self.assertRaises(SystemExit) as exited:
+                publish_candidate(
+                    module=PublicationModule.ENTRY,
+                    typed_revision_id=entry.id,
+                    actor=self.actor,
+                    expected_pointer_version=-1,
+                )
+        assert_redacted_process_control(
+            exited.exception,
+            "second-sensitive-marker",
+        )
+        self.assertFalse(AuditEvent.objects.exists())
+
+    def test_current_publication_subject_cannot_be_rejected(self):
+        entry = self.make_entry()
+        published = publish_candidate(
+            module=PublicationModule.ENTRY,
+            typed_revision_id=entry.id,
+            actor=self.actor,
+            expected_pointer_version=0,
+        )
+        self.assertEqual(published.code, PublicationCode.PUBLISHED)
+
+        rejected = reject_candidate(
+            module=PublicationModule.ENTRY,
+            typed_revision_id=entry.id,
+            actor=self.actor,
+        )
+
+        self.assertEqual(rejected.code, PublicationCode.INVALID_TARGET)
+        self.assertEqual(
+            ReviewDecision.objects.filter(
+                entry_fact_revision=entry,
+                decision=ReviewDecision.Decision.REJECTED,
+            ).count(),
+            0,
+        )
+        self.assertEqual(
+            AuditEvent.objects.filter(
+                outcome=AuditEvent.Outcome.FAILED,
+                failure_code=AuditEvent.FailureCode.INVALID_TARGET,
+            ).count(),
+            1,
+        )
+        with self.assertRaises(DatabaseError), transaction.atomic():
+            ReviewDecision.objects.create(
+                **self.review_values(
+                    entry,
+                    decision=ReviewDecision.Decision.REJECTED,
+                    reason=ReviewDecision.ReasonCode.REJECT_ENTRY,
+                    sequence=2,
+                )
+            )
+
+    def test_historical_subject_requires_explicit_rollback(self):
+        first = self.make_entry(period="90일")
+        second = self.make_entry(period="30일")
+        self.assertEqual(
+            publish_candidate(
+                module=PublicationModule.ENTRY,
+                typed_revision_id=first.id,
+                actor=self.actor,
+                expected_pointer_version=0,
+            ).code,
+            PublicationCode.PUBLISHED,
+        )
+        self.assertEqual(
+            publish_candidate(
+                module=PublicationModule.ENTRY,
+                typed_revision_id=second.id,
+                actor=self.actor,
+                expected_pointer_version=1,
+            ).code,
+            PublicationCode.PUBLISHED,
+        )
+
+        bypass = publish_candidate(
+            module=PublicationModule.ENTRY,
+            typed_revision_id=first.id,
+            actor=self.actor,
+            expected_pointer_version=2,
+        )
+
+        self.assertEqual(bypass.code, PublicationCode.INVALID_TARGET)
+        pointer = PublishedEntryFacts.objects.get()
+        self.assertEqual(pointer.version, 2)
+        self.assertEqual(
+            pointer.current_publication.entry_fact_revision_id,
+            second.id,
+        )
+        self.assertEqual(PublicationRevision.objects.count(), 2)
+        self.assertEqual(
+            AuditEvent.objects.filter(
+                outcome=AuditEvent.Outcome.FAILED,
+                failure_code=AuditEvent.FailureCode.INVALID_TARGET,
+            ).count(),
+            1,
+        )
+
+    def test_deferred_closure_rejects_every_partial_success_graph(self):
+        entry = self.make_entry()
+
+        for decision, reason in (
+            (
+                ReviewDecision.Decision.APPROVED,
+                ReviewDecision.ReasonCode.APPROVE_ENTRY,
+            ),
+            (
+                ReviewDecision.Decision.REJECTED,
+                ReviewDecision.ReasonCode.REJECT_ENTRY,
+            ),
+        ):
+            with self.assertRaises(DatabaseError), transaction.atomic():
+                ReviewDecision.objects.create(
+                    **self.review_values(
+                        entry,
+                        decision=decision,
+                        reason=reason,
+                    )
+                )
+                self.force_deferred_checks()
+            self.assertFalse(ReviewDecision.objects.exists())
+
+        with self.assertRaises(DatabaseError), transaction.atomic():
+            review = ReviewDecision.objects.create(
+                **self.review_values(entry)
+            )
+            PublicationRevision.objects.create(
+                **self.publication_values(
+                    entry,
+                    review,
+                    generation=1,
+                )
+            )
+            self.force_deferred_checks()
+        self.assertFalse(ReviewDecision.objects.exists())
+        self.assertFalse(PublicationRevision.objects.exists())
+
+        with self.assertRaises(DatabaseError), transaction.atomic():
+            review = ReviewDecision.objects.create(
+                **self.review_values(entry)
+            )
+            publication = PublicationRevision.objects.create(
+                **self.publication_values(
+                    entry,
+                    review,
+                    generation=1,
+                )
+            )
+            AuditEvent.objects.create(
+                **self.audit_values(
+                    action=AuditEvent.Action.PUBLISH,
+                    subject=entry,
+                    review=review,
+                    publication=publication,
+                )
+            )
+            self.force_deferred_checks()
+        self.assertFalse(AuditEvent.objects.exists())
+
+        with self.assertRaises(DatabaseError), transaction.atomic():
+            review = ReviewDecision.objects.create(
+                **self.review_values(entry)
+            )
+            publication = PublicationRevision.objects.create(
+                **self.publication_values(
+                    entry,
+                    review,
+                    generation=1,
+                )
+            )
+            PublishedEntryFacts.objects.filter(version=0).update(
+                current_publication=publication,
+                version=1,
+                updated_at=timezone.now(),
+            )
+        self.assertFalse(PublicationRevision.objects.exists())
+        self.assertEqual(PublishedEntryFacts.objects.get().version, 0)
+
+        complete = publish_candidate(
+            module=PublicationModule.ENTRY,
+            typed_revision_id=entry.id,
+            actor=self.actor,
+            expected_pointer_version=0,
+        )
+        self.assertEqual(complete.code, PublicationCode.PUBLISHED)
+        self.assertEqual(complete.generation, 1)
+
+    def test_deferred_closure_blocks_reject_before_final_pointer_update(self):
+        entry = self.make_entry()
+
+        with self.assertRaises(DatabaseError), transaction.atomic():
+            approved = ReviewDecision.objects.create(
+                **self.review_values(entry)
+            )
+            publication = PublicationRevision.objects.create(
+                **self.publication_values(
+                    entry,
+                    approved,
+                    generation=1,
+                )
+            )
+            AuditEvent.objects.create(
+                **self.audit_values(
+                    action=AuditEvent.Action.PUBLISH,
+                    subject=entry,
+                    review=approved,
+                    publication=publication,
+                )
+            )
+            rejected = ReviewDecision.objects.create(
+                **self.review_values(
+                    entry,
+                    decision=ReviewDecision.Decision.REJECTED,
+                    reason=ReviewDecision.ReasonCode.REJECT_ENTRY,
+                    sequence=2,
+                )
+            )
+            AuditEvent.objects.create(
+                **self.audit_values(
+                    action=AuditEvent.Action.REVIEW_REJECT,
+                    subject=entry,
+                    review=rejected,
+                )
+            )
+            PublishedEntryFacts.objects.filter(version=0).update(
+                current_publication=publication,
+                version=1,
+                updated_at=timezone.now(),
+            )
+            self.force_deferred_checks()
+
+        self.assertFalse(ReviewDecision.objects.exists())
+        self.assertFalse(PublicationRevision.objects.exists())
+        self.assertFalse(AuditEvent.objects.exists())
+        pointer = PublishedEntryFacts.objects.get()
+        self.assertEqual(pointer.version, 0)
+        self.assertIsNone(pointer.current_publication_id)
+
+    def test_db_action_binding_rejects_cross_permission_bypasses(self):
+        first = self.make_entry(period="90일")
+        second = self.make_entry(period="30일")
+        publish_only_subject = self.make_entry(period="20일")
+        rollback_only_subject = self.make_entry(period="10일")
+        publish_only = self.actor_with_permission(
+            "publish-only-operator",
+            "publish_entry",
+        )
+        rollback_only = self.actor_with_permission(
+            "rollback-only-operator",
+            "rollback_entry",
+        )
+        publish_candidate(
+            module=PublicationModule.ENTRY,
+            typed_revision_id=first.id,
+            actor=self.actor,
+            expected_pointer_version=0,
+        )
+        first_publication = PublicationRevision.objects.get(generation=1)
+        publish_candidate(
+            module=PublicationModule.ENTRY,
+            typed_revision_id=second.id,
+            actor=self.actor,
+            expected_pointer_version=1,
+        )
+        second_publication = PublicationRevision.objects.get(generation=2)
+
+        with self.assertRaises(DatabaseError), transaction.atomic():
+            review = ReviewDecision.objects.create(
+                **self.review_values(
+                    publish_only_subject,
+                    actor=publish_only,
+                )
+            )
+            PublicationRevision.objects.create(
+                **self.publication_values(
+                    publish_only_subject,
+                    review,
+                    generation=3,
+                    supersedes=second_publication,
+                    rollback_target=first_publication,
+                )
+            )
+
+        with self.assertRaises(DatabaseError), transaction.atomic():
+            review = ReviewDecision.objects.create(
+                **self.review_values(
+                    rollback_only_subject,
+                    actor=rollback_only,
+                    reason=ReviewDecision.ReasonCode.ROLLBACK_ENTRY,
+                )
+            )
+            PublicationRevision.objects.create(
+                **self.publication_values(
+                    rollback_only_subject,
+                    review,
+                    generation=3,
+                    supersedes=second_publication,
+                )
+            )
+
+        with self.assertRaises(DatabaseError), transaction.atomic():
+            rollback_review = ReviewDecision.objects.create(
+                **self.review_values(
+                    first,
+                    actor=rollback_only,
+                    reason=ReviewDecision.ReasonCode.ROLLBACK_ENTRY,
+                    sequence=2,
+                )
+            )
+            rollback_revision = PublicationRevision.objects.create(
+                **self.publication_values(
+                    first,
+                    rollback_review,
+                    generation=3,
+                    supersedes=second_publication,
+                    rollback_target=first_publication,
+                )
+            )
+            AuditEvent.objects.create(
+                **self.audit_values(
+                    action=AuditEvent.Action.PUBLISH,
+                    subject=first,
+                    review=rollback_review,
+                    publication=rollback_revision,
+                    prior=second_publication,
+                )
+            )
+
+        self.assertEqual(PublicationRevision.objects.count(), 2)
+        self.assertEqual(
+            ReviewDecision.objects.filter(
+                reason_code=ReviewDecision.ReasonCode.ROLLBACK_ENTRY
+            ).count(),
+            0,
+        )
+
+    def test_success_audit_is_exactly_one_and_shares_review_correlation(self):
+        entry = self.make_entry()
+        outcome = publish_candidate(
+            module=PublicationModule.ENTRY,
+            typed_revision_id=entry.id,
+            actor=self.actor,
+            expected_pointer_version=0,
+        )
+        self.assertEqual(outcome.code, PublicationCode.PUBLISHED)
+        review = ReviewDecision.objects.get()
+        publication = PublicationRevision.objects.get()
+        audit = AuditEvent.objects.get(outcome=AuditEvent.Outcome.SUCCEEDED)
+        self.assertEqual(audit.correlation_id, review.correlation_id)
+
+        with self.assertRaises(IntegrityError), transaction.atomic():
+            AuditEvent.objects.create(
+                **self.audit_values(
+                    action=AuditEvent.Action.PUBLISH,
+                    subject=entry,
+                    review=review,
+                    publication=publication,
+                )
+            )
+        self.assertEqual(
+            AuditEvent.objects.filter(outcome=AuditEvent.Outcome.SUCCEEDED).count(),
+            1,
+        )
+
+        second = self.make_entry(period="30일")
+        with self.assertRaises(DatabaseError), transaction.atomic():
+            second_review = ReviewDecision.objects.create(
+                **self.review_values(second)
+            )
+            second_publication = PublicationRevision.objects.create(
+                **self.publication_values(
+                    second,
+                    second_review,
+                    generation=2,
+                    supersedes=publication,
+                )
+            )
+            AuditEvent.objects.create(
+                **self.audit_values(
+                    action=AuditEvent.Action.PUBLISH,
+                    subject=second,
+                    review=second_review,
+                    publication=second_publication,
+                    prior=publication,
+                    correlation_id=uuid.uuid4(),
+                )
+            )
+        self.assertEqual(PublicationRevision.objects.count(), 1)
+
+    def test_two_full_publishers_have_one_winner_and_one_stale_loser(self):
+        entry = self.make_entry()
+        barrier = Barrier(2)
+        actor_id = self.actor.id
+
+        def contender():
+            close_old_connections()
+            try:
+                actor = get_user_model().objects.get(pk=actor_id)
+                with connection.cursor() as cursor:
+                    cursor.execute("SET lock_timeout = '5s'")
+                barrier.wait(timeout=5)
+                return publish_candidate(
+                    module=PublicationModule.ENTRY,
+                    typed_revision_id=entry.id,
+                    actor=actor,
+                    expected_pointer_version=0,
+                ).code
+            finally:
+                close_old_connections()
+
+        with ThreadPoolExecutor(max_workers=2) as pool:
+            codes = sorted(
+                future.result(timeout=15)
+                for future in (pool.submit(contender), pool.submit(contender))
+            )
+
+        self.assertEqual(
+            codes,
+            sorted([PublicationCode.PUBLISHED, PublicationCode.STALE_POINTER]),
+        )
+        self.assertEqual(PublicationRevision.objects.count(), 1)
+        self.assertEqual(
+            ReviewDecision.objects.filter(
+                decision=ReviewDecision.Decision.APPROVED
+            ).count(),
+            1,
+        )
+        self.assertEqual(
+            AuditEvent.objects.filter(outcome=AuditEvent.Outcome.SUCCEEDED).count(),
+            1,
+        )
+        self.assertEqual(
+            AuditEvent.objects.filter(
+                outcome=AuditEvent.Outcome.FAILED,
+                failure_code=AuditEvent.FailureCode.STALE_POINTER,
+            ).count(),
+            1,
+        )
+        pointer = PublishedEntryFacts.objects.get()
+        self.assertEqual(pointer.version, 1)
+        self.assertEqual(
+            pointer.current_publication.entry_fact_revision_id,
+            entry.id,
+        )
+
+    def test_direct_reject_and_service_publish_race_is_linearized(self):
+        entry = self.make_entry()
+        barrier = Barrier(2)
+        actor_id = self.actor.id
+
+        def publisher():
+            close_old_connections()
+            try:
+                actor = get_user_model().objects.get(pk=actor_id)
+                with connection.cursor() as cursor:
+                    cursor.execute("SET lock_timeout = '5s'")
+                barrier.wait(timeout=5)
+                return publish_candidate(
+                    module=PublicationModule.ENTRY,
+                    typed_revision_id=entry.id,
+                    actor=actor,
+                    expected_pointer_version=0,
+                ).code
+            finally:
+                close_old_connections()
+
+        def direct_reviewer():
+            close_old_connections()
+            try:
+                actor = get_user_model().objects.get(pk=actor_id)
+                with connection.cursor() as cursor:
+                    cursor.execute("SET lock_timeout = '5s'")
+                barrier.wait(timeout=5)
+                try:
+                    with transaction.atomic():
+                        review = ReviewDecision.objects.create(
+                            **self.review_values(
+                                entry,
+                                actor=actor,
+                                decision=ReviewDecision.Decision.REJECTED,
+                                reason=ReviewDecision.ReasonCode.REJECT_ENTRY,
+                            )
+                        )
+                        AuditEvent.objects.create(
+                            **self.audit_values(
+                                action=AuditEvent.Action.REVIEW_REJECT,
+                                subject=entry,
+                                review=review,
+                            )
+                        )
+                    return "DIRECT_REJECTED"
+                except DatabaseError as error:
+                    return getattr(error.__cause__, "sqlstate", "DB_ERROR")
+            finally:
+                close_old_connections()
+
+        with ThreadPoolExecutor(max_workers=2) as pool:
+            publish_future = pool.submit(publisher)
+            reject_future = pool.submit(direct_reviewer)
+            publish_code = publish_future.result(timeout=15)
+            direct_code = reject_future.result(timeout=15)
+
+        self.assertEqual(publish_code, PublicationCode.PUBLISHED)
+        self.assertIn(direct_code, {"DIRECT_REJECTED", "23514"})
+        self.assertNotIn(direct_code, {"40P01", "55P03", "57014"})
+        pointer = PublishedEntryFacts.objects.get()
+        self.assertEqual(
+            pointer.current_publication.entry_fact_revision_id,
+            entry.id,
+        )
+        latest = ReviewDecision.objects.filter(
+            entry_fact_revision=entry
+        ).latest("decision_sequence")
+        self.assertEqual(latest.decision, ReviewDecision.Decision.APPROVED)
+
+    def test_rollback_creates_new_revision_with_explicit_historical_target(self):
+        first = self.make_entry(period="90일")
+        second = self.make_entry(period="30일")
+        publish_candidate(
+            module=PublicationModule.ENTRY,
+            typed_revision_id=first.id,
+            actor=self.actor,
+            expected_pointer_version=0,
+        )
+        first_publication = PublicationRevision.objects.get(generation=1)
+        publish_candidate(
+            module=PublicationModule.ENTRY,
+            typed_revision_id=second.id,
+            actor=self.actor,
+            expected_pointer_version=1,
+        )
+        second_publication = PublicationRevision.objects.get(generation=2)
+
+        outcome = rollback_publication(
+            module=PublicationModule.ENTRY,
+            target_publication_id=first_publication.id,
+            actor=self.actor,
+            expected_pointer_version=2,
+        )
+
+        self.assertEqual(outcome.code, PublicationCode.ROLLED_BACK)
+        restored = PublicationRevision.objects.get(generation=3)
+        self.assertEqual(restored.supersedes_id, second_publication.id)
+        self.assertEqual(restored.rollback_target_id, first_publication.id)
+        self.assertEqual(restored.entry_fact_revision_id, first.id)
+        pointer = PublishedEntryFacts.objects.get()
+        self.assertEqual(pointer.version, 3)
+        self.assertEqual(pointer.current_publication_id, restored.id)
+
+    def test_invalid_and_stale_rollback_leave_pointer_atomic(self):
+        entry = self.make_entry()
+        warning = self.make_warning()
+        publish_candidate(
+            module=PublicationModule.ENTRY,
+            typed_revision_id=entry.id,
+            actor=self.actor,
+            expected_pointer_version=0,
+        )
+        publish_candidate(
+            module=PublicationModule.TRAVEL_WARNING,
+            typed_revision_id=warning.id,
+            actor=self.actor,
+            expected_pointer_version=0,
+        )
+        warning_publication = PublicationRevision.objects.get(
+            module=PublicationModule.TRAVEL_WARNING
+        )
+        entry_pointer = PublishedEntryFacts.objects.get()
+
+        invalid = rollback_publication(
+            module=PublicationModule.ENTRY,
+            target_publication_id=warning_publication.id,
+            actor=self.actor,
+            expected_pointer_version=1,
+        )
+        stale = rollback_publication(
+            module=PublicationModule.ENTRY,
+            target_publication_id=entry_pointer.current_publication_id,
+            actor=self.actor,
+            expected_pointer_version=2,
+        )
+
+        self.assertEqual(invalid.code, PublicationCode.INVALID_TARGET)
+        self.assertEqual(stale.code, PublicationCode.STALE_POINTER)
+        entry_pointer.refresh_from_db()
+        self.assertEqual(entry_pointer.version, 1)
+        self.assertEqual(PublicationRevision.objects.count(), 2)
+
+    def test_forced_failure_rolls_back_review_publication_and_pointer(self):
+        entry = self.make_entry()
+
+        with patch(
+            "reviews.publication._audit_success",
+            side_effect=RuntimeError("synthetic internal failure"),
+        ):
+            outcome = publish_candidate(
+                module=PublicationModule.ENTRY,
+                typed_revision_id=entry.id,
+                actor=self.actor,
+                expected_pointer_version=0,
+            )
+
+        self.assertEqual(outcome.code, PublicationCode.TRANSACTION_ABORTED)
+        self.assertEqual(PublicationRevision.objects.count(), 0)
+        self.assertEqual(ReviewDecision.objects.count(), 0)
+        pointer = PublishedEntryFacts.objects.get()
+        self.assertEqual(pointer.version, 0)
+        self.assertIsNone(pointer.current_publication_id)
+        failure = AuditEvent.objects.get(outcome=AuditEvent.Outcome.FAILED)
+        self.assertEqual(
+            failure.failure_code,
+            AuditEvent.FailureCode.TRANSACTION_ABORTED,
+        )
+
+    def test_history_and_pointer_bypass_mutations_are_blocked(self):
+        entry = self.make_entry()
+        publish_candidate(
+            module=PublicationModule.ENTRY,
+            typed_revision_id=entry.id,
+            actor=self.actor,
+            expected_pointer_version=0,
+        )
+        publication = PublicationRevision.objects.get()
+        review = ReviewDecision.objects.get()
+        audit = AuditEvent.objects.get(outcome=AuditEvent.Outcome.SUCCEEDED)
+
+        for operation in (
+            lambda: PublicationRevision.objects.filter(pk=publication.pk).update(
+                source_owner_snapshot="changed"
+            ),
+            lambda: ReviewDecision.objects.filter(pk=review.pk).update(
+                reason_code=ReviewDecision.ReasonCode.REJECT_ENTRY
+            ),
+            lambda: AuditEvent.objects.filter(pk=audit.pk).update(
+                redaction_state="changed"
+            ),
+            lambda: PublishedEntryFacts.objects.update(version=2),
+        ):
+            with self.assertRaises(DatabaseError), transaction.atomic():
+                operation()
+
+        for table, row_id in (
+            ("reviews_publicationrevision", publication.id),
+            ("reviews_reviewdecision", review.id),
+            ("reviews_auditevent", audit.id),
+        ):
+            with self.assertRaises(DatabaseError), transaction.atomic():
+                with connection.cursor() as cursor:
+                    cursor.execute(
+                        f"DELETE FROM {table} WHERE id = %s",
+                        [row_id],
+                    )
+
+    def test_pointer_seed_refuses_missing_or_reinitialized_history(self):
+        entry = self.make_entry()
+        outcome = publish_candidate(
+            module=PublicationModule.ENTRY,
+            typed_revision_id=entry.id,
+            actor=self.actor,
+            expected_pointer_version=0,
+        )
+        self.assertEqual(outcome.code, PublicationCode.PUBLISHED)
+        first_publication = PublicationRevision.objects.get(generation=1)
+        second = self.make_entry(period="30일")
+        second_outcome = publish_candidate(
+            module=PublicationModule.ENTRY,
+            typed_revision_id=second.id,
+            actor=self.actor,
+            expected_pointer_version=1,
+        )
+        self.assertEqual(second_outcome.code, PublicationCode.PUBLISHED)
+        pointer = PublishedEntryFacts.objects.get()
+        schema_editor = SimpleNamespace(connection=connection)
+
+        def mutate_pointer(sql, params=()):
+            with connection.cursor() as cursor:
+                cursor.execute(
+                    "ALTER TABLE reviews_publishedentryfacts "
+                    "DISABLE TRIGGER USER"
+                )
+                try:
+                    cursor.execute(sql, params)
+                finally:
+                    cursor.execute(
+                        "ALTER TABLE reviews_publishedentryfacts "
+                        "ENABLE TRIGGER USER"
+                    )
+
+        mutate_pointer(
+            "DELETE FROM reviews_publishedentryfacts WHERE id = %s",
+            [ENTRY_POINTER_ID],
+        )
+        try:
+            with self.assertRaisesRegex(RuntimeError, "no current pointer"):
+                migration_module.seed_module_pointers(
+                    django_apps,
+                    schema_editor,
+                )
+        finally:
+            mutate_pointer(
+                "INSERT INTO reviews_publishedentryfacts "
+                "(id, country_id, passport_policy_id, "
+                "current_publication_id, version, updated_at) "
+                "VALUES (%s, %s, %s, %s, %s, %s)",
+                [
+                    ENTRY_POINTER_ID,
+                    JP_COUNTRY_ID,
+                    PASSPORT_POLICY_ID,
+                    pointer.current_publication_id,
+                    pointer.version,
+                    pointer.updated_at,
+                ],
+            )
+
+        mutate_pointer(
+            "UPDATE reviews_publishedentryfacts "
+            "SET current_publication_id = NULL, version = 0 WHERE id = %s",
+            [ENTRY_POINTER_ID],
+        )
+        try:
+            with self.assertRaisesRegex(RuntimeError, "initial pointer"):
+                migration_module.seed_module_pointers(
+                    django_apps,
+                    schema_editor,
+                )
+        finally:
+            mutate_pointer(
+                "UPDATE reviews_publishedentryfacts "
+                "SET current_publication_id = %s, version = %s, "
+                "updated_at = %s WHERE id = %s",
+                [
+                    pointer.current_publication_id,
+                    pointer.version,
+                    pointer.updated_at,
+                    ENTRY_POINTER_ID,
+                ],
+            )
+
+        mutate_pointer(
+            "UPDATE reviews_publishedentryfacts "
+            "SET current_publication_id = %s, version = 1 WHERE id = %s",
+            [first_publication.id, ENTRY_POINTER_ID],
+        )
+        try:
+            with self.assertRaisesRegex(RuntimeError, "fixed scope"):
+                migration_module.seed_module_pointers(
+                    django_apps,
+                    schema_editor,
+                )
+        finally:
+            mutate_pointer(
+                "UPDATE reviews_publishedentryfacts "
+                "SET current_publication_id = %s, version = %s, "
+                "updated_at = %s WHERE id = %s",
+                [
+                    pointer.current_publication_id,
+                    pointer.version,
+                    pointer.updated_at,
+                    ENTRY_POINTER_ID,
+                ],
+            )
+
+        restored = PublishedEntryFacts.objects.get()
+        self.assertEqual(restored.version, pointer.version)
+        self.assertEqual(
+            restored.current_publication_id,
+            pointer.current_publication_id,
+        )
+
+    def test_populated_reverse_fails_without_losing_schema_or_history(self):
+        entry = self.make_entry()
+        outcome = publish_candidate(
+            module=PublicationModule.ENTRY,
+            typed_revision_id=entry.id,
+            actor=self.actor,
+            expected_pointer_version=0,
+        )
+        self.assertEqual(outcome.code, PublicationCode.PUBLISHED)
+        publication_id = PublishedEntryFacts.objects.get().current_publication_id
+        executor = MigrationExecutor(connection)
+        self.addCleanup(lambda: restore_canonical_migration_graph(connection))
+
+        with self.assertRaises(DatabaseError):
+            executor.migrate([("reviews", None)])
+
+        with connection.cursor() as cursor:
+            cursor.execute("SELECT to_regclass('reviews_publicationrevision')")
+            self.assertEqual(cursor.fetchone()[0], "reviews_publicationrevision")
+            cursor.execute(
+                "SELECT count(*) FROM reviews_publicationrevision WHERE id = %s",
+                [publication_id],
+            )
+            self.assertEqual(cursor.fetchone()[0], 1)
+            cursor.execute(
+                "SELECT count(*) FROM pg_trigger "
+                "WHERE tgname = 'reviews_review_deferred_closure' "
+                "AND tgenabled = 'O'"
+            )
+            self.assertEqual(cursor.fetchone()[0], 1)
+            cursor.execute(
+                "SELECT count(*) FROM django_migrations "
+                "WHERE app = 'reviews' AND name = '0001_initial'"
+            )
+            self.assertEqual(cursor.fetchone()[0], 1)
+
+
+class PublicationSchemaTests(SimpleTestCase):
+    def test_review_schema_has_no_trip_secret_raw_or_raw_hash_fields(self):
+        models_to_check = (
+            ReviewDecision,
+            PublicationRevision,
+            PublishedEntryFacts,
+            PublishedTravelWarning,
+            AuditEvent,
+        )
+        field_names = {
+            field.name.lower()
+            for model in models_to_check
+            for field in model._meta.get_fields()
+            if getattr(field, "concrete", False)
+        }
+        for forbidden in (
+            "destination",
+            "travel_date",
+            "departure",
+            "return_date",
+            "secret",
+            "credential",
+            "raw",
+            "body_sha",
+        ):
+            self.assertFalse(
+                any(forbidden in name for name in field_names),
+                msg=f"forbidden review persistence field: {forbidden}",
+            )
+
+    def test_reverse_guard_is_access_exclusive_and_empty_only(self):
+        sql = migration_module.REVIEWS_GUARDS_REVERSE_SQL
+        self.assertIn("ACCESS EXCLUSIVE MODE", sql)
+        self.assertIn("reviews_auditevent", sql)
+        self.assertIn("reviews_publicationrevision", sql)
+        self.assertIn("reviews_reviewdecision", sql)
+        self.assertIn("current_publication_id IS NOT NULL", sql)
+
+
+class PublicationMigrationTests(TransactionTestCase):
+    def test_empty_boundary_can_reverse_and_restore(self):
+        executor = MigrationExecutor(connection)
+        self.addCleanup(
+            lambda: restore_canonical_migration_graph(connection)
+        )
+
+        executor.migrate([("reviews", None)])
+        with connection.cursor() as cursor:
+            cursor.execute("SELECT to_regclass('reviews_publicationrevision')")
+            self.assertIsNone(cursor.fetchone()[0])
+
+        restore_canonical_migration_graph(connection)
+        with connection.cursor() as cursor:
+            cursor.execute("SELECT to_regclass('reviews_publicationrevision')")
+            self.assertEqual(cursor.fetchone()[0], "reviews_publicationrevision")
diff --git a/travel_readiness/settings.py b/travel_readiness/settings.py
index c835678..c57d815 100644
--- a/travel_readiness/settings.py
+++ b/travel_readiness/settings.py
@@ -39,6 +39,7 @@ INSTALLED_APPS = [
     "sources",
     "entry_requirements",
     "travel_warnings",
+    "reviews",
     "django.contrib.admin",
     "django.contrib.auth",
     "django.contrib.contenttypes",


