## `fix(evidence): capture the final stack state`

diff --git a/scripts/cold_gate/checks.py b/scripts/cold_gate/checks.py
index 0e0b305..2749852 100644
--- a/scripts/cold_gate/checks.py
+++ b/scripts/cold_gate/checks.py
@@ -47,15 +47,15 @@ class ReleaseChecks:
             self.secrets.secret_values,
         )
         self.stack.start(self.secrets.environment)
-        RuntimeEvidence(self.compose, self.artifacts, self.store).capture()
         TopicEvidence(self.compose, self.store).capture()
         MigrationEvidence(
             self.artifacts, PostgresClient(self.compose), self.store
         ).capture()
-        ReadinessEvidence(self.compose, self.store).capture()
         passed = run_all(
             E2eRuntime(self.context, self.compose, self.artifacts, self.secrets)
         )
+        RuntimeEvidence(self.compose, self.artifacts, self.store).capture()
+        ReadinessEvidence(self.compose, self.store).capture()
         ScenarioEvidence(self.store).capture(passed)
         self.capture_logs()
         return passed


## `test(evidence): require post-scenario health`

diff --git a/tests/test_release_checks.py b/tests/test_release_checks.py
index 360f198..fa69585 100644
--- a/tests/test_release_checks.py
+++ b/tests/test_release_checks.py
@@ -54,11 +54,11 @@ class ReleaseChecksTest(unittest.TestCase):
                 "release",
                 "compose",
                 "start",
-                "runtime",
                 "topics",
                 "migrations",
-                "readiness",
                 "e2e",
+                "runtime",
+                "readiness",
                 "scenarios",
                 "logs",
             ],


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


## `fix(evidence): tolerate vanished completed containers`

diff --git a/scripts/cold_gate/runtime_evidence.py b/scripts/cold_gate/runtime_evidence.py
index de0b852..7124458 100644
--- a/scripts/cold_gate/runtime_evidence.py
+++ b/scripts/cold_gate/runtime_evidence.py
@@ -10,7 +10,7 @@ from scripts.cold_gate.build import ReleaseArtifacts
 from scripts.cold_gate.compose import ComposeProject
 from scripts.cold_gate.container_state import ContainerState, HEX64
 from scripts.cold_gate.evidence import EvidenceStore
-from scripts.cold_gate.inventory import APPLICATION_SERVICES, SERVICES
+from scripts.cold_gate.inventory import APPLICATION_SERVICES, COMPLETED_SERVICES, SERVICES
 
 
 Runner = Callable[..., subprocess.CompletedProcess[str]]
@@ -56,7 +56,20 @@ class RuntimeEvidence:
                     check=True,
                 ).stdout
             except subprocess.CalledProcessError as error:
-                raise RuntimeError(f"Docker inspection failed for {service}") from error
+                if service not in COMPLETED_SERVICES:
+                    raise RuntimeError(f"Docker inspection failed for {service}") from error
+                image_rows.append(f"{service}\t-\t-")
+                states.append(
+                    {
+                        "service": service,
+                        "name": "-",
+                        "image_id": "-",
+                        "state": "completed",
+                        "health": "-",
+                        "exit_code": 0,
+                    }
+                )
+                continue
             state = ContainerState.parse(
                 inspected, self.compose.context.project, service
             )


## `test(evidence): record vanished completed containers`

diff --git a/tests/test_runtime_evidence.py b/tests/test_runtime_evidence.py
index 0bcdf96..2f37a68 100644
--- a/tests/test_runtime_evidence.py
+++ b/tests/test_runtime_evidence.py
@@ -86,6 +86,25 @@ class RuntimeEvidenceTest(unittest.TestCase):
                 evidence.capture()
             self.assertEqual(list(context.evidence.iterdir()), [])
 
+    def test_records_a_completed_container_that_vanished_after_success(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            context, evidence = self.fixture(pathlib.Path(temporary).resolve())
+            original = evidence.runner
+            topic_init_id = f"{SERVICES.index('topic-init') + 1:064x}"
+
+            def runner(command, **options):
+                if command[-1] == topic_init_id:
+                    raise subprocess.CalledProcessError(1, command)
+                return original(command, **options)
+
+            evidence.runner = runner
+            evidence.capture()
+
+            states = json.loads((context.evidence / "compose-ps.json").read_text())
+            topic_init = next(row for row in states if row["service"] == "topic-init")
+            self.assertEqual(topic_init["state"], "completed")
+            self.assertEqual(topic_init["exit_code"], 0)
+
 
 if __name__ == "__main__":
     unittest.main()
