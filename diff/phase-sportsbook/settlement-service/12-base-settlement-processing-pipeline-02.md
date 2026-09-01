## `feat(settlement): claim pending bets atomically`

diff --git a/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java b/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java
new file mode 100644
index 0000000..02df647
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java
@@ -0,0 +1,55 @@
+package com.sportsbook.settlement.execution;
+
+import static com.sportsbook.settlement.persistence.JdbcTimestamps.required;
+
+import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.stereotype.Repository;
+
+@Repository
+public class SettlementAttemptRepository {
+
+  private static final String CLAIM_SQL =
+      """
+      INSERT INTO settlement_attempt (
+          bet_id, action, event_id, result, void_reason,
+          committed_amount, payout_amount, locked_release_amount,
+          locked_forfeit_amount, house_profit_amount, currency,
+          lease_token, lease_until, attempt_count, last_error, created_at, updated_at)
+      SELECT ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?
+      FROM bet
+      WHERE bet_id = ? AND status = 'PENDING'
+      ON CONFLICT (bet_id) DO NOTHING
+      """;
+
+  private final JdbcTemplate jdbc;
+
+  public SettlementAttemptRepository(JdbcTemplate jdbc) {
+    this.jdbc = jdbc;
+  }
+
+  public boolean claimPending(SettlementAttempt attempt) {
+    SettlementMoneyPlan money = attempt.money();
+    int inserted =
+        jdbc.update(
+            CLAIM_SQL,
+            attempt.betId(),
+            attempt.action().name(),
+            attempt.eventId(),
+            attempt.result() == null ? null : attempt.result().name(),
+            attempt.voidReason(),
+            money.committed().amount(),
+            money.payout().amount(),
+            money.lockedRelease().amount(),
+            money.lockedForfeit().amount(),
+            money.houseProfit().amount(),
+            money.committed().currency().name(),
+            attempt.lease().token(),
+            required(attempt.lease().until()),
+            attempt.attemptCount(),
+            attempt.lastError(),
+            required(attempt.createdAt()),
+            required(attempt.updatedAt()),
+            attempt.betId());
+    return inserted == 1;
+  }
+}


## `feat(settlement): fan out independent executions`

diff --git a/src/main/java/com/sportsbook/settlement/execution/SettlementExecutionRunner.java b/src/main/java/com/sportsbook/settlement/execution/SettlementExecutionRunner.java
new file mode 100644
index 0000000..279e0dd
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/execution/SettlementExecutionRunner.java
@@ -0,0 +1,61 @@
+package com.sportsbook.settlement.execution;
+
+import java.time.Clock;
+import java.time.Instant;
+import java.util.List;
+import org.springframework.stereotype.Component;
+
+@Component
+public class SettlementExecutionRunner {
+
+  private final SettlementWalletExecutor wallet;
+  private final SettlementFinalizer finalizer;
+  private final SettlementAttemptRepository attempts;
+  private final Clock clock;
+
+  public SettlementExecutionRunner(
+      SettlementWalletExecutor wallet,
+      SettlementFinalizer finalizer,
+      SettlementAttemptRepository attempts,
+      Clock clock) {
+    this.wallet = wallet;
+    this.finalizer = finalizer;
+    this.attempts = attempts;
+    this.clock = clock;
+  }
+
+  public void execute(SettlementExecution execution) {
+    SettlementAttempt attempt = execution.attempt();
+    try {
+      wallet.releaseLocked(attempt, execution.userId());
+      wallet.forfeitLocked(attempt, execution.userId());
+      wallet.payHouseProfit(attempt, execution.userId());
+      Instant now = clock.instant();
+      boolean finalized =
+          attempt.action() == SettlementAttempt.Action.SETTLE
+              ? finalizer.settle(attempt, now)
+              : finalizer.voidBet(attempt, now);
+      if (!finalized) {
+        throw new IllegalStateException("Settlement lease was lost before finalization");
+      }
+    } catch (RuntimeException failure) {
+      attempts.releaseForRecovery(attempt, failure, clock.instant());
+      throw failure;
+    }
+  }
+
+  public BatchResult fanOut(List<SettlementExecution> executions) {
+    int succeeded = 0;
+    for (SettlementExecution execution : executions) {
+      try {
+        execute(execution);
+        succeeded++;
+      } catch (RuntimeException ignored) {
+        // Every failure was durably released above; other independent bets must continue.
+      }
+    }
+    return new BatchResult(succeeded, executions.size() - succeeded);
+  }
+
+  public record BatchResult(int succeeded, int failed) {}
+}


## `feat(settlement): finalize resolved bets atomically`

diff --git a/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java b/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java
index 02df647..467bf15 100644
--- a/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java
+++ b/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java
@@ -52,4 +52,13 @@ public class SettlementAttemptRepository {
             attempt.betId());
     return inserted == 1;
   }
+
+  public boolean consumeLease(SettlementAttempt attempt) {
+    return jdbc.update(
+            "delete from settlement_attempt where bet_id = ? and action = ? and lease_token = ?",
+            attempt.betId(),
+            attempt.action().name(),
+            attempt.lease().token())
+        == 1;
+  }
 }
diff --git a/src/main/java/com/sportsbook/settlement/execution/SettlementFinalizer.java b/src/main/java/com/sportsbook/settlement/execution/SettlementFinalizer.java
new file mode 100644
index 0000000..04a2860
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/execution/SettlementFinalizer.java
@@ -0,0 +1,46 @@
+package com.sportsbook.settlement.execution;
+
+import com.sportsbook.settlement.config.SettlementTopics;
+import com.sportsbook.settlement.domain.Bet;
+import com.sportsbook.settlement.domain.SettlementStatus;
+import com.sportsbook.settlement.outbox.OutboxEventRepository;
+import com.sportsbook.settlement.outbox.SettlementEventFactory;
+import com.sportsbook.settlement.outbox.StrictAvroEncoder;
+import com.sportsbook.settlement.persistence.BetRepository;
+import java.time.Instant;
+import org.springframework.stereotype.Component;
+import org.springframework.transaction.annotation.Transactional;
+
+@Component
+public class SettlementFinalizer {
+
+  private final BetRepository bets;
+  private final SettlementAttemptRepository attempts;
+  private final OutboxEventRepository outbox;
+  private final SettlementEventFactory events;
+
+  public SettlementFinalizer(
+      BetRepository bets,
+      SettlementAttemptRepository attempts,
+      OutboxEventRepository outbox,
+      SettlementTopics topics) {
+    this.bets = bets;
+    this.attempts = attempts;
+    this.outbox = outbox;
+    this.events = new SettlementEventFactory(topics, new StrictAvroEncoder());
+  }
+
+  @Transactional
+  public boolean settle(SettlementAttempt attempt, Instant now) {
+    if (attempt.action() != SettlementAttempt.Action.SETTLE) {
+      throw new IllegalArgumentException("Resolved finalization requires SETTLE action");
+    }
+    Bet bet = bets.findForUpdateById(attempt.betId()).orElseThrow();
+    if (bet.status() != SettlementStatus.PENDING || !attempts.consumeLease(attempt)) {
+      return false;
+    }
+    bet.recordSettled(attempt.result(), attempt.money().payout(), now);
+    outbox.save(events.settled(bet, attempt.eventId()));
+    return true;
+  }
+}


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


## `feat(recovery): schedule incomplete settlement attempts`

diff --git a/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRecovery.java b/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRecovery.java
new file mode 100644
index 0000000..1784de3
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRecovery.java
@@ -0,0 +1,32 @@
+package com.sportsbook.settlement.execution;
+
+import com.sportsbook.settlement.config.SettlementRuntimeProperties;
+import com.sportsbook.settlement.config.SettlementWorkerConfiguration;
+import org.springframework.scheduling.annotation.Scheduled;
+import org.springframework.stereotype.Component;
+
+@Component
+public class SettlementAttemptRecovery {
+
+  private final SettlementAttemptRepository attempts;
+  private final SettlementExecutionRunner runner;
+  private final SettlementRuntimeProperties runtime;
+
+  public SettlementAttemptRecovery(
+      SettlementAttemptRepository attempts,
+      SettlementExecutionRunner runner,
+      SettlementRuntimeProperties runtime) {
+    this.attempts = attempts;
+    this.runner = runner;
+    this.runtime = runtime;
+  }
+
+  @Scheduled(
+      fixedDelayString = "${settlement.runtime.recovery-interval:PT1S}",
+      initialDelayString = "${settlement.runtime.recovery-interval:PT1S}",
+      scheduler = SettlementWorkerConfiguration.RECOVERY)
+  public SettlementExecutionRunner.BatchResult recover() {
+    var claimed = attempts.claimRecoveryBatch(runtime.leaseDuration(), runtime.batchSize());
+    return runner.fanOut(claimed);
+  }
+}
