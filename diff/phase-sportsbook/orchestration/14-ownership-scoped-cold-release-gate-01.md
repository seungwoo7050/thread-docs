# 소유권 기반 콜드 릴리스 게이트

## `build(gate): create unique cold project context`

diff --git a/scripts/cold_gate/context.py b/scripts/cold_gate/context.py
new file mode 100644
index 0000000..c9177aa
--- /dev/null
+++ b/scripts/cold_gate/context.py
@@ -0,0 +1,70 @@
+from __future__ import annotations
+
+import dataclasses
+import re
+import secrets
+from pathlib import Path
+
+
+PROJECT_PATTERN = re.compile(r"^sb-gate-[0-9a-f]{12}-[0-9a-f]{8}$")
+
+
+@dataclasses.dataclass(frozen=True)
+class ColdGateContext:
+    root: Path
+    project: str
+    runtime: Path
+    evidence: Path
+    lock: Path
+
+    @classmethod
+    def create(
+        cls, root: Path, commit: str, nonce: str | None = None
+    ) -> "ColdGateContext":
+        root = root.resolve(strict=True)
+        if not re.fullmatch(r"[0-9a-f]{40}", commit):
+            raise ValueError("commit must be a full lowercase SHA")
+        nonce = nonce or secrets.token_hex(4)
+        if not re.fullmatch(r"[0-9a-f]{8}", nonce):
+            raise ValueError("nonce must be eight lowercase hex characters")
+        project = f"sb-gate-{commit[:12]}-{nonce}"
+        if not PROJECT_PATTERN.fullmatch(project):
+            raise ValueError("generated project name is invalid")
+
+        runtime_root = root / ".runtime"
+        runtime_root.mkdir(exist_ok=True)
+        lock = runtime_root / "cold-gate.lock"
+        try:
+            lock.mkdir()
+        except FileExistsError as exception:
+            raise RuntimeError("another cold gate owns this worktree") from exception
+
+        runtime = runtime_root / "cold-gate" / project
+        evidence = root / "evidence" / "cold-gate" / project
+        try:
+            if runtime.exists() or evidence.exists():
+                raise RuntimeError("cold gate run path already exists")
+            runtime.mkdir(parents=True)
+            evidence.mkdir(parents=True)
+            marker = f"project={project}\nroot={root}\n"
+            (runtime / ".owner").write_text(marker)
+            (lock / "owner").write_text(marker)
+        except BaseException:
+            lock.rmdir()
+            raise
+        return cls(root, project, runtime, evidence, lock)
+
+    def require_owned(self) -> None:
+        if not PROJECT_PATTERN.fullmatch(self.project):
+            raise RuntimeError("invalid cold gate project")
+        expected = f"project={self.project}\nroot={self.root}\n"
+        if (self.runtime / ".owner").read_text() != expected:
+            raise RuntimeError("runtime ownership marker mismatch")
+        if (self.lock / "owner").read_text() != expected:
+            raise RuntimeError("lock ownership marker mismatch")
+        for path, parent in (
+            (self.runtime, self.root / ".runtime" / "cold-gate"),
+            (self.evidence, self.root / "evidence" / "cold-gate"),
+        ):
+            if path.is_symlink() or path.parent.resolve() != parent.resolve():
+                raise RuntimeError("cold gate path escaped its owned parent")


## `build(gate): scope every Compose command`

diff --git a/scripts/cold_gate/compose.py b/scripts/cold_gate/compose.py
new file mode 100644
index 0000000..20d8876
--- /dev/null
+++ b/scripts/cold_gate/compose.py
@@ -0,0 +1,74 @@
+from __future__ import annotations
+
+import os
+import subprocess
+from collections.abc import Callable, Iterable
+from pathlib import Path
+
+from scripts.cold_gate.context import ColdGateContext
+
+
+Runner = Callable[..., subprocess.CompletedProcess[str]]
+
+
+class ComposeProject:
+    def __init__(
+        self,
+        context: ColdGateContext,
+        overlays: Iterable[Path] = (),
+        runner: Runner = subprocess.run,
+    ) -> None:
+        self.context = context
+        self.files = (context.root / "compose.yaml", *overlays)
+        if not all(path.is_file() for path in self.files):
+            raise RuntimeError("a tracked Compose input is missing")
+        self.runner = runner
+
+    def command(self, *arguments: str) -> list[str]:
+        command = [
+            "docker",
+            "compose",
+            "--project-name",
+            self.context.project,
+            "--project-directory",
+            str(self.context.root),
+        ]
+        for path in self.files:
+            command.extend(("--file", str(path)))
+        return [*command, *arguments]
+
+    def run(
+        self,
+        *arguments: str,
+        environment: dict[str, str] | None = None,
+        check: bool = True,
+        capture_output: bool = False,
+    ) -> subprocess.CompletedProcess[str]:
+        self.context.require_owned()
+        return self.runner(
+            self.command(*arguments),
+            cwd=self.context.root,
+            env=environment or os.environ.copy(),
+            text=True,
+            capture_output=capture_output,
+            check=check,
+        )
+
+    def require_absent(self) -> None:
+        self.context.require_owned()
+        label = f"label=com.docker.compose.project={self.context.project}"
+        commands = (
+            ["docker", "ps", "--all", "--quiet", "--filter", label],
+            ["docker", "network", "ls", "--quiet", "--filter", label],
+            ["docker", "volume", "ls", "--quiet", "--filter", label],
+        )
+        for command in commands:
+            result = self.runner(
+                command,
+                cwd=self.context.root,
+                text=True,
+                capture_output=True,
+                check=True,
+            )
+            if result.stdout.strip():
+                raise RuntimeError("cold project already owns Docker resources")


## `test(gate): verify exact Compose ownership`

diff --git a/tests/test_cold_gate_compose.py b/tests/test_cold_gate_compose.py
new file mode 100644
index 0000000..28846c0
--- /dev/null
+++ b/tests/test_cold_gate_compose.py
@@ -0,0 +1,79 @@
+import pathlib
+import subprocess
+import tempfile
+import unittest
+
+from scripts.cold_gate.compose import ComposeProject
+from scripts.cold_gate.context import ColdGateContext
+
+
+SHA = "0123456789abcdef0123456789abcdef01234567"
+
+
+class ColdGateComposeTest(unittest.TestCase):
+    def context(self, root: pathlib.Path) -> ColdGateContext:
+        (root / "compose.yaml").write_text("services: {}\n")
+        (root / "compose.toxiproxy.yaml").write_text("services: {}\n")
+        return ColdGateContext.create(root, SHA, "89abcdef")
+
+    def test_scopes_commands_to_exact_project_directory_and_files(self) -> None:
+        calls = []
+
+        def runner(command, **options):
+            calls.append((command, options))
+            return subprocess.CompletedProcess(command, 0, stdout="")
+
+        with tempfile.TemporaryDirectory() as temporary:
+            root = pathlib.Path(temporary).resolve()
+            context = self.context(root)
+            compose = ComposeProject(
+                context, (root / "compose.toxiproxy.yaml",), runner
+            )
+
+            compose.run("config", "--quiet")
+            compose.require_absent()
+
+        command = calls[0][0]
+        self.assertEqual(command[:6], [
+            "docker",
+            "compose",
+            "--project-name",
+            "sb-gate-0123456789ab-89abcdef",
+            "--project-directory",
+            str(root),
+        ])
+        self.assertEqual(
+            command[6:],
+            [
+                "--file",
+                str(root / "compose.yaml"),
+                "--file",
+                str(root / "compose.toxiproxy.yaml"),
+                "config",
+                "--quiet",
+            ],
+        )
+        labels = [call[0][-1] for call in calls[1:]]
+        self.assertEqual(
+            labels,
+            [
+                "label=com.docker.compose.project=sb-gate-0123456789ab-89abcdef"
+            ]
+            * 3,
+        )
+
+    def test_rejects_existing_project_resources_without_down(self) -> None:
+        def runner(command, **_options):
+            output = "owned\n" if command[1:3] == ["ps", "--all"] else ""
+            return subprocess.CompletedProcess(command, 0, stdout=output)
+
+        with tempfile.TemporaryDirectory() as temporary:
+            root = pathlib.Path(temporary).resolve()
+            compose = ComposeProject(self.context(root), runner=runner)
+
+            with self.assertRaisesRegex(RuntimeError, "already owns"):
+                compose.require_absent()
+
+
+if __name__ == "__main__":
+    unittest.main()


## `build(gate): clean only owned cold resources`

diff --git a/scripts/cold_gate/cleanup.py b/scripts/cold_gate/cleanup.py
new file mode 100644
index 0000000..52bc04b
--- /dev/null
+++ b/scripts/cold_gate/cleanup.py
@@ -0,0 +1,60 @@
+from __future__ import annotations
+
+import shutil
+import subprocess
+from collections.abc import Callable
+from pathlib import Path
+
+from scripts.cold_gate.compose import ComposeProject
+from scripts.cold_gate.context import ColdGateContext
+
+
+Runner = Callable[..., subprocess.CompletedProcess[str]]
+
+
+class ScopedCleanup:
+    def __init__(
+        self,
+        context: ColdGateContext,
+        compose: ComposeProject,
+        runner: Runner = subprocess.run,
+    ) -> None:
+        self.context = context
+        self.compose = compose
+        self.runner = runner
+
+    def run(self, sources: Path | None = None) -> None:
+        self.context.require_owned()
+        if sources is not None:
+            expected = self.context.runtime / "sources"
+            if sources != expected or sources.is_symlink() or not sources.is_dir():
+                raise RuntimeError("source cleanup target is not owned by this run")
+
+        self.compose.run(
+            "down",
+            "--volumes",
+            "--remove-orphans",
+            "--rmi",
+            "local",
+            "--timeout",
+            "30",
+        )
+        self.compose.require_absent()
+
+        if sources is not None:
+            self.runner(
+                [
+                    str(self.context.root / "scripts" / "materialize-sources.sh"),
+                    str(sources),
+                    "cleanup",
+                ],
+                cwd=self.context.root,
+                text=True,
+                capture_output=True,
+                check=True,
+            )
+
+        self.context.require_owned()
+        shutil.rmtree(self.context.runtime)
+        (self.context.lock / "owner").unlink()
+        self.context.lock.rmdir()


## `test(gate): reject unsafe cleanup targets`

diff --git a/tests/test_scoped_cleanup.py b/tests/test_scoped_cleanup.py
new file mode 100644
index 0000000..4e77265
--- /dev/null
+++ b/tests/test_scoped_cleanup.py
@@ -0,0 +1,93 @@
+import pathlib
+import subprocess
+import tempfile
+import unittest
+
+from scripts.cold_gate.cleanup import ScopedCleanup
+from scripts.cold_gate.context import ColdGateContext
+
+
+SHA = "0123456789abcdef0123456789abcdef01234567"
+
+
+class FakeCompose:
+    def __init__(self) -> None:
+        self.calls = []
+        self.absence_checks = 0
+
+    def run(self, *arguments: str) -> None:
+        self.calls.append(arguments)
+
+    def require_absent(self) -> None:
+        self.absence_checks += 1
+
+
+class ScopedCleanupTest(unittest.TestCase):
+    def test_removes_only_owned_runtime_and_preserves_evidence(self) -> None:
+        materializer_calls = []
+
+        def runner(command, **_options):
+            materializer_calls.append(command)
+            pathlib.Path(command[1]).rmdir()
+            return subprocess.CompletedProcess(command, 0, stdout="")
+
+        with tempfile.TemporaryDirectory() as temporary:
+            root = pathlib.Path(temporary).resolve()
+            context = ColdGateContext.create(root, SHA, "00000001")
+            sources = context.runtime / "sources"
+            sources.mkdir()
+            compose = FakeCompose()
+
+            ScopedCleanup(context, compose, runner).run(sources)
+
+            self.assertEqual(
+                compose.calls,
+                [
+                    (
+                        "down",
+                        "--volumes",
+                        "--remove-orphans",
+                        "--rmi",
+                        "local",
+                        "--timeout",
+                        "30",
+                    )
+                ],
+            )
+            self.assertEqual(compose.absence_checks, 1)
+            self.assertEqual(materializer_calls[0][1:], [str(sources), "cleanup"])
+            self.assertFalse(context.runtime.exists())
+            self.assertFalse(context.lock.exists())
+            self.assertTrue(context.evidence.is_dir())
+
+    def test_rejects_foreign_or_symlinked_source_targets(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            root = pathlib.Path(temporary).resolve()
+            context = ColdGateContext.create(root, SHA, "00000001")
+            foreign = root / "foreign"
+            foreign.mkdir()
+            compose = FakeCompose()
+
+            with self.assertRaisesRegex(RuntimeError, "not owned"):
+                ScopedCleanup(context, compose).run(foreign)
+
+            (context.runtime / "sources").symlink_to(foreign, target_is_directory=True)
+            with self.assertRaisesRegex(RuntimeError, "not owned"):
+                ScopedCleanup(context, compose).run(context.runtime / "sources")
+            self.assertEqual(compose.calls, [])
+
+    def test_rejects_tampered_ownership_before_docker_commands(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            context = ColdGateContext.create(
+                pathlib.Path(temporary), SHA, "00000001"
+            )
+            (context.runtime / ".owner").write_text("project=foreign\n")
+            compose = FakeCompose()
+
+            with self.assertRaisesRegex(RuntimeError, "marker mismatch"):
+                ScopedCleanup(context, compose).run()
+            self.assertEqual(compose.calls, [])
+
+
+if __name__ == "__main__":
+    unittest.main()


## `fix(gate): pin cold Compose inputs`

diff --git a/scripts/cold_gate/compose.py b/scripts/cold_gate/compose.py
index 20d8876..9e8531b 100644
--- a/scripts/cold_gate/compose.py
+++ b/scripts/cold_gate/compose.py
@@ -2,10 +2,10 @@ from __future__ import annotations
 
 import os
 import subprocess
-from collections.abc import Callable, Iterable
-from pathlib import Path
+from collections.abc import Callable
 
 from scripts.cold_gate.context import ColdGateContext
+from scripts.cold_gate.owned_path import require_regular_file
 
 
 Runner = Callable[..., subprocess.CompletedProcess[str]]
@@ -15,13 +15,19 @@ class ComposeProject:
     def __init__(
         self,
         context: ColdGateContext,
-        overlays: Iterable[Path] = (),
         runner: Runner = subprocess.run,
     ) -> None:
         self.context = context
-        self.files = (context.root / "compose.yaml", *overlays)
-        if not all(path.is_file() for path in self.files):
-            raise RuntimeError("a tracked Compose input is missing")
+        self.files = tuple(
+            context.root / name
+            for name in (
+                "compose.yaml",
+                "compose.toxiproxy.yaml",
+                "compose.observability.yaml",
+            )
+        )
+        for path in self.files:
+            require_regular_file(path)
         self.runner = runner
 
     def command(self, *arguments: str) -> list[str]:
@@ -48,7 +54,7 @@ class ComposeProject:
         return self.runner(
             self.command(*arguments),
             cwd=self.context.root,
-            env=environment or os.environ.copy(),
+            env=os.environ.copy() if environment is None else environment,
             text=True,
             capture_output=capture_output,
             check=check,


## `test(gate): reject unscoped Compose inputs`

diff --git a/tests/test_cold_gate_compose.py b/tests/test_cold_gate_compose.py
index 28846c0..48e0eab 100644
--- a/tests/test_cold_gate_compose.py
+++ b/tests/test_cold_gate_compose.py
@@ -14,6 +14,7 @@ class ColdGateComposeTest(unittest.TestCase):
     def context(self, root: pathlib.Path) -> ColdGateContext:
         (root / "compose.yaml").write_text("services: {}\n")
         (root / "compose.toxiproxy.yaml").write_text("services: {}\n")
+        (root / "compose.observability.yaml").write_text("services: {}\n")
         return ColdGateContext.create(root, SHA, "89abcdef")
 
     def test_scopes_commands_to_exact_project_directory_and_files(self) -> None:
@@ -26,9 +27,7 @@ class ColdGateComposeTest(unittest.TestCase):
         with tempfile.TemporaryDirectory() as temporary:
             root = pathlib.Path(temporary).resolve()
             context = self.context(root)
-            compose = ComposeProject(
-                context, (root / "compose.toxiproxy.yaml",), runner
-            )
+            compose = ComposeProject(context, runner)
 
             compose.run("config", "--quiet")
             compose.require_absent()
@@ -49,6 +48,8 @@ class ColdGateComposeTest(unittest.TestCase):
                 str(root / "compose.yaml"),
                 "--file",
                 str(root / "compose.toxiproxy.yaml"),
+                "--file",
+                str(root / "compose.observability.yaml"),
                 "config",
                 "--quiet",
             ],
@@ -74,6 +75,34 @@ class ColdGateComposeTest(unittest.TestCase):
             with self.assertRaisesRegex(RuntimeError, "already owns"):
                 compose.require_absent()
 
+    def test_preserves_an_explicitly_empty_environment(self) -> None:
+        captured = []
+
+        def runner(command, **options):
+            captured.append(options["env"])
+            return subprocess.CompletedProcess(command, 0, stdout="")
+
+        with tempfile.TemporaryDirectory() as temporary:
+            root = pathlib.Path(temporary).resolve()
+            ComposeProject(self.context(root), runner).run(
+                "config", "--quiet", environment={}
+            )
+
+        self.assertEqual(captured, [{}])
+
+    def test_rejects_symlinked_compose_inputs(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            root = pathlib.Path(temporary).resolve()
+            context = self.context(root)
+            observability = root / "compose.observability.yaml"
+            target = root / "foreign.yaml"
+            target.write_text("services: {}\n")
+            observability.unlink()
+            observability.symlink_to(target)
+
+            with self.assertRaisesRegex(RuntimeError, "regular file"):
+                ComposeProject(context)
+
 
 if __name__ == "__main__":
     unittest.main()


## `build(gate): start one bounded cold stack`

diff --git a/scripts/cold_gate/stack.py b/scripts/cold_gate/stack.py
new file mode 100644
index 0000000..7b05118
--- /dev/null
+++ b/scripts/cold_gate/stack.py
@@ -0,0 +1,65 @@
+from __future__ import annotations
+
+import re
+import subprocess
+
+from scripts.cold_gate.compose import ComposeProject
+from scripts.cold_gate.context import ColdGateContext
+from scripts.cold_gate.evidence import LOG_SERVICES
+
+
+class ColdStack:
+    def __init__(self, context: ColdGateContext, compose: ComposeProject) -> None:
+        if compose.context is not context:
+            raise RuntimeError("cold stack Compose project has different ownership")
+        self.context = context
+        self.compose = compose
+
+    def start(self, environment: dict[str, str]) -> None:
+        self.context.require_owned()
+        if environment.get("COMPOSE_PROJECT_NAME") != self.context.project:
+            raise RuntimeError("Compose environment does not own this cold project")
+        self.compose.require_absent()
+        try:
+            self.compose.run(
+                "config",
+                "--quiet",
+                environment=environment,
+                capture_output=True,
+            )
+            self.compose.run(
+                "up",
+                "--detach",
+                "--build",
+                "--wait",
+                "--wait-timeout",
+                "900",
+                environment=environment,
+                capture_output=True,
+            )
+        except subprocess.CalledProcessError as error:
+            raise RuntimeError("cold stack startup failed") from error
+
+    def loopback_port(self, service: str, container_port: int) -> int:
+        if service not in {"gateway", "toxiproxy", "grafana"} or container_port not in {
+            8080,
+            8474,
+            3000,
+        }:
+            raise ValueError("published port is outside the cold stack contract")
+        result = self.compose.run(
+            "port", service, str(container_port), capture_output=True
+        )
+        endpoint = result.stdout.strip()
+        match = re.fullmatch(r"127\.0\.0\.1:([1-9][0-9]{0,4})", endpoint)
+        if match is None or int(match.group(1)) > 65535:
+            raise RuntimeError(f"{service} did not publish one loopback port")
+        return int(match.group(1))
+
+    def logs(self, service: str) -> str:
+        if service not in LOG_SERVICES:
+            raise ValueError("log service is outside the cold stack inventory")
+        result = self.compose.run(
+            "logs", "--no-color", "--tail", "2000", service, capture_output=True
+        )
+        return result.stdout


