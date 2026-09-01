## `test(evidence): reject unhealthy runtime states`

diff --git a/tests/test_container_state.py b/tests/test_container_state.py
new file mode 100644
index 0000000..c79cced
--- /dev/null
+++ b/tests/test_container_state.py
@@ -0,0 +1,59 @@
+import unittest
+
+from scripts.cold_gate.container_state import ContainerState
+
+
+PROJECT = "sb-gate-0123456789ab-00000001"
+CONTAINER = "a" * 64
+IMAGE = "sha256:" + "b" * 64
+
+
+def receipt(service: str, state: str, health: str, exit_code: int) -> str:
+    return "\t".join(
+        (
+            CONTAINER,
+            f"/{PROJECT}-{service}-1",
+            IMAGE,
+            state,
+            health,
+            str(exit_code),
+            PROJECT,
+            service,
+        )
+    ) + "\n"
+
+
+class ContainerStateTest(unittest.TestCase):
+    def test_accepts_running_and_completed_service_contracts(self) -> None:
+        wallet = ContainerState.parse(
+            receipt("wallet", "running", "healthy", 0), PROJECT, "wallet"
+        )
+        topic_init = ContainerState.parse(
+            receipt("topic-init", "exited", "-", 0), PROJECT, "topic-init"
+        )
+
+        self.assertEqual(wallet.state, "running")
+        self.assertEqual(wallet.image_id, IMAGE)
+        self.assertEqual(topic_init.exit_code, 0)
+
+    def test_rejects_wrong_health_exit_labels_and_ids(self) -> None:
+        invalid = (
+            (receipt("wallet", "running", "unhealthy", 0), "wallet"),
+            (receipt("topic-init", "exited", "-", 1), "topic-init"),
+            (receipt("wallet", "running", "healthy", 0).replace(PROJECT, "other", 1), "wallet"),
+            (receipt("wallet", "running", "healthy", 0).replace(CONTAINER, "short"), "wallet"),
+        )
+        for output, service in invalid:
+            with self.subTest(output=output):
+                with self.assertRaises(RuntimeError):
+                    ContainerState.parse(output, PROJECT, service)
+
+    def test_rejects_unknown_services_and_ambiguous_receipts(self) -> None:
+        with self.assertRaisesRegex(RuntimeError, "invalid"):
+            ContainerState.parse(receipt("wallet", "running", "healthy", 0), PROJECT, "unknown")
+        with self.assertRaisesRegex(RuntimeError, "invalid"):
+            ContainerState.parse("too\tfew\tfields\n", PROJECT, "wallet")
+
+
+if __name__ == "__main__":
+    unittest.main()


## `build(evidence): record scoped runtime identities`

diff --git a/scripts/cold_gate/runtime_evidence.py b/scripts/cold_gate/runtime_evidence.py
new file mode 100644
index 0000000..de0b852
--- /dev/null
+++ b/scripts/cold_gate/runtime_evidence.py
@@ -0,0 +1,87 @@
+from __future__ import annotations
+
+import json
+import re
+import subprocess
+from collections.abc import Callable
+
+from scripts.cold_gate.artifacts import verify_release_artifacts
+from scripts.cold_gate.build import ReleaseArtifacts
+from scripts.cold_gate.compose import ComposeProject
+from scripts.cold_gate.container_state import ContainerState, HEX64
+from scripts.cold_gate.evidence import EvidenceStore
+from scripts.cold_gate.inventory import APPLICATION_SERVICES, SERVICES
+
+
+Runner = Callable[..., subprocess.CompletedProcess[str]]
+INSPECT_FORMAT = (
+    '{{.Id}}\t{{.Name}}\t{{.Image}}\t{{.State.Status}}\t'
+    '{{if .State.Health}}{{.State.Health.Status}}{{else}}-{{end}}\t'
+    '{{.State.ExitCode}}\t{{index .Config.Labels "com.docker.compose.project"}}\t'
+    '{{index .Config.Labels "com.docker.compose.service"}}'
+)
+JAR_SUM = re.compile(r"^([0-9a-f]{64})  /app/app\.jar\n?$")
+
+
+class RuntimeEvidence:
+    def __init__(
+        self,
+        compose: ComposeProject,
+        artifacts: ReleaseArtifacts,
+        store: EvidenceStore,
+        runner: Runner = subprocess.run,
+    ) -> None:
+        self.compose = compose
+        self.artifacts = artifacts
+        self.store = store
+        self.runner = runner
+
+    def capture(self) -> None:
+        self.compose.context.require_owned()
+        release_hashes = verify_release_artifacts(self.artifacts.service_jars)
+        states = []
+        image_rows = ["service\timage_id\tembedded_jar_sha256"]
+        for service in SERVICES:
+            container_id = self.compose.run(
+                "ps", "--all", "--quiet", service, capture_output=True
+            ).stdout.strip()
+            if HEX64.fullmatch(container_id) is None:
+                raise RuntimeError(f"{service} does not have one scoped container")
+            try:
+                inspected = self.runner(
+                    ["docker", "inspect", "--format", INSPECT_FORMAT, container_id],
+                    cwd=self.compose.context.root,
+                    text=True,
+                    capture_output=True,
+                    check=True,
+                ).stdout
+            except subprocess.CalledProcessError as error:
+                raise RuntimeError(f"Docker inspection failed for {service}") from error
+            state = ContainerState.parse(
+                inspected, self.compose.context.project, service
+            )
+            embedded = "-"
+            if service in APPLICATION_SERVICES:
+                output = self.compose.run(
+                    "exec", "-T", service, "sha256sum", "/app/app.jar", capture_output=True
+                ).stdout
+                match = JAR_SUM.fullmatch(output)
+                if match is None or match.group(1) != release_hashes[f"{service}.jar"]:
+                    raise RuntimeError(f"{service} embedded release JAR drifted")
+                embedded = match.group(1)
+            image_rows.append(f"{service}\t{state.image_id}\t{embedded}")
+            states.append(
+                {
+                    "service": service,
+                    "name": state.name,
+                    "image_id": state.image_id,
+                    "state": state.state,
+                    "health": state.health,
+                    "exit_code": state.exit_code,
+                }
+            )
+        self.store.write("images.tsv", "\n".join(image_rows) + "\n")
+        self.store.write(
+            "compose-ps.json",
+            json.dumps(states, sort_keys=True, separators=(",", ":")) + "\n",
+        )


## `test(evidence): detect runtime identity drift`

diff --git a/tests/test_runtime_evidence.py b/tests/test_runtime_evidence.py
new file mode 100644
index 0000000..0bcdf96
--- /dev/null
+++ b/tests/test_runtime_evidence.py
@@ -0,0 +1,91 @@
+import hashlib
+import json
+import pathlib
+import subprocess
+import tempfile
+import unittest
+
+from scripts.cold_gate.build import ReleaseArtifacts
+from scripts.cold_gate.context import ColdGateContext
+from scripts.cold_gate.evidence import EvidenceStore
+from scripts.cold_gate.inventory import APPLICATION_SERVICES, COMPLETED_SERVICES, SERVICES
+from scripts.cold_gate.redaction import EvidenceRedactor
+from scripts.cold_gate.runtime_evidence import RuntimeEvidence
+from tests.test_release_artifact_identity import write_release
+
+
+SHA = "0" * 40
+IMAGE = "sha256:" + "b" * 64
+
+
+class FakeCompose:
+    def __init__(self, context, hashes, drift=None):
+        self.context = context
+        self.hashes = hashes
+        self.drift = drift
+        self.ids = {service: f"{index + 1:064x}" for index, service in enumerate(SERVICES)}
+
+    def run(self, *arguments, **_options):
+        if arguments[0] == "ps":
+            return subprocess.CompletedProcess(arguments, 0, stdout=self.ids[arguments[-1]] + "\n")
+        service = arguments[2]
+        digest = "f" * 64 if service == self.drift else self.hashes[f"{service}.jar"]
+        return subprocess.CompletedProcess(arguments, 0, stdout=f"{digest}  /app/app.jar\n")
+
+
+class RuntimeEvidenceTest(unittest.TestCase):
+    def fixture(self, root, drift=None):
+        for name in ("compose.yaml", "compose.toxiproxy.yaml", "compose.observability.yaml"):
+            (root / name).write_text("services: {}\n")
+        context = ColdGateContext.create(root, SHA, "00000001")
+        jars = root / "jars"
+        jars.mkdir()
+        write_release(jars)
+        hashes = {
+            path.name: hashlib.sha256(path.read_bytes()).hexdigest()
+            for path in jars.glob("*.jar")
+        }
+        fixture = context.runtime / "fixture.jar"
+        fixture.write_bytes(b"fixture")
+        artifacts = ReleaseArtifacts(context.runtime, context.runtime, jars, fixture)
+        store = EvidenceStore(context, EvidenceRedactor(["redaction-secret-value"]))
+        compose = FakeCompose(context, hashes, drift)
+
+        def runner(command, **_options):
+            service = next(name for name, value in compose.ids.items() if value == command[-1])
+            completed = service in COMPLETED_SERVICES
+            fields = (
+                compose.ids[service], f"/{context.project}-{service}-1", IMAGE,
+                "exited" if completed else "running", "-" if completed else "healthy",
+                "0", context.project, service,
+            )
+            return subprocess.CompletedProcess(command, 0, stdout="\t".join(fields) + "\n")
+
+        return context, RuntimeEvidence(compose, artifacts, store, runner)
+
+    def test_records_exact_runtime_and_embedded_release_identities(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            context, evidence = self.fixture(pathlib.Path(temporary).resolve())
+
+            evidence.capture()
+
+            images = (context.evidence / "images.tsv").read_text().splitlines()
+            states = json.loads((context.evidence / "compose-ps.json").read_text())
+            self.assertEqual([row.split("\t")[0] for row in images[1:]], list(SERVICES))
+            self.assertEqual([row["service"] for row in states], list(SERVICES))
+            self.assertEqual(
+                sum(row.split("\t")[2] != "-" for row in images[1:]),
+                len(APPLICATION_SERVICES),
+            )
+
+    def test_rejects_embedded_jar_drift_before_writing(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            context, evidence = self.fixture(pathlib.Path(temporary).resolve(), "wallet")
+
+            with self.assertRaisesRegex(RuntimeError, "wallet embedded"):
+                evidence.capture()
+            self.assertEqual(list(context.evidence.iterdir()), [])
+
+
+if __name__ == "__main__":
+    unittest.main()


## `build(evidence): capture bounded service logs`

diff --git a/scripts/cold_gate/log_evidence.py b/scripts/cold_gate/log_evidence.py
new file mode 100644
index 0000000..a0a10fc
--- /dev/null
+++ b/scripts/cold_gate/log_evidence.py
@@ -0,0 +1,21 @@
+from __future__ import annotations
+
+from scripts.cold_gate.evidence import EvidenceStore
+from scripts.cold_gate.inventory import SERVICES
+from scripts.cold_gate.stack import ColdStack
+
+
+class LogEvidence:
+    def __init__(self, stack: ColdStack, store: EvidenceStore) -> None:
+        if stack.context is not store.context:
+            raise RuntimeError("log evidence ownership mismatch")
+        self.stack = stack
+        self.store = store
+
+    def capture(self) -> None:
+        self.stack.context.require_owned()
+        for service in SERVICES:
+            content = self.stack.logs(service)
+            if "\0" in content:
+                raise RuntimeError(f"{service} emitted an unsafe log")
+            self.store.write(f"logs/{service}.log", content)


## `test(evidence): enforce exact log inventory`

diff --git a/tests/test_log_evidence.py b/tests/test_log_evidence.py
new file mode 100644
index 0000000..89531df
--- /dev/null
+++ b/tests/test_log_evidence.py
@@ -0,0 +1,58 @@
+import pathlib
+import tempfile
+import unittest
+
+from scripts.cold_gate.context import ColdGateContext
+from scripts.cold_gate.evidence import EvidenceStore
+from scripts.cold_gate.inventory import SERVICES
+from scripts.cold_gate.log_evidence import LogEvidence
+from scripts.cold_gate.redaction import EvidenceRedactor
+
+
+SHA = "0" * 40
+SECRET = "service-secret-value-000000000000"
+
+
+class FakeStack:
+    def __init__(self, context):
+        self.context = context
+        self.calls = []
+
+    def logs(self, service):
+        self.calls.append(service)
+        return f"{service} key={SECRET}\n"
+
+
+class LogEvidenceTest(unittest.TestCase):
+    def test_captures_exact_bounded_redacted_log_inventory(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            context = ColdGateContext.create(
+                pathlib.Path(temporary).resolve(), SHA, "00000001"
+            )
+            store = EvidenceStore(context, EvidenceRedactor((SECRET,)))
+            stack = FakeStack(context)
+
+            LogEvidence(stack, store).capture()
+
+            self.assertEqual(stack.calls, list(SERVICES))
+            logs = sorted(path.name for path in (context.evidence / "logs").iterdir())
+            self.assertEqual(logs, sorted(f"{service}.log" for service in SERVICES))
+            self.assertNotIn(SECRET, (context.evidence / "logs/wallet.log").read_text())
+
+    def test_rejects_foreign_ownership_and_unsafe_log_bytes(self):
+        with tempfile.TemporaryDirectory() as left, tempfile.TemporaryDirectory() as right:
+            context = ColdGateContext.create(pathlib.Path(left), SHA, "00000001")
+            foreign = ColdGateContext.create(pathlib.Path(right), SHA, "00000002")
+            store = EvidenceStore(context, EvidenceRedactor((SECRET,)))
+            with self.assertRaisesRegex(RuntimeError, "ownership"):
+                LogEvidence(FakeStack(foreign), store)
+
+            stack = FakeStack(context)
+            stack.logs = lambda _service: "unsafe\0log"
+            with self.assertRaisesRegex(RuntimeError, "unsafe"):
+                LogEvidence(stack, store).capture()
+            self.assertFalse((context.evidence / "logs").exists())
+
+
+if __name__ == "__main__":
+    unittest.main()


## `build(evidence): verify Kafka topic semantics`

diff --git a/scripts/cold_gate/topic_evidence.py b/scripts/cold_gate/topic_evidence.py
new file mode 100644
index 0000000..f63356c
--- /dev/null
+++ b/scripts/cold_gate/topic_evidence.py
@@ -0,0 +1,99 @@
+from __future__ import annotations
+import re
+import subprocess
+from scripts.cold_gate.compose import ComposeProject
+from scripts.cold_gate.evidence import EvidenceStore
+from scripts.cold_gate.owned_path import require_regular_file
+MIN_DLT_RETENTION = 604_800_000
+SOURCE_NAMES = frozenset(
+    "wallet.debited.v1 wallet.credited.v1 wallet.debit-failed.v1 "
+    "risk.limit.violated risk.pattern.suspected odds.changed market.status.changed "
+    "event.lifecycle match.result bet.placed.v1 bet.settled.v1 bet.voided.v1 "
+    "bet.resolution.revised.v1 admin.action".split()
+)
+DLT_NAMES = frozenset(
+    "wallet.debited.v1.DLT wallet.debit-failed.v1.DLT odds.changed.DLT "
+    "event.lifecycle.DLT match.result.DLT bet.placed.v1.DLT bet.settled.v1.DLT "
+    "bet.voided.v1.DLT bet.resolution.revised.v1.DLT".split()
+)
+EXPECTED_NAMES = SOURCE_NAMES | DLT_NAMES
+RETENTION = re.compile(r"(?:^|\s)retention\.ms=(\d+)(?=\s|,|$)", re.MULTILINE)
+
+class TopicEvidence:
+    def __init__(self, compose: ComposeProject, store: EvidenceStore) -> None:
+        if compose.context is not store.context: raise RuntimeError("topic evidence ownership mismatch")
+        self.compose = compose
+        self.store = store
+    def capture(self) -> None:
+        rows = self._manifest()
+        listed = self._query("kafka-topics.sh", "--list")
+        names = [line.strip() for line in listed.splitlines() if line.strip()]
+        public = [name for name in names if not name.startswith("__")]
+        if len(public) != len(set(public)) or set(public) != EXPECTED_NAMES:
+            raise RuntimeError("Kafka topic inventory drifted")
+        evidence = ["topic\tpartitions\treplication_factor\tretention_ms"]
+        for topic, expected_retention in rows:
+            described = self._query("kafka-topics.sh", "--describe", "--topic", topic)
+            partitions = self._single(r"\bPartitionCount:\s*(\d+)\b", described)
+            replication = self._single(r"\bReplicationFactor:\s*(\d+)\b", described)
+            if partitions != 3 or replication != 1:
+                raise RuntimeError(f"{topic} partition or replication drifted")
+            configured = self._query(
+                "kafka-configs.sh", "--entity-type", "topics",
+                "--entity-name", topic, "--describe",
+            )
+            retentions = RETENTION.findall(configured.split("synonyms=", 1)[0])
+            if topic in SOURCE_NAMES:
+                if retentions:
+                    raise RuntimeError(f"{topic} has an undeclared retention override")
+                retention = "-"
+            else:
+                if len(retentions) != 1 or int(retentions[0]) < expected_retention:
+                    raise RuntimeError(f"{topic} retention is missing or too short")
+                retention = retentions[0]
+            evidence.append(f"{topic}\t3\t1\t{retention}")
+        self.store.write("topics.tsv", "\n".join(evidence) + "\n")
+    def _manifest(self) -> tuple[tuple[str, int], ...]:
+        path = self.compose.context.root / "docker/kafka/topics.manifest"
+        require_regular_file(path)
+        rows = []
+        for line in path.read_text().splitlines():
+            if not line or line.startswith("#"):
+                continue
+            fields = line.split("|")
+            if len(fields) != 4 or fields[0] not in EXPECTED_NAMES:
+                raise RuntimeError("Kafka topic manifest is invalid")
+            topic, partitions, replication, retention = fields
+            if partitions != "3" or replication != "1":
+                raise RuntimeError("Kafka topic manifest topology drifted")
+            if topic in SOURCE_NAMES:
+                if retention != "-":
+                    raise RuntimeError("source topic retention contract drifted")
+                minimum = 0
+            elif not retention.isdigit() or int(retention) < MIN_DLT_RETENTION:
+                raise RuntimeError("DLT retention contract drifted")
+            else:
+                minimum = int(retention)
+            rows.append((topic, minimum))
+        if len(rows) != 23 or {topic for topic, _value in rows} != EXPECTED_NAMES:
+            raise RuntimeError("Kafka topic manifest inventory drifted")
+        return tuple(rows)
+
+    def _query(self, tool: str, *arguments: str) -> str:
+        try:
+            result = self.compose.run(
+                "exec", "-T", "kafka", f"/opt/kafka/bin/{tool}",
+                "--bootstrap-server", "kafka:9092", *arguments, capture_output=True,
+            )
+        except subprocess.CalledProcessError as error:
+            raise RuntimeError(f"Kafka evidence query failed: {tool}") from error
+        if not isinstance(result.stdout, str):
+            raise RuntimeError("Kafka evidence query returned no text")
+        return result.stdout
+
+    @staticmethod
+    def _single(pattern: str, value: str) -> int:
+        matches = re.findall(pattern, value)
+        if len(matches) != 1:
+            raise RuntimeError("Kafka topic description is malformed")
+        return int(matches[0])


## `test(evidence): reject Kafka inventory drift`

diff --git a/tests/test_topic_evidence.py b/tests/test_topic_evidence.py
new file mode 100644
index 0000000..280cde7
--- /dev/null
+++ b/tests/test_topic_evidence.py
@@ -0,0 +1,99 @@
+import pathlib
+import subprocess
+import tempfile
+import unittest
+
+from scripts.cold_gate.context import ColdGateContext
+from scripts.cold_gate.evidence import EvidenceStore
+from scripts.cold_gate.redaction import EvidenceRedactor
+from scripts.cold_gate.topic_evidence import DLT_NAMES, EXPECTED_NAMES, TopicEvidence
+
+
+ROOT = pathlib.Path(__file__).resolve().parents[1]
+SHA = "0123456789abcdef0123456789abcdef01234567"
+SECRET = "topic-evidence-secret-000000000000"
+
+
+class FakeCompose:
+    def __init__(self, context: ColdGateContext, fault: str = "") -> None:
+        self.context = context
+        self.fault = fault
+        self.calls = []
+
+    def run(self, *arguments, **_options):
+        self.calls.append(arguments)
+        program = arguments[3]
+        if program.endswith("kafka-topics.sh") and "--list" in arguments:
+            names = sorted(EXPECTED_NAMES)
+            if self.fault == "extra":
+                names.append("undeclared.topic")
+            output = "__consumer_offsets\n" + "\n".join(names) + "\n"
+        elif program.endswith("kafka-topics.sh"):
+            topic = arguments[-1]
+            partitions = 2 if self.fault == "partition" and topic == "admin.action" else 3
+            output = (
+                f"Topic: {topic} TopicId: id PartitionCount: {partitions} "
+                "ReplicationFactor: 1 Configs:\n"
+            )
+        else:
+            topic = arguments[arguments.index("--entity-name") + 1]
+            if topic in DLT_NAMES:
+                retention = 60_000 if self.fault == "short" else 604_800_000
+                output = (
+                    f"Dynamic configs for topic {topic} are: retention.ms={retention} "
+                    f"sensitive=false synonyms={{DYNAMIC_TOPIC_CONFIG:retention.ms={retention}, "
+                    "DEFAULT_CONFIG:log.retention.ms=null}}\n"
+                )
+            elif self.fault == "source-retention" and topic == "admin.action":
+                output = "Dynamic configs for topic admin.action are: retention.ms=1\n"
+            else:
+                output = f"Dynamic configs for topic {topic} are:\n"
+        return subprocess.CompletedProcess(arguments, 0, stdout=output)
+
+
+class TopicEvidenceTest(unittest.TestCase):
+    def fixture(self, root: pathlib.Path, fault: str = ""):
+        manifest = (ROOT / "docker/kafka/topics.manifest").read_text()
+        target = root / "docker/kafka"
+        target.mkdir(parents=True)
+        if fault == "manifest":
+            manifest = manifest.replace("admin.action|3|1|-\n", "")
+        (target / "topics.manifest").write_text(manifest)
+        context = ColdGateContext.create(root, SHA, "00000001")
+        store = EvidenceStore(context, EvidenceRedactor([SECRET]))
+        return context, store, FakeCompose(context, fault)
+
+    def test_records_the_exact_validated_broker_inventory(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            context, store, compose = self.fixture(pathlib.Path(temporary).resolve())
+
+            TopicEvidence(compose, store).capture()
+
+            lines = (context.evidence / "topics.tsv").read_text().splitlines()
+            self.assertEqual(lines[0], "topic\tpartitions\treplication_factor\tretention_ms")
+            self.assertEqual(len(lines), 24)
+            rows = {line.split("\t")[0]: line.split("\t")[1:] for line in lines[1:]}
+            self.assertEqual(set(rows), EXPECTED_NAMES)
+            self.assertTrue(all(row[:2] == ["3", "1"] for row in rows.values()))
+            self.assertTrue(all(rows[name][2] == "-" for name in EXPECTED_NAMES - DLT_NAMES))
+            self.assertTrue(all(int(rows[name][2]) >= 604_800_000 for name in DLT_NAMES))
+            self.assertEqual(len(compose.calls), 47)
+
+    def test_rejects_manifest_or_broker_drift_before_writing(self) -> None:
+        faults = {
+            "manifest": "manifest inventory",
+            "extra": "inventory drifted",
+            "partition": "partition or replication",
+            "short": "retention is missing or too short",
+            "source-retention": "undeclared retention override",
+        }
+        for fault, message in faults.items():
+            with self.subTest(fault=fault), tempfile.TemporaryDirectory() as temporary:
+                context, store, compose = self.fixture(pathlib.Path(temporary).resolve(), fault)
+                with self.assertRaisesRegex(RuntimeError, message):
+                    TopicEvidence(compose, store).capture()
+                self.assertEqual(list(context.evidence.iterdir()), [])
+
+
+if __name__ == "__main__":
+    unittest.main()


