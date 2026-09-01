## `test(messaging): verify producer delivery settings`

diff --git a/src/test/java/com/sportsbook/betting/config/KafkaConfigTest.java b/src/test/java/com/sportsbook/betting/config/KafkaConfigTest.java
new file mode 100644
index 0000000..e789586
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/config/KafkaConfigTest.java
@@ -0,0 +1,22 @@
+package com.sportsbook.betting.config;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.util.Map;
+import org.apache.kafka.clients.producer.ProducerConfig;
+import org.junit.jupiter.api.Test;
+
+class KafkaConfigTest {
+
+  @Test
+  void requiresBrokerAcknowledgementAndIdempotence() {
+    Map<String, Object> properties = KafkaConfig.producerProperties("broker:9092");
+
+    assertThat(properties.get(ProducerConfig.ACKS_CONFIG)).isEqualTo("all");
+    assertThat(properties.get(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG)).isEqualTo(true);
+    assertThat(properties.get(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION)).isEqualTo(5);
+    assertThat(properties.get(ProducerConfig.MAX_BLOCK_MS_CONFIG)).isEqualTo(5_000);
+    assertThat(properties.get(ProducerConfig.REQUEST_TIMEOUT_MS_CONFIG)).isEqualTo(5_000);
+    assertThat(properties.get(ProducerConfig.DELIVERY_TIMEOUT_MS_CONFIG)).isEqualTo(10_000);
+  }
+}


## `feat(placement): finalize outcomes with outbox atomically`

diff --git a/src/main/java/com/sportsbook/betting/placement/BetStore.java b/src/main/java/com/sportsbook/betting/placement/BetStore.java
index ad86c5b..ba56d02 100644
--- a/src/main/java/com/sportsbook/betting/placement/BetStore.java
+++ b/src/main/java/com/sportsbook/betting/placement/BetStore.java
@@ -1,6 +1,8 @@
 package com.sportsbook.betting.placement;
 
 import com.sportsbook.betting.domain.Bet;
+import com.sportsbook.betting.outbox.OutboxEvent;
+import com.sportsbook.betting.outbox.OutboxEventRepository;
 import com.sportsbook.betting.persistence.BetRepository;
 import com.sportsbook.betting.persistence.PlacementRequestRepository;
 import com.sportsbook.protocol.domain.BetStatus;
@@ -15,10 +17,13 @@ import org.springframework.transaction.annotation.Transactional;
 public class BetStore {
 
   private final BetRepository bets;
+  private final OutboxEventRepository outbox;
   private final PlacementRequestRepository requests;
 
-  public BetStore(BetRepository bets, PlacementRequestRepository requests) {
+  public BetStore(
+      BetRepository bets, OutboxEventRepository outbox, PlacementRequestRepository requests) {
     this.bets = bets;
+    this.outbox = outbox;
     this.requests = requests;
   }
 
@@ -91,14 +96,48 @@ public class BetStore {
     pending(betId).completeWalletRefund(operationId, now);
   }
 
+  @Transactional
+  public Bet rejectAtCreation(UUID betId, ErrorCode reason, String detail, Instant now) {
+    Bet bet = locked(betId);
+    if (bet.status() == BetStatus.PENDING) {
+      bet.rejectAtCreation(reason.name(), detail, now);
+    }
+    return bet;
+  }
+
+  @Transactional
+  public Bet rejectAfterCompensation(UUID betId, Instant now) {
+    Bet bet = locked(betId);
+    if (bet.status() == BetStatus.PENDING) {
+      bet.rejectAfterCompensation(now);
+    }
+    return bet;
+  }
+
+  @Transactional
+  public Bet acceptAndEnqueue(UUID betId, OutboxEvent event, Instant now) {
+    Bet bet = locked(betId);
+    if (bet.status() == BetStatus.ACCEPTED) {
+      return bet;
+    }
+    if (bet.status() != BetStatus.PENDING) {
+      throw new IllegalStateException("Cannot accept terminal bet " + betId);
+    }
+    bet.accept(now);
+    outbox.save(event);
+    return bet;
+  }
+
   private Bet pending(UUID betId) {
-    Bet bet =
-        bets.findLockedByBetId(betId)
-            .orElseThrow(
-                () -> new IllegalStateException("Bet vanished during placement: " + betId));
+    Bet bet = locked(betId);
     if (bet.status() != BetStatus.PENDING) {
       throw new IllegalStateException("Placement cannot update terminal bet " + betId);
     }
     return bet;
   }
+
+  private Bet locked(UUID betId) {
+    return bets.findLockedByBetId(betId)
+        .orElseThrow(() -> new IllegalStateException("Bet vanished during placement: " + betId));
+  }
 }


## `feat(outbox): publish acknowledged pending events`

diff --git a/src/main/java/com/sportsbook/betting/outbox/OutboxPublisher.java b/src/main/java/com/sportsbook/betting/outbox/OutboxPublisher.java
new file mode 100644
index 0000000..d04eaef
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/outbox/OutboxPublisher.java
@@ -0,0 +1,55 @@
+package com.sportsbook.betting.outbox;
+
+import java.time.Clock;
+import java.util.concurrent.TimeUnit;
+import org.apache.kafka.clients.producer.ProducerRecord;
+import org.slf4j.Logger;
+import org.slf4j.LoggerFactory;
+import org.springframework.data.domain.PageRequest;
+import org.springframework.kafka.core.KafkaTemplate;
+import org.springframework.scheduling.annotation.Scheduled;
+import org.springframework.stereotype.Component;
+import org.springframework.transaction.annotation.Transactional;
+
+@Component
+public class OutboxPublisher {
+
+  private static final Logger LOG = LoggerFactory.getLogger(OutboxPublisher.class);
+  private static final int BATCH_SIZE = 100;
+  static final long TIMEOUT_SECONDS = 11;
+
+  private final OutboxEventRepository repository;
+  private final KafkaTemplate<String, byte[]> kafka;
+  private final Clock clock;
+
+  public OutboxPublisher(
+      OutboxEventRepository repository, KafkaTemplate<String, byte[]> kafka, Clock clock) {
+    this.repository = repository;
+    this.kafka = kafka;
+    this.clock = clock;
+  }
+
+  @Scheduled(
+      fixedDelayString = "${betting.outbox.poll-interval-ms:1000}",
+      scheduler = "outboxTaskScheduler")
+  @Transactional
+  public void publishPending() {
+    for (OutboxEvent event : repository.findUnpublished(PageRequest.of(0, BATCH_SIZE))) {
+      publish(event);
+    }
+  }
+
+  private void publish(OutboxEvent event) {
+    try {
+      ProducerRecord<String, byte[]> record =
+          new ProducerRecord<>(event.topic(), event.partitionKey(), event.payload());
+      kafka.send(record).get(TIMEOUT_SECONDS, TimeUnit.SECONDS);
+      event.markPublished(clock.instant());
+    } catch (InterruptedException exception) {
+      Thread.currentThread().interrupt();
+      LOG.warn("Interrupted publishing outbox event {}", event.eventId());
+    } catch (Exception exception) {
+      LOG.warn("Outbox event {} remains pending", event.eventId());
+    }
+  }
+}


## `test(outbox): verify acknowledged publication proof`

diff --git a/src/test/java/com/sportsbook/betting/outbox/OutboxPublisherTest.java b/src/test/java/com/sportsbook/betting/outbox/OutboxPublisherTest.java
new file mode 100644
index 0000000..6b4c374
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/outbox/OutboxPublisherTest.java
@@ -0,0 +1,47 @@
+package com.sportsbook.betting.outbox;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.ArgumentMatchers.any;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.when;
+
+import java.time.Clock;
+import java.time.Instant;
+import java.time.ZoneOffset;
+import java.util.List;
+import java.util.UUID;
+import java.util.concurrent.CompletableFuture;
+import org.junit.jupiter.api.Test;
+import org.springframework.data.domain.Pageable;
+import org.springframework.kafka.core.KafkaTemplate;
+import org.springframework.scheduling.annotation.Scheduled;
+
+class OutboxPublisherTest {
+
+  @Test
+  void recordsPublicationOnlyAfterKafkaAcknowledgement() {
+    OutboxEventRepository repository = mock(OutboxEventRepository.class);
+    @SuppressWarnings("unchecked")
+    KafkaTemplate<String, byte[]> kafka = mock(KafkaTemplate.class);
+    OutboxEvent event =
+        OutboxEvent.pending(
+            UUID.randomUUID(), "topic", "key", "Schema", new byte[] {1}, Instant.EPOCH);
+    when(repository.findUnpublished(any(Pageable.class))).thenReturn(List.of(event));
+    when(kafka.send(any(org.apache.kafka.clients.producer.ProducerRecord.class)))
+        .thenReturn(CompletableFuture.completedFuture(null));
+    Instant now = Instant.parse("2026-08-22T00:00:00Z");
+
+    new OutboxPublisher(repository, kafka, Clock.fixed(now, ZoneOffset.UTC)).publishPending();
+
+    assertThat(event.publishedAt()).isEqualTo(now);
+  }
+
+  @Test
+  void usesTheIsolatedSchedulerAndBoundedDeliveryWait() throws Exception {
+    Scheduled scheduled =
+        OutboxPublisher.class.getMethod("publishPending").getAnnotation(Scheduled.class);
+
+    assertThat(scheduled.scheduler()).isEqualTo("outboxTaskScheduler");
+    assertThat(OutboxPublisher.TIMEOUT_SECONDS).isEqualTo(11);
+  }
+}


## `test(integration): verify PostgreSQL placement outbox`

diff --git a/src/test/java/com/sportsbook/betting/integration/PostgresSagaIntegrationTest.java b/src/test/java/com/sportsbook/betting/integration/PostgresSagaIntegrationTest.java
new file mode 100644
index 0000000..f38442d
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/integration/PostgresSagaIntegrationTest.java
@@ -0,0 +1,92 @@
+package com.sportsbook.betting.integration;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.betting.domain.Bet;
+import com.sportsbook.betting.domain.BetDraft;
+import com.sportsbook.betting.domain.BetLeg;
+import com.sportsbook.betting.outbox.OutboxEvent;
+import com.sportsbook.betting.outbox.OutboxEventRepository;
+import com.sportsbook.betting.persistence.BetRepository;
+import com.sportsbook.betting.persistence.PlacementRequestRepository;
+import com.sportsbook.betting.placement.BetStore;
+import com.sportsbook.betting.support.PostgresIntegrationSupport;
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.domain.BetStatus;
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.Odds;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.test.context.ActiveProfiles;
+import org.testcontainers.junit.jupiter.Testcontainers;
+
+@Testcontainers
+@ActiveProfiles("test")
+@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
+class PostgresSagaIntegrationTest extends PostgresIntegrationSupport {
+
+  @Autowired BetStore store;
+  @Autowired BetRepository bets;
+  @Autowired OutboxEventRepository outbox;
+  @Autowired PlacementRequestRepository requests;
+  @Autowired JdbcTemplate jdbc;
+
+  @Test
+  void migratesAndCommitsPlacementWithItsOutbox() {
+    Instant now = Instant.parse("2026-08-22T00:00:00Z");
+    UUID betId = UUID.randomUUID();
+    UUID userId = UUID.randomUUID();
+    Bet bet =
+        Bet.pending(
+            new BetDraft(
+                betId,
+                userId,
+                "B-2026-08-22-00000001",
+                new BetSlipType.Single(),
+                Money.krw(1_000),
+                Money.krw(2_000),
+                IdempotencyKey.of("postgres-saga-" + betId),
+                "a".repeat(64),
+                now),
+            List.of(
+                BetLeg.create(
+                    UUID.randomUUID(),
+                    UUID.randomUUID(),
+                    UUID.randomUUID(),
+                    Odds.ofDecimal("2.0"))));
+
+    store.savePending(bet);
+    store.recordRiskReservation(
+        betId, now.plusSeconds(120), "b".repeat(64), false, now.plusSeconds(1));
+    store.confirmWallet(betId, UUID.randomUUID(), now.plusSeconds(2));
+    store.commitRisk(betId, now.plusSeconds(3));
+    UUID outboxId = UUID.randomUUID();
+    store.acceptAndEnqueue(
+        betId,
+        OutboxEvent.pending(
+            outboxId,
+            "bet.placed.v1",
+            userId.toString(),
+            "BetPlacedRequested",
+            new byte[] {1},
+            now),
+        now.plusSeconds(4));
+
+    Bet reloaded = bets.findWithLegsByBetId(betId).orElseThrow();
+    assertThat(reloaded.status()).isEqualTo(BetStatus.ACCEPTED);
+    assertThat(reloaded.riskReservationToken()).isEqualTo("b".repeat(64));
+    assertThat(reloaded.legs()).hasSize(1);
+    assertThat(outbox.findById(outboxId)).isPresent();
+    assertThat(requests.findById("postgres-saga-" + betId)).isPresent();
+    assertThat(
+            jdbc.queryForObject(
+                "select count(*) from flyway_schema_history where success", Integer.class))
+        .isEqualTo(9);
+  }
+}
diff --git a/src/test/resources/application-test.yml b/src/test/resources/application-test.yml
index beb59da..23b3f3c 100644
--- a/src/test/resources/application-test.yml
+++ b/src/test/resources/application-test.yml
@@ -6,7 +6,7 @@ spring:
     redis:
       timeout: 2s
 
-BETTING_GATEWAY_API_KEY: ${BETTING_GATEWAY_API_KEY:gggggggggggggggggggggggggggggggg}
+BETTING_GATEWAY_API_KEY: gggggggggggggggggggggggggggggggg
 
 betting:
   clients:
