# 트랜잭셔널 아웃박스와 승인 기반 최소 한 번 전달

## `feat(outbox): model immutable pending events`

diff --git a/src/main/java/com/sportsbook/betting/outbox/OutboxEvent.java b/src/main/java/com/sportsbook/betting/outbox/OutboxEvent.java
new file mode 100644
index 0000000..cbfe13e
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/outbox/OutboxEvent.java
@@ -0,0 +1,96 @@
+package com.sportsbook.betting.outbox;
+
+import jakarta.persistence.Column;
+import jakarta.persistence.Entity;
+import jakarta.persistence.Id;
+import jakarta.persistence.Table;
+import java.time.Instant;
+import java.util.Objects;
+import java.util.UUID;
+
+@Entity
+@Table(name = "outbox_event")
+public class OutboxEvent {
+
+  @Id
+  @Column(name = "event_id", nullable = false, updatable = false)
+  private UUID eventId;
+
+  @Column(name = "topic", nullable = false, updatable = false, length = 64)
+  private String topic;
+
+  @Column(name = "partition_key", nullable = false, updatable = false, length = 64)
+  private String partitionKey;
+
+  @Column(name = "schema_name", nullable = false, updatable = false, length = 64)
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
+    this.topic = requireText(topic, "topic");
+    this.partitionKey = requireText(partitionKey, "partitionKey");
+    this.schemaName = requireText(schemaName, "schemaName");
+    this.payload = Objects.requireNonNull(payload, "payload").clone();
+    this.createdAt = Objects.requireNonNull(createdAt, "createdAt");
+  }
+
+  public static OutboxEvent pending(
+      UUID eventId,
+      String topic,
+      String partitionKey,
+      String schemaName,
+      byte[] payload,
+      Instant createdAt) {
+    return new OutboxEvent(eventId, topic, partitionKey, schemaName, payload, createdAt);
+  }
+
+  public void markPublished(Instant at) {
+    if (publishedAt == null) {
+      publishedAt = Objects.requireNonNull(at, "at");
+    }
+  }
+
+  private static String requireText(String value, String name) {
+    if (value == null || value.isBlank()) {
+      throw new IllegalArgumentException(name + " must not be blank");
+    }
+    return value;
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
+    return payload.clone();
+  }
+
+  public Instant publishedAt() {
+    return publishedAt;
+  }
+}


## `test(outbox): verify event immutability`

diff --git a/src/test/java/com/sportsbook/betting/outbox/OutboxEventTest.java b/src/test/java/com/sportsbook/betting/outbox/OutboxEventTest.java
new file mode 100644
index 0000000..0de82b3
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/outbox/OutboxEventTest.java
@@ -0,0 +1,26 @@
+package com.sportsbook.betting.outbox;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.time.Instant;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class OutboxEventTest {
+
+  @Test
+  void ownsPayloadAndKeepsFirstPublicationProof() {
+    byte[] source = {1, 2};
+    OutboxEvent event =
+        OutboxEvent.pending(UUID.randomUUID(), "topic", "key", "Schema", source, Instant.EPOCH);
+    source[0] = 9;
+    event.payload()[1] = 9;
+
+    Instant first = Instant.parse("2026-08-22T00:00:00Z");
+    event.markPublished(first);
+    event.markPublished(first.plusSeconds(1));
+
+    assertThat(event.payload()).containsExactly(1, 2);
+    assertThat(event.publishedAt()).isEqualTo(first);
+  }
+}


## `feat(database): add transactional outbox`

diff --git a/src/main/resources/db/migration/V2__outbox.sql b/src/main/resources/db/migration/V2__outbox.sql
new file mode 100644
index 0000000..3db5a81
--- /dev/null
+++ b/src/main/resources/db/migration/V2__outbox.sql
@@ -0,0 +1,29 @@
+-- V2: transactional outbox table (ADR-0006 Saga + Outbox).
+--
+-- The placement flow writes a BetPlacedRequested row here in the SAME DB
+-- transaction that flips the bet to ACCEPTED (ADR-0017 step 7), so the event
+-- can neither vanish (rolled back with the acceptance) nor be emitted for a bet
+-- that was never accepted. A scheduled publisher drains unpublished rows to
+-- Kafka and stamps published_at on ack.
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
+COMMENT ON COLUMN outbox_event.partition_key IS 'Kafka partition key = userId (per-user ordering, co-locates a user''s bets for risk aggregation).';
+COMMENT ON COLUMN outbox_event.schema_name   IS 'Avro record name (BetPlacedRequested) for diagnostics.';
+COMMENT ON COLUMN outbox_event.payload       IS 'Avro binary-encoded record bytes (no Schema Registry in V1, ADR-0014).';
+COMMENT ON COLUMN outbox_event.published_at  IS 'Set when the publisher receives a Kafka ack; NULL means still pending.';
+
+-- Hot read path: "unpublished rows oldest first". Partial index on the rare
+-- NULL state keeps the scan small as the table grows.
+CREATE INDEX ix_outbox_unpublished
+    ON outbox_event (created_at)
+    WHERE published_at IS NULL;


## `test(database): lock outbox schema checksum`

diff --git a/src/test/java/com/sportsbook/betting/persistence/MigrationContractTest.java b/src/test/java/com/sportsbook/betting/persistence/MigrationContractTest.java
index 5905fa3..eeae7d1 100644
--- a/src/test/java/com/sportsbook/betting/persistence/MigrationContractTest.java
+++ b/src/test/java/com/sportsbook/betting/persistence/MigrationContractTest.java
@@ -17,6 +17,12 @@ class MigrationContractTest {
         .isEqualTo("ecab4d9c22ab9bc83b35999db9c8a5c08abf8a80fc8e9263f23bcd369a84f29c");
   }
 
+  @Test
+  void preservesTransactionalOutboxSchema() {
+    assertThat(sha256("V2__outbox.sql"))
+        .isEqualTo("28161d23320d94a41d17b64a1dd0e2c9513fdfa74ac10ea1fb86bc4edf2c3d39");
+  }
+
   private String sha256(String migration) {
     try (InputStream input =
         getClass().getResourceAsStream("/db/migration/" + migration)) {


## `feat(outbox): query bounded pending batches`

diff --git a/src/main/java/com/sportsbook/betting/outbox/OutboxEventRepository.java b/src/main/java/com/sportsbook/betting/outbox/OutboxEventRepository.java
new file mode 100644
index 0000000..34b3da4
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/outbox/OutboxEventRepository.java
@@ -0,0 +1,13 @@
+package com.sportsbook.betting.outbox;
+
+import java.util.List;
+import java.util.UUID;
+import org.springframework.data.domain.Pageable;
+import org.springframework.data.jpa.repository.JpaRepository;
+import org.springframework.data.jpa.repository.Query;
+
+public interface OutboxEventRepository extends JpaRepository<OutboxEvent, UUID> {
+
+  @Query("select e from OutboxEvent e where e.publishedAt is null order by e.createdAt asc")
+  List<OutboxEvent> findUnpublished(Pageable pageable);
+}


## `test(outbox): verify pending batch query`

diff --git a/src/test/java/com/sportsbook/betting/outbox/OutboxEventRepositoryTest.java b/src/test/java/com/sportsbook/betting/outbox/OutboxEventRepositoryTest.java
new file mode 100644
index 0000000..dd80226
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/outbox/OutboxEventRepositoryTest.java
@@ -0,0 +1,20 @@
+package com.sportsbook.betting.outbox;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.lang.reflect.Method;
+import org.junit.jupiter.api.Test;
+import org.springframework.data.domain.Pageable;
+import org.springframework.data.jpa.repository.Query;
+
+class OutboxEventRepositoryTest {
+
+  @Test
+  void ordersOnlyUnpublishedRowsByCreationTime() throws NoSuchMethodException {
+    Method method = OutboxEventRepository.class.getMethod("findUnpublished", Pageable.class);
+
+    assertThat(method.getAnnotation(Query.class).value())
+        .isEqualTo(
+            "select e from OutboxEvent e where e.publishedAt is null order by e.createdAt asc");
+  }
+}


## `feat(outbox): encode strict Avro payloads`

diff --git a/src/main/java/com/sportsbook/betting/outbox/AvroSerializer.java b/src/main/java/com/sportsbook/betting/outbox/AvroSerializer.java
new file mode 100644
index 0000000..4e7c2db
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/outbox/AvroSerializer.java
@@ -0,0 +1,44 @@
+package com.sportsbook.betting.outbox;
+
+import java.io.ByteArrayOutputStream;
+import java.io.IOException;
+import java.util.Objects;
+import org.apache.avro.io.BinaryDecoder;
+import org.apache.avro.io.BinaryEncoder;
+import org.apache.avro.io.DecoderFactory;
+import org.apache.avro.io.EncoderFactory;
+import org.apache.avro.specific.SpecificDatumReader;
+import org.apache.avro.specific.SpecificDatumWriter;
+import org.apache.avro.specific.SpecificRecord;
+
+public final class AvroSerializer {
+
+  public static byte[] serialize(SpecificRecord record) {
+    Objects.requireNonNull(record, "record");
+    try {
+      ByteArrayOutputStream output = new ByteArrayOutputStream();
+      BinaryEncoder encoder = EncoderFactory.get().binaryEncoder(output, null);
+      new SpecificDatumWriter<SpecificRecord>(record.getSchema()).write(record, encoder);
+      encoder.flush();
+      return output.toByteArray();
+    } catch (IOException exception) {
+      throw new IllegalArgumentException("Could not encode Avro record", exception);
+    }
+  }
+
+  public static <T extends SpecificRecord> T deserialize(byte[] payload, Class<T> type) {
+    Objects.requireNonNull(payload, "payload");
+    try {
+      BinaryDecoder decoder = DecoderFactory.get().binaryDecoder(payload, null);
+      T value = new SpecificDatumReader<T>(type).read(null, decoder);
+      if (!decoder.isEnd()) {
+        throw new IllegalArgumentException("Trailing bytes after Avro record");
+      }
+      return value;
+    } catch (IOException exception) {
+      throw new IllegalArgumentException("Could not decode Avro record", exception);
+    }
+  }
+
+  private AvroSerializer() {}
+}


## `test(outbox): reject trailing Avro bytes`

diff --git a/src/test/java/com/sportsbook/betting/outbox/AvroSerializerTest.java b/src/test/java/com/sportsbook/betting/outbox/AvroSerializerTest.java
new file mode 100644
index 0000000..7908aa1
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/outbox/AvroSerializerTest.java
@@ -0,0 +1,24 @@
+package com.sportsbook.betting.outbox;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.protocol.event.Money;
+import java.util.Arrays;
+import org.junit.jupiter.api.Test;
+
+class AvroSerializerTest {
+
+  @Test
+  void roundTripsOneRecordAndRejectsTrailingBytes() {
+    Money money = Money.newBuilder().setAmount(1_000L).setCurrency("KRW").build();
+    byte[] encoded = AvroSerializer.serialize(money);
+
+    assertThat(AvroSerializer.deserialize(encoded, Money.class)).isEqualTo(money);
+
+    byte[] tainted = Arrays.copyOf(encoded, encoded.length + 1);
+    assertThatThrownBy(() -> AvroSerializer.deserialize(tainted, Money.class))
+        .isInstanceOf(IllegalArgumentException.class)
+        .hasMessageContaining("Trailing");
+  }
+}


## `feat(outbox): build unit-stake placement events`

diff --git a/src/main/java/com/sportsbook/betting/outbox/BetEventFactory.java b/src/main/java/com/sportsbook/betting/outbox/BetEventFactory.java
new file mode 100644
index 0000000..3412170
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/outbox/BetEventFactory.java
@@ -0,0 +1,64 @@
+package com.sportsbook.betting.outbox;
+
+import com.sportsbook.betting.config.BettingTopics;
+import com.sportsbook.betting.domain.Bet;
+import com.sportsbook.betting.infrastructure.id.UuidV7;
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.event.BetPlacedRequested;
+import com.sportsbook.protocol.event.BetSlipTypeTag;
+import com.sportsbook.protocol.event.RequestedSelection;
+import java.time.Instant;
+import org.springframework.stereotype.Component;
+
+@Component
+public class BetEventFactory {
+
+  public OutboxEvent placedRequested(Bet bet, Instant now) {
+    BetSlipType type = bet.slipType();
+    BetPlacedRequested record =
+        BetPlacedRequested.newBuilder()
+            .setBetId(bet.betId().toString())
+            .setUserId(bet.userId().toString())
+            .setSlipType(tag(type))
+            .setSystemMinWins(type instanceof BetSlipType.System system ? system.minWins() : null)
+            .setSystemTotalSelections(
+                type instanceof BetSlipType.System system ? system.totalSelections() : null)
+            .setSelections(
+                bet.legs().stream()
+                    .map(
+                        leg ->
+                            RequestedSelection.newBuilder()
+                                .setEventId(leg.eventId().toString())
+                                .setMarketId(leg.marketId().toString())
+                                .setSelectionId(leg.selectionId().toString())
+                                .setOddsAtSubmission(
+                                    leg.oddsAtSubmission().decimal().toPlainString())
+                                .build())
+                    .toList())
+            .setStake(
+                com.sportsbook.protocol.event.Money.newBuilder()
+                    .setAmount(bet.stake().amount())
+                    .setCurrency(bet.stake().currency().name())
+                    .build())
+            .setIdempotencyKey(bet.idempotencyKey())
+            .setRequestedAt(bet.createdAt())
+            .build();
+    return OutboxEvent.pending(
+        UuidV7.generate(),
+        BettingTopics.BET_PLACED,
+        bet.userId().toString(),
+        BetPlacedRequested.getClassSchema().getName(),
+        AvroSerializer.serialize(record),
+        now);
+  }
+
+  private static BetSlipTypeTag tag(BetSlipType type) {
+    if (type instanceof BetSlipType.Single) {
+      return BetSlipTypeTag.SINGLE;
+    }
+    if (type instanceof BetSlipType.Multiple) {
+      return BetSlipTypeTag.MULTIPLE;
+    }
+    return BetSlipTypeTag.SYSTEM;
+  }
+}


## `test(outbox): lock system unit-stake contract`

diff --git a/src/test/java/com/sportsbook/betting/outbox/BetEventFactoryTest.java b/src/test/java/com/sportsbook/betting/outbox/BetEventFactoryTest.java
new file mode 100644
index 0000000..70a1998
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/outbox/BetEventFactoryTest.java
@@ -0,0 +1,81 @@
+package com.sportsbook.betting.outbox;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.betting.domain.Bet;
+import com.sportsbook.betting.domain.BetDraft;
+import com.sportsbook.betting.domain.BetLeg;
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.event.BetPlacedRequested;
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.Odds;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.junit.jupiter.params.ParameterizedTest;
+import org.junit.jupiter.params.provider.MethodSource;
+
+class BetEventFactoryTest {
+
+  @Test
+  void publishesSystemUnitStakeRatherThanExposure() {
+    BetDraft draft =
+        new BetDraft(
+            UUID.randomUUID(),
+            UUID.randomUUID(),
+            "B-2026-08-22-00000000",
+            new BetSlipType.System(2, 3),
+            Money.krw(100),
+            Money.krw(2_600),
+            IdempotencyKey.of("request-1"),
+            "a".repeat(64),
+            Instant.EPOCH);
+    Bet bet = Bet.pending(draft, List.of(leg(), leg(), leg()));
+
+    OutboxEvent event = new BetEventFactory().placedRequested(bet, Instant.EPOCH);
+    BetPlacedRequested payload =
+        AvroSerializer.deserialize(event.payload(), BetPlacedRequested.class);
+
+    assertThat(payload.getStake().getAmount()).isEqualTo(100);
+    assertThat(payload.getSelections()).hasSize(3);
+    assertThat(event.partitionKey()).isEqualTo(bet.userId().toString());
+  }
+
+  @ParameterizedTest
+  @MethodSource("nonSystemTypes")
+  void omitsBothSystemFieldsForNonSystemSlips(BetSlipType type) {
+    List<BetLeg> legs = type instanceof BetSlipType.Single ? List.of(leg()) : List.of(leg(), leg());
+    Bet bet =
+        Bet.pending(
+            new BetDraft(
+                UUID.randomUUID(),
+                UUID.randomUUID(),
+                "B-2026-08-22-00000001",
+                type,
+                Money.krw(100),
+                Money.krw(200),
+                IdempotencyKey.of("request-2"),
+                "b".repeat(64),
+                Instant.EPOCH),
+            legs);
+
+    BetPlacedRequested payload =
+        AvroSerializer.deserialize(
+            new BetEventFactory().placedRequested(bet, Instant.EPOCH).payload(),
+            BetPlacedRequested.class);
+
+    assertThat(payload.getSystemMinWins()).isNull();
+    assertThat(payload.getSystemTotalSelections()).isNull();
+  }
+
+  static List<BetSlipType> nonSystemTypes() {
+    return List.of(new BetSlipType.Single(), new BetSlipType.Multiple());
+  }
+
+  private BetLeg leg() {
+    return BetLeg.create(
+        UUID.randomUUID(), UUID.randomUUID(), UUID.randomUUID(), Odds.ofDecimal("2.0"));
+  }
+}


## `feat(messaging): configure idempotent Kafka producer`

diff --git a/src/main/java/com/sportsbook/betting/config/KafkaConfig.java b/src/main/java/com/sportsbook/betting/config/KafkaConfig.java
new file mode 100644
index 0000000..61d3eb5
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/config/KafkaConfig.java
@@ -0,0 +1,49 @@
+package com.sportsbook.betting.config;
+
+import java.util.HashMap;
+import java.util.Map;
+import org.apache.kafka.clients.producer.ProducerConfig;
+import org.apache.kafka.common.serialization.ByteArraySerializer;
+import org.apache.kafka.common.serialization.StringSerializer;
+import org.springframework.beans.factory.annotation.Value;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.kafka.core.DefaultKafkaProducerFactory;
+import org.springframework.kafka.core.KafkaTemplate;
+import org.springframework.kafka.core.ProducerFactory;
+
+@Configuration
+public class KafkaConfig {
+
+  private final String bootstrapServers;
+
+  public KafkaConfig(@Value("${spring.kafka.bootstrap-servers}") String bootstrapServers) {
+    this.bootstrapServers = bootstrapServers;
+  }
+
+  @Bean
+  ProducerFactory<String, byte[]> bettingProducerFactory() {
+    return new DefaultKafkaProducerFactory<>(producerProperties(bootstrapServers));
+  }
+
+  @Bean
+  KafkaTemplate<String, byte[]> bettingKafkaTemplate(
+      ProducerFactory<String, byte[]> bettingProducerFactory) {
+    return new KafkaTemplate<>(bettingProducerFactory);
+  }
+
+  static Map<String, Object> producerProperties(String bootstrapServers) {
+    Map<String, Object> properties = new HashMap<>();
+    properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
+    properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
+    properties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, ByteArraySerializer.class);
+    properties.put(ProducerConfig.ACKS_CONFIG, "all");
+    properties.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
+    properties.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);
+    properties.put(ProducerConfig.MAX_BLOCK_MS_CONFIG, 5_000);
+    properties.put(ProducerConfig.REQUEST_TIMEOUT_MS_CONFIG, 5_000);
+    properties.put(ProducerConfig.DELIVERY_TIMEOUT_MS_CONFIG, 10_000);
+    properties.put(ProducerConfig.CLIENT_ID_CONFIG, "betting-service-outbox");
+    return Map.copyOf(properties);
+  }
+}


