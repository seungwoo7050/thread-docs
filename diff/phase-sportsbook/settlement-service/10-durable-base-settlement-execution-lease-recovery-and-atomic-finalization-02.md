## `feat(recovery): hydrate durable attempt rows`

diff --git a/src/main/java/com/sportsbook/settlement/execution/SettlementRecoveryRow.java b/src/main/java/com/sportsbook/settlement/execution/SettlementRecoveryRow.java
new file mode 100644
index 0000000..aa0afff
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/execution/SettlementRecoveryRow.java
@@ -0,0 +1,60 @@
+package com.sportsbook.settlement.execution;
+
+import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
+import java.sql.ResultSet;
+import java.sql.SQLException;
+import java.time.Instant;
+import java.util.UUID;
+
+record SettlementRecoveryRow(
+    UUID betId,
+    SettlementAttempt.Action action,
+    UUID eventId,
+    SettlementResult result,
+    String voidReason,
+    SettlementMoneyPlan money,
+    int attemptCount,
+    Instant createdAt,
+    UUID userId) {
+
+  static SettlementRecoveryRow read(ResultSet row) throws SQLException {
+    Currency currency = Currency.valueOf(row.getString("currency"));
+    SettlementMoneyPlan money =
+        new SettlementMoneyPlan(
+            new Money(row.getLong("committed_amount"), currency),
+            new Money(row.getLong("payout_amount"), currency),
+            new Money(row.getLong("locked_release_amount"), currency),
+            new Money(row.getLong("locked_forfeit_amount"), currency),
+            new Money(row.getLong("house_profit_amount"), currency));
+    String result = row.getString("result");
+    return new SettlementRecoveryRow(
+        row.getObject("bet_id", UUID.class),
+        SettlementAttempt.Action.valueOf(row.getString("action")),
+        row.getObject("event_id", UUID.class),
+        result == null ? null : SettlementResult.valueOf(result),
+        row.getString("void_reason"),
+        money,
+        row.getInt("attempt_count"),
+        row.getTimestamp("created_at").toInstant(),
+        row.getObject("user_id", UUID.class));
+  }
+
+  SettlementExecution execution(SettlementLease lease, Instant updatedAt) {
+    SettlementAttempt attempt =
+        new SettlementAttempt(
+            betId,
+            action,
+            eventId,
+            result,
+            voidReason,
+            money,
+            lease,
+            attemptCount + 1,
+            null,
+            createdAt,
+            updatedAt);
+    return new SettlementExecution(attempt, userId);
+  }
+}


## `feat(recovery): claim ordered attempt batches`

diff --git a/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java b/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java
index 9b9d7fc..5e2cb49 100644
--- a/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java
+++ b/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java
@@ -3,9 +3,14 @@ package com.sportsbook.settlement.execution;
 import static com.sportsbook.settlement.persistence.JdbcTimestamps.required;
 
 import com.sportsbook.settlement.client.WalletFailurePolicy;
+import java.time.Duration;
 import java.time.Instant;
+import java.util.ArrayList;
+import java.util.List;
+import java.util.UUID;
 import org.springframework.jdbc.core.JdbcTemplate;
 import org.springframework.stereotype.Repository;
+import org.springframework.transaction.annotation.Transactional;
 
 @Repository
 public class SettlementAttemptRepository {
@@ -81,4 +86,52 @@ public class SettlementAttemptRepository {
             attempt.lease().token())
         == 1;
   }
+
+  @Transactional
+  public List<SettlementExecution> claimRecoveryBatch(Duration leaseDuration, int limit) {
+    long leaseMillis = leaseDuration == null ? 0 : leaseDuration.toMillis();
+    if (leaseMillis < 1 || limit < 1 || limit > 1000) {
+      throw new IllegalArgumentException("Recovery batch size must be between 1 and 1000");
+    }
+    List<SettlementRecoveryRow> rows =
+        jdbc.query(
+            """
+            select a.*, b.user_id
+            from settlement_attempt a join bet b on b.bet_id = a.bet_id
+            where b.status = 'PENDING'
+                and (a.lease_until is null or a.lease_until <= current_timestamp)
+            order by a.updated_at, a.bet_id limit ? for update of a skip locked
+            """,
+            (result, rowNumber) -> SettlementRecoveryRow.read(result),
+            limit);
+    List<SettlementExecution> executions = new ArrayList<>(rows.size());
+    for (SettlementRecoveryRow row : rows) {
+      UUID token = UUID.randomUUID();
+      RecoveryClock clock =
+          jdbc
+              .query(
+                  """
+                  update settlement_attempt set lease_token = ?,
+                      lease_until = current_timestamp + (? * interval '1 millisecond'),
+                      attempt_count = attempt_count + 1, last_error = null,
+                      updated_at = current_timestamp
+                  where bet_id = ? returning lease_until, updated_at
+                  """,
+                  (result, rowNumber) ->
+                      new RecoveryClock(
+                          result.getTimestamp("lease_until").toInstant(),
+                          result.getTimestamp("updated_at").toInstant()),
+                  token,
+                  leaseMillis,
+                  row.betId())
+              .stream()
+              .findFirst()
+              .orElseThrow(() -> new IllegalStateException("Recovery claim lost locked attempt"));
+      SettlementLease lease = new SettlementLease(token, clock.leaseUntil());
+      executions.add(row.execution(lease, clock.updatedAt()));
+    }
+    return List.copyOf(executions);
+  }
+
+  private record RecoveryClock(Instant leaseUntil, Instant updatedAt) {}
 }


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


## `fix(settlement): fence finalization with database time`

diff --git a/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java b/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java
index 349bfbd..cbd9a51 100644
--- a/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java
+++ b/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java
@@ -82,13 +82,20 @@ public class SettlementAttemptRepository {
                     clock.updatedAt()));
   }
 
-  public boolean consumeLease(SettlementAttempt attempt) {
-    return jdbc.update(
-            "delete from settlement_attempt where bet_id = ? and action = ? and lease_token = ?",
+  public Optional<Instant> consumeLease(SettlementAttempt attempt) {
+    return jdbc
+        .query(
+            """
+            delete from settlement_attempt where bet_id = ? and action = ?
+                and lease_token = ? and lease_until > current_timestamp
+            returning date_trunc('milliseconds', current_timestamp) as finalized_at
+            """,
+            (result, rowNumber) -> result.getTimestamp("finalized_at").toInstant(),
             attempt.betId(),
             attempt.action().name(),
             attempt.lease().token())
-        == 1;
+        .stream()
+        .findFirst();
   }
 
   public boolean releaseForRecovery(SettlementAttempt attempt, Throwable failure, Instant now) {
diff --git a/src/main/java/com/sportsbook/settlement/execution/SettlementFinalizer.java b/src/main/java/com/sportsbook/settlement/execution/SettlementFinalizer.java
index 0cfb9dc..c396f0e 100644
--- a/src/main/java/com/sportsbook/settlement/execution/SettlementFinalizer.java
+++ b/src/main/java/com/sportsbook/settlement/execution/SettlementFinalizer.java
@@ -37,10 +37,14 @@ public class SettlementFinalizer {
       throw new IllegalArgumentException("Resolved finalization requires SETTLE action");
     }
     Bet bet = bets.findForUpdateById(attempt.betId()).orElseThrow();
-    if (bet.status() != SettlementStatus.PENDING || !attempts.consumeLease(attempt)) {
+    if (bet.status() != SettlementStatus.PENDING) {
       return false;
     }
-    bet.recordSettled(attempt.result(), attempt.money().payout(), now);
+    var finalizedAt = attempts.consumeLease(attempt);
+    if (finalizedAt.isEmpty()) {
+      return false;
+    }
+    bet.recordSettled(attempt.result(), attempt.money().payout(), finalizedAt.orElseThrow());
     outbox.save(events.settled(bet, attempt.eventId()));
     return true;
   }
@@ -51,10 +55,14 @@ public class SettlementFinalizer {
       throw new IllegalArgumentException("Whole slip finalization requires VOID action");
     }
     Bet bet = bets.findForUpdateById(attempt.betId()).orElseThrow();
-    if (bet.status() != SettlementStatus.PENDING || !attempts.consumeLease(attempt)) {
+    if (bet.status() != SettlementStatus.PENDING) {
+      return false;
+    }
+    var finalizedAt = attempts.consumeLease(attempt);
+    if (finalizedAt.isEmpty()) {
       return false;
     }
-    bet.recordVoided(attempt.money().payout(), now);
+    bet.recordVoided(attempt.money().payout(), finalizedAt.orElseThrow());
     outbox.save(events.voided(bet, attempt.eventId(), VoidReason.valueOf(attempt.voidReason())));
     return true;
   }


## `test(recovery): exercise PostgreSQL attempt recovery`

diff --git a/src/test/java/com/sportsbook/settlement/persistence/PostgresRecoveryIntegrationTest.java b/src/test/java/com/sportsbook/settlement/persistence/PostgresRecoveryIntegrationTest.java
new file mode 100644
index 0000000..e56d569
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/persistence/PostgresRecoveryIntegrationTest.java
@@ -0,0 +1,81 @@
+package com.sportsbook.settlement.persistence;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.settlement.execution.SettlementAttempt;
+import com.sportsbook.settlement.execution.SettlementAttemptRepository;
+import com.sportsbook.settlement.execution.SettlementExecution;
+import com.sportsbook.settlement.execution.SettlementLease;
+import com.sportsbook.settlement.execution.SettlementMoneyPlan;
+import java.sql.Timestamp;
+import java.time.Duration;
+import java.time.Instant;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+
+class PostgresRecoveryIntegrationTest extends PostgresIntegrationSupport {
+
+  @Autowired private SettlementAttemptRepository attempts;
+
+  @Test
+  void reclaimsExpiredAndReleasedLeasesWithoutTakingActiveWork() {
+    Instant now = jdbc.queryForObject("select current_timestamp", Timestamp.class).toInstant();
+    PendingBet expiredBet = insertPendingBet(UUID.randomUUID());
+    PendingBet releasedBet = insertPendingBet(UUID.randomUUID());
+    PendingBet activeBet = insertPendingBet(UUID.randomUUID());
+    SettlementAttempt expired = attempt(expiredBet, now.minusSeconds(60), now.minusSeconds(30));
+    SettlementAttempt released = attempt(releasedBet, now.minusSeconds(20), now.plusSeconds(10));
+    SettlementAttempt active = attempt(activeBet, now.minusSeconds(10), now.plusSeconds(30));
+    assertThat(attempts.claimPending(expired)).isTrue();
+    assertThat(attempts.claimPending(released)).isTrue();
+    assertThat(attempts.claimPending(active)).isTrue();
+
+    assertThat(
+            attempts.releaseForRecovery(
+                released, new IllegalStateException("secret response body"), now.minusSeconds(5)))
+        .isTrue();
+    assertThat(
+            jdbc.queryForObject(
+                "select last_error from settlement_attempt where bet_id = ?",
+                String.class,
+                released.betId()))
+        .isEqualTo("IllegalStateException");
+
+    List<SettlementExecution> claimed = attempts.claimRecoveryBatch(Duration.ofSeconds(30), 10);
+
+    assertThat(claimed)
+        .extracting(execution -> execution.attempt().betId())
+        .containsExactly(expired.betId(), released.betId());
+    assertThat(claimed)
+        .allSatisfy(execution -> assertThat(execution.attempt().attemptCount()).isEqualTo(2));
+    assertThat(claimed)
+        .allSatisfy(
+            execution -> {
+              Instant until = execution.attempt().lease().until();
+              Instant updated = execution.attempt().updatedAt();
+              assertThat(until).isEqualTo(updated.plusSeconds(30));
+            });
+  }
+
+  private SettlementAttempt attempt(PendingBet bet, Instant createdAt, Instant leaseUntil) {
+    SettlementMoneyPlan money =
+        new SettlementMoneyPlan(
+            Money.krw(100), Money.krw(200), Money.krw(100), Money.krw(0), Money.krw(100));
+    return new SettlementAttempt(
+        bet.betId(),
+        SettlementAttempt.Action.SETTLE,
+        bet.eventId(),
+        SettlementResult.WON,
+        null,
+        money,
+        new SettlementLease(UUID.randomUUID(), leaseUntil),
+        1,
+        null,
+        createdAt,
+        createdAt);
+  }
+}


## `test(load): validate concurrent recovery claims`

diff --git a/src/test/java/com/sportsbook/settlement/persistence/PostgresRecoveryLoadIntegrationTest.java b/src/test/java/com/sportsbook/settlement/persistence/PostgresRecoveryLoadIntegrationTest.java
new file mode 100644
index 0000000..5e70de3
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/persistence/PostgresRecoveryLoadIntegrationTest.java
@@ -0,0 +1,68 @@
+package com.sportsbook.settlement.persistence;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.settlement.execution.SettlementAttempt;
+import com.sportsbook.settlement.execution.SettlementAttemptDraft;
+import com.sportsbook.settlement.execution.SettlementAttemptRepository;
+import com.sportsbook.settlement.execution.SettlementFinalizer;
+import com.sportsbook.settlement.execution.SettlementMoneyPlan;
+import java.time.Duration;
+import java.util.ArrayList;
+import java.util.List;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+
+class PostgresRecoveryLoadIntegrationTest extends PostgresIntegrationSupport {
+
+  private static final int ATTEMPTS = 128;
+
+  @Autowired private SettlementAttemptRepository attempts;
+  @Autowired private SettlementFinalizer finalizer;
+
+  @Test
+  void distributesClaimsAndFencesTheSupersededOwner() throws Exception {
+    List<SettlementAttempt> original = new ArrayList<>(ATTEMPTS);
+    for (int index = 0; index < ATTEMPTS; index++) {
+      PendingBet bet = insertPendingBet(UUID.randomUUID());
+      original.add(attempts.claimPending(draft(bet), Duration.ofSeconds(30)).orElseThrow());
+    }
+    jdbc.update("update bet_selection set outcome = 'WON'");
+    jdbc.update(
+        "update settlement_attempt set lease_until = current_timestamp - interval '1 second'");
+
+    var recovered = RecoveryClaimLoadHarness.claim(attempts, 4, 32);
+
+    assertThat(recovered).hasSize(ATTEMPTS);
+    assertThat(recovered).extracting(item -> item.attempt().betId()).doesNotHaveDuplicates();
+    assertThat(recovered)
+        .extracting(item -> item.attempt().lease().token())
+        .doesNotHaveDuplicates();
+    assertThat(recovered)
+        .allSatisfy(item -> assertThat(item.attempt().attemptCount()).isEqualTo(2));
+    SettlementAttempt stale = original.get(0);
+    SettlementAttempt owner =
+        recovered.stream()
+            .map(item -> item.attempt())
+            .filter(attempt -> attempt.betId().equals(stale.betId()))
+            .findFirst()
+            .orElseThrow();
+
+    assertThat(finalizer.settle(stale)).isFalse();
+    assertThat(finalizer.settle(owner)).isTrue();
+    assertThat(finalizer.settle(owner)).isFalse();
+    assertThat(jdbc.queryForObject("select count(*) from bet where status='SETTLED'", Long.class))
+        .isEqualTo(1);
+    assertThat(jdbc.queryForObject("select count(*) from outbox_event", Long.class)).isEqualTo(1);
+  }
+
+  private static SettlementAttemptDraft draft(PendingBet bet) {
+    SettlementMoneyPlan money =
+        new SettlementMoneyPlan(
+            Money.krw(100), Money.krw(200), Money.krw(100), Money.krw(0), Money.krw(100));
+    return SettlementAttemptDraft.resolved(bet.betId(), bet.eventId(), SettlementResult.WON, money);
+  }
+}
