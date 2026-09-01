## `feat(source): collect official route duration reference`

diff --git a/sources/management/commands/register_approved_sources.py b/sources/management/commands/register_approved_sources.py
index c1cb614..030be75 100644
--- a/sources/management/commands/register_approved_sources.py
+++ b/sources/management/commands/register_approved_sources.py
@@ -19,12 +19,15 @@ from travel_windows.contracts import (
     CITY_V2_SOURCE_REVISION,
     DATA_GO_SECRET_REFERENCE,
     DURATION_ATTRIBUTION,
-    DURATION_CONTRACT_FINGERPRINT_SHA256,
     DURATION_FIELD_SCOPE,
+    DURATION_REFERENCE_CONTRACT_FINGERPRINT_SHA256,
     DURATION_SOURCE_CODE,
     DURATION_SOURCE_LOCATOR,
     DURATION_SOURCE_OWNER,
     DURATION_SOURCE_REVISION,
+    DURATION_V1_CONTRACT_FINGERPRINT_SHA256,
+    DURATION_V1_FIELD_SCOPE,
+    DURATION_V1_SOURCE_REVISION,
     LEGACY_SCHEDULE_ATTRIBUTION,
     LEGACY_SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
     LEGACY_SCHEDULE_FIELD_SCOPE,
@@ -136,6 +139,22 @@ def _with_city_contract_history(
     return replace(spec, prior_contracts=(prior_v1, prior_v2))
 
 
+def _with_duration_contract_history(
+    spec: ApprovedSourceSpec,
+) -> ApprovedSourceSpec:
+    prior_v1 = replace(
+        spec,
+        revision=DURATION_V1_SOURCE_REVISION,
+        field_scope_code=DURATION_V1_FIELD_SCOPE,
+        contract_fingerprint_sha256=(
+            DURATION_V1_CONTRACT_FINGERPRINT_SHA256
+        ),
+        decision_basis_code="TRAVEL_OPPORTUNITY_CONTRACT_20260831",
+        prior_contracts=(),
+    )
+    return replace(spec, prior_contracts=(prior_v1,))
+
+
 APPROVED_SOURCE_SPECS = (
     ApprovedSourceSpec(
         code="MOFA_ENTRY_CSV",
@@ -258,18 +277,22 @@ APPROVED_SOURCE_SPECS = (
         ),
         fingerprint=LEGACY_SCHEDULE_V1_CONTRACT_FINGERPRINT_SHA256,
     ),
-    ApprovedSourceSpec(
-        code=DURATION_SOURCE_CODE,
-        revision=DURATION_SOURCE_REVISION,
-        module=SourceConfiguration.Module.AVIATION,
-        owner=DURATION_SOURCE_OWNER,
-        official_locator=DURATION_SOURCE_LOCATOR,
-        secret_reference_name="",
-        access_mode=SourceRightsDecision.AccessMode.ANONYMOUS_PUBLIC,
-        field_scope_code=DURATION_FIELD_SCOPE,
-        contract_fingerprint_sha256=DURATION_CONTRACT_FINGERPRINT_SHA256,
-        decision_basis_code="TRAVEL_OPPORTUNITY_CONTRACT_20260831",
-        attribution_text=DURATION_ATTRIBUTION,
+    _with_duration_contract_history(
+        ApprovedSourceSpec(
+            code=DURATION_SOURCE_CODE,
+            revision=DURATION_SOURCE_REVISION,
+            module=SourceConfiguration.Module.AVIATION,
+            owner=DURATION_SOURCE_OWNER,
+            official_locator=DURATION_SOURCE_LOCATOR,
+            secret_reference_name="",
+            access_mode=SourceRightsDecision.AccessMode.ANONYMOUS_PUBLIC,
+            field_scope_code=DURATION_FIELD_SCOPE,
+            contract_fingerprint_sha256=(
+                DURATION_REFERENCE_CONTRACT_FINGERPRINT_SHA256
+            ),
+            decision_basis_code="ROUTE_DURATION_REFERENCE_20260831",
+            attribution_text=DURATION_ATTRIBUTION,
+        )
     ),
 )
 
diff --git a/sources/migrations/0016_route_duration_reference_contract.py b/sources/migrations/0016_route_duration_reference_contract.py
new file mode 100644
index 0000000..d67f91e
--- /dev/null
+++ b/sources/migrations/0016_route_duration_reference_contract.py
@@ -0,0 +1,357 @@
+from django.db import migrations, models
+from django.db.models import Q
+
+
+FUNCTION_NAME = "sources_guard_parse_run_change"
+INPUT_FUNCTION_NAME = "sources_guard_parse_run_input"
+DURATION_SOURCE_CODE = "PORT_LOGISTICS_ROUTE_DURATION_15151728"
+DURATION_REFERENCE_CONTRACT_FINGERPRINT_SHA256 = (
+    "837607c701eff8e7f0055dda2803ad3af208c1412c968ea4a0c9aa07acdf1040"
+)
+DURATION_REFERENCE_SCHEMA_FINGERPRINT_SHA256 = (
+    "8ebd993988e8939060bde5d2c3577993c4f01838b645c5383e2dbcf83e176dee"
+)
+OLD_DURATION_CLAUSE = """            OR (source_module = 'AVIATION'
+                AND NEW.parser_name = 'ROUTE_DURATION_CSV'
+                AND NEW.parser_version = 'V1'
+                AND NEW.parser_contract_fingerprint_sha256 IN (
+                    '3018365ff3d3549765d5a428a1413ea730071209434def786ab6734f89f8c2ba',
+                    '430903be2d56367dfc1ca9617e69e6317ff76dadb54912fd02ca1e34d88fee01'
+                )
+                AND NEW.expected_schema_fingerprint_sha256 =
+                    '0fe301b62df9abd8b449aeeb1e6ea62cbca80ab7c61e35b6cee552aa58278307')"""
+NEW_DURATION_CLAUSE = """            OR (source_module = 'AVIATION'
+                AND NEW.parser_name = 'ROUTE_DURATION_REFERENCE_FILE'
+                AND NEW.parser_version = 'V2'
+                AND NEW.parser_contract_fingerprint_sha256 =
+                    '837607c701eff8e7f0055dda2803ad3af208c1412c968ea4a0c9aa07acdf1040'
+                AND NEW.expected_schema_fingerprint_sha256 =
+                    '8ebd993988e8939060bde5d2c3577993c4f01838b645c5383e2dbcf83e176dee')"""
+OLD_ORDERED_PARSER_LIST = """            'ICN_FLIGHT_SCHEDULE_JSON',
+            'ICN_DESTINATION_CITY_JSON',
+            'ICN_LEGACY_ARRIVALS_JSON'"""
+NEW_ORDERED_PARSER_LIST = """            'ICN_FLIGHT_SCHEDULE_JSON',
+            'ICN_DESTINATION_CITY_JSON',
+            'ICN_LEGACY_ARRIVALS_JSON',
+            'ROUTE_DURATION_REFERENCE_FILE'"""
+OLD_CLOSE_PARSER_LIST = """        'ICN_FLIGHT_SCHEDULE_JSON',
+        'ICN_DESTINATION_CITY_JSON',
+        'ICN_LEGACY_ARRIVALS_JSON'"""
+NEW_CLOSE_PARSER_LIST = """        'ICN_FLIGHT_SCHEDULE_JSON',
+        'ICN_DESTINATION_CITY_JSON',
+        'ICN_LEGACY_ARRIVALS_JSON',
+        'ROUTE_DURATION_REFERENCE_FILE'"""
+OLD_LEGACY_CLOSE_CLAUSE = """        ELSIF NEW.parser_name = 'ICN_LEGACY_ARRIVALS_JSON' AND EXISTS (
+            SELECT 1 FROM sources_parseruninput
+             WHERE parse_run_id = NEW.id AND role <> 'LEGACY_ARRIVAL'
+        ) THEN
+            RAISE EXCEPTION 'legacy parse requires only arrival inputs'
+                USING ERRCODE = 'check_violation';"""
+NEW_DURATION_CLOSE_CLAUSE = """        ELSIF NEW.parser_name = 'ROUTE_DURATION_REFERENCE_FILE' AND (
+            input_count <> 1
+            OR EXISTS (
+                SELECT 1 FROM sources_parseruninput
+                 WHERE parse_run_id = NEW.id
+                   AND (role <> 'DURATION_ARCHIVE' OR role_sequence <> 1)
+            )
+        ) THEN
+            RAISE EXCEPTION 'duration parse requires exactly one archive input'
+                USING ERRCODE = 'check_violation';"""
+OLD_INPUT_ROLE_CLAUSE = """       OR (run_parser = 'ICN_LEGACY_ARRIVALS_JSON'
+           AND NEW.role <> 'LEGACY_ARRIVAL')"""
+NEW_INPUT_ROLE_CLAUSE = """       OR (run_parser = 'ROUTE_DURATION_REFERENCE_FILE'
+           AND NEW.role <> 'DURATION_ARCHIVE')"""
+
+
+def _function_body(schema_editor, function_name):
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
+            [function_name],
+        )
+        rows = cursor.fetchall()
+    if len(rows) != 1:
+        raise RuntimeError(f"expected one trigger function named {function_name}")
+    return rows[0][0]
+
+
+def _replace_function(schema_editor, function_name, replacements):
+    body = _function_body(schema_editor, function_name)
+    for old, new, expected_count in replacements:
+        if body.count(old) != expected_count:
+            raise RuntimeError(f"unexpected {function_name} trigger definition")
+        body = body.replace(old, new)
+    quoted_name = schema_editor.quote_name(function_name)
+    with schema_editor.connection.cursor() as cursor:
+        cursor.execute(
+            f"""
+            CREATE OR REPLACE FUNCTION {quoted_name}() RETURNS trigger
+            LANGUAGE plpgsql AS $route_duration_reference_contract$
+            {body}
+            $route_duration_reference_contract$
+            """
+        )
+
+
+def allow_route_duration_reference_contract(apps, schema_editor):
+    _replace_function(
+        schema_editor,
+        FUNCTION_NAME,
+        (
+            (
+                OLD_DURATION_CLAUSE,
+                OLD_DURATION_CLAUSE + "\n" + NEW_DURATION_CLAUSE,
+                1,
+            ),
+            (OLD_ORDERED_PARSER_LIST, NEW_ORDERED_PARSER_LIST, 2),
+            (OLD_CLOSE_PARSER_LIST, NEW_CLOSE_PARSER_LIST, 1),
+            (
+                OLD_LEGACY_CLOSE_CLAUSE,
+                OLD_LEGACY_CLOSE_CLAUSE + "\n" + NEW_DURATION_CLOSE_CLAUSE,
+                1,
+            ),
+        ),
+    )
+    _replace_function(
+        schema_editor,
+        INPUT_FUNCTION_NAME,
+        (
+            (OLD_ORDERED_PARSER_LIST, NEW_ORDERED_PARSER_LIST, 1),
+            (
+                OLD_INPUT_ROLE_CLAUSE,
+                OLD_INPUT_ROLE_CLAUSE + "\n" + NEW_INPUT_ROLE_CLAUSE,
+                1,
+            ),
+        ),
+    )
+
+
+def restore_route_duration_v1(apps, schema_editor):
+    FetchAttempt = apps.get_model("sources", "FetchAttempt")
+    ParseRun = apps.get_model("sources", "ParseRun")
+    ParseRunInput = apps.get_model("sources", "ParseRunInput")
+    SourceConfiguration = apps.get_model("sources", "SourceConfiguration")
+    SourceRightsDecision = apps.get_model("sources", "SourceRightsDecision")
+    if (
+        ParseRun.objects.filter(
+            Q(parser_name="ROUTE_DURATION_REFERENCE_FILE")
+            | Q(
+                parser_contract_fingerprint_sha256=(
+                    DURATION_REFERENCE_CONTRACT_FINGERPRINT_SHA256
+                ),
+                expected_schema_fingerprint_sha256=(
+                    DURATION_REFERENCE_SCHEMA_FINGERPRINT_SHA256
+                ),
+            )
+        ).exists()
+        or ParseRunInput.objects.filter(role="DURATION_ARCHIVE").exists()
+        or FetchAttempt.objects.filter(
+            source__code=DURATION_SOURCE_CODE,
+            source_revision="travel-v2",
+        ).exists()
+        or SourceRightsDecision.objects.filter(
+            source__code=DURATION_SOURCE_CODE,
+            source_revision="travel-v2",
+        ).exists()
+        or SourceConfiguration.objects.filter(
+            code=DURATION_SOURCE_CODE,
+            revision="travel-v2",
+        ).exists()
+    ):
+        raise RuntimeError(
+            "route duration reference contract rollback requires no V2 source "
+            "evidence"
+        )
+    _replace_function(
+        schema_editor,
+        FUNCTION_NAME,
+        (
+            (
+                OLD_DURATION_CLAUSE + "\n" + NEW_DURATION_CLAUSE,
+                OLD_DURATION_CLAUSE,
+                1,
+            ),
+            (NEW_ORDERED_PARSER_LIST, OLD_ORDERED_PARSER_LIST, 2),
+            (NEW_CLOSE_PARSER_LIST, OLD_CLOSE_PARSER_LIST, 1),
+            (
+                OLD_LEGACY_CLOSE_CLAUSE + "\n" + NEW_DURATION_CLOSE_CLAUSE,
+                OLD_LEGACY_CLOSE_CLAUSE,
+                1,
+            ),
+        ),
+    )
+    _replace_function(
+        schema_editor,
+        INPUT_FUNCTION_NAME,
+        (
+            (NEW_ORDERED_PARSER_LIST, OLD_ORDERED_PARSER_LIST, 1),
+            (
+                OLD_INPUT_ROLE_CLAUSE + "\n" + NEW_INPUT_ROLE_CLAUSE,
+                OLD_INPUT_ROLE_CLAUSE,
+                1,
+            ),
+        ),
+    )
+
+
+class Migration(migrations.Migration):
+    dependencies = [("sources", "0015_destination_identity_contract")]
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
+                            "ICN_LEGACY_ARRIVALS_API",
+                        ),
+                        revision__in=("travel-v1", "travel-v2"),
+                        secret_reference_name="DATA_GO_KR_SERVICE_KEY",
+                    )
+                    | Q(
+                        code="ICN_DESTINATION_CITY_API_15095067",
+                        revision__in=("travel-v1", "travel-v2", "travel-v3"),
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
+                            DURATION_SOURCE_CODE,
+                        ),
+                        revision__in=("travel-v1", "travel-v2"),
+                        secret_reference_name="",
+                    )
+                ),
+                name="source_aviation_approved_activation",
+            ),
+        ),
+        migrations.RemoveConstraint(
+            model_name="parserun",
+            name="parse_name_allowlist",
+        ),
+        migrations.AlterField(
+            model_name="parserun",
+            name="parser_name",
+            field=models.CharField(
+                choices=[
+                    ("MOFA_ENTRY_CSV", "MOFA entry CSV"),
+                    ("MOFA_TRAVEL_ALARM_JSON", "MOFA travel alarm JSON"),
+                    (
+                        "ICN_FLIGHT_SCHEDULE_JSON",
+                        "ICN scheduled flights JSON",
+                    ),
+                    (
+                        "ICN_DESTINATION_CITY_JSON",
+                        "ICN destination city JSON",
+                    ),
+                    (
+                        "ICN_LEGACY_ARRIVALS_JSON",
+                        "ICN legacy arrivals JSON",
+                    ),
+                    ("ROUTE_DURATION_CSV", "Route duration CSV"),
+                    (
+                        "ROUTE_DURATION_REFERENCE_FILE",
+                        "Route duration reference file",
+                    ),
+                ],
+                max_length=64,
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="parserun",
+            constraint=models.CheckConstraint(
+                condition=Q(
+                    parser_name__in=(
+                        "MOFA_ENTRY_CSV",
+                        "MOFA_TRAVEL_ALARM_JSON",
+                        "ICN_FLIGHT_SCHEDULE_JSON",
+                        "ICN_DESTINATION_CITY_JSON",
+                        "ICN_LEGACY_ARRIVALS_JSON",
+                        "ROUTE_DURATION_CSV",
+                        "ROUTE_DURATION_REFERENCE_FILE",
+                    )
+                ),
+                name="parse_name_allowlist",
+            ),
+        ),
+        migrations.RemoveConstraint(
+            model_name="parserun",
+            name="parse_input_identity_shape",
+        ),
+        migrations.AddConstraint(
+            model_name="parserun",
+            constraint=models.CheckConstraint(
+                condition=(
+                    Q(input_identity_sha256="")
+                    | Q(
+                        parser_name__in=(
+                            "ICN_FLIGHT_SCHEDULE_JSON",
+                            "ICN_DESTINATION_CITY_JSON",
+                            "ICN_LEGACY_ARRIVALS_JSON",
+                            "ROUTE_DURATION_REFERENCE_FILE",
+                        ),
+                        input_identity_sha256__regex=r"^[0-9a-f]{64}$",
+                    )
+                ),
+                name="parse_input_identity_shape",
+            ),
+        ),
+        migrations.AlterField(
+            model_name="parseruninput",
+            name="role",
+            field=models.CharField(
+                choices=[
+                    ("SCHEDULE_DEPARTURE", "Scheduled departure page"),
+                    ("SCHEDULE_ARRIVAL", "Scheduled arrival page"),
+                    ("DESTINATION_CITY", "Destination city response"),
+                    ("LEGACY_ARRIVAL", "Legacy arrival page"),
+                    ("DURATION_ARCHIVE", "Route duration archive"),
+                ],
+                max_length=32,
+            ),
+        ),
+        migrations.RemoveConstraint(
+            model_name="parseruninput",
+            name="parse_input_role_known",
+        ),
+        migrations.AddConstraint(
+            model_name="parseruninput",
+            constraint=models.CheckConstraint(
+                condition=Q(
+                    role__in=(
+                        "SCHEDULE_DEPARTURE",
+                        "SCHEDULE_ARRIVAL",
+                        "DESTINATION_CITY",
+                        "LEGACY_ARRIVAL",
+                        "DURATION_ARCHIVE",
+                    )
+                ),
+                name="parse_input_role_known",
+            ),
+        ),
+        migrations.RunPython(
+            allow_route_duration_reference_contract,
+            restore_route_duration_v1,
+        ),
+    ]
diff --git a/sources/models.py b/sources/models.py
index 1e9e131..a911485 100644
--- a/sources/models.py
+++ b/sources/models.py
@@ -15,6 +15,7 @@ AVIATION_PARSE_INPUT_ROLES = frozenset(
         "SCHEDULE_ARRIVAL",
         "DESTINATION_CITY",
         "LEGACY_ARRIVAL",
+        "DURATION_ARCHIVE",
     }
 )
 _SHA256_HEX = re.compile(r"^[0-9a-f]{64}$")
@@ -109,7 +110,7 @@ class SourceConfiguration(models.Model):
                             "PORT_LOGISTICS_ROUTE_DURATION",
                             "PORT_LOGISTICS_ROUTE_DURATION_15151728",
                         ],
-                        revision="travel-v1",
+                        revision__in=["travel-v1", "travel-v2"],
                         secret_reference_name="",
                     )
                 ),
@@ -308,6 +309,10 @@ class ParseRun(models.Model):
             "ICN legacy arrivals JSON",
         )
         ROUTE_DURATION_CSV = "ROUTE_DURATION_CSV", "Route duration CSV"
+        ROUTE_DURATION_REFERENCE_FILE = (
+            "ROUTE_DURATION_REFERENCE_FILE",
+            "Route duration reference file",
+        )
 
     class ParserVersion(models.TextChoices):
         V1 = "V1", "Version 1"
@@ -364,6 +369,7 @@ class ParseRun(models.Model):
                         "ICN_DESTINATION_CITY_JSON",
                         "ICN_LEGACY_ARRIVALS_JSON",
                         "ROUTE_DURATION_CSV",
+                        "ROUTE_DURATION_REFERENCE_FILE",
                     ]
                 ),
                 name="parse_name_allowlist",
@@ -382,6 +388,7 @@ class ParseRun(models.Model):
                             "ICN_FLIGHT_SCHEDULE_JSON",
                             "ICN_DESTINATION_CITY_JSON",
                             "ICN_LEGACY_ARRIVALS_JSON",
+                            "ROUTE_DURATION_REFERENCE_FILE",
                         ],
                         input_identity_sha256__regex=r"^[0-9a-f]{64}$",
                     )
@@ -424,6 +431,10 @@ class ParseRunInput(models.Model):
             "LEGACY_ARRIVAL",
             "Legacy arrival page",
         )
+        DURATION_ARCHIVE = (
+            "DURATION_ARCHIVE",
+            "Route duration archive",
+        )
 
     parse_run = models.ForeignKey(
         ParseRun,
@@ -457,6 +468,7 @@ class ParseRunInput(models.Model):
                         "SCHEDULE_ARRIVAL",
                         "DESTINATION_CITY",
                         "LEGACY_ARRIVAL",
+                        "DURATION_ARCHIVE",
                     ]
                 ),
                 name="parse_input_role_known",
diff --git a/sources/tests/test_register_approved_sources.py b/sources/tests/test_register_approved_sources.py
index 00f9336..c6a8754 100644
--- a/sources/tests/test_register_approved_sources.py
+++ b/sources/tests/test_register_approved_sources.py
@@ -475,6 +475,81 @@ class ApprovedSourceRegistrationCommandTests(TransactionTestCase):
             },
         )
 
+    def test_duration_v1_contract_upgrades_and_reruns_idempotently(self):
+        current = next(
+            spec
+            for spec in registration.APPROVED_SOURCE_SPECS
+            if spec.code == "PORT_LOGISTICS_ROUTE_DURATION_15151728"
+        )
+        (prior_v1,) = current.prior_contracts
+        self.assertEqual(prior_v1.revision, "travel-v1")
+        self.assertEqual(prior_v1.field_scope_code, "ROUTE_DURATION_V1")
+        self.assertEqual(current.revision, "travel-v2")
+        self.assertEqual(
+            current.field_scope_code,
+            "ROUTE_DURATION_DERIVATION_V2",
+        )
+        self.assertEqual(
+            current.access_mode,
+            SourceRightsDecision.AccessMode.ANONYMOUS_PUBLIC,
+        )
+        self.assertTrue(current.rights_values()["automated_collection_allowed"])
+
+        source = self.make_exact_source(prior_v1)
+        prior_approval = self.make_exact_approval(source, prior_v1)
+        self.activate_source(source)
+        prior_identity = (prior_approval.pk, prior_approval.decided_at)
+
+        with mock.patch.object(
+            registration,
+            "APPROVED_SOURCE_SPECS",
+            (current,),
+        ):
+            output = self.call_registration("--apply")
+            source.refresh_from_db()
+            prior_approval.refresh_from_db()
+            self.assertEqual(
+                (source.revision, source.state, source.enabled),
+                ("travel-v2", SourceConfiguration.State.ACTIVE, True),
+            )
+            self.assertEqual(
+                (prior_approval.pk, prior_approval.decided_at),
+                prior_identity,
+            )
+            self.assertEqual(
+                set(
+                    source.rights_decisions.values_list(
+                        "source_revision",
+                        "field_scope_code",
+                        "contract_fingerprint_sha256",
+                    )
+                ),
+                {
+                    (
+                        prior_v1.revision,
+                        prior_v1.field_scope_code,
+                        prior_v1.contract_fingerprint_sha256,
+                    ),
+                    (
+                        current.revision,
+                        current.field_scope_code,
+                        current.contract_fingerprint_sha256,
+                    ),
+                },
+            )
+            self.assertIn("result=UPGRADED_AND_ACTIVATED", output)
+
+            rights_identity = set(
+                source.rights_decisions.values_list("id", "decided_at")
+            )
+            output = self.call_registration("--apply")
+
+        self.assertEqual(
+            set(source.rights_decisions.values_list("id", "decided_at")),
+            rights_identity,
+        )
+        self.assertIn("result=ALREADY_ACTIVE", output)
+
     def test_upgrade_rejects_mismatched_prior_fingerprint(self):
         prior, current = self.make_upgrade_contracts("MISMATCH")
         source = self.make_exact_source(prior)
diff --git a/sources/tests/test_route_duration_reference_migration.py b/sources/tests/test_route_duration_reference_migration.py
new file mode 100644
index 0000000..6c2c739
--- /dev/null
+++ b/sources/tests/test_route_duration_reference_migration.py
@@ -0,0 +1,99 @@
+import importlib
+
+from django.db import connection
+from django.db.migrations.executor import MigrationExecutor
+from django.db.migrations.recorder import MigrationRecorder
+from django.test import TransactionTestCase
+
+from operations.tests.migration_helpers import restore_canonical_migration_graph
+from sources.models import SourceConfiguration
+from travel_windows.contracts import (
+    DURATION_SOURCE_CODE,
+    DURATION_SOURCE_LOCATOR,
+)
+
+
+migration = importlib.import_module(
+    "sources.migrations.0016_route_duration_reference_contract"
+)
+
+
+class RouteDurationReferenceMigrationTests(TransactionTestCase):
+    latest = [("sources", "0016_route_duration_reference_contract")]
+    previous = [("sources", "0015_destination_identity_contract")]
+
+    def setUp(self):
+        restore_canonical_migration_graph(connection)
+
+    def _fixture_teardown(self):
+        try:
+            super()._fixture_teardown()
+        finally:
+            restore_canonical_migration_graph(connection)
+
+    def function_body(self, name):
+        with connection.cursor() as cursor:
+            cursor.execute(
+                "SELECT prosrc FROM pg_proc WHERE oid = %s::regprocedure",
+                [f"{name}()"],
+            )
+            return cursor.fetchone()[0]
+
+    def test_v2_evidence_blocks_reverse_without_weakening_the_guard(self):
+        SourceConfiguration.objects.create(
+            code=DURATION_SOURCE_CODE,
+            revision="travel-v2",
+            module=SourceConfiguration.Module.AVIATION,
+            owner="Synthetic owner",
+            official_locator=DURATION_SOURCE_LOCATOR,
+        )
+        before_parse = self.function_body(migration.FUNCTION_NAME)
+        before_input = self.function_body(migration.INPUT_FUNCTION_NAME)
+        self.assertIn(migration.OLD_DURATION_CLAUSE, before_parse)
+        self.assertIn(migration.NEW_DURATION_CLAUSE, before_parse)
+        self.assertIn(migration.NEW_DURATION_CLOSE_CLAUSE, before_parse)
+        self.assertIn(migration.NEW_INPUT_ROLE_CLAUSE, before_input)
+
+        with self.assertRaisesMessage(
+            RuntimeError,
+            "route duration reference contract rollback requires no V2 source "
+            "evidence",
+        ):
+            MigrationExecutor(connection).migrate(self.previous)
+
+        self.assertEqual(
+            self.function_body(migration.FUNCTION_NAME),
+            before_parse,
+        )
+        self.assertEqual(
+            self.function_body(migration.INPUT_FUNCTION_NAME),
+            before_input,
+        )
+        self.assertIn(
+            ("sources", "0016_route_duration_reference_contract"),
+            MigrationRecorder(connection).applied_migrations(),
+        )
+
+    def test_empty_reverse_and_reapply_restore_both_parse_guards(self):
+        self.assertFalse(SourceConfiguration.objects.exists())
+
+        MigrationExecutor(connection).migrate(self.previous)
+
+        prior_parse = self.function_body(migration.FUNCTION_NAME)
+        prior_input = self.function_body(migration.INPUT_FUNCTION_NAME)
+        self.assertIn(migration.OLD_DURATION_CLAUSE, prior_parse)
+        self.assertNotIn(migration.NEW_DURATION_CLAUSE, prior_parse)
+        self.assertNotIn(migration.NEW_DURATION_CLOSE_CLAUSE, prior_parse)
+        self.assertNotIn(migration.NEW_INPUT_ROLE_CLAUSE, prior_input)
+        self.assertNotIn(
+            ("sources", "0016_route_duration_reference_contract"),
+            MigrationRecorder(connection).applied_migrations(),
+        )
+
+        MigrationExecutor(connection).migrate(self.latest)
+
+        current_parse = self.function_body(migration.FUNCTION_NAME)
+        current_input = self.function_body(migration.INPUT_FUNCTION_NAME)
+        self.assertIn(migration.NEW_DURATION_CLAUSE, current_parse)
+        self.assertIn(migration.NEW_DURATION_CLOSE_CLAUSE, current_parse)
+        self.assertIn(migration.NEW_INPUT_ROLE_CLAUSE, current_input)
diff --git a/sources/tests/test_transport.py b/sources/tests/test_transport.py
index 918cc3a..c6a0d43 100644
--- a/sources/tests/test_transport.py
+++ b/sources/tests/test_transport.py
@@ -31,12 +31,18 @@ from sources.transport import (
     ROLE_LEGACY_ARRIVAL,
     ROLE_SCHEDULE_ARRIVAL,
     ROLE_SCHEDULE_DEPARTURE,
+    ROLE_DURATION_ARCHIVE,
+    ROLE_DURATION_LIMIT,
+    ROLE_DURATION_METADATA,
+    ROLE_DURATION_PAGE,
+    DURATION_SOURCE_LOCATOR,
     RequestFingerprint,
     SingleAttemptResult,
     destination_city_request_fingerprint,
     fetch_data_go_reference_evidence,
     fetch_data_go_schedule_pages,
     fetch_entry_attachment,
+    fetch_route_duration_reference,
     fetch_travel_alarm_jp,
     legacy_arrival_request_fingerprint,
     schedule_page_request_fingerprint,
@@ -53,16 +59,31 @@ class FakeSocket:
 
 
 class FakeResponse:
-    def __init__(self, status, body, *, content_length=None, read_error=None):
+    def __init__(
+        self,
+        status,
+        body,
+        *,
+        content_length=None,
+        content_type="",
+        content_disposition="",
+        read_error=None,
+    ):
         self.status = status
         self._body = body
         self._content_length = content_length
+        self._content_type = content_type
+        self._content_disposition = content_disposition
         self._read_error = read_error
         self.read_amounts = []
 
     def getheader(self, name, default=None):
         if name == "Content-Length":
             return self._content_length
+        if name == "Content-Type":
+            return self._content_type
+        if name == "Content-Disposition":
+            return self._content_disposition
         return default
 
     def read(self, amount=None):
@@ -155,6 +176,40 @@ def data_go_page(*, page=1, page_size=100, total=1):
 
 
 class TransportTestCase(unittest.TestCase):
+    def duration_page(self):
+        return (
+            '<meta name="title" property="og:title" '
+            'content="해양수산부_항공 스케줄 기본정보_20241212">'
+            '<button onclick="fileDetailObj.fn_fileDataDown('
+            "'15151728', 'uddi:ebe570de-68fb-42d4-ad47-9a5d565d6c5a',"
+            "'','1', '2')\">다운로드</button>"
+        ).encode("utf-8")
+
+    def duration_metadata(self):
+        return json.dumps(
+            {
+                "status": True,
+                "atchFileId": "FILE_000000003520744",
+                "fileDetailSn": "1",
+                "dpk": "uddi:ebe570de-68fb-42d4-ad47-9a5d565d6c5a",
+                "fileDataRegistVO": {
+                    "atchFileId": "FILE_000000003520744",
+                    "fileDetailSn": "1",
+                    "orginlFileNm": "수출입 물류 플랫폼_항공 스케줄 기본정보.zip",
+                },
+                "dataSetFileDetailInfo": {
+                    "publicDataPk": "15151728",
+                    "publicDataDetailPk": (
+                        "uddi:ebe570de-68fb-42d4-ad47-9a5d565d6c5a"
+                    ),
+                    "stdrDe": "12/12/2024",
+                    "dataNm": "해양수산부_항공 스케줄 기본정보_20241212",
+                },
+            },
+            ensure_ascii=False,
+            separators=(",", ":"),
+        ).encode("utf-8")
+
     def entry_fetch(self, connection):
         return fetch_entry_attachment(
             official_locator=ENTRY_SOURCE_LOCATOR,
@@ -200,6 +255,136 @@ class TransportTestCase(unittest.TestCase):
             "e9ed083968ebdcd1341df0fbb0da3014ce62f61ba8114a558446b30cfe21f913",
         )
 
+    def test_duration_reference_discovers_captcha_checks_and_fetches_archive(self):
+        archive = b"PK synthetic official archive"
+        connections = [
+            FakeConnection(FakeResponse(200, self.duration_page())),
+            FakeConnection(FakeResponse(200, self.duration_metadata())),
+            FakeConnection(FakeResponse(200, b'{"needCaptcha":false}')),
+            FakeConnection(
+                FakeResponse(
+                    200,
+                    archive,
+                    content_type="application/octet-stream; charset=UTF-8",
+                    content_disposition=(
+                        'attachment; filename="official-duration.zip"'
+                    ),
+                )
+            ),
+        ]
+        factory_calls = []
+
+        def factory(host, port, timeout):
+            factory_calls.append((host, port, timeout))
+            return connections.pop(0)
+
+        executor_roles = []
+
+        def executor(*, source_code, request_fingerprint, call_once):
+            executor_roles.append((source_code, request_fingerprint.revision))
+            return call_once()
+
+        result = fetch_route_duration_reference(
+            official_locator=DURATION_SOURCE_LOCATOR,
+            connect_timeout_seconds=4,
+            read_timeout_seconds=12,
+            connection_factory=factory,
+            attempt_executor=executor,
+        )
+
+        self.assertTrue(result.succeeded)
+        self.assertEqual(result.reference_date, "2024-12-12")
+        self.assertRegex(result.reference_version_sha256, r"^[0-9a-f]{64}$")
+        self.assertFalse(result.unchanged)
+        self.assertEqual(result.archive, archive)
+        self.assertNotIn("synthetic official archive", repr(result))
+        self.assertEqual(len(factory_calls), 4)
+        self.assertEqual(
+            [(row.role, row.role_sequence) for row in result.lineage],
+            [
+                (ROLE_DURATION_PAGE, 1),
+                (ROLE_DURATION_METADATA, 1),
+                (ROLE_DURATION_LIMIT, 1),
+                (ROLE_DURATION_ARCHIVE, 1),
+            ],
+        )
+        self.assertEqual(
+            [revision for _source, revision in executor_roles],
+            [
+                "ROUTE_DURATION_PAGE_V1",
+                "ROUTE_DURATION_METADATA_V1",
+                "ROUTE_DURATION_LIMIT_V1",
+                "ROUTE_DURATION_ARCHIVE_V1",
+            ],
+        )
+
+    def test_duration_reference_skips_archive_when_identity_is_unchanged(self):
+        first_connections = [
+            FakeConnection(FakeResponse(200, self.duration_page())),
+            FakeConnection(FakeResponse(200, self.duration_metadata())),
+            FakeConnection(FakeResponse(200, b'{"needCaptcha":false}')),
+            FakeConnection(
+                FakeResponse(
+                    200,
+                    b"PK first archive",
+                    content_type="application/octet-stream",
+                    content_disposition='attachment; filename="duration.zip"',
+                )
+            ),
+        ]
+        first = fetch_route_duration_reference(
+            official_locator=DURATION_SOURCE_LOCATOR,
+            connect_timeout_seconds=4,
+            read_timeout_seconds=12,
+            connection_factory=lambda *_args: first_connections.pop(0),
+        )
+        second_connections = [
+            FakeConnection(FakeResponse(200, self.duration_page())),
+            FakeConnection(FakeResponse(200, self.duration_metadata())),
+        ]
+        second = fetch_route_duration_reference(
+            official_locator=DURATION_SOURCE_LOCATOR,
+            connect_timeout_seconds=4,
+            read_timeout_seconds=12,
+            known_reference_version_sha256=(
+                first.reference_version_sha256
+            ),
+            connection_factory=lambda *_args: second_connections.pop(0),
+        )
+
+        self.assertTrue(second.succeeded)
+        self.assertTrue(second.unchanged)
+        self.assertEqual(second.archive, b"")
+        self.assertEqual(len(second.lineage), 2)
+        self.assertEqual(second.reference_date, first.reference_date)
+        self.assertEqual(
+            second.reference_version_sha256,
+            first.reference_version_sha256,
+        )
+        self.assertEqual(second_connections, [])
+
+    def test_duration_reference_stops_when_portal_requires_captcha(self):
+        connections = [
+            FakeConnection(FakeResponse(200, self.duration_page())),
+            FakeConnection(FakeResponse(200, self.duration_metadata())),
+            FakeConnection(FakeResponse(200, b'{"needCaptcha":true}')),
+        ]
+
+        def factory(_host, _port, _timeout):
+            return connections.pop(0)
+
+        result = fetch_route_duration_reference(
+            official_locator=DURATION_SOURCE_LOCATOR,
+            connect_timeout_seconds=4,
+            read_timeout_seconds=12,
+            connection_factory=factory,
+        )
+
+        self.assertFalse(result.succeeded)
+        self.assertEqual(result.failure_code, FAILURE_AUTHENTICATION)
+        self.assertEqual(result.archive, b"")
+        self.assertEqual(len(result.lineage), 3)
+
     def test_entry_success_is_exact_bounded_and_body_is_repr_redacted(self):
         raw_marker = b"synthetic-entry-response"
         response = FakeResponse(200, raw_marker)
diff --git a/sources/transport.py b/sources/transport.py
index 4065da3..3db5262 100644
--- a/sources/transport.py
+++ b/sources/transport.py
@@ -15,6 +15,7 @@ import os
 import re
 import socket
 from dataclasses import dataclass, field
+from datetime import date
 from typing import Callable, Protocol
 from urllib.parse import parse_qsl, quote, quote_plus, unquote, urlencode, urlsplit
 from xml.etree import ElementTree
@@ -25,6 +26,8 @@ from travel_windows.contracts import (
     DATA_GO_SECRET_REFERENCE,
     LEGACY_SCHEDULE_SOURCE_CODE,
     LEGACY_SCHEDULE_ARRIVALS_LOCATOR,
+    DURATION_SOURCE_CODE,
+    DURATION_SOURCE_LOCATOR,
     SCHEDULE_SOURCE_CODE,
     SCHEDULE_ARRIVALS_LOCATOR,
     SCHEDULE_DEPARTURES_LOCATOR,
@@ -48,6 +51,10 @@ WARNING_MAX_RESPONSE_BYTES = 4_096
 SCHEDULE_PAGE_MAX_RESPONSE_BYTES = 1_048_576
 SCHEDULE_PAGE_SIZE = 100
 SCHEDULE_MAX_PAGES_PER_DIRECTION = 100
+DURATION_PAGE_MAX_RESPONSE_BYTES = 524_288
+DURATION_METADATA_MAX_RESPONSE_BYTES = 262_144
+DURATION_LIMIT_MAX_RESPONSE_BYTES = 4_096
+DURATION_ARCHIVE_MAX_RESPONSE_BYTES = 16_777_216
 
 ATTEMPT_SUCCEEDED = "SUCCEEDED"
 ATTEMPT_RETRYABLE_FAILED = "RETRYABLE_FAILED"
@@ -73,6 +80,25 @@ ROLE_SCHEDULE_DEPARTURE = "SCHEDULE_DEPARTURE"
 ROLE_SCHEDULE_ARRIVAL = "SCHEDULE_ARRIVAL"
 ROLE_DESTINATION_CITY = "DESTINATION_CITY"
 ROLE_LEGACY_ARRIVAL = "LEGACY_ARRIVAL"
+ROLE_DURATION_PAGE = "DURATION_PAGE"
+ROLE_DURATION_METADATA = "DURATION_METADATA"
+ROLE_DURATION_LIMIT = "DURATION_LIMIT"
+ROLE_DURATION_ARCHIVE = "DURATION_ARCHIVE"
+
+_DURATION_DATASET_ID = "15151728"
+_DURATION_DATASET_PATH = "/data/15151728/fileData.do"
+_DURATION_METADATA_PATH = "/tcs/dss/selectFileDataDownload.do"
+_DURATION_LIMIT_PATH = "/cmm/cmm/check-limit.json"
+_DURATION_DOWNLOAD_PATH = "/cmm/cmm/fileDownload.do"
+_DURATION_DETAIL_PATTERN = re.compile(
+    rb"fileDetailObj\.fn_fileDataDown\('15151728',\s*"
+    rb"'(uddi:[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12})',"
+    rb"\s*'',\s*'1',\s*'[0-9]+'\)"
+)
+_DURATION_TITLE_DATE_PATTERN = re.compile(
+    rb"content=\"[^\"]*_[^\"]*_(20[0-9]{6})\""
+)
+_DURATION_ATTACHMENT_ID = re.compile(r"^FILE_[0-9]{15}$")
 
 
 def load_aviation_secret_reference(
@@ -112,6 +138,20 @@ class AviationReferenceFetchResult:
     )
 
 
+@dataclass(frozen=True, slots=True)
+class RouteDurationReferenceFetchResult:
+    succeeded: bool
+    reference_date: str = ""
+    reference_version_sha256: str = ""
+    unchanged: bool = False
+    archive: bytes = field(default=b"", repr=False)
+    failure_code: str = ""
+    lineage: tuple[AviationResponseLineage, ...] = field(
+        default=(),
+        repr=False,
+    )
+
+
 @dataclass(frozen=True, slots=True)
 class RequestFingerprint:
     revision: str
@@ -282,6 +322,8 @@ _MAX_SECRET_CHARACTERS = 512
 class _WireResponse:
     status: int
     body: bytes = field(repr=False)
+    content_type: str = ""
+    content_disposition: str = ""
 
 
 @dataclass(frozen=True, slots=True)
@@ -312,6 +354,8 @@ def _read_once(
     max_response_bytes: int,
     request_headers: dict[str, str],
     connection_factory: ConnectionFactory,
+    method: str = "GET",
+    request_body: bytes | None = None,
 ) -> _WireResponse | _WireFailure:
     connection: _ConnectionLike | None = None
     observed_status: int | None = None
@@ -324,10 +368,12 @@ def _read_once(
         if connection.sock is None:
             return _WireFailure(FAILURE_TRANSPORT)
         connection.sock.settimeout(float(read_timeout_seconds))
+        if method not in {"GET", "POST"}:
+            return _WireFailure(FAILURE_TRANSPORT)
         connection.request(
-            "GET",
+            method,
             request_target,
-            body=None,
+            body=request_body,
             headers=dict(request_headers),
         )
         response = connection.getresponse()
@@ -353,7 +399,12 @@ def _read_once(
             return _WireFailure(
                 FAILURE_RESPONSE_TOO_LARGE, http_status=response.status
             )
-        return _WireResponse(response.status, response_body)
+        return _WireResponse(
+            response.status,
+            response_body,
+            response.getheader("Content-Type", "") or "",
+            response.getheader("Content-Disposition", "") or "",
+        )
     except (TimeoutError, socket.timeout):
         return _WireFailure(FAILURE_TIMEOUT, http_status=observed_status)
     except (OSError, http.client.HTTPException):
@@ -634,6 +685,370 @@ def _execute_aviation_attempt(
     return result
 
 
+def _duration_request_fingerprint(
+    *,
+    revision: str,
+    method: str,
+    path: str,
+    body: bytes = b"",
+) -> RequestFingerprint:
+    canonical = b"\n".join(
+        (
+            method.encode("ascii"),
+            b"https",
+            b"www.data.go.kr",
+            path.encode("ascii"),
+            hashlib.sha256(body).hexdigest().encode("ascii"),
+        )
+    )
+    return RequestFingerprint(
+        revision=revision,
+        normalized_request_sha256=hashlib.sha256(canonical).hexdigest(),
+    )
+
+
+def _duration_wire_attempt(
+    *,
+    role: str,
+    role_sequence: int,
+    revision: str,
+    method: str,
+    path: str,
+    body: bytes,
+    accept: str,
+    max_response_bytes: int,
+    connect_timeout_seconds: int | float,
+    read_timeout_seconds: int | float,
+    connection_factory: ConnectionFactory,
+    attempt_executor: AviationAttemptExecutor,
+) -> AviationResponseLineage:
+    fingerprint = _duration_request_fingerprint(
+        revision=revision,
+        method=method,
+        path=path,
+        body=body,
+    )
+    headers = {
+        **_COMMON_REQUEST_HEADERS,
+        "Accept": accept,
+    }
+    if method == "POST":
+        headers["Content-Type"] = "application/x-www-form-urlencoded"
+        headers["Content-Length"] = str(len(body))
+
+    def call_once() -> SingleAttemptResult:
+        wire = _read_once(
+            host="www.data.go.kr",
+            request_target=path,
+            connect_timeout_seconds=connect_timeout_seconds,
+            read_timeout_seconds=read_timeout_seconds,
+            max_response_bytes=max_response_bytes,
+            request_headers=headers,
+            connection_factory=connection_factory,
+            method=method,
+            request_body=body or None,
+        )
+        if isinstance(wire, _WireFailure):
+            if wire.http_status is not None:
+                classified = _classify_http_failure(
+                    fingerprint,
+                    wire.http_status,
+                )
+                if classified is not None:
+                    return classified
+            return _failed(
+                fingerprint,
+                wire.failure_code,
+                http_status=(
+                    wire.http_status
+                    if wire.failure_code == FAILURE_RESPONSE_TOO_LARGE
+                    else None
+                ),
+            )
+        classified = _classify_http_failure(fingerprint, wire.status)
+        if classified is not None:
+            return classified
+        if role == ROLE_DURATION_ARCHIVE:
+            content_type = wire.content_type.partition(";")[0].strip().lower()
+            disposition = wire.content_disposition.casefold()
+            if (
+                content_type != "application/octet-stream"
+                or "attachment" not in disposition
+                or ".zip" not in disposition
+                or not wire.body.startswith(b"PK")
+            ):
+                return _failed(
+                    fingerprint,
+                    FAILURE_PROVIDER_ERROR,
+                    http_status=wire.status,
+                )
+        return _succeeded(
+            fingerprint,
+            http_status=wire.status,
+            body=wire.body,
+        )
+
+    attempt = _execute_aviation_attempt(
+        source_code=DURATION_SOURCE_CODE,
+        fingerprint=fingerprint,
+        call_once=call_once,
+        attempt_executor=attempt_executor,
+    )
+    return AviationResponseLineage(
+        source_code=DURATION_SOURCE_CODE,
+        role=role,
+        role_sequence=role_sequence,
+        attempt=attempt,
+    )
+
+
+def _duration_page_identity(body: bytes) -> tuple[str, str] | None:
+    try:
+        body.decode("utf-8", errors="strict")
+    except UnicodeError:
+        return None
+    details = set(_DURATION_DETAIL_PATTERN.findall(body))
+    dates = set(_DURATION_TITLE_DATE_PATTERN.findall(body))
+    if len(details) != 1 or len(dates) != 1:
+        return None
+    detail = next(iter(details)).decode("ascii")
+    compact_date = next(iter(dates)).decode("ascii")
+    try:
+        reference_date = (
+            f"{compact_date[0:4]}-{compact_date[4:6]}-{compact_date[6:8]}"
+        )
+        if date.fromisoformat(reference_date).strftime("%Y%m%d") != compact_date:
+            return None
+    except ValueError:
+        return None
+    return detail, reference_date
+
+
+def _duration_download_identity(
+    body: bytes,
+    *,
+    detail_id: str,
+    reference_date: str,
+) -> tuple[str, str] | None:
+    try:
+        document = json.loads(body.decode("utf-8"))
+        if not isinstance(document, dict) or document.get("status") is not True:
+            return None
+        attachment_id = document["atchFileId"]
+        file_detail_sn = str(document["fileDetailSn"])
+        detail = document["dpk"]
+        file_info = document["fileDataRegistVO"]
+        dataset_info = document["dataSetFileDetailInfo"]
+        if (
+            not isinstance(attachment_id, str)
+            or _DURATION_ATTACHMENT_ID.fullmatch(attachment_id) is None
+            or file_detail_sn != "1"
+            or detail != detail_id
+            or not isinstance(file_info, dict)
+            or file_info.get("atchFileId") != attachment_id
+            or str(file_info.get("fileDetailSn")) != "1"
+            or file_info.get("orginlFileNm")
+            != "수출입 물류 플랫폼_항공 스케줄 기본정보.zip"
+            or not isinstance(dataset_info, dict)
+            or dataset_info.get("publicDataPk") != _DURATION_DATASET_ID
+            or dataset_info.get("publicDataDetailPk") != detail_id
+            or dataset_info.get("stdrDe")
+            != date.fromisoformat(reference_date).strftime("%m/%d/%Y")
+            or not str(dataset_info.get("dataNm", "")).endswith(
+                date.fromisoformat(reference_date).strftime("_%Y%m%d")
+            )
+        ):
+            return None
+    except (KeyError, TypeError, ValueError, UnicodeError, json.JSONDecodeError):
+        return None
+    return attachment_id, file_detail_sn
+
+
+def fetch_route_duration_reference(
+    *,
+    official_locator: str,
+    connect_timeout_seconds: int | float,
+    read_timeout_seconds: int | float,
+    known_reference_version_sha256: str = "",
+    connection_factory: ConnectionFactory = _default_connection_factory,
+    attempt_executor: AviationAttemptExecutor = _direct_attempt_executor,
+) -> RouteDurationReferenceFetchResult:
+    """Discover and fetch the current official route-duration archive once."""
+
+    if (
+        official_locator != DURATION_SOURCE_LOCATOR
+        or not _valid_timeout(connect_timeout_seconds)
+        or not _valid_timeout(read_timeout_seconds)
+        or (
+            known_reference_version_sha256
+            and re.fullmatch(
+                r"[0-9a-f]{64}", known_reference_version_sha256
+            )
+            is None
+        )
+    ):
+        return RouteDurationReferenceFetchResult(
+            False,
+            failure_code=FAILURE_HTTP_CLIENT,
+        )
+
+    lineage: list[AviationResponseLineage] = []
+    page = _duration_wire_attempt(
+        role=ROLE_DURATION_PAGE,
+        role_sequence=1,
+        revision="ROUTE_DURATION_PAGE_V1",
+        method="GET",
+        path=_DURATION_DATASET_PATH,
+        body=b"",
+        accept="text/html",
+        max_response_bytes=DURATION_PAGE_MAX_RESPONSE_BYTES,
+        connect_timeout_seconds=connect_timeout_seconds,
+        read_timeout_seconds=read_timeout_seconds,
+        connection_factory=connection_factory,
+        attempt_executor=attempt_executor,
+    )
+    lineage.append(page)
+    if not page.succeeded:
+        return RouteDurationReferenceFetchResult(
+            False,
+            failure_code=page.attempt.failure_code,
+            lineage=tuple(lineage),
+        )
+    page_identity = _duration_page_identity(page.attempt.body)
+    if page_identity is None:
+        return RouteDurationReferenceFetchResult(
+            False,
+            failure_code=FAILURE_PROVIDER_ERROR,
+            lineage=tuple(lineage),
+        )
+    detail_id, reference_date = page_identity
+
+    metadata_body = urlencode(
+        (
+            ("publicDataDetailPk", detail_id),
+            ("publicDataPk", _DURATION_DATASET_ID),
+            ("atchFileId", ""),
+            ("fileDetailSn", "1"),
+            ("publicDataTyCode", "PR0051"),
+        )
+    ).encode("ascii")
+    metadata = _duration_wire_attempt(
+        role=ROLE_DURATION_METADATA,
+        role_sequence=1,
+        revision="ROUTE_DURATION_METADATA_V1",
+        method="POST",
+        path=_DURATION_METADATA_PATH,
+        body=metadata_body,
+        accept="application/json, text/plain",
+        max_response_bytes=DURATION_METADATA_MAX_RESPONSE_BYTES,
+        connect_timeout_seconds=connect_timeout_seconds,
+        read_timeout_seconds=read_timeout_seconds,
+        connection_factory=connection_factory,
+        attempt_executor=attempt_executor,
+    )
+    lineage.append(metadata)
+    if not metadata.succeeded:
+        return RouteDurationReferenceFetchResult(
+            False,
+            failure_code=metadata.attempt.failure_code,
+            lineage=tuple(lineage),
+        )
+    download_identity = _duration_download_identity(
+        metadata.attempt.body,
+        detail_id=detail_id,
+        reference_date=reference_date,
+    )
+    if download_identity is None:
+        return RouteDurationReferenceFetchResult(
+            False,
+            failure_code=FAILURE_PROVIDER_ERROR,
+            lineage=tuple(lineage),
+        )
+    attachment_id, file_detail_sn = download_identity
+    reference_version_sha256 = hashlib.sha256(
+        (
+            "ROUTE_DURATION_REFERENCE_VERSION_V1\n"
+            f"{detail_id}\n{attachment_id}\n{file_detail_sn}\n"
+            f"{reference_date}\n"
+        ).encode("ascii")
+    ).hexdigest()
+    if known_reference_version_sha256 == reference_version_sha256:
+        return RouteDurationReferenceFetchResult(
+            True,
+            reference_date=reference_date,
+            reference_version_sha256=reference_version_sha256,
+            unchanged=True,
+            lineage=tuple(lineage),
+        )
+
+    limit_body = urlencode(
+        (("atchFileId", attachment_id), ("fileDetailSn", file_detail_sn))
+    ).encode("ascii")
+    limit = _duration_wire_attempt(
+        role=ROLE_DURATION_LIMIT,
+        role_sequence=1,
+        revision="ROUTE_DURATION_LIMIT_V1",
+        method="POST",
+        path=_DURATION_LIMIT_PATH,
+        body=limit_body,
+        accept="application/json",
+        max_response_bytes=DURATION_LIMIT_MAX_RESPONSE_BYTES,
+        connect_timeout_seconds=connect_timeout_seconds,
+        read_timeout_seconds=read_timeout_seconds,
+        connection_factory=connection_factory,
+        attempt_executor=attempt_executor,
+    )
+    lineage.append(limit)
+    if not limit.succeeded:
+        return RouteDurationReferenceFetchResult(
+            False,
+            failure_code=limit.attempt.failure_code,
+            lineage=tuple(lineage),
+        )
+    try:
+        limit_document = json.loads(limit.attempt.body.decode("utf-8"))
+    except (UnicodeError, json.JSONDecodeError):
+        limit_document = None
+    if limit_document != {"needCaptcha": False}:
+        return RouteDurationReferenceFetchResult(
+            False,
+            failure_code=FAILURE_AUTHENTICATION,
+            lineage=tuple(lineage),
+        )
+
+    download_query = urlencode(
+        (("atchFileId", attachment_id), ("fileDetailSn", file_detail_sn))
+    )
+    archive = _duration_wire_attempt(
+        role=ROLE_DURATION_ARCHIVE,
+        role_sequence=1,
+        revision="ROUTE_DURATION_ARCHIVE_V1",
+        method="GET",
+        path=f"{_DURATION_DOWNLOAD_PATH}?{download_query}",
+        body=b"",
+        accept="application/octet-stream",
+        max_response_bytes=DURATION_ARCHIVE_MAX_RESPONSE_BYTES,
+        connect_timeout_seconds=connect_timeout_seconds,
+        read_timeout_seconds=read_timeout_seconds,
+        connection_factory=connection_factory,
+        attempt_executor=attempt_executor,
+    )
+    lineage.append(archive)
+    if not archive.succeeded:
+        return RouteDurationReferenceFetchResult(
+            False,
+            failure_code=archive.attempt.failure_code,
+            lineage=tuple(lineage),
+        )
+    return RouteDurationReferenceFetchResult(
+        True,
+        reference_date=reference_date,
+        reference_version_sha256=reference_version_sha256,
+        archive=archive.attempt.body,
+        lineage=tuple(lineage),
+    )
+
+
 def _fetch_data_go_json_once(
     *,
     locator: str,
diff --git a/travel_windows/contracts.py b/travel_windows/contracts.py
index 7dee196..d46bfc1 100644
--- a/travel_windows/contracts.py
+++ b/travel_windows/contracts.py
@@ -48,13 +48,15 @@ LEGACY_SCHEDULE_ATTRIBUTION = "인천국제공항공사|공공데이터포털"
 LEGACY_SCHEDULE_FIELD_SCOPE = "ICN_LEGACY_ARRIVALS_CROSSCHECK_V1"
 
 DURATION_SOURCE_CODE = "PORT_LOGISTICS_ROUTE_DURATION_15151728"
-DURATION_SOURCE_REVISION = "travel-v1"
+DURATION_V1_SOURCE_REVISION = "travel-v1"
+DURATION_SOURCE_REVISION = "travel-v2"
 DURATION_SOURCE_OWNER = "대한민국 해양수산부"
 DURATION_SOURCE_LOCATOR = (
     "https://www.data.go.kr/data/15151728/fileData.do"
 )
 DURATION_ATTRIBUTION = "해양수산부 수출입 물류 플랫폼|공공데이터포털"
-DURATION_FIELD_SCOPE = "ROUTE_DURATION_V1"
+DURATION_V1_FIELD_SCOPE = "ROUTE_DURATION_V1"
+DURATION_FIELD_SCOPE = "ROUTE_DURATION_DERIVATION_V2"
 
 SCHEDULE_CONTRACT_TEXT = (
     "ICN schedule data.go v2|response(header(resultCode,resultMsg),body("
@@ -70,14 +72,34 @@ SCHEDULE_SCHEMA_TEXT = (
     "carrier_name,flight_number,master_flight_number,destination_iata,"
     "icn_event_time,valid_from,valid_until,weekdays])"
 )
-DURATION_CONTRACT_TEXT = (
+DURATION_V1_CONTRACT_TEXT = (
     "route duration v1|destination_iata,outbound_minutes,inbound_minutes,"
     "reference_date,reference_locator|exact-dataset-15151728"
 )
-DURATION_SCHEMA_TEXT = (
+DURATION_V1_SCHEMA_TEXT = (
     "csv(destination_iata,outbound_minutes,inbound_minutes,reference_date,"
     "reference_locator)"
 )
+# Historical names remain stable for sealed V1 evidence and rollback readers.
+DURATION_CONTRACT_TEXT = DURATION_V1_CONTRACT_TEXT
+DURATION_SCHEMA_TEXT = DURATION_V1_SCHEMA_TEXT
+DURATION_REFERENCE_CONTRACT_TEXT = (
+    "route duration reference v2|dataset=15151728|anonymous-page-metadata-"
+    "captcha-gated-download|outer-zip(codebook-utf8,nested-zip(inner-cp949-"
+    "csv*3))|ICN-curated13|hash-only-raw-evidence|review-required"
+)
+DURATION_REFERENCE_SCHEMA_TEXT = (
+    "route-duration-archive-v1|outer-zip:codebook-csv+nested-zip|inner:3-"
+    "cp949-csv|exact-48-columns|flight-duration:minutes-or-H:MM|reference-"
+    "date:YYYYMMDD"
+)
+DURATION_DERIVATION_TEXT = (
+    "route-duration-derivation-v1|ICN-curated13|directional|use-Y|cancel-and-"
+    "30..720-invalid|occurrence(route,direction,flight-date,planned-dates,"
+    "aircraft)|upper-median-occurrence|minimum-5-occurrences-5-days|MAD:max("
+    "30,3x1.4826)|retained>=5-and>=70pct|IQR-upper-median-delta<=15|upper-"
+    "median-final"
+)
 
 
 def _sha256(value: str) -> str:
@@ -86,8 +108,19 @@ def _sha256(value: str) -> str:
 
 SCHEDULE_CONTRACT_FINGERPRINT_SHA256 = _sha256(SCHEDULE_CONTRACT_TEXT)
 SCHEDULE_SCHEMA_FINGERPRINT_SHA256 = _sha256(SCHEDULE_SCHEMA_TEXT)
-DURATION_CONTRACT_FINGERPRINT_SHA256 = _sha256(DURATION_CONTRACT_TEXT)
-DURATION_SCHEMA_FINGERPRINT_SHA256 = _sha256(DURATION_SCHEMA_TEXT)
+DURATION_V1_CONTRACT_FINGERPRINT_SHA256 = _sha256(DURATION_V1_CONTRACT_TEXT)
+DURATION_V1_SCHEMA_FINGERPRINT_SHA256 = _sha256(DURATION_V1_SCHEMA_TEXT)
+DURATION_CONTRACT_FINGERPRINT_SHA256 = (
+    DURATION_V1_CONTRACT_FINGERPRINT_SHA256
+)
+DURATION_SCHEMA_FINGERPRINT_SHA256 = DURATION_V1_SCHEMA_FINGERPRINT_SHA256
+DURATION_REFERENCE_CONTRACT_FINGERPRINT_SHA256 = _sha256(
+    DURATION_REFERENCE_CONTRACT_TEXT
+)
+DURATION_REFERENCE_SCHEMA_FINGERPRINT_SHA256 = _sha256(
+    DURATION_REFERENCE_SCHEMA_TEXT
+)
+DURATION_DERIVATION_FINGERPRINT_SHA256 = _sha256(DURATION_DERIVATION_TEXT)
 CITY_CONTRACT_TEXT = (
     "ICN destination city v3|response(header(resultCode,resultMsg),body("
     "items(rows-or-item-wrapper(countryName,airportName,airportCode))))|exact-curated-13|"
diff --git a/travel_windows/duration_reference.py b/travel_windows/duration_reference.py
new file mode 100644
index 0000000..4930e5a
--- /dev/null
+++ b/travel_windows/duration_reference.py
@@ -0,0 +1,671 @@
+from __future__ import annotations
+
+import csv
+import hashlib
+import io
+import re
+import stat
+import statistics
+import zipfile
+import zlib
+from dataclasses import dataclass, field
+from datetime import date, datetime
+from pathlib import PurePosixPath
+
+from .contracts import (
+    DURATION_DERIVATION_FINGERPRINT_SHA256,
+    DURATION_REFERENCE_SCHEMA_FINGERPRINT_SHA256,
+    DURATION_SOURCE_LOCATOR,
+)
+
+
+# The limits are deliberately independent at each archive layer.  Central-directory
+# sizes are checked before a member is read and the actual read is bounded again.
+MAX_ARCHIVE_BYTES = 64 * 1024 * 1024
+MAX_NESTED_ARCHIVE_BYTES = 64 * 1024 * 1024
+MAX_CODEBOOK_BYTES = 2 * 1024 * 1024
+MAX_INNER_CSV_BYTES = 128 * 1024 * 1024
+MAX_INNER_TOTAL_BYTES = 256 * 1024 * 1024
+MAX_COMPRESSION_RATIO = 250
+MAX_SOURCE_ROWS = 1_000_000
+
+OFFICIAL_DURATION_REFERENCE_HEADER = (
+    "항공편번호",
+    "항공편일시",
+    "항공편순번",
+    "부터공항코드",
+    "까지공항코드",
+    "항공기운송상태값",
+    "항공기운송종류값",
+    "항공기종류값",
+    "항공기등록번호",
+    "항공사명",
+    "국제민간항공기구코드",
+    "부터계획일시",
+    "부터예상일시",
+    "부터일시",
+    "까지계획일시",
+    "까지예상일시",
+    "까지일시",
+    "비행시간",
+    "생성사용자아이디",
+    "생성일시",
+    "수정사용자아이디",
+    "수정일시",
+    "부터위치코드",
+    "부터위치명",
+    "부터도시명",
+    "부터한글도시명",
+    "부터나라코드",
+    "부터나라명",
+    "부터표시순번",
+    "부터생성사용자아이디",
+    "부터생성일시",
+    "부터수정사용자아이디",
+    "부터수정일시",
+    "까지위치코드",
+    "까지위치명",
+    "까지도시명",
+    "까지한글도시명",
+    "까지나라코드",
+    "까지나라명",
+    "까지표시순번",
+    "까지생성사용자아이디",
+    "까지생성일시",
+    "까지수정사용자아이디",
+    "까지수정일시",
+    "국제항공운송협회코드",
+    "한글항공사명",
+    "나라코드",
+    "사용여부",
+)
+_OFFICIAL_CODEBOOK_HEADER = ("항목명", "코드", "내용", "참조항목명")
+
+_IATA = re.compile(r"^[A-Z]{3}$")
+_INTEGER_MINUTES = re.compile(r"^[0-9]+$")
+_HOURS_AND_MINUTES = re.compile(r"^([0-9]{1,2}):([0-9]{2})$")
+_ALLOWED_COMPRESSION = {zipfile.ZIP_STORED, zipfile.ZIP_DEFLATED}
+
+_FLIGHT_DATE = 1
+_FROM_AIRPORT = 3
+_TO_AIRPORT = 4
+_STATE = 5
+_AIRCRAFT_REGISTRATION = 8
+_FROM_PLANNED_DATE = 11
+_TO_PLANNED_DATE = 14
+_DURATION = 17
+_IS_ACTIVE = 47
+
+
+@dataclass(frozen=True, slots=True)
+class DirectionDerivationCounts:
+    """Auditable counts for one route direction, without source row contents."""
+
+    source_row_count: int
+    invalid_row_count: int
+    cancelled_row_count: int
+    occurrence_count: int
+    outlier_count: int
+    retained_count: int
+
+
+@dataclass(frozen=True, slots=True)
+class DerivedRouteDuration:
+    destination_iata: str
+    outbound_minutes: int
+    inbound_minutes: int
+    outbound: DirectionDerivationCounts
+    inbound: DirectionDerivationCounts
+
+    @property
+    def outbound_counts(self) -> DirectionDerivationCounts:
+        return self.outbound
+
+    @property
+    def inbound_counts(self) -> DirectionDerivationCounts:
+        return self.inbound
+
+    @property
+    def outbound_source_rows(self) -> int:
+        return self.outbound.source_row_count
+
+    @property
+    def outbound_invalid_rows(self) -> int:
+        return self.outbound.invalid_row_count
+
+    @property
+    def outbound_cancelled_rows(self) -> int:
+        return self.outbound.cancelled_row_count
+
+    @property
+    def outbound_occurrences(self) -> int:
+        return self.outbound.occurrence_count
+
+    @property
+    def outbound_outliers(self) -> int:
+        return self.outbound.outlier_count
+
+    @property
+    def outbound_retained(self) -> int:
+        return self.outbound.retained_count
+
+    @property
+    def inbound_source_rows(self) -> int:
+        return self.inbound.source_row_count
+
+    @property
+    def inbound_invalid_rows(self) -> int:
+        return self.inbound.invalid_row_count
+
+    @property
+    def inbound_cancelled_rows(self) -> int:
+        return self.inbound.cancelled_row_count
+
+    @property
+    def inbound_occurrences(self) -> int:
+        return self.inbound.occurrence_count
+
+    @property
+    def inbound_outliers(self) -> int:
+        return self.inbound.outlier_count
+
+    @property
+    def inbound_retained(self) -> int:
+        return self.inbound.retained_count
+
+    @property
+    def source_row_count(self) -> int:
+        return self.outbound.source_row_count + self.inbound.source_row_count
+
+    @property
+    def invalid_row_count(self) -> int:
+        return self.outbound.invalid_row_count + self.inbound.invalid_row_count
+
+    @property
+    def cancelled_row_count(self) -> int:
+        return self.outbound.cancelled_row_count + self.inbound.cancelled_row_count
+
+    @property
+    def occurrence_count(self) -> int:
+        return self.outbound.occurrence_count + self.inbound.occurrence_count
+
+    @property
+    def outlier_count(self) -> int:
+        return self.outbound.outlier_count + self.inbound.outlier_count
+
+    @property
+    def retained_count(self) -> int:
+        return self.outbound.retained_count + self.inbound.retained_count
+
+
+@dataclass(frozen=True, slots=True)
+class DurationReferenceSuccess:
+    routes: tuple[DerivedRouteDuration, ...]
+    source_row_count: int
+    reference_date: date
+    reference_locator: str
+    body_sha256: str
+    source_byte_count: int
+    observed_schema_fingerprint_sha256: str = (
+        DURATION_REFERENCE_SCHEMA_FINGERPRINT_SHA256
+    )
+    derivation_fingerprint_sha256: str = (
+        DURATION_DERIVATION_FINGERPRINT_SHA256
+    )
+
+    @property
+    def source_sha256(self) -> str:
+        return self.body_sha256
+
+    @property
+    def archive_sha256(self) -> str:
+        return self.body_sha256
+
+    @property
+    def byte_count(self) -> int:
+        return self.source_byte_count
+
+    @property
+    def derivation_contract_fingerprint_sha256(self) -> str:
+        return self.derivation_fingerprint_sha256
+
+
+@dataclass(frozen=True, slots=True)
+class DurationReferenceFailure:
+    failure_code: str
+    observed_schema_fingerprint_sha256: str = ""
+
+
+# Explicit aliases keep the result terminology convenient at collection call sites.
+DurationDerivationSuccess = DurationReferenceSuccess
+DurationDerivationFailure = DurationReferenceFailure
+
+
+class _ReferenceProblem(Exception):
+    def __init__(self, failure_code: str, observed_schema: str = "") -> None:
+        super().__init__(failure_code)
+        self.failure_code = failure_code
+        self.observed_schema = observed_schema
+
+
+@dataclass(slots=True)
+class _DirectionSource:
+    source_row_count: int = 0
+    invalid_row_count: int = 0
+    cancelled_row_count: int = 0
+    occurrences: dict[tuple[str, str, str, str], list[int]] = field(
+        default_factory=dict
+    )
+
+
+@dataclass(frozen=True, slots=True)
+class _DirectionResult:
+    minutes: int
+    counts: DirectionDerivationCounts
+
+
+def _schema_hash(fields: tuple[str, ...]) -> str:
+    return hashlib.sha256("\x1f".join(fields).encode("utf-8")).hexdigest()
+
+
+def _is_safe_member_name(name: str) -> bool:
+    if (
+        not name
+        or "\x00" in name
+        or "/" in name
+        or "\\" in name
+        or ":" in name
+        or name in {".", ".."}
+    ):
+        return False
+    path = PurePosixPath(name)
+    return not path.is_absolute() and path.name == name
+
+
+def _checked_members(
+    archive: zipfile.ZipFile,
+    *,
+    expected_count: int,
+    total_limit: int,
+) -> tuple[zipfile.ZipInfo, ...]:
+    members = tuple(archive.infolist())
+    if len(members) != expected_count or len({row.filename for row in members}) != len(
+        members
+    ):
+        raise _ReferenceProblem("SCHEMA_MISMATCH")
+
+    total_size = 0
+    for member in members:
+        original_name = getattr(member, "orig_filename", member.filename)
+        unix_mode = (member.external_attr >> 16) & 0xFFFF
+        if (
+            not _is_safe_member_name(original_name)
+            or member.is_dir()
+            or stat.S_ISLNK(unix_mode)
+            or member.flag_bits & 0x41
+            or member.compress_type not in _ALLOWED_COMPRESSION
+            or member.file_size <= 0
+            or member.compress_size <= 0
+            or member.file_size > total_limit
+            or member.file_size > member.compress_size * MAX_COMPRESSION_RATIO
+        ):
+            raise _ReferenceProblem("INVALID_VALUE")
+        total_size += member.file_size
+        if total_size > total_limit:
+            raise _ReferenceProblem("INVALID_VALUE")
+    return members
+
+
+def _read_member(
+    archive: zipfile.ZipFile,
+    member: zipfile.ZipInfo,
+    *,
+    byte_limit: int,
+) -> bytes:
+    if member.file_size > byte_limit:
+        raise _ReferenceProblem("INVALID_VALUE")
+    chunks: list[bytes] = []
+    total = 0
+    with archive.open(member, "r") as stream:
+        while True:
+            chunk = stream.read(min(64 * 1024, byte_limit + 1 - total))
+            if not chunk:
+                break
+            chunks.append(chunk)
+            total += len(chunk)
+            if total > byte_limit:
+                raise _ReferenceProblem("INVALID_VALUE")
+    if total != member.file_size:
+        raise _ReferenceProblem("SYNTAX_ERROR")
+    return b"".join(chunks)
+
+
+def _validate_codebook(payload: bytes) -> None:
+    if not payload.startswith(b"\xef\xbb\xbf"):
+        raise _ReferenceProblem("ENCODING_ERROR")
+    try:
+        text = payload.decode("utf-8-sig", errors="strict")
+    except UnicodeDecodeError as error:
+        raise _ReferenceProblem("ENCODING_ERROR") from error
+    if "\x00" in text:
+        raise _ReferenceProblem("SYNTAX_ERROR")
+    try:
+        reader = csv.reader(io.StringIO(text, newline=""), strict=True)
+        header = tuple(next(reader))
+        if header != _OFFICIAL_CODEBOOK_HEADER:
+            raise _ReferenceProblem("SCHEMA_MISMATCH", _schema_hash(header))
+        row_count = 0
+        for row in reader:
+            row_count += 1
+            if len(row) != len(_OFFICIAL_CODEBOOK_HEADER):
+                raise _ReferenceProblem("SCHEMA_MISMATCH")
+        if row_count == 0:
+            raise _ReferenceProblem("REQUIRED_VALUE_MISSING")
+    except StopIteration as error:
+        raise _ReferenceProblem("SCHEMA_MISMATCH") from error
+    except csv.Error as error:
+        raise _ReferenceProblem("SYNTAX_ERROR") from error
+
+
+def _open_outer_archive(archive: bytes) -> tuple[bytes, ...]:
+    if not archive.startswith(b"PK\x03\x04"):
+        raise _ReferenceProblem("SYNTAX_ERROR")
+    try:
+        with zipfile.ZipFile(io.BytesIO(archive), "r") as outer:
+            members = _checked_members(
+                outer,
+                expected_count=2,
+                total_limit=MAX_ARCHIVE_BYTES + MAX_CODEBOOK_BYTES,
+            )
+            csv_members = tuple(
+                row for row in members if row.filename.lower().endswith(".csv")
+            )
+            zip_members = tuple(
+                row for row in members if row.filename.lower().endswith(".zip")
+            )
+            if len(csv_members) != 1 or len(zip_members) != 1:
+                raise _ReferenceProblem("SCHEMA_MISMATCH")
+            # Reading the codebook validates its CRC even though its descriptive
+            # contents are not part of the route derivation.
+            codebook = _read_member(
+                outer, csv_members[0], byte_limit=MAX_CODEBOOK_BYTES
+            )
+            _validate_codebook(codebook)
+            nested_payload = _read_member(
+                outer, zip_members[0], byte_limit=MAX_NESTED_ARCHIVE_BYTES
+            )
+
+        if not nested_payload.startswith(b"PK\x03\x04"):
+            raise _ReferenceProblem("SYNTAX_ERROR")
+        with zipfile.ZipFile(io.BytesIO(nested_payload), "r") as inner:
+            members = _checked_members(
+                inner,
+                expected_count=3,
+                total_limit=MAX_INNER_TOTAL_BYTES,
+            )
+            if any(not row.filename.lower().endswith(".csv") for row in members):
+                raise _ReferenceProblem("SCHEMA_MISMATCH")
+            csv_payloads = tuple(
+                _read_member(inner, row, byte_limit=MAX_INNER_CSV_BYTES)
+                for row in sorted(members, key=lambda value: value.filename)
+            )
+        return csv_payloads
+    except _ReferenceProblem:
+        raise
+    except (
+        zipfile.BadZipFile,
+        zipfile.LargeZipFile,
+        EOFError,
+        NotImplementedError,
+        OSError,
+        RuntimeError,
+        ValueError,
+        zlib.error,
+    ) as error:
+        raise _ReferenceProblem("SYNTAX_ERROR") from error
+
+
+def _minutes(value: str) -> int | None:
+    normalized = value.strip()
+    if _INTEGER_MINUTES.fullmatch(normalized):
+        if len(normalized) > 6:
+            return None
+        result = int(normalized)
+    else:
+        match = _HOURS_AND_MINUTES.fullmatch(normalized)
+        if match is None:
+            return None
+        hours = int(match.group(1))
+        minutes = int(match.group(2))
+        if minutes >= 60:
+            return None
+        result = hours * 60 + minutes
+    if not 30 <= result <= 720:
+        return None
+    return result
+
+
+def _exact_iso_date(value: str) -> str | None:
+    normalized = value.strip()
+    try:
+        parsed = date.fromisoformat(normalized)
+    except ValueError:
+        return None
+    if parsed.isoformat() != normalized:
+        return None
+    return normalized
+
+
+def _is_cancelled(value: str) -> bool:
+    normalized = value.strip().casefold()
+    return normalized.startswith("cancel") or normalized.startswith("취소")
+
+
+def _upper_median(values: list[int]) -> int:
+    ordered = sorted(values)
+    return ordered[len(ordered) // 2]
+
+
+def _tukey_quartiles(values: list[int]) -> tuple[float, float]:
+    ordered = sorted(values)
+    midpoint = len(ordered) // 2
+    lower = ordered[:midpoint]
+    upper = ordered[midpoint:] if len(ordered) % 2 == 0 else ordered[midpoint + 1 :]
+    return float(statistics.median(lower)), float(statistics.median(upper))
+
+
+def _derive_direction(source: _DirectionSource) -> _DirectionResult:
+    occurrence_values = [
+        _upper_median(values) for values in source.occurrences.values()
+    ]
+    occurrence_count = len(occurrence_values)
+    distinct_dates = {identity[0] for identity in source.occurrences}
+    if occurrence_count < 5 or len(distinct_dates) < 5:
+        raise _ReferenceProblem("REQUIRED_VALUE_MISSING")
+
+    center = float(statistics.median(occurrence_values))
+    mad = float(
+        statistics.median([abs(value - center) for value in occurrence_values])
+    )
+    threshold = max(30.0, 3.0 * 1.4826 * mad)
+    retained = [
+        value for value in occurrence_values if abs(value - center) <= threshold
+    ]
+    if len(retained) < 5 or len(retained) * 10 < occurrence_count * 7:
+        raise _ReferenceProblem("CONFLICTING_VALUE")
+
+    final_minutes = _upper_median(retained)
+    if occurrence_count >= 8:
+        first_quartile, third_quartile = _tukey_quartiles(occurrence_values)
+        interquartile_range = third_quartile - first_quartile
+        lower_fence = first_quartile - 1.5 * interquartile_range
+        upper_fence = third_quartile + 1.5 * interquartile_range
+        iqr_retained = [
+            value
+            for value in occurrence_values
+            if lower_fence <= value <= upper_fence
+        ]
+        if not iqr_retained or abs(_upper_median(iqr_retained) - final_minutes) > 15:
+            raise _ReferenceProblem("CONFLICTING_VALUE")
+
+    return _DirectionResult(
+        minutes=final_minutes,
+        counts=DirectionDerivationCounts(
+            source_row_count=source.source_row_count,
+            invalid_row_count=source.invalid_row_count,
+            cancelled_row_count=source.cancelled_row_count,
+            occurrence_count=occurrence_count,
+            outlier_count=occurrence_count - len(retained),
+            retained_count=len(retained),
+        ),
+    )
+
+
+def _consume_csv(
+    payload: bytes,
+    *,
+    directions: dict[tuple[str, str], _DirectionSource],
+    required_iata_codes: frozenset[str],
+    source_row_count: int,
+) -> int:
+    try:
+        text = payload.decode("cp949", errors="strict")
+    except UnicodeDecodeError as error:
+        raise _ReferenceProblem("ENCODING_ERROR") from error
+    if "\x00" in text:
+        raise _ReferenceProblem("SYNTAX_ERROR")
+
+    try:
+        reader = csv.reader(io.StringIO(text, newline=""), strict=True)
+        header = tuple(next(reader))
+        if header != OFFICIAL_DURATION_REFERENCE_HEADER:
+            raise _ReferenceProblem("SCHEMA_MISMATCH", _schema_hash(header))
+
+        for row in reader:
+            source_row_count += 1
+            if source_row_count > MAX_SOURCE_ROWS:
+                raise _ReferenceProblem("INVALID_VALUE")
+            if len(row) != len(OFFICIAL_DURATION_REFERENCE_HEADER):
+                raise _ReferenceProblem(
+                    "SCHEMA_MISMATCH",
+                    _schema_hash((f"COLUMN_COUNT:{len(row)}",)),
+                )
+
+            from_airport = row[_FROM_AIRPORT].strip()
+            to_airport = row[_TO_AIRPORT].strip()
+            route_key: tuple[str, str] | None = None
+            if from_airport == "ICN" and to_airport in required_iata_codes:
+                route_key = (to_airport, "OUTBOUND")
+            elif to_airport == "ICN" and from_airport in required_iata_codes:
+                route_key = (from_airport, "INBOUND")
+            if route_key is None or row[_IS_ACTIVE].strip() != "Y":
+                continue
+
+            direction = directions[route_key]
+            direction.source_row_count += 1
+            if _is_cancelled(row[_STATE]):
+                direction.cancelled_row_count += 1
+                continue
+
+            duration = _minutes(row[_DURATION])
+            flight_date = _exact_iso_date(row[_FLIGHT_DATE])
+            from_planned_date = _exact_iso_date(row[_FROM_PLANNED_DATE])
+            to_planned_date = _exact_iso_date(row[_TO_PLANNED_DATE])
+            aircraft_registration = row[_AIRCRAFT_REGISTRATION].strip()
+            if (
+                duration is None
+                or flight_date is None
+                or from_planned_date is None
+                or to_planned_date is None
+                or not aircraft_registration
+            ):
+                direction.invalid_row_count += 1
+                continue
+
+            occurrence = (
+                flight_date,
+                from_planned_date,
+                to_planned_date,
+                aircraft_registration,
+            )
+            direction.occurrences.setdefault(occurrence, []).append(duration)
+    except StopIteration as error:
+        raise _ReferenceProblem("SCHEMA_MISMATCH") from error
+    except csv.Error as error:
+        raise _ReferenceProblem("SYNTAX_ERROR") from error
+    return source_row_count
+
+
+def derive_route_durations(
+    archive: bytes,
+    reference_date: date,
+    required_iata_codes: frozenset[str],
+) -> DurationReferenceSuccess | DurationReferenceFailure:
+    """Derive reviewed route-duration candidates from the exact official archive.
+
+    The returned failure contains only a closed code and, when safely available,
+    a header fingerprint.  Archive names, CSV values, and raw rows are never
+    included in the result.
+    """
+
+    if (
+        type(archive) is not bytes
+        or not 0 < len(archive) <= MAX_ARCHIVE_BYTES
+        or type(reference_date) is not date
+        or isinstance(reference_date, datetime)
+        or type(required_iata_codes) is not frozenset
+        or not required_iata_codes
+        or len(required_iata_codes) > 13
+        or "ICN" in required_iata_codes
+        or any(
+            type(code) is not str or _IATA.fullmatch(code) is None
+            for code in required_iata_codes
+        )
+    ):
+        return DurationReferenceFailure("INVALID_VALUE")
+
+    directions = {
+        (code, direction): _DirectionSource()
+        for code in required_iata_codes
+        for direction in ("OUTBOUND", "INBOUND")
+    }
+    try:
+        csv_payloads = _open_outer_archive(archive)
+        source_row_count = 0
+        for payload in csv_payloads:
+            source_row_count = _consume_csv(
+                payload,
+                directions=directions,
+                required_iata_codes=required_iata_codes,
+                source_row_count=source_row_count,
+            )
+        if source_row_count == 0:
+            raise _ReferenceProblem("REQUIRED_VALUE_MISSING")
+
+        routes: list[DerivedRouteDuration] = []
+        for code in sorted(required_iata_codes):
+            outbound = _derive_direction(directions[(code, "OUTBOUND")])
+            inbound = _derive_direction(directions[(code, "INBOUND")])
+            routes.append(
+                DerivedRouteDuration(
+                    destination_iata=code,
+                    outbound_minutes=outbound.minutes,
+                    inbound_minutes=inbound.minutes,
+                    outbound=outbound.counts,
+                    inbound=inbound.counts,
+                )
+            )
+    except _ReferenceProblem as error:
+        return DurationReferenceFailure(
+            error.failure_code,
+            error.observed_schema,
+        )
+
+    return DurationReferenceSuccess(
+        routes=tuple(routes),
+        source_row_count=source_row_count,
+        reference_date=reference_date,
+        reference_locator=DURATION_SOURCE_LOCATOR,
+        body_sha256=hashlib.sha256(archive).hexdigest(),
+        source_byte_count=len(archive),
+    )
diff --git a/travel_windows/tests/test_duration_reference.py b/travel_windows/tests/test_duration_reference.py
new file mode 100644
index 0000000..5b8a165
--- /dev/null
+++ b/travel_windows/tests/test_duration_reference.py
@@ -0,0 +1,274 @@
+import csv
+import hashlib
+import io
+import unittest
+import zipfile
+from datetime import date, timedelta
+
+from travel_windows.contracts import (
+    DURATION_DERIVATION_FINGERPRINT_SHA256,
+    DURATION_REFERENCE_SCHEMA_FINGERPRINT_SHA256,
+    DURATION_SOURCE_LOCATOR,
+)
+from travel_windows.duration_reference import (
+    OFFICIAL_DURATION_REFERENCE_HEADER,
+    DurationReferenceFailure,
+    DurationReferenceSuccess,
+    derive_route_durations,
+)
+
+
+def _row(
+    *,
+    direction="OUTBOUND",
+    day=0,
+    duration="120",
+    registration=None,
+    state="SCHEDULED",
+    active="Y",
+):
+    values = [""] * len(OFFICIAL_DURATION_REFERENCE_HEADER)
+    flight_date = date(2024, 1, 1) + timedelta(days=day)
+    values[0] = f"KE{700 + day}"
+    values[1] = flight_date.isoformat()
+    values[2] = str(day + 1)
+    values[3] = "ICN" if direction == "OUTBOUND" else "NRT"
+    values[4] = "NRT" if direction == "OUTBOUND" else "ICN"
+    values[5] = state
+    values[8] = registration or f"HL{8000 + day}"
+    values[11] = flight_date.isoformat()
+    values[14] = flight_date.isoformat()
+    values[17] = duration
+    values[47] = active
+    return values
+
+
+def _csv_payload(rows, *, header=OFFICIAL_DURATION_REFERENCE_HEADER):
+    text = io.StringIO(newline="")
+    writer = csv.writer(text, lineterminator="\r\n")
+    writer.writerow(header)
+    writer.writerows(rows)
+    return text.getvalue().encode("cp949")
+
+
+def _archive(
+    rows,
+    *,
+    header=OFFICIAL_DURATION_REFERENCE_HEADER,
+    outer_nested_name="05_항공 스케줄.zip",
+):
+    partitions = (rows[0::3], rows[1::3], rows[2::3])
+    nested_buffer = io.BytesIO()
+    with zipfile.ZipFile(
+        nested_buffer, "w", compression=zipfile.ZIP_DEFLATED
+    ) as nested:
+        for ordinal, partition in enumerate(partitions, start=1):
+            nested.writestr(
+                f"05_항공 스케줄({ordinal}).csv",
+                _csv_payload(partition, header=header),
+            )
+
+    outer_buffer = io.BytesIO()
+    with zipfile.ZipFile(
+        outer_buffer, "w", compression=zipfile.ZIP_DEFLATED
+    ) as outer:
+        outer.writestr(
+            "05_항공 스케줄_코드북.csv",
+            (
+                "\ufeff항목명,코드,내용,참조항목명\r\n"
+                "비행시간,,분 또는 시:분,\r\n"
+            ).encode("utf-8"),
+        )
+        outer.writestr(outer_nested_name, nested_buffer.getvalue())
+    return outer_buffer.getvalue()
+
+
+def _corrupt_first_member(payload):
+    with zipfile.ZipFile(io.BytesIO(payload), "r") as archive:
+        member = archive.infolist()[0]
+        header_offset = member.header_offset
+        name_length = int.from_bytes(payload[header_offset + 26 : header_offset + 28], "little")
+        extra_length = int.from_bytes(payload[header_offset + 28 : header_offset + 30], "little")
+        data_offset = header_offset + 30 + name_length + extra_length
+        corruption_offset = data_offset + member.compress_size // 2
+    corrupted = bytearray(payload)
+    corrupted[corruption_offset] ^= 0xFF
+    return bytes(corrupted)
+
+
+class DurationReferenceDerivationTests(unittest.TestCase):
+    def test_derives_bidirectional_upper_medians_and_audit_counts(self):
+        self.assertEqual(len(OFFICIAL_DURATION_REFERENCE_HEADER), 48)
+        self.assertEqual(
+            hashlib.sha256(
+                "\x1f".join(OFFICIAL_DURATION_REFERENCE_HEADER).encode("utf-8")
+            ).hexdigest(),
+            "2f613b3c7d278d4185d4af3fecc3f65b95f7950a52b78f4d464d639bad618c5d",
+        )
+        rows = [
+            *(
+                _row(direction="OUTBOUND", day=day, duration=str(minutes))
+                for day, minutes in enumerate((115, 120, 125, 120, 121))
+            ),
+            *(
+                _row(direction="INBOUND", day=day, duration=str(minutes))
+                for day, minutes in enumerate((145, 150, 155, 150, 151))
+            ),
+        ]
+        payload = _archive(rows)
+
+        result = derive_route_durations(
+            payload,
+            reference_date=date(2024, 12, 12),
+            required_iata_codes=frozenset({"NRT"}),
+        )
+
+        self.assertIsInstance(result, DurationReferenceSuccess)
+        self.assertEqual(result.source_row_count, 10)
+        self.assertEqual(result.reference_date, date(2024, 12, 12))
+        self.assertEqual(result.reference_locator, DURATION_SOURCE_LOCATOR)
+        self.assertEqual(
+            result.observed_schema_fingerprint_sha256,
+            DURATION_REFERENCE_SCHEMA_FINGERPRINT_SHA256,
+        )
+        self.assertEqual(
+            result.derivation_fingerprint_sha256,
+            DURATION_DERIVATION_FINGERPRINT_SHA256,
+        )
+        self.assertEqual(len(result.routes), 1)
+        route = result.routes[0]
+        self.assertEqual(
+            (route.destination_iata, route.outbound_minutes, route.inbound_minutes),
+            ("NRT", 120, 150),
+        )
+        self.assertEqual(
+            (
+                route.outbound.source_row_count,
+                route.outbound.occurrence_count,
+                route.outbound.retained_count,
+                route.outbound.outlier_count,
+            ),
+            (5, 5, 5, 0),
+        )
+        self.assertEqual(
+            (
+                route.inbound.source_row_count,
+                route.inbound.occurrence_count,
+                route.inbound.retained_count,
+                route.inbound.outlier_count,
+            ),
+            (5, 5, 5, 0),
+        )
+
+    def test_normalizes_numeric_and_h_mm_and_excludes_invalid_or_cancelled_rows(self):
+        rows = []
+        for day in range(5):
+            rows.extend(
+                (
+                    _row(
+                        direction="OUTBOUND",
+                        day=day,
+                        duration="119",
+                        registration=f"HL{8100 + day}",
+                    ),
+                    _row(
+                        direction="OUTBOUND",
+                        day=day,
+                        duration="2:00",
+                        registration=f"HL{8100 + day}",
+                    ),
+                    _row(direction="INBOUND", day=day, duration="2:30"),
+                )
+            )
+        rows.extend(
+            _row(direction="OUTBOUND", day=20 + offset, duration=value)
+            for offset, value in enumerate(("0", "29", "721", "bad", "1:60"))
+        )
+        rows.append(
+            _row(
+                direction="OUTBOUND",
+                day=30,
+                duration="999",
+                state="CANCELLED_BY_OPERATOR",
+            )
+        )
+
+        result = derive_route_durations(
+            _archive(rows),
+            reference_date=date(2024, 12, 12),
+            required_iata_codes=frozenset({"NRT"}),
+        )
+
+        self.assertIsInstance(result, DurationReferenceSuccess)
+        route = result.routes[0]
+        self.assertEqual((route.outbound_minutes, route.inbound_minutes), (120, 150))
+        self.assertEqual(
+            (
+                route.outbound.source_row_count,
+                route.outbound.invalid_row_count,
+                route.outbound.cancelled_row_count,
+                route.outbound.occurrence_count,
+                route.outbound.retained_count,
+            ),
+            (16, 5, 1, 5, 5),
+        )
+
+    def test_rejects_schema_drift_and_unsafe_zip_members(self):
+        rows = [
+            *(_row(direction="OUTBOUND", day=day) for day in range(5)),
+            *(_row(direction="INBOUND", day=day) for day in range(5)),
+        ]
+        changed_header = list(OFFICIAL_DURATION_REFERENCE_HEADER)
+        changed_header[17] = "예상비행시간"
+
+        cases = (
+            (_archive(rows, header=tuple(changed_header)), "SCHEMA_MISMATCH"),
+            (_archive(rows, outer_nested_name="../nested.zip"), "INVALID_VALUE"),
+            (_corrupt_first_member(_archive(rows)), "SYNTAX_ERROR"),
+        )
+        for payload, expected_code in cases:
+            with self.subTest(expected_code=expected_code):
+                result = derive_route_durations(
+                    payload,
+                    reference_date=date(2024, 12, 12),
+                    required_iata_codes=frozenset({"NRT"}),
+                )
+                self.assertIsInstance(result, DurationReferenceFailure)
+                self.assertEqual(result.failure_code, expected_code)
+                self.assertFalse(hasattr(result, "rows"))
+
+    def test_rejects_insufficient_and_unstable_direction_samples(self):
+        inbound = [_row(direction="INBOUND", day=day) for day in range(5)]
+        cases = (
+            (
+                [
+                    *(_row(direction="OUTBOUND", day=day) for day in range(4)),
+                    *inbound,
+                ],
+                "REQUIRED_VALUE_MISSING",
+            ),
+            (
+                [
+                    *(
+                        _row(direction="OUTBOUND", day=day, duration=str(minutes))
+                        for day, minutes in enumerate((100, 100, 100, 300, 300))
+                    ),
+                    *inbound,
+                ],
+                "CONFLICTING_VALUE",
+            ),
+        )
+
+        for rows, expected_code in cases:
+            with self.subTest(expected_code=expected_code):
+                result = derive_route_durations(
+                    _archive(rows),
+                    reference_date=date(2024, 12, 12),
+                    required_iata_codes=frozenset({"NRT"}),
+                )
+                self.assertIsInstance(result, DurationReferenceFailure)
+                self.assertEqual(result.failure_code, expected_code)
+
+
+if __name__ == "__main__":
+    unittest.main()


