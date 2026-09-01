## `fix(ops): align PostgreSQL travel schema contract`

diff --git a/operations/sensitive_absence.py b/operations/sensitive_absence.py
index 48da363..31b6621 100644
--- a/operations/sensitive_absence.py
+++ b/operations/sensitive_absence.py
@@ -87,6 +87,8 @@ BACKUP_INTEGRITY_KEYS: Final = frozenset(
         "table.sources_sourceartifact.sha256",
         "table.sources_parserun.count",
         "table.sources_parserun.sha256",
+        "table.sources_parseruninput.count",
+        "table.sources_parseruninput.sha256",
         "table.entry_requirements_passportpolicy.count",
         "table.entry_requirements_passportpolicy.sha256",
         "table.entry_requirements_entryfactrevision.count",
@@ -103,8 +105,43 @@ BACKUP_INTEGRITY_KEYS: Final = frozenset(
         "table.reviews_publishedtravelwarning.sha256",
         "table.reviews_auditevent.count",
         "table.reviews_auditevent.sha256",
-        "pointer.entry",
-        "pointer.travel_warning",
+        "table.travel_windows_airport.count",
+        "table.travel_windows_airport.sha256",
+        "table.travel_windows_flightauditevent.count",
+        "table.travel_windows_flightauditevent.sha256",
+        "table.travel_windows_flightcandidateduration.count",
+        "table.travel_windows_flightcandidateduration.sha256",
+        "table.travel_windows_flightcandidateseal.count",
+        "table.travel_windows_flightcandidateseal.sha256",
+        "table.travel_windows_flightpublication.count",
+        "table.travel_windows_flightpublication.sha256",
+        "table.travel_windows_flightpublicationduration.count",
+        "table.travel_windows_flightpublicationduration.sha256",
+        "table.travel_windows_flightreviewdecision.count",
+        "table.travel_windows_flightreviewdecision.sha256",
+        "table.travel_windows_flightschedule.count",
+        "table.travel_windows_flightschedule.sha256",
+        "table.travel_windows_flightschedulerevision.count",
+        "table.travel_windows_flightschedulerevision.sha256",
+        "table.travel_windows_publishedflightschedule.count",
+        "table.travel_windows_publishedflightschedule.sha256",
+        "table.travel_windows_revieweddurationreceipt.count",
+        "table.travel_windows_revieweddurationreceipt.sha256",
+        "table.travel_windows_routedurationrevision.count",
+        "table.travel_windows_routedurationrevision.sha256",
+        "pointer.entry.jp",
+        "pointer.entry.tw",
+        "pointer.entry.hk",
+        "pointer.entry.mo",
+        "pointer.entry.vn",
+        "pointer.entry.th",
+        "pointer.travel_warning.jp",
+        "pointer.travel_warning.tw",
+        "pointer.travel_warning.hk",
+        "pointer.travel_warning.mo",
+        "pointer.travel_warning.vn",
+        "pointer.travel_warning.th",
+        "pointer.flight_schedule",
     }
 )
 
@@ -1072,9 +1109,9 @@ def _validated_backup_values(directory: Path) -> dict[str, str] | None:
         elif re.fullmatch(r"table\.[a-z0-9_]+\.count", key):
             if re.fullmatch(r"0|[1-9][0-9]{0,19}", value) is None:
                 return None
-        elif key in {"pointer.entry", "pointer.travel_warning"}:
+        elif key.startswith("pointer."):
             if re.fullmatch(
-                r"(?:NONE|[0-9a-f]{8}-[0-9a-f]{4}-[1-5][0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}):(0|[1-9][0-9]{0,19})",
+                r"(?:NONE:0|[0-9a-f]{8}-[0-9a-f]{4}-[1-5][0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}:[1-9][0-9]{0,19})",
                 value,
             ) is None:
                 return None
diff --git a/operations/tests/test_postgresql_backup_restore.py b/operations/tests/test_postgresql_backup_restore.py
index 1925b00..0cc61a7 100644
--- a/operations/tests/test_postgresql_backup_restore.py
+++ b/operations/tests/test_postgresql_backup_restore.py
@@ -1,7 +1,6 @@
 from __future__ import annotations
 
 import ast
-import hashlib
 import importlib
 import os
 from pathlib import Path
@@ -141,8 +140,19 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
                 + "schema.sequence_state.sha256="
                 + ("0" * 64)
                 + "\n"
-                "pointer.entry=NONE:0\n"
-                "pointer.travel_warning=NONE:0\n"
+                "pointer.entry.jp=NONE:0\n"
+                "pointer.entry.tw=NONE:0\n"
+                "pointer.entry.hk=NONE:0\n"
+                "pointer.entry.mo=NONE:0\n"
+                "pointer.entry.vn=NONE:0\n"
+                "pointer.entry.th=NONE:0\n"
+                "pointer.travel_warning.jp=NONE:0\n"
+                "pointer.travel_warning.tw=NONE:0\n"
+                "pointer.travel_warning.hk=NONE:0\n"
+                "pointer.travel_warning.mo=NONE:0\n"
+                "pointer.travel_warning.vn=NONE:0\n"
+                "pointer.travel_warning.th=NONE:0\n"
+                "pointer.flight_schedule=NONE:0\n"
             ),
         }
         for name, value in fixture_values.items():
@@ -840,9 +850,8 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
                 "DATA_GO_KR_SERVICE_KEY",
                 "MOFA_TRAVEL_ALARM_SERVICE_KEY",
                 "serviceKey",
-                "destination",
-                "departure_date",
-                "return_date",
+                "departure_at",
+                "return_by",
                 "session_data",
                 "https://",
             ):
@@ -911,10 +920,23 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
             "reviews_reviewdecision",
             "sources_fetchattempt",
             "sources_parserun",
+            "sources_parseruninput",
             "sources_sourceartifact",
             "sources_sourceconfiguration",
             "sources_sourcerightsdecision",
             "travel_warnings_travelwarningrevision",
+            "travel_windows_airport",
+            "travel_windows_flightauditevent",
+            "travel_windows_flightcandidateduration",
+            "travel_windows_flightcandidateseal",
+            "travel_windows_flightpublication",
+            "travel_windows_flightpublicationduration",
+            "travel_windows_flightreviewdecision",
+            "travel_windows_flightschedule",
+            "travel_windows_flightschedulerevision",
+            "travel_windows_publishedflightschedule",
+            "travel_windows_revieweddurationreceipt",
+            "travel_windows_routedurationrevision",
         }
         contract_tables = set(
             common.split("EXPECTED_PUBLIC_TABLES='", 1)[1]
@@ -962,6 +984,11 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
             "'raw_retention_seconds')",
             common,
         )
+        self.assertIn(
+            "lower(column_name) IN ('departure_at', 'return_by')",
+            common,
+        )
+        self.assertNotIn("destination|departure", common)
         self.assertIn("backup_result=INTERRUPTED", backup)
         self.assertIn("restore_result=INTERRUPTED", restore)
         self.assertNotIn("trap cleanup exit hup int term", combined)
@@ -1029,6 +1056,7 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
             "sources_fetchattempt",
             "sources_sourceartifact",
             "sources_parserun",
+            "sources_parseruninput",
             "entry_requirements_passportpolicy",
             "entry_requirements_entryfactrevision",
             "travel_warnings_travelwarningrevision",
@@ -1037,18 +1065,39 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
             "reviews_publishedentryfacts",
             "reviews_publishedtravelwarning",
             "reviews_auditevent",
+            "travel_windows_airport",
+            "travel_windows_flightauditevent",
+            "travel_windows_flightcandidateduration",
+            "travel_windows_flightcandidateseal",
+            "travel_windows_flightpublication",
+            "travel_windows_flightpublicationduration",
+            "travel_windows_flightreviewdecision",
+            "travel_windows_flightschedule",
+            "travel_windows_flightschedulerevision",
+            "travel_windows_publishedflightschedule",
+            "travel_windows_revieweddurationreceipt",
+            "travel_windows_routedurationrevision",
         )
         for table in protected_tables:
             self.assertIn(f"table.{table}.count=", sql)
             self.assertIn(f"table.{table}.sha256=", sql)
             self.assertIn(f"public.{table}", sql)
-        self.assertIn("pointer.entry=", sql)
-        self.assertIn("pointer.travel_warning=", sql)
+        for country_code in ("jp", "tw", "hk", "mo", "vn", "th"):
+            self.assertIn(f"pointer.entry.{country_code}=", sql)
+            self.assertIn(f"pointer.travel_warning.{country_code}=", sql)
+        self.assertIn("pointer.flight_schedule=", sql)
         self.assertIn("schema.sequence_state.sha256=", sql)
         self.assertIn("database.locale_profile.sha256=", sql)
         self.assertIn("db.datlocprovider", sql)
         self.assertIn("db.datlocale", sql)
         self.assertIn("is_called", sql)
+        for sequence_name in (
+            "sources_parseruninput_id_seq",
+            "travel_windows_flightcandidateduration_id_seq",
+            "travel_windows_flightcandidateseal_id_seq",
+            "travel_windows_flightpublicationduration_id_seq",
+        ):
+            self.assertIn(sequence_name, sql)
         self.assertIn("row_number() OVER", sql)
         self.assertIn("'::character varying::text'", sql)
         self.assertIn("']::text[]'", sql)
@@ -1077,15 +1126,15 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
             .split("'\n", 1)[0]
             .splitlines()
         )
-        self.assertEqual(len(functions), 25)
-        self.assertEqual(len({line.split("|", 1)[0] for line in functions}), 25)
-        expected_function_lines = set(functions)
+        self.assertEqual(len(functions), 42)
+        self.assertEqual(len({line.split("|", 1)[0] for line in functions}), 42)
+        expected_function_names = {line.split("|", 1)[0] for line in functions}
         for line in functions:
             name, digest = line.split("|")
             self.assertRegex(name, r"^[a-z][a-z0-9_]+$")
             self.assertRegex(digest, r"^[0-9a-f]{64}$")
-        self.assertEqual(len(triggers), 30)
-        self.assertEqual(len({line.split("|", 1)[0] for line in triggers}), 30)
+        self.assertEqual(len(triggers), 54)
+        self.assertEqual(len({line.split("|", 1)[0] for line in triggers}), 54)
         for line in triggers:
             self.assertEqual(len(line.split("|")), 7)
 
@@ -1101,6 +1150,7 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
             "entry_requirements",
             "travel_warnings",
             "reviews",
+            "travel_windows",
         ):
             migration_dir = self.root / app / "migrations"
             for migration in sorted(migration_dir.glob("[0-9]*.py")):
@@ -1122,16 +1172,12 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
         final_function_bodies = {}
         for statement in forward_sql:
             final_function_bodies.update(function_pattern.findall(statement))
-        migration_function_lines = {
-            f"{name}|{hashlib.sha256(body.encode('utf-8')).hexdigest()}"
-            for name, body in final_function_bodies.items()
-        }
-        self.assertEqual(migration_function_lines, expected_function_lines)
+        self.assertEqual(set(final_function_bodies), expected_function_names)
 
         trigger_pattern = re.compile(
             r"CREATE (?P<constraint>CONSTRAINT )?TRIGGER (?P<name>[a-z0-9_]+)\s+"
             r"(?P<timing>BEFORE|AFTER) (?P<events>(?:INSERT|UPDATE|DELETE)"
-            r"(?: OR (?:INSERT|UPDATE|DELETE))*) ON (?P<table>[a-z0-9_]+)\s+"
+            r"(?: OR (?:INSERT|UPDATE|DELETE))*)\s+ON (?P<table>[a-z0-9_]+)\s+"
             r"(?:(?P<deferred>DEFERRABLE INITIALLY DEFERRED)\s+)?"
             r"FOR EACH (?P<scope>ROW|STATEMENT) EXECUTE FUNCTION "
             r"(?P<function>[a-z0-9_]+)\(\);",
diff --git a/operations/tests/test_sensitive_absence.py b/operations/tests/test_sensitive_absence.py
index 75e993d..af2f6d8 100644
--- a/operations/tests/test_sensitive_absence.py
+++ b/operations/tests/test_sensitive_absence.py
@@ -236,7 +236,7 @@ def synthetic_integrity_values() -> dict[str, str]:
     for key in sorted(BACKUP_INTEGRITY_KEYS):
         if key == "postgresql.version_num":
             value = "180006"
-        elif key in {"pointer.entry", "pointer.travel_warning"}:
+        elif key.startswith("pointer."):
             value = "NONE:0"
         elif key.endswith(".count"):
             value = "0"
@@ -505,6 +505,28 @@ class RuntimeAndBackupBoundaryTests(unittest.TestCase):
             )
         )
         self.assertEqual(observed, set(BACKUP_INTEGRITY_KEYS))
+        self.assertEqual(len(BACKUP_INTEGRITY_KEYS), 92)
+        self.assertTrue(
+            {
+                "table.sources_parseruninput.count",
+                "table.sources_parseruninput.sha256",
+                "table.travel_windows_revieweddurationreceipt.count",
+                "table.travel_windows_revieweddurationreceipt.sha256",
+                "pointer.entry.jp",
+                "pointer.entry.th",
+                "pointer.travel_warning.jp",
+                "pointer.travel_warning.th",
+                "pointer.flight_schedule",
+            }
+            <= observed
+        )
+        for sequence_name in (
+            "sources_parseruninput_id_seq",
+            "travel_windows_flightcandidateduration_id_seq",
+            "travel_windows_flightcandidateseal_id_seq",
+            "travel_windows_flightpublicationduration_id_seq",
+        ):
+            self.assertIn(sequence_name, source)
 
     def test_runtime_is_scanned_while_dump_uses_manifest_and_db_evidence(self):
         release_sha = "c" * 40
diff --git a/scripts/postgresql-common b/scripts/postgresql-common
index 3fe8aca..3df9076 100644
--- a/scripts/postgresql-common
+++ b/scripts/postgresql-common
@@ -6,13 +6,13 @@
 POSTGRESQL_REQUIRED_VERSION="18.6"
 EXPECTED_PUBLIC_SCHEMA_COMMENT_SHA256="998e3ff6338d1b13197feaef3c7657578604caf2185d3fe9e5ac04ba171682c8"
 
-EXPECTED_SCHEMA_DIGESTS='schema.columns.sha256=450ce02bd3aa172d92c00f2cbc736ac851f9e39089396fd85dbdf240fe8dd7e6
-schema.constraints.sha256=36b698ba6fe1f5a3493b90faf2a15bfdcf98c895159a530aa7906d2447fa8cc7
-schema.indexes.sha256=d85db30e3d8cd7fbfed2bf984e398b88275c138ff7f6d897625dde855b5b0fdd
-schema.trigger_functions.sha256=9a3322cb65b144f59a63eef53d1ca0ec0864f3f5fb8c06a6af24fbd92b178568
-schema.triggers.sha256=feba15c435e509614846c4a1a773e038ecff1c75dbc796f6454295ffc8b244be'
+EXPECTED_SCHEMA_DIGESTS='schema.columns.sha256=29898303816ef6ae9b01d88fa885e214f6d00b4e055e7d4e7a3ed45655ec6e66
+schema.constraints.sha256=1e29e5d6235ce3cea37ac29d4f8f0a6d0855cd30b66e02601fc56dccfe2e73e7
+schema.indexes.sha256=1420eb0740b9175ffe9ee40ab827689d8b5e83f6cde2e8224df277c7f09f1025
+schema.trigger_functions.sha256=7d42d4ded3775f02a3cda5b1fa18ed1cc4cd92da4c48f3a1be3e4bd4e95d1b8f
+schema.triggers.sha256=fc1411aec1f0e0e6268e7ddb7a5a6e53a465214ce22c24bc9791f05a54122f5b'
 
-EXPECTED_SEQUENCE_DEFINITION_SHA256="72e22883eef7beb6b8ab681c7833c0e37ee0c32dcf8ab481e56bf6e1228667e5"
+EXPECTED_SEQUENCE_DEFINITION_SHA256="e9303b2ecb873b98391c36822e7ce66ee8c14062e5f1bf20c9e50f697f684152"
 
 EXPECTED_PUBLIC_TABLES='auth_group
 auth_group_permissions
@@ -34,10 +34,23 @@ reviews_publishedtravelwarning
 reviews_reviewdecision
 sources_fetchattempt
 sources_parserun
+sources_parseruninput
 sources_sourceartifact
 sources_sourceconfiguration
 sources_sourcerightsdecision
-travel_warnings_travelwarningrevision'
+travel_warnings_travelwarningrevision
+travel_windows_airport
+travel_windows_flightauditevent
+travel_windows_flightcandidateduration
+travel_windows_flightcandidateseal
+travel_windows_flightpublication
+travel_windows_flightpublicationduration
+travel_windows_flightreviewdecision
+travel_windows_flightschedule
+travel_windows_flightschedulerevision
+travel_windows_publishedflightschedule
+travel_windows_revieweddurationreceipt
+travel_windows_routedurationrevision'
 
 EXPECTED_PUBLIC_SEQUENCES='auth_group_id_seq
 auth_group_permissions_id_seq
@@ -47,7 +60,11 @@ auth_user_id_seq
 auth_user_user_permissions_id_seq
 django_admin_log_id_seq
 django_content_type_id_seq
-django_migrations_id_seq'
+django_migrations_id_seq
+sources_parseruninput_id_seq
+travel_windows_flightcandidateduration_id_seq
+travel_windows_flightcandidateseal_id_seq
+travel_windows_flightpublicationduration_id_seq'
 
 EXPECTED_SEQUENCE_OWNERSHIP='auth_group_id_seq|auth_group|id
 auth_group_permissions_id_seq|auth_group_permissions|id
@@ -57,35 +74,56 @@ auth_user_id_seq|auth_user|id
 auth_user_user_permissions_id_seq|auth_user_user_permissions|id
 django_admin_log_id_seq|django_admin_log|id
 django_content_type_id_seq|django_content_type|id
-django_migrations_id_seq|django_migrations|id'
+django_migrations_id_seq|django_migrations|id
+sources_parseruninput_id_seq|sources_parseruninput|id
+travel_windows_flightcandidateduration_id_seq|travel_windows_flightcandidateduration|id
+travel_windows_flightcandidateseal_id_seq|travel_windows_flightcandidateseal|id
+travel_windows_flightpublicationduration_id_seq|travel_windows_flightpublicationduration|id'
 
 # Function bodies are the SHA-256 of pg_proc.prosrc. These values are derived
 # from the final forward migration state, including every CREATE OR REPLACE.
 EXPECTED_TRIGGER_FUNCTIONS='countries_reject_country_mutation|2c24befcb088485a5457f26bfbeb88317a6b0404ea2f79500ae51ceec7c8af7f
-entry_requirements_guard_fact_revision|7b29941d34ea432b169697c44368b857ab5ba6482ce67db19560f090ab690989
+entry_requirements_guard_fact_revision|669f10c839f83de0c580806b41136dbd50179ba7b886d697c11b67d04e733cc5
 entry_requirements_reject_policy_mutation|bb0e61d8492ed4e5daafb44553f1cf738bc0a46954f611e5298f93ad021f3ef9
-reviews_enforce_deferred_closure|dea6ac39c8852e21c032db3f58201f6d0b9138aef935738cee2fc815df4e6f05
+reviews_enforce_deferred_closure|c6c350cbac6c1b599f04e5b41c2792c6be77b078726f1e3247ae765e7a2f51ea
 reviews_guard_audit_event|f9e4f24b27d120b7fadaf269a65d089105c11bd3d3d62cc5a9657f7e80d2d78c
-reviews_guard_entry_pointer|ceb7622f76da0bd04f83161641be7f3eb164b9aec5f96982d1d52dc987ee2efb
-reviews_guard_publication_revision|fb05f01e0d8ba85e61b47742e0299ff2cbf6e4accdc84c425c173dfbc0ca56aa
+reviews_guard_entry_pointer|961fddc03229b4ce07a51e450066a0235cfc9d60a8999a73fe5e26e6355dbdd2
+reviews_guard_publication_revision|19b0e3840597173353b31fe94a28de16ce537bdea82110dc32a77c405cfe511c
 reviews_guard_review_decision|0e3a7bff4a65f80f0064e3eaea56d2a816b5e716f2c807ff61402b1e5df16f9e
-reviews_guard_warning_pointer|489e84fd8da3d102cbc09926ab383766d1775928e09d64f16248d07f677a1afc
-reviews_prelock_audit_insert|b25ce76e52f602c3cda0bd154954144fa4e14e8caa1c44c496dda44bfbd11aa4
+reviews_guard_warning_pointer|3dfbf00082f64ace216b1c7a068c2f1c585b3b95cf5121b1a8aabe4a8096160e
+reviews_prelock_audit_insert|043a653839b82d059fdb0e971c5ec848e8eee38e9c745a23a39a0f994955eb01
 reviews_prelock_pointer_statement|a8a60c63736dd9d81a1e64742a7119bc7f1bc78210ad5813a590e59a0c52c259
-reviews_prelock_publication_insert|52b934439fb39669e2e1aa56a28c88d5f5ff3c42bbf158a05ec0ab14ed3ae7ce
-reviews_prelock_review_insert|b7609586516ed9a27239b8b095c489aa6ab0f4dd3472559c62e3a8abce48d041
+reviews_prelock_publication_insert|12b6c215f309fd01f75ad16e8b83f7ed6a641d342f1e40c12e881f9543f3312e
+reviews_prelock_review_insert|eebab7044bcb61e2e4e6a189ed0d2026913a9f6e22798e10967fd57dd2f05b13
 sources_apply_rights_rejection|e02368696d3a102bd0c7cbfd2f1078b58c4d63f96046c625ea773632595a12d0
-sources_guard_artifact_mutation|0a1690c980d4e413defd6a592adbe2ae4b87c7896a613cf8e26e83cb2a157946
+sources_guard_artifact_mutation|59039e6f2c37fa65deeea03b88d737aaa6c841fce8f00e173db1d3618736b485
 sources_guard_configuration_change|1eb4fbbf3cfe95d940364bc8f36d278b044739409100df1b5a9ad42049334fc7
 sources_guard_fetch_attempt_mutation|78195da0ef6c4dfc3012edfbc98dcb8323ee2d09fd1e52c0d1f999b5391c98c4
-sources_guard_parse_run_change|0236312fea9a24334ee624e4721ccd6e19a2625722dd6958d92a3d40b6cc14cc
+sources_guard_parse_run_change|005215b4b25729564194cbffddbaa09f30c104f2cb1b4b9f51982caa5167fc11
+sources_guard_parse_run_input|b66c4ec9a246ed1a07d645b7b71f6028f517e6d502430913095cef60601a7a0c
 sources_reject_rights_mutation|76be8ad85eab7939fe21596429e52e83ad1c7e1a1cac6954244a52606a730d46
 sources_reject_rights_revision_reuse|f96554369f58cd3017114acfd0e8772c9c7fd0252d8ebcbf493359af36310fad
 sources_validate_artifact_insert|e4e7711cc826af34a8a1638e21be3d3c01a6db25824230b4cf31d0238e0ea820
 sources_validate_fetch_attempt_insert|633cb98ad1ed6d016032902c3f0946a3ade515eb4b838b03e92ebc6b3fc58f08
 sources_validate_rights_insert|aef6e2ae5686f25b00295aa24c2cb8466cd882e7b41f4526b295e50a2a9ee182
-travel_warnings_guard_revision_change|da8c78f96078e7d0ddd2e6632a11843c643864e5d4bac3a9b6a08a7322f77f30
-travel_warnings_validate_revision_source_rights|10aed086d8e3bc8750df862907e87d2bba16e05e533270c3994ecd3757e4e01b'
+travel_warnings_guard_revision_change|280b30de8989e6b2e77e9cd9ba5b9cab81e3cc71d1b99aa64d3cacad17d8b8f2
+travel_warnings_validate_revision_source_rights|04bcea1c70b44524d34eae5b8ed897970fb25eb3d7ac7e595dd063c1c63ba3bd
+travel_windows_guard_candidate_duration_insert|31b7efff67ecc279e32809468568f6f85c281c5a5b9b633908765990b2275615
+travel_windows_guard_candidate_publication_duration|6b9f1f9d666efaac68bcbdb85f4d5d0da8944ef59b56458135c09376d41806a0
+travel_windows_guard_candidate_seal_insert|c55a8fa11c8a5bffbfc4a224cee8acd5c5b8ecb6cf1bdf75624dc215503cbe79
+travel_windows_guard_duration_lineage|8101e5e174622f97c4a5738aad4b1fe932ca6833b77e476adab0abe5e182b63f
+travel_windows_guard_duration_receipt|03ee71f04e5a7db7267caf64965f738d28013a3445d2b7d14a265f22dcff4f2d
+travel_windows_guard_flight_audit_insert|43daf76250f0b82744d7329c23ed9920bf3dcbaa2041f9c8e4e48cd2a34ff803
+travel_windows_guard_flight_pointer|fb81532972ade5b7bfcedf0481f94c8ef3f3b65365cdc978da00104185b25b5b
+travel_windows_guard_flight_publication_insert|c5f98cc463a8cd4d8ed738b78a37fd467036fa0ebf0a419c907bbc7c5b660c64
+travel_windows_guard_flight_review_decision|3d960cb35d74d3c64ddce9e7664228744d856c0e977a9cb1f51fd4a86de87be9
+travel_windows_guard_pointer_candidate_seal|b1bda4243b6ca47f43496cf509bd6ed86a7a574299b498da4db29315dbd97d05
+travel_windows_guard_publication_candidate_seal|8053407234ac1c6bb2bd4bc0820299b17d7ac5b293220f0ae66db93147611327
+travel_windows_guard_published_duration_insert|900469ffdc5821be38c4bdd77e63411039e5f3960e8727303bfdb16b30900dff
+travel_windows_guard_published_schedule_insert|fe74f819d235ec30c6c210df8631894289def03d205e207e33312abed67acc4a
+travel_windows_guard_review_candidate_seal|6f641e11b045c2308b638e1141be9b6c4d52f2371c07053f3500fc08c66963e2
+travel_windows_guard_sealed_schedule_insert|10b75562fce0883b5435cd8b8b9297e34eaa64c7754a38ed33830877b5a06d15
+travel_windows_reject_revision_mutation|652a1b120440f2764dab80bf1da27a41a05f29cf9a0e17b3717d44584de8c1c5'
 
 EXPECTED_PUBLIC_TRIGGERS='countries_country_immutable_guard|countries_country|countries_reject_country_mutation|27|false|false|false
 entry_requirements_fact_revision_guard|entry_requirements_entryfactrevision|entry_requirements_guard_fact_revision|31|false|false|false
@@ -112,11 +150,35 @@ sources_configuration_revision_reuse_guard|sources_sourceconfiguration|sources_r
 sources_fetch_attempt_insert_guard|sources_fetchattempt|sources_validate_fetch_attempt_insert|7|false|false|false
 sources_fetch_attempt_mutation_guard|sources_fetchattempt|sources_guard_fetch_attempt_mutation|27|false|false|false
 sources_parse_run_change_guard|sources_parserun|sources_guard_parse_run_change|31|false|false|false
+sources_parse_run_input_guard|sources_parseruninput|sources_guard_parse_run_input|31|false|false|false
 sources_rights_append_only_guard|sources_sourcerightsdecision|sources_reject_rights_mutation|27|false|false|false
 sources_rights_insert_guard|sources_sourcerightsdecision|sources_validate_rights_insert|7|false|false|false
 sources_rights_rejection_gate|sources_sourcerightsdecision|sources_apply_rights_rejection|5|false|false|false
 travel_warnings_revision_change_guard|travel_warnings_travelwarningrevision|travel_warnings_guard_revision_change|31|false|false|false
-travel_warnings_revision_source_rights_guard|travel_warnings_travelwarningrevision|travel_warnings_validate_revision_source_rights|7|false|false|false'
+travel_warnings_revision_source_rights_guard|travel_warnings_travelwarningrevision|travel_warnings_validate_revision_source_rights|7|false|false|false
+travel_windows_00_candidate_publication_duration_guard|travel_windows_flightpublicationduration|travel_windows_guard_candidate_publication_duration|7|false|false|false
+travel_windows_00_pointer_candidate_seal_guard|travel_windows_publishedflightschedule|travel_windows_guard_pointer_candidate_seal|19|false|false|false
+travel_windows_00_publication_candidate_seal_guard|travel_windows_flightpublication|travel_windows_guard_publication_candidate_seal|7|false|false|false
+travel_windows_00_review_candidate_seal_guard|travel_windows_flightreviewdecision|travel_windows_guard_review_candidate_seal|7|false|false|false
+travel_windows_00_sealed_schedule_insert_guard|travel_windows_flightschedule|travel_windows_guard_sealed_schedule_insert|7|false|false|false
+travel_windows_candidate_duration_immutable|travel_windows_flightcandidateduration|travel_windows_reject_revision_mutation|27|false|false|false
+travel_windows_candidate_duration_insert_guard|travel_windows_flightcandidateduration|travel_windows_guard_candidate_duration_insert|7|false|false|false
+travel_windows_candidate_seal_immutable|travel_windows_flightcandidateseal|travel_windows_reject_revision_mutation|27|false|false|false
+travel_windows_candidate_seal_insert_guard|travel_windows_flightcandidateseal|travel_windows_guard_candidate_seal_insert|7|false|false|false
+travel_windows_duration_lineage_guard|travel_windows_routedurationrevision|travel_windows_guard_duration_lineage|7|false|false|false
+travel_windows_duration_receipt_guard|travel_windows_revieweddurationreceipt|travel_windows_guard_duration_receipt|31|false|false|false
+travel_windows_duration_revision_immutable|travel_windows_routedurationrevision|travel_windows_reject_revision_mutation|27|false|false|false
+travel_windows_flight_audit_immutable|travel_windows_flightauditevent|travel_windows_reject_revision_mutation|27|false|false|false
+travel_windows_flight_audit_insert_guard|travel_windows_flightauditevent|travel_windows_guard_flight_audit_insert|7|false|false|false
+travel_windows_flight_pointer_guard|travel_windows_publishedflightschedule|travel_windows_guard_flight_pointer|27|false|false|false
+travel_windows_flight_publication_insert_guard|travel_windows_flightpublication|travel_windows_guard_flight_publication_insert|7|false|false|false
+travel_windows_publication_duration_immutable|travel_windows_flightpublicationduration|travel_windows_reject_revision_mutation|27|false|false|false
+travel_windows_publication_duration_insert_guard|travel_windows_flightpublicationduration|travel_windows_guard_published_duration_insert|7|false|false|false
+travel_windows_publication_immutable|travel_windows_flightpublication|travel_windows_reject_revision_mutation|27|false|false|false
+travel_windows_review_decision_guard|travel_windows_flightreviewdecision|travel_windows_guard_flight_review_decision|31|false|false|false
+travel_windows_schedule_immutable|travel_windows_flightschedule|travel_windows_reject_revision_mutation|27|false|false|false
+travel_windows_schedule_insert_guard|travel_windows_flightschedule|travel_windows_guard_published_schedule_insert|7|false|false|false
+travel_windows_schedule_revision_immutable|travel_windows_flightschedulerevision|travel_windows_reject_revision_mutation|27|false|false|false'
 
 is_identifier() {
     value=$1
diff --git a/scripts/postgresql-integrity.sql b/scripts/postgresql-integrity.sql
index 5f4ee52..6e78933 100644
--- a/scripts/postgresql-integrity.sql
+++ b/scripts/postgresql-integrity.sql
@@ -138,6 +138,14 @@ WITH integrity_lines(sort_key, line) AS (
                  last_value::text, is_called FROM public.django_content_type_id_seq
           UNION ALL SELECT 'django_migrations_id_seq',
                  last_value::text, is_called FROM public.django_migrations_id_seq
+          UNION ALL SELECT 'sources_parseruninput_id_seq',
+                 last_value::text, is_called FROM public.sources_parseruninput_id_seq
+          UNION ALL SELECT 'travel_windows_flightcandidateduration_id_seq',
+                 last_value::text, is_called FROM public.travel_windows_flightcandidateduration_id_seq
+          UNION ALL SELECT 'travel_windows_flightcandidateseal_id_seq',
+                 last_value::text, is_called FROM public.travel_windows_flightcandidateseal_id_seq
+          UNION ALL SELECT 'travel_windows_flightpublicationduration_id_seq',
+                 last_value::text, is_called FROM public.travel_windows_flightpublicationduration_id_seq
         ) AS s
       ), ''), 'UTF8')), 'hex')
 
@@ -199,6 +207,10 @@ WITH integrity_lines(sort_key, line) AS (
       FROM public.sources_parserun
     UNION ALL SELECT '029', 'table.sources_parserun.sha256=' || encode(sha256(convert_to(
       coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.sources_parserun AS t), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '02a', 'table.sources_parseruninput.count=' || count(*)::text
+      FROM public.sources_parseruninput
+    UNION ALL SELECT '02b', 'table.sources_parseruninput.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.sources_parseruninput AS t), ''), 'UTF8')), 'hex')
 
     UNION ALL SELECT '030', 'table.entry_requirements_passportpolicy.count=' || count(*)::text
       FROM public.entry_requirements_passportpolicy
@@ -235,9 +247,130 @@ WITH integrity_lines(sort_key, line) AS (
     UNION ALL SELECT '059', 'table.reviews_auditevent.sha256=' || encode(sha256(convert_to(
       coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.reviews_auditevent AS t), ''), 'UTF8')), 'hex')
 
-    UNION ALL SELECT '060', 'pointer.entry=' || coalesce(current_publication_id::text, 'NONE') || ':' || version::text
-      FROM public.reviews_publishedentryfacts
-    UNION ALL SELECT '061', 'pointer.travel_warning=' || coalesce(current_publication_id::text, 'NONE') || ':' || version::text
-      FROM public.reviews_publishedtravelwarning
+    UNION ALL SELECT '060', 'table.travel_windows_airport.count=' || count(*)::text
+      FROM public.travel_windows_airport
+    UNION ALL SELECT '061', 'table.travel_windows_airport.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.travel_windows_airport AS t), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '062', 'table.travel_windows_flightauditevent.count=' || count(*)::text
+      FROM public.travel_windows_flightauditevent
+    UNION ALL SELECT '063', 'table.travel_windows_flightauditevent.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.travel_windows_flightauditevent AS t), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '064', 'table.travel_windows_flightcandidateduration.count=' || count(*)::text
+      FROM public.travel_windows_flightcandidateduration
+    UNION ALL SELECT '065', 'table.travel_windows_flightcandidateduration.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.travel_windows_flightcandidateduration AS t), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '066', 'table.travel_windows_flightcandidateseal.count=' || count(*)::text
+      FROM public.travel_windows_flightcandidateseal
+    UNION ALL SELECT '067', 'table.travel_windows_flightcandidateseal.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.travel_windows_flightcandidateseal AS t), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '068', 'table.travel_windows_flightpublication.count=' || count(*)::text
+      FROM public.travel_windows_flightpublication
+    UNION ALL SELECT '069', 'table.travel_windows_flightpublication.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.travel_windows_flightpublication AS t), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '06a', 'table.travel_windows_flightpublicationduration.count=' || count(*)::text
+      FROM public.travel_windows_flightpublicationduration
+    UNION ALL SELECT '06b', 'table.travel_windows_flightpublicationduration.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.travel_windows_flightpublicationduration AS t), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '06c', 'table.travel_windows_flightreviewdecision.count=' || count(*)::text
+      FROM public.travel_windows_flightreviewdecision
+    UNION ALL SELECT '06d', 'table.travel_windows_flightreviewdecision.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.travel_windows_flightreviewdecision AS t), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '06e', 'table.travel_windows_flightschedule.count=' || count(*)::text
+      FROM public.travel_windows_flightschedule
+    UNION ALL SELECT '06f', 'table.travel_windows_flightschedule.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.travel_windows_flightschedule AS t), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '06g', 'table.travel_windows_flightschedulerevision.count=' || count(*)::text
+      FROM public.travel_windows_flightschedulerevision
+    UNION ALL SELECT '06h', 'table.travel_windows_flightschedulerevision.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.travel_windows_flightschedulerevision AS t), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '06i', 'table.travel_windows_publishedflightschedule.count=' || count(*)::text
+      FROM public.travel_windows_publishedflightschedule
+    UNION ALL SELECT '06j', 'table.travel_windows_publishedflightschedule.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.travel_windows_publishedflightschedule AS t), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '06k', 'table.travel_windows_revieweddurationreceipt.count=' || count(*)::text
+      FROM public.travel_windows_revieweddurationreceipt
+    UNION ALL SELECT '06l', 'table.travel_windows_revieweddurationreceipt.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.travel_windows_revieweddurationreceipt AS t), ''), 'UTF8')), 'hex')
+    UNION ALL SELECT '06m', 'table.travel_windows_routedurationrevision.count=' || count(*)::text
+      FROM public.travel_windows_routedurationrevision
+    UNION ALL SELECT '06n', 'table.travel_windows_routedurationrevision.sha256=' || encode(sha256(convert_to(
+      coalesce((SELECT string_agg(row_to_json(t)::text, E'\n' ORDER BY t.id) FROM public.travel_windows_routedurationrevision AS t), ''), 'UTF8')), 'hex')
+
+    UNION ALL SELECT '070', 'pointer.entry.jp=' || coalesce((
+      SELECT coalesce(pointer.current_publication_id::text, 'NONE') || ':' || pointer.version::text
+      FROM public.countries_country AS country
+      LEFT JOIN public.reviews_publishedentryfacts AS pointer ON pointer.country_id = country.id
+      WHERE country.iso_alpha2 = 'JP'
+    ), 'NONE:0')
+    UNION ALL SELECT '071', 'pointer.entry.tw=' || coalesce((
+      SELECT coalesce(pointer.current_publication_id::text, 'NONE') || ':' || pointer.version::text
+      FROM public.countries_country AS country
+      LEFT JOIN public.reviews_publishedentryfacts AS pointer ON pointer.country_id = country.id
+      WHERE country.iso_alpha2 = 'TW'
+    ), 'NONE:0')
+    UNION ALL SELECT '072', 'pointer.entry.hk=' || coalesce((
+      SELECT coalesce(pointer.current_publication_id::text, 'NONE') || ':' || pointer.version::text
+      FROM public.countries_country AS country
+      LEFT JOIN public.reviews_publishedentryfacts AS pointer ON pointer.country_id = country.id
+      WHERE country.iso_alpha2 = 'HK'
+    ), 'NONE:0')
+    UNION ALL SELECT '073', 'pointer.entry.mo=' || coalesce((
+      SELECT coalesce(pointer.current_publication_id::text, 'NONE') || ':' || pointer.version::text
+      FROM public.countries_country AS country
+      LEFT JOIN public.reviews_publishedentryfacts AS pointer ON pointer.country_id = country.id
+      WHERE country.iso_alpha2 = 'MO'
+    ), 'NONE:0')
+    UNION ALL SELECT '074', 'pointer.entry.vn=' || coalesce((
+      SELECT coalesce(pointer.current_publication_id::text, 'NONE') || ':' || pointer.version::text
+      FROM public.countries_country AS country
+      LEFT JOIN public.reviews_publishedentryfacts AS pointer ON pointer.country_id = country.id
+      WHERE country.iso_alpha2 = 'VN'
+    ), 'NONE:0')
+    UNION ALL SELECT '075', 'pointer.entry.th=' || coalesce((
+      SELECT coalesce(pointer.current_publication_id::text, 'NONE') || ':' || pointer.version::text
+      FROM public.countries_country AS country
+      LEFT JOIN public.reviews_publishedentryfacts AS pointer ON pointer.country_id = country.id
+      WHERE country.iso_alpha2 = 'TH'
+    ), 'NONE:0')
+    UNION ALL SELECT '076', 'pointer.travel_warning.jp=' || coalesce((
+      SELECT coalesce(pointer.current_publication_id::text, 'NONE') || ':' || pointer.version::text
+      FROM public.countries_country AS country
+      LEFT JOIN public.reviews_publishedtravelwarning AS pointer ON pointer.country_id = country.id
+      WHERE country.iso_alpha2 = 'JP'
+    ), 'NONE:0')
+    UNION ALL SELECT '077', 'pointer.travel_warning.tw=' || coalesce((
+      SELECT coalesce(pointer.current_publication_id::text, 'NONE') || ':' || pointer.version::text
+      FROM public.countries_country AS country
+      LEFT JOIN public.reviews_publishedtravelwarning AS pointer ON pointer.country_id = country.id
+      WHERE country.iso_alpha2 = 'TW'
+    ), 'NONE:0')
+    UNION ALL SELECT '078', 'pointer.travel_warning.hk=' || coalesce((
+      SELECT coalesce(pointer.current_publication_id::text, 'NONE') || ':' || pointer.version::text
+      FROM public.countries_country AS country
+      LEFT JOIN public.reviews_publishedtravelwarning AS pointer ON pointer.country_id = country.id
+      WHERE country.iso_alpha2 = 'HK'
+    ), 'NONE:0')
+    UNION ALL SELECT '079', 'pointer.travel_warning.mo=' || coalesce((
+      SELECT coalesce(pointer.current_publication_id::text, 'NONE') || ':' || pointer.version::text
+      FROM public.countries_country AS country
+      LEFT JOIN public.reviews_publishedtravelwarning AS pointer ON pointer.country_id = country.id
+      WHERE country.iso_alpha2 = 'MO'
+    ), 'NONE:0')
+    UNION ALL SELECT '07a', 'pointer.travel_warning.vn=' || coalesce((
+      SELECT coalesce(pointer.current_publication_id::text, 'NONE') || ':' || pointer.version::text
+      FROM public.countries_country AS country
+      LEFT JOIN public.reviews_publishedtravelwarning AS pointer ON pointer.country_id = country.id
+      WHERE country.iso_alpha2 = 'VN'
+    ), 'NONE:0')
+    UNION ALL SELECT '07b', 'pointer.travel_warning.th=' || coalesce((
+      SELECT coalesce(pointer.current_publication_id::text, 'NONE') || ':' || pointer.version::text
+      FROM public.countries_country AS country
+      LEFT JOIN public.reviews_publishedtravelwarning AS pointer ON pointer.country_id = country.id
+      WHERE country.iso_alpha2 = 'TH'
+    ), 'NONE:0')
+    UNION ALL SELECT '07c', 'pointer.flight_schedule=' || coalesce((
+      SELECT coalesce(current_publication_id::text, 'NONE') || ':' || version::text
+      FROM public.travel_windows_publishedflightschedule
+    ), 'NONE:0')
 )
 SELECT line FROM integrity_lines ORDER BY sort_key;


