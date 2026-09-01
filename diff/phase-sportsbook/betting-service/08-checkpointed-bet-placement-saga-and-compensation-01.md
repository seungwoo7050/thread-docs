# 체크포인트 기반 베팅 접수 사가와 보상

## `feat(placement): define durable saga phases`

diff --git a/src/main/java/com/sportsbook/betting/domain/PlacementPhase.java b/src/main/java/com/sportsbook/betting/domain/PlacementPhase.java
new file mode 100644
index 0000000..842fbf0
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/domain/PlacementPhase.java
@@ -0,0 +1,12 @@
+package com.sportsbook.betting.domain;
+
+public enum PlacementPhase {
+  CREATED,
+  RISK_RESERVED,
+  WALLET_CONFIRMED,
+  RISK_COMMITTED;
+
+  public boolean precedes(PlacementPhase other) {
+    return ordinal() < other.ordinal();
+  }
+}


## `feat(placement): model compensation progress`

diff --git a/src/main/java/com/sportsbook/betting/domain/CompensationAction.java b/src/main/java/com/sportsbook/betting/domain/CompensationAction.java
new file mode 100644
index 0000000..8736241
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/domain/CompensationAction.java
@@ -0,0 +1,7 @@
+package com.sportsbook.betting.domain;
+
+public enum CompensationAction {
+  NONE,
+  RISK_RELEASE,
+  WALLET_REFUND
+}
diff --git a/src/main/java/com/sportsbook/betting/domain/CompensationState.java b/src/main/java/com/sportsbook/betting/domain/CompensationState.java
new file mode 100644
index 0000000..bbc5f83
--- /dev/null
+++ b/src/main/java/com/sportsbook/betting/domain/CompensationState.java
@@ -0,0 +1,12 @@
+package com.sportsbook.betting.domain;
+
+public enum CompensationState {
+  NONE,
+  REQUIRED,
+  IN_PROGRESS,
+  COMPLETED;
+
+  public boolean pending() {
+    return this == REQUIRED || this == IN_PROGRESS;
+  }
+}


## `feat(database): add placement recovery evidence`

diff --git a/src/main/resources/db/migration/V5__placement_recovery.sql b/src/main/resources/db/migration/V5__placement_recovery.sql
new file mode 100644
index 0000000..018e5df
--- /dev/null
+++ b/src/main/resources/db/migration/V5__placement_recovery.sql
@@ -0,0 +1,28 @@
+-- Durable checkpoints and rejection replay data for the recoverable placement saga.
+ALTER TABLE bet
+    ADD COLUMN request_fingerprint           VARCHAR(64),
+    ADD COLUMN placement_phase               VARCHAR(32) NOT NULL DEFAULT 'CREATED',
+    ADD COLUMN rejection_detail              VARCHAR(1024),
+    ADD COLUMN risk_reservation_expires_at   TIMESTAMP WITH TIME ZONE,
+    ADD COLUMN risk_commit_observed          BOOLEAN NOT NULL DEFAULT FALSE,
+    ADD COLUMN wallet_operation_id           UUID;
+
+-- Existing terminal bets predate the saga checkpoints. They have already completed all placement
+-- side effects, while old rejected/PENDING rows remain at the conservative CREATED phase. The
+-- nullable fingerprint deliberately permits safe actor-only replay of pre-migration records.
+UPDATE bet
+SET placement_phase = 'RISK_COMMITTED',
+    risk_commit_observed = TRUE
+WHERE status IN ('ACCEPTED', 'SETTLED', 'CANCELLED', 'VOIDED');
+
+ALTER TABLE bet
+    ADD CONSTRAINT bet_placement_phase_valid CHECK (
+        placement_phase IN ('CREATED', 'RISK_RESERVED', 'WALLET_CONFIRMED', 'RISK_COMMITTED')
+    );
+
+COMMENT ON COLUMN bet.request_fingerprint IS
+    'SHA-256 of the canonical placement request; prevents Idempotency-Key payload reuse.';
+COMMENT ON COLUMN bet.placement_phase IS
+    'Durable CREATED -> RISK_RESERVED -> WALLET_CONFIRMED -> RISK_COMMITTED checkpoint.';
+COMMENT ON COLUMN bet.rejection_detail IS
+    'Original definitive rejection detail replayed through RFC 7807.';


## `feat(database): add compensation verdict schema`

diff --git a/src/main/resources/db/migration/V6__placement_compensation_and_verdict.sql b/src/main/resources/db/migration/V6__placement_compensation_and_verdict.sql
new file mode 100644
index 0000000..00b1db3
--- /dev/null
+++ b/src/main/resources/db/migration/V6__placement_compensation_and_verdict.sql
@@ -0,0 +1,79 @@
+-- Irreversible compensation checkpoints and one Idempotency-Key namespace for both bets and
+-- preflight rejections.
+ALTER TABLE bet
+    ADD COLUMN compensation_action       VARCHAR(24),
+    ADD COLUMN compensation_state        VARCHAR(16) NOT NULL DEFAULT 'NONE',
+    ADD COLUMN compensation_operation_id UUID;
+
+ALTER TABLE bet
+    ADD CONSTRAINT bet_compensation_valid CHECK (
+        (compensation_state = 'NONE'
+            AND compensation_action IS NULL
+            AND compensation_operation_id IS NULL)
+        OR
+        (compensation_state IN ('REQUIRED', 'IN_PROGRESS')
+            AND compensation_action IN ('RISK_RELEASE', 'WALLET_REFUND')
+            AND compensation_operation_id IS NULL)
+        OR
+        (compensation_state = 'COMPLETED'
+            AND compensation_action = 'RISK_RELEASE'
+            AND compensation_operation_id IS NULL)
+        OR
+        (compensation_state = 'COMPLETED'
+            AND compensation_action = 'WALLET_REFUND'
+            AND compensation_operation_id IS NOT NULL)
+    );
+
+COMMENT ON COLUMN bet.compensation_action IS
+    'Irreversible rollback branch: release the risk reservation or refund the wallet debit.';
+COMMENT ON COLUMN bet.compensation_state IS
+    'Durable NONE -> REQUIRED -> IN_PROGRESS -> COMPLETED compensation checkpoint.';
+COMMENT ON COLUMN bet.compensation_operation_id IS
+    'Wallet operation proof for a completed WALLET_REFUND; null for risk release.';
+
+CREATE TABLE placement_request (
+    idempotency_key    VARCHAR(128)             PRIMARY KEY,
+    user_id            UUID                     NOT NULL,
+    request_fingerprint VARCHAR(64),
+    outcome            VARCHAR(16)              NOT NULL,
+    bet_id             UUID REFERENCES bet (bet_id) ON DELETE CASCADE,
+    error_code         VARCHAR(64),
+    error_detail       VARCHAR(1024),
+    created_at         TIMESTAMP WITH TIME ZONE NOT NULL,
+    CONSTRAINT uk_placement_request_bet UNIQUE (bet_id),
+    CONSTRAINT placement_request_outcome_valid CHECK (
+        (outcome = 'BET'
+            AND bet_id IS NOT NULL
+            AND error_code IS NULL
+            AND error_detail IS NULL)
+        OR
+        (outcome = 'REJECTION'
+            AND bet_id IS NULL
+            AND error_code IS NOT NULL
+            AND error_detail IS NOT NULL)
+    )
+);
+
+-- Existing rows already own their Idempotency-Key. Rejected rows with a Bet aggregate remain BET
+-- outcomes; their original error is replayed from bet.rejection_reason/rejection_detail.
+INSERT INTO placement_request (
+    idempotency_key,
+    user_id,
+    request_fingerprint,
+    outcome,
+    bet_id,
+    created_at
+)
+SELECT
+    idempotency_key,
+    user_id,
+    request_fingerprint,
+    'BET',
+    bet_id,
+    created_at
+FROM bet;
+
+COMMENT ON TABLE placement_request IS
+    'Authoritative placement Idempotency-Key outcome: a bet pointer or durable preflight rejection.';
+COMMENT ON COLUMN placement_request.request_fingerprint IS
+    'Canonical request SHA-256; nullable only for legacy rows migrated from bet.';


## `feat(placement): checkpoint external side effects`

diff --git a/src/main/java/com/sportsbook/betting/placement/BetStore.java b/src/main/java/com/sportsbook/betting/placement/BetStore.java
index 78e3f2d..ad86c5b 100644
--- a/src/main/java/com/sportsbook/betting/placement/BetStore.java
+++ b/src/main/java/com/sportsbook/betting/placement/BetStore.java
@@ -3,6 +3,7 @@ package com.sportsbook.betting.placement;
 import com.sportsbook.betting.domain.Bet;
 import com.sportsbook.betting.persistence.BetRepository;
 import com.sportsbook.betting.persistence.PlacementRequestRepository;
+import com.sportsbook.protocol.domain.BetStatus;
 import com.sportsbook.protocol.error.ErrorCode;
 import java.time.Instant;
 import java.util.Optional;
@@ -48,4 +49,56 @@ public class BetStore {
       String key, UUID userId, String fingerprint, ErrorCode code, String detail, Instant now) {
     requests.saveAndFlush(PlacementRequest.rejected(key, userId, fingerprint, code, detail, now));
   }
+
+  @Transactional
+  public void recordRiskReservation(
+      UUID betId, Instant expiresAt, String token, boolean alreadyCommitted, Instant now) {
+    pending(betId).recordRiskReservation(expiresAt, token, alreadyCommitted, now);
+  }
+
+  @Transactional
+  public void confirmWallet(UUID betId, UUID operationId, Instant now) {
+    pending(betId).confirmWallet(operationId, now);
+  }
+
+  @Transactional
+  public void commitRisk(UUID betId, Instant now) {
+    pending(betId).commitRisk(now);
+  }
+
+  @Transactional
+  public void requireRiskRelease(UUID betId, ErrorCode reason, String detail, Instant now) {
+    pending(betId).requireRiskRelease(reason.name(), detail, now);
+  }
+
+  @Transactional
+  public void requireWalletRefund(UUID betId, ErrorCode reason, String detail, Instant now) {
+    pending(betId).requireWalletRefund(reason.name(), detail, now);
+  }
+
+  @Transactional
+  public void beginCompensation(UUID betId, Instant now) {
+    pending(betId).beginCompensation(now);
+  }
+
+  @Transactional
+  public void completeRiskRelease(UUID betId, boolean committedConflict, Instant now) {
+    pending(betId).completeRiskRelease(committedConflict, now);
+  }
+
+  @Transactional
+  public void completeWalletRefund(UUID betId, UUID operationId, Instant now) {
+    pending(betId).completeWalletRefund(operationId, now);
+  }
+
+  private Bet pending(UUID betId) {
+    Bet bet =
+        bets.findLockedByBetId(betId)
+            .orElseThrow(
+                () -> new IllegalStateException("Bet vanished during placement: " + betId));
+    if (bet.status() != BetStatus.PENDING) {
+      throw new IllegalStateException("Placement cannot update terminal bet " + betId);
+    }
+    return bet;
+  }
 }


## `feat(placement): reserve exposure with durable proof`

diff --git a/src/main/java/com/sportsbook/betting/placement/BetPlacementService.java b/src/main/java/com/sportsbook/betting/placement/BetPlacementService.java
index 9d1bdef..1446ce4 100644
--- a/src/main/java/com/sportsbook/betting/placement/BetPlacementService.java
+++ b/src/main/java/com/sportsbook/betting/placement/BetPlacementService.java
@@ -1,12 +1,21 @@
 package com.sportsbook.betting.placement;
 
+import com.sportsbook.betting.client.RiskClient;
+import com.sportsbook.betting.client.RiskClient.Reservation;
 import com.sportsbook.betting.domain.Bet;
+import com.sportsbook.betting.domain.BetLeg;
+import com.sportsbook.betting.domain.PlacementPhase;
+import com.sportsbook.betting.domain.SystemBetCalculator;
 import com.sportsbook.betting.error.BetPlacementException;
+import com.sportsbook.betting.error.DuplicateBetException;
 import com.sportsbook.betting.error.MarketClosedException;
 import com.sportsbook.betting.error.OddsDriftException;
+import com.sportsbook.betting.error.RiskLimitException;
 import com.sportsbook.betting.error.ValidationFailedException;
 import java.time.Clock;
+import java.util.List;
 import java.util.Optional;
+import java.util.UUID;
 import org.springframework.dao.DataIntegrityViolationException;
 import org.springframework.stereotype.Service;
 
@@ -14,11 +23,20 @@ import org.springframework.stereotype.Service;
 public class BetPlacementService {
 
   private final BetAssembler assembler;
+  private final RiskClient risk;
+  private final SystemBetCalculator calculator;
   private final BetStore store;
   private final Clock clock;
 
-  public BetPlacementService(BetAssembler assembler, BetStore store, Clock clock) {
+  public BetPlacementService(
+      BetAssembler assembler,
+      RiskClient risk,
+      SystemBetCalculator calculator,
+      BetStore store,
+      Clock clock) {
     this.assembler = assembler;
+    this.risk = risk;
+    this.calculator = calculator;
     this.store = store;
     this.clock = clock;
   }
@@ -61,7 +79,47 @@ public class BetPlacementService {
     } catch (DataIntegrityViolationException collision) {
       return replayKnown(key, command, fingerprint, collision);
     }
-    return bet;
+    return advance(bet.betId(), true);
+  }
+
+  private Bet advance(UUID betId, boolean surfaceRejection) {
+    Bet current = store.findById(betId);
+    if (current.placementPhase() == PlacementPhase.CREATED) {
+      reserveRisk(current, surfaceRejection);
+      return store.findById(betId);
+    }
+    return current;
+  }
+
+  private void reserveRisk(Bet bet, boolean surfaceRejection) {
+    try {
+      Reservation reservation =
+          risk.reserve(bet.betId(), bet.userId(), totalExposure(bet), selectionIds(bet.legs()));
+      store.recordRiskReservation(
+          bet.betId(),
+          reservation.expiresAt(),
+          reservation.token(),
+          reservation.alreadyCommitted(),
+          clock.instant());
+    } catch (RiskLimitException | DuplicateBetException | ValidationFailedException rejection) {
+      Bet rejected =
+          store.rejectAtCreation(
+              bet.betId(), rejection.errorCode(), rejection.getMessage(), clock.instant());
+      if (surfaceRejection) {
+        throw rejection;
+      }
+      if (rejected.status() == com.sportsbook.protocol.domain.BetStatus.PENDING) {
+        throw new IllegalStateException("Risk rejection was not persisted");
+      }
+    }
+  }
+
+  private com.sportsbook.protocol.value.Money totalExposure(Bet bet) {
+    return calculator.totalStake(bet.slipType(), bet.stake(), bet.legs().size());
+  }
+
+  private static List<UUID> selectionIds(List<BetLeg> legs) {
+    return legs.stream().map(BetLeg::selectionId).toList();
   }
 
   private Bet replayKnown(


## `test(placement): verify reservation proof ordering`

diff --git a/src/test/java/com/sportsbook/betting/placement/BetPlacementServiceTest.java b/src/test/java/com/sportsbook/betting/placement/BetPlacementServiceTest.java
index 27a3e0c..71e01e3 100644
--- a/src/test/java/com/sportsbook/betting/placement/BetPlacementServiceTest.java
+++ b/src/test/java/com/sportsbook/betting/placement/BetPlacementServiceTest.java
@@ -3,10 +3,16 @@ package com.sportsbook.betting.placement;
 import static org.assertj.core.api.Assertions.catchThrowableOfType;
 import static org.mockito.ArgumentMatchers.any;
 import static org.mockito.ArgumentMatchers.eq;
+import static org.mockito.Mockito.inOrder;
 import static org.mockito.Mockito.mock;
 import static org.mockito.Mockito.verifyNoInteractions;
 import static org.mockito.Mockito.when;
 
+import com.sportsbook.betting.client.RiskClient;
+import com.sportsbook.betting.domain.Bet;
+import com.sportsbook.betting.domain.BetDraft;
+import com.sportsbook.betting.domain.BetLeg;
+import com.sportsbook.betting.domain.SystemBetCalculator;
 import com.sportsbook.betting.error.PersistedRejectionException;
 import com.sportsbook.protocol.domain.BetSlipType;
 import com.sportsbook.protocol.error.ErrorCode;
@@ -20,30 +26,87 @@ import java.util.List;
 import java.util.Optional;
 import java.util.UUID;
 import org.junit.jupiter.api.Test;
+import org.mockito.InOrder;
 
 class BetPlacementServiceTest {
 
   @Test
   void replaysOwnedVerdictBeforeRepeatingValidationOrSideEffects() {
     BetAssembler assembler = mock(BetAssembler.class);
+    RiskClient risk = mock(RiskClient.class);
     BetStore store = mock(BetStore.class);
     PlaceBetCommand command = command();
-    String fingerprint = RequestFingerprint.of(command);
     PlacementRequest request =
         PlacementRequest.rejected(
             "request-1",
             command.userId(),
-            fingerprint,
+            RequestFingerprint.of(command),
             ErrorCode.VALIDATION_FAILED,
             "saved verdict",
             Instant.EPOCH);
     when(store.findPlacementRequest("request-1")).thenReturn(Optional.of(request));
-    BetPlacementService service =
-        new BetPlacementService(assembler, store, Clock.fixed(Instant.EPOCH, ZoneOffset.UTC));
+    BetPlacementService service = service(assembler, risk, store);
 
     catchThrowableOfType(() -> service.place(command), PersistedRejectionException.class);
 
-    verifyNoInteractions(assembler);
+    verifyNoInteractions(assembler, risk);
+  }
+
+  @Test
+  void storesReservationTokenBeforeReturningFromReserveStep() {
+    BetAssembler assembler = mock(BetAssembler.class);
+    RiskClient risk = mock(RiskClient.class);
+    BetStore store = mock(BetStore.class);
+    PlaceBetCommand command = command();
+    Bet bet = pending(command);
+    Instant expiresAt = Instant.EPOCH.plusSeconds(60);
+    when(store.findPlacementRequest("request-1")).thenReturn(Optional.empty());
+    when(store.findByIdempotencyKey("request-1")).thenReturn(Optional.empty());
+    when(assembler.assemble(eq(command), any(String.class))).thenReturn(bet);
+    when(store.findById(bet.betId())).thenReturn(bet);
+    when(risk.reserve(eq(bet.betId()), eq(bet.userId()), eq(Money.krw(1_000)), any()))
+        .thenReturn(
+            new RiskClient.Reservation(
+                RiskClient.ReservationState.RESERVED, expiresAt, "b".repeat(64)));
+
+    service(assembler, risk, store).place(command);
+
+    InOrder order = inOrder(risk, store);
+    order.verify(risk).reserve(eq(bet.betId()), eq(bet.userId()), eq(Money.krw(1_000)), any());
+    order
+        .verify(store)
+        .recordRiskReservation(bet.betId(), expiresAt, "b".repeat(64), false, Instant.EPOCH);
+  }
+
+  @Test
+  void persistsRiskValidationAfterPendingCreation() {
+    BetAssembler assembler = mock(BetAssembler.class);
+    RiskClient risk = mock(RiskClient.class);
+    BetStore store = mock(BetStore.class);
+    PlaceBetCommand command = command();
+    Bet bet = pending(command);
+    when(assembler.assemble(eq(command), any(String.class))).thenReturn(bet);
+    when(store.findById(bet.betId())).thenReturn(bet);
+    when(risk.reserve(any(), any(), any(), any()))
+        .thenThrow(new com.sportsbook.betting.error.ValidationFailedException("invalid"));
+    when(store.rejectAtCreation(any(), any(), any(), any())).thenReturn(bet);
+
+    catchThrowableOfType(
+        () -> service(assembler, risk, store).place(command),
+        com.sportsbook.betting.error.ValidationFailedException.class);
+
+    org.mockito.Mockito.verify(store)
+        .rejectAtCreation(bet.betId(), ErrorCode.VALIDATION_FAILED, "invalid", Instant.EPOCH);
+  }
+
+  private static BetPlacementService service(
+      BetAssembler assembler, RiskClient risk, BetStore store) {
+    return new BetPlacementService(
+        assembler,
+        risk,
+        new SystemBetCalculator(),
+        store,
+        Clock.fixed(Instant.EPOCH, ZoneOffset.UTC));
   }
 
   @Test
@@ -56,8 +119,7 @@ class BetPlacementServiceTest {
             new com.sportsbook.betting.error.ValidationFailedException(
                 "Duplicate selection is not allowed"));
 
-    BetPlacementService service =
-        new BetPlacementService(assembler, store, Clock.fixed(Instant.EPOCH, ZoneOffset.UTC));
+    BetPlacementService service = service(assembler, mock(RiskClient.class), store);
     catchThrowableOfType(
         () -> service.place(command),
         com.sportsbook.betting.error.ValidationFailedException.class);
@@ -83,4 +145,23 @@ class BetPlacementServiceTest {
         Money.krw(1_000),
         IdempotencyKey.of("request-1"));
   }
+
+  private static Bet pending(PlaceBetCommand command) {
+    BetDraft draft =
+        new BetDraft(
+            UUID.randomUUID(),
+            command.userId(),
+            "B-2026-08-22-00000000",
+            command.slipType(),
+            command.unitStake(),
+            Money.krw(2_000),
+            command.idempotencyKey(),
+            RequestFingerprint.of(command),
+            Instant.EPOCH);
+    PlaceBetCommand.SelectionInput input = command.selections().get(0);
+    BetLeg leg =
+        BetLeg.create(
+            input.eventId(), input.marketId(), input.selectionId(), input.oddsAtSubmission());
+    return Bet.pending(draft, List.of(leg));
+  }
 }


## `feat(placement): confirm wallet from durable reservation`

diff --git a/src/main/java/com/sportsbook/betting/placement/BetPlacementService.java b/src/main/java/com/sportsbook/betting/placement/BetPlacementService.java
index 1446ce4..0a24ee0 100644
--- a/src/main/java/com/sportsbook/betting/placement/BetPlacementService.java
+++ b/src/main/java/com/sportsbook/betting/placement/BetPlacementService.java
@@ -2,16 +2,22 @@ package com.sportsbook.betting.placement;
 
 import com.sportsbook.betting.client.RiskClient;
 import com.sportsbook.betting.client.RiskClient.Reservation;
+import com.sportsbook.betting.client.WalletClient;
+import com.sportsbook.betting.client.WalletOperationResponse;
 import com.sportsbook.betting.domain.Bet;
 import com.sportsbook.betting.domain.BetLeg;
-import com.sportsbook.betting.domain.PlacementPhase;
+import com.sportsbook.betting.domain.CompensationState;
 import com.sportsbook.betting.domain.SystemBetCalculator;
 import com.sportsbook.betting.error.BetPlacementException;
+import com.sportsbook.betting.error.DependencyUnavailableException;
 import com.sportsbook.betting.error.DuplicateBetException;
+import com.sportsbook.betting.error.InsufficientBalanceException;
 import com.sportsbook.betting.error.MarketClosedException;
 import com.sportsbook.betting.error.OddsDriftException;
 import com.sportsbook.betting.error.RiskLimitException;
 import com.sportsbook.betting.error.ValidationFailedException;
+import com.sportsbook.betting.error.WalletRejectedException;
+import com.sportsbook.protocol.domain.BetStatus;
 import java.time.Clock;
 import java.util.List;
 import java.util.Optional;
@@ -22,8 +28,11 @@ import org.springframework.stereotype.Service;
 @Service
 public class BetPlacementService {
 
+  private static final int MAX_ADVANCE_STEPS = 8;
+
   private final BetAssembler assembler;
   private final RiskClient risk;
+  private final WalletClient wallet;
   private final SystemBetCalculator calculator;
   private final BetStore store;
   private final Clock clock;
@@ -31,11 +40,13 @@ public class BetPlacementService {
   public BetPlacementService(
       BetAssembler assembler,
       RiskClient risk,
+      WalletClient wallet,
       SystemBetCalculator calculator,
       BetStore store,
       Clock clock) {
     this.assembler = assembler;
     this.risk = risk;
+    this.wallet = wallet;
     this.calculator = calculator;
     this.store = store;
     this.clock = clock;
@@ -79,16 +90,50 @@ public class BetPlacementService {
     } catch (DataIntegrityViolationException collision) {
       return replayKnown(key, command, fingerprint, collision);
     }
-    return advance(bet.betId(), true);
+    return advance(bet.betId(), false, true);
+  }
+
+  private Bet advance(UUID betId, boolean recovery, boolean surfaceRejection) {
+    for (int step = 0; step < MAX_ADVANCE_STEPS; step++) {
+      Bet current = store.findById(betId);
+      if (current.status() != BetStatus.PENDING) {
+        return current;
+      }
+      if (current.compensationState() != CompensationState.NONE) {
+        return current;
+      }
+      try {
+        switch (current.placementPhase()) {
+          case CREATED -> reserveRisk(current, surfaceRejection);
+          case RISK_RESERVED -> confirmWallet(current, recovery);
+          default -> {
+            return current;
+          }
+        }
+      } catch (DependencyUnavailableException unavailable) {
+        return store.findById(betId);
+      }
+    }
+    return store.findById(betId);
   }
 
-  private Bet advance(UUID betId, boolean surfaceRejection) {
-    Bet current = store.findById(betId);
-    if (current.placementPhase() == PlacementPhase.CREATED) {
-      reserveRisk(current, surfaceRejection);
-      return store.findById(betId);
+  private void confirmWallet(Bet bet, boolean recovery) {
+    try {
+      UUID operationId =
+          recovery
+              ? wallet
+                  .findDebit(bet.betId(), bet.userId(), totalExposure(bet))
+                  .map(WalletOperationResponse::operationGroupId)
+                  .orElse(null)
+              : null;
+      if (operationId == null) {
+        operationId = wallet.debit(bet.betId(), bet.userId(), totalExposure(bet));
+      }
+      store.confirmWallet(bet.betId(), operationId, clock.instant());
+    } catch (InsufficientBalanceException | WalletRejectedException rejection) {
+      store.requireRiskRelease(
+          bet.betId(), rejection.errorCode(), rejection.getMessage(), clock.instant());
     }
-    return current;
   }
 
   private void reserveRisk(Bet bet, boolean surfaceRejection) {


