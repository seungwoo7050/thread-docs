# 리스 기반 FIFO 아웃박스 전달

## `feat(outbox): model fenced delivery leases`

diff --git a/src/main/java/com/sportsbook/wallet/outbox/LeasedOutboxMessage.java b/src/main/java/com/sportsbook/wallet/outbox/LeasedOutboxMessage.java
new file mode 100644
index 0000000..d194c9e
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/outbox/LeasedOutboxMessage.java
@@ -0,0 +1,46 @@
+package com.sportsbook.wallet.outbox;
+
+import java.time.Instant;
+import java.util.Objects;
+
+public record LeasedOutboxMessage(
+    OutboxLease lease,
+    String topic,
+    String partitionKey,
+    String schemaName,
+    byte[] payload,
+    long streamSequence,
+    boolean leaseTakeover,
+    int attemptCount,
+    Instant createdAt) {
+
+  public LeasedOutboxMessage {
+    lease = Objects.requireNonNull(lease, "lease");
+    topic = required(topic, "topic");
+    partitionKey = required(partitionKey, "partitionKey");
+    schemaName = required(schemaName, "schemaName");
+    payload = Objects.requireNonNull(payload, "payload").clone();
+    if (payload.length == 0) {
+      throw new IllegalArgumentException("payload must not be empty");
+    }
+    if (streamSequence < 1L) {
+      throw new IllegalArgumentException("streamSequence must be positive");
+    }
+    if (attemptCount < 1) {
+      throw new IllegalArgumentException("attemptCount must be positive");
+    }
+    createdAt = Objects.requireNonNull(createdAt, "createdAt");
+  }
+
+  @Override
+  public byte[] payload() {
+    return payload.clone();
+  }
+
+  private static String required(String value, String name) {
+    if (value == null || value.isBlank()) {
+      throw new IllegalArgumentException(name + " must not be blank");
+    }
+    return value;
+  }
+}
diff --git a/src/main/java/com/sportsbook/wallet/outbox/OutboxLease.java b/src/main/java/com/sportsbook/wallet/outbox/OutboxLease.java
new file mode 100644
index 0000000..5411303
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/outbox/OutboxLease.java
@@ -0,0 +1,23 @@
+package com.sportsbook.wallet.outbox;
+
+import java.time.Instant;
+import java.util.Objects;
+import java.util.UUID;
+
+public record OutboxLease(UUID eventId, String owner, long version, Instant leaseUntil) {
+
+  public OutboxLease {
+    eventId = Objects.requireNonNull(eventId, "eventId");
+    if (owner == null || owner.isBlank()) {
+      throw new IllegalArgumentException("owner must not be blank");
+    }
+    if (version < 1) {
+      throw new IllegalArgumentException("version must be positive");
+    }
+    leaseUntil = Objects.requireNonNull(leaseUntil, "leaseUntil");
+  }
+
+  public boolean isOwnedBy(String candidateOwner, long candidateVersion) {
+    return owner.equals(candidateOwner) && version == candidateVersion;
+  }
+}


## `feat(repository): claim strict FIFO outbox stream heads`

diff --git a/src/main/java/com/sportsbook/wallet/persistence/OutboxDeliveryRepository.java b/src/main/java/com/sportsbook/wallet/persistence/OutboxDeliveryRepository.java
new file mode 100644
index 0000000..cd86978
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/persistence/OutboxDeliveryRepository.java
@@ -0,0 +1,98 @@
+package com.sportsbook.wallet.persistence;
+
+import com.sportsbook.wallet.outbox.LeasedOutboxMessage;
+import com.sportsbook.wallet.outbox.OutboxLease;
+import java.sql.ResultSet;
+import java.sql.SQLException;
+import java.time.Duration;
+import java.util.Comparator;
+import java.util.List;
+import java.util.Map;
+import org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate;
+import org.springframework.stereotype.Repository;
+import org.springframework.transaction.annotation.Propagation;
+import org.springframework.transaction.annotation.Transactional;
+
+@Repository
+public class OutboxDeliveryRepository {
+
+  private static final String CLAIM_SQL =
+      """
+      WITH db_clock AS MATERIALIZED (
+          SELECT clock_timestamp() AS now
+      ), candidates AS MATERIALIZED (
+          SELECT e.event_id, e.lease_owner IS NOT NULL AS lease_takeover
+          FROM outbox_event e CROSS JOIN db_clock c
+          WHERE e.published_at IS NULL
+            AND e.available_at <= c.now
+            AND (e.lease_until IS NULL OR e.lease_until <= c.now)
+            AND NOT EXISTS (
+                SELECT 1 FROM outbox_event older
+                WHERE older.topic = e.topic
+                  AND older.partition_key = e.partition_key
+                  AND older.published_at IS NULL
+                  AND older.stream_sequence < e.stream_sequence
+            )
+          ORDER BY e.available_at, e.created_at, e.event_id
+          FOR UPDATE OF e SKIP LOCKED
+          LIMIT :batchSize
+      )
+      UPDATE outbox_event e
+      SET lease_owner = :owner,
+          lease_until = c.now + CAST(:leaseMillis AS bigint) * interval '1 millisecond',
+          lease_version = e.lease_version + 1,
+          attempt_count = e.attempt_count + 1
+      FROM candidates candidate CROSS JOIN db_clock c
+      WHERE e.event_id = candidate.event_id
+      RETURNING e.event_id, e.topic, e.partition_key, e.schema_name, e.payload,
+                e.stream_sequence, candidate.lease_takeover, e.attempt_count, e.created_at,
+                e.lease_owner, e.lease_version, e.lease_until
+      """;
+
+  private static final Comparator<LeasedOutboxMessage> DELIVERY_ORDER =
+      Comparator.comparing(LeasedOutboxMessage::createdAt)
+          .thenComparing(message -> message.lease().eventId());
+
+  private final NamedParameterJdbcTemplate jdbc;
+
+  public OutboxDeliveryRepository(NamedParameterJdbcTemplate jdbc) {
+    this.jdbc = jdbc;
+  }
+
+  @Transactional(propagation = Propagation.REQUIRES_NEW, timeout = 5)
+  public List<LeasedOutboxMessage> claim(String owner, int batchSize, Duration leaseDuration) {
+    if (owner == null || owner.isBlank()) {
+      throw new IllegalArgumentException("owner must not be blank");
+    }
+    if (batchSize < 1 || leaseDuration.toMillis() < 1L) {
+      throw new IllegalArgumentException("batch size and lease duration must be positive");
+    }
+    Map<String, Object> parameters =
+        Map.of(
+            "owner", owner,
+            "batchSize", batchSize,
+            "leaseMillis", leaseDuration.toMillis());
+    List<LeasedOutboxMessage> messages = jdbc.query(CLAIM_SQL, parameters, this::map);
+    messages.sort(DELIVERY_ORDER);
+    return List.copyOf(messages);
+  }
+
+  private LeasedOutboxMessage map(ResultSet resultSet, int rowNumber) throws SQLException {
+    OutboxLease lease =
+        new OutboxLease(
+            resultSet.getObject("event_id", java.util.UUID.class),
+            resultSet.getString("lease_owner"),
+            resultSet.getLong("lease_version"),
+            resultSet.getTimestamp("lease_until").toInstant());
+    return new LeasedOutboxMessage(
+        lease,
+        resultSet.getString("topic"),
+        resultSet.getString("partition_key"),
+        resultSet.getString("schema_name"),
+        resultSet.getBytes("payload"),
+        resultSet.getLong("stream_sequence"),
+        resultSet.getBoolean("lease_takeover"),
+        resultSet.getInt("attempt_count"),
+        resultSet.getTimestamp("created_at").toInstant());
+  }
+}


## `test(outbox): verify disjoint claims expiry and ordering`

diff --git a/src/test/java/com/sportsbook/wallet/persistence/OutboxDeliveryRepositoryTest.java b/src/test/java/com/sportsbook/wallet/persistence/OutboxDeliveryRepositoryTest.java
new file mode 100644
index 0000000..470d41d
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/persistence/OutboxDeliveryRepositoryTest.java
@@ -0,0 +1,72 @@
+package com.sportsbook.wallet.persistence;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.wallet.outbox.OutboxAppender;
+import java.time.Duration;
+import java.time.Instant;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
+import org.springframework.context.annotation.Import;
+import org.springframework.transaction.annotation.Propagation;
+import org.springframework.transaction.annotation.Transactional;
+import org.testcontainers.junit.jupiter.Testcontainers;
+
+@DataJpaTest(properties = "spring.test.database.replace=NONE")
+@Testcontainers
+@Transactional(propagation = Propagation.NOT_SUPPORTED)
+@Import({OutboxAppender.class, OutboxDeliveryRepository.class, OutboxStreamLock.class})
+class OutboxDeliveryRepositoryTest extends OutboxDeliveryRepositoryFixture {
+  @Autowired OutboxDeliveryRepository delivery;
+
+  @Test
+  void claimsDisjointKeysAcrossWorkers() {
+    Instant created = Instant.parse("2999-08-21T00:00:00Z");
+    persist("operation-a", "key-a", "dedup-a1", created);
+    persist("operation-b", "key-b", "dedup-b1", created.plusMillis(1));
+
+    var first = delivery.claim("worker-a", 1, Duration.ofSeconds(30));
+    var disjoint = delivery.claim("worker-b", 10, Duration.ofSeconds(30));
+
+    assertThat(first.get(0).partitionKey()).isEqualTo("key-a");
+    assertThat(first.get(0).streamSequence()).isEqualTo(1L);
+    assertThat(first.get(0).leaseTakeover()).isFalse();
+    assertThat(disjoint.get(0).partitionKey()).isEqualTo("key-b");
+  }
+
+  @Test
+  void reclaimsAnExpiredHeadWithANewFencingVersion() {
+    persist("operation-a", "key-a", "dedup-a1", Instant.parse("2026-08-21T00:00:00Z"));
+    persist("operation-a2", "key-a", "dedup-a2", Instant.parse("2026-08-21T00:00:01Z"));
+    var first = delivery.claim("worker-a", 1, Duration.ofSeconds(30)).get(0);
+    jdbc.update(
+        "UPDATE outbox_event SET lease_until=clock_timestamp()-interval '1 second' WHERE event_id=?",
+        first.lease().eventId());
+
+    var reclaimed = delivery.claim("worker-b", 10, Duration.ofSeconds(30));
+
+    assertThat(reclaimed).hasSize(1);
+    assertThat(reclaimed.get(0).lease().eventId()).isEqualTo(first.lease().eventId());
+    assertThat(reclaimed.get(0).streamSequence()).isEqualTo(first.streamSequence());
+    assertThat(reclaimed.get(0).leaseTakeover()).isTrue();
+    assertThat(reclaimed.get(0).lease().version()).isEqualTo(first.lease().version() + 1);
+  }
+
+  @Test
+  void rejectsASecondSemanticEventForOneOperation() {
+    Instant created = Instant.parse("2026-08-21T00:00:00Z");
+    persist("operation-unique", "key-a", "dedup-a1", created);
+
+    assertThatThrownBy(
+            () ->
+                persist(
+                    "operation-unique",
+                    "wallet.credited.v1",
+                    "key-a",
+                    "dedup-a2",
+                    created.plusMillis(1)))
+        .isInstanceOf(org.springframework.dao.DataIntegrityViolationException.class);
+  }
+}


## `feat(outbox): encode shared Avro records`

diff --git a/src/main/java/com/sportsbook/wallet/outbox/AvroSerializer.java b/src/main/java/com/sportsbook/wallet/outbox/AvroSerializer.java
new file mode 100644
index 0000000..6a1a7b1
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/outbox/AvroSerializer.java
@@ -0,0 +1,40 @@
+package com.sportsbook.wallet.outbox;
+
+import java.io.ByteArrayOutputStream;
+import java.io.IOException;
+import java.util.Objects;
+import org.apache.avro.io.DecoderFactory;
+import org.apache.avro.io.EncoderFactory;
+import org.apache.avro.specific.SpecificDatumReader;
+import org.apache.avro.specific.SpecificDatumWriter;
+import org.apache.avro.specific.SpecificRecord;
+
+public final class AvroSerializer {
+
+  private AvroSerializer() {}
+
+  public static byte[] serialize(SpecificRecord record) {
+    Objects.requireNonNull(record, "record");
+    try {
+      ByteArrayOutputStream output = new ByteArrayOutputStream();
+      var encoder = EncoderFactory.get().binaryEncoder(output, null);
+      new SpecificDatumWriter<SpecificRecord>(record.getSchema()).write(record, encoder);
+      encoder.flush();
+      return output.toByteArray();
+    } catch (IOException failure) {
+      throw new IllegalStateException(
+          "Could not encode Avro schema " + record.getSchema().getFullName(), failure);
+    }
+  }
+
+  public static <T extends SpecificRecord> T deserialize(byte[] payload, Class<T> type) {
+    Objects.requireNonNull(payload, "payload");
+    Objects.requireNonNull(type, "type");
+    try {
+      return new SpecificDatumReader<>(type)
+          .read(null, DecoderFactory.get().binaryDecoder(payload, null));
+    } catch (IOException failure) {
+      throw new IllegalStateException("Could not decode Avro type " + type.getName(), failure);
+    }
+  }
+}


## `config(kafka): configure an idempotent producer`

diff --git a/src/main/java/com/sportsbook/wallet/config/KafkaProducerConfig.java b/src/main/java/com/sportsbook/wallet/config/KafkaProducerConfig.java
new file mode 100644
index 0000000..f6b4cf2
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/config/KafkaProducerConfig.java
@@ -0,0 +1,45 @@
+package com.sportsbook.wallet.config;
+
+import java.time.Duration;
+import java.util.HashMap;
+import java.util.Map;
+import org.apache.kafka.clients.producer.ProducerConfig;
+import org.apache.kafka.common.serialization.ByteArraySerializer;
+import org.apache.kafka.common.serialization.StringSerializer;
+import org.springframework.boot.autoconfigure.kafka.KafkaProperties;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import org.springframework.kafka.core.DefaultKafkaProducerFactory;
+import org.springframework.kafka.core.KafkaTemplate;
+import org.springframework.kafka.core.ProducerFactory;
+
+@Configuration
+public class KafkaProducerConfig {
+
+  private static final int MAX_IN_FLIGHT = 5;
+  public static final Duration DELIVERY_TIMEOUT = Duration.ofSeconds(5L);
+  public static final Duration MAX_BLOCK_TIME = Duration.ofSeconds(5L);
+  private static final int REQUEST_TIMEOUT_MILLIS = 4_000;
+
+  @Bean
+  public ProducerFactory<String, byte[]> walletProducerFactory(KafkaProperties properties) {
+    Map<String, Object> configuration = new HashMap<>(properties.buildProducerProperties());
+    configuration.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
+    configuration.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, ByteArraySerializer.class);
+    configuration.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
+    configuration.put(ProducerConfig.ACKS_CONFIG, "all");
+    configuration.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, MAX_IN_FLIGHT);
+    configuration.put(
+        ProducerConfig.DELIVERY_TIMEOUT_MS_CONFIG, Math.toIntExact(DELIVERY_TIMEOUT.toMillis()));
+    configuration.put(ProducerConfig.REQUEST_TIMEOUT_MS_CONFIG, REQUEST_TIMEOUT_MILLIS);
+    configuration.put(
+        ProducerConfig.MAX_BLOCK_MS_CONFIG, Math.toIntExact(MAX_BLOCK_TIME.toMillis()));
+    return new DefaultKafkaProducerFactory<>(configuration);
+  }
+
+  @Bean
+  public KafkaTemplate<String, byte[]> walletKafkaTemplate(
+      ProducerFactory<String, byte[]> walletProducerFactory) {
+    return new KafkaTemplate<>(walletProducerFactory);
+  }
+}


## `test(outbox): attach canonical event identifiers to Kafka records`

diff --git a/src/test/java/com/sportsbook/wallet/outbox/KafkaOutboxDispatcherTest.java b/src/test/java/com/sportsbook/wallet/outbox/KafkaOutboxDispatcherTest.java
new file mode 100644
index 0000000..eb9df5e
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/outbox/KafkaOutboxDispatcherTest.java
@@ -0,0 +1,56 @@
+package com.sportsbook.wallet.outbox;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.ArgumentMatchers.any;
+import static org.mockito.Mockito.verify;
+import static org.mockito.Mockito.when;
+
+import java.nio.charset.StandardCharsets;
+import java.time.Instant;
+import java.util.UUID;
+import java.util.concurrent.CompletableFuture;
+import org.apache.kafka.clients.producer.ProducerRecord;
+import org.junit.jupiter.api.Test;
+import org.mockito.ArgumentCaptor;
+import org.springframework.kafka.core.KafkaTemplate;
+
+class KafkaOutboxDispatcherTest {
+
+  @Test
+  @SuppressWarnings("unchecked")
+  void sendsTheKeyPayloadAndOneCanonicalEventIdHeader() {
+    KafkaTemplate<String, byte[]> kafka = org.mockito.Mockito.mock(KafkaTemplate.class);
+    when(kafka.send(any(ProducerRecord.class))).thenReturn(CompletableFuture.completedFuture(null));
+    KafkaOutboxDispatcher dispatcher = new KafkaOutboxDispatcher(kafka);
+    LeasedOutboxMessage message = message();
+
+    dispatcher.dispatch(message).toCompletableFuture().join();
+
+    ArgumentCaptor<ProducerRecord<String, byte[]>> recordCaptor =
+        ArgumentCaptor.forClass(ProducerRecord.class);
+    verify(kafka).send(recordCaptor.capture());
+    ProducerRecord<String, byte[]> record = recordCaptor.getValue();
+    assertThat(record.topic()).isEqualTo(message.topic());
+    assertThat(record.key()).isEqualTo(message.partitionKey());
+    assertThat(record.value()).containsExactly(message.payload());
+    assertThat(record.headers().toArray()).hasSize(1);
+    assertThat(record.headers().lastHeader(KafkaOutboxDispatcher.EVENT_ID_HEADER).value())
+        .asString(StandardCharsets.US_ASCII)
+        .isEqualTo(message.lease().eventId().toString());
+  }
+
+  private LeasedOutboxMessage message() {
+    UUID eventId = UUID.fromString("0198ca71-8000-7000-8000-0000000000af");
+    Instant created = Instant.parse("2026-08-21T00:00:00Z");
+    return new LeasedOutboxMessage(
+        new OutboxLease(eventId, "worker-a", 1, created.plusSeconds(30)),
+        "wallet.debited.v1",
+        "bet-1",
+        "WalletDebited",
+        new byte[] {1, 2, 3},
+        1L,
+        false,
+        1,
+        created);
+  }
+}


## `feat(outbox): dispatch leased messages asynchronously`

diff --git a/src/main/java/com/sportsbook/wallet/outbox/KafkaOutboxDispatcher.java b/src/main/java/com/sportsbook/wallet/outbox/KafkaOutboxDispatcher.java
new file mode 100644
index 0000000..d7955ee
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/outbox/KafkaOutboxDispatcher.java
@@ -0,0 +1,35 @@
+package com.sportsbook.wallet.outbox;
+
+import java.nio.charset.StandardCharsets;
+import java.util.List;
+import java.util.concurrent.CompletionStage;
+import org.apache.kafka.clients.producer.ProducerRecord;
+import org.apache.kafka.common.header.Header;
+import org.apache.kafka.common.header.internals.RecordHeader;
+import org.springframework.kafka.core.KafkaTemplate;
+import org.springframework.stereotype.Component;
+
+@Component
+public class KafkaOutboxDispatcher implements OutboxDispatcher {
+
+  public static final String EVENT_ID_HEADER = "event-id";
+
+  private final KafkaTemplate<String, byte[]> kafka;
+
+  public KafkaOutboxDispatcher(KafkaTemplate<String, byte[]> kafka) {
+    this.kafka = kafka;
+  }
+
+  @Override
+  public CompletionStage<Void> dispatch(LeasedOutboxMessage message) {
+    List<Header> headers =
+        List.of(
+            new RecordHeader(
+                EVENT_ID_HEADER,
+                message.lease().eventId().toString().getBytes(StandardCharsets.US_ASCII)));
+    ProducerRecord<String, byte[]> record =
+        new ProducerRecord<>(
+            message.topic(), null, null, message.partitionKey(), message.payload(), headers);
+    return kafka.send(record).thenApply(result -> null);
+  }
+}
diff --git a/src/main/java/com/sportsbook/wallet/outbox/OutboxDispatcher.java b/src/main/java/com/sportsbook/wallet/outbox/OutboxDispatcher.java
new file mode 100644
index 0000000..9bb3a92
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/outbox/OutboxDispatcher.java
@@ -0,0 +1,9 @@
+package com.sportsbook.wallet.outbox;
+
+import java.util.concurrent.CompletionStage;
+
+@FunctionalInterface
+public interface OutboxDispatcher {
+
+  CompletionStage<Void> dispatch(LeasedOutboxMessage message);
+}


## `feat(outbox): fence publish and failure completions`

diff --git a/src/main/java/com/sportsbook/wallet/outbox/OutboxPublisher.java b/src/main/java/com/sportsbook/wallet/outbox/OutboxPublisher.java
new file mode 100644
index 0000000..300dc33
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/outbox/OutboxPublisher.java
@@ -0,0 +1,86 @@
+package com.sportsbook.wallet.outbox;
+
+import com.sportsbook.wallet.persistence.OutboxDeliveryRepository;
+import java.time.Duration;
+import java.util.List;
+import java.util.concurrent.Executor;
+import java.util.concurrent.Semaphore;
+import org.springframework.beans.factory.annotation.Qualifier;
+import org.springframework.beans.factory.annotation.Value;
+import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
+import org.springframework.scheduling.annotation.Scheduled;
+import org.springframework.stereotype.Component;
+
+@Component
+@ConditionalOnProperty(name = "wallet.outbox.scheduling-enabled", havingValue = "true")
+public class OutboxPublisher {
+
+  private final OutboxDeliveryRepository delivery;
+  private final OutboxDispatcher dispatcher;
+  private final OutboxRetryPolicy retryPolicy;
+  private final Executor completionExecutor;
+  private final String owner;
+  private final int batchSize;
+  private final Duration leaseDuration;
+  private final Semaphore inFlight;
+
+  public OutboxPublisher(
+      OutboxDeliveryRepository delivery,
+      OutboxDispatcher dispatcher,
+      OutboxRetryPolicy retryPolicy,
+      @Qualifier("applicationTaskExecutor") Executor completionExecutor,
+      @Value("${wallet.outbox.owner:${HOSTNAME:wallet-service}-${random.uuid}}") String owner,
+      @Value("${wallet.outbox.batch-size:20}") int batchSize,
+      @Value("${wallet.outbox.max-in-flight:100}") int maximumInFlight,
+      @Value("${wallet.outbox.lease-duration:PT30S}") Duration leaseDuration) {
+    if (batchSize < 1 || maximumInFlight < 1 || batchSize > maximumInFlight) {
+      throw new IllegalArgumentException("invalid outbox delivery limits");
+    }
+    this.delivery = delivery;
+    this.dispatcher = dispatcher;
+    this.retryPolicy = retryPolicy;
+    this.completionExecutor = completionExecutor;
+    this.owner = owner;
+    this.batchSize = batchSize;
+    this.leaseDuration = leaseDuration;
+    this.inFlight = new Semaphore(maximumInFlight);
+  }
+
+  @Scheduled(fixedDelayString = "${wallet.outbox.poll-interval:PT0.1S}")
+  public synchronized void poll() {
+    int capacity = Math.min(batchSize, inFlight.availablePermits());
+    if (capacity == 0) {
+      return;
+    }
+    List<LeasedOutboxMessage> messages = delivery.claim(owner, capacity, leaseDuration);
+    messages.forEach(this::dispatch);
+  }
+
+  private void dispatch(LeasedOutboxMessage message) {
+    if (!inFlight.tryAcquire()) {
+      throw new IllegalStateException("claimed beyond in-flight capacity");
+    }
+    try {
+      dispatcher
+          .dispatch(message)
+          .whenCompleteAsync((ignored, failure) -> complete(message, failure), completionExecutor);
+    } catch (RuntimeException failure) {
+      complete(message, failure);
+    }
+  }
+
+  private void complete(LeasedOutboxMessage message, Throwable failure) {
+    try {
+      if (failure == null) {
+        delivery.markPublished(message.lease());
+      } else {
+        delivery.releaseForRetry(
+            message.lease(),
+            retryPolicy.delayForAttempt(message.attemptCount()),
+            retryPolicy.describe(failure));
+      }
+    } finally {
+      inFlight.release();
+    }
+  }
+}


