## `test(placement): verify wallet debit ordering`

diff --git a/src/test/java/com/sportsbook/betting/placement/BetPlacementServiceTest.java b/src/test/java/com/sportsbook/betting/placement/BetPlacementServiceTest.java
index 71e01e3..07db39a 100644
--- a/src/test/java/com/sportsbook/betting/placement/BetPlacementServiceTest.java
+++ b/src/test/java/com/sportsbook/betting/placement/BetPlacementServiceTest.java
@@ -3,12 +3,14 @@ package com.sportsbook.betting.placement;
 import static org.assertj.core.api.Assertions.catchThrowableOfType;
 import static org.mockito.ArgumentMatchers.any;
 import static org.mockito.ArgumentMatchers.eq;
+import static org.mockito.Mockito.doAnswer;
 import static org.mockito.Mockito.inOrder;
 import static org.mockito.Mockito.mock;
 import static org.mockito.Mockito.verifyNoInteractions;
 import static org.mockito.Mockito.when;
 
 import com.sportsbook.betting.client.RiskClient;
+import com.sportsbook.betting.client.WalletClient;
 import com.sportsbook.betting.domain.Bet;
 import com.sportsbook.betting.domain.BetDraft;
 import com.sportsbook.betting.domain.BetLeg;
@@ -34,6 +36,7 @@ class BetPlacementServiceTest {
   void replaysOwnedVerdictBeforeRepeatingValidationOrSideEffects() {
     BetAssembler assembler = mock(BetAssembler.class);
     RiskClient risk = mock(RiskClient.class);
+    WalletClient wallet = mock(WalletClient.class);
     BetStore store = mock(BetStore.class);
     PlaceBetCommand command = command();
     PlacementRequest request =
@@ -45,21 +48,24 @@ class BetPlacementServiceTest {
             "saved verdict",
             Instant.EPOCH);
     when(store.findPlacementRequest("request-1")).thenReturn(Optional.of(request));
-    BetPlacementService service = service(assembler, risk, store);
 
-    catchThrowableOfType(() -> service.place(command), PersistedRejectionException.class);
+    catchThrowableOfType(
+        () -> service(assembler, risk, wallet, store).place(command),
+        PersistedRejectionException.class);
 
-    verifyNoInteractions(assembler, risk);
+    verifyNoInteractions(assembler, risk, wallet);
   }
 
   @Test
-  void storesReservationTokenBeforeReturningFromReserveStep() {
+  void persistsReservationProofBeforeWalletDebit() {
     BetAssembler assembler = mock(BetAssembler.class);
     RiskClient risk = mock(RiskClient.class);
+    WalletClient wallet = mock(WalletClient.class);
     BetStore store = mock(BetStore.class);
     PlaceBetCommand command = command();
     Bet bet = pending(command);
     Instant expiresAt = Instant.EPOCH.plusSeconds(60);
+    UUID operationId = UUID.randomUUID();
     when(store.findPlacementRequest("request-1")).thenReturn(Optional.empty());
     when(store.findByIdempotencyKey("request-1")).thenReturn(Optional.empty());
     when(assembler.assemble(eq(command), any(String.class))).thenReturn(bet);
@@ -68,20 +74,38 @@ class BetPlacementServiceTest {
         .thenReturn(
             new RiskClient.Reservation(
                 RiskClient.ReservationState.RESERVED, expiresAt, "b".repeat(64)));
-
-    service(assembler, risk, store).place(command);
-
-    InOrder order = inOrder(risk, store);
+    doAnswer(
+            ignored -> {
+              bet.recordRiskReservation(expiresAt, "b".repeat(64), false, Instant.EPOCH);
+              return null;
+            })
+        .when(store)
+        .recordRiskReservation(bet.betId(), expiresAt, "b".repeat(64), false, Instant.EPOCH);
+    when(wallet.debit(bet.betId(), bet.userId(), Money.krw(1_000))).thenReturn(operationId);
+    doAnswer(
+            ignored -> {
+              bet.confirmWallet(operationId, Instant.EPOCH);
+              return null;
+            })
+        .when(store)
+        .confirmWallet(bet.betId(), operationId, Instant.EPOCH);
+
+    service(assembler, risk, wallet, store).place(command);
+
+    InOrder order = inOrder(risk, store, wallet);
     order.verify(risk).reserve(eq(bet.betId()), eq(bet.userId()), eq(Money.krw(1_000)), any());
     order
         .verify(store)
         .recordRiskReservation(bet.betId(), expiresAt, "b".repeat(64), false, Instant.EPOCH);
+    order.verify(wallet).debit(bet.betId(), bet.userId(), Money.krw(1_000));
+    order.verify(store).confirmWallet(bet.betId(), operationId, Instant.EPOCH);
   }
 
   @Test
   void persistsRiskValidationAfterPendingCreation() {
     BetAssembler assembler = mock(BetAssembler.class);
     RiskClient risk = mock(RiskClient.class);
+    WalletClient wallet = mock(WalletClient.class);
     BetStore store = mock(BetStore.class);
     PlaceBetCommand command = command();
     Bet bet = pending(command);
@@ -92,18 +116,20 @@ class BetPlacementServiceTest {
     when(store.rejectAtCreation(any(), any(), any(), any())).thenReturn(bet);
 
     catchThrowableOfType(
-        () -> service(assembler, risk, store).place(command),
+        () -> service(assembler, risk, wallet, store).place(command),
         com.sportsbook.betting.error.ValidationFailedException.class);
 
     org.mockito.Mockito.verify(store)
         .rejectAtCreation(bet.betId(), ErrorCode.VALIDATION_FAILED, "invalid", Instant.EPOCH);
+    verifyNoInteractions(wallet);
   }
 
   private static BetPlacementService service(
-      BetAssembler assembler, RiskClient risk, BetStore store) {
+      BetAssembler assembler, RiskClient risk, WalletClient wallet, BetStore store) {
     return new BetPlacementService(
         assembler,
         risk,
+        wallet,
         new SystemBetCalculator(),
         store,
         Clock.fixed(Instant.EPOCH, ZoneOffset.UTC));
@@ -119,7 +145,8 @@ class BetPlacementServiceTest {
             new com.sportsbook.betting.error.ValidationFailedException(
                 "Duplicate selection is not allowed"));
 
-    BetPlacementService service = service(assembler, mock(RiskClient.class), store);
+    BetPlacementService service =
+        service(assembler, mock(RiskClient.class), mock(WalletClient.class), store);
     catchThrowableOfType(
         () -> service.place(command),
         com.sportsbook.betting.error.ValidationFailedException.class);


## `feat(placement): commit risk before atomic acceptance`

diff --git a/src/main/java/com/sportsbook/betting/placement/BetPlacementService.java b/src/main/java/com/sportsbook/betting/placement/BetPlacementService.java
index 0a24ee0..974f607 100644
--- a/src/main/java/com/sportsbook/betting/placement/BetPlacementService.java
+++ b/src/main/java/com/sportsbook/betting/placement/BetPlacementService.java
@@ -17,7 +17,11 @@ import com.sportsbook.betting.error.OddsDriftException;
 import com.sportsbook.betting.error.RiskLimitException;
 import com.sportsbook.betting.error.ValidationFailedException;
 import com.sportsbook.betting.error.WalletRejectedException;
+import com.sportsbook.betting.outbox.BetEventFactory;
+import com.sportsbook.betting.outbox.OutboxEvent;
 import com.sportsbook.protocol.domain.BetStatus;
+import com.sportsbook.protocol.error.ErrorCode;
+import com.sportsbook.protocol.value.IdempotencyKey;
 import java.time.Clock;
 import java.util.List;
 import java.util.Optional;
@@ -34,6 +38,8 @@ public class BetPlacementService {
   private final RiskClient risk;
   private final WalletClient wallet;
   private final SystemBetCalculator calculator;
+  private final BetEventFactory events;
+  private final IdempotencyCache idempotency;
   private final BetStore store;
   private final Clock clock;
 
@@ -42,12 +48,16 @@ public class BetPlacementService {
       RiskClient risk,
       WalletClient wallet,
       SystemBetCalculator calculator,
+      BetEventFactory events,
+      IdempotencyCache idempotency,
       BetStore store,
       Clock clock) {
     this.assembler = assembler;
     this.risk = risk;
     this.wallet = wallet;
     this.calculator = calculator;
+    this.events = events;
+    this.idempotency = idempotency;
     this.store = store;
     this.clock = clock;
   }
@@ -106,8 +116,9 @@ public class BetPlacementService {
         switch (current.placementPhase()) {
           case CREATED -> reserveRisk(current, surfaceRejection);
           case RISK_RESERVED -> confirmWallet(current, recovery);
-          default -> {
-            return current;
+          case WALLET_CONFIRMED -> commitRisk(current);
+          case RISK_COMMITTED -> {
+            return accept(current);
           }
         }
       } catch (DependencyUnavailableException unavailable) {
@@ -136,6 +147,31 @@ public class BetPlacementService {
     }
   }
 
+  private void commitRisk(Bet bet) {
+    if (bet.riskCommitObserved()) {
+      store.commitRisk(bet.betId(), clock.instant());
+      return;
+    }
+    RiskClient.CommitResult result = risk.commit(bet.betId(), bet.riskReservationToken());
+    if (result == RiskClient.CommitResult.COMMITTED) {
+      store.commitRisk(bet.betId(), clock.instant());
+      return;
+    }
+    ErrorCode code =
+        result == RiskClient.CommitResult.CONFLICT
+            ? ErrorCode.DUPLICATE_BET
+            : ErrorCode.LIMIT_EXCEEDED;
+    store.requireWalletRefund(
+        bet.betId(), code, "Risk reservation commit failed: " + result, clock.instant());
+  }
+
+  private Bet accept(Bet bet) {
+    OutboxEvent event = events.placedRequested(bet, clock.instant());
+    Bet accepted = store.acceptAndEnqueue(bet.betId(), event, clock.instant());
+    idempotency.markProcessed(IdempotencyKey.of(accepted.idempotencyKey()), accepted.betId());
+    return accepted;
+  }
+
   private void reserveRisk(Bet bet, boolean surfaceRejection) {
     try {
       Reservation reservation =


## `test(placement): verify risk commit acceptance order`

diff --git a/src/test/java/com/sportsbook/betting/placement/BetPlacementServiceTest.java b/src/test/java/com/sportsbook/betting/placement/BetPlacementServiceTest.java
index 07db39a..264d33a 100644
--- a/src/test/java/com/sportsbook/betting/placement/BetPlacementServiceTest.java
+++ b/src/test/java/com/sportsbook/betting/placement/BetPlacementServiceTest.java
@@ -16,6 +16,8 @@ import com.sportsbook.betting.domain.BetDraft;
 import com.sportsbook.betting.domain.BetLeg;
 import com.sportsbook.betting.domain.SystemBetCalculator;
 import com.sportsbook.betting.error.PersistedRejectionException;
+import com.sportsbook.betting.outbox.BetEventFactory;
+import com.sportsbook.betting.outbox.OutboxEvent;
 import com.sportsbook.protocol.domain.BetSlipType;
 import com.sportsbook.protocol.error.ErrorCode;
 import com.sportsbook.protocol.value.IdempotencyKey;
@@ -37,6 +39,8 @@ class BetPlacementServiceTest {
     BetAssembler assembler = mock(BetAssembler.class);
     RiskClient risk = mock(RiskClient.class);
     WalletClient wallet = mock(WalletClient.class);
+    BetEventFactory events = mock(BetEventFactory.class);
+    IdempotencyCache idempotency = mock(IdempotencyCache.class);
     BetStore store = mock(BetStore.class);
     PlaceBetCommand command = command();
     PlacementRequest request =
@@ -50,7 +54,7 @@ class BetPlacementServiceTest {
     when(store.findPlacementRequest("request-1")).thenReturn(Optional.of(request));
 
     catchThrowableOfType(
-        () -> service(assembler, risk, wallet, store).place(command),
+        () -> service(assembler, risk, wallet, events, idempotency, store).place(command),
         PersistedRejectionException.class);
 
     verifyNoInteractions(assembler, risk, wallet);
@@ -61,6 +65,8 @@ class BetPlacementServiceTest {
     BetAssembler assembler = mock(BetAssembler.class);
     RiskClient risk = mock(RiskClient.class);
     WalletClient wallet = mock(WalletClient.class);
+    BetEventFactory events = mock(BetEventFactory.class);
+    IdempotencyCache idempotency = mock(IdempotencyCache.class);
     BetStore store = mock(BetStore.class);
     PlaceBetCommand command = command();
     Bet bet = pending(command);
@@ -89,8 +95,32 @@ class BetPlacementServiceTest {
             })
         .when(store)
         .confirmWallet(bet.betId(), operationId, Instant.EPOCH);
+    when(risk.commit(bet.betId(), "b".repeat(64))).thenReturn(RiskClient.CommitResult.COMMITTED);
+    doAnswer(
+            ignored -> {
+              bet.commitRisk(Instant.EPOCH);
+              return null;
+            })
+        .when(store)
+        .commitRisk(bet.betId(), Instant.EPOCH);
+    OutboxEvent event =
+        OutboxEvent.pending(
+            UUID.randomUUID(),
+            "bet.placed.v1",
+            bet.userId().toString(),
+            "schema",
+            new byte[] {1},
+            Instant.EPOCH);
+    when(events.placedRequested(bet, Instant.EPOCH)).thenReturn(event);
+    doAnswer(
+            ignored -> {
+              bet.accept(Instant.EPOCH);
+              return bet;
+            })
+        .when(store)
+        .acceptAndEnqueue(bet.betId(), event, Instant.EPOCH);
 
-    service(assembler, risk, wallet, store).place(command);
+    service(assembler, risk, wallet, events, idempotency, store).place(command);
 
     InOrder order = inOrder(risk, store, wallet);
     order.verify(risk).reserve(eq(bet.betId()), eq(bet.userId()), eq(Money.krw(1_000)), any());
@@ -99,6 +129,12 @@ class BetPlacementServiceTest {
         .recordRiskReservation(bet.betId(), expiresAt, "b".repeat(64), false, Instant.EPOCH);
     order.verify(wallet).debit(bet.betId(), bet.userId(), Money.krw(1_000));
     order.verify(store).confirmWallet(bet.betId(), operationId, Instant.EPOCH);
+    order.verify(risk).commit(bet.betId(), "b".repeat(64));
+    order.verify(store).commitRisk(bet.betId(), Instant.EPOCH);
+    order.verify(store).acceptAndEnqueue(bet.betId(), event, Instant.EPOCH);
+    InOrder completion = inOrder(store, idempotency);
+    completion.verify(store).acceptAndEnqueue(bet.betId(), event, Instant.EPOCH);
+    completion.verify(idempotency).markProcessed(IdempotencyKey.of("request-1"), bet.betId());
   }
 
   @Test
@@ -125,12 +161,19 @@ class BetPlacementServiceTest {
   }
 
   private static BetPlacementService service(
-      BetAssembler assembler, RiskClient risk, WalletClient wallet, BetStore store) {
+      BetAssembler assembler,
+      RiskClient risk,
+      WalletClient wallet,
+      BetEventFactory events,
+      IdempotencyCache idempotency,
+      BetStore store) {
     return new BetPlacementService(
         assembler,
         risk,
         wallet,
         new SystemBetCalculator(),
+        events,
+        idempotency,
         store,
         Clock.fixed(Instant.EPOCH, ZoneOffset.UTC));
   }


## `feat(placement): compensate definitive dependency failures`

diff --git a/src/main/java/com/sportsbook/betting/placement/BetPlacementService.java b/src/main/java/com/sportsbook/betting/placement/BetPlacementService.java
index 974f607..53158ca 100644
--- a/src/main/java/com/sportsbook/betting/placement/BetPlacementService.java
+++ b/src/main/java/com/sportsbook/betting/placement/BetPlacementService.java
@@ -6,6 +6,7 @@ import com.sportsbook.betting.client.WalletClient;
 import com.sportsbook.betting.client.WalletOperationResponse;
 import com.sportsbook.betting.domain.Bet;
 import com.sportsbook.betting.domain.BetLeg;
+import com.sportsbook.betting.domain.CompensationAction;
 import com.sportsbook.betting.domain.CompensationState;
 import com.sportsbook.betting.domain.SystemBetCalculator;
 import com.sportsbook.betting.error.BetPlacementException;
@@ -109,10 +110,18 @@ public class BetPlacementService {
       if (current.status() != BetStatus.PENDING) {
         return current;
       }
-      if (current.compensationState() != CompensationState.NONE) {
-        return current;
-      }
       try {
+        if (current.compensationState() != CompensationState.NONE) {
+          switch (current.compensationState()) {
+            case REQUIRED -> store.beginCompensation(betId, clock.instant());
+            case IN_PROGRESS -> performCompensation(current);
+            case COMPLETED -> {
+              return finishCompensatedRejection(current, surfaceRejection);
+            }
+            case NONE -> throw new IllegalStateException("Unreachable compensation state");
+          }
+          continue;
+        }
         switch (current.placementPhase()) {
           case CREATED -> reserveRisk(current, surfaceRejection);
           case RISK_RESERVED -> confirmWallet(current, recovery);
@@ -128,6 +137,10 @@ public class BetPlacementService {
     return store.findById(betId);
   }
 
+  Bet reconcile(UUID betId) {
+    return advance(betId, true, false);
+  }
+
   private void confirmWallet(Bet bet, boolean recovery) {
     try {
       UUID operationId =
@@ -172,6 +185,29 @@ public class BetPlacementService {
     return accepted;
   }
 
+  private void performCompensation(Bet bet) {
+    if (bet.compensationAction() == CompensationAction.RISK_RELEASE) {
+      RiskClient.ReleaseResult result = risk.release(bet.betId());
+      store.completeRiskRelease(
+          bet.betId(), result == RiskClient.ReleaseResult.COMMITTED, clock.instant());
+      return;
+    }
+    if (bet.compensationAction() == CompensationAction.WALLET_REFUND) {
+      UUID operationId = wallet.refund(bet.betId(), bet.userId(), totalExposure(bet));
+      store.completeWalletRefund(bet.betId(), operationId, clock.instant());
+      return;
+    }
+    throw new IllegalStateException("PENDING compensation has no action");
+  }
+
+  private Bet finishCompensatedRejection(Bet bet, boolean surfaceRejection) {
+    Bet rejected = store.rejectAfterCompensation(bet.betId(), clock.instant());
+    if (surfaceRejection) {
+      return PlacementReplay.bet(rejected, rejected.userId(), rejected.requestFingerprint());
+    }
+    return rejected;
+  }
+
   private void reserveRisk(Bet bet, boolean surfaceRejection) {
     try {
       Reservation reservation =


## `test(placement): verify definitive compensation paths`

diff --git a/src/test/java/com/sportsbook/betting/placement/BetPlacementServiceTest.java b/src/test/java/com/sportsbook/betting/placement/BetPlacementServiceTest.java
index 264d33a..b455def 100644
--- a/src/test/java/com/sportsbook/betting/placement/BetPlacementServiceTest.java
+++ b/src/test/java/com/sportsbook/betting/placement/BetPlacementServiceTest.java
@@ -1,5 +1,6 @@
 package com.sportsbook.betting.placement;
 
+import static org.assertj.core.api.Assertions.assertThat;
 import static org.assertj.core.api.Assertions.catchThrowableOfType;
 import static org.mockito.ArgumentMatchers.any;
 import static org.mockito.ArgumentMatchers.eq;
@@ -19,6 +20,7 @@ import com.sportsbook.betting.error.PersistedRejectionException;
 import com.sportsbook.betting.outbox.BetEventFactory;
 import com.sportsbook.betting.outbox.OutboxEvent;
 import com.sportsbook.protocol.domain.BetSlipType;
+import com.sportsbook.protocol.domain.BetStatus;
 import com.sportsbook.protocol.error.ErrorCode;
 import com.sportsbook.protocol.value.IdempotencyKey;
 import com.sportsbook.protocol.value.Money;
@@ -152,7 +154,9 @@ class BetPlacementServiceTest {
     when(store.rejectAtCreation(any(), any(), any(), any())).thenReturn(bet);
 
     catchThrowableOfType(
-        () -> service(assembler, risk, wallet, store).place(command),
+        () ->
+            service(assembler, risk, wallet, mock(), mock(), store)
+                .place(command),
         com.sportsbook.betting.error.ValidationFailedException.class);
 
     org.mockito.Mockito.verify(store)
@@ -160,6 +164,69 @@ class BetPlacementServiceTest {
     verifyNoInteractions(wallet);
   }
 
+  @Test
+  void refundsWithoutRereservingWhenRiskCommitIsMissing() {
+    BetAssembler assembler = mock(BetAssembler.class);
+    RiskClient risk = mock(RiskClient.class);
+    WalletClient wallet = mock(WalletClient.class);
+    BetEventFactory events = mock(BetEventFactory.class);
+    IdempotencyCache idempotency = mock(IdempotencyCache.class);
+    BetStore store = mock(BetStore.class);
+    Bet bet = pending(command());
+    bet.recordRiskReservation(Instant.EPOCH.plusSeconds(60), "c".repeat(64), false, Instant.EPOCH);
+    bet.confirmWallet(UUID.randomUUID(), Instant.EPOCH);
+    when(store.findById(bet.betId())).thenReturn(bet);
+    when(risk.commit(bet.betId(), "c".repeat(64))).thenReturn(RiskClient.CommitResult.NOT_FOUND);
+    doAnswer(
+            ignored -> {
+              bet.requireWalletRefund("LIMIT_EXCEEDED", "missing reservation", Instant.EPOCH);
+              return null;
+            })
+        .when(store)
+        .requireWalletRefund(
+            bet.betId(),
+            ErrorCode.LIMIT_EXCEEDED,
+            "Risk reservation commit failed: NOT_FOUND",
+            Instant.EPOCH);
+    checkpointCompensation(store, bet);
+    UUID refundId = UUID.randomUUID();
+    when(wallet.refund(bet.betId(), bet.userId(), Money.krw(1_000))).thenReturn(refundId);
+    doAnswer(
+            ignored -> {
+              bet.completeWalletRefund(refundId, Instant.EPOCH);
+              return null;
+            })
+        .when(store)
+        .completeWalletRefund(bet.betId(), refundId, Instant.EPOCH);
+    rejectAfterCompensation(store, bet);
+    Bet result =
+        service(assembler, risk, wallet, events, idempotency, store).reconcile(bet.betId());
+
+    assertThat(result.status()).isEqualTo(BetStatus.REJECTED);
+    org.mockito.Mockito.verify(risk, org.mockito.Mockito.never())
+        .reserve(any(), any(), any(), any());
+  }
+
+  private static void checkpointCompensation(BetStore store, Bet bet) {
+    doAnswer(
+            ignored -> {
+              bet.beginCompensation(Instant.EPOCH);
+              return null;
+            })
+        .when(store)
+        .beginCompensation(bet.betId(), Instant.EPOCH);
+  }
+
+  private static void rejectAfterCompensation(BetStore store, Bet bet) {
+    doAnswer(
+            ignored -> {
+              bet.rejectAfterCompensation(Instant.EPOCH);
+              return bet;
+            })
+        .when(store)
+        .rejectAfterCompensation(bet.betId(), Instant.EPOCH);
+  }
+
   private static BetPlacementService service(
       BetAssembler assembler,
       RiskClient risk,
@@ -189,7 +256,13 @@ class BetPlacementServiceTest {
                 "Duplicate selection is not allowed"));
 
     BetPlacementService service =
-        service(assembler, mock(RiskClient.class), mock(WalletClient.class), store);
+        service(
+            assembler,
+            mock(RiskClient.class),
+            mock(WalletClient.class),
+            mock(BetEventFactory.class),
+            mock(IdempotencyCache.class),
+            store);
     catchThrowableOfType(
         () -> service.place(command),
         com.sportsbook.betting.error.ValidationFailedException.class);


