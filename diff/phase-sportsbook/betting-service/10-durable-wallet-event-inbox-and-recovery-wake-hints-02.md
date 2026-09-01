## `test(wallet-events): verify hint deduplication boundary`

diff --git a/src/test/java/com/sportsbook/betting/placement/WalletEventInboxTest.java b/src/test/java/com/sportsbook/betting/placement/WalletEventInboxTest.java
new file mode 100644
index 0000000..a10bca6
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/placement/WalletEventInboxTest.java
@@ -0,0 +1,69 @@
+package com.sportsbook.betting.placement;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+import static org.mockito.Mockito.inOrder;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.verifyNoInteractions;
+import static org.mockito.Mockito.when;
+
+import com.sportsbook.betting.domain.Bet;
+import com.sportsbook.betting.persistence.BetRepository;
+import com.sportsbook.betting.persistence.WalletEventReceiptRepository;
+import java.time.Clock;
+import java.time.Instant;
+import java.time.ZoneOffset;
+import java.util.Optional;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.mockito.InOrder;
+
+class WalletEventInboxTest {
+
+  @Test
+  void savesReceiptBeforeMarkingTheBetForHttpReconciliation() {
+    WalletEventReceiptRepository receipts = mock(WalletEventReceiptRepository.class);
+    BetRepository bets = mock(BetRepository.class);
+    Bet bet = mock(Bet.class);
+    UUID eventId = UUID.randomUUID();
+    UUID betId = UUID.randomUUID();
+    UUID userId = UUID.randomUUID();
+    when(receipts.findById(eventId)).thenReturn(Optional.empty());
+    when(bets.findLockedByBetId(betId)).thenReturn(Optional.of(bet));
+    when(bet.userId()).thenReturn(userId);
+    WalletEventInbox inbox = inbox(receipts, bets);
+
+    WalletEventReceipt receipt =
+        inbox.record(eventId, "wallet.debited.v1", betId, userId, "a".repeat(64));
+
+    assertThat(receipt.payloadSha256()).isEqualTo("a".repeat(64));
+    InOrder order = inOrder(receipts, bet);
+    order.verify(receipts).saveAndFlush(receipt);
+    order.verify(bet).requestReconciliation(Instant.EPOCH);
+  }
+
+  @Test
+  void rejectsConflictingPayloadUnderTheSameEventId() {
+    WalletEventReceiptRepository receipts = mock(WalletEventReceiptRepository.class);
+    BetRepository bets = mock(BetRepository.class);
+    UUID eventId = UUID.randomUUID();
+    UUID betId = UUID.randomUUID();
+    UUID userId = UUID.randomUUID();
+    WalletEventReceipt saved =
+        WalletEventReceipt.pending(
+            eventId, "wallet.debited.v1", betId, userId, "a".repeat(64), Instant.EPOCH);
+    when(receipts.findById(eventId)).thenReturn(Optional.of(saved));
+
+    assertThatThrownBy(
+            () ->
+                inbox(receipts, bets)
+                    .record(eventId, "wallet.debited.v1", betId, userId, "b".repeat(64)))
+        .isInstanceOf(IllegalStateException.class)
+        .hasMessageContaining("Conflicting");
+    verifyNoInteractions(bets);
+  }
+
+  private static WalletEventInbox inbox(WalletEventReceiptRepository receipts, BetRepository bets) {
+    return new WalletEventInbox(receipts, bets, Clock.fixed(Instant.EPOCH, ZoneOffset.UTC));
+  }
+}


## `feat(wallet-events): preserve raw record identity`

diff --git a/src/main/java/com/sportsbook/betting/placement/WalletEventListener.java b/src/main/java/com/sportsbook/betting/placement/WalletEventListener.java
index 117e495..4093156 100644
--- a/src/main/java/com/sportsbook/betting/placement/WalletEventListener.java
+++ b/src/main/java/com/sportsbook/betting/placement/WalletEventListener.java
@@ -1,7 +1,8 @@
 package com.sportsbook.betting.placement;
 
 import com.sportsbook.betting.config.BettingTopics;
-import com.sportsbook.betting.outbox.AvroSerializer;
+import com.sportsbook.betting.config.KafkaMessageValidator;
+import com.sportsbook.betting.config.PermanentKafkaException;
 import com.sportsbook.protocol.event.WalletDebitFailed;
 import com.sportsbook.protocol.event.WalletDebited;
 import java.nio.charset.StandardCharsets;
@@ -30,7 +31,7 @@ public class WalletEventListener {
   @KafkaListener(
       topics = {BettingTopics.WALLET_DEBITED, BettingTopics.WALLET_DEBIT_FAILED},
       groupId = "betting-wallet")
-  public void onWalletEvent(ConsumerRecord<String, byte[]> record) throws NoSuchAlgorithmException {
+  public void onWalletEvent(ConsumerRecord<byte[], byte[]> record) throws NoSuchAlgorithmException {
     EventIdentity identity = identity(record);
     UUID eventId = eventId(record);
     inbox.record(
@@ -39,42 +40,38 @@ public class WalletEventListener {
     inbox.markProcessed(eventId);
   }
 
-  private static EventIdentity identity(ConsumerRecord<String, byte[]> record) {
+  private static EventIdentity identity(ConsumerRecord<byte[], byte[]> record) {
     String userId;
     String betId;
     if (BettingTopics.WALLET_DEBITED.equals(record.topic())) {
-      WalletDebited event = AvroSerializer.deserialize(record.value(), WalletDebited.class);
+      WalletDebited event = KafkaMessageValidator.decode(record.value(), WalletDebited.class);
       userId = event.getUserId();
       betId = event.getIdempotencyKey();
     } else if (BettingTopics.WALLET_DEBIT_FAILED.equals(record.topic())) {
-      WalletDebitFailed event = AvroSerializer.deserialize(record.value(), WalletDebitFailed.class);
+      WalletDebitFailed event =
+          KafkaMessageValidator.decode(record.value(), WalletDebitFailed.class);
       userId = event.getUserId();
       betId = event.getIdempotencyKey();
     } else {
-      throw new IllegalArgumentException("Unsupported wallet topic");
+      throw new PermanentKafkaException("Unsupported wallet topic");
     }
-    UUID user = canonical(userId, "userId");
-    if (!userId.equals(record.key())) {
-      throw new IllegalArgumentException("Wallet Kafka key does not match userId");
-    }
-    return new EventIdentity(canonical(betId, "betId"), user);
+    UUID user = KafkaMessageValidator.canonical(userId, "userId");
+    KafkaMessageValidator.requireKey(record.key(), userId, "Wallet userId");
+    return new EventIdentity(KafkaMessageValidator.canonical(betId, "betId"), user);
   }
 
-  private static UUID eventId(ConsumerRecord<String, byte[]> record) {
+  private static UUID eventId(ConsumerRecord<byte[], byte[]> record) {
     List<Header> values =
         StreamSupport.stream(record.headers().headers("event-id").spliterator(), false).toList();
     if (values.size() != 1) {
-      throw new IllegalArgumentException("Exactly one event-id header is required");
+      throw new PermanentKafkaException("Exactly one event-id header is required");
     }
-    return canonical(new String(values.get(0).value(), StandardCharsets.US_ASCII), "event-id");
-  }
-
-  private static UUID canonical(String value, String name) {
-    UUID parsed = UUID.fromString(value);
-    if (!parsed.toString().equals(value)) {
-      throw new IllegalArgumentException(name + " must be a canonical lowercase UUID");
+    byte[] rawEventId = values.get(0).value();
+    if (rawEventId == null) {
+      throw new PermanentKafkaException("event-id header value is required");
     }
-    return parsed;
+    return KafkaMessageValidator.canonical(
+        new String(rawEventId, StandardCharsets.US_ASCII), "event-id");
   }
 
   private static String sha256(byte[] value) throws NoSuchAlgorithmException {
diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index 4313494..67e8c3d 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -26,7 +26,7 @@ spring:
     consumer:
       enable-auto-commit: false
       auto-offset-reset: earliest
-      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
+      key-deserializer: org.apache.kafka.common.serialization.ByteArrayDeserializer
       value-deserializer: org.apache.kafka.common.serialization.ByteArrayDeserializer
     listener:
       ack-mode: record


## `test(wallet-events): verify raw identity preservation`

diff --git a/src/test/java/com/sportsbook/betting/config/RuntimeConfigurationTest.java b/src/test/java/com/sportsbook/betting/config/RuntimeConfigurationTest.java
index 0cd08e4..6c99810 100644
--- a/src/test/java/com/sportsbook/betting/config/RuntimeConfigurationTest.java
+++ b/src/test/java/com/sportsbook/betting/config/RuntimeConfigurationTest.java
@@ -18,6 +18,8 @@ class RuntimeConfigurationTest {
     assertThat(value(sources, "spring.jpa.hibernate.ddl-auto")).isEqualTo("validate");
     assertThat(value(sources, "spring.flyway.locations")).isEqualTo("classpath:db/migration");
     assertThat(value(sources, "spring.kafka.consumer.enable-auto-commit")).isEqualTo(false);
+    assertThat(value(sources, "spring.kafka.consumer.key-deserializer"))
+        .isEqualTo("org.apache.kafka.common.serialization.ByteArrayDeserializer");
     assertThat(value(sources, "spring.kafka.consumer.value-deserializer"))
         .isEqualTo("org.apache.kafka.common.serialization.ByteArrayDeserializer");
     assertThat(value(sources, "spring.kafka.listener.ack-mode")).isEqualTo("record");
diff --git a/src/test/java/com/sportsbook/betting/placement/WalletEventListenerTest.java b/src/test/java/com/sportsbook/betting/placement/WalletEventListenerTest.java
index 2d2df5c..93fd4b4 100644
--- a/src/test/java/com/sportsbook/betting/placement/WalletEventListenerTest.java
+++ b/src/test/java/com/sportsbook/betting/placement/WalletEventListenerTest.java
@@ -1,11 +1,14 @@
 package com.sportsbook.betting.placement;
 
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
 import static org.mockito.ArgumentMatchers.anyString;
 import static org.mockito.ArgumentMatchers.eq;
 import static org.mockito.Mockito.inOrder;
 import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.verifyNoInteractions;
 
 import com.sportsbook.betting.config.BettingTopics;
+import com.sportsbook.betting.config.PermanentKafkaException;
 import com.sportsbook.betting.outbox.AvroSerializer;
 import com.sportsbook.protocol.event.WalletDebited;
 import java.nio.charset.StandardCharsets;
@@ -36,9 +39,13 @@ class WalletEventListenerTest {
             .setLedgerTxId(UUID.randomUUID().toString())
             .setOccurredAt(Instant.EPOCH)
             .build();
-    ConsumerRecord<String, byte[]> record =
+    ConsumerRecord<byte[], byte[]> record =
         new ConsumerRecord<>(
-            BettingTopics.WALLET_DEBITED, 2, 7, userId.toString(), AvroSerializer.serialize(event));
+            BettingTopics.WALLET_DEBITED,
+            2,
+            7,
+            userId.toString().getBytes(StandardCharsets.US_ASCII),
+            AvroSerializer.serialize(event));
     record.headers().add("event-id", eventId.toString().getBytes(StandardCharsets.US_ASCII));
 
     new WalletEventListener(inbox, placement).onWalletEvent(record);
@@ -50,4 +57,36 @@ class WalletEventListenerTest {
     order.verify(placement).reconcile(betId);
     order.verify(inbox).markProcessed(eventId);
   }
+
+  @Test
+  void classifiesANullEventIdHeaderAsPermanent() {
+    WalletEventInbox inbox = mock(WalletEventInbox.class);
+    BetPlacementService placement = mock(BetPlacementService.class);
+    UUID userId = UUID.randomUUID();
+    WalletDebited event =
+        WalletDebited.newBuilder()
+            .setUserId(userId.toString())
+            .setAmount(
+                com.sportsbook.protocol.event.Money.newBuilder()
+                    .setAmount(1_000L)
+                    .setCurrency("KRW")
+                    .build())
+            .setIdempotencyKey(UUID.randomUUID().toString())
+            .setLedgerTxId(UUID.randomUUID().toString())
+            .setOccurredAt(Instant.EPOCH)
+            .build();
+    ConsumerRecord<byte[], byte[]> record =
+        new ConsumerRecord<>(
+            BettingTopics.WALLET_DEBITED,
+            0,
+            0,
+            userId.toString().getBytes(StandardCharsets.US_ASCII),
+            AvroSerializer.serialize(event));
+    record.headers().add("event-id", null);
+
+    assertThatThrownBy(() -> new WalletEventListener(inbox, placement).onWalletEvent(record))
+        .isInstanceOf(PermanentKafkaException.class)
+        .hasMessageContaining("header value");
+    verifyNoInteractions(inbox, placement);
+  }
 }


## `feat(wallet-events): isolate receipt identity conflicts`

diff --git a/src/main/java/com/sportsbook/betting/placement/WalletEventInbox.java b/src/main/java/com/sportsbook/betting/placement/WalletEventInbox.java
index cb7e8a5..b585c99 100644
--- a/src/main/java/com/sportsbook/betting/placement/WalletEventInbox.java
+++ b/src/main/java/com/sportsbook/betting/placement/WalletEventInbox.java
@@ -1,5 +1,6 @@
 package com.sportsbook.betting.placement;
 
+import com.sportsbook.betting.config.PermanentKafkaException;
 import com.sportsbook.betting.domain.Bet;
 import com.sportsbook.betting.persistence.BetRepository;
 import com.sportsbook.betting.persistence.WalletEventReceiptRepository;
@@ -33,13 +34,13 @@ public class WalletEventInbox {
           && existing.payloadSha256().equals(payloadHash)) {
         return existing;
       }
-      throw new IllegalStateException("Conflicting wallet event replay: " + eventId);
+      throw new PermanentKafkaException("Conflicting wallet event replay: " + eventId);
     }
     Bet bet =
         bets.findLockedByBetId(betId)
-            .orElseThrow(() -> new IllegalStateException("Wallet event references unknown bet"));
+            .orElseThrow(() -> new PermanentKafkaException("Wallet event references unknown bet"));
     if (!bet.userId().equals(userId)) {
-      throw new IllegalStateException("Wallet event actor does not own bet");
+      throw new PermanentKafkaException("Wallet event actor does not own bet");
     }
     Instant now = clock.instant();
     WalletEventReceipt receipt =


## `test(wallet-events): verify receipt identity conflicts`

diff --git a/src/test/java/com/sportsbook/betting/placement/WalletEventInboxTest.java b/src/test/java/com/sportsbook/betting/placement/WalletEventInboxTest.java
index a10bca6..980b600 100644
--- a/src/test/java/com/sportsbook/betting/placement/WalletEventInboxTest.java
+++ b/src/test/java/com/sportsbook/betting/placement/WalletEventInboxTest.java
@@ -7,6 +7,7 @@ import static org.mockito.Mockito.mock;
 import static org.mockito.Mockito.verifyNoInteractions;
 import static org.mockito.Mockito.when;
 
+import com.sportsbook.betting.config.PermanentKafkaException;
 import com.sportsbook.betting.domain.Bet;
 import com.sportsbook.betting.persistence.BetRepository;
 import com.sportsbook.betting.persistence.WalletEventReceiptRepository;
@@ -58,7 +59,7 @@ class WalletEventInboxTest {
             () ->
                 inbox(receipts, bets)
                     .record(eventId, "wallet.debited.v1", betId, userId, "b".repeat(64)))
-        .isInstanceOf(IllegalStateException.class)
+        .isInstanceOf(PermanentKafkaException.class)
         .hasMessageContaining("Conflicting");
     verifyNoInteractions(bets);
   }
