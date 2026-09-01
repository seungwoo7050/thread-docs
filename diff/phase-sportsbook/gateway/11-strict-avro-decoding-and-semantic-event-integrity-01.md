# 엄격한 Avro 디코딩과 이벤트 의미 무결성

## `feat(kafka): define permanent contract failures`

diff --git a/src/main/java/com/sportsbook/gateway/kafka/GatewayEventContractException.java b/src/main/java/com/sportsbook/gateway/kafka/GatewayEventContractException.java
new file mode 100644
index 0000000..73c3332
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/kafka/GatewayEventContractException.java
@@ -0,0 +1,13 @@
+package com.sportsbook.gateway.kafka;
+
+/** A permanent event contract violation that must bypass transient delivery retries. */
+public final class GatewayEventContractException extends RuntimeException {
+
+  public GatewayEventContractException(String message) {
+    super(message);
+  }
+
+  public GatewayEventContractException(String message, Throwable cause) {
+    super(message, cause);
+  }
+}


## `feat(events): decode strict Avro records`

diff --git a/src/main/java/com/sportsbook/gateway/events/StrictAvroDecoder.java b/src/main/java/com/sportsbook/gateway/events/StrictAvroDecoder.java
new file mode 100644
index 0000000..69f3e56
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/events/StrictAvroDecoder.java
@@ -0,0 +1,36 @@
+package com.sportsbook.gateway.events;
+
+import com.sportsbook.gateway.kafka.GatewayEventContractException;
+import java.io.IOException;
+import org.apache.avro.io.BinaryDecoder;
+import org.apache.avro.io.DatumReader;
+import org.apache.avro.io.DecoderFactory;
+import org.apache.avro.specific.SpecificDatumReader;
+import org.apache.avro.specific.SpecificRecord;
+
+/** Decodes one complete binary Avro record using its shared protocol generated type. */
+public final class StrictAvroDecoder {
+
+  public static <T extends SpecificRecord> T decode(byte[] payload, Class<T> type) {
+    if (payload == null) {
+      throw new GatewayEventContractException("Kafka event payload must not be null");
+    }
+    try {
+      DatumReader<T> reader = new SpecificDatumReader<>(type);
+      BinaryDecoder decoder = DecoderFactory.get().binaryDecoder(payload, null);
+      T decoded = reader.read(null, decoder);
+      if (!decoder.isEnd()) {
+        throw new GatewayEventContractException(
+            "Avro record has trailing bytes: " + type.getName());
+      }
+      return decoded;
+    } catch (GatewayEventContractException failure) {
+      throw failure;
+    } catch (IOException | RuntimeException failure) {
+      throw new GatewayEventContractException(
+          "Failed to decode Avro record " + type.getName(), failure);
+    }
+  }
+
+  private StrictAvroDecoder() {}
+}


## `test(events): reject malformed Avro records`

diff --git a/src/test/java/com/sportsbook/gateway/events/AvroTestSupport.java b/src/test/java/com/sportsbook/gateway/events/AvroTestSupport.java
new file mode 100644
index 0000000..bde3cf1
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/events/AvroTestSupport.java
@@ -0,0 +1,27 @@
+package com.sportsbook.gateway.events;
+
+import java.io.ByteArrayOutputStream;
+import java.io.IOException;
+import org.apache.avro.io.BinaryEncoder;
+import org.apache.avro.io.DatumWriter;
+import org.apache.avro.io.EncoderFactory;
+import org.apache.avro.specific.SpecificDatumWriter;
+import org.apache.avro.specific.SpecificRecord;
+
+public final class AvroTestSupport {
+
+  public static byte[] encode(SpecificRecord record) {
+    try {
+      ByteArrayOutputStream output = new ByteArrayOutputStream();
+      BinaryEncoder encoder = EncoderFactory.get().binaryEncoder(output, null);
+      DatumWriter<SpecificRecord> writer = new SpecificDatumWriter<>(record.getSchema());
+      writer.write(record, encoder);
+      encoder.flush();
+      return output.toByteArray();
+    } catch (IOException failure) {
+      throw new AssertionError("Unable to encode Avro test record", failure);
+    }
+  }
+
+  private AvroTestSupport() {}
+}
diff --git a/src/test/java/com/sportsbook/gateway/events/StrictAvroDecoderTest.java b/src/test/java/com/sportsbook/gateway/events/StrictAvroDecoderTest.java
new file mode 100644
index 0000000..a072c80
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/events/StrictAvroDecoderTest.java
@@ -0,0 +1,48 @@
+package com.sportsbook.gateway.events;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.gateway.kafka.GatewayEventContractException;
+import com.sportsbook.protocol.event.OddsChanged;
+import java.time.Instant;
+import java.util.Arrays;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class StrictAvroDecoderTest {
+
+  @Test
+  void decodesOneCompleteSharedProtocolRecord() {
+    OddsChanged event = oddsChanged();
+
+    assertThat(StrictAvroDecoder.decode(AvroTestSupport.encode(event), OddsChanged.class))
+        .isEqualTo(event);
+  }
+
+  @Test
+  void rejectsNullTruncatedAndTrailingPayloads() {
+    byte[] encoded = AvroTestSupport.encode(oddsChanged());
+    byte[] truncated = Arrays.copyOf(encoded, encoded.length - 1);
+    byte[] trailing = Arrays.copyOf(encoded, encoded.length + 1);
+
+    assertThatThrownBy(() -> StrictAvroDecoder.decode(null, OddsChanged.class))
+        .isInstanceOf(GatewayEventContractException.class);
+    assertThatThrownBy(() -> StrictAvroDecoder.decode(truncated, OddsChanged.class))
+        .isInstanceOf(GatewayEventContractException.class);
+    assertThatThrownBy(() -> StrictAvroDecoder.decode(trailing, OddsChanged.class))
+        .isInstanceOf(GatewayEventContractException.class)
+        .hasMessageContaining("trailing bytes");
+  }
+
+  private static OddsChanged oddsChanged() {
+    return OddsChanged.newBuilder()
+        .setEventId(UUID.randomUUID().toString())
+        .setMarketId(UUID.randomUUID().toString())
+        .setSelectionId(UUID.randomUUID().toString())
+        .setPreviousOdds("1.8500")
+        .setNewOdds("1.9000")
+        .setChangedAt(Instant.parse("2026-08-21T00:00:00Z"))
+        .build();
+  }
+}


## `feat(events): validate odds event identities`

diff --git a/src/main/java/com/sportsbook/gateway/events/GatewayEventContract.java b/src/main/java/com/sportsbook/gateway/events/GatewayEventContract.java
new file mode 100644
index 0000000..ece006f
--- /dev/null
+++ b/src/main/java/com/sportsbook/gateway/events/GatewayEventContract.java
@@ -0,0 +1,63 @@
+package com.sportsbook.gateway.events;
+
+import com.sportsbook.gateway.kafka.GatewayEventContractException;
+import com.sportsbook.protocol.event.OddsChanged;
+import java.nio.ByteBuffer;
+import java.nio.charset.CharacterCodingException;
+import java.nio.charset.CodingErrorAction;
+import java.nio.charset.StandardCharsets;
+import java.util.UUID;
+import org.apache.kafka.clients.consumer.ConsumerRecord;
+
+/** Binds raw Kafka partition keys to canonical identities in decoded event payloads. */
+public final class GatewayEventContract {
+
+  public static OddsChanged oddsChanged(ConsumerRecord<byte[], byte[]> record) {
+    OddsChanged event = StrictAvroDecoder.decode(record.value(), OddsChanged.class);
+    requireCanonicalUuid(event.getEventId(), "eventId");
+    requireCanonicalUuid(event.getMarketId(), "marketId");
+    requireCanonicalUuid(event.getSelectionId(), "selectionId");
+    requirePartitionKey(record.key(), event.getEventId(), record.topic());
+    return event;
+  }
+
+  private static void requirePartitionKey(byte[] rawKey, String expected, String topic) {
+    if (rawKey == null) {
+      throw new GatewayEventContractException("Kafka key is required for " + topic);
+    }
+    String actual = decodeUtf8(rawKey, topic);
+    if (!expected.equals(actual)) {
+      throw new GatewayEventContractException(
+          "Kafka key does not match payload identity for " + topic);
+    }
+  }
+
+  private static String decodeUtf8(byte[] rawKey, String topic) {
+    try {
+      return StandardCharsets.UTF_8
+          .newDecoder()
+          .onMalformedInput(CodingErrorAction.REPORT)
+          .onUnmappableCharacter(CodingErrorAction.REPORT)
+          .decode(ByteBuffer.wrap(rawKey))
+          .toString();
+    } catch (CharacterCodingException failure) {
+      throw new GatewayEventContractException("Kafka key is not valid UTF-8 for " + topic, failure);
+    }
+  }
+
+  private static void requireCanonicalUuid(String value, String field) {
+    if (value == null) {
+      throw new GatewayEventContractException(field + " is required");
+    }
+    try {
+      UUID parsed = UUID.fromString(value);
+      if (!parsed.toString().equals(value)) {
+        throw new GatewayEventContractException(field + " must be a canonical UUID");
+      }
+    } catch (IllegalArgumentException failure) {
+      throw new GatewayEventContractException(field + " must be a canonical UUID", failure);
+    }
+  }
+
+  private GatewayEventContract() {}
+}


## `test(events): reject invalid odds identities`

diff --git a/src/test/java/com/sportsbook/gateway/events/GatewayOddsEventContractTest.java b/src/test/java/com/sportsbook/gateway/events/GatewayOddsEventContractTest.java
new file mode 100644
index 0000000..fd489f4
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/events/GatewayOddsEventContractTest.java
@@ -0,0 +1,70 @@
+package com.sportsbook.gateway.events;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.gateway.kafka.GatewayEventContractException;
+import com.sportsbook.protocol.event.OddsChanged;
+import java.nio.charset.StandardCharsets;
+import java.time.Instant;
+import java.util.UUID;
+import org.apache.kafka.clients.consumer.ConsumerRecord;
+import org.junit.jupiter.api.Test;
+
+class GatewayOddsEventContractTest {
+
+  @Test
+  void acceptsCanonicalPayloadAndPartitionKeyIdentities() {
+    OddsChanged event = oddsChanged();
+
+    assertThat(GatewayEventContract.oddsChanged(record(event, bytes(event.getEventId()))))
+        .isEqualTo(event);
+  }
+
+  @Test
+  void rejectsMissingMismatchedAndMalformedKeys() {
+    OddsChanged event = oddsChanged();
+
+    assertRejected(record(event, null));
+    assertRejected(record(event, bytes(UUID.randomUUID().toString())));
+    assertRejected(record(event, new byte[] {(byte) 0xc3, (byte) 0x28}));
+  }
+
+  @Test
+  void rejectsEveryNoncanonicalPayloadIdentity() {
+    OddsChanged event = oddsChanged();
+
+    assertInvalid(
+        OddsChanged.newBuilder(event).setEventId(event.getEventId().toUpperCase()).build());
+    assertInvalid(OddsChanged.newBuilder(event).setMarketId("not-a-uuid").build());
+    assertInvalid(OddsChanged.newBuilder(event).setSelectionId("1-1-1-1-1").build());
+  }
+
+  private static void assertInvalid(OddsChanged event) {
+    assertRejected(record(event, bytes(event.getEventId())));
+  }
+
+  private static void assertRejected(ConsumerRecord<byte[], byte[]> record) {
+    assertThatThrownBy(() -> GatewayEventContract.oddsChanged(record))
+        .isInstanceOf(GatewayEventContractException.class);
+  }
+
+  private static ConsumerRecord<byte[], byte[]> record(OddsChanged event, byte[] key) {
+    return new ConsumerRecord<>("odds.changed", 0, 0, key, AvroTestSupport.encode(event));
+  }
+
+  private static byte[] bytes(String value) {
+    return value.getBytes(StandardCharsets.UTF_8);
+  }
+
+  private static OddsChanged oddsChanged() {
+    return OddsChanged.newBuilder()
+        .setEventId(UUID.randomUUID().toString())
+        .setMarketId(UUID.randomUUID().toString())
+        .setSelectionId(UUID.randomUUID().toString())
+        .setPreviousOdds("1.8500")
+        .setNewOdds("1.9000")
+        .setChangedAt(Instant.parse("2026-08-21T00:00:00Z"))
+        .build();
+  }
+}


## `feat(events): validate terminal bet identities`

diff --git a/src/main/java/com/sportsbook/gateway/events/GatewayEventContract.java b/src/main/java/com/sportsbook/gateway/events/GatewayEventContract.java
index ece006f..6e5485e 100644
--- a/src/main/java/com/sportsbook/gateway/events/GatewayEventContract.java
+++ b/src/main/java/com/sportsbook/gateway/events/GatewayEventContract.java
@@ -1,6 +1,8 @@
 package com.sportsbook.gateway.events;
 
 import com.sportsbook.gateway.kafka.GatewayEventContractException;
+import com.sportsbook.protocol.event.BetSettled;
+import com.sportsbook.protocol.event.BetVoided;
 import com.sportsbook.protocol.event.OddsChanged;
 import java.nio.ByteBuffer;
 import java.nio.charset.CharacterCodingException;
@@ -21,6 +23,26 @@ public final class GatewayEventContract {
     return event;
   }
 
+  public static BetSettled betSettled(ConsumerRecord<byte[], byte[]> record) {
+    BetSettled event = StrictAvroDecoder.decode(record.value(), BetSettled.class);
+    requireBetIdentity(event.getBetId(), event.getUserId(), event.getEventId());
+    requirePartitionKey(record.key(), event.getEventId(), record.topic());
+    return event;
+  }
+
+  public static BetVoided betVoided(ConsumerRecord<byte[], byte[]> record) {
+    BetVoided event = StrictAvroDecoder.decode(record.value(), BetVoided.class);
+    requireBetIdentity(event.getBetId(), event.getUserId(), event.getEventId());
+    requirePartitionKey(record.key(), event.getEventId(), record.topic());
+    return event;
+  }
+
+  private static void requireBetIdentity(String betId, String userId, String eventId) {
+    requireCanonicalUuid(betId, "betId");
+    requireCanonicalUuid(userId, "userId");
+    requireCanonicalUuid(eventId, "eventId");
+  }
+
   private static void requirePartitionKey(byte[] rawKey, String expected, String topic) {
     if (rawKey == null) {
       throw new GatewayEventContractException("Kafka key is required for " + topic);


## `test(events): reject invalid settled bet identities`

diff --git a/src/test/java/com/sportsbook/gateway/events/GatewaySettledEventContractTest.java b/src/test/java/com/sportsbook/gateway/events/GatewaySettledEventContractTest.java
new file mode 100644
index 0000000..48e9af4
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/events/GatewaySettledEventContractTest.java
@@ -0,0 +1,69 @@
+package com.sportsbook.gateway.events;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.gateway.kafka.GatewayEventContractException;
+import com.sportsbook.protocol.event.BetSettled;
+import com.sportsbook.protocol.event.Money;
+import com.sportsbook.protocol.event.SettlementResultAvro;
+import java.nio.charset.StandardCharsets;
+import java.time.Instant;
+import java.util.UUID;
+import org.apache.kafka.clients.consumer.ConsumerRecord;
+import org.junit.jupiter.api.Test;
+
+class GatewaySettledEventContractTest {
+
+  @Test
+  void acceptsCanonicalSettledPayloadAndEventKey() {
+    BetSettled event = settled();
+
+    assertThat(GatewayEventContract.betSettled(record(event, event.getEventId()))).isEqualTo(event);
+  }
+
+  @Test
+  void rejectsMismatchedSettledEventKey() {
+    assertRejected(settled(), UUID.randomUUID().toString());
+  }
+
+  @Test
+  void rejectsNoncanonicalSettledBetUserAndEventIds() {
+    BetSettled event = settled();
+
+    assertRejected(BetSettled.newBuilder(event).setBetId("not-a-uuid").build());
+    assertRejected(BetSettled.newBuilder(event).setUserId("NOT-A-UUID").build());
+    assertRejected(BetSettled.newBuilder(event).setEventId("1-1-1-1-1").build());
+  }
+
+  private static void assertRejected(BetSettled event) {
+    assertRejected(event, event.getEventId());
+  }
+
+  private static void assertRejected(BetSettled event, String key) {
+    assertThatThrownBy(() -> GatewayEventContract.betSettled(record(event, key)))
+        .isInstanceOf(GatewayEventContractException.class);
+  }
+
+  private static ConsumerRecord<byte[], byte[]> record(BetSettled event, String key) {
+    return new ConsumerRecord<>(
+        "bet.settled.v1",
+        0,
+        0,
+        key.getBytes(StandardCharsets.UTF_8),
+        AvroTestSupport.encode(event));
+  }
+
+  private static BetSettled settled() {
+    Money stake = Money.newBuilder().setAmount(10_000).setCurrency("KRW").build();
+    return BetSettled.newBuilder()
+        .setBetId(UUID.randomUUID().toString())
+        .setUserId(UUID.randomUUID().toString())
+        .setEventId(UUID.randomUUID().toString())
+        .setResult(SettlementResultAvro.WON)
+        .setStake(stake)
+        .setPayout(stake)
+        .setSettledAt(Instant.parse("2026-08-21T00:00:01Z"))
+        .build();
+  }
+}


## `test(events): reject invalid voided bet identities`

diff --git a/src/test/java/com/sportsbook/gateway/events/GatewayVoidedEventContractTest.java b/src/test/java/com/sportsbook/gateway/events/GatewayVoidedEventContractTest.java
new file mode 100644
index 0000000..c6613a2
--- /dev/null
+++ b/src/test/java/com/sportsbook/gateway/events/GatewayVoidedEventContractTest.java
@@ -0,0 +1,64 @@
+package com.sportsbook.gateway.events;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.gateway.kafka.GatewayEventContractException;
+import com.sportsbook.protocol.event.BetVoided;
+import com.sportsbook.protocol.event.Money;
+import com.sportsbook.protocol.event.VoidReason;
+import java.nio.charset.StandardCharsets;
+import java.time.Instant;
+import java.util.UUID;
+import org.apache.kafka.clients.consumer.ConsumerRecord;
+import org.junit.jupiter.api.Test;
+
+class GatewayVoidedEventContractTest {
+
+  @Test
+  void acceptsCanonicalVoidedPayloadAndEventKey() {
+    BetVoided event = voided();
+
+    assertThat(GatewayEventContract.betVoided(record(event, event.getEventId()))).isEqualTo(event);
+  }
+
+  @Test
+  void rejectsMismatchedVoidedEventKey() {
+    assertRejected(voided(), UUID.randomUUID().toString());
+  }
+
+  @Test
+  void rejectsNoncanonicalVoidedBetUserAndEventIds() {
+    BetVoided event = voided();
+
+    assertRejected(BetVoided.newBuilder(event).setBetId("not-a-uuid").build());
+    assertRejected(BetVoided.newBuilder(event).setUserId("NOT-A-UUID").build());
+    assertRejected(BetVoided.newBuilder(event).setEventId("1-1-1-1-1").build());
+  }
+
+  private static void assertRejected(BetVoided event) {
+    assertRejected(event, event.getEventId());
+  }
+
+  private static void assertRejected(BetVoided event, String key) {
+    assertThatThrownBy(() -> GatewayEventContract.betVoided(record(event, key)))
+        .isInstanceOf(GatewayEventContractException.class);
+  }
+
+  private static ConsumerRecord<byte[], byte[]> record(BetVoided event, String key) {
+    return new ConsumerRecord<>(
+        "bet.voided.v1", 0, 0, key.getBytes(StandardCharsets.UTF_8), AvroTestSupport.encode(event));
+  }
+
+  private static BetVoided voided() {
+    Money refund = Money.newBuilder().setAmount(10_000).setCurrency("KRW").build();
+    return BetVoided.newBuilder()
+        .setBetId(UUID.randomUUID().toString())
+        .setUserId(UUID.randomUUID().toString())
+        .setEventId(UUID.randomUUID().toString())
+        .setReason(VoidReason.EVENT_POSTPONED)
+        .setRefund(refund)
+        .setVoidedAt(Instant.parse("2026-08-21T00:00:01Z"))
+        .build();
+  }
+}


## `feat(events): validate resolution revisions`

diff --git a/src/main/java/com/sportsbook/gateway/events/GatewayEventContract.java b/src/main/java/com/sportsbook/gateway/events/GatewayEventContract.java
index 6e5485e..e85de5e 100644
--- a/src/main/java/com/sportsbook/gateway/events/GatewayEventContract.java
+++ b/src/main/java/com/sportsbook/gateway/events/GatewayEventContract.java
@@ -1,6 +1,7 @@
 package com.sportsbook.gateway.events;
 
 import com.sportsbook.gateway.kafka.GatewayEventContractException;
+import com.sportsbook.protocol.event.BetResolutionRevised;
 import com.sportsbook.protocol.event.BetSettled;
 import com.sportsbook.protocol.event.BetVoided;
 import com.sportsbook.protocol.event.OddsChanged;
@@ -37,6 +38,24 @@ public final class GatewayEventContract {
     return event;
   }
 
+  public static BetResolutionRevised betResolutionRevised(ConsumerRecord<byte[], byte[]> record) {
+    BetResolutionRevised event =
+        StrictAvroDecoder.decode(record.value(), BetResolutionRevised.class);
+    requireCanonicalUuid(event.getRevisionId(), "revisionId");
+    requireBetIdentity(event.getBetId(), event.getUserId(), event.getEventId());
+    if (event.getRevisionNumber() < 1) {
+      throw new GatewayEventContractException("revisionNumber must be positive");
+    }
+    if (!event.getPreviousPayout().getCurrency().equals(event.getNewPayout().getCurrency())) {
+      throw new GatewayEventContractException("revision payout currencies must match");
+    }
+    if (event.getSourceResultSettledAt().isAfter(event.getRevisedAt())) {
+      throw new GatewayEventContractException("revisedAt cannot precede sourceResultSettledAt");
+    }
+    requirePartitionKey(record.key(), event.getBetId(), record.topic());
+    return event;
+  }
+
   private static void requireBetIdentity(String betId, String userId, String eventId) {
     requireCanonicalUuid(betId, "betId");
     requireCanonicalUuid(userId, "userId");


