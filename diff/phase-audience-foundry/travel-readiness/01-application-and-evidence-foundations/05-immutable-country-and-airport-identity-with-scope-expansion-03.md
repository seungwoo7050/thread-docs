## `fix(flights): align destination source identities`

diff --git a/docs/TRAVEL-OPPORTUNITY-CONTRACT.md b/docs/TRAVEL-OPPORTUNITY-CONTRACT.md
index ee433bb..31f386b 100644
--- a/docs/TRAVEL-OPPORTUNITY-CONTRACT.md
+++ b/docs/TRAVEL-OPPORTUNITY-CONTRACT.md
@@ -61,8 +61,12 @@ public result context는 `search_window`, `flight_state`, `opportunities`를 제
   공항만 typed row로 투영하며, 실제 시즌에 포함된 각 공항은 출발·도착 방향이 모두
   있어야 합니다.
 - `StatusOfSrvDestinations/getServiceDestinationInfo` 취항도시 전체 응답에서 curated
-  13개 IATA와 국문 국가·도시 명칭이 정확한 subset인지 검증합니다. extra 도시는
-  허용하지만 중복 IATA나 curated identity 불일치는 게시를 닫습니다.
+  13개 IATA와 국문 목적지 표기를 검증합니다. `airportName`은 curated 도시명 또는
+  `도시명/공항명`이어야 하고 `/` 뒤 공항명은 비어 있을 수 없습니다. 홍콩·마카오는
+  provider의 중국 그룹 표기와 제품 목적지명을 모두 허용하며, 그 밖의 국가는 curated
+  국문 국가명과 정확히 같아야 합니다. 공개 국가·도시명은 provider 표기를 복사하지 않고
+  검수된 curated mapping을 사용합니다. extra 도시는 허용하지만 중복 IATA나 이 identity
+  경계 불일치는 게시를 닫습니다.
 - `PaxFltSched/getPaxFltSchedArrivals` complete pages를 보조 증거로 수집하고, curated
   inbound 노선의 시즌·요일·ICN 도착시각이 관광플랫폼 운항표와 일치하지 않으면
   게시하지 않습니다.
diff --git a/sources/management/commands/register_approved_sources.py b/sources/management/commands/register_approved_sources.py
index c8be003..c1cb614 100644
--- a/sources/management/commands/register_approved_sources.py
+++ b/sources/management/commands/register_approved_sources.py
@@ -14,6 +14,9 @@ from travel_windows.contracts import (
     CITY_SOURCE_OWNER,
     CITY_SOURCE_REVISION,
     CITY_V1_CONTRACT_FINGERPRINT_SHA256,
+    CITY_V2_CONTRACT_FINGERPRINT_SHA256,
+    CITY_V2_FIELD_SCOPE,
+    CITY_V2_SOURCE_REVISION,
     DATA_GO_SECRET_REFERENCE,
     DURATION_ATTRIBUTION,
     DURATION_CONTRACT_FINGERPRINT_SHA256,
@@ -113,6 +116,26 @@ def _with_prior_aviation_contract(
     return replace(spec, prior_contracts=(prior,))
 
 
+def _with_city_contract_history(
+    spec: ApprovedSourceSpec,
+) -> ApprovedSourceSpec:
+    prior_v2 = replace(
+        spec,
+        revision=CITY_V2_SOURCE_REVISION,
+        field_scope_code=CITY_V2_FIELD_SCOPE,
+        contract_fingerprint_sha256=CITY_V2_CONTRACT_FINGERPRINT_SHA256,
+        decision_basis_code="LIVE_AVIATION_ENVELOPE_20260831",
+        prior_contracts=(),
+    )
+    prior_v1 = replace(
+        prior_v2,
+        revision=AVIATION_PRIOR_SOURCE_REVISION,
+        contract_fingerprint_sha256=CITY_V1_CONTRACT_FINGERPRINT_SHA256,
+        decision_basis_code="TRAVEL_OPPORTUNITY_CONTRACT_20260831",
+    )
+    return replace(spec, prior_contracts=(prior_v1, prior_v2))
+
+
 APPROVED_SOURCE_SPECS = (
     ApprovedSourceSpec(
         code="MOFA_ENTRY_CSV",
@@ -202,7 +225,7 @@ APPROVED_SOURCE_SPECS = (
         ),
         fingerprint=SCHEDULE_V1_CONTRACT_FINGERPRINT_SHA256,
     ),
-    _with_prior_aviation_contract(
+    _with_city_contract_history(
         ApprovedSourceSpec(
             code=CITY_SOURCE_CODE,
             revision=CITY_SOURCE_REVISION,
@@ -213,10 +236,9 @@ APPROVED_SOURCE_SPECS = (
             access_mode=SourceRightsDecision.AccessMode.CREDENTIAL_REFERENCE,
             field_scope_code=CITY_FIELD_SCOPE,
             contract_fingerprint_sha256=CITY_CONTRACT_FINGERPRINT_SHA256,
-            decision_basis_code="LIVE_AVIATION_ENVELOPE_20260831",
+            decision_basis_code="LIVE_DESTINATION_IDENTITY_20260831",
             attribution_text=CITY_ATTRIBUTION,
-        ),
-        fingerprint=CITY_V1_CONTRACT_FINGERPRINT_SHA256,
+        )
     ),
     _with_prior_aviation_contract(
         ApprovedSourceSpec(
diff --git a/sources/migrations/0015_destination_identity_contract.py b/sources/migrations/0015_destination_identity_contract.py
new file mode 100644
index 0000000..cc38335
--- /dev/null
+++ b/sources/migrations/0015_destination_identity_contract.py
@@ -0,0 +1,172 @@
+from django.db import migrations, models
+from django.db.models import Q
+
+
+FUNCTION_NAME = "sources_guard_parse_run_change"
+CITY_SOURCE_CODE = "ICN_DESTINATION_CITY_API_15095067"
+OLD_CITY_CLAUSE = """            OR (source_module = 'AVIATION'
+                AND NEW.parser_name = 'ICN_DESTINATION_CITY_JSON'
+                AND NEW.parser_version = 'V2'
+                AND NEW.parser_contract_fingerprint_sha256 =
+                    '9e274716712f0ca14374c99d9f9d36a3569ff6f89b74155e3dc62ec65b1c1b1e'
+                AND NEW.expected_schema_fingerprint_sha256 =
+                    '06475295174c264253686a12d4a15cb905c88ba730eaf6e8ef58652170522703')"""
+NEW_CITY_CLAUSE = """            OR (source_module = 'AVIATION'
+                AND NEW.parser_name = 'ICN_DESTINATION_CITY_JSON'
+                AND NEW.parser_version = 'V3'
+                AND NEW.parser_contract_fingerprint_sha256 =
+                    '315311c7edade79a1ef94fb1f76943041932f611828b1e0eff852a52e5262fb9'
+                AND NEW.expected_schema_fingerprint_sha256 =
+                    '06475295174c264253686a12d4a15cb905c88ba730eaf6e8ef58652170522703')"""
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
+def _replace_clause(schema_editor, old, new):
+    body = _function_body(schema_editor)
+    if body.count(old) != 1:
+        raise RuntimeError(f"unexpected {FUNCTION_NAME} trigger definition")
+    body = body.replace(old, new)
+    quoted_name = schema_editor.quote_name(FUNCTION_NAME)
+    with schema_editor.connection.cursor() as cursor:
+        cursor.execute(
+            f"""
+            CREATE OR REPLACE FUNCTION {quoted_name}() RETURNS trigger
+            LANGUAGE plpgsql AS $destination_identity_contract$
+            {body}
+            $destination_identity_contract$
+            """
+        )
+
+
+def allow_destination_identity_contract(apps, schema_editor):
+    _replace_clause(
+        schema_editor,
+        OLD_CITY_CLAUSE,
+        OLD_CITY_CLAUSE + "\n" + NEW_CITY_CLAUSE,
+    )
+
+
+def restore_destination_city_v2(apps, schema_editor):
+    FetchAttempt = apps.get_model("sources", "FetchAttempt")
+    ParseRun = apps.get_model("sources", "ParseRun")
+    SourceConfiguration = apps.get_model("sources", "SourceConfiguration")
+    SourceRightsDecision = apps.get_model("sources", "SourceRightsDecision")
+    if (
+        ParseRun.objects.filter(
+            parser_name="ICN_DESTINATION_CITY_JSON",
+            parser_version="V3",
+        ).exists()
+        or FetchAttempt.objects.filter(
+            source__code=CITY_SOURCE_CODE,
+            source_revision="travel-v3",
+        ).exists()
+        or SourceRightsDecision.objects.filter(
+            source__code=CITY_SOURCE_CODE,
+            source_revision="travel-v3",
+        ).exists()
+        or SourceConfiguration.objects.filter(
+            code=CITY_SOURCE_CODE,
+            revision="travel-v3",
+        ).exists()
+    ):
+        raise RuntimeError(
+            "destination identity contract rollback requires no V3 source evidence"
+        )
+    _replace_clause(
+        schema_editor,
+        OLD_CITY_CLAUSE + "\n" + NEW_CITY_CLAUSE,
+        OLD_CITY_CLAUSE,
+    )
+
+
+class Migration(migrations.Migration):
+    dependencies = [("sources", "0014_live_aviation_envelope_contracts")]
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
+                        code=CITY_SOURCE_CODE,
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
+                choices=[
+                    ("V1", "Version 1"),
+                    ("V2", "Version 2"),
+                    ("V3", "Version 3"),
+                ],
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
+                condition=Q(parser_version__in=("V1", "V2", "V3")),
+                name="parse_version_allowlist",
+            ),
+        ),
+        migrations.RunPython(
+            allow_destination_identity_contract,
+            restore_destination_city_v2,
+        ),
+    ]
diff --git a/sources/models.py b/sources/models.py
index ec3035c..1e9e131 100644
--- a/sources/models.py
+++ b/sources/models.py
@@ -89,12 +89,16 @@ class SourceConfiguration(models.Model):
                     | Q(
                         code__in=[
                             "ICN_SCHEDULE_API",
-                            "ICN_DESTINATION_CITY_API_15095067",
                             "ICN_LEGACY_ARRIVALS_API",
                         ],
                         revision__in=["travel-v1", "travel-v2"],
                         secret_reference_name="DATA_GO_KR_SERVICE_KEY",
                     )
+                    | Q(
+                        code="ICN_DESTINATION_CITY_API_15095067",
+                        revision__in=["travel-v1", "travel-v2", "travel-v3"],
+                        secret_reference_name="DATA_GO_KR_SERVICE_KEY",
+                    )
                     | Q(
                         code="ICN_DESTINATION_CITY_API",
                         revision="travel-v1",
@@ -308,6 +312,7 @@ class ParseRun(models.Model):
     class ParserVersion(models.TextChoices):
         V1 = "V1", "Version 1"
         V2 = "V2", "Version 2"
+        V3 = "V3", "Version 3"
 
     class Outcome(models.TextChoices):
         STARTED = "STARTED", "Started"
@@ -364,7 +369,7 @@ class ParseRun(models.Model):
                 name="parse_name_allowlist",
             ),
             models.CheckConstraint(
-                condition=Q(parser_version__in=["V1", "V2"]),
+                condition=Q(parser_version__in=["V1", "V2", "V3"]),
                 name="parse_version_allowlist",
             ),
             models.CheckConstraint(condition=Q(parser_contract_fingerprint_sha256__regex=r"^[0-9a-f]{64}$"), name="parse_contract_fingerprint_format"),
diff --git a/sources/tests/test_destination_identity_migration.py b/sources/tests/test_destination_identity_migration.py
new file mode 100644
index 0000000..be1798f
--- /dev/null
+++ b/sources/tests/test_destination_identity_migration.py
@@ -0,0 +1,61 @@
+import importlib
+
+from django.db import connection
+from django.db.migrations.executor import MigrationExecutor
+from django.db.migrations.recorder import MigrationRecorder
+from django.test import TransactionTestCase
+
+from operations.tests.migration_helpers import restore_canonical_migration_graph
+from sources.models import SourceConfiguration
+from travel_windows.contracts import CITY_SOURCE_CODE, CITY_SOURCE_LOCATOR
+
+
+migration = importlib.import_module(
+    "sources.migrations.0015_destination_identity_contract"
+)
+
+
+class DestinationIdentityMigrationTests(TransactionTestCase):
+    latest = [("sources", "0015_destination_identity_contract")]
+    previous = [("sources", "0014_live_aviation_envelope_contracts")]
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
+    def parse_guard_body(self):
+        with connection.cursor() as cursor:
+            cursor.execute(
+                "SELECT prosrc FROM pg_proc "
+                "WHERE oid = 'sources_guard_parse_run_change()'::regprocedure"
+            )
+            return cursor.fetchone()[0]
+
+    def test_v3_evidence_blocks_reverse_without_weakening_the_guard(self):
+        SourceConfiguration.objects.create(
+            code=CITY_SOURCE_CODE,
+            revision="travel-v3",
+            module=SourceConfiguration.Module.AVIATION,
+            owner="Synthetic owner",
+            official_locator=CITY_SOURCE_LOCATOR,
+        )
+        before = self.parse_guard_body()
+        self.assertIn(migration.OLD_CITY_CLAUSE, before)
+        self.assertIn(migration.NEW_CITY_CLAUSE, before)
+
+        with self.assertRaisesMessage(
+            RuntimeError,
+            "destination identity contract rollback requires no V3 source evidence",
+        ):
+            MigrationExecutor(connection).migrate(self.previous)
+
+        self.assertEqual(self.parse_guard_body(), before)
+        self.assertIn(
+            ("sources", "0015_destination_identity_contract"),
+            MigrationRecorder(connection).applied_migrations(),
+        )
diff --git a/sources/tests/test_live_aviation_envelope_migration.py b/sources/tests/test_live_aviation_envelope_migration.py
index 6ede241..b2d9b49 100644
--- a/sources/tests/test_live_aviation_envelope_migration.py
+++ b/sources/tests/test_live_aviation_envelope_migration.py
@@ -20,6 +20,7 @@ class LiveAviationEnvelopeMigrationTests(TransactionTestCase):
 
     def setUp(self):
         restore_canonical_migration_graph(connection)
+        MigrationExecutor(connection).migrate(self.latest)
 
     def _fixture_teardown(self):
         try:
diff --git a/sources/tests/test_parse_run.py b/sources/tests/test_parse_run.py
index dda4125..ff0ded6 100644
--- a/sources/tests/test_parse_run.py
+++ b/sources/tests/test_parse_run.py
@@ -441,7 +441,7 @@ class ParseRunTests(ParseRunFixtureMixin, TestCase):
         self.assert_integrity_error(
             lambda: ParseRun.objects.create(**self.run_values(self.artifact))
         )
-        self.assertEqual(ParseRun.ParserVersion.values, ["V1", "V2"])
+        self.assertEqual(ParseRun.ParserVersion.values, ["V1", "V2", "V3"])
 
         identity_constraint = next(
             constraint
diff --git a/sources/tests/test_register_approved_sources.py b/sources/tests/test_register_approved_sources.py
index 8784e50..00f9336 100644
--- a/sources/tests/test_register_approved_sources.py
+++ b/sources/tests/test_register_approved_sources.py
@@ -431,6 +431,50 @@ class ApprovedSourceRegistrationCommandTests(TransactionTestCase):
         )
         self.assertIn("result=ALREADY_ACTIVE", output)
 
+    def test_city_v2_contract_upgrades_with_complete_history(self):
+        current = next(
+            spec
+            for spec in registration.APPROVED_SOURCE_SPECS
+            if spec.code == "ICN_DESTINATION_CITY_API_15095067"
+        )
+        prior_v1, prior_v2 = current.prior_contracts
+        source = self.make_exact_source(prior_v1)
+        self.make_exact_approval(source, prior_v1)
+        self.activate_source(source)
+
+        with mock.patch.object(
+            registration,
+            "APPROVED_SOURCE_SPECS",
+            (replace(prior_v2, prior_contracts=(prior_v1,)),),
+        ):
+            self.call_registration("--apply")
+
+        with mock.patch.object(
+            registration,
+            "APPROVED_SOURCE_SPECS",
+            (current,),
+        ):
+            output = self.call_registration("--apply")
+
+        source.refresh_from_db()
+        self.assertEqual(source.revision, current.revision)
+        self.assertIn("result=UPGRADED_AND_ACTIVATED", output)
+        self.assertEqual(
+            set(
+                source.rights_decisions.values_list(
+                    "source_revision",
+                    "contract_fingerprint_sha256",
+                )
+            ),
+            {
+                (
+                    contract.revision,
+                    contract.contract_fingerprint_sha256,
+                )
+                for contract in (*current.prior_contracts, current)
+            },
+        )
+
     def test_upgrade_rejects_mismatched_prior_fingerprint(self):
         prior, current = self.make_upgrade_contracts("MISMATCH")
         source = self.make_exact_source(prior)
diff --git a/travel_windows/contracts.py b/travel_windows/contracts.py
index 0bb9e82..7dee196 100644
--- a/travel_windows/contracts.py
+++ b/travel_windows/contracts.py
@@ -24,14 +24,16 @@ SCHEDULE_ATTRIBUTION = "인천국제공항공사|공공데이터포털"
 SCHEDULE_FIELD_SCOPE = "ICN_SCHEDULE_V1"
 
 CITY_SOURCE_CODE = "ICN_DESTINATION_CITY_API_15095067"
-CITY_SOURCE_REVISION = "travel-v2"
+CITY_V2_SOURCE_REVISION = "travel-v2"
+CITY_SOURCE_REVISION = "travel-v3"
 CITY_SOURCE_OWNER = "인천국제공항공사"
 CITY_SOURCE_LOCATOR = (
     "https://apis.data.go.kr/B551177/StatusOfSrvDestinations/"
     "getServiceDestinationInfo"
 )
 CITY_ATTRIBUTION = "인천국제공항공사|공공데이터포털"
-CITY_FIELD_SCOPE = "ICN_DESTINATION_CITY_V1"
+CITY_V2_FIELD_SCOPE = "ICN_DESTINATION_CITY_V1"
+CITY_FIELD_SCOPE = "ICN_DESTINATION_IDENTITY_V2"
 
 LEGACY_SCHEDULE_SOURCE_CODE = "ICN_LEGACY_ARRIVALS_API"
 LEGACY_SCHEDULE_SOURCE_REVISION = "travel-v2"
@@ -87,9 +89,11 @@ SCHEDULE_SCHEMA_FINGERPRINT_SHA256 = _sha256(SCHEDULE_SCHEMA_TEXT)
 DURATION_CONTRACT_FINGERPRINT_SHA256 = _sha256(DURATION_CONTRACT_TEXT)
 DURATION_SCHEMA_FINGERPRINT_SHA256 = _sha256(DURATION_SCHEMA_TEXT)
 CITY_CONTRACT_TEXT = (
-    "ICN destination city v2|response(header(resultCode,resultMsg),body("
+    "ICN destination city v3|response(header(resultCode,resultMsg),body("
     "items(rows-or-item-wrapper(countryName,airportName,airportCode))))|exact-curated-13|"
-    "Korean-country-city-names"
+    "airport-label-is-city-or-city-slash-nonempty-airport|"
+    "HK-MO-country-is-product-name-or-China|other-country-name-exact|"
+    "curated-IATA-country-city-normalization"
 )
 CITY_SCHEMA_TEXT = (
     "data-go-json-v2(response.header+response.body.items-direct-list-or-item-wrapper+"
@@ -131,6 +135,12 @@ CITY_V1_CONTRACT_FINGERPRINT_SHA256 = (
 CITY_V1_SCHEMA_FINGERPRINT_SHA256 = (
     "907670feb0293b0a1a40e6d7065cae2c803e4ba458828b22631dbee002017c18"
 )
+CITY_V2_CONTRACT_FINGERPRINT_SHA256 = (
+    "9e274716712f0ca14374c99d9f9d36a3569ff6f89b74155e3dc62ec65b1c1b1e"
+)
+CITY_V2_SCHEMA_FINGERPRINT_SHA256 = (
+    "06475295174c264253686a12d4a15cb905c88ba730eaf6e8ef58652170522703"
+)
 LEGACY_SCHEDULE_V1_CONTRACT_FINGERPRINT_SHA256 = (
     "a88a91166bb7ea567f59637159f07f2285981f2ae0588e921ec2882e1c73bbfb"
 )
diff --git a/travel_windows/ingestion.py b/travel_windows/ingestion.py
index a73d5c8..b74cfd6 100644
--- a/travel_windows/ingestion.py
+++ b/travel_windows/ingestion.py
@@ -594,6 +594,7 @@ def _persist_parse_run(
     *,
     inputs: tuple[_CollectedInput, ...],
     parser_name: str,
+    parser_version: str,
     contract_fingerprint: str,
     schema_fingerprint: str,
     parsed: object,
@@ -634,7 +635,7 @@ def _persist_parse_run(
             .filter(
                 artifact=artifacts[inputs[0].artifact.pk],
                 parser_name=parser_name,
-                parser_version=ParseRun.ParserVersion.V2,
+                parser_version=parser_version,
                 input_identity_sha256=input_identity,
             )
             .first()
@@ -643,7 +644,7 @@ def _persist_parse_run(
             run = ParseRun.objects.create(
                 artifact=artifacts[inputs[0].artifact.pk],
                 parser_name=parser_name,
-                parser_version=ParseRun.ParserVersion.V2,
+                parser_version=parser_version,
                 parser_contract_fingerprint_sha256=contract_fingerprint,
                 expected_schema_fingerprint_sha256=schema_fingerprint,
                 input_identity_sha256=input_identity,
@@ -850,6 +851,7 @@ def ingest_flight_evidence_candidate(
         schedule_run = _persist_parse_run(
             inputs=schedule_inputs,
             parser_name=ParseRun.ParserName.ICN_FLIGHT_SCHEDULE_JSON,
+            parser_version=ParseRun.ParserVersion.V2,
             contract_fingerprint=SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
             schema_fingerprint=SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
             parsed=schedule,
@@ -858,6 +860,7 @@ def ingest_flight_evidence_candidate(
         city_reference_run = _persist_parse_run(
             inputs=(city_input,),
             parser_name=ParseRun.ParserName.ICN_DESTINATION_CITY_JSON,
+            parser_version=ParseRun.ParserVersion.V3,
             contract_fingerprint=CITY_CONTRACT_FINGERPRINT_SHA256,
             schema_fingerprint=CITY_SCHEMA_FINGERPRINT_SHA256,
             parsed=city_reference,
@@ -866,6 +869,7 @@ def ingest_flight_evidence_candidate(
         legacy_arrivals_run = _persist_parse_run(
             inputs=legacy_inputs,
             parser_name=ParseRun.ParserName.ICN_LEGACY_ARRIVALS_JSON,
+            parser_version=ParseRun.ParserVersion.V2,
             contract_fingerprint=(
                 LEGACY_SCHEDULE_CONTRACT_FINGERPRINT_SHA256
             ),
diff --git a/travel_windows/publication.py b/travel_windows/publication.py
index b591a0a..ec4c4c9 100644
--- a/travel_windows/publication.py
+++ b/travel_windows/publication.py
@@ -36,6 +36,9 @@ from .contracts import (
     CITY_SOURCE_REVISION,
     CITY_V1_CONTRACT_FINGERPRINT_SHA256,
     CITY_V1_SCHEMA_FINGERPRINT_SHA256,
+    CITY_V2_CONTRACT_FINGERPRINT_SHA256,
+    CITY_V2_SCHEMA_FINGERPRINT_SHA256,
+    CITY_V2_SOURCE_REVISION,
     DURATION_CONTRACT_FINGERPRINT_SHA256,
     DURATION_SCHEMA_FINGERPRINT_SHA256,
     DURATION_SOURCE_CODE,
@@ -82,6 +85,12 @@ from countries.models import SUPPORTED_COUNTRY_ROWS
 
 PUBLICATION_LOCK_NAMESPACE = 1_414_678_614
 PUBLICATION_LOCK_KEY = 20_260_831
+# 15095067 groups these airport destinations under China, while the product
+# keeps Hong Kong and Macao as independent destination scopes.
+_CITY_REFERENCE_COUNTRY_ALIASES = {
+    "HK": frozenset({"중국"}),
+    "MO": frozenset({"중국"}),
+}
 
 
 class FlightPublicationCode:
@@ -420,7 +429,7 @@ def _reference_evidence_is_exact(
         )
     }
     expected_cities = {
-        iata: (country_names[country_iso2], city_name)
+        iata: (country_iso2, country_names[country_iso2], city_name)
         for (
             _airport_id,
             iata,
@@ -435,11 +444,30 @@ def _reference_evidence_is_exact(
         row.airport_code: (row.country_name_ko, row.city_name_ko)
         for row in city_reference.destinations
     }
-    if any(
-        observed_cities.get(airport_code) != identity
-        for airport_code, identity in expected_cities.items()
-    ):
-        return False
+    for airport_code, (
+        country_iso2,
+        expected_country,
+        expected_city,
+    ) in expected_cities.items():
+        observed = observed_cities.get(airport_code)
+        if observed is None:
+            return False
+        observed_country, observed_airport_label = observed
+        allowed_countries = {
+            expected_country,
+            *_CITY_REFERENCE_COUNTRY_ALIASES.get(country_iso2, ()),
+        }
+        if observed_country not in allowed_countries:
+            return False
+        # The provider uses either a city label or ``city/airport`` here.
+        if observed_airport_label == expected_city:
+            continue
+        prefix = f"{expected_city}/"
+        if not (
+            observed_airport_label.startswith(prefix)
+            and observed_airport_label.removeprefix(prefix).strip()
+        ):
+            return False
     required_inbound = {
         (
             row.destination_iata,
@@ -531,7 +559,7 @@ def stage_flight_evidence(
                 source_code=CITY_SOURCE_CODE,
                 source_revision=CITY_SOURCE_REVISION,
                 parser_name=ParseRun.ParserName.ICN_DESTINATION_CITY_JSON,
-                parser_version=ParseRun.ParserVersion.V2,
+                parser_version=ParseRun.ParserVersion.V3,
                 contract_fingerprint=CITY_CONTRACT_FINGERPRINT_SHA256,
                 schema_fingerprint=CITY_SCHEMA_FINGERPRINT_SHA256,
             )
@@ -754,11 +782,17 @@ def _load_valid_candidate(
         allow_legacy_lineage
         and revision.parse_run.parser_version == ParseRun.ParserVersion.V1
     )
-    historical_city = (
+    historical_city_v1 = (
         allow_legacy_lineage
         and revision.city_reference_parse_run.parser_version
         == ParseRun.ParserVersion.V1
     )
+    historical_city_v2 = (
+        allow_legacy_lineage
+        and revision.city_reference_parse_run.parser_version
+        == ParseRun.ParserVersion.V2
+    )
+    historical_city = historical_city_v1 or historical_city_v2
     historical_legacy = (
         allow_legacy_lineage
         and revision.legacy_arrivals_parse_run.parser_version
@@ -797,24 +831,40 @@ def _load_valid_candidate(
         source_code=CITY_SOURCE_CODE,
         source_revision=(
             AVIATION_PRIOR_SOURCE_REVISION
-            if historical_city
-            else CITY_SOURCE_REVISION
+            if historical_city_v1
+            else (
+                CITY_V2_SOURCE_REVISION
+                if historical_city_v2
+                else CITY_SOURCE_REVISION
+            )
         ),
         parser_name=ParseRun.ParserName.ICN_DESTINATION_CITY_JSON,
         parser_version=(
             ParseRun.ParserVersion.V1
-            if historical_city
-            else ParseRun.ParserVersion.V2
+            if historical_city_v1
+            else (
+                ParseRun.ParserVersion.V2
+                if historical_city_v2
+                else ParseRun.ParserVersion.V3
+            )
         ),
         contract_fingerprint=(
             CITY_V1_CONTRACT_FINGERPRINT_SHA256
-            if historical_city
-            else CITY_CONTRACT_FINGERPRINT_SHA256
+            if historical_city_v1
+            else (
+                CITY_V2_CONTRACT_FINGERPRINT_SHA256
+                if historical_city_v2
+                else CITY_CONTRACT_FINGERPRINT_SHA256
+            )
         ),
         schema_fingerprint=(
             CITY_V1_SCHEMA_FINGERPRINT_SHA256
-            if historical_city
-            else CITY_SCHEMA_FINGERPRINT_SHA256
+            if historical_city_v1
+            else (
+                CITY_V2_SCHEMA_FINGERPRINT_SHA256
+                if historical_city_v2
+                else CITY_SCHEMA_FINGERPRINT_SHA256
+            )
         ),
         allow_legacy_unordered=allow_legacy_lineage,
         allow_historical_source_revision=historical_city,
diff --git a/travel_windows/tests/test_review_lifecycle_migration.py b/travel_windows/tests/test_review_lifecycle_migration.py
index 44de263..c211b60 100644
--- a/travel_windows/tests/test_review_lifecycle_migration.py
+++ b/travel_windows/tests/test_review_lifecycle_migration.py
@@ -61,7 +61,7 @@ class FlightReviewLifecycleMigrationTests(TransactionTestCase):
         executor.migrate(self.previous)
         self.old_apps = executor.loader.project_state(self.previous).apps
         historical_specs = tuple(
-            spec.prior_contracts[-1] if spec.prior_contracts else spec
+            spec.prior_contracts[0] if spec.prior_contracts else spec
             for spec in registration.APPROVED_SOURCE_SPECS
         )
         with mock.patch.object(
diff --git a/travel_windows/tests/test_source_publication.py b/travel_windows/tests/test_source_publication.py
index bcbf9a5..5c3b5dc 100644
--- a/travel_windows/tests/test_source_publication.py
+++ b/travel_windows/tests/test_source_publication.py
@@ -796,7 +796,11 @@ class FlightEvidencePublicationTests(TransactionTestCase):
         )
         self.assertEqual(
             [run.parser_version for run in parse_runs],
-            [ParseRun.ParserVersion.V2] * 3,
+            [
+                ParseRun.ParserVersion.V2,
+                ParseRun.ParserVersion.V3,
+                ParseRun.ParserVersion.V2,
+            ],
         )
         self.assertTrue(
             all(
@@ -1160,6 +1164,12 @@ class FlightEvidencePublicationTests(TransactionTestCase):
                 city_reference_payload(),
                 legacy_arrival_page([legacy_arrival_row(st="2016")]),
             ),
+            (
+                city_reference_payload(
+                    overrides={"NRT": {"countryName": "대한민국"}}
+                ),
+                legacy_arrival_page([legacy_arrival_row()]),
+            ),
         )
         for city_payload, legacy_page in cases:
             with self.subTest(city_sha=len(city_payload), legacy_sha=len(legacy_page)):
@@ -1182,6 +1192,43 @@ class FlightEvidencePublicationTests(TransactionTestCase):
             ).exists()
         )
 
+    def test_provider_airport_labels_map_to_curated_destinations(self):
+        outcome = collect_and_stage_fixture(
+            departure_pages=(official_page([official_row()]),),
+            arrival_pages=(
+                official_page(
+                    [
+                        official_row(
+                            flightId="KE704",
+                            masterFlightId="KE704",
+                            st="2015",
+                        )
+                    ]
+                ),
+            ),
+            city_payload=city_reference_payload(
+                overrides={
+                    "NRT": {"airportName": "도쿄/나리타"},
+                    "HKG": {"countryName": "중국"},
+                    "MFM": {"countryName": "중국"},
+                }
+            ),
+            legacy_arrival_pages=(
+                legacy_arrival_page([legacy_arrival_row()]),
+            ),
+            duration_csv=(
+                b"destination_iata,outbound_minutes,inbound_minutes,reference_date,"
+                b"reference_locator\r\nNRT,150,165,2026-08-01,"
+                + DURATION_SOURCE_LOCATOR.encode("ascii")
+                + b"\r\n"
+            ),
+            source_date=date(2026, 8, 31),
+            source_checked_at=timezone.now(),
+        )
+
+        self.assertEqual(outcome.code, FlightIngestionCode.STAGED)
+        self.assertEqual(FlightScheduleRevision.objects.count(), 1)
+
     def test_full_official_rows_project_to_curated_bidirectional_routes(self):
         departure = official_page(
             [


