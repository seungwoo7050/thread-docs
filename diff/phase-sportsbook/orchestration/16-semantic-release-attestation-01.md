# 의미 기반 릴리스 증명

## `build(evidence): share runtime service inventory`

diff --git a/scripts/cold_gate/evidence.py b/scripts/cold_gate/evidence.py
index 52726ec..315b5cb 100644
--- a/scripts/cold_gate/evidence.py
+++ b/scripts/cold_gate/evidence.py
@@ -6,6 +6,7 @@ import tempfile
 from pathlib import Path, PurePosixPath
 
 from scripts.cold_gate.context import ColdGateContext
+from scripts.cold_gate.inventory import SERVICES
 from scripts.cold_gate.owned_path import (
     ensure_directory,
     require_directory,
@@ -29,14 +30,7 @@ REQUIRED_FILES = frozenset(
         "cleanup.tsv",
     }
 )
-LOG_SERVICES = frozenset(
-    {
-        "postgres", "kafka", "topic-init", "secret-preflight",
-        "redis-risk", "redis-odds", "redis-wallet", "redis-gateway",
-        "wallet", "risk", "odds", "betting", "gateway", "settlement", "admin",
-        "consumer-assignment", "toxiproxy", "prometheus", "loki", "grafana", "promtail",
-    }
-)
+LOG_SERVICES = frozenset(SERVICES)
 LOG_PATTERN = re.compile(r"logs/([a-z0-9][a-z0-9-]*)\.log")
 
 


## `test(evidence): prevent service inventory drift`

diff --git a/tests/test_gate_inventory.py b/tests/test_gate_inventory.py
index e8090cb..45e8b17 100644
--- a/tests/test_gate_inventory.py
+++ b/tests/test_gate_inventory.py
@@ -11,6 +11,7 @@ from scripts.cold_gate.inventory import (
     MIGRATION_VERSIONS,
     SERVICES,
 )
+from scripts.cold_gate.evidence import LOG_SERVICES
 
 
 ROOT = pathlib.Path(__file__).resolve().parents[1]
@@ -39,6 +40,7 @@ class GateInventoryTest(unittest.TestCase):
 
     def test_partitions_runtime_roles_and_exact_migrations(self) -> None:
         self.assertEqual(len(SERVICES), 21)
+        self.assertEqual(LOG_SERVICES, set(SERVICES))
         self.assertEqual(len(LONG_RUNNING_SERVICES), 18)
         self.assertEqual(len(COMPLETED_SERVICES), 3)
         self.assertEqual(LONG_RUNNING_SERVICES | COMPLETED_SERVICES, set(SERVICES))


## `build(evidence): validate locked release rows`

diff --git a/scripts/cold_gate/release_identity.py b/scripts/cold_gate/release_identity.py
new file mode 100644
index 0000000..167c74d
--- /dev/null
+++ b/scripts/cold_gate/release_identity.py
@@ -0,0 +1,32 @@
+from __future__ import annotations
+
+import hashlib
+import re
+from pathlib import Path
+
+from scripts.cold_gate.artifacts import SERVICES
+
+
+SHA = re.compile(r"^[0-9a-f]{40}$")
+
+
+def lock_entries(content: str) -> list[tuple[str, str, str, str]]:
+    rows = [
+        tuple(line.split("|"))
+        for line in content.splitlines()
+        if line and not line.startswith("#")
+    ]
+    if len(rows) != 8 or any(
+        len(row) != 4 or SHA.fullmatch(row[2]) is None for row in rows
+    ):
+        raise RuntimeError("services lock evidence is invalid")
+    if [row[0] for row in rows] != ["shared", *SERVICES]:
+        raise RuntimeError("services lock evidence order drifted")
+    return rows
+
+
+def jar_row(
+    logical: str, source_sha: str, source: str, staged: str, path: Path
+) -> str:
+    digest = hashlib.sha256(path.read_bytes()).hexdigest()
+    return f"{logical}\t{source_sha}\t{source}\t{staged}\t{digest}"


## `test(evidence): reject locked release drift`

diff --git a/tests/test_release_identity.py b/tests/test_release_identity.py
new file mode 100644
index 0000000..1b94a5f
--- /dev/null
+++ b/tests/test_release_identity.py
@@ -0,0 +1,52 @@
+import hashlib
+import pathlib
+import tempfile
+import unittest
+
+from scripts.cold_gate.release_identity import jar_row, lock_entries
+
+
+ROOT = pathlib.Path(__file__).resolve().parents[1]
+
+
+class ReleaseIdentityTest(unittest.TestCase):
+    def test_accepts_the_exact_eight_locked_release_rows(self) -> None:
+        rows = lock_entries((ROOT / "services.lock").read_text())
+
+        self.assertEqual(len(rows), 8)
+        self.assertEqual(rows[0][0], "shared")
+        self.assertEqual(rows[-1][0], "admin")
+        self.assertTrue(all(len(row[2]) == 40 for row in rows))
+
+    def test_rejects_missing_reordered_or_malformed_locks(self) -> None:
+        lines = (ROOT / "services.lock").read_text().splitlines()
+        variants = (
+            "\n".join(lines[:-1]) + "\n",
+            "\n".join([lines[0], lines[2], lines[1], *lines[3:]]) + "\n",
+            (ROOT / "services.lock").read_text().replace(rows_sha(lines[1]), "invalid", 1),
+        )
+        for content in variants:
+            with self.subTest(content=content):
+                with self.assertRaises(RuntimeError):
+                    lock_entries(content)
+
+    def test_records_the_actual_staged_artifact_hash(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            artifact = pathlib.Path(temporary) / "wallet.jar"
+            artifact.write_bytes(b"release")
+            digest = hashlib.sha256(b"release").hexdigest()
+
+            row = jar_row("wallet", "a" * 40, "source.jar", "wallet.jar", artifact)
+
+            self.assertEqual(
+                row,
+                f"wallet\t{'a' * 40}\tsource.jar\twallet.jar\t{digest}",
+            )
+
+
+def rows_sha(line: str) -> str:
+    return line.split("|")[2]
+
+
+if __name__ == "__main__":
+    unittest.main()


## `build(evidence): record locked release identities`

diff --git a/scripts/cold_gate/release_evidence.py b/scripts/cold_gate/release_evidence.py
new file mode 100644
index 0000000..c6c9fa1
--- /dev/null
+++ b/scripts/cold_gate/release_evidence.py
@@ -0,0 +1,93 @@
+from __future__ import annotations
+
+import hashlib
+import subprocess
+from collections.abc import Callable
+
+from scripts.cold_gate.artifacts import SERVICES, verify_release_artifacts
+from scripts.cold_gate.build import ReleaseArtifacts
+from scripts.cold_gate.context import ColdGateContext
+from scripts.cold_gate.evidence import EvidenceStore
+from scripts.cold_gate.owned_path import require_regular_file
+from scripts.cold_gate.release_identity import SHA, jar_row, lock_entries
+
+
+Runner = Callable[..., subprocess.CompletedProcess[str]]
+class ReleaseEvidence:
+    def __init__(
+        self,
+        context: ColdGateContext,
+        artifacts: ReleaseArtifacts,
+        store: EvidenceStore,
+        runner: Runner = subprocess.run,
+    ) -> None:
+        self.context = context
+        self.artifacts = artifacts
+        self.store = store
+        self.runner = runner
+
+    def capture(self, orchestration_sha: str) -> None:
+        self.context.require_owned()
+        if not SHA.fullmatch(orchestration_sha) or orchestration_sha[:12] not in self.context.project:
+            raise RuntimeError("orchestration identity does not own this cold run")
+        head = self._git("rev-parse", "HEAD")
+        dirty = self._git("status", "--porcelain", "--untracked-files=no")
+        if head != orchestration_sha or dirty:
+            raise RuntimeError("orchestration checkout is not the clean requested SHA")
+
+        lock_path = self.context.root / "services.lock"
+        require_regular_file(lock_path)
+        lock_content = lock_path.read_text()
+        entries = lock_entries(lock_content)
+        for logical, _branch, commit, _artifact in entries:
+            source = self.artifacts.sources / logical
+            observed = self._git("-C", str(source), "rev-parse", "HEAD")
+            if observed != commit:
+                raise RuntimeError(f"{logical} materialized SHA drifted")
+
+        service_hashes = verify_release_artifacts(self.artifacts.service_jars)
+        rows = ["logical\tsource_sha\tsource_artifact\tstaged_artifact\tsha256"]
+        shared = (
+            self.artifacts.maven_repository
+            / "com/sportsbook/shared-protocol/1.0.0/shared-protocol-1.0.0.jar"
+        )
+        require_regular_file(shared)
+        shared_entry = entries[0]
+        rows.append(jar_row("shared", shared_entry[2], shared_entry[3], shared.name, shared))
+        entry_by_name = {entry[0]: entry for entry in entries}
+        for logical in SERVICES:
+            entry = entry_by_name[logical]
+            jar = self.artifacts.service_jars / f"{logical}.jar"
+            row = jar_row(logical, entry[2], entry[3], jar.name, jar)
+            if row.rsplit("\t", 1)[1] != service_hashes[jar.name]:
+                raise RuntimeError(f"{logical} staged checksum drifted")
+            rows.append(row)
+        require_regular_file(self.artifacts.fixture_jar)
+        rows.append(
+            jar_row(
+                "fixture",
+                orchestration_sha,
+                "fixtures/avro-publisher/pom.xml",
+                self.artifacts.fixture_jar.name,
+                self.artifacts.fixture_jar,
+            )
+        )
+        lock_hash = hashlib.sha256(lock_content.encode()).hexdigest()
+        self.store.write(
+            "run.tsv",
+            "field\tvalue\n"
+            f"project\t{self.context.project}\n"
+            f"orchestration_sha\t{orchestration_sha}\n"
+            f"services_lock_sha256\t{lock_hash}\n",
+        )
+        self.store.write("services.lock", lock_content)
+        self.store.write("jars.sha256", "\n".join(rows) + "\n")
+
+    def _git(self, *arguments: str) -> str:
+        result = self.runner(
+            ["git", "-C", str(self.context.root), *arguments],
+            text=True,
+            capture_output=True,
+            check=True,
+        )
+        return result.stdout.strip()


## `test(evidence): reject release identity drift`

diff --git a/tests/test_release_evidence.py b/tests/test_release_evidence.py
new file mode 100644
index 0000000..21d4030
--- /dev/null
+++ b/tests/test_release_evidence.py
@@ -0,0 +1,92 @@
+import pathlib
+import subprocess
+import tempfile
+import unittest
+
+from scripts.cold_gate.build import ReleaseArtifacts
+from scripts.cold_gate.context import ColdGateContext
+from scripts.cold_gate.evidence import EvidenceStore
+from scripts.cold_gate.redaction import EvidenceRedactor
+from scripts.cold_gate.release_evidence import ReleaseEvidence
+from scripts.cold_gate.release_identity import lock_entries
+from tests.test_release_artifact_identity import write_release
+
+
+ROOT = pathlib.Path(__file__).resolve().parents[1]
+ORCHESTRATION_SHA = "0123456789abcdef0123456789abcdef01234567"
+
+
+class ReleaseEvidenceTest(unittest.TestCase):
+    def fixture(self, root: pathlib.Path):
+        lock_content = (ROOT / "services.lock").read_text()
+        (root / "services.lock").write_text(lock_content)
+        context = ColdGateContext.create(root, ORCHESTRATION_SHA, "00000001")
+        sources = context.runtime / "sources"
+        sources.mkdir()
+        commits = {}
+        for logical, _branch, commit, _artifact in lock_entries(lock_content):
+            (sources / logical).mkdir()
+            commits[logical] = commit
+        repository = context.runtime / "m2/repository"
+        shared = repository / "com/sportsbook/shared-protocol/1.0.0/shared-protocol-1.0.0.jar"
+        shared.parent.mkdir(parents=True)
+        shared.write_bytes(b"shared")
+        service_jars = root / "docker/.jars/generation.test"
+        service_jars.mkdir(parents=True)
+        write_release(service_jars)
+        fixture_dir = context.runtime / "fixtures"
+        fixture_dir.mkdir()
+        fixture_jar = fixture_dir / "avro-fixture-publisher.jar"
+        fixture_jar.write_bytes(b"fixture")
+        artifacts = ReleaseArtifacts(sources, repository, service_jars, fixture_jar)
+        store = EvidenceStore(context, EvidenceRedactor(["redaction-secret-value"]))
+        return context, artifacts, store, commits
+
+    def runner(self, root: pathlib.Path, commits: dict[str, str], mismatch: str | None = None):
+        def run(command, **_options):
+            target = pathlib.Path(command[max(index for index, value in enumerate(command) if value == "-C") + 1])
+            if "status" in command:
+                output = ""
+            elif target == root:
+                output = ORCHESTRATION_SHA + "\n"
+            else:
+                output = ("0" * 40 if target.name == mismatch else commits[target.name]) + "\n"
+            return subprocess.CompletedProcess(command, 0, stdout=output)
+
+        return run
+
+    def test_records_release_sources_and_actual_artifact_hashes(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            root = pathlib.Path(temporary).resolve()
+            context, artifacts, store, commits = self.fixture(root)
+
+            ReleaseEvidence(
+                context, artifacts, store, self.runner(root, commits)
+            ).capture(ORCHESTRATION_SHA)
+
+            run = (context.evidence / "run.tsv").read_text()
+            jars = (context.evidence / "jars.sha256").read_text().splitlines()
+            self.assertIn(f"project\t{context.project}", run)
+            self.assertIn(f"orchestration_sha\t{ORCHESTRATION_SHA}", run)
+            self.assertEqual(len(jars), 10)
+            self.assertEqual([line.split("\t")[0] for line in jars[1:]],
+                             ["shared", "wallet", "risk", "odds", "betting", "gateway", "settlement", "admin", "fixture"])
+            self.assertEqual(
+                (context.evidence / "services.lock").read_text(),
+                (root / "services.lock").read_text(),
+            )
+
+    def test_rejects_a_materialized_source_mismatch_before_writing(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            root = pathlib.Path(temporary).resolve()
+            context, artifacts, store, commits = self.fixture(root)
+
+            with self.assertRaisesRegex(RuntimeError, "wallet materialized"):
+                ReleaseEvidence(
+                    context, artifacts, store, self.runner(root, commits, "wallet")
+                ).capture(ORCHESTRATION_SHA)
+            self.assertEqual(list(context.evidence.iterdir()), [])
+
+
+if __name__ == "__main__":
+    unittest.main()


## `build(evidence): record canonical Compose identity`

diff --git a/scripts/cold_gate/compose_evidence.py b/scripts/cold_gate/compose_evidence.py
new file mode 100644
index 0000000..9137a27
--- /dev/null
+++ b/scripts/cold_gate/compose_evidence.py
@@ -0,0 +1,44 @@
+from __future__ import annotations
+
+import hashlib
+import json
+from collections.abc import Iterable
+
+from scripts.cold_gate.compose import ComposeProject
+from scripts.cold_gate.evidence import EvidenceStore
+from scripts.cold_gate.inventory import SERVICES
+
+
+def capture_compose_config(
+    compose: ComposeProject,
+    store: EvidenceStore,
+    environment: dict[str, str],
+    secret_values: Iterable[str],
+) -> str:
+    if environment.get("COMPOSE_PROJECT_NAME") != compose.context.project:
+        raise RuntimeError("Compose evidence environment owns another project")
+    result = compose.run(
+        "config",
+        "--no-interpolate",
+        "--format",
+        "json",
+        environment=environment,
+        capture_output=True,
+    )
+    if any(secret and secret in result.stdout for secret in secret_values):
+        raise RuntimeError("rendered Compose config contains a runtime secret")
+    try:
+        config = json.loads(result.stdout)
+    except json.JSONDecodeError as error:
+        raise RuntimeError("rendered Compose config is not JSON") from error
+    if (
+        not isinstance(config, dict)
+        or config.get("name") != compose.context.project
+        or not isinstance(config.get("services"), dict)
+        or set(config["services"]) != set(SERVICES)
+    ):
+        raise RuntimeError("rendered Compose inventory drifted")
+    canonical = json.dumps(config, sort_keys=True, separators=(",", ":")).encode()
+    digest = hashlib.sha256(canonical).hexdigest()
+    store.write("compose.sha256", f"artifact\tsha256\ncombined-config\t{digest}\n")
+    return digest


## `test(evidence): reject Compose identity drift`

diff --git a/tests/test_compose_evidence.py b/tests/test_compose_evidence.py
new file mode 100644
index 0000000..7678c44
--- /dev/null
+++ b/tests/test_compose_evidence.py
@@ -0,0 +1,74 @@
+import json
+import pathlib
+import subprocess
+import tempfile
+import unittest
+
+from scripts.cold_gate.compose_evidence import capture_compose_config
+from scripts.cold_gate.context import ColdGateContext
+from scripts.cold_gate.evidence import EvidenceStore
+from scripts.cold_gate.inventory import SERVICES
+from scripts.cold_gate.redaction import EvidenceRedactor
+
+
+SHA = "0123456789abcdef0123456789abcdef01234567"
+SECRET = "runtime-secret-value-000000000000"
+
+
+class FakeCompose:
+    def __init__(self, context: ColdGateContext, output: str) -> None:
+        self.context = context
+        self.output = output
+        self.calls = []
+
+    def run(self, *arguments, **options):
+        self.calls.append((arguments, options))
+        return subprocess.CompletedProcess(arguments, 0, stdout=self.output)
+
+
+class ComposeEvidenceTest(unittest.TestCase):
+    def fixture(self, root: pathlib.Path):
+        context = ColdGateContext.create(root, SHA, "00000001")
+        store = EvidenceStore(context, EvidenceRedactor([SECRET]))
+        environment = {"COMPOSE_PROJECT_NAME": context.project}
+        config = {
+            "services": {service: {"image": service} for service in reversed(SERVICES)},
+            "name": context.project,
+            "networks": {"backend": {"internal": True}},
+        }
+        return context, store, environment, config
+
+    def test_records_only_the_canonical_combined_config_hash(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            context, store, environment, config = self.fixture(
+                pathlib.Path(temporary).resolve()
+            )
+            compose = FakeCompose(context, json.dumps(config))
+
+            digest = capture_compose_config(compose, store, environment, [SECRET])
+
+            self.assertRegex(digest, r"^[0-9a-f]{64}$")
+            self.assertEqual(
+                (context.evidence / "compose.sha256").read_text(),
+                f"artifact\tsha256\ncombined-config\t{digest}\n",
+            )
+            self.assertEqual(
+                compose.calls[0][0],
+                ("config", "--no-interpolate", "--format", "json"),
+            )
+
+    def test_rejects_secret_or_service_inventory_drift_before_writing(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            context, store, environment, config = self.fixture(
+                pathlib.Path(temporary).resolve()
+            )
+            config["credential"] = SECRET
+            with self.assertRaisesRegex(RuntimeError, "runtime secret"):
+                capture_compose_config(
+                    FakeCompose(context, json.dumps(config)), store, environment, [SECRET]
+                )
+            self.assertEqual(list(context.evidence.iterdir()), [])
+
+
+if __name__ == "__main__":
+    unittest.main()


## `build(evidence): validate scoped container states`

diff --git a/scripts/cold_gate/container_state.py b/scripts/cold_gate/container_state.py
new file mode 100644
index 0000000..f3dddbd
--- /dev/null
+++ b/scripts/cold_gate/container_state.py
@@ -0,0 +1,46 @@
+from __future__ import annotations
+
+import dataclasses
+import re
+
+from scripts.cold_gate.inventory import COMPLETED_SERVICES, LONG_RUNNING_SERVICES, SERVICES
+
+
+HEX64 = re.compile(r"^[0-9a-f]{64}$")
+IMAGE_ID = re.compile(r"^sha256:[0-9a-f]{64}$")
+
+
+@dataclasses.dataclass(frozen=True)
+class ContainerState:
+    service: str
+    name: str
+    image_id: str
+    state: str
+    health: str
+    exit_code: int
+
+    @classmethod
+    def parse(cls, output: str, project: str, expected_service: str) -> "ContainerState":
+        fields = output.rstrip("\n").split("\t")
+        if len(fields) != 8 or expected_service not in SERVICES:
+            raise RuntimeError("Docker inspection receipt is invalid")
+        container_id, raw_name, image_id, state, health, exit_code, label_project, service = fields
+        if (
+            HEX64.fullmatch(container_id) is None
+            or IMAGE_ID.fullmatch(image_id) is None
+            or label_project != project
+            or service != expected_service
+            or raw_name != f"/{project}-{expected_service}-1"
+            or not exit_code.isdigit()
+        ):
+            raise RuntimeError("Docker inspection identity drifted")
+        observed_exit = int(exit_code)
+        if expected_service in LONG_RUNNING_SERVICES and (
+            state != "running" or health != "healthy" or observed_exit != 0
+        ):
+            raise RuntimeError(f"{expected_service} is not running and healthy")
+        if expected_service in COMPLETED_SERVICES and (
+            state != "exited" or health != "-" or observed_exit != 0
+        ):
+            raise RuntimeError(f"{expected_service} did not complete successfully")
+        return cls(expected_service, raw_name[1:], image_id, state, health, observed_exit)


