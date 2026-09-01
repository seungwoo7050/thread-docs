# 입국요건 CSV의 엄격 파싱과 내구성 있는 수집

## `feat(entry): validate typed JP snapshot facts`

diff --git a/entry_requirements/__init__.py b/entry_requirements/__init__.py
new file mode 100644
index 0000000..8b13789
--- /dev/null
+++ b/entry_requirements/__init__.py
@@ -0,0 +1 @@
+
diff --git a/entry_requirements/apps.py b/entry_requirements/apps.py
new file mode 100644
index 0000000..ef1430e
--- /dev/null
+++ b/entry_requirements/apps.py
@@ -0,0 +1,6 @@
+from django.apps import AppConfig
+
+
+class EntryRequirementsConfig(AppConfig):
+    default_auto_field = "django.db.models.BigAutoField"
+    name = "entry_requirements"
diff --git a/entry_requirements/migrations/0001_initial.py b/entry_requirements/migrations/0001_initial.py
new file mode 100644
index 0000000..42d2265
--- /dev/null
+++ b/entry_requirements/migrations/0001_initial.py
@@ -0,0 +1,440 @@
+import uuid
+from datetime import date
+
+import django.db.models.deletion
+from django.db import migrations, models
+from django.db.models import F, Q
+
+
+JP_COUNTRY_ID = uuid.UUID("575fa8b9-14f9-526e-9464-ebd1dea76da9")
+PASSPORT_POLICY_ID = uuid.UUID("f461f7a7-18f7-5b0d-9831-bfbd47b695e5")
+PASSPORT_POLICY = (
+    PASSPORT_POLICY_ID,
+    "KOR_ORDINARY_SHORT_TOURISM",
+    "V1",
+)
+ENTRY_SNAPSHOT_DATE = date(2025, 1, 20)
+
+
+ENTRY_BOUNDARY_GUARDS_SQL = """
+CREATE FUNCTION entry_requirements_reject_policy_mutation() RETURNS trigger
+LANGUAGE plpgsql AS $$
+BEGIN
+    RAISE EXCEPTION 'passport policy identity is immutable'
+        USING ERRCODE = 'check_violation';
+END;
+$$;
+CREATE TRIGGER entry_requirements_policy_immutable_guard
+BEFORE UPDATE OR DELETE ON entry_requirements_passportpolicy
+FOR EACH ROW EXECUTE FUNCTION entry_requirements_reject_policy_mutation();
+
+CREATE FUNCTION entry_requirements_guard_fact_revision() RETURNS trigger
+LANGUAGE plpgsql AS $$
+DECLARE
+    parse_outcome text;
+    parser_name text;
+    parser_version text;
+    parser_contract text;
+    expected_schema text;
+    observed_schema text;
+    parse_started_at timestamptz;
+    parse_completed_at timestamptz;
+    artifact_state text;
+    artifact_byte_count bigint;
+    artifact_body_sha text;
+    source_module text;
+    source_code text;
+    source_revision text;
+    source_state text;
+    source_enabled boolean;
+    attempt_result text;
+    attempt_source_revision text;
+    attempt_response_sha text;
+    first_observation timestamptz;
+    rights_id uuid;
+    latest_rights_id uuid;
+    rights_source_revision text;
+    rights_decision text;
+    rights_access_mode text;
+    rights_access_allowed boolean;
+    rights_automation_allowed boolean;
+    rights_typed_storage_allowed boolean;
+    rights_raw_storage_allowed boolean;
+    rights_typed_republication_allowed boolean;
+    rights_raw_retention_seconds integer;
+    rights_typed_retention text;
+    rights_evidence_retention text;
+    rights_field_scope text;
+    rights_attribution text;
+    rights_contract text;
+    canonical_projection text;
+BEGIN
+    IF TG_OP = 'UPDATE' OR TG_OP = 'DELETE' THEN
+        RAISE EXCEPTION 'entry fact revisions are immutable'
+            USING ERRCODE = 'check_violation';
+    END IF;
+
+    IF NEW.country_id <> '575fa8b9-14f9-526e-9464-ebd1dea76da9'::uuid
+       OR NEW.passport_policy_id <> 'f461f7a7-18f7-5b0d-9831-bfbd47b695e5'::uuid THEN
+        RAISE EXCEPTION 'entry fact identity is outside the approved allowlist'
+            USING ERRCODE = 'check_violation';
+    END IF;
+
+    SELECT parse_run.outcome,
+           parse_run.parser_name,
+           parse_run.parser_version,
+           parse_run.parser_contract_fingerprint_sha256,
+           parse_run.expected_schema_fingerprint_sha256,
+           parse_run.observed_schema_fingerprint_sha256,
+           parse_run.started_at,
+           parse_run.completed_at,
+           artifact.state,
+           artifact.byte_count,
+           artifact.body_sha256,
+           source.module,
+           source.code,
+           source.revision,
+           source.state,
+           source.enabled,
+           attempt.result,
+           attempt.source_revision,
+           attempt.response_sha256,
+           attempt.completed_at,
+           rights.id,
+           (
+               SELECT latest.id
+                 FROM sources_sourcerightsdecision AS latest
+                WHERE latest.source_id = source.id
+                  AND latest.source_revision = source.revision
+                ORDER BY latest.decision_sequence DESC
+                LIMIT 1
+           ),
+           rights.source_revision,
+           rights.decision,
+           rights.access_mode,
+           rights.access_allowed,
+           rights.automated_collection_allowed,
+           rights.typed_field_storage_allowed,
+           rights.raw_body_storage_allowed,
+           rights.typed_republication_allowed,
+           rights.raw_retention_seconds,
+           rights.typed_retention,
+           rights.evidence_retention,
+           rights.field_scope_code,
+           rights.attribution_text,
+           rights.contract_fingerprint_sha256
+      INTO parse_outcome,
+           parser_name,
+           parser_version,
+           parser_contract,
+           expected_schema,
+           observed_schema,
+           parse_started_at,
+           parse_completed_at,
+           artifact_state,
+           artifact_byte_count,
+           artifact_body_sha,
+           source_module,
+           source_code,
+           source_revision,
+           source_state,
+           source_enabled,
+           attempt_result,
+           attempt_source_revision,
+           attempt_response_sha,
+           first_observation,
+           rights_id,
+           latest_rights_id,
+           rights_source_revision,
+           rights_decision,
+           rights_access_mode,
+           rights_access_allowed,
+           rights_automation_allowed,
+           rights_typed_storage_allowed,
+           rights_raw_storage_allowed,
+           rights_typed_republication_allowed,
+           rights_raw_retention_seconds,
+           rights_typed_retention,
+           rights_evidence_retention,
+           rights_field_scope,
+           rights_attribution,
+           rights_contract
+      FROM sources_parserun AS parse_run
+      JOIN sources_sourceartifact AS artifact
+        ON artifact.id = parse_run.artifact_id
+      JOIN sources_sourceconfiguration AS source
+        ON source.id = artifact.source_id
+      JOIN sources_fetchattempt AS attempt
+        ON attempt.id = artifact.first_successful_attempt_id
+      JOIN sources_sourcerightsdecision AS rights
+        ON rights.id = attempt.rights_decision_id
+     WHERE parse_run.id = NEW.parse_run_id
+       FOR UPDATE OF source;
+
+    IF NOT FOUND
+       OR parse_outcome <> 'VALIDATED'
+       OR parser_name <> 'MOFA_ENTRY_CSV'
+       OR parser_version <> 'V1'
+       OR parser_contract <>
+          '622a399317ab730f9e9780f51b3ac837073cd99939f07f248323636e97676021'
+       OR expected_schema <>
+          '46bb5428a08a94810e6d381541cb7a3aed3f2f5110a529c2ae32e101cd5e1e0b'
+       OR observed_schema <> expected_schema
+       OR artifact_state <> 'REVIEW_REQUIRED'
+       OR artifact_byte_count < 1
+       OR artifact_byte_count > 262144
+       OR source_module <> 'ENTRY'
+       OR source_code <> 'MOFA_ENTRY_CSV'
+       OR source_revision <> 'rights-v1'
+       OR source_state <> 'ACTIVE'
+       OR NOT source_enabled
+       OR attempt_result <> 'SUCCEEDED'
+       OR attempt_source_revision <> source_revision
+       OR attempt_response_sha <> artifact_body_sha
+       OR rights_id IS DISTINCT FROM latest_rights_id
+       OR rights_source_revision <> source_revision
+       OR rights_decision <> 'APPROVED'
+       OR rights_access_mode <> 'ANONYMOUS_PUBLIC'
+       OR NOT rights_access_allowed
+       OR NOT rights_automation_allowed
+       OR NOT rights_typed_storage_allowed
+       OR rights_raw_storage_allowed
+       OR NOT rights_typed_republication_allowed
+       OR rights_raw_retention_seconds <> 0
+       OR rights_typed_retention <> 'PRODUCT_HISTORY'
+       OR rights_evidence_retention <> 'PRODUCT_HISTORY'
+       OR rights_field_scope <> 'JP_ORDINARY_TEXT_V1'
+       OR rights_attribution <> '외교부|공공데이터포털'
+       OR rights_contract <>
+          '622a399317ab730f9e9780f51b3ac837073cd99939f07f248323636e97676021' THEN
+        RAISE EXCEPTION 'entry fact requires the approved validated entry parse'
+            USING ERRCODE = 'check_violation';
+    END IF;
+
+    IF NEW.first_observed_at IS DISTINCT FROM first_observation
+       OR parse_started_at < first_observation
+       OR parse_completed_at < parse_started_at
+       OR NEW.created_at < parse_completed_at THEN
+        RAISE EXCEPTION 'entry first observation must match artifact evidence'
+            USING ERRCODE = 'check_violation';
+    END IF;
+
+    canonical_projection :=
+        '{"basis_text":' || to_jsonb(NEW.basis_text)::text ||
+        ',"country_iso2":"JP"' ||
+        ',"ordinary_passport_period_text":' ||
+            to_jsonb(NEW.ordinary_passport_period_text)::text ||
+        ',"passport_policy_code":"KOR_ORDINARY_SHORT_TOURISM"' ||
+        ',"passport_policy_revision":"V1"' ||
+        ',"snapshot_date":' ||
+            to_jsonb(to_char(NEW.snapshot_date, 'YYYY-MM-DD'))::text || '}';
+    IF NEW.typed_fingerprint_sha256 IS DISTINCT FROM
+       encode(sha256(convert_to(canonical_projection, 'UTF8')), 'hex') THEN
+        RAISE EXCEPTION 'entry typed fingerprint does not match its projection'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    RETURN NEW;
+END;
+$$;
+CREATE TRIGGER entry_requirements_fact_revision_guard
+BEFORE INSERT OR UPDATE OR DELETE ON entry_requirements_entryfactrevision
+FOR EACH ROW EXECUTE FUNCTION entry_requirements_guard_fact_revision();
+"""
+
+
+ENTRY_BOUNDARY_GUARDS_REVERSE_SQL = """
+DROP TRIGGER IF EXISTS entry_requirements_fact_revision_guard
+    ON entry_requirements_entryfactrevision;
+DROP FUNCTION IF EXISTS entry_requirements_guard_fact_revision();
+DROP TRIGGER IF EXISTS entry_requirements_policy_immutable_guard
+    ON entry_requirements_passportpolicy;
+DROP FUNCTION IF EXISTS entry_requirements_reject_policy_mutation();
+"""
+
+
+def seed_passport_policy(apps, schema_editor):
+    PassportPolicy = apps.get_model("entry_requirements", "PassportPolicy")
+    alias = schema_editor.connection.alias
+    policy, _ = PassportPolicy.objects.using(alias).get_or_create(
+        id=PASSPORT_POLICY_ID,
+        defaults={"code": PASSPORT_POLICY[1], "revision": PASSPORT_POLICY[2]},
+    )
+    actual = (policy.id, policy.code, policy.revision)
+    if actual != PASSPORT_POLICY:
+        raise RuntimeError("passport policy conflicts with the approved singleton")
+
+
+def require_empty_unreferenced_boundary(apps, schema_editor):
+    PassportPolicy = apps.get_model("entry_requirements", "PassportPolicy")
+    EntryFactRevision = apps.get_model("entry_requirements", "EntryFactRevision")
+    alias = schema_editor.connection.alias
+    with schema_editor.connection.cursor() as cursor:
+        cursor.execute(
+            "LOCK TABLE entry_requirements_entryfactrevision, "
+            "entry_requirements_passportpolicy IN ACCESS EXCLUSIVE MODE"
+        )
+        cursor.execute(
+            """
+            SELECT conrelid::regclass::text
+              FROM pg_constraint
+             WHERE contype = 'f'
+               AND (
+                    confrelid = 'entry_requirements_entryfactrevision'::regclass
+                    OR (
+                        confrelid = 'entry_requirements_passportpolicy'::regclass
+                        AND conrelid <> 'entry_requirements_entryfactrevision'::regclass
+                    )
+               )
+             LIMIT 1
+            """
+        )
+        external_reference = cursor.fetchone()
+    policies = list(
+        PassportPolicy.objects.using(alias)
+        .order_by("code")
+        .values_list("id", "code", "revision")
+    )
+    if (
+        EntryFactRevision.objects.using(alias).exists()
+        or policies != [PASSPORT_POLICY]
+        or external_reference is not None
+    ):
+        raise RuntimeError(
+            "entry boundary rollback requires no fact revisions, the untouched "
+            "policy singleton, and no external references"
+        )
+
+
+class Migration(migrations.Migration):
+    initial = True
+
+    dependencies = [
+        ("countries", "0001_initial"),
+        ("sources", "0009_aviation_draft_gate"),
+    ]
+
+    operations = [
+        migrations.CreateModel(
+            name="PassportPolicy",
+            fields=[
+                (
+                    "id",
+                    models.UUIDField(
+                        default=PASSPORT_POLICY_ID,
+                        editable=False,
+                        primary_key=True,
+                        serialize=False,
+                    ),
+                ),
+                ("code", models.CharField(max_length=32, unique=True)),
+                ("revision", models.CharField(max_length=16)),
+            ],
+            options={
+                "constraints": [
+                    models.CheckConstraint(
+                        condition=Q(
+                            id=PASSPORT_POLICY_ID,
+                            code="KOR_ORDINARY_SHORT_TOURISM",
+                            revision="V1",
+                        ),
+                        name="entry_passport_policy_exact",
+                    )
+                ]
+            },
+        ),
+        migrations.CreateModel(
+            name="EntryFactRevision",
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
+                (
+                    "ordinary_passport_period_text",
+                    models.CharField(max_length=32),
+                ),
+                ("basis_text", models.CharField(max_length=2000)),
+                ("snapshot_date", models.DateField()),
+                ("first_observed_at", models.DateTimeField()),
+                ("typed_fingerprint_sha256", models.CharField(max_length=64)),
+                ("created_at", models.DateTimeField(auto_now_add=True)),
+                (
+                    "country",
+                    models.ForeignKey(
+                        on_delete=django.db.models.deletion.PROTECT,
+                        related_name="entry_fact_revisions",
+                        to="countries.country",
+                    ),
+                ),
+                (
+                    "parse_run",
+                    models.OneToOneField(
+                        on_delete=django.db.models.deletion.PROTECT,
+                        related_name="entry_fact_revision",
+                        to="sources.parserun",
+                    ),
+                ),
+                (
+                    "passport_policy",
+                    models.ForeignKey(
+                        on_delete=django.db.models.deletion.PROTECT,
+                        related_name="entry_fact_revisions",
+                        to="entry_requirements.passportpolicy",
+                    ),
+                ),
+            ],
+            options={
+                "constraints": [
+                    models.CheckConstraint(
+                        condition=Q(country_id=JP_COUNTRY_ID),
+                        name="entry_fact_country_jp",
+                    ),
+                    models.CheckConstraint(
+                        condition=Q(passport_policy_id=PASSPORT_POLICY_ID),
+                        name="entry_fact_policy_exact",
+                    ),
+                    models.CheckConstraint(
+                        condition=Q(
+                            ordinary_passport_period_text__regex=r"^[1-9][0-9]*일$"
+                        ),
+                        name="entry_fact_period_explicit_day_unit",
+                    ),
+                    models.CheckConstraint(
+                        condition=Q(
+                            basis_text__regex=(
+                                r"일반여권[ \t]*소지자[ \t]*:[ \t]*[^[:space:]]"
+                            )
+                        ),
+                        name="entry_fact_basis_statement",
+                    ),
+                    models.CheckConstraint(
+                        condition=Q(snapshot_date=ENTRY_SNAPSHOT_DATE),
+                        name="entry_fact_snapshot_exact",
+                    ),
+                    models.CheckConstraint(
+                        condition=Q(
+                            typed_fingerprint_sha256__regex=r"^[0-9a-f]{64}$"
+                        ),
+                        name="entry_fact_typed_hash_format",
+                    ),
+                    models.CheckConstraint(
+                        condition=Q(created_at__gte=F("first_observed_at")),
+                        name="entry_fact_created_after_observed",
+                    ),
+                ]
+            },
+        ),
+        migrations.RunSQL(
+            ENTRY_BOUNDARY_GUARDS_SQL,
+            ENTRY_BOUNDARY_GUARDS_REVERSE_SQL,
+        ),
+        migrations.RunPython(
+            seed_passport_policy,
+            require_empty_unreferenced_boundary,
+        ),
+    ]
diff --git a/entry_requirements/migrations/__init__.py b/entry_requirements/migrations/__init__.py
new file mode 100644
index 0000000..8b13789
--- /dev/null
+++ b/entry_requirements/migrations/__init__.py
@@ -0,0 +1 @@
+
diff --git a/entry_requirements/models.py b/entry_requirements/models.py
new file mode 100644
index 0000000..2ea2da7
--- /dev/null
+++ b/entry_requirements/models.py
@@ -0,0 +1,104 @@
+import uuid
+from datetime import date
+
+from django.db import models
+from django.db.models import F, Q
+
+from countries.models import JP_COUNTRY_ID, Country
+from sources.models import ParseRun
+
+
+PASSPORT_POLICY_ID = uuid.UUID("f461f7a7-18f7-5b0d-9831-bfbd47b695e5")
+PASSPORT_POLICY_CODE = "KOR_ORDINARY_SHORT_TOURISM"
+PASSPORT_POLICY_REVISION = "V1"
+ENTRY_SNAPSHOT_DATE = date(2025, 1, 20)
+ENTRY_PERIOD_TEXT_MAX_LENGTH = 32
+ENTRY_BASIS_TEXT_MAX_LENGTH = 2000
+
+
+class PassportPolicy(models.Model):
+    id = models.UUIDField(
+        primary_key=True,
+        default=PASSPORT_POLICY_ID,
+        editable=False,
+    )
+    code = models.CharField(max_length=32, unique=True)
+    revision = models.CharField(max_length=16)
+
+    class Meta:
+        constraints = [
+            models.CheckConstraint(
+                condition=Q(
+                    id=PASSPORT_POLICY_ID,
+                    code=PASSPORT_POLICY_CODE,
+                    revision=PASSPORT_POLICY_REVISION,
+                ),
+                name="entry_passport_policy_exact",
+            )
+        ]
+
+    def __str__(self) -> str:
+        return f"{self.code}@{self.revision}"
+
+
+class EntryFactRevision(models.Model):
+    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
+    country = models.ForeignKey(
+        Country,
+        on_delete=models.PROTECT,
+        related_name="entry_fact_revisions",
+    )
+    passport_policy = models.ForeignKey(
+        PassportPolicy,
+        on_delete=models.PROTECT,
+        related_name="entry_fact_revisions",
+    )
+    parse_run = models.OneToOneField(
+        ParseRun,
+        on_delete=models.PROTECT,
+        related_name="entry_fact_revision",
+    )
+    ordinary_passport_period_text = models.CharField(
+        max_length=ENTRY_PERIOD_TEXT_MAX_LENGTH
+    )
+    basis_text = models.CharField(max_length=ENTRY_BASIS_TEXT_MAX_LENGTH)
+    snapshot_date = models.DateField()
+    first_observed_at = models.DateTimeField()
+    typed_fingerprint_sha256 = models.CharField(max_length=64)
+    created_at = models.DateTimeField(auto_now_add=True)
+
+    class Meta:
+        constraints = [
+            models.CheckConstraint(
+                condition=Q(country_id=JP_COUNTRY_ID),
+                name="entry_fact_country_jp",
+            ),
+            models.CheckConstraint(
+                condition=Q(passport_policy_id=PASSPORT_POLICY_ID),
+                name="entry_fact_policy_exact",
+            ),
+            models.CheckConstraint(
+                condition=Q(ordinary_passport_period_text__regex=r"^[1-9][0-9]*일$"),
+                name="entry_fact_period_explicit_day_unit",
+            ),
+            models.CheckConstraint(
+                condition=Q(
+                    basis_text__regex=(
+                        r"일반여권[ \t]*소지자[ \t]*:[ \t]*[^[:space:]]"
+                    )
+                ),
+                name="entry_fact_basis_statement",
+            ),
+            models.CheckConstraint(
+                condition=Q(snapshot_date=ENTRY_SNAPSHOT_DATE),
+                name="entry_fact_snapshot_exact",
+            ),
+            models.CheckConstraint(
+                condition=Q(typed_fingerprint_sha256__regex=r"^[0-9a-f]{64}$"),
+                name="entry_fact_typed_hash_format",
+            ),
+            models.CheckConstraint(
+                condition=Q(created_at__gte=F("first_observed_at")),
+                name="entry_fact_created_after_observed",
+            ),
+        ]
diff --git a/entry_requirements/parser.py b/entry_requirements/parser.py
new file mode 100644
index 0000000..eb4663d
--- /dev/null
+++ b/entry_requirements/parser.py
@@ -0,0 +1,222 @@
+import csv
+import hashlib
+import io
+import json
+import re
+from dataclasses import dataclass
+from datetime import date
+
+from sources.models import ParseRun
+
+from .models import (
+    ENTRY_BASIS_TEXT_MAX_LENGTH,
+    ENTRY_PERIOD_TEXT_MAX_LENGTH,
+    ENTRY_SNAPSHOT_DATE,
+    PASSPORT_POLICY_CODE,
+    PASSPORT_POLICY_REVISION,
+)
+
+
+MAX_ENTRY_CSV_BYTES = 262_144
+ENTRY_SCHEMA_FINGERPRINT_SHA256 = (
+    "46bb5428a08a94810e6d381541cb7a3aed3f2f5110a529c2ae32e101cd5e1e0b"
+)
+ENTRY_HEADERS = (
+    "국가",
+    "입국시 소지여부",
+    "일반여권소지자 - 입국가능여부",
+    "일반여권소지자 - 입국가능기간",
+    "관용여권소지자 - 입국가능여부",
+    "관용여권소지자 - 입국가능기간",
+    "외교관여권소지자 - 입국가능여부",
+    "외교관여권소지자 - 입국가능기간",
+    "무비자 입국 근거",
+    "비고",
+)
+
+_COUNTRY_NAME = "일본"
+_COUNTRY_INDEX = 0
+_VALIDATION_MARKER_INDEX = 2
+_PERIOD_INDEX = 3
+_BASIS_INDEX = 8
+_VALIDATION_MARKER_LENGTH = 1
+_VALIDATION_MARKER_UTF8_SHA256 = (
+    "18f5384d58bcb1bba0bcd9e6a6781d1a6ac2cc280c330ecbab6cb7931b721552"
+)
+_PERIOD_PATTERN = re.compile(r"[1-9][0-9]*일")
+_BASIS_STATEMENT_PATTERN = re.compile(
+    r"일반여권[ \t]*소지자[ \t]*:[ \t]*\S"
+)
+
+
+@dataclass(frozen=True, slots=True)
+class EntryProjection:
+    country_iso2: str
+    passport_policy_code: str
+    passport_policy_revision: str
+    ordinary_passport_period_text: str
+    basis_text: str
+    snapshot_date: date
+    typed_fingerprint_sha256: str
+
+
+@dataclass(frozen=True, slots=True)
+class EntryParseResult:
+    outcome: str
+    failure_code: str
+    observed_schema_fingerprint_sha256: str
+    projection: EntryProjection | None
+
+
+def schema_fingerprint(headers: tuple[str, ...] | list[str]) -> str:
+    canonical = json.dumps(
+        list(headers),
+        ensure_ascii=False,
+        separators=(",", ":"),
+    ).encode("utf-8")
+    return hashlib.sha256(canonical).hexdigest()
+
+
+def typed_fingerprint_sha256(
+    ordinary_passport_period_text: str,
+    basis_text: str,
+) -> str:
+    canonical = json.dumps(
+        {
+            "basis_text": basis_text,
+            "country_iso2": "JP",
+            "ordinary_passport_period_text": ordinary_passport_period_text,
+            "passport_policy_code": PASSPORT_POLICY_CODE,
+            "passport_policy_revision": PASSPORT_POLICY_REVISION,
+            "snapshot_date": ENTRY_SNAPSHOT_DATE.isoformat(),
+        },
+        ensure_ascii=False,
+        sort_keys=True,
+        separators=(",", ":"),
+    ).encode("utf-8")
+    return hashlib.sha256(canonical).hexdigest()
+
+
+def _result(
+    outcome: str,
+    failure_code: str = "",
+    observed_schema_fingerprint_sha256: str = "",
+    projection: EntryProjection | None = None,
+) -> EntryParseResult:
+    return EntryParseResult(
+        outcome=outcome,
+        failure_code=failure_code,
+        observed_schema_fingerprint_sha256=observed_schema_fingerprint_sha256,
+        projection=projection,
+    )
+
+
+def _quarantined(failure_code: str, observed: str = "") -> EntryParseResult:
+    return _result(ParseRun.Outcome.QUARANTINED, failure_code, observed)
+
+
+def _parse_entry_csv(payload: bytes) -> EntryParseResult:
+    if len(payload) > MAX_ENTRY_CSV_BYTES:
+        return _quarantined(ParseRun.FailureCode.SYNTAX_ERROR)
+
+    try:
+        decoded = payload.decode("cp949", errors="strict")
+    except UnicodeDecodeError:
+        return _quarantined(ParseRun.FailureCode.ENCODING_ERROR)
+
+    if "\x00" in decoded:
+        return _quarantined(ParseRun.FailureCode.SYNTAX_ERROR)
+
+    try:
+        reader = csv.reader(io.StringIO(decoded, newline=""), strict=True)
+        try:
+            header = next(reader)
+        except StopIteration:
+            header = []
+
+        observed = schema_fingerprint(header)
+        if tuple(header) != ENTRY_HEADERS:
+            return _quarantined(ParseRun.FailureCode.SCHEMA_MISMATCH, observed)
+
+        country_rows: list[tuple[str, str, str]] = []
+        for row in reader:
+            if len(row) != len(ENTRY_HEADERS):
+                return _quarantined(ParseRun.FailureCode.SYNTAX_ERROR)
+            if row[_COUNTRY_INDEX] == _COUNTRY_NAME:
+                country_rows.append(
+                    (
+                        row[_VALIDATION_MARKER_INDEX],
+                        row[_PERIOD_INDEX],
+                        row[_BASIS_INDEX],
+                    )
+                )
+    except csv.Error:
+        return _quarantined(ParseRun.FailureCode.SYNTAX_ERROR)
+
+    if not country_rows:
+        return _quarantined(
+            ParseRun.FailureCode.IDENTITY_MISMATCH,
+            ENTRY_SCHEMA_FINGERPRINT_SHA256,
+        )
+    if len(country_rows) > 1:
+        failure_code = (
+            ParseRun.FailureCode.DUPLICATE_RECORD
+            if len(set(country_rows)) == 1
+            else ParseRun.FailureCode.CONFLICTING_VALUE
+        )
+        return _quarantined(failure_code, ENTRY_SCHEMA_FINGERPRINT_SHA256)
+
+    validation_marker, period_text, basis_text = country_rows[0]
+    if validation_marker == "" or period_text == "" or basis_text == "":
+        return _quarantined(
+            ParseRun.FailureCode.REQUIRED_VALUE_MISSING,
+            ENTRY_SCHEMA_FINGERPRINT_SHA256,
+        )
+    if (
+        len(validation_marker) != _VALIDATION_MARKER_LENGTH
+        or hashlib.sha256(validation_marker.encode("utf-8")).hexdigest()
+        != _VALIDATION_MARKER_UTF8_SHA256
+        or len(period_text) > ENTRY_PERIOD_TEXT_MAX_LENGTH
+        or len(basis_text) > ENTRY_BASIS_TEXT_MAX_LENGTH
+        or _PERIOD_PATTERN.fullmatch(period_text) is None
+        or _BASIS_STATEMENT_PATTERN.search(basis_text) is None
+    ):
+        return _quarantined(
+            ParseRun.FailureCode.INVALID_VALUE,
+            ENTRY_SCHEMA_FINGERPRINT_SHA256,
+        )
+
+    typed_hash = typed_fingerprint_sha256(period_text, basis_text)
+    projection = EntryProjection(
+        country_iso2="JP",
+        passport_policy_code=PASSPORT_POLICY_CODE,
+        passport_policy_revision=PASSPORT_POLICY_REVISION,
+        ordinary_passport_period_text=period_text,
+        basis_text=basis_text,
+        snapshot_date=ENTRY_SNAPSHOT_DATE,
+        typed_fingerprint_sha256=typed_hash,
+    )
+    return _result(
+        ParseRun.Outcome.VALIDATED,
+        observed_schema_fingerprint_sha256=ENTRY_SCHEMA_FINGERPRINT_SHA256,
+        projection=projection,
+    )
+
+
+def parse_entry_csv(payload: bytes) -> EntryParseResult:
+    if not isinstance(payload, bytes):
+        return _result(
+            ParseRun.Outcome.FAILED,
+            ParseRun.FailureCode.INTERNAL_PARSER_ERROR,
+        )
+    try:
+        return _parse_entry_csv(payload)
+    except Exception:
+        return _result(
+            ParseRun.Outcome.FAILED,
+            ParseRun.FailureCode.INTERNAL_PARSER_ERROR,
+        )
+
+
+if schema_fingerprint(ENTRY_HEADERS) != ENTRY_SCHEMA_FINGERPRINT_SHA256:
+    raise RuntimeError("entry schema registry fingerprint does not match its headers")
diff --git a/entry_requirements/tests/__init__.py b/entry_requirements/tests/__init__.py
new file mode 100644
index 0000000..8b13789
--- /dev/null
+++ b/entry_requirements/tests/__init__.py
@@ -0,0 +1 @@
+
diff --git a/entry_requirements/tests/test_models.py b/entry_requirements/tests/test_models.py
new file mode 100644
index 0000000..2302b57
--- /dev/null
+++ b/entry_requirements/tests/test_models.py
@@ -0,0 +1,542 @@
+import hashlib
+import uuid
+from datetime import date, timedelta
+from io import StringIO
+
+from django.core.management import call_command
+from django.db import IntegrityError, connection, models, transaction
+from django.db.migrations.executor import MigrationExecutor
+from django.test import TestCase, TransactionTestCase
+from django.utils import timezone
+
+from countries.models import JP_COUNTRY_ID, Country
+from entry_requirements.models import (
+    ENTRY_SNAPSHOT_DATE,
+    PASSPORT_POLICY_CODE,
+    PASSPORT_POLICY_ID,
+    PASSPORT_POLICY_REVISION,
+    EntryFactRevision,
+    PassportPolicy,
+)
+from entry_requirements.parser import typed_fingerprint_sha256
+from sources.models import (
+    FetchAttempt,
+    ParseRun,
+    SourceArtifact,
+    SourceConfiguration,
+    SourceRightsDecision,
+)
+from sources.management.commands.register_approved_sources import (
+    APPROVED_SOURCE_SPECS,
+)
+
+
+ENTRY_CONTRACT = "622a399317ab730f9e9780f51b3ac837073cd99939f07f248323636e97676021"
+ENTRY_SCHEMA = "46bb5428a08a94810e6d381541cb7a3aed3f2f5110a529c2ae32e101cd5e1e0b"
+WARNING_CONTRACT = "c43c5f7e7e6f37f13ba7e5d9e2448b29b24d5524012b41fc5e851abb907c55f6"
+WARNING_SCHEMA = "64a98b89c39d6c04e0daf2bbac9353de053320b835a62076a37f620cc07c4f0b"
+PERIOD_TEXT = "90일"
+BASIS_TEXT = "일반여권 소지자 : synthetic evidence"
+
+
+class EntryFactFixtureMixin:
+    @classmethod
+    def register_sources(cls):
+        call_command("register_approved_sources", "--apply", stdout=StringIO())
+
+    def make_parse_run(
+        self,
+        *,
+        source_code="MOFA_ENTRY_CSV",
+        terminal=True,
+        validate_artifact=True,
+        byte_count=100,
+        parse_before_attempt=False,
+    ):
+        source = SourceConfiguration.objects.get(code=source_code)
+        rights = source.rights_decisions.get()
+        body_hash = hashlib.sha256(uuid.uuid4().bytes).hexdigest()
+        attempt = FetchAttempt.objects.create(
+            source=source,
+            source_revision=source.revision,
+            rights_decision=rights,
+            operation_id=uuid.uuid4(),
+            attempt_number=1,
+            request_fingerprint_revision="ENTRY_TEST_V1",
+            normalized_request_sha256="1" * 64,
+        )
+        completed_at = timezone.now()
+        FetchAttempt.objects.filter(pk=attempt.pk).update(
+            result=FetchAttempt.Result.SUCCEEDED,
+            completed_at=completed_at,
+            http_status=200,
+            response_sha256=body_hash,
+        )
+        attempt.refresh_from_db()
+        artifact = SourceArtifact.objects.create(
+            source=source,
+            body_sha256=body_hash,
+            byte_count=byte_count,
+            first_successful_attempt=attempt,
+        )
+        is_entry = source.module == SourceConfiguration.Module.ENTRY
+        parse_started_at = (
+            attempt.completed_at - timedelta(seconds=2)
+            if parse_before_attempt
+            else timezone.now()
+        )
+        run = ParseRun.objects.create(
+            artifact=artifact,
+            parser_name=(
+                ParseRun.ParserName.MOFA_ENTRY_CSV
+                if is_entry
+                else ParseRun.ParserName.MOFA_TRAVEL_ALARM_JSON
+            ),
+            parser_version=ParseRun.ParserVersion.V1,
+            parser_contract_fingerprint_sha256=(
+                ENTRY_CONTRACT if is_entry else WARNING_CONTRACT
+            ),
+            expected_schema_fingerprint_sha256=(
+                ENTRY_SCHEMA if is_entry else WARNING_SCHEMA
+            ),
+            started_at=parse_started_at,
+        )
+        if terminal:
+            parse_completed_at = (
+                attempt.completed_at - timedelta(seconds=1)
+                if parse_before_attempt
+                else timezone.now()
+            )
+            ParseRun.objects.filter(pk=run.pk).update(
+                outcome=ParseRun.Outcome.VALIDATED,
+                completed_at=parse_completed_at,
+                observed_schema_fingerprint_sha256=(
+                    ENTRY_SCHEMA if is_entry else WARNING_SCHEMA
+                ),
+            )
+            run.refresh_from_db()
+            if validate_artifact:
+                SourceArtifact.objects.filter(pk=artifact.pk).update(
+                    state=SourceArtifact.State.REVIEW_REQUIRED
+                )
+                artifact.refresh_from_db()
+        return run, artifact, attempt
+
+    def fact_values(self, run, attempt, **overrides):
+        values = {
+            "country": Country.objects.get(pk=JP_COUNTRY_ID),
+            "passport_policy": PassportPolicy.objects.get(pk=PASSPORT_POLICY_ID),
+            "parse_run": run,
+            "ordinary_passport_period_text": PERIOD_TEXT,
+            "basis_text": BASIS_TEXT,
+            "snapshot_date": ENTRY_SNAPSHOT_DATE,
+            "first_observed_at": attempt.completed_at,
+            "typed_fingerprint_sha256": typed_fingerprint_sha256(
+                PERIOD_TEXT,
+                BASIS_TEXT,
+            ),
+        }
+        if "country_id" in overrides:
+            values.pop("country")
+        if "passport_policy_id" in overrides:
+            values.pop("passport_policy")
+        values.update(overrides)
+        return values
+
+    def assert_integrity_error(self, action):
+        with self.assertRaises(IntegrityError), transaction.atomic():
+            action()
+
+
+class PassportPolicyTests(TestCase):
+    def test_exact_singleton_is_seeded_and_immutable(self):
+        self.assertEqual(PassportPolicy.objects.count(), 1)
+        policy = PassportPolicy.objects.get()
+        self.assertEqual(policy.id, PASSPORT_POLICY_ID)
+        self.assertEqual(policy.code, PASSPORT_POLICY_CODE)
+        self.assertEqual(policy.revision, PASSPORT_POLICY_REVISION)
+
+        with self.assertRaises(IntegrityError), transaction.atomic():
+            PassportPolicy.objects.filter(pk=policy.pk).update(revision="V2")
+        with self.assertRaises(IntegrityError), transaction.atomic():
+            PassportPolicy.objects.filter(pk=policy.pk).delete()
+        with self.assertRaises(IntegrityError), transaction.atomic():
+            PassportPolicy.objects.create(
+                id=uuid.uuid4(),
+                code=PASSPORT_POLICY_CODE,
+                revision=PASSPORT_POLICY_REVISION,
+            )
+
+
+class EntryFactRevisionTests(EntryFactFixtureMixin, TestCase):
+    @classmethod
+    def setUpTestData(cls):
+        cls.register_sources()
+
+    def test_validated_entry_projection_is_typed_utc_and_raw_free(self):
+        run, artifact, attempt = self.make_parse_run()
+        fact = EntryFactRevision.objects.create(**self.fact_values(run, attempt))
+
+        self.assertEqual(fact.country_id, JP_COUNTRY_ID)
+        self.assertEqual(fact.passport_policy_id, PASSPORT_POLICY_ID)
+        self.assertEqual(fact.parse_run_id, run.id)
+        self.assertEqual(fact.ordinary_passport_period_text, PERIOD_TEXT)
+        self.assertEqual(fact.basis_text, BASIS_TEXT)
+        self.assertEqual(fact.snapshot_date, date(2025, 1, 20))
+        self.assertEqual(fact.first_observed_at, attempt.completed_at)
+        self.assertEqual(fact.first_observed_at.utcoffset(), timedelta(0))
+        self.assertEqual(fact.created_at.utcoffset(), timedelta(0))
+        self.assertEqual(
+            fact.typed_fingerprint_sha256,
+            typed_fingerprint_sha256(PERIOD_TEXT, BASIS_TEXT),
+        )
+        self.assertEqual(fact.parse_run.artifact_id, artifact.id)
+
+        fields = {field.name: field for field in EntryFactRevision._meta.fields}
+        self.assertEqual(
+            set(fields),
+            {
+                "id",
+                "country",
+                "passport_policy",
+                "parse_run",
+                "ordinary_passport_period_text",
+                "basis_text",
+                "snapshot_date",
+                "first_observed_at",
+                "typed_fingerprint_sha256",
+                "created_at",
+            },
+        )
+        self.assertFalse(
+            any(
+                isinstance(
+                    field,
+                    (models.BinaryField, models.JSONField, models.FileField),
+                )
+                for field in fields.values()
+            )
+        )
+        banned = {
+            "possible",
+            "allowed",
+            "denied",
+            "purpose",
+            "effective",
+            "expiry",
+            "destination",
+            "departure",
+            "arrival",
+            "raw",
+            "numeric_stay_limit",
+        }
+        self.assertFalse(
+            any(token in field_name for field_name in fields for token in banned)
+        )
+        self.assertIs(fields["country"].remote_field.on_delete, models.PROTECT)
+        self.assertIs(
+            fields["passport_policy"].remote_field.on_delete,
+            models.PROTECT,
+        )
+        self.assertIs(fields["parse_run"].remote_field.on_delete, models.PROTECT)
+
+    def test_fact_revision_update_and_delete_are_rejected(self):
+        run, _, attempt = self.make_parse_run()
+        fact = EntryFactRevision.objects.create(**self.fact_values(run, attempt))
+        self.assert_integrity_error(
+            lambda: EntryFactRevision.objects.filter(pk=fact.pk).update(
+                basis_text="일반여권 소지자 : changed"
+            )
+        )
+        self.assert_integrity_error(
+            lambda: EntryFactRevision.objects.filter(pk=fact.pk).delete()
+        )
+
+    def test_only_validated_approved_entry_parse_and_artifact_are_accepted(self):
+        started_run, _, started_attempt = self.make_parse_run(terminal=False)
+        self.assert_integrity_error(
+            lambda: EntryFactRevision.objects.create(
+                **self.fact_values(started_run, started_attempt)
+            )
+        )
+
+        run, _, attempt = self.make_parse_run(validate_artifact=False)
+        self.assert_integrity_error(
+            lambda: EntryFactRevision.objects.create(**self.fact_values(run, attempt))
+        )
+
+        warning_run, _, warning_attempt = self.make_parse_run(
+            source_code="MOFA_TRAVEL_ALARM_API_JP"
+        )
+        self.assert_integrity_error(
+            lambda: EntryFactRevision.objects.create(
+                **self.fact_values(warning_run, warning_attempt)
+            )
+        )
+
+        for byte_count in (0, 262_145):
+            with self.subTest(byte_count=byte_count):
+                bounded_run, _, bounded_attempt = self.make_parse_run(
+                    byte_count=byte_count
+                )
+                self.assert_integrity_error(
+                    lambda: EntryFactRevision.objects.create(
+                        **self.fact_values(bounded_run, bounded_attempt)
+                    )
+                )
+
+    def test_first_observation_must_match_the_artifact_first_success(self):
+        run, _, attempt = self.make_parse_run()
+        self.assert_integrity_error(
+            lambda: EntryFactRevision.objects.create(
+                **self.fact_values(
+                    run,
+                    attempt,
+                    first_observed_at=attempt.completed_at + timedelta(seconds=1),
+                )
+            )
+        )
+
+        early_run, _, early_attempt = self.make_parse_run(
+            parse_before_attempt=True
+        )
+        self.assert_integrity_error(
+            lambda: EntryFactRevision.objects.create(
+                **self.fact_values(early_run, early_attempt)
+            )
+        )
+
+    def test_later_rights_rejection_blocks_delayed_candidate_materialization(self):
+        run, _, attempt = self.make_parse_run()
+        approval = attempt.rights_decision
+        SourceRightsDecision.objects.create(
+            source=approval.source,
+            source_revision=approval.source_revision,
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
+                approval.contract_fingerprint_sha256
+            ),
+            decided_by="PROJECT_OWNER_REQUEST",
+            decision_basis_code="SYNTHETIC_REJECTION_TEST",
+        )
+
+        self.assert_integrity_error(
+            lambda: EntryFactRevision.objects.create(
+                **self.fact_values(run, attempt)
+            )
+        )
+
+    def test_database_recomputes_the_typed_fingerprint_on_direct_insert(self):
+        run, _, attempt = self.make_parse_run()
+
+        def direct_insert(period_text, basis_text, fingerprint):
+            with connection.cursor() as cursor:
+                cursor.execute(
+                    """
+                    INSERT INTO entry_requirements_entryfactrevision
+                        (id, country_id, passport_policy_id, parse_run_id,
+                         ordinary_passport_period_text, basis_text,
+                         snapshot_date, first_observed_at,
+                         typed_fingerprint_sha256, created_at)
+                    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
+                    """,
+                    [
+                        uuid.uuid4(),
+                        JP_COUNTRY_ID,
+                        PASSPORT_POLICY_ID,
+                        run.id,
+                        period_text,
+                        basis_text,
+                        ENTRY_SNAPSHOT_DATE,
+                        attempt.completed_at,
+                        fingerprint,
+                        timezone.now(),
+                    ],
+                )
+
+        self.assert_integrity_error(
+            lambda: direct_insert(PERIOD_TEXT, BASIS_TEXT, "0" * 64)
+        )
+        self.assert_integrity_error(
+            lambda: direct_insert(
+                PERIOD_TEXT,
+                "일반여권 소지자 : changed typed value",
+                typed_fingerprint_sha256(PERIOD_TEXT, BASIS_TEXT),
+            )
+        )
+
+    def test_database_and_parser_hash_match_for_json_escaped_typed_text(self):
+        run, _, attempt = self.make_parse_run()
+        escaped_basis = (
+            '일반여권 소지자 : quoted "value" \\ path\nsecond line'
+        )
+        fact = EntryFactRevision.objects.create(
+            **self.fact_values(
+                run,
+                attempt,
+                basis_text=escaped_basis,
+                typed_fingerprint_sha256=typed_fingerprint_sha256(
+                    PERIOD_TEXT,
+                    escaped_basis,
+                ),
+            )
+        )
+        self.assertEqual(fact.basis_text, escaped_basis)
+
+    def test_typed_text_snapshot_and_hash_constraints_are_closed(self):
+        overrides = (
+            {"ordinary_passport_period_text": "90"},
+            {"basis_text": "synthetic evidence"},
+            {"snapshot_date": date(2025, 1, 21)},
+            {"typed_fingerprint_sha256": "not-a-hash"},
+            {"country_id": uuid.uuid4()},
+            {"passport_policy_id": uuid.uuid4()},
+        )
+        for override in overrides:
+            with self.subTest(override=tuple(override)):
+                run, _, attempt = self.make_parse_run()
+                self.assert_integrity_error(
+                    lambda values=override: EntryFactRevision.objects.create(
+                        **self.fact_values(run, attempt, **values)
+                    )
+                )
+
+
+class EntryRightsCapabilityTests(EntryFactFixtureMixin, TestCase):
+    def activate_entry_source(self, **rights_overrides):
+        spec = APPROVED_SOURCE_SPECS[0]
+        source = SourceConfiguration.objects.create(
+            **spec.configuration_values(),
+            state=SourceConfiguration.State.DRAFT,
+            enabled=False,
+        )
+        rights_values = spec.rights_values()
+        rights_values.update(rights_overrides)
+        SourceRightsDecision.objects.create(source=source, **rights_values)
+        SourceConfiguration.objects.filter(pk=source.pk).update(
+            state=SourceConfiguration.State.RIGHTS_APPROVED
+        )
+        SourceConfiguration.objects.filter(pk=source.pk).update(
+            state=SourceConfiguration.State.ACTIVE,
+            enabled=True,
+        )
+
+    def assert_candidate_rejected(self):
+        run, _, attempt = self.make_parse_run()
+        self.assert_integrity_error(
+            lambda: EntryFactRevision.objects.create(
+                **self.fact_values(run, attempt)
+            )
+        )
+
+    def test_typed_storage_and_republication_are_required(self):
+        self.activate_entry_source(
+            typed_field_storage_allowed=False,
+            typed_republication_allowed=False,
+            typed_retention=SourceRightsDecision.Retention.NONE,
+            field_scope_code="",
+        )
+        self.assert_candidate_rejected()
+
+    def test_exact_typed_field_scope_is_required(self):
+        self.activate_entry_source(field_scope_code="JP_OTHER_SCOPE")
+        self.assert_candidate_rejected()
+
+    def test_rights_contract_must_match_the_parser_contract(self):
+        self.activate_entry_source(contract_fingerprint_sha256="f" * 64)
+        self.assert_candidate_rejected()
+
+
+class EntryMigrationBoundaryTests(EntryFactFixtureMixin, TransactionTestCase):
+    reset_sequences = True
+
+    def ensure_seed_identities(self):
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
+
+    def setUp(self):
+        self.ensure_seed_identities()
+
+    def _fixture_teardown(self):
+        super()._fixture_teardown()
+        self.ensure_seed_identities()
+
+    def test_reverse_is_blocked_when_a_fact_revision_exists(self):
+        self.register_sources()
+        run, _, attempt = self.make_parse_run()
+        EntryFactRevision.objects.create(**self.fact_values(run, attempt))
+
+        with self.assertRaisesMessage(
+            RuntimeError,
+            "entry boundary rollback requires no fact revisions",
+        ):
+            MigrationExecutor(connection).migrate([("entry_requirements", None)])
+
+        self.assertIn(
+            "entry_requirements_entryfactrevision",
+            connection.introspection.table_names(),
+        )
+
+    def test_empty_reverse_requires_no_external_reference_and_reapplies(self):
+        with connection.cursor() as cursor:
+            cursor.execute(
+                """
+                CREATE TABLE entry_policy_reference_probe (
+                    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
+                    policy_id uuid NOT NULL
+                        REFERENCES entry_requirements_passportpolicy(id)
+                )
+                """
+            )
+            cursor.execute(
+                "INSERT INTO entry_policy_reference_probe (policy_id) VALUES (%s)",
+                [PASSPORT_POLICY_ID],
+            )
+        try:
+            with self.assertRaisesMessage(
+                RuntimeError,
+                "entry boundary rollback requires no fact revisions",
+            ):
+                MigrationExecutor(connection).migrate([("entry_requirements", None)])
+        finally:
+            with connection.cursor() as cursor:
+                cursor.execute("DROP TABLE IF EXISTS entry_policy_reference_probe")
+
+        try:
+            MigrationExecutor(connection).migrate([("entry_requirements", None)])
+            self.assertNotIn(
+                "entry_requirements_passportpolicy",
+                connection.introspection.table_names(),
+            )
+        finally:
+            MigrationExecutor(connection).migrate(
+                [("entry_requirements", "0001_initial")]
+            )
+
+        self.assertEqual(PassportPolicy.objects.count(), 1)
diff --git a/entry_requirements/tests/test_parser.py b/entry_requirements/tests/test_parser.py
new file mode 100644
index 0000000..63ae879
--- /dev/null
+++ b/entry_requirements/tests/test_parser.py
@@ -0,0 +1,296 @@
+import csv
+import hashlib
+import io
+from dataclasses import fields
+from functools import lru_cache
+
+from django.test import SimpleTestCase
+
+from entry_requirements.parser import (
+    ENTRY_HEADERS,
+    ENTRY_SCHEMA_FINGERPRINT_SHA256,
+    MAX_ENTRY_CSV_BYTES,
+    EntryParseResult,
+    EntryProjection,
+    parse_entry_csv,
+    schema_fingerprint,
+    typed_fingerprint_sha256,
+)
+from sources.models import ParseRun
+
+
+VALIDATION_MARKER_SHA256 = (
+    "18f5384d58bcb1bba0bcd9e6a6781d1a6ac2cc280c330ecbab6cb7931b721552"
+)
+PERIOD_TEXT = "90일"
+BASIS_TEXT = "일반여권 소지자 : synthetic evidence"
+
+
+@lru_cache(maxsize=1)
+def validation_marker() -> str:
+    for codepoint in range(0x110000):
+        if 0xD800 <= codepoint <= 0xDFFF:
+            continue
+        candidate = chr(codepoint)
+        if hashlib.sha256(candidate.encode("utf-8")).hexdigest() != VALIDATION_MARKER_SHA256:
+            continue
+        try:
+            candidate.encode("cp949", errors="strict")
+        except UnicodeEncodeError:
+            break
+        return candidate
+    raise AssertionError("approved validation marker could not be reconstructed")
+
+
+def make_row(
+    *,
+    country="일본",
+    marker=None,
+    period=PERIOD_TEXT,
+    basis=BASIS_TEXT,
+):
+    row = [""] * len(ENTRY_HEADERS)
+    row[0] = country
+    row[2] = validation_marker() if marker is None else marker
+    row[3] = period
+    row[8] = basis
+    return row
+
+
+def encode_csv(
+    rows,
+    *,
+    headers=ENTRY_HEADERS,
+    delimiter=",",
+    lineterminator="\r\n",
+):
+    output = io.StringIO(newline="")
+    writer = csv.writer(
+        output,
+        delimiter=delimiter,
+        lineterminator=lineterminator,
+    )
+    writer.writerow(headers)
+    writer.writerows(rows)
+    return output.getvalue().encode("cp949", errors="strict")
+
+
+class EntryCsvParserTests(SimpleTestCase):
+    def assert_quarantine(self, result, failure_code, observed=ENTRY_SCHEMA_FINGERPRINT_SHA256):
+        self.assertEqual(result.outcome, ParseRun.Outcome.QUARANTINED)
+        self.assertEqual(result.failure_code, failure_code)
+        self.assertEqual(result.observed_schema_fingerprint_sha256, observed)
+        self.assertIsNone(result.projection)
+
+    def test_positive_projection_is_exact_typed_and_deterministic(self):
+        ignored_row = make_row(
+            country="synthetic unmapped country",
+            marker="",
+            period="",
+            basis="",
+        )
+        payload = encode_csv([ignored_row, make_row()])
+
+        first = parse_entry_csv(payload)
+        second = parse_entry_csv(payload)
+
+        self.assertEqual(first, second)
+        self.assertEqual(first.outcome, ParseRun.Outcome.VALIDATED)
+        self.assertEqual(first.failure_code, "")
+        self.assertEqual(
+            first.observed_schema_fingerprint_sha256,
+            ENTRY_SCHEMA_FINGERPRINT_SHA256,
+        )
+        self.assertIsNotNone(first.projection)
+        projection = first.projection
+        self.assertEqual(projection.country_iso2, "JP")
+        self.assertEqual(
+            projection.passport_policy_code,
+            "KOR_ORDINARY_SHORT_TOURISM",
+        )
+        self.assertEqual(projection.passport_policy_revision, "V1")
+        self.assertEqual(projection.ordinary_passport_period_text, PERIOD_TEXT)
+        self.assertEqual(projection.basis_text, BASIS_TEXT)
+        self.assertEqual(projection.snapshot_date.isoformat(), "2025-01-20")
+        self.assertEqual(
+            projection.typed_fingerprint_sha256,
+            typed_fingerprint_sha256(PERIOD_TEXT, BASIS_TEXT),
+        )
+
+    def test_quoted_commas_and_newlines_are_parsed_without_changing_typed_hash(self):
+        quoted_basis = BASIS_TEXT + ", synthetic detail\nsecond line"
+        crlf = parse_entry_csv(encode_csv([make_row(basis=quoted_basis)]))
+        lf = parse_entry_csv(
+            encode_csv([make_row(basis=quoted_basis)], lineterminator="\n")
+        )
+
+        self.assertEqual(crlf.outcome, ParseRun.Outcome.VALIDATED)
+        self.assertEqual(lf.outcome, ParseRun.Outcome.VALIDATED)
+        self.assertEqual(crlf.projection, lf.projection)
+        self.assertEqual(crlf.projection.basis_text, quoted_basis)
+
+    def test_strict_cp949_decode_and_byte_bound_fail_closed(self):
+        self.assert_quarantine(
+            parse_entry_csv(b"\x81"),
+            ParseRun.FailureCode.ENCODING_ERROR,
+            observed="",
+        )
+        self.assert_quarantine(
+            parse_entry_csv(b"x" * (MAX_ENTRY_CSV_BYTES + 1)),
+            ParseRun.FailureCode.SYNTAX_ERROR,
+            observed="",
+        )
+
+    def test_header_order_extra_and_missing_are_schema_mismatches(self):
+        variants = (
+            ENTRY_HEADERS[1:] + ENTRY_HEADERS[:1],
+            ENTRY_HEADERS + ("synthetic extra",),
+            ENTRY_HEADERS[:-1],
+        )
+        for headers in variants:
+            with self.subTest(header_count=len(headers)):
+                result = parse_entry_csv(encode_csv([], headers=headers))
+                self.assertEqual(result.outcome, ParseRun.Outcome.QUARANTINED)
+                self.assertEqual(
+                    result.failure_code,
+                    ParseRun.FailureCode.SCHEMA_MISMATCH,
+                )
+                self.assertEqual(
+                    result.observed_schema_fingerprint_sha256,
+                    schema_fingerprint(headers),
+                )
+                self.assertNotEqual(
+                    result.observed_schema_fingerprint_sha256,
+                    ENTRY_SCHEMA_FINGERPRINT_SHA256,
+                )
+                self.assertIsNone(result.projection)
+
+    def test_row_width_and_csv_syntax_are_quarantined_without_observed_schema(self):
+        short_row = make_row()[:-1]
+        self.assert_quarantine(
+            parse_entry_csv(encode_csv([short_row])),
+            ParseRun.FailureCode.SYNTAX_ERROR,
+            observed="",
+        )
+        malformed = encode_csv([]) + b'"unterminated'
+        self.assert_quarantine(
+            parse_entry_csv(malformed),
+            ParseRun.FailureCode.SYNTAX_ERROR,
+            observed="",
+        )
+
+    def test_no_jp_duplicate_jp_and_conflicting_jp_are_distinct(self):
+        self.assert_quarantine(
+            parse_entry_csv(encode_csv([make_row(country="unmapped")])) ,
+            ParseRun.FailureCode.IDENTITY_MISMATCH,
+        )
+        self.assert_quarantine(
+            parse_entry_csv(encode_csv([make_row(), make_row()])),
+            ParseRun.FailureCode.DUPLICATE_RECORD,
+        )
+        self.assert_quarantine(
+            parse_entry_csv(
+                encode_csv([make_row(), make_row(period="91일")])
+            ),
+            ParseRun.FailureCode.CONFLICTING_VALUE,
+        )
+
+    def test_required_marker_period_and_basis_are_not_returned_on_missing_values(self):
+        missing_rows = (
+            make_row(marker=""),
+            make_row(period=""),
+            make_row(basis=""),
+        )
+        for row in missing_rows:
+            with self.subTest(empty_index=row.index("")):
+                self.assert_quarantine(
+                    parse_entry_csv(encode_csv([row])),
+                    ParseRun.FailureCode.REQUIRED_VALUE_MISSING,
+                )
+
+    def test_marker_unit_and_basis_statement_are_exactly_validated(self):
+        marker = validation_marker()
+        different_marker = next(candidate for candidate in ("X", "Y", "Z") if candidate != marker)
+        invalid_rows = (
+            make_row(marker=different_marker),
+            make_row(period="90"),
+            make_row(period="0일"),
+            make_row(period="9" * 32 + "일"),
+            make_row(basis="synthetic evidence"),
+            make_row(basis="일반여권 소지자 : " + "가" * 2000),
+        )
+        for row in invalid_rows:
+            with self.subTest():
+                self.assert_quarantine(
+                    parse_entry_csv(encode_csv([row])),
+                    ParseRun.FailureCode.INVALID_VALUE,
+                )
+
+    def test_typed_text_lengths_accept_the_exact_model_boundaries(self):
+        period = "9" * 31 + "일"
+        basis_prefix = "일반여권 소지자 : "
+        basis = basis_prefix + "가" * (2000 - len(basis_prefix))
+
+        result = parse_entry_csv(
+            encode_csv([make_row(period=period, basis=basis)])
+        )
+
+        self.assertEqual(result.outcome, ParseRun.Outcome.VALIDATED)
+        self.assertEqual(len(result.projection.ordinary_passport_period_text), 32)
+        self.assertEqual(len(result.projection.basis_text), 2000)
+
+    def test_outputs_and_model_boundary_have_no_boolean_raw_or_trip_fields(self):
+        projection_fields = {field.name for field in fields(EntryProjection)}
+        result_fields = {field.name for field in fields(EntryParseResult)}
+        self.assertEqual(
+            projection_fields,
+            {
+                "country_iso2",
+                "passport_policy_code",
+                "passport_policy_revision",
+                "ordinary_passport_period_text",
+                "basis_text",
+                "snapshot_date",
+                "typed_fingerprint_sha256",
+            },
+        )
+        self.assertEqual(
+            result_fields,
+            {
+                "outcome",
+                "failure_code",
+                "observed_schema_fingerprint_sha256",
+                "projection",
+            },
+        )
+        banned = {
+            "possible",
+            "allowed",
+            "denied",
+            "boolean",
+            "raw",
+            "purpose",
+            "effective",
+            "expiry",
+            "destination",
+            "departure",
+            "arrival",
+        }
+        self.assertFalse(
+            any(token in name for name in projection_fields | result_fields for token in banned)
+        )
+
+    def test_result_codes_are_closed_to_parse_run_choices(self):
+        outcomes = {value for value, _ in ParseRun.Outcome.choices}
+        failure_codes = {value for value, _ in ParseRun.FailureCode.choices}
+        results = (
+            parse_entry_csv(encode_csv([make_row()])),
+            parse_entry_csv(b"\x81"),
+            parse_entry_csv(encode_csv([])),
+            parse_entry_csv(object()),
+        )
+        for result in results:
+            self.assertIn(result.outcome, outcomes)
+            if result.failure_code:
+                self.assertIn(result.failure_code, failure_codes)
+            self.assertEqual(result.projection is not None, result.outcome == "VALIDATED")
diff --git a/travel_readiness/settings.py b/travel_readiness/settings.py
index d52a632..c835678 100644
--- a/travel_readiness/settings.py
+++ b/travel_readiness/settings.py
@@ -37,6 +37,7 @@ INSTALLED_APPS = [
     "operations",
     "countries",
     "sources",
+    "entry_requirements",
     "travel_warnings",
     "django.contrib.admin",
     "django.contrib.auth",


