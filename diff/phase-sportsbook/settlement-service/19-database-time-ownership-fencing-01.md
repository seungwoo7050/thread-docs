# 데이터베이스 시각 기반 소유권 펜싱

## `feat(persistence): expose transaction database time`

diff --git a/src/main/java/com/sportsbook/settlement/persistence/DatabaseTimeSource.java b/src/main/java/com/sportsbook/settlement/persistence/DatabaseTimeSource.java
new file mode 100644
index 0000000..0363f00
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/persistence/DatabaseTimeSource.java
@@ -0,0 +1,22 @@
+package com.sportsbook.settlement.persistence;
+
+import java.sql.Timestamp;
+import java.time.Instant;
+import java.util.Objects;
+import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.stereotype.Component;
+
+@Component
+public final class DatabaseTimeSource {
+
+  private final JdbcTemplate jdbc;
+
+  public DatabaseTimeSource(JdbcTemplate jdbc) {
+    this.jdbc = jdbc;
+  }
+
+  public Instant currentTimestamp() {
+    Timestamp timestamp = jdbc.queryForObject("select current_timestamp", Timestamp.class);
+    return Objects.requireNonNull(timestamp, "Database timestamp").toInstant();
+  }
+}


## `refactor(settlement): remove caller timed initial claims`

diff --git a/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java b/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java
index 46403fe..349bfbd 100644
--- a/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java
+++ b/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java
@@ -16,19 +16,6 @@ import org.springframework.transaction.annotation.Transactional;
 @Repository
 public class SettlementAttemptRepository {
 
-  private static final String CLAIM_SQL =
-      """
-      INSERT INTO settlement_attempt (
-          bet_id, action, event_id, result, void_reason,
-          committed_amount, payout_amount, locked_release_amount,
-          locked_forfeit_amount, house_profit_amount, currency,
-          lease_token, lease_until, attempt_count, last_error, created_at, updated_at)
-      SELECT ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?
-      FROM bet
-      WHERE bet_id = ? AND status = 'PENDING'
-      ON CONFLICT (bet_id) DO NOTHING
-      """;
-
   private final JdbcTemplate jdbc;
 
   public SettlementAttemptRepository(JdbcTemplate jdbc) {
@@ -95,32 +82,6 @@ public class SettlementAttemptRepository {
                     clock.updatedAt()));
   }
 
-  public boolean claimPending(SettlementAttempt attempt) {
-    SettlementMoneyPlan money = attempt.money();
-    int inserted =
-        jdbc.update(
-            CLAIM_SQL,
-            attempt.betId(),
-            attempt.action().name(),
-            attempt.eventId(),
-            attempt.result() == null ? null : attempt.result().name(),
-            attempt.voidReason(),
-            money.committed().amount(),
-            money.payout().amount(),
-            money.lockedRelease().amount(),
-            money.lockedForfeit().amount(),
-            money.houseProfit().amount(),
-            money.committed().currency().name(),
-            attempt.lease().token(),
-            required(attempt.lease().until()),
-            attempt.attemptCount(),
-            attempt.lastError(),
-            required(attempt.createdAt()),
-            required(attempt.updatedAt()),
-            attempt.betId());
-    return inserted == 1;
-  }
-
   public boolean consumeLease(SettlementAttempt attempt) {
     return jdbc.update(
             "delete from settlement_attempt where bet_id = ? and action = ? and lease_token = ?",


## `feat(settlement): claim attempts with database time`

diff --git a/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java b/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java
index 5e2cb49..752e572 100644
--- a/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java
+++ b/src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java
@@ -7,6 +7,7 @@ import java.time.Duration;
 import java.time.Instant;
 import java.util.ArrayList;
 import java.util.List;
+import java.util.Optional;
 import java.util.UUID;
 import org.springframework.jdbc.core.JdbcTemplate;
 import org.springframework.stereotype.Repository;
@@ -34,6 +35,58 @@ public class SettlementAttemptRepository {
     this.jdbc = jdbc;
   }
 
+  public Optional<SettlementAttempt> claimPending(
+      SettlementAttemptDraft draft, Duration leaseDuration) {
+    long leaseMillis = leaseDuration == null ? 0 : leaseDuration.toMillis();
+    if (leaseMillis < 1) {
+      throw new IllegalArgumentException("Initial settlement lease must be positive");
+    }
+    SettlementMoneyPlan money = draft.money();
+    UUID token = UUID.randomUUID();
+    return jdbc
+        .query(
+            """
+            insert into settlement_attempt (
+                bet_id, action, event_id, result, void_reason,
+                committed_amount, payout_amount, locked_release_amount,
+                locked_forfeit_amount, house_profit_amount, currency,
+                lease_token, lease_until, attempt_count, last_error, created_at, updated_at)
+            select ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?,
+                current_timestamp + (? * interval '1 millisecond'),
+                1, null, current_timestamp, current_timestamp
+            from bet where bet_id = ? and status = 'PENDING'
+            on conflict (bet_id) do nothing
+            returning lease_until, created_at, updated_at
+            """,
+            (result, rowNumber) ->
+                new InitialClock(
+                    result.getTimestamp("lease_until").toInstant(),
+                    result.getTimestamp("created_at").toInstant(),
+                    result.getTimestamp("updated_at").toInstant()),
+            draft.betId(),
+            draft.action().name(),
+            draft.eventId(),
+            draft.result() == null ? null : draft.result().name(),
+            draft.voidReason(),
+            money.committed().amount(),
+            money.payout().amount(),
+            money.lockedRelease().amount(),
+            money.lockedForfeit().amount(),
+            money.houseProfit().amount(),
+            money.committed().currency().name(),
+            token,
+            leaseMillis,
+            draft.betId())
+        .stream()
+        .findFirst()
+        .map(
+            clock ->
+                draft.claimed(
+                    new SettlementLease(token, clock.leaseUntil()),
+                    clock.createdAt(),
+                    clock.updatedAt()));
+  }
+
   public boolean claimPending(SettlementAttempt attempt) {
     SettlementMoneyPlan money = attempt.money();
     int inserted =
@@ -134,4 +187,6 @@ public class SettlementAttemptRepository {
   }
 
   private record RecoveryClock(Instant leaseUntil, Instant updatedAt) {}
+
+  private record InitialClock(Instant leaseUntil, Instant createdAt, Instant updatedAt) {}
 }


## `feat(lifecycle): prepare database timed void claims`

diff --git a/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleSettlementPreparer.java b/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleSettlementPreparer.java
new file mode 100644
index 0000000..a84ab9f
--- /dev/null
+++ b/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleSettlementPreparer.java
@@ -0,0 +1,65 @@
+package com.sportsbook.settlement.lifecycle;
+
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.settlement.config.SettlementRuntimeProperties;
+import com.sportsbook.settlement.domain.SettlementStatus;
+import com.sportsbook.settlement.execution.SettlementAttemptDraft;
+import com.sportsbook.settlement.execution.SettlementAttemptRepository;
+import com.sportsbook.settlement.execution.SettlementExecution;
+import com.sportsbook.settlement.persistence.BetRepository;
+import java.util.Optional;
+import java.util.UUID;
+import org.springframework.stereotype.Service;
+import org.springframework.transaction.annotation.Transactional;
+
+@Service
+public class LifecycleSettlementPreparer {
+
+  private final BetRepository bets;
+  private final SettlementAttemptRepository attempts;
+  private final SettlementRuntimeProperties runtime;
+
+  public LifecycleSettlementPreparer(
+      BetRepository bets,
+      SettlementAttemptRepository attempts,
+      SettlementRuntimeProperties runtime) {
+    this.bets = bets;
+    this.attempts = attempts;
+    this.runtime = runtime;
+  }
+
+  @Transactional
+  public Optional<SettlementExecution> prepare(UUID betId, UUID eventId, String reason) {
+    var bet = bets.findForUpdateById(betId).orElseThrow();
+    if (bet.status() != SettlementStatus.PENDING || attempts.exists(betId)) {
+      return Optional.empty();
+    }
+    var draft =
+        SettlementAttemptDraft.wholeSlipVoid(
+            bet.betId(), eventId, reason, totalExposure(bet.stake(), bet.slipType()));
+    return attempts
+        .claimPending(draft, runtime.leaseDuration())
+        .map(claimed -> new SettlementExecution(claimed, bet.userId()))
+        .or(
+            () -> {
+              throw new IllegalStateException("Initial lifecycle claim lost after bet lock");
+            });
+  }
+
+  private static Money totalExposure(Money unitStake, BetSlipType slipType) {
+    long lines = 1;
+    if (slipType instanceof BetSlipType.System system) {
+      lines = combinations(system.totalSelections(), system.minWins());
+    }
+    return unitStake.multiply(lines);
+  }
+
+  private static long combinations(int n, int k) {
+    long result = 1;
+    for (int factor = 1; factor <= k; factor++) {
+      result = Math.multiplyExact(result, n - k + factor) / factor;
+    }
+    return result;
+  }
+}


## `fix(correction): persist plans with database time`

diff --git a/src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java b/src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java
index 2dce2f1..8533dfe 100644
--- a/src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java
+++ b/src/main/java/com/sportsbook/settlement/correction/RevisionPlanRepository.java
@@ -43,8 +43,10 @@ public class RevisionPlanRepository {
                 source_result_settled_at, state, lease_token, lease_until, attempt_count,
                 next_retry_at, created_at, updated_at)
             select ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, 'PENDING', ?,
-                current_timestamp + (? * interval '1 millisecond'), 1, null, ?, ?
+                current_timestamp + (? * interval '1 millisecond'), 1, null,
+                current_timestamp, current_timestamp
             from bet where bet_id = ? and status = 'SETTLED' and revision_number = ?
+                and ? <= current_timestamp
             on conflict do nothing
             returning lease_until
             """,
@@ -67,10 +69,9 @@ public class RevisionPlanRepository {
             required(target.sourceResultSettledAt()),
             leaseToken,
             leaseMillis,
-            required(plan.createdAt()),
-            required(plan.createdAt()),
             target.betId(),
-            target.revisionNumber() - 1);
+            target.revisionNumber() - 1,
+            required(target.sourceResultSettledAt()));
     if (!claimed.isEmpty()) {
       jdbc.batchUpdate(
           """


## `feat(correction): gate candidate decisions by database time`

diff --git a/src/main/java/com/sportsbook/settlement/correction/ResultCandidateIntake.java b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateIntake.java
index 3ddb556..458b2c7 100644
--- a/src/main/java/com/sportsbook/settlement/correction/ResultCandidateIntake.java
+++ b/src/main/java/com/sportsbook/settlement/correction/ResultCandidateIntake.java
@@ -26,7 +26,7 @@ public class ResultCandidateIntake {
 
   @Transactional
   public IntakeResult ingest(MatchResultRecord result) {
-    var accepted = store.findAcceptedCandidate(result.eventId());
+    var acceptedAtRecord = store.findAcceptedCandidate(result.eventId());
     String fingerprint =
         fingerprints.fingerprint(result.eventId(), result.mode(), result.outcomes());
     ResultCandidate candidate =
@@ -37,33 +37,36 @@ public class ResultCandidateIntake {
             result.outcomes(),
             result.settledAt(),
             result.receivedAt(),
-            accepted.map(ResultCandidateStore.AcceptedCandidate::candidateId).orElse(null));
+            acceptedAtRecord.map(ResultCandidateStore.AcceptedCandidate::candidateId).orElse(null));
     ResultCandidateStore.RecordOutcome recorded = store.record(candidate);
-    if (recorded.kind() != ResultCandidateStore.RecordKind.CREATED) {
-      if (recorded.state() == ResultCandidateState.ACCEPTED) {
-        return IntakeResult.ACCEPTED_REPLAY;
-      }
-      return recorded.kind() == ResultCandidateStore.RecordKind.EXACT_REPLAY
-          ? IntakeResult.EXACT_REPLAY
-          : IntakeResult.NO_CHANGE;
+    if (recorded.state() == ResultCandidateState.ACCEPTED) {
+      return IntakeResult.ACCEPTED_REPLAY;
+    }
+    if (recorded.state() != ResultCandidateState.PENDING) {
+      return IntakeResult.NO_CHANGE;
     }
+    if (store.holdWhileFuture(recorded.candidateId())) {
+      return IntakeResult.FUTURE_HELD;
+    }
+    var accepted = store.findAcceptedCandidate(result.eventId());
+    var candidateReceivedAt =
+        recorded.receivedAt() == null ? result.receivedAt() : recorded.receivedAt();
     if (accepted.isEmpty()) {
-      if (store.acceptFirst(candidate.candidateId(), result.receivedAt())) {
+      if (store.acceptFirst(recorded.candidateId(), result.receivedAt())) {
         return IntakeResult.FIRST_ACCEPTED;
       }
-      return store.supersedeStale(candidate.candidateId(), result.receivedAt())
+      return store.supersedeStale(recorded.candidateId(), result.receivedAt())
           ? IntakeResult.CORRECTION_SUPERSEDED
           : IntakeResult.CORRECTION_PENDING;
     }
     ResultCandidateStore.AcceptedCandidate current = accepted.orElseThrow();
-    if (result.receivedAt().isAfter(current.receivedAt().plus(runtime.correctionWindow()))) {
+    if (candidateReceivedAt.isAfter(current.receivedAt().plus(runtime.correctionWindow()))) {
       return IntakeResult.LATE_HELD;
     }
-    if (store.replaceAccepted(
-        candidate.candidateId(), current.candidateId(), result.receivedAt())) {
+    if (store.replaceAccepted(recorded.candidateId(), current.candidateId(), result.receivedAt())) {
       return IntakeResult.AUTO_CORRECTION_ACCEPTED;
     }
-    return store.supersedeStale(candidate.candidateId(), result.receivedAt())
+    return store.supersedeStale(recorded.candidateId(), result.receivedAt())
         ? IntakeResult.CORRECTION_SUPERSEDED
         : IntakeResult.CORRECTION_PENDING;
   }
@@ -74,6 +77,7 @@ public class ResultCandidateIntake {
     NO_CHANGE,
     FIRST_ACCEPTED,
     AUTO_CORRECTION_ACCEPTED,
+    FUTURE_HELD,
     LATE_HELD,
     CORRECTION_SUPERSEDED,
     CORRECTION_PENDING


## `refactor(settlement): remove caller finalization clock`

diff --git a/src/main/java/com/sportsbook/settlement/execution/SettlementExecutionRunner.java b/src/main/java/com/sportsbook/settlement/execution/SettlementExecutionRunner.java
index 279e0dd..e5d1161 100644
--- a/src/main/java/com/sportsbook/settlement/execution/SettlementExecutionRunner.java
+++ b/src/main/java/com/sportsbook/settlement/execution/SettlementExecutionRunner.java
@@ -1,7 +1,6 @@
 package com.sportsbook.settlement.execution;
 
 import java.time.Clock;
-import java.time.Instant;
 import java.util.List;
 import org.springframework.stereotype.Component;
 
@@ -30,11 +29,10 @@ public class SettlementExecutionRunner {
       wallet.releaseLocked(attempt, execution.userId());
       wallet.forfeitLocked(attempt, execution.userId());
       wallet.payHouseProfit(attempt, execution.userId());
-      Instant now = clock.instant();
       boolean finalized =
           attempt.action() == SettlementAttempt.Action.SETTLE
-              ? finalizer.settle(attempt, now)
-              : finalizer.voidBet(attempt, now);
+              ? finalizer.settle(attempt)
+              : finalizer.voidBet(attempt);
       if (!finalized) {
         throw new IllegalStateException("Settlement lease was lost before finalization");
       }
diff --git a/src/main/java/com/sportsbook/settlement/execution/SettlementFinalizer.java b/src/main/java/com/sportsbook/settlement/execution/SettlementFinalizer.java
index c396f0e..cd22361 100644
--- a/src/main/java/com/sportsbook/settlement/execution/SettlementFinalizer.java
+++ b/src/main/java/com/sportsbook/settlement/execution/SettlementFinalizer.java
@@ -8,7 +8,6 @@ import com.sportsbook.settlement.outbox.OutboxEventRepository;
 import com.sportsbook.settlement.outbox.SettlementEventFactory;
 import com.sportsbook.settlement.outbox.StrictAvroEncoder;
 import com.sportsbook.settlement.persistence.BetRepository;
-import java.time.Instant;
 import org.springframework.stereotype.Component;
 import org.springframework.transaction.annotation.Transactional;
 
@@ -32,7 +31,7 @@ public class SettlementFinalizer {
   }
 
   @Transactional
-  public boolean settle(SettlementAttempt attempt, Instant now) {
+  public boolean settle(SettlementAttempt attempt) {
     if (attempt.action() != SettlementAttempt.Action.SETTLE) {
       throw new IllegalArgumentException("Resolved finalization requires SETTLE action");
     }
@@ -50,7 +49,7 @@ public class SettlementFinalizer {
   }
 
   @Transactional
-  public boolean voidBet(SettlementAttempt attempt, Instant now) {
+  public boolean voidBet(SettlementAttempt attempt) {
     if (attempt.action() != SettlementAttempt.Action.VOID) {
       throw new IllegalArgumentException("Whole slip finalization requires VOID action");
     }


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


