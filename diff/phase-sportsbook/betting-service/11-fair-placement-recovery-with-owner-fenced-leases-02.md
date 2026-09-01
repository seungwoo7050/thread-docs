## `test(integration): verify concurrent recovery leases`

diff --git a/src/test/java/com/sportsbook/betting/integration/PostgresRecoveryClaimIntegrationTest.java b/src/test/java/com/sportsbook/betting/integration/PostgresRecoveryClaimIntegrationTest.java
new file mode 100644
index 0000000..0f3688e
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/integration/PostgresRecoveryClaimIntegrationTest.java
@@ -0,0 +1,87 @@
+package com.sportsbook.betting.integration;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import com.sportsbook.betting.persistence.BetRepository;
+import com.sportsbook.betting.support.PostgresIntegrationSupport;
+import java.util.HashSet;
+import java.util.List;
+import java.util.UUID;
+import java.util.concurrent.CompletableFuture;
+import java.util.concurrent.ExecutorService;
+import java.util.concurrent.Executors;
+import java.util.concurrent.TimeUnit;
+import org.junit.jupiter.api.BeforeEach;
+import org.junit.jupiter.api.Test;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.jdbc.core.JdbcTemplate;
+import org.springframework.test.context.ActiveProfiles;
+import org.testcontainers.junit.jupiter.Testcontainers;
+
+@Testcontainers
+@ActiveProfiles("test")
+@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
+class PostgresRecoveryClaimIntegrationTest extends PostgresIntegrationSupport {
+
+  @Autowired BetRepository bets;
+  @Autowired JdbcTemplate jdbc;
+
+  @BeforeEach
+  void seedPendingBets() {
+    jdbc.execute("DELETE FROM bet");
+    jdbc.execute(
+        """
+        INSERT INTO bet (
+            bet_id, user_id, bet_reference, slip_type, stake_amount, stake_currency,
+            max_payout_amount, max_payout_currency, status, idempotency_key, created_at, updated_at
+        )
+        SELECT
+            ('00000000-0000-4000-8000-' || lpad(n::text, 12, '0'))::uuid,
+            ('10000000-0000-4000-8000-' || lpad(n::text, 12, '0'))::uuid,
+            'B-RECOVERY-' || lpad(n::text, 6, '0'), 'SINGLE', 1000, 'KRW',
+            2000, 'KRW', 'PENDING', 'recovery-' || n,
+            CURRENT_TIMESTAMP - INTERVAL '1 hour', CURRENT_TIMESTAMP - INTERVAL '1 hour'
+        FROM generate_series(1, 120) n
+        """);
+  }
+
+  @Test
+  void claimsEveryEligibleRowOnceAndRecoversAnExpiredLease() throws Exception {
+    ExecutorService workers = Executors.newFixedThreadPool(2);
+    try {
+      CompletableFuture<List<UUID>> first = claim("worker-a", workers);
+      CompletableFuture<List<UUID>> second = claim("worker-b", workers);
+      List<UUID> a = first.get(10, TimeUnit.SECONDS);
+      List<UUID> b = second.get(10, TimeUnit.SECONDS);
+      HashSet<UUID> all = new HashSet<>(a);
+
+      assertThat(List.of(a.size(), b.size())).containsExactlyInAnyOrder(100, 20);
+      assertThat(java.util.Collections.disjoint(a, b)).isTrue();
+      all.addAll(b);
+      assertThat(all).hasSize(120);
+
+      UUID expired = a.get(0);
+      jdbc.update(
+          "UPDATE bet SET reconciliation_claim_until = CURRENT_TIMESTAMP - INTERVAL '1 second', "
+              + "reconciliation_eligible_at = CURRENT_TIMESTAMP - INTERVAL '1 second' "
+              + "WHERE bet_id = ?",
+          expired);
+      assertThat(bets.claimReconciliationBatch("worker-c", 1, 60_000, 10_000, 1))
+          .containsExactly(expired);
+      assertThat(bets.clearReconciliationClaim(expired, "worker-a")).isZero();
+
+      jdbc.update("UPDATE bet SET status = 'REJECTED' WHERE bet_id = ?", expired);
+      assertThat(bets.claimReconciliationBatch("worker-d", 1, 60_000, 10_000, 1)).isEmpty();
+      assertThat(bets.clearReconciliationClaim(expired, "worker-c")).isOne();
+    } finally {
+      workers.shutdownNow();
+      assertThat(workers.awaitTermination(10, TimeUnit.SECONDS)).isTrue();
+    }
+  }
+
+  private CompletableFuture<List<UUID>> claim(String owner, ExecutorService workers) {
+    return CompletableFuture.supplyAsync(
+        () -> bets.claimReconciliationBatch(owner, 1, 60_000, 10_000, 100), workers);
+  }
+}


## `feat(recovery): make checkpoint replay idempotent`

diff --git a/src/main/java/com/sportsbook/betting/placement/BetStore.java b/src/main/java/com/sportsbook/betting/placement/BetStore.java
index ba56d02..6d1bcca 100644
--- a/src/main/java/com/sportsbook/betting/placement/BetStore.java
+++ b/src/main/java/com/sportsbook/betting/placement/BetStore.java
@@ -1,6 +1,9 @@
 package com.sportsbook.betting.placement;
 
 import com.sportsbook.betting.domain.Bet;
+import com.sportsbook.betting.domain.CompensationAction;
+import com.sportsbook.betting.domain.CompensationState;
+import com.sportsbook.betting.domain.PlacementPhase;
 import com.sportsbook.betting.outbox.OutboxEvent;
 import com.sportsbook.betting.outbox.OutboxEventRepository;
 import com.sportsbook.betting.persistence.BetRepository;
@@ -68,32 +71,70 @@ public class BetStore {
 
   @Transactional
   public void commitRisk(UUID betId, Instant now) {
-    pending(betId).commitRisk(now);
+    Bet bet = locked(betId);
+    if (bet.status() == BetStatus.ACCEPTED
+        || (bet.status() == BetStatus.PENDING
+            && bet.placementPhase() == PlacementPhase.RISK_COMMITTED)) {
+      return;
+    }
+    requirePending(bet).commitRisk(now);
   }
 
   @Transactional
   public void requireRiskRelease(UUID betId, ErrorCode reason, String detail, Instant now) {
-    pending(betId).requireRiskRelease(reason.name(), detail, now);
+    Bet bet = locked(betId);
+    if (bet.status() == BetStatus.REJECTED
+        || bet.compensationAction() == CompensationAction.RISK_RELEASE) {
+      return;
+    }
+    requirePending(bet).requireRiskRelease(reason.name(), detail, now);
   }
 
   @Transactional
   public void requireWalletRefund(UUID betId, ErrorCode reason, String detail, Instant now) {
-    pending(betId).requireWalletRefund(reason.name(), detail, now);
+    Bet bet = locked(betId);
+    if (bet.status() == BetStatus.REJECTED
+        || bet.compensationAction() == CompensationAction.WALLET_REFUND) {
+      return;
+    }
+    requirePending(bet).requireWalletRefund(reason.name(), detail, now);
   }
 
   @Transactional
   public void beginCompensation(UUID betId, Instant now) {
-    pending(betId).beginCompensation(now);
+    Bet bet = locked(betId);
+    if (bet.status() == BetStatus.REJECTED
+        || bet.compensationState() == CompensationState.IN_PROGRESS
+        || bet.compensationState() == CompensationState.COMPLETED) {
+      return;
+    }
+    requirePending(bet).beginCompensation(now);
   }
 
   @Transactional
   public void completeRiskRelease(UUID betId, boolean committedConflict, Instant now) {
-    pending(betId).completeRiskRelease(committedConflict, now);
+    Bet bet = locked(betId);
+    if (bet.status() == BetStatus.REJECTED
+        || (bet.compensationAction() == CompensationAction.RISK_RELEASE
+            && bet.compensationState() == CompensationState.COMPLETED)) {
+      return;
+    }
+    requirePending(bet).completeRiskRelease(committedConflict, now);
   }
 
   @Transactional
   public void completeWalletRefund(UUID betId, UUID operationId, Instant now) {
-    pending(betId).completeWalletRefund(operationId, now);
+    Bet bet = locked(betId);
+    if (bet.status() == BetStatus.REJECTED) {
+      return;
+    }
+    if (bet.compensationState() == CompensationState.COMPLETED) {
+      if (!operationId.equals(bet.compensationOperationId())) {
+        throw new IllegalStateException("Wallet returned conflicting refund operation ids");
+      }
+      return;
+    }
+    requirePending(bet).completeWalletRefund(operationId, now);
   }
 
   @Transactional
@@ -129,9 +170,12 @@ public class BetStore {
   }
 
   private Bet pending(UUID betId) {
-    Bet bet = locked(betId);
+    return requirePending(locked(betId));
+  }
+
+  private Bet requirePending(Bet bet) {
     if (bet.status() != BetStatus.PENDING) {
-      throw new IllegalStateException("Placement cannot update terminal bet " + betId);
+      throw new IllegalStateException("Placement cannot update terminal bet " + bet.betId());
     }
     return bet;
   }


## `test(recovery): repeat commit and compensation checkpoints`

diff --git a/src/test/java/com/sportsbook/betting/placement/BetStoreReplayTest.java b/src/test/java/com/sportsbook/betting/placement/BetStoreReplayTest.java
new file mode 100644
index 0000000..23f1173
--- /dev/null
+++ b/src/test/java/com/sportsbook/betting/placement/BetStoreReplayTest.java
@@ -0,0 +1,86 @@
+package com.sportsbook.betting.placement;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.when;
+
+import com.sportsbook.betting.domain.Bet;
+import com.sportsbook.betting.domain.BetDraft;
+import com.sportsbook.betting.domain.BetLeg;
+import com.sportsbook.betting.domain.CompensationState;
+import com.sportsbook.betting.domain.PlacementPhase;
+import com.sportsbook.betting.outbox.OutboxEventRepository;
+import com.sportsbook.betting.persistence.BetRepository;
+import com.sportsbook.betting.persistence.PlacementRequestRepository;
+import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.domain.BetStatus;
+import com.sportsbook.protocol.error.ErrorCode;
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.protocol.value.Odds;
+import java.time.Instant;
+import java.util.List;
+import java.util.Optional;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class BetStoreReplayTest {
+
+  @Test
+  void repeatsRiskCommitWithoutRegressingItsCheckpoint() {
+    Bet bet = reserved();
+    bet.confirmWallet(UUID.randomUUID(), Instant.EPOCH);
+    BetStore store = store(bet);
+
+    store.commitRisk(bet.betId(), Instant.EPOCH);
+    store.commitRisk(bet.betId(), Instant.EPOCH.plusSeconds(1));
+
+    assertThat(bet.placementPhase()).isEqualTo(PlacementPhase.RISK_COMMITTED);
+  }
+
+  @Test
+  void repeatsEveryRiskReleaseCheckpointUntilTerminal() {
+    Bet bet = reserved();
+    BetStore store = store(bet);
+
+    store.requireRiskRelease(bet.betId(), ErrorCode.VALIDATION_FAILED, "reject", Instant.EPOCH);
+    store.requireRiskRelease(bet.betId(), ErrorCode.VALIDATION_FAILED, "reject", Instant.EPOCH);
+    store.beginCompensation(bet.betId(), Instant.EPOCH);
+    store.beginCompensation(bet.betId(), Instant.EPOCH);
+    store.completeRiskRelease(bet.betId(), false, Instant.EPOCH);
+    store.completeRiskRelease(bet.betId(), false, Instant.EPOCH);
+    store.rejectAfterCompensation(bet.betId(), Instant.EPOCH);
+    store.rejectAfterCompensation(bet.betId(), Instant.EPOCH);
+
+    assertThat(bet.compensationState()).isEqualTo(CompensationState.COMPLETED);
+    assertThat(bet.status()).isEqualTo(BetStatus.REJECTED);
+  }
+
+  private static BetStore store(Bet bet) {
+    BetRepository bets = mock(BetRepository.class);
+    when(bets.findLockedByBetId(bet.betId())).thenReturn(Optional.of(bet));
+    return new BetStore(
+        bets, mock(OutboxEventRepository.class), mock(PlacementRequestRepository.class));
+  }
+
+  private static Bet reserved() {
+    UUID betId = UUID.randomUUID();
+    Bet bet =
+        Bet.pending(
+            new BetDraft(
+                betId,
+                UUID.randomUUID(),
+                "B-2026-08-22-REPLAY",
+                new BetSlipType.Single(),
+                Money.krw(1_000),
+                Money.krw(2_000),
+                IdempotencyKey.of("replay-" + betId),
+                "a".repeat(64),
+                Instant.EPOCH),
+            List.of(
+                BetLeg.create(
+                    UUID.randomUUID(), UUID.randomUUID(), UUID.randomUUID(), Odds.ofDecimal("2"))));
+    bet.recordRiskReservation(Instant.EPOCH.plusSeconds(30), "b".repeat(64), false, Instant.EPOCH);
+    return bet;
+  }
+}
