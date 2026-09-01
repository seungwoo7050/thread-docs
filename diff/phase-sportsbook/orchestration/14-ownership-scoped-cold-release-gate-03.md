## `test(cleanup): reject ambiguous partial targets`

diff --git a/tests/test_cleanup_targets.py b/tests/test_cleanup_targets.py
new file mode 100644
index 0000000..4576716
--- /dev/null
+++ b/tests/test_cleanup_targets.py
@@ -0,0 +1,68 @@
+import pathlib
+import tempfile
+import unittest
+
+from scripts.cold_gate.cleanup_targets import discover_cleanup_targets
+from scripts.cold_gate.context import ColdGateContext
+
+
+SHA = "0" * 40
+
+
+class CleanupTargetsTest(unittest.TestCase):
+    def context(self, root):
+        return ColdGateContext.create(root, SHA, "00000001")
+
+    def test_discovers_absent_or_exact_owned_targets(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            root = pathlib.Path(temporary).resolve()
+            context = self.context(root)
+            self.assertEqual(discover_cleanup_targets(context), (None, None))
+
+            sources = context.runtime / "sources"
+            sources.mkdir()
+            jars = root / "docker/.jars/generation.release"
+            jars.mkdir(parents=True)
+            (root / "docker/jars").symlink_to(".jars/generation.release")
+
+            self.assertEqual(discover_cleanup_targets(context), (sources, jars))
+
+    def test_rejects_escaped_link_and_ambiguous_generations(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            root = pathlib.Path(temporary).resolve()
+            context = self.context(root)
+            docker = root / "docker"
+            docker.mkdir()
+            foreign = root / "foreign"
+            foreign.mkdir()
+            (docker / "jars").symlink_to(foreign, target_is_directory=True)
+            with self.assertRaisesRegex(RuntimeError, "escaped"):
+                discover_cleanup_targets(context)
+
+            (docker / "jars").unlink()
+            first = docker / ".jars/first"
+            second = docker / ".jars/second"
+            first.mkdir(parents=True)
+            second.mkdir()
+            (docker / "jars").symlink_to(".jars/first")
+            with self.assertRaisesRegex(RuntimeError, "ambiguous"):
+                discover_cleanup_targets(context)
+
+    def test_rejects_symlinked_sources_and_orphan_generations(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            root = pathlib.Path(temporary).resolve()
+            context = self.context(root)
+            foreign = root / "foreign"
+            foreign.mkdir()
+            (context.runtime / "sources").symlink_to(foreign, target_is_directory=True)
+            with self.assertRaisesRegex(RuntimeError, "source target"):
+                discover_cleanup_targets(context)
+
+            (context.runtime / "sources").unlink()
+            (root / "docker/.jars/orphan").mkdir(parents=True)
+            with self.assertRaisesRegex(RuntimeError, "without its owned link"):
+                discover_cleanup_targets(context)
+
+
+if __name__ == "__main__":
+    unittest.main()


## `fix(cleanup): handle interrupted release gates`

diff --git a/scripts/cold_gate/gate.py b/scripts/cold_gate/gate.py
index 96ecaac..6c9d033 100644
--- a/scripts/cold_gate/gate.py
+++ b/scripts/cold_gate/gate.py
@@ -32,25 +32,25 @@ def run_release_gate(root: Path, commit: str) -> Path:
         ScopedCleanup(context, compose).run(
             artifacts.sources, artifacts.service_jars, cleanup_evidence
         )
-    except Exception as error:
+    except BaseException as error:
         failures = [error]
         if checks is not None:
             try:
                 checks.capture_logs()
-            except Exception as log_error:
+            except BaseException as log_error:
                 failures.append(log_error)
         if compose_bound:
             try:
                 sources, service_jars = discover_cleanup_targets(context)
                 ScopedCleanup(context, compose).run(sources, service_jars)
-            except Exception as cleanup_error:
+            except BaseException as cleanup_error:
                 failures.append(cleanup_error)
         else:
             try:
                 remove_owned_context(context)
-            except Exception as cleanup_error:
+            except BaseException as cleanup_error:
                 failures.append(cleanup_error)
         if len(failures) > 1:
-            raise ExceptionGroup("cold release gate and cleanup failed", failures)
+            raise BaseExceptionGroup("cold release gate and cleanup failed", failures)
         raise
     return context.evidence


## `test(cleanup): recover from gate interrupts`

diff --git a/tests/test_release_gate.py b/tests/test_release_gate.py
index 5277648..0b2076e 100644
--- a/tests/test_release_gate.py
+++ b/tests/test_release_gate.py
@@ -85,6 +85,18 @@ class ReleaseGateTest(unittest.TestCase):
             ["primary", "logs", "cleanup"],
         )
 
+    def test_interrupt_captures_logs_cleans_and_rethrows(self):
+        values = self.fixture()
+        _context, _compose, _secrets, artifacts, _store, checks, cleanup, replacements = values
+        checks.run.side_effect = KeyboardInterrupt()
+
+        with mock.patch.multiple(subject, **replacements):
+            with self.assertRaises(KeyboardInterrupt):
+                subject.run_release_gate(pathlib.Path("/release"), COMMIT)
+
+        checks.capture_logs.assert_called_once_with()
+        cleanup.run.assert_called_once_with(artifacts.sources, artifacts.service_jars)
+
     def test_releases_context_when_compose_or_secret_preflight_fails(self):
         for stage in ("ComposeProject", "RuntimeSecrets"):
             with self.subTest(stage=stage):


## `ci(archive): verify the cold release once`

diff --git a/.github/workflows/archive.yml b/.github/workflows/archive.yml
new file mode 100644
index 0000000..8f58244
--- /dev/null
+++ b/.github/workflows/archive.yml
@@ -0,0 +1,54 @@
+name: Orchestration archive verification
+
+on:
+  workflow_dispatch:
+  push:
+    branches: [orchestration]
+
+permissions:
+  contents: read
+
+concurrency:
+  group: orchestration-archive-${{ github.ref }}
+  cancel-in-progress: false
+
+jobs:
+  verify:
+    runs-on: ubuntu-24.04
+    timeout-minutes: 120
+    steps:
+      - name: Check out the exact archive revision
+        uses: actions/checkout@v4
+        with:
+          ref: ${{ github.sha }}
+          fetch-depth: 0
+          persist-credentials: false
+
+      - name: Select Java 17
+        uses: actions/setup-java@v4
+        with:
+          distribution: temurin
+          java-version: "17"
+
+      - name: Select Python 3.12
+        uses: actions/setup-python@v5
+        with:
+          python-version: "3.12"
+
+      - name: Verify the archive history
+        run: python3 scripts/history_guard.py
+
+      - name: Verify static contracts
+        run: python3 -m unittest discover -s tests
+
+      - name: Run the one cold release gate
+        run: python3 scripts/cold_release_gate.py
+
+      - name: Upload redacted release evidence
+        if: always()
+        uses: actions/upload-artifact@v4
+        with:
+          name: orchestration-evidence-${{ github.sha }}
+          path: evidence/cold-gate/**
+          if-no-files-found: warn
+          retention-days: 14


## `test(archive): lock the single release workflow`

diff --git a/tests/test_archive_workflow.py b/tests/test_archive_workflow.py
new file mode 100644
index 0000000..65e5b73
--- /dev/null
+++ b/tests/test_archive_workflow.py
@@ -0,0 +1,52 @@
+import pathlib
+import unittest
+
+
+ROOT = pathlib.Path(__file__).resolve().parents[1]
+WORKFLOW = ROOT / ".github/workflows/archive.yml"
+
+
+class ArchiveWorkflowTest(unittest.TestCase):
+    def test_owns_one_exact_sha_archive_workflow(self):
+        workflows = sorted(path.name for path in WORKFLOW.parent.glob("*.y*ml"))
+        text = WORKFLOW.read_text()
+
+        self.assertEqual(workflows, ["archive.yml"])
+        required = (
+            "branches: [orchestration]",
+            "permissions:\n  contents: read",
+            "uses: actions/checkout@v4",
+            "ref: ${{ github.sha }}",
+            "fetch-depth: 0",
+            "persist-credentials: false",
+            "uses: actions/setup-java@v4",
+            "distribution: temurin",
+            'java-version: "17"',
+            "uses: actions/setup-python@v5",
+            'python-version: "3.12"',
+            "if: always()",
+            "uses: actions/upload-artifact@v4",
+            "path: evidence/cold-gate/**",
+        )
+        for value in required:
+            with self.subTest(value=value):
+                self.assertIn(value, text)
+        self.assertNotIn("repository:", text)
+        self.assertNotIn("load-workflow", text)
+
+    def test_runs_static_guards_before_one_cold_stack_gate(self):
+        text = WORKFLOW.read_text()
+        commands = (
+            "python3 scripts/history_guard.py",
+            "python3 -m unittest discover -s tests",
+            "python3 scripts/cold_release_gate.py",
+        )
+        for command in commands:
+            self.assertEqual(text.count(command), 1)
+        positions = tuple(text.index(command) for command in commands)
+        self.assertEqual(positions, tuple(sorted(positions)))
+        self.assertEqual(text.count("Run the one cold release gate"), 1)
+
+
+if __name__ == "__main__":
+    unittest.main()


## `fix(gate): bootstrap direct archive commands`

diff --git a/scripts/cold_release_gate.py b/scripts/cold_release_gate.py
index 57397ee..f376145 100755
--- a/scripts/cold_release_gate.py
+++ b/scripts/cold_release_gate.py
@@ -2,8 +2,13 @@
 from __future__ import annotations
 
 import subprocess
+import sys
 from pathlib import Path
 
+ROOT = Path(__file__).resolve().parents[1]
+if str(ROOT) not in sys.path:
+    sys.path.insert(0, str(ROOT))
+
 from scripts.cold_gate.gate import run_release_gate
 from scripts.cold_gate.release_identity import SHA
 
@@ -27,8 +32,7 @@ def clean_head(root: Path) -> str:
 
 
 def main() -> None:
-    root = Path(__file__).resolve().parents[1]
-    evidence = run_release_gate(root, clean_head(root))
+    evidence = run_release_gate(ROOT, clean_head(ROOT))
     print(f"cold_gate_evidence={evidence}")
 
 
diff --git a/scripts/history_guard.py b/scripts/history_guard.py
index d5e5363..3a04818 100755
--- a/scripts/history_guard.py
+++ b/scripts/history_guard.py
@@ -1,15 +1,19 @@
 #!/usr/bin/env python3
 from __future__ import annotations
 
+import sys
 from pathlib import Path
 
+ROOT = Path(__file__).resolve().parents[1]
+if str(ROOT) not in sys.path:
+    sys.path.insert(0, str(ROOT))
+
 from scripts.cold_gate.history_policy import verify_history
 from scripts.cold_gate.history_repository import load_history
 
 
 def main() -> None:
-    root = Path(__file__).resolve().parents[1]
-    commits = load_history(root)
+    commits = load_history(ROOT)
     verify_history(commits)
     print(f"history_guard_commits={len(commits)}")
 
