## `test(gate): verify single cold startup boundary`

diff --git a/tests/test_cold_stack.py b/tests/test_cold_stack.py
new file mode 100644
index 0000000..bda58ce
--- /dev/null
+++ b/tests/test_cold_stack.py
@@ -0,0 +1,81 @@
+import pathlib
+import subprocess
+import tempfile
+import unittest
+
+from scripts.cold_gate.context import ColdGateContext
+from scripts.cold_gate.stack import ColdStack
+
+
+SHA = "0123456789abcdef0123456789abcdef01234567"
+
+
+class FakeCompose:
+    def __init__(self, context: ColdGateContext, failure: bool = False) -> None:
+        self.context = context
+        self.failure = failure
+        self.calls = []
+        self.absence_checks = 0
+
+    def require_absent(self):
+        self.absence_checks += 1
+
+    def run(self, *arguments, **options):
+        self.calls.append((arguments, options))
+        if self.failure:
+            raise subprocess.CalledProcessError(1, arguments, stderr="secret")
+        if arguments[0] == "port":
+            output = "127.0.0.1:54321\n"
+        elif arguments[0] == "logs":
+            output = "bounded log\n"
+        else:
+            output = ""
+        return subprocess.CompletedProcess(arguments, 0, stdout=output)
+
+
+class ColdStackTest(unittest.TestCase):
+    def context(self, root: pathlib.Path) -> ColdGateContext:
+        return ColdGateContext.create(root, SHA, "00000001")
+
+    def test_validates_then_starts_the_full_stack_once(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            context = self.context(pathlib.Path(temporary).resolve())
+            compose = FakeCompose(context)
+            environment = {"COMPOSE_PROJECT_NAME": context.project}
+
+            ColdStack(context, compose).start(environment)
+
+            self.assertEqual(compose.absence_checks, 1)
+            self.assertEqual(compose.calls[0][0], ("config", "--quiet"))
+            self.assertEqual(
+                compose.calls[1][0],
+                ("up", "--detach", "--build", "--wait", "--wait-timeout", "900"),
+            )
+            self.assertIs(compose.calls[1][1]["environment"], environment)
+
+    def test_discovers_only_dynamic_loopback_ports_and_bounded_logs(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            context = self.context(pathlib.Path(temporary).resolve())
+            compose = FakeCompose(context)
+            stack = ColdStack(context, compose)
+
+            self.assertEqual(stack.loopback_port("gateway", 8080), 54321)
+            self.assertEqual(stack.logs("wallet"), "bounded log\n")
+            with self.assertRaises(ValueError):
+                stack.loopback_port("postgres", 5432)
+            with self.assertRaises(ValueError):
+                stack.logs("unknown")
+
+    def test_rejects_wrong_ownership_and_hides_startup_output(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            context = self.context(pathlib.Path(temporary).resolve())
+            compose = FakeCompose(context, failure=True)
+            with self.assertRaisesRegex(RuntimeError, "environment"):
+                ColdStack(context, compose).start({"COMPOSE_PROJECT_NAME": "wrong"})
+            with self.assertRaisesRegex(RuntimeError, "startup failed") as captured:
+                ColdStack(context, compose).start({"COMPOSE_PROJECT_NAME": context.project})
+            self.assertNotIn("secret", str(captured.exception))
+
+
+if __name__ == "__main__":
+    unittest.main()


## `build(gate): run semantic release checks`

diff --git a/scripts/cold_gate/checks.py b/scripts/cold_gate/checks.py
new file mode 100644
index 0000000..0e0b305
--- /dev/null
+++ b/scripts/cold_gate/checks.py
@@ -0,0 +1,67 @@
+from __future__ import annotations
+
+from e2e.runtime import E2eRuntime
+from e2e.scenarios import run_all
+from scripts.cold_gate.build import ReleaseArtifacts
+from scripts.cold_gate.compose import ComposeProject
+from scripts.cold_gate.compose_evidence import capture_compose_config
+from scripts.cold_gate.context import ColdGateContext
+from scripts.cold_gate.database import PostgresClient
+from scripts.cold_gate.evidence import EvidenceStore
+from scripts.cold_gate.log_evidence import LogEvidence
+from scripts.cold_gate.migration_evidence import MigrationEvidence
+from scripts.cold_gate.readiness_evidence import ReadinessEvidence
+from scripts.cold_gate.release_evidence import ReleaseEvidence
+from scripts.cold_gate.runtime_evidence import RuntimeEvidence
+from scripts.cold_gate.scenario_evidence import ScenarioEvidence
+from scripts.cold_gate.secrets import RuntimeSecrets
+from scripts.cold_gate.stack import ColdStack
+from scripts.cold_gate.topic_evidence import TopicEvidence
+
+
+class ReleaseChecks:
+    def __init__(
+        self,
+        context: ColdGateContext,
+        compose: ComposeProject,
+        artifacts: ReleaseArtifacts,
+        secrets: RuntimeSecrets,
+        store: EvidenceStore,
+    ) -> None:
+        if compose.context is not context or store.context is not context:
+            raise RuntimeError("release checks ownership mismatch")
+        self.context = context
+        self.compose = compose
+        self.artifacts = artifacts
+        self.secrets = secrets
+        self.store = store
+        self.stack = ColdStack(context, compose)
+        self.logs_captured = False
+
+    def run(self, commit: str) -> tuple[str, ...]:
+        ReleaseEvidence(self.context, self.artifacts, self.store).capture(commit)
+        capture_compose_config(
+            self.compose,
+            self.store,
+            self.secrets.environment,
+            self.secrets.secret_values,
+        )
+        self.stack.start(self.secrets.environment)
+        RuntimeEvidence(self.compose, self.artifacts, self.store).capture()
+        TopicEvidence(self.compose, self.store).capture()
+        MigrationEvidence(
+            self.artifacts, PostgresClient(self.compose), self.store
+        ).capture()
+        ReadinessEvidence(self.compose, self.store).capture()
+        passed = run_all(
+            E2eRuntime(self.context, self.compose, self.artifacts, self.secrets)
+        )
+        ScenarioEvidence(self.store).capture(passed)
+        self.capture_logs()
+        return passed
+
+    def capture_logs(self) -> None:
+        if self.logs_captured:
+            return
+        LogEvidence(self.stack, self.store).capture()
+        self.logs_captured = True


## `build(gate): own cold release lifecycle`

diff --git a/scripts/cold_gate/gate.py b/scripts/cold_gate/gate.py
new file mode 100644
index 0000000..e558305
--- /dev/null
+++ b/scripts/cold_gate/gate.py
@@ -0,0 +1,48 @@
+from __future__ import annotations
+
+from pathlib import Path
+
+from scripts.cold_gate.build import ReleaseBuilder
+from scripts.cold_gate.checks import ReleaseChecks
+from scripts.cold_gate.cleanup import ScopedCleanup
+from scripts.cold_gate.cleanup_evidence import CleanupEvidence
+from scripts.cold_gate.cleanup_targets import discover_cleanup_targets
+from scripts.cold_gate.compose import ComposeProject
+from scripts.cold_gate.context import ColdGateContext
+from scripts.cold_gate.evidence import EvidenceStore
+from scripts.cold_gate.redaction import EvidenceRedactor
+from scripts.cold_gate.secrets import RuntimeSecrets
+
+
+def run_release_gate(root: Path, commit: str) -> Path:
+    context = ColdGateContext.create(root, commit)
+    compose = None
+    checks = None
+    try:
+        compose = ComposeProject(context)
+        secrets = RuntimeSecrets.generate(context)
+        store = EvidenceStore(context, EvidenceRedactor(secrets.secret_values))
+        artifacts = ReleaseBuilder(context, secrets.environment).build()
+        checks = ReleaseChecks(context, compose, artifacts, secrets, store)
+        checks.run(commit)
+        cleanup_evidence = CleanupEvidence(context, store)
+        ScopedCleanup(context, compose).run(
+            artifacts.sources, artifacts.service_jars, cleanup_evidence
+        )
+    except Exception as error:
+        failures = [error]
+        if checks is not None:
+            try:
+                checks.capture_logs()
+            except Exception as log_error:
+                failures.append(log_error)
+        if compose is not None:
+            try:
+                sources, service_jars = discover_cleanup_targets(context)
+                ScopedCleanup(context, compose).run(sources, service_jars)
+            except Exception as cleanup_error:
+                failures.append(cleanup_error)
+        if len(failures) > 1:
+            raise ExceptionGroup("cold release gate and cleanup failed", failures)
+        raise
+    return context.evidence


## `build(gate): expose the cold release command`

diff --git a/scripts/cold_release_gate.py b/scripts/cold_release_gate.py
new file mode 100755
index 0000000..57397ee
--- /dev/null
+++ b/scripts/cold_release_gate.py
@@ -0,0 +1,36 @@
+#!/usr/bin/env python3
+from __future__ import annotations
+
+import subprocess
+from pathlib import Path
+
+from scripts.cold_gate.gate import run_release_gate
+from scripts.cold_gate.release_identity import SHA
+
+
+def clean_head(root: Path) -> str:
+    head = subprocess.run(
+        ["git", "-C", str(root), "rev-parse", "--verify", "HEAD"],
+        text=True,
+        capture_output=True,
+        check=True,
+    ).stdout.strip()
+    dirty = subprocess.run(
+        ["git", "-C", str(root), "status", "--porcelain", "--untracked-files=no"],
+        text=True,
+        capture_output=True,
+        check=True,
+    ).stdout
+    if SHA.fullmatch(head) is None or dirty:
+        raise RuntimeError("cold release gate requires one clean exact HEAD")
+    return head
+
+
+def main() -> None:
+    root = Path(__file__).resolve().parents[1]
+    evidence = run_release_gate(root, clean_head(root))
+    print(f"cold_gate_evidence={evidence}")
+
+
+if __name__ == "__main__":
+    main()


## `test(gate): reject dirty release invocations`

diff --git a/tests/test_cold_release_command.py b/tests/test_cold_release_command.py
new file mode 100644
index 0000000..e8b03db
--- /dev/null
+++ b/tests/test_cold_release_command.py
@@ -0,0 +1,56 @@
+import io
+import pathlib
+import subprocess
+import unittest
+from contextlib import redirect_stdout
+from unittest import mock
+
+import scripts.cold_release_gate as subject
+
+
+SHA = "0123456789abcdef0123456789abcdef01234567"
+
+
+class ColdReleaseCommandTest(unittest.TestCase):
+    def test_reads_one_clean_exact_head(self):
+        runner = mock.Mock(
+            side_effect=(
+                subprocess.CompletedProcess([], 0, stdout=SHA + "\n"),
+                subprocess.CompletedProcess([], 0, stdout=""),
+            )
+        )
+        with mock.patch.object(subject.subprocess, "run", runner):
+            observed = subject.clean_head(pathlib.Path("/release"))
+
+        self.assertEqual(observed, SHA)
+        self.assertEqual(runner.call_count, 2)
+        self.assertEqual(runner.call_args_list[1].args[0][-2:], ["--porcelain", "--untracked-files=no"])
+
+    def test_rejects_dirty_or_malformed_checkout(self):
+        for head, dirty in ((SHA, " M compose.yaml\n"), ("short", "")):
+            with self.subTest(head=head), mock.patch.object(
+                subject.subprocess,
+                "run",
+                side_effect=(
+                    subprocess.CompletedProcess([], 0, stdout=head + "\n"),
+                    subprocess.CompletedProcess([], 0, stdout=dirty),
+                ),
+            ):
+                with self.assertRaisesRegex(RuntimeError, "clean exact HEAD"):
+                    subject.clean_head(pathlib.Path("/release"))
+
+    def test_runs_gate_for_script_root_and_prints_evidence_path(self):
+        output = io.StringIO()
+        with mock.patch.object(subject, "clean_head", return_value=SHA) as head, mock.patch.object(
+            subject, "run_release_gate", return_value=pathlib.Path("/evidence/run")
+        ) as gate, redirect_stdout(output):
+            subject.main()
+
+        root = pathlib.Path(subject.__file__).resolve().parents[1]
+        head.assert_called_once_with(root)
+        gate.assert_called_once_with(root, SHA)
+        self.assertEqual(output.getvalue(), "cold_gate_evidence=/evidence/run\n")
+
+
+if __name__ == "__main__":
+    unittest.main()


## `fix(gate): fail closed during scoped cleanup`

diff --git a/scripts/cold_gate/cleanup.py b/scripts/cold_gate/cleanup.py
index 52bc04b..c81608b 100644
--- a/scripts/cold_gate/cleanup.py
+++ b/scripts/cold_gate/cleanup.py
@@ -21,6 +21,8 @@ class ScopedCleanup:
     ) -> None:
         self.context = context
         self.compose = compose
+        if compose.context is not context:
+            raise RuntimeError("cleanup Compose project has different ownership")
         self.runner = runner
 
     def run(self, sources: Path | None = None) -> None:
@@ -53,6 +55,8 @@ class ScopedCleanup:
                 capture_output=True,
                 check=True,
             )
+            if sources.exists() or sources.is_symlink():
+                raise RuntimeError("materialized sources remain after cleanup")
 
         self.context.require_owned()
         shutil.rmtree(self.context.runtime)


## `build(cleanup): attest scoped resource removal`

diff --git a/scripts/cold_gate/cleanup_evidence.py b/scripts/cold_gate/cleanup_evidence.py
new file mode 100644
index 0000000..3de7086
--- /dev/null
+++ b/scripts/cold_gate/cleanup_evidence.py
@@ -0,0 +1,63 @@
+from __future__ import annotations
+
+import subprocess
+from collections.abc import Callable
+from pathlib import Path
+
+from scripts.cold_gate.context import ColdGateContext
+from scripts.cold_gate.evidence import EvidenceStore
+
+
+Runner = Callable[..., subprocess.CompletedProcess[str]]
+
+
+class CleanupEvidence:
+    def __init__(
+        self,
+        context: ColdGateContext,
+        store: EvidenceStore,
+        runner: Runner = subprocess.run,
+    ) -> None:
+        if store.context is not context:
+            raise RuntimeError("cleanup evidence ownership mismatch")
+        self.context = context
+        self.store = store
+        self.runner = runner
+
+    def capture(self, sources: Path, service_jars: Path) -> None:
+        self.context.require_owned()
+        expected_sources = self.context.runtime / "sources"
+        expected_jars = self.context.root / "docker/.jars" / service_jars.name
+        if sources != expected_sources or service_jars != expected_jars:
+            raise RuntimeError("cleanup evidence targets are not owned")
+        label = f"label=com.docker.compose.project={self.context.project}"
+        commands = (
+            ("containers", ["docker", "ps", "--all", "--quiet", "--filter", label]),
+            ("networks", ["docker", "network", "ls", "--quiet", "--filter", label]),
+            ("volumes", ["docker", "volume", "ls", "--quiet", "--filter", label]),
+        )
+        rows = ["resource\tremaining"]
+        for resource, command in commands:
+            try:
+                output = self.runner(
+                    command,
+                    cwd=self.context.root,
+                    text=True,
+                    capture_output=True,
+                    check=True,
+                ).stdout
+            except subprocess.CalledProcessError as error:
+                raise RuntimeError(f"cleanup query failed for {resource}") from error
+            if output.strip():
+                raise RuntimeError(f"cleanup left scoped {resource}")
+            rows.append(f"{resource}\t0")
+        paths = (
+            ("sources", sources),
+            ("jar-link", self.context.root / "docker/jars"),
+            ("jar-generation", service_jars),
+        )
+        for resource, path in paths:
+            if path.exists() or path.is_symlink():
+                raise RuntimeError(f"cleanup left {resource}")
+            rows.append(f"{resource}\t0")
+        self.store.write("cleanup.tsv", "\n".join(rows) + "\n")


## `build(cleanup): emit removal evidence before unlock`

diff --git a/scripts/cold_gate/cleanup.py b/scripts/cold_gate/cleanup.py
index 7c9252a..40cdc9f 100644
--- a/scripts/cold_gate/cleanup.py
+++ b/scripts/cold_gate/cleanup.py
@@ -6,6 +6,7 @@ import subprocess
 from collections.abc import Callable
 from pathlib import Path
 
+from scripts.cold_gate.cleanup_evidence import CleanupEvidence
 from scripts.cold_gate.compose import ComposeProject
 from scripts.cold_gate.context import ColdGateContext
 from scripts.cold_gate.owned_path import require_directory
@@ -27,8 +28,17 @@ class ScopedCleanup:
             raise RuntimeError("cleanup Compose project has different ownership")
         self.runner = runner
 
-    def run(self, sources: Path | None = None, service_jars: Path | None = None) -> None:
+    def run(
+        self,
+        sources: Path | None = None,
+        service_jars: Path | None = None,
+        evidence: CleanupEvidence | None = None,
+    ) -> None:
         self.context.require_owned()
+        if evidence is not None and (
+            evidence.context is not self.context or sources is None or service_jars is None
+        ):
+            raise RuntimeError("cleanup evidence targets have different ownership")
         if sources is not None:
             expected = self.context.runtime / "sources"
             if sources != expected or sources.is_symlink() or not sources.is_dir():
@@ -74,6 +84,8 @@ class ScopedCleanup:
             jars_link.unlink()
             shutil.rmtree(service_jars)
             service_jars.parent.rmdir()
+        if evidence is not None:
+            evidence.capture(sources, service_jars)
 
         self.context.require_owned()
         shutil.rmtree(self.context.runtime)


## `build(cleanup): verify evidence before unlock`

diff --git a/scripts/cold_gate/cleanup.py b/scripts/cold_gate/cleanup.py
index 40cdc9f..c59574e 100644
--- a/scripts/cold_gate/cleanup.py
+++ b/scripts/cold_gate/cleanup.py
@@ -86,6 +86,7 @@ class ScopedCleanup:
             service_jars.parent.rmdir()
         if evidence is not None:
             evidence.capture(sources, service_jars)
+            evidence.store.verify(complete=True)
 
         self.context.require_owned()
         shutil.rmtree(self.context.runtime)


## `build(cleanup): discover owned partial targets`

diff --git a/scripts/cold_gate/cleanup_targets.py b/scripts/cold_gate/cleanup_targets.py
new file mode 100644
index 0000000..ca2d5f2
--- /dev/null
+++ b/scripts/cold_gate/cleanup_targets.py
@@ -0,0 +1,40 @@
+from __future__ import annotations
+
+import os
+from pathlib import Path, PurePosixPath
+
+from scripts.cold_gate.context import ColdGateContext
+from scripts.cold_gate.owned_path import require_directory
+
+
+def discover_cleanup_targets(
+    context: ColdGateContext,
+) -> tuple[Path | None, Path | None]:
+    context.require_owned()
+    sources = context.runtime / "sources"
+    if sources.is_symlink() or (sources.exists() and not sources.is_dir()):
+        raise RuntimeError("materialized source target is not an owned directory")
+    source_target = sources if sources.is_dir() else None
+
+    jars_parent = context.root / "docker/.jars"
+    jars_link = context.root / "docker/jars"
+    if jars_link.exists() and not jars_link.is_symlink():
+        raise RuntimeError("release JAR link is not an owned symlink")
+    if jars_link.is_symlink():
+        relative = PurePosixPath(os.readlink(jars_link))
+        if (
+            relative.is_absolute()
+            or len(relative.parts) != 2
+            or relative.parts[0] != ".jars"
+            or relative.parts[1] in ("", ".", "..")
+        ):
+            raise RuntimeError("release JAR link escaped its owned generation")
+        service_jars = jars_parent / relative.parts[1]
+        require_directory(service_jars)
+        generations = list(jars_parent.iterdir())
+        if generations != [service_jars]:
+            raise RuntimeError("release JAR generation inventory is ambiguous")
+        return source_target, service_jars
+    if jars_parent.exists() or jars_parent.is_symlink():
+        raise RuntimeError("release JAR generation exists without its owned link")
+    return source_target, None


