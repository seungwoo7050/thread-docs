## `feat(source): version live aviation envelope contract`

diff --git a/scripts/postgresql-common b/scripts/postgresql-common
index 3df9076..fad7eee 100644
--- a/scripts/postgresql-common
+++ b/scripts/postgresql-common
@@ -7,9 +7,9 @@ POSTGRESQL_REQUIRED_VERSION="18.6"
 EXPECTED_PUBLIC_SCHEMA_COMMENT_SHA256="998e3ff6338d1b13197feaef3c7657578604caf2185d3fe9e5ac04ba171682c8"
 
 EXPECTED_SCHEMA_DIGESTS='schema.columns.sha256=29898303816ef6ae9b01d88fa885e214f6d00b4e055e7d4e7a3ed45655ec6e66
-schema.constraints.sha256=1e29e5d6235ce3cea37ac29d4f8f0a6d0855cd30b66e02601fc56dccfe2e73e7
+schema.constraints.sha256=afb800d818f6a0c428786c6311c8cb5e54f5f4fe35fbde4f51d636931d18aef4
 schema.indexes.sha256=1420eb0740b9175ffe9ee40ab827689d8b5e83f6cde2e8224df277c7f09f1025
-schema.trigger_functions.sha256=7d42d4ded3775f02a3cda5b1fa18ed1cc4cd92da4c48f3a1be3e4bd4e95d1b8f
+schema.trigger_functions.sha256=14d6611459b45315aae8aa82f520f1083fbe3a03f72da0f034878901f5bc056e
 schema.triggers.sha256=fc1411aec1f0e0e6268e7ddb7a5a6e53a465214ce22c24bc9791f05a54122f5b'
 
 EXPECTED_SEQUENCE_DEFINITION_SHA256="e9303b2ecb873b98391c36822e7ce66ee8c14062e5f1bf20c9e50f697f684152"
@@ -99,7 +99,7 @@ sources_apply_rights_rejection|e02368696d3a102bd0c7cbfd2f1078b58c4d63f96046c625e
 sources_guard_artifact_mutation|59039e6f2c37fa65deeea03b88d737aaa6c841fce8f00e173db1d3618736b485
 sources_guard_configuration_change|1eb4fbbf3cfe95d940364bc8f36d278b044739409100df1b5a9ad42049334fc7
 sources_guard_fetch_attempt_mutation|78195da0ef6c4dfc3012edfbc98dcb8323ee2d09fd1e52c0d1f999b5391c98c4
-sources_guard_parse_run_change|005215b4b25729564194cbffddbaa09f30c104f2cb1b4b9f51982caa5167fc11
+sources_guard_parse_run_change|87001036fbaf43537034a8ab8a335b5908bb1a06a7adb81bc13ff11a44460397
 sources_guard_parse_run_input|b66c4ec9a246ed1a07d645b7b71f6028f517e6d502430913095cef60601a7a0c
 sources_reject_rights_mutation|76be8ad85eab7939fe21596429e52e83ad1c7e1a1cac6954244a52606a730d46
 sources_reject_rights_revision_reuse|f96554369f58cd3017114acfd0e8772c9c7fd0252d8ebcbf493359af36310fad
diff --git a/sources/management/commands/register_approved_sources.py b/sources/management/commands/register_approved_sources.py
index 2f97b34..c8be003 100644
--- a/sources/management/commands/register_approved_sources.py
+++ b/sources/management/commands/register_approved_sources.py
@@ -1,10 +1,11 @@
-from dataclasses import dataclass
+from dataclasses import dataclass, replace
 
 from django.core.management.base import BaseCommand, CommandError
 from django.db import DatabaseError, connection, transaction
 
 from sources.models import SourceConfiguration, SourceRightsDecision
 from travel_windows.contracts import (
+    AVIATION_PRIOR_SOURCE_REVISION,
     CITY_ATTRIBUTION,
     CITY_CONTRACT_FINGERPRINT_SHA256,
     CITY_FIELD_SCOPE,
@@ -12,6 +13,7 @@ from travel_windows.contracts import (
     CITY_SOURCE_LOCATOR,
     CITY_SOURCE_OWNER,
     CITY_SOURCE_REVISION,
+    CITY_V1_CONTRACT_FINGERPRINT_SHA256,
     DATA_GO_SECRET_REFERENCE,
     DURATION_ATTRIBUTION,
     DURATION_CONTRACT_FINGERPRINT_SHA256,
@@ -27,6 +29,7 @@ from travel_windows.contracts import (
     LEGACY_SCHEDULE_SOURCE_LOCATOR,
     LEGACY_SCHEDULE_SOURCE_OWNER,
     LEGACY_SCHEDULE_SOURCE_REVISION,
+    LEGACY_SCHEDULE_V1_CONTRACT_FINGERPRINT_SHA256,
     SCHEDULE_ATTRIBUTION,
     SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
     SCHEDULE_FIELD_SCOPE,
@@ -34,6 +37,7 @@ from travel_windows.contracts import (
     SCHEDULE_SOURCE_LOCATOR,
     SCHEDULE_SOURCE_OWNER,
     SCHEDULE_SOURCE_REVISION,
+    SCHEDULE_V1_CONTRACT_FINGERPRINT_SHA256,
 )
 
 
@@ -94,6 +98,21 @@ class ApprovedSourceSpec:
         }
 
 
+def _with_prior_aviation_contract(
+    spec: ApprovedSourceSpec,
+    *,
+    fingerprint: str,
+) -> ApprovedSourceSpec:
+    prior = replace(
+        spec,
+        revision=AVIATION_PRIOR_SOURCE_REVISION,
+        contract_fingerprint_sha256=fingerprint,
+        decision_basis_code="TRAVEL_OPPORTUNITY_CONTRACT_20260831",
+        prior_contracts=(),
+    )
+    return replace(spec, prior_contracts=(prior,))
+
+
 APPROVED_SOURCE_SPECS = (
     ApprovedSourceSpec(
         code="MOFA_ENTRY_CSV",
@@ -167,46 +186,55 @@ APPROVED_SOURCE_SPECS = (
         decision_basis_code="USER_TRAVEL6_SCOPE_20260831",
         attribution_text="외교부|공공데이터포털",
     ),
-    ApprovedSourceSpec(
-        code=SCHEDULE_SOURCE_CODE,
-        revision=SCHEDULE_SOURCE_REVISION,
-        module=SourceConfiguration.Module.AVIATION,
-        owner=SCHEDULE_SOURCE_OWNER,
-        official_locator=SCHEDULE_SOURCE_LOCATOR,
-        secret_reference_name=DATA_GO_SECRET_REFERENCE,
-        access_mode=SourceRightsDecision.AccessMode.CREDENTIAL_REFERENCE,
-        field_scope_code=SCHEDULE_FIELD_SCOPE,
-        contract_fingerprint_sha256=SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
-        decision_basis_code="TRAVEL_OPPORTUNITY_CONTRACT_20260831",
-        attribution_text=SCHEDULE_ATTRIBUTION,
+    _with_prior_aviation_contract(
+        ApprovedSourceSpec(
+            code=SCHEDULE_SOURCE_CODE,
+            revision=SCHEDULE_SOURCE_REVISION,
+            module=SourceConfiguration.Module.AVIATION,
+            owner=SCHEDULE_SOURCE_OWNER,
+            official_locator=SCHEDULE_SOURCE_LOCATOR,
+            secret_reference_name=DATA_GO_SECRET_REFERENCE,
+            access_mode=SourceRightsDecision.AccessMode.CREDENTIAL_REFERENCE,
+            field_scope_code=SCHEDULE_FIELD_SCOPE,
+            contract_fingerprint_sha256=SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
+            decision_basis_code="LIVE_AVIATION_ENVELOPE_20260831",
+            attribution_text=SCHEDULE_ATTRIBUTION,
+        ),
+        fingerprint=SCHEDULE_V1_CONTRACT_FINGERPRINT_SHA256,
     ),
-    ApprovedSourceSpec(
-        code=CITY_SOURCE_CODE,
-        revision=CITY_SOURCE_REVISION,
-        module=SourceConfiguration.Module.AVIATION,
-        owner=CITY_SOURCE_OWNER,
-        official_locator=CITY_SOURCE_LOCATOR,
-        secret_reference_name=DATA_GO_SECRET_REFERENCE,
-        access_mode=SourceRightsDecision.AccessMode.CREDENTIAL_REFERENCE,
-        field_scope_code=CITY_FIELD_SCOPE,
-        contract_fingerprint_sha256=CITY_CONTRACT_FINGERPRINT_SHA256,
-        decision_basis_code="TRAVEL_OPPORTUNITY_CONTRACT_20260831",
-        attribution_text=CITY_ATTRIBUTION,
+    _with_prior_aviation_contract(
+        ApprovedSourceSpec(
+            code=CITY_SOURCE_CODE,
+            revision=CITY_SOURCE_REVISION,
+            module=SourceConfiguration.Module.AVIATION,
+            owner=CITY_SOURCE_OWNER,
+            official_locator=CITY_SOURCE_LOCATOR,
+            secret_reference_name=DATA_GO_SECRET_REFERENCE,
+            access_mode=SourceRightsDecision.AccessMode.CREDENTIAL_REFERENCE,
+            field_scope_code=CITY_FIELD_SCOPE,
+            contract_fingerprint_sha256=CITY_CONTRACT_FINGERPRINT_SHA256,
+            decision_basis_code="LIVE_AVIATION_ENVELOPE_20260831",
+            attribution_text=CITY_ATTRIBUTION,
+        ),
+        fingerprint=CITY_V1_CONTRACT_FINGERPRINT_SHA256,
     ),
-    ApprovedSourceSpec(
-        code=LEGACY_SCHEDULE_SOURCE_CODE,
-        revision=LEGACY_SCHEDULE_SOURCE_REVISION,
-        module=SourceConfiguration.Module.AVIATION,
-        owner=LEGACY_SCHEDULE_SOURCE_OWNER,
-        official_locator=LEGACY_SCHEDULE_SOURCE_LOCATOR,
-        secret_reference_name=DATA_GO_SECRET_REFERENCE,
-        access_mode=SourceRightsDecision.AccessMode.CREDENTIAL_REFERENCE,
-        field_scope_code=LEGACY_SCHEDULE_FIELD_SCOPE,
-        contract_fingerprint_sha256=(
-            LEGACY_SCHEDULE_CONTRACT_FINGERPRINT_SHA256
+    _with_prior_aviation_contract(
+        ApprovedSourceSpec(
+            code=LEGACY_SCHEDULE_SOURCE_CODE,
+            revision=LEGACY_SCHEDULE_SOURCE_REVISION,
+            module=SourceConfiguration.Module.AVIATION,
+            owner=LEGACY_SCHEDULE_SOURCE_OWNER,
+            official_locator=LEGACY_SCHEDULE_SOURCE_LOCATOR,
+            secret_reference_name=DATA_GO_SECRET_REFERENCE,
+            access_mode=SourceRightsDecision.AccessMode.CREDENTIAL_REFERENCE,
+            field_scope_code=LEGACY_SCHEDULE_FIELD_SCOPE,
+            contract_fingerprint_sha256=(
+                LEGACY_SCHEDULE_CONTRACT_FINGERPRINT_SHA256
+            ),
+            decision_basis_code="LIVE_AVIATION_ENVELOPE_20260831",
+            attribution_text=LEGACY_SCHEDULE_ATTRIBUTION,
         ),
-        decision_basis_code="TRAVEL_OPPORTUNITY_CONTRACT_20260831",
-        attribution_text=LEGACY_SCHEDULE_ATTRIBUTION,
+        fingerprint=LEGACY_SCHEDULE_V1_CONTRACT_FINGERPRINT_SHA256,
     ),
     ApprovedSourceSpec(
         code=DURATION_SOURCE_CODE,
diff --git a/sources/migrations/0014_live_aviation_envelope_contracts.py b/sources/migrations/0014_live_aviation_envelope_contracts.py
new file mode 100644
index 0000000..e9ab84d
--- /dev/null
+++ b/sources/migrations/0014_live_aviation_envelope_contracts.py
@@ -0,0 +1,192 @@
+from django.db import migrations, models
+from django.db.models import Q
+
+
+FUNCTION_NAME = "sources_guard_parse_run_change"
+
+OLD_SCHEDULE_CLAUSE = """            OR (source_module = 'AVIATION'
+                AND NEW.parser_name = 'ICN_FLIGHT_SCHEDULE_JSON'
+                AND NEW.parser_version = 'V1'
+                AND NEW.parser_contract_fingerprint_sha256 =
+                    '3b4295504d24cfb1e0da398399d61c328afa0f8124d7c298f5d4e4f950dfd372'
+                AND NEW.expected_schema_fingerprint_sha256 =
+                    '3d8d37c4a23731d11ed6c3b3ff2d87324c9dcbde2933c158b5f4ace689b82074')"""
+NEW_SCHEDULE_CLAUSE = """            OR (source_module = 'AVIATION'
+                AND NEW.parser_name = 'ICN_FLIGHT_SCHEDULE_JSON'
+                AND NEW.parser_version = 'V2'
+                AND NEW.parser_contract_fingerprint_sha256 =
+                    '8075bd2c7b1bc8ba80585fbc6c658163d1052f909b1e8434b0e00d847927dc3c'
+                AND NEW.expected_schema_fingerprint_sha256 =
+                    '8cf49088b3cf00ff92386f02b3ab6feea2b9ca9118dd616eae76cb439e1df9b7')"""
+OLD_CITY_CLAUSE = """            OR (source_module = 'AVIATION'
+                AND NEW.parser_name = 'ICN_DESTINATION_CITY_JSON'
+                AND NEW.parser_version = 'V1'
+                AND NEW.parser_contract_fingerprint_sha256 =
+                    '18a1baa88e4077dbb64a4108620d046b24882dad72e6ec0d56629396391d669f'
+                AND NEW.expected_schema_fingerprint_sha256 =
+                    '907670feb0293b0a1a40e6d7065cae2c803e4ba458828b22631dbee002017c18')"""
+NEW_CITY_CLAUSE = """            OR (source_module = 'AVIATION'
+                AND NEW.parser_name = 'ICN_DESTINATION_CITY_JSON'
+                AND NEW.parser_version = 'V2'
+                AND NEW.parser_contract_fingerprint_sha256 =
+                    '9e274716712f0ca14374c99d9f9d36a3569ff6f89b74155e3dc62ec65b1c1b1e'
+                AND NEW.expected_schema_fingerprint_sha256 =
+                    '06475295174c264253686a12d4a15cb905c88ba730eaf6e8ef58652170522703')"""
+OLD_LEGACY_CLAUSE = """            OR (source_module = 'AVIATION'
+                AND NEW.parser_name = 'ICN_LEGACY_ARRIVALS_JSON'
+                AND NEW.parser_version = 'V1'
+                AND NEW.parser_contract_fingerprint_sha256 =
+                    'a88a91166bb7ea567f59637159f07f2285981f2ae0588e921ec2882e1c73bbfb'
+                AND NEW.expected_schema_fingerprint_sha256 =
+                    'e32b4246a27c5266b52aa0ea24923018eb7d474d09cab1cbcc36ebb85f32ebbd')"""
+NEW_LEGACY_CLAUSE = """            OR (source_module = 'AVIATION'
+                AND NEW.parser_name = 'ICN_LEGACY_ARRIVALS_JSON'
+                AND NEW.parser_version = 'V2'
+                AND NEW.parser_contract_fingerprint_sha256 =
+                    '89d910af859faf83cdf18114134b7fd8229dca990b8d5a866aa5db4d1cb6574d'
+                AND NEW.expected_schema_fingerprint_sha256 =
+                    '4b0730f761634d97d61617437086978e815ae288dfafd34e21c67d704c108fa1')"""
+
+
+def _function_body(schema_editor):
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
+    if len(rows) != 1:
+        raise RuntimeError(f"expected one trigger function named {FUNCTION_NAME}")
+    return rows[0][0]
+
+
+def _replace_clauses(schema_editor, pairs):
+    body = _function_body(schema_editor)
+    for old, new in pairs:
+        if body.count(old) != 1:
+            raise RuntimeError(f"unexpected {FUNCTION_NAME} trigger definition")
+        body = body.replace(old, new)
+    quoted_name = schema_editor.quote_name(FUNCTION_NAME)
+    with schema_editor.connection.cursor() as cursor:
+        cursor.execute(
+            f"""
+            CREATE OR REPLACE FUNCTION {quoted_name}() RETURNS trigger
+            LANGUAGE plpgsql AS $live_aviation_envelopes$
+            {body}
+            $live_aviation_envelopes$
+            """
+        )
+
+
+def allow_live_aviation_contracts(apps, schema_editor):
+    _replace_clauses(
+        schema_editor,
+        (
+            (OLD_SCHEDULE_CLAUSE, OLD_SCHEDULE_CLAUSE + "\n" + NEW_SCHEDULE_CLAUSE),
+            (OLD_CITY_CLAUSE, OLD_CITY_CLAUSE + "\n" + NEW_CITY_CLAUSE),
+            (OLD_LEGACY_CLAUSE, OLD_LEGACY_CLAUSE + "\n" + NEW_LEGACY_CLAUSE),
+        ),
+    )
+
+
+def restore_prior_aviation_contracts(apps, schema_editor):
+    FetchAttempt = apps.get_model("sources", "FetchAttempt")
+    ParseRun = apps.get_model("sources", "ParseRun")
+    SourceConfiguration = apps.get_model("sources", "SourceConfiguration")
+    SourceRightsDecision = apps.get_model("sources", "SourceRightsDecision")
+    if (
+        ParseRun.objects.filter(parser_version="V2").exists()
+        or FetchAttempt.objects.filter(source_revision="travel-v2").exists()
+        or SourceRightsDecision.objects.filter(
+            source_revision="travel-v2"
+        ).exists()
+        or SourceConfiguration.objects.filter(
+            module="AVIATION",
+            revision="travel-v2",
+        ).exists()
+    ):
+        raise RuntimeError(
+            "live aviation contract rollback requires no V2 source evidence"
+        )
+    _replace_clauses(
+        schema_editor,
+        (
+            (OLD_SCHEDULE_CLAUSE + "\n" + NEW_SCHEDULE_CLAUSE, OLD_SCHEDULE_CLAUSE),
+            (OLD_CITY_CLAUSE + "\n" + NEW_CITY_CLAUSE, OLD_CITY_CLAUSE),
+            (OLD_LEGACY_CLAUSE + "\n" + NEW_LEGACY_CLAUSE, OLD_LEGACY_CLAUSE),
+        ),
+    )
+
+
+class Migration(migrations.Migration):
+    dependencies = [("sources", "0013_aviation_parse_inputs")]
+
+    operations = [
+        migrations.RemoveConstraint(
+            model_name="sourceconfiguration",
+            name="source_aviation_approved_activation",
+        ),
+        migrations.AddConstraint(
+            model_name="sourceconfiguration",
+            constraint=models.CheckConstraint(
+                condition=(
+                    ~Q(module="AVIATION")
+                    | Q(state="DRAFT", enabled=False)
+                    | Q(
+                        code__in=(
+                            "ICN_SCHEDULE_API",
+                            "ICN_DESTINATION_CITY_API_15095067",
+                            "ICN_LEGACY_ARRIVALS_API",
+                        ),
+                        revision__in=("travel-v1", "travel-v2"),
+                        secret_reference_name="DATA_GO_KR_SERVICE_KEY",
+                    )
+                    | Q(
+                        code="ICN_DESTINATION_CITY_API",
+                        revision="travel-v1",
+                        secret_reference_name="DATA_GO_KR_SERVICE_KEY",
+                    )
+                    | Q(
+                        code__in=(
+                            "PORT_LOGISTICS_ROUTE_DURATION",
+                            "PORT_LOGISTICS_ROUTE_DURATION_15151728",
+                        ),
+                        revision="travel-v1",
+                        secret_reference_name="",
+                    )
+                ),
+                name="source_aviation_approved_activation",
+            ),
+        ),
+        migrations.AlterField(
+            model_name="parserun",
+            name="parser_version",
+            field=models.CharField(
+                choices=[("V1", "Version 1"), ("V2", "Version 2")],
+                max_length=64,
+            ),
+        ),
+        migrations.RemoveConstraint(
+            model_name="parserun",
+            name="parse_version_allowlist",
+        ),
+        migrations.AddConstraint(
+            model_name="parserun",
+            constraint=models.CheckConstraint(
+                condition=Q(parser_version__in=("V1", "V2")),
+                name="parse_version_allowlist",
+            ),
+        ),
+        migrations.RunPython(
+            allow_live_aviation_contracts,
+            restore_prior_aviation_contracts,
+        ),
+    ]
diff --git a/sources/models.py b/sources/models.py
index 3446906..ec3035c 100644
--- a/sources/models.py
+++ b/sources/models.py
@@ -89,10 +89,14 @@ class SourceConfiguration(models.Model):
                     | Q(
                         code__in=[
                             "ICN_SCHEDULE_API",
-                            "ICN_DESTINATION_CITY_API",
                             "ICN_DESTINATION_CITY_API_15095067",
                             "ICN_LEGACY_ARRIVALS_API",
                         ],
+                        revision__in=["travel-v1", "travel-v2"],
+                        secret_reference_name="DATA_GO_KR_SERVICE_KEY",
+                    )
+                    | Q(
+                        code="ICN_DESTINATION_CITY_API",
                         revision="travel-v1",
                         secret_reference_name="DATA_GO_KR_SERVICE_KEY",
                     )
@@ -303,6 +307,7 @@ class ParseRun(models.Model):
 
     class ParserVersion(models.TextChoices):
         V1 = "V1", "Version 1"
+        V2 = "V2", "Version 2"
 
     class Outcome(models.TextChoices):
         STARTED = "STARTED", "Started"
@@ -358,7 +363,10 @@ class ParseRun(models.Model):
                 ),
                 name="parse_name_allowlist",
             ),
-            models.CheckConstraint(condition=Q(parser_version="V1"), name="parse_version_allowlist"),
+            models.CheckConstraint(
+                condition=Q(parser_version__in=["V1", "V2"]),
+                name="parse_version_allowlist",
+            ),
             models.CheckConstraint(condition=Q(parser_contract_fingerprint_sha256__regex=r"^[0-9a-f]{64}$"), name="parse_contract_fingerprint_format"),
             models.CheckConstraint(condition=Q(expected_schema_fingerprint_sha256__regex=r"^[0-9a-f]{64}$"), name="parse_expected_schema_format"),
             models.CheckConstraint(
diff --git a/sources/tests/test_aviation_parse_input_migration.py b/sources/tests/test_aviation_parse_input_migration.py
index d14a385..1dbaf2f 100644
--- a/sources/tests/test_aviation_parse_input_migration.py
+++ b/sources/tests/test_aviation_parse_input_migration.py
@@ -7,14 +7,16 @@ from django.test import TransactionTestCase
 from django.utils import timezone
 
 from operations.tests.migration_helpers import restore_canonical_migration_graph
-from sources.management.commands.register_approved_sources import (
-    register_approved_sources,
-)
 from sources.models import ParseRun, ParseRunInput
 from travel_windows.contracts import (
-    SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
-    SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
+    AVIATION_PRIOR_SOURCE_REVISION,
+    SCHEDULE_ATTRIBUTION,
+    SCHEDULE_FIELD_SCOPE,
     SCHEDULE_SOURCE_CODE,
+    SCHEDULE_SOURCE_LOCATOR,
+    SCHEDULE_SOURCE_OWNER,
+    SCHEDULE_V1_CONTRACT_FINGERPRINT_SHA256,
+    SCHEDULE_V1_SCHEMA_FINGERPRINT_SHA256,
 )
 
 
@@ -36,7 +38,6 @@ class AviationParseInputMigrationTests(TransactionTestCase):
         try:
             executor.migrate(self.previous)
             old_apps = executor.loader.project_state(self.previous).apps
-            register_approved_sources(apply=True)
             SourceConfiguration = old_apps.get_model(
                 "sources", "SourceConfiguration"
             )
@@ -47,11 +48,46 @@ class AviationParseInputMigrationTests(TransactionTestCase):
             SourceArtifact = old_apps.get_model("sources", "SourceArtifact")
             OldParseRun = old_apps.get_model("sources", "ParseRun")
 
-            source = SourceConfiguration.objects.get(code=SCHEDULE_SOURCE_CODE)
-            rights = SourceRightsDecision.objects.get(
+            source = SourceConfiguration.objects.create(
+                code=SCHEDULE_SOURCE_CODE,
+                revision=AVIATION_PRIOR_SOURCE_REVISION,
+                module="AVIATION",
+                owner=SCHEDULE_SOURCE_OWNER,
+                official_locator=SCHEDULE_SOURCE_LOCATOR,
+                state="DRAFT",
+                enabled=False,
+                connect_timeout_seconds=5,
+                read_timeout_seconds=15,
+                max_retries=2,
+                secret_reference_name="DATA_GO_KR_SERVICE_KEY",
+            )
+            rights = SourceRightsDecision.objects.create(
                 source=source,
-                source_revision=source.revision,
+                source_revision=AVIATION_PRIOR_SOURCE_REVISION,
+                decision_sequence=1,
                 decision="APPROVED",
+                access_mode="CREDENTIAL_REFERENCE",
+                access_allowed=True,
+                automated_collection_allowed=True,
+                typed_field_storage_allowed=True,
+                raw_body_storage_allowed=False,
+                typed_republication_allowed=True,
+                raw_retention_seconds=0,
+                typed_retention="PRODUCT_HISTORY",
+                evidence_retention="PRODUCT_HISTORY",
+                field_scope_code=SCHEDULE_FIELD_SCOPE,
+                attribution_text=SCHEDULE_ATTRIBUTION,
+                contract_fingerprint_sha256=(
+                    SCHEDULE_V1_CONTRACT_FINGERPRINT_SHA256
+                ),
+                decided_by="PROJECT_OWNER_REQUEST",
+                decision_basis_code="TRAVEL_OPPORTUNITY_CONTRACT_20260831",
+            )
+            SourceConfiguration.objects.filter(pk=source.pk).update(
+                state="RIGHTS_APPROVED"
+            )
+            SourceConfiguration.objects.filter(pk=source.pk).update(
+                state="ACTIVE", enabled=True
             )
             body = b"legacy composite aviation response"
             body_sha256 = hashlib.sha256(body).hexdigest()
@@ -83,17 +119,17 @@ class AviationParseInputMigrationTests(TransactionTestCase):
                 parser_name="ICN_FLIGHT_SCHEDULE_JSON",
                 parser_version="V1",
                 parser_contract_fingerprint_sha256=(
-                    SCHEDULE_CONTRACT_FINGERPRINT_SHA256
+                    SCHEDULE_V1_CONTRACT_FINGERPRINT_SHA256
                 ),
                 expected_schema_fingerprint_sha256=(
-                    SCHEDULE_SCHEMA_FINGERPRINT_SHA256
+                    SCHEDULE_V1_SCHEMA_FINGERPRINT_SHA256
                 ),
             )
             OldParseRun.objects.filter(pk=legacy_run.pk).update(
                 completed_at=timezone.now(),
                 outcome="VALIDATED",
                 observed_schema_fingerprint_sha256=(
-                    SCHEDULE_SCHEMA_FINGERPRINT_SHA256
+                    SCHEDULE_V1_SCHEMA_FINGERPRINT_SHA256
                 ),
             )
 
diff --git a/sources/tests/test_aviation_parse_inputs.py b/sources/tests/test_aviation_parse_inputs.py
index 124912c..ad8ff09 100644
--- a/sources/tests/test_aviation_parse_inputs.py
+++ b/sources/tests/test_aviation_parse_inputs.py
@@ -119,7 +119,7 @@ class AviationParseInputGuardTests(TransactionTestCase):
         run = ParseRun.objects.create(
             artifact=first,
             parser_name=ParseRun.ParserName.ICN_FLIGHT_SCHEDULE_JSON,
-            parser_version=ParseRun.ParserVersion.V1,
+            parser_version=ParseRun.ParserVersion.V2,
             parser_contract_fingerprint_sha256=(
                 SCHEDULE_CONTRACT_FINGERPRINT_SHA256
             ),
@@ -200,7 +200,7 @@ class AviationParseInputGuardTests(TransactionTestCase):
             ParseRun.objects.create(
                 artifact=artifact,
                 parser_name=ParseRun.ParserName.ICN_FLIGHT_SCHEDULE_JSON,
-                parser_version=ParseRun.ParserVersion.V1,
+                parser_version=ParseRun.ParserVersion.V2,
                 parser_contract_fingerprint_sha256=(
                     SCHEDULE_CONTRACT_FINGERPRINT_SHA256
                 ),
diff --git a/sources/tests/test_parse_run.py b/sources/tests/test_parse_run.py
index 87d5668..dda4125 100644
--- a/sources/tests/test_parse_run.py
+++ b/sources/tests/test_parse_run.py
@@ -441,7 +441,7 @@ class ParseRunTests(ParseRunFixtureMixin, TestCase):
         self.assert_integrity_error(
             lambda: ParseRun.objects.create(**self.run_values(self.artifact))
         )
-        self.assertEqual(ParseRun.ParserVersion.values, ["V1"])
+        self.assertEqual(ParseRun.ParserVersion.values, ["V1", "V2"])
 
         identity_constraint = next(
             constraint
diff --git a/sources/tests/test_register_approved_sources.py b/sources/tests/test_register_approved_sources.py
index 9d271ac..8784e50 100644
--- a/sources/tests/test_register_approved_sources.py
+++ b/sources/tests/test_register_approved_sources.py
@@ -76,6 +76,13 @@ class ApprovedSourceRegistrationCommandTests(TransactionTestCase):
         )
 
     def make_upgrade_contracts(self, suffix=""):
+        if not suffix:
+            current = next(
+                spec
+                for spec in registration.APPROVED_SOURCE_SPECS
+                if spec.code == "ICN_SCHEDULE_API"
+            )
+            return current.prior_contracts[0], current
         code_suffix = f"_{suffix}" if suffix else ""
         locator_suffix = suffix.lower() if suffix else "default"
         prior = replace(
diff --git a/travel_windows/contracts.py b/travel_windows/contracts.py
index 503ba52..0bb9e82 100644
--- a/travel_windows/contracts.py
+++ b/travel_windows/contracts.py
@@ -3,9 +3,10 @@ import hashlib
 
 DATA_GO_SECRET_REFERENCE = "DATA_GO_KR_SERVICE_KEY"
 LEGACY_DATA_GO_SECRET_REFERENCE = "MOFA_TRAVEL_ALARM_SERVICE_KEY"
+AVIATION_PRIOR_SOURCE_REVISION = "travel-v1"
 
 SCHEDULE_SOURCE_CODE = "ICN_SCHEDULE_API"
-SCHEDULE_SOURCE_REVISION = "travel-v1"
+SCHEDULE_SOURCE_REVISION = "travel-v2"
 SCHEDULE_SOURCE_OWNER = "인천국제공항공사"
 SCHEDULE_SOURCE_LOCATOR = (
     "https://apis.data.go.kr/B551177/statusOfSPaxFlt4TripPlatform"
@@ -23,7 +24,7 @@ SCHEDULE_ATTRIBUTION = "인천국제공항공사|공공데이터포털"
 SCHEDULE_FIELD_SCOPE = "ICN_SCHEDULE_V1"
 
 CITY_SOURCE_CODE = "ICN_DESTINATION_CITY_API_15095067"
-CITY_SOURCE_REVISION = "travel-v1"
+CITY_SOURCE_REVISION = "travel-v2"
 CITY_SOURCE_OWNER = "인천국제공항공사"
 CITY_SOURCE_LOCATOR = (
     "https://apis.data.go.kr/B551177/StatusOfSrvDestinations/"
@@ -33,7 +34,7 @@ CITY_ATTRIBUTION = "인천국제공항공사|공공데이터포털"
 CITY_FIELD_SCOPE = "ICN_DESTINATION_CITY_V1"
 
 LEGACY_SCHEDULE_SOURCE_CODE = "ICN_LEGACY_ARRIVALS_API"
-LEGACY_SCHEDULE_SOURCE_REVISION = "travel-v1"
+LEGACY_SCHEDULE_SOURCE_REVISION = "travel-v2"
 LEGACY_SCHEDULE_SOURCE_OWNER = "인천국제공항공사"
 LEGACY_SCHEDULE_SOURCE_LOCATOR = (
     "https://apis.data.go.kr/B551177/PaxFltSched"
@@ -54,14 +55,14 @@ DURATION_ATTRIBUTION = "해양수산부 수출입 물류 플랫폼|공공데이
 DURATION_FIELD_SCOPE = "ROUTE_DURATION_V1"
 
 SCHEDULE_CONTRACT_TEXT = (
-    "ICN schedule data.go v1|response(header(resultCode,resultMsg),body("
-    "items(item(codeshare,masterFlightId,flightId,st,season,firstdate,"
+    "ICN schedule data.go v2|response(header(resultCode,resultMsg),body("
+    "items(rows-or-item-wrapper(codeshare,masterFlightId,flightId,st,season,firstdate,"
     "lastdate,ynMon,ynTue,ynWed,ynThu,ynFri,ynSat,ynSun,terminalId,airline,"
     "airlineCode,airport,airportCode,tmp1,tmp2)),pageNo,numOfRows,totalCount))|"
     "departures+arrivals|complete-pages|codeshare-master-dedupe"
 )
 SCHEDULE_SCHEMA_TEXT = (
-    "documented-data-go-json(response.header+response.body.items.item+"
+    "data-go-json-v2(response.header+response.body.items-direct-list-or-item-wrapper+"
     "pageNo+numOfRows+totalCount); normalized-object(source_date:date,"
     "season:string,coverage_complete:true,flights:list[direction,carrier_code,"
     "carrier_name,flight_number,master_flight_number,destination_iata,"
@@ -86,23 +87,23 @@ SCHEDULE_SCHEMA_FINGERPRINT_SHA256 = _sha256(SCHEDULE_SCHEMA_TEXT)
 DURATION_CONTRACT_FINGERPRINT_SHA256 = _sha256(DURATION_CONTRACT_TEXT)
 DURATION_SCHEMA_FINGERPRINT_SHA256 = _sha256(DURATION_SCHEMA_TEXT)
 CITY_CONTRACT_TEXT = (
-    "ICN destination city v1|response(header(resultCode,resultMsg),body("
-    "items(item(countryName,airportName,airportCode))))|exact-curated-13|"
+    "ICN destination city v2|response(header(resultCode,resultMsg),body("
+    "items(rows-or-item-wrapper(countryName,airportName,airportCode))))|exact-curated-13|"
     "Korean-country-city-names"
 )
 CITY_SCHEMA_TEXT = (
-    "documented-data-go-json(response.header+response.body.items.item+"
+    "data-go-json-v2(response.header+response.body.items-direct-list-or-item-wrapper+"
     "countryName+airportName+airportCode)"
 )
 LEGACY_SCHEDULE_CONTRACT_TEXT = (
-    "ICN legacy arrivals v1|response(header(resultCode,resultMsg),body("
-    "items(item(airline,airport,airportcode,firstdate,flightid,friday,"
+    "ICN legacy arrivals v2|response(header(resultCode,resultMsg),body("
+    "items(rows-or-item-wrapper(airline,airport,airportcode,firstdate,flightid,friday,"
     "lastdate,monday,saturday,season,st,sunday,thursday,tuesday,wednesday)),"
     "pageNo,numOfRows,totalCount))|complete-pages|inbound-route-day-time-"
     "crosscheck|fail-closed"
 )
 LEGACY_SCHEDULE_SCHEMA_TEXT = (
-    "documented-data-go-json(response.header+response.body.items.item+"
+    "data-go-json-v2(response.header+response.body.items-direct-list-or-item-wrapper+"
     "pageNo+numOfRows+totalCount); typed-arrival(iata,flight,time,season,"
     "valid-range,weekdays)"
 )
@@ -116,6 +117,27 @@ LEGACY_SCHEDULE_SCHEMA_FINGERPRINT_SHA256 = _sha256(
     LEGACY_SCHEDULE_SCHEMA_TEXT
 )
 
+# Exact historical identities remain readable for rollback after the V2
+# wire-envelope correction. New collection never writes these identities.
+SCHEDULE_V1_CONTRACT_FINGERPRINT_SHA256 = (
+    "3b4295504d24cfb1e0da398399d61c328afa0f8124d7c298f5d4e4f950dfd372"
+)
+SCHEDULE_V1_SCHEMA_FINGERPRINT_SHA256 = (
+    "3d8d37c4a23731d11ed6c3b3ff2d87324c9dcbde2933c158b5f4ace689b82074"
+)
+CITY_V1_CONTRACT_FINGERPRINT_SHA256 = (
+    "18a1baa88e4077dbb64a4108620d046b24882dad72e6ec0d56629396391d669f"
+)
+CITY_V1_SCHEMA_FINGERPRINT_SHA256 = (
+    "907670feb0293b0a1a40e6d7065cae2c803e4ba458828b22631dbee002017c18"
+)
+LEGACY_SCHEDULE_V1_CONTRACT_FINGERPRINT_SHA256 = (
+    "a88a91166bb7ea567f59637159f07f2285981f2ae0588e921ec2882e1c73bbfb"
+)
+LEGACY_SCHEDULE_V1_SCHEMA_FINGERPRINT_SHA256 = (
+    "e32b4246a27c5266b52aa0ea24923018eb7d474d09cab1cbcc36ebb85f32ebbd"
+)
+
 
 def load_data_go_service_key(environment: dict[str, str]) -> str | None:
     """Return the canonical key, falling back without exposing either value."""
diff --git a/travel_windows/ingestion.py b/travel_windows/ingestion.py
index ebc41f4..a73d5c8 100644
--- a/travel_windows/ingestion.py
+++ b/travel_windows/ingestion.py
@@ -634,7 +634,7 @@ def _persist_parse_run(
             .filter(
                 artifact=artifacts[inputs[0].artifact.pk],
                 parser_name=parser_name,
-                parser_version=ParseRun.ParserVersion.V1,
+                parser_version=ParseRun.ParserVersion.V2,
                 input_identity_sha256=input_identity,
             )
             .first()
@@ -643,7 +643,7 @@ def _persist_parse_run(
             run = ParseRun.objects.create(
                 artifact=artifacts[inputs[0].artifact.pk],
                 parser_name=parser_name,
-                parser_version=ParseRun.ParserVersion.V1,
+                parser_version=ParseRun.ParserVersion.V2,
                 parser_contract_fingerprint_sha256=contract_fingerprint,
                 expected_schema_fingerprint_sha256=schema_fingerprint,
                 input_identity_sha256=input_identity,


