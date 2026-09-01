# 트랜잭션 아웃박스와 브로커 확인 경계

## `feat(outbox): encode raw Avro payloads`

diff --git a/src/main/java/com/sportsbook/settlement/outbox/StrictAvroEncoder.java b/src/main/java/com/sportsbook/settlement/outbox/StrictAvroEncoder.java
new file mode 100644
index 0000000..7357460
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/outbox/StrictAvroEncoder.java
@@ -0,0 +1,26 @@
+package com.sportsbook.settlement.outbox;
+
+import java.io.ByteArrayOutputStream;
+import java.util.Objects;
+import org.apache.avro.io.BinaryEncoder;
+import org.apache.avro.io.EncoderFactory;
+import org.apache.avro.specific.SpecificDatumWriter;
+import org.apache.avro.specific.SpecificRecordBase;
+
+/** Encodes deterministic raw Avro bytes without a schema-registry prefix. */
+public final class StrictAvroEncoder {
+
+  public <T extends SpecificRecordBase> byte[] encode(T record) {
+    Objects.requireNonNull(record, "record");
+    try {
+      ByteArrayOutputStream output = new ByteArrayOutputStream();
+      BinaryEncoder encoder = EncoderFactory.get().binaryEncoder(output, null);
+      new SpecificDatumWriter<T>(record.getSchema()).write(record, encoder);
+      encoder.flush();
+      return output.toByteArray();
+    } catch (Exception exception) {
+      throw new IllegalStateException(
+          "Failed to encode " + record.getClass().getSimpleName(), exception);
+    }
+  }
+}


## `test(outbox): verify strict Avro round trips`

diff --git a/src/test/java/com/sportsbook/settlement/outbox/StrictAvroEncoderTest.java b/src/test/java/com/sportsbook/settlement/outbox/StrictAvroEncoderTest.java
new file mode 100644
index 0000000..088e517
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/outbox/StrictAvroEncoderTest.java
@@ -0,0 +1,23 @@
+package com.sportsbook.settlement.outbox;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.event.Money;
+import com.sportsbook.settlement.event.StrictAvroDecoder;
+import org.junit.jupiter.api.Test;
+
+class StrictAvroEncoderTest {
+
+  private final StrictAvroEncoder encoder = new StrictAvroEncoder();
+
+  @Test
+  void producesDeterministicRawBytesAcceptedByTheStrictDecoder() {
+    Money record = Money.newBuilder().setAmount(1_000L).setCurrency("KRW").build();
+
+    byte[] first = encoder.encode(record);
+    byte[] second = encoder.encode(record);
+
+    assertThat(first).containsExactly(second);
+    assertThat(new StrictAvroDecoder().decode(first, Money.class)).isEqualTo(record);
+  }
+}


## `feat(outbox): model durable pending events`

diff --git a/src/main/java/com/sportsbook/settlement/outbox/OutboxEvent.java b/src/main/java/com/sportsbook/settlement/outbox/OutboxEvent.java
new file mode 100644
index 0000000..2fe274f
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/outbox/OutboxEvent.java
@@ -0,0 +1,94 @@
+package com.sportsbook.settlement.outbox;
+
+import com.sportsbook.settlement.infrastructure.id.UuidV7;
+import jakarta.persistence.Column;
+import jakarta.persistence.Entity;
+import jakarta.persistence.Id;
+import jakarta.persistence.Table;
+import java.time.Instant;
+import java.util.Arrays;
+import java.util.Objects;
+import java.util.UUID;
+
+/** Durable raw event payload committed in the same transaction as settlement state. */
+@Entity
+@Table(name = "outbox_event")
+public class OutboxEvent {
+
+  @Id
+  @Column(name = "event_id", nullable = false, updatable = false)
+  private UUID eventId;
+
+  @Column(name = "topic", nullable = false, length = 64, updatable = false)
+  private String topic;
+
+  @Column(name = "partition_key", nullable = false, length = 64, updatable = false)
+  private String partitionKey;
+
+  @Column(name = "schema_name", nullable = false, length = 64, updatable = false)
+  private String schemaName;
+
+  @Column(name = "payload", nullable = false, updatable = false)
+  private byte[] payload;
+
+  @Column(name = "created_at", nullable = false, updatable = false)
+  private Instant createdAt;
+
+  @Column(name = "published_at")
+  private Instant publishedAt;
+
+  protected OutboxEvent() {}
+
+  private OutboxEvent(
+      UUID eventId,
+      String topic,
+      String partitionKey,
+      String schemaName,
+      byte[] payload,
+      Instant createdAt) {
+    this.eventId = Objects.requireNonNull(eventId, "eventId");
+    this.topic = required(topic, "topic");
+    this.partitionKey = required(partitionKey, "partitionKey");
+    this.schemaName = required(schemaName, "schemaName");
+    this.payload = Arrays.copyOf(Objects.requireNonNull(payload, "payload"), payload.length);
+    this.createdAt = Objects.requireNonNull(createdAt, "createdAt");
+  }
+
+  public static OutboxEvent pending(
+      String topic, String partitionKey, String schemaName, byte[] payload, Instant createdAt) {
+    return new OutboxEvent(UuidV7.generate(), topic, partitionKey, schemaName, payload, createdAt);
+  }
+
+  public void markPublished(Instant when) {
+    if (publishedAt == null) {
+      publishedAt = Objects.requireNonNull(when, "when");
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
+  public String partitionKey() {
+    return partitionKey;
+  }
+
+  public byte[] payload() {
+    return Arrays.copyOf(payload, payload.length);
+  }
+
+  public Instant publishedAt() {
+    return publishedAt;
+  }
+
+  private static String required(String value, String field) {
+    if (value == null || value.isBlank()) {
+      throw new IllegalArgumentException(field + " is required");
+    }
+    return value;
+  }
+}


## `build(flyway): add V3 transactional outbox`

diff --git a/src/main/resources/db/migration/V3__outbox.sql b/src/main/resources/db/migration/V3__outbox.sql
new file mode 100644
index 0000000..a9b136b
--- /dev/null
+++ b/src/main/resources/db/migration/V3__outbox.sql
@@ -0,0 +1,29 @@
+-- V3: transactional outbox table (ADR-0006 Saga + Outbox).
+--
+-- The settlement flow writes a BetSettled / BetVoided row here in the SAME DB
+-- transaction that flips the bet to SETTLED / VOIDED, so the event can neither
+-- vanish (rolled back with the transition) nor be emitted for a bet that was
+-- never settled. A scheduled publisher drains unpublished rows to Kafka and
+-- stamps published_at on ack.
+
+CREATE TABLE outbox_event (
+    event_id      UUID                     PRIMARY KEY,
+    topic         VARCHAR(64)              NOT NULL,
+    partition_key VARCHAR(64)              NOT NULL,
+    schema_name   VARCHAR(64)              NOT NULL,
+    payload       BYTEA                    NOT NULL,
+    created_at    TIMESTAMP WITH TIME ZONE NOT NULL,
+    published_at  TIMESTAMP WITH TIME ZONE
+);
+
+COMMENT ON TABLE  outbox_event               IS 'Transactional outbox — rows are published to Kafka by OutboxPublisher.';
+COMMENT ON COLUMN outbox_event.partition_key IS 'Kafka partition key = eventId (ADR-0006: a match''s settlement events share a partition, preserving order).';
+COMMENT ON COLUMN outbox_event.schema_name   IS 'Avro record name (BetSettled / BetVoided) for diagnostics.';
+COMMENT ON COLUMN outbox_event.payload       IS 'Avro binary-encoded record bytes (no Schema Registry in V1, ADR-0014).';
+COMMENT ON COLUMN outbox_event.published_at  IS 'Set when the publisher receives a Kafka ack; NULL means still pending.';
+
+-- Hot read path: "unpublished rows oldest first". Partial index on the rare
+-- NULL state keeps the scan small as the table grows.
+CREATE INDEX ix_outbox_unpublished
+    ON outbox_event (created_at)
+    WHERE published_at IS NULL;


## `feat(outbox): create BetSettled events`

diff --git a/src/main/java/com/sportsbook/settlement/outbox/SettlementEventFactory.java b/src/main/java/com/sportsbook/settlement/outbox/SettlementEventFactory.java
new file mode 100644
index 0000000..162b7b8
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/outbox/SettlementEventFactory.java
@@ -0,0 +1,65 @@
+package com.sportsbook.settlement.outbox;
+
+import com.sportsbook.protocol.event.BetSettled;
+import com.sportsbook.protocol.event.SettlementResultAvro;
+import com.sportsbook.settlement.config.SettlementTopics;
+import com.sportsbook.settlement.domain.Bet;
+import com.sportsbook.settlement.domain.BetSelection;
+import com.sportsbook.settlement.domain.SettlementStatus;
+import java.util.LinkedHashMap;
+import java.util.Map;
+import java.util.UUID;
+
+/** Creates fixed wire-v1 settlement records and their durable outbox envelopes. */
+public final class SettlementEventFactory {
+
+  private final SettlementTopics topics;
+  private final StrictAvroEncoder encoder;
+
+  public SettlementEventFactory(SettlementTopics topics, StrictAvroEncoder encoder) {
+    this.topics = topics;
+    this.encoder = encoder;
+  }
+
+  public OutboxEvent settled(Bet bet, UUID drivingEventId) {
+    if (bet.status() != SettlementStatus.SETTLED || bet.result() == null || bet.payout() == null) {
+      throw new IllegalStateException("Bet must be SETTLED before creating BetSettled");
+    }
+    BetSettled event =
+        BetSettled.newBuilder()
+            .setBetId(bet.betId().toString())
+            .setUserId(bet.userId().toString())
+            .setEventId(drivingEventId.toString())
+            .setResult(SettlementResultAvro.valueOf(bet.result().name()))
+            .setStake(money(bet.stake()))
+            .setPayout(money(bet.payout()))
+            .setSettledAt(bet.settledAt())
+            .setResultDetail(resultDetail(bet))
+            .build();
+    return OutboxEvent.pending(
+        topics.betSettled(),
+        drivingEventId.toString(),
+        BetSettled.class.getSimpleName(),
+        encoder.encode(event),
+        bet.settledAt());
+  }
+
+  private static com.sportsbook.protocol.event.Money money(
+      com.sportsbook.protocol.value.Money value) {
+    return com.sportsbook.protocol.event.Money.newBuilder()
+        .setAmount(value.amount())
+        .setCurrency(value.currency().name())
+        .build();
+  }
+
+  private static Map<String, String> resultDetail(Bet bet) {
+    Map<String, String> detail = new LinkedHashMap<>();
+    for (BetSelection selection : bet.selections()) {
+      if (selection.outcome() == null) {
+        throw new IllegalStateException("BetSettled requires every selection outcome");
+      }
+      detail.put(selection.selectionId().toString(), selection.outcome().name());
+    }
+    return Map.copyOf(detail);
+  }
+}


## `feat(outbox): create BetVoided events`

diff --git a/src/main/java/com/sportsbook/settlement/outbox/SettlementEventFactory.java b/src/main/java/com/sportsbook/settlement/outbox/SettlementEventFactory.java
index 162b7b8..fc31957 100644
--- a/src/main/java/com/sportsbook/settlement/outbox/SettlementEventFactory.java
+++ b/src/main/java/com/sportsbook/settlement/outbox/SettlementEventFactory.java
@@ -1,7 +1,9 @@
 package com.sportsbook.settlement.outbox;
 
 import com.sportsbook.protocol.event.BetSettled;
+import com.sportsbook.protocol.event.BetVoided;
 import com.sportsbook.protocol.event.SettlementResultAvro;
+import com.sportsbook.protocol.event.VoidReason;
 import com.sportsbook.settlement.config.SettlementTopics;
 import com.sportsbook.settlement.domain.Bet;
 import com.sportsbook.settlement.domain.BetSelection;
@@ -44,6 +46,27 @@ public final class SettlementEventFactory {
         bet.settledAt());
   }
 
+  public OutboxEvent voided(Bet bet, UUID drivingEventId, VoidReason reason) {
+    if (bet.status() != SettlementStatus.VOIDED || bet.payout() == null) {
+      throw new IllegalStateException("Bet must be VOIDED before creating BetVoided");
+    }
+    BetVoided event =
+        BetVoided.newBuilder()
+            .setBetId(bet.betId().toString())
+            .setUserId(bet.userId().toString())
+            .setEventId(drivingEventId.toString())
+            .setReason(reason)
+            .setRefund(money(bet.payout()))
+            .setVoidedAt(bet.settledAt())
+            .build();
+    return OutboxEvent.pending(
+        topics.betVoided(),
+        drivingEventId.toString(),
+        BetVoided.class.getSimpleName(),
+        encoder.encode(event),
+        bet.settledAt());
+  }
+
   private static com.sportsbook.protocol.event.Money money(
       com.sportsbook.protocol.value.Money value) {
     return com.sportsbook.protocol.event.Money.newBuilder()


## `feat(outbox): publish locked pending events`

diff --git a/src/main/java/com/sportsbook/settlement/outbox/OutboxEventRepository.java b/src/main/java/com/sportsbook/settlement/outbox/OutboxEventRepository.java
new file mode 100644
index 0000000..cc92bfd
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/outbox/OutboxEventRepository.java
@@ -0,0 +1,17 @@
+package com.sportsbook.settlement.outbox;
+
+import java.util.List;
+import java.util.UUID;
+import org.springframework.data.jpa.repository.JpaRepository;
+import org.springframework.data.jpa.repository.Query;
+import org.springframework.data.repository.query.Param;
+
+public interface OutboxEventRepository extends JpaRepository<OutboxEvent, UUID> {
+
+  @Query(
+      value =
+          "select * from outbox_event where published_at is null "
+              + "order by created_at, event_id for update skip locked limit :limit",
+      nativeQuery = true)
+  List<OutboxEvent> lockNextUnpublished(@Param("limit") int limit);
+}
diff --git a/src/main/java/com/sportsbook/settlement/outbox/OutboxPublisher.java b/src/main/java/com/sportsbook/settlement/outbox/OutboxPublisher.java
new file mode 100644
index 0000000..1eceb19
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/outbox/OutboxPublisher.java
@@ -0,0 +1,67 @@
+package com.sportsbook.settlement.outbox;
+
+import com.sportsbook.settlement.config.RawKafkaProducerConfiguration;
+import com.sportsbook.settlement.config.SettlementRuntimeProperties;
+import com.sportsbook.settlement.config.SettlementWorkerConfiguration;
+import java.nio.charset.StandardCharsets;
+import java.time.Clock;
+import java.util.concurrent.ExecutionException;
+import java.util.concurrent.TimeUnit;
+import java.util.concurrent.TimeoutException;
+import org.apache.kafka.clients.producer.ProducerRecord;
+import org.springframework.beans.factory.annotation.Qualifier;
+import org.springframework.kafka.KafkaException;
+import org.springframework.kafka.core.KafkaOperations;
+import org.springframework.scheduling.annotation.Scheduled;
+import org.springframework.stereotype.Component;
+import org.springframework.transaction.annotation.Transactional;
+
+/** Publishes locked outbox rows at least once and marks them only after broker acknowledgement. */
+@Component
+public class OutboxPublisher {
+
+  private static final long SEND_TIMEOUT_SECONDS = 11;
+
+  private final OutboxEventRepository repository;
+  private final KafkaOperations<byte[], byte[]> kafka;
+  private final SettlementRuntimeProperties runtime;
+  private final Clock clock;
+
+  public OutboxPublisher(
+      OutboxEventRepository repository,
+      @Qualifier(RawKafkaProducerConfiguration.OPERATIONS) KafkaOperations<byte[], byte[]> kafka,
+      SettlementRuntimeProperties runtime,
+      Clock clock) {
+    this.repository = repository;
+    this.kafka = kafka;
+    this.runtime = runtime;
+    this.clock = clock;
+  }
+
+  @Transactional
+  @Scheduled(
+      fixedDelayString = "${settlement.outbox.interval:PT1S}",
+      scheduler = SettlementWorkerConfiguration.OUTBOX)
+  public int publishBatch() {
+    var pending = repository.lockNextUnpublished(runtime.batchSize());
+    for (OutboxEvent event : pending) {
+      publish(event);
+      event.markPublished(clock.instant());
+    }
+    return pending.size();
+  }
+
+  private void publish(OutboxEvent event) {
+    ProducerRecord<byte[], byte[]> record =
+        new ProducerRecord<>(
+            event.topic(), event.partitionKey().getBytes(StandardCharsets.UTF_8), event.payload());
+    try {
+      kafka.send(record).get(SEND_TIMEOUT_SECONDS, TimeUnit.SECONDS);
+    } catch (InterruptedException exception) {
+      Thread.currentThread().interrupt();
+      throw new KafkaException("Interrupted while publishing outbox event", exception);
+    } catch (ExecutionException | TimeoutException exception) {
+      throw new KafkaException("Failed to publish outbox event", exception);
+    }
+  }
+}


## `test(outbox): verify broker ack publication boundary`

diff --git a/src/test/java/com/sportsbook/settlement/outbox/OutboxPublisherTest.java b/src/test/java/com/sportsbook/settlement/outbox/OutboxPublisherTest.java
new file mode 100644
index 0000000..aae1424
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/outbox/OutboxPublisherTest.java
@@ -0,0 +1,81 @@
+package com.sportsbook.settlement.outbox;
+
+import static java.nio.charset.StandardCharsets.UTF_8;
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+import static org.mockito.ArgumentMatchers.any;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
+
+import com.sportsbook.settlement.config.SettlementRuntimeProperties;
+import com.sportsbook.settlement.config.SettlementWorkerConfiguration;
+import java.time.Clock;
+import java.time.Instant;
+import java.time.ZoneOffset;
+import java.util.List;
+import java.util.concurrent.CompletableFuture;
+import org.apache.kafka.clients.producer.ProducerRecord;
+import org.junit.jupiter.api.Test;
+import org.mockito.ArgumentCaptor;
+import org.springframework.kafka.KafkaException;
+import org.springframework.kafka.core.KafkaOperations;
+import org.springframework.scheduling.annotation.Scheduled;
+
+class OutboxPublisherTest {
+
+  private static final Instant NOW = Instant.parse("2026-01-01T00:00:00Z");
+  private final OutboxEventRepository repository = mock(OutboxEventRepository.class);
+
+  @SuppressWarnings("unchecked")
+  private final KafkaOperations<byte[], byte[]> kafka = mock(KafkaOperations.class);
+
+  private final OutboxPublisher publisher =
+      new OutboxPublisher(
+          repository,
+          kafka,
+          new SettlementRuntimeProperties(null, null, null, 10),
+          Clock.fixed(NOW, ZoneOffset.UTC));
+
+  @Test
+  void marksRowsOnlyAfterTheRawBrokerSendCompletes() {
+    OutboxEvent event = pending();
+    when(repository.lockNextUnpublished(10)).thenReturn(List.of(event));
+    when(kafka.send(any(ProducerRecord.class))).thenReturn(CompletableFuture.completedFuture(null));
+
+    assertThat(publisher.publishBatch()).isOne();
+
+    ArgumentCaptor<ProducerRecord<byte[], byte[]>> sent =
+        ArgumentCaptor.forClass(ProducerRecord.class);
+    verify(kafka).send(sent.capture());
+    assertThat(new String(sent.getValue().key(), UTF_8)).isEqualTo("event-id");
+    assertThat(sent.getValue().value()).containsExactly(1, 2, 3);
+    assertThat(event.publishedAt()).isEqualTo(NOW);
+  }
+
+  @Test
+  void leavesTheRowPendingWhenTheBrokerSendFails() {
+    OutboxEvent event = pending();
+    when(repository.lockNextUnpublished(10)).thenReturn(List.of(event));
+    CompletableFuture<org.springframework.kafka.support.SendResult<byte[], byte[]>> failed =
+        new CompletableFuture<>();
+    failed.completeExceptionally(new IllegalStateException("broker unavailable"));
+    when(kafka.send(any(ProducerRecord.class))).thenReturn(failed);
+
+    assertThatThrownBy(publisher::publishBatch).isInstanceOf(KafkaException.class);
+    assertThat(event.publishedAt()).isNull();
+  }
+
+  @Test
+  void runsOnTheIsolatedOutboxScheduler() throws NoSuchMethodException {
+    Scheduled scheduled =
+        OutboxPublisher.class.getMethod("publishBatch").getAnnotation(Scheduled.class);
+
+    assertThat(scheduled.scheduler()).isEqualTo(SettlementWorkerConfiguration.OUTBOX);
+  }
+
+  private static OutboxEvent pending() {
+    return OutboxEvent.pending(
+        "bet.settled.v1", "event-id", "BetSettled", new byte[] {1, 2, 3}, Instant.EPOCH);
+  }
+}


