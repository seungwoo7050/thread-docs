# 복구 부채와 FIFO 회수

## `feat(account): track recovery debt and queue sequence`

diff --git a/src/main/java/com/sportsbook/wallet/domain/Account.java b/src/main/java/com/sportsbook/wallet/domain/Account.java
index 51e4bbd..b0752ea 100644
--- a/src/main/java/com/sportsbook/wallet/domain/Account.java
+++ b/src/main/java/com/sportsbook/wallet/domain/Account.java
@@ -121,6 +121,45 @@ public class Account {
     return nextAdjustmentSequence;
   }
 
+  public boolean isOutboundFrozen() {
+    return recoveryDebtAmount.signum() > 0;
+  }
+
+  public long queueRecoveryDebt(Money amount, Instant now) {
+    requirePositive(amount);
+    requireSameCurrency(amount);
+    Objects.requireNonNull(now, "now");
+    long sequence = nextAdjustmentSequence;
+    long nextSequence = Math.incrementExact(sequence);
+    BigInteger nextDebt = recoveryDebtAmount.add(BigInteger.valueOf(amount.amount()));
+    nextAdjustmentSequence = nextSequence;
+    recoveryDebtAmount = nextDebt;
+    if (recoveryFrozenAt == null) {
+      recoveryFrozenAt = now;
+    }
+    updatedAt = now;
+    return sequence;
+  }
+
+  public void recoverAvailable(Money amount, Instant now) {
+    requirePositive(amount);
+    requireSameCurrency(amount);
+    Objects.requireNonNull(now, "now");
+    if (available.amount() < amount.amount()) {
+      throw new InsufficientBalanceException(userId, amount, available.toMoney());
+    }
+    BigInteger nextDebt = recoveryDebtAmount.subtract(BigInteger.valueOf(amount.amount()));
+    if (nextDebt.signum() < 0) {
+      throw new IllegalArgumentException("Recovery payment exceeds outstanding debt");
+    }
+    available = new EmbeddedMoney(available.amount() - amount.amount(), currency());
+    recoveryDebtAmount = nextDebt;
+    if (nextDebt.signum() == 0) {
+      recoveryFrozenAt = null;
+    }
+    updatedAt = now;
+  }
+
   public void increaseAvailable(Money amount, Instant now) {
     requirePositive(amount);
     requireSameCurrency(amount);


## `feat(account): block outbound spending during recovery`

diff --git a/src/main/java/com/sportsbook/wallet/domain/Account.java b/src/main/java/com/sportsbook/wallet/domain/Account.java
index b0752ea..b2a6fd6 100644
--- a/src/main/java/com/sportsbook/wallet/domain/Account.java
+++ b/src/main/java/com/sportsbook/wallet/domain/Account.java
@@ -2,6 +2,7 @@ package com.sportsbook.wallet.domain;
 
 import com.sportsbook.protocol.value.Currency;
 import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.error.AccountRecoveryBlockedException;
 import com.sportsbook.wallet.domain.error.BalanceLimitExceededException;
 import com.sportsbook.wallet.domain.error.CurrencyMismatchException;
 import com.sportsbook.wallet.domain.error.InsufficientBalanceException;
@@ -125,6 +126,12 @@ public class Account {
     return recoveryDebtAmount.signum() > 0;
   }
 
+  public void requireOutboundAllowed() {
+    if (isOutboundFrozen()) {
+      throw new AccountRecoveryBlockedException(userId);
+    }
+  }
+
   public long queueRecoveryDebt(Money amount, Instant now) {
     requirePositive(amount);
     requireSameCurrency(amount);
@@ -176,6 +183,7 @@ public class Account {
     requirePositive(amount);
     requireSameCurrency(amount);
     Objects.requireNonNull(now, "now");
+    requireOutboundAllowed();
     if (available.amount() < amount.amount()) {
       throw new InsufficientBalanceException(userId, amount, available.toMoney());
     }
@@ -187,6 +195,7 @@ public class Account {
     requirePositive(amount);
     requireSameCurrency(amount);
     Objects.requireNonNull(now, "now");
+    requireOutboundAllowed();
     if (available.amount() < amount.amount()) {
       throw new InsufficientBalanceException(userId, amount, available.toMoney());
     }
diff --git a/src/main/java/com/sportsbook/wallet/domain/error/AccountRecoveryBlockedException.java b/src/main/java/com/sportsbook/wallet/domain/error/AccountRecoveryBlockedException.java
new file mode 100644
index 0000000..37a6f0d
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/domain/error/AccountRecoveryBlockedException.java
@@ -0,0 +1,18 @@
+package com.sportsbook.wallet.domain.error;
+
+import java.util.Objects;
+import java.util.UUID;
+
+/** Signals that recovery debt has frozen new outbound spending for an account. */
+public final class AccountRecoveryBlockedException extends RuntimeException {
+  private final UUID userId;
+
+  public AccountRecoveryBlockedException(UUID userId) {
+    super("Account is frozen for recovery: " + Objects.requireNonNull(userId, "userId"));
+    this.userId = userId;
+  }
+
+  public UUID userId() {
+    return userId;
+  }
+}


## `test(account): allow inflows and forfeit while recovery frozen`

diff --git a/src/test/java/com/sportsbook/wallet/domain/AccountRecoveryFreezeTest.java b/src/test/java/com/sportsbook/wallet/domain/AccountRecoveryFreezeTest.java
new file mode 100644
index 0000000..7bff626
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/domain/AccountRecoveryFreezeTest.java
@@ -0,0 +1,47 @@
+package com.sportsbook.wallet.domain;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.assertj.core.api.Assertions.assertThatThrownBy;
+
+import com.sportsbook.protocol.value.Currency;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.error.AccountRecoveryBlockedException;
+import java.time.Instant;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class AccountRecoveryFreezeTest {
+  private static final UUID USER_ID = UUID.fromString("019b76da-a000-7000-8000-000000000112");
+  private static final Instant NOW = Instant.parse("2026-01-01T00:00:00Z");
+
+  @Test
+  void rejectsOutboundAuthorizationWhileRecoveryDebtExists() {
+    Account account = Account.openFor(USER_ID, Currency.KRW, NOW);
+    account.queueRecoveryDebt(Money.krw(20L), NOW.plusSeconds(1));
+
+    assertThatThrownBy(account::requireOutboundAllowed)
+        .isInstanceOf(AccountRecoveryBlockedException.class)
+        .extracting("userId")
+        .isEqualTo(USER_ID);
+    assertThatThrownBy(() -> account.decreaseAvailable(Money.krw(1L), NOW.plusSeconds(2)))
+        .isInstanceOf(AccountRecoveryBlockedException.class);
+    assertThatThrownBy(() -> account.moveAvailableToLocked(Money.krw(1L), NOW.plusSeconds(2)))
+        .isInstanceOf(AccountRecoveryBlockedException.class);
+  }
+
+  @Test
+  void allowsInflowsAndForfeitWhileOutboundIsFrozen() {
+    Account account = Account.openFor(USER_ID, Currency.KRW, NOW);
+    account.increaseAvailable(Money.krw(100L), NOW.plusSeconds(1));
+    account.moveAvailableToLocked(Money.krw(50L), NOW.plusSeconds(2));
+    account.queueRecoveryDebt(Money.krw(20L), NOW.plusSeconds(3));
+
+    account.increaseAvailable(Money.krw(10L), NOW.plusSeconds(4));
+    account.moveLockedToAvailable(Money.krw(10L), NOW.plusSeconds(5));
+    account.forfeitLocked(Money.krw(10L), NOW.plusSeconds(6));
+
+    assertThat(account.available()).isEqualTo(Money.krw(70L));
+    assertThat(account.locked()).isEqualTo(Money.krw(30L));
+    assertThat(account.isOutboundFrozen()).isTrue();
+  }
+}


## `feat(recovery): wake the oldest blocked adjustment`

diff --git a/src/main/java/com/sportsbook/wallet/service/RecoveryWakeService.java b/src/main/java/com/sportsbook/wallet/service/RecoveryWakeService.java
new file mode 100644
index 0000000..b2b6fc4
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/service/RecoveryWakeService.java
@@ -0,0 +1,33 @@
+package com.sportsbook.wallet.service;
+
+import com.sportsbook.wallet.domain.Account;
+import com.sportsbook.wallet.domain.WalletAdjustment;
+import com.sportsbook.wallet.persistence.DatabaseClock;
+import com.sportsbook.wallet.persistence.WalletAdjustmentRepository;
+import org.springframework.stereotype.Component;
+
+/** Moves only a frozen account's FIFO head forward after inflow mutation. */
+@Component
+public class RecoveryWakeService {
+  private final WalletAdjustmentRepository adjustments;
+  private final DatabaseClock databaseClock;
+
+  public RecoveryWakeService(WalletAdjustmentRepository adjustments, DatabaseClock databaseClock) {
+    this.adjustments = adjustments;
+    this.databaseClock = databaseClock;
+  }
+
+  public void wake(Account lockedAccount) {
+    if (!lockedAccount.isOutboundFrozen()) {
+      return;
+    }
+    WalletAdjustment head =
+        adjustments
+            .findOldestBlockedForUpdate(lockedAccount.userId())
+            .orElseThrow(
+                () ->
+                    new IllegalStateException(
+                        "Recovery debt has no FIFO head for " + lockedAccount.userId()));
+    head.wake(databaseClock.now());
+  }
+}


## `feat(recovery): wake blocked heads after wallet inflows`

diff --git a/src/main/java/com/sportsbook/wallet/service/WalletTransferExecutor.java b/src/main/java/com/sportsbook/wallet/service/WalletTransferExecutor.java
index 28de4e3..bf5adf9 100644
--- a/src/main/java/com/sportsbook/wallet/service/WalletTransferExecutor.java
+++ b/src/main/java/com/sportsbook/wallet/service/WalletTransferExecutor.java
@@ -31,6 +31,7 @@ public class WalletTransferExecutor {
   private final WalletTransferWriter transfers;
   private final WalletOutcomeResolver outcomes;
   private final OutboxAppender outboxAppender;
+  private final RecoveryWakeService recoveryWake;
   private final Clock clock;
   private final WalletEventFactory eventFactory = new WalletEventFactory();
 
@@ -40,12 +41,14 @@ public class WalletTransferExecutor {
       WalletTransferWriter transfers,
       WalletOutcomeResolver outcomes,
       OutboxAppender outboxAppender,
+      RecoveryWakeService recoveryWake,
       Clock clock) {
     this.accounts = accounts;
     this.operations = operations;
     this.transfers = transfers;
     this.outcomes = outcomes;
     this.outboxAppender = outboxAppender;
+    this.recoveryWake = recoveryWake;
     this.clock = clock;
   }
 
@@ -150,6 +153,9 @@ public class WalletTransferExecutor {
     try {
       Account account = lockAccount(userId, amount);
       WalletTransferPlan plan = mutation.apply(account, now);
+      if (kind == WalletOperationKind.DEPOSIT || credit != null) {
+        recoveryWake.wake(account);
+      }
       receipt =
           transfers.writeReceipt(
               plan.destination(), plan.source(), amount, plan.reason(), key, userId, now);


## `feat(recovery): define full debt collection transfers`

diff --git a/src/main/java/com/sportsbook/wallet/service/AdjustmentTransfers.java b/src/main/java/com/sportsbook/wallet/service/AdjustmentTransfers.java
index 73522cb..345f1f1 100644
--- a/src/main/java/com/sportsbook/wallet/service/AdjustmentTransfers.java
+++ b/src/main/java/com/sportsbook/wallet/service/AdjustmentTransfers.java
@@ -1,10 +1,13 @@
 package com.sportsbook.wallet.service;
 
+import com.sportsbook.protocol.value.Money;
 import com.sportsbook.wallet.domain.Account;
+import com.sportsbook.wallet.domain.AdjustmentStatus;
 import com.sportsbook.wallet.domain.BalanceBucket;
 import com.sportsbook.wallet.domain.LedgerEntry;
 import com.sportsbook.wallet.domain.LedgerReason;
 import com.sportsbook.wallet.domain.SystemAccountIds;
+import com.sportsbook.wallet.domain.WalletAdjustment;
 import com.sportsbook.wallet.service.command.AdjustmentCommand;
 import java.time.Instant;
 
@@ -32,5 +35,19 @@ final class AdjustmentTransfers {
         LedgerReason.BET_ADJUSTMENT);
   }
 
+  static WalletTransferPlan recover(Account account, WalletAdjustment proof, Instant now) {
+    if (proof.status() != AdjustmentStatus.BLOCKED || proof.deltaAmount() >= 0L) {
+      throw new IllegalArgumentException("Recovery requires a blocked negative adjustment");
+    }
+    if (!account.userId().equals(proof.userId())) {
+      throw new IllegalArgumentException("Recovery account does not own the adjustment");
+    }
+    account.recoverAvailable(new Money(-proof.deltaAmount(), proof.currency()), now);
+    return new WalletTransferPlan(
+        new LedgerEntry.TransferLeg(SystemAccountIds.HOUSE, BalanceBucket.AVAILABLE),
+        new LedgerEntry.TransferLeg(proof.userId(), BalanceBucket.AVAILABLE),
+        LedgerReason.BET_ADJUSTMENT);
+  }
+
   private AdjustmentTransfers() {}
 }


## `feat(recovery): bound automatic retry delays`

diff --git a/src/main/java/com/sportsbook/wallet/service/RecoveryRetryPolicy.java b/src/main/java/com/sportsbook/wallet/service/RecoveryRetryPolicy.java
new file mode 100644
index 0000000..0b86508
--- /dev/null
+++ b/src/main/java/com/sportsbook/wallet/service/RecoveryRetryPolicy.java
@@ -0,0 +1,50 @@
+package com.sportsbook.wallet.service;
+
+import java.time.Duration;
+import java.time.Instant;
+import java.util.Objects;
+import org.springframework.beans.factory.annotation.Value;
+import org.springframework.stereotype.Component;
+
+/** Exponential recovery retry delay with a bounded configured cap. */
+@Component
+public class RecoveryRetryPolicy {
+  private static final int MAX_EXPONENT = 62;
+
+  private final Duration baseDelay;
+  private final Duration maximumDelay;
+
+  public RecoveryRetryPolicy(
+      @Value("${wallet.recovery.retry-base:PT1S}") Duration baseDelay,
+      @Value("${wallet.recovery.retry-cap:PT60S}") Duration maximumDelay) {
+    this.baseDelay = positive(baseDelay, "baseDelay");
+    this.maximumDelay = positive(maximumDelay, "maximumDelay");
+    if (baseDelay.compareTo(maximumDelay) > 0) {
+      throw new IllegalArgumentException("baseDelay must not exceed maximumDelay");
+    }
+  }
+
+  public Instant retryAt(Instant attemptedAt, int completedRetries) {
+    Objects.requireNonNull(attemptedAt, "attemptedAt");
+    if (completedRetries < 0) {
+      throw new IllegalArgumentException("completedRetries must be nonnegative");
+    }
+    long multiplier = 1L << Math.min(completedRetries, MAX_EXPONENT);
+    Duration delay;
+    try {
+      Duration candidate = baseDelay.multipliedBy(multiplier);
+      delay = candidate.compareTo(maximumDelay) < 0 ? candidate : maximumDelay;
+    } catch (ArithmeticException overflow) {
+      delay = maximumDelay;
+    }
+    return attemptedAt.plus(delay);
+  }
+
+  private static Duration positive(Duration duration, String name) {
+    Objects.requireNonNull(duration, name);
+    if (duration.toMillis() < 1L) {
+      throw new IllegalArgumentException(name + " must be positive");
+    }
+    return duration;
+  }
+}


## `test(recovery): preserve underfunded FIFO heads`

diff --git a/src/test/java/com/sportsbook/wallet/service/RecoveryHeadProcessorFundsTest.java b/src/test/java/com/sportsbook/wallet/service/RecoveryHeadProcessorFundsTest.java
new file mode 100644
index 0000000..58e6dd8
--- /dev/null
+++ b/src/test/java/com/sportsbook/wallet/service/RecoveryHeadProcessorFundsTest.java
@@ -0,0 +1,74 @@
+package com.sportsbook.wallet.service;
+
+import static org.assertj.core.api.Assertions.assertThat;
+import static org.mockito.Mockito.mock;
+import static org.mockito.Mockito.verifyNoInteractions;
+import static org.mockito.Mockito.when;
+
+import com.sportsbook.protocol.value.IdempotencyKey;
+import com.sportsbook.protocol.value.Money;
+import com.sportsbook.wallet.domain.Account;
+import com.sportsbook.wallet.domain.WalletAdjustment;
+import com.sportsbook.wallet.domain.WalletCaller;
+import com.sportsbook.wallet.domain.WalletOperation;
+import com.sportsbook.wallet.persistence.WalletAdjustmentRepository;
+import com.sportsbook.wallet.persistence.WalletOperationRepository;
+import com.sportsbook.wallet.service.command.AdjustmentCommand;
+import java.math.BigInteger;
+import java.time.Instant;
+import java.util.UUID;
+import org.junit.jupiter.api.Test;
+
+class RecoveryHeadProcessorFundsTest {
+  private static final UUID USER_ID = UUID.fromString("019b76da-a000-7000-8000-000000000183");
+  private static final Instant NOW = Instant.parse("2026-01-01T00:00:00Z");
+
+  @Test
+  void underfundedHeadOnlyReceivesDurableBackoff() {
+    RecoveryRetryPolicy retries = mock(RecoveryRetryPolicy.class);
+    WalletTransferWriter transfers = mock(WalletTransferWriter.class);
+    WalletAdjustmentRepository adjustments = mock(WalletAdjustmentRepository.class);
+    WalletOperationRepository operations = mock(WalletOperationRepository.class);
+    RecoveryHeadProcessor processor =
+        new RecoveryHeadProcessor(retries, transfers, adjustments, operations);
+    AdjustmentCommand command = command();
+    Account account = Account.openFor(USER_ID, Money.krw(0L).currency(), NOW);
+    account.queueRecoveryDebt(command.absoluteDelta(), NOW);
+    account.increaseAvailable(Money.krw(100L), NOW);
+    WalletAdjustment proof = WalletAdjustment.blocked(command, 1L, NOW);
+    WalletOperation operation = blockedOperation(command);
+    when(retries.retryAt(NOW.plusSeconds(2L), 0)).thenReturn(NOW.plusSeconds(3L));
+
+    RecoveryHeadProcessor.Result result =
+        processor.process(RecoveryClaim.locked(account, proof, operation), NOW.plusSeconds(2L));
+
+    assertThat(result).isEqualTo(RecoveryHeadProcessor.Result.DEFERRED_FUNDS);
+    assertThat(proof.retryCount()).isEqualTo(1);
+    assertThat(proof.nextAttemptAt()).isEqualTo(NOW.plusSeconds(3L));
+    assertThat(account.available()).isEqualTo(Money.krw(100L));
+    assertThat(account.recoveryDebtAmount()).isEqualTo(BigInteger.valueOf(300L));
+    verifyNoInteractions(transfers, adjustments, operations);
+  }
+
+  private AdjustmentCommand command() {
+    UUID revisionId = UUID.fromString("019b76da-a000-7000-8000-000000000184");
+    return new AdjustmentCommand(
+        revisionId,
+        UUID.fromString("019b76da-a000-7000-8000-000000000185"),
+        1L,
+        USER_ID,
+        Money.krw(500L),
+        Money.krw(200L),
+        IdempotencyKey.of("settlement:revision:" + revisionId));
+  }
+
+  private WalletOperation blockedOperation(AdjustmentCommand command) {
+    return WalletOperation.blockedFunds(
+        command.idempotencyKey(),
+        WalletCaller.SETTLEMENT,
+        USER_ID,
+        command.absoluteDelta(),
+        "b".repeat(64),
+        NOW);
+  }
+}


## `feat(recovery): lock due accounts before FIFO operations`

diff --git a/src/main/java/com/sportsbook/wallet/persistence/AccountRepository.java b/src/main/java/com/sportsbook/wallet/persistence/AccountRepository.java
index f22b353..0667c9a 100644
--- a/src/main/java/com/sportsbook/wallet/persistence/AccountRepository.java
+++ b/src/main/java/com/sportsbook/wallet/persistence/AccountRepository.java
@@ -15,4 +15,28 @@ public interface AccountRepository extends JpaRepository<Account, UUID> {
   @Lock(LockModeType.PESSIMISTIC_WRITE)
   @Query("select account from Account account where account.userId = :userId")
   Optional<Account> findByUserIdForUpdate(@Param("userId") UUID userId);
+
+  @Query(
+      value =
+          """
+          WITH db_clock AS (SELECT clock_timestamp() AS now)
+          SELECT a.*
+          FROM wallet_adjustment h
+          JOIN account a ON a.user_id = h.user_id
+          CROSS JOIN db_clock c
+          WHERE h.status = 'BLOCKED'
+            AND h.next_attempt_at <= c.now
+            AND a.recovery_debt_amount > 0
+            AND NOT EXISTS (
+              SELECT 1 FROM wallet_adjustment older
+              WHERE older.user_id = h.user_id
+                AND older.status = 'BLOCKED'
+                AND older.queue_sequence < h.queue_sequence
+            )
+          ORDER BY h.next_attempt_at, h.user_id
+          FOR UPDATE OF a SKIP LOCKED
+          LIMIT 1
+          """,
+      nativeQuery = true)
+  Optional<Account> lockNextDueRecoveryAccount();
 }
diff --git a/src/main/java/com/sportsbook/wallet/persistence/WalletOperationRepository.java b/src/main/java/com/sportsbook/wallet/persistence/WalletOperationRepository.java
index 9d29de8..e7fc9fe 100644
--- a/src/main/java/com/sportsbook/wallet/persistence/WalletOperationRepository.java
+++ b/src/main/java/com/sportsbook/wallet/persistence/WalletOperationRepository.java
@@ -1,7 +1,17 @@
 package com.sportsbook.wallet.persistence;
 
 import com.sportsbook.wallet.domain.WalletOperation;
+import jakarta.persistence.LockModeType;
+import java.util.Optional;
 import org.springframework.data.jpa.repository.JpaRepository;
+import org.springframework.data.jpa.repository.Lock;
+import org.springframework.data.jpa.repository.Query;
+import org.springframework.data.repository.query.Param;
 
-/** Durable operation outcomes. Reads intentionally remain lock-free because rows are terminal. */
-public interface WalletOperationRepository extends JpaRepository<WalletOperation, String> {}
+/** Durable outcomes with an explicit recovery lock for mutable blocked operations. */
+public interface WalletOperationRepository extends JpaRepository<WalletOperation, String> {
+
+  @Lock(LockModeType.PESSIMISTIC_WRITE)
+  @Query("select operation from WalletOperation operation where operation.idempotencyKey = :key")
+  Optional<WalletOperation> findByIdForUpdate(@Param("key") String key);
+}


