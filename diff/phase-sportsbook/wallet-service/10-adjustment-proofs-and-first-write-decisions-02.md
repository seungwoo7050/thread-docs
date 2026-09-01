## `feat(adjustment): wake blocked FIFO proofs`

diff --git a/src/main/java/com/sportsbook/wallet/domain/WalletAdjustment.java b/src/main/java/com/sportsbook/wallet/domain/WalletAdjustment.java
index 01b314d..c4c03d2 100644
--- a/src/main/java/com/sportsbook/wallet/domain/WalletAdjustment.java
+++ b/src/main/java/com/sportsbook/wallet/domain/WalletAdjustment.java
@@ -122,6 +122,17 @@ public class WalletAdjustment {
     return proof;
   }
 
+  public void wake(Instant now) {
+    if (status != AdjustmentStatus.BLOCKED) {
+      throw new IllegalStateException("Only blocked adjustments can be woken");
+    }
+    Objects.requireNonNull(now, "now");
+    if (now.isBefore(nextAttemptAt)) {
+      nextAttemptAt = now;
+    }
+    updatedAt = now;
+  }
+
   public UUID revisionId() {
     return revisionId;
   }


## `feat(adjustment): define correction ledger transfers`

diff --git a/src/main/java/com/sportsbook/wallet/service/AdjustmentTransfers.java b/src/main/java/com/sportsbook/wallet/service/AdjustmentTransfers.java
new file mode 100644
index 0000000..73522cb
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/service/AdjustmentTransfers.java
@@ -0,0 +1,36 @@
+package com.sportsbook.wallet.service;
+
+import com.sportsbook.wallet.domain.Account;
+import com.sportsbook.wallet.domain.BalanceBucket;
+import com.sportsbook.wallet.domain.LedgerEntry;
+import com.sportsbook.wallet.domain.LedgerReason;
+import com.sportsbook.wallet.domain.SystemAccountIds;
+import com.sportsbook.wallet.service.command.AdjustmentCommand;
+import java.time.Instant;
+
+/** Mutates an account and describes the matching adjustment ledger topology. */
+final class AdjustmentTransfers {
+  static WalletTransferPlan increase(Account account, AdjustmentCommand command, Instant now) {
+    if (command.deltaAmount() <= 0L) {
+      throw new IllegalArgumentException("Increase transfer requires a positive delta");
+    }
+    account.increaseAvailable(command.absoluteDelta(), now);
+    return new WalletTransferPlan(
+        new LedgerEntry.TransferLeg(command.userId(), BalanceBucket.AVAILABLE),
+        new LedgerEntry.TransferLeg(SystemAccountIds.HOUSE, BalanceBucket.AVAILABLE),
+        LedgerReason.BET_ADJUSTMENT);
+  }
+
+  static WalletTransferPlan decrease(Account account, AdjustmentCommand command, Instant now) {
+    if (command.deltaAmount() >= 0L) {
+      throw new IllegalArgumentException("Decrease transfer requires a negative delta");
+    }
+    account.decreaseAvailable(command.absoluteDelta(), now);
+    return new WalletTransferPlan(
+        new LedgerEntry.TransferLeg(SystemAccountIds.HOUSE, BalanceBucket.AVAILABLE),
+        new LedgerEntry.TransferLeg(command.userId(), BalanceBucket.AVAILABLE),
+        LedgerReason.BET_ADJUSTMENT);
+  }
+
+  private AdjustmentTransfers() {}
+}


## `feat(adjustment): apply positive corrections atomically`

diff --git a/src/main/java/com/sportsbook/wallet/service/AdjustmentProofWriter.java b/src/main/java/com/sportsbook/wallet/service/AdjustmentProofWriter.java
new file mode 100644
index 0000000..104f4de
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/service/AdjustmentProofWriter.java
@@ -0,0 +1,54 @@
+package com.sportsbook.wallet.service;
+
+import com.sportsbook.wallet.domain.Account;
+import com.sportsbook.wallet.domain.WalletAdjustment;
+import com.sportsbook.wallet.domain.WalletCaller;
+import com.sportsbook.wallet.domain.WalletOperation;
+import com.sportsbook.wallet.domain.WalletOperationKind;
+import com.sportsbook.wallet.persistence.WalletAdjustmentRepository;
+import com.sportsbook.wallet.service.command.AdjustmentCommand;
+import java.time.Instant;
+import java.util.Optional;
+import org.springframework.stereotype.Component;
+
+/** Persists adjustment ledger effects, proof, and operation outcome in the caller transaction. */
+@Component
+public class AdjustmentProofWriter {
+  private final WalletTransferWriter transfers;
+  private final WalletAdjustmentRepository adjustments;
+
+  public AdjustmentProofWriter(
+      WalletTransferWriter transfers, WalletAdjustmentRepository adjustments) {
+    this.transfers = transfers;
+    this.adjustments = adjustments;
+  }
+
+  public WalletOperation applyIncrease(
+      AdjustmentCommand command,
+      String fingerprint,
+      Account account,
+      Optional<WalletAdjustment> blockedHead,
+      Instant now) {
+    WalletTransferPlan plan = AdjustmentTransfers.increase(account, command, now);
+    WalletOperationResult result =
+        transfers.write(
+            plan.destination(),
+            plan.source(),
+            command.absoluteDelta(),
+            plan.reason(),
+            command.idempotencyKey(),
+            command.userId(),
+            now);
+    adjustments.save(WalletAdjustment.applied(command, result.operationGroupId(), result.at()));
+    blockedHead.ifPresent(head -> head.wake(now));
+    return WalletOperation.succeeded(
+        command.idempotencyKey(),
+        WalletCaller.SETTLEMENT,
+        WalletOperationKind.BET_ADJUSTMENT,
+        command.userId(),
+        command.absoluteDelta(),
+        fingerprint,
+        result.operationGroupId(),
+        now);
+  }
+}


## `feat(adjustment): apply affordable negative corrections`

diff --git a/src/main/java/com/sportsbook/wallet/service/AdjustmentProofWriter.java b/src/main/java/com/sportsbook/wallet/service/AdjustmentProofWriter.java
index 104f4de..45e5679 100644
--- a/src/main/java/com/sportsbook/wallet/service/AdjustmentProofWriter.java
+++ b/src/main/java/com/sportsbook/wallet/service/AdjustmentProofWriter.java
@@ -51,4 +51,28 @@ public class AdjustmentProofWriter {
         result.operationGroupId(),
         now);
   }
+
+  public WalletOperation applyDecrease(
+      AdjustmentCommand command, String fingerprint, Account account, Instant now) {
+    WalletTransferPlan plan = AdjustmentTransfers.decrease(account, command, now);
+    WalletOperationResult result =
+        transfers.write(
+            plan.destination(),
+            plan.source(),
+            command.absoluteDelta(),
+            plan.reason(),
+            command.idempotencyKey(),
+            command.userId(),
+            now);
+    adjustments.save(WalletAdjustment.applied(command, result.operationGroupId(), result.at()));
+    return WalletOperation.succeeded(
+        command.idempotencyKey(),
+        WalletCaller.SETTLEMENT,
+        WalletOperationKind.BET_ADJUSTMENT,
+        command.userId(),
+        command.absoluteDelta(),
+        fingerprint,
+        result.operationGroupId(),
+        now);
+  }
 }


## `feat(adjustment): queue blocked negative corrections`

diff --git a/src/main/java/com/sportsbook/wallet/service/AdjustmentProofWriter.java b/src/main/java/com/sportsbook/wallet/service/AdjustmentProofWriter.java
index 45e5679..151d26a 100644
--- a/src/main/java/com/sportsbook/wallet/service/AdjustmentProofWriter.java
+++ b/src/main/java/com/sportsbook/wallet/service/AdjustmentProofWriter.java
@@ -75,4 +75,17 @@ public class AdjustmentProofWriter {
         result.operationGroupId(),
         now);
   }
+
+  public WalletOperation block(
+      AdjustmentCommand command, String fingerprint, Account account, Instant now) {
+    long queueSequence = account.queueRecoveryDebt(command.absoluteDelta(), now);
+    adjustments.save(WalletAdjustment.blocked(command, queueSequence, now));
+    return WalletOperation.blockedFunds(
+        command.idempotencyKey(),
+        WalletCaller.SETTLEMENT,
+        command.userId(),
+        command.absoluteDelta(),
+        fingerprint,
+        now);
+  }
 }


## `feat(adjustment): persist rejected correction proofs`

diff --git a/src/main/java/com/sportsbook/wallet/service/AdjustmentProofWriter.java b/src/main/java/com/sportsbook/wallet/service/AdjustmentProofWriter.java
index 151d26a..ee06d7a 100644
--- a/src/main/java/com/sportsbook/wallet/service/AdjustmentProofWriter.java
+++ b/src/main/java/com/sportsbook/wallet/service/AdjustmentProofWriter.java
@@ -3,6 +3,7 @@ package com.sportsbook.wallet.service;
 import com.sportsbook.wallet.domain.Account;
 import com.sportsbook.wallet.domain.WalletAdjustment;
 import com.sportsbook.wallet.domain.WalletCaller;
+import com.sportsbook.wallet.domain.WalletFailureSnapshot;
 import com.sportsbook.wallet.domain.WalletOperation;
 import com.sportsbook.wallet.domain.WalletOperationKind;
 import com.sportsbook.wallet.persistence.WalletAdjustmentRepository;
@@ -88,4 +89,19 @@ public class AdjustmentProofWriter {
         fingerprint,
         now);
   }
+
+  public WalletOperation reject(
+      AdjustmentCommand command, String fingerprint, RuntimeException failure, Instant now) {
+    WalletFailureSnapshot snapshot = WalletFailureMapper.snapshot(failure, command.absoluteDelta());
+    adjustments.save(WalletAdjustment.rejected(command, now));
+    return WalletOperation.rejected(
+        command.idempotencyKey(),
+        WalletCaller.SETTLEMENT,
+        WalletOperationKind.BET_ADJUSTMENT,
+        command.userId(),
+        command.absoluteDelta(),
+        fingerprint,
+        snapshot,
+        now);
+  }
 }


## `feat(adjustment): choose locked correction outcomes`

diff --git a/src/main/java/com/sportsbook/wallet/service/AdjustmentFirstWriter.java b/src/main/java/com/sportsbook/wallet/service/AdjustmentFirstWriter.java
new file mode 100644
index 0000000..dee2e18
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/service/AdjustmentFirstWriter.java
@@ -0,0 +1,88 @@
+package com.sportsbook.wallet.service;
+
+import com.sportsbook.wallet.domain.Account;
+import com.sportsbook.wallet.domain.WalletAdjustment;
+import com.sportsbook.wallet.domain.WalletOperation;
+import com.sportsbook.wallet.domain.error.AccountNotFoundException;
+import com.sportsbook.wallet.domain.error.BalanceLimitExceededException;
+import com.sportsbook.wallet.domain.error.CurrencyMismatchException;
+import com.sportsbook.wallet.domain.error.IdempotencyConflictException;
+import com.sportsbook.wallet.persistence.AccountRepository;
+import com.sportsbook.wallet.persistence.AdjustmentPairLock;
+import com.sportsbook.wallet.persistence.DatabaseClock;
+import com.sportsbook.wallet.persistence.WalletAdjustmentRepository;
+import com.sportsbook.wallet.service.command.AdjustmentCommand;
+import java.time.Instant;
+import java.util.Optional;
+import org.springframework.stereotype.Component;
+
+/** Applies a first adjustment writer under the established idempotency-key transaction lock. */
+@Component
+public class AdjustmentFirstWriter {
+  private final AdjustmentPairLock pairLocks;
+  private final WalletAdjustmentRepository adjustments;
+  private final AccountRepository accounts;
+  private final DatabaseClock databaseClock;
+  private final AdjustmentProofWriter proofWriter;
+
+  public AdjustmentFirstWriter(
+      AdjustmentPairLock pairLocks,
+      WalletAdjustmentRepository adjustments,
+      AccountRepository accounts,
+      DatabaseClock databaseClock,
+      AdjustmentProofWriter proofWriter) {
+    this.pairLocks = pairLocks;
+    this.adjustments = adjustments;
+    this.accounts = accounts;
+    this.databaseClock = databaseClock;
+    this.proofWriter = proofWriter;
+  }
+
+  public WalletOperation write(AdjustmentCommand command, String fingerprint) {
+    pairLocks.acquire(command.betId(), command.revisionNumber());
+    if (adjustments
+        .findByBetIdAndRevisionNumber(command.betId(), command.revisionNumber())
+        .isPresent()) {
+      throw new IdempotencyConflictException(command.idempotencyKey());
+    }
+
+    Account account;
+    Optional<WalletAdjustment> blockedHead;
+    try {
+      account =
+          accounts
+              .findByUserIdForUpdate(command.userId())
+              .orElseThrow(() -> new AccountNotFoundException(command.userId()));
+      if (account.currency() != command.previousPayout().currency()) {
+        throw new CurrencyMismatchException(
+            account.currency(), command.previousPayout().currency());
+      }
+      blockedHead = adjustments.findOldestBlockedForUpdate(command.userId());
+      requireConsistentRecovery(account, blockedHead);
+    } catch (AccountNotFoundException | CurrencyMismatchException rejected) {
+      return proofWriter.reject(command, fingerprint, rejected, databaseClock.now());
+    }
+
+    Instant now = databaseClock.now();
+    try {
+      if (command.deltaAmount() > 0L) {
+        return proofWriter.applyIncrease(command, fingerprint, account, blockedHead, now);
+      }
+      if (blockedHead.isPresent()
+          || account.available().amount() < command.absoluteDelta().amount()) {
+        return proofWriter.block(command, fingerprint, account, now);
+      }
+      return proofWriter.applyDecrease(command, fingerprint, account, now);
+    } catch (BalanceLimitExceededException rejected) {
+      return proofWriter.reject(command, fingerprint, rejected, now);
+    }
+  }
+
+  private static void requireConsistentRecovery(
+      Account account, Optional<WalletAdjustment> blockedHead) {
+    if (account.isOutboundFrozen() != blockedHead.isPresent()) {
+      throw new IllegalStateException(
+          "Recovery debt and FIFO head disagree for " + account.userId());
+    }
+  }
+}


## `feat(adjustment): execute durable correction requests`

diff --git a/src/main/java/com/sportsbook/wallet/service/WalletAdjustmentService.java b/src/main/java/com/sportsbook/wallet/service/WalletAdjustmentService.java
new file mode 100644
index 0000000..fd8574b
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/service/WalletAdjustmentService.java
@@ -0,0 +1,56 @@
+package com.sportsbook.wallet.service;
+
+import com.sportsbook.wallet.domain.WalletAdjustment;
+import com.sportsbook.wallet.domain.WalletCaller;
+import com.sportsbook.wallet.domain.WalletOperation;
+import com.sportsbook.wallet.domain.WalletOperationKind;
+import com.sportsbook.wallet.domain.WalletOperationStatus;
+import com.sportsbook.wallet.domain.error.WalletRejectedException;
+import com.sportsbook.wallet.persistence.WalletAdjustmentRepository;
+import com.sportsbook.wallet.service.command.AdjustmentCommand;
+import org.springframework.stereotype.Service;
+
+/** Executes settlement corrections and returns their durable wallet proof. */
+@Service
+public class WalletAdjustmentService {
+  private final WalletOperationExecutor operations;
+  private final AdjustmentFirstWriter firstWriter;
+  private final WalletAdjustmentRepository adjustments;
+
+  public WalletAdjustmentService(
+      WalletOperationExecutor operations,
+      AdjustmentFirstWriter firstWriter,
+      WalletAdjustmentRepository adjustments) {
+    this.operations = operations;
+    this.firstWriter = firstWriter;
+    this.adjustments = adjustments;
+  }
+
+  public WalletAdjustment adjust(AdjustmentCommand command) {
+    OperationFingerprint fingerprint =
+        OperationFingerprint.adjustment(
+            WalletCaller.SETTLEMENT,
+            command.userId(),
+            command.previousPayout(),
+            command.newPayout(),
+            command.revisionId(),
+            command.betId(),
+            command.revisionNumber());
+    WalletOperation outcome =
+        operations.execute(
+            command.idempotencyKey(),
+            WalletCaller.SETTLEMENT,
+            WalletOperationKind.BET_ADJUSTMENT,
+            command.userId(),
+            command.absoluteDelta(),
+            fingerprint,
+            requestHash -> firstWriter.write(command, requestHash));
+    if (outcome.status() == WalletOperationStatus.REJECTED) {
+      throw new WalletRejectedException(outcome.failure());
+    }
+    return adjustments
+        .findById(command.revisionId())
+        .filter(proof -> proof.idempotencyKey().equals(command.idempotencyKey().value()))
+        .orElseThrow(() -> new IllegalStateException("Adjustment outcome has no matching proof"));
+  }
+}


## `test(adjustment): persist blocked proofs without partial ledgers`

diff --git a/src/test/java/com/sportsbook/wallet/service/AdjustmentProofWriterTest.java b/src/test/java/com/sportsbook/wallet/service/AdjustmentProofWriterTest.java
index 0af212f..8a50d4c 100644
--- a/src/test/java/com/sportsbook/wallet/service/AdjustmentProofWriterTest.java
+++ b/src/test/java/com/sportsbook/wallet/service/AdjustmentProofWriterTest.java
@@ -95,6 +95,28 @@ class AdjustmentProofWriterTest {
         .containsExactly(-300L, AdjustmentStatus.APPLIED);
   }
 
+  @Test
+  void queuesAnUnaffordableDecreaseWithoutMoneyOrLedgerChanges() {
+    Account account = Account.openFor(USER_ID, Currency.KRW, NOW);
+    account.increaseAvailable(Money.krw(200L), NOW);
+    accounts.saveAndFlush(account);
+    AdjustmentCommand command = command(1_000L, 700L);
+    Account locked = accounts.findByUserIdForUpdate(USER_ID).orElseThrow();
+
+    WalletOperation operation = writer.block(command, "c".repeat(64), locked, NOW);
+    operations.saveAndFlush(operation);
+    adjustments.flush();
+
+    assertThat(locked.available()).isEqualTo(Money.krw(200L));
+    assertThat(locked.recoveryDebtAmount()).isEqualTo(java.math.BigInteger.valueOf(300L));
+    assertThat(locked.isOutboundFrozen()).isTrue();
+    assertThat(ledger.findByIdempotencyKey(command.idempotencyKey().value())).isEmpty();
+    assertThat(adjustments.findById(REVISION_ID))
+        .get()
+        .extracting(proof -> proof.status(), proof -> proof.queueSequence())
+        .containsExactly(AdjustmentStatus.BLOCKED, 1L);
+  }
+
   private AdjustmentCommand command(long previous, long next) {
     return new AdjustmentCommand(
         REVISION_ID,


## `test(adjustment): fail closed on recovery drift`

diff --git a/src/test/java/com/sportsbook/wallet/service/AdjustmentFirstWriterTest.java b/src/test/java/com/sportsbook/wallet/service/AdjustmentFirstWriterTest.java
index 997169e..2ebc27a 100644
--- a/src/test/java/com/sportsbook/wallet/service/AdjustmentFirstWriterTest.java
+++ b/src/test/java/com/sportsbook/wallet/service/AdjustmentFirstWriterTest.java
@@ -1,6 +1,7 @@
 package com.sportsbook.wallet.service;
 
 import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatIllegalStateException;
 import static org.mockito.ArgumentMatchers.any;
 import static org.mockito.ArgumentMatchers.eq;
 import static org.mockito.Mockito.inOrder;
@@ -150,6 +151,33 @@ class AdjustmentFirstWriterTest {
     verify(proofWriter).reject(command, "1".repeat(64), overflow, NOW);
   }
 
+  @Test
+  void failsClosedWhenAFreezeHasNoBlockedHead() {
+    AdjustmentCommand command = command(700L, 1_000L);
+    when(adjustments.findByBetIdAndRevisionNumber(BET_ID, 1L)).thenReturn(Optional.empty());
+    when(accounts.findByUserIdForUpdate(USER_ID)).thenReturn(Optional.of(account));
+    when(account.currency()).thenReturn(Currency.KRW);
+    when(account.isOutboundFrozen()).thenReturn(true);
+    when(adjustments.findOldestBlockedForUpdate(USER_ID)).thenReturn(Optional.empty());
+
+    assertThatIllegalStateException()
+        .isThrownBy(() -> writer.write(command, "2".repeat(64)))
+        .withMessageContaining("FIFO head disagree");
+  }
+
+  @Test
+  void failsClosedWhenABlockedHeadHasNoFreeze() {
+    AdjustmentCommand command = command(700L, 1_000L);
+    when(adjustments.findByBetIdAndRevisionNumber(BET_ID, 1L)).thenReturn(Optional.empty());
+    when(accounts.findByUserIdForUpdate(USER_ID)).thenReturn(Optional.of(account));
+    when(account.currency()).thenReturn(Currency.KRW);
+    when(adjustments.findOldestBlockedForUpdate(USER_ID)).thenReturn(Optional.of(head));
+
+    assertThatIllegalStateException()
+        .isThrownBy(() -> writer.write(command, "3".repeat(64)))
+        .withMessageContaining("FIFO head disagree");
+  }
+
   private void prepareUnfrozenAccount(Money available) {
     when(adjustments.findByBetIdAndRevisionNumber(BET_ID, 1L)).thenReturn(Optional.empty());
     when(accounts.findByUserIdForUpdate(USER_ID)).thenReturn(Optional.of(account));
