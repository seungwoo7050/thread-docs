## `fix(ops): cover warning snapshot integrity`

diff --git a/operations/sensitive_absence.py b/operations/sensitive_absence.py
index ab422e0..e0a22a2 100644
--- a/operations/sensitive_absence.py
+++ b/operations/sensitive_absence.py
@@ -95,6 +95,8 @@ BACKUP_INTEGRITY_KEYS: Final = frozenset(
         "table.entry_requirements_entryfactrevision.sha256",
         "table.travel_warnings_travelwarningrevision.count",
         "table.travel_warnings_travelwarningrevision.sha256",
+        "table.travel_warnings_travelwarningfact.count",
+        "table.travel_warnings_travelwarningfact.sha256",
         "table.reviews_reviewdecision.count",
         "table.reviews_reviewdecision.sha256",
         "table.reviews_publicationrevision.count",
diff --git a/operations/tests/test_postgresql_backup_restore.py b/operations/tests/test_postgresql_backup_restore.py
index e863fb2..ee8bdef 100644
--- a/operations/tests/test_postgresql_backup_restore.py
+++ b/operations/tests/test_postgresql_backup_restore.py
@@ -1007,6 +1007,7 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
             "sources_sourceartifact",
             "sources_sourceconfiguration",
             "sources_sourcerightsdecision",
+            "travel_warnings_travelwarningfact",
             "travel_warnings_travelwarningrevision",
             "travel_windows_airport",
             "travel_windows_flightauditevent",
@@ -1144,6 +1145,7 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
             "sources_parseruninput",
             "entry_requirements_passportpolicy",
             "entry_requirements_entryfactrevision",
+            "travel_warnings_travelwarningfact",
             "travel_warnings_travelwarningrevision",
             "reviews_reviewdecision",
             "reviews_publicationrevision",
@@ -1213,15 +1215,15 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
             .split("'\n", 1)[0]
             .splitlines()
         )
-        self.assertEqual(len(functions), 44)
-        self.assertEqual(len({line.split("|", 1)[0] for line in functions}), 44)
+        self.assertEqual(len(functions), 49)
+        self.assertEqual(len({line.split("|", 1)[0] for line in functions}), 49)
         expected_function_names = {line.split("|", 1)[0] for line in functions}
         for line in functions:
             name, digest = line.split("|")
             self.assertRegex(name, r"^[a-z][a-z0-9_]+$")
             self.assertRegex(digest, r"^[0-9a-f]{64}$")
-        self.assertEqual(len(triggers), 58)
-        self.assertEqual(len({line.split("|", 1)[0] for line in triggers}), 58)
+        self.assertEqual(len(triggers), 63)
+        self.assertEqual(len({line.split("|", 1)[0] for line in triggers}), 63)
         for line in triggers:
             self.assertEqual(len(line.split("|")), 7)
 
@@ -1250,6 +1252,11 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
                     if isinstance(operation, RunSQL)
                     and isinstance(operation.sql, str)
                 )
+                warning_snapshot_sql = getattr(
+                    module, "WARNING_SNAPSHOT_PUBLICATION_SQL", None
+                )
+                if isinstance(warning_snapshot_sql, str):
+                    forward_sql.append(warning_snapshot_sql)
 
         function_pattern = re.compile(
             r"CREATE(?: OR REPLACE)? FUNCTION ([a-z0-9_]+)\(\)\s+"
@@ -1266,13 +1273,21 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
             r"(?P<timing>BEFORE|AFTER) (?P<events>(?:INSERT|UPDATE|DELETE)"
             r"(?: OR (?:INSERT|UPDATE|DELETE))*)\s+ON (?P<table>[a-z0-9_]+)\s+"
             r"(?:(?P<deferred>DEFERRABLE INITIALLY DEFERRED)\s+)?"
-            r"FOR EACH (?P<scope>ROW|STATEMENT) EXECUTE FUNCTION "
+            r"FOR EACH (?P<scope>ROW|STATEMENT)\s+"
+            r"(?:WHEN\s+\(.*?\)\s+)?EXECUTE FUNCTION "
             r"(?P<function>[a-z0-9_]+)\(\);",
             re.DOTALL,
         )
+        drop_trigger_pattern = re.compile(
+            r"DROP TRIGGER IF EXISTS (?P<name>[a-z0-9_]+)\s+ON\s+"
+            r"[a-z0-9_]+;",
+            re.DOTALL,
+        )
         event_bits = {"INSERT": 4, "DELETE": 8, "UPDATE": 16}
-        migration_trigger_lines = set()
+        migration_trigger_lines = {}
         for statement in forward_sql:
+            for match in drop_trigger_pattern.finditer(statement):
+                migration_trigger_lines.pop(match["name"], None)
             for match in trigger_pattern.finditer(statement):
                 trigger_type = 2 if match["timing"] == "BEFORE" else 0
                 if match["scope"] == "ROW":
@@ -1284,20 +1299,18 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
                 is_constraint = match["constraint"] is not None
                 is_deferred = match["deferred"] is not None
                 bool_text = lambda value: "true" if value else "false"
-                migration_trigger_lines.add(
-                    "|".join(
-                        (
-                            match["name"],
-                            match["table"],
-                            match["function"],
-                            str(trigger_type),
-                            bool_text(is_deferred),
-                            bool_text(is_deferred),
-                            bool_text(is_constraint),
-                        )
+                migration_trigger_lines[match["name"]] = "|".join(
+                    (
+                        match["name"],
+                        match["table"],
+                        match["function"],
+                        str(trigger_type),
+                        bool_text(is_deferred),
+                        bool_text(is_deferred),
+                        bool_text(is_constraint),
                     )
                 )
-        self.assertEqual(migration_trigger_lines, set(triggers))
+        self.assertEqual(set(migration_trigger_lines.values()), set(triggers))
 
     def test_runbook_requires_real_restore_and_ssr_rehearsal(self):
         runbook = self.runbook.read_text(encoding="utf-8")
diff --git a/operations/tests/test_sensitive_absence.py b/operations/tests/test_sensitive_absence.py
index 7b619a0..1433bb6 100644
--- a/operations/tests/test_sensitive_absence.py
+++ b/operations/tests/test_sensitive_absence.py
@@ -505,11 +505,13 @@ class RuntimeAndBackupBoundaryTests(unittest.TestCase):
             )
         )
         self.assertEqual(observed, set(BACKUP_INTEGRITY_KEYS))
-        self.assertEqual(len(BACKUP_INTEGRITY_KEYS), 96)
+        self.assertEqual(len(BACKUP_INTEGRITY_KEYS), 98)
         self.assertTrue(
             {
                 "table.sources_parseruninput.count",
                 "table.sources_parseruninput.sha256",
+                "table.travel_warnings_travelwarningfact.count",
+                "table.travel_warnings_travelwarningfact.sha256",
                 "table.travel_windows_revieweddurationreceipt.count",
                 "table.travel_windows_revieweddurationreceipt.sha256",
                 "table.travel_windows_routedurationderivation.count",
@@ -524,6 +526,14 @@ class RuntimeAndBackupBoundaryTests(unittest.TestCase):
             }
             <= observed
         )
+        outer_sort_keys = re.findall(
+            r"(?:SELECT|UNION ALL SELECT) '([0-9a-z]{3})',", source
+        )
+        self.assertEqual(len(outer_sort_keys), len(set(outer_sort_keys)))
+        self.assertIn(
+            "ORDER BY t.revision_id, t.source_position, t.id",
+            source,
+        )
         for sequence_name in (
             "sources_parseruninput_id_seq",
             "travel_windows_flightcandidateduration_id_seq",
diff --git a/scripts/postgresql-common b/scripts/postgresql-common
index 65b626a..8fccdf6 100644
--- a/scripts/postgresql-common
+++ b/scripts/postgresql-common
@@ -6,11 +6,11 @@
 POSTGRESQL_REQUIRED_VERSION="18.6"
 EXPECTED_PUBLIC_SCHEMA_COMMENT_SHA256="998e3ff6338d1b13197feaef3c7657578604caf2185d3fe9e5ac04ba171682c8"
 
-EXPECTED_SCHEMA_DIGESTS='schema.columns.sha256=2f2028991fa88816da01a45724254d18be52f296612372116e6ed2f9ab4da78f
-schema.constraints.sha256=cea6ee189d10c940d3513493a9b43d2e4a4f74424a1f46fb2db7e557adf0946e
-schema.indexes.sha256=4fe835fee9a73da2cd3109666dd6a064a5279f17398bcc13bf8fcc90e1a9ed03
-schema.trigger_functions.sha256=e8d28f3dc681bea77288ca8c9da6fd552a8ade69939e6873028ce7f8edab2d04
-schema.triggers.sha256=48c9c858620a341cd8f2513d9593f7a2146bf88886b1ba76049dac2140af953e'
+EXPECTED_SCHEMA_DIGESTS='schema.columns.sha256=80edad74ac1d0e59e23461d5e86de1b8f16d849eaa9c9eeaeaffc7bc6553f8ca
+schema.constraints.sha256=07219e82ff211ece27c4bd3349c4b856d4c86bf1c5066b878beb3620b5da1d4f
+schema.indexes.sha256=8d769d916c99ef5440fc15845cae2b0d891c9e722c11d21b5a7f30ec9cecb339
+schema.trigger_functions.sha256=746a94423501b1b11947b1e514067cad68c027c2e72afffd79320334e344b2e9
+schema.triggers.sha256=72d21b1958bf9bb1f949d57c424b2f8690e90f1596fe7d8e4def835e932fef97'
 
 EXPECTED_SEQUENCE_DEFINITION_SHA256="e9303b2ecb873b98391c36822e7ce66ee8c14062e5f1bf20c9e50f697f684152"
 
@@ -38,6 +38,7 @@ sources_parseruninput
 sources_sourceartifact
 sources_sourceconfiguration
 sources_sourcerightsdecision
+travel_warnings_travelwarningfact
 travel_warnings_travelwarningrevision
 travel_windows_airport
 travel_windows_flightauditevent
@@ -90,9 +91,10 @@ entry_requirements_reject_policy_mutation|bb0e61d8492ed4e5daafb44553f1cf738bc0a4
 reviews_enforce_deferred_closure|c6c350cbac6c1b599f04e5b41c2792c6be77b078726f1e3247ae765e7a2f51ea
 reviews_guard_audit_event|f9e4f24b27d120b7fadaf269a65d089105c11bd3d3d62cc5a9657f7e80d2d78c
 reviews_guard_entry_pointer|961fddc03229b4ce07a51e450066a0235cfc9d60a8999a73fe5e26e6355dbdd2
-reviews_guard_publication_revision|19b0e3840597173353b31fe94a28de16ce537bdea82110dc32a77c405cfe511c
+reviews_guard_publication_revision|dac36f47b31dd3284036903dc88056412c9c535c2b7c41f46655ecd6246534be
 reviews_guard_review_decision|0e3a7bff4a65f80f0064e3eaea56d2a816b5e716f2c807ff61402b1e5df16f9e
 reviews_guard_warning_pointer|3dfbf00082f64ace216b1c7a068c2f1c585b3b95cf5121b1a8aabe4a8096160e
+reviews_guard_warning_snapshot_publication|9ed8539f3adbb0c40dcdcfdf0fadac28ebafaebe5a4eadc76bb839698a709abe
 reviews_prelock_audit_insert|043a653839b82d059fdb0e971c5ec848e8eee38e9c745a23a39a0f994955eb01
 reviews_prelock_pointer_statement|a8a60c63736dd9d81a1e64742a7119bc7f1bc78210ad5813a590e59a0c52c259
 reviews_prelock_publication_insert|12b6c215f309fd01f75ad16e8b83f7ed6a641d342f1e40c12e881f9543f3312e
@@ -101,15 +103,19 @@ sources_apply_rights_rejection|e02368696d3a102bd0c7cbfd2f1078b58c4d63f96046c625e
 sources_guard_artifact_mutation|59039e6f2c37fa65deeea03b88d737aaa6c841fce8f00e173db1d3618736b485
 sources_guard_configuration_change|1eb4fbbf3cfe95d940364bc8f36d278b044739409100df1b5a9ad42049334fc7
 sources_guard_fetch_attempt_mutation|78195da0ef6c4dfc3012edfbc98dcb8323ee2d09fd1e52c0d1f999b5391c98c4
-sources_guard_parse_run_change|5dc5edd87da5cbc3174e29067a89db0d1a26c6bb60f67e4e3ed80400aa462cc6
+sources_guard_parse_run_change|d4fc55be1e15372b440cc8b462f365ba3cbfd4e4bc2ec61850869edca818bba7
 sources_guard_parse_run_input|fec41e4f9fa3f80853b429572303ad3eb234ac2d480711082bde3f8d6fe1f8d9
 sources_reject_rights_mutation|76be8ad85eab7939fe21596429e52e83ad1c7e1a1cac6954244a52606a730d46
 sources_reject_rights_revision_reuse|f96554369f58cd3017114acfd0e8772c9c7fd0252d8ebcbf493359af36310fad
 sources_validate_artifact_insert|e4e7711cc826af34a8a1638e21be3d3c01a6db25824230b4cf31d0238e0ea820
 sources_validate_fetch_attempt_insert|633cb98ad1ed6d016032902c3f0946a3ade515eb4b838b03e92ebc6b3fc58f08
 sources_validate_rights_insert|aef6e2ae5686f25b00295aa24c2cb8466cd882e7b41f4526b295e50a2a9ee182
-travel_warnings_guard_revision_change|280b30de8989e6b2e77e9cd9ba5b9cab81e3cc71d1b99aa64d3cacad17d8b8f2
+travel_warnings_guard_fact_change|b6f526f2698ed08035a1c305276a8851f8fae22aabd1284cdfc633afd26d157f
+travel_warnings_guard_revision_change|9c0f4011e5ec339aa9b9aac6c8ba95ad4cc8ce132a7398530bfd8a546db9ebca
+travel_warnings_guard_revision_immutable|829e88037987f15e60018111352f524a2966f3c5bd828a649d8f16f70e2067c9
 travel_warnings_validate_revision_source_rights|04bcea1c70b44524d34eae5b8ed897970fb25eb3d7ac7e595dd063c1c63ba3bd
+travel_warnings_validate_snapshot_closure|8e3e62c51759e44e8d130de84cdd08707a9dc958dccfcd7b0eb2ac304f0b9b6c
+travel_warnings_validate_snapshot_revision|f504dc14246a6c6ec3413349e0b0bdda77566319072fdb50aac1a88906bac798
 travel_windows_guard_candidate_duration_insert|31b7efff67ecc279e32809468568f6f85c281c5a5b9b633908765990b2275615
 travel_windows_guard_candidate_publication_duration|6b9f1f9d666efaac68bcbdb85f4d5d0da8944ef59b56458135c09376d41806a0
 travel_windows_guard_candidate_seal_insert|c55a8fa11c8a5bffbfc4a224cee8acd5c5b8ecb6cf1bdf75624dc215503cbe79
@@ -147,6 +153,7 @@ reviews_review_decision_guard|reviews_reviewdecision|reviews_guard_review_decisi
 reviews_review_deferred_closure|reviews_reviewdecision|reviews_enforce_deferred_closure|5|true|true|true
 reviews_warning_pointer_deferred_closure|reviews_publishedtravelwarning|reviews_enforce_deferred_closure|17|true|true|true
 reviews_warning_pointer_guard|reviews_publishedtravelwarning|reviews_guard_warning_pointer|31|false|false|false
+reviews_warning_snapshot_publication_guard|reviews_publicationrevision|reviews_guard_warning_snapshot_publication|7|false|false|false
 sources_artifact_insert_guard|sources_sourceartifact|sources_validate_artifact_insert|7|false|false|false
 sources_artifact_mutation_guard|sources_sourceartifact|sources_guard_artifact_mutation|27|false|false|false
 sources_configuration_change_guard|sources_sourceconfiguration|sources_guard_configuration_change|23|false|false|false
@@ -158,8 +165,12 @@ sources_parse_run_input_guard|sources_parseruninput|sources_guard_parse_run_inpu
 sources_rights_append_only_guard|sources_sourcerightsdecision|sources_reject_rights_mutation|27|false|false|false
 sources_rights_insert_guard|sources_sourcerightsdecision|sources_validate_rights_insert|7|false|false|false
 sources_rights_rejection_gate|sources_sourcerightsdecision|sources_apply_rights_rejection|5|false|false|false
-travel_warnings_revision_change_guard|travel_warnings_travelwarningrevision|travel_warnings_guard_revision_change|31|false|false|false
+travel_warnings_fact_change_guard|travel_warnings_travelwarningfact|travel_warnings_guard_fact_change|31|false|false|false
+travel_warnings_revision_change_guard|travel_warnings_travelwarningrevision|travel_warnings_guard_revision_change|7|false|false|false
+travel_warnings_revision_immutable_guard|travel_warnings_travelwarningrevision|travel_warnings_guard_revision_immutable|27|false|false|false
 travel_warnings_revision_source_rights_guard|travel_warnings_travelwarningrevision|travel_warnings_validate_revision_source_rights|7|false|false|false
+travel_warnings_snapshot_closure_guard|travel_warnings_travelwarningrevision|travel_warnings_validate_snapshot_closure|5|true|true|true
+travel_warnings_snapshot_revision_guard|travel_warnings_travelwarningrevision|travel_warnings_validate_snapshot_revision|7|false|false|false
 travel_windows_00_candidate_publication_duration_guard|travel_windows_flightpublicationduration|travel_windows_guard_candidate_publication_duration|7|false|false|false
 travel_windows_00_pointer_candidate_seal_guard|travel_windows_publishedflightschedule|travel_windows_guard_pointer_candidate_seal|19|false|false|false
 travel_windows_00_publication_candidate_seal_guard|travel_windows_flightpublication|travel_windows_guard_publication_candidate_seal|7|false|false|false
diff --git a/scripts/postgresql-integrity.sql b/scripts/postgresql-integrity.sql
index 0af9455..4605b89 100644
--- a/scripts/postgresql-integrity.sql
+++ b/scripts/postgresql-integrity.sql
@@ -225,6 +225,10 @@ WITH integrity_lines(sort_key, line) AS (
       FROM public.travel_warnings_travelwarningrevision
     UNION ALL SELECT '041', 'table.travel_warnings_travelwarningrevision.sha256=' || encode(sha256(convert_to(
       coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.travel_warnings_travelwarningrevision AS t), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '042', 'table.travel_warnings_travelwarningfact.count=' || count(*)::text
+      FROM public.travel_warnings_travelwarningfact
+    UNION ALL SELECT '043', 'table.travel_warnings_travelwarningfact.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.revision_id, t.source_position, t.id) FROM public.travel_warnings_travelwarningfact AS t), ''), 'UTF8')), 'hex')
 
     UNION ALL SELECT '050', 'table.reviews_reviewdecision.count=' || count(*)::text
       FROM public.reviews_reviewdecision


## `fix(ops): replay current warning snapshots`

diff --git a/operations/source_replay.py b/operations/source_replay.py
index 1026b97..23ebd0a 100644
--- a/operations/source_replay.py
+++ b/operations/source_replay.py
@@ -43,16 +43,16 @@ from sources.models import (
 from sources.management.commands.register_approved_sources import (
     APPROVED_SOURCE_SPECS,
 )
+from travel_warnings.contracts import WARNING_SNAPSHOT_SOURCE_CODE
 from travel_warnings.ingestion import (
-    WARNING_SOURCE_CODE,
     TravelWarningIngestionCode,
     TravelWarningIngestionOutcome,
-    ingest_travel_warning,
+    ingest_travel_warning_snapshot,
 )
 from travel_warnings.models import TravelWarningRevision
 from travel_warnings.parser import (
-    TravelWarningParseResult,
-    parse_travel_alarm_jp,
+    TravelWarningSnapshotParseResult,
+    parse_travel_alarm_snapshot,
 )
 
 
@@ -98,7 +98,7 @@ _APPROVED_SOURCE_CODES = frozenset(
 EntryIngestor = Callable[..., EntryIngestionOutcome]
 WarningIngestor = Callable[..., TravelWarningIngestionOutcome]
 EntryParser = Callable[[bytes], EntryParseResult]
-WarningParser = Callable[[bytes], TravelWarningParseResult]
+WarningParser = Callable[..., TravelWarningSnapshotParseResult]
 SecretLoader = Callable[[str], object]
 
 
@@ -263,7 +263,7 @@ def _source_baseline_is_exact() -> bool:
 
 
 def _evidence_is_exact() -> bool:
-    expected_sources = {ENTRY_SOURCE_CODE, WARNING_SOURCE_CODE}
+    expected_sources = {ENTRY_SOURCE_CODE, WARNING_SNAPSHOT_SOURCE_CODE}
     attempts = FetchAttempt.objects.all()
     artifacts = SourceArtifact.objects.all()
     parse_runs = ParseRun.objects.all()
@@ -298,9 +298,9 @@ def _evidence_is_exact() -> bool:
 def verify_live_parser_replay(
     *,
     entry_ingestor: EntryIngestor = ingest_entry_snapshot,
-    warning_ingestor: WarningIngestor = ingest_travel_warning,
+    warning_ingestor: WarningIngestor = ingest_travel_warning_snapshot,
     entry_parser: EntryParser = parse_entry_csv,
-    warning_parser: WarningParser = parse_travel_alarm_jp,
+    warning_parser: WarningParser = parse_travel_alarm_snapshot,
     secret_loader: SecretLoader = os.environ.get,
     database_guard: Callable[[], bool] = _disposable_database_is_approved,
     baseline_guard: Callable[[], bool] = _source_baseline_is_exact,
@@ -319,10 +319,14 @@ def verify_live_parser_replay(
             raise RuntimeError from None
         return first
 
-    def replay_warning(payload: bytes) -> TravelWarningParseResult:
-        first = warning_parser(payload)
+    def replay_warning(
+        payload: bytes,
+        *,
+        country_iso2: str,
+    ) -> TravelWarningSnapshotParseResult:
+        first = warning_parser(payload, country_iso2=country_iso2)
         parser_calls[SOURCE_WARNING] += 1
-        second = warning_parser(payload)
+        second = warning_parser(payload, country_iso2=country_iso2)
         parser_calls[SOURCE_WARNING] += 1
         if first != second:
             raise RuntimeError from None
diff --git a/operations/tests/test_live_parser_replay.py b/operations/tests/test_live_parser_replay.py
index 6ee8499..770a2f8 100644
--- a/operations/tests/test_live_parser_replay.py
+++ b/operations/tests/test_live_parser_replay.py
@@ -47,7 +47,8 @@ from travel_warnings.ingestion import (
 )
 from travel_warnings.parser import (
     ParsedTravelWarning,
-    TravelWarningParseSuccess,
+    ParsedTravelWarningSnapshot,
+    TravelWarningSnapshotParseSuccess,
 )
 
 
@@ -73,16 +74,23 @@ def _entry_success(period="90일"):
 
 
 def _warning_success(level="1"):
-    return TravelWarningParseSuccess(
-        warning=ParsedTravelWarning(
+    warning = ParsedTravelWarning(
+        country_iso2="JP",
+        country_name_ko="일본",
+        country_name_en="Japan",
+        source_alarm_level_code=level,
+        source_scope_type="일부",
+        source_scope_text="unit-test scope",
+        source_written_date=date(2026, 8, 1),
+        typed_fingerprint_sha256="c" * 64,
+    )
+    return TravelWarningSnapshotParseSuccess(
+        snapshot=ParsedTravelWarningSnapshot(
             country_iso2="JP",
             country_name_ko="일본",
             country_name_en="Japan",
-            source_alarm_level_code=level,
-            source_scope_type="일부",
-            source_scope_text="unit-test scope",
-            source_written_date=date(2026, 8, 1),
-            typed_fingerprint_sha256="c" * 64,
+            facts=(warning,),
+            typed_fingerprint_sha256="e" * 64,
         ),
         observed_schema_fingerprint_sha256="d" * 64,
     )
@@ -160,7 +168,10 @@ class LiveParserReplayTests(SimpleTestCase):
                 secret_loader(DATA_GO_SECRET_REFERENCE),
                 SECRET_MARKER,
             )
-            self.assertEqual(parser(RAW_MARKER), _warning_success())
+            self.assertEqual(
+                parser(RAW_MARKER, country_iso2="JP"),
+                _warning_success(),
+            )
             return TravelWarningIngestionOutcome(
                 TravelWarningIngestionCode.REVIEW_REQUIRED,
                 1,
@@ -170,7 +181,9 @@ class LiveParserReplayTests(SimpleTestCase):
             "entry_ingestor": entry_ingestor,
             "warning_ingestor": warning_ingestor,
             "entry_parser": lambda body: _entry_success(),
-            "warning_parser": lambda body: _warning_success(),
+            "warning_parser": (
+                lambda body, *, country_iso2: _warning_success()
+            ),
             "secret_loader": lambda name: SECRET_MARKER,
             "database_guard": lambda: True,
             "baseline_guard": lambda: True,
@@ -187,8 +200,9 @@ class LiveParserReplayTests(SimpleTestCase):
             parser_calls["entry"] += 1
             return _entry_success()
 
-        def warning_parser(body):
+        def warning_parser(body, *, country_iso2):
             self.assertEqual(body, RAW_MARKER)
+            self.assertEqual(country_iso2, "JP")
             parser_calls["warning"] += 1
             return _warning_success()
 
@@ -271,14 +285,14 @@ class LiveParserReplayTests(SimpleTestCase):
         self.assertNotIn("fetch_travel_alarm_jp", source)
         self.assertNotIn("sources.transport", source)
         self.assertIn("ingest_entry_snapshot", source)
-        self.assertIn("ingest_travel_warning", source)
+        self.assertIn("ingest_travel_warning_snapshot", source)
 
     def test_registered_source_allowlist_is_the_exact_replay_baseline(self):
         self.assertEqual(
             _APPROVED_SOURCE_CODES,
             frozenset(spec.code for spec in APPROVED_SOURCE_SPECS),
         )
-        self.assertEqual(len(_APPROVED_SOURCE_CODES), 8)
+        self.assertEqual(len(_APPROVED_SOURCE_CODES), 9)
 
     def test_storage_boundary_rejects_only_exact_transient_input_fields(self):
         fake_connection = MagicMock()
