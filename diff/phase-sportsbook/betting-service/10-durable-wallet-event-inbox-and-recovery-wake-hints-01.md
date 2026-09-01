# 지갑 이벤트 인박스와 복구 웨이크 힌트

## `feat(database): track wallet reconciliation hints`

diff --git a/src/main/resources/db/migration/V8__wallet_event_reconciliation.sql b/src/main/resources/db/migration/V8__wallet_event_reconciliation.sql
new file mode 100644
index 0000000..5310cec
--- /dev/null
+++ b/src/main/resources/db/migration/V8__wallet_event_reconciliation.sql
@@ -0,0 +1,30 @@
+-- Wallet Kafka events are durable wake-up hints. HTTP remains authoritative.
+ALTER TABLE bet
+    ADD COLUMN reconciliation_requested_at TIMESTAMP WITH TIME ZONE;
+
+CREATE TABLE wallet_event_receipt (
+    event_id        UUID                     PRIMARY KEY,
+    topic           VARCHAR(64)              NOT NULL,
+    bet_id          UUID                     NOT NULL REFERENCES bet (bet_id),
+    user_id         UUID                     NOT NULL,
+    payload_sha256  VARCHAR(64)              NOT NULL,
+    received_at     TIMESTAMP WITH TIME ZONE NOT NULL,
+    processed_at    TIMESTAMP WITH TIME ZONE,
+    CONSTRAINT wallet_event_topic_valid CHECK (
+        topic IN ('wallet.debited.v1', 'wallet.debit-failed.v1')
+    ),
+    CONSTRAINT wallet_event_payload_hash_valid CHECK (
+        payload_sha256 ~ '^[0-9a-f]{64}$'
+    )
+);
+
+COMMENT ON TABLE wallet_event_receipt IS
+    'Deduplicates at-least-once wallet wake-up events by their event-id header.';
+COMMENT ON COLUMN wallet_event_receipt.payload_sha256 IS
+    'Detects a conflicting payload replay under the same event-id.';
+COMMENT ON COLUMN bet.reconciliation_requested_at IS
+    'Latest durable request to confirm placement through authoritative HTTP state.';
+
+CREATE INDEX ix_wallet_event_pending
+    ON wallet_event_receipt (received_at)
+    WHERE processed_at IS NULL;


## `test(database): verify wallet receipt evidence`

diff --git a/src/test/java/com/sportsbook/betting/persistence/MigrationContractTest.java b/src/test/java/com/sportsbook/betting/persistence/MigrationContractTest.java
index b368a85..3880ce3 100644
--- a/src/test/java/com/sportsbook/betting/persistence/MigrationContractTest.java
+++ b/src/test/java/com/sportsbook/betting/persistence/MigrationContractTest.java
@@ -54,6 +54,14 @@ class MigrationContractTest {
         .contains("^[0-9a-f]{64}$");
   }
 
+  @Test
+  void deduplicatesWalletHintsByEventHeader() {
+    assertThat(migrationText("V8__wallet_event_reconciliation.sql"))
+        .contains("event_id        UUID                     PRIMARY KEY")
+        .contains("payload_sha256 ~ '^[0-9a-f]{64}$'")
+        .contains("WHERE processed_at IS NULL");
+  }
+
   private String migrationText(String migration) {
     try (InputStream input = getClass().getResourceAsStream("/db/migration/" + migration)) {
       if (input == null) {


## `feat(wallet-events): persist durable wake receipts`

diff --git a/src/main/java/com/sportsbook/betting/placement/WalletEventReceipt.java b/src/main/java/com/sportsbook/betting/placement/WalletEventReceipt.java
new file mode 100644
index 0000000..15a1d70
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/placement/WalletEventReceipt.java
@@ -0,0 +1,94 @@
+package com.sportsbook.betting.placement;
+
+import jakarta.persistence.Column;
+import jakarta.persistence.Entity;
+import jakarta.persistence.Id;
+import jakarta.persistence.Table;
+import java.time.Instant;
+import java.util.Objects;
+import java.util.Set;
+import java.util.UUID;
+
+@Entity
+@Table(name = "wallet_event_receipt")
+public class WalletEventReceipt {
+
+  private static final Set<String> TOPICS = Set.of("wallet.debited.v1", "wallet.debit-failed.v1");
+
+  @Id
+  @Column(name = "event_id", nullable = false, updatable = false)
+  private UUID eventId;
+
+  @Column(name = "topic", nullable = false, updatable = false, length = 64)
+  private String topic;
+
+  @Column(name = "bet_id", nullable = false, updatable = false)
+  private UUID betId;
+
+  @Column(name = "user_id", nullable = false, updatable = false)
+  private UUID userId;
+
+  @Column(name = "payload_sha256", nullable = false, updatable = false, length = 64)
+  private String payloadSha256;
+
+  @Column(name = "received_at", nullable = false, updatable = false)
+  private Instant receivedAt;
+
+  @Column(name = "processed_at")
+  private Instant processedAt;
+
+  protected WalletEventReceipt() {}
+
+  public static WalletEventReceipt pending(
+      UUID eventId,
+      String topic,
+      UUID betId,
+      UUID userId,
+      String payloadSha256,
+      Instant receivedAt) {
+    if (!TOPICS.contains(topic)) {
+      throw new IllegalArgumentException("Unsupported wallet event topic");
+    }
+    if (payloadSha256 == null || !payloadSha256.matches("[0-9a-f]{64}")) {
+      throw new IllegalArgumentException("payloadSha256 must be lowercase SHA-256");
+    }
+    WalletEventReceipt receipt = new WalletEventReceipt();
+    receipt.eventId = Objects.requireNonNull(eventId, "eventId");
+    receipt.topic = topic;
+    receipt.betId = Objects.requireNonNull(betId, "betId");
+    receipt.userId = Objects.requireNonNull(userId, "userId");
+    receipt.payloadSha256 = payloadSha256;
+    receipt.receivedAt = Objects.requireNonNull(receivedAt, "receivedAt");
+    return receipt;
+  }
+
+  public void markProcessed(Instant at) {
+    if (processedAt == null) {
+      processedAt = Objects.requireNonNull(at, "at");
+    }
+  }
+
+  public UUID eventId() {
+    return eventId;
+  }
+
+  public String topic() {
+    return topic;
+  }
+
+  public UUID betId() {
+    return betId;
+  }
+
+  public UUID userId() {
+    return userId;
+  }
+
+  public String payloadSha256() {
+    return payloadSha256;
+  }
+
+  public Instant processedAt() {
+    return processedAt;
+  }
+}


## `test(wallet-events): verify durable receipt identity`

diff --git a/src/test/java/com/sportsbook/betting/placement/WalletEventReceiptTest.java b/src/test/java/com/sportsbook/betting/placement/WalletEventReceiptTest.java
new file mode 100644
index 0000000..3326617
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/placement/WalletEventReceiptTest.java
@@ -0,0 +1,40 @@
+package com.sportsbook.betting.placement;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import java.time.Instant;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class WalletEventReceiptTest {
+
+  @Test
+  void ownsOneValidWalletWakeEventIdentity() {
+    UUID eventId = UUID.randomUUID();
+    UUID betId = UUID.randomUUID();
+    WalletEventReceipt receipt =
+        WalletEventReceipt.pending(
+            eventId, "wallet.debited.v1", betId, UUID.randomUUID(), "a".repeat(64), Instant.EPOCH);
+
+    receipt.markProcessed(Instant.EPOCH.plusSeconds(1));
+
+    assertThat(receipt.eventId()).isEqualTo(eventId);
+    assertThat(receipt.betId()).isEqualTo(betId);
+    assertThat(receipt.processedAt()).isEqualTo(Instant.EPOCH.plusSeconds(1));
+  }
+
+  @Test
+  void rejectsUnknownTopicsAndUnverifiablePayloads() {
+    assertThatThrownBy(
+            () ->
+                WalletEventReceipt.pending(
+                    UUID.randomUUID(),
+                    "wallet.unknown.v1",
+                    UUID.randomUUID(),
+                    UUID.randomUUID(),
+                    "not-a-hash",
+                    Instant.EPOCH))
+        .isInstanceOf(IllegalArgumentException.class);
+  }
+}


## `feat(wallet-events): mark reconciliation wake requests`

diff --git a/src/main/java/com/sportsbook/betting/domain/Bet.java b/src/main/java/com/sportsbook/betting/domain/Bet.java
index 29b9e0b..9d5b735 100644
--- a/src/main/java/com/sportsbook/betting/domain/Bet.java
+++ b/src/main/java/com/sportsbook/betting/domain/Bet.java
@@ -155,6 +155,9 @@ public class Bet {
   @Column(name = "updated_at", nullable = false)
   private Instant updatedAt;
 
+  @Column(name = "reconciliation_requested_at")
+  private Instant reconciliationRequestedAt;
+
   @OneToMany(mappedBy = "bet", cascade = CascadeType.ALL, orphanRemoval = true)
   @OrderBy("legIndex ASC")
   private List<BetLeg> legs = new ArrayList<>();
@@ -591,6 +594,14 @@ public class Bet {
     return updatedAt;
   }
 
+  public void requestReconciliation(Instant at) {
+    reconciliationRequestedAt = Objects.requireNonNull(at, "at");
+  }
+
+  public Instant reconciliationRequestedAt() {
+    return reconciliationRequestedAt;
+  }
+
   public List<BetLeg> legs() {
     return List.copyOf(legs);
   }


## `test(wallet-events): verify reconciliation wake checkpoint`

diff --git a/src/test/java/com/sportsbook/betting/domain/BetTest.java b/src/test/java/com/sportsbook/betting/domain/BetTest.java
index a10cbd9..3a7f025 100644
--- a/src/test/java/com/sportsbook/betting/domain/BetTest.java
+++ b/src/test/java/com/sportsbook/betting/domain/BetTest.java
@@ -62,6 +62,16 @@ class BetTest {
     assertThat(bet.riskCommitObserved()).isFalse();
   }
 
+  @Test
+  void recordsTheLatestWalletReconciliationWake() {
+    Bet bet = Bet.pending(draft(UUID.randomUUID(), new BetSlipType.Single()), List.of(leg("2.0")));
+    Instant requestedAt = NOW.plusSeconds(5);
+
+    bet.requestReconciliation(requestedAt);
+
+    assertThat(bet.reconciliationRequestedAt()).isEqualTo(requestedAt);
+  }
+
   @Test
   void advancesOnlyFromPersistedReservationToWalletProof() {
     Bet bet = Bet.pending(draft(UUID.randomUUID(), new BetSlipType.Single()), List.of(leg("2.0")));


## `feat(wallet-events): consume durable wake hints`

diff --git a/src/main/java/com/sportsbook/betting/config/KafkaConfig.java b/src/main/java/com/sportsbook/betting/config/KafkaConfig.java
index 61d3eb5..5c4bc5e 100644
--- a/src/main/java/com/sportsbook/betting/config/KafkaConfig.java
+++ b/src/main/java/com/sportsbook/betting/config/KafkaConfig.java
@@ -3,6 +3,7 @@ package com.sportsbook.betting.config;
 import java.util.HashMap;
 import java.util.Map;
 import org.apache.kafka.clients.producer.ProducerConfig;
+import org.apache.kafka.common.TopicPartition;
 import org.apache.kafka.common.serialization.ByteArraySerializer;
 import org.apache.kafka.common.serialization.StringSerializer;
 import org.springframework.beans.factory.annotation.Value;
@@ -11,6 +12,10 @@ import org.springframework.context.annotation.Configuration;
 import org.springframework.kafka.core.DefaultKafkaProducerFactory;
 import org.springframework.kafka.core.KafkaTemplate;
 import org.springframework.kafka.core.ProducerFactory;
+import org.springframework.kafka.listener.CommonErrorHandler;
+import org.springframework.kafka.listener.DeadLetterPublishingRecoverer;
+import org.springframework.kafka.listener.DefaultErrorHandler;
+import org.springframework.util.backoff.FixedBackOff;
 
 @Configuration
 public class KafkaConfig {
@@ -32,6 +37,15 @@ public class KafkaConfig {
     return new KafkaTemplate<>(bettingProducerFactory);
   }
 
+  @Bean
+  CommonErrorHandler walletConsumerErrorHandler(KafkaTemplate<String, byte[]> kafka) {
+    DeadLetterPublishingRecoverer recoverer =
+        new DeadLetterPublishingRecoverer(
+            kafka,
+            (record, failure) -> new TopicPartition(record.topic() + ".dlt", record.partition()));
+    return new DefaultErrorHandler(recoverer, new FixedBackOff(500, 3));
+  }
+
   static Map<String, Object> producerProperties(String bootstrapServers) {
     Map<String, Object> properties = new HashMap<>();
     properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
diff --git a/src/main/java/com/sportsbook/betting/placement/WalletEventListener.java b/src/main/java/com/sportsbook/betting/placement/WalletEventListener.java
new file mode 100644
index 0000000..117e495
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/placement/WalletEventListener.java
@@ -0,0 +1,85 @@
+package com.sportsbook.betting.placement;
+
+import com.sportsbook.betting.config.BettingTopics;
+import com.sportsbook.betting.outbox.AvroSerializer;
+import com.sportsbook.protocol.event.WalletDebitFailed;
+import com.sportsbook.protocol.event.WalletDebited;
+import java.nio.charset.StandardCharsets;
+import java.security.MessageDigest;
+import java.security.NoSuchAlgorithmException;
+import java.util.HexFormat;
+import java.util.List;
+import java.util.UUID;
+import java.util.stream.StreamSupport;
+import org.apache.kafka.clients.consumer.ConsumerRecord;
+import org.apache.kafka.common.header.Header;
+import org.springframework.kafka.annotation.KafkaListener;
+import org.springframework.stereotype.Component;
+
+@Component
+public class WalletEventListener {
+
+  private final WalletEventInbox inbox;
+  private final BetPlacementService placement;
+
+  public WalletEventListener(WalletEventInbox inbox, BetPlacementService placement) {
+    this.inbox = inbox;
+    this.placement = placement;
+  }
+
+  @KafkaListener(
+      topics = {BettingTopics.WALLET_DEBITED, BettingTopics.WALLET_DEBIT_FAILED},
+      groupId = "betting-wallet")
+  public void onWalletEvent(ConsumerRecord<String, byte[]> record) throws NoSuchAlgorithmException {
+    EventIdentity identity = identity(record);
+    UUID eventId = eventId(record);
+    inbox.record(
+        eventId, record.topic(), identity.betId(), identity.userId(), sha256(record.value()));
+    placement.reconcile(identity.betId());
+    inbox.markProcessed(eventId);
+  }
+
+  private static EventIdentity identity(ConsumerRecord<String, byte[]> record) {
+    String userId;
+    String betId;
+    if (BettingTopics.WALLET_DEBITED.equals(record.topic())) {
+      WalletDebited event = AvroSerializer.deserialize(record.value(), WalletDebited.class);
+      userId = event.getUserId();
+      betId = event.getIdempotencyKey();
+    } else if (BettingTopics.WALLET_DEBIT_FAILED.equals(record.topic())) {
+      WalletDebitFailed event = AvroSerializer.deserialize(record.value(), WalletDebitFailed.class);
+      userId = event.getUserId();
+      betId = event.getIdempotencyKey();
+    } else {
+      throw new IllegalArgumentException("Unsupported wallet topic");
+    }
+    UUID user = canonical(userId, "userId");
+    if (!userId.equals(record.key())) {
+      throw new IllegalArgumentException("Wallet Kafka key does not match userId");
+    }
+    return new EventIdentity(canonical(betId, "betId"), user);
+  }
+
+  private static UUID eventId(ConsumerRecord<String, byte[]> record) {
+    List<Header> values =
+        StreamSupport.stream(record.headers().headers("event-id").spliterator(), false).toList();
+    if (values.size() != 1) {
+      throw new IllegalArgumentException("Exactly one event-id header is required");
+    }
+    return canonical(new String(values.get(0).value(), StandardCharsets.US_ASCII), "event-id");
+  }
+
+  private static UUID canonical(String value, String name) {
+    UUID parsed = UUID.fromString(value);
+    if (!parsed.toString().equals(value)) {
+      throw new IllegalArgumentException(name + " must be a canonical lowercase UUID");
+    }
+    return parsed;
+  }
+
+  private static String sha256(byte[] value) throws NoSuchAlgorithmException {
+    return HexFormat.of().formatHex(MessageDigest.getInstance("SHA-256").digest(value));
+  }
+
+  private record EventIdentity(UUID betId, UUID userId) {}
+}


## `test(wallet-events): verify durable consumer boundary`

diff --git a/src/test/java/com/sportsbook/betting/placement/WalletEventListenerTest.java b/src/test/java/com/sportsbook/betting/placement/WalletEventListenerTest.java
new file mode 100644
index 0000000..2d2df5c
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/placement/WalletEventListenerTest.java
@@ -0,0 +1,53 @@
+package com.sportsbook.betting.placement;
+
+import static org.mockito.ArgumentMatchers.anyString;
+import static org.mockito.ArgumentMatchers.eq;
+import static org.mockito.Mockito.inOrder;
+import static org.mockito.Mockito.mock;
+
+import com.sportsbook.betting.config.BettingTopics;
+import com.sportsbook.betting.outbox.AvroSerializer;
+import com.sportsbook.protocol.event.WalletDebited;
+import java.nio.charset.StandardCharsets;
+import java.time.Instant;
+import java.util.UUID;
+import org.apache.kafka.clients.consumer.ConsumerRecord;
+import org.junit.jupiter.api.Test;
+import org.mockito.InOrder;
+
+class WalletEventListenerTest {
+
+  @Test
+  void checkpointsThenReconcilesBeforeAcknowledgementReturns() throws Exception {
+    WalletEventInbox inbox = mock(WalletEventInbox.class);
+    BetPlacementService placement = mock(BetPlacementService.class);
+    UUID eventId = UUID.randomUUID();
+    UUID betId = UUID.randomUUID();
+    UUID userId = UUID.randomUUID();
+    WalletDebited event =
+        WalletDebited.newBuilder()
+            .setUserId(userId.toString())
+            .setAmount(
+                com.sportsbook.protocol.event.Money.newBuilder()
+                    .setAmount(1_000L)
+                    .setCurrency("KRW")
+                    .build())
+            .setIdempotencyKey(betId.toString())
+            .setLedgerTxId(UUID.randomUUID().toString())
+            .setOccurredAt(Instant.EPOCH)
+            .build();
+    ConsumerRecord<String, byte[]> record =
+        new ConsumerRecord<>(
+            BettingTopics.WALLET_DEBITED, 2, 7, userId.toString(), AvroSerializer.serialize(event));
+    record.headers().add("event-id", eventId.toString().getBytes(StandardCharsets.US_ASCII));
+
+    new WalletEventListener(inbox, placement).onWalletEvent(record);
+
+    InOrder order = inOrder(inbox, placement);
+    order
+        .verify(inbox)
+        .record(eq(eventId), eq(BettingTopics.WALLET_DEBITED), eq(betId), eq(userId), anyString());
+    order.verify(placement).reconcile(betId);
+    order.verify(inbox).markProcessed(eventId);
+  }
+}


## `feat(wallet-events): deduplicate reconciliation hints`

diff --git a/src/main/java/com/sportsbook/betting/persistence/WalletEventReceiptRepository.java b/src/main/java/com/sportsbook/betting/persistence/WalletEventReceiptRepository.java
new file mode 100644
index 0000000..52e92ef
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/persistence/WalletEventReceiptRepository.java
@@ -0,0 +1,7 @@
+package com.sportsbook.betting.persistence;
+
+import com.sportsbook.betting.placement.WalletEventReceipt;
+import java.util.UUID;
+import org.springframework.data.jpa.repository.JpaRepository;
+
+public interface WalletEventReceiptRepository extends JpaRepository<WalletEventReceipt, UUID> {}
diff --git a/src/main/java/com/sportsbook/betting/placement/WalletEventInbox.java b/src/main/java/com/sportsbook/betting/placement/WalletEventInbox.java
new file mode 100644
index 0000000..cb7e8a5
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/placement/WalletEventInbox.java
@@ -0,0 +1,60 @@
+package com.sportsbook.betting.placement;
+
+import com.sportsbook.betting.domain.Bet;
+import com.sportsbook.betting.persistence.BetRepository;
+import com.sportsbook.betting.persistence.WalletEventReceiptRepository;
+import java.time.Clock;
+import java.time.Instant;
+import java.util.UUID;
+import org.springframework.stereotype.Component;
+import org.springframework.transaction.annotation.Transactional;
+
+@Component
+public class WalletEventInbox {
+
+  private final WalletEventReceiptRepository receipts;
+  private final BetRepository bets;
+  private final Clock clock;
+
+  public WalletEventInbox(WalletEventReceiptRepository receipts, BetRepository bets, Clock clock) {
+    this.receipts = receipts;
+    this.bets = bets;
+    this.clock = clock;
+  }
+
+  @Transactional
+  public WalletEventReceipt record(
+      UUID eventId, String topic, UUID betId, UUID userId, String payloadHash) {
+    WalletEventReceipt existing = receipts.findById(eventId).orElse(null);
+    if (existing != null) {
+      if (existing.topic().equals(topic)
+          && existing.betId().equals(betId)
+          && existing.userId().equals(userId)
+          && existing.payloadSha256().equals(payloadHash)) {
+        return existing;
+      }
+      throw new IllegalStateException("Conflicting wallet event replay: " + eventId);
+    }
+    Bet bet =
+        bets.findLockedByBetId(betId)
+            .orElseThrow(() -> new IllegalStateException("Wallet event references unknown bet"));
+    if (!bet.userId().equals(userId)) {
+      throw new IllegalStateException("Wallet event actor does not own bet");
+    }
+    Instant now = clock.instant();
+    WalletEventReceipt receipt =
+        WalletEventReceipt.pending(eventId, topic, betId, userId, payloadHash, now);
+    receipts.saveAndFlush(receipt);
+    bet.requestReconciliation(now);
+    return receipt;
+  }
+
+  @Transactional
+  public void markProcessed(UUID eventId) {
+    WalletEventReceipt receipt =
+        receipts
+            .findById(eventId)
+            .orElseThrow(() -> new IllegalStateException("Wallet receipt disappeared"));
+    receipt.markProcessed(clock.instant());
+  }
+}


