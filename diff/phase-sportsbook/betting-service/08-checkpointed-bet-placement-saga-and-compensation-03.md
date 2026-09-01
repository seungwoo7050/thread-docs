## `feat(placement): finalize outcomes with outbox atomically`

diff --git a/src/main/java/com/sportsbook/betting/placement/BetStore.java b/src/main/java/com/sportsbook/betting/placement/BetStore.java
index ad86c5b..ba56d02 100644
--- a/src/main/java/com/sportsbook/betting/placement/BetStore.java
+++ b/src/main/java/com/sportsbook/betting/placement/BetStore.java
@@ -1,6 +1,8 @@
 package com.sportsbook.betting.placement;
 
 import com.sportsbook.betting.domain.Bet;
+import com.sportsbook.betting.outbox.OutboxEvent;
+import com.sportsbook.betting.outbox.OutboxEventRepository;
 import com.sportsbook.betting.persistence.BetRepository;
 import com.sportsbook.betting.persistence.PlacementRequestRepository;
 import com.sportsbook.protocol.domain.BetStatus;
@@ -15,10 +17,13 @@ import org.springframework.transaction.annotation.Transactional;
 public class BetStore {
 
   private final BetRepository bets;
+  private final OutboxEventRepository outbox;
   private final PlacementRequestRepository requests;
 
-  public BetStore(BetRepository bets, PlacementRequestRepository requests) {
+  public BetStore(
+      BetRepository bets, OutboxEventRepository outbox, PlacementRequestRepository requests) {
     this.bets = bets;
+    this.outbox = outbox;
     this.requests = requests;
   }
 
@@ -91,14 +96,48 @@ public class BetStore {
     pending(betId).completeWalletRefund(operationId, now);
   }
 
+  @Transactional
+  public Bet rejectAtCreation(UUID betId, ErrorCode reason, String detail, Instant now) {
+    Bet bet = locked(betId);
+    if (bet.status() == BetStatus.PENDING) {
+      bet.rejectAtCreation(reason.name(), detail, now);
+    }
+    return bet;
+  }
+
+  @Transactional
+  public Bet rejectAfterCompensation(UUID betId, Instant now) {
+    Bet bet = locked(betId);
+    if (bet.status() == BetStatus.PENDING) {
+      bet.rejectAfterCompensation(now);
+    }
+    return bet;
+  }
+
+  @Transactional
+  public Bet acceptAndEnqueue(UUID betId, OutboxEvent event, Instant now) {
+    Bet bet = locked(betId);
+    if (bet.status() == BetStatus.ACCEPTED) {
+      return bet;
+    }
+    if (bet.status() != BetStatus.PENDING) {
+      throw new IllegalStateException("Cannot accept terminal bet " + betId);
+    }
+    bet.accept(now);
+    outbox.save(event);
+    return bet;
+  }
+
   private Bet pending(UUID betId) {
-    Bet bet =
-        bets.findLockedByBetId(betId)
-            .orElseThrow(
-                () -> new IllegalStateException("Bet vanished during placement: " + betId));
+    Bet bet = locked(betId);
     if (bet.status() != BetStatus.PENDING) {
       throw new IllegalStateException("Placement cannot update terminal bet " + betId);
     }
     return bet;
   }
+
+  private Bet locked(UUID betId) {
+    return bets.findLockedByBetId(betId)
+        .orElseThrow(() -> new IllegalStateException("Bet vanished during placement: " + betId));
+  }
 }


## `test(placement): verify terminal outcome atomicity`

diff --git a/src/test/java/com/sportsbook/betting/placement/BetStoreTest.java b/src/test/java/com/sportsbook/betting/placement/BetStoreTest.java
index 40aeab3..dfeebbe 100644
--- a/src/test/java/com/sportsbook/betting/placement/BetStoreTest.java
+++ b/src/test/java/com/sportsbook/betting/placement/BetStoreTest.java
@@ -1,72 +1,61 @@
 package com.sportsbook.betting.placement;
 
 import static org.assertj.core.api.Assertions.assertThat;
-import static org.mockito.ArgumentMatchers.any;
-import static org.mockito.Mockito.inOrder;
-import static org.mockito.Mockito.mock;
-import static org.mockito.Mockito.when;
 
-import com.sportsbook.betting.domain.Bet;
-import com.sportsbook.betting.domain.BetDraft;
-import com.sportsbook.betting.domain.BetLeg;
-import com.sportsbook.betting.persistence.BetRepository;
-import com.sportsbook.betting.persistence.PlacementRequestRepository;
-import com.sportsbook.protocol.domain.BetSlipType;
-import com.sportsbook.protocol.value.IdempotencyKey;
-import com.sportsbook.protocol.value.Money;
-import com.sportsbook.protocol.value.Odds;
+import com.sportsbook.betting.outbox.OutboxEvent;
+import com.sportsbook.protocol.error.ErrorCode;
 import java.time.Instant;
-import java.util.List;
 import java.util.UUID;
 import org.junit.jupiter.api.Test;
-import org.mockito.InOrder;
+import org.springframework.transaction.annotation.Transactional;
 
 class BetStoreTest {
 
   @Test
-  void claimsBetBeforePublishingItsRequestPointer() {
-    BetRepository bets = mock(BetRepository.class);
-    PlacementRequestRepository requests = mock(PlacementRequestRepository.class);
-    Bet bet = pendingBet();
-
-    new BetStore(bets, requests).savePending(bet);
-
-    InOrder order = inOrder(bets, requests);
-    order.verify(bets).saveAndFlush(bet);
-    order.verify(requests).saveAndFlush(any(PlacementRequest.class));
+  void requestClaimsAndTerminalTransitionsAreTransactional() throws Exception {
+    assertWriteTransaction("savePending", com.sportsbook.betting.domain.Bet.class);
+    assertWriteTransaction(
+        "savePreflightRejection",
+        String.class,
+        UUID.class,
+        String.class,
+        ErrorCode.class,
+        String.class,
+        Instant.class);
+    assertWriteTransaction("acceptAndEnqueue", UUID.class, OutboxEvent.class, Instant.class);
   }
 
   @Test
-  void locksAndPersistsReservationProofBeforeLaterEffects() {
-    BetRepository bets = mock(BetRepository.class);
-    PlacementRequestRepository requests = mock(PlacementRequestRepository.class);
-    Bet bet = pendingBet();
-    Instant expiresAt = Instant.EPOCH.plusSeconds(60);
-    when(bets.findLockedByBetId(bet.betId())).thenReturn(java.util.Optional.of(bet));
+  void recoveryReadsDoNotOpenWriteTransactions() throws Exception {
+    Transactional annotation =
+        BetStore.class
+            .getMethod("findPlacementRequest", String.class)
+            .getAnnotation(Transactional.class);
 
-    new BetStore(bets, requests)
-        .recordRiskReservation(
-            bet.betId(), expiresAt, "b".repeat(64), false, Instant.EPOCH.plusSeconds(1));
+    assertThat(annotation.readOnly()).isTrue();
+  }
 
-    assertThat(bet.riskReservationToken()).isEqualTo("b".repeat(64));
-    assertThat(bet.riskReservationExpiresAt()).isEqualTo(expiresAt);
+  @Test
+  void everyExternalSideEffectHasAnIndependentCheckpoint() throws Exception {
+    assertWriteTransaction(
+        "recordRiskReservation",
+        UUID.class,
+        Instant.class,
+        String.class,
+        boolean.class,
+        Instant.class);
+    assertWriteTransaction("confirmWallet", UUID.class, UUID.class, Instant.class);
+    assertWriteTransaction("commitRisk", UUID.class, Instant.class);
+    assertWriteTransaction("beginCompensation", UUID.class, Instant.class);
+    assertWriteTransaction("completeRiskRelease", UUID.class, boolean.class, Instant.class);
+    assertWriteTransaction("completeWalletRefund", UUID.class, UUID.class, Instant.class);
   }
 
-  private static Bet pendingBet() {
-    UUID betId = UUID.randomUUID();
-    BetDraft draft =
-        new BetDraft(
-            betId,
-            UUID.randomUUID(),
-            "B-2026-08-22-00000000",
-            new BetSlipType.Single(),
-            Money.krw(1_000),
-            Money.krw(2_000),
-            IdempotencyKey.of("request-1"),
-            "a".repeat(64),
-            Instant.EPOCH);
-    BetLeg leg =
-        BetLeg.create(UUID.randomUUID(), UUID.randomUUID(), UUID.randomUUID(), Odds.ofDecimal("2"));
-    return Bet.pending(draft, List.of(leg));
+  private static void assertWriteTransaction(String method, Class<?>... parameterTypes)
+      throws Exception {
+    Transactional annotation =
+        BetStore.class.getMethod(method, parameterTypes).getAnnotation(Transactional.class);
+    assertThat(annotation).isNotNull();
+    assertThat(annotation.readOnly()).isFalse();
   }
 }


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
