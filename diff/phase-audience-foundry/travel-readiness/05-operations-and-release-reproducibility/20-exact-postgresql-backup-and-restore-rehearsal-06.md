## `fix(ops): cover official duration integrity`

diff --git a/operations/sensitive_absence.py b/operations/sensitive_absence.py
index 31b6621..ab422e0 100644
--- a/operations/sensitive_absence.py
+++ b/operations/sensitive_absence.py
@@ -127,6 +127,10 @@ BACKUP_INTEGRITY_KEYS: Final = frozenset(
         "table.travel_windows_publishedflightschedule.sha256",
         "table.travel_windows_revieweddurationreceipt.count",
         "table.travel_windows_revieweddurationreceipt.sha256",
+        "table.travel_windows_routedurationderivation.count",
+        "table.travel_windows_routedurationderivation.sha256",
+        "table.travel_windows_routedurationderivedroute.count",
+        "table.travel_windows_routedurationderivedroute.sha256",
         "table.travel_windows_routedurationrevision.count",
         "table.travel_windows_routedurationrevision.sha256",
         "pointer.entry.jp",
diff --git a/operations/tests/test_postgresql_backup_restore.py b/operations/tests/test_postgresql_backup_restore.py
index 3ead59c..e863fb2 100644
--- a/operations/tests/test_postgresql_backup_restore.py
+++ b/operations/tests/test_postgresql_backup_restore.py
@@ -1019,6 +1019,8 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
             "travel_windows_flightschedulerevision",
             "travel_windows_publishedflightschedule",
             "travel_windows_revieweddurationreceipt",
+            "travel_windows_routedurationderivation",
+            "travel_windows_routedurationderivedroute",
             "travel_windows_routedurationrevision",
         }
         contract_tables = set(
@@ -1159,6 +1161,8 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
             "travel_windows_flightschedulerevision",
             "travel_windows_publishedflightschedule",
             "travel_windows_revieweddurationreceipt",
+            "travel_windows_routedurationderivation",
+            "travel_windows_routedurationderivedroute",
             "travel_windows_routedurationrevision",
         )
         for table in protected_tables:
@@ -1209,15 +1213,15 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
             .split("'\n", 1)[0]
             .splitlines()
         )
-        self.assertEqual(len(functions), 42)
-        self.assertEqual(len({line.split("|", 1)[0] for line in functions}), 42)
+        self.assertEqual(len(functions), 44)
+        self.assertEqual(len({line.split("|", 1)[0] for line in functions}), 44)
         expected_function_names = {line.split("|", 1)[0] for line in functions}
         for line in functions:
             name, digest = line.split("|")
             self.assertRegex(name, r"^[a-z][a-z0-9_]+$")
             self.assertRegex(digest, r"^[0-9a-f]{64}$")
-        self.assertEqual(len(triggers), 54)
-        self.assertEqual(len({line.split("|", 1)[0] for line in triggers}), 54)
+        self.assertEqual(len(triggers), 58)
+        self.assertEqual(len({line.split("|", 1)[0] for line in triggers}), 58)
         for line in triggers:
             self.assertEqual(len(line.split("|")), 7)
 
@@ -1248,7 +1252,7 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
                 )
 
         function_pattern = re.compile(
-            r"CREATE(?: OR REPLACE)? FUNCTION ([a-z0-9_]+)\(\) "
+            r"CREATE(?: OR REPLACE)? FUNCTION ([a-z0-9_]+)\(\)\s+"
             r"RETURNS trigger\s+LANGUAGE plpgsql AS \$\$(.*?)\$\$;",
             re.DOTALL,
         )
diff --git a/operations/tests/test_sensitive_absence.py b/operations/tests/test_sensitive_absence.py
index af2f6d8..7b619a0 100644
--- a/operations/tests/test_sensitive_absence.py
+++ b/operations/tests/test_sensitive_absence.py
@@ -505,13 +505,17 @@ class RuntimeAndBackupBoundaryTests(unittest.TestCase):
             )
         )
         self.assertEqual(observed, set(BACKUP_INTEGRITY_KEYS))
-        self.assertEqual(len(BACKUP_INTEGRITY_KEYS), 92)
+        self.assertEqual(len(BACKUP_INTEGRITY_KEYS), 96)
         self.assertTrue(
             {
                 "table.sources_parseruninput.count",
                 "table.sources_parseruninput.sha256",
                 "table.travel_windows_revieweddurationreceipt.count",
                 "table.travel_windows_revieweddurationreceipt.sha256",
+                "table.travel_windows_routedurationderivation.count",
+                "table.travel_windows_routedurationderivation.sha256",
+                "table.travel_windows_routedurationderivedroute.count",
+                "table.travel_windows_routedurationderivedroute.sha256",
                 "pointer.entry.jp",
                 "pointer.entry.th",
                 "pointer.travel_warning.jp",
diff --git a/scripts/postgresql-common b/scripts/postgresql-common
index fad7eee..65b626a 100644
--- a/scripts/postgresql-common
+++ b/scripts/postgresql-common
@@ -6,11 +6,11 @@
 POSTGRESQL_REQUIRED_VERSION="18.6"
 EXPECTED_PUBLIC_SCHEMA_COMMENT_SHA256="998e3ff6338d1b13197feaef3c7657578604caf2185d3fe9e5ac04ba171682c8"
 
-EXPECTED_SCHEMA_DIGESTS='schema.columns.sha256=29898303816ef6ae9b01d88fa885e214f6d00b4e055e7d4e7a3ed45655ec6e66
-schema.constraints.sha256=afb800d818f6a0c428786c6311c8cb5e54f5f4fe35fbde4f51d636931d18aef4
-schema.indexes.sha256=1420eb0740b9175ffe9ee40ab827689d8b5e83f6cde2e8224df277c7f09f1025
-schema.trigger_functions.sha256=14d6611459b45315aae8aa82f520f1083fbe3a03f72da0f034878901f5bc056e
-schema.triggers.sha256=fc1411aec1f0e0e6268e7ddb7a5a6e53a465214ce22c24bc9791f05a54122f5b'
+EXPECTED_SCHEMA_DIGESTS='schema.columns.sha256=2f2028991fa88816da01a45724254d18be52f296612372116e6ed2f9ab4da78f
+schema.constraints.sha256=cea6ee189d10c940d3513493a9b43d2e4a4f74424a1f46fb2db7e557adf0946e
+schema.indexes.sha256=4fe835fee9a73da2cd3109666dd6a064a5279f17398bcc13bf8fcc90e1a9ed03
+schema.trigger_functions.sha256=e8d28f3dc681bea77288ca8c9da6fd552a8ade69939e6873028ce7f8edab2d04
+schema.triggers.sha256=48c9c858620a341cd8f2513d9593f7a2146bf88886b1ba76049dac2140af953e'
 
 EXPECTED_SEQUENCE_DEFINITION_SHA256="e9303b2ecb873b98391c36822e7ce66ee8c14062e5f1bf20c9e50f697f684152"
 
@@ -50,6 +50,8 @@ travel_windows_flightschedule
 travel_windows_flightschedulerevision
 travel_windows_publishedflightschedule
 travel_windows_revieweddurationreceipt
+travel_windows_routedurationderivation
+travel_windows_routedurationderivedroute
 travel_windows_routedurationrevision'
 
 EXPECTED_PUBLIC_SEQUENCES='auth_group_id_seq
@@ -99,8 +101,8 @@ sources_apply_rights_rejection|e02368696d3a102bd0c7cbfd2f1078b58c4d63f96046c625e
 sources_guard_artifact_mutation|59039e6f2c37fa65deeea03b88d737aaa6c841fce8f00e173db1d3618736b485
 sources_guard_configuration_change|1eb4fbbf3cfe95d940364bc8f36d278b044739409100df1b5a9ad42049334fc7
 sources_guard_fetch_attempt_mutation|78195da0ef6c4dfc3012edfbc98dcb8323ee2d09fd1e52c0d1f999b5391c98c4
-sources_guard_parse_run_change|87001036fbaf43537034a8ab8a335b5908bb1a06a7adb81bc13ff11a44460397
-sources_guard_parse_run_input|b66c4ec9a246ed1a07d645b7b71f6028f517e6d502430913095cef60601a7a0c
+sources_guard_parse_run_change|5dc5edd87da5cbc3174e29067a89db0d1a26c6bb60f67e4e3ed80400aa462cc6
+sources_guard_parse_run_input|fec41e4f9fa3f80853b429572303ad3eb234ac2d480711082bde3f8d6fe1f8d9
 sources_reject_rights_mutation|76be8ad85eab7939fe21596429e52e83ad1c7e1a1cac6954244a52606a730d46
 sources_reject_rights_revision_reuse|f96554369f58cd3017114acfd0e8772c9c7fd0252d8ebcbf493359af36310fad
 sources_validate_artifact_insert|e4e7711cc826af34a8a1638e21be3d3c01a6db25824230b4cf31d0238e0ea820
@@ -111,8 +113,10 @@ travel_warnings_validate_revision_source_rights|04bcea1c70b44524d34eae5b8ed89797
 travel_windows_guard_candidate_duration_insert|31b7efff67ecc279e32809468568f6f85c281c5a5b9b633908765990b2275615
 travel_windows_guard_candidate_publication_duration|6b9f1f9d666efaac68bcbdb85f4d5d0da8944ef59b56458135c09376d41806a0
 travel_windows_guard_candidate_seal_insert|c55a8fa11c8a5bffbfc4a224cee8acd5c5b8ecb6cf1bdf75624dc215503cbe79
+travel_windows_guard_duration_derivation_insert|4d5073ea90ae4fb7d9a007fb743ff97988ded9f39fc5f4c2c3c51b1db85609db
+travel_windows_guard_duration_derived_route_insert|2a64b5394abb34f93c0726b6984d1081fad022f7e27ed4cf24a6009898742ded
 travel_windows_guard_duration_lineage|8101e5e174622f97c4a5738aad4b1fe932ca6833b77e476adab0abe5e182b63f
-travel_windows_guard_duration_receipt|03ee71f04e5a7db7267caf64965f738d28013a3445d2b7d14a265f22dcff4f2d
+travel_windows_guard_duration_receipt|83c6a46173737c75b4ac92f433ed7983bd263a43766cc3dca460a3cf61da20de
 travel_windows_guard_flight_audit_insert|43daf76250f0b82744d7329c23ed9920bf3dcbaa2041f9c8e4e48cd2a34ff803
 travel_windows_guard_flight_pointer|fb81532972ade5b7bfcedf0481f94c8ef3f3b65365cdc978da00104185b25b5b
 travel_windows_guard_flight_publication_insert|c5f98cc463a8cd4d8ed738b78a37fd467036fa0ebf0a419c907bbc7c5b660c64
@@ -165,6 +169,10 @@ travel_windows_candidate_duration_immutable|travel_windows_flightcandidatedurati
 travel_windows_candidate_duration_insert_guard|travel_windows_flightcandidateduration|travel_windows_guard_candidate_duration_insert|7|false|false|false
 travel_windows_candidate_seal_immutable|travel_windows_flightcandidateseal|travel_windows_reject_revision_mutation|27|false|false|false
 travel_windows_candidate_seal_insert_guard|travel_windows_flightcandidateseal|travel_windows_guard_candidate_seal_insert|7|false|false|false
+travel_windows_duration_derivation_immutable|travel_windows_routedurationderivation|travel_windows_reject_revision_mutation|27|false|false|false
+travel_windows_duration_derivation_insert_guard|travel_windows_routedurationderivation|travel_windows_guard_duration_derivation_insert|7|false|false|false
+travel_windows_duration_derived_route_immutable|travel_windows_routedurationderivedroute|travel_windows_reject_revision_mutation|27|false|false|false
+travel_windows_duration_derived_route_insert_guard|travel_windows_routedurationderivedroute|travel_windows_guard_duration_derived_route_insert|7|false|false|false
 travel_windows_duration_lineage_guard|travel_windows_routedurationrevision|travel_windows_guard_duration_lineage|7|false|false|false
 travel_windows_duration_receipt_guard|travel_windows_revieweddurationreceipt|travel_windows_guard_duration_receipt|31|false|false|false
 travel_windows_duration_revision_immutable|travel_windows_routedurationrevision|travel_windows_reject_revision_mutation|27|false|false|false
diff --git a/scripts/postgresql-integrity.sql b/scripts/postgresql-integrity.sql
index 6e78933..0af9455 100644
--- a/scripts/postgresql-integrity.sql
+++ b/scripts/postgresql-integrity.sql
@@ -291,9 +291,17 @@ WITH integrity_lines(sort_key, line) AS (
       FROM public.travel_windows_revieweddurationreceipt
     UNION ALL SELECT '06l', 'table.travel_windows_revieweddurationreceipt.sha256=' || encode(sha256(convert_to(
       coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.travel_windows_revieweddurationreceipt AS t), ''), 'UTF8')), 'hex')
-    UNION ALL SELECT '06m', 'table.travel_windows_routedurationrevision.count=' || count(*)::text
+    UNION ALL SELECT '06m', 'table.travel_windows_routedurationderivation.count=' || count(*)::text
+      FROM public.travel_windows_routedurationderivation
+    UNION ALL SELECT '06n', 'table.travel_windows_routedurationderivation.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.travel_windows_routedurationderivation AS t), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '06o', 'table.travel_windows_routedurationderivedroute.count=' || count(*)::text
+      FROM public.travel_windows_routedurationderivedroute
+    UNION ALL SELECT '06p', 'table.travel_windows_routedurationderivedroute.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.travel_windows_routedurationderivedroute AS t), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '06q', 'table.travel_windows_routedurationrevision.count=' || count(*)::text
       FROM public.travel_windows_routedurationrevision
-    UNION ALL SELECT '06n', 'table.travel_windows_routedurationrevision.sha256=' || encode(sha256(convert_to(
+    UNION ALL SELECT '06r', 'table.travel_windows_routedurationrevision.sha256=' || encode(sha256(convert_to(
       coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.travel_windows_routedurationrevision AS t), ''), 'UTF8')), 'hex')
 
     UNION ALL SELECT '070', 'pointer.entry.jp=' || coalesce((


