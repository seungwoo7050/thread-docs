## `test(outbox): verify lease ownership and fencing`

diff --git a/src/test/java/com/sportsbook/wallet/outbox/OutboxLeaseTest.java b/src/test/java/com/sportsbook/wallet/outbox/OutboxLeaseTest.java
new file mode 100644
index 0000000..4d90fa9
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/outbox/OutboxLeaseTest.java
@@ -0,0 +1,53 @@
+package com.sportsbook.wallet.outbox;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatIllegalArgumentException;
+
+import java.time.Instant;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class OutboxLeaseTest {
+
+  private static final UUID EVENT_ID = UUID.fromString("0198ca71-8000-7000-8000-000000000001");
+  private static final Instant UNTIL = Instant.parse("2026-08-21T00:00:30Z");
+
+  @Test
+  void ownershipRequiresTheSameWorkerAndFencingVersion() {
+    OutboxLease lease = new OutboxLease(EVENT_ID, "worker-a", 3, UNTIL);
+
+    assertThat(lease.isOwnedBy("worker-a", 3)).isTrue();
+    assertThat(lease.isOwnedBy("worker-b", 3)).isFalse();
+    assertThat(lease.isOwnedBy("worker-a", 2)).isFalse();
+  }
+
+  @Test
+  void rejectsAnUnfencedLease() {
+    assertThatIllegalArgumentException()
+        .isThrownBy(() -> new OutboxLease(EVENT_ID, "worker-a", 0, UNTIL));
+  }
+
+  @Test
+  void leasedMessagesProtectTheirPayload() {
+    byte[] payload = {1, 2};
+    LeasedOutboxMessage message =
+        new LeasedOutboxMessage(
+            new OutboxLease(EVENT_ID, "worker-a", 1, UNTIL),
+            "wallet.debited.v1",
+            "bet-1",
+            "WalletDebited",
+            payload,
+            5L,
+            true,
+            1,
+            Instant.parse("2026-08-21T00:00:00Z"));
+
+    payload[0] = 9;
+    byte[] exposed = message.payload();
+    exposed[1] = 9;
+
+    assertThat(message.payload()).containsExactly(1, 2);
+    assertThat(message.streamSequence()).isEqualTo(5L);
+    assertThat(message.leaseTakeover()).isTrue();
+  }
+}


## `feat(outbox): define capped retry policy`

diff --git a/src/main/java/com/sportsbook/wallet/outbox/OutboxRetryPolicy.java b/src/main/java/com/sportsbook/wallet/outbox/OutboxRetryPolicy.java
new file mode 100644
index 0000000..5886ec5
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/outbox/OutboxRetryPolicy.java
@@ -0,0 +1,56 @@
+package com.sportsbook.wallet.outbox;
+
+import java.time.Duration;
+import java.util.Objects;
+import org.springframework.beans.factory.annotation.Value;
+import org.springframework.stereotype.Component;
+
+@Component
+public class OutboxRetryPolicy {
+
+  private static final int MAX_ERROR_LENGTH = 1024;
+  private static final int MAX_EXPONENT = 62;
+
+  private final Duration baseDelay;
+  private final Duration maximumDelay;
+
+  public OutboxRetryPolicy(
+      @Value("${wallet.outbox.retry-base:PT1S}") Duration baseDelay,
+      @Value("${wallet.outbox.retry-cap:PT60S}") Duration maximumDelay) {
+    this.baseDelay = positive(baseDelay, "baseDelay");
+    this.maximumDelay = positive(maximumDelay, "maximumDelay");
+    if (baseDelay.compareTo(maximumDelay) > 0) {
+      throw new IllegalArgumentException("baseDelay must not exceed maximumDelay");
+    }
+  }
+
+  public Duration delayForAttempt(int attemptCount) {
+    if (attemptCount < 1) {
+      throw new IllegalArgumentException("attemptCount must be positive");
+    }
+    long multiplier = 1L << Math.min(attemptCount - 1, MAX_EXPONENT);
+    try {
+      Duration delay = baseDelay.multipliedBy(multiplier);
+      return delay.compareTo(maximumDelay) < 0 ? delay : maximumDelay;
+    } catch (ArithmeticException overflow) {
+      return maximumDelay;
+    }
+  }
+
+  public String describe(Throwable failure) {
+    Objects.requireNonNull(failure, "failure");
+    String message = failure.getMessage();
+    String description =
+        failure.getClass().getSimpleName()
+            + (message == null ? "" : ": " + message.replaceAll("[\\r\\n]+", " "));
+    return description.substring(0, Math.min(description.length(), MAX_ERROR_LENGTH));
+  }
+
+  private static Duration positive(Duration duration, String name) {
+    Objects.requireNonNull(duration, name);
+    if (duration.toMillis() < 1L) {
+      throw new IllegalArgumentException(name + " must be positive");
+    }
+    return duration;
+  }
+}


## `feat(outbox): schedule indefinitely retried bounded backoff`

diff --git a/src/main/java/com/sportsbook/wallet/persistence/OutboxDeliveryRepository.java b/src/main/java/com/sportsbook/wallet/persistence/OutboxDeliveryRepository.java
index cd86978..37faf54 100644
--- a/src/main/java/com/sportsbook/wallet/persistence/OutboxDeliveryRepository.java
+++ b/src/main/java/com/sportsbook/wallet/persistence/OutboxDeliveryRepository.java
@@ -16,6 +16,30 @@ import org.springframework.transaction.annotation.Transactional;
 @Repository
 public class OutboxDeliveryRepository {
 
+  private static final int MAX_ERROR_LENGTH = 1024;
+
+  private static final String PUBLISH_SQL =
+      """
+      WITH db_clock AS MATERIALIZED (SELECT clock_timestamp() AS now)
+      UPDATE outbox_event e
+      SET published_at = c.now,
+          lease_owner = NULL, lease_until = NULL, last_error = NULL
+      FROM db_clock c
+      WHERE e.event_id = :eventId AND e.lease_owner = :owner
+        AND e.lease_version = :version AND e.published_at IS NULL
+      """;
+
+  private static final String RETRY_SQL =
+      """
+      WITH db_clock AS MATERIALIZED (SELECT clock_timestamp() AS now)
+      UPDATE outbox_event e
+      SET available_at = c.now + CAST(:delayMillis AS bigint) * interval '1 millisecond',
+          lease_owner = NULL, lease_until = NULL, last_error = :error
+      FROM db_clock c
+      WHERE e.event_id = :eventId AND e.lease_owner = :owner
+        AND e.lease_version = :version AND e.published_at IS NULL
+      """;
+
   private static final String CLAIM_SQL =
       """
       WITH db_clock AS MATERIALIZED (
@@ -77,6 +101,26 @@ public class OutboxDeliveryRepository {
     return List.copyOf(messages);
   }
 
+  @Transactional(propagation = Propagation.REQUIRES_NEW, timeout = 5)
+  public boolean markPublished(OutboxLease lease) {
+    return jdbc.update(PUBLISH_SQL, leaseParameters(lease)) == 1;
+  }
+
+  @Transactional(propagation = Propagation.REQUIRES_NEW, timeout = 5)
+  public boolean releaseForRetry(OutboxLease lease, Duration delay, String error) {
+    if (delay.isNegative() || error == null || error.length() > MAX_ERROR_LENGTH) {
+      throw new IllegalArgumentException("invalid retry completion");
+    }
+    Map<String, Object> parameters = new java.util.HashMap<>(leaseParameters(lease));
+    parameters.put("delayMillis", delay.toMillis());
+    parameters.put("error", error);
+    return jdbc.update(RETRY_SQL, parameters) == 1;
+  }
+
+  private Map<String, Object> leaseParameters(OutboxLease lease) {
+    return Map.of("eventId", lease.eventId(), "owner", lease.owner(), "version", lease.version());
+  }
+
   private LeasedOutboxMessage map(ResultSet resultSet, int rowNumber) throws SQLException {
     OutboxLease lease =
         new OutboxLease(


## `config(outbox): validate delivery lease safety`

diff --git a/src/main/java/com/sportsbook/wallet/outbox/OutboxPublisher.java b/src/main/java/com/sportsbook/wallet/outbox/OutboxPublisher.java
index 300dc33..0c132ef 100644
--- a/src/main/java/com/sportsbook/wallet/outbox/OutboxPublisher.java
+++ b/src/main/java/com/sportsbook/wallet/outbox/OutboxPublisher.java
@@ -1,10 +1,12 @@
 package com.sportsbook.wallet.outbox;
 
+import com.sportsbook.wallet.config.KafkaProducerConfig;
 import com.sportsbook.wallet.persistence.OutboxDeliveryRepository;
 import java.time.Duration;
 import java.util.List;
 import java.util.concurrent.Executor;
 import java.util.concurrent.Semaphore;
+import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.beans.factory.annotation.Qualifier;
 import org.springframework.beans.factory.annotation.Value;
 import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
@@ -15,6 +17,9 @@ import org.springframework.stereotype.Component;
 @ConditionalOnProperty(name = "wallet.outbox.scheduling-enabled", havingValue = "true")
 public class OutboxPublisher {
 
+  private static final Duration COMPLETION_MARGIN = Duration.ofSeconds(5);
+  private static final int MAXIMUM_OWNER_LENGTH = 128;
+
   private final OutboxDeliveryRepository delivery;
   private final OutboxDispatcher dispatcher;
   private final OutboxRetryPolicy retryPolicy;
@@ -36,6 +41,16 @@ public class OutboxPublisher {
     if (batchSize < 1 || maximumInFlight < 1 || batchSize > maximumInFlight) {
       throw new IllegalArgumentException("invalid outbox delivery limits");
     }
+    if (owner == null || owner.isBlank() || owner.length() > MAXIMUM_OWNER_LENGTH) {
+      throw new IllegalArgumentException("invalid outbox owner");
+    }
+    Duration minimumLease =
+        KafkaProducerConfig.MAX_BLOCK_TIME
+            .plus(KafkaProducerConfig.DELIVERY_TIMEOUT)
+            .plus(COMPLETION_MARGIN);
+    if (leaseDuration == null || leaseDuration.compareTo(minimumLease) <= 0) {
+      throw new IllegalArgumentException("outbox lease must exceed delivery timeout and margin");
+    }
     this.delivery = delivery;
     this.dispatcher = dispatcher;
     this.retryPolicy = retryPolicy;
@@ -46,7 +61,14 @@ public class OutboxPublisher {
     this.inFlight = new Semaphore(maximumInFlight);
   }
 
-  @Scheduled(fixedDelayString = "${wallet.outbox.poll-interval:PT0.1S}")
+  @Autowired
+  void validatePollInterval(@Value("${wallet.outbox.poll-interval:PT1S}") Duration pollInterval) {
+    if (pollInterval == null || pollInterval.toMillis() < 1L) {
+      throw new IllegalArgumentException("outbox poll interval must be at least one millisecond");
+    }
+  }
+
+  @Scheduled(fixedDelayString = "${wallet.outbox.poll-interval:PT1S}")
   public synchronized void poll() {
     int capacity = Math.min(batchSize, inFlight.availablePermits());
     if (capacity == 0) {


## `test(outbox): reject unsafe delivery timing`

diff --git a/src/test/java/com/sportsbook/wallet/outbox/OutboxPublisherConfigurationTest.java b/src/test/java/com/sportsbook/wallet/outbox/OutboxPublisherConfigurationTest.java
new file mode 100644
index 0000000..e798ec9
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/outbox/OutboxPublisherConfigurationTest.java
@@ -0,0 +1,45 @@
+package com.sportsbook.wallet.outbox;
+
+import static org.assertj.core.api.Assertions.assertThatCode;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import java.time.Duration;
+import org.junit.jupiter.api.Test;
+
+class OutboxPublisherConfigurationTest {
+
+  @Test
+  void requiresALeaseLongerThanKafkaBlockingDeliveryAndCompletion() {
+    assertThatThrownBy(
+            () -> publisher("worker-a", 1, 1, Duration.ofSeconds(15), Duration.ofMillis(1)))
+        .isInstanceOf(IllegalArgumentException.class);
+    assertThatCode(
+            () -> publisher("worker-a", 1, 1, Duration.ofMillis(15_001), Duration.ofMillis(1)))
+        .doesNotThrowAnyException();
+    assertThatCode(() -> publisher("worker-a", 1, 1, Duration.ofSeconds(30), Duration.ofSeconds(1)))
+        .doesNotThrowAnyException();
+  }
+
+  @Test
+  void rejectsUnusablePollingIdentityAndCapacity() {
+    assertThatThrownBy(
+            () -> publisher("worker-a", 1, 1, Duration.ofSeconds(30), Duration.ofNanos(1)))
+        .isInstanceOf(IllegalArgumentException.class);
+    assertThatThrownBy(() -> publisher(" ", 1, 1, Duration.ofSeconds(30), Duration.ofMillis(1)))
+        .isInstanceOf(IllegalArgumentException.class);
+    assertThatThrownBy(
+            () -> publisher("x".repeat(129), 1, 1, Duration.ofSeconds(30), Duration.ofMillis(1)))
+        .isInstanceOf(IllegalArgumentException.class);
+    assertThatThrownBy(
+            () -> publisher("worker-a", 2, 1, Duration.ofSeconds(30), Duration.ofMillis(1)))
+        .isInstanceOf(IllegalArgumentException.class);
+  }
+
+  private OutboxPublisher publisher(
+      String owner, int batchSize, int inFlight, Duration lease, Duration poll) {
+    OutboxPublisher publisher =
+        new OutboxPublisher(null, null, null, Runnable::run, owner, batchSize, inFlight, lease);
+    publisher.validatePollInterval(poll);
+    return publisher;
+  }
+}


## `config(outbox): activate safe delivery polling`

diff --git a/src/main/java/com/sportsbook/wallet/WalletServiceApplication.java b/src/main/java/com/sportsbook/wallet/WalletServiceApplication.java
index cde9531..f20f818 100644
--- a/src/main/java/com/sportsbook/wallet/WalletServiceApplication.java
+++ b/src/main/java/com/sportsbook/wallet/WalletServiceApplication.java
@@ -2,8 +2,10 @@ package com.sportsbook.wallet;
 
 import org.springframework.boot.SpringApplication;
 import org.springframework.boot.autoconfigure.SpringBootApplication;
+import org.springframework.scheduling.annotation.EnableScheduling;
 
 @SpringBootApplication
+@EnableScheduling
 @SuppressWarnings("checkstyle:HideUtilityClassConstructor")
 public class WalletServiceApplication {
 
diff --git a/src/main/resources/application.yml b/src/main/resources/application.yml
index 13fda02..8fca06b 100644
--- a/src/main/resources/application.yml
+++ b/src/main/resources/application.yml
@@ -68,3 +68,7 @@ logging:
     root: INFO
     com.sportsbook.wallet: INFO
     org.hibernate.SQL: WARN
+
+wallet:
+  outbox:
+    scheduling-enabled: ${WALLET_OUTBOX_ENABLED:false}
diff --git a/src/test/resources/application-test.yml b/src/test/resources/application-test.yml
index b6d2cd9..6214d73 100644
--- a/src/test/resources/application-test.yml
+++ b/src/test/resources/application-test.yml
@@ -17,3 +17,7 @@ management:
 logging:
   level:
     org.testcontainers: INFO
+
+wallet:
+  outbox:
+    scheduling-enabled: false


## `test(outbox): converge two publishers after worker loss`

diff --git a/src/test/java/com/sportsbook/wallet/persistence/OutboxDeliveryRepositoryTest.java b/src/test/java/com/sportsbook/wallet/persistence/OutboxDeliveryRepositoryTest.java
index cce2c91..e24bc30 100644
--- a/src/test/java/com/sportsbook/wallet/persistence/OutboxDeliveryRepositoryTest.java
+++ b/src/test/java/com/sportsbook/wallet/persistence/OutboxDeliveryRepositoryTest.java
@@ -4,8 +4,11 @@ import static org.assertj.core.api.Assertions.assertThat;
 import static org.assertj.core.api.Assertions.assertThatThrownBy;
 
 import com.sportsbook.wallet.outbox.OutboxAppender;
+import com.sportsbook.wallet.outbox.OutboxPublisher;
+import com.sportsbook.wallet.outbox.OutboxRetryPolicy;
 import java.time.Duration;
 import java.time.Instant;
+import java.util.concurrent.CompletableFuture;
 import org.junit.jupiter.api.Test;
 import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
@@ -135,4 +138,56 @@ class OutboxDeliveryRepositoryTest extends OutboxDeliveryRepositoryFixture {
     assertThat(delivery.claim("worker-b", 1, Duration.ofSeconds(30)).get(0).streamSequence())
         .isEqualTo(2L);
   }
+
+  @Test
+  void twoPublishersConvergeAfterTheFirstWorkerIsLost() {
+    Instant created = Instant.parse("2026-08-21T00:00:00Z");
+    persist("operation-a", "key-a", "dedup-a1", created);
+    persist("operation-a2", "key-a", "dedup-a2", created.plusMillis(1));
+    CompletableFuture<Void> abandonedSend = new CompletableFuture<>();
+    java.util.List<com.sportsbook.wallet.outbox.LeasedOutboxMessage> recovered =
+        new java.util.concurrent.CopyOnWriteArrayList<>();
+    OutboxRetryPolicy policy = new OutboxRetryPolicy(Duration.ofMillis(1), Duration.ofSeconds(1));
+    OutboxPublisher workerA =
+        new OutboxPublisher(
+            delivery,
+            ignored -> abandonedSend,
+            policy,
+            Runnable::run,
+            "worker-a",
+            1,
+            1,
+            Duration.ofSeconds(30));
+    OutboxPublisher workerB =
+        new OutboxPublisher(
+            delivery,
+            message -> {
+              recovered.add(message);
+              return CompletableFuture.completedFuture(null);
+            },
+            policy,
+            Runnable::run,
+            "worker-b",
+            1,
+            1,
+            Duration.ofSeconds(30));
+
+    workerA.poll();
+    jdbc.update(
+        "UPDATE outbox_event SET lease_until=clock_timestamp()-interval '1 second' WHERE lease_owner='worker-a'");
+    workerB.poll();
+    abandonedSend.complete(null);
+    workerB.poll();
+
+    assertThat(
+            jdbc.queryForObject(
+                "SELECT count(*) FROM outbox_event WHERE published_at IS NOT NULL", Integer.class))
+        .isEqualTo(2);
+    assertThat(jdbc.queryForObject("SELECT sum(attempt_count) FROM outbox_event", Integer.class))
+        .isEqualTo(3);
+    assertThat(recovered)
+        .extracting(message -> message.leaseTakeover())
+        .containsExactly(true, false);
+    assertThat(recovered).extracting(message -> message.streamSequence()).containsExactly(1L, 2L);
+  }
 }


## `test(outbox): publish fenced events through Kafka`

diff --git a/src/test/java/com/sportsbook/wallet/persistence/KafkaOutboxDeliveryTest.java b/src/test/java/com/sportsbook/wallet/persistence/KafkaOutboxDeliveryTest.java
new file mode 100644
index 0000000..8f4e391
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/persistence/KafkaOutboxDeliveryTest.java
@@ -0,0 +1,100 @@
+package com.sportsbook.wallet.persistence;
+
+import static java.nio.charset.StandardCharsets.US_ASCII;
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.awaitility.Awaitility.await;
+
+import com.sportsbook.wallet.config.KafkaProducerConfig;
+import com.sportsbook.wallet.outbox.KafkaOutboxDispatcher;
+import com.sportsbook.wallet.outbox.OutboxAppender;
+import com.sportsbook.wallet.outbox.OutboxPublisher;
+import com.sportsbook.wallet.outbox.OutboxRetryPolicy;
+import java.time.Duration;
+import java.time.Instant;
+import java.util.List;
+import java.util.stream.StreamSupport;
+import org.apache.kafka.common.serialization.ByteArrayDeserializer;
+import org.apache.kafka.common.serialization.StringDeserializer;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.autoconfigure.kafka.KafkaProperties;
+import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
+import org.springframework.context.annotation.Import;
+import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
+import org.springframework.kafka.test.EmbeddedKafkaBroker;
+import org.springframework.kafka.test.context.EmbeddedKafka;
+import org.springframework.kafka.test.utils.KafkaTestUtils;
+import org.springframework.transaction.annotation.Propagation;
+import org.springframework.transaction.annotation.Transactional;
+import org.testcontainers.junit.jupiter.Testcontainers;
+
+@DataJpaTest(properties = "spring.test.database.replace=NONE")
+@Testcontainers
+@EmbeddedKafka(partitions = 1, topics = KafkaOutboxDeliveryTest.TOPIC)
+@Transactional(propagation = Propagation.NOT_SUPPORTED)
+@Import({OutboxAppender.class, OutboxDeliveryRepository.class, OutboxStreamLock.class})
+class KafkaOutboxDeliveryTest extends OutboxDeliveryRepositoryFixture {
+  static final String TOPIC = "wallet.debited.v1";
+
+  @Autowired EmbeddedKafkaBroker broker;
+  @Autowired OutboxDeliveryRepository delivery;
+
+  @Test
+  void publishesOnlyAfterBrokerAckWithOneCanonicalEventHeader() {
+    persist("operation-kafka", "user-1", "dedup-kafka", Instant.EPOCH);
+    var event = events.findAll().get(0);
+    KafkaProperties properties = new KafkaProperties();
+    properties.setBootstrapServers(List.of(broker.getBrokersAsString()));
+    KafkaProducerConfig configuration = new KafkaProducerConfig();
+    var producerFactory = configuration.walletProducerFactory(properties);
+    var template = configuration.walletKafkaTemplate(producerFactory);
+    var consumerFactory =
+        new DefaultKafkaConsumerFactory<>(
+            KafkaTestUtils.consumerProps("wallet-outbox-test", "true", broker),
+            new StringDeserializer(),
+            new ByteArrayDeserializer());
+
+    try (var consumer = consumerFactory.createConsumer()) {
+      broker.consumeFromAnEmbeddedTopic(consumer, TOPIC);
+      OutboxPublisher publisher =
+          new OutboxPublisher(
+              delivery,
+              new KafkaOutboxDispatcher(template),
+              new OutboxRetryPolicy(Duration.ofMillis(10), Duration.ofSeconds(1)),
+              Runnable::run,
+              "embedded-worker",
+              1,
+              1,
+              Duration.ofSeconds(30));
+
+      assertThat(isUnpublished(event.eventId())).isTrue();
+      publisher.poll();
+      var record = KafkaTestUtils.getSingleRecord(consumer, TOPIC, Duration.ofSeconds(10));
+      var headers =
+          StreamSupport.stream(
+                  record.headers().headers(KafkaOutboxDispatcher.EVENT_ID_HEADER).spliterator(),
+                  false)
+              .toList();
+
+      assertThat(record.key()).isEqualTo(event.partitionKey());
+      assertThat(record.value()).containsExactly(event.payload());
+      assertThat(headers)
+          .singleElement()
+          .extracting(header -> new String(header.value(), US_ASCII))
+          .isEqualTo(event.eventId().toString());
+      await()
+          .atMost(Duration.ofSeconds(10))
+          .untilAsserted(() -> assertThat(isUnpublished(event.eventId())).isFalse());
+    } finally {
+      producerFactory.reset();
+    }
+  }
+
+  private boolean isUnpublished(java.util.UUID eventId) {
+    return Boolean.TRUE.equals(
+        jdbc.queryForObject(
+            "SELECT published_at IS NULL FROM outbox_event WHERE event_id=?",
+            Boolean.class,
+            eventId));
+  }
+}


