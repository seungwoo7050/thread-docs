## `build(fixtures): parse broker publication receipts`

diff --git a/scripts/cold_gate/fixture_receipt.py b/scripts/cold_gate/fixture_receipt.py
new file mode 100644
index 0000000..ad6079f
--- /dev/null
+++ b/scripts/cold_gate/fixture_receipt.py
@@ -0,0 +1,45 @@
+from __future__ import annotations
+
+import dataclasses
+import re
+import uuid
+
+
+TOPICS = frozenset(
+    {"event.lifecycle", "match.result", "bet.settled.v1", "bet.resolution.revised.v1"}
+)
+RECEIPT = re.compile(
+    r"^topic=([^\t]+)\tkey=([^\t]+)\tpartition=([0-2])\toffset=([0-9]+)"
+    r"\tsha256=([0-9a-f]{64})\tfingerprint=([0-9a-f]{16}|malformed)\n?$"
+)
+
+
+@dataclasses.dataclass(frozen=True)
+class FixtureReceipt:
+    topic: str
+    key: str
+    partition: int
+    offset: int
+    sha256: str
+    fingerprint: str
+
+    @classmethod
+    def parse(cls, output: str) -> "FixtureReceipt":
+        match = RECEIPT.fullmatch(output)
+        if match is None or match.group(1) not in TOPICS:
+            raise RuntimeError("fixture publication receipt is invalid")
+        key = match.group(2)
+        try:
+            parsed = uuid.UUID(key)
+        except ValueError as error:
+            raise RuntimeError("fixture receipt key is not a UUID") from error
+        if str(parsed) != key:
+            raise RuntimeError("fixture receipt key is not canonical")
+        return cls(
+            topic=match.group(1),
+            key=key,
+            partition=int(match.group(3)),
+            offset=int(match.group(4)),
+            sha256=match.group(5),
+            fingerprint=match.group(6),
+        )


## `test(fixtures): reject forged publication receipts`

diff --git a/tests/test_fixture_receipt.py b/tests/test_fixture_receipt.py
new file mode 100644
index 0000000..18a77f5
--- /dev/null
+++ b/tests/test_fixture_receipt.py
@@ -0,0 +1,50 @@
+import unittest
+
+from scripts.cold_gate.fixture_receipt import FixtureReceipt
+
+
+KEY = "11000000-0000-7000-8000-0000000000ab"
+HASH = "a" * 64
+
+
+class FixtureReceiptTest(unittest.TestCase):
+    def test_parses_an_exact_acknowledged_publication(self) -> None:
+        receipt = FixtureReceipt.parse(
+            f"topic=match.result\tkey={KEY}\tpartition=2\toffset=19"
+            f"\tsha256={HASH}\tfingerprint=3f39fbc4bbfea727\n"
+        )
+
+        self.assertEqual(receipt.topic, "match.result")
+        self.assertEqual(receipt.key, KEY)
+        self.assertEqual(receipt.partition, 2)
+        self.assertEqual(receipt.offset, 19)
+        self.assertEqual(receipt.sha256, HASH)
+        self.assertEqual(receipt.fingerprint, "3f39fbc4bbfea727")
+
+    def test_accepts_only_the_fixed_poison_receipt_shape(self) -> None:
+        receipt = FixtureReceipt.parse(
+            f"topic=match.result\tkey={KEY}\tpartition=2\toffset=0"
+            f"\tsha256={HASH}\tfingerprint=malformed\n"
+        )
+        self.assertEqual(receipt.fingerprint, "malformed")
+
+    def test_rejects_unowned_topics_and_malformed_fields(self) -> None:
+        valid = (
+            f"topic=match.result\tkey={KEY}\tpartition=2\toffset=0"
+            f"\tsha256={HASH}\tfingerprint=malformed\n"
+        )
+        invalid = (
+            valid.replace("match.result", "unknown.topic"),
+            valid.replace("partition=2", "partition=3"),
+            valid.replace("offset=0", "offset=-1"),
+            valid.replace(KEY, KEY.upper()),
+            valid + "extra\n",
+        )
+        for output in invalid:
+            with self.subTest(output=output):
+                with self.assertRaises(RuntimeError):
+                    FixtureReceipt.parse(output)
+
+
+if __name__ == "__main__":
+    unittest.main()


## `build(fixtures): publish scoped Avro scenarios`

diff --git a/scripts/cold_gate/fixtures.py b/scripts/cold_gate/fixtures.py
new file mode 100644
index 0000000..7656944
--- /dev/null
+++ b/scripts/cold_gate/fixtures.py
@@ -0,0 +1,95 @@
+from __future__ import annotations
+
+import hashlib
+import json
+import subprocess
+import tempfile
+import uuid
+from pathlib import Path
+
+from scripts.cold_gate.compose import ComposeProject
+from scripts.cold_gate.context import ColdGateContext
+from scripts.cold_gate.fixture_receipt import FixtureReceipt
+from scripts.cold_gate.owned_path import ensure_directory, require_regular_file
+
+
+FIXTURE_TYPES = {
+    "EventLifecycle": ("event.lifecycle", "e47d6dbd952bc721"),
+    "MatchResult": ("match.result", "3f39fbc4bbfea727"),
+    "BetSettled": ("bet.settled.v1", "113bc9d5037a850c"),
+    "BetResolutionRevised": ("bet.resolution.revised.v1", "b05cdf4b95651059"),
+}
+
+
+class FixturePublisher:
+    def __init__(
+        self, context: ColdGateContext, compose: ComposeProject, fixture_jar: Path
+    ) -> None:
+        expected = context.runtime / "fixtures/avro-fixture-publisher.jar"
+        if fixture_jar != expected or compose.context is not context:
+            raise RuntimeError("fixture publisher is not owned by this cold gate")
+        context.require_owned()
+        require_regular_file(fixture_jar)
+        self.context = context
+        self.compose = compose
+        self.fixture_jar = fixture_jar
+
+    def publish(
+        self, fixture_type: str, payload: dict[str, object], partition: int | None = None
+    ) -> FixtureReceipt:
+        if fixture_type not in FIXTURE_TYPES or partition not in (None, 0, 1, 2):
+            raise ValueError("fixture publication is outside the schema contract")
+        inputs = self.context.runtime / "fixture-inputs"
+        ensure_directory(inputs)
+        pending: Path | None = None
+        try:
+            with tempfile.NamedTemporaryFile(
+                mode="w", dir=inputs, prefix="fixture.", suffix=".json", delete=False
+            ) as output:
+                json.dump(payload, output, sort_keys=True, separators=(",", ":"))
+                pending = Path(output.name)
+            pending.chmod(0o600)
+            arguments = ["publish", "kafka:9092", fixture_type, "/fixture.json"]
+            if partition is not None:
+                arguments.append(str(partition))
+            receipt = self._run(arguments, pending)
+        finally:
+            if pending is not None:
+                pending.unlink(missing_ok=True)
+        topic, fingerprint = FIXTURE_TYPES[fixture_type]
+        if receipt.topic != topic or receipt.fingerprint != fingerprint:
+            raise RuntimeError("fixture publisher returned the wrong schema identity")
+        if partition is not None and receipt.partition != partition:
+            raise RuntimeError("fixture publisher returned the wrong partition")
+        return receipt
+
+    def poison_match_result(self, event_id: str) -> FixtureReceipt:
+        parsed = uuid.UUID(event_id)
+        if str(parsed) != event_id:
+            raise ValueError("poison event ID must be canonical")
+        receipt = self._run(["poison", "kafka:9092", event_id])
+        poison_hash = hashlib.sha256(b"\x80").hexdigest()
+        if (
+            receipt.topic != "match.result"
+            or receipt.key != event_id
+            or receipt.partition != 2
+            or receipt.sha256 != poison_hash
+            or receipt.fingerprint != "malformed"
+        ):
+            raise RuntimeError("poison publisher returned the wrong record identity")
+        return receipt
+
+    def _run(self, arguments: list[str], input_file: Path | None = None) -> FixtureReceipt:
+        command = [
+            "run", "--rm", "--no-deps", "--entrypoint", "java",
+            "--volume", f"{self.fixture_jar}:/fixture.jar:ro",
+        ]
+        if input_file is not None:
+            require_regular_file(input_file)
+            command.extend(("--volume", f"{input_file}:/fixture.json:ro"))
+        command.extend(("wallet", "-jar", "/fixture.jar", *arguments))
+        try:
+            result = self.compose.run(*command, capture_output=True)
+        except subprocess.CalledProcessError as error:
+            raise RuntimeError("fixture publication failed") from error
+        return FixtureReceipt.parse(result.stdout)


## `build(fixtures): read scoped Kafka records`

diff --git a/scripts/cold_gate/probe.py b/scripts/cold_gate/probe.py
new file mode 100644
index 0000000..dd5e33d
--- /dev/null
+++ b/scripts/cold_gate/probe.py
@@ -0,0 +1,59 @@
+from __future__ import annotations
+
+import subprocess
+from pathlib import Path
+
+from scripts.cold_gate.compose import ComposeProject
+from scripts.cold_gate.context import ColdGateContext
+from scripts.cold_gate.kafka_record import KafkaRecord, TOPIC
+from scripts.cold_gate.owned_path import require_regular_file
+
+
+PROBE_CLASS = "com.sportsbook.orchestration.fixture.KafkaProbe"
+
+
+class KafkaProbe:
+    def __init__(
+        self, context: ColdGateContext, compose: ComposeProject, fixture_jar: Path
+    ) -> None:
+        expected = context.runtime / "fixtures/avro-fixture-publisher.jar"
+        if compose.context is not context or fixture_jar != expected:
+            raise RuntimeError("Kafka probe is not owned by this cold gate")
+        context.require_owned()
+        require_regular_file(fixture_jar)
+        self.context = context
+        self.compose = compose
+        self.fixture_jar = fixture_jar
+
+    def read(
+        self,
+        topic: str,
+        partition: int,
+        offset: int,
+        schema: Path | None = None,
+    ) -> KafkaRecord:
+        if TOPIC.fullmatch(topic) is None or partition not in range(3) or offset < 0:
+            raise ValueError("Kafka probe target is outside the gate contract")
+        command = [
+            "run", "--rm", "--no-deps", "--entrypoint", "java",
+            "--volume", f"{self.fixture_jar}:/fixture.jar:ro",
+        ]
+        schema_argument = []
+        if schema is not None:
+            require_regular_file(schema)
+            source_root = self.context.runtime / "sources"
+            if not schema.is_relative_to(source_root):
+                raise RuntimeError("Kafka probe schema is outside locked sources")
+            command.extend(("--volume", f"{schema}:/fixture.avsc:ro"))
+            schema_argument = ["/fixture.avsc"]
+        command.extend(
+            (
+                "wallet", "-cp", "/fixture.jar", PROBE_CLASS,
+                "kafka:9092", topic, str(partition), str(offset), *schema_argument,
+            )
+        )
+        try:
+            result = self.compose.run(*command, capture_output=True)
+        except subprocess.CalledProcessError as error:
+            raise RuntimeError("Kafka record probe failed") from error
+        return KafkaRecord.parse(result.stdout, topic, partition, offset)


## `test(fixtures): verify scoped Kafka probe execution`

diff --git a/tests/test_kafka_probe_runtime.py b/tests/test_kafka_probe_runtime.py
new file mode 100644
index 0000000..633868b
--- /dev/null
+++ b/tests/test_kafka_probe_runtime.py
@@ -0,0 +1,90 @@
+import base64
+import hashlib
+import json
+import pathlib
+import subprocess
+import tempfile
+import unittest
+
+from scripts.cold_gate.context import ColdGateContext
+from scripts.cold_gate.probe import KafkaProbe, PROBE_CLASS
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
+
+    def run(self, *arguments, **options):
+        self.calls.append((arguments, options))
+        if self.failure:
+            raise subprocess.CalledProcessError(1, arguments, stderr="sensitive")
+        value = b"record"
+        output = json.dumps(
+            {
+                "topic": "admin.action",
+                "partition": 1,
+                "offset": 4,
+                "key": "e2e-admin",
+                "valueBase64": base64.b64encode(value).decode(),
+                "valueSha256": hashlib.sha256(value).hexdigest(),
+                "headers": {"event-id": [base64.b64encode(b"id").decode()]},
+                "avro": {"outcome": "SUCCESS"},
+            }
+        )
+        return subprocess.CompletedProcess(arguments, 0, stdout=output)
+
+
+class KafkaProbeRuntimeTest(unittest.TestCase):
+    def fixture(self, root: pathlib.Path):
+        context = ColdGateContext.create(root, SHA, "00000001")
+        fixture_dir = context.runtime / "fixtures"
+        fixture_dir.mkdir()
+        jar = fixture_dir / "avro-fixture-publisher.jar"
+        jar.write_bytes(b"fixture")
+        schema_dir = context.runtime / "sources/admin/src/main/avro"
+        schema_dir.mkdir(parents=True)
+        schema = schema_dir / "AdminActionRecorded.avsc"
+        schema.write_text("{}")
+        return context, jar, schema
+
+    def test_runs_the_staged_probe_with_one_locked_schema(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            context, jar, schema = self.fixture(pathlib.Path(temporary).resolve())
+            compose = FakeCompose(context)
+
+            record = KafkaProbe(context, compose, jar).read(
+                "admin.action", 1, 4, schema
+            )
+
+            arguments, options = compose.calls[0]
+            self.assertEqual(arguments[:5], ("run", "--rm", "--no-deps", "--entrypoint", "java"))
+            self.assertIn(f"{jar}:/fixture.jar:ro", arguments)
+            self.assertIn(f"{schema}:/fixture.avsc:ro", arguments)
+            self.assertIn(PROBE_CLASS, arguments)
+            self.assertEqual(arguments[-5:], ("kafka:9092", "admin.action", "1", "4", "/fixture.avsc"))
+            self.assertEqual(options, {"capture_output": True})
+            self.assertEqual(record.key, "e2e-admin")
+            self.assertEqual(record.avro, {"outcome": "SUCCESS"})
+
+    def test_rejects_external_schemas_and_hides_probe_failures(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            root = pathlib.Path(temporary).resolve()
+            context, jar, _schema = self.fixture(root)
+            external = root / "external.avsc"
+            external.write_text("{}")
+            with self.assertRaisesRegex(RuntimeError, "outside locked"):
+                KafkaProbe(context, FakeCompose(context), jar).read("admin.action", 1, 4, external)
+            with self.assertRaisesRegex(RuntimeError, "probe failed") as captured:
+                KafkaProbe(context, FakeCompose(context, failure=True), jar).read(
+                    "match.result.DLT", 2, 0
+                )
+            self.assertNotIn("sensitive", str(captured.exception))
+
+
+if __name__ == "__main__":
+    unittest.main()
