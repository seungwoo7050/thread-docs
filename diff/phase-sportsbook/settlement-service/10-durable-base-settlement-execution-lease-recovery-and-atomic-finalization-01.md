# 내구성 있는 기본 정산 실행, 임대 복구, 원자적 확정

## `feat(settlement): define initial attempt drafts`

diff --git a/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptDraft.java b/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptDraft.java
new file mode 100644
index 0000000..2e1b306
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptDraft.java
@@ -0,0 +1,56 @@
+package com.sportsbook.settlement.execution;
+
+import com.sportsbook.protocol.domain.SettlementResult;
+import com.sportsbook.protocol.value.Money;
+import java.time.Instant;
+import java.util.Objects;
+import java.util.Set;
+import java.util.UUID;
+
+/** Immutable monetary plan waiting for its first database-owned lease. */
+public record SettlementAttemptDraft(
+    UUID betId,
+    SettlementAttempt.Action action,
+    UUID eventId,
+    SettlementResult result,
+    String voidReason,
+    SettlementMoneyPlan money) {
+
+  private static final Set<String> VOID_REASONS =
+      Set.of("EVENT_CANCELLED", "EVENT_POSTPONED", "ADMIN_VOID");
+
+  public SettlementAttemptDraft {
+    Objects.requireNonNull(betId, "betId");
+    Objects.requireNonNull(action, "action");
+    Objects.requireNonNull(eventId, "eventId");
+    Objects.requireNonNull(money, "money");
+    boolean settle =
+        action == SettlementAttempt.Action.SETTLE && result != null && voidReason == null;
+    boolean voided =
+        action == SettlementAttempt.Action.VOID
+            && result == null
+            && VOID_REASONS.contains(voidReason);
+    if (!settle && !voided) {
+      throw new IllegalArgumentException("Invalid settlement attempt draft");
+    }
+  }
+
+  public static SettlementAttemptDraft resolved(
+      UUID betId, UUID eventId, SettlementResult result, SettlementMoneyPlan money) {
+    return new SettlementAttemptDraft(
+        betId, SettlementAttempt.Action.SETTLE, eventId, result, null, money);
+  }
+
+  public SettlementAttempt claimed(SettlementLease lease, Instant createdAt, Instant updatedAt) {
+    return new SettlementAttempt(
+        betId, action, eventId, result, voidReason, money, lease, 1, null, createdAt, updatedAt);
+  }
+
+  public static SettlementAttemptDraft wholeSlipVoid(
+      UUID betId, UUID eventId, String reason, Money totalExposure) {
+    Money zero = Money.zero(totalExposure.currency());
+    var refund = new SettlementMoneyPlan(totalExposure, totalExposure, totalExposure, zero, zero);
+    return new SettlementAttemptDraft(
+        betId, SettlementAttempt.Action.VOID, eventId, null, reason, refund);
+  }
+}


## `feat(settlement): prepare resolved attempts`

diff --git a/src/main/java/com/sportsbook/settlement/execution/SettlementAttempt.java b/src/main/java/com/sportsbook/settlement/execution/SettlementAttempt.java
new file mode 100644
index 0000000..ca717e6
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/execution/SettlementAttempt.java
@@ -0,0 +1,52 @@
+package com.sportsbook.settlement.execution;
+
+import com.sportsbook.protocol.domain.SettlementResult;
+import java.time.Instant;
+import java.util.Objects;
+import java.util.UUID;
+
+public record SettlementAttempt(
+    UUID betId,
+    Action action,
+    UUID eventId,
+    SettlementResult result,
+    String voidReason,
+    SettlementMoneyPlan money,
+    SettlementLease lease,
+    int attemptCount,
+    String lastError,
+    Instant createdAt,
+    Instant updatedAt) {
+
+  public SettlementAttempt {
+    Objects.requireNonNull(betId, "betId");
+    Objects.requireNonNull(action, "action");
+    Objects.requireNonNull(eventId, "eventId");
+    Objects.requireNonNull(money, "money");
+    Objects.requireNonNull(lease, "lease");
+    Objects.requireNonNull(createdAt, "createdAt");
+    Objects.requireNonNull(updatedAt, "updatedAt");
+    boolean resolved = action == Action.SETTLE && result != null && voidReason == null;
+    boolean voided =
+        action == Action.VOID && result == null && voidReason != null && !voidReason.isBlank();
+    if ((!resolved && !voided) || attemptCount < 1) {
+      throw new IllegalArgumentException("Invalid settlement attempt action");
+    }
+  }
+
+  public static SettlementAttempt resolved(
+      UUID betId,
+      UUID eventId,
+      SettlementResult result,
+      SettlementMoneyPlan money,
+      SettlementLease lease,
+      Instant now) {
+    return new SettlementAttempt(
+        betId, Action.SETTLE, eventId, result, null, money, lease, 1, null, now, now);
+  }
+
+  public enum Action {
+    SETTLE,
+    VOID
+  }
+}


## `feat(settlement): preserve attempt migration`

diff --git a/src/main/resources/db/migration/V5__settlement_attempt.sql b/src/main/resources/db/migration/V5__settlement_attempt.sql
new file mode 100644
index 0000000..e6008f3
--- /dev/null
+++ b/src/main/resources/db/migration/V5__settlement_attempt.sql
@@ -0,0 +1,45 @@
+-- V5: durable PENDING-only settlement claim.
+--
+-- A wallet operation is external to the database transaction. Persisting the complete money
+-- plan before the first call gives retries and crash recovery one immutable source of truth.
+
+CREATE TABLE settlement_attempt (
+    bet_id                 UUID                     PRIMARY KEY REFERENCES bet (bet_id),
+    action                 VARCHAR(8)               NOT NULL,
+    event_id               UUID                     NOT NULL,
+    result                 VARCHAR(8),
+    void_reason            VARCHAR(32),
+    committed_amount       BIGINT                   NOT NULL,
+    payout_amount          BIGINT                   NOT NULL,
+    locked_release_amount  BIGINT                   NOT NULL,
+    locked_forfeit_amount  BIGINT                   NOT NULL,
+    house_profit_amount    BIGINT                   NOT NULL,
+    currency               VARCHAR(3)               NOT NULL,
+    lease_token            UUID,
+    lease_until            TIMESTAMP WITH TIME ZONE,
+    attempt_count          INT                      NOT NULL,
+    last_error             VARCHAR(1000),
+    created_at             TIMESTAMP WITH TIME ZONE NOT NULL,
+    updated_at             TIMESTAMP WITH TIME ZONE NOT NULL,
+    CONSTRAINT ck_settlement_attempt_action
+        CHECK ((action = 'SETTLE' AND result IS NOT NULL AND void_reason IS NULL)
+            OR (action = 'VOID' AND result IS NULL AND void_reason IS NOT NULL)),
+    CONSTRAINT ck_settlement_attempt_amounts_nonnegative
+        CHECK (committed_amount >= 0 AND payout_amount >= 0
+            AND locked_release_amount >= 0 AND locked_forfeit_amount >= 0
+            AND house_profit_amount >= 0),
+    CONSTRAINT ck_settlement_attempt_committed_conservation
+        CHECK (locked_release_amount + locked_forfeit_amount = committed_amount),
+    CONSTRAINT ck_settlement_attempt_payout_conservation
+        CHECK (locked_release_amount + house_profit_amount = payout_amount),
+    CONSTRAINT ck_settlement_attempt_lease_pair
+        CHECK ((lease_token IS NULL AND lease_until IS NULL)
+            OR (lease_token IS NOT NULL AND lease_until IS NOT NULL)),
+    CONSTRAINT ck_settlement_attempt_count CHECK (attempt_count >= 1)
+);
+
+COMMENT ON TABLE settlement_attempt IS
+    'Durable first-action settlement claim; one immutable wallet plan per PENDING bet.';
+
+CREATE INDEX ix_settlement_attempt_recovery
+    ON settlement_attempt (lease_until, updated_at);


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


## `feat(settlement): fence execution with leases`

diff --git a/src/main/java/com/sportsbook/settlement/execution/SettlementLease.java b/src/main/java/com/sportsbook/settlement/execution/SettlementLease.java
new file mode 100644
index 0000000..b6a1960
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/execution/SettlementLease.java
@@ -0,0 +1,26 @@
+package com.sportsbook.settlement.execution;
+
+import java.time.Duration;
+import java.time.Instant;
+import java.util.Objects;
+import java.util.UUID;
+
+public record SettlementLease(UUID token, Instant until) {
+
+  public SettlementLease {
+    Objects.requireNonNull(token, "token");
+    Objects.requireNonNull(until, "until");
+  }
+
+  public static SettlementLease acquire(Instant now, Duration duration) {
+    Objects.requireNonNull(now, "now");
+    if (duration == null || duration.isZero() || duration.isNegative()) {
+      throw new IllegalArgumentException("Lease duration must be positive");
+    }
+    return new SettlementLease(UUID.randomUUID(), now.plus(duration));
+  }
+
+  public boolean isExpiredAt(Instant instant) {
+    return !until.isAfter(Objects.requireNonNull(instant, "instant"));
+  }
+}


## `feat(settlement): detect immutable attempt ownership`

diff --git a/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java b/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java
index 752e572..46403fe 100644
--- a/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java
+++ b/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java
@@ -35,6 +35,14 @@ public class SettlementAttemptRepository {
     this.jdbc = jdbc;
   }
 
+  public boolean exists(UUID betId) {
+    return Boolean.TRUE.equals(
+        jdbc.queryForObject(
+            "select exists (select 1 from settlement_attempt where bet_id = ?)",
+            Boolean.class,
+            betId));
+  }
+
   public Optional<SettlementAttempt> claimPending(
       SettlementAttemptDraft draft, Duration leaseDuration) {
     long leaseMillis = leaseDuration == null ? 0 : leaseDuration.toMillis();


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


## `feat(settlement): finalize whole slip voids atomically`

diff --git a/src/main/java/com/sportsbook/settlement/execution/SettlementFinalizer.java b/src/main/java/com/sportsbook/settlement/execution/SettlementFinalizer.java
index 04a2860..0cfb9dc 100644
--- a/src/main/java/com/sportsbook/settlement/execution/SettlementFinalizer.java
+++ b/src/main/java/com/sportsbook/settlement/execution/SettlementFinalizer.java
@@ -1,5 +1,6 @@
 package com.sportsbook.settlement.execution;
 
+import com.sportsbook.protocol.event.VoidReason;
 import com.sportsbook.settlement.config.SettlementTopics;
 import com.sportsbook.settlement.domain.Bet;
 import com.sportsbook.settlement.domain.SettlementStatus;
@@ -43,4 +44,18 @@ public class SettlementFinalizer {
     outbox.save(events.settled(bet, attempt.eventId()));
     return true;
   }
+
+  @Transactional
+  public boolean voidBet(SettlementAttempt attempt, Instant now) {
+    if (attempt.action() != SettlementAttempt.Action.VOID) {
+      throw new IllegalArgumentException("Whole slip finalization requires VOID action");
+    }
+    Bet bet = bets.findForUpdateById(attempt.betId()).orElseThrow();
+    if (bet.status() != SettlementStatus.PENDING || !attempts.consumeLease(attempt)) {
+      return false;
+    }
+    bet.recordVoided(attempt.money().payout(), now);
+    outbox.save(events.voided(bet, attempt.eventId(), VoidReason.valueOf(attempt.voidReason())));
+    return true;
+  }
 }


## `feat(recovery): release failed attempts safely`

diff --git a/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java b/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java
index 467bf15..9b9d7fc 100644
--- a/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java
+++ b/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java
@@ -2,6 +2,8 @@ package com.sportsbook.settlement.execution;
 
 import static com.sportsbook.settlement.persistence.JdbcTimestamps.required;
 
+import com.sportsbook.settlement.client.WalletFailurePolicy;
+import java.time.Instant;
 import org.springframework.jdbc.core.JdbcTemplate;
 import org.springframework.stereotype.Repository;
 
@@ -61,4 +63,22 @@ public class SettlementAttemptRepository {
             attempt.lease().token())
         == 1;
   }
+
+  public boolean releaseForRecovery(SettlementAttempt attempt, Throwable failure, Instant now) {
+    String summary =
+        failure instanceof WalletFailurePolicy.Failure walletFailure
+            ? "WalletFailure:" + walletFailure.errorCode()
+            : failure.getClass().getSimpleName();
+    return jdbc.update(
+            """
+            update settlement_attempt
+            set lease_token = null, lease_until = null, last_error = ?, updated_at = ?
+            where bet_id = ? and lease_token = ?
+            """,
+            summary,
+            required(now),
+            attempt.betId(),
+            attempt.lease().token())
+        == 1;
+  }
 }


