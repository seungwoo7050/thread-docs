## `test(startup): verify exact consumer assignment gate`

diff --git a/tests/fake_consumer_groups.py b/tests/fake_consumer_groups.py
new file mode 100755
index 0000000..4dd65cd
--- /dev/null
+++ b/tests/fake_consumer_groups.py
@@ -0,0 +1,22 @@
+#!/usr/bin/env python3
+import os
+import sys
+
+
+group = sys.argv[sys.argv.index("--group") + 1]
+mode = os.environ.get("ASSIGNMENT_MODE", "ready")
+topics = (
+    "bet.resolution.revised.v1",
+    "bet.settled.v1",
+    "bet.voided.v1",
+)
+rows = [(topic, partition) for topic in topics for partition in range(3)]
+if mode == "missing":
+    rows.pop()
+elif mode == "extra":
+    rows.append(("unexpected.topic", 0))
+
+print("GROUP TOPIC PARTITION CURRENT-OFFSET LOG-END-OFFSET LAG CONSUMER-ID HOST CLIENT-ID")
+for topic, partition in rows:
+    consumer = "-" if mode == "inactive" and partition == 2 else "consumer-1"
+    print(f"{group} {topic} {partition} 0 0 0 {consumer} /host client-1")
diff --git a/tests/test_consumer_assignment_gate.py b/tests/test_consumer_assignment_gate.py
new file mode 100644
index 0000000..893ada3
--- /dev/null
+++ b/tests/test_consumer_assignment_gate.py
@@ -0,0 +1,50 @@
+import os
+import pathlib
+import subprocess
+import unittest
+
+
+ROOT = pathlib.Path(__file__).resolve().parents[1]
+SCRIPT = ROOT / "docker/kafka/wait-consumer-assignments.sh"
+FAKE = ROOT / "tests/fake_consumer_groups.py"
+
+
+class ConsumerAssignmentGateTest(unittest.TestCase):
+    def run_gate(self, mode: str) -> subprocess.CompletedProcess[str]:
+        environment = os.environ.copy()
+        environment.update(
+            {
+                "KAFKA_CONSUMER_GROUPS": str(FAKE),
+                "ASSIGNMENT_MODE": mode,
+                "ASSIGNMENT_TIMEOUT_SECONDS": "1",
+                "ASSIGNMENT_POLL_SECONDS": "0.01",
+            }
+        )
+        return subprocess.run(
+            [str(SCRIPT)],
+            cwd=ROOT,
+            env=environment,
+            text=True,
+            capture_output=True,
+            check=False,
+        )
+
+    def test_accepts_both_exact_active_nine_partition_assignments(self) -> None:
+        result = self.run_gate("ready")
+
+        self.assertEqual(result.returncode, 0, result.stderr)
+        self.assertEqual(
+            result.stdout.strip(),
+            "consumer-assignment: gateway-bets=9 betting-resolution=9",
+        )
+
+    def test_rejects_incomplete_inactive_or_extra_assignments(self) -> None:
+        for mode in ("missing", "inactive", "extra"):
+            with self.subTest(mode=mode):
+                result = self.run_gate(mode)
+                self.assertNotEqual(result.returncode, 0)
+                self.assertIn("timed out", result.stderr)
+
+
+if __name__ == "__main__":
+    unittest.main()


## `test(gate): reject incomplete Kafka assignments`

diff --git a/tests/test_cold_gate_kafka.py b/tests/test_cold_gate_kafka.py
new file mode 100644
index 0000000..d331c5f
--- /dev/null
+++ b/tests/test_cold_gate_kafka.py
@@ -0,0 +1,61 @@
+import subprocess
+import unittest
+
+from scripts.cold_gate.kafka import KafkaAdmin
+
+
+class FakeCompose:
+    def __init__(self, output: str, failure: bool = False) -> None:
+        self.output = output
+        self.failure = failure
+        self.calls = []
+
+    def run(self, *arguments, **options):
+        self.calls.append((arguments, options))
+        if self.failure:
+            raise subprocess.CalledProcessError(1, arguments, stderr="credential")
+        return subprocess.CompletedProcess(arguments, 0, stdout=self.output)
+
+
+class ColdGateKafkaTest(unittest.TestCase):
+    def test_parses_exact_topic_partition_assignments(self) -> None:
+        output = """
+GROUP TOPIC PARTITION CURRENT-OFFSET LOG-END-OFFSET LAG CONSUMER-ID HOST CLIENT-ID
+settlement-service match.result 0 4 4 0 consumer /172.1 client
+settlement-service match.result 1 2 2 0 consumer /172.1 client
+settlement-service event.lifecycle 2 0 0 0 consumer /172.1 client
+"""
+        compose = FakeCompose(output)
+
+        assignments = KafkaAdmin(compose).assignments("settlement-service")
+
+        self.assertEqual(
+            assignments,
+            {("match.result", 0), ("match.result", 1), ("event.lifecycle", 2)},
+        )
+        arguments, options = compose.calls[0]
+        self.assertEqual(arguments[:3], ("exec", "-T", "kafka"))
+        self.assertEqual(arguments[-2:], ("--group", "settlement-service"))
+        self.assertEqual(options, {"capture_output": True})
+
+    def test_rejects_unknown_groups_duplicate_rows_and_invalid_partitions(self) -> None:
+        with self.assertRaisesRegex(ValueError, "outside"):
+            KafkaAdmin(FakeCompose("")).assignments("unknown")
+        duplicate = "\n".join(
+            ["settlement-service match.result 0 0 0 0 c h id"] * 2
+        )
+        with self.assertRaisesRegex(RuntimeError, "duplicated"):
+            KafkaAdmin(FakeCompose(duplicate)).assignments("settlement-service")
+        with self.assertRaisesRegex(RuntimeError, "out of range"):
+            KafkaAdmin(
+                FakeCompose("settlement-service match.result 3 0 0 0 c h id\n")
+            ).assignments("settlement-service")
+
+    def test_hides_kafka_transport_output(self) -> None:
+        with self.assertRaisesRegex(RuntimeError, "settlement-service") as captured:
+            KafkaAdmin(FakeCompose("", failure=True)).assignments("settlement-service")
+        self.assertNotIn("credential", str(captured.exception))
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
