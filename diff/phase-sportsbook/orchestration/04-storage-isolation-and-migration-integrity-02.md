## `test(evidence): reject migration history drift`

diff --git a/tests/test_migration_evidence.py b/tests/test_migration_evidence.py
new file mode 100644
index 0000000..447b53e
--- /dev/null
+++ b/tests/test_migration_evidence.py
@@ -0,0 +1,98 @@
+import pathlib
+import tempfile
+import unittest
+
+from scripts.cold_gate.build import ReleaseArtifacts
+from scripts.cold_gate.context import ColdGateContext
+from scripts.cold_gate.evidence import EvidenceStore
+from scripts.cold_gate.inventory import MIGRATION_VERSIONS
+from scripts.cold_gate.migration_evidence import HISTORY_QUERY, MigrationEvidence, flyway_checksum
+from scripts.cold_gate.redaction import EvidenceRedactor
+
+
+SHA = "0123456789abcdef0123456789abcdef01234567"
+class FakeDatabase:
+    def __init__(self, histories):
+        self.histories = histories
+        self.calls = []
+
+    def query(self, database, statement):
+        self.calls.append((database, statement))
+        return [row.copy() for row in self.histories[database]]
+
+
+class MigrationEvidenceTest(unittest.TestCase):
+    def fixture(self, root: pathlib.Path):
+        context = ColdGateContext.create(root, SHA, "00000001")
+        sources = context.runtime / "sources"
+        sources.mkdir()
+        histories = {}
+        for database, versions in MIGRATION_VERSIONS.items():
+            migration_dir = sources / database / "src/main/resources/db/migration"
+            migration_dir.mkdir(parents=True)
+            histories[database] = []
+            for rank, version in enumerate(versions, 1):
+                path = migration_dir / f"V{version}__{database}_{version}.sql"
+                path.write_text(f"CREATE TABLE {database}_{version}(id INT);\n")
+                histories[database].append(
+                    {
+                        "installed_rank": str(rank),
+                        "version": version,
+                        "script": path.name,
+                        "checksum": str(flyway_checksum(path)),
+                        "success": "true",
+                    }
+                )
+        artifacts = ReleaseArtifacts(sources, context.runtime / "m2", context.runtime / "jars",
+                                     context.runtime / "fixture.jar")
+        store = EvidenceStore(context, EvidenceRedactor(["migration-secret-value"]))
+        return context, artifacts, store, histories
+
+    def test_records_exact_source_backed_flyway_history(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            context, artifacts, store, histories = self.fixture(
+                pathlib.Path(temporary).resolve()
+            )
+            database = FakeDatabase(histories)
+
+            MigrationEvidence(artifacts, database, store).capture()
+
+            lines = (context.evidence / "migrations.tsv").read_text().splitlines()
+            self.assertEqual(
+                lines[0], "database\tinstalled_rank\tversion\tscript\tchecksum\tsuccess"
+            )
+            self.assertEqual(len(lines), 26)
+            self.assertEqual([call[0] for call in database.calls], list(MIGRATION_VERSIONS))
+            self.assertTrue(all(call[1] == HISTORY_QUERY for call in database.calls))
+            self.assertEqual(sum(line.endswith("\ttrue") for line in lines[1:]), 25)
+
+    def test_rejects_database_or_source_inventory_drift_without_evidence(self) -> None:
+        for drift in ("checksum", "source"):
+            with self.subTest(drift=drift), tempfile.TemporaryDirectory() as temporary:
+                context, artifacts, store, histories = self.fixture(
+                    pathlib.Path(temporary).resolve()
+                )
+                if drift == "checksum":
+                    histories["settlement"][0]["checksum"] = "0"
+                else:
+                    directory = artifacts.sources / "wallet/src/main/resources/db/migration"
+                    (directory / "V5__unexpected.sql").write_text("SELECT 1;\n")
+
+                with self.assertRaisesRegex(RuntimeError, "drifted|versions"):
+                    MigrationEvidence(artifacts, FakeDatabase(histories), store).capture()
+                self.assertEqual(list(context.evidence.iterdir()), [])
+
+    def test_checksum_matches_flyway_line_and_signed_integer_rules(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            root = pathlib.Path(temporary).resolve()
+            unix = root / "unix.sql"
+            windows = root / "windows.sql"
+            unix.write_bytes(b"a\n\n")
+            windows.write_bytes(b"\xef\xbb\xbfa\r\n\r\n")
+
+            self.assertEqual(flyway_checksum(unix), -390611389)
+            self.assertEqual(flyway_checksum(windows), -390611389)
+
+
+if __name__ == "__main__":
+    unittest.main()


## `fix(evidence): sort migration ranks numerically`

diff --git a/scripts/cold_gate/migration_evidence.py b/scripts/cold_gate/migration_evidence.py
index 2bd094a..f5e64ea 100644
--- a/scripts/cold_gate/migration_evidence.py
+++ b/scripts/cold_gate/migration_evidence.py
@@ -15,7 +15,7 @@ MIGRATION_NAME = re.compile(r"^V([1-9][0-9]*)__([A-Za-z0-9_]+)\.sql$")
 HISTORY_QUERY = (
     "SELECT installed_rank::text AS installed_rank, version, script, "
     "checksum::text AS checksum, success::text AS success "
-    "FROM flyway_schema_history ORDER BY installed_rank"
+    "FROM flyway_schema_history ORDER BY flyway_schema_history.installed_rank"
 )
 
 


## `test(evidence): require numeric migration ordering`

diff --git a/tests/test_migration_evidence.py b/tests/test_migration_evidence.py
index 447b53e..fe0653e 100644
--- a/tests/test_migration_evidence.py
+++ b/tests/test_migration_evidence.py
@@ -64,6 +64,7 @@ class MigrationEvidenceTest(unittest.TestCase):
             self.assertEqual(len(lines), 26)
             self.assertEqual([call[0] for call in database.calls], list(MIGRATION_VERSIONS))
             self.assertTrue(all(call[1] == HISTORY_QUERY for call in database.calls))
+            self.assertIn("ORDER BY flyway_schema_history.installed_rank", HISTORY_QUERY)
             self.assertEqual(sum(line.endswith("\ttrue") for line in lines[1:]), 25)
 
     def test_rejects_database_or_source_inventory_drift_without_evidence(self) -> None:
