# 결정적 Avro 픽스처와 Kafka 프로브

## `build(fixtures): define locked publisher module`

diff --git a/fixtures/avro-publisher/pom.xml b/fixtures/avro-publisher/pom.xml
new file mode 100644
index 0000000..405f694
--- /dev/null
+++ b/fixtures/avro-publisher/pom.xml
@@ -0,0 +1,76 @@
+<?xml version="1.0" encoding="UTF-8"?>
+<project xmlns="http://maven.apache.org/POM/4.0.0"
+         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
+         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
+  <modelVersion>4.0.0</modelVersion>
+
+  <groupId>com.sportsbook.orchestration</groupId>
+  <artifactId>avro-fixture-publisher</artifactId>
+  <version>1.0.0</version>
+
+  <properties>
+    <maven.compiler.release>17</maven.compiler.release>
+    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
+    <kafka.version>3.8.0</kafka.version>
+    <junit.version>5.11.0</junit.version>
+    <compiler.version>3.13.0</compiler.version>
+    <surefire.version>3.5.1</surefire.version>
+    <shade.version>3.6.0</shade.version>
+  </properties>
+
+  <dependencies>
+    <dependency>
+      <groupId>com.sportsbook</groupId>
+      <artifactId>shared-protocol</artifactId>
+      <version>1.0.0</version>
+    </dependency>
+    <dependency>
+      <groupId>org.apache.kafka</groupId>
+      <artifactId>kafka-clients</artifactId>
+      <version>${kafka.version}</version>
+    </dependency>
+    <dependency>
+      <groupId>org.junit.jupiter</groupId>
+      <artifactId>junit-jupiter</artifactId>
+      <version>${junit.version}</version>
+      <scope>test</scope>
+    </dependency>
+  </dependencies>
+
+  <build>
+    <plugins>
+      <plugin>
+        <groupId>org.apache.maven.plugins</groupId>
+        <artifactId>maven-compiler-plugin</artifactId>
+        <version>${compiler.version}</version>
+      </plugin>
+      <plugin>
+        <groupId>org.apache.maven.plugins</groupId>
+        <artifactId>maven-surefire-plugin</artifactId>
+        <version>${surefire.version}</version>
+      </plugin>
+      <plugin>
+        <groupId>org.apache.maven.plugins</groupId>
+        <artifactId>maven-shade-plugin</artifactId>
+        <version>${shade.version}</version>
+        <executions>
+          <execution>
+            <phase>package</phase>
+            <goals>
+              <goal>shade</goal>
+            </goals>
+            <configuration>
+              <finalName>avro-fixture-publisher</finalName>
+              <createDependencyReducedPom>false</createDependencyReducedPom>
+              <transformers>
+                <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
+                  <mainClass>com.sportsbook.orchestration.fixture.FixturePublisher</mainClass>
+                </transformer>
+              </transformers>
+            </configuration>
+          </execution>
+        </executions>
+      </plugin>
+    </plugins>
+  </build>
+</project>


## `test(fixtures): verify locked publisher dependencies`

diff --git a/tests/test_fixture_pom.py b/tests/test_fixture_pom.py
new file mode 100644
index 0000000..595d112
--- /dev/null
+++ b/tests/test_fixture_pom.py
@@ -0,0 +1,43 @@
+import pathlib
+import unittest
+import xml.etree.ElementTree as ET
+
+
+ROOT = pathlib.Path(__file__).resolve().parents[1]
+POM = ROOT / "fixtures/avro-publisher/pom.xml"
+NS = {"m": "http://maven.apache.org/POM/4.0.0"}
+
+
+class FixturePomTest(unittest.TestCase):
+    def test_uses_java_17_locked_protocol_and_kafka_dependencies(self) -> None:
+        root = ET.parse(POM).getroot()
+        properties = root.find("m:properties", NS)
+        self.assertEqual(properties.find("m:maven.compiler.release", NS).text, "17")
+
+        dependencies = {
+            (
+                dependency.find("m:groupId", NS).text,
+                dependency.find("m:artifactId", NS).text,
+            ): dependency.find("m:version", NS).text
+            for dependency in root.findall("m:dependencies/m:dependency", NS)
+        }
+        self.assertEqual(dependencies["com.sportsbook", "shared-protocol"], "1.0.0")
+        self.assertEqual(dependencies["org.apache.kafka", "kafka-clients"], "${kafka.version}")
+        self.assertEqual(properties.find("m:kafka.version", NS).text, "3.8.0")
+
+        plugins = {
+            plugin.find("m:artifactId", NS).text
+            for plugin in root.findall("m:build/m:plugins/m:plugin", NS)
+        }
+        self.assertEqual(
+            plugins,
+            {"maven-compiler-plugin", "maven-surefire-plugin", "maven-shade-plugin"},
+        )
+        source = POM.read_text()
+        self.assertNotIn("<repositories>", source)
+        self.assertNotIn("SNAPSHOT", source)
+        self.assertNotIn("systemPath", source)
+
+
+if __name__ == "__main__":
+    unittest.main()


## `feat(fixtures): register locked event contracts`

diff --git a/fixtures/avro-publisher/src/main/java/com/sportsbook/orchestration/fixture/FixtureType.java b/fixtures/avro-publisher/src/main/java/com/sportsbook/orchestration/fixture/FixtureType.java
new file mode 100644
index 0000000..963465d
--- /dev/null
+++ b/fixtures/avro-publisher/src/main/java/com/sportsbook/orchestration/fixture/FixtureType.java
@@ -0,0 +1,70 @@
+package com.sportsbook.orchestration.fixture;
+
+import com.sportsbook.protocol.event.BetResolutionRevised;
+import com.sportsbook.protocol.event.BetSettled;
+import com.sportsbook.protocol.event.EventLifecycle;
+import com.sportsbook.protocol.event.MatchResult;
+import java.util.Arrays;
+import java.util.UUID;
+import org.apache.avro.Schema;
+import org.apache.avro.SchemaNormalization;
+import org.apache.avro.generic.GenericRecord;
+
+public enum FixtureType {
+  EVENT_LIFECYCLE(
+      "EventLifecycle", EventLifecycle.getClassSchema(), "event.lifecycle", "eventId", 0xe47d6dbd952bc721L),
+  MATCH_RESULT(
+      "MatchResult", MatchResult.getClassSchema(), "match.result", "eventId", 0x3f39fbc4bbfea727L),
+  BET_SETTLED(
+      "BetSettled", BetSettled.getClassSchema(), "bet.settled.v1", "eventId", 0x113bc9d5037a850cL),
+  BET_RESOLUTION_REVISED(
+      "BetResolutionRevised",
+      BetResolutionRevised.getClassSchema(),
+      "bet.resolution.revised.v1",
+      "betId",
+      0xb05cdf4b95651059L);
+
+  private final String cliName;
+  private final Schema schema;
+  private final String topic;
+  private final String keyField;
+  private final long fingerprint;
+
+  FixtureType(String cliName, Schema schema, String topic, String keyField, long fingerprint) {
+    this.cliName = cliName;
+    this.schema = schema;
+    this.topic = topic;
+    this.keyField = keyField;
+    this.fingerprint = fingerprint;
+    if (SchemaNormalization.parsingFingerprint64(schema) != fingerprint) {
+      throw new IllegalStateException(cliName + " schema fingerprint mismatch");
+    }
+  }
+
+  public static FixtureType fromCliName(String value) {
+    return Arrays.stream(values())
+        .filter(type -> type.cliName.equals(value))
+        .findFirst()
+        .orElseThrow(() -> new IllegalArgumentException("unsupported fixture type: " + value));
+  }
+
+  public Schema schema() {
+    return schema;
+  }
+
+  public String topic() {
+    return topic;
+  }
+
+  public String key(GenericRecord record) {
+    String key = String.valueOf(record.get(keyField));
+    if (!UUID.fromString(key).toString().equals(key)) {
+      throw new IllegalArgumentException(keyField + " must be a canonical UUID");
+    }
+    return key;
+  }
+
+  public String fingerprint() {
+    return "%016x".formatted(fingerprint);
+  }
+}


## `test(fixtures): round trip every locked schema`

diff --git a/fixtures/avro-publisher/src/test/java/com/sportsbook/orchestration/fixture/FixtureSchemaCompatibilityTest.java b/fixtures/avro-publisher/src/test/java/com/sportsbook/orchestration/fixture/FixtureSchemaCompatibilityTest.java
new file mode 100644
index 0000000..56182f2
--- /dev/null
+++ b/fixtures/avro-publisher/src/test/java/com/sportsbook/orchestration/fixture/FixtureSchemaCompatibilityTest.java
@@ -0,0 +1,71 @@
+package com.sportsbook.orchestration.fixture;
+
+import static org.junit.jupiter.api.Assertions.assertEquals;
+
+import java.io.ByteArrayInputStream;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.util.Map;
+import org.apache.avro.generic.GenericDatumReader;
+import org.apache.avro.generic.GenericRecord;
+import org.apache.avro.io.DecoderFactory;
+import org.junit.jupiter.api.Test;
+import org.junit.jupiter.api.io.TempDir;
+
+class FixtureSchemaCompatibilityTest {
+  private static final String EVENT_ID = "00000000-0000-0000-0000-0000000000ab";
+  private static final String BET_ID = "00000000-0000-0000-0000-0000000000cd";
+  private static final String USER_ID = "00000000-0000-0000-0000-000000000001";
+
+  @TempDir Path temporaryDirectory;
+
+  @Test
+  void encodesEveryLockedSharedSchema() throws Exception {
+    Map<FixtureType, String> fixtures =
+        Map.of(
+            FixtureType.EVENT_LIFECYCLE,
+            """
+            {"eventId":"%s","status":"FINISHED",
+             "occurredAt":1700000000000,"scheduledStartAt":1699990000000}
+            """
+                .formatted(EVENT_ID),
+            FixtureType.MATCH_RESULT,
+            """
+            {"eventId":"%s","score":"2-1","finalStatus":"COMPLETED",
+             "resultDetail":{"selection":"WON"},"settledAt":1700000001000}
+            """
+                .formatted(EVENT_ID),
+            FixtureType.BET_SETTLED,
+            """
+            {"betId":"%s","userId":"%s","eventId":"%s","result":"WON",
+             "stake":{"amount":1000,"currency":"KRW"},
+             "payout":{"amount":2000,"currency":"KRW"},
+             "settledAt":1700000002000,"resultDetail":null}
+            """
+                .formatted(BET_ID, USER_ID, EVENT_ID),
+            FixtureType.BET_RESOLUTION_REVISED,
+            """
+            {"revisionId":"00000000-0000-0000-0000-0000000000ef",
+             "revisionNumber":1,"betId":"%s","userId":"%s","eventId":"%s",
+             "previousResult":"LOST","newResult":"WON",
+             "previousPayout":{"amount":0,"currency":"KRW"},
+             "newPayout":{"amount":2000,"currency":"KRW"},
+             "sourceResultSettledAt":1700000003000,"revisedAt":1700000004000}
+            """
+                .formatted(BET_ID, USER_ID, EVENT_ID));
+
+    for (Map.Entry<FixtureType, String> fixture : fixtures.entrySet()) {
+      FixtureType type = fixture.getKey();
+      Path json = Files.writeString(temporaryDirectory.resolve(type.name() + ".json"), fixture.getValue());
+      FixtureEncoder.EncodedFixture encoded = FixtureEncoder.encode(type, json);
+      GenericRecord decoded =
+          new GenericDatumReader<GenericRecord>(type.schema())
+              .read(
+                  null,
+                  DecoderFactory.get()
+                      .directBinaryDecoder(new ByteArrayInputStream(encoded.payload()), null));
+
+      assertEquals(encoded.key(), type.key(decoded));
+    }
+  }
+}


## `test(fixtures): lock event topics and fingerprints`

diff --git a/fixtures/avro-publisher/src/test/java/com/sportsbook/orchestration/fixture/FixtureTypeTest.java b/fixtures/avro-publisher/src/test/java/com/sportsbook/orchestration/fixture/FixtureTypeTest.java
new file mode 100644
index 0000000..a94bf91
--- /dev/null
+++ b/fixtures/avro-publisher/src/test/java/com/sportsbook/orchestration/fixture/FixtureTypeTest.java
@@ -0,0 +1,60 @@
+package com.sportsbook.orchestration.fixture;
+
+import static org.junit.jupiter.api.Assertions.assertEquals;
+import static org.junit.jupiter.api.Assertions.assertThrows;
+
+import java.util.Map;
+import java.util.Set;
+import org.apache.avro.generic.GenericData;
+import org.apache.avro.generic.GenericRecord;
+import org.junit.jupiter.api.Test;
+
+class FixtureTypeTest {
+  private static final String UUID = "00000000-0000-0000-0000-0000000000ab";
+
+  @Test
+  void exposesOnlyLockedEventContracts() {
+    Map<FixtureType, ExpectedContract> expected =
+        Map.of(
+            FixtureType.EVENT_LIFECYCLE,
+            new ExpectedContract("EventLifecycle", "event.lifecycle", "e47d6dbd952bc721"),
+            FixtureType.MATCH_RESULT,
+            new ExpectedContract("MatchResult", "match.result", "3f39fbc4bbfea727"),
+            FixtureType.BET_SETTLED,
+            new ExpectedContract("BetSettled", "bet.settled.v1", "113bc9d5037a850c"),
+            FixtureType.BET_RESOLUTION_REVISED,
+            new ExpectedContract(
+                "BetResolutionRevised", "bet.resolution.revised.v1", "b05cdf4b95651059"));
+
+    assertEquals(expected.keySet(), Set.of(FixtureType.values()));
+    expected.forEach(
+        (type, contract) -> {
+          assertEquals(type, FixtureType.fromCliName(contract.cliName()));
+          assertEquals(contract.topic(), type.topic());
+          assertEquals(contract.fingerprint(), type.fingerprint());
+          assertEquals(UUID, type.key(recordWithKey(type, UUID)));
+        });
+  }
+
+  @Test
+  void rejectsUnsupportedTypeAndNonCanonicalKey() {
+    assertThrows(IllegalArgumentException.class, () -> FixtureType.fromCliName("BetPlaced"));
+    assertThrows(
+        IllegalArgumentException.class,
+        () -> FixtureType.MATCH_RESULT.key(recordWithKey(FixtureType.MATCH_RESULT, "not-a-uuid")));
+    assertThrows(
+        IllegalArgumentException.class,
+        () ->
+            FixtureType.MATCH_RESULT.key(
+                recordWithKey(FixtureType.MATCH_RESULT, UUID.toUpperCase())));
+  }
+
+  private static GenericRecord recordWithKey(FixtureType type, String key) {
+    GenericRecord record = new GenericData.Record(type.schema());
+    String field = type == FixtureType.BET_RESOLUTION_REVISED ? "betId" : "eventId";
+    record.put(field, key);
+    return record;
+  }
+
+  private record ExpectedContract(String cliName, String topic, String fingerprint) {}
+}


## `feat(fixtures): encode canonical Avro records`

diff --git a/fixtures/avro-publisher/src/main/java/com/sportsbook/orchestration/fixture/FixtureEncoder.java b/fixtures/avro-publisher/src/main/java/com/sportsbook/orchestration/fixture/FixtureEncoder.java
new file mode 100644
index 0000000..6cdb61f
--- /dev/null
+++ b/fixtures/avro-publisher/src/main/java/com/sportsbook/orchestration/fixture/FixtureEncoder.java
@@ -0,0 +1,58 @@
+package com.sportsbook.orchestration.fixture;
+
+import java.io.ByteArrayOutputStream;
+import java.io.EOFException;
+import java.io.IOException;
+import java.io.InputStream;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import org.apache.avro.generic.GenericDatumReader;
+import org.apache.avro.generic.GenericDatumWriter;
+import org.apache.avro.generic.GenericRecord;
+import org.apache.avro.io.BinaryEncoder;
+import org.apache.avro.io.Decoder;
+import org.apache.avro.io.DecoderFactory;
+import org.apache.avro.io.EncoderFactory;
+
+public final class FixtureEncoder {
+  private FixtureEncoder() {}
+
+  public static EncodedFixture encode(FixtureType type, Path jsonPath) throws IOException {
+    GenericRecord record = decodeSingleRecord(type, jsonPath);
+    return new EncodedFixture(type.key(record), encodeBinary(type, record));
+  }
+
+  private static GenericRecord decodeSingleRecord(FixtureType type, Path jsonPath)
+      throws IOException {
+    try (InputStream input = Files.newInputStream(jsonPath)) {
+      Decoder decoder = DecoderFactory.get().jsonDecoder(type.schema(), input);
+      GenericDatumReader<GenericRecord> reader = new GenericDatumReader<>(type.schema());
+      GenericRecord record = reader.read(null, decoder);
+      try {
+        reader.read(null, decoder);
+        throw new IllegalArgumentException("fixture must contain exactly one Avro record");
+      } catch (EOFException expected) {
+        return record;
+      }
+    }
+  }
+
+  private static byte[] encodeBinary(FixtureType type, GenericRecord record) throws IOException {
+    ByteArrayOutputStream output = new ByteArrayOutputStream();
+    BinaryEncoder encoder = EncoderFactory.get().directBinaryEncoder(output, null);
+    new GenericDatumWriter<GenericRecord>(type.schema()).write(record, encoder);
+    encoder.flush();
+    return output.toByteArray();
+  }
+
+  public record EncodedFixture(String key, byte[] payload) {
+    public EncodedFixture {
+      payload = payload.clone();
+    }
+
+    @Override
+    public byte[] payload() {
+      return payload.clone();
+    }
+  }
+}


## `test(fixtures): verify deterministic raw encoding`

diff --git a/fixtures/avro-publisher/src/test/java/com/sportsbook/orchestration/fixture/FixtureEncoderTest.java b/fixtures/avro-publisher/src/test/java/com/sportsbook/orchestration/fixture/FixtureEncoderTest.java
new file mode 100644
index 0000000..05dfecd
--- /dev/null
+++ b/fixtures/avro-publisher/src/test/java/com/sportsbook/orchestration/fixture/FixtureEncoderTest.java
@@ -0,0 +1,75 @@
+package com.sportsbook.orchestration.fixture;
+
+import static org.junit.jupiter.api.Assertions.assertArrayEquals;
+import static org.junit.jupiter.api.Assertions.assertEquals;
+import static org.junit.jupiter.api.Assertions.assertThrows;
+
+import java.io.ByteArrayInputStream;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import org.apache.avro.generic.GenericDatumReader;
+import org.apache.avro.generic.GenericRecord;
+import org.apache.avro.io.DecoderFactory;
+import org.junit.jupiter.api.Test;
+import org.junit.jupiter.api.io.TempDir;
+
+class FixtureEncoderTest {
+  private static final String EVENT_ID = "00000000-0000-0000-0000-0000000000ab";
+  private static final String EVENT_JSON =
+      """
+      {
+        "eventId": "%s",
+        "status": "FINISHED",
+        "occurredAt": 1700000000000,
+        "scheduledStartAt": 1699990000000
+      }
+      """
+          .formatted(EVENT_ID);
+
+  @TempDir Path temporaryDirectory;
+
+  @Test
+  void producesDeterministicRawAvroWithoutTrailingBytes() throws Exception {
+    Path input = write("event.json", EVENT_JSON);
+
+    FixtureEncoder.EncodedFixture first =
+        FixtureEncoder.encode(FixtureType.EVENT_LIFECYCLE, input);
+    FixtureEncoder.EncodedFixture second =
+        FixtureEncoder.encode(FixtureType.EVENT_LIFECYCLE, input);
+
+    assertEquals(EVENT_ID, first.key());
+    assertArrayEquals(first.payload(), second.payload());
+
+    ByteArrayInputStream bytes = new ByteArrayInputStream(first.payload());
+    GenericRecord decoded =
+        new GenericDatumReader<GenericRecord>(FixtureType.EVENT_LIFECYCLE.schema())
+            .read(null, DecoderFactory.get().directBinaryDecoder(bytes, null));
+    assertEquals(EVENT_ID, decoded.get("eventId").toString());
+    assertEquals(0, bytes.available());
+  }
+
+  @Test
+  void rejectsMoreThanOneJsonRecord() throws Exception {
+    Path input = write("two-events.json", EVENT_JSON + EVENT_JSON);
+
+    assertThrows(
+        IllegalArgumentException.class,
+        () -> FixtureEncoder.encode(FixtureType.EVENT_LIFECYCLE, input));
+  }
+
+  @Test
+  void returnsDefensivePayloadCopies() throws Exception {
+    FixtureEncoder.EncodedFixture encoded =
+        FixtureEncoder.encode(FixtureType.EVENT_LIFECYCLE, write("event.json", EVENT_JSON));
+    byte expected = encoded.payload()[0];
+
+    byte[] callerCopy = encoded.payload();
+    callerCopy[0] = (byte) (callerCopy[0] + 1);
+
+    assertEquals(expected, encoded.payload()[0]);
+  }
+
+  private Path write(String name, String contents) throws Exception {
+    return Files.writeString(temporaryDirectory.resolve(name), contents);
+  }
+}


## `feat(fixtures): parse closed publisher commands`

diff --git a/fixtures/avro-publisher/src/main/java/com/sportsbook/orchestration/fixture/FixtureArguments.java b/fixtures/avro-publisher/src/main/java/com/sportsbook/orchestration/fixture/FixtureArguments.java
new file mode 100644
index 0000000..312f88a
--- /dev/null
+++ b/fixtures/avro-publisher/src/main/java/com/sportsbook/orchestration/fixture/FixtureArguments.java
@@ -0,0 +1,38 @@
+package com.sportsbook.orchestration.fixture;
+
+import java.io.IOException;
+import java.nio.file.Path;
+
+public record FixtureArguments(String bootstrapServers, FixtureRecord fixture) {
+  public FixtureArguments {
+    if (bootstrapServers == null || bootstrapServers.isBlank()) {
+      throw new IllegalArgumentException("bootstrap servers must not be blank");
+    }
+  }
+
+  public static FixtureArguments parse(String[] arguments) throws IOException {
+    if (arguments.length == 3 && arguments[0].equals("poison")) {
+      return new FixtureArguments(
+          arguments[1], FixtureRecord.poisonMatchResult(arguments[2]));
+    }
+    if ((arguments.length == 4 || arguments.length == 5)
+        && arguments[0].equals("publish")) {
+      Integer partition = arguments.length == 5 ? parsePartition(arguments[4]) : null;
+      FixtureRecord fixture =
+          FixtureRecord.fromJson(
+              FixtureType.fromCliName(arguments[2]), Path.of(arguments[3]), partition);
+      return new FixtureArguments(arguments[1], fixture);
+    }
+    throw new IllegalArgumentException(
+        "usage: publish <bootstrap> <type> <json> [partition]"
+            + " | poison <bootstrap> <event-id>");
+  }
+
+  private static int parsePartition(String value) {
+    try {
+      return Integer.parseInt(value);
+    } catch (NumberFormatException exception) {
+      throw new IllegalArgumentException("partition must be an integer", exception);
+    }
+  }
+}


