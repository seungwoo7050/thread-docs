## `feat(repository): allocate committed stream positions`

diff --git a/src/main/java/com/sportsbook/wallet/persistence/OutboxStreamLock.java b/src/main/java/com/sportsbook/wallet/persistence/OutboxStreamLock.java
new file mode 100644
index 0000000..7672b4c
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/persistence/OutboxStreamLock.java
@@ -0,0 +1,55 @@
+package com.sportsbook.wallet.persistence;
+
+import java.util.Map;
+import java.util.Objects;
+import org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate;
+import org.springframework.stereotype.Component;
+import org.springframework.transaction.annotation.Propagation;
+import org.springframework.transaction.annotation.Transactional;
+
+/** Allocates positions while holding the stream row until the surrounding transaction commits. */
+@Component
+public class OutboxStreamLock {
+  private static final String CREATE_STREAM =
+      """
+      INSERT INTO outbox_stream(topic, partition_key, last_sequence)
+      VALUES (:topic, :partitionKey, 0)
+      ON CONFLICT (topic, partition_key) DO NOTHING
+      """;
+  private static final String NEXT_SEQUENCE =
+      """
+      UPDATE outbox_stream
+      SET last_sequence = last_sequence + 1
+      WHERE topic = :topic AND partition_key = :partitionKey
+      RETURNING last_sequence
+      """;
+
+  private final NamedParameterJdbcTemplate jdbc;
+
+  public OutboxStreamLock(NamedParameterJdbcTemplate jdbc) {
+    this.jdbc = Objects.requireNonNull(jdbc, "jdbc");
+  }
+
+  @Transactional(propagation = Propagation.MANDATORY)
+  public long nextSequence(String topic, String partitionKey) {
+    Map<String, ?> parameters =
+        Map.of(
+            "topic",
+            required(topic, "topic"),
+            "partitionKey",
+            required(partitionKey, "partitionKey"));
+    jdbc.update(CREATE_STREAM, parameters);
+    Long sequence = jdbc.queryForObject(NEXT_SEQUENCE, parameters, Long.class);
+    if (sequence == null || sequence < 1L) {
+      throw new IllegalStateException("Outbox stream did not allocate a positive sequence");
+    }
+    return sequence;
+  }
+
+  private static String required(String value, String name) {
+    if (value == null || value.isBlank()) {
+      throw new IllegalArgumentException(name + " must not be blank");
+    }
+    return value;
+  }
+}


## `feat(outbox): append one event per operation`

diff --git a/src/main/java/com/sportsbook/wallet/outbox/OutboxAppender.java b/src/main/java/com/sportsbook/wallet/outbox/OutboxAppender.java
new file mode 100644
index 0000000..656c366
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/outbox/OutboxAppender.java
@@ -0,0 +1,27 @@
+package com.sportsbook.wallet.outbox;
+
+import com.sportsbook.wallet.persistence.OutboxEventRepository;
+import com.sportsbook.wallet.persistence.OutboxStreamLock;
+import java.util.Objects;
+import org.springframework.stereotype.Component;
+import org.springframework.transaction.annotation.Propagation;
+import org.springframework.transaction.annotation.Transactional;
+
+/** Appends one immutable event at a stream position owned by the caller's transaction. */
+@Component
+public class OutboxAppender {
+  private final OutboxStreamLock streams;
+  private final OutboxEventRepository events;
+
+  public OutboxAppender(OutboxStreamLock streams, OutboxEventRepository events) {
+    this.streams = Objects.requireNonNull(streams, "streams");
+    this.events = Objects.requireNonNull(events, "events");
+  }
+
+  @Transactional(propagation = Propagation.MANDATORY)
+  public OutboxEvent append(PendingOutboxMessage message) {
+    Objects.requireNonNull(message, "message");
+    long sequence = streams.nextSequence(message.topic(), message.partitionKey());
+    return events.save(OutboxEvent.pending(message, sequence));
+  }
+}


## `test(outbox): preserve stream sequence ordering`

diff --git a/src/test/java/com/sportsbook/wallet/persistence/OutboxDeliveryRepositoryTest.java b/src/test/java/com/sportsbook/wallet/persistence/OutboxDeliveryRepositoryTest.java
index 470d41d..adaf2de 100644
--- a/src/test/java/com/sportsbook/wallet/persistence/OutboxDeliveryRepositoryTest.java
+++ b/src/test/java/com/sportsbook/wallet/persistence/OutboxDeliveryRepositoryTest.java
@@ -69,4 +69,21 @@ class OutboxDeliveryRepositoryTest extends OutboxDeliveryRepositoryFixture {
                     created.plusMillis(1)))
         .isInstanceOf(org.springframework.dao.DataIntegrityViolationException.class);
   }
+
+  @Test
+  void preservesStreamSequenceOrdering() {
+    Instant createdLater = Instant.parse("2026-08-21T00:00:01Z");
+    Instant createdEarlier = createdLater.minusSeconds(1L);
+    persist("operation-a", "key-a", "dedup-a1", createdLater);
+    persist("operation-a2", "key-a", "dedup-a2", createdEarlier);
+
+    var head = delivery.claim("worker-a", 10, Duration.ofSeconds(30));
+    var blockedSuccessor = delivery.claim("worker-b", 10, Duration.ofSeconds(30));
+
+    assertThat(head)
+        .singleElement()
+        .satisfies(message -> assertThat(message.streamSequence()).isOne());
+    assertThat(head.get(0).createdAt()).isEqualTo(createdLater);
+    assertThat(blockedSuccessor).isEmpty();
+  }
 }


## `test(repository): reuse rolled-back stream positions`

diff --git a/src/test/java/com/sportsbook/wallet/persistence/OutboxStreamLockTest.java b/src/test/java/com/sportsbook/wallet/persistence/OutboxStreamLockTest.java
index 059afe0..4216ad2 100644
--- a/src/test/java/com/sportsbook/wallet/persistence/OutboxStreamLockTest.java
+++ b/src/test/java/com/sportsbook/wallet/persistence/OutboxStreamLockTest.java
@@ -1,6 +1,7 @@
 package com.sportsbook.wallet.persistence;
 
 import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
 
 import java.util.concurrent.CompletableFuture;
 import java.util.concurrent.CountDownLatch;
@@ -69,6 +70,25 @@ class OutboxStreamLockTest {
     assertThat(second.get()).isEqualTo(2L);
   }
 
+  @Test
+  void reusesAPositionWhoseTransactionRolledBack() {
+    TransactionTemplate transaction = new TransactionTemplate(transactions);
+    RuntimeException rollback = new RuntimeException("rollback stream append");
+
+    assertThatThrownBy(
+            () ->
+                transaction.execute(
+                    ignored -> {
+                      assertThat(streams.nextSequence("wallet.credited.v1", "user-rollback"))
+                          .isEqualTo(1L);
+                      throw rollback;
+                    }))
+        .isSameAs(rollback);
+    Long reused =
+        transaction.execute(ignored -> streams.nextSequence("wallet.credited.v1", "user-rollback"));
+    assertThat(reused).isEqualTo(1L);
+  }
+
   private static void await(CountDownLatch latch) {
     try {
       latch.await();


## `test(outbox): roll back duplicate operation appends`

diff --git a/src/test/java/com/sportsbook/wallet/outbox/OutboxAppenderTest.java b/src/test/java/com/sportsbook/wallet/outbox/OutboxAppenderTest.java
index 75ebcce..cfcee66 100644
--- a/src/test/java/com/sportsbook/wallet/outbox/OutboxAppenderTest.java
+++ b/src/test/java/com/sportsbook/wallet/outbox/OutboxAppenderTest.java
@@ -17,6 +17,7 @@ import org.junit.jupiter.api.Test;
 import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
 import org.springframework.context.annotation.Import;
+import org.springframework.dao.DataIntegrityViolationException;
 import org.springframework.test.context.DynamicPropertyRegistry;
 import org.springframework.test.context.DynamicPropertySource;
 import org.springframework.transaction.IllegalTransactionStateException;
@@ -70,11 +71,47 @@ class OutboxAppenderTest {
         .containsExactly(1L, 2L);
   }
 
+  @Test
+  void rollsBackTheStreamPositionForADuplicateOperation() {
+    TransactionTemplate transaction = new TransactionTemplate(transactions);
+    PendingOutboxMessage first = messageOn("append:duplicate", "bet-duplicate-1", "user-duplicate");
+    transaction.executeWithoutResult(
+        ignored -> {
+          appender.append(first);
+          operations.save(operation(first.operationKey()));
+        });
+
+    assertThatThrownBy(
+            () ->
+                transaction.executeWithoutResult(
+                    ignored ->
+                        appender.append(
+                            messageOn("append:duplicate", "bet-duplicate-2", "user-duplicate"))))
+        .isInstanceOf(DataIntegrityViolationException.class);
+    PendingOutboxMessage next =
+        messageOn("append:after-rollback", "bet-after-rollback", "user-duplicate");
+    transaction.executeWithoutResult(
+        ignored -> {
+          appender.append(next);
+          operations.save(operation(next.operationKey()));
+        });
+
+    assertThat(events.findAllByOrderByTopicAscPartitionKeyAscStreamSequenceAsc())
+        .filteredOn(event -> event.partitionKey().equals("user-duplicate"))
+        .extracting(OutboxEvent::streamSequence)
+        .contains(1L, 2L);
+  }
+
   private static PendingOutboxMessage message(String operationKey, String deduplicationKey) {
+    return messageOn(operationKey, deduplicationKey, USER_ID.toString());
+  }
+
+  private static PendingOutboxMessage messageOn(
+      String operationKey, String deduplicationKey, String partitionKey) {
     return PendingOutboxMessage.create(
         operationKey,
         "wallet.debited.v1",
-        USER_ID.toString(),
+        partitionKey,
         "WalletDebited",
         deduplicationKey,
         new byte[] {1},


## `test(outbox): wire transactional appenders in wallet slices`

diff --git a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
index 85b957b..4a3800f 100644
--- a/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
+++ b/src/test/java/com/sportsbook/wallet/persistence/WalletPersistenceTest.java
@@ -26,6 +26,7 @@ import com.sportsbook.wallet.domain.error.IdempotencyConflictException;
 import com.sportsbook.wallet.domain.error.WalletBusyException;
 import com.sportsbook.wallet.domain.error.WalletRejectedException;
 import com.sportsbook.wallet.integrity.OperationCommitted;
+import com.sportsbook.wallet.outbox.OutboxAppender;
 import com.sportsbook.wallet.service.IdempotencyCache;
 import com.sportsbook.wallet.service.WalletOperationExecutor;
 import com.sportsbook.wallet.service.WalletOperationResult;
@@ -67,6 +68,8 @@ import org.testcontainers.junit.jupiter.Testcontainers;
 @Transactional(propagation = Propagation.NOT_SUPPORTED)
 @Import({
   IdempotencyKeyLock.class,
+  OutboxAppender.class,
+  OutboxStreamLock.class,
   WalletInfrastructureConfig.class,
   WalletOperationExecutor.class,
   WalletOutcomeResolver.class,
