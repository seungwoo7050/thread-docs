## `test(e2e): bind replay invariance oracle`

diff --git a/e2e/runtime.py b/e2e/runtime.py
index 5f4200a..63b5157 100644
--- a/e2e/runtime.py
+++ b/e2e/runtime.py
@@ -7,6 +7,7 @@ from e2e.bet_api import BetApi
 from e2e.correction_oracles import CorrectionOracles
 from e2e.model import ScenarioIds
 from e2e.placement_oracles import PlacementOracles
+from e2e.replay_oracle import ReplayOracle
 from e2e.settlement_admin_api import SettlementAdminApi
 from e2e.wallet_api import WalletApi
 from scripts.cold_gate.build import ReleaseArtifacts
@@ -60,6 +61,7 @@ class E2eRuntime:
         self.base = BaseOracles(database)
         self.placements = PlacementOracles(database)
         self.corrections = CorrectionOracles(database)
+        self.replays = ReplayOracle(database, self.base)
         self.odds = RedisClient(compose, "redis-odds")
         self.risk = RedisClient(compose, "redis-risk")
         self.kafka = KafkaAdmin(compose)


## `test(e2e): wait for exact consumer progress`

diff --git a/e2e/kafka_barrier.py b/e2e/kafka_barrier.py
new file mode 100644
index 0000000..5f7980a
--- /dev/null
+++ b/e2e/kafka_barrier.py
@@ -0,0 +1,21 @@
+from __future__ import annotations
+
+from e2e.runtime import E2eRuntime
+from scripts.cold_gate.fixture_receipt import FixtureReceipt
+from scripts.cold_gate.polling import poll_until
+
+
+def wait_consumed(
+    runtime: E2eRuntime,
+    group: str,
+    receipt: FixtureReceipt,
+    *,
+    timeout: float = 60,
+) -> None:
+    poll_until(
+        f"{group} consumption of {receipt.topic}",
+        lambda: runtime.kafka.committed_offset(group, receipt.topic, receipt.partition),
+        lambda offset: offset > receipt.offset,
+        timeout=timeout,
+        interval=0.5,
+    )


## `test(e2e): verify replay invariance`

diff --git a/e2e/scenario_11_replay_invariance.py b/e2e/scenario_11_replay_invariance.py
new file mode 100644
index 0000000..88eb95b
--- /dev/null
+++ b/e2e/scenario_11_replay_invariance.py
@@ -0,0 +1,44 @@
+from __future__ import annotations
+
+import time
+
+from e2e.assertions import wait_fields
+from e2e.kafka_barrier import wait_consumed
+from e2e.model import ScenarioIds
+from e2e.runtime import E2eRuntime
+from e2e.terminal import wait_base_settlement
+
+
+NAME = "replay-invariance"
+
+
+def run(runtime: E2eRuntime) -> None:
+    fixture = ScenarioIds.create(11)
+    runtime.seed(fixture)
+    placement = runtime.bets.place(fixture, runtime.user_token(fixture))
+    wait_fields(
+        "replay placement projection",
+        lambda: runtime.base.settlement(placement.bet_id),
+        {"status": "PENDING"},
+        terminal={"status": frozenset({"SETTLED", "VOIDED"})},
+    )
+    payload = fixture.match_result("WON", int(time.time() * 1000) - 5_000)
+    original = runtime.fixtures.publish("MatchResult", payload)
+    wait_consumed(runtime, "settlement-service", original)
+    wait_base_settlement(runtime, fixture, placement.bet_id, "WON", 20_000, 110_000)
+    before = runtime.replays.snapshot(fixture, placement.bet_id)
+
+    for _attempt in range(3):
+        replay = runtime.fixtures.publish("MatchResult", payload)
+        if replay.sha256 != original.sha256:
+            raise RuntimeError("replayed MatchResult bytes drifted")
+        wait_consumed(runtime, "settlement-service", replay)
+    after = runtime.replays.snapshot(fixture, placement.bet_id)
+    if after != before:
+        raise RuntimeError("replayed MatchResult changed durable projections")
+
+    candidates = runtime.corrections.candidates(fixture.event)
+    if len(candidates) != 1 or candidates[0]["state"] != "ACCEPTED":
+        raise RuntimeError("replayed MatchResult changed candidate identity")
+    if runtime.corrections.revision(placement.bet_id) is not None:
+        raise RuntimeError("replayed MatchResult created a correction")


## `test(e2e): fence replay wallet receipts`

diff --git a/e2e/replay_oracle.py b/e2e/replay_oracle.py
index 31825f6..12a954f 100644
--- a/e2e/replay_oracle.py
+++ b/e2e/replay_oracle.py
@@ -12,6 +12,18 @@ class ReplayOracle:
         self.database = database
         self.base = base
 
+    def wallet_receipt(self, bet_id: str) -> dict[str, str]:
+        bet = uuid_literal(bet_id)
+        return self.database.one(
+            "betting",
+            f"""
+            SELECT count(*)::text AS receipt_count,
+                   count(*) FILTER (WHERE processed_at IS NOT NULL)::text AS processed
+            FROM wallet_event_receipt
+            WHERE bet_id = {bet}
+            """,
+        )
+
     def snapshot(self, fixture: ScenarioIds, bet_id: str) -> str:
         bet = uuid_literal(bet_id)
         event = uuid_literal(fixture.event)
diff --git a/e2e/scenario_11_replay_invariance.py b/e2e/scenario_11_replay_invariance.py
index 88eb95b..9b27a40 100644
--- a/e2e/scenario_11_replay_invariance.py
+++ b/e2e/scenario_11_replay_invariance.py
@@ -26,6 +26,11 @@ def run(runtime: E2eRuntime) -> None:
     original = runtime.fixtures.publish("MatchResult", payload)
     wait_consumed(runtime, "settlement-service", original)
     wait_base_settlement(runtime, fixture, placement.bet_id, "WON", 20_000, 110_000)
+    wait_fields(
+        "replay Wallet receipt",
+        lambda: runtime.replays.wallet_receipt(placement.bet_id),
+        {"receipt_count": "1", "processed": "1"},
+    )
     before = runtime.replays.snapshot(fixture, placement.bet_id)
 
     for _attempt in range(3):
diff --git a/tests/test_replay_oracle.py b/tests/test_replay_oracle.py
new file mode 100644
index 0000000..c048199
--- /dev/null
+++ b/tests/test_replay_oracle.py
@@ -0,0 +1,33 @@
+import unittest
+
+from e2e.replay_oracle import ReplayOracle
+
+
+BET = "66000000-0000-7000-8000-000000000011"
+
+
+class FakeDatabase:
+    def __init__(self) -> None:
+        self.calls: list[tuple[str, str]] = []
+
+    def one(self, service: str, statement: str) -> dict[str, str]:
+        self.calls.append((service, statement))
+        return {"receipt_count": "1", "processed": "1"}
+
+
+class ReplayOracleTest(unittest.TestCase):
+    def test_reads_one_processed_wallet_receipt_for_the_bet(self) -> None:
+        database = FakeDatabase()
+
+        receipt = ReplayOracle(database, object()).wallet_receipt(BET)
+
+        self.assertEqual(receipt, {"receipt_count": "1", "processed": "1"})
+        self.assertEqual(database.calls[0][0], "betting")
+        statement = database.calls[0][1]
+        self.assertIn("FROM wallet_event_receipt", statement)
+        self.assertIn("processed_at IS NOT NULL", statement)
+        self.assertIn(BET, statement)
+
+
+if __name__ == "__main__":
+    unittest.main()


## `feat(fixtures): probe exact Kafka records`

diff --git a/fixtures/avro-publisher/src/main/java/com/sportsbook/orchestration/fixture/KafkaProbe.java b/fixtures/avro-publisher/src/main/java/com/sportsbook/orchestration/fixture/KafkaProbe.java
new file mode 100644
index 0000000..64d7fed
--- /dev/null
+++ b/fixtures/avro-publisher/src/main/java/com/sportsbook/orchestration/fixture/KafkaProbe.java
@@ -0,0 +1,96 @@
+package com.sportsbook.orchestration.fixture;
+
+import com.fasterxml.jackson.databind.ObjectMapper;
+import java.io.ByteArrayInputStream;
+import java.nio.charset.StandardCharsets;
+import java.nio.file.Path;
+import java.security.MessageDigest;
+import java.time.Duration;
+import java.util.ArrayList;
+import java.util.Base64;
+import java.util.LinkedHashMap;
+import java.util.List;
+import java.util.Map;
+import java.util.Properties;
+import java.util.UUID;
+import org.apache.avro.Schema;
+import org.apache.avro.generic.GenericDatumReader;
+import org.apache.avro.generic.GenericData;
+import org.apache.avro.generic.GenericRecord;
+import org.apache.avro.io.DecoderFactory;
+import org.apache.kafka.clients.consumer.ConsumerConfig;
+import org.apache.kafka.clients.consumer.ConsumerRecord;
+import org.apache.kafka.clients.consumer.KafkaConsumer;
+import org.apache.kafka.common.TopicPartition;
+import org.apache.kafka.common.serialization.ByteArrayDeserializer;
+
+public final class KafkaProbe {
+  private static final ObjectMapper JSON = new ObjectMapper();
+
+  private KafkaProbe() {}
+
+  public static void main(String[] arguments) throws Exception {
+    if (arguments.length < 4 || arguments.length > 5) {
+      throw new IllegalArgumentException(
+          "usage: <bootstrap> <topic> <partition> <offset> [avro-schema]");
+    }
+    int partition = Integer.parseInt(arguments[2]);
+    long offset = Long.parseLong(arguments[3]);
+    if (partition < 0 || partition > 2 || offset < 0) {
+      throw new IllegalArgumentException("partition or offset is out of range");
+    }
+    Path schema = arguments.length == 5 ? Path.of(arguments[4]) : null;
+    System.out.println(read(arguments[0], arguments[1], partition, offset, schema));
+  }
+
+  static String read(String bootstrap, String topic, int partition, long offset, Path schema)
+      throws Exception {
+    Properties properties = new Properties();
+    properties.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrap);
+    properties.put(ConsumerConfig.GROUP_ID_CONFIG, "fixture-probe-" + UUID.randomUUID());
+    properties.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
+    properties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, ByteArrayDeserializer.class);
+    properties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, ByteArrayDeserializer.class);
+    try (KafkaConsumer<byte[], byte[]> consumer = new KafkaConsumer<>(properties)) {
+      TopicPartition target = new TopicPartition(topic, partition);
+      consumer.assign(List.of(target));
+      consumer.seek(target, offset);
+      long deadline = System.nanoTime() + Duration.ofSeconds(30).toNanos();
+      while (System.nanoTime() < deadline) {
+        for (ConsumerRecord<byte[], byte[]> record : consumer.poll(Duration.ofMillis(250))) {
+          if (record.offset() == offset) {
+            return format(record, schema);
+          }
+        }
+      }
+      throw new IllegalStateException("record did not arrive before the probe deadline");
+    }
+  }
+
+  static String format(ConsumerRecord<byte[], byte[]> record, Path schemaPath) throws Exception {
+    Map<String, Object> output = new LinkedHashMap<>();
+    output.put("topic", record.topic());
+    output.put("partition", record.partition());
+    output.put("offset", record.offset());
+    output.put("key", new String(record.key(), StandardCharsets.UTF_8));
+    output.put("valueBase64", Base64.getEncoder().encodeToString(record.value()));
+    output.put("valueSha256", java.util.HexFormat.of().formatHex(
+        MessageDigest.getInstance("SHA-256").digest(record.value())));
+    Map<String, List<String>> headers = new LinkedHashMap<>();
+    record.headers().forEach(
+        header -> headers.computeIfAbsent(header.key(), ignored -> new ArrayList<>())
+            .add(Base64.getEncoder().encodeToString(header.value())));
+    output.put("headers", headers);
+    if (schemaPath != null) {
+      Schema schema = new Schema.Parser().parse(schemaPath.toFile());
+      ByteArrayInputStream input = new ByteArrayInputStream(record.value());
+      GenericRecord decoded = new GenericDatumReader<GenericRecord>(schema)
+          .read(null, DecoderFactory.get().directBinaryDecoder(input, null));
+      if (input.available() != 0) {
+        throw new IllegalArgumentException("Avro payload has trailing bytes");
+      }
+      output.put("avro", JSON.readTree(GenericData.get().toString(decoded)));
+    }
+    return JSON.writeValueAsString(output);
+  }
+}


## `test(gate): verify uppercase DLT offsets`

diff --git a/tests/test_cold_gate_kafka.py b/tests/test_cold_gate_kafka.py
index b6ddff3..e8d539b 100644
--- a/tests/test_cold_gate_kafka.py
+++ b/tests/test_cold_gate_kafka.py
@@ -89,6 +89,16 @@ settlement-service event.lifecycle 2 0 0 0 consumer /172.1 client
                 "betting-resolution", "bet.settled.v1"
             )
 
+    def test_reads_one_uppercase_dlt_end_offset(self) -> None:
+        admin = KafkaAdmin(FakeCompose("match.result.DLT:2:17\n"))
+
+        self.assertEqual(admin.end_offset("match.result.DLT", 2), 17)
+
+        with self.assertRaisesRegex(RuntimeError, "receipt"):
+            KafkaAdmin(FakeCompose("other:2:17\n")).end_offset("match.result.DLT", 2)
+        with self.assertRaisesRegex(ValueError, "outside"):
+            admin.end_offset("INVALID", 2)
+
 
 if __name__ == "__main__":
     unittest.main()


## `test(e2e): bind raw Kafka record probes`

diff --git a/e2e/runtime.py b/e2e/runtime.py
index 63b5157..89dda30 100644
--- a/e2e/runtime.py
+++ b/e2e/runtime.py
@@ -21,6 +21,7 @@ from scripts.cold_gate.http import HostHttpClient
 from scripts.cold_gate.jwt import JwtSigner
 from scripts.cold_gate.kafka import KafkaAdmin
 from scripts.cold_gate.polling import poll_until
+from scripts.cold_gate.probe import KafkaProbe
 from scripts.cold_gate.redis import RedisClient
 from scripts.cold_gate.secrets import RuntimeSecrets
 from scripts.cold_gate.stack import ColdStack
@@ -66,6 +67,7 @@ class E2eRuntime:
         self.risk = RedisClient(compose, "redis-risk")
         self.kafka = KafkaAdmin(compose)
         self.fixtures = FixturePublisher(context, compose, artifacts.fixture_jar)
+        self.probe = KafkaProbe(context, compose, artifacts.fixture_jar)
         self.chaos = ChaosClient(compose)
 
     def user_token(self, fixture: ScenarioIds) -> str:


## `test(e2e): verify partition-preserving DLT`

diff --git a/e2e/scenario_12_partition_dlt.py b/e2e/scenario_12_partition_dlt.py
new file mode 100644
index 0000000..2c125fb
--- /dev/null
+++ b/e2e/scenario_12_partition_dlt.py
@@ -0,0 +1,74 @@
+from __future__ import annotations
+
+from e2e.kafka_barrier import wait_consumed
+from e2e.model import ScenarioIds
+from e2e.runtime import E2eRuntime
+from scripts.cold_gate.polling import poll_until
+
+
+NAME = "partition-two-poison-dlt"
+DLT = "match.result.DLT"
+POISON_HASH = "76be8b528d0075f7aae98d6fa57a6d3c83ae480a8469e668d7b0af968995ac71"
+
+
+def run(runtime: E2eRuntime) -> None:
+    fixture = ScenarioIds.create(12)
+    start = runtime.kafka.end_offset(DLT, 2)
+    source = runtime.fixtures.poison_match_result(fixture.event)
+
+    def appended(end: int) -> bool:
+        if end > start + 1:
+            raise RuntimeError("poison scenario appended more than one DLT record")
+        return end == start + 1
+
+    poll_until(
+        "partition two DLT append",
+        lambda: runtime.kafka.end_offset(DLT, 2),
+        appended,
+        timeout=60,
+        interval=0.5,
+    )
+    record = runtime.probe.read(DLT, 2, start)
+    if (
+        source.partition != 2
+        or source.sha256 != POISON_HASH
+        or record.key != fixture.event
+        or record.value != b"\x80"
+        or record.value_sha256 != POISON_HASH
+    ):
+        raise RuntimeError("partition two DLT did not preserve record identity")
+    expected_headers = {
+        "settlement-dlt-original-topic",
+        "settlement-dlt-original-partition",
+        "settlement-dlt-original-offset",
+        "settlement-dlt-original-timestamp",
+        "settlement-dlt-consumer-group",
+        "settlement-dlt-exception-type",
+    }
+    if set(record.headers) != expected_headers or any(
+        len(values) != 1 for values in record.headers.values()
+    ):
+        raise RuntimeError("partition two DLT header inventory drifted")
+    if record.headers["settlement-dlt-original-topic"][0] != b"match.result":
+        raise RuntimeError("DLT original topic drifted")
+    if int.from_bytes(
+        record.headers["settlement-dlt-original-partition"][0], "big", signed=True
+    ) != 2:
+        raise RuntimeError("DLT original partition drifted")
+    if int.from_bytes(
+        record.headers["settlement-dlt-original-offset"][0], "big", signed=True
+    ) != source.offset:
+        raise RuntimeError("DLT original offset drifted")
+    if int.from_bytes(
+        record.headers["settlement-dlt-original-timestamp"][0], "big", signed=True
+    ) <= 0:
+        raise RuntimeError("DLT original timestamp is invalid")
+    if record.headers["settlement-dlt-consumer-group"][0] != b"settlement-service":
+        raise RuntimeError("DLT consumer group drifted")
+    try:
+        exception_type = record.headers["settlement-dlt-exception-type"][0].decode("utf-8")
+    except UnicodeDecodeError as error:
+        raise RuntimeError("DLT exception type is not UTF-8") from error
+    if not exception_type:
+        raise RuntimeError("DLT exception type is empty")
+    wait_consumed(runtime, "settlement-service", source)
