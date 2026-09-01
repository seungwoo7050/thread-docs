# Kafka 토픽과 소비자 준비 계약

## `build(kafka): persist single-node KRaft metadata`

diff --git a/compose.yaml b/compose.yaml
index 6e5a546..b4eb850 100644
--- a/compose.yaml
+++ b/compose.yaml
@@ -17,8 +17,37 @@ services:
       retries: 30
     networks: [backend]
 
+  kafka:
+    image: apache/kafka:3.8.0
+    environment:
+      CLUSTER_ID: MkU3OEVBNTcwNTJENDM2Qk
+      KAFKA_NODE_ID: "1"
+      KAFKA_PROCESS_ROLES: broker,controller
+      KAFKA_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
+      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
+      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
+      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
+      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
+      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
+      KAFKA_LOG_DIRS: /var/lib/kafka/data
+      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: "1"
+      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: "1"
+      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: "1"
+      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: "0"
+    volumes:
+      - kafka-data:/var/lib/kafka/data
+    healthcheck:
+      test:
+        - CMD-SHELL
+        - /opt/kafka/bin/kafka-metadata-quorum.sh --bootstrap-server localhost:9092 describe --status >/dev/null
+      interval: 3s
+      timeout: 5s
+      retries: 40
+    networks: [backend]
+
 volumes:
   postgres-data:
+  kafka-data:
 
 networks:
   backend:


## `test(kafka): verify persistent KRaft restart`

diff --git a/tests/test_kraft_persistence.py b/tests/test_kraft_persistence.py
new file mode 100644
index 0000000..2574551
--- /dev/null
+++ b/tests/test_kraft_persistence.py
@@ -0,0 +1,69 @@
+from tests.compose_fixture import ComposeFixture
+
+
+class KraftPersistenceTest(ComposeFixture):
+    def test_preserves_cluster_metadata_across_container_recreation(self) -> None:
+        started = self.compose("up", "--detach", "--wait", "kafka")
+        self.assertEqual(started.returncode, 0, started.stderr)
+
+        quorum = self.kafka(
+            "kafka-metadata-quorum.sh",
+            "--bootstrap-server",
+            "localhost:9092",
+            "describe",
+            "--status",
+        )
+        self.assertEqual(quorum.returncode, 0, quorum.stderr)
+        self.assertIn("ClusterId:              MkU3OEVBNTcwNTJENDM2Qk", quorum.stdout)
+        self.assertRegex(quorum.stdout, r"LeaderId:\s+1")
+
+        created = self.kafka(
+            "kafka-topics.sh",
+            "--bootstrap-server",
+            "localhost:9092",
+            "--create",
+            "--topic",
+            "kraft.persistence.probe",
+            "--partitions",
+            "3",
+            "--replication-factor",
+            "1",
+        )
+        self.assertEqual(created.returncode, 0, created.stderr)
+        metadata_before = self.compose(
+            "exec", "--no-TTY", "kafka", "cat", "/var/lib/kafka/data/meta.properties"
+        )
+        self.assertEqual(metadata_before.returncode, 0, metadata_before.stderr)
+
+        stopped = self.compose("down")
+        self.assertEqual(stopped.returncode, 0, stopped.stderr)
+        restarted = self.compose("up", "--detach", "--wait", "kafka")
+        self.assertEqual(restarted.returncode, 0, restarted.stderr)
+
+        metadata_after = self.compose(
+            "exec", "--no-TTY", "kafka", "cat", "/var/lib/kafka/data/meta.properties"
+        )
+        described = self.kafka(
+            "kafka-topics.sh",
+            "--bootstrap-server",
+            "localhost:9092",
+            "--describe",
+            "--topic",
+            "kraft.persistence.probe",
+        )
+        self.assertEqual(metadata_after.returncode, 0, metadata_after.stderr)
+        self.assertEqual(metadata_after.stdout, metadata_before.stdout)
+        self.assertEqual(described.returncode, 0, described.stderr)
+        self.assertIn("PartitionCount: 3", described.stdout)
+        self.assertIn("ReplicationFactor: 1", described.stdout)
+
+    def kafka(self, command: str, *arguments: str):
+        return self.compose(
+            "exec", "--no-TTY", "kafka", f"/opt/kafka/bin/{command}", *arguments
+        )
+
+
+if __name__ == "__main__":
+    import unittest
+
+    unittest.main()


## `build(kafka): declare exact topic inventory`

diff --git a/docker/kafka/topics.manifest b/docker/kafka/topics.manifest
new file mode 100644
index 0000000..498212a
--- /dev/null
+++ b/docker/kafka/topics.manifest
@@ -0,0 +1,24 @@
+# topic|partitions|replication-factor|retention-ms
+wallet.debited.v1|3|1|-
+wallet.credited.v1|3|1|-
+wallet.debit-failed.v1|3|1|-
+risk.limit.violated|3|1|-
+risk.pattern.suspected|3|1|-
+odds.changed|3|1|-
+market.status.changed|3|1|-
+event.lifecycle|3|1|-
+match.result|3|1|-
+bet.placed.v1|3|1|-
+bet.settled.v1|3|1|-
+bet.voided.v1|3|1|-
+bet.resolution.revised.v1|3|1|-
+admin.action|3|1|-
+wallet.debited.v1.DLT|3|1|604800000
+wallet.debit-failed.v1.DLT|3|1|604800000
+odds.changed.DLT|3|1|604800000
+event.lifecycle.DLT|3|1|604800000
+match.result.DLT|3|1|604800000
+bet.placed.v1.DLT|3|1|604800000
+bet.settled.v1.DLT|3|1|604800000
+bet.voided.v1.DLT|3|1|604800000
+bet.resolution.revised.v1.DLT|3|1|604800000


## `test(kafka): verify exact topic inventory`

diff --git a/tests/test_topic_manifest.py b/tests/test_topic_manifest.py
new file mode 100644
index 0000000..7cd8a7d
--- /dev/null
+++ b/tests/test_topic_manifest.py
@@ -0,0 +1,64 @@
+import pathlib
+import unittest
+
+
+ROOT = pathlib.Path(__file__).resolve().parents[1]
+MANIFEST = ROOT / "docker/kafka/topics.manifest"
+SOURCES = {
+    "wallet.debited.v1",
+    "wallet.credited.v1",
+    "wallet.debit-failed.v1",
+    "risk.limit.violated",
+    "risk.pattern.suspected",
+    "odds.changed",
+    "market.status.changed",
+    "event.lifecycle",
+    "match.result",
+    "bet.placed.v1",
+    "bet.settled.v1",
+    "bet.voided.v1",
+    "bet.resolution.revised.v1",
+    "admin.action",
+}
+DLTS = {
+    "wallet.debited.v1.DLT",
+    "wallet.debit-failed.v1.DLT",
+    "odds.changed.DLT",
+    "event.lifecycle.DLT",
+    "match.result.DLT",
+    "bet.placed.v1.DLT",
+    "bet.settled.v1.DLT",
+    "bet.voided.v1.DLT",
+    "bet.resolution.revised.v1.DLT",
+}
+
+
+class TopicManifestTest(unittest.TestCase):
+    def test_declares_the_exact_source_and_uppercase_dlt_inventory(self) -> None:
+        rows = [
+            line.split("|")
+            for line in MANIFEST.read_text().splitlines()
+            if line and not line.startswith("#")
+        ]
+        self.assertTrue(all(len(row) == 4 for row in rows))
+        self.assertEqual(len(rows), 23)
+        self.assertEqual(len({row[0] for row in rows}), len(rows))
+
+        sources = {row[0] for row in rows if not row[0].endswith(".DLT")}
+        dlts = {row[0] for row in rows if row[0].endswith(".DLT")}
+        self.assertEqual(sources, SOURCES)
+        self.assertEqual(dlts, DLTS)
+
+        for topic, partitions, replication, retention in rows:
+            with self.subTest(topic=topic):
+                self.assertEqual(partitions, "3")
+                self.assertEqual(replication, "1")
+                if topic.endswith(".DLT"):
+                    self.assertGreaterEqual(int(retention), 604_800_000)
+                    self.assertIn(topic.removesuffix(".DLT"), SOURCES)
+                else:
+                    self.assertEqual(retention, "-")
+
+
+if __name__ == "__main__":
+    unittest.main()


## `build(kafka): provision topics without mutation`

diff --git a/compose.yaml b/compose.yaml
index 890f4ac..8295b66 100644
--- a/compose.yaml
+++ b/compose.yaml
@@ -46,6 +46,19 @@ services:
       retries: 40
     networks: [backend]
 
+  topic-init:
+    image: apache/kafka:3.8.0
+    entrypoint: ["/bin/sh", "/opt/sportsbook/init-topics.sh"]
+    environment:
+      KAFKA_BOOTSTRAP_SERVERS: kafka:9092
+    volumes:
+      - ./docker/kafka/init-topics.sh:/opt/sportsbook/init-topics.sh:ro
+      - ./docker/kafka/topics.manifest:/opt/sportsbook/topics.manifest:ro
+    depends_on:
+      kafka:
+        condition: service_healthy
+    networks: [backend]
+
 volumes:
   postgres-data:
   kafka-data:
diff --git a/docker/kafka/init-topics.sh b/docker/kafka/init-topics.sh
new file mode 100755
index 0000000..b58ead6
--- /dev/null
+++ b/docker/kafka/init-topics.sh
@@ -0,0 +1,63 @@
+#!/bin/sh
+set -eu
+
+BOOTSTRAP=${KAFKA_BOOTSTRAP_SERVERS:-kafka:9092}
+MANIFEST=${TOPIC_MANIFEST:-/opt/sportsbook/topics.manifest}
+BIN=/opt/kafka/bin
+WORK=$(mktemp -d)
+trap 'rm -rf "$WORK"' EXIT INT TERM
+
+fail() {
+  printf 'topic-init: %s\n' "$1" >&2
+  exit 1
+}
+
+awk -F'|' '
+  NF && $1 !~ /^#/ {
+    if (NF != 4 || $2 !~ /^[0-9]+$/ || $3 !~ /^[0-9]+$/) exit 2
+    if (seen[$1]++) exit 3
+    print
+  }
+' "$MANIFEST" >"$WORK/manifest" || fail "invalid manifest"
+cut -d'|' -f1 "$WORK/manifest" >"$WORK/names"
+"$BIN/kafka-topics.sh" --bootstrap-server "$BOOTSTRAP" --list >"$WORK/existing"
+
+while IFS= read -r topic; do
+  case "$topic" in
+    ""|__*) ;;
+    *) grep -Fqx "$topic" "$WORK/names" || fail "undeclared topic exists: $topic" ;;
+  esac
+done <"$WORK/existing"
+
+: >"$WORK/missing"
+while IFS='|' read -r topic partitions replication retention; do
+  if ! grep -Fqx "$topic" "$WORK/existing"; then
+    printf '%s|%s|%s|%s\n' "$topic" "$partitions" "$replication" "$retention" \
+      >>"$WORK/missing"
+    continue
+  fi
+
+  "$BIN/kafka-topics.sh" --bootstrap-server "$BOOTSTRAP" --describe --topic "$topic" \
+    >"$WORK/describe"
+  actual_partitions=$(sed -n 's/.*PartitionCount: \([0-9][0-9]*\).*/\1/p' "$WORK/describe" | head -n 1)
+  actual_replication=$(sed -n 's/.*ReplicationFactor: \([0-9][0-9]*\).*/\1/p' "$WORK/describe" | head -n 1)
+  [ "$actual_partitions" = "$partitions" ] || fail "$topic partition mismatch"
+  [ "$actual_replication" = "$replication" ] || fail "$topic replication mismatch"
+
+  if [ "$retention" != "-" ]; then
+    "$BIN/kafka-configs.sh" --bootstrap-server "$BOOTSTRAP" --entity-type topics \
+      --entity-name "$topic" --describe >"$WORK/config"
+    actual_retention=$(sed -n 's/.*retention.ms=\([0-9][0-9]*\).*/\1/p' "$WORK/config" | head -n 1)
+    [ -n "$actual_retention" ] || fail "$topic retention is not explicit"
+    [ "$actual_retention" -ge "$retention" ] || fail "$topic retention is too short"
+  fi
+done <"$WORK/manifest"
+
+while IFS='|' read -r topic partitions replication retention; do
+  set -- --bootstrap-server "$BOOTSTRAP" --create --topic "$topic" \
+    --partitions "$partitions" --replication-factor "$replication"
+  if [ "$retention" != "-" ]; then
+    set -- "$@" --config "retention.ms=$retention"
+  fi
+  "$BIN/kafka-topics.sh" "$@"
+done <"$WORK/missing"


## `test(kafka): verify idempotent topic initialization`

diff --git a/tests/kafka_fixture.py b/tests/kafka_fixture.py
new file mode 100644
index 0000000..c7fb9d7
--- /dev/null
+++ b/tests/kafka_fixture.py
@@ -0,0 +1,55 @@
+import re
+
+from tests.compose_fixture import ComposeFixture
+
+
+class KafkaFixture(ComposeFixture):
+    def setUp(self) -> None:
+        super().setUp()
+        started = self.compose("up", "--detach", "--wait", "kafka")
+        self.assertEqual(started.returncode, 0, started.stderr)
+
+    def kafka(self, command: str, *arguments: str):
+        return self.compose(
+            "exec", "--no-TTY", "kafka", f"/opt/kafka/bin/{command}", *arguments
+        )
+
+    def initialize_topics(self):
+        return self.compose("run", "--rm", "--no-deps", "topic-init")
+
+    def topic_state(self) -> dict[str, tuple[int, int, int | None]]:
+        listed = self.kafka(
+            "kafka-topics.sh", "--bootstrap-server", "localhost:9092", "--list"
+        )
+        self.assertEqual(listed.returncode, 0, listed.stderr)
+        topics = sorted(topic for topic in listed.stdout.splitlines() if not topic.startswith("__"))
+        state = {}
+        for topic in topics:
+            described = self.kafka(
+                "kafka-topics.sh",
+                "--bootstrap-server",
+                "localhost:9092",
+                "--describe",
+                "--topic",
+                topic,
+            )
+            self.assertEqual(described.returncode, 0, described.stderr)
+            partitions = int(re.search(r"PartitionCount: (\d+)", described.stdout).group(1))
+            replication = int(re.search(r"ReplicationFactor: (\d+)", described.stdout).group(1))
+            retention = self.topic_retention(topic) if topic.endswith(".DLT") else None
+            state[topic] = (partitions, replication, retention)
+        return state
+
+    def topic_retention(self, topic: str) -> int:
+        config = self.kafka(
+            "kafka-configs.sh",
+            "--bootstrap-server",
+            "localhost:9092",
+            "--entity-type",
+            "topics",
+            "--entity-name",
+            topic,
+            "--describe",
+        )
+        self.assertEqual(config.returncode, 0, config.stderr)
+        return int(re.search(r"retention.ms=(\d+)", config.stdout).group(1))
diff --git a/tests/test_topic_initialization.py b/tests/test_topic_initialization.py
new file mode 100644
index 0000000..7ff6465
--- /dev/null
+++ b/tests/test_topic_initialization.py
@@ -0,0 +1,30 @@
+from tests.kafka_fixture import KafkaFixture
+from tests.test_topic_manifest import DLTS, SOURCES
+
+
+class TopicInitializationTest(KafkaFixture):
+    def test_repeated_initialization_preserves_exact_broker_state(self) -> None:
+        first = self.initialize_topics()
+        self.assertEqual(first.returncode, 0, first.stderr)
+        first_state = self.topic_state()
+
+        second = self.initialize_topics()
+        self.assertEqual(second.returncode, 0, second.stderr)
+        second_state = self.topic_state()
+
+        self.assertEqual(second_state, first_state)
+        self.assertEqual(set(second_state), SOURCES | DLTS)
+        for topic, (partitions, replication, retention) in second_state.items():
+            with self.subTest(topic=topic):
+                self.assertEqual(partitions, 3)
+                self.assertEqual(replication, 1)
+                if topic in DLTS:
+                    self.assertGreaterEqual(retention, 604_800_000)
+                else:
+                    self.assertIsNone(retention)
+
+
+if __name__ == "__main__":
+    import unittest
+
+    unittest.main()


## `build(kafka): disable automatic topic creation`

diff --git a/compose.yaml b/compose.yaml
index b4eb850..890f4ac 100644
--- a/compose.yaml
+++ b/compose.yaml
@@ -25,6 +25,7 @@ services:
       KAFKA_PROCESS_ROLES: broker,controller
       KAFKA_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
       KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
+      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "false"
       KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
       KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
       KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT


## `test(kafka): verify broker rejects undeclared topics`

diff --git a/tests/test_kafka_autocreate.py b/tests/test_kafka_autocreate.py
new file mode 100644
index 0000000..92d5cda
--- /dev/null
+++ b/tests/test_kafka_autocreate.py
@@ -0,0 +1,46 @@
+from tests.compose_fixture import ComposeFixture
+
+
+class KafkaAutoCreateTest(ComposeFixture):
+    def test_rejects_production_to_an_undeclared_topic(self) -> None:
+        started = self.compose("up", "--detach", "--wait", "kafka")
+        self.assertEqual(started.returncode, 0, started.stderr)
+
+        produced = self.compose(
+            "exec",
+            "--no-TTY",
+            "kafka",
+            "/opt/kafka/bin/kafka-producer-perf-test.sh",
+            "--topic",
+            "undeclared.contract.probe",
+            "--num-records",
+            "1",
+            "--record-size",
+            "1",
+            "--throughput",
+            "-1",
+            "--producer-props",
+            "bootstrap.servers=localhost:9092",
+            "max.block.ms=3000",
+        )
+        self.assertIn("0 records sent", produced.stdout)
+        self.assertIn("UNKNOWN_TOPIC_OR_PARTITION", produced.stderr)
+        self.assertIn("not present in metadata", produced.stderr)
+
+        listed = self.compose(
+            "exec",
+            "--no-TTY",
+            "kafka",
+            "/opt/kafka/bin/kafka-topics.sh",
+            "--bootstrap-server",
+            "localhost:9092",
+            "--list",
+        )
+        self.assertEqual(listed.returncode, 0, listed.stderr)
+        self.assertNotIn("undeclared.contract.probe", listed.stdout.splitlines())
+
+
+if __name__ == "__main__":
+    import unittest
+
+    unittest.main()


## `test(kafka): fail closed on topic mismatch`

diff --git a/tests/kafka_fixture.py b/tests/kafka_fixture.py
index c7fb9d7..2ab6241 100644
--- a/tests/kafka_fixture.py
+++ b/tests/kafka_fixture.py
@@ -17,6 +17,22 @@ class KafkaFixture(ComposeFixture):
     def initialize_topics(self):
         return self.compose("run", "--rm", "--no-deps", "topic-init")
 
+    def create_topic(self, topic: str, partitions: int, retention: int | None = None):
+        arguments = [
+            "--bootstrap-server",
+            "localhost:9092",
+            "--create",
+            "--topic",
+            topic,
+            "--partitions",
+            str(partitions),
+            "--replication-factor",
+            "1",
+        ]
+        if retention is not None:
+            arguments.extend(["--config", f"retention.ms={retention}"])
+        return self.kafka("kafka-topics.sh", *arguments)
+
     def topic_state(self) -> dict[str, tuple[int, int, int | None]]:
         listed = self.kafka(
             "kafka-topics.sh", "--bootstrap-server", "localhost:9092", "--list"
diff --git a/tests/test_topic_mismatch.py b/tests/test_topic_mismatch.py
new file mode 100644
index 0000000..61a1f1d
--- /dev/null
+++ b/tests/test_topic_mismatch.py
@@ -0,0 +1,35 @@
+from tests.kafka_fixture import KafkaFixture
+
+
+class TopicMismatchTest(KafkaFixture):
+    def test_partition_mismatch_fails_before_creating_other_topics(self) -> None:
+        created = self.create_topic("wallet.debited.v1", partitions=2)
+        self.assertEqual(created.returncode, 0, created.stderr)
+        before = self.topic_state()
+
+        initialized = self.initialize_topics()
+
+        self.assertNotEqual(initialized.returncode, 0)
+        self.assertIn("wallet.debited.v1 partition mismatch", initialized.stderr)
+        self.assertEqual(self.topic_state(), before)
+        self.assertEqual(set(before), {"wallet.debited.v1"})
+
+    def test_short_dlt_retention_fails_without_altering_the_topic(self) -> None:
+        created = self.create_topic(
+            "match.result.DLT", partitions=3, retention=60_000
+        )
+        self.assertEqual(created.returncode, 0, created.stderr)
+        before = self.topic_state()
+
+        initialized = self.initialize_topics()
+
+        self.assertNotEqual(initialized.returncode, 0)
+        self.assertIn("match.result.DLT retention is too short", initialized.stderr)
+        self.assertEqual(self.topic_state(), before)
+        self.assertEqual(before["match.result.DLT"], (3, 1, 60_000))
+
+
+if __name__ == "__main__":
+    import unittest
+
+    unittest.main()


## `build(startup): gate the service dependency DAG`

diff --git a/compose.yaml b/compose.yaml
index bd24633..0438c90 100644
--- a/compose.yaml
+++ b/compose.yaml
@@ -74,8 +74,38 @@ services:
     depends_on:
       kafka:
         condition: service_healthy
+      postgres:
+        condition: service_healthy
+      redis-risk:
+        condition: service_healthy
+      redis-odds:
+        condition: service_healthy
+      redis-wallet:
+        condition: service_healthy
+      redis-gateway:
+        condition: service_healthy
     networks: [backend]
 
+  secret-preflight:
+    image: python:3.12-alpine
+    entrypoint: ["python3", "/opt/sportsbook/check-secrets.py"]
+    environment:
+      GATEWAY_BETTING_API_KEY: ${GATEWAY_BETTING_API_KEY:-}
+      GATEWAY_WALLET_API_KEY: ${GATEWAY_WALLET_API_KEY:-}
+      BETTING_RISK_API_KEY: ${BETTING_RISK_API_KEY:-}
+      BETTING_WALLET_API_KEY: ${BETTING_WALLET_API_KEY:-}
+      SETTLEMENT_WALLET_API_KEY: ${SETTLEMENT_WALLET_API_KEY:-}
+      ADMIN_WALLET_API_KEY: ${ADMIN_WALLET_API_KEY:-}
+      ADMIN_RISK_API_KEY: ${ADMIN_RISK_API_KEY:-}
+      ADMIN_ODDS_FEED_API_KEY: ${ADMIN_ODDS_FEED_API_KEY:-}
+      ADMIN_SETTLEMENT_API_KEY: ${ADMIN_SETTLEMENT_API_KEY:-}
+      WALLET_PLATFORM_API_KEY: ${WALLET_PLATFORM_API_KEY:-}
+      INTERNAL_PLATFORM_API_KEY: ${INTERNAL_PLATFORM_API_KEY:-}
+    volumes:
+      - ./scripts/check-secrets.py:/opt/sportsbook/check-secrets.py:ro
+      - ./config/required-secrets.txt:/opt/config/required-secrets.txt:ro
+    network_mode: none
+
   redis-risk:
     <<: *redis-service
     volumes:
@@ -131,6 +161,8 @@ services:
         condition: service_healthy
       topic-init:
         condition: service_completed_successfully
+      secret-preflight:
+        condition: service_completed_successfully
     healthcheck:
       test: ["CMD-SHELL", "curl --fail --silent --show-error http://localhost:8081/actuator/health >/dev/null"]
       interval: 3s
@@ -162,6 +194,8 @@ services:
         condition: service_healthy
       topic-init:
         condition: service_completed_successfully
+      secret-preflight:
+        condition: service_completed_successfully
     healthcheck:
       test: ["CMD-SHELL", "curl --fail --silent --show-error http://localhost:8083/actuator/health/readiness >/dev/null"]
       interval: 3s
@@ -195,6 +229,8 @@ services:
         condition: service_healthy
       topic-init:
         condition: service_completed_successfully
+      secret-preflight:
+        condition: service_completed_successfully
     healthcheck:
       test: ["CMD-SHELL", "curl --fail --silent --show-error http://localhost:8085/actuator/health/readiness >/dev/null"]
       interval: 3s
@@ -299,6 +335,25 @@ services:
     networks: [backend]
     restart: unless-stopped
 
+  consumer-assignment:
+    image: apache/kafka:3.8.0
+    entrypoint: ["/bin/sh", "/opt/sportsbook/wait-consumer-assignments.sh"]
+    environment:
+      KAFKA_BOOTSTRAP_SERVERS: kafka:9092
+      ASSIGNMENT_TIMEOUT_SECONDS: "180"
+    volumes:
+      - ./docker/kafka/wait-consumer-assignments.sh:/opt/sportsbook/wait-consumer-assignments.sh:ro
+    depends_on:
+      kafka:
+        condition: service_healthy
+      topic-init:
+        condition: service_completed_successfully
+      betting:
+        condition: service_healthy
+      gateway:
+        condition: service_healthy
+    networks: [backend]
+
   settlement:
     build:
       context: ./docker
@@ -334,6 +389,8 @@ services:
         condition: service_completed_successfully
       wallet:
         condition: service_healthy
+      consumer-assignment:
+        condition: service_completed_successfully
     healthcheck:
       test: ["CMD-SHELL", "curl --fail --silent --show-error http://localhost:8084/actuator/health/readiness >/dev/null"]
       interval: 3s
diff --git a/docker/kafka/wait-consumer-assignments.sh b/docker/kafka/wait-consumer-assignments.sh
new file mode 100755
index 0000000..9ebcaf4
--- /dev/null
+++ b/docker/kafka/wait-consumer-assignments.sh
@@ -0,0 +1,43 @@
+#!/bin/sh
+set -eu
+
+BOOTSTRAP=${KAFKA_BOOTSTRAP_SERVERS:-kafka:9092}
+CONSUMER_GROUPS=${KAFKA_CONSUMER_GROUPS:-/opt/kafka/bin/kafka-consumer-groups.sh}
+TIMEOUT=${ASSIGNMENT_TIMEOUT_SECONDS:-180}
+POLL=${ASSIGNMENT_POLL_SECONDS:-2}
+WORK=$(mktemp -d)
+trap 'rm -rf "$WORK"' EXIT INT TERM
+
+cat >"$WORK/expected" <<'EOF'
+bet.resolution.revised.v1:0
+bet.resolution.revised.v1:1
+bet.resolution.revised.v1:2
+bet.settled.v1:0
+bet.settled.v1:1
+bet.settled.v1:2
+bet.voided.v1:0
+bet.voided.v1:1
+bet.voided.v1:2
+EOF
+
+group_ready() {
+  group=$1
+  "$CONSUMER_GROUPS" --bootstrap-server "$BOOTSTRAP" --describe --group "$group" \
+    >"$WORK/$group.out" 2>/dev/null || return 1
+  awk -v group="$group" '
+    $1 == group && $7 != "-" { print $2 ":" $3 }
+  ' "$WORK/$group.out" | sort -u >"$WORK/$group.actual"
+  cmp -s "$WORK/expected" "$WORK/$group.actual"
+}
+
+deadline=$(( $(date +%s) + TIMEOUT ))
+while [ "$(date +%s)" -lt "$deadline" ]; do
+  if group_ready gateway-bets && group_ready betting-resolution; then
+    printf 'consumer-assignment: gateway-bets=9 betting-resolution=9\n'
+    exit 0
+  fi
+  sleep "$POLL"
+done
+
+printf 'consumer-assignment: timed out waiting for exact active assignments\n' >&2
+exit 1


