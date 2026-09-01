## `feat(settlement): preserve raw resolution keys`

diff --git a/src/main/java/com/sportsbook/betting/settlement/SettlementResultListener.java b/src/main/java/com/sportsbook/betting/settlement/SettlementResultListener.java
index 4b271c8..fdae84d 100644
--- a/src/main/java/com/sportsbook/betting/settlement/SettlementResultListener.java
+++ b/src/main/java/com/sportsbook/betting/settlement/SettlementResultListener.java
@@ -1,7 +1,8 @@
 package com.sportsbook.betting.settlement;
 
 import com.sportsbook.betting.config.BettingTopics;
-import com.sportsbook.betting.outbox.AvroSerializer;
+import com.sportsbook.betting.config.KafkaMessageValidator;
+import com.sportsbook.betting.config.PermanentKafkaException;
 import com.sportsbook.protocol.event.BetResolutionRevised;
 import com.sportsbook.protocol.event.BetSettled;
 import com.sportsbook.protocol.event.BetVoided;
@@ -28,32 +29,32 @@ public class SettlementResultListener {
         BettingTopics.BET_RESOLUTION_REVISED
       },
       groupId = "betting-resolution")
-  public void onResolution(ConsumerRecord<String, byte[]> record) throws NoSuchAlgorithmException {
+  public void onResolution(ConsumerRecord<byte[], byte[]> record) throws NoSuchAlgorithmException {
+    if (record.value() == null) {
+      throw new PermanentKafkaException("Resolution payload is required");
+    }
     String hash = sha256(record.value());
     switch (record.topic()) {
       case BettingTopics.BET_SETTLED -> {
-        BetSettled event = AvroSerializer.deserialize(record.value(), BetSettled.class);
-        requireKey(record, event.getEventId());
+        BetSettled event = KafkaMessageValidator.decode(record.value(), BetSettled.class);
+        KafkaMessageValidator.requireKey(record.key(), event.getEventId(), "Settlement eventId");
         settlement.apply(event, hash);
       }
       case BettingTopics.BET_VOIDED -> {
-        BetVoided event = AvroSerializer.deserialize(record.value(), BetVoided.class);
-        requireKey(record, event.getEventId());
+        BetVoided event = KafkaMessageValidator.decode(record.value(), BetVoided.class);
+        if (event.getReason() == com.sportsbook.protocol.event.VoidReason.MARKET_VOID) {
+          throw new PermanentKafkaException("MARKET_VOID must use a settled VOID result");
+        }
+        KafkaMessageValidator.requireKey(record.key(), event.getEventId(), "Void eventId");
         settlement.apply(event, hash);
       }
       case BettingTopics.BET_RESOLUTION_REVISED -> {
         BetResolutionRevised event =
-            AvroSerializer.deserialize(record.value(), BetResolutionRevised.class);
-        requireKey(record, event.getBetId());
+            KafkaMessageValidator.decode(record.value(), BetResolutionRevised.class);
+        KafkaMessageValidator.requireKey(record.key(), event.getBetId(), "Revision betId");
         settlement.apply(event, hash);
       }
-      default -> throw new IllegalArgumentException("Unsupported resolution topic");
-    }
-  }
-
-  private static void requireKey(ConsumerRecord<String, byte[]> record, String eventId) {
-    if (!eventId.equals(record.key())) {
-      throw new IllegalArgumentException("Resolution Kafka key does not match eventId");
+      default -> throw new PermanentKafkaException("Unsupported resolution topic");
     }
   }
 


## `test(settlement): verify raw resolution keys`

diff --git a/src/test/java/com/sportsbook/betting/settlement/SettlementResultListenerTest.java b/src/test/java/com/sportsbook/betting/settlement/SettlementResultListenerTest.java
index 43ee507..eec848f 100644
--- a/src/test/java/com/sportsbook/betting/settlement/SettlementResultListenerTest.java
+++ b/src/test/java/com/sportsbook/betting/settlement/SettlementResultListenerTest.java
@@ -8,11 +8,16 @@ import static org.mockito.Mockito.verify;
 import static org.mockito.Mockito.verifyNoInteractions;
 
 import com.sportsbook.betting.config.BettingTopics;
+import com.sportsbook.betting.config.PermanentKafkaException;
 import com.sportsbook.betting.outbox.AvroSerializer;
 import com.sportsbook.protocol.event.BetResolutionRevised;
+import com.sportsbook.protocol.event.BetSettled;
+import com.sportsbook.protocol.event.BetVoided;
 import com.sportsbook.protocol.event.SettlementResultAvro;
+import java.nio.charset.StandardCharsets;
 import java.time.Instant;
 import java.util.UUID;
+import org.apache.avro.specific.SpecificRecord;
 import org.apache.kafka.clients.consumer.ConsumerRecord;
 import org.junit.jupiter.api.Test;
 
@@ -22,7 +27,7 @@ class SettlementResultListenerTest {
   void dispatchesStrictRevisionBytesWithTheBetKey() throws Exception {
     BetSettlementService settlement = mock(BetSettlementService.class);
     BetResolutionRevised event = revision();
-    ConsumerRecord<String, byte[]> record = record(event, event.getBetId());
+    ConsumerRecord<byte[], byte[]> record = record(event, event.getBetId());
 
     new SettlementResultListener(settlement).onResolution(record);
 
@@ -33,17 +38,86 @@ class SettlementResultListenerTest {
   void rejectsAKeyMismatchBeforeMutatingTheProjection() {
     BetSettlementService settlement = mock(BetSettlementService.class);
     BetResolutionRevised event = revision();
-    ConsumerRecord<String, byte[]> record = record(event, UUID.randomUUID().toString());
+    ConsumerRecord<byte[], byte[]> record = record(event, UUID.randomUUID().toString());
 
     assertThatThrownBy(() -> new SettlementResultListener(settlement).onResolution(record))
-        .isInstanceOf(IllegalArgumentException.class)
+        .isInstanceOf(PermanentKafkaException.class)
         .hasMessageContaining("Kafka key");
     verifyNoInteractions(settlement);
   }
 
-  private static ConsumerRecord<String, byte[]> record(BetResolutionRevised event, String key) {
+  @Test
+  void classifiesATombstoneAsPermanentBeforeHashing() {
+    BetSettlementService settlement = mock(BetSettlementService.class);
+    ConsumerRecord<byte[], byte[]> record =
+        new ConsumerRecord<>(
+            BettingTopics.BET_SETTLED,
+            0,
+            0,
+            UUID.randomUUID().toString().getBytes(StandardCharsets.US_ASCII),
+            null);
+
+    assertThatThrownBy(() -> new SettlementResultListener(settlement).onResolution(record))
+        .isInstanceOf(PermanentKafkaException.class)
+        .hasMessageContaining("payload");
+    verifyNoInteractions(settlement);
+  }
+
+  @Test
+  void rejectsMarketVoidOnTheWholeSlipChannel() {
+    BetSettlementService settlement = mock(BetSettlementService.class);
+    BetVoided event =
+        BetVoided.newBuilder()
+            .setBetId(UUID.randomUUID().toString())
+            .setUserId(UUID.randomUUID().toString())
+            .setEventId(UUID.randomUUID().toString())
+            .setReason(com.sportsbook.protocol.event.VoidReason.MARKET_VOID)
+            .setRefund(money(1_000))
+            .setVoidedAt(Instant.EPOCH)
+            .build();
+
+    assertThatThrownBy(
+            () ->
+                new SettlementResultListener(settlement)
+                    .onResolution(record(BettingTopics.BET_VOIDED, event, event.getEventId())))
+        .isInstanceOf(PermanentKafkaException.class)
+        .hasMessageContaining("settled VOID");
+    verifyNoInteractions(settlement);
+  }
+
+  @Test
+  void acceptsMarketVoidAsASettledVoidResult() throws Exception {
+    BetSettlementService settlement = mock(BetSettlementService.class);
+    BetSettled event =
+        BetSettled.newBuilder()
+            .setBetId(UUID.randomUUID().toString())
+            .setUserId(UUID.randomUUID().toString())
+            .setEventId(UUID.randomUUID().toString())
+            .setResult(SettlementResultAvro.VOID)
+            .setStake(money(1_000))
+            .setPayout(money(1_000))
+            .setSettledAt(Instant.EPOCH)
+            .setResultDetail(java.util.Map.of("reason", "MARKET_VOID"))
+            .build();
+
+    new SettlementResultListener(settlement)
+        .onResolution(record(BettingTopics.BET_SETTLED, event, event.getEventId()));
+
+    verify(settlement).apply(eq(event), anyString());
+  }
+
+  private static ConsumerRecord<byte[], byte[]> record(BetResolutionRevised event, String key) {
+    return record(BettingTopics.BET_RESOLUTION_REVISED, event, key);
+  }
+
+  private static ConsumerRecord<byte[], byte[]> record(
+      String topic, SpecificRecord event, String key) {
     return new ConsumerRecord<>(
-        BettingTopics.BET_RESOLUTION_REVISED, 0, 0, key, AvroSerializer.serialize(event));
+        topic,
+        0,
+        0,
+        key.getBytes(java.nio.charset.StandardCharsets.US_ASCII),
+        AvroSerializer.serialize(event));
   }
 
   private static BetResolutionRevised revision() {
