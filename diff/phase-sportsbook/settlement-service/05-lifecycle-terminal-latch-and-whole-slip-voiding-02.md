## `refactor(lifecycle): use durable void preparation`

diff --git a/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleFanout.java b/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleFanout.java
index 04c75b7..aa819e6 100644
--- a/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleFanout.java
+++ b/src/main/java/com/sportsbook/settlement/lifecycle/LifecycleFanout.java
@@ -1,18 +1,9 @@
 package com.sportsbook.settlement.lifecycle;
 
-import com.sportsbook.protocol.domain.BetSlipType;
 import com.sportsbook.protocol.event.EventLifecycleStatus;
-import com.sportsbook.protocol.value.Money;
-import com.sportsbook.settlement.config.SettlementRuntimeProperties;
-import com.sportsbook.settlement.domain.Bet;
-import com.sportsbook.settlement.execution.SettlementAttempt;
-import com.sportsbook.settlement.execution.SettlementAttemptRepository;
 import com.sportsbook.settlement.execution.SettlementExecution;
 import com.sportsbook.settlement.execution.SettlementExecutionRunner;
-import com.sportsbook.settlement.execution.SettlementLease;
 import com.sportsbook.settlement.persistence.BetRepository;
-import java.time.Clock;
-import java.time.Instant;
 import java.util.ArrayList;
 import java.util.List;
 import org.springframework.stereotype.Service;
@@ -21,41 +12,21 @@ import org.springframework.stereotype.Service;
 public class LifecycleFanout {
 
   private final BetRepository bets;
-  private final SettlementAttemptRepository attempts;
+  private final LifecycleSettlementPreparer preparer;
   private final SettlementExecutionRunner runner;
-  private final SettlementRuntimeProperties runtime;
-  private final Clock clock;
 
   public LifecycleFanout(
-      BetRepository bets,
-      SettlementAttemptRepository attempts,
-      SettlementExecutionRunner runner,
-      SettlementRuntimeProperties runtime,
-      Clock clock) {
+      BetRepository bets, LifecycleSettlementPreparer preparer, SettlementExecutionRunner runner) {
     this.bets = bets;
-    this.attempts = attempts;
+    this.preparer = preparer;
     this.runner = runner;
-    this.runtime = runtime;
-    this.clock = clock;
   }
 
   public SettlementExecutionRunner.BatchResult fanOut(LifecycleObservation tombstone) {
     String reason = reason(tombstone.status());
-    Instant now = clock.instant();
     List<SettlementExecution> executions = new ArrayList<>();
     for (var betId : bets.findPendingIdsByEvent(tombstone.eventId())) {
-      Bet bet = bets.findWithSelectionsById(betId).orElseThrow();
-      SettlementAttempt attempt =
-          SettlementAttempt.wholeSlipVoid(
-              bet.betId(),
-              tombstone.eventId(),
-              reason,
-              totalExposure(bet),
-              SettlementLease.acquire(now, runtime.leaseDuration()),
-              now);
-      if (attempts.claimPending(attempt)) {
-        executions.add(new SettlementExecution(attempt, bet.userId()));
-      }
+      preparer.prepare(betId, tombstone.eventId(), reason).ifPresent(executions::add);
     }
     return runner.fanOut(executions);
   }
@@ -67,20 +38,4 @@ public class LifecycleFanout {
       default -> throw new IllegalArgumentException("Lifecycle fanout requires terminal status");
     };
   }
-
-  private static Money totalExposure(Bet bet) {
-    long lines = 1;
-    if (bet.slipType() instanceof BetSlipType.System system) {
-      lines = combinations(system.totalSelections(), system.minWins());
-    }
-    return bet.stake().multiply(lines);
-  }
-
-  private static long combinations(int n, int k) {
-    long result = 1;
-    for (int factor = 1; factor <= k; factor++) {
-      result = Math.multiplyExact(result, n - k + factor) / factor;
-    }
-    return result;
-  }
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


## `test(lifecycle): exercise PostgreSQL tombstone ownership`

diff --git a/src/test/java/com/sportsbook/settlement/persistence/PostgresLifecycleIntegrationTest.java b/src/test/java/com/sportsbook/settlement/persistence/PostgresLifecycleIntegrationTest.java
new file mode 100644
index 0000000..5044060
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/persistence/PostgresLifecycleIntegrationTest.java
@@ -0,0 +1,69 @@
+package com.sportsbook.settlement.persistence;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.protocol.event.EventLifecycleStatus;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.settlement.execution.SettlementAttempt;
+import com.sportsbook.settlement.execution.SettlementAttemptRepository;
+import com.sportsbook.settlement.execution.SettlementLease;
+import com.sportsbook.settlement.lifecycle.LifecycleObservation;
+import com.sportsbook.settlement.lifecycle.LifecycleStore;
+import java.time.Duration;
+import java.time.Instant;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+
+class PostgresLifecycleIntegrationTest extends PostgresIntegrationSupport {
+
+  @Autowired private LifecycleStore lifecycles;
+  @Autowired private SettlementAttemptRepository attempts;
+
+  @Test
+  void persistsTypedTimestampsAndDoesNotStarveUnclaimedTombstones() {
+    Instant now = Instant.parse("2026-08-22T01:00:00Z");
+    UUID claimedEvent = UUID.randomUUID();
+    UUID actionableEvent = UUID.randomUUID();
+    PendingBet claimed = insertPendingBet(claimedEvent);
+    insertPendingBet(actionableEvent);
+    LifecycleObservation scheduled =
+        LifecycleObservation.observe(
+            claimedEvent,
+            EventLifecycleStatus.SCHEDULED,
+            now.minusSeconds(60),
+            now.plusSeconds(3600),
+            now);
+    LifecycleObservation cancelled =
+        LifecycleObservation.observe(
+            claimedEvent, EventLifecycleStatus.CANCELLED, now, null, now.plusSeconds(1));
+    LifecycleObservation postponed =
+        LifecycleObservation.observe(
+            actionableEvent, EventLifecycleStatus.POSTPONED, now, null, now.plusSeconds(2));
+
+    assertThat(lifecycles.record(scheduled)).isEqualTo(LifecycleStore.RecordResult.OBSERVED);
+    assertThat(lifecycles.record(cancelled))
+        .isEqualTo(LifecycleStore.RecordResult.TERMINAL_LATCHED);
+    assertThat(lifecycles.record(postponed))
+        .isEqualTo(LifecycleStore.RecordResult.TERMINAL_LATCHED);
+    assertThat(attempts.claimPending(attempt(claimed, now))).isTrue();
+
+    assertThat(lifecycles.findTombstone(claimedEvent))
+        .get()
+        .extracting(LifecycleObservation::occurredAt, LifecycleObservation::receivedAt)
+        .containsExactly(now, now.plusSeconds(1));
+    assertThat(lifecycles.findActionableTombstones(1))
+        .extracting(LifecycleObservation::eventId)
+        .containsExactly(actionableEvent);
+  }
+
+  private SettlementAttempt attempt(PendingBet bet, Instant now) {
+    return SettlementAttempt.wholeSlipVoid(
+        bet.betId(),
+        bet.eventId(),
+        "EVENT_CANCELLED",
+        Money.krw(100),
+        SettlementLease.acquire(now, Duration.ofSeconds(30)),
+        now);
+  }
+}


## `test(lifecycle): verify durable void fanout`

diff --git a/src/test/java/com/sportsbook/settlement/lifecycle/LifecycleFanoutTest.java b/src/test/java/com/sportsbook/settlement/lifecycle/LifecycleFanoutTest.java
index b45fb4e..2b8fb78 100644
--- a/src/test/java/com/sportsbook/settlement/lifecycle/LifecycleFanoutTest.java
+++ b/src/test/java/com/sportsbook/settlement/lifecycle/LifecycleFanoutTest.java
@@ -6,62 +6,55 @@ import static org.mockito.Mockito.mock;
 import static org.mockito.Mockito.verify;
 import static org.mockito.Mockito.when;
 
-import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.domain.SettlementResult;
 import com.sportsbook.protocol.event.EventLifecycleStatus;
 import com.sportsbook.protocol.value.Money;
-import com.sportsbook.settlement.config.SettlementRuntimeProperties;
-import com.sportsbook.settlement.domain.Bet;
 import com.sportsbook.settlement.execution.SettlementAttempt;
-import com.sportsbook.settlement.execution.SettlementAttemptRepository;
 import com.sportsbook.settlement.execution.SettlementExecution;
 import com.sportsbook.settlement.execution.SettlementExecutionRunner;
+import com.sportsbook.settlement.execution.SettlementLease;
+import com.sportsbook.settlement.execution.SettlementMoneyPlan;
 import com.sportsbook.settlement.persistence.BetRepository;
-import java.time.Clock;
+import java.time.Duration;
 import java.time.Instant;
-import java.time.ZoneOffset;
 import java.util.List;
 import java.util.Optional;
 import java.util.UUID;
 import org.junit.jupiter.api.Test;
-import org.mockito.ArgumentCaptor;
 
 class LifecycleFanoutTest {
 
   @Test
   @SuppressWarnings("unchecked")
-  void claimsFullSystemExposureForEveryPendingBet() {
+  void preparesEveryPendingBetBeforeExecutingTheBatch() {
     BetRepository bets = mock(BetRepository.class);
-    SettlementAttemptRepository attempts = mock(SettlementAttemptRepository.class);
+    LifecycleSettlementPreparer preparer = mock(LifecycleSettlementPreparer.class);
     SettlementExecutionRunner runner = mock(SettlementExecutionRunner.class);
     Instant now = Instant.parse("2026-08-22T00:00:00Z");
-    LifecycleFanout fanout =
-        new LifecycleFanout(
-            bets,
-            attempts,
-            runner,
-            new SettlementRuntimeProperties(null, null, null, 0),
-            Clock.fixed(now, ZoneOffset.UTC));
+    LifecycleFanout fanout = new LifecycleFanout(bets, preparer, runner);
     UUID eventId = UUID.randomUUID();
     UUID betId = UUID.randomUUID();
-    UUID userId = UUID.randomUUID();
-    Bet bet = mock(Bet.class);
-    when(bet.betId()).thenReturn(betId);
-    when(bet.userId()).thenReturn(userId);
-    when(bet.stake()).thenReturn(Money.krw(1000));
-    when(bet.slipType()).thenReturn(new BetSlipType.System(2, 3));
+    SettlementMoneyPlan money =
+        new SettlementMoneyPlan(
+            Money.krw(100), Money.krw(100), Money.krw(100), Money.krw(0), Money.krw(0));
+    SettlementAttempt attempt =
+        SettlementAttempt.resolved(
+            betId,
+            eventId,
+            SettlementResult.VOID,
+            money,
+            SettlementLease.acquire(now, Duration.ofSeconds(30)),
+            now);
+    SettlementExecution execution = new SettlementExecution(attempt, UUID.randomUUID());
     when(bets.findPendingIdsByEvent(eventId)).thenReturn(List.of(betId));
-    when(bets.findWithSelectionsById(betId)).thenReturn(Optional.of(bet));
-    when(attempts.claimPending(any())).thenReturn(true);
+    when(preparer.prepare(betId, eventId, "EVENT_CANCELLED")).thenReturn(Optional.of(execution));
     when(runner.fanOut(any())).thenReturn(new SettlementExecutionRunner.BatchResult(1, 0));
 
     LifecycleObservation tombstone =
         LifecycleObservation.observe(eventId, EventLifecycleStatus.CANCELLED, now, null, now);
     assertThat(fanout.fanOut(tombstone)).isEqualTo(new SettlementExecutionRunner.BatchResult(1, 0));
 
-    ArgumentCaptor<SettlementAttempt> claim = ArgumentCaptor.forClass(SettlementAttempt.class);
-    verify(attempts).claimPending(claim.capture());
-    assertThat(claim.getValue().money().committed()).isEqualTo(Money.krw(3000));
-    assertThat(claim.getValue().voidReason()).isEqualTo("EVENT_CANCELLED");
+    verify(preparer).prepare(betId, eventId, "EVENT_CANCELLED");
     verify(runner).fanOut((List<SettlementExecution>) any(List.class));
   }
 }


## `test(lifecycle): verify database timed void claims`

diff --git a/src/test/java/com/sportsbook/settlement/lifecycle/LifecycleSettlementPreparerTest.java b/src/test/java/com/sportsbook/settlement/lifecycle/LifecycleSettlementPreparerTest.java
new file mode 100644
index 0000000..79a61fd
--- /dev/null
+++ b/src/test/java/com/sportsbook/settlement/lifecycle/LifecycleSettlementPreparerTest.java
@@ -0,0 +1,59 @@
+package com.sportsbook.settlement.lifecycle;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.ArgumentMatchers.any;
+import static org.mockito.ArgumentMatchers.eq;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.when;
+
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.settlement.config.SettlementRuntimeProperties;
+import com.sportsbook.settlement.domain.Bet;
+import com.sportsbook.settlement.execution.SettlementAttemptDraft;
+import com.sportsbook.settlement.execution.SettlementAttemptRepository;
+import com.sportsbook.settlement.execution.SettlementLease;
+import com.sportsbook.settlement.persistence.BetRepository;
+import java.time.Duration;
+import java.time.Instant;
+import java.util.Optional;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class LifecycleSettlementPreparerTest {
+
+  @Test
+  void locksAndClaimsTheFullSystemExposureWithDatabaseTime() {
+    BetRepository bets = mock(BetRepository.class);
+    SettlementAttemptRepository attempts = mock(SettlementAttemptRepository.class);
+    Bet bet = mock(Bet.class);
+    UUID betId = UUID.randomUUID();
+    UUID eventId = UUID.randomUUID();
+    when(bet.betId()).thenReturn(betId);
+    when(bet.userId()).thenReturn(UUID.randomUUID());
+    when(bet.stake()).thenReturn(Money.krw(100));
+    when(bet.slipType()).thenReturn(new BetSlipType.System(2, 3));
+    when(bet.status()).thenReturn(com.sportsbook.settlement.domain.SettlementStatus.PENDING);
+    when(bets.findForUpdateById(betId)).thenReturn(Optional.of(bet));
+    when(attempts.claimPending(any(), eq(Duration.ofSeconds(30))))
+        .thenAnswer(
+            invocation -> {
+              SettlementAttemptDraft draft = invocation.getArgument(0);
+              return Optional.of(
+                  draft.claimed(
+                      new SettlementLease(UUID.randomUUID(), Instant.EPOCH.plusSeconds(30)),
+                      Instant.EPOCH,
+                      Instant.EPOCH));
+            });
+
+    var execution =
+        new LifecycleSettlementPreparer(
+                bets, attempts, new SettlementRuntimeProperties(null, null, null, 0))
+            .prepare(betId, eventId, "EVENT_CANCELLED")
+            .orElseThrow();
+
+    assertThat(execution.attempt().money().committed()).isEqualTo(Money.krw(300));
+    assertThat(execution.attempt().voidReason()).isEqualTo("EVENT_CANCELLED");
+    assertThat(execution.attempt().attemptCount()).isEqualTo(1);
+  }
+}
