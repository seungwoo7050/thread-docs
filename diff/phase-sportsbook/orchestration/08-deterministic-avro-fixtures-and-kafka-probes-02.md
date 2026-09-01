## `test(fixtures): reject open-ended publisher input`

diff --git a/fixtures/avro-publisher/src/test/java/com/sportsbook/orchestration/fixture/FixtureArgumentsTest.java b/fixtures/avro-publisher/src/test/java/com/sportsbook/orchestration/fixture/FixtureArgumentsTest.java
new file mode 100644
index 0000000..ebde2d6
--- /dev/null
+++ b/fixtures/avro-publisher/src/test/java/com/sportsbook/orchestration/fixture/FixtureArgumentsTest.java
@@ -0,0 +1,69 @@
+package com.sportsbook.orchestration.fixture;
+
+import static org.junit.jupiter.api.Assertions.assertEquals;
+import static org.junit.jupiter.api.Assertions.assertNull;
+import static org.junit.jupiter.api.Assertions.assertThrows;
+
+import java.nio.file.Files;
+import java.nio.file.Path;
+import org.junit.jupiter.api.Test;
+import org.junit.jupiter.api.io.TempDir;
+
+class FixtureArgumentsTest {
+  private static final String EVENT_ID = "00000000-0000-0000-0000-0000000000ab";
+
+  @TempDir Path temporaryDirectory;
+
+  @Test
+  void parsesPublishWithOptionalLockedPartition() throws Exception {
+    Path json = writeEvent();
+
+    FixtureArguments automatic =
+        FixtureArguments.parse(
+            new String[] {"publish", "kafka:9092", "EventLifecycle", json.toString()});
+    FixtureArguments explicit =
+        FixtureArguments.parse(
+            new String[] {"publish", "kafka:9092", "EventLifecycle", json.toString(), "2"});
+
+    assertEquals("kafka:9092", automatic.bootstrapServers());
+    assertEquals("event.lifecycle", automatic.fixture().topic());
+    assertNull(automatic.fixture().partition());
+    assertEquals(2, explicit.fixture().partition());
+  }
+
+  @Test
+  void parsesOnlyTheFixedPoisonCommand() throws Exception {
+    FixtureArguments arguments =
+        FixtureArguments.parse(new String[] {"poison", "kafka:9092", EVENT_ID});
+
+    assertEquals("match.result", arguments.fixture().topic());
+    assertEquals(2, arguments.fixture().partition());
+  }
+
+  @Test
+  void rejectsOpenEndedOrMalformedCommands() {
+    assertThrows(
+        IllegalArgumentException.class,
+        () -> FixtureArguments.parse(new String[] {"raw", "kafka:9092", "topic", "key"}));
+    assertThrows(
+        IllegalArgumentException.class,
+        () -> FixtureArguments.parse(new String[] {"poison", "", EVENT_ID}));
+    assertThrows(
+        IllegalArgumentException.class,
+        () ->
+            FixtureArguments.parse(
+                new String[] {
+                  "publish", "kafka:9092", "EventLifecycle", "missing.json", "partition"
+                }));
+  }
+
+  private Path writeEvent() throws Exception {
+    return Files.writeString(
+        temporaryDirectory.resolve("event.json"),
+        """
+        {"eventId":"%s","status":"FINISHED",
+         "occurredAt":1700000000000,"scheduledStartAt":1699990000000}
+        """
+            .formatted(EVENT_ID));
+  }
+}


## `feat(fixtures): publish broker-acknowledged records`

diff --git a/fixtures/avro-publisher/src/main/java/com/sportsbook/orchestration/fixture/FixturePublisher.java b/fixtures/avro-publisher/src/main/java/com/sportsbook/orchestration/fixture/FixturePublisher.java
new file mode 100644
index 0000000..04e9786
--- /dev/null
+++ b/fixtures/avro-publisher/src/main/java/com/sportsbook/orchestration/fixture/FixturePublisher.java
@@ -0,0 +1,64 @@
+package com.sportsbook.orchestration.fixture;
+
+import java.security.MessageDigest;
+import java.security.NoSuchAlgorithmException;
+import java.util.HexFormat;
+import java.util.Properties;
+import java.util.concurrent.TimeUnit;
+import org.apache.kafka.clients.producer.KafkaProducer;
+import org.apache.kafka.clients.producer.Producer;
+import org.apache.kafka.clients.producer.ProducerConfig;
+import org.apache.kafka.clients.producer.RecordMetadata;
+import org.apache.kafka.common.serialization.ByteArraySerializer;
+import org.apache.kafka.common.serialization.StringSerializer;
+
+public final class FixturePublisher {
+  private FixturePublisher() {}
+
+  public static void main(String[] arguments) throws Exception {
+    FixtureArguments parsed = FixtureArguments.parse(arguments);
+    try (Producer<String, byte[]> producer =
+        new KafkaProducer<>(producerProperties(parsed.bootstrapServers()))) {
+      System.out.println(publish(producer, parsed.fixture()).format());
+    }
+  }
+
+  static PublicationReceipt publish(Producer<String, byte[]> producer, FixtureRecord fixture)
+      throws Exception {
+    RecordMetadata metadata = producer.send(fixture.producerRecord()).get(30, TimeUnit.SECONDS);
+    return new PublicationReceipt(
+        metadata.topic(),
+        fixture.key(),
+        metadata.partition(),
+        metadata.offset(),
+        sha256(fixture.payload()),
+        fixture.fingerprint());
+  }
+
+  static Properties producerProperties(String bootstrapServers) {
+    Properties properties = new Properties();
+    properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
+    properties.put(ProducerConfig.ACKS_CONFIG, "all");
+    properties.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, "true");
+    properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
+    properties.put(
+        ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, ByteArraySerializer.class.getName());
+    return properties;
+  }
+
+  private static String sha256(byte[] payload) {
+    try {
+      return HexFormat.of().formatHex(MessageDigest.getInstance("SHA-256").digest(payload));
+    } catch (NoSuchAlgorithmException exception) {
+      throw new IllegalStateException("SHA-256 is unavailable", exception);
+    }
+  }
+
+  record PublicationReceipt(
+      String topic, String key, int partition, long offset, String sha256, String fingerprint) {
+    String format() {
+      return "topic=%s\tkey=%s\tpartition=%d\toffset=%d\tsha256=%s\tfingerprint=%s"
+          .formatted(topic, key, partition, offset, sha256, fingerprint);
+    }
+  }
+}


## `test(fixtures): verify acknowledged publication receipts`

diff --git a/fixtures/avro-publisher/src/test/java/com/sportsbook/orchestration/fixture/FixturePublisherTest.java b/fixtures/avro-publisher/src/test/java/com/sportsbook/orchestration/fixture/FixturePublisherTest.java
new file mode 100644
index 0000000..7069ccd
--- /dev/null
+++ b/fixtures/avro-publisher/src/test/java/com/sportsbook/orchestration/fixture/FixturePublisherTest.java
@@ -0,0 +1,76 @@
+package com.sportsbook.orchestration.fixture;
+
+import static org.junit.jupiter.api.Assertions.assertArrayEquals;
+import static org.junit.jupiter.api.Assertions.assertEquals;
+import static org.junit.jupiter.api.Assertions.assertFalse;
+
+import java.util.List;
+import java.util.Properties;
+import java.util.Set;
+import org.apache.kafka.clients.producer.MockProducer;
+import org.apache.kafka.clients.producer.ProducerConfig;
+import org.apache.kafka.clients.producer.ProducerRecord;
+import org.apache.kafka.common.Cluster;
+import org.apache.kafka.common.Node;
+import org.apache.kafka.common.PartitionInfo;
+import org.apache.kafka.common.serialization.ByteArraySerializer;
+import org.apache.kafka.common.serialization.StringSerializer;
+import org.junit.jupiter.api.Test;
+
+class FixturePublisherTest {
+  private static final String EVENT_ID = "00000000-0000-0000-0000-0000000000ab";
+
+  @Test
+  void requiresAcknowledgedIdempotentPublication() {
+    Properties properties = FixturePublisher.producerProperties("kafka:9092");
+
+    assertEquals("kafka:9092", properties.get(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG));
+    assertEquals("all", properties.get(ProducerConfig.ACKS_CONFIG));
+    assertEquals("true", properties.get(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG));
+    assertEquals(
+        StringSerializer.class.getName(),
+        properties.get(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG));
+    assertEquals(
+        ByteArraySerializer.class.getName(),
+        properties.get(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG));
+  }
+
+  @Test
+  void waitsForBrokerMetadataAndEmitsASecretFreeReceipt() throws Exception {
+    MockProducer<String, byte[]> producer =
+        new MockProducer<>(cluster(), true, new StringSerializer(), new ByteArraySerializer());
+    FixtureRecord poison = FixtureRecord.poisonMatchResult(EVENT_ID);
+
+    FixturePublisher.PublicationReceipt receipt = FixturePublisher.publish(producer, poison);
+
+    ProducerRecord<String, byte[]> sent = producer.history().get(0);
+    assertEquals("match.result", sent.topic());
+    assertEquals(2, sent.partition());
+    assertEquals(EVENT_ID, sent.key());
+    assertArrayEquals(new byte[] {(byte) 0x80}, sent.value());
+    assertEquals("match.result", receipt.topic());
+    assertEquals(2, receipt.partition());
+    assertEquals(0, receipt.offset());
+    assertEquals(
+        "76be8b528d0075f7aae98d6fa57a6d3c83ae480a8469e668d7b0af968995ac71",
+        receipt.sha256());
+    assertFalse(receipt.format().contains("bootstrap"));
+    assertFalse(receipt.format().contains("payload"));
+  }
+
+  private static Cluster cluster() {
+    Node broker = new Node(0, "localhost", 9092);
+    List<PartitionInfo> partitions =
+        java.util.stream.IntStream.range(0, 3)
+            .mapToObj(
+                partition ->
+                    new PartitionInfo(
+                        "match.result",
+                        partition,
+                        broker,
+                        new Node[] {broker},
+                        new Node[] {broker}))
+            .toList();
+    return new Cluster("fixture", List.of(broker), partitions, Set.of(), Set.of());
+  }
+}


## `build(fixtures): stage executable publisher`

diff --git a/scripts/stage-fixture-publisher.sh b/scripts/stage-fixture-publisher.sh
new file mode 100755
index 0000000..f1d8fd8
--- /dev/null
+++ b/scripts/stage-fixture-publisher.sh
@@ -0,0 +1,70 @@
+#!/usr/bin/env bash
+set -euo pipefail
+
+SCRIPT_DIR=$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd -P)
+ROOT=$(git -C "$SCRIPT_DIR" rev-parse --show-toplevel)
+SOURCE_ROOT=${1:-$ROOT/.runtime/sources}
+MAVEN_REPO=${2:-$ROOT/.runtime/m2/repository}
+OUTPUT=${3:-$ROOT/.runtime/fixtures}
+PENDING=
+
+fail() {
+  printf 'fixture-stage: %s\n' "$1" >&2
+  exit 1
+}
+
+cleanup() {
+  local status=$?
+  [[ -z $PENDING || ! -f $PENDING ]] || rm -f "$PENDING"
+  exit "$status"
+}
+trap cleanup EXIT INT TERM
+
+[[ -d $SOURCE_ROOT && ! -L $SOURCE_ROOT ]] || fail "source root is not materialized"
+SOURCE_ROOT=$(cd "$SOURCE_ROOT" && pwd -P)
+[[ -d $MAVEN_REPO && ! -L $MAVEN_REPO ]] || fail "isolated Maven repository is missing"
+MAVEN_REPO=$(cd "$MAVEN_REPO" && pwd -P)
+[[ -d $OUTPUT && ! -L $OUTPUT ]] || fail "output directory is not owned"
+OUTPUT=$(cd "$OUTPUT" && pwd -P)
+SHARED=$SOURCE_ROOT/shared
+[[ -f $SHARED/.git && $(git -C "$SHARED" rev-parse HEAD) == \
+  f9de6bc1e533761ab4bb1454d8d4ab8175cdf001 ]] || fail "shared source mismatch"
+! git -C "$SHARED" symbolic-ref -q HEAD >/dev/null || fail "shared source is attached"
+[[ -z $(git -C "$SHARED" status --porcelain) ]] || fail "shared source is dirty"
+
+JAVA_BIN=${JAVA_HOME:+$JAVA_HOME/bin/}java
+JAVAC_BIN=${JAVA_HOME:+$JAVA_HOME/bin/}javac
+JAVA_MAJOR=$($JAVA_BIN -version 2>&1 | awk -F'[."]' '/version/ {print $2; exit}')
+JAVAC_MAJOR=$($JAVAC_BIN -version 2>&1 | awk '{print $2; exit}' | cut -d. -f1)
+[[ $JAVA_MAJOR == 17 && $JAVAC_MAJOR == 17 ]] || fail "Java 17 JDK is required"
+
+RUNNER=${MAVEN_RUNNER:-$SHARED/mvnw}
+[[ -x $RUNNER ]] || fail "Maven runner is not executable"
+RUNNER=$(cd "$(dirname "$RUNNER")" && pwd -P)/$(basename "$RUNNER")
+"$RUNNER" -B -f "$ROOT/fixtures/avro-publisher/pom.xml" \
+  "-Dmaven.repo.local=$MAVEN_REPO" clean package
+
+BUILT=$ROOT/fixtures/avro-publisher/target/avro-fixture-publisher.jar
+[[ -f $BUILT && ! -L $BUILT ]] || fail "shaded publisher is missing"
+PENDING=$(mktemp "$OUTPUT/.fixture.XXXXXX")
+cp "$BUILT" "$PENDING"
+MANIFEST=$(unzip -p "$PENDING" META-INF/MANIFEST.MF) || fail "manifest is missing"
+[[ $MANIFEST == *"Main-Class: com.sportsbook.orchestration.fixture.FixturePublisher"* ]] \
+  || fail "publisher Main-Class mismatch"
+INVENTORY=$(jar tf "$PENDING") || fail "publisher JAR is unreadable"
+for entry in \
+  com/sportsbook/orchestration/fixture/FixturePublisher.class \
+  com/sportsbook/protocol/event/EventLifecycle.class \
+  org/apache/kafka/clients/producer/KafkaProducer.class; do
+  grep -qx "$entry" <<<"$INVENTORY" || fail "publisher dependency is missing: $entry"
+done
+PROPERTIES=$(unzip -p "$PENDING" META-INF/maven/com.sportsbook/shared-protocol/pom.properties)
+[[ $PROPERTIES == *"version=1.0.0"* ]] || fail "shared protocol version mismatch"
+MAJOR=$(javap -verbose -classpath "$PENDING" \
+  com.sportsbook.orchestration.fixture.FixturePublisher | awk '/major version/ {print $3; exit}')
+[[ $MAJOR == 61 ]] || fail "publisher bytecode is not Java 17"
+
+mv -f "$PENDING" "$OUTPUT/avro-fixture-publisher.jar"
+PENDING=
+shasum -a 256 "$OUTPUT/avro-fixture-publisher.jar"
+trap - EXIT INT TERM


## `test(fixtures): verify exact shaded staging`

diff --git a/tests/test_fixture_staging.py b/tests/test_fixture_staging.py
new file mode 100644
index 0000000..8ac0d2d
--- /dev/null
+++ b/tests/test_fixture_staging.py
@@ -0,0 +1,37 @@
+import hashlib
+import zipfile
+
+from tests.fixture_staging_fixture import FixtureStagingFixture
+
+
+class FixtureStagingTest(FixtureStagingFixture):
+    def test_stages_only_the_shaded_java17_publisher(self) -> None:
+        result = self.stage()
+
+        self.assertEqual(result.returncode, 0, result.stderr)
+        staged = self.output / "avro-fixture-publisher.jar"
+        self.assertEqual([path.name for path in self.output.iterdir()], [staged.name])
+        self.assertIn(hashlib.sha256(staged.read_bytes()).hexdigest(), result.stdout)
+        with zipfile.ZipFile(staged) as archive:
+            names = set(archive.namelist())
+            self.assertIn(
+                "com/sportsbook/orchestration/fixture/FixturePublisher.class", names
+            )
+            self.assertIn(
+                "com/sportsbook/protocol/event/EventLifecycle.class", names
+            )
+            self.assertIn(
+                "org/apache/kafka/clients/producer/KafkaProducer.class", names
+            )
+            self.assertEqual(
+                archive.read(
+                    "META-INF/maven/com.sportsbook/shared-protocol/pom.properties"
+                ),
+                b"version=1.0.0\n",
+            )
+
+
+if __name__ == "__main__":
+    import unittest
+
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


## `build(fixtures): stage the Kafka record probe`

diff --git a/scripts/stage-fixture-publisher.sh b/scripts/stage-fixture-publisher.sh
index f1d8fd8..7d6d4b0 100755
--- a/scripts/stage-fixture-publisher.sh
+++ b/scripts/stage-fixture-publisher.sh
@@ -54,6 +54,7 @@ MANIFEST=$(unzip -p "$PENDING" META-INF/MANIFEST.MF) || fail "manifest is missin
 INVENTORY=$(jar tf "$PENDING") || fail "publisher JAR is unreadable"
 for entry in \
   com/sportsbook/orchestration/fixture/FixturePublisher.class \
+  com/sportsbook/orchestration/fixture/KafkaProbe.class \
   com/sportsbook/protocol/event/EventLifecycle.class \
   org/apache/kafka/clients/producer/KafkaProducer.class; do
   grep -qx "$entry" <<<"$INVENTORY" || fail "publisher dependency is missing: $entry"


